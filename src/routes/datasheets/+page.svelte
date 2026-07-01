<script lang="ts">
	import { onMount } from 'svelte';
	import NavBar from '$lib/components/NavBar.svelte';
	import { authUser, getToken } from '$lib/stores/auth';
	import { api } from '$lib/api';

	interface Datasheet {
		_id: string;
		name: string;
		size: string;
		parsed: boolean;
		uploadedAt: string;
		cogneeConfig: Record<string, unknown> | null;
	}

	let datasheets = $state<Datasheet[]>([]);
	let loading = $state(true);
	let uploading = $state(false);
	let selectedId = $state<string | null>(null);
	let pdfBlobUrl = $state<string | null>(null);
	let loadingPdf = $state(false);
	let uploadError = $state('');

	onMount(async () => {
		if (!$authUser) { window.location.href = '/login'; return; }
		try {
			datasheets = await api.get<Datasheet[]>('/datasheets');
		} catch { }
		finally { loading = false; }
	});

	async function viewDatasheet(id: string) {
		if (pdfBlobUrl) URL.revokeObjectURL(pdfBlobUrl);
		selectedId = id;
		loadingPdf = true;
		try {
			const token = getToken();
			const res = await fetch(`http://localhost:3000/api/datasheets/${id}/file`, {
				headers: token ? { Authorization: `Bearer ${token}` } : {}
			});
			if (!res.ok) throw new Error('Failed to load PDF');
			const blob = await res.blob();
			pdfBlobUrl = URL.createObjectURL(blob);
		} catch {
			pdfBlobUrl = null;
		} finally {
			loadingPdf = false;
		}
	}

	async function handleUpload(e: Event) {
		const input = e.target as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;
		uploadError = '';
		uploading = true;
		try {
			const form = new FormData();
			form.append('file', file);
			const sheet = await api.upload<Datasheet>('/datasheets', form);
			datasheets = [sheet, ...datasheets];
		} catch (err: any) {
			uploadError = err.message;
		} finally {
			uploading = false;
			input.value = '';
		}
	}

	async function deleteDatasheet(id: string, e: MouseEvent) {
		e.stopPropagation();
		if (!confirm('Delete this datasheet?')) return;
		try {
			await api.delete(`/datasheets/${id}`);
			if (selectedId === id) { selectedId = null; if (pdfBlobUrl) { URL.revokeObjectURL(pdfBlobUrl); pdfBlobUrl = null; } }
			datasheets = datasheets.filter((d) => d._id !== id);
		} catch (err: any) {
			alert(err.message);
		}
	}

	function formatDate(iso: string) {
		try { return new Date(iso).toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' }); }
		catch { return iso; }
	}
</script>

<div class="h-screen flex flex-col bg-gray-50 overflow-hidden">
	<NavBar activeLink="datasheets" onLogoClick={() => (window.location.href = '/')} />

	<div class="flex-1 flex overflow-hidden">
		<!-- Left: PDF Viewer -->
		<div class="flex-1 flex flex-col bg-white border-r border-gray-200 overflow-hidden">
			{#if loadingPdf}
				<div class="flex-1 flex items-center justify-center">
					<div class="flex flex-col items-center gap-3 text-gray-400">
						<div class="w-8 h-8 border-2 border-blue-400 border-t-transparent rounded-full animate-spin"></div>
						<p class="text-sm">Loading PDF…</p>
					</div>
				</div>
			{:else if pdfBlobUrl}
				<embed src={pdfBlobUrl} type="application/pdf" class="w-full h-full" />
			{:else}
				<div class="flex-1 flex flex-col items-center justify-center text-gray-300 select-none">
					<svg class="w-20 h-20 mb-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="1" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
					</svg>
					<p class="text-base font-medium text-gray-400">Select a datasheet to preview</p>
					<p class="text-sm text-gray-300 mt-1">PDF will render here</p>
				</div>
			{/if}
		</div>

		<!-- Right: List + Upload -->
		<div class="w-80 flex flex-col bg-white overflow-hidden shrink-0">
			<div class="p-4 border-b border-gray-200">
				<h2 class="text-sm font-semibold text-gray-900 mb-3">Datasheet Library</h2>

				{#if uploadError}
				<div class="mb-2 p-2 bg-red-50 border border-red-200 rounded text-xs text-red-600">{uploadError}</div>
				{/if}

				<label class="flex items-center justify-center gap-2 w-full py-2.5 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition cursor-pointer {uploading ? 'opacity-60 pointer-events-none' : ''}">
					<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12"/>
					</svg>
					{uploading ? 'Uploading…' : 'Upload PDF Datasheet'}
					<input type="file" accept=".pdf" class="hidden" onchange={handleUpload} disabled={uploading} />
				</label>
			</div>

			<div class="flex-1 overflow-y-auto">
				{#if loading}
					{#each [1, 2, 3, 4] as i (i)}
					<div class="p-4 border-b border-gray-100 animate-pulse">
						<div class="h-4 bg-gray-200 rounded w-3/4 mb-2"></div>
						<div class="h-3 bg-gray-100 rounded w-1/2"></div>
					</div>
					{/each}
				{:else if datasheets.length === 0}
					<div class="p-8 text-center text-gray-400">
						<svg class="w-10 h-10 mx-auto mb-3 text-gray-300" fill="currentColor" viewBox="0 0 24 24">
							<path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 2l5 5h-5V4z"/>
						</svg>
						<p class="text-sm font-medium">No datasheets yet</p>
						<p class="text-xs mt-1">Upload a PDF to get started.</p>
					</div>
				{:else}
					{#each datasheets as doc (doc._id)}
					<div
						role="button"
						tabindex="0"
						onclick={() => viewDatasheet(doc._id)}
						onkeydown={(e) => e.key === 'Enter' && viewDatasheet(doc._id)}
						class="w-full text-left p-4 border-b border-gray-100 hover:bg-gray-50 transition-colors cursor-pointer group {selectedId === doc._id ? 'bg-blue-50 border-l-2 border-l-blue-500' : ''}"
					>
						<div class="flex items-start gap-3">
							<svg class="w-5 h-5 mt-0.5 text-red-400 shrink-0" fill="currentColor" viewBox="0 0 24 24">
								<path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 2l5 5h-5V4zM8 14h8v2H8v-2zm0-4h8v2H8v-2z"/>
							</svg>
							<div class="flex-1 min-w-0">
								<p class="text-sm font-medium text-gray-900 truncate {selectedId === doc._id ? 'text-blue-700' : ''}">{doc.name}</p>
								<div class="flex items-center gap-2 mt-1 flex-wrap">
									<span class="text-xs text-gray-400">{doc.size}</span>
									{#if doc.parsed}
									<span class="text-[10px] font-semibold text-green-700 bg-green-50 px-1.5 py-0.5 rounded border border-green-200">Parsed</span>
									{:else}
									<span class="text-[10px] font-semibold text-yellow-700 bg-yellow-50 px-1.5 py-0.5 rounded border border-yellow-200">Processing…</span>
									{/if}
								</div>
								<p class="text-xs text-gray-400 mt-1">{formatDate(doc.uploadedAt)}</p>
							</div>
							<button
								onclick={(e) => deleteDatasheet(doc._id, e)}
								class="opacity-0 group-hover:opacity-100 p-1 text-gray-300 hover:text-red-500 transition-all rounded shrink-0"
								title="Delete"
							>
								<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"/>
								</svg>
							</button>
						</div>

						{#if doc.cogneeConfig}
						<div class="mt-2 ml-8 p-2 bg-gray-50 rounded text-xs text-gray-500 text-left">
							<p class="font-medium text-gray-600 mb-1">Extracted specs</p>
							{#each Object.entries(doc.cogneeConfig).slice(0, 3) as [k, v]}
							<p><span class="text-gray-400">{k}:</span> {JSON.stringify(v)}</p>
							{/each}
						</div>
						{/if}
					</div>
					{/each}
				{/if}
			</div>
		</div>
	</div>
</div>

<style>
	:global(body) { overflow: hidden; }
</style>


<div class="min-h-screen flex flex-col bg-gray-50">
	<NavBar activeLink="datasheets" onLogoClick={goHome} />

	<div class="flex-1 max-w-5xl mx-auto w-full px-6 py-10">
		<div class="flex justify-between items-center mb-8">
			<div>
				<h2 class="text-2xl font-bold text-gray-900">Datasheet Library</h2>
				<p class="text-gray-500 text-sm mt-1">Manage documents that AI relies on for extracting pinouts and schematic rules.</p>
			</div>
			<button
				onclick={startUpload}
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