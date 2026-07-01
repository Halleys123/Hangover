<script lang="ts">
	import { fade, scale } from 'svelte/transition';
	import { api } from '$lib/api';

	interface Datasheet {
		_id: string;
		name: string;
		size: string;
		parsed: boolean;
		status?: string;
		verificationStatus?: 'unverified' | 'accepted' | 'rejected';
		userNotes?: string;
		cogneeConfig: Record<string, unknown> | null;
	}

	interface Props {
		datasheet: Datasheet | null;
		onClose: () => void;
		onUpdated: (updated: Datasheet) => void;
	}

	let { datasheet, onClose, onUpdated }: Props = $props();

	let isImproving = $state(false);
	let feedbackText = $state('');
	let editedSpecsJson = $state('');
	let jsonError = $state('');
	let loadingAction = $state(false);

	$effect(() => {
		if (datasheet?.cogneeConfig) {
			editedSpecsJson = JSON.stringify(datasheet.cogneeConfig, null, 2);
		}
		feedbackText = datasheet?.userNotes || '';
	});

	async function handleAccept() {
		if (!datasheet) return;
		loadingAction = true;
		try {
			const updated = await api.patch<Datasheet>(`/datasheets/${datasheet._id}/review`, {
				action: 'accept'
			});
			onUpdated(updated);
			onClose();
		} catch (err: any) {
			alert(err.message || 'Failed to accept specifications');
		} finally {
			loadingAction = false;
		}
	}

	async function handleImprove() {
		if (!datasheet) return;
		if (!isImproving) {
			isImproving = true;
			return;
		}

		jsonError = '';
		let parsedSpecs = null;
		try {
			parsedSpecs = JSON.parse(editedSpecsJson);
		} catch {
			jsonError = 'Invalid JSON syntax. Please check your formatting.';
			return;
		}

		loadingAction = true;
		try {
			const updated = await api.patch<Datasheet>(`/datasheets/${datasheet._id}/review`, {
				action: 'improve',
				feedback: feedbackText.trim(),
				updatedSpecs: parsedSpecs
			});
			onUpdated(updated);
			isImproving = false;
			onClose();
		} catch (err: any) {
			alert(err.message || 'Failed to save improvements');
		} finally {
			loadingAction = false;
		}
	}

	async function handleForget() {
		if (!datasheet) return;
		if (!confirm('Are you sure you want to forget and discard all extracted knowledge for this datasheet?')) return;
		loadingAction = true;
		try {
			const updated = await api.patch<Datasheet>(`/datasheets/${datasheet._id}/review`, {
				action: 'forget'
			});
			onUpdated(updated);
			onClose();
		} catch (err: any) {
			alert(err.message || 'Failed to discard specifications');
		} finally {
			loadingAction = false;
		}
	}
</script>

