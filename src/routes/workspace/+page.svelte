<script lang="ts">
	import { onMount } from 'svelte';
	import NavBar from '$lib/components/NavBar.svelte';
	import { authUser } from '$lib/stores/auth';
	import { api } from '$lib/api';

	interface Project {
		_id: string;
		name: string;
		description: string;
		components: string[];
		date: string;
		status: 'in-progress' | 'completed';
	}

	let projects = $state<Project[]>([]);
	let loading = $state(true);
	let creating = $state(false);
	let newName = $state('');
	let showForm = $state(false);
	let apiError = $state('');

	onMount(async () => {
		if (!$authUser) { window.location.href = '/login'; return; }
		try {
			projects = await api.get<Project[]>('/projects');
		} catch (err: any) {
			if (err.message?.includes('Unauthorized')) { window.location.href = '/login'; return; }
			apiError = err.message;
		} finally {
			loading = false;
		}
	});

	async function createProject(e: SubmitEvent) {
		e.preventDefault();
		if (!newName.trim()) return;
		creating = true;
		try {
			const project = await api.post<{ _id: string }>('/projects', { name: newName.trim() });
			window.location.href = `/workspace/${project._id}`;
		} catch (err: any) {
			apiError = err.message;
			creating = false;
		}
	}
</script>

<div class="min-h-screen flex flex-col bg-white">
	<NavBar activeLink="workspace" onLogoClick={() => (window.location.href = '/')} />

	<div class="flex-1 max-w-5xl mx-auto w-full px-6 py-10">
		<div class="flex justify-between items-center mb-8">
			<div>
				<h2 class="text-2xl font-bold text-gray-900">
					{#if $authUser}Welcome back, {$authUser.name.split(' ')[0]}{:else}Projects{/if}
				</h2>
				<p class="text-gray-500 text-sm mt-1">Your hardware design workspaces.</p>
			</div>
			<button onclick={() => (showForm = !showForm)} class="px-5 py-2.5 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition">
				+ New Project
			</button>
		</div>

		{#if showForm}
		<form onsubmit={createProject} class="mb-6 flex gap-3 p-4 bg-blue-50 border border-blue-200 rounded-lg">
			<input
				type="text"
				bind:value={newName}
				placeholder="Project name, e.g. Smart Plant Monitor"
				autofocus
				class="flex-1 px-3 py-2 border border-gray-300 rounded text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
			/>
			<button type="submit" disabled={creating || !newName.trim()} class="px-4 py-2 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition disabled:opacity-50">
				{creating ? 'Creating…' : 'Create'}
			</button>
			<button type="button" onclick={() => { showForm = false; newName = ''; }} class="px-4 py-2 text-gray-600 text-sm font-medium rounded hover:bg-gray-100 transition">
				Cancel
			</button>
		</form>
		{/if}

		{#if apiError}
		<div class="mb-4 p-3 bg-red-50 border border-red-200 rounded text-sm text-red-700">{apiError}</div>
		{/if}

		{#if loading}
		<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
			{#each [1, 2, 3] as i (i)}
			<div class="border border-gray-200 rounded-lg p-5 animate-pulse">
				<div class="h-5 bg-gray-200 rounded w-3/4 mb-3"></div>
				<div class="h-3 bg-gray-100 rounded w-full mb-2"></div>
				<div class="h-3 bg-gray-100 rounded w-2/3"></div>
			</div>
			{/each}
		</div>

		{:else if projects.length === 0}
		<div class="text-center py-20 bg-gray-50 rounded-xl border border-dashed border-gray-300">
			<svg class="w-12 h-12 text-gray-300 mx-auto mb-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
				<path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M12 9v3m0 0v3m0-3h3m-3 0H9m12 0a9 9 0 11-18 0 9 9 0 0118 0z"/>
			</svg>
			<p class="text-gray-500 font-medium mb-2">No projects yet</p>
			<p class="text-gray-400 text-sm mb-4">Create your first hardware design workspace.</p>
			<button onclick={() => (showForm = true)} class="px-5 py-2.5 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition">
				Create First Project
			</button>
		</div>

		{:else}
		<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
			{#each projects as project (project._id)}
				<button onclick={() => (window.location.href = `/workspace/${project._id}`)} class="bg-white border border-gray-200 rounded-lg p-5 hover:border-blue-400 hover:shadow-md transition-all text-left group flex flex-col h-full">
					<div class="flex items-start justify-between mb-2">
						<h3 class="text-lg font-semibold text-gray-900 group-hover:text-blue-600 transition-colors">{project.name}</h3>
						<span class="px-2.5 py-1 rounded text-[10px] font-bold uppercase tracking-wide {project.status === 'completed' ? 'bg-green-100 text-green-700' : 'bg-yellow-100 text-yellow-700'}">
							{project.status === 'completed' ? 'Completed' : 'In Progress'}
						</span>
					</div>
					<p class="text-gray-500 text-sm mt-1 mb-4 flex-grow">{project.description || 'No description.'}</p>
					{#if project.components.length > 0}
					<div class="flex flex-wrap gap-2 mt-auto">
						{#each project.components as component (component)}
							<span class="px-2.5 py-1 bg-gray-50 text-gray-600 text-xs rounded-md border border-gray-200">{component}</span>
						{/each}
					</div>
					{/if}
				</button>
			{/each}
		</div>
		{/if}
	</div>
</div>


<div class="min-h-screen flex flex-col bg-white">
	<NavBar activeLink="workspace" onLogoClick={goHome} />

	<div class="flex-1 max-w-5xl mx-auto w-full px-6 py-10">
		<div class="flex justify-between items-center mb-8">
			<div>
				<h2 class="text-2xl font-bold text-gray-900">Select a Project</h2>
				<p class="text-gray-500 text-sm mt-1">Choose an existing workspace to continue, or start a new one.</p>
			</div>
			<button
				on:click={startNewProject}
				class="px-5 py-2.5 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition"
			>
				+ New Project
			</button>
		</div>

		<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
			{#each projects as project (project.id)}
				<button
					on:click={() => openProject(project.id)}
					class="bg-white border border-gray-200 rounded-lg p-5 hover:border-blue-400 hover:shadow-md transition-all text-left group flex flex-col h-full"
				>
					<div class="flex items-start justify-between mb-2">
						<h3 class="text-lg font-semibold text-gray-900 group-hover:text-blue-600 transition-colors">
							{project.name}
						</h3>
						<span
							class="px-2.5 py-1 rounded text-[10px] font-bold uppercase tracking-wide {project.status === 'completed'
								? 'bg-green-100 text-green-700'
								: 'bg-yellow-100 text-yellow-700'}"
						>
							{project.status === 'completed' ? 'Completed' : 'In Progress'}
						</span>
					</div>

					<p class="text-gray-500 text-sm mt-1 mb-4 flex-grow">{project.description}</p>

					<div class="flex flex-wrap gap-2 mt-auto">
						{#each project.components as component (component)}
							<span class="px-2.5 py-1 bg-gray-50 text-gray-600 text-xs rounded-md border border-gray-200">
								{component}
							</span>
						{/each}
					</div>
				</button>
			{/each}
		</div>

		<div class="mt-8 text-center">
			<button on:click={goHome} class="text-sm text-gray-500 hover:text-gray-800 font-medium">
				&larr; Back to Home
			</button>
		</div>
	</div>
</div>
