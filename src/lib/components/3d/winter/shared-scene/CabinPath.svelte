<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	let { scene }: { scene: THREE.Scene } = $props();

	// Konfigurasi posisi dan ukuran terrain.
	const TERRAIN = {
		centerX: 8,
		centerY: -0.5,
		centerZ: -25,
		width: 70,
		depth: 52
	} as const;

	// Posisi dan ukuran batu dapat disesuaikan nanti.
	const PATH = {
		startX: -16,
		startZ: -22.5,
		spacing: 0.7,
		stoneCount: 6,
		width: 0.48,
		height: 0.1,
		depth: 0.34,
		embedDepth: 0.05
	} as const;

	// Mengikuti rumus ketinggian permukaan dari SnowTerrain.svelte.
	function riverCenterNX(nz: number): number {
		return -0.35 - 0.22 * nz + 0.07 * Math.sin(nz * 3);
	}

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

	// Menghitung tinggi salju berdasarkan koordinat dunia.
	function getGroundY(x: number, z: number): number {
		const localX = x - TERRAIN.centerX;
		const localZ = z - TERRAIN.centerZ;

		return TERRAIN.centerY + terrainHeight(localX, localZ);
	}

	onMount(() => {
		const pathGroup = new THREE.Group();
		pathGroup.name = 'CabinPath';

		const stoneGeometry = new THREE.BoxGeometry(1, 1, 1);

		const stoneMaterial = new THREE.MeshStandardMaterial({
			color: '#62676c',
			roughness: 1,
			metalness: 0
		});

		for (let i = 0; i < PATH.stoneCount; i++) {
			const x = PATH.startX;
			const z = PATH.startZ + i * PATH.spacing;

			const groundY = getGroundY(x, z);

			const stone = new THREE.Mesh(stoneGeometry, stoneMaterial);

			stone.name = `CabinPathStone_${i + 1}`;

			stone.scale.set(PATH.width, PATH.height, PATH.depth);

			// Sedikit variasi rotasi agar batu tidak terlihat terlalu kaku.
			stone.rotation.y = (i % 2 === 0 ? 1 : -1) * 0.035;

			// Bagian bawah batu masuk sedikit ke dalam permukaan salju.
			stone.position.set(x, groundY + PATH.height / 2 - PATH.embedDepth, z);

			stone.castShadow = true;
			stone.receiveShadow = true;

			pathGroup.add(stone);
		}

		scene.add(pathGroup);

		return () => {
			scene.remove(pathGroup);
			stoneGeometry.dispose();
			stoneMaterial.dispose();
		};
	});
</script>
