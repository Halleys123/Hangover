<script lang="ts">
	import { onMount } from 'svelte';

	let projects = $state([
		{
			id: 1,
			name: '3D Printer Build',
			description: 'Custom FDM printer with RAMPS 1.4 and TMC2209 drivers',
			components: ['RAMPS 1.4', 'TMC2209', 'NEMA 17'],
			date: '2024-12-15',
			status: 'in-progress'
		},
		{
			id: 2,
			name: 'Smart Plant Monitor',
			description: 'IoT moisture sensor with ESP32 and capacitive soil sensor',
			components: ['ESP32', 'Capacitive Soil Sensor', 'MCP3008'],
			date: '2024-12-10',
			status: 'completed'
		},
		{
			id: 3,
			name: 'Weather Station',
			description: 'Arduino-based weather monitoring system',
			components: ['Arduino Uno', 'DHT22', 'BMP280'],
			date: '2024-12-05',
			status: 'in-progress'
		}
	]);

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

	function handleSendMessage() {
		if (inputValue.trim()) {
			messages.push({
				role: 'user',
				text: inputValue
			});
			inputValue = '';

			// Simulate copilot response
			setTimeout(() => {
				messages.push({
					role: 'assistant',
					text: 'That sounds interesting! To help you, I\'ll need the datasheets for your components. You can upload them in the workspace view where we can validate connections in real-time.'
				});
			}, 600);
		}
	}

	function startNewProject() {
		// New path logic: when creating a project, jump directly to standard workspace ID to avoid /new matching conflicts
		window.location.href = '/workspace/demo-project';
	}

	function openProject(id: number) {
		window.location.href = `/workspace/${id}`;
	}
</script>

<svelte:window on:pointermove={handlePointerMove} on:pointerup={handlePointerUp} />

<div class="min-h-screen flex flex-col bg-white">
	<!-- Top Navigation -->
	<nav class="bg-white border-b border-gray-200">
		<div class="max-w-7xl mx-auto px-6 py-4 flex items-center justify-between">
			<div class="flex items-center gap-3">
				<div class="w-8 h-8 flex items-center justify-center text-blue-600">
					<svg class="w-6 h-6" fill="currentColor" viewBox="0 0 24 24">
						<path d="M12 2L2 7l10 5 10-5-10-5zm0 10L2 17l10 5 10-5-10-5z" fill="currentColor"/>
					</svg>
				</div>
				<h1 class="text-xl font-bold text-gray-900">Hardware Prototyping Copilot</h1>
			</div>
			<div class="flex items-center gap-6">
				<a href="/workspace" class="text-blue-600 font-medium border-b-2 border-blue-600 pb-5 -mb-5">Workspace</a>
				<a href="/datasheets" class="text-gray-500 font-medium hover:text-gray-900">Datasheets</a>
				<button
					on:click={startNewProject}
					class="px-4 py-2 bg-blue-600 text-white font-medium rounded hover:bg-blue-700 transition-colors"
				>
					Export
				</button>
			</div>
		</div>
	</nav>

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
				
				<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
					{#each filteredProjects as project (project.id)}
						<button
							on:click={() => openProject(project.id)}
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

				{#if filteredProjects.length === 0}
					<div class="text-center py-16 bg-white rounded-lg border border-gray-200 mt-4">
						<svg class="w-12 h-12 text-gray-300 mx-auto mb-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.172 16.172a4 4 0 015.656 0M9 10h.01M15 10h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
						</svg>
						<p class="text-gray-500 mb-4 font-medium">No projects found matching '{searchQuery}'</p>
						{#if searchQuery}
							<button
								on:click={() => searchQuery = ''}
								class="px-4 py-2 bg-gray-100 text-gray-600 font-medium rounded hover:bg-gray-200 transition"
							>
								Clear Search
							</button>
						{/if}
					</div>
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
