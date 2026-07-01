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
			const payload = JSON.stringify({
				type: 'hardware',
				name: comp.name,
				diagram: comp.diagram
			});
			event.dataTransfer.setData('application/svelteflow', payload);
			event.dataTransfer.setData('text/plain', payload);
			event.dataTransfer.effectAllowed = 'move';
		}
	}
</script>

<!-- RIGHT PANEL: COMPONENT & DATASHEET LIBRARY -->
<div class="flex border-b border-slate-200 dark:border-slate-800 bg-white dark:bg-slate-900 transition-colors">
	<button class="flex-1 py-3 text-sm font-semibold transition-colors {rightTab === 'components' ? 'text-blue-600 dark:text-blue-400 border-b-2 border-blue-600 dark:border-blue-400' : 'text-slate-500 dark:text-slate-400 hover:text-slate-700 dark:hover:text-slate-200'}" onclick={() => (rightTab = 'components')}>Components</button>
	<button class="flex-1 py-3 text-sm font-semibold transition-colors {rightTab === 'datasheets' ? 'text-blue-600 dark:text-blue-400 border-b-2 border-blue-600 dark:border-blue-400' : 'text-slate-500 dark:text-slate-400 hover:text-slate-700 dark:hover:text-slate-200'}" onclick={() => (rightTab = 'datasheets')}>Datasheets</button>
</div>

