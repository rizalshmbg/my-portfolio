<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import Moon from './Moon.svelte';
	import Stars from './Stars.svelte';
	import Clouds from './Clouds.svelte';
	import Hills from './Hills.svelte';
	import ForegroundHills from './ForegroundHills.svelte';
	import Trees from './Trees.svelte';
	import MountainFog from './MountainFog.svelte';

	let canvas = $state<HTMLCanvasElement | undefined>();
	let scene = $state<THREE.Scene | null>(null);

	let updateStars: ((time: number) => void) | undefined;
	let updateClouds: ((time: number) => void) | undefined;

	onMount(() => {
		if (!canvas) return;

		const currentCanvas = canvas;

		scene = new THREE.Scene();
		scene.background = new THREE.Color('#17191d');

		const camera = new THREE.PerspectiveCamera(
			45,
			currentCanvas.clientWidth / currentCanvas.clientHeight,
			0.1,
			100
		);

		camera.position.set(0, 2, 8);
		camera.lookAt(0, 4, -10);

		const renderer = new THREE.WebGLRenderer({
			canvas: currentCanvas,
			antialias: true
		});

		renderer.shadowMap.enabled = true;
		renderer.shadowMap.type = THREE.PCFSoftShadowMap;
		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5));
		renderer.setSize(currentCanvas.clientWidth, currentCanvas.clientHeight, false);
		renderer.outputColorSpace = THREE.SRGBColorSpace;

		const ambientLight = new THREE.AmbientLight('#8791a3', 0.65);
		scene.add(ambientLight);

		const directionalLight = new THREE.DirectionalLight('#f5dfbd', 2);

		directionalLight.position.set(10, 7, -20);
		directionalLight.target.position.set(0, 0, 0);

		directionalLight.castShadow = true;

		directionalLight.shadow.mapSize.set(2048, 2048);
		directionalLight.shadow.camera.near = 0.5;
		directionalLight.shadow.camera.far = 50;

		directionalLight.shadow.camera.left = -30;
		directionalLight.shadow.camera.right = 30;
		directionalLight.shadow.camera.top = 30;
		directionalLight.shadow.camera.bottom = -30;

		directionalLight.shadow.bias = -0.0001;
		directionalLight.shadow.normalBias = 0.04;

		scene.add(directionalLight);
		scene.add(directionalLight.target);

		let animationFrame = 0;

		const clock = new THREE.Clock();

		function animate() {
			animationFrame = requestAnimationFrame(animate);

			const time = clock.getElapsedTime();

			updateStars?.(time);
			updateClouds?.(time);

			renderer.render(scene!, camera);
		}

		animate();

		function resize() {
			const width = currentCanvas.clientWidth;
			const height = currentCanvas.clientHeight;

			if (!width || !height) return;

			camera.aspect = width / height;
			camera.updateProjectionMatrix();

			renderer.setSize(width, height, false);
		}

		window.addEventListener('resize', resize);

		return () => {
			cancelAnimationFrame(animationFrame);
			window.removeEventListener('resize', resize);
			renderer.dispose();

			scene = null;
		};
	});
</script>

<div class="scene">
	<canvas bind:this={canvas}></canvas>

	{#if scene}
		<Moon {scene} />
		<Stars {scene} registerUpdate={(callback) => (updateStars = callback)} />
		<Clouds {scene} registerUpdate={(callback) => (updateClouds = callback)} />
		<Hills {scene} />
		<MountainFog {scene} />
		<ForegroundHills {scene} />
		<Trees {scene} />
	{/if}
</div>

<style lang="scss">
	.scene {
		width: 100%;
		height: 100vh;
		min-height: 600px;
		overflow: hidden;
		background: #17191d;

		canvas {
			display: block;
			width: 100%;
			height: 100%;
		}
	}
</style>
