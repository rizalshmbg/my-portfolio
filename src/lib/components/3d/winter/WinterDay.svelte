<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { gsap } from 'gsap';

	import Sun from './day-scene/Sun.svelte';
	import Clouds from './day-scene/Clouds.svelte';
	import Hills from './day-scene/Hills.svelte';
	import MountainFog from './day-scene/MountainFog.svelte';
	import Cabin from './day-scene/Cabin.svelte';
	import CabinPath from './shared-scene/CabinPath.svelte';
	import SnowTerrain from './day-scene/SnowTerrain.svelte';
	import SnowRocks from './day-scene/SnowRocks.svelte';

	import WinterGrass from './shared-scene/WinterGrass.svelte';
	import Trees from './night-scene/Trees.svelte';
	import Wolf from './night-scene/Wolf.svelte';
	import SnowTrees from './night-scene/SnowTrees.svelte';

	interface Props {
		progress: number;
	}

	let { progress }: Props = $props();

	let canvas = $state<HTMLCanvasElement | undefined>();
	let scene = $state<THREE.Scene | null>(null);
	let setCameraProgress = $state<((value: number) => void) | undefined>(undefined);

	let updateClouds: ((time: number) => void) | undefined;
	let updateWolf: ((time: number) => void) | undefined;

	$effect(() => {
		setCameraProgress?.(progress);
	});

	onMount(() => {
		if (!canvas) return;

		const currentCanvas = canvas;

		// 1. Membuat scene
		const currentScene = new THREE.Scene();

		currentScene.background = new THREE.Color('#87c5f5');
		currentScene.fog = new THREE.Fog('#c5e1f2', 45, 100);

		scene = currentScene;

		// 2. Membuat kamera
		const camera = new THREE.PerspectiveCamera(
			45,
			currentCanvas.clientWidth / currentCanvas.clientHeight,
			0.1,
			100
		);

		const cameraBasePosition = new THREE.Vector3(0, 2, 8);
		const cameraBaseTarget = new THREE.Vector3(0, 4, -10);

		const cameraState = {
			x: cameraBasePosition.x,
			y: cameraBasePosition.y,
			z: cameraBasePosition.z,
			targetX: cameraBaseTarget.x,
			targetY: cameraBaseTarget.y,
			targetZ: cameraBaseTarget.z
		};

		// 3. Animasi kamera menggunakan GSAP
		const cameraTimeline = gsap.timeline({
			paused: true,
			onUpdate: () => {
				cameraBasePosition.set(cameraState.x, cameraState.y, cameraState.z);

				cameraBaseTarget.set(cameraState.targetX, cameraState.targetY, cameraState.targetZ);
			}
		});

		cameraTimeline
			// Home → About: mendekati kabin
			.to(cameraState, {
				x: -12,
				y: 2,
				z: -12,
				targetX: -18,
				targetY: 3,
				targetZ: -25,
				duration: 1,
				ease: 'none'
			})
			// About → Experience: mendekati kawanan serigala
			.to(cameraState, {
				x: 6,
				y: 2,
				z: -13,
				targetX: 6,
				targetY: 1,
				targetZ: -21,
				duration: 1,
				ease: 'none'
			})
			// Experience → Contact: mengarah ke langit
			.to(cameraState, {
				x: -7,
				y: 16,
				z: -29,
				targetX: -7,
				targetY: 20,
				targetZ: -50,
				duration: 1,
				ease: 'none'
			});

		setCameraProgress = (value: number) => {
			const normalizedProgress = THREE.MathUtils.clamp(value, 0, 1);

			cameraTimeline.progress(normalizedProgress);
		};

		setCameraProgress(progress);

		camera.position.copy(cameraBasePosition);
		camera.lookAt(cameraBaseTarget);

		// 4. Menyiapkan parallax mouse
		const mouse = new THREE.Vector2(0, 0);
		const smoothMouse = new THREE.Vector2(0, 0);

		const cameraTargetPosition = new THREE.Vector3();
		const lookTarget = new THREE.Vector3();

		// 5. Membuat renderer
		const renderer = new THREE.WebGLRenderer({
			canvas: currentCanvas,
			antialias: true
		});

		renderer.shadowMap.enabled = true;
		renderer.shadowMap.type = THREE.PCFShadowMap;

		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5));

		renderer.setSize(currentCanvas.clientWidth, currentCanvas.clientHeight, false);

		renderer.outputColorSpace = THREE.SRGBColorSpace;

		// 6. Pencahayaan untuk suasana siang

		// Cahaya dasar agar area yang teduh tetap terlihat.
		const ambientLight = new THREE.AmbientLight('#e1efff', 2);

		currentScene.add(ambientLight);

		const sunlight = new THREE.DirectionalLight('#fff0c2', 3);

		sunlight.position.set(-8, 20, -50);
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

		// Fill light untuk mempertahankan detail salju.
		const hemisphereLight = new THREE.HemisphereLight('#d8edff', '#a5b5c7', 1.5);

		currentScene.add(hemisphereLight);

		// 7. Animasi dan rendering
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

			sunlight.dispose();

			renderer.dispose();
			cameraTimeline.kill();

			setCameraProgress = undefined;
			updateClouds = undefined;
			updateWolf = undefined;

			scene = null;
		};
	});
</script>

<div class="scene">
	<canvas bind:this={canvas}></canvas>

	{#if scene}
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
