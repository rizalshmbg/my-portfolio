<script lang="ts">
	import WinterNight from '#lib/components/3d/winter/WinterNight.svelte';
	import WinterDay from '#lib/components/3d/winter/WinterDay.svelte';

	let theme = $state<'dark' | 'light'>('dark');
</script>

<svelte:head>
	<title>Portfolio - Rizal Sihombing</title>
	<meta name="description" content="Portfolio of Rizal Sihombing, Full-Stack Developer." />
</svelte:head>

<main class="world" data-theme={theme}>
	<div class="world-scene">
		{#if theme === 'dark'}
			<WinterNight
				onThemeChange={(nextTheme) => {
					theme = nextTheme;
				}}
			/>
		{:else}
			<WinterDay
				onThemeChange={(nextTheme) => {
					theme = nextTheme;
				}}
			/>
		{/if}
	</div>
</main>

<style lang="scss">
	.world {
		position: relative;
		width: 100%;
		height: 100vh;
		height: 100dvh;
		overflow: hidden;
		overscroll-behavior: none;
	}

	.world-scene {
		position: absolute;
		inset: 0;
		width: 100%;
		height: 100%;
		overflow: hidden;
	}
</style>
