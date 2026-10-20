<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import Moon from './Moon.svelte';
	import Stars from './Stars.svelte';
	import Clouds from './Clouds.svelte';
	import Hills from './Hills.svelte';
	import ForegroundHills from './ForegroundHills.svelte';
	import Trees from './Trees.svelte';

	let canvas = $state<HTMLCanvasElement | undefined>();
	let scene = $state<THREE.Scene | null>(null);

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
		renderer.shadowMap.type = THREE.PCFShadowMap;
		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5));
		renderer.setSize(currentCanvas.clientWidth, currentCanvas.clientHeight, false);
		renderer.outputColorSpace = THREE.SRGBColorSpace;

		const ambientLight = new THREE.AmbientLight('#cbd5e1', 1.5);
		scene.add(ambientLight);

		const directionalLight = new THREE.DirectionalLight('#f5ead5', 1.2);

		directionalLight.position.set(10, 7, -20);
		directionalLight.target.position.set(0, 0, 0);
		directionalLight.castShadow = true;

		directionalLight.shadow.mapSize.set(1024, 1024);
		directionalLight.shadow.camera.near = 0.5;
		directionalLight.shadow.camera.far = 50;

		directionalLight.shadow.camera.left = -30;
		directionalLight.shadow.camera.right = 30;
		directionalLight.shadow.camera.top = 30;
		directionalLight.shadow.camera.bottom = -30;

		directionalLight.shadow.bias = -0.0005;
		directionalLight.shadow.normalBias = 0.03;

		scene.add(directionalLight);

		let animationFrame = 0;

		function animate() {
			animationFrame = requestAnimationFrame(animate);
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
		<Stars {scene} />
		<Clouds {scene} />
		<Hills {scene} />
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
