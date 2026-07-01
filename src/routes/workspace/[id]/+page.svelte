<script lang="ts">
	import { page } from '$app/stores';
	import { writable } from 'svelte/store';
	import {
		SvelteFlow,
		Background,
		Controls,
		SvelteFlowProvider,
		useSvelteFlow,
		type Node,
		type Edge,
		type NodeTypes
	} from '@xyflow/svelte';
	import '@xyflow/svelte/dist/style.css';
	import HardwareNode from '$lib/components/HardwareNode.svelte';
	import CanvasInner from '$lib/components/CanvasInner.svelte';
	import VoltagePopover from '$lib/components/workspace/VoltagePopover.svelte';
	import AIChatPanel from '$lib/components/workspace/AIChatPanel.svelte';
	import RightLibraryPanel from '$lib/components/workspace/RightLibraryPanel.svelte';

	const nodeTypes: NodeTypes = {
		hardware: HardwareNode
	};

	let svelteFlowInstance = $state<any>(null);

	interface Component {
		id: string;
		name: string;
		type: string;
		uploaded: boolean;
		diagram?: {
			theme: 'orange' | 'blue';
			pins: {
				left: Array<{id: string, label: string, color: string}>;
				right: Array<{id: string, label: string, color: string}>;
			}
		}
	}

	interface Message {
		role: 'user' | 'assistant';
		text: string;
		timestamp: Date;
	}

	let workspaceId = $state($page.params.id);
	let rightTab = $state<'components' | 'datasheets'>('components');
	let isPopoverOpen = $state(false);

	let leftWidth = $state(400);
	let rightWidth = $state(320);
	let leftCollapsed = $state(false);
	let rightCollapsed = $state(false);
	let draggingSide = $state<'left' | 'right' | null>(null);

	const MIN_PANEL_WIDTH = 260;
	const MAX_LEFT_WIDTH = 600;
	const MAX_RIGHT_WIDTH = 500;
	const COLLAPSE_THRESHOLD = 150;

	function startDrag(side: 'left' | 'right') {
		draggingSide = side;
	}

	function handlePointerMove(e: PointerEvent) {
		if (!draggingSide) return;
		if (draggingSide === 'left') {
			const w = e.clientX;
			if (w < COLLAPSE_THRESHOLD) {
				leftCollapsed = true;
			} else {
				leftCollapsed = false;
				leftWidth = Math.min(Math.max(w, MIN_PANEL_WIDTH), MAX_LEFT_WIDTH);
			}
		} else {
			const w = window.innerWidth - e.clientX;
			if (w < COLLAPSE_THRESHOLD) {
				rightCollapsed = true;
			} else {
				rightCollapsed = false;
				rightWidth = Math.min(Math.max(w, MIN_PANEL_WIDTH), MAX_RIGHT_WIDTH);
			}
		}
	}

	function handlePointerUp() {
		draggingSide = null;
	}

	let nodes = writable<Node[]>([
		{
			id: '1',
			type: 'hardware',
			position: { x: 100, y: 100 },
			data: {
				label: 'TMC2209',
				theme: 'orange',
				pins: {
					left: [
						{ id: 'vm', label: 'VM', color: 'red' },
						{ id: 'gnd_l', label: 'GND', color: 'gray' },
						{ id: 'step', label: 'STEP', color: 'blue' },
						{ id: 'dir', label: 'DIR', color: 'blue' },
						{ id: 'en', label: 'EN', color: 'blue' }
					],
					right: [
						{ id: 'vio', label: 'VIO', color: 'red' },
						{ id: 'gnd_r', label: 'GND', color: 'gray' },
						{ id: 'uart', label: 'UART', color: 'blue' },
						{ id: 'a1', label: 'A1', color: 'green' },
						{ id: 'a2', label: 'A2', color: 'green' }
					]
				}
			}
		},
		{
			id: '2',
			type: 'hardware',
			position: { x: 400, y: 50 },
			data: {
				label: 'RAMPS-1.4',
				theme: 'blue',
				pins: {
					left: [
						{ id: '12v', label: '12V IN', color: 'red' },
						{ id: 'gnd1', label: 'GND', color: 'gray' },
						{ id: '5v', label: '5V OUT', color: 'red' },
						{ id: 'x_step', label: 'X-STEP', color: 'green' },
						{ id: 'y_step', label: 'Y-STEP', color: 'green' },
						{ id: 'z_step', label: 'Z-STEP', color: 'green' },
						{ id: 'e0_step', label: 'E0-STEP', color: 'green' },
						{ id: 't0', label: 'T0', color: 'purple' }
					],
					right: [
						{ id: 'heat_0', label: 'HEAT0', color: 'green' },
						{ id: 'fan_0', label: 'FAN0', color: 'yellow' },
						{ id: 'servo_1', label: 'SERVO1', color: 'yellow' },
						{ id: 'sda', label: 'SDA', color: 'pink' },
						{ id: 'scl', label: 'SCL', color: 'pink' },
						{ id: 'gnd2', label: 'GND', color: 'gray' },
						{ id: 'aux5v', label: 'AUX 5V', color: 'red' },
						{ id: 'reset', label: 'RESET', color: 'blue' }
					]
				}
			}
		}
	]);

	let edges = writable<Edge[]>([]);

	function goBack() {
		window.location.href = '/';
	}
	
	function openDatasheetIngestion(id?: string) {
		window.location.href = '/datasheets/new';
	}

	function handleDragStart(event: DragEvent, type: string, name: string) {
		if (event.dataTransfer) {
			const comp = componentsDatabase.find(c => c.name === name);
			if (comp) {
				event.dataTransfer.setData('application/svelteflow', JSON.stringify({ 
					type: 'hardware', 
					name,
					diagram: comp.diagram
				}));
			} else {
				event.dataTransfer.setData('application/svelteflow', JSON.stringify({ type: 'hardware', name }));
			}
			event.dataTransfer.effectAllowed = 'move';
		}
	}

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

	function handleDropData(data: any, position: { x: number, y: number }) {
		const newNode = {
			id: Math.random().toString(),
			type: data.type,
			position,
			data: {
				label: data.name,
				theme: data.diagram?.theme || 'blue',
				pins: data.diagram?.pins || {
					left: [
						{ id: '1', label: 'PIN1', color: 'gray' },
						{ id: '2', label: 'PIN2', color: 'gray' }
					],
					right: [
						{ id: '3', label: 'PIN3', color: 'gray' },
						{ id: '4', label: 'PIN4', color: 'gray' }
					]
				}
			}
		};

		nodes.update((nds) => [...nds, newNode]);
	}

	function handleDragOver(event: DragEvent) {
		event.preventDefault();
		if (event.dataTransfer) {
			event.dataTransfer.dropEffect = 'move';
		}
	}

