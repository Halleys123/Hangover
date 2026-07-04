<script lang="ts">
	import { onMount } from "svelte";
	import NavBar from "$lib/components/NavBar.svelte";
	import { authUser } from "$lib/stores/auth";
	import { api } from "$lib/api";
	import { fade, fly } from "svelte/transition";
	import { goto } from "$app/navigation";

	interface Project {
		_id: string;
		name: string;
		description: string;
		components: string[];
		date: string;
		status: "in-progress" | "completed";
	}

	let projects = $state<Project[]>([]);
	let loading = $state(true);
	let creating = $state(false);

	// Modal Form State
	let showModal = $state(false);
	let newName = $state("");
	let newDescription = $state("");
	let newStatus = $state<"in-progress" | "completed">("in-progress");
	let apiError = $state("");

	onMount(async () => {
		if (!$authUser) {
			goto("/login");
			return;
		}
		try {
			projects = await api.get<Project[]>("/projects");
		} catch (err: any) {
			if (err.message?.includes("Unauthorized")) {
				goto("/login");
				return;
			}
			apiError = err.message;
		} finally {
			loading = false;
		}
	});

	function openCreateModal() {
		newName = "";
		newDescription = "";
		newStatus = "in-progress";
		apiError = "";
		showModal = true;
	}

	function closeModal() {
		showModal = false;
	}

	async function createProject(e: SubmitEvent) {
		e.preventDefault();
		if (!newName.trim()) return;
		creating = true;
		apiError = "";
		try {
			const project = await api.post<{ _id: string }>("/projects", {
				name: newName.trim(),
				description: newDescription.trim(),
				status: newStatus,
			});
			goto(`/workspace/${project._id}`);
		} catch (err: any) {
			apiError = err.message;
			creating = false;
		}
	}

	// Edit Details Form State
	let showEditModal = $state(false);
	let editingProject = $state<Project | null>(null);
	let editName = $state("");
	let editDescription = $state("");
	let editStatus = $state<"in-progress" | "completed">("in-progress");

	function openEditModal(project: Project) {
		editingProject = project;
		editName = project.name;
		editDescription = project.description || "";
		editStatus = project.status;
		apiError = "";
		showEditModal = true;
	}

	function closeEditModal() {
		showEditModal = false;
		editingProject = null;
	}

	async function saveProjectDetails(e: SubmitEvent) {
		e.preventDefault();
		if (!editingProject || !editName.trim()) return;
		creating = true;
		apiError = "";
		try {
			const updated = await api.put<Project>(
				`/projects/${editingProject._id}`,
				{
					name: editName.trim(),
					description: editDescription.trim(),
					status: editStatus,
				},
			);
			projects = projects.map((p) =>
				p._id === updated._id ? updated : p,
			);
			closeEditModal();
		} catch (err: any) {
			apiError = err.message;
		} finally {
			creating = false;
		}
	}
</script>

<svelte:head>
	<title>Workspace</title>
</svelte:head>

<!-- WORKSPACE PROJECTS LIST PAGE -->
<div
	class="min-h-screen flex flex-col bg-[#F8FAFC] dark:bg-zinc-950 transition-colors duration-200"
