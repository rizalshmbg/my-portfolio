<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	import Sky from './day-scene/Sky.svelte';
	import Sun from './day-scene/Sun.svelte';
	import Clouds from './day-scene/Clouds.svelte';
	import Hills from './day-scene/Hills.svelte';
	import MountainFog from './day-scene/MountainFog.svelte';
	import Cabin from './day-scene/Cabin.svelte';
	import CabinPath from './shared-scene/CabinPath.svelte';
	import SnowTerrain from './day-scene/SnowTerrain.svelte';
	import SnowRocks from './day-scene/SnowRocks.svelte';
	import WinterGrass from './shared-scene/WinterGrass.svelte';
	import Trees from './day-scene/Trees.svelte';
	import Wolf from './day-scene/Wolf.svelte';
	import SnowTrees from './day-scene/SnowTrees.svelte';
	import Snowfall from './day-scene/Snowfall.svelte';

	let canvas = $state<HTMLCanvasElement | undefined>();
	let scene = $state<THREE.Scene | null>(null);

	let updateClouds: ((time: number) => void) | undefined;
	let updateWolf: ((time: number) => void) | undefined;
	let updateSnowfall: ((time: number) => void) | undefined;

	onMount(() => {
		if (!canvas) return;

		const currentCanvas = canvas;

		// 1. CREATE SCENE
		const currentScene = new THREE.Scene();
		currentScene.background = new THREE.Color('#83c5f7');
		currentScene.fog = new THREE.Fog('#83c5f7', 45, 100);

		scene = currentScene;

		// 2. CREATE CAMERA
		const camera = new THREE.PerspectiveCamera(
			45,
			currentCanvas.clientWidth / currentCanvas.clientHeight,
			0.1,
			100
		);

		const cameraBasePosition = new THREE.Vector3(0, 2, 8);
		const cameraBaseTarget = new THREE.Vector3(0, 4, -10);

		camera.position.copy(cameraBasePosition);
		camera.lookAt(cameraBaseTarget);

		// 3. CREATE PARALLAX MOUSE
		const mouse = new THREE.Vector2(0, 0);
		const smoothMouse = new THREE.Vector2(0, 0);
		const cameraTargetPosition = new THREE.Vector3();
		const lookTarget = new THREE.Vector3();

		// 4. CREATE RENDER CANVAS
		const renderer = new THREE.WebGLRenderer({
			canvas: currentCanvas,
			antialias: true
		});

		renderer.shadowMap.enabled = true;
		renderer.shadowMap.type = THREE.PCFShadowMap;
		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5));
		renderer.setSize(currentCanvas.clientWidth, currentCanvas.clientHeight, false);
		renderer.outputColorSpace = THREE.SRGBColorSpace;

		// 5. CREATE AMBIENT LIGHT
		const ambientLight = new THREE.AmbientLight('#b9cce3', 2);
		currentScene.add(ambientLight);

		// 6. CREATE SUN LIGHT
		const sunlight = new THREE.DirectionalLight('#dce8f5', 2);
		sunlight.position.set(-5, 20, -50);
		sunlight.target.position.set(0, 0, -10);

		sunlight.castShadow = true;

		sunlight.shadow.mapSize.set(2048, 2048);
		sunlight.shadow.camera.near = 0.5;
		sunlight.shadow.camera.far = 100;

		sunlight.shadow.camera.left = -35;
		sunlight.shadow.camera.right = 35;
		sunlight.shadow.camera.top = 35;
		sunlight.shadow.camera.bottom = -35;

		sunlight.shadow.bias = -0.0001;
		sunlight.shadow.normalBias = 0.04;

		currentScene.add(sunlight);
		currentScene.add(sunlight.target);

		// 7. CREATE HEMISPHERE LIGHT
		const hemisphereLight = new THREE.HemisphereLight('#c5d9f0', '#8495aa', 1.5);
		currentScene.add(hemisphereLight);

		// 8. ANIMATION AND RENDERING
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

			updateClouds?.(time);
			updateWolf?.(time);
			updateSnowfall?.(time * 1000);

			// Smoothing agar parallax tidak bergantung pada FPS.
			const smoothing = 1 - Math.exp(-5 * delta);

			smoothMouse.lerp(mouse, smoothing);

			cameraTargetPosition.set(
				cameraBasePosition.x + smoothMouse.x * 1.1,
				cameraBasePosition.y + smoothMouse.y * 0.65,
				cameraBasePosition.z
			);

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

		// 8. Menyesuaikan ukuran canvas
		function resize() {
			const width = currentCanvas.clientWidth;
			const height = currentCanvas.clientHeight;

			if (!width || !height) return;

			camera.aspect = width / height;
			camera.updateProjectionMatrix();

			renderer.setSize(width, height, false);
		}

		window.addEventListener('resize', resize);

		// 9. Membersihkan resource
		return () => {
			cancelAnimationFrame(animationFrame);

			window.removeEventListener('resize', resize);
			window.removeEventListener('mousemove', handleMouseMove);

			currentScene.remove(sunlight);
			currentScene.remove(sunlight.target);
			currentScene.remove(ambientLight);
			currentScene.remove(hemisphereLight);

			renderer.dispose();

			updateClouds = undefined;
			updateWolf = undefined;
			updateSnowfall = undefined;

			scene = null;
		};
	});
</script>

<div class="scene">
	<canvas bind:this={canvas}></canvas>

	{#if scene}
		<Sky {scene} />
		<Sun {scene} />
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
		<SnowTrees {scene} />
		<Snowfall {scene} registerUpdate={(callback) => (updateSnowfall = callback)} />
	{/if}
</div>

<style lang="scss">
	.scene {
		width: 100%;
		height: 100vh;
		overflow: hidden;
		background: #87c5f5;

		canvas {
			display: block;
			width: 100%;
			height: 100%;
		}
	}
</style>
