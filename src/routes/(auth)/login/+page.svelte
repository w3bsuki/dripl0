<script lang="ts">
	import { enhance } from '$app/forms'
	import { goto } from '$app/navigation'
	import { auth } from '$lib/stores/auth'
	import { Eye, EyeOff, Github } from 'lucide-svelte'
	import { toast } from 'svelte-sonner'
	import * as m from '$lib/paraglide/messages.js'
	import Spinner from '$lib/components/ui/Spinner.svelte'

	let email = ''
	let password = ''
	let showPassword = false
	let rememberMe = false
	let loading = false

	async function handleLogin() {
		if (!email || !password) {
			toast.error(m.auth_fill_all_fields())
			return
		}

		loading = true
		try {
			await auth.signIn(email, password, rememberMe)
			toast.success(m.auth_welcome_back_toast())
			goto('/')
		} catch (error: any) {
			toast.error(error.message || m.auth_login_failed())
		} finally {
			loading = false
		}
	}

	async function handleOAuth(provider: 'google' | 'github') {
		loading = true
		try {
			await auth.signInWithProvider(provider)
		} catch (error: any) {
			toast.error(error.message || m.auth_oauth_failed())
			loading = false
		}
	}
</script>

<svelte:head>
	<title>Sign In | Driplo</title>
	<meta name="description" content="Sign in to your Driplo account" />
</svelte:head>

<div class="min-h-screen flex items-center justify-center bg-gray-50 px-4">
	<div class="w-full max-w-md">
		<div class="bg-white rounded-xl shadow-lg p-6">
			<!-- Logo -->
			<div class="text-center mb-6">
				<h1 class="text-3xl font-bold text-blue-400">Driplo</h1>
				<p class="text-gray-600 text-sm mt-1">Welcome back</p>
			</div>
			
			<!-- OAuth Buttons -->
			<div class="space-y-3 mb-6">
				<button
					class="w-full flex items-center justify-center gap-3 px-4 py-2.5 border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors text-sm font-medium"
					onclick={() => handleOAuth('google')}
					disabled={loading}
				>
					<svg class="w-5 h-5" viewBox="0 0 24 24">
						<path fill="#4285F4" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"/>
						<path fill="#34A853" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/>
						<path fill="#FBBC05" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"/>
						<path fill="#EA4335" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/>
					</svg>
					Continue with Google
				</button>
				<button
					class="w-full flex items-center justify-center gap-3 px-4 py-2.5 border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors text-sm font-medium"
					onclick={() => handleOAuth('github')}
					disabled={loading}
				>
					<Github class="w-4 h-4" />
					Continue with GitHub
				</button>
			</div>

			<!-- Divider -->
			<div class="relative mb-6">
				<div class="absolute inset-0 flex items-center">
					<div class="w-full border-t border-gray-200"></div>
				</div>
				<div class="relative flex justify-center text-xs">
					<span class="px-3 bg-white text-gray-500">Or</span>
				</div>
			</div>

			<!-- Login Form -->
			<form onsubmit={(e) => { e.preventDefault(); handleLogin(); }} class="space-y-4">
				<div>
					<label for="email" class="block text-sm font-medium text-gray-700 mb-1">
						Email
					</label>
					<input
						id="email"
						type="email"
						bind:value={email}
						placeholder="Enter your email"
						required
						disabled={loading}
						class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400 focus:border-blue-400 text-sm"
					/>
				</div>

				<div>
					<label for="password" class="block text-sm font-medium text-gray-700 mb-1">
						Password
					</label>
					<div class="relative">
						<input
							id="password"
							type={showPassword ? 'text' : 'password'}
							bind:value={password}
							placeholder="Enter your password"
							required
							disabled={loading}
							class="w-full px-3 py-2 pr-10 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-400 focus:border-blue-400 text-sm"
						/>
						<button
							type="button"
							class="absolute inset-y-0 right-0 pr-3 flex items-center"
							onclick={() => showPassword = !showPassword}
						>
							{#if showPassword}
								<EyeOff class="h-4 w-4 text-gray-400" />
							{:else}
								<Eye class="h-4 w-4 text-gray-400" />
							{/if}
						</button>
					</div>
				</div>

				<div class="flex items-center justify-between text-sm">
					<label class="flex items-center">
						<input 
							type="checkbox" 
							bind:checked={rememberMe}
							class="rounded border-gray-300 text-blue-400 focus:ring-blue-400" 
						/>
						<span class="ml-2 text-gray-600">Remember me for 30 days</span>
					</label>
					<a href="/forgot-password" class="text-blue-400 hover:text-blue-500">
						Forgot password?
					</a>
				</div>

				<button 
					type="submit" 
					class="w-full py-2.5 bg-blue-400 text-white font-medium rounded-lg hover:bg-blue-500 transition-colors disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center"
					disabled={loading}
				>
					{#if loading}
						<Spinner size="sm" color="white" />
					{:else}
						Sign in
					{/if}
				</button>
			</form>

			<!-- Sign up link -->
			<p class="text-center text-sm text-gray-600 mt-6">
				Don't have an account?
				<a href="/register" class="text-blue-400 hover:text-blue-500 font-medium">
					Sign up
				</a>
			</p>
		</div>
	</div>
</div>