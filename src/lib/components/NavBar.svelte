<script lang="ts">
	import { authUser } from '$lib/stores/auth';

	interface Props {
		activeLink?: 'workspace' | 'datasheets';
		onLogoClick?: () => void;
		actionLabel?: string;
		onAction?: () => void;
		wide?: boolean;
		compact?: boolean;
	}

	let {
		activeLink = 'workspace',
		onLogoClick,
		actionLabel,
		onAction,
		wide = false,
		compact = false
	}: Props = $props();

	function logout() {
		authUser.logout();
		window.location.href = '/login';
	}
</script>

<nav class="bg-white border-b border-gray-200">
	<div class="{wide ? 'w-full' : 'max-w-7xl mx-auto'} px-6 {compact ? 'py-3' : 'py-4'} flex items-center justify-between">
		<div
			class="flex items-center gap-3 {onLogoClick ? 'cursor-pointer' : ''}"
			onclick={onLogoClick}
		>
			<div class="w-8 h-8 flex items-center justify-center text-blue-600">
				<svg class="w-6 h-6" fill="currentColor" viewBox="0 0 24 24">
					<path d="M12 2L2 7l10 5 10-5-10-5zm0 10L2 17l10 5 10-5-10-5z" fill="currentColor"/>
				</svg>
			</div>
			<h1 class="text-xl font-bold text-gray-900">Hardware Prototyping Copilot</h1>
		</div>
		<div class="flex items-center gap-6">
			<a
				href="/workspace"
				class={activeLink === 'workspace'
					? `text-blue-600 font-medium border-b-2 border-blue-600 ${compact ? 'pb-4 -mb-4' : 'pb-5 -mb-5'}`
					: 'text-gray-500 font-medium hover:text-gray-900'}
			>Workspace</a>
			<a
				href="/datasheets"
				class={activeLink === 'datasheets'
					? `text-blue-600 font-medium border-b-2 border-blue-600 ${compact ? 'pb-4 -mb-4' : 'pb-5 -mb-5'}`
					: 'text-gray-500 font-medium hover:text-gray-900'}
			>Datasheets</a>
			{#if actionLabel}
				<button
					onclick={onAction}
					class="px-4 py-2 bg-blue-600 text-white font-medium rounded hover:bg-blue-700 transition-colors"
				>
					{actionLabel}
				</button>
			{/if}
			{#if $authUser}
				<div class="flex items-center gap-3 border-l border-gray-200 pl-6">
					<span class="text-sm text-gray-700 font-medium">{$authUser.name}</span>
					<button
						onclick={logout}
						class="text-sm text-gray-500 hover:text-gray-900 font-medium"
					>Logout</button>
				</div>
			{:else}
				<div class="flex items-center gap-3 border-l border-gray-200 pl-6">
					<a href="/login" class="text-sm text-gray-600 font-medium hover:text-gray-900">Log in</a>
					<a
						href="/signup"
						class="px-3.5 py-1.5 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition-colors"
					>Sign up</a>
				</div>
			{/if}
		</div>
	</div>
</nav>
