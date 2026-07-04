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
		datasheetId?: string;
	}

	interface Sheet {
		_id: string;
		name: string;
		size: string;
		parsed: boolean;
		status?: 'waiting' | 'processing' | 'completed' | 'failed';
		error?: string;
		uploadedAt: string;
	}

	interface Props {
		workspaceId?: string;
		projectDatasheets?: Sheet[];
		projectComponents?: string[];
		onComponentClick?: (comp: Comp) => void;
	}

	let { workspaceId, projectDatasheets = $bindable([]), projectComponents = $bindable([]), onComponentClick }: Props = $props();

	let rightTab = $state<'components' | 'datasheets'>('components');
	let personalComponents = $state<Comp[]>([]);
	let catalogComponents = $state<Comp[]>([]);
	let allDatasheets = $state<Sheet[]>([]);
	let loadingComps = $state(true);
	let loadingSheets = $state(true);
	let componentSearchQuery = $state('');
	let showCatalog = $state(false);
	let addingId = $state<string | null>(null);
	let showAttachModal = $state(false);
	let attachingSheetId = $state<string | null>(null);

	let availableDatasheets = $derived(allDatasheets.filter(d => !projectDatasheets.find(pd => pd._id === d._id)));
	let pendingDatasheets = $derived(allDatasheets.filter(d => !d.parsed && d.status !== 'failed'));
	
	let hasAnalyzedDatasheet = $derived(
		projectDatasheets.some(d => d.parsed || d.status === 'completed') ||
		allDatasheets.some(d => d.parsed || d.status === 'completed')
	);

	// Scoped personal library for the current project
	let myLibraryList = $derived(
		workspaceId
			? Array.from(new Map(personalComponents
				.filter(c => projectComponents.includes(c.name) || projectComponents.includes(c._id) || (c.id && projectComponents.includes(c.id)))
				.map(c => [c.name.trim().toLowerCase(), c])).values())
			: Array.from(new Map(personalComponents.map(c => [c.name.trim().toLowerCase(), c])).values())
	);

	// Catalog items available to add to this project (only user's extracted components not yet added and matching project's datasheets)
	let availableCatalogList = $derived(
		workspaceId
			? Array.from(new Map([...personalComponents, ...catalogComponents]
				.filter(c => {
					if (!c.datasheetId) return false;
					const cDsId = typeof c.datasheetId === 'object' ? (c.datasheetId as any)._id : c.datasheetId;
					return (
						!projectComponents.includes(c.name) && 
						!projectComponents.includes(c._id) && 
						projectDatasheets.some(d => d._id === cDsId)
					);
				})
				.map(c => [c.name.trim().toLowerCase(), c])).values())
			: Array.from(new Map([...personalComponents, ...catalogComponents].map(c => [c.name.trim().toLowerCase(), c])).values())
	);

	onMount(async () => {
		const [personalRes, catalogRes, sheetsRes] = await Promise.allSettled([
			api.get<Comp[]>('/components'),
			api.get<Comp[]>('/components/catalog'),
			api.get<Sheet[]>('/datasheets')
		]);
		if (personalRes.status === 'fulfilled') personalComponents = personalRes.value;
		if (catalogRes.status === 'fulfilled') catalogComponents = catalogRes.value;
		if (sheetsRes.status === 'fulfilled') allDatasheets = sheetsRes.value;
		loadingComps = false;
		loadingSheets = false;
	});

	// Live polling while any datasheet is uploading/parsing
	let pollTimer: any = null;
	$effect(() => {
		if (pendingDatasheets.length > 0 && !pollTimer) {
			pollTimer = setInterval(async () => {
				try {
					const [sheets, comps] = await Promise.all([
						api.get<Sheet[]>('/datasheets'),
						api.get<Comp[]>('/components')
					]);
					allDatasheets = sheets;
					personalComponents = comps;
				} catch {}
			}, 3000);
		} else if (pendingDatasheets.length === 0 && pollTimer) {
			clearInterval(pollTimer);
			pollTimer = null;
		}
		return () => { if (pollTimer) clearInterval(pollTimer); };
	});

	async function addToLibrary(comp: Comp) {
		addingId = comp.id ?? comp._id ?? comp.name;
		try {
			if (workspaceId) {
				await api.post(`/projects/${workspaceId}/components`, { name: comp.name });
				if (!projectComponents.includes(comp.name)) {
					projectComponents = [...projectComponents, comp.name];
				}
			}
			if (!personalComponents.some(p => (p.name && p.name.trim().toLowerCase() === comp.name.trim().toLowerCase()) || p._id === comp._id)) {
				const added = await api.post<Comp>('/components', {
					category: comp.category,
					name: comp.name,
					description: comp.description,
					diagram: comp.diagram
				});
				personalComponents = [...personalComponents, added];
			}
		} catch { } finally {
			addingId = null;
		}
	}

	async function removeFromProject(compName: string, e: MouseEvent) {
		e.preventDefault();
		e.stopPropagation();
		if (!workspaceId) return;
		try {
			await api.delete(`/projects/${workspaceId}/components/${encodeURIComponent(compName)}`);
			projectComponents = projectComponents.filter(n => n !== compName);
		} catch {}
	}

	async function attachDatasheet(sheet: Sheet) {
		if (!workspaceId) return;
		attachingSheetId = sheet._id;
		try {
			await api.post(`/projects/${workspaceId}/datasheets`, { datasheetId: sheet._id });
			projectDatasheets = [...projectDatasheets, sheet];
			window.dispatchEvent(new CustomEvent('chat-reset'));
		} catch (err) {
			console.error(err);
		} finally {
			attachingSheetId = null;
			showAttachModal = false;
		}
	}

	async function detachDatasheet(sheetId: string, e: MouseEvent) {
		e.preventDefault();
		e.stopPropagation();
		if (!workspaceId) return;
		try {
			await api.delete(`/projects/${workspaceId}/datasheets/${sheetId}`);
			projectDatasheets = projectDatasheets.filter(d => d._id !== sheetId);
		} catch (err) {
			console.error(err);
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

	let filteredPersonal = $derived(groupByCategory(filterList(myLibraryList)));
	let filteredCatalog = $derived(groupByCategory(filterList(availableCatalogList)));
	let hasPersonalMatches = $derived(Object.keys(filteredPersonal).length > 0);

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
<div class="flex border-b border-slate-200 dark:border-zinc-800 bg-white dark:bg-zinc-900 transition-colors">
	<button class="flex-1 py-3 text-sm font-semibold transition-colors {rightTab === 'components' ? 'text-blue-600 dark:text-blue-400 border-b-2 border-blue-600 dark:border-blue-400' : 'text-slate-500 dark:text-zinc-400 hover:text-slate-700 dark:hover:text-zinc-200'}" onclick={() => (rightTab = 'components')}>Components</button>
	<button class="flex-1 py-3 text-sm font-semibold transition-colors {rightTab === 'datasheets' ? 'text-blue-600 dark:text-blue-400 border-b-2 border-blue-600 dark:border-blue-400' : 'text-slate-500 dark:text-zinc-400 hover:text-slate-700 dark:hover:text-zinc-200'}" onclick={() => (rightTab = 'datasheets')}>Datasheets</button>
</div>

{#if rightTab === 'components'}
	<div class="p-3 border-b border-slate-200 dark:border-zinc-800 bg-white dark:bg-zinc-900">
		<div class="relative">
			<svg class="w-4 h-4 absolute left-3 top-1/2 -translate-y-1/2 text-slate-400" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"/></svg>
			<input type="text" bind:value={componentSearchQuery} placeholder="Search components…" class="w-full pl-9 pr-3 py-2 bg-slate-50 dark:bg-zinc-800 border border-slate-200 dark:border-zinc-700 rounded text-sm text-slate-900 dark:text-white placeholder:text-slate-400 focus:outline-none focus:bg-white dark:focus:bg-slate-900 focus:border-blue-500 transition-all" />
		</div>
	</div>

	<div class="flex-1 overflow-y-auto transition-colors">
		<!-- My Library (Scoped to current project) -->
		<div class="p-3 pb-1">
			<h3 class="text-[10px] font-bold text-slate-400 dark:text-zinc-500 uppercase tracking-wider mb-1">My Library</h3>
			<p class="text-[10px] text-slate-400 dark:text-zinc-500 leading-normal">
				💡 Drag components onto the canvas or click them to place instantly.
			</p>
		</div>

		{#if loadingComps}
			{#each [1,2,3] as i (i)}<div class="mx-3 mb-2 h-14 bg-slate-100 dark:bg-zinc-800 rounded animate-pulse"></div>{/each}
		{:else if !hasPersonalMatches && componentSearchQuery}
			<p class="px-3 pb-3 text-xs text-slate-400 dark:text-zinc-500">No results for "{componentSearchQuery}"</p>
		{:else if myLibraryList.length === 0}
			<div class="px-3 pb-3 text-xs text-slate-400 dark:text-zinc-500">
				<p>No components added to this project yet.</p>
				{#if hasAnalyzedDatasheet || availableCatalogList.length > 0}
					<button onclick={() => (showCatalog = true)} class="text-blue-500 dark:text-blue-400 font-medium hover:underline mt-1">Browse catalog below to add →</button>
				{:else}
					<p class="mt-1 text-[10px] text-amber-500 dark:text-amber-400">Upload & analyze a datasheet to unlock the component catalog!</p>
				{/if}
			</div>
		{:else}
			{#each Object.entries(filteredPersonal) as [category, comps]}
			<div class="px-3 mb-4">
				<p class="text-[10px] font-semibold text-slate-400 dark:text-zinc-500 uppercase tracking-wider mb-1.5">{category}</p>
				<div class="space-y-1.5">
					{#each comps as comp (comp._id || comp.name)}
					<div 
						draggable="true" 
						ondragstart={(e) => handleDragStart(e, comp)} 
						onclick={() => onComponentClick && onComponentClick(comp)}
						class="group p-2.5 border border-slate-200 dark:border-zinc-800 rounded cursor-pointer hover:bg-slate-50 dark:hover:bg-zinc-800/50 hover:border-blue-300 dark:hover:border-blue-500 hover:shadow-sm bg-white dark:bg-zinc-800/80 flex items-center justify-between gap-2.5 transition-colors"
					>
						<div class="flex items-center gap-2.5 min-w-0">
							<svg class="w-4 h-4 text-slate-400 dark:text-zinc-500 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m14-6h2m-2 6h2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z"/></svg>
							<div class="min-w-0">
								<p class="text-xs font-semibold text-slate-900 dark:text-white truncate">{comp.name}</p>
								<p class="text-[10px] text-slate-400 dark:text-zinc-400 truncate">{comp.description}</p>
							</div>
						</div>
						{#if workspaceId}
							<button onclick={(e) => removeFromProject(comp.name, e)} title="Remove from project library" class="opacity-0 group-hover:opacity-100 text-slate-400 hover:text-rose-500 p-1 transition-opacity">
								<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/></svg>
							</button>
						{/if}
					</div>
					{/each}
				</div>
			</div>
			{/each}
		{/if}

		<!-- Catalog Section -->
		{#if hasAnalyzedDatasheet || availableCatalogList.length > 0}
		<div class="px-3 pt-1 pb-1 border-t border-slate-100 dark:border-zinc-800 mt-1">
			<button onclick={() => (showCatalog = !showCatalog)} class="flex items-center gap-1 text-[10px] font-bold text-slate-400 dark:text-zinc-500 uppercase tracking-wider hover:text-slate-600 dark:hover:text-zinc-300 w-full transition-colors">
				<svg class="w-3 h-3 transition-transform {showCatalog ? 'rotate-90' : ''}" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/></svg>
				Catalog {#if availableCatalogList.length > 0}({availableCatalogList.length}){/if}
			</button>
		</div>

		{#if showCatalog}
			<!-- Live Datasheet Processing Status -->
			{#if pendingDatasheets.length > 0}
			<div class="px-3 mb-3">
				<p class="text-[10px] font-semibold text-amber-600 dark:text-amber-400 uppercase tracking-wider mb-1.5 flex items-center gap-1.5">
					<span class="w-2 h-2 rounded-full bg-amber-500 animate-ping"></span> Live AI Datasheet Indexing
				</p>
				<div class="space-y-1.5">
					{#each pendingDatasheets as sheet (sheet._id)}
					<div class="p-2.5 border border-amber-200 dark:border-amber-900/60 rounded bg-amber-50/50 dark:bg-amber-950/30 flex items-center justify-between text-xs">
						<div class="min-w-0 flex items-center gap-2">
							<svg class="w-4 h-4 text-amber-500 animate-spin shrink-0" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path></svg>
							<span class="font-medium text-slate-800 dark:text-zinc-200 truncate">{sheet.name}</span>
						</div>
						<span class="text-[10px] text-amber-600 dark:text-amber-400 font-semibold uppercase">{sheet.status || 'parsing...'}</span>
					</div>
					{/each}
				</div>
			</div>
			{/if}

			{#each Object.entries(filteredCatalog) as [category, comps]}
			<div class="px-3 mb-3">
				<p class="text-[10px] font-semibold text-slate-400 dark:text-zinc-500 uppercase tracking-wider mb-1.5">{category}</p>
				<div class="space-y-1.5">
					{#each comps as comp (comp.id ?? comp.name)}
					<div class="p-2.5 border border-dashed border-slate-200 dark:border-zinc-800 rounded bg-slate-50 dark:bg-zinc-800/40 flex items-center gap-2.5 group transition-colors">
						<div draggable="true" ondragstart={(e) => handleDragStart(e, comp)} class="flex-1 flex items-center gap-2 cursor-grab min-w-0">
							<svg class="w-4 h-4 text-slate-300 dark:text-zinc-600 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m14-6h2m-2 6h2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z"/></svg>
							<div class="min-w-0">
								<p class="text-xs font-semibold text-slate-700 dark:text-zinc-200 truncate">{comp.name}</p>
								<p class="text-[10px] text-slate-400 dark:text-zinc-400 truncate">{comp.description}</p>
							</div>
						</div>
						<button onclick={() => addToLibrary(comp)} disabled={addingId === (comp.id ?? comp.name)} class="shrink-0 px-2 py-1 bg-blue-50 dark:bg-blue-950/60 text-[10px] text-blue-600 dark:text-blue-400 hover:bg-blue-100 dark:hover:bg-blue-900 rounded font-bold disabled:opacity-50 transition-colors">
							{addingId === (comp.id ?? comp.name) ? '…' : '+ Add'}
						</button>
					</div>
					{/each}
				</div>
			</div>
			{/each}
		{/if}
		{/if}
	</div>

{:else}
	<!-- Datasheets Tab -->
	<div class="p-3 border-b border-slate-200 dark:border-zinc-800 bg-white dark:bg-zinc-900 transition-colors">
		<button onclick={() => (showAttachModal = !showAttachModal)} class="flex items-center justify-center gap-2 w-full py-2 bg-slate-800 dark:bg-zinc-800 text-white text-xs font-medium rounded hover:bg-slate-700 dark:hover:bg-zinc-700 transition">
			<svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4"/></svg>
			Attach from Library
		</button>
	</div>

	<div class="flex-1 overflow-y-auto transition-colors">
		<div class="p-3 pb-1">
			<h3 class="text-[10px] font-bold text-slate-400 dark:text-zinc-500 uppercase tracking-wider mb-2">Attached Datasheets</h3>
		</div>

		{#if loadingSheets}
			{#each [1,2] as i (i)}<div class="mx-3 mb-2 h-14 bg-slate-100 dark:bg-zinc-800 rounded animate-pulse"></div>{/each}
		{:else if projectDatasheets.length === 0}
			<div class="px-3 pb-3 text-xs text-slate-400 dark:text-zinc-500">
				No datasheets attached to this project yet. Use "Attach from Library" above.
			</div>
		{:else}
			<div class="px-3 space-y-2">
				{#each projectDatasheets as sheet (sheet._id)}
				<div class="p-3 border border-slate-200 dark:border-zinc-800 rounded bg-white dark:bg-zinc-800/80 flex items-center justify-between gap-2 transition-colors">
					<div class="flex items-center gap-2 min-w-0">
						<svg class="w-4 h-4 text-rose-500 shrink-0" fill="currentColor" viewBox="0 0 20 20"><path fill-rule="evenodd" d="M4 4a2 2 0 012-2h4.586A2 2 0 0112 2.586L15.414 6A2 2 0 0116 7.414V16a2 2 0 01-2 2H6a2 2 0 01-2-2zm2 6a1 1 0 011-1h6a1 1 0 110 2H7a1 1 0 01-1-1zm1 3a1 1 0 100 2h6a1 1 0 100-2H7z" clip-rule="evenodd"/></svg>
						<div class="min-w-0">
							<p class="text-xs font-semibold text-slate-900 dark:text-white truncate">{sheet.name}</p>
							<div class="flex items-center gap-2 text-[10px] text-slate-400 dark:text-zinc-400">
								<span>{sheet.size}</span>
								{#if sheet.parsed}
									<span class="text-emerald-500 dark:text-emerald-400 font-medium">● Indexed</span>
								{:else}
									<span class="text-amber-500 dark:text-amber-400 font-medium animate-pulse">● {sheet.status || 'Processing'}</span>
								{/if}
							</div>
						</div>
					</div>
					<button onclick={(e) => detachDatasheet(sheet._id, e)} class="text-slate-400 hover:text-rose-500 p-1 transition-colors" title="Detach from project">
						<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/></svg>
					</button>
				</div>
				{/each}
			</div>
		{/if}
	</div>
{/if}

<!-- Attach Modal -->
{#if showAttachModal}
<div class="fixed inset-0 z-50 flex items-center justify-center bg-slate-900/50 dark:bg-black/60 p-4 backdrop-blur-sm">
	<div class="bg-white dark:bg-zinc-900 border border-slate-200 dark:border-zinc-800 rounded-xl shadow-xl w-full max-w-md overflow-hidden transition-colors">
		<div class="px-4 py-3 border-b border-slate-200 dark:border-zinc-800 flex items-center justify-between">
			<h3 class="text-sm font-semibold text-slate-900 dark:text-white">Attach Datasheet to Project</h3>
			<button onclick={() => (showAttachModal = false)} class="text-slate-400 hover:text-slate-600 dark:hover:text-white">
				<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/></svg>
			</button>
		</div>
		<div class="p-4 max-h-80 overflow-y-auto space-y-2">
			{#if availableDatasheets.length === 0}
				<p class="text-xs text-slate-400 text-center py-4">No available datasheets found in library.</p>
			{:else}
				{#each availableDatasheets as sheet (sheet._id)}
				<div class="p-2.5 border border-slate-200 dark:border-zinc-800 rounded flex items-center justify-between gap-2">
					<div class="min-w-0">
						<p class="text-xs font-medium text-slate-900 dark:text-white truncate">{sheet.name}</p>
						<span class="text-[10px] text-slate-400">{sheet.size}</span>
					</div>
					<button
						onclick={() => attachDatasheet(sheet)}
						disabled={attachingSheetId === sheet._id}
						class="px-3 py-1.5 bg-blue-600 text-white text-xs font-medium rounded hover:bg-blue-700 disabled:opacity-50 transition"
					>
						{attachingSheetId === sheet._id ? 'Attaching…' : 'Attach'}
					</button>
				</div>
				{/each}
			{/if}
		</div>
	</div>
</div>
{/if}