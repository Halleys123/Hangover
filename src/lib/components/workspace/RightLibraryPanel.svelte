<script lang="ts">
	let rightTab = $state<'components' | 'datasheets'>('components');

	let componentsDatabase = [
		{
			id: 'esp32', category: 'microcontroller', name: 'ESP32 DevKit V1', description: '3.3V Logic • WiFi/BT',
			diagram: { theme: 'blue', pins: { left: [{ id: '3v3', label: '3V3', color: 'red' }, { id: 'en', label: 'EN', color: 'blue' }, { id: 'vp', label: 'VP', color: 'green' }, { id: 'vn', label: 'VN', color: 'green' }, { id: 'd34', label: 'D34', color: 'gray' }, { id: 'd35', label: 'D35', color: 'gray' }], right: [{ id: 'gnd', label: 'GND', color: 'gray' }, { id: 'd23', label: 'D23', color: 'blue' }, { id: 'd22', label: 'D22', color: 'blue' }, { id: 'tx0', label: 'TX0', color: 'purple' }, { id: 'rx0', label: 'RX0', color: 'purple' }, { id: 'd21', label: 'D21', color: 'blue' }] } }
		},
		{
			id: 'uno', category: 'microcontroller', name: 'Arduino Uno R3', description: '5V Logic • ATmega328P',
			diagram: { theme: 'blue', pins: { left: [{ id: '5v', label: '5V', color: 'red' }, { id: '3v3', label: '3.3V', color: 'red' }, { id: 'gnd1', label: 'GND', color: 'gray' }, { id: 'gnd2', label: 'GND', color: 'gray' }, { id: 'vin', label: 'VIN', color: 'red' }], right: [{ id: 'a0', label: 'A0', color: 'green' }, { id: 'a1', label: 'A1', color: 'green' }, { id: 'd0', label: 'D0 (RX)', color: 'purple' }, { id: 'd1', label: 'D1 (TX)', color: 'purple' }, { id: 'd2', label: 'D2', color: 'blue' }] } }
		},
		{
			id: 'dht22', category: 'sensor', name: 'DHT22 Temp/Humid', description: '3.3V-5V • Digital 1-Wire',
			diagram: { theme: 'orange', pins: { left: [{ id: 'vcc', label: 'VCC', color: 'red' }, { id: 'data', label: 'DATA', color: 'blue' }, { id: 'nc', label: 'NC', color: 'gray' }, { id: 'gnd', label: 'GND', color: 'gray' }], right: [] } }
		},
		{
			id: 'mpu6050', category: 'sensor', name: 'MPU6050 IMU', description: '3.3V • I2C Interface',
			diagram: { theme: 'orange', pins: { left: [{ id: 'vcc', label: 'VCC', color: 'red' }, { id: 'gnd', label: 'GND', color: 'gray' }, { id: 'scl', label: 'SCL', color: 'pink' }, { id: 'sda', label: 'SDA', color: 'pink' }], right: [{ id: 'xda', label: 'XDA', color: 'purple' }, { id: 'xcl', label: 'XCL', color: 'purple' }, { id: 'ado', label: 'AD0', color: 'green' }, { id: 'int', label: 'INT', color: 'blue' }] } }
		}
	];

	let datasheets = $state([
		{ id: '1', name: 'ESP32-WROOM-32_Datasheet.pdf', size: '1.2 MB', parsed: true },
		{ id: '2', name: 'DHT22_Specifications.pdf', size: '450 KB', parsed: true }
	]);

	let componentSearchQuery = $state('');

	let filteredComponents = $derived(componentsDatabase.filter(c => {
		const q = componentSearchQuery.trim().toLowerCase();
		if (!q) return true;
		return c.name.toLowerCase().includes(q) || c.description.toLowerCase().includes(q) || c.category.toLowerCase().includes(q);
	}));

	let componentsByCategory = $derived(filteredComponents.reduce((acc, c) => {
		if (!acc[c.category]) acc[c.category] = [];
		acc[c.category].push(c);
		return acc;
	}, {} as Record<string, typeof componentsDatabase>));

	let anyComponentMatches = $derived(filteredComponents.length > 0);

	function handleDragStart(event: DragEvent, type: string, name: string) {
		if (event.dataTransfer) {
			const comp = componentsDatabase.find(c => c.name === name);
			if (comp) {
				event.dataTransfer.setData('application/svelteflow', JSON.stringify({ type: 'hardware', name, diagram: comp.diagram }));
			} else {
				event.dataTransfer.setData('application/svelteflow', JSON.stringify({ type: 'hardware', name }));
			}
			event.dataTransfer.effectAllowed = 'move';
		}
	}

	function openDatasheetIngestion() {
		window.location.href = '/datasheets/new';
	}
