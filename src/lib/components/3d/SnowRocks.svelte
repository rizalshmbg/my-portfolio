<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	let { scene }: Props = $props();

	onMount(() => {
		const rockGroup = new THREE.Group();
		rockGroup.name = 'SnowRocks';

		const geometries: THREE.BufferGeometry[] = [];
		const materials: THREE.Material[] = [];

		const TERRAIN = {
			centerX: 8,
			centerY: -0.5,
			centerZ: -18,
			width: 70,
			depth: 38
		};

		// Warna batu dingin agar menyatu dengan malam.
		const rockMaterials = [
			new THREE.MeshStandardMaterial({
				color: '#454a52',
				roughness: 1,
				flatShading: true
			}),
			new THREE.MeshStandardMaterial({
				color: '#565b63',
				roughness: 1,
				flatShading: true
			}),
			new THREE.MeshStandardMaterial({
				color: '#686b70',
				roughness: 1,
				flatShading: true
			})
		];

		materials.push(...rockMaterials);

		// Geometri batu kecil tetap seperti sebelumnya.
		const rockGeometry = new THREE.IcosahedronGeometry(1, 0);
		geometries.push(rockGeometry);

		// Geometri khusus batu besar: lebih halus dan sedikit tidak beraturan.
		const largeRockGeometry = new THREE.IcosahedronGeometry(1, 1);
		const positions = largeRockGeometry.attributes.position;

		// Kubah salju dengan bentuk sedikit tidak beraturan.
		const largeSnowGeometry = new THREE.SphereGeometry(1, 12, 6, 0, Math.PI * 2, 0, Math.PI / 2);
		geometries.push(largeSnowGeometry);

		const smallSnowGeometry = new THREE.IcosahedronGeometry(1, 1);
		geometries.push(smallSnowGeometry);

		// Material salju digunakan bersama.
		const snowMaterial = new THREE.MeshStandardMaterial({
			color: '#d9dce2',
			roughness: 1,
			flatShading: true,
			side: THREE.DoubleSide
		});

		materials.push(snowMaterial);

		for (let i = 0; i < positions.count; i++) {
			const x = positions.getX(i);
			const y = positions.getY(i);
			const z = positions.getZ(i);

			const variation = 1 + 0.1 * Math.sin(x * 7.1 + z * 4.3) * Math.cos(y * 5.7 + x * 2.8);

			positions.setXYZ(i, x * variation, y * variation, z * variation);
		}

		largeRockGeometry.computeVertexNormals();
		geometries.push(largeRockGeometry);

		// Random deterministik.
		function createRandom(seed: number) {
			let state = seed;

			return function random(): number {
				state = (state * 16807) % 2147483647;
				return (state - 1) / 2147483646;
			};
		}

		function range(random: () => number, min: number, max: number): number {
			return min + random() * (max - min);
		}

		function riverCenterNX(nz: number): number {
			return -0.35 - 0.22 * nz + 0.07 * Math.sin(nz * 3);
		}

		// Harus konsisten dengan terrainHeight di SnowTerrain.svelte.
		function terrainHeight(x: number, z: number): number {
			const nx = x / (TERRAIN.width / 2);
			const nz = z / (TERRAIN.depth / 2);

			const ridgeCenter = -0.28 + 0.2 * nx * nx + 0.035 * Math.sin(nx * 3);
			const distance = nz - ridgeCenter;

			const ridge = 0.9 * Math.exp(-Math.pow(distance / 0.58, 2));
			const shoulder = 0.38 * Math.exp(-Math.pow(distance / 1.0, 2));

			const frontSlope = -0.4 * THREE.MathUtils.smoothstep(distance, 0.05, 1.0);

			const riverDistance = nx - riverCenterNX(nz);
			const riverMask = Math.exp(-Math.pow(riverDistance / 0.25, 4));
			const riverBed = -0.72 * riverMask;

			const riverBanks = 0.3 * Math.exp(-Math.pow((Math.abs(riverDistance) - 0.3) / 0.12, 2));

			const sideMask = THREE.MathUtils.smoothstep(Math.abs(nx), 0.3, 0.95);
			const frontMask = THREE.MathUtils.smoothstep(nz, -0.05, 0.85);
			const sideRise = 0.28 * sideMask * frontMask;

			const rolling = 0.025 * Math.sin(nx * 2.4 + nz * 0.8) + 0.018 * Math.sin(nx * 4.0 - nz * 1.1);

			const broadSnow =
				0.052 * Math.sin(x * 0.15 + z * 0.08) * Math.sin(z * 0.12 - x * 0.045) +
				0.022 * Math.sin(x * 0.095 - z * 0.13 + 1.4);

			const fineSnow = 0.007 * Math.sin(x * 0.34 + z * 0.19) * Math.sin(z * 0.25 - x * 0.14);

			const foregroundMask = THREE.MathUtils.smoothstep(nz, -0.05, 0.8);
			const riverDistanceWorld = Math.abs(riverDistance) * (TERRAIN.width / 2);

			const riverFade = THREE.MathUtils.smoothstep(riverDistanceWorld, 2.8, 7.0);

			const snowContours = (broadSnow + fineSnow) * riverFade * (0.35 + 0.65 * foregroundMask);

			return (
				-0.55 +
				ridge +
				shoulder +
				frontSlope +
				sideRise +
				riverBanks +
				riverBed +
				rolling +
				snowContours
			);
		}

		function addRock(localX: number, localZ: number, seed: number, size: number, isLarge = false) {
			const random = createRandom(seed);
			const height = terrainHeight(localX, localZ);

			const rock = new THREE.Mesh(
				isLarge ? largeRockGeometry : rockGeometry,
				rockMaterials[Math.floor(random() * rockMaterials.length)]
			);

			// Lebar dan tinggi tidak sama agar bentuknya alami.
			const sx = size * (isLarge ? range(random, 0.85, 1.1) : range(random, 0.9, 1.15));
			const sy = size * (isLarge ? range(random, 0.95, 1.25) : range(random, 0.75, 1.05));
			const sz = size * (isLarge ? range(random, 0.85, 1.1) : range(random, 0.9, 1.15));

			rock.scale.set(sx, sy, sz);

			// Sebagian batu tertanam di dalam permukaan salju.
			rock.position.set(
				TERRAIN.centerX + localX,
				TERRAIN.centerY + height - sy * (isLarge ? 0.68 : 0.58),
				TERRAIN.centerZ + localZ
			);

			rock.rotation.set(range(random, -0.15, 0.15), random() * Math.PI, range(random, -0.18, 0.18));

			rock.castShadow = true;
			rock.receiveShadow = true;

			rockGroup.add(rock);

			// Batu besar menggunakan kubah salju.
			// Batu kecil tetap menggunakan bentuk salju sebelumnya.
			const snowGeometry = isLarge ? largeSnowGeometry : smallSnowGeometry;

			const snowCap = new THREE.Mesh(snowGeometry, snowMaterial);

			if (isLarge) {
				snowCap.scale.set(sx * 1.05, sy * 0.38, sz * 1.05);

				snowCap.position.set(rock.position.x, rock.position.y + sy * 0.68, rock.position.z);
			} else {
				// Pertahankan salju batu kecil.
				snowCap.scale.set(sx * 1.08, sy * 0.28, sz * 1.08);

				snowCap.position.set(rock.position.x, rock.position.y + sy * 0.68, rock.position.z);
			}

			snowCap.rotation.copy(rock.rotation);
			snowCap.castShadow = false;
			snowCap.receiveShadow = true;

			rockGroup.add(snowCap);
		}

		// ================================================
		// ROCKS ALONG BOTH RIVER BANKS
		// ================================================

		const random = createRandom(8612);
		let rockId = 0;

		const riverHalfWidth = 3.8;

		for (let localZ = 1; localZ <= 16; localZ += 1.25) {
			for (const side of [-1, 1]) {
				if (random() < 0.35) continue;

				// Posisi akhir batu sepanjang sungai.
				const rockZ = localZ + range(random, -0.35, 0.35);
				const nz = rockZ / (TERRAIN.depth / 2);

				const riverCenterX = riverCenterNX(nz) * (TERRAIN.width / 2);

				// Samakan perhitungan lebar dengan SnowTerrain.svelte.
				const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

				const t = (rockZ - (-TERRAIN.depth / 2 + 0.5)) / (TERRAIN.depth - 1);

				const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0, 0.12);

				const actualRiverWidth = riverHalfWidth * widthVariation * endFade;

				// Letakkan batu sedikit di sisi salju,
				// bukan di dalam area air.
				const bankOffset = actualRiverWidth + range(random, 0.15, 0.5);

				const localX = riverCenterX + side * bankOffset;

				const size = range(random, 0.16, 0.34);

				addRock(localX, rockZ, 1000 + rockId * 37, size);

				rockId++;
			}
		}

		// ================================================
		// BIG ROCKS ALONG THE RIVER
		// ================================================

		const largeRockRandom = createRandom(47291);
		let largeRockId = 0;

		// Batu besar di kedua sisi sungai
		for (let localZ = 2; localZ <= 15; localZ += 2.3) {
			for (const side of [-1, 1]) {
				// Sisakan beberapa bagian tepian tanpa batu agar alami
				if (largeRockRandom() < (side === -1 ? 0.45 : 0.05)) continue;

				const rockZ = localZ + range(largeRockRandom, -0.75, 0.75);

				const nz = rockZ / (TERRAIN.depth / 2);
				const riverCenterX = riverCenterNX(nz) * (TERRAIN.width / 2);

				// Samakan lebar sungai dengan SnowTerrain.svelte
				const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

				const t = (rockZ - (-TERRAIN.depth / 2 + 0.5)) / (TERRAIN.depth - 1);

				const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0, 0.12);

				const actualRiverWidth = riverHalfWidth * widthVariation * endFade;

				// Ukuran batu utama lebih besar daripada batu yang sudah ada
				const size = range(largeRockRandom, 0.45, 0.9);

				// Variasikan jarak batu dari air agar tidak membentuk garis.
				const bankOffset =
					actualRiverWidth +
					size * range(largeRockRandom, 0.05, 0.55) +
					range(largeRockRandom, -0.2, 0.45);

				const localX = riverCenterX + side * bankOffset;

				addRock(localX, rockZ, 5000 + largeRockId * 53, size, true);

				largeRockId++;
			}
		}

		// ================================================
		// A FEW ROCKS NEAR TREE BASES
		// ================================================

		const treeAnchors = [
			{ x: -20, z: -15, seed: 210 },
			{ x: -10, z: -13, seed: 320 },
			{ x: 11, z: -15, seed: 430 },
			{ x: 12, z: -10, seed: 540 },
			{ x: 17, z: -12, seed: 650 }
		];

		for (const tree of treeAnchors) {
			const random = createRandom(tree.seed);

			// Konversi koordinat world pohon ke lokal terrain.
			const localTreeX = tree.x - TERRAIN.centerX;
			const localTreeZ = tree.z - TERRAIN.centerZ;

			for (let i = 0; i < 2; i++) {
				const angle = random() * Math.PI * 2;
				const radius = range(random, 1.0, 1.8);

				const localX = localTreeX + Math.cos(angle) * radius;
				const localZ = localTreeZ + Math.sin(angle) * radius;

				const nx = localX / (TERRAIN.width / 2);
				const nz = localZ / (TERRAIN.depth / 2);

				// Hindari menaruh batu di sungai.
				const riverDistance = Math.abs(nx - riverCenterNX(nz)) * (TERRAIN.width / 2);

				if (riverDistance < 3.0) continue;

				addRock(localX, localZ, tree.seed + i * 17, range(random, 0.16, 0.28));
			}
		}

		scene.add(rockGroup);

		return () => {
			scene.remove(rockGroup);

			for (const geometry of geometries) {
				geometry.dispose();
			}

			for (const material of materials) {
				material.dispose();
			}
		};
	});
</script>
