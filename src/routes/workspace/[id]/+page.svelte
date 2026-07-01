<script lang="ts">
    import { page } from "$app/stores";
    import { writable, get } from "svelte/store";
    import { onMount, onDestroy } from "svelte";
    import {
        SvelteFlow,
        Background,
        Controls,
        SvelteFlowProvider,
        useSvelteFlow,
        type Node,
        type Edge,
        type NodeTypes,
    } from "@xyflow/svelte";
    import "@xyflow/svelte/dist/style.css";
    import HardwareNode from "$lib/components/HardwareNode.svelte";
    import CanvasInner from "$lib/components/CanvasInner.svelte";
    import VoltagePopover from "$lib/components/workspace/VoltagePopover.svelte";
    import AIChatPanel from "$lib/components/workspace/AIChatPanel.svelte";
    import RightLibraryPanel from "$lib/components/workspace/RightLibraryPanel.svelte";
    import NavBar from "$lib/components/NavBar.svelte";
    import { authUser } from "$lib/stores/auth";
    import { api } from "$lib/api";

    const nodeTypes: NodeTypes = {
        hardware: HardwareNode,
    };

    let svelteFlowInstance = $state<any>(null);

    interface Component {
        id: string;
        name: string;
        type: string;
        uploaded: boolean;
        diagram?: {
            theme: "orange" | "blue";
            pins: {
                left: Array<{ id: string; label: string; color: string }>;
                right: Array<{ id: string; label: string; color: string }>;
            };
        };
    }

    interface Message {
        role: "user" | "assistant";
        text: string;
        timestamp: Date;
    }

    let workspaceId = $state($page.params.id);
    let projectName = $state('');
    let projectLoading = $state(true);
    let saveStatus = $state<'idle' | 'saving' | 'saved'>('idle');
    let saveTimer: ReturnType<typeof setTimeout> | null = null;
    let isLoaded = false;
    let rightTab = $state<"components" | "datasheets">("components");
    let isPopoverOpen = $state(false);

    let leftWidth = $state(400);
    let rightWidth = $state(320);
    let leftCollapsed = $state(false);
    let rightCollapsed = $state(false);
    let draggingSide = $state<"left" | "right" | null>(null);

    const MIN_PANEL_WIDTH = 260;
    const MAX_LEFT_WIDTH = 600;
    const MAX_RIGHT_WIDTH = 500;
    const COLLAPSE_THRESHOLD = 150;

    function startDrag(side: "left" | "right") {
        draggingSide = side;
    }

    function handlePointerMove(e: PointerEvent) {
        if (!draggingSide) return;
        if (draggingSide === "left") {
            const w = e.clientX;
            if (w < COLLAPSE_THRESHOLD) {
                leftCollapsed = true;
            } else {
                leftCollapsed = false;
                leftWidth = Math.min(
                    Math.max(w, MIN_PANEL_WIDTH),
                    MAX_LEFT_WIDTH,
                );
            }
        } else {
            const w = window.innerWidth - e.clientX;
            if (w < COLLAPSE_THRESHOLD) {
                rightCollapsed = true;
            } else {
                rightCollapsed = false;
                rightWidth = Math.min(
                    Math.max(w, MIN_PANEL_WIDTH),
                    MAX_RIGHT_WIDTH,
                );
            }
        }
    }

    function handlePointerUp() {
        draggingSide = null;
    }

    import { beforeNavigate } from "$app/navigation";

    let nodes = writable<Node[]>([]);
    let edges = writable<Edge[]>([]);
    let hasUnsavedChanges = $state(false);

    beforeNavigate((nav) => {
        if (hasUnsavedChanges || saveStatus === 'saving') {
            if (!confirm('You have unsaved changes in your canvas. Are you sure you want to leave without saving?')) {
                nav.cancel();
            }
        }
    });

    function confirmLeave(): boolean {
        if (hasUnsavedChanges || saveStatus === 'saving') {
            return confirm('You have unsaved changes in your canvas. Are you sure you want to leave without saving?');
        }
        return true;
    }

    function goBack() {
        if (confirmLeave()) {
            window.location.href = "/workspace";
        }
    }

    function openDatasheetIngestion() {
        if (confirmLeave()) {
            window.location.href = "/datasheets";
        }
    }

    async function manualSaveCanvas() {
        if (saveTimer) clearTimeout(saveTimer);
        saveStatus = 'saving';
        try {
            await api.put(`/projects/${workspaceId}/canvas`, {
                nodes: get(nodes),
                edges: get(edges)
            });
            saveStatus = 'saved';
            hasUnsavedChanges = false;
            setTimeout(() => {
                if (saveStatus === 'saved') saveStatus = 'idle';
            }, 2000);
        } catch {
            saveStatus = 'idle';
        }
    }

    function scheduleCanvasSave() {
        if (saveTimer) clearTimeout(saveTimer);
        saveStatus = 'saving';
        hasUnsavedChanges = true;
        saveTimer = setTimeout(async () => {
            try {
                await api.put(`/projects/${workspaceId}/canvas`, {
                    nodes: get(nodes),
                    edges: get(edges)
                });
                saveStatus = 'saved';
                hasUnsavedChanges = false;
                setTimeout(() => {
                    if (saveStatus === 'saved') saveStatus = 'idle';
                }, 2000);
            } catch {
                saveStatus = 'idle';
            }
        }, 2000);
    }

    let unsubNodes: (() => void) | null = null;
    let unsubEdges: (() => void) | null = null;

    onMount(async () => {
        if (!$authUser) { window.location.href = '/login'; return; }
        try {
            const project = await api.get<any>(`/projects/${workspaceId}`);
            projectName = project.name;
            nodes.set(project.canvas?.nodes ?? []);
            edges.set(project.canvas?.edges ?? []);
            
            projectLoading = false;
            isLoaded = true;
            let initialLoadDone = false;
            unsubNodes = nodes.subscribe(() => { if (initialLoadDone && isLoaded) scheduleCanvasSave(); });
            unsubEdges = edges.subscribe(() => { if (initialLoadDone && isLoaded) scheduleCanvasSave(); });
            setTimeout(() => { initialLoadDone = true; }, 600);
        } catch (err: any) {
            if (err.message?.includes('Unauthorized')) { 
                window.location.href = '/login'; 
                return; 
            }
            window.location.href = '/workspace';
        }
    });

    onDestroy(() => {
        unsubNodes?.();
        unsubEdges?.();
        if (saveTimer) clearTimeout(saveTimer);
    });

    let isCanvasLocked = $state(false);
    let isFullscreen = $state(false);

    function toggleFullscreen() {
        isFullscreen = !isFullscreen;
        if (isFullscreen) {
            leftCollapsed = true;
            rightCollapsed = true;
        } else {
            leftCollapsed = false;
            rightCollapsed = false;
        }
    }

    function handleDropData(data: any, position: { x: number; y: number }) {
        const defaultLeft = [
            { id: "1", label: "PIN1", color: "gray" },
            { id: "2", label: "PIN2", color: "gray" },
        ];
        const defaultRight = [
            { id: "3", label: "PIN3", color: "gray" },
            { id: "4", label: "PIN4", color: "gray" },
        ];
        const newNode = {
            id: Math.random().toString(),
            type: data.type || 'hardware',
            position,
            data: {
                label: data.name || 'Component',
                theme: data.diagram?.theme || "blue",
                pins: {
                    left: Array.isArray(data.diagram?.pins?.left) ? data.diagram.pins.left : defaultLeft,
                    right: Array.isArray(data.diagram?.pins?.right) ? data.diagram.pins.right : defaultRight,
                },
            },
        };

        nodes.update((nds) => [...nds, newNode]);
    }

    function handleDragOver(event: DragEvent) {
        event.preventDefault();
        if (event.dataTransfer) {
            event.dataTransfer.dropEffect = "move";
        }
    }

    function handleBeforeUnload(e: BeforeUnloadEvent) {
        if (hasUnsavedChanges || saveStatus === 'saving') {
            e.preventDefault();
            e.returnValue = '';
        }
    }