</script>

<svelte:window
	on:pointermove={handlePointerMove}
	on:pointerup={handlePointerUp}
/>

<div class="h-screen flex flex-col bg-white overflow-hidden text-gray-900 font-sans">
	{#if !isFullscreen}
	<nav class="bg-white border-b border-gray-200">
		<div class="w-full px-6 py-3 flex items-center justify-between">
			<div class="flex items-center gap-3 cursor-pointer" on:click={goBack}>
				<div class="w-8 h-8 flex items-center justify-center text-blue-600">
					<svg class="w-6 h-6" fill="currentColor" viewBox="0 0 24 24">
						<path d="M12 2L2 7l10 5 10-5-10-5zm0 10L2 17l10 5 10-5-10-5z" fill="currentColor"/>
					</svg>
				</div>
				<h1 class="text-xl font-bold text-gray-900">Hardware Prototyping Copilot</h1>
			</div>
			<div class="flex items-center gap-6">
				<a href="#" class="text-blue-600 font-medium border-b-2 border-blue-600 pb-4 -mb-4">Workspace</a>
				<a href="/datasheets" class="text-gray-500 font-medium hover:text-gray-900">Datasheets</a>
				<button class="px-5 py-2 bg-blue-600 text-white font-medium rounded hover:bg-blue-700 transition">
					Export
				</button>
			</div>
		</div>
	</nav>
	{/if}

	<div class="flex-1 flex w-full">
		{#if leftCollapsed}
			<button
				on:click={() => (leftCollapsed = false)}
				class="w-6 shrink-0 flex items-center justify-center border-r border-gray-200 bg-white hover:bg-gray-50 group"
				title="Show AI Copilot panel"
			>
				<svg class="w-4 h-4 text-gray-400 group-hover:text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
				</svg>
			</button>
		{:else}
		<div class="flex flex-col border-r border-gray-200 bg-[#FAFAFA] shrink-0" style="width: {leftWidth}px;">
			<div class="px-4 py-3 border-b border-gray-200 flex items-center gap-2 bg-white">
				<svg class="w-5 h-5 text-blue-600" fill="currentColor" viewBox="0 0 20 20">
					<path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"/>
				</svg>
				<h2 class="font-semibold text-sm">AI Copilot</h2>
			</div>
			<AIChatPanel bind:isPopoverOpen />
		</div>
		<div
			class="w-1 shrink-0 cursor-col-resize bg-gray-200 hover:bg-blue-400 active:bg-blue-500 transition-colors"
			on:pointerdown={() => startDrag('left')}
		></div>
		{/if}

		<div class="flex-1 bg-[#FAFAFA] relative overflow-hidden flex flex-col z-0">
			<SvelteFlowProvider>
				<CanvasInner 
					bind:nodes={$nodes} 
					bind:edges={$edges} 
					{nodeTypes} 
					{isCanvasLocked}
					{isFullscreen}
					onToggleFullscreen={toggleFullscreen}
					onToggleLock={() => isCanvasLocked = !isCanvasLocked}
					onDropData={handleDropData}
				/>
			</SvelteFlowProvider>
			<VoltagePopover bind:isOpen={isPopoverOpen} />
		</div>

		{#if rightCollapsed}
			<button
				on:click={() => (rightCollapsed = false)}
				class="w-6 shrink-0 flex items-center justify-center border-l border-gray-200 bg-white hover:bg-gray-50 group"
				title="Show Components panel"
			>
				<svg class="w-4 h-4 text-gray-400 group-hover:text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
				</svg>
			</button>
		{:else}
		<div
			class="w-1 shrink-0 cursor-col-resize bg-gray-200 hover:bg-blue-400 active:bg-blue-500 transition-colors"
			on:pointerdown={() => startDrag('right')}
		></div>
		<div class="bg-white border-l border-gray-200 flex flex-col z-20 shrink-0" style="width: {rightWidth}px;">
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
