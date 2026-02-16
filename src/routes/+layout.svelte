<script lang="ts">
	import './layout.css';
	import favicon from '$lib/assets/pub.svg';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<meta property="og:image" content="/dinner.jpg" />
	<meta name="twitter:card" content="summary_large_image" />
	<meta name="twitter:image" content="/dinner.jpg" />
</svelte:head>
{@render children()}