</script>

<svelte:window
    on:pointermove={handlePointerMove}
    on:pointerup={handlePointerUp}
    onbeforeunload={handleBeforeUnload}
/>

<!-- MAIN WORKSPACE CANVAS PAGE -->
<div class="h-screen flex flex-col bg-white dark:bg-slate-950 overflow-hidden text-slate-900 dark:text-slate-100 font-sans transition-colors duration-200">
    {#if !isFullscreen}
        <NavBar
            activeLink="workspace"
            onLogoClick={goBack}
            actionLabel={saveStatus === 'saving' ? 'Saving…' : saveStatus === 'saved' ? 'Saved ✓' : 'Save Project'}
            onAction={manualSaveCanvas}
            wide
            compact
        />
    {/if}

    <div class="flex-1 flex w-full min-h-0">
        {#if leftCollapsed}
            <button
                onclick={() => (leftCollapsed = false)}
                class="w-6 shrink-0 flex items-center justify-center border-r border-gray-200 bg-white hover:bg-gray-50 group"
                title="Show AI Copilot panel"
            >
                <svg
                    class="w-4 h-4 text-gray-400 group-hover:text-blue-600"
                    fill="none"
                    stroke="currentColor"
                    viewBox="0 0 24 24"
                >
                    <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="2"
                        d="M9 5l7 7-7 7"
                    />
                </svg>
            </button>
        {:else}
            <div
                class="flex flex-col border-r border-slate-200 dark:border-slate-800 bg-[#FAFAFA] dark:bg-slate-900 shrink-0 min-h-0 transition-colors"
                style="width: {leftWidth}px;"
            >
                <div
                    class="px-4 py-3 border-b border-slate-200 dark:border-slate-800 flex items-center gap-2 bg-white dark:bg-slate-900 text-slate-900 dark:text-white transition-colors"
                >
                    <svg
                        class="w-5 h-5 text-blue-600 dark:text-blue-400"
                        fill="currentColor"
                        viewBox="0 0 20 20"
                    >
                        <path
                            d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"
                        />
                    </svg>
                    <h2 class="font-semibold text-sm">AI Copilot</h2>
                </div>
                <AIChatPanel bind:isPopoverOpen />
            </div>
            <div
                class="w-1 shrink-0 cursor-col-resize bg-slate-200 dark:bg-slate-800 hover:bg-blue-400 active:bg-blue-500 transition-colors"
                onpointerdown={() => startDrag("left")}
            ></div>
        {/if}

        <div
            class="flex-1 bg-[#FAFAFA] dark:bg-slate-950 relative overflow-hidden flex flex-col z-0 transition-colors"
        >
            <SvelteFlowProvider>
                <CanvasInner
                    bind:nodes={$nodes}
                    bind:edges={$edges}
                    {nodeTypes}
                    {isCanvasLocked}
                    {isFullscreen}
                    onToggleFullscreen={toggleFullscreen}
                    onToggleLock={() => (isCanvasLocked = !isCanvasLocked)}
                    onDropData={handleDropData}
                />
            </SvelteFlowProvider>
            <VoltagePopover bind:isOpen={isPopoverOpen} />
        </div>

        {#if rightCollapsed}
            <button
                onclick={() => (rightCollapsed = false)}
                class="w-6 shrink-0 flex items-center justify-center border-l border-slate-200 dark:border-slate-800 bg-white dark:bg-slate-900 hover:bg-slate-50 dark:hover:bg-slate-800 group transition-colors"
                title="Show Components panel"
            >
                <svg
                    class="w-4 h-4 text-slate-400 dark:text-slate-500 group-hover:text-blue-600 dark:group-hover:text-blue-400"
                    fill="none"
                    stroke="currentColor"
                    viewBox="0 0 24 24"
                >
                    <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="2"
                        d="M15 19l-7-7 7-7"
                    />
                </svg>
            </button>
        {:else}
            <div
                class="w-1 shrink-0 cursor-col-resize bg-slate-200 dark:bg-slate-800 hover:bg-blue-400 active:bg-blue-500 transition-colors"
                onpointerdown={() => startDrag("right")}
            ></div>
            <div
                class="bg-white dark:bg-slate-900 border-l border-slate-200 dark:border-slate-800 flex flex-col z-20 shrink-0 min-h-0 text-slate-900 dark:text-white transition-colors"
                style="width: {rightWidth}px;"
            >
                <RightLibraryPanel />
            </div>
        {/if}
    </div>
</div>

<style>
    :global(body) {
        overflow: hidden;
    }
</style>