</script>

<div class="flex border-b border-gray-200">
	<button class="flex-1 py-3 text-sm font-semibold {rightTab === 'components' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-gray-500 hover:text-gray-700'}" on:click={() => rightTab = 'components'}>Components</button>
	<button class="flex-1 py-3 text-sm font-semibold {rightTab === 'datasheets' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-gray-500 hover:text-gray-700'}" on:click={() => rightTab = 'datasheets'}>Datasheets</button>
</div>

{#if rightTab === 'components'}
	<div class="p-4 border-b border-gray-200">
		<div class="relative">
			<svg class="w-4 h-4 absolute left-3 top-1/2 -translate-y-1/2 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" /></svg>
			<input type="text" bind:value={componentSearchQuery} placeholder="Search ICs, sensors..." class="w-full pl-9 pr-3 py-2 bg-gray-50 border border-gray-200 rounded text-sm focus:outline-none focus:bg-white focus:border-blue-500 focus:ring-1 focus:ring-blue-500" />
		</div>
	</div>

	<div class="flex-1 overflow-y-auto p-4">
		{#each Object.entries(componentsByCategory) as [category, categoryComponents]}
		<div class="mb-6">
			<h3 class="text-[11px] font-bold text-gray-500 uppercase tracking-wider mb-2">{category}</h3>
			<div class="space-y-2">
				{#each categoryComponents as comp (comp.id)}
				<div draggable="true" on:dragstart={(e) => handleDragStart(e, 'hardware', comp.name)} class="p-3 border border-gray-200 rounded-md hover:border-gray-300 cursor-grab flex items-center justify-between bg-white shadow-sm">
					<div class="flex items-center gap-3">
						<svg class="w-5 h-5 text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m14-6h2m-2 6h2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z" /></svg>
						<div>
							<p class="text-sm font-semibold text-gray-900">{comp.name}</p>
							<p class="text-[11px] text-gray-500">{comp.description}</p>
						</div>
					</div>
					<div class="grid grid-cols-2 gap-0.5 text-gray-300">
						<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
						<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
						<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
					</div>
				</div>
				{/each}
			</div>
		</div>
		{/each}

		{#if !anyComponentMatches}
		<div class="text-center py-12">
			<svg class="w-10 h-10 text-gray-300 mx-auto mb-3" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" /></svg>
			<p class="text-sm text-gray-500">No components found matching '{componentSearchQuery}'</p>
		</div>
		{/if}

		<div class="p-4 border-t border-gray-200 bg-gray-50">
			<button class="w-full py-2 text-sm text-blue-600 font-medium hover:text-blue-700 flex items-center justify-center gap-2">
				<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4" /></svg>
				Import Custom Part
			</button>
		</div>
	</div>
{:else}
	<div class="p-4 border-b border-gray-200">
		<button on:click={() => openDatasheetIngestion()} class="w-full py-2.5 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition flex items-center justify-center gap-2 shadow-sm">
			<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" /></svg>
			Upload Datasheet PDF
		</button>
	</div>
	
	<div class="flex-1 overflow-y-auto p-4">
		<h3 class="text-[11px] font-bold text-gray-500 uppercase tracking-wider mb-3">Project Documents</h3>
		<div class="space-y-3">
			{#each datasheets as doc}
				<div class="p-3 border border-gray-200 rounded-md bg-white hover:border-blue-400 transition-colors cursor-pointer group">
					<div class="flex items-start gap-3">
						<div class="mt-0.5 text-red-500">
							<svg class="w-5 h-5" fill="currentColor" viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 2l5 5h-5V4zM8 14h8v2H8v-2zm0-4h8v2H8v-2z" /></svg>
						</div>
						<div class="flex-1 min-w-0">
							<p class="text-sm font-medium text-gray-900 truncate group-hover:text-blue-600">{doc.name}</p>
							<div class="flex items-center gap-2 mt-1">
								<span class="text-xs text-gray-500">{doc.size}</span>
								<span class="w-1 h-1 bg-gray-300 rounded-full"></span>
								<span class="flex items-center gap-1 text-[10px] font-medium text-green-600 bg-green-50 px-1.5 py-0.5 rounded">
									<svg class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/></svg>
									Parsed
								</span>
							</div>
						</div>
					</div>
				</div>
			{/each}
		</div>
	</div>
{/if}