<script lang="ts">
	import type { LayoutProps } from './$types';
	import './globals.css';

	let { children }: LayoutProps = $props();
</script>

<svelte:head>
	<title>Tiket Creator Machine</title>
</svelte:head>

{@render children()}
