<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import Moon from './night-scene/Moon.svelte';
	import Stars from './night-scene/Stars.svelte';
	import Clouds from './night-scene/Clouds.svelte';
	import Hills from './night-scene/Hills.svelte';
	import MountainFog from './night-scene/MountainFog.svelte';
	import Cabin from './night-scene/Cabin.svelte';
	import CabinPath from './night-scene/CabinPath.svelte';
	import SnowTerrain from './night-scene/SnowTerrain.svelte';
	import SnowRocks from './night-scene/SnowRocks.svelte';
	import WinterGrass from './night-scene/WinterGrass.svelte';
	import Trees from './night-scene/Trees.svelte';
	import Wolf from './night-scene/Wolf.svelte';
	import Snowfall from './night-scene/Snowfall.svelte';

	let canvas = $state<HTMLCanvasElement | undefined>();
	let scene = $state<THREE.Scene | null>(null);

	let updateStars: ((time: number) => void) | undefined;
	let updateClouds: ((time: number) => void) | undefined;
	let updateWolf: ((time: number) => void) | undefined;
	let updateSnowfall: ((time: number) => void) | undefined;

	onMount(() => {
		if (!canvas) return;

		const currentCanvas = canvas;

		scene = new THREE.Scene();
		const currentScene = scene;
		currentScene.background = new THREE.Color('#17191d');

		const camera = new THREE.PerspectiveCamera(
			45,
			currentCanvas.clientWidth / currentCanvas.clientHeight,
			0.1,
			100
		);

		camera.position.set(0, 2, 8);
		camera.lookAt(0, 4, -10);

		const cameraBasePosition = camera.position.clone();
		const cameraBaseTarget = new THREE.Vector3(0, 4, -10);

		const mouse = new THREE.Vector2(0, 0);
		const smoothMouse = new THREE.Vector2(0, 0);
		const cameraTargetPosition = new THREE.Vector3();
		const lookTarget = new THREE.Vector3();

		const renderer = new THREE.WebGLRenderer({
			canvas: currentCanvas,
			antialias: true
		});

		renderer.shadowMap.enabled = true;
		renderer.shadowMap.type = THREE.PCFShadowMap;
		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5));
		renderer.setSize(currentCanvas.clientWidth, currentCanvas.clientHeight, false);
		renderer.outputColorSpace = THREE.SRGBColorSpace;

		const ambientLight = new THREE.AmbientLight('#a0abc0', 1);
		currentScene.add(ambientLight);

		const directionalLight = new THREE.DirectionalLight('#f5dfbd', 1);

		directionalLight.position.set(-8, 20, -50);
		directionalLight.target.position.set(0, 0, -10);

		directionalLight.castShadow = true;

		directionalLight.shadow.mapSize.set(2048, 2048);
		directionalLight.shadow.camera.near = 0.5;
		directionalLight.shadow.camera.far = 100;

		directionalLight.shadow.camera.left = -35;
		directionalLight.shadow.camera.right = 35;
		directionalLight.shadow.camera.top = 35;
		directionalLight.shadow.camera.bottom = -35;

		directionalLight.shadow.bias = -0.0001;
		directionalLight.shadow.normalBias = 0.04;

		currentScene.add(directionalLight);
		currentScene.add(directionalLight.target);

		// Fill light dingin untuk membantu menampilkan detail salju
		// dan siluet serigala tanpa menambah bayangan baru.
		const snowFillLight = new THREE.PointLight('#9eafcf', 12, 28, 2);

		snowFillLight.position.set(8, 5, -20);
		snowFillLight.castShadow = false;

		currentScene.add(snowFillLight);

		let animationFrame = 0;

		const clock = new THREE.Clock();

		function handleMouseMove(event: MouseEvent) {
			const rect = currentCanvas.getBoundingClientRect();

			if (!rect.width || !rect.height) return;

			const normalizedX = ((event.clientX - rect.left) / rect.width) * 2 - 1;
			const normalizedY = -(((event.clientY - rect.top) / rect.height) * 2 - 1);

			mouse.set(
				THREE.MathUtils.clamp(normalizedX, -1, 1),
				THREE.MathUtils.clamp(normalizedY, -1, 1)
			);
		}

		window.addEventListener('mousemove', handleMouseMove);

		function animate() {
			animationFrame = requestAnimationFrame(animate);

			const delta = clock.getDelta();
			const time = clock.elapsedTime;

			updateStars?.(time);
			updateClouds?.(time);
			updateWolf?.(time);
			updateSnowfall?.(time * 1000);

			// Smoothing berbasis waktu agar tidak bergantung pada FPS.
			const smoothing = 1 - Math.exp(-5 * delta);

			// Mouse bergerak menuju input terbaru secara bertahap.
			smoothMouse.lerp(mouse, smoothing);

			// Posisi dasar kamera + offset dari mouse.
			cameraTargetPosition.set(
				cameraBasePosition.x + smoothMouse.x * 1.1,
				cameraBasePosition.y + smoothMouse.y * 0.65,
				cameraBasePosition.z
			);

			// Arah pandang kamera juga mengikuti mouse secara halus.
			lookTarget.set(
				cameraBaseTarget.x + smoothMouse.x * 0.6,
				cameraBaseTarget.y + smoothMouse.y * 0.4,
				cameraBaseTarget.z
			);

			camera.position.lerp(cameraTargetPosition, smoothing);
			camera.lookAt(lookTarget);

			renderer.render(currentScene, camera);
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
			window.removeEventListener('mousemove', handleMouseMove);
			currentScene.remove(snowFillLight);
			snowFillLight.dispose();

			currentScene.remove(directionalLight);
			currentScene.remove(directionalLight.target);
			currentScene.remove(ambientLight);

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
		<Cabin {scene} />
		<CabinPath {scene} />
		<SnowTerrain {scene} />
		<SnowRocks {scene} />
		<WinterGrass {scene} />
		<Trees {scene} />
		<Wolf
			{scene}
			registerUpdate={(callback) => {
				updateWolf = callback;
			}}
		/>
		<Snowfall {scene} registerUpdate={(callback) => (updateSnowfall = callback)} />
	{/if}
</div>

<style lang="scss">
	.scene {
		width: 100%;
		height: 100vh;
		/* min-height: 600px; */
		overflow: hidden;
		background: #17191d;

		canvas {
			display: block;
			width: 100%;
			height: 100%;
		}
	}
</style>
