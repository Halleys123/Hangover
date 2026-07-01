<script lang="ts">
	import './layout.css';

	const { children } = $props();
</script>

<div class="min-h-screen bg-gray-50 text-gray-900 font-sans">
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
