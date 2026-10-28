<script lang="ts">
	import { onMount } from 'svelte';
	import Lenis from 'lenis';
	import 'lenis/dist/lenis.css';

	import WinterNight from '#lib/components/3d/winter/WinterNight.svelte';
	import WinterDay from '#lib/components/3d/winter/WinterDay.svelte';

	import Navbar from '#lib/components/portfolio/Navbar.svelte';
	import Home from '#lib/components/portfolio/Home.svelte';
	import About from '#lib/components/portfolio/About.svelte';
	import Experience from '#lib/components/portfolio/Experience.svelte';
	import Contact from '#lib/components/portfolio/Contact.svelte';

	let theme = $state<'dark' | 'light'>('dark');
	let activeSection = $state('home');
	let cameraProgress = $state(0);

	let lenis: Lenis | undefined;
	let viewport: HTMLElement;
	let isNavigating = false;
	let touchStartY = 0;
	let touchStartX = 0;
	let touchStartContent: HTMLElement | null = null;

	const TOUCH_THRESHOLD = 50;

	function toggleTheme() {
		theme = theme === 'dark' ? 'light' : 'dark';
	}

	function goToSection(id: string) {
		const target = viewport?.querySelector<HTMLElement>(`#${id}`);

		if (!target || !lenis || !viewport || isNavigating) return;
		if (id === activeSection) return;

		isNavigating = true;
		activeSection = id;

		lenis.scrollTo(target, {
			offset: 0,
			duration: 2.5,
			easing: (t: number) => {
				return t < 0.5 ? 8 * t * t * t * t : 1 - Math.pow(-2 * t + 2, 4) / 2;
			},
			onComplete: () => {
				activeSection = id;
				isNavigating = false;
			}
		});
	}

	function handleWheel(event: WheelEvent) {
		if (!viewport || !lenis) return;

		// Kita hanya menangani scroll vertikal pada mouse wheel.
		if (event.deltaY === 0) return;

		const content = (event.target as HTMLElement).closest<HTMLElement>('.panel-content');

		if (!content) return;

		const canScrollVertically = content.scrollHeight > content.clientHeight;

		if (!canScrollVertically) {
			event.preventDefault();
			navigateByWheel(event.deltaY);
			return;
		}

		const atTop = content.scrollTop <= 0;

		const atBottom = content.scrollTop + content.clientHeight >= content.scrollHeight - 1;

		// Masih ada konten yang bisa dibaca ke arah scroll.
		if (event.deltaY < 0 && !atTop) return;
		if (event.deltaY > 0 && !atBottom) return;

		// Sudah mencapai batas konten: pindah ke section berikutnya.
		event.preventDefault();
		navigateByWheel(event.deltaY);
	}

	function navigateByWheel(deltaY: number) {
		if (!viewport || !lenis || isNavigating) return;

		const panels = Array.from(viewport.querySelectorAll<HTMLElement>('.panel[id]'));

		const currentIndex = panels.findIndex((panel) => panel.id === activeSection);

		if (currentIndex === -1) return;

		const nextIndex = deltaY > 0 ? currentIndex + 1 : currentIndex - 1;

		const nextPanel = panels[nextIndex];

		if (!nextPanel) return;

		goToSection(nextPanel.id);
	}

	function handleTouchStart(event: TouchEvent) {
		if (event.touches.length !== 1) {
			touchStartContent = null;
			return;
		}

		const target = event.target;

		touchStartContent =
			target instanceof HTMLElement ? target.closest<HTMLElement>('.panel-content') : null;

		touchStartY = event.touches[0].clientY;
		touchStartX = event.touches[0].clientX;
	}

	function handleTouchEnd(event: TouchEvent) {
		if (!viewport || !lenis || isNavigating) return;

		if (event.changedTouches.length !== 1) return;

		const content = touchStartContent;
		touchStartContent = null;

		if (!content || !content.isConnected) return;

		const touchEndY = event.changedTouches[0].clientY;
		const touchEndX = event.changedTouches[0].clientX;

		const deltaY = touchStartY - touchEndY;
		const deltaX = touchStartX - touchEndX;

		if (Math.abs(deltaY) < TOUCH_THRESHOLD) return;
		if (Math.abs(deltaX) > Math.abs(deltaY)) return;

		const panel = content.closest<HTMLElement>('.panel[id]');

		if (!panel || panel.id !== activeSection) return;

		const atTop = content.scrollTop <= 1;

		const atBottom = content.scrollTop + content.clientHeight >= content.scrollHeight - 2;

		if (deltaY > 0 && atBottom) {
			navigateByWheel(1);
		} else if (deltaY < 0 && atTop) {
			navigateByWheel(-1);
		}
	}

	onMount(() => {
		if (!viewport) return;

		// LENIS
		const track = viewport.querySelector<HTMLElement>('.horizontal-track');

		if (!track) return;

		lenis = new Lenis({
			wrapper: viewport,
			content: track,
			orientation: 'horizontal',
			gestureOrientation: 'both',
			smoothWheel: true,
			syncTouch: false,
			lerp: 0.1,
			wheelMultiplier: 1,
			prevent: (node) => node.closest('.panel-content') !== null
		});

		let animationFrame = 0;

		function raf(time: number) {
			lenis?.raf(time);
			animationFrame = requestAnimationFrame(raf);
		}

		animationFrame = requestAnimationFrame(raf);

		lenis.on('scroll', ({ scroll, limit }) => {
			cameraProgress = limit > 0 ? scroll / limit : 0;
		});

		// OBSERVER
		const observer = new IntersectionObserver(
			(entries) => {
				if (isNavigating) return;

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

		viewport.addEventListener('wheel', handleWheel, {
			passive: false
		});

		viewport.addEventListener('touchstart', handleTouchStart, {
			passive: true
		});

		viewport.addEventListener('touchend', handleTouchEnd, {
			passive: true
		});

		return () => {
			viewport.removeEventListener('wheel', handleWheel);
			viewport.removeEventListener('touchstart', handleTouchStart);
			viewport.removeEventListener('touchend', handleTouchEnd);
			cancelAnimationFrame(animationFrame);
			lenis?.destroy();
			observer.disconnect();
		};
	});
</script>

<svelte:head>
	<title>Portfolio - Rizal Sihombing</title>
	<meta name="description" content="Portfolio of Rizal Sihombing, Full-Stack Developer." />
</svelte:head>

<main class="relative h-dvh w-full overflow-hidden" data-theme={theme}>
	<div class="scene-background absolute inset-0">
		{#if theme === 'dark'}
			<WinterNight progress={cameraProgress} />
		{:else}
			<WinterDay progress={cameraProgress} />
		{/if}
	</div>

	<Navbar {activeSection} onNavigate={goToSection} {theme} onToggleTheme={toggleTheme} />

	<div bind:this={viewport} class="horizontal-viewport">
		<div class="horizontal-track">
			<div id="home" class="panel">
				<div class="panel-content">
					<Home />
				</div>
			</div>

			<div id="about" class="panel">
				<div class="panel-content">
					<About />
				</div>
			</div>

			<div id="experience" class="panel">
				<div class="panel-content">
					<Experience />
				</div>
			</div>

			<div id="contact" class="panel">
				<div class="panel-content">
					<Contact />
				</div>
			</div>
		</div>
	</div>
</main>

<style lang="scss">
	.scene-background {
		z-index: 0;
	}

	.horizontal-viewport {
		position: relative;
		/* z-index: 1; */
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
		flex: 0 0 100vw;
		width: 100vw;
		height: 100%;
		min-width: 0;
		overflow: hidden;
		box-sizing: border-box;
	}

	.panel-content {
		width: 100%;
		height: 100%;
		overflow-x: hidden;
		overflow-y: auto;
		box-sizing: border-box;
		padding-block: 120px 80px;
		overscroll-behavior-y: contain;
		scrollbar-width: thin;
	}

	@media (max-width: 700px) {
		.panel-content {
			padding-block: 110px 70px;
		}
	}
</style>
