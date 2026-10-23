<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	let { scene }: Props = $props();

	onMount(() => {
		const moonGroup = new THREE.Group();

		// Membuat tekstur bulan secara prosedural
		const canvas = document.createElement('canvas');
		canvas.width = 512;
		canvas.height = 512;

		const context = canvas.getContext('2d');

		if (!context) return;

		// Warna dasar krem hangat
		context.fillStyle = '#FBEAD6';
		context.fillRect(0, 0, 512, 512);

		// Tambahkan variasi permukaan bulan
		for (let i = 0; i < 1800; i++) {
			const x = Math.random() * 512;
			const y = Math.random() * 512;
			const radius = Math.random() * 12 + 2;

			context.beginPath();
			context.arc(x, y, radius, 0, Math.PI * 2);

			context.fillStyle =
				Math.random() > 0.5
					? `rgba(185, 151, 104, ${Math.random() * 0.12})`
					: `rgba(255, 247, 222, ${Math.random() * 0.2})`;

			context.fill();
		}

		const moonTexture = new THREE.CanvasTexture(canvas);
		moonTexture.colorSpace = THREE.SRGBColorSpace;

		// Ukuran bulan
		const moonGeometry = new THREE.SphereGeometry(5, 64, 64);

		const moonMaterial = new THREE.MeshBasicMaterial({
			map: moonTexture,
			color: '#fff4dc'
		});

		const moon = new THREE.Mesh(moonGeometry, moonMaterial);
		moon.scale.set(1, 1, 1);
		moonGroup.add(moon);

		// Cahaya bulan yang menerangi lingkungan sekitar.
		const moonLight = new THREE.PointLight('#ffe6b5', 100, 65, 2);
		moonLight.position.set(-1, 0, 0);
		moonLight.castShadow = false;
		moonGroup.add(moonLight);

		// Cahaya halo di sekitar bulan
		const glowCanvas = document.createElement('canvas');
		glowCanvas.width = 256;
		glowCanvas.height = 256;
		const glowContext = glowCanvas.getContext('2d');

		if (!glowContext) {
			moonGeometry.dispose();
			moonMaterial.dispose();
			moonTexture.dispose();
			return;
		}

		const gradient = glowContext.createRadialGradient(128, 128, 35, 128, 128, 128);

		gradient.addColorStop(0, 'rgba(255, 226, 180, 0.28)');
		gradient.addColorStop(0.35, 'rgba(255, 220, 170, 0.12)');
		gradient.addColorStop(0.7, 'rgba(255, 215, 160, 0.035)');
		gradient.addColorStop(1, 'rgba(255, 215, 160, 0)');

		glowContext.fillStyle = gradient;
		glowContext.fillRect(0, 0, 256, 256);

		const glowTexture = new THREE.CanvasTexture(glowCanvas);

		const glowMaterial = new THREE.SpriteMaterial({
			map: glowTexture,
			transparent: true,
			opacity: 0.8,
			blending: THREE.AdditiveBlending,
			depthWrite: false,
			depthTest: true
		});

		const glow = new THREE.Sprite(glowMaterial);
		glow.position.set(0, 0, -0.2);
		glow.scale.set(17, 17, 1);

		moonGroup.add(glow);

		// Posisi bulan di langit
		moonGroup.position.set(10, 7, -20);

		scene.add(moonGroup);

		return () => {
			scene.remove(moonGroup);

			moonGeometry.dispose();
			moonMaterial.dispose();
			moonTexture.dispose();
			glowMaterial.dispose();
			glowTexture.dispose();
		};
	});
</script>
