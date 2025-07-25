<script lang="ts">
	import { goto } from '$app/navigation'
	import { page } from '$app/stores'
	import { auth } from '$lib/stores/auth'
	import { Eye, EyeOff, Github, CheckCircle, Store, User } from 'lucide-svelte'
	import { toast } from 'svelte-sonner'
	import { z } from 'zod'
	import * as m from '$lib/paraglide/messages.js'
	import Spinner from '$lib/components/ui/Spinner.svelte'
	import { cn } from '$lib/utils'

	let email = $state('')
	let password = $state('')
	let confirmPassword = $state('')
	let username = $state('')
	let fullName = $state('')
	let showPassword = $state(false)
	let showConfirmPassword = $state(false)
	let loading = $state(false)
	let agreedToTerms = $state(false)
	let accountType = $state<'personal' | 'brand'>('personal')
	
	// Brand-specific fields
	let brandName = $state('')
	let brandCategory = $state('')
	let brandWebsite = $state('')
	
	// Check if showing success message
	let showSuccess = $derived($page.url.searchParams.get('success') === 'true')

	const registerSchema = z.object({
		email: z.string().email('Please enter a valid email address'),
		password: z.string().min(8, 'Password must be at least 8 characters'),
		confirmPassword: z.string(),
		username: z.string()
			.min(3, 'Username must be at least 3 characters')
			.max(30, 'Username must be less than 30 characters')
			.regex(/^[a-zA-Z0-9_]+$/, 'Username can only contain letters, numbers, and underscores'),
		fullName: z.string().optional(),
		agreedToTerms: z.boolean().refine(val => val === true, 'You must agree to the terms of service'),
		accountType: z.enum(['personal', 'brand']),
		brandName: z.string().optional(),
		brandCategory: z.string().optional(),
		brandWebsite: z.string().url().optional().or(z.literal(''))
	}).refine(data => data.password === data.confirmPassword, {
		message: "Passwords don't match",
		path: ["confirmPassword"]
	}).refine(data => {
		if (data.accountType === 'brand') {
			return data.brandName && data.brandName.length >= 2
		}
		return true
	}, {
		message: "Brand name is required for brand accounts",
		path: ["brandName"]
	})

	async function handleRegister() {
		try {
			const validatedData = registerSchema.parse({
				email,
				password,
				confirmPassword,
				username,
				fullName: fullName || undefined,
				agreedToTerms,
				accountType,
				brandName: brandName || undefined,
				brandCategory: brandCategory || undefined,
				brandWebsite: brandWebsite || undefined
			})

			loading = true
			
			// Sign up with additional metadata
			await auth.signUp(email, password, username, fullName || undefined, {
				account_type: accountType,
				brand_name: accountType === 'brand' ? brandName : undefined,
				brand_category: accountType === 'brand' ? brandCategory : undefined,
				brand_website: accountType === 'brand' ? brandWebsite : undefined
			})
			
			// Show success message
			toast.success('Account created! Please check your email to verify your account.')
			goto('/register?success=true')
		} catch (error: any) {
			if (error.issues) {
				// Zod validation errors
				error.issues.forEach((issue: any) => {
					toast.error(issue.message)
				})
			} else {
				toast.error(error.message || 'Registration failed')
			}
		} finally {
			loading = false
		}
	}

	async function handleOAuth(provider: 'google' | 'github') {
		loading = true
		try {
			// Store account type preference in localStorage for OAuth callback
			if (accountType === 'brand') {
				localStorage.setItem('pending_account_type', 'brand')
			}
			await auth.signInWithProvider(provider)
		} catch (error: any) {
			toast.error(error.message || 'OAuth registration failed')
			loading = false
		}
	}
</script>

<svelte:head>
	<title>Sign Up | Driplo</title>
	<meta name="description" content="Create your Driplo account - Personal or Brand" />
</svelte:head>

