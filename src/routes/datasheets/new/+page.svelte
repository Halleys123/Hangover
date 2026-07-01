<script lang="ts">
	let isScanning = $state(false);
	let isUploading = $state(false);
	let uploadProgress = $state(0);
	let isParsed = $state(false);
	let hasFile = $state(false);
	
	function goBack() {
		window.history.back();
	}
	
	function handleRescan() {
		isScanning = true;
		setTimeout(() => {
			isScanning = false;
		}, 2000);
	}

	function simulateUpload() {
		isUploading = true;
		uploadProgress = 0;
		const interval = setInterval(() => {
			uploadProgress += 10;
			if (uploadProgress >= 100) {
				clearInterval(interval);
				isUploading = false;
				hasFile = true;
			}
		}, 200);
	}

	function handleParse() {
		isScanning = true;
		setTimeout(() => {
			isScanning = false;
			isParsed = true;
		}, 1500);
	}
</script>

<div class="min-h-screen bg-[#F8FAFC] flex flex-col font-sans">
	<!-- Header -->
	<header class="bg-white border-b border-gray-200 px-6 py-4 flex items-center justify-between shadow-sm">
		<div class="flex items-center gap-4">
			<button on:click={goBack} class="p-2 -ml-2 text-gray-400 hover:text-gray-900 transition-colors rounded-full hover:bg-gray-50">
				<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 19l-7-7m0 0l7-7m-7 7h18" /></svg>
			</button>
			<div>
				<h1 class="text-xl font-bold text-gray-900 leading-tight">Datasheet Ingestion</h1>
				<p class="text-sm text-gray-500 font-medium">{hasFile ? 'ESP32-WROOM-32' : 'Upload Document'}</p>
			</div>
		</div>
		{#if hasFile}
			<button 
				on:click={handleRescan}
				disabled={isScanning}
				class="flex items-center gap-2 px-4 py-2 bg-white border border-gray-200 shadow-sm rounded-lg text-sm font-medium text-gray-700 hover:bg-gray-50 hover:border-gray-300 transition-all disabled:opacity-50"
			>
				<svg class="w-4 h-4 {isScanning ? 'animate-spin' : ''}" fill="none" stroke="currentColor" viewBox="0 0 24 24">
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
				</svg>
				Re-scan Document
			</button>
		{/if}
	</header>

	<!-- Main Content Area -->
	<div class="flex-1 flex gap-6 p-6 max-w-[1600px] mx-auto w-full h-[calc(100vh-81px)] overflow-hidden">
		<!-- Left: PDF Viewer or Upload Area -->
		<div class="flex-1 bg-white border border-gray-200 rounded-xl shadow-sm flex flex-col overflow-hidden">
			{#if !hasFile}
				<!-- Upload Area -->
				<div class="flex-1 flex flex-col items-center justify-center p-12 bg-slate-50/50">
					<div class="w-full max-w-md bg-white border-2 border-dashed border-blue-200 rounded-xl p-12 text-center transition-all hover:border-blue-400 hover:bg-blue-50/30">
						<div class="w-16 h-16 bg-blue-50 text-blue-500 rounded-full flex items-center justify-center mx-auto mb-6">
							<svg class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" /></svg>
						</div>
						<h3 class="text-lg font-bold text-gray-900 mb-2">Upload Datasheet PDF</h3>
						<p class="text-sm text-gray-500 mb-8">Drag and drop your file here, or click to browse</p>
						
						{#if isUploading}
							<div class="w-full max-w-xs mx-auto">
								<div class="flex justify-between text-xs text-gray-500 mb-2">
									<span>Uploading...</span>
									<span>{uploadProgress}%</span>
								</div>
								<div class="w-full bg-gray-200 rounded-full h-2">
									<div class="bg-blue-600 h-2 rounded-full transition-all duration-200" style="width: {uploadProgress}%"></div>
								</div>
							</div>
						{:else}
							<button 
								on:click={simulateUpload}
								class="px-6 py-2.5 bg-blue-600 text-white font-medium rounded-lg hover:bg-blue-700 transition shadow-sm"
							>
								Select File
							</button>
						{/if}
					</div>
				</div>
			{:else}
				<!-- PDF Toolbar -->
				<div class="px-4 py-3 border-b border-gray-100 flex items-center justify-between text-sm text-gray-600 bg-gray-50/50">
					<div class="flex items-center gap-2">
						<svg class="w-4 h-4 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 21h10a2 2 0 002-2V9.414a1 1 0 00-.293-.707l-5.414-5.414A1 1 0 0012.586 3H7a2 2 0 00-2 2v14a2 2 0 002 2z" /></svg>
						<span class="font-medium">ESP32-WROOM-32_Datasheet.pdf</span>
					</div>
					<div class="flex items-center gap-3">
						<button class="p-1 hover:text-gray-900 rounded hover:bg-gray-100">-</button>
						<span class="font-medium text-xs text-gray-500">100%</span>
						<button class="p-1 hover:text-gray-900 rounded hover:bg-gray-100">+</button>
					</div>
				</div>

				<!-- PDF Skeleton Content -->
				<div class="flex-1 bg-[#F8FAFC] overflow-y-auto p-8 flex justify-center">
					<div class="w-full max-w-[600px] bg-white shadow-sm border border-gray-200 rounded p-10 space-y-8 min-h-[800px]">
						<div class="h-8 bg-slate-200/60 rounded-md w-3/4"></div>
						<div class="space-y-4">
							<div class="h-4 bg-slate-100 rounded w-full"></div>
							<div class="h-4 bg-slate-100 rounded w-full"></div>
							<div class="h-4 bg-slate-100 rounded w-5/6"></div>
						</div>
						
						<!-- Table Skeleton -->
						<div class="border border-slate-200 rounded-lg overflow-hidden mt-12">
							<div class="h-10 bg-slate-50 border-b border-slate-200"></div>
							<div class="h-10 border-b border-slate-100"></div>
							<div class="h-10 border-b border-slate-100 bg-slate-50/50"></div>
							<div class="h-10 border-b border-slate-100"></div>
							<div class="h-10 border-b border-slate-100 bg-slate-50/50"></div>
							<div class="h-10"></div>
						</div>

						<div class="space-y-4 mt-12">
							<div class="h-4 bg-slate-100 rounded w-full"></div>
							<div class="h-4 bg-slate-100 rounded w-5/6"></div>
						</div>
					</div>
				</div>
			{/if}
		</div>

		<!-- Right: Extracted Data -->
		<div class="w-[500px] bg-white border border-gray-200 rounded-xl shadow-sm flex flex-col">
			<div class="p-5 border-b border-gray-100 flex items-center justify-between">
				<h2 class="text-lg font-bold text-gray-900">Extracted Pinout Specifications</h2>
				{#if isParsed}
					<span class="flex items-center gap-1.5 px-2.5 py-1 bg-green-50 text-green-700 text-xs font-semibold rounded-full border border-green-100">
						<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" /></svg>
						Parsed Successfully
					</span>
				{/if}
			</div>
			
			<div class="flex-1 overflow-auto bg-slate-50/30">
				{#if !hasFile}
					<div class="h-full flex items-center justify-center text-center p-8">
						<p class="text-gray-400 text-sm">Upload a datasheet to begin extraction.</p>
					</div>
				{:else if !isParsed}
					<div class="h-full flex flex-col items-center justify-center text-center p-8">
						<p class="text-gray-600 mb-6 text-sm">Document loaded. Ready to extract pinout map and specifications into the knowledge graph.</p>
						<button 
							on:click={handleParse}
							disabled={isScanning}
							class="px-8 py-3 bg-[#0F172A] text-white font-medium rounded-lg hover:bg-[#1E293B] transition shadow-md flex items-center gap-3 disabled:opacity-75"
						>
							{#if isScanning}
								<svg class="w-5 h-5 animate-spin" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
								</svg>
								Parsing Document...
							{:else}
								<svg class="w-5 h-5 text-blue-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19.428 15.428a2 2 0 00-1.022-.547l-2.387-.477a6 6 0 00-3.86.517l-.318.158a6 6 0 01-3.86.517L6.05 15.21a2 2 0 00-1.806.547M8 4h8l-1 1v5.172a2 2 0 00.586 1.414l5 5c1.26 1.26.367 3.414-1.415 3.414H4.828c-1.782 0-2.674-2.154-1.414-3.414l5-5A2 2 0 009 10.172V5L8 4z" />
								</svg>
								Extract & Parse Parameters
							{/if}
						</button>
					</div>
				{:else}
					<table class="w-full text-sm text-left bg-white">
						<thead class="text-xs text-gray-500 bg-gray-50 border-b border-gray-100 uppercase tracking-wide">
							<tr>
								<th class="px-6 py-4 font-semibold">Pin</th>
								<th class="px-6 py-4 font-semibold">Name</th>
								<th class="px-6 py-4 font-semibold text-right">Type</th>
							</tr>
						</thead>
						<tbody class="divide-y divide-gray-50">
							<tr class="hover:bg-gray-50/50">
								<td class="px-6 py-3 border-l-2 border-transparent">1</td>
								<td class="px-6 py-3 font-mono text-gray-800">3V3</td>
								<td class="px-6 py-3 text-right"><span class="px-2 py-0.5 bg-red-50 text-red-600 rounded text-xs">Power</span></td>
							</tr>
							<tr class="hover:bg-gray-50/50">
								<td class="px-6 py-3 border-l-2 border-transparent">2</td>
								<td class="px-6 py-3 font-mono text-gray-800">EN</td>
								<td class="px-6 py-3 text-right"><span class="px-2 py-0.5 bg-blue-50 text-blue-600 rounded text-xs">Input</span></td>
							</tr>
							<tr class="hover:bg-gray-50/50">
								<td class="px-6 py-3 border-l-2 border-transparent">3</td>
								<td class="px-6 py-3 font-mono text-gray-800">SENSOR_VP</td>
								<td class="px-6 py-3 text-right"><span class="px-2 py-0.5 bg-indigo-50 text-indigo-600 rounded text-xs">Analog I</span></td>
							</tr>
							<tr class="hover:bg-gray-50/50">
								<td class="px-6 py-3 border-l-2 border-transparent">4</td>
								<td class="px-6 py-3 font-mono text-gray-800">SENSOR_VN</td>
								<td class="px-6 py-3 text-right"><span class="px-2 py-0.5 bg-indigo-50 text-indigo-600 rounded text-xs">Analog I</span></td>
							</tr>
							<tr class="hover:bg-gray-50/50">
								<td class="px-6 py-3 border-l-2 border-transparent">15</td>
								<td class="px-6 py-3 font-mono text-gray-800">GND</td>
								<td class="px-6 py-3 text-right"><span class="px-2 py-0.5 bg-gray-100 text-gray-600 rounded text-xs">Ground</span></td>
							</tr>
						</tbody>
					</table>
					
					<div class="p-6 border-t border-gray-100 bg-white sticky bottom-0">
						<button class="w-full px-4 py-2.5 bg-[#0F172A] text-white font-medium rounded-lg hover:bg-[#1E293B] transition shadow-md">
							Confirm & Commit to Graph
						</button>
					</div>
				{/if}
			</div>
		</div>
	</div>
</div>