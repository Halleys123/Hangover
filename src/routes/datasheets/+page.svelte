<script lang="ts">
	let datasheets = $state([
		{ id: '1', name: 'ESP32-WROOM-32_Datasheet.pdf', size: '1.2 MB', parsed: true, uploadedAt: '1 hour ago' },
		{ id: '2', name: 'DHT22_Specifications.pdf', size: '450 KB', parsed: true, uploadedAt: 'Yesterday' },
		{ id: '3', name: 'TMC2209_Datasheet_Rev1.1.pdf', size: '2.5 MB', parsed: false, uploadedAt: 'Dec 15, 2024' },
		{ id: '4', name: 'ATmega328P_Summary.pdf', size: '150 KB', parsed: true, uploadedAt: 'Dec 10, 2024' }
	]);

	function goHome() {
		window.location.href = '/';
	}
	
	function startUpload() {
		window.location.href = '/datasheets/new';
	}
</script>

<div class="min-h-screen flex flex-col bg-gray-50">
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
				<a href="/workspace" class="text-gray-500 font-medium hover:text-gray-900">Workspace</a>
				<a href="/datasheets" class="text-blue-600 font-medium border-b-2 border-blue-600 pb-5 -mb-5">Datasheets</a>
			</div>
		</div>
	</nav>

	<div class="flex-1 max-w-5xl mx-auto w-full px-6 py-10">
		<div class="flex justify-between items-center mb-8">
			<div>
				<h2 class="text-2xl font-bold text-gray-900">Datasheet Library</h2>
				<p class="text-gray-500 text-sm mt-1">Manage documents that AI relies on for extracting pinouts and schematic rules.</p>
			</div>
			<button
				on:click={startUpload}
				class="px-5 py-2.5 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition"
			>
				Upload PDF Datasheet
			</button>
		</div>

		<div class="bg-white border border-gray-200 rounded-lg overflow-hidden shadow-sm">
			<table class="w-full text-left border-collapse">
				<thead>
					<tr class="bg-gray-50 text-xs font-semibold text-gray-500 uppercase tracking-wider border-b border-gray-200">
						<th class="px-6 py-4">Document Name</th>
						<th class="px-6 py-4">Status</th>
						<th class="px-6 py-4">Size</th>
						<th class="px-6 py-4">Uploaded</th>
						<th class="px-6 py-4 text-right">Actions</th>
					</tr>
				</thead>
				<tbody class="divide-y text-sm">
					{#each datasheets as doc (doc.id)}
					<tr class="hover:bg-gray-50 transition-colors">
						<td class="px-6 py-4 font-medium text-gray-900 flex items-center gap-3">
							<svg class="w-5 h-5 text-red-500" fill="currentColor" viewBox="0 0 24 24">
								<path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 2l5 5h-5V4zM8 14h8v2H8v-2zm0-4h8v2H8v-2z" />
							</svg>
							{doc.name}
						</td>
						<td class="px-6 py-4">
							{#if doc.parsed}
							<span class="inline-flex items-center gap-1.5 px-2.5 py-1 rounded bg-green-50 text-green-700 text-xs font-semibold tracking-wide border border-green-200">
								<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7"/></svg>
								Parsed
							</span>
							{:else}
							<span class="inline-flex items-center gap-1.5 px-2.5 py-1 rounded bg-yellow-50 text-yellow-700 text-xs font-semibold tracking-wide border border-yellow-200">
								Processing...
							</span>
							{/if}
						</td>
						<td class="px-6 py-4 text-gray-500">{doc.size}</td>
						<td class="px-6 py-4 text-gray-500">{doc.uploadedAt}</td>
						<td class="px-6 py-4 text-right">
							<button class="text-gray-400 hover:text-blue-600 transition-colors">
								<svg class="w-5 h-5 inline-block" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 5v.01M12 12v.01M12 19v.01M12 6a1 1 0 110-2 1 1 0 010 2zm0 7a1 1 0 110-2 1 1 0 010 2zm0 7a1 1 0 110-2 1 1 0 010 2z"/></svg>
							</button>
						</td>
					</tr>
					{/each}
					
					{#if datasheets.length === 0}
					<tr>
						<td colspan="5" class="px-6 py-16 text-center text-gray-500">
							<p>No datasheets uploaded yet.</p>
							<a href="/datasheets/new" class="text-blue-600 font-medium hover:underline mt-2 inline-block">Upload one now</a>
						</td>
					</tr>
					{/if}
				</tbody>
			</table>
		</div>
	</div>
</div>