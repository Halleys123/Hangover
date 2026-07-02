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

	async function createProject(e: SubmitEvent) {
		e.preventDefault();
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

	function handleSendMessage(e: SubmitEvent) {
		e.preventDefault();
		if (!inputValue.trim()) return;
		const text = inputValue.trim();
		messages = [...messages, { role: 'user', text }];
		inputValue = '';
		setTimeout(() => {
			messages = [
				...messages,
				{
					role: 'assistant',
					text: `Great idea! You can create a project for "${text}" using the "+ New Project" button above or switch to your workspace to start designing schematics.`
				}
			];
		}, 500);
	}
</script>

<svelte:window on:pointermove={handlePointerMove} on:pointerup={handlePointerUp} />

<!-- MAIN HOMEPAGE WRAPPER -->
<div class="min-h-screen flex flex-col bg-white dark:bg-zinc-950 transition-colors duration-200">
	<NavBar activeLink="workspace" onAction={startNewProject} />

	<div class="flex-1 flex gap-0 h-[calc(100vh-73px)]">
		<!-- Left: AI Chat Interface -->
		{#if leftCollapsed}
			<button
				onclick={() => (leftCollapsed = false)}
				class="w-6 shrink-0 flex items-center justify-center border-r border-gray-200 dark:border-zinc-800 bg-white dark:bg-zinc-900 hover:bg-gray-50 dark:hover:bg-zinc-800 group"
				title="Show AI Copilot panel"
			>
				<svg class="w-4 h-4 text-gray-400 dark:text-zinc-500 group-hover:text-blue-600 dark:group-hover:text-blue-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
				</svg>
			</button>
		{:else}
		<div class="flex flex-col border-r border-gray-200 dark:border-zinc-800 bg-[#FAFAFA] dark:bg-zinc-900 shrink-0" style="width: {leftWidth}px;">
			<!-- Chat Header -->
			<div class="px-4 py-3 border-b border-gray-200 dark:border-zinc-800 flex items-center gap-2 bg-white dark:bg-zinc-900/90">
				<svg class="w-5 h-5 text-blue-600 dark:text-blue-400" fill="currentColor" viewBox="0 0 20 20">
					<path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"/>
				</svg>
				<h2 class="text-gray-900 dark:text-white font-semibold text-sm">AI Copilot</h2>
			</div>

			<!-- Messages Area -->
			<div class="flex-1 overflow-y-auto p-4 space-y-6">
				{#each messages as message (message)}
					<div class="flex gap-3 {message.role === 'user' ? 'justify-end' : 'justify-start'}">
						{#if message.role === 'assistant'}
							<div class="w-6 h-6 rounded bg-blue-100 dark:bg-blue-950/80 text-blue-600 dark:text-blue-400 flex-shrink-0 flex items-center justify-center mt-1 border border-blue-200/50 dark:border-blue-800/50">
								<svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
									<path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"/>
								</svg>
							</div>
						{/if}
						<div
							class="max-w-[85%] px-4 py-3 rounded-2xl {message.role === 'user'
								? 'bg-blue-600 text-white rounded-br-sm shadow-md shadow-blue-600/10'
								: 'bg-white dark:bg-zinc-800/90 border border-gray-200 dark:border-zinc-700/80 text-gray-800 dark:text-zinc-100 rounded-bl-sm shadow-sm'}"
						>
							<p class="text-sm leading-relaxed">{message.text}</p>
						</div>
						{#if message.role === 'user'}
							<div class="w-6 h-6 rounded-full bg-gray-300 dark:bg-zinc-700 flex-shrink-0 mt-1 flex items-center justify-center">
								<svg class="w-4 h-4 text-gray-600 dark:text-zinc-300" fill="currentColor" viewBox="0 0 20 20">
									<path fill-rule="evenodd" d="M10 9a3 3 0 100-6 3 3 0 000 6zm-7 9a7 7 0 1114 0H3z" clip-rule="evenodd"/>
								</svg>
							</div>
						{/if}
					</div>
				{/each}
			</div>

			<!-- Input Area -->
			<div class="p-4 bg-white dark:bg-zinc-900 border-t border-gray-200 dark:border-zinc-800">
				<form onsubmit={handleSendMessage} class="relative">
					<input
						type="text"
						bind:value={inputValue}
						placeholder="Ask about wiring, components..."
						class="w-full pl-4 pr-10 py-2.5 bg-white dark:bg-zinc-800 border border-gray-300 dark:border-zinc-700 rounded-lg text-gray-900 dark:text-white placeholder-gray-400 dark:placeholder-zinc-500 focus:outline-none focus:ring-1 focus:ring-blue-500 focus:border-blue-500 text-sm shadow-sm"
					/>
					<button
						type="submit"
						class="absolute right-2 top-1/2 -translate-y-1/2 p-1.5 text-blue-600 dark:text-blue-400 hover:bg-blue-50 dark:hover:bg-zinc-700 rounded transition-colors"
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
			class="w-1 shrink-0 cursor-col-resize bg-gray-200 dark:bg-zinc-800 hover:bg-blue-400 dark:hover:bg-blue-500 active:bg-blue-500 transition-colors"
			onpointerdown={startDrag}
		></div>
		{/if}

		<!-- Right: Projects List -->
		<div class="flex-1 bg-gray-50 dark:bg-zinc-950 p-8 overflow-y-auto w-full">
			<div class="max-w-4xl mx-auto">
				<div class="flex justify-between items-center mb-8">
					<div>
						<h2 class="text-2xl font-bold tracking-tight text-slate-900 dark:text-white">Recent Projects</h2>
						<p class="text-slate-500 dark:text-zinc-400 text-sm mt-1">Pick up where you left off or start a new hardware workspace.</p>
					</div>
					<button
						onclick={() => (window.location.href = '/workspace')}
						class="px-4 py-2.5 bg-gradient-to-r from-blue-600 to-indigo-600 text-white text-sm font-medium rounded-xl shadow-sm shadow-indigo-500/20 hover:from-blue-700 hover:to-indigo-700 active:scale-[0.98] transition-all flex items-center gap-2"
					>
						<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M12 4v16m8-8H4" />
						</svg>
						New Project
					</button>
				</div>
				
				<div class="mb-6 relative">
					<svg class="w-5 h-5 absolute left-3 top-1/2 -translate-y-1/2 text-gray-400 dark:text-zinc-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
					</svg>
					<input
						type="text"
						bind:value={searchQuery}
						placeholder="Search projects, descriptions, or components..."
						class="w-full pl-10 pr-4 py-3 bg-white dark:bg-zinc-900 border border-gray-200 dark:border-zinc-800 rounded-lg text-sm text-slate-900 dark:text-white placeholder:text-slate-400 dark:placeholder:text-zinc-500 focus:outline-none focus:border-blue-500 dark:focus:border-blue-400 focus:ring-1 focus:ring-blue-500 shadow-sm"
					/>
				</div>
				
				{#if projectsLoading}
				<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
					{#each [1,2,3] as i (i)}<div class="border border-gray-200 dark:border-zinc-800 bg-white dark:bg-zinc-900 rounded-lg p-5 animate-pulse"><div class="h-5 bg-gray-200 dark:bg-zinc-800 rounded w-3/4 mb-3"></div><div class="h-3 bg-gray-100 dark:bg-zinc-800 rounded w-full mb-2"></div><div class="h-3 bg-gray-100 dark:bg-zinc-800 rounded w-2/3"></div></div>{/each}
				</div>
				{:else}
				<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
					{#each filteredProjects as project (project._id)}
						<button
							onclick={() => openProject(project._id)}
							class="bg-white dark:bg-zinc-900 border border-gray-200 dark:border-zinc-800 rounded-lg p-5 hover:border-blue-400 dark:hover:border-blue-500 hover:shadow-md transition-all text-left group flex flex-col h-full"
						>
							<div class="flex items-start justify-between mb-2">
								<div>
									<h3 class="text-lg font-semibold text-gray-900 dark:text-white group-hover:text-blue-600 dark:group-hover:text-blue-400 transition-colors">
										{project.name}
									</h3>
								</div>
								<span
									class="px-2.5 py-1 rounded text-[10px] font-bold uppercase tracking-wide {project.status === 'completed'
										? 'bg-green-100 dark:bg-green-950/70 text-green-700 dark:text-green-400 border border-green-200/50 dark:border-green-800/50'
										: 'bg-yellow-100 dark:bg-yellow-950/70 text-yellow-700 dark:text-yellow-400 border border-yellow-200/50 dark:border-yellow-800/50'}"
								>
									{project.status === 'completed' ? 'Completed' : 'In Progress'}
								</span>
							</div>
							
							<p class="text-gray-500 dark:text-zinc-400 text-sm mt-1 mb-4 flex-grow leading-relaxed">{project.description}</p>

							<div class="flex flex-wrap gap-2 mt-auto">
								{#each project.components as component (component)}
									<span class="px-2.5 py-1 bg-gray-50 dark:bg-zinc-800 text-gray-600 dark:text-zinc-300 text-xs rounded-md border border-gray-200 dark:border-zinc-700">
										{component}
									</span>
								{/each}
							</div>

							<div class="mt-4 pt-3 border-t border-gray-100 dark:border-zinc-800/80 text-xs text-gray-400 dark:text-zinc-500 flex items-center gap-1">
								<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" /></svg>
								Last modified: {new Date(project.date).toLocaleDateString()}
							</div>
						</button>
					{/each}
				</div>

				{#if filteredProjects.length === 0 && !projectsLoading}
					<div class="text-center py-16 bg-white dark:bg-zinc-900 rounded-lg border border-gray-200 dark:border-zinc-800 mt-4 shadow-sm">
						{#if projects.length === 0}
							<p class="text-gray-500 dark:text-zinc-400 mb-2 font-medium">No projects yet</p>
							<button onclick={startNewProject} class="text-blue-600 dark:text-blue-400 text-sm font-medium hover:underline">Create your first project</button>
						{:else}
							<p class="text-gray-500 dark:text-zinc-400 font-medium">No projects match '{searchQuery}'</p>
							<button onclick={() => searchQuery = ''} class="mt-2 text-blue-600 dark:text-blue-400 text-sm font-medium hover:underline">Clear search</button>
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
