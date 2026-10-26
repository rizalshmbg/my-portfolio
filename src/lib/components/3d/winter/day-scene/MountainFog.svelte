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

		// Membuat tekstur kabut biru muda dengan gradasi transparan.
		const canvas = document.createElement('canvas');
		canvas.width = 256;
		canvas.height = 256;

		const context = canvas.getContext('2d');

		if (!context) return;

		const gradient = context.createLinearGradient(0, 0, 0, 256);

		gradient.addColorStop(0, 'rgba(210, 229, 245, 0)');
		gradient.addColorStop(0.25, 'rgba(210, 229, 245, 0.03)');
		gradient.addColorStop(0.5, 'rgba(210, 229, 245, 0.12)');
		gradient.addColorStop(0.75, 'rgba(190, 216, 239, 0.05)');
		gradient.addColorStop(1, 'rgba(175, 205, 233, 0)');

		context.fillStyle = gradient;
		context.fillRect(0, 0, 256, 256);

		// Mask horizontal untuk melembutkan ujung kabut.
		const horizontalGradient = context.createLinearGradient(0, 0, 256, 0);

		horizontalGradient.addColorStop(0, 'rgba(0, 0, 0, 0)');
		horizontalGradient.addColorStop(0.2, 'rgba(0, 0, 0, 0.65)');
		horizontalGradient.addColorStop(0.5, 'rgba(0, 0, 0, 1)');
		horizontalGradient.addColorStop(0.8, 'rgba(0, 0, 0, 0.65)');
		horizontalGradient.addColorStop(1, 'rgba(0, 0, 0, 0)');

		context.globalCompositeOperation = 'destination-in';
		context.fillStyle = horizontalGradient;
		context.fillRect(0, 0, 256, 256);
		context.globalCompositeOperation = 'source-over';

		const texture = new THREE.CanvasTexture(canvas);
		texture.colorSpace = THREE.SRGBColorSpace;

		// Posisi dan ukuran sama dengan MountainFog WinterNight.
		const fogConfigs = [
			{ x: -20, y: 2, z: -44, width: 26, height: 4 },
			{ x: 5, y: 1.5, z: -46, width: 32, height: 4.5 },
			{ x: 25, y: 2, z: -44, width: 24, height: 4 }
		];

		for (const config of fogConfigs) {
			const material = new THREE.SpriteMaterial({
				map: texture,
				transparent: true,
				opacity: 0.25,
				depthWrite: false,
				depthTest: false,
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
