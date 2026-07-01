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

	let messages = $state<Message[]>([
		{
			role: 'assistant',
			text: 'Hello! I\'m ready to help you build your circuit. What are we prototyping today?',
			timestamp: new Date()
		},
		{
			role: 'user',
			text: 'I need to connect an ESP32 to a DHT22 temperature sensor.',
			timestamp: new Date()
		},
		{
			role: 'assistant',
			text: 'Great choice. The DHT22 requires 3.3V to 5V power, a ground connection, and one digital I/O pin for data.\n\nI\'ve added the ESP32 and DHT22 to your canvas.',
			timestamp: new Date()
		}
	]);

	let inputValue = $state('');
	
	let datasheets = $state([
		{ id: '1', name: 'ESP32-WROOM-32_Datasheet.pdf', size: '1.2 MB', parsed: true },
		{ id: '2', name: 'DHT22_Specifications.pdf', size: '450 KB', parsed: true }
	]);
	
	let components = $state<Component[]>([
		{ id: '1', name: 'ESP32 DevKit V1', type: '3.3V Logic • WiFi/BT', uploaded: true },
		{ id: '2', name: 'Arduino Uno R3', type: '5V Logic • ATmega328P', uploaded: true },
		{ id: '3', name: 'DHT22 Temp/Humid', type: '3.3V-5V • Digital 1-Wire', uploaded: true },
		{ id: '4', name: 'MPU6050 IMU', type: '3.3V • I2C Interface', uploaded: true }
	]);

	function handleSendMessage() {
		if (inputValue.trim()) {
			messages.push({
				role: 'user',
				text: inputValue,
				timestamp: new Date()
			});
			inputValue = '';

			setTimeout(() => {
				const responses = [
					'I\'ve analyzed your design. The connections look valid so far.',
					'Be careful with voltage levels when connecting different components.'
				];
				messages.push({
					role: 'assistant',
					text: responses[Math.floor(Math.random() * responses.length)],
					timestamp: new Date()
				});
				
				// Trigger popover for demo if they mention voltage
				if (messages[messages.length-1].text.includes('voltage')) {
					isPopoverOpen = true;
				}
			}, 800);
		}
	}

	function goBack() {
		window.location.href = '/';
	}
	
	function openDatasheetIngestion(id?: string) {
		window.location.href = '/datasheets/new';
	}

	function handleDragStart(event: DragEvent, type: string, name: string) {
		if (event.dataTransfer) {
			event.dataTransfer.setData('application/svelteflow', JSON.stringify({ type: 'hardware', name }));
			event.dataTransfer.effectAllowed = 'move';
		}
	}

	function handleDrop(event: DragEvent) {
		event.preventDefault();

		if (!event.dataTransfer || !svelteFlowInstance) {
			return;
		}

		const dataStr = event.dataTransfer.getData('application/svelteflow');
		if (!dataStr) return;

		const data = JSON.parse(dataStr);

		const position = svelteFlowInstance.screenToFlowPosition({
			x: event.clientX,
			y: event.clientY
		});

		const newNode = {
			id: Math.random().toString(),
			type: data.type,
			position,
			data: {
				label: data.name,
				theme: 'blue',
				pins: {
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

	let zoomLevel = $state(100);

	function handleAutoRoute() {
		if (svelteFlowInstance) {
			svelteFlowInstance.fitView({ duration: 800, padding: 0.2 });
		}
	}

	// This is a separate component so we can use useSvelteFlow
	let CanvasControls = $derived(
		function CanvasControlsInner() {
			const { zoomIn, zoomOut, getZoom } = useSvelteFlow();
			
			// We can't easily auto-update the zoom level text without subscribing to the viewport store,
			// but we can make the buttons work natively.
			
			function handleZoomIn() { zoomIn({ duration: 200 }); }
			function handleZoomOut() { zoomOut({ duration: 200 }); }

			return { handleZoomIn, handleZoomOut };
		}
	);

</script>

<div class="h-screen flex flex-col bg-white overflow-hidden text-gray-900 font-sans">
	<!-- Top Navigation -->
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

	<div class="flex-1 flex w-full">
		<!-- Left Pane: AI Chatbot -->
		<div class="w-[400px] flex flex-col border-r border-gray-200 bg-[#FAFAFA]">
			<div class="px-4 py-3 border-b border-gray-200 flex items-center gap-2 bg-white">
				<svg class="w-5 h-5 text-blue-600" fill="currentColor" viewBox="0 0 20 20">
					<path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"/>
				</svg>
				<h2 class="font-semibold text-sm">AI Copilot</h2>
			</div>

			<div class="flex-1 overflow-y-auto p-4 space-y-4">
				{#each messages as msg}
					<div class="flex gap-3 {msg.role === 'user' ? 'justify-end' : 'justify-start'}">
						{#if msg.role === 'assistant'}
							<div class="w-6 h-6 rounded bg-blue-100 text-blue-600 flex-shrink-0 flex items-center justify-center mt-1">
								<svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
									<path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"/>
								</svg>
							</div>
						{/if}
						<div
							class="max-w-[85%] px-4 py-3 rounded-2xl {msg.role === 'user'
								? 'bg-blue-600 text-white rounded-br-sm'
								: 'bg-white border border-gray-200 text-gray-800 rounded-bl-sm shadow-sm'} text-sm leading-relaxed"
						>
							{#if msg.text.includes('\n\n')}
								{#each msg.text.split('\n\n') as para, i}
									<p class={i > 0 ? "mt-3 text-gray-500" : ""}>{para}</p>
								{/each}
							{:else}
								<p>{msg.text}</p>
							{/if}
						</div>
						{#if msg.role === 'user'}
							<div class="w-6 h-6 rounded-full bg-gray-300 flex-shrink-0 mt-1 flex items-center justify-center">
								<svg class="w-4 h-4 text-gray-600" fill="currentColor" viewBox="0 0 20 20">
									<path fill-rule="evenodd" d="M10 9a3 3 0 100-6 3 3 0 000 6zm-7 9a7 7 0 1114 0H3z" clip-rule="evenodd"/>
								</svg>
							</div>
						{/if}
					</div>
				{/each}
			</div>

			<div class="p-4 bg-white border-t border-gray-200">
				<form on:submit|preventDefault={handleSendMessage} class="relative">
					<input
						type="text"
						bind:value={inputValue}
						placeholder="Ask about wiring, components..."
						class="w-full pl-4 pr-10 py-2.5 bg-white border border-gray-300 rounded-lg text-sm focus:outline-none focus:border-blue-500 focus:ring-1 focus:ring-blue-500"
					/>
					<button type="submit" class="absolute right-2 top-1/2 -translate-y-1/2 p-1.5 text-blue-600 hover:bg-blue-50 rounded">
						<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8"/>
						</svg>
					</button>
				</form>
			</div>
		</div>

		<!-- Center Pane: Visual Assembly Canvas -->
		<div class="flex-1 bg-[#FAFAFA] relative overflow-hidden flex flex-col z-0"
			 on:drop={handleDrop}
			 on:dragover={handleDragOver}
		>
			<SvelteFlowProvider>
				<SvelteFlow nodes={$nodes} edges={$edges} {nodeTypes} fitView oninit={(instance) => svelteFlowInstance = instance}>
					<Background />
					<Controls showZoom={false} showInteractive={true} showFitView={false} />
				</SvelteFlow>
			</SvelteFlowProvider>
			
			<!-- Bottom Canvas Controls (Overlay) -->
			<SvelteFlowProvider>
				<!-- Dummy provider just to satisfy svelteflow context if we were to move controls inside, but we need instance -->
			</SvelteFlowProvider>
			<div class="absolute bottom-6 left-1/2 -translate-x-1/2 bg-white rounded-full shadow-sm border border-gray-200 px-4 py-2 flex items-center gap-4 text-sm text-gray-600 z-10 pointer-events-auto">
				<button class="hover:text-gray-900 w-6 h-6 flex items-center justify-center font-bold" on:click={() => svelteFlowInstance?.zoomOut({duration: 200})}>-</button>
				<span>Zoom</span>
				<button class="hover:text-gray-900 w-6 h-6 flex items-center justify-center font-bold" on:click={() => svelteFlowInstance?.zoomIn({duration: 200})}>+</button>
				<div class="w-px h-4 bg-gray-300"></div>
				<button on:click={handleAutoRoute} class="flex items-center gap-1 text-blue-600 font-medium hover:text-blue-700">
					<!-- <svg class="w-4 h-4" fill="none" class="text-blue-600" viewBox="0 0 24 24" stroke="currentColor">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z" />
					</svg> -->
					Auto-Route / Cente View
				</button>
			</div>
			
			<!-- Popover Alert Demo -->
			{#if isPopoverOpen}
				<div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-white rounded-xl shadow-[0_8px_30px_rgb(0,0,0,0.12)] border border-gray-200 p-6 w-[450px] z-50">
					<div class="flex items-start justify-between mb-4">
						<div class="flex items-center gap-3">
							<div class="w-10 h-10 rounded-full bg-orange-50 flex items-center justify-center">
								<svg class="w-5 h-5 text-orange-500" fill="currentColor" viewBox="0 0 20 20">
									<path fill-rule="evenodd" d="M8.257 3.099c.765-1.36 2.722-1.36 3.486 0l5.58 9.92c.75 1.334-.213 2.98-1.742 2.98H4.42c-1.53 0-2.493-1.646-1.743-2.98l5.58-9.92zM11 13a1 1 0 11-2 0 1 1 0 012 0zm-1-8a1 1 0 00-1 1v3a1 1 0 002 0V6a1 1 0 00-1-1z" clip-rule="evenodd"/>
								</svg>
							</div>
							<h3 class="text-lg font-bold text-gray-900">Voltage Mismatch Detected</h3>
						</div>
						<button on:click={() => isPopoverOpen = false} class="text-gray-400 hover:text-gray-600">
							<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
							</svg>
						</button>
					</div>
					
					<p class="text-gray-600 mb-4 text-sm leading-relaxed">
						Applying <code class="bg-gray-100 px-1 py-0.5 rounded text-gray-800 font-mono text-xs">5V</code> to the ESP32 RX pin will damage the board.
						The ESP32 logic level operates safely at <code class="bg-gray-100 px-1 py-0.5 rounded text-gray-800 font-mono text-xs">3.3V</code>.
					</p>
					
					<div class="bg-slate-50 border border-slate-100 rounded-lg p-3 mb-6 flex gap-3 text-sm">
						<svg class="w-5 h-5 text-blue-500 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.663 17h4.673M12 3v1m6.364 1.636l-.707.707M21 12h-1M4 12H3m3.343-5.657l-.707-.707m2.828 9.9a5 5 0 117.072 0l-.548.547A3.374 3.374 0 0014 18.469V19a2 2 0 11-4 0v-.531c0-.895-.356-1.754-.988-2.386l-.548-.547z"/>
						</svg>
						<p class="text-slate-700 font-medium">An AI auto-fix is available to safely bridge this connection.</p>
					</div>
					
					<div class="flex gap-3 justify-end mt-4">
						<button on:click={() => isPopoverOpen = false} class="px-4 py-2 border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-50 font-medium text-sm transition flex-1">
							Cancel Connection
						</button>
						<button on:click={() => { isPopoverOpen = false; }} class="px-4 py-2 bg-[#0F172A] text-white rounded-lg hover:bg-[#1E293B] font-medium text-sm transition flex-[1.5]">
							Insert Logic Level Converter
						</button>
					</div>
				</div>
				<!-- Backdrop -->
				<div class="absolute inset-0 bg-gray-500/20 z-40 backdrop-blur-[1px]"></div>
			{/if}
		</div>

		<!-- Right Pane: Components & Datasheets -->
		<div class="w-[320px] bg-white border-l border-gray-200 flex flex-col z-20">
			<!-- Tab Headers -->
			<div class="flex border-b border-gray-200">
				<button 
					class="flex-1 py-3 text-sm font-semibold {rightTab === 'components' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-gray-500 hover:text-gray-700'}"
					on:click={() => rightTab = 'components'}
				>
					Components
				</button>
				<button 
					class="flex-1 py-3 text-sm font-semibold {rightTab === 'datasheets' ? 'text-blue-600 border-b-2 border-blue-600' : 'text-gray-500 hover:text-gray-700'}"
					on:click={() => rightTab = 'datasheets'}
				>
					Datasheets
				</button>
			</div>

			{#if rightTab === 'components'}
				<div class="p-4 border-b border-gray-200">
					<div class="relative">
						<svg class="w-4 h-4 absolute left-3 top-1/2 -translate-y-1/2 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
						</svg>
						<input type="text" placeholder="Search ICs, sensors..." class="w-full pl-9 pr-3 py-2 bg-gray-50 border border-gray-200 rounded text-sm focus:outline-none focus:bg-white focus:border-blue-500 focus:ring-1 focus:ring-blue-500" />
					</div>
				</div>

				<div class="flex-1 overflow-y-auto p-4">
					<div class="mb-6">
						<h3 class="text-[11px] font-bold text-gray-500 uppercase tracking-wider mb-2">Microcontrollers</h3>
						<div class="space-y-2">
							<div draggable="true" on:dragstart={(e) => handleDragStart(e, 'hardware', 'ESP32 DevKit V1')} class="p-3 border border-gray-200 rounded-md hover:border-gray-300 cursor-grab flex items-center justify-between bg-white shadow-sm">
								<div class="flex items-center gap-3">
									<svg class="w-5 h-5 text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m14-6h2m-2 6h2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z" /></svg>
									<div>
										<p class="text-sm font-semibold text-gray-900">ESP32 DevKit V1</p>
										<p class="text-[11px] text-gray-500">3.3V Logic • WiFi/BT</p>
									</div>
								</div>
								<div class="grid grid-cols-2 gap-0.5 text-gray-300">
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
								</div>
							</div>
							
							<div draggable="true" on:dragstart={(e) => handleDragStart(e, 'hardware', 'Arduino Uno R3')} class="p-3 border border-gray-200 rounded-md hover:border-gray-300 cursor-grab flex items-center justify-between bg-white">
								<div class="flex items-center gap-3">
									<svg class="w-5 h-5 text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m14-6h2m-2 6h2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z" /></svg>
									<div>
										<p class="text-sm font-semibold text-gray-900">Arduino Uno R3</p>
										<p class="text-[11px] text-gray-500">5V Logic • ATmega328P</p>
									</div>
								</div>
								<div class="grid grid-cols-2 gap-0.5 text-gray-300">
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
								</div>
							</div>
						</div>
					</div>

					<div>
						<h3 class="text-[11px] font-bold text-gray-500 uppercase tracking-wider mb-2">Sensors</h3>
						<div class="space-y-2">
							<div draggable="true" on:dragstart={(e) => handleDragStart(e, 'hardware', 'DHT22 Temp/Humid')} class="p-3 border border-gray-200 rounded-md hover:border-gray-300 cursor-grab flex items-center justify-between bg-white shadow-sm">
								<div class="flex items-center gap-3">
									<svg class="w-5 h-5 text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z" /></svg>
									<div>
										<p class="text-sm font-semibold text-gray-900">DHT22 Temp/Humid</p>
										<p class="text-[11px] text-gray-500">3.3V-5V • Digital 1-Wire</p>
									</div>
								</div>
								<div class="grid grid-cols-2 gap-0.5 text-gray-300">
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
								</div>
							</div>

							<div draggable="true" on:dragstart={(e) => handleDragStart(e, 'hardware', 'MPU6050 IMU')} class="p-3 border border-gray-200 rounded-md hover:border-gray-300 cursor-grab flex items-center justify-between bg-white">
								<div class="flex items-center gap-3">
									<svg class="w-5 h-5 text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M3 15a4 4 0 004 4h9a5 5 0 10-.1-9.999 5.002 5.002 0 10-9.78 2.096A4.001 4.001 0 003 15z" /></svg>
									<div>
										<p class="text-sm font-semibold text-gray-900">MPU6050 IMU</p>
										<p class="text-[11px] text-gray-500">3.3V • I2C Interface</p>
									</div>
								</div>
								<div class="grid grid-cols-2 gap-0.5 text-gray-300">
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
									<div class="w-1 h-1 bg-current rounded-full"></div><div class="w-1 h-1 bg-current rounded-full"></div>
								</div>
							</div>
						</div>
					</div>

					<div class="p-4 border-t border-gray-200 bg-gray-50">
						<button class="w-full py-2 text-sm text-blue-600 font-medium hover:text-blue-700 flex items-center justify-center gap-2">
							<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4" />
							</svg>
							Import Custom Part
						</button>
					</div>
				</div>
			{:else}
				<!-- Datasheets Tab -->
				<div class="p-4 border-b border-gray-200">
					<button 
						on:click={() => openDatasheetIngestion()}
						class="w-full py-2.5 bg-blue-600 text-white text-sm font-medium rounded hover:bg-blue-700 transition flex items-center justify-center gap-2 shadow-sm"
					>
						<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" />
						</svg>
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
										<svg class="w-5 h-5" fill="currentColor" viewBox="0 0 24 24">
											<path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 2l5 5h-5V4zM8 14h8v2H8v-2zm0-4h8v2H8v-2z" />
										</svg>
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
		</div>
	</div>
</div>

<style>
	:global(body) {
		overflow: hidden;
	}
</style>
