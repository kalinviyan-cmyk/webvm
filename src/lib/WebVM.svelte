<script>
	import { onMount, tick } from 'svelte';
	import { get } from 'svelte/store';
	import Nav from 'labs/packages/global-navbar/src/Nav.svelte';
	import SideBar from '$lib/SideBar.svelte';
	import '$lib/global.css';
	import '@xterm/xterm/css/xterm.css'
	import '@fortawesome/fontawesome-free/css/all.min.css'
	import { networkInterface, startLogin } from '$lib/network.js'
	import { cpuActivity, diskActivity, cpuPercentage, diskLatency } from '$lib/activities.js'
	import { introMessage, errorMessage, unexpectedErrorMessage } from '$lib/messages.js'
	import { displayConfig, handleToolImpl } from '$lib/anthropic.js'
	import { tryPlausible } from '$lib/plausible.js'

	export let configObj = null;
	export let processCallback = null;
	export let cacheId = null;
	export let cpuActivityEvents = [];
	export let diskLatencies = [];
	export let activityEventsInterval = 0;

	var term = null;
	var cx = null;
	var fitAddon = null;
	var cxReadFunc = null;
	var blockCache = null;
	var processCount = 0;
	var curVT = 0;
	let guiReady = false;
	let startupStatus = 'Preparing browser terminal…';
	let startupError = '';
	let startupSlow = false;
	let startupWatchdog = null;
	var sideBarPinned = false;
	function writeData(buf, vt)
	{
		if(vt != 1)
			return;
		term.write(new Uint8Array(buf));
	}
	function readData(str)
	{
		if(cxReadFunc == null)
			return;
		for(var i=0;i<str.length;i++)
			cxReadFunc(str.charCodeAt(i));
	}
	function printMessage(msg)
	{
		for(var i=0;i<msg.length;i++)
			term.write(msg[i] + "\n");
	}
	function expireEvents(list, curTime, limitTime)
	{
		while(list.length > 1)
		{
			if(list[1].t < limitTime)
			{
				list.shift();
			}
			else
			{
				break;
			}
		}
	}
	function cleanupEvents()
	{
		var curTime = Date.now();
		var limitTime = curTime - 10000;
		expireEvents(cpuActivityEvents, curTime, limitTime);
		computeCpuActivity(curTime, limitTime);
		if(cpuActivityEvents.length == 0)
		{
			clearInterval(activityEventsInterval);
			activityEventsInterval = 0;
		}
	}
	function computeCpuActivity(curTime, limitTime)
	{
		var totalActiveTime = 0;
		var lastActiveTime = limitTime;
		var lastWasActive = false;
		for(var i=0;i<cpuActivityEvents.length;i++)
		{
			var e = cpuActivityEvents[i];
			// NOTE: The first event could be before the limit,
			//       we need at least one event to correctly mark
			//       active time when there is long time under load
			var eTime = e.t;
			if(eTime < limitTime)
				eTime = limitTime;
			if(e.state == "ready")
			{
				// Inactive state, add the time from lastActiveTime
				totalActiveTime += (eTime - lastActiveTime);
				lastWasActive = false;
			}
			else
			{
				// Active state
				lastActiveTime = eTime;
				lastWasActive = true;
			}
		}
		// Add the last interval if needed
		if(lastWasActive)
		{
			totalActiveTime += (curTime - lastActiveTime);
		}
		cpuPercentage.set(Math.ceil((totalActiveTime / 10000) * 100));
	}
	function hddCallback(state)
	{
		diskActivity.set(state != "ready");
	}
	function latencyCallback(latency)
	{
		diskLatencies.push(latency);
		if(diskLatencies.length > 30)
			diskLatencies.shift();
		// Average the latency over at most 30 blocks
		var total = 0;
		for(var i=0;i<diskLatencies.length;i++)
			total += diskLatencies[i];
		var avg = total / diskLatencies.length;
		diskLatency.set(Math.ceil(avg));
	}
	function cpuCallback(state)
	{
		cpuActivity.set(state != "ready");
		var curTime = Date.now();
		var limitTime = curTime - 10000;
		expireEvents(cpuActivityEvents, curTime, limitTime);
		cpuActivityEvents.push({t: curTime, state: state});
		computeCpuActivity(curTime, limitTime);
		// Start an interval timer to cleanup old samples when no further activity is received
		if(activityEventsInterval != 0)
			clearInterval(activityEventsInterval);
		activityEventsInterval = setInterval(cleanupEvents, 2000);
	}
	function computeXTermFontSize()
	{
		return parseInt(getComputedStyle(document.body).fontSize);
	}
	function setScreenSize(display)
	{
		var displayWidth = display.offsetWidth;
		var displayHeight = display.offsetHeight;
		// Android tablets with 4 GB RAM need a smaller framebuffer. The old
		// 1024x768 minimum could upscale small screens and waste memory/bandwidth.
		var isLowMemoryDevice = /Android/i.test(navigator.userAgent) ||
			(typeof navigator.deviceMemory === "number" && navigator.deviceMemory <= 4);
		var internalMult = 1.0;
		if(isLowMemoryDevice)
		{
			// Keep the aspect ratio and cap the framebuffer at about 0.4 megapixels.
			var maxPixels = 800 * 500;
			var pixelCount = displayWidth * displayHeight;
			if(pixelCount > maxPixels)
				internalMult = Math.sqrt(maxPixels / pixelCount);
		}
		else
		{
			// Preserve the original desktop minimum resolution on computers.
			var minWidth = 1024;
			var minHeight = 768;
			if(displayWidth < minWidth)
				internalMult = minWidth / displayWidth;
			if(displayHeight < minHeight)
				internalMult = Math.max(internalMult, minHeight / displayHeight);
		}
		var internalWidth = Math.max(1, Math.floor(displayWidth * internalMult));
		var internalHeight = Math.max(1, Math.floor(displayHeight * internalMult));
		cx.setKmsCanvas(display, internalWidth, internalHeight);
		// Compute the size to be used for AI screenshots
		var screenshotMult = 1.0;
		var maxWidth = 1024;
		var maxHeight = 768;
		if(internalWidth > maxWidth)
			screenshotMult = maxWidth / internalWidth;
		if(internalHeight > maxHeight)
			screenshotMult = Math.min(screenshotMult, maxHeight / internalHeight);
		var screenshotWidth = Math.floor(internalWidth * screenshotMult);
		var screenshotHeight = Math.floor(internalHeight * screenshotMult);
		// Track the state of the mouse as requested by the AI, to avoid losing the position due to user movement
		displayConfig.set({width: screenshotWidth, height: screenshotHeight, mouseMult: internalMult * screenshotMult});
	}
	var curInnerWidth = 0;
	var curInnerHeight = 0;
	function handleResize()
	{
		// Avoid spurious resize events caused by the soft keyboard
		if(curInnerWidth == window.innerWidth && curInnerHeight == window.innerHeight)
			return;
		curInnerWidth = window.innerWidth;
		curInnerHeight = window.innerHeight;
		triggerResize();
	}
	function triggerResize()
	{
		term.options.fontSize = computeXTermFontSize();
		fitAddon.fit();
		const display = document.getElementById("display");
		if(display)
			setScreenSize(display);
	}
	async function initTerminal()
	{
		// Fetch the terminal and VM engine modules concurrently instead of serially.
		startupStatus = 'Loading the Linux virtualization engine…';
		const [xtermModule, fitModule, linksModule, cheerpxPromise] = await Promise.all([
			import('@xterm/xterm'),
			import('@xterm/addon-fit'),
			import('@xterm/addon-web-links'),
			import('@leaningtech/cheerpx')
		]);
		const { Terminal } = xtermModule;
		const { FitAddon } = fitModule;
		const { WebLinksAddon } = linksModule;
		term = new Terminal({cursorBlink:true, convertEol:true, fontFamily:"monospace", fontWeight: 400, fontWeightBold: 700, fontSize: computeXTermFontSize()});
		fitAddon = new FitAddon();
		term.loadAddon(fitAddon);
		var linkAddon = new WebLinksAddon();
		term.loadAddon(linkAddon);
		const consoleDiv = document.getElementById("console");
		term.open(consoleDiv);
		term.scrollToTop();
		fitAddon.fit();
		window.addEventListener("resize", handleResize);
		term.focus();
		term.onData(readData);
		// Avoid undesired default DnD handling
		function preventDefaults (e) {
			e.preventDefault()
			e.stopPropagation()
		}
		consoleDiv.addEventListener("dragover", preventDefaults, false);
		consoleDiv.addEventListener("dragenter", preventDefaults, false);
		consoleDiv.addEventListener("dragleave", preventDefaults, false);
		consoleDiv.addEventListener("drop", preventDefaults, false);
		curInnerWidth = window.innerWidth;
		curInnerHeight = window.innerHeight;
		if(configObj.printIntro)
			printMessage(introMessage);
		try
		{
			startupStatus = 'Connecting to the Alpine Linux disk…';
			await initCheerpX(cheerpxPromise);
		}
		catch(e)
		{
			startupError = e?.message || String(e);
			startupStatus = 'Linux startup failed';
			printMessage(unexpectedErrorMessage);
			printMessage([String(e?.stack || e)]);
			return;
		}
	}
	function handleActivateConsole(vt)
	{
		if(curVT == vt)
			return;
		curVT = vt;
		if(vt != 7)
			return;
		guiReady = true;
		startupError = '';
		startupStatus = 'Desktop ready';
		if(startupWatchdog) clearTimeout(startupWatchdog);
		// Raise the display to the foreground
		const display = document.getElementById("display");
		display.parentElement.style.zIndex = 5;
		tryPlausible("Display activated");
	}
	function handleProcessCreated()
	{
		processCount++;
		if(processCallback)
			processCallback(processCount);
	}
	async function initCheerpX(cheerpxPromise)
	{
		const CheerpX = await cheerpxPromise;
		startupStatus = 'Connecting to the remote Alpine disk…';
		var blockDevice = null;
		switch(configObj.diskImageType)
		{
			case "cloud":
				try
				{
					blockDevice = await CheerpX.CloudDevice.create(configObj.diskImageUrl);
				}
				catch(e)
				{
					// Report the failure and try again with plain HTTP
					var wssProtocol = "wss:";
					if(configObj.diskImageUrl.startsWith(wssProtocol))
					{
						// WebSocket protocol failed, try agin using plain HTTP
						tryPlausible("WS Disk failure");
						blockDevice = await CheerpX.CloudDevice.create("https:" + configObj.diskImageUrl.substr(wssProtocol.length));
					}
					else
					{
						// No other recovery option
						throw e;
					}
				}
				break;
			case "bytes":
				blockDevice = await CheerpX.HttpBytesDevice.create(configObj.diskImageUrl);
				break;
			case "github":
				blockDevice = await CheerpX.GitHubDevice.create(configObj.diskImageUrl);
				break;
			default:
				throw new Error("Unrecognized device type");
		}
		startupStatus = 'Preparing the local disk cache…';
		blockCache = await CheerpX.IDBDevice.create(cacheId);
		var overlayDevice = await CheerpX.OverlayDevice.create(blockDevice, blockCache);
		var webDevice = await CheerpX.WebDevice.create("");
		var documentsDevice = await CheerpX.WebDevice.create("documents");
		var dataDevice = await CheerpX.DataDevice.create();
		var mountPoints = [
			// The root filesystem, as an Ext2 image
			{type:"ext2", dev:overlayDevice, path:"/"},
			// Access to files on the Web server, relative to the current page
			{type:"dir", dev:webDevice, path:"/web"},
			// Access to read-only data coming from JavaScript
			{type:"dir", dev:dataDevice, path:"/data"},
			// Automatically created device files
			{type:"devs", path:"/dev"},
			// Pseudo-terminals
			{type:"devpts", path:"/dev/pts"},
			// The Linux 'proc' filesystem which provides information about running processes
			{type:"proc", path:"/proc"},
			// The Linux 'sysfs' filesystem which is used to enumerate emulated devices
			{type:"sys", path:"/sys"},
			// Convenient access to sample documents in the user directory
			{type:"dir", dev:documentsDevice, path:"/home/user/documents"}
		];
		startupStatus = 'Creating the Linux virtual machine…';
		try
		{
			cx = await CheerpX.Linux.create({mounts: mountPoints, networkInterface: networkInterface});
		}
		catch(e)
		{
			throw new Error('Could not create the Linux VM: ' + (e?.message || String(e)));
		}
		cx.registerCallback("cpuActivity", cpuCallback);
		cx.registerCallback("diskActivity", hddCallback);
		cx.registerCallback("diskLatency", latencyCallback);
		cx.registerCallback("processCreated", handleProcessCreated);
		term.scrollToBottom();
		cxReadFunc = cx.setCustomConsole(writeData, term.cols, term.rows);
		const display = document.getElementById("display");
		if(display)
		{
			setScreenSize(display);
			cx.setActivateConsole(handleActivateConsole);
		}
		// Run the command in a loop, in case the user exits.
		startupStatus = 'Booting Alpine Linux and starting the graphical desktop…';
		while (true)
		{
			try
			{
				await cx.run(configObj.cmd, configObj.args, configObj.opts);
			}
			catch(e)
			{
				throw new Error('Linux boot stopped: ' + (e?.message || String(e)));
			}
		}
	}
	onMount(() => {
		startupWatchdog = setTimeout(() => { startupSlow = true; }, 45000);
		initTerminal();
		return () => {
			if(startupWatchdog) clearTimeout(startupWatchdog);
			window.removeEventListener('resize', handleResize);
		};
	});
	async function handleConnect()
	{
		const w = window.open("login.html", "_blank");
		cx.networkLogin();
		try
		{
			w.location.href = await startLogin();
		}
		catch(e)
		{
			w.close();
			console.warn(e);
		}
	}
	async function handleReset()
	{
		// Be robust before initialization
		if(blockCache == null)
			return;
		await blockCache.reset();
		location.reload();
	}
	async function handleTool(tool)
	{
		return await handleToolImpl(tool, term);
	}
	async function handleSidebarPinChange(event)
	{
		sideBarPinned = event.detail;
		// Make sure the pinning state of reflected in the layout
		await tick();
		// Adjust the layout based on the new sidebar state
		triggerResize();
	}
