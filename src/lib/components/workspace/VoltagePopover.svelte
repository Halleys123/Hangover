<script lang="ts">
	let { isOpen = $bindable(false) } = $props();
</script>

{#if isOpen}
<div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-white rounded-xl shadow-[0_8px_30px_rgb(0,0,0,0.12)] border border-gray-200 p-6 w-112.5 z-50">
	<div class="flex items-start justify-between mb-4">
		<div class="flex items-center gap-3">
			<div class="w-10 h-10 rounded-full bg-orange-50 flex items-center justify-center">
				<svg class="w-5 h-5 text-orange-500" fill="currentColor" viewBox="0 0 20 20"><path fill-rule="evenodd" d="M8.257 3.099c.765-1.36 2.722-1.36 3.486 0l5.58 9.92c.75 1.334-.213 2.98-1.742 2.98H4.42c-1.53 0-2.493-1.646-1.743-2.98l5.58-9.92zM11 13a1 1 0 11-2 0 1 1 0 012 0zm-1-8a1 1 0 00-1 1v3a1 1 0 002 0V6a1 1 0 00-1-1z" clip-rule="evenodd"/></svg>
			</div>
			<h3 class="text-lg font-bold text-gray-900">Voltage Mismatch Detected</h3>
		</div>
		<button onclick={() => isOpen = false} class="text-gray-400 hover:text-gray-600">
			<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/></svg>
		</button>
	</div>
	<p class="text-gray-600 mb-4 text-sm leading-relaxed">
		Applying <code class="bg-gray-100 px-1 py-0.5 rounded text-gray-800 font-mono text-xs">5V</code> to the ESP32 RX pin will damage the board.
		The ESP32 logic level operates safely at <code class="bg-gray-100 px-1 py-0.5 rounded text-gray-800 font-mono text-xs">3.3V</code>.
	</p>
	<div class="bg-slate-50 border border-slate-100 rounded-lg p-3 mb-6 flex gap-3 text-sm">
		<svg class="w-5 h-5 text-blue-500 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.663 17h4.673M12 3v1m6.364 1.636l-.707.707M21 12h-1M4 12H3m3.343-5.657l-.707-.707m2.828 9.9a5 5 0 117.072 0l-.548.547A3.374 3.374 0 0014 18.469V19a2 2 0 11-4 0v-.531c0-.895-.356-1.754-.988-2.386l-.548-.547z"/></svg>
		<p class="text-slate-700 font-medium">An AI auto-fix is available to safely bridge this connection.</p>
	</div>
	<div class="flex gap-3 justify-end mt-4">
		<button onclick={() => isOpen = false} class="px-4 py-2 border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-50 font-medium text-sm transition flex-1">Cancel Connection</button>
		<button onclick={() => isOpen = false} class="px-4 py-2 bg-[#0F172A] text-white rounded-lg hover:bg-[#1E293B] font-medium text-sm transition flex-[1.5]">Insert Logic Level Converter</button>
	</div>
</div>
<div class="absolute inset-0 bg-gray-500/20 z-40 backdrop-blur-[1px]"></div>
{/if}