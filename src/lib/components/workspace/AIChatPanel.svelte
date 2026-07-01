<script lang="ts">
	let { isPopoverOpen = $bindable(false) } = $props();

	interface Message {
		role: 'user' | 'assistant';
		text: string;
		timestamp: Date;
	}

	let messages = $state<Message[]>([
		{ role: 'assistant', text: 'Hello! I\'m ready to help you build your circuit. What are we prototyping today?', timestamp: new Date() }
	]);

	let inputValue = $state('');

	function handleSendMessage() {
		if (inputValue.trim()) {
			messages.push({ role: 'user', text: inputValue, timestamp: new Date() });
			inputValue = '';
			setTimeout(() => {
				const responses = ['I\'ve analyzed your design. The connections look valid so far.', 'Be careful with voltage levels when connecting different components.'];
				messages.push({ role: 'assistant', text: responses[Math.floor(Math.random() * responses.length)], timestamp: new Date() });
				if (messages[messages.length-1].text.includes('voltage')) {
					isPopoverOpen = true;
				}
			}, 800);
		}
	}
</script>

<div class="flex-1 overflow-y-auto p-4 space-y-4">
	{#each messages as msg}
		<div class="flex gap-3 {msg.role === 'user' ? 'justify-end' : 'justify-start'}">
			{#if msg.role === 'assistant'}
				<div class="w-6 h-6 rounded bg-blue-100 dark:bg-blue-950 text-blue-600 dark:text-blue-400 shrink-0 flex items-center justify-center mt-1">
					<svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20"><path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"/></svg>
				</div>
			{/if}
			<div class="max-w-[85%] px-4 py-3 rounded-2xl {msg.role === 'user' ? 'bg-blue-600 text-white rounded-br-sm' : 'bg-white dark:bg-zinc-800 border border-slate-200 dark:border-zinc-700 text-slate-800 dark:text-zinc-200 rounded-bl-sm shadow-sm'} text-sm leading-relaxed transition-colors">
				{#if msg.text.includes('\n\n')}
					{#each msg.text.split('\n\n') as para, i}
						<p class={i > 0 ? "mt-3 text-slate-500 dark:text-zinc-400" : ""}>{para}</p>
					{/each}
				{:else}
					<p>{msg.text}</p>
				{/if}
			</div>
			{#if msg.role === 'user'}
				<div class="w-6 h-6 rounded-full bg-slate-300 dark:bg-zinc-700 shrink-0 mt-1 flex items-center justify-center">
					<svg class="w-4 h-4 text-slate-600 dark:text-zinc-300" fill="currentColor" viewBox="0 0 20 20"><path fill-rule="evenodd" d="M10 9a3 3 0 100-6 3 3 0 000 6zm-7 9a7 7 0 1114 0H3z" clip-rule="evenodd"/></svg>
				</div>
			{/if}
		</div>
	{/each}
</div>

<div class="p-4 bg-white dark:bg-zinc-900 border-t border-slate-200 dark:border-zinc-800 transition-colors">
	<form onsubmit={handleSendMessage} class="relative">
		<input type="text" bind:value={inputValue} placeholder="Ask about wiring, components..." class="w-full pl-4 pr-10 py-2.5 bg-white dark:bg-zinc-800 border border-slate-300 dark:border-zinc-700 rounded-lg text-sm text-slate-900 dark:text-white placeholder:text-slate-400 focus:outline-none focus:border-blue-500 focus:ring-1 focus:ring-blue-500 transition-colors" />
		<button type="submit" class="absolute right-2 top-1/2 -translate-y-1/2 p-1.5 text-blue-600 dark:text-blue-400 hover:bg-blue-50 dark:hover:bg-zinc-700 rounded transition-colors">
			<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8"/></svg>
		</button>
	</form>
</div>