<script lang="ts">
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

	function openProject(id: number) {
		window.location.href = `/workspace/${id}`;
	}

	function startNewProject() {
		window.location.href = '/workspace/demo-project';
	}

	function goHome() {
		window.location.href = '/';
	}
</script>

<div class="min-h-screen flex flex-col bg-white">
	<!-- Top Navigation -->
	<nav class="bg-white border-b border-gray-200">
		<div class="max-w-7xl mx-auto px-6 py-4 flex items-center justify-between">
			<div class="flex items-center gap-3 cursor-pointer" on:click={goHome}>
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
			</div>
		</div>
	</nav>

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