<div class="min-h-screen flex items-center justify-center bg-gray-50 px-4 py-4">
	<div class="w-full max-w-md">
		{#if showSuccess}
			<!-- Success message after signup -->
			<div class="bg-white rounded-xl shadow-lg p-8 text-center">
				<div class="mx-auto w-16 h-16 bg-green-100 rounded-full flex items-center justify-center mb-4">
					<CheckCircle class="w-8 h-8 text-green-600" />
				</div>
				<h2 class="text-2xl font-bold text-gray-900 mb-2">Check your email!</h2>
				<p class="text-gray-600 mb-6">
					We've sent a verification link to your email address. 
					Click the link to activate your account and get started.
				</p>
				{#if accountType === 'brand'}
					<div class="bg-blue-50 border border-blue-200 rounded-lg p-4 mb-6">
						<p class="text-sm text-blue-700">
							<strong>Next steps for brand accounts:</strong> After verifying your email, 
							you'll need to complete brand verification to access all features.
						</p>
					</div>
				{:else}
					<div class="bg-blue-50 border border-blue-200 rounded-lg p-4 mb-6">
						<p class="text-sm text-blue-700">
							<strong>Tip:</strong> If you don't see the email, check your spam folder or wait a few minutes.
						</p>
					</div>
				{/if}
				<a href="/login" class="text-[#87CEEB] hover:text-[#87CEEB]/80 font-medium">
					Return to login
				</a>
			</div>
		{:else}
		<div class="bg-white rounded-xl shadow-lg p-6">
			<!-- Logo -->
			<div class="text-center mb-4">
				<h1 class="text-3xl font-bold text-[#87CEEB]">Driplo</h1>
				<p class="text-gray-600 text-sm mt-1">Create your account</p>
			</div>
			
			<!-- Account Type Selection -->
			<div class="mb-4">
				<label class="block text-xs font-medium text-gray-700 mb-2">
					Account Type
				</label>
				<div class="grid grid-cols-2 gap-2">
					<button
						type="button"
						onclick={() => accountType = 'personal'}
						class={cn(
							"relative p-3 rounded-lg border-2 transition-all",
							accountType === 'personal' 
								? "border-[#87CEEB] bg-[#87CEEB]/5" 
								: "border-gray-200 hover:border-gray-300"
						)}
					>
						<div class="text-center">
							<User class={cn("w-6 h-6 mx-auto mb-1", 
								accountType === 'personal' ? "text-[#87CEEB]" : "text-gray-400"
							)} />
							<div class="font-medium text-sm">Personal</div>
							<div class="text-xs text-gray-500">Buy & sell fashion</div>
						</div>
					</button>
					<button
						type="button"
						onclick={() => accountType = 'brand'}
						class={cn(
							"relative p-3 rounded-lg border-2 transition-all",
							accountType === 'brand' 
								? "border-[#87CEEB] bg-[#87CEEB]/5" 
								: "border-gray-200 hover:border-gray-300"
						)}
					>
						<div class="text-center">
							<Store class={cn("w-6 h-6 mx-auto mb-1", 
								accountType === 'brand' ? "text-[#87CEEB]" : "text-gray-400"
							)} />
							<div class="font-medium text-sm">Brand</div>
							<div class="text-xs text-gray-500">Sell as a business</div>
						</div>
						<div class="absolute -top-2 -right-2 bg-[#87CEEB] text-white text-[10px] px-1.5 py-0.5 rounded-full">
							Pro
						</div>
					</button>
				</div>
			</div>

			<!-- OAuth Buttons -->
			<div class="space-y-2 mb-4">
				<button
					class="w-full flex items-center justify-center gap-3 px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors text-sm font-medium"
					onclick={() => handleOAuth('google')}
					disabled={loading}
				>
					<svg class="w-4 h-4" viewBox="0 0 24 24">
						<path fill="#4285F4" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"/>
						<path fill="#34A853" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/>
						<path fill="#FBBC05" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"/>
						<path fill="#EA4335" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/>
					</svg>
					Continue with Google
				</button>
				<button
					class="w-full flex items-center justify-center gap-3 px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors text-sm font-medium"
					onclick={() => handleOAuth('github')}
					disabled={loading}
				>
					<Github class="w-4 h-4" />
					Continue with GitHub
				</button>
			</div>

			<!-- Divider -->
			<div class="relative mb-4">
				<div class="absolute inset-0 flex items-center">
					<div class="w-full border-t border-gray-200"></div>
				</div>
				<div class="relative flex justify-center text-xs">
					<span class="px-3 bg-white text-gray-500">Or</span>
				</div>
			</div>

			<!-- Registration Form -->
			<form onsubmit={(e) => { e.preventDefault(); handleRegister(); }} class="space-y-3">
				<div class="grid grid-cols-2 gap-3">
					<div>
						<label for="fullName" class="block text-xs font-medium text-gray-700 mb-1">
							Full name
						</label>
						<input
							id="fullName"
							type="text"
							bind:value={fullName}
							placeholder="John Doe"
							disabled={loading}
							class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#87CEEB] focus:border-[#87CEEB] text-sm"
						/>
					</div>
					<div>
						<label for="username" class="block text-xs font-medium text-gray-700 mb-1">
							Username *
						</label>
						<input
							id="username"
							type="text"
							bind:value={username}
							placeholder="johndoe"
							required
							disabled={loading}
							class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#87CEEB] focus:border-[#87CEEB] text-sm"
						/>
					</div>
				</div>

				<div>
					<label for="email" class="block text-xs font-medium text-gray-700 mb-1">
						Email *
					</label>
					<input
						id="email"
						type="email"
						bind:value={email}
						placeholder="Enter your email"
						required
						disabled={loading}
						class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#87CEEB] focus:border-[#87CEEB] text-sm"
					/>
				</div>

				{#if accountType === 'brand'}
					<div class="space-y-3 p-3 bg-blue-50 rounded-lg border border-blue-100">
						<div>
							<label for="brandName" class="block text-xs font-medium text-gray-700 mb-1">
								Brand Name *
							</label>
							<input
								id="brandName"
								type="text"
								bind:value={brandName}
								placeholder="Your brand name"
								required
								disabled={loading}
								class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#87CEEB] focus:border-[#87CEEB] text-sm bg-white"
							/>
						</div>
						<div>
							<label for="brandCategory" class="block text-xs font-medium text-gray-700 mb-1">
								Brand Category
							</label>
							<select
								id="brandCategory"
								bind:value={brandCategory}
								disabled={loading}
								class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#87CEEB] focus:border-[#87CEEB] text-sm bg-white"
							>
								<option value="">Select category</option>
								<option value="fashion">Fashion & Apparel</option>
								<option value="accessories">Accessories</option>
								<option value="footwear">Footwear</option>
								<option value="jewelry">Jewelry</option>
								<option value="bags">Bags & Luggage</option>
								<option value="beauty">Beauty & Cosmetics</option>
								<option value="vintage">Vintage & Thrift</option>
								<option value="handmade">Handmade & Artisan</option>
								<option value="sustainable">Sustainable Fashion</option>
								<option value="other">Other</option>
							</select>
						</div>
						<div>
							<label for="brandWebsite" class="block text-xs font-medium text-gray-700 mb-1">
								Website (optional)
							</label>
							<input
								id="brandWebsite"
								type="url"
								bind:value={brandWebsite}
								placeholder="https://yourbrand.com"
								disabled={loading}
								class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#87CEEB] focus:border-[#87CEEB] text-sm bg-white"
							/>
						</div>
						<p class="text-xs text-blue-700">
							You'll need to verify your brand after registration to access all features.
						</p>
					</div>
				{/if}

				<div>
					<label for="password" class="block text-xs font-medium text-gray-700 mb-1">
						Password *
					</label>
					<div class="relative">
						<input
							id="password"
							type={showPassword ? 'text' : 'password'}
							bind:value={password}
							placeholder="Min 8 characters"
							required
							disabled={loading}
							class="w-full px-3 py-2 pr-10 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#87CEEB] focus:border-[#87CEEB] text-sm"
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

				<div>
					<label for="confirmPassword" class="block text-xs font-medium text-gray-700 mb-1">
						Confirm password *
					</label>
					<div class="relative">
						<input
							id="confirmPassword"
							type={showConfirmPassword ? 'text' : 'password'}
							bind:value={confirmPassword}
							placeholder="Confirm password"
							required
							disabled={loading}
							class="w-full px-3 py-2 pr-10 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#87CEEB] focus:border-[#87CEEB] text-sm"
						/>
						<button
							type="button"
							class="absolute inset-y-0 right-0 pr-3 flex items-center"
							onclick={() => showConfirmPassword = !showConfirmPassword}
						>
							{#if showConfirmPassword}
								<EyeOff class="h-4 w-4 text-gray-400" />
							{:else}
								<Eye class="h-4 w-4 text-gray-400" />
							{/if}
						</button>
					</div>
				</div>

				<div class="flex items-start">
					<input
						type="checkbox"
						id="terms"
						bind:checked={agreedToTerms}
						class="mt-0.5 rounded border-gray-300 text-[#87CEEB] focus:ring-2 focus:ring-[#87CEEB]/30"
						required
					/>
					<label for="terms" class="ml-2 text-xs text-gray-600">
						I agree to the
						<a href="/terms" class="text-[#87CEEB] hover:text-[#87CEEB]/80">Terms of Service</a>
						and
						<a href="/privacy" class="text-[#87CEEB] hover:text-[#87CEEB]/80">Privacy Policy</a>
					</label>
				</div>

				<button 
					type="submit" 
					class="w-full py-2.5 bg-[#87CEEB] text-white font-medium rounded-lg hover:bg-[#87CEEB]/90 transition-colors disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center"
					disabled={loading || !agreedToTerms}
				>
					{#if loading}
						<Spinner size="sm" color="white" />
					{:else}
						Sign up {accountType === 'brand' ? 'as Brand' : ''}
					{/if}
				</button>
			</form>
			

			<!-- Sign in link -->
			<p class="text-center text-sm text-gray-600 mt-4">
				Already have an account?
				<a href="/login" class="text-[#87CEEB] hover:text-[#87CEEB]/80 font-medium">
					Sign in
				</a>
			</p>
		</div>
		{/if}
	</div>
</div>