<script lang="ts">
	import { getContext } from 'svelte';
	import { cn } from '$lib/utils';
	import type { Snippet } from 'svelte';
	import type { Writable } from 'svelte/store';

	interface Props {
		value: string;
		class?: string;
		children: Snippet;
	}

	let { value, class: className, children }: Props = $props();

	const context = getContext<{
		activeTab: Writable<string>;
		setActiveTab: (tab: string) => void;
	}>('tabs');

	const { activeTab, setActiveTab } = context;
	
	const isActive = $derived($activeTab === value);
</script>

<button
	type="button"
	onclick={() => setActiveTab(value)}
	class={cn(
		'inline-flex items-center justify-center whitespace-nowrap rounded-sm px-3 py-1.5 text-sm font-medium ring-offset-background transition-all focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
		isActive
			? 'bg-background text-foreground shadow-sm'
			: 'text-muted-foreground hover:text-foreground',
		className
	)}
>
	{@render children()}
</button>