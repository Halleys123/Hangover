<script lang="ts">
	import { onMount } from 'svelte';
	import { api } from '$lib/api';

	interface Comp {
		_id: string;
		id?: string;
		category: string;
		name: string;
		description: string;
		diagram: { theme: string; pins: { left: any[]; right: any[] } };
	}

	interface Sheet {
		_id: string;
		name: string;
		size: string;
		parsed: boolean;
		uploadedAt: string;
	}

	let rightTab = $state<'components' | 'datasheets'>('components');
	let personalComponents = $state<Comp[]>([]);
	let catalogComponents = $state<Comp[]>([]);
	let datasheets = $state<Sheet[]>([]);
	let loadingComps = $state(true);
	let loadingSheets = $state(true);
	let componentSearchQuery = $state('');
	let showCatalog = $state(false);
	let addingId = $state<string | null>(null);

	onMount(async () => {
		const [personalRes, catalogRes, sheetsRes] = await Promise.allSettled([
			api.get<Comp[]>('/components'),
			api.get<Comp[]>('/components/catalog'),
			api.get<Sheet[]>('/datasheets')
		]);
		if (personalRes.status === 'fulfilled') personalComponents = personalRes.value;
		if (catalogRes.status === 'fulfilled') catalogComponents = catalogRes.value;
		if (sheetsRes.status === 'fulfilled') datasheets = sheetsRes.value;
		loadingComps = false;
		loadingSheets = false;
	});

	async function addToLibrary(comp: Comp) {
		addingId = comp.id ?? comp._id;
		try {
			const added = await api.post<Comp>('/components', {
				category: comp.category,
				name: comp.name,
				description: comp.description,
				diagram: comp.diagram
			});
			personalComponents = [...personalComponents, added];
		} catch { } finally {
			addingId = null;
		}
	}

	function filterList(list: Comp[]) {
		const q = componentSearchQuery.trim().toLowerCase();
		if (!q) return list;
		return list.filter(c =>
			c.name.toLowerCase().includes(q) ||
			c.description.toLowerCase().includes(q) ||
			c.category.toLowerCase().includes(q)
		);
	}

	function groupByCategory(list: Comp[]) {
		return list.reduce((acc, c) => {
			if (!acc[c.category]) acc[c.category] = [];
			acc[c.category].push(c);
			return acc;
		}, {} as Record<string, Comp[]>);
	}

	let filteredPersonal = $derived(groupByCategory(filterList(personalComponents)));
	let filteredCatalog = $derived(groupByCategory(filterList(catalogComponents)));
	let hasPersonalMatches = $derived(Object.keys(filteredPersonal).length > 0);
	let personalIds = $derived(new Set(personalComponents.map(c => c.name)));

	function handleDragStart(event: DragEvent, comp: Comp) {
		if (event.dataTransfer) {
			event.dataTransfer.setData('application/svelteflow', JSON.stringify({
				type: 'hardware',
				name: comp.name,
				diagram: comp.diagram
			}));
			event.dataTransfer.effectAllowed = 'move';
		}
	}
</script>

<div class="flex border-b border-gray-200">
	<button class="flex-1 py-3 text-sm font-semibold {rightTab === 'components' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-gray-500 hover:text-gray-700'}" onclick={() => (rightTab = 'components')}>Components</button>
	<button class="flex-1 py-3 text-sm font-semibold {rightTab === 'datasheets' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-gray-500 hover:text-gray-700'}" onclick={() => (rightTab = 'datasheets')}>Datasheets</button>
</div>

