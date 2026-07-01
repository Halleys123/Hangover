<script lang="ts">
	import { authUser } from '$lib/stores/auth';
	import { api } from '$lib/api';

	let name = $state('');
	let email = $state('');
	let password = $state('');
	let error = $state('');
	let loading = $state(false);

	async function handleSignup(e: SubmitEvent) {
		e.preventDefault();
		error = '';
		loading = true;
		try {
			const { token, user } = await api.post<{ token: string; user: any }>('/auth/signup', {
				name,
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
			<h1 class="text-2xl font-bold text-gray-900 mb-1">Create an account</h1>
			<p class="text-gray-500 text-sm mb-6">Start designing hardware with AI assistance.</p>

			{#if error}
				<div class="mb-4 px-4 py-3 bg-red-50 border border-red-200 rounded text-sm text-red-700">
					{error}
				</div>
			{/if}

			<form onsubmit={handleSignup} class="space-y-4">
				<div>
					<label class="block text-sm font-medium text-gray-700 mb-1" for="name">Name</label>
					<input
						id="name"
						type="text"
						bind:value={name}
						required
						autocomplete="name"
						placeholder="Ada Lovelace"
						class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
					/>
				</div>

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
						minlength={8}
						autocomplete="new-password"
						placeholder="Min. 8 characters"
						class="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
					/>
				</div>

				<button
					type="submit"
					disabled={loading}
					class="w-full py-2.5 bg-blue-600 text-white font-medium rounded-lg hover:bg-blue-700 transition disabled:opacity-50"
				>
					{loading ? 'Creating account…' : 'Sign Up'}
				</button>
			</form>

			<p class="mt-5 text-center text-sm text-gray-500">
				Already have an account?
				<a href="/login" class="text-blue-600 font-medium hover:underline">Sign in</a>
			</p>
		</div>
	</div>
</div>
