<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	let { scene }: Props = $props();

	onMount(() => {
		const fogObjects: THREE.Sprite[] = [];
		const fogTextures: THREE.CanvasTexture[] = [];
		const fogMaterials: THREE.SpriteMaterial[] = [];

		// Membuat tekstur kabut dengan gradasi transparan.
		const canvas = document.createElement('canvas');
		canvas.width = 256;
		canvas.height = 256;

		const context = canvas.getContext('2d');

		if (!context) return;

		const gradient = context.createRadialGradient(128, 128, 5, 128, 128, 128);

		gradient.addColorStop(0, 'rgba(145, 164, 190, 0.18)');
		gradient.addColorStop(0.4, 'rgba(120, 143, 174, 0.08)');
		gradient.addColorStop(1, 'rgba(100, 125, 160, 0)');

		context.fillStyle = gradient;
		context.fillRect(0, 0, 256, 256);

		const texture = new THREE.CanvasTexture(canvas);
		texture.colorSpace = THREE.SRGBColorSpace;

		// Posisi dan ukuran kabut di belakang pegunungan.
		const fogConfigs = [
			{ x: -20, y: 1.5, z: -34, width: 24, height: 6 },
			{ x: 5, y: 1.2, z: -36, width: 30, height: 7 },
			{ x: 25, y: 1.8, z: -34, width: 22, height: 6 }
		];

		for (const config of fogConfigs) {
			const material = new THREE.SpriteMaterial({
				map: texture,
				transparent: true,
				opacity: 0.35,
				depthWrite: false,
				depthTest: true,
				toneMapped: true
			});

			const fog = new THREE.Sprite(material);
			fog.position.set(config.x, config.y, config.z);
			fog.scale.set(config.width, config.height, 1);

			scene.add(fog);

			fogObjects.push(fog);
			fogMaterials.push(material);
		}

		fogTextures.push(texture);

		return () => {
			for (const fog of fogObjects) {
				scene.remove(fog);
			}

			for (const material of fogMaterials) {
				material.dispose();
			}

			for (const texture of fogTextures) {
				texture.dispose();
			}
		};
	});
</script>
