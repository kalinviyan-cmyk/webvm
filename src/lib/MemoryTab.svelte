<script>
	import { onMount, onDestroy } from 'svelte';

	let usedMB = null;
	let totalMB = null;
	let heapLimitMB = null;
	let supported = false;
	let deviceRAM = null;
	let timer = null;

	function updateMemory()
	{
		// Chromium exposes the page's JavaScript heap, not the Linux guest's RAM.
		const memory = performance && performance.memory;
		if(memory)
		{
			supported = true;
			usedMB = memory.usedJSHeapSize / 1048576;
			totalMB = memory.totalJSHeapSize / 1048576;
			heapLimitMB = memory.jsHeapSizeLimit / 1048576;
		}
		else
		{
			supported = false;
			usedMB = null;
			totalMB = null;
			heapLimitMB = null;
		}
		deviceRAM = typeof navigator.deviceMemory === 'number' ? navigator.deviceMemory : null;
	}

	onMount(() => {
		updateMemory();
		timer = setInterval(updateMemory, 1000);
	});
	onDestroy(() => clearInterval(timer));

	function formatMB(value)
	{
		return value === null ? '—' : value.toFixed(1) + ' MB';
	}
</script>

<h1 class="text-lg font-bold">RAM usage</h1>
{#if supported}
	<div class="rounded bg-neutral-700 p-3 space-y-2">
		<p><span class="font-bold">Used by this WebVM browser tab:</span> {formatMB(usedMB)}</p>
		<p><span class="font-bold">Allocated JS heap:</span> {formatMB(totalMB)}</p>
		<p><span class="font-bold">Browser heap limit:</span> {formatMB(heapLimitMB)}</p>
		<div class="h-3 w-full overflow-hidden rounded bg-neutral-900">
			<div class="h-full bg-sky-400" style="width: {Math.min(100, totalMB ? usedMB / totalMB * 100 : 0)}%"></div>
		</div>
	</div>
{:else}
	<p>Live browser memory statistics are not exposed by this browser.</p>
{/if}
{#if deviceRAM !== null}
	<p><span class="font-bold">Device RAM estimate:</span> {deviceRAM} GB</p>
{/if}
<p class="text-sm text-gray-300">Updates every second. Browser heap usage is not the same as total Android RAM or the Alpine Linux guest's RAM.</p>
<p class="text-sm text-gray-300">The current CheerpX integration does not expose guest RAM usage directly, so this panel does not pretend these numbers are Linux VM memory.</p>
