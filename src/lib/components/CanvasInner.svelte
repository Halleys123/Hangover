<script lang="ts">
	import { SvelteFlow, Background, Controls, useSvelteFlow, type Node, type Edge, type NodeTypes } from '@xyflow/svelte';
	
	interface Props {
		nodes: any;
		edges: any;
		nodeTypes: NodeTypes;
		isCanvasLocked: boolean;
		isFullscreen: boolean;
		onToggleFullscreen: () => void;
		onToggleLock: () => void;
		onDropData: (data: any, position: { x: number, y: number }) => void;
	}

	let {
		nodes = $bindable(),
		edges = $bindable(),
		nodeTypes,
		isCanvasLocked,
		isFullscreen,
		onToggleFullscreen,
		onToggleLock,
		onDropData
	}: Props = $props();

	const { screenToFlowPosition, fitView, zoomIn, zoomOut } = useSvelteFlow();

	function handleDropWrapper(event: DragEvent) {
		event.preventDefault();
		const dataStr = event.dataTransfer?.getData('application/svelteflow');
		if (!dataStr) return;
		
		const data = JSON.parse(dataStr);
		const position = screenToFlowPosition({ x: event.clientX, y: event.clientY });
		
		onDropData(data, position);
	}

	function handleDragOver(event: DragEvent) {
		event.preventDefault();
		if (event.dataTransfer) event.dataTransfer.dropEffect = 'move';
	}
</script>

<div class="w-full h-full relative" on:drop={handleDropWrapper} on:dragover={handleDragOver}>
	<SvelteFlow 
		bind:nodes
		bind:edges
		{nodeTypes} 
		fitView 
		nodesDraggable={!isCanvasLocked}
		nodesConnectable={!isCanvasLocked}
		elementsSelectable={!isCanvasLocked}
		panOnDrag={!isCanvasLocked}
		zoomOnScroll={!isCanvasLocked}
	>
		<Background />
		<!-- Regular floating controls -->
		<Controls showZoom={true} showFitView={true} showInteractive={false} showLock={false} />
	</SvelteFlow>

	<!-- Action Bar Overlay -->
	<div class="absolute bottom-6 left-1/2 -translate-x-1/2 bg-white/95 backdrop-blur-sm rounded-full shadow-[0_8px_30px_rgb(0,0,0,0.12)] border border-gray-200 px-3 py-1.5 flex items-center gap-3 text-sm text-gray-700 z-10 pointer-events-auto">
		<button title="Zoom Out" class="hover:text-blue-600 w-8 h-8 flex items-center justify-center font-bold text-xl bg-gray-50 hover:bg-blue-50 rounded-full transition-colors" on:click={() => zoomOut({duration: 200})}>-</button>
		<span class="font-bold text-[10px] tracking-widest text-gray-400">CANVAS</span>
		<button title="Zoom In" class="hover:text-blue-600 w-8 h-8 flex items-center justify-center font-bold text-xl bg-gray-50 hover:bg-blue-50 rounded-full transition-colors" on:click={() => zoomIn({duration: 200})}>+</button>
		
		<div class="w-px h-5 bg-gray-300"></div>
		
		<button title="Route & Center View" on:click={() => fitView({duration: 800, padding: 0.2})} class="flex items-center gap-1.5 font-medium hover:text-blue-600 transition-colors text-xs px-2 py-1 rounded hover:bg-blue-50">
			<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 8V4m0 0h4M4 4l5 5m11-1V4m0 0h-4m4 0l-5 5M4 16v4m0 0h4m-4 0l5-5m11 5l-5-5m5 5v-4m0 4h-4" /></svg>
			Auto / Center
		</button>
		
		<div class="w-px h-5 bg-gray-300"></div>
		
		<button title="Lock / Unlock Items" class="flex items-center gap-1.5 font-medium hover:text-blue-600 transition-colors text-xs px-2 py-1 rounded hover:bg-blue-50 {isCanvasLocked ? 'text-blue-600' : ''}" on:click={onToggleLock}>
			<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
				{#if isCanvasLocked}
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 15v2m-6 4h12a2 2 0 002-2v-6a2 2 0 00-2-2H6a2 2 0 00-2 2v6a2 2 0 002 2zm10-10V7a4 4 0 00-8 0v4h8z" />
				{:else}
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 11V7a4 4 0 118 0m-4 8v2m-6 4h12a2 2 0 002-2v-6a2 2 0 00-2-2H6a2 2 0 00-2 2v6a2 2 0 002 2z" />
				{/if}
			</svg>
			{isCanvasLocked ? 'Locked' : 'Lock'}
		</button>
		
		<div class="w-px h-5 bg-gray-300"></div>
		
		<button title="Toggle Fullscreen" class="flex items-center gap-1.5 font-medium hover:text-blue-600 transition-colors text-xs px-2 py-1 rounded hover:bg-blue-50" on:click={onToggleFullscreen}>
			<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
				{#if isFullscreen}
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 9L4 4m0 0v4m0-4h4m6 5l5-5m0 0v4m0-4h-4M9 15l-5 5m0 0v-4m0 4h4m6-5l5 5m0 0v-4m0 4h-4" />
				{:else}
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 8V4m0 0h4M4 4l5 5m11-1V4m0 0h-4m4 0l-5 5M4 16v4m0 0h4m-4 0l5-5m11 5l-5-5m5 5v-4m0 4h-4" />
				{/if}
			</svg>
			{isFullscreen ? 'Exit Full' : 'Full Screen'}
		</button>
	</div>
</div>