>
	<NavBar
		activeLink="workspace"
		showBack={true}
		onLogoClick={() => goto("/")}
	/>

	<div class="flex-1 max-w-5xl mx-auto w-full px-6 py-10">
		<!-- WORKSPACE HEADER SECTION -->
		<div class="flex justify-between items-center mb-8">
			<div>
				<h2
					class="text-2xl font-bold tracking-tight text-slate-900 dark:text-white"
				>
					{#if $authUser}Welcome back, {$authUser.name.split(
							" ",
						)[0]}{:else}Projects{/if}
				</h2>
				<p class="text-slate-500 dark:text-zinc-400 text-sm mt-1">
					Manage and prototype your hardware architecture designs.
				</p>
			</div>
			<button
				onclick={openCreateModal}
				class="px-4 py-2.5 bg-gradient-to-r from-blue-600 to-indigo-600 text-white text-sm font-medium rounded-xl shadow-sm shadow-indigo-500/20 hover:from-blue-700 hover:to-indigo-700 active:scale-[0.98] transition-all duration-150 flex items-center gap-2"
			>
				<svg
					class="w-4 h-4"
					fill="none"
					stroke="currentColor"
					viewBox="0 0 24 24"
				>
					<path
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-width="2.5"
						d="M12 4v16m8-8H4"
					/>
				</svg>
				New Project
			</button>
		</div>

		{#if apiError && !showModal}
			<div
				class="mb-6 p-4 bg-rose-50 dark:bg-rose-950/40 border border-rose-200 dark:border-rose-900/60 rounded-xl text-sm text-rose-700 dark:text-rose-300 flex items-center gap-3 shadow-sm"
			>
				<svg
					class="w-5 h-5 text-rose-500 shrink-0"
					fill="none"
					stroke="currentColor"
					viewBox="0 0 24 24"
				>
					<path
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-width="2"
						d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
					/>
				</svg>
				<span>{apiError}</span>
			</div>
		{/if}

		<!-- PROJECTS GRID SECTION -->
		{#if loading}
			<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
				{#each [1, 2, 3, 4] as i (i)}
					<div
						class="bg-white dark:bg-zinc-900 border border-slate-200/80 dark:border-zinc-800 rounded-2xl p-6 shadow-sm animate-pulse"
					>
						<div
							class="h-5 bg-slate-200 dark:bg-zinc-800 rounded-lg w-3/4 mb-4"
						></div>
						<div
							class="h-3.5 bg-slate-100 dark:bg-zinc-800 rounded-md w-full mb-2"
						></div>
						<div
							class="h-3.5 bg-slate-100 dark:bg-zinc-800 rounded-md w-2/3 mb-6"
						></div>
						<div class="flex gap-2">
							<div
								class="h-6 w-16 bg-slate-100 dark:bg-zinc-800 rounded-md"
							></div>
							<div
								class="h-6 w-20 bg-slate-100 dark:bg-zinc-800 rounded-md"
							></div>
						</div>
					</div>
				{/each}
			</div>
		{:else if projects.length === 0}
			<div
				class="text-center py-20 bg-white dark:bg-zinc-900 rounded-2xl border border-dashed border-slate-300 dark:border-zinc-800 shadow-sm"
			>
				<div
					class="w-14 h-14 bg-indigo-50 dark:bg-zinc-900/50 text-indigo-600 dark:text-indigo-400 rounded-2xl flex items-center justify-center mx-auto mb-4 shadow-inner"
				>
					<svg
						class="w-7 h-7"
						fill="none"
						stroke="currentColor"
						viewBox="0 0 24 24"
					>
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="1.5"
							d="M19.5 14.25v-2.625a3.375 3.375 0 00-3.375-3.375h-1.5A1.125 1.125 0 0113.5 7.125v-1.5a3.375 3.375 0 00-3.375-3.375H8.25m3.75 9v6m3-3H9m1.5-12H5.625c-.621 0-1.125.504-1.125 1.125v17.25c0 .621.504 1.125 1.125 1.125h12.75c.621 0 1.125-.504 1.125-1.125V11.25a9 9 0 00-9-9z"
						/>
					</svg>
				</div>
				<h3
					class="text-slate-900 dark:text-white font-semibold text-base mb-1"
				>
					No hardware projects yet
				</h3>
				<p
					class="text-slate-500 dark:text-zinc-400 text-sm mb-6 max-w-sm mx-auto"
				>
					Create your first workspace to prototype schematics, inspect
					datasheets, and simulate pinouts.
				</p>
				<button
					onclick={openCreateModal}
					class="px-5 py-2.5 bg-gradient-to-r from-blue-600 to-indigo-600 text-white text-sm font-medium rounded-xl shadow-sm shadow-indigo-500/20 hover:from-blue-700 hover:to-indigo-700 transition-all"
				>
					Create First Project
				</button>
			</div>
		{:else}
			<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
				{#each projects as project (project._id)}
					<div
						role="button"
						tabindex="0"
						onclick={(e) => {
							const target = e.target as HTMLElement;
							if (!target.closest("button")) {
								goto(`/workspace/${project._id}`);
							}
						}}
						onkeydown={(e) => {
							if (e.key === "Enter" || e.key === " ") {
								goto(`/workspace/${project._id}`);
							}
						}}
						class="bg-white dark:bg-zinc-900 border border-slate-200/80 dark:border-zinc-800 rounded-2xl p-6 hover:border-indigo-500/40 dark:hover:border-indigo-500/60 hover:shadow-md hover:shadow-indigo-500/5 transition-all duration-200 text-left group flex flex-col h-full relative overflow-hidden cursor-pointer"
					>
						<div
							class="flex items-start justify-between gap-4 mb-2.5 w-full"
						>
							<div class="flex items-center min-w-0">
								<h3
									class="text-base font-semibold text-slate-900 dark:text-white group-hover:text-indigo-600 dark:group-hover:text-indigo-400 transition-colors truncate"
								>
									{project.name}
								</h3>
								<button
									type="button"
									onclick={(e) => {
										e.stopPropagation();
										openEditModal(project);
									}}
									class="p-1 rounded-lg text-slate-400 hover:text-indigo-600 dark:hover:text-indigo-400 hover:bg-slate-100 dark:hover:bg-zinc-850 transition-colors ml-2 shrink-0"
									title="Edit project details"
								>
									<svg
										class="w-3.5 h-3.5"
										fill="none"
										stroke="currentColor"
										viewBox="0 0 24 24"
									>
										<path
											stroke-linecap="round"
											stroke-linejoin="round"
											stroke-width="2"
											d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z"
										/>
									</svg>
								</button>
							</div>
							<span
								class="shrink-0 px-2.5 py-1 rounded-full text-[11px] font-medium tracking-wide {project.status ===
								'completed'
									? 'bg-emerald-50 dark:bg-emerald-950/60 text-emerald-700 dark:text-emerald-400 border border-emerald-200/60 dark:border-emerald-800/60'
									: 'bg-amber-50 dark:bg-amber-950/60 text-amber-700 dark:text-amber-400 border border-amber-200/60 dark:border-amber-800/60'}"
							>
								{project.status === "completed"
									? "Completed"
									: "In Progress"}
							</span>
						</div>
						<p
							class="text-slate-500 dark:text-zinc-400 text-sm mb-6 line-clamp-2 flex-grow leading-relaxed"
						>
							{project.description ||
								"No project description provided."}
						</p>
						<div
							class="flex items-center justify-between pt-4 border-t border-slate-100 dark:border-zinc-800/60 mt-auto w-full"
						>
							<div class="flex flex-wrap gap-1.5 items-center">
								{#if project.components && project.components.length > 0}
									{#each project.components.slice(0, 3) as comp}
										<span
											class="px-2 py-0.5 bg-slate-100 dark:bg-zinc-800 text-slate-600 dark:text-zinc-300 text-xs font-medium rounded-md"
										>
											{comp}
										</span>
									{/each}
									{#if project.components.length > 3}
										<span
											class="text-xs text-slate-400 font-medium pl-1"
											>+{project.components.length - 3} more</span
										>
									{/if}
								{:else}
									<span
										class="text-xs text-slate-400 dark:text-zinc-500 italic"
										>No components added</span
									>
								{/if}
							</div>
							<span
								class="text-xs font-medium text-indigo-600 dark:text-indigo-400 opacity-0 group-hover:opacity-100 transition-opacity flex items-center gap-1"
							>
								Open
								<svg
									class="w-3.5 h-3.5"
									fill="none"
									stroke="currentColor"
									viewBox="0 0 24 24"
									><path
										stroke-linecap="round"
										stroke-linejoin="round"
										stroke-width="2"
										d="M9 5l7 7-7 7"
									/></svg
								>
							</span>
						</div>
					</div>
				{/each}
			</div>
		{/if}
	</div>
</div>

<!-- NEW PROJECT MODAL POPOVER WITH ANIMATIONS -->
{#if showModal}
	<div
		transition:fade={{ duration: 150 }}
		onclick={(e) => {
			if (e.target === e.currentTarget) closeModal();
		}}
		class="fixed inset-0 z-50 flex items-center justify-center px-4 bg-slate-900/40 dark:bg-zinc-950/60 backdrop-blur-sm"
	>
		<div
			transition:fly={{ y: -20, duration: 250 }}
			class="bg-white dark:bg-zinc-900 rounded-2xl shadow-xl border border-slate-200/80 dark:border-zinc-800 w-full max-w-lg overflow-hidden"
		>
			<div
				class="px-6 py-5 border-b border-slate-100 dark:border-zinc-800 flex items-center justify-between bg-slate-50/50 dark:bg-zinc-900/50"
			>
				<h3
					class="text-lg font-semibold text-slate-900 dark:text-white"
				>
					Create Hardware Project
				</h3>
				<button
					type="button"
					onclick={closeModal}
					class="w-8 h-8 rounded-lg flex items-center justify-center text-slate-400 hover:text-slate-600 dark:hover:text-zinc-200 hover:bg-slate-100 dark:hover:bg-zinc-800 transition-colors"
				>
					<svg
						class="w-5 h-5"
						fill="none"
						stroke="currentColor"
						viewBox="0 0 24 24"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M6 18L18 6M6 6l12 12"
						/></svg
					>
				</button>
			</div>

			<form onsubmit={createProject} class="p-6 space-y-5">
				{#if apiError}
					<div
						class="p-3.5 bg-rose-50 dark:bg-rose-950/40 border border-rose-200 dark:border-rose-900/60 rounded-xl text-sm text-rose-700 dark:text-rose-300"
					>
						{apiError}
					</div>
				{/if}

				<div>
					<label
						class="block text-sm font-medium text-slate-700 dark:text-zinc-300 mb-1.5"
						for="project-name"
					>
						Project Name <span class="text-rose-500">*</span>
					</label>
					<input
						id="project-name"
						type="text"
						bind:value={newName}
						placeholder="e.g., IoT Environmental Monitor v2"
						required
						class="w-full px-3.5 py-2.5 bg-white dark:bg-zinc-800 border border-slate-300 dark:border-zinc-700 rounded-xl text-sm text-slate-900 dark:text-white placeholder:text-slate-400 focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-600 transition-all"
					/>
				</div>

				<div>
					<label
						class="block text-sm font-medium text-slate-700 dark:text-zinc-300 mb-1.5"
						for="project-desc"
					>
						Description <span
							class="text-slate-400 text-xs font-normal"
							>(Optional)</span
						>
					</label>
					<textarea
						id="project-desc"
						bind:value={newDescription}
						rows="3"
						placeholder="Brief overview of the schematic, microcontrollers, or power requirements..."
						class="w-full px-3.5 py-2.5 bg-white dark:bg-zinc-800 border border-slate-300 dark:border-zinc-700 rounded-xl text-sm text-slate-900 dark:text-white placeholder:text-slate-400 focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-600 transition-all resize-none"
					></textarea>
				</div>

				<div>
					<label
						class="block text-sm font-medium text-slate-700 dark:text-zinc-300 mb-2"
						id="status-label">Initial Status</label
					>
					<div
						class="grid grid-cols-2 gap-3"
						aria-labelledby="status-label"
					>
						<button
							type="button"
							onclick={() => (newStatus = "in-progress")}
							class="p-3 border rounded-xl flex items-center gap-3 text-left transition-all {newStatus ===
							'in-progress'
								? 'border-indigo-600 bg-indigo-50/50 dark:bg-zinc-900/40 text-indigo-900 dark:text-indigo-200 ring-1 ring-indigo-600'
								: 'border-slate-200 dark:border-zinc-800 bg-white dark:bg-zinc-800/60 hover:border-slate-300 text-slate-700 dark:text-zinc-300'}"
						>
							<span
								class="w-3 h-3 rounded-full bg-amber-500 shrink-0"
							></span>
							<div>
								<p class="text-sm font-medium">In Progress</p>
								<p
									class="text-xs text-slate-500 dark:text-zinc-400 mt-0.5"
								>
									Actively prototyping
								</p>
							</div>
						</button>
						<button
							type="button"
							onclick={() => (newStatus = "completed")}
							class="p-3 border rounded-xl flex items-center gap-3 text-left transition-all {newStatus ===
							'completed'
								? 'border-emerald-600 bg-emerald-50/50 dark:bg-emerald-950/40 text-emerald-900 dark:text-emerald-200 ring-1 ring-emerald-600'
								: 'border-slate-200 dark:border-zinc-800 bg-white dark:bg-zinc-800/60 hover:border-slate-300 text-slate-700 dark:text-zinc-300'}"
						>
							<span
								class="w-3 h-3 rounded-full bg-emerald-500 shrink-0"
							></span>
							<div>
								<p class="text-sm font-medium">Completed</p>
								<p
									class="text-xs text-slate-500 dark:text-zinc-400 mt-0.5"
								>
									Finalized design
								</p>
							</div>
						</button>
					</div>
				</div>

				<div
					class="pt-3 border-t border-slate-100 dark:border-zinc-800 flex items-center justify-end gap-3"
				>
					<button
						type="button"
						onclick={closeModal}
						class="px-4 py-2.5 rounded-xl text-sm font-medium text-slate-600 dark:text-zinc-400 hover:bg-slate-100 dark:hover:bg-zinc-800 hover:text-slate-900 dark:hover:text-white transition-all"
					>
						Cancel
					</button>
					<button
						type="submit"
						disabled={creating || !newName.trim()}
						class="px-5 py-2.5 bg-gradient-to-r from-blue-600 to-indigo-600 text-white text-sm font-medium rounded-xl shadow-sm shadow-indigo-500/20 hover:from-blue-700 hover:to-indigo-700 active:scale-[0.98] transition-all disabled:opacity-50 disabled:pointer-events-none flex items-center gap-2"
					>
						{#if creating}
							<svg
								class="w-4 h-4 animate-spin"
								fill="none"
								viewBox="0 0 24 24"
							>
								<circle
									class="opacity-25"
									cx="12"
									cy="12"
									r="10"
									stroke="currentColor"
									stroke-width="4"
								></circle>
								<path
									class="opacity-75"
									fill="currentColor"
									d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
								></path>
							</svg>
							<span>Creating…</span>
						{:else}
							<span>Create Project</span>
						{/if}
					</button>
				</div>
			</form>
		</div>
	</div>
{/if}

<!-- EDIT PROJECT DETAILS MODAL POPOVER WITH ANIMATIONS -->
{#if showEditModal && editingProject}
	<div
		transition:fade={{ duration: 150 }}
		onclick={(e) => {
			if (e.target === e.currentTarget) closeEditModal();
		}}
		class="fixed inset-0 z-50 flex items-center justify-center px-4 bg-slate-900/40 dark:bg-zinc-950/60 backdrop-blur-sm"
	>
		<div
			transition:fly={{ y: -20, duration: 250 }}
			class="bg-white dark:bg-zinc-900 rounded-2xl shadow-xl border border-slate-200/80 dark:border-zinc-800 w-full max-w-lg overflow-hidden"
		>
			<div
				class="px-6 py-5 border-b border-slate-100 dark:border-zinc-800 flex items-center justify-between bg-slate-50/50 dark:bg-zinc-900/50"
			>
				<h3
					class="text-lg font-semibold text-slate-900 dark:text-white"
				>
					Edit Project Details
				</h3>
				<button
					type="button"
					onclick={closeEditModal}
					class="w-8 h-8 rounded-lg flex items-center justify-center text-slate-400 hover:text-slate-600 dark:hover:text-zinc-200 hover:bg-slate-100 dark:hover:bg-zinc-800 transition-colors"
				>
					<svg
						class="w-5 h-5"
						fill="none"
						stroke="currentColor"
						viewBox="0 0 24 24"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M6 18L18 6M6 6l12 12"
						/></svg
					>
				</button>
			</div>

			<form onsubmit={saveProjectDetails} class="p-6 space-y-5">
				{#if apiError}
					<div
						class="p-3.5 bg-rose-50 dark:bg-rose-950/40 border border-rose-200 dark:border-rose-900/60 rounded-xl text-sm text-rose-700 dark:text-rose-300"
					>
						{apiError}
					</div>
				{/if}

				<div>
					<label
						class="block text-sm font-medium text-slate-700 dark:text-zinc-300 mb-1.5"
						for="edit-project-name"
					>
						Project Name <span class="text-rose-500">*</span>
					</label>
					<input
						id="edit-project-name"
						type="text"
						bind:value={editName}
						placeholder="e.g., IoT Environmental Monitor v2"
						required
						class="w-full px-3.5 py-2.5 bg-white dark:bg-zinc-800 border border-slate-300 dark:border-zinc-700 rounded-xl text-sm text-slate-900 dark:text-white placeholder:text-slate-400 focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-600 transition-all"
					/>
				</div>

				<div>
					<label
						class="block text-sm font-medium text-slate-700 dark:text-zinc-300 mb-1.5"
						for="edit-project-desc"
					>
						Description <span
							class="text-slate-400 text-xs font-normal"
							>(Optional)</span
						>
					</label>
					<textarea
						id="edit-project-desc"
						bind:value={editDescription}
						rows="3"
						placeholder="Brief overview of the schematic, microcontrollers, or power requirements..."
						class="w-full px-3.5 py-2.5 bg-white dark:bg-zinc-800 border border-slate-300 dark:border-zinc-700 rounded-xl text-sm text-slate-900 dark:text-white placeholder:text-slate-400 focus:outline-none focus:ring-2 focus:ring-indigo-500/20 focus:border-indigo-600 transition-all resize-none"
					></textarea>
				</div>

				<div>
					<label
						class="block text-sm font-medium text-slate-700 dark:text-zinc-300 mb-2"
						id="edit-status-label">Status</label
					>
					<div
						class="grid grid-cols-2 gap-3"
						aria-labelledby="edit-status-label"
					>
						<button
							type="button"
							onclick={() => (editStatus = "in-progress")}
							class="p-3 border rounded-xl flex items-center gap-3 text-left transition-all {editStatus ===
							'in-progress'
								? 'border-indigo-600 bg-indigo-50/50 dark:bg-zinc-900/40 text-indigo-900 dark:text-indigo-200 ring-1 ring-indigo-600'
								: 'border-slate-200 dark:border-zinc-800 bg-white dark:bg-zinc-800/60 hover:border-slate-300 text-slate-700 dark:text-zinc-300'}"
						>
							<span
								class="w-3 h-3 rounded-full bg-amber-500 shrink-0"
							></span>
							<div>
								<p class="text-sm font-medium">In Progress</p>
								<p
									class="text-xs text-slate-500 dark:text-zinc-400 mt-0.5"
								>
									Actively prototyping
								</p>
							</div>
						</button>
						<button
							type="button"
							onclick={() => (editStatus = "completed")}
							class="p-3 border rounded-xl flex items-center gap-3 text-left transition-all {editStatus ===
							'completed'
								? 'border-emerald-600 bg-emerald-50/50 dark:bg-emerald-950/40 text-emerald-900 dark:text-emerald-200 ring-1 ring-emerald-600'
								: 'border-slate-200 dark:border-zinc-800 bg-white dark:bg-zinc-800/60 hover:border-slate-300 text-slate-700 dark:text-zinc-300'}"
						>
							<span
								class="w-3 h-3 rounded-full bg-emerald-500 shrink-0"
							></span>
							<div>
								<p class="text-sm font-medium">Completed</p>
								<p
									class="text-xs text-slate-500 dark:text-zinc-400 mt-0.5"
								>
									Finalized design
								</p>
							</div>
						</button>
					</div>
				</div>

				<div
					class="pt-3 border-t border-slate-100 dark:border-zinc-800 flex items-center justify-end gap-3"
				>
					<button
						type="button"
						onclick={closeEditModal}
						class="px-4 py-2.5 rounded-xl text-sm font-medium text-slate-600 dark:text-zinc-400 hover:bg-slate-100 dark:hover:bg-zinc-800 hover:text-slate-900 dark:hover:text-white transition-all"
					>
						Cancel
					</button>
					<button
						type="submit"
						disabled={creating || !editName.trim()}
						class="px-5 py-2.5 bg-gradient-to-r from-blue-600 to-indigo-600 text-white text-sm font-medium rounded-xl shadow-sm shadow-indigo-500/20 hover:from-blue-700 hover:to-indigo-700 active:scale-[0.98] transition-all disabled:opacity-50 disabled:pointer-events-none flex items-center gap-2"
					>
						{#if creating}
							<svg
								class="w-4 h-4 animate-spin"
								fill="none"
								viewBox="0 0 24 24"
							>
								<circle
									class="opacity-25"
									cx="12"
									cy="12"
									r="10"
									stroke="currentColor"
									stroke-width="4"
								></circle>
								<path
									class="opacity-75"
									fill="currentColor"
									d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
								></path>
							</svg>
							<span>Saving…</span>
						{:else}
							<span>Save Changes</span>
						{/if}
					</button>
				</div>
			</form>
		</div>
	</div>
{/if}