</script>

<main class="relative w-full h-full">
	<Nav />
	<div class="absolute top-10 bottom-0 left-0 right-0">
		<SideBar on:connect={handleConnect} on:reset={handleReset} handleTool={!configObj.needsDisplay || curVT == 7 ? handleTool : null} on:sidebarPinChange={handleSidebarPinChange}>
			<slot></slot>
		</SideBar>
		{#if configObj.needsDisplay}
			<div class="absolute top-0 bottom-0 {sideBarPinned ? 'left-[23.5rem]' : 'left-14'} right-0 z-10">
				<canvas class="w-full h-full cursor-none" id="display"></canvas>
			</div>
			{#if !guiReady}
				<div class="absolute top-0 bottom-0 {sideBarPinned ? 'left-[23.5rem]' : 'left-14'} right-0 z-20 flex flex-col items-center justify-center gap-4 bg-neutral-950 text-gray-200">
					<div class="h-10 w-10 rounded-full border-4 border-neutral-700 border-t-sky-400 animate-spin"></div>
					<p class="text-lg font-semibold">{startupError ? 'Linux startup failed' : startupStatus}</p>
					{#if startupError}
						<div class="max-w-2xl rounded bg-red-950 p-3 text-sm text-red-200 break-words">{startupError}</div>
					{:else if startupSlow}
						<p class="max-w-xl text-center text-sm text-amber-300">Startup is taking longer than expected. The VM may be waiting for the graphical display to initialize.</p>
					{:else}
						<p class="text-sm text-gray-400">Please wait while Alpine Linux starts…</p>
					{/if}
				</div>
			{/if}
		{/if}
		<div class="absolute top-0 bottom-0 {sideBarPinned ? 'left-[23.5rem]' : 'left-14'} right-0 p-1 scrollbar" id="console">
		</div>
	</div>
</main>
