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
	let projectsLoading = $state(true);
	let creating = $state(false);
	let showNewForm = $state(false);
	let newProjectName = $state('');

	let messages = $state<Array<{ role: 'user' | 'assistant'; text: string }>>([
		{
			role: 'assistant',
			text: 'Hello! I\'m the Hardware Prototyping Copilot. I can help you design and validate your electrical circuits. What are you prototyping today?'
		}
	]);

	let inputValue = $state('');
	let searchQuery = $state('');

	// Resizable / collapsible left panel
	let leftWidth = $state(400);
	let leftCollapsed = $state(false);
	let isDragging = $state(false);

	const MIN_PANEL_WIDTH = 260;
	const MAX_PANEL_WIDTH = 600;
	const COLLAPSE_THRESHOLD = 150;

	function startDrag() {
		isDragging = true;
	}

	function handlePointerMove(e: PointerEvent) {
		if (!isDragging) return;
		const w = e.clientX;
		if (w < COLLAPSE_THRESHOLD) {
			leftCollapsed = true;
		} else {
			leftCollapsed = false;
			leftWidth = Math.min(Math.max(w, MIN_PANEL_WIDTH), MAX_PANEL_WIDTH);
		}
	}

	function handlePointerUp() {
		isDragging = false;
	}

	let filteredProjects = $derived(
		projects.filter(
			(p) =>
				p.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
				p.description.toLowerCase().includes(searchQuery.toLowerCase()) ||
				p.components.some((c) => c.toLowerCase().includes(searchQuery.toLowerCase()))
		)
	);

	onMount(async () => {
		if (!$authUser) { window.location.href = '/login'; return; }
		try {
			projects = await api.get<Project[]>('/projects');
		} catch (err: any) {
			if (err.message?.includes('Unauthorized')) { window.location.href = '/login'; return; }
		} finally {
			projectsLoading = false;
		}
	});

	async function startNewProject() {
		showNewForm = true;
	}

	async function createProject() {
		if (!newProjectName.trim()) return;
		creating = true;
		try {
			const project = await api.post<{ _id: string }>('/projects', { name: newProjectName.trim() });
			window.location.href = `/workspace/${project._id}`;
		} catch { creating = false; }
	}

	function openProject(id: string) {
		window.location.href = `/workspace/${id}`;
	}
</script>

<svelte:window on:pointermove={handlePointerMove} on:pointerup={handlePointerUp} />

