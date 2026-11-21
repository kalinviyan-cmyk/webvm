<script>
	import { createEventDispatcher } from 'svelte';
	import Icon from './Icon.svelte';
	import InformationTab from './InformationTab.svelte';
	import NetworkingTab from './NetworkingTab.svelte';
	import CpuTab from './CpuTab.svelte';
	import DiskTab from './DiskTab.svelte';
	import MemoryTab from './MemoryTab.svelte';
	import AnthropicTab from './AnthropicTab.svelte';
	import OtherAITab from './OtherAITab.svelte';
	import PostsTab from './PostsTab.svelte';
	import DiscordTab from './DiscordTab.svelte';
	import GitHubTab from './GitHubTab.svelte';
	import SmallButton from './SmallButton.svelte';
	import { cpuActivity, diskActivity, aiActivity } from './activities.js';
	const icons = [
		{ icon: 'fas fa-info-circle', info: 'Information', activity: null },
		{ icon: 'fas fa-wifi', info: 'Networking', activity: null },
		{ icon: 'fas fa-microchip', info: 'CPU', activity: cpuActivity },
		{ icon: 'fas fa-compact-disc', info: 'Disk', activity: diskActivity },
		{ icon: 'fas fa-memory', info: 'RAM', activity: null },
		{ icon: 'fas fa-robot', info: 'ClaudeAI', activity: aiActivity },
		{ icon: 'fas fa-wand-magic-sparkles', info: 'Gemini/OpenAI', activity: aiActivity },
		null,
		{ icon: 'fas fa-book-open', info: 'Posts', activity: null },
		{ icon: 'fab fa-discord', info: 'Discord', activity: null },
		{ icon: 'fab fa-github', info: 'GitHub', activity: null },
	];
	let dispatch = createEventDispatcher();
	let activeInfo = null; // Tracks currently visible info.
	export let sideBarPinned;

	// Menus open only on an explicit click/tap. Hover-driven opening and
	// mouseleave closing are unreliable on touchscreens.
	function handleClick(icon) {
		activeInfo = activeInfo === icon.info ? null : icon.info;
	}

	function toggleSidebarPin() {
		sideBarPinned = !sideBarPinned;
		dispatch('sidebarPinChange', sideBarPinned);
	}

	export let handleTool;
</script>

<div class="relative z-[100] flex flex-row w-14 h-full bg-neutral-700">
	<div class="flex flex-col shrink-0 w-14 text-gray-300">
		{#each icons as i}
			{#if i}
				<Icon
					icon={i.icon}
					info={i.info}
					activity={i.activity}
					on:click={() => handleClick(i)}
				/>
			{:else}
				<div class="grow" on:mouseenter={handleMouseEnterPanel}></div>
			{/if}
		{/each}
	</div>
	<div
		class="absolute left-14 top-0 bottom-0 flex flex-col gap-5 w-80 max-w-[calc(100vw-3.5rem)] z-[100] p-2 bg-neutral-600 text-gray-100 opacity-95 shadow-xl"
		class:hidden={!activeInfo}
	>
		<div class="absolute right-2 top-2">
			<SmallButton
				buttonIcon="fa-solid fa-thumbtack"
				clickHandler={toggleSidebarPin}
				buttonTooltip={sideBarPinned ? "Unpin Sidebar" : "Pin Sidebar"}
				bgColor={sideBarPinned ? "bg-neutral-500" : "bg-neutral-700"}
			/>
		</div>
		{#if activeInfo === 'Information'}
			<InformationTab>
				<slot></slot>
			</InformationTab>
		{:else if activeInfo === 'Networking'}
			<NetworkingTab on:connect/>
		{:else if activeInfo === 'CPU'}
			<CpuTab/>
		{:else if activeInfo === 'Disk'}
			<DiskTab on:reset/>
		{:else if activeInfo === 'RAM'}
			<MemoryTab/>
		{:else if activeInfo === 'ClaudeAI'}
			<AnthropicTab handleTool={handleTool} />
		{:else if activeInfo === 'Gemini/OpenAI'}
			<OtherAITab handleTool={handleTool} />
		{:else if activeInfo === 'Posts'}
			<PostsTab/>
		{:else if activeInfo === 'Discord'}
			<DiscordTab/>
		{:else if activeInfo === 'GitHub'}
			<GitHubTab/>
		{:else}
			<p>TODO: {activeInfo}</p>
		{/if}

		<div class="mt-auto text-sm text-gray-300">
			<div class="pt-1 pb-1">
				<a href="https://cheerpx.io/" target="_blank">
					<span>Powered by CheerpX</span>
					<img src="assets/cheerpx.svg" alt="CheerpX Logo" class="w-6 h-6 inline-block">
				</a>
			</div>
			<hr class="border-t border-solid border-gray-300">
			<div class="pt-1 pb-1">
				<a href="https://leaningtech.com/" target="”_blank”">© 2022-2025 Leaning Technologies</a>
			</div>
		</div>
	</div>
</div>
