<script lang="ts">
	import { onMount } from 'svelte';

	import WinterNight from '#lib/components/3d/winter/WinterNight.svelte';
	import Navbar from '#lib/components/portfolio/Navbar.svelte';
	import Home from '#lib/components/portfolio/Home.svelte';
	import About from '#lib/components/portfolio/About.svelte';
	import Experience from '#lib/components/portfolio/Experience.svelte';
	import Contact from '#lib/components/portfolio/Contact.svelte';

	let theme = $state<'dark' | 'light'>('dark');
	let activeSection = $state('home');

	let viewport: HTMLElement;

	function toggleTheme() {
		theme = theme === 'dark' ? 'light' : 'dark';
	}

	function goToSection(id: string) {
		viewport?.querySelector<HTMLElement>(`#${id}`)?.scrollIntoView({
			behavior: 'smooth',
			block: 'nearest',
			inline: 'start'
		});
	}

	onMount(() => {
		if (!viewport) return;

		const observer = new IntersectionObserver(
			(entries) => {
				for (const entry of entries) {
					if (entry.isIntersecting && entry.intersectionRatio >= 0.5) {
						activeSection = entry.target.id;
					}
				}
			},
			{
				root: viewport,
				threshold: [0.5, 0.75]
			}
		);

		viewport.querySelectorAll('.panel[id]').forEach((section) => {
			observer.observe(section);
		});

		function handleWheel(event: WheelEvent) {
			if (event.ctrlKey || window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
				return;
			}

			// Jika gerakan dominan vertikal, ubah menjadi scroll horizontal.
			if (Math.abs(event.deltaY) > Math.abs(event.deltaX)) {
				event.preventDefault();
				viewport.scrollLeft += event.deltaY;
			}
		}

		viewport.addEventListener('wheel', handleWheel, { passive: false });

		return () => {
			observer.disconnect();
			viewport.removeEventListener('wheel', handleWheel);
		};
	});
</script>

<svelte:head>
	<title>Portfolio - Rizal Sihombing</title>
	<meta name="description" content="Portfolio of Rizal Sihombing, Full-Stack Developer." />
</svelte:head>

<main class="portfolio relative h-dvh w-full overflow-hidden">
	<div class="scene-background absolute inset-0">
		{#if theme === 'dark'}
			<WinterNight />
		{:else}
			<div class="winter-day-placeholder"></div>
		{/if}
	</div>

	<Navbar {activeSection} onNavigate={goToSection} {theme} onToggleTheme={toggleTheme} />

	<div bind:this={viewport} class="horizontal-viewport">
		<div id="home" class="panel">
			<Home />
		</div>

		<div id="about" class="panel">
			<About />
		</div>

		<div id="experience" class="panel">
			<Experience />
		</div>

		<div id="contact" class="panel">
			<Contact />
		</div>
	</div>
</main>

<style lang="scss">
	.portfolio {
		color: #f5f0e8;
		background: #101114;
	}

	.scene-background {
		z-index: 0;
	}

	.winter-day-placeholder {
		width: 100%;
		height: 100%;
		background: linear-gradient(#c9d9e8, #f5e8c9);
	}

	.horizontal-viewport {
		position: relative;
		z-index: 1;
		display: flex;
		width: 100%;
		height: 100%;
		overflow-x: auto;
		overflow-y: hidden;
		scrollbar-width: none;
		overscroll-behavior: contain;

		&::-webkit-scrollbar {
			display: none;
		}
	}

	.panel {
		position: relative;
		display: flex;
		flex: 0 0 100%;
		width: 100%;
		height: 100%;
		min-width: 0;
		box-sizing: border-box;
		align-items: center;
		padding: 120px 13.3% 80px;
	}

	@media (max-width: 700px) {
		.panel {
			padding: 110px 8% 70px;
		}
	}
</style>