<div class="min-h-screen flex flex-col bg-white">
	<NavBar activeLink="workspace" onAction={startNewProject} />

	<div class="flex-1 flex gap-0 h-[calc(100vh-73px)]">
		<!-- Left: AI Chat Interface -->
		{#if leftCollapsed}
			<button
				on:click={() => (leftCollapsed = false)}
				class="w-6 shrink-0 flex items-center justify-center border-r border-gray-200 bg-white hover:bg-gray-50 group"
				title="Show AI Copilot panel"
			>
				<svg class="w-4 h-4 text-gray-400 group-hover:text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
				</svg>
			</button>
		{:else}
		<div class="flex flex-col border-r border-gray-200 bg-[#FAFAFA] shrink-0" style="width: {leftWidth}px;">
			<!-- Chat Header -->
			<div class="px-4 py-3 border-b border-gray-200 flex items-center gap-2 bg-white">
				<svg class="w-5 h-5 text-blue-600" fill="currentColor" viewBox="0 0 20 20">
					<path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"/>
				</svg>
				<h2 class="text-gray-900 font-semibold text-sm">AI Copilot</h2>
			</div>

			<!-- Messages Area -->
			<div class="flex-1 overflow-y-auto p-4 space-y-6">
				{#each messages as message (message)}
					<div class="flex gap-3 {message.role === 'user' ? 'justify-end' : 'justify-start'}">
						{#if message.role === 'assistant'}
							<div class="w-6 h-6 rounded bg-blue-100 text-blue-600 flex-shrink-0 flex items-center justify-center mt-1">
								<svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
									<path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"/>
								</svg>
							</div>
						{/if}
						<div
							class="max-w-[85%] px-4 py-3 rounded-2xl {message.role === 'user'
								? 'bg-blue-600 text-white rounded-br-sm'
								: 'bg-white border border-gray-200 text-gray-800 rounded-bl-sm shadow-sm'}"
						>
							<p class="text-sm leading-relaxed">{message.text}</p>
						</div>
						{#if message.role === 'user'}
							<div class="w-6 h-6 rounded-full bg-gray-300 flex-shrink-0 mt-1 flex items-center justify-center">
								<svg class="w-4 h-4 text-gray-600" fill="currentColor" viewBox="0 0 20 20">
									<path fill-rule="evenodd" d="M10 9a3 3 0 100-6 3 3 0 000 6zm-7 9a7 7 0 1114 0H3z" clip-rule="evenodd"/>
								</svg>
							</div>
						{/if}
					</div>
				{/each}
			</div>

			<!-- Input Area -->
			<div class="p-4 bg-white border-t border-gray-200">
				<form on:submit|preventDefault={handleSendMessage} class="relative">
					<input
						type="text"
						bind:value={inputValue}
						placeholder="Ask about wiring, components..."
						class="w-full pl-4 pr-10 py-2.5 bg-white border border-gray-300 rounded-lg text-gray-900 placeholder-gray-400 focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 text-sm"
					/>
					<button
						type="submit"
						class="absolute right-2 top-1/2 -translate-y-1/2 p-1.5 text-blue-600 hover:bg-blue-50 rounded"
					>
						<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8"/>
						</svg>
					</button>
				</form>
			</div>
		</div>
		<!-- Left Divider -->
		<div
			class="w-1 shrink-0 cursor-col-resize bg-gray-200 hover:bg-blue-400 active:bg-blue-500 transition-colors"
			on:pointerdown={startDrag}
		></div>
		{/if}

		<!-- Right: Projects List -->
		<div class="flex-1 bg-gray-50 p-8 overflow-y-auto w-full">
			<div class="max-w-4xl mx-auto">
				<div class="flex justify-between items-center mb-8">
					<h2 class="text-2xl font-bold text-gray-900">Recent Projects</h2>
					<button
						on:click={startNewProject}
						class="px-5 py-2.5 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition"
					>
						+ New Project
					</button>
				</div>

				{#if showNewForm}
				<form onsubmit|preventDefault={createProject} class="mb-4 flex gap-2 p-3 bg-blue-50 border border-blue-200 rounded-lg">
					<input type="text" bind:value={newProjectName} placeholder="Project name" autofocus
						class="flex-1 px-3 py-1.5 border border-gray-300 rounded text-sm focus:outline-none focus:ring-1 focus:ring-blue-500" />
					<button type="submit" disabled={creating || !newProjectName.trim()} class="px-3 py-1.5 bg-blue-600 text-white text-sm rounded hover:bg-blue-700 disabled:opacity-50">{creating ? '…' : 'Create'}</button>
					<button type="button" on:click={() => { showNewForm = false; newProjectName = ''; }} class="px-3 py-1.5 text-gray-500 text-sm rounded hover:bg-gray-100">✕</button>
				</form>
				{/if}
				
				<div class="mb-6 relative">
					<svg class="w-5 h-5 absolute left-3 top-1/2 -translate-y-1/2 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
					</svg>
					<input
						type="text"
						bind:value={searchQuery}
						placeholder="Search projects, descriptions, or components..."
						class="w-full pl-10 pr-4 py-3 bg-white border border-gray-200 rounded-lg text-sm focus:outline-none focus:border-blue-500 focus:ring-1 focus:ring-blue-500 shadow-sm"
					/>
				</div>
				
				{#if projectsLoading}
				<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
					{#each [1,2,3] as i (i)}<div class="border border-gray-200 rounded-lg p-5 animate-pulse"><div class="h-5 bg-gray-200 rounded w-3/4 mb-3"></div><div class="h-3 bg-gray-100 rounded w-full mb-2"></div><div class="h-3 bg-gray-100 rounded w-2/3"></div></div>{/each}
				</div>
				{:else}
				<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
					{#each filteredProjects as project (project._id)}
						<button
							on:click={() => openProject(project._id)}
							class="bg-white border border-gray-200 rounded-lg p-5 hover:border-blue-400 hover:shadow-md transition-all text-left group flex flex-col h-full"
						>
							<div class="flex items-start justify-between mb-2">
								<div>
									<h3 class="text-lg font-semibold text-gray-900 group-hover:text-blue-600 transition-colors">
										{project.name}
									</h3>
								</div>
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

							<div class="mt-4 pt-3 border-t border-gray-100 text-xs text-gray-400 flex items-center gap-1">
								<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" /></svg>
								Last modified: {new Date(project.date).toLocaleDateString()}
							</div>
						</button>
					{/each}
				</div>

				{#if filteredProjects.length === 0 && !projectsLoading}
					<div class="text-center py-16 bg-white rounded-lg border border-gray-200 mt-4">
						{#if projects.length === 0}
							<p class="text-gray-500 mb-2 font-medium">No projects yet</p>
							<button on:click={startNewProject} class="text-blue-600 text-sm font-medium hover:underline">Create your first project</button>
						{:else}
							<p class="text-gray-500 font-medium">No projects match '{searchQuery}'</p>
							<button on:click={() => searchQuery = ''} class="mt-2 text-blue-600 text-sm font-medium hover:underline">Clear search</button>
						{/if}
					</div>
				{/if}
				{/if}
			</div>
		</div>
	</div>
</div>

<style>
	:global(body) {
		overflow: hidden;
	}
</style>
