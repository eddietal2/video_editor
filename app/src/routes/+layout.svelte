<script lang="ts">
	import favicon from '$lib/assets/favicon.svg';
	import '../app.css';
	import SideNav from '$lib/components/SideNav.svelte';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<div class="bg-slate-950 min-h-screen">
	<SideNav />

	<main class="ml-64 min-h-screen bg-slate-950">
		{@render children()}
	</main>
</div>

<style global>
	:global(body) {
		margin: 0;
		padding: 0;
	}
</style>
