<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	let { scene }: Props = $props();

	onMount(() => {
		const objects: THREE.Object3D[] = [];
		const geometries: THREE.BufferGeometry[] = [];
		const materials: THREE.Material[] = [];

		// --------------------------------------------------
		// 1. Bintang kecil — distribusi mengikuti kode awal
		// --------------------------------------------------
		const starCount = 200;
		const positions = new Float32Array(starCount * 3);
		const colors = new Float32Array(starCount * 3);

		const warmWhite = new THREE.Color('#f5e9d5');
		const coolWhite = new THREE.Color('#cbd5e1');

		// Seed tetap supaya distribusi tidak berubah saat refresh.
		let seed = 12345;

		function random() {
			seed = (seed * 16807) % 2147483647;
			return (seed - 1) / 2147483646;
		}

		for (let i = 0; i < starCount; i++) {
			const i3 = i * 3;

			positions[i3] = (random() - 0.5) * 65;
			positions[i3 + 1] = random() * 25 + 2;
			positions[i3 + 2] = -38 - random() * 4;

			const color = random() > 0.65 ? warmWhite : coolWhite;

			colors[i3] = color.r;
			colors[i3 + 1] = color.g;
			colors[i3 + 2] = color.b;
		}

		const starGeometry = new THREE.BufferGeometry();

		starGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

		starGeometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

		const starMaterial = new THREE.PointsMaterial({
			size: 0.12,
			sizeAttenuation: true,
			vertexColors: true,
			transparent: true,
			opacity: 0.8,
			depthTest: true,
			depthWrite: false,
			toneMapped: false
		});

		const stars = new THREE.Points(starGeometry, starMaterial);

		scene.add(stars);

		objects.push(stars);
		geometries.push(starGeometry);
		materials.push(starMaterial);

		// --------------------------------------------------
		// 2. Membuat bintang terang berbentuk empat ujung
		// --------------------------------------------------
		function createSparkle(x: number, y: number, z: number, size: number, color = '#fff1d9') {
			const verticalShape = new THREE.Shape();

			verticalShape.moveTo(0, size);
			verticalShape.quadraticCurveTo(size * 0.12, size * 0.12, size * 0.2, 0);
			verticalShape.quadraticCurveTo(size * 0.12, -size * 0.12, 0, -size);
			verticalShape.quadraticCurveTo(-size * 0.12, -size * 0.12, -size * 0.2, 0);
			verticalShape.quadraticCurveTo(-size * 0.12, size * 0.12, 0, size);

			const horizontalShape = new THREE.Shape();

			horizontalShape.moveTo(-size * 0.75, 0);
			horizontalShape.quadraticCurveTo(0, size * 0.07, size * 0.75, 0);
			horizontalShape.quadraticCurveTo(0, -size * 0.07, -size * 0.75, 0);

			const verticalGeometry = new THREE.ShapeGeometry(verticalShape, 8);
			const horizontalGeometry = new THREE.ShapeGeometry(horizontalShape, 8);

			const material = new THREE.MeshBasicMaterial({
				color,
				transparent: true,
				opacity: 0.95,
				side: THREE.DoubleSide,
				depthTest: true,
				depthWrite: false,
				toneMapped: false
			});

			const sparkle = new THREE.Group();

			const verticalMesh = new THREE.Mesh(verticalGeometry, material);
			const horizontalMesh = new THREE.Mesh(horizontalGeometry, material);

			horizontalMesh.position.z = 0.001;

			sparkle.add(verticalMesh, horizontalMesh);
			sparkle.position.set(x, y, z);

			scene.add(sparkle);

			objects.push(sparkle);
			geometries.push(verticalGeometry, horizontalGeometry);
			materials.push(material);
		}

		// Posisi aksen bintang terang, tersebar seperti desain.
		createSparkle(-20, 12, -30, 0.38);
		createSparkle(-7, 15, -32, 0.48);
		createSparkle(5, 8, -30, 0.32);
		createSparkle(15, 13, -34, 0.42);
		createSparkle(-28, 5, -29, 0.28);
		createSparkle(23, 7, -31, 0.3);

		// --------------------------------------------------
		// 3. Cleanup
		// --------------------------------------------------
		return () => {
			for (const object of objects) {
				scene.remove(object);
			}

			for (const geometry of geometries) {
				geometry.dispose();
			}

			for (const material of materials) {
				material.dispose();
			}
		};
	});
</script>
