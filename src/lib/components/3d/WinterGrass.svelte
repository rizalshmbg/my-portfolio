<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	interface TreeAnchor {
		x: number;
		z: number;
		count: number;
		seed: number;
	}

	let { scene }: Props = $props();

	onMount(() => {
		const grassGroup = new THREE.Group();
		grassGroup.name = 'WinterGrass';

		const geometries: THREE.BufferGeometry[] = [];
		const materials: THREE.Material[] = [];

		// ================================================
		// GRASS GEOMETRY & MATERIALS
		// ================================================

		const grassGeometry = new THREE.CylinderGeometry(0.001, 0.008, 1, 4);

		geometries.push(grassGeometry);

		const grassMaterials = [
			new THREE.MeshStandardMaterial({
				color: '#514d49',
				roughness: 1
			}),
			new THREE.MeshStandardMaterial({
				color: '#777064',
				roughness: 1
			}),
			new THREE.MeshStandardMaterial({
				color: '#626970',
				roughness: 1
			})
		];

		materials.push(...grassMaterials);

		// ================================================
		// TERRAIN CONFIGURATION
		// Harus sesuai dengan SnowTerrain.svelte.
		// ================================================

		const TERRAIN = {
			centerX: 8,
			centerY: -0.5,
			centerZ: -18,
			width: 70,
			depth: 38
		};

		function riverCenterNX(nz: number): number {
			return -0.35 - 0.22 * nz + 0.07 * Math.sin(nz * 3);
		}

		// ================================================
		// TERRAIN HEIGHT
		// ================================================

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

		// ================================================
		// SEEDED RANDOM
		// Generator terpisah agar setiap pohon stabil.
		// ================================================

		function createRandom(seed: number) {
			let state = seed;

			return function random(): number {
				state = (state * 16807) % 2147483647;
				return (state - 1) / 2147483646;
			};
		}

		function randomRange(random: () => number, min: number, max: number): number {
			return min + random() * (max - min);
		}

		// ================================================
		// GRASS STEM CREATION
		// ================================================

		function addGrassStem(
			tuft: THREE.Group,
			start: THREE.Vector3,
			end: THREE.Vector3,
			material: THREE.Material
		): void {
			const direction = new THREE.Vector3().subVectors(end, start);
			const length = direction.length();

			if (length < 0.01) return;

			const stem = new THREE.Mesh(grassGeometry, material);

			stem.position.copy(start).add(end).multiplyScalar(0.5);

			stem.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), direction.normalize());

			stem.scale.y = length;
			stem.castShadow = false;
			stem.receiveShadow = true;

			tuft.add(stem);
		}

		function createGrassTuft(random: () => number): THREE.Group {
			const tuft = new THREE.Group();

			// Rumput diperbesar agar terbaca dari kamera utama.
			const stemCount = 8;

			for (let i = 0; i < stemCount; i++) {
				const angle = random() * Math.PI * 2;

				const stemHeight = randomRange(random, 0.55, 0.85);
				const lean = randomRange(random, 0.12, 0.3);

				const material = grassMaterials[Math.floor(random() * grassMaterials.length)];

				const start = new THREE.Vector3(
					randomRange(random, -0.06, 0.06),
					0.015,
					randomRange(random, -0.06, 0.06)
				);

				const end = new THREE.Vector3(Math.cos(angle) * lean, stemHeight, Math.sin(angle) * lean);

				addGrassStem(tuft, start, end, material);

				// Cabang kecil dengan panjang bervariasi.
				const branchCount = random() > 0.45 ? 2 : 1;

				for (let j = 0; j < branchCount; j++) {
					const branchT = randomRange(random, 0.42, 0.78);

					const branchStart = start.clone().lerp(end, branchT);

					const branchAngle = angle + (random() > 0.5 ? 1 : -1) * randomRange(random, 0.65, 1.35);

					const branchLength = randomRange(random, 0.16, 0.28);

					const branchEnd = branchStart
						.clone()
						.add(
							new THREE.Vector3(
								Math.cos(branchAngle) * branchLength,
								branchLength * randomRange(random, 0.25, 0.65),
								Math.sin(branchAngle) * branchLength
							)
						);

					addGrassStem(tuft, branchStart, branchEnd, material);
				}
			}

			// Ukuran setiap kelompok juga deterministik.
			const scale = randomRange(random, 0.8, 1.2);
			tuft.scale.setScalar(scale);

			tuft.rotation.y = random() * Math.PI;

			return tuft;
		}

		// ================================================
		// TREE ANCHORS
		// Posisi pohon dari Trees.svelte dalam koordinat
		// lokal terrain.
		// ================================================

		const treeAnchors: TreeAnchor[] = [
			{ x: -28, z: 3, count: 7, seed: 101 },
			{ x: -18, z: 5, count: 5, seed: 202 },
			{ x: 3, z: 3, count: 3, seed: 303 },
			{ x: 4, z: 8, count: 6, seed: 404 },
			{ x: 9, z: 6, count: 5, seed: 505 }
		];

		// ================================================
		// PLACE GRASS AROUND EACH TREE
		// ================================================

		for (const anchor of treeAnchors) {
			const random = createRandom(anchor.seed);

			let placed = 0;
			let attempts = 0;

			while (placed < anchor.count && attempts < 100) {
				attempts++;

				const angle = random() * Math.PI * 2;
				const radius = randomRange(random, 1.0, 2.8);

				const localX = anchor.x + Math.cos(angle) * radius;
				const localZ = anchor.z + Math.sin(angle) * radius;

				const nx = localX / (TERRAIN.width / 2);
				const nz = localZ / (TERRAIN.depth / 2);

				// Hindari permukaan sungai.
				const riverDistance = Math.abs(nx - riverCenterNX(nz)) * (TERRAIN.width / 2);

				if (riverDistance < 2.8) continue;

				// Hindari batang pohon.
				if (Math.hypot(localX - anchor.x, localZ - anchor.z) < 0.75) {
					continue;
				}

				// Hindari bagian luar terrain.
				if (Math.abs(localX) > TERRAIN.width / 2 - 2) continue;

				const height = terrainHeight(localX, localZ);
				const tuft = createGrassTuft(random);

				tuft.position.set(
					TERRAIN.centerX + localX,
					TERRAIN.centerY + height,
					TERRAIN.centerZ + localZ
				);

				grassGroup.add(tuft);
				placed++;
			}
		}

		// ================================================
		// GRASS AROUND ROCKS ON THE RIGHT RIVER BANK
		// ================================================

		const rightBankRandom = createRandom(73521);

		for (let z = 2.5; z <= 15; z += 2.3) {
			// Sisakan beberapa bagian tepian tanpa rumput.
			if (rightBankRandom() < 0.25) continue;

			const localZ = z + randomRange(rightBankRandom, -0.65, 0.65);
			const nz = localZ / (TERRAIN.depth / 2);

			const riverCenterX = riverCenterNX(nz) * (TERRAIN.width / 2);

			// Samakan lebar sungai dengan SnowTerrain.svelte.
			const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

			const t = (localZ - (-TERRAIN.depth / 2 + 0.5)) / (TERRAIN.depth - 1);

			const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0, 0.12);

			const riverWidth = 3.8 * widthVariation * endFade;

			// Tempatkan rumput di daratan, dekat bebatuan kanan.
			const localX = riverCenterX + riverWidth + randomRange(rightBankRandom, 0.45, 1.15);

			const nx = localX / (TERRAIN.width / 2);
			const riverDistance = Math.abs(nx - riverCenterNX(nz)) * (TERRAIN.width / 2);

			// Pastikan rumput tidak masuk ke sungai atau keluar terrain.
			if (riverDistance < 3.0) continue;
			if (Math.abs(localX) > TERRAIN.width / 2 - 2) continue;

			const height = terrainHeight(localX, localZ);
			const tuft = createGrassTuft(rightBankRandom);

			tuft.position.set(
				TERRAIN.centerX + localX,
				TERRAIN.centerY + height,
				TERRAIN.centerZ + localZ
			);

			grassGroup.add(tuft);

			// Kadang tambahkan satu rumpun kecil di dekat rumpun utama.
			if (rightBankRandom() > 0.55) {
				const nearbyX = localX + randomRange(rightBankRandom, -0.45, 0.45);

				const nearbyZ = localZ + randomRange(rightBankRandom, -0.35, 0.35);

				const nearbyHeight = terrainHeight(nearbyX, nearbyZ);

				const nearbyTuft = createGrassTuft(rightBankRandom);
				nearbyTuft.scale.multiplyScalar(0.7);

				nearbyTuft.position.set(
					TERRAIN.centerX + nearbyX,
					TERRAIN.centerY + nearbyHeight,
					TERRAIN.centerZ + nearbyZ
				);

				grassGroup.add(nearbyTuft);
			}
		}

		// ================================================
		// GRASS AROUND FOREGROUND RIVER ROCKS
		// ================================================

		const foregroundGrassRandom = createRandom(84217);

		// Area foreground yang berdekatan dengan batu sungai.
		// Koordinat Z mengikuti sistem koordinat lokal terrain.
		const foregroundAnchors = [
			// Sisi kiri sungai: area yang dilingkari merah
			{ z: 15.0, side: -1, count: 3 },
			{ z: 16.0, side: -1, count: 4 },
			{ z: 17.0, side: -1, count: 4 },
			{ z: 17.5, side: -1, count: 4 },
			{ z: 18.0, side: -1, count: 3 },

			// Sisi kanan sungai: pertahankan sebaran sebelumnya
			{ z: 13.5, side: 1, count: 3 },
			{ z: 15.0, side: 1, count: 4 },
			{ z: 16.0, side: 1, count: 3 }
		];

		for (const anchor of foregroundAnchors) {
			for (let i = 0; i < anchor.count; i++) {
				const grassZ = anchor.z + randomRange(foregroundGrassRandom, -0.5, 0.5);

				const nz = grassZ / (TERRAIN.depth / 2);

				const riverCenterX = riverCenterNX(nz) * (TERRAIN.width / 2);

				const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

				const t = (grassZ - (-TERRAIN.depth / 2 + 0.5)) / (TERRAIN.depth - 1);

				const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0, 0.12);

				const riverWidth = 3.8 * widthVariation * endFade;

				// Variasikan jarak rumpun dari tepian air.
				const bankOffset = randomRange(foregroundGrassRandom, 0.3, 0.85);

				const localX = riverCenterX + anchor.side * (riverWidth + bankOffset);

				const nx = localX / (TERRAIN.width / 2);

				const riverDistance = Math.abs(nx - riverCenterNX(nz)) * (TERRAIN.width / 2);

				if (riverDistance < 3.0) continue;

				if (Math.abs(localX) > TERRAIN.width / 2 - 2 || Math.abs(grassZ) > TERRAIN.depth / 2 - 1) {
					continue;
				}

				const height = terrainHeight(localX, grassZ);
				const tuft = createGrassTuft(foregroundGrassRandom);

				tuft.scale.multiplyScalar(randomRange(foregroundGrassRandom, 0.65, 0.9));

				tuft.position.set(
					TERRAIN.centerX + localX,
					TERRAIN.centerY + height,
					TERRAIN.centerZ + grassZ
				);

				grassGroup.add(tuft);
			}
		}

		// ================================================
		// EXTRA GRASS - LEFT FOREGROUND
		// ================================================

		const leftForegroundRandom = createRandom(91357);

		const leftForegroundAnchors = [
			{ z: 15.5, count: 3 },
			{ z: 16.5, count: 4 },
			{ z: 17.5, count: 4 }
		];

		for (const anchor of leftForegroundAnchors) {
			for (let i = 0; i < anchor.count; i++) {
				const grassZ = anchor.z + randomRange(leftForegroundRandom, -0.35, 0.35);

				const nz = grassZ / (TERRAIN.depth / 2);
				const riverCenterX = riverCenterNX(nz) * (TERRAIN.width / 2);

				const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

				const t = (grassZ - (-TERRAIN.depth / 2 + 0.5)) / (TERRAIN.depth - 1);

				const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0, 0.12);

				const riverWidth = 3.8 * widthVariation * endFade;

				const localX = riverCenterX - riverWidth - randomRange(leftForegroundRandom, 0.35, 0.8);

				const nx = localX / (TERRAIN.width / 2);

				const riverDistance = Math.abs(nx - riverCenterNX(nz)) * (TERRAIN.width / 2);

				if (riverDistance < 3.0) continue;

				const height = terrainHeight(localX, grassZ);
				const tuft = createGrassTuft(leftForegroundRandom);

				tuft.scale.multiplyScalar(randomRange(leftForegroundRandom, 0.6, 0.85));

				tuft.position.set(
					TERRAIN.centerX + localX,
					TERRAIN.centerY + height,
					TERRAIN.centerZ + grassZ
				);

				grassGroup.add(tuft);
			}
		}

		// ================================================
		// GRASS SIDE ROCKS
		// ================================================
		const rockSideGrassRandom = createRandom(8612);
		let rockSideGrassId = 0;

		for (let localZ = 1; localZ <= 16; localZ += 1.25) {
			for (const side of [-1, 1]) {
				// Harus sama dengan peluang skip di SnowRocks.svelte.
				if (rockSideGrassRandom() < 0.35) continue;

				const rockZ = localZ + randomRange(rockSideGrassRandom, -0.35, 0.35);

				const nz = rockZ / (TERRAIN.depth / 2);
				const riverCenterX = riverCenterNX(nz) * (TERRAIN.width / 2);

				const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

				const t = (rockZ - (-TERRAIN.depth / 2 + 0.5)) / (TERRAIN.depth - 1);

				const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0, 0.12);

				const riverWidth = 3.8 * widthVariation * endFade;

				// Konsumsi angka random yang sama dengan loop batu.
				const bankOffset = riverWidth + randomRange(rockSideGrassRandom, 0.15, 0.5);

				// Ukuran batu tidak diperlukan untuk rumput, tetapi angka
				// random ini harus tetap dikonsumsi agar urutan seed cocok.
				randomRange(rockSideGrassRandom, 0.16, 0.34);

				// Fokus pada tepi kiri sungai.
				if (side !== -1) continue;

				// Letakkan rumput sedikit ke arah luar dari batu.
				const grassX = riverCenterX - bankOffset - randomRange(rockSideGrassRandom, 0.12, 0.32);

				// Batasi agar tidak terlalu dekat dengan air atau tepi terrain.
				const nx = grassX / (TERRAIN.width / 2);
				const riverDistance = Math.abs(nx - riverCenterNX(nz)) * (TERRAIN.width / 2);

				if (riverDistance < 3.0) continue;
				if (Math.abs(grassX) > TERRAIN.width / 2 - 2) continue;
				if (Math.abs(rockZ) > TERRAIN.depth / 2 - 1) continue;

				const height = terrainHeight(grassX, rockZ);
				const tuft = createGrassTuft(createRandom(61000 + rockSideGrassId * 31));

				tuft.scale.multiplyScalar(randomRange(rockSideGrassRandom, 0.65, 0.9));

				tuft.position.set(
					TERRAIN.centerX + grassX,
					TERRAIN.centerY + height,
					TERRAIN.centerZ + rockZ
				);

				grassGroup.add(tuft);
				rockSideGrassId++;

				// Sebagian batu diberi satu rumpun tambahan agar terlihat alami.
				if (rockSideGrassRandom() > 0.55) {
					const extraX = grassX - randomRange(rockSideGrassRandom, 0.12, 0.25);
					const extraZ = rockZ + randomRange(rockSideGrassRandom, -0.18, 0.18);

					const extraHeight = terrainHeight(extraX, extraZ);
					const extraTuft = createGrassTuft(createRandom(72000 + rockSideGrassId * 37));

					extraTuft.scale.multiplyScalar(0.65);

					extraTuft.position.set(
						TERRAIN.centerX + extraX,
						TERRAIN.centerY + extraHeight,
						TERRAIN.centerZ + extraZ
					);

					grassGroup.add(extraTuft);
					rockSideGrassId++;
				}
			}
		}

		// ================================================
		// CLUSTER GRASS AROUND LEFT-BANK ROCKS
		// ================================================

		const rockSimulationRandom = createRandom(8612);
		const clusterGrassRandom = createRandom(92541);

		for (let localZ = 1; localZ <= 16; localZ += 1.25) {
			for (const side of [-1, 1]) {
				if (rockSimulationRandom() < 0.35) continue;

				const rockZ = localZ + randomRange(rockSimulationRandom, -0.35, 0.35);

				const nz = rockZ / (TERRAIN.depth / 2);
				const riverCenterX = riverCenterNX(nz) * (TERRAIN.width / 2);

				const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

				const t = (rockZ - (-TERRAIN.depth / 2 + 0.5)) / (TERRAIN.depth - 1);

				const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0, 0.12);

				const riverWidth = 3.8 * widthVariation * endFade;

				const rockOffset = riverWidth + randomRange(rockSimulationRandom, 0.15, 0.5);

				const rockSize = randomRange(rockSimulationRandom, 0.16, 0.34);

				// Fokus pada sisi kiri sungai.
				if (side !== -1) continue;

				// Pilih sebagian batu agar rumput tidak terlalu padat.
				if (clusterGrassRandom() < 0.35) continue;

				const rockX = riverCenterX - rockOffset;

				// Rumpun utama berada dekat sisi luar batu.
				const baseX = rockX - rockSize * 1.1 - randomRange(clusterGrassRandom, 0.08, 0.18);

				const baseZ = rockZ + randomRange(clusterGrassRandom, -0.12, 0.12);

				const positions = [
					{ x: baseX, z: baseZ, scale: 0.72 },
					{
						x: baseX - randomRange(clusterGrassRandom, 0.12, 0.22),
						z: baseZ + randomRange(clusterGrassRandom, -0.16, 0.16),
						scale: 0.55
					}
				];

				for (const position of positions) {
					const nx = position.x / (TERRAIN.width / 2);
					const pz = position.z / (TERRAIN.depth / 2);

					const riverDistance = Math.abs(nx - riverCenterNX(pz)) * (TERRAIN.width / 2);

					if (riverDistance < 3.0) continue;
					if (Math.abs(position.x) > TERRAIN.width / 2 - 2) continue;
					if (Math.abs(position.z) > TERRAIN.depth / 2 - 1) continue;

					const tuft = createGrassTuft(clusterGrassRandom);
					tuft.scale.multiplyScalar(position.scale * randomRange(clusterGrassRandom, 0.85, 1.15));

					tuft.position.set(
						TERRAIN.centerX + position.x,
						TERRAIN.centerY + terrainHeight(position.x, position.z),
						TERRAIN.centerZ + position.z
					);

					grassGroup.add(tuft);
				}
			}
		}

		scene.add(grassGroup);

		// ================================================
		// CLEANUP
		// ================================================

		return () => {
			scene.remove(grassGroup);

			for (const geometry of geometries) {
				geometry.dispose();
			}

			for (const material of materials) {
				material.dispose();
			}
		};
	});
</script>
