<script lang="ts">
	import { authUser } from '$lib/stores/auth';
	import { api } from '$lib/api';

	let email = $state('');
	let password = $state('');
	let error = $state('');
	let loading = $state(false);

	async function handleLogin(e: SubmitEvent) {
		e.preventDefault();
		error = '';
		loading = true;
		try {
			const { token, user } = await api.post<{ token: string; user: any }>('/auth/login', {
				email,
				password
			});
			authUser.login(user, token);
			window.location.href = '/workspace';
		} catch (err: any) {
			error = err.message;
		} finally {
			loading = false;
		}
	}
</script>

<div class="min-h-screen bg-gray-50 flex items-center justify-center px-4">
	<div class="w-full max-w-sm">
		<div class="flex items-center gap-3 justify-center mb-8">
			<div class="w-8 h-8 flex items-center justify-center text-blue-600">
				<svg class="w-6 h-6" fill="currentColor" viewBox="0 0 24 24">
					<path d="M12 2L2 7l10 5 10-5-10-5zm0 10L2 17l10 5 10-5-10-5z" />
				</svg>
			</div>
			<span class="text-xl font-bold text-gray-900">Hardware Prototyping Copilot</span>
		</div>

		<div class="bg-white rounded-xl shadow-sm border border-gray-200 p-8">
			<h1 class="text-2xl font-bold text-gray-900 mb-1">Welcome back</h1>
			<p class="text-gray-500 text-sm mb-6">Sign in to your account to continue.</p>

			{#if error}
				<div class="mb-4 px-4 py-3 bg-red-50 border border-red-200 rounded text-sm text-red-700">
					{error}
				</div>
			{/if}

			<form onsubmit={handleLogin} class="space-y-4">
				<div>
					<label class="block text-sm font-medium text-gray-700 mb-1" for="email">Email</label>
					<input
						id="email"
						type="email"
						bind:value={email}
						required
						autocomplete="email"
						placeholder="you@example.com"
						class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
					/>
				</div>

				<div>
					<label class="block text-sm font-medium text-gray-700 mb-1" for="password"
						>Password</label
					>
					<input
						id="password"
						type="password"
						bind:value={password}
						required
						autocomplete="current-password"
						placeholder="••••••••"
						class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
					/>
				</div>

				<button
					type="submit"
					disabled={loading}
					class="w-full py-2.5 bg-blue-600 text-white font-medium rounded-lg hover:bg-blue-700 transition disabled:opacity-50"
				>
					{loading ? 'Signing in…' : 'Sign In'}
				</button>
			</form>

			<p class="mt-5 text-center text-sm text-gray-500">
				Don't have an account?
				<a href="/signup" class="text-blue-600 font-medium hover:underline">Sign up</a>
			</p>

			<div class="mt-4 pt-4 border-t border-gray-100 text-center">
				<p class="text-xs text-gray-400">Demo account</p>
				<p class="text-xs text-gray-500 font-mono mt-0.5">demo@hangover.dev / demo1234</p>
			</div>
		</div>
	</div>
</div>
