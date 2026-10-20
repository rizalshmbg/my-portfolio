<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
		registerUpdate?: (callback: (time: number) => void) => void;
	}

	let { scene, registerUpdate }: Props = $props();

	onMount(() => {
		const objects: THREE.Object3D[] = [];
		const geometries: THREE.BufferGeometry[] = [];
		const materials: THREE.Material[] = [];

		// --------------------------------------------------
		// 1. Bintang kecil — distribusi mengikuti kode awal
		// --------------------------------------------------
		const starCount = 400;
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

			positions[i3] = (random() - 0.5) * 90;
			positions[i3 + 1] = random() * 30 + 2;
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

		createSparkle(-14, 7, -31, 0.3);
		createSparkle(-11, 13, -35, 0.25);
		createSparkle(-2, 10, -33, 0.28);
		createSparkle(9, 15, -36, 0.32);
		createSparkle(19, 5, -30, 0.25);
		createSparkle(27, 13, -34, 0.3);
		createSparkle(-25, 16, -35, 0.32);
		createSparkle(2, 5, -29, 0.22);
		createSparkle(12, 10, -32, 0.24);
		createSparkle(-17, 4, -30, 0.2);

		// Sisi kiri langit
		createSparkle(-34, 17, -34, 0.3);
		createSparkle(-38, 12, -32, 0.24);
		createSparkle(-32, 8, -35, 0.28);
		createSparkle(-36, 4, -31, 0.22);

		// Sisi kanan langit
		createSparkle(32, 16, -34, 0.3);
		createSparkle(38, 12, -32, 0.24);
		createSparkle(34, 8, -35, 0.28);
		createSparkle(37, 4, -31, 0.22);

		// --------------------------------------------------
		// 3. Animasi twinkle
		// --------------------------------------------------
		const sparkleObjects = objects.filter(
			(object): object is THREE.Group => object instanceof THREE.Group
		);

		const sparkleMaterials = sparkleObjects.map((sparkle) => {
			const mesh = sparkle.children[0] as THREE.Mesh;
			return mesh.material as THREE.MeshBasicMaterial;
		});

		const sparklePhases = sparkleObjects.map((_, index) => index * 1.7);

		// Bintang kecil: perubahan kecerahan lebih terlihat.
		const updateAnimation = (time: number) => {
			const smallStarPulse = 0.35 + 0.65 * ((Math.sin(time * 2.4) + 1) / 2);
			starMaterial.opacity = smallStarPulse;

			// Bintang aksen: berdenyut dengan fase dan kecepatan berbeda.
			sparkleMaterials.forEach((material, index) => {
				const pulse =
					0.25 + 0.75 * ((Math.sin(time * (2.2 + index * 0.12) + sparklePhases[index]) + 1) / 2);

				material.opacity = pulse;
			});
		};

		registerUpdate?.(updateAnimation);

		// --------------------------------------------------
		// 4. Cleanup
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

			registerUpdate?.(() => {});
		};
	});
</script>
