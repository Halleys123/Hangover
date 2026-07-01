<script lang="ts">
	import './layout.css';
	import { onMount } from 'svelte';
	import { navigating } from '$app/stores';
	import { fade } from 'svelte/transition';

	const { children } = $props();

	onMount(() => {
		// Initialize theme from localStorage or system preference
		const saved = localStorage.getItem('theme');
		if (saved === 'dark' || (!saved && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
			document.documentElement.classList.add('dark');
		} else {
			document.documentElement.classList.remove('dark');
		}
	});
</script>

<!-- ROOT LAYOUT WRAPPER -->
<div class="min-h-screen bg-slate-50 dark:bg-zinc-950 text-slate-900 dark:text-zinc-100 font-sans transition-colors duration-200 relative">
	{#if $navigating}
		<!-- Sleek Top Progress Bar -->
		<div transition:fade={{ duration: 150 }} class="fixed top-0 left-0 right-0 h-1 z-50 overflow-hidden bg-blue-100 dark:bg-blue-950/50">
			<div class="h-full bg-gradient-to-r from-blue-600 via-indigo-500 to-cyan-400 animate-pulse w-full transition-all"></div>
		</div>

		<!-- Subtle Ghost Shimmer Overlay -->
		<div transition:fade={{ duration: 150 }} class="fixed inset-0 z-40 pointer-events-none flex items-center justify-center bg-white/30 dark:bg-zinc-950/30 backdrop-blur-[1px]">
			<div class="flex items-center gap-3 px-4 py-2.5 rounded-2xl bg-white dark:bg-zinc-900 border border-slate-200 dark:border-zinc-800 shadow-xl animate-pulse">
				<div class="w-4 h-4 border-2 border-indigo-600 border-t-transparent rounded-full animate-spin"></div>
				<span class="text-xs font-semibold text-slate-700 dark:text-zinc-200 tracking-wide">Loading Screen…</span>
			</div>
		</div>
	{/if}

	{@render children()}
</div>

<style global>
	:global(html, body) {
		margin: 0;
		padding: 0;
		width: 100%;
		height: 100%;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
	}
</style>