{#if rightTab === 'components'}
	<div class="p-3 border-b border-slate-200 dark:border-slate-800 bg-white dark:bg-slate-900">
		<div class="relative">
			<svg class="w-4 h-4 absolute left-3 top-1/2 -translate-y-1/2 text-slate-400" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"/></svg>
			<input type="text" bind:value={componentSearchQuery} placeholder="Search components…" class="w-full pl-9 pr-3 py-2 bg-slate-50 dark:bg-slate-800 border border-slate-200 dark:border-slate-700 rounded text-sm text-slate-900 dark:text-white placeholder:text-slate-400 focus:outline-none focus:bg-white dark:focus:bg-slate-900 focus:border-blue-500 transition-all" />
		</div>
	</div>

	<div class="flex-1 overflow-y-auto transition-colors">
		<!-- My Library -->
		<div class="p-3 pb-1">
			<h3 class="text-[10px] font-bold text-slate-400 dark:text-slate-500 uppercase tracking-wider mb-2">My Library</h3>
		</div>

		{#if loadingComps}
			{#each [1,2,3] as i (i)}<div class="mx-3 mb-2 h-14 bg-slate-100 dark:bg-slate-800 rounded animate-pulse"></div>{/each}
		{:else if !hasPersonalMatches && componentSearchQuery}
			<p class="px-3 pb-3 text-xs text-slate-400 dark:text-slate-500">No results for "{componentSearchQuery}"</p>
		{:else if personalComponents.length === 0}
			<div class="px-3 pb-3 text-xs text-slate-400 dark:text-slate-500">
				<p>Your library is empty.</p>
				<button onclick={() => (showCatalog = true)} class="text-blue-500 dark:text-blue-400 hover:underline mt-1">Browse catalog →</button>
			</div>
		{:else}
			{#each Object.entries(filteredPersonal) as [category, comps]}
			<div class="px-3 mb-4">
				<p class="text-[10px] font-semibold text-slate-400 dark:text-slate-500 uppercase tracking-wider mb-1.5">{category}</p>
				<div class="space-y-1.5">
					{#each comps as comp (comp._id)}
					<div draggable="true" ondragstart={(e) => handleDragStart(e, comp)} class="p-2.5 border border-slate-200 dark:border-slate-800 rounded cursor-grab hover:border-blue-300 dark:hover:border-blue-500 hover:shadow-sm bg-white dark:bg-slate-800/80 flex items-center gap-2.5 transition-colors">
						<svg class="w-4 h-4 text-slate-400 dark:text-slate-500 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m14-6h2m-2 6h2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z"/></svg>
						<div class="min-w-0">
							<p class="text-xs font-semibold text-slate-900 dark:text-white truncate">{comp.name}</p>
							<p class="text-[10px] text-slate-400 dark:text-slate-400 truncate">{comp.description}</p>
						</div>
					</div>
					{/each}
				</div>
			</div>
			{/each}
		{/if}

		<!-- Catalog -->
		<div class="px-3 pt-1 pb-1 border-t border-slate-100 dark:border-slate-800 mt-1">
			<button onclick={() => (showCatalog = !showCatalog)} class="flex items-center gap-1 text-[10px] font-bold text-slate-400 dark:text-slate-500 uppercase tracking-wider hover:text-slate-600 dark:hover:text-slate-300 w-full transition-colors">
				<svg class="w-3 h-3 transition-transform {showCatalog ? 'rotate-90' : ''}" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/></svg>
				Catalog {#if catalogComponents.length > 0}({catalogComponents.length}){/if}
			</button>
		</div>

		{#if showCatalog}
		{#each Object.entries(filteredCatalog) as [category, comps]}
		<div class="px-3 mb-3">
			<p class="text-[10px] font-semibold text-slate-400 dark:text-slate-500 uppercase tracking-wider mb-1.5">{category}</p>
			<div class="space-y-1.5">
				{#each comps as comp (comp.id ?? comp.name)}
				<div class="p-2.5 border border-dashed border-slate-200 dark:border-slate-800 rounded bg-slate-50 dark:bg-slate-800/40 flex items-center gap-2.5 group transition-colors">
					<div draggable="true" ondragstart={(e) => handleDragStart(e, comp)} class="flex-1 flex items-center gap-2 cursor-grab min-w-0">
						<svg class="w-4 h-4 text-slate-300 dark:text-slate-600 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m14-6h2m-2 6h2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z"/></svg>
						<div class="min-w-0">
							<p class="text-xs font-semibold text-slate-700 dark:text-slate-200 truncate">{comp.name}</p>
							<p class="text-[10px] text-slate-400 dark:text-slate-400 truncate">{comp.description}</p>
						</div>
					</div>
					{#if !personalIds.has(comp.name)}
					<button onclick={() => addToLibrary(comp)} disabled={addingId === (comp.id ?? comp.name)} class="shrink-0 text-[10px] text-blue-500 dark:text-blue-400 hover:text-blue-700 dark:hover:text-blue-300 font-medium disabled:opacity-50 transition-colors">
						{addingId === (comp.id ?? comp.name) ? '…' : '+Add'}
					</button>
					{:else}
					<span class="shrink-0 text-[10px] text-emerald-500 dark:text-emerald-400">✓</span>
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
	<div class="p-3 border-b border-slate-200 dark:border-slate-800 bg-white dark:bg-slate-900 transition-colors">
		<a href="/datasheets" class="flex items-center justify-center gap-2 w-full py-2 bg-blue-600 text-white text-xs font-medium rounded hover:bg-blue-700 transition">
			<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12"/></svg>
			Upload &amp; Manage Datasheets
		</a>
	</div>

	<div class="flex-1 overflow-y-auto p-3 transition-colors">
		{#if loadingSheets}
			{#each [1,2] as i (i)}<div class="mb-2 h-16 bg-slate-100 dark:bg-slate-800 rounded animate-pulse"></div>{/each}
		{:else if datasheets.length === 0}
			<div class="text-center py-8 text-slate-400 dark:text-slate-500">
				<svg class="w-8 h-8 mx-auto mb-2 text-slate-300 dark:text-slate-600" fill="currentColor" viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 2l5 5h-5V4z"/></svg>
				<p class="text-xs">No datasheets uploaded yet.</p>
				<a href="/datasheets" class="text-blue-500 dark:text-blue-400 text-xs hover:underline mt-1 inline-block">Upload one →</a>
			</div>
		{:else}
			<h3 class="text-[10px] font-bold text-slate-400 dark:text-slate-500 uppercase tracking-wider mb-2">Project Documents</h3>
			<div class="space-y-2">
				{#each datasheets as doc (doc._id)}
				<a href="/datasheets" class="block p-2.5 border border-slate-200 dark:border-slate-800 rounded bg-white dark:bg-slate-800/80 hover:border-blue-300 dark:hover:border-blue-500 hover:shadow-sm transition group">
					<div class="flex items-start gap-2">
						<svg class="w-4 h-4 mt-0.5 text-rose-400 dark:text-rose-500 shrink-0" fill="currentColor" viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 2l5 5h-5V4z"/></svg>
						<div class="flex-1 min-w-0">
							<p class="text-xs font-medium text-slate-800 dark:text-slate-200 truncate group-hover:text-blue-600 dark:group-hover:text-blue-400">{doc.name}</p>
							<div class="flex items-center gap-1.5 mt-0.5">
								<span class="text-[10px] text-slate-400 dark:text-slate-500">{doc.size}</span>
								{#if doc.parsed}
								<span class="text-[9px] font-semibold text-emerald-600 dark:text-emerald-300 bg-emerald-50 dark:bg-emerald-950/60 px-1 py-0.5 rounded">Parsed</span>
								{:else}
								<span class="text-[9px] font-semibold text-amber-600 dark:text-amber-300 bg-amber-50 dark:bg-amber-950/60 px-1 py-0.5 rounded">Processing…</span>
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