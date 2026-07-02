<script lang="ts">
	import { onMount } from 'svelte';
	import { marked } from 'marked';
	import { api } from '$lib/api';

	let {
		isPopoverOpen = $bindable(false),
		workspaceId = '',
		onCircuitGenerated = undefined as ((nodes: any[], edges: any[]) => void) | undefined
	} = $props();

	interface Message {
		role: 'user' | 'assistant';
		text: string;
		timestamp: Date | string;
	}

	let messages = $state<Message[]>([
		{ role: 'assistant', text: 'Hello! I am your **AI Hardware Architect**. Tell me what project or circuit goal you want to build!', timestamp: new Date() }
	]);

	let inputValue = $state('');
	let loading = $state(false);
	let startingNewChat = $state(false);

	onMount(() => {
		const loadChat = async () => {
			if (workspaceId) {
				try {
					const history = await api.get<Message[]>(`/projects/${workspaceId}/chat`);
					if (Array.isArray(history) && history.length > 0) {
						messages = history;
					}
				} catch {}
			}
		};
		loadChat();
		window.addEventListener('chat-reset', loadChat);
		return () => window.removeEventListener('chat-reset', loadChat);
	});

	async function handleNewChat() {
		if (!workspaceId || startingNewChat || loading) return;
		startingNewChat = true;
		try {
			const res = await api.post<any>(`/projects/${workspaceId}/chat/new`);
			if (res && res.chatHistory) {
				messages = res.chatHistory;
			}
		} catch (err) {
			console.error('Failed to start new chat:', err);
		} finally {
			startingNewChat = false;
		}
	}

	async function handleSendMessage(e?: Event) {
		if (e) e.preventDefault();
		const userText = inputValue.trim();
		if (!userText || loading) return;

		messages.push({ role: 'user', text: userText, timestamp: new Date() });
		inputValue = '';
		loading = true;

		try {
			if (workspaceId) {
				const res = await api.post<any>(`/projects/${workspaceId}/chat`, { query: userText });
				messages.push({
					role: 'assistant',
					text: res.message || res.reply || 'Processed request.',
					timestamp: new Date()
				});

				if (res.type === 'circuit_generated' && res.nodes && res.edges && onCircuitGenerated) {
					onCircuitGenerated(res.nodes, res.edges);
				}
			} else {
				const res = await api.post<any>('/chat', { message: userText, history: messages });
				messages.push({
					role: 'assistant',
					text: res.reply || res.message || 'I analyzed your query.',
					timestamp: new Date()
				});
			}
		} catch (err: any) {
			messages.push({
				role: 'assistant',
				text: `Error communicating with AI service: ${err.message}`,
				timestamp: new Date()
			});
		} finally {
			loading = false;
		}
	}
</script>

