<script lang="ts">
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<style>
		@import url('https://fonts.googleapis.com/css2?family=EB+Garamond:ital,wght@0,400..800;1,400..800&family=Noto+Serif+KR:wght@200..900&display=swap');

		body {
			margin: 0;
			font-family: 'EB Garamond', 'Noto Serif KR', serif;
		}
	</style>
</svelte:head>

{@render children()}
