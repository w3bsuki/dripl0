<script lang="ts">
	import { cn } from '$lib/utils'
	
	interface Props {
		class?: string
		type?: string
		value?: string | number
		placeholder?: string
		disabled?: boolean
		required?: boolean
		id?: string
		onblur?: (e: FocusEvent) => void
		onchange?: (e: Event) => void
		onclick?: (e: MouseEvent) => void
		onfocus?: (e: FocusEvent) => void
		onkeydown?: (e: KeyboardEvent) => void
		onkeypress?: (e: KeyboardEvent) => void
		onkeyup?: (e: KeyboardEvent) => void
		onmouseover?: (e: MouseEvent) => void
		onmouseenter?: (e: MouseEvent) => void
		onmouseleave?: (e: MouseEvent) => void
		onpaste?: (e: ClipboardEvent) => void
		oninput?: (e: Event) => void
		[key: string]: any
	}
	
	let {
		class: className,
		type = 'text',
		value = $bindable(),
		placeholder,
		disabled = false,
		required = false,
		id,
		onblur,
		onchange,
		onclick,
		onfocus,
		onkeydown,
		onkeypress,
		onkeyup,
		onmouseover,
		onmouseenter,
		onmouseleave,
		onpaste,
		oninput,
		...restProps
	}: Props = $props()
</script>

<input
	{id}
	{type}
	{placeholder}
	{disabled}
	{required}
	bind:value
	class={cn(
		'flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-base ring-offset-background file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50 [font-size:16px] sm:text-sm',
		className
	)}
	{onblur}
	{onchange}
	{onclick}
	{onfocus}
	{onkeydown}
	{onkeypress}
	{onkeyup}
	{onmouseover}
	{onmouseenter}
	{onmouseleave}
	{onpaste}
	{oninput}
	{...restProps}
/>