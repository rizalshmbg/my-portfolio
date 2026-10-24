<script lang="ts">
	import { onMount } from 'svelte';
	import Lenis from 'lenis';
	import 'lenis/dist/lenis.css';

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

		const track = viewport.querySelector<HTMLElement>('.horizontal-track');

		if (!track) return;

		const lenis = new Lenis({
			wrapper: viewport,
			content: track,
			orientation: 'horizontal',
			gestureOrientation: 'both',
			smoothWheel: true,
			syncTouch: false,
			lerp: 0.1,
			wheelMultiplier: 1
		});

		let animationFrame = 0;

		function raf(time: number) {
			lenis.raf(time);
			animationFrame = requestAnimationFrame(raf);
		}

		animationFrame = requestAnimationFrame(raf);

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

		return () => {
			cancelAnimationFrame(animationFrame);
			lenis.destroy();
			observer.disconnect();
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
		<div class="horizontal-track">
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
		width: 100%;
		height: 100%;
		overflow: hidden;
		overscroll-behavior: contain;
	}

	.horizontal-track {
		display: flex;
		width: max-content;
		height: 100%;
	}

	.panel {
		position: relative;
		display: flex;
		flex: 0 0 100vw;
		width: 100vw;
		height: 100%;
		min-width: 0;
		box-sizing: border-box;
		align-items: center;
		padding: 120px 4rem 80px;
	}

	@media (max-width: 700px) {
		.panel {
			padding: 110px 8% 70px;
		}
	}
</style>
