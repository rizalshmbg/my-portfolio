<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	type HillShape = 'peak' | 'rolling' | 'ridge';

	let { scene }: Props = $props();

	onMount(() => {
		const hillMeshes: THREE.Mesh[] = [];
		const hillMaterials: THREE.Material[] = [];
		const hillTextures: THREE.CanvasTexture[] = [];

		/**
		 * Membuat siluet badan gunung.
		 * Bentuk dan proporsi mengikuti WinterNight.
		 */
		function createHillShape(width: number, height: number, shapeType: HillShape): THREE.Shape {
			const shape = new THREE.Shape();

			shape.moveTo(-width / 2, -height * 0.15);

			if (shapeType === 'peak') {
				shape.lineTo(-width * 0.38, height * 0.2);
				shape.lineTo(-width * 0.18, height * 0.55);
				shape.lineTo(-width * 0.04, height);
				shape.lineTo(width * 0.08, height * 0.62);
				shape.lineTo(width * 0.22, height * 0.35);
				shape.lineTo(width * 0.38, height * 0.18);
			} else if (shapeType === 'rolling') {
				shape.lineTo(-width * 0.38, height * 0.15);

				shape.quadraticCurveTo(-width * 0.2, height * 0.85, -width * 0.02, height * 0.65);

				shape.quadraticCurveTo(width * 0.2, height * 0.48, width * 0.36, height * 0.12);
			} else {
				shape.lineTo(-width * 0.38, height * 0.25);
				shape.lineTo(-width * 0.23, height * 0.48);
				shape.lineTo(-width * 0.08, height * 0.34);
				shape.lineTo(width * 0.08, height * 0.68);
				shape.lineTo(width * 0.22, height * 0.42);
				shape.lineTo(width * 0.38, height * 0.2);
			}

			shape.lineTo(width / 2, -height * 0.15);
			shape.closePath();

			return shape;
		}

		/**
		 * Membuat tekstur gradasi biru untuk badan gunung.
		 */
		function createHillTexture(color: string): THREE.CanvasTexture {
			const canvas = document.createElement('canvas');

			canvas.width = 512;
			canvas.height = 512;

			const context = canvas.getContext('2d');

			if (!context) {
				throw new Error('Gagal membuat tekstur gunung.');
			}

			const baseColor = new THREE.Color(color);
			const middleColor = baseColor.clone().lerp(new THREE.Color('#25496f'), 0.2);
			const bottomColor = baseColor.clone().lerp(new THREE.Color('#142d49'), 0.4);

			const gradient = context.createLinearGradient(0, 512, 0, 0);

			gradient.addColorStop(0, bottomColor.getStyle());
			gradient.addColorStop(0.5, middleColor.getStyle());
			gradient.addColorStop(1, baseColor.getStyle());

			context.fillStyle = gradient;
			context.fillRect(0, 0, 512, 512);

			const texture = new THREE.CanvasTexture(canvas);

			texture.colorSpace = THREE.SRGBColorSpace;

			hillTextures.push(texture);

			return texture;
		}

		function createSnowFacet(
			points: Array<[number, number]>,
			width: number,
			height: number,
			position: THREE.Vector3,
			color: string,
			depthOffset = 0.08
		) {
			const shape = new THREE.Shape();

			shape.moveTo(points[0][0] * width, points[0][1] * height);

			for (let i = 1; i < points.length; i++) {
				shape.lineTo(points[i][0] * width, points[i][1] * height);
			}

			shape.closePath();

			const geometry = new THREE.ShapeGeometry(shape);

			const material = new THREE.MeshBasicMaterial({
				color,
				side: THREE.DoubleSide,
				depthTest: true,
				depthWrite: false
			});

			const snow = new THREE.Mesh(geometry, material);

			snow.position.copy(position);
			snow.position.z += depthOffset;

			scene.add(snow);

			hillMeshes.push(snow);
			hillMaterials.push(material);
		}

		function createPeakSnow(width: number, height: number, position: THREE.Vector3) {
			// Lapisan utama salju pada puncak.
			createSnowFacet(
				[
					[-0.18, 0.55],
					[-0.04, 1.0],
					[0.08, 0.62],
					[0.015, 0.72],
					[-0.025, 0.66],
					[-0.08, 0.58]
				],
				width,
				height,
				position,
				'#F7FBFF'
			);

			// Bayangan biru muda di sisi kanan puncak.
			createSnowFacet(
				[
					[-0.04, 1.0],
					[0.08, 0.62],
					[0.015, 0.72]
				],
				width,
				height,
				position,
				'#C6E0F5',
				0.09
			);

			// Ujung salju runcing di lereng kiri.
			createSnowFacet(
				[
					[-0.18, 0.55],
					[-0.08, 0.58],
					[-0.25, 0.36]
				],
				width,
				height,
				position,
				'#244A79',
				0.12
			);

			// Ujung salju runcing di lereng kanan.
			createSnowFacet(
				[
					[0.015, 0.72],
					[0.08, 0.62],
					[0.22, 0.35],
					[0.1, 0.54]
				],
				width,
				height,
				position,
				'#244A79',
				0.12
			);
		}

		function createRidgeSnow(width: number, height: number, position: THREE.Vector3) {
			// Salju pada puncak kiri.
			createSnowFacet(
				[
					[-0.38, 0.25],
					[-0.23, 0.48],
					[-0.08, 0.34],
					[-0.18, 0.4],
					[-0.23, 0.39]
				],
				width,
				height,
				position,
				'#F7FBFF'
			);

			// Ujung salju pada lereng puncak kiri.
			createSnowFacet(
				[
					[-0.23, 0.39],
					[-0.08, 0.34],
					[-0.035, 0.27],
					[-0.14, 0.35]
				],
				width,
				height,
				position,
				'#244A79',
				0.12
			);

			// Salju pada puncak kanan yang lebih tinggi.
			createSnowFacet(
				[
					[-0.08, 0.34],
					[0.08, 0.68],
					[0.22, 0.42],
					[0.13, 0.51],
					[0.015, 0.53]
				],
				width,
				height,
				position,
				'#F7FBFF'
			);

			// Bayangan salju pada sisi kanan puncak.
			createSnowFacet(
				[
					[0.08, 0.68],
					[0.22, 0.42],
					[0.13, 0.51]
				],
				width,
				height,
				position,
				'#C6E0F5',
				0.09
			);

			// Ujung salju pada lereng kanan.
			createSnowFacet(
				[
					[0.13, 0.51],
					[0.22, 0.42],
					[0.19, 0.3],
					[0.16, 0.43]
				],
				width,
				height,
				position,
				'#244A79',
				0.12
			);
		}

		function createHill(
			width: number,
			height: number,
			color: string,
			position: THREE.Vector3,
			shapeType: HillShape = 'peak'
		) {
			const geometry = new THREE.ShapeGeometry(createHillShape(width, height, shapeType), 16);

			const texture = createHillTexture(color);

			const material = new THREE.MeshStandardMaterial({
				map: texture,
				color: '#FFFFFF',
				roughness: 1,
				metalness: 0,
				side: THREE.DoubleSide
			});

			const hill = new THREE.Mesh(geometry, material);

			hill.position.copy(position);

			scene.add(hill);

			hillMeshes.push(hill);
			hillMaterials.push(material);

			// Hanya gunung berpuncak yang diberi topi salju.
			if (shapeType === 'peak') {
				createPeakSnow(width, height, position);
			} else if (shapeType === 'ridge') {
				createRidgeSnow(width, height, position);
			}
		}

		// Posisi dan ukuran tetap mengikuti WinterNight.
		// Gunung 1 — rolling
		createHill(90, 12, '#719AC5', new THREE.Vector3(-60, 0, -48), 'rolling');

		// Gunung 2 — peak
		createHill(30, 9, '#4778B0', new THREE.Vector3(-28, -0.5, -47), 'peak');

		// Gunung 3 — ridge
		createHill(35, 10, '#5B89BC', new THREE.Vector3(0, 0, -46), 'ridge');

		// Gunung 4 — rolling
		createHill(50, 5, '#719AC5', new THREE.Vector3(29, 0, -47), 'rolling');

		// Gunung 5 — peak
		createHill(50, 7, '#527FAF', new THREE.Vector3(45, 0, -48), 'peak');

		return () => {
			for (const mesh of hillMeshes) {
				scene.remove(mesh);
				mesh.geometry.dispose();
			}

			for (const material of hillMaterials) {
				material.dispose();
			}

			for (const texture of hillTextures) {
				texture.dispose();
			}

			hillMeshes.length = 0;
			hillMaterials.length = 0;
			hillTextures.length = 0;
		};
	});
</script>