{#if rightTab === 'components'}
	<div class="p-3 border-b border-gray-200">
		<div class="relative">
			<svg class="w-4 h-4 absolute left-3 top-1/2 -translate-y-1/2 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"/></svg>
			<input type="text" bind:value={componentSearchQuery} placeholder="Search components…" class="w-full pl-9 pr-3 py-2 bg-gray-50 border border-gray-200 rounded text-sm focus:outline-none focus:bg-white focus:border-blue-500" />
		</div>
	</div>

	<div class="flex-1 overflow-y-auto">
		<!-- My Library -->
		<div class="p-3 pb-1">
			<h3 class="text-[10px] font-bold text-gray-400 uppercase tracking-wider mb-2">My Library</h3>
		</div>

		{#if loadingComps}
			{#each [1,2,3] as i (i)}<div class="mx-3 mb-2 h-14 bg-gray-100 rounded animate-pulse"></div>{/each}
		{:else if !hasPersonalMatches && componentSearchQuery}
			<p class="px-3 pb-3 text-xs text-gray-400">No results for "{componentSearchQuery}"</p>
		{:else if personalComponents.length === 0}
			<div class="px-3 pb-3 text-xs text-gray-400">
				<p>Your library is empty.</p>
				<button onclick={() => (showCatalog = true)} class="text-blue-500 hover:underline mt-1">Browse catalog →</button>
			</div>
		{:else}
			{#each Object.entries(filteredPersonal) as [category, comps]}
			<div class="px-3 mb-4">
				<p class="text-[10px] font-semibold text-gray-400 uppercase tracking-wider mb-1.5">{category}</p>
				<div class="space-y-1.5">
					{#each comps as comp (comp._id)}
					<div draggable="true" ondragstart={(e) => handleDragStart(e, comp)} class="p-2.5 border border-gray-200 rounded cursor-grab hover:border-blue-300 hover:shadow-sm bg-white flex items-center gap-2.5">
						<svg class="w-4 h-4 text-gray-400 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m14-6h2m-2 6h2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z"/></svg>
						<div class="min-w-0">
							<p class="text-xs font-semibold text-gray-900 truncate">{comp.name}</p>
							<p class="text-[10px] text-gray-400 truncate">{comp.description}</p>
						</div>
					</div>
					{/each}
				</div>
			</div>
			{/each}
		{/if}

		<!-- Catalog -->
		<div class="px-3 pt-1 pb-1 border-t border-gray-100 mt-1">
			<button onclick={() => (showCatalog = !showCatalog)} class="flex items-center gap-1 text-[10px] font-bold text-gray-400 uppercase tracking-wider hover:text-gray-600 w-full">
				<svg class="w-3 h-3 transition-transform {showCatalog ? 'rotate-90' : ''}" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/></svg>
				Catalog {#if catalogComponents.length > 0}({catalogComponents.length}){/if}
			</button>
		</div>

		{#if showCatalog}
		{#each Object.entries(filteredCatalog) as [category, comps]}
		<div class="px-3 mb-3">
			<p class="text-[10px] font-semibold text-gray-400 uppercase tracking-wider mb-1.5">{category}</p>
			<div class="space-y-1.5">
				{#each comps as comp (comp.id ?? comp.name)}
				<div class="p-2.5 border border-dashed border-gray-200 rounded bg-gray-50 flex items-center gap-2.5 group">
					<div draggable="true" ondragstart={(e) => handleDragStart(e, comp)} class="flex-1 flex items-center gap-2 cursor-grab min-w-0">
						<svg class="w-4 h-4 text-gray-300 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m14-6h2m-2 6h2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z"/></svg>
						<div class="min-w-0">
							<p class="text-xs font-semibold text-gray-600 truncate">{comp.name}</p>
							<p class="text-[10px] text-gray-400 truncate">{comp.description}</p>
						</div>
					</div>
					{#if !personalIds.has(comp.name)}
					<button onclick={() => addToLibrary(comp)} disabled={addingId === (comp.id ?? comp.name)} class="shrink-0 text-[10px] text-blue-500 hover:text-blue-700 font-medium disabled:opacity-50">
						{addingId === (comp.id ?? comp.name) ? '…' : '+Add'}
					</button>
					{:else}
					<span class="shrink-0 text-[10px] text-green-500">✓</span>
					{/if}
				</div>
				{/each}
			</div>
		</div>
		{/each}
		{/if}
	</div>

{:else}
	<!-- Datasheets Tab -->
	<div class="p-3 border-b border-gray-200">
		<a href="/datasheets" class="flex items-center justify-center gap-2 w-full py-2 bg-blue-600 text-white text-xs font-medium rounded hover:bg-blue-700 transition">
			<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12"/></svg>
			Upload &amp; Manage Datasheets
		</a>
	</div>

	<div class="flex-1 overflow-y-auto p-3">
		{#if loadingSheets}
			{#each [1,2] as i (i)}<div class="mb-2 h-16 bg-gray-100 rounded animate-pulse"></div>{/each}
		{:else if datasheets.length === 0}
			<div class="text-center py-8 text-gray-400">
				<svg class="w-8 h-8 mx-auto mb-2 text-gray-300" fill="currentColor" viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 2l5 5h-5V4z"/></svg>
				<p class="text-xs">No datasheets uploaded yet.</p>
				<a href="/datasheets" class="text-blue-500 text-xs hover:underline mt-1 inline-block">Upload one →</a>
			</div>
		{:else}
			<h3 class="text-[10px] font-bold text-gray-400 uppercase tracking-wider mb-2">Project Documents</h3>
			<div class="space-y-2">
				{#each datasheets as doc (doc._id)}
				<a href="/datasheets" class="block p-2.5 border border-gray-200 rounded bg-white hover:border-blue-300 hover:shadow-sm transition group">
					<div class="flex items-start gap-2">
						<svg class="w-4 h-4 mt-0.5 text-red-400 shrink-0" fill="currentColor" viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 2l5 5h-5V4z"/></svg>
						<div class="flex-1 min-w-0">
							<p class="text-xs font-medium text-gray-800 truncate group-hover:text-blue-600">{doc.name}</p>
							<div class="flex items-center gap-1.5 mt-0.5">
								<span class="text-[10px] text-gray-400">{doc.size}</span>
								{#if doc.parsed}
								<span class="text-[9px] font-semibold text-green-600 bg-green-50 px-1 py-0.5 rounded">Parsed</span>
								{:else}
								<span class="text-[9px] font-semibold text-yellow-600 bg-yellow-50 px-1 py-0.5 rounded">Processing…</span>
								{/if}
							</div>
						</div>
					</div>
				</a>
				{/each}
			</div>
		{/if}
	</div>
{/if}
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