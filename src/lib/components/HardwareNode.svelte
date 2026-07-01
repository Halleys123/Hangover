<script lang="ts">
	import { Handle, Position } from '@xyflow/svelte';
	
	interface Props {
		data: {
			label: string;
			theme: 'orange' | 'blue';
			pins: {
				left: Array<{ id: string; label: string; color: string }>;
				right: Array<{ id: string; label: string; color: string }>;
			}
		}
	}
	let { data }: Props = $props();

	const themeColors = {
		orange: {
			border: 'border-orange-500/30',
			header: 'text-orange-500',
			shadow: 'shadow-[0_0_15px_rgba(249,115,22,0.1)]',
			bg: 'bg-[#0f1115]',
			chip: 'bg-[#1a1311] border-orange-500/20'
		},
		blue: {
			border: 'border-blue-500/30',
			header: 'text-blue-500',
			shadow: 'shadow-[0_0_15px_rgba(59,130,246,0.1)]',
			bg: 'bg-[#0f1115]',
			chip: 'bg-[#11131a] border-blue-500/20'
		}
	};

	let colors = $derived(themeColors[data.theme] || themeColors.blue);

	function getPinColorClass(color: string) {
		const map: Record<string, string> = {
			'red': 'bg-red-500 shadow-[0_0_8px_rgba(239,68,68,0.6)]',
			'gray': 'bg-gray-500 shadow-[0_0_8px_rgba(107,114,128,0.6)]',
			'blue': 'bg-blue-400 shadow-[0_0_8px_rgba(96,165,250,0.6)]',
			'green': 'bg-emerald-400 shadow-[0_0_8px_rgba(52,211,153,0.6)]',
			'yellow': 'bg-yellow-400 shadow-[0_0_8px_rgba(250,204,21,0.6)]',
			'pink': 'bg-fuchsia-400 shadow-[0_0_8px_rgba(232,121,249,0.6)]',
			'purple': 'bg-indigo-400 shadow-[0_0_8px_rgba(129,140,248,0.6)]'
		};
		return map[color] || map['gray'];
	}
</script>

<!-- Outer Container -->
<div class="{colors.bg} rounded-xl border {colors.border} {colors.shadow} p-4 font-mono w-48 relative"
	 style="background-image: linear-gradient(rgba(255,255,255,0.03) 1px, transparent 1px), linear-gradient(90deg, rgba(255,255,255,0.03) 1px, transparent 1px); background-size: 10px 10px;">
	
	<!-- Header -->
	<div class="flex items-center gap-2 mb-6">
		<div class="w-2 h-2 rounded-full {data.theme === 'orange' ? 'bg-orange-500 shadow-[0_0_8px_rgba(249,115,22,0.8)]' : 'bg-blue-500 shadow-[0_0_8px_rgba(59,130,246,0.8)]'}"></div>
		<h3 class="{colors.header} text-sm font-bold tracking-widest">{data.label}</h3>
	</div>

	<div class="flex justify-between relative min-h-[120px]">
		<!-- Center Chip -->
		<div class="absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 w-16 h-16 rounded {colors.chip} border"></div>

		<!-- Left Pins -->
		<div class="flex flex-col gap-4 relative z-10 w-full">
			{#each data.pins.left as pin, i}
				<div class="flex items-center justify-start relative w-full h-4">
					<Handle 
						type="target" 
						position={Position.Left} 
						id={pin.id} 
						style="left: -20px; background: transparent; border: none; width: 12px; height: 12px; margin-top: -6px;" 
					>
						<div class="w-3 h-3 rounded-full {getPinColorClass(pin.color)}"></div>
					</Handle>
					<span class="text-[9px] text-slate-400 ml-1 font-bold">{pin.label}</span>
				</div>
			{/each}
		</div>

		<!-- Right Pins -->
		<div class="flex flex-col gap-4 relative z-10 w-full text-right mt-auto">
			{#each data.pins.right as pin, i}
				<div class="flex items-center justify-end relative w-full h-4">
					<span class="text-[9px] text-slate-400 mr-1 font-bold">{pin.label}</span>
					<Handle 
						type="source" 
						position={Position.Right} 
						id={pin.id} 
						style="right: -20px; background: transparent; border: none; width: 12px; height: 12px; margin-top: -6px;"
					>
						<div class="w-3 h-3 rounded-full {getPinColorClass(pin.color)}"></div>
					</Handle>
				</div>
			{/each}
		</div>
	</div>
</div>