<div class="flex-1 flex flex-col min-h-0 h-full w-full">
	<!-- Chat Panel Header with New Chat Button -->
	{#if workspaceId}
	<div class="px-4 py-2 border-b border-slate-200 dark:border-zinc-800 bg-slate-50/50 dark:bg-zinc-900/50 flex items-center justify-between shrink-0">
		<span class="text-[10px] font-bold text-slate-400 dark:text-zinc-500 uppercase tracking-wider">AI Conversation</span>
		<button
			onclick={handleNewChat}
			disabled={startingNewChat || loading}
			class="flex items-center gap-1.5 px-2.5 py-1 bg-white dark:bg-zinc-800 border border-slate-200 dark:border-zinc-700 hover:border-blue-500 dark:hover:border-blue-500 text-slate-600 dark:text-zinc-300 hover:text-blue-600 dark:hover:text-blue-400 rounded text-[11px] font-semibold transition shadow-sm disabled:opacity-50"
			title="Start a fresh chat while preserving conversation summary for AI"
		>
			<svg class="w-3 h-3 {startingNewChat ? 'animate-spin' : ''}" fill="none" stroke="currentColor" viewBox="0 0 24 24">
				{#if startingNewChat}
					<circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
				{:else}
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6v6m0 0v6m0-6h6m-6 0H6"/>
				{/if}
			</svg>
			{startingNewChat ? 'Summarizing context…' : 'New Chat'}
		</button>
	</div>
	{/if}

	<!-- Messages List -->
	<div class="flex-1 overflow-y-auto p-4 space-y-4">
		{#each messages as msg}
			<div class="flex gap-3 {msg.role === 'user' ? 'justify-end' : 'justify-start'}">
				{#if msg.role === 'assistant'}
					<div class="w-6 h-6 rounded bg-blue-100 dark:bg-blue-950 text-blue-600 dark:text-blue-400 shrink-0 flex items-center justify-center mt-1">
						<svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20"><path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 12a1 1 0 11-2 0 1 1 0 012 0zm4 0a1 1 0 11-2 0 1 1 0 012 0zm-2-4a1 1 0 11-2 0 1 1 0 012 0z"/></svg>
					</div>
				{/if}
				<div class="max-w-[85%] px-4 py-3 rounded-2xl {msg.role === 'user' ? 'bg-blue-600 text-white rounded-br-sm' : 'bg-white dark:bg-zinc-800 border border-slate-200 dark:border-zinc-700 text-slate-800 dark:text-zinc-200 rounded-bl-sm shadow-sm'} text-sm leading-relaxed transition-colors overflow-hidden">
					{#if msg.role === 'assistant'}
						<div class="markdown-body text-xs sm:text-sm leading-relaxed">
							{@html marked.parse(msg.text || '')}
						</div>
					{:else}
						<p class="whitespace-pre-wrap">{msg.text}</p>
					{/if}
				</div>
				{#if msg.role === 'user'}
					<div class="w-6 h-6 rounded-full bg-slate-300 dark:bg-zinc-700 shrink-0 mt-1 flex items-center justify-center">
						<svg class="w-4 h-4 text-slate-600 dark:text-zinc-300" fill="currentColor" viewBox="0 0 20 20"><path fill-rule="evenodd" d="M10 9a3 3 0 100-6 3 3 0 000 6zm-7 9a7 7 0 1114 0H3z" clip-rule="evenodd"/></svg>
					</div>
				{/if}
			</div>
		{/each}

		{#if loading || startingNewChat}
			<div class="flex gap-3 justify-start">
				<div class="w-6 h-6 rounded bg-blue-100 dark:bg-blue-950 text-blue-600 dark:text-blue-400 shrink-0 flex items-center justify-center mt-1">
					<svg class="w-4 h-4 animate-spin" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path></svg>
				</div>
				<div class="px-4 py-3 rounded-2xl bg-white dark:bg-zinc-800 border border-slate-200 dark:border-zinc-700 text-slate-500 dark:text-zinc-400 text-xs rounded-bl-sm shadow-sm animate-pulse">
					{startingNewChat ? 'Summarizing project history & preparing new session…' : 'AI Architect is thinking & checking Cognee graph...'}
				</div>
			</div>
		{/if}
	</div>

	<!-- Input Area -->
	<div class="p-4 bg-white dark:bg-zinc-900 border-t border-slate-200 dark:border-zinc-800 transition-colors shrink-0">
		<form onsubmit={handleSendMessage} class="relative">
			<input type="text" bind:value={inputValue} disabled={loading || startingNewChat} placeholder="Ask about wiring, components..." class="w-full pl-4 pr-10 py-2.5 bg-white dark:bg-zinc-800 border border-slate-300 dark:border-zinc-700 rounded-lg text-sm text-slate-900 dark:text-white placeholder:text-slate-400 focus:outline-none focus:border-blue-500 focus:ring-1 focus:ring-blue-500 disabled:opacity-60 transition-colors" />
			<button type="submit" disabled={loading || startingNewChat} class="absolute right-2 top-1/2 -translate-y-1/2 p-1.5 text-blue-600 dark:text-blue-400 hover:bg-blue-50 dark:hover:bg-zinc-700 disabled:opacity-40 rounded transition-colors">
				<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8"/></svg>
			</button>
		</form>
	</div>
</div>

<style>
	:global(.markdown-body p) { margin-bottom: 0.6rem; }
	:global(.markdown-body p:last-child) { margin-bottom: 0; }
	:global(.markdown-body h1), :global(.markdown-body h2), :global(.markdown-body h3), :global(.markdown-body h4) { font-weight: 700; margin-top: 0.85rem; margin-bottom: 0.4rem; color: inherit; }
	:global(.markdown-body ul) { list-style-type: disc; padding-left: 1.25rem; margin-top: 0.4rem; margin-bottom: 0.6rem; }
	:global(.markdown-body ol) { list-style-type: decimal; padding-left: 1.25rem; margin-top: 0.4rem; margin-bottom: 0.6rem; }
	:global(.markdown-body li) { margin-bottom: 0.3rem; }
	:global(.markdown-body strong) { font-weight: 600; color: inherit; }
	:global(.markdown-body code) { background-color: rgba(128,128,128,0.18); padding: 0.15rem 0.4rem; border-radius: 0.25rem; font-family: monospace; font-size: 0.88em; }
	:global(.markdown-body pre) { background-color: rgba(15,23,42,0.85); color: #f8fafc; padding: 0.75rem; border-radius: 0.5rem; overflow-x: auto; font-family: monospace; font-size: 0.85em; margin-top: 0.5rem; margin-bottom: 0.5rem; }
	:global(.markdown-body pre code) { background-color: transparent; padding: 0; }
</style>