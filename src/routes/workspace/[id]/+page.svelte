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
    let projectDatasheets = $state<any[]>([]);
    let projectComponents = $state<string[]>([]);
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

    import { beforeNavigate, goto } from "$app/navigation";

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
            goto("/workspace");
        }
    }

    function openDatasheetIngestion() {
        if (confirmLeave()) {
            goto("/datasheets");
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
        if (!$authUser) { goto('/login'); return; }
        try {
            const project = await api.get<any>(`/projects/${workspaceId}`);
            projectName = project.name;
            projectDatasheets = project.datasheets || [];
            projectComponents = project.components || [];
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
                goto('/login'); 
                return; 
            }
            goto('/workspace');
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

    /**
     * Semantic Drop Handler:
     * Evaluates dropped component attributes when dragging from the hardware library onto the canvas.
     * Enforces strict physical wire accuracy for 2-lead cooling hardware (Peltier TEC, DC Fans, Heatsinks)
     * by assigning exactly (+) RED / VCC and (-) BLACK / GND leads without bogus SIG/DATA pins.
     */
    function handleDropData(data: any, position: { x: number; y: number }) {
        const compName = (data.name || '').toLowerCase();
        const compDesc = (data.description || '').toLowerCase();
        const compClass = ((data.cogneeConfig && data.cogneeConfig['Component Classification']) || '').toString().toLowerCase();
        const is2WireCoolingUnit = compName.includes('tec') || compName.includes('peltier') || compName.includes('fhs') || compName.includes('cooler') || compName.includes('fan') || compClass.includes('cooler') || compClass.includes('fan');

        const defaultLeft = [
            { id: "vcc", label: "VCC", color: "red" },
            { id: "gnd", label: "GND", color: 'gray' },
        ];
        const defaultRight = [
            { id: "sig", label: "SIG / DATA", color: "blue" },
        ];

        // If dropping a 2-wire cooler/fan, strictly restrict to 2 power leads
        const leftPins = is2WireCoolingUnit ? [
            { id: 'vcc', label: '(+) RED / VCC', color: 'red' },
            { id: 'gnd', label: '(-) BLACK / GND', color: 'gray' }
        ] : (Array.isArray(data.diagram?.pins?.left) ? data.diagram.pins.left : defaultLeft);

        const rightPins = is2WireCoolingUnit ? [] : (Array.isArray(data.diagram?.pins?.right) ? data.diagram.pins.right : defaultRight);

        const newNode = {
            id: Math.random().toString(),
            type: data.type || 'hardware',
            position,
            data: {
                label: data.name || 'Component',
                theme: data.diagram?.theme || "blue",
                pins: {
                    left: leftPins,
                    right: rightPins,
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

    function handleCircuitGenerated(newNodes: any[], newEdges: any[]) {
        nodes.set(newNodes);
        edges.set(newEdges);
        scheduleCanvasSave();
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
<div class="h-screen flex flex-col bg-white dark:bg-zinc-950 overflow-hidden text-slate-900 dark:text-zinc-100 font-sans transition-colors duration-200">
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
                class="flex flex-col border-r border-slate-200 dark:border-zinc-800 bg-[#FAFAFA] dark:bg-zinc-900 shrink-0 min-h-0 transition-colors"
                style="width: {leftWidth}px;"
            >
                <div
                    class="px-4 py-3 border-b border-slate-200 dark:border-zinc-800 flex items-center gap-2 bg-white dark:bg-zinc-900 text-slate-900 dark:text-white transition-colors shrink-0"
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
                <div class="flex-1 flex flex-col min-h-0 w-full overflow-hidden">
                    <AIChatPanel bind:isPopoverOpen {workspaceId} onCircuitGenerated={handleCircuitGenerated} />
                </div>
            </div>
            <div
                class="w-1 shrink-0 cursor-col-resize bg-slate-200 dark:bg-zinc-800 hover:bg-blue-400 active:bg-blue-500 transition-colors"
                onpointerdown={() => startDrag("left")}
            ></div>
        {/if}

        <div
            class="flex-1 bg-[#FAFAFA] dark:bg-zinc-950 relative overflow-hidden flex flex-col z-0 transition-colors"
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
                class="w-6 shrink-0 flex items-center justify-center border-l border-slate-200 dark:border-zinc-800 bg-white dark:bg-zinc-900 hover:bg-slate-50 dark:hover:bg-zinc-800 group transition-colors"
                title="Show Components panel"
            >
                <svg
                    class="w-4 h-4 text-slate-400 dark:text-zinc-500 group-hover:text-blue-600 dark:group-hover:text-blue-400"
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
                class="w-1 shrink-0 cursor-col-resize bg-slate-200 dark:bg-zinc-800 hover:bg-blue-400 active:bg-blue-500 transition-colors"
                onpointerdown={() => startDrag("right")}
            ></div>
            <div
                class="bg-white dark:bg-zinc-900 border-l border-slate-200 dark:border-zinc-800 flex flex-col z-20 shrink-0 min-h-0 text-slate-900 dark:text-white transition-colors"
                style="width: {rightWidth}px;"
            >
                <RightLibraryPanel {workspaceId} bind:projectDatasheets bind:projectComponents />
            </div>
        {/if}
    </div>
</div>

<style>
    :global(body) {
        overflow: hidden;
    }
</style>