{#if datasheet}
<div transition:fade={{ duration: 150 }} class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/60 dark:bg-black/70 backdrop-blur-sm">
	<div transition:scale={{ duration: 180, start: 0.95 }} class="w-full max-w-2xl bg-white dark:bg-zinc-900 border border-slate-200 dark:border-zinc-800 rounded-3xl shadow-2xl overflow-hidden flex flex-col max-h-[85vh]">
		
		<!-- Modal Header -->
		<div class="px-6 py-5 border-b border-slate-200 dark:border-zinc-800 flex items-center justify-between bg-slate-50/80 dark:bg-zinc-900/80">
			<div class="flex items-center gap-3">
				<div class="w-10 h-10 rounded-2xl bg-gradient-to-tr from-blue-600 to-indigo-600 flex items-center justify-center text-white shadow-md shadow-indigo-500/20">
					<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m5.618-4.016A11.955 11.955 0 0112 2.944a11.955 11.955 0 01-8.618 3.04A12.02 12.02 0 003 9c0 5.591 3.824 10.29 9 11.622 5.176-1.332 9-6.03 9-11.622 0-1.042-.133-2.052-.382-3.016z"/>
					</svg>
				</div>
				<div>
					<h3 class="text-base font-bold text-slate-900 dark:text-white truncate max-w-md">{datasheet.name}</h3>
					<div class="flex items-center gap-2 mt-0.5">
						<span class="text-xs text-slate-500 dark:text-zinc-400">Cognee Knowledge Graph</span>
						{#if datasheet.verificationStatus === 'accepted'}
							<span class="text-[10px] font-bold text-emerald-700 dark:text-emerald-400 bg-emerald-100 dark:bg-emerald-950/60 px-2 py-0.5 rounded-full border border-emerald-300 dark:border-emerald-800">Verified &amp; Active</span>
						{:else if datasheet.verificationStatus === 'rejected'}
							<span class="text-[10px] font-bold text-rose-700 dark:text-rose-400 bg-rose-100 dark:bg-rose-950/60 px-2 py-0.5 rounded-full border border-rose-300 dark:border-rose-800">Discarded</span>
						{:else}
							<span class="text-[10px] font-bold text-amber-700 dark:text-amber-400 bg-amber-100 dark:bg-amber-950/60 px-2 py-0.5 rounded-full border border-amber-300 dark:border-amber-800">Pending Review</span>
						{/if}
					</div>
				</div>
			</div>
			<button onclick={onClose} class="w-8 h-8 rounded-full flex items-center justify-center text-slate-400 hover:text-slate-600 dark:hover:text-white hover:bg-slate-200 dark:hover:bg-zinc-800 transition">
				<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/></svg>
			</button>
		</div>

		<!-- Modal Body -->
		<div class="p-6 overflow-y-auto flex-1 space-y-6">
			<div class="bg-blue-50/60 dark:bg-indigo-950/30 border border-blue-200/60 dark:border-indigo-900/50 rounded-2xl p-4 text-xs text-blue-900 dark:text-indigo-200 leading-relaxed">
				<strong>Why verify?</strong> Verified hardware parameters are indexed into your Cognee Knowledge Graph, allowing the AI Copilot to accurately inspect pin compatibility, logic voltage thresholds, and pull-up requirements during schematic prototyping.
			</div>

			{#if !isImproving}
				<!-- View Mode: Rendered Specs -->
				<div>
					<h4 class="text-xs font-bold text-slate-400 dark:text-zinc-500 uppercase tracking-wider mb-3">Extracted Electrical Parameters</h4>
					{#if !datasheet.cogneeConfig || Object.keys(datasheet.cogneeConfig).length === 0}
						<div class="p-8 text-center text-slate-400 dark:text-zinc-500 bg-slate-50 dark:bg-zinc-800/50 rounded-2xl border border-dashed border-slate-200 dark:border-zinc-800">
							No extracted details available. Click 'Improve / Refine' to manually input specifications.
						</div>
					{:else}
						<div class="grid grid-cols-1 sm:grid-cols-2 gap-3">
							{#each Object.entries(datasheet.cogneeConfig) as [key, value]}
								<div class="p-3.5 bg-slate-50 dark:bg-zinc-800/80 border border-slate-200/80 dark:border-zinc-700/80 rounded-2xl">
									<span class="block text-[11px] font-semibold text-slate-500 dark:text-zinc-400 mb-1">{key}</span>
									<span class="text-sm font-bold text-slate-900 dark:text-white break-words">{typeof value === 'string' ? value : JSON.stringify(value)}</span>
								</div>
							{/each}
						</div>
					{/if}
				</div>
			{:else}
				<!-- Improve Mode: Edit JSON & Notes -->
				<div class="space-y-4 animate-fadeIn">
					<div class="flex items-center justify-between">
						<h4 class="text-xs font-bold text-slate-900 dark:text-white uppercase tracking-wider">Refine Specifications (JSON Editor)</h4>
						<button onclick={() => (isImproving = false)} class="text-xs text-blue-600 dark:text-blue-400 font-medium hover:underline">Cancel Editing</button>
					</div>

					{#if jsonError}
						<div class="p-3 bg-rose-50 dark:bg-rose-950/40 border border-rose-200 dark:border-rose-900/60 rounded-xl text-xs text-rose-600 dark:text-rose-400 font-medium">{jsonError}</div>
					{/if}

					<div>
						<label class="block text-xs font-semibold text-slate-700 dark:text-zinc-300 mb-1.5" for="specs-json">Parameter Map (JSON)</label>
						<textarea id="specs-json" bind:value={editedSpecsJson} rows="8" class="w-full font-mono text-xs p-3.5 rounded-2xl bg-slate-50 dark:bg-zinc-950 border border-slate-300 dark:border-zinc-800 text-slate-900 dark:text-zinc-100 focus:outline-none focus:ring-2 focus:ring-indigo-500"></textarea>
					</div>

					<div>
						<label class="block text-xs font-semibold text-slate-700 dark:text-zinc-300 mb-1.5" for="feedback-notes">Engineering Notes / Correction Prompt</label>
						<input id="feedback-notes" type="text" bind:value={feedbackText} placeholder="e.g. Pin 12 is MISO, logic level is 3.3V max" class="w-full text-xs p-3 rounded-xl bg-slate-50 dark:bg-zinc-950 border border-slate-300 dark:border-zinc-800 text-slate-900 dark:text-zinc-100 focus:outline-none focus:ring-2 focus:ring-indigo-500" />
					</div>
				</div>
			{/if}
		</div>

		<!-- Modal Footer Actions -->
		<div class="px-6 py-4 border-t border-slate-200 dark:border-zinc-800 bg-slate-50/80 dark:bg-zinc-900/80 flex items-center justify-between flex-wrap gap-3">
			<button
				onclick={handleForget}
				disabled={loadingAction}
				class="px-4 py-2 bg-rose-50 dark:bg-rose-950/40 hover:bg-rose-100 dark:hover:bg-rose-900/60 text-rose-600 dark:text-rose-400 text-xs font-bold rounded-xl border border-rose-200 dark:border-rose-900/60 transition flex items-center gap-1.5 disabled:opacity-50"
			>
				<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"/></svg>
				Forget / Discard
			</button>

			<div class="flex items-center gap-3">
				<button
					onclick={handleImprove}
					disabled={loadingAction}
					class="px-4 py-2 bg-slate-200 dark:bg-zinc-800 hover:bg-slate-300 dark:hover:bg-zinc-700 text-slate-800 dark:text-zinc-200 text-xs font-bold rounded-xl transition flex items-center gap-1.5 disabled:opacity-50"
				>
					<svg class="w-4 h-4 text-blue-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z"/></svg>
					{isImproving ? 'Save Improvements' : 'Improve / Edit'}
				</button>

				{#if !isImproving}
					<button
						onclick={handleAccept}
						disabled={loadingAction || datasheet.verificationStatus === 'accepted'}
						class="px-5 py-2 bg-gradient-to-r from-blue-600 to-indigo-600 hover:from-blue-700 hover:to-indigo-700 text-white text-xs font-bold rounded-xl shadow-md shadow-indigo-500/20 transition flex items-center gap-1.5 disabled:opacity-50"
					>
						<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M5 13l4 4L19 7"/></svg>
						{datasheet.verificationStatus === 'accepted' ? 'Accepted' : 'Accept Knowledge'}
					</button>
				{/if}
			</div>
		</div>

	</div>
</div>
{/if}
