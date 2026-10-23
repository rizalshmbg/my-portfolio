<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	let { scene }: { scene: THREE.Scene } = $props();

	onMount(() => {
		const path = new THREE.Group();
		path.name = 'CabinPath';

		// Material batu yang cocok dengan suasana bersalju.
		const stoneMaterial = new THREE.MeshStandardMaterial({
			color: '#62676c',
			roughness: 1,
			flatShading: true
		});

		const snowMaterial = new THREE.MeshStandardMaterial({
			color: '#e0e4e8',
			roughness: 1,
			flatShading: true
		});

		const stones = [
			{ x: -14.3, z: -22.2, width: 0.28, depth: 0.32, rotation: -0.15 },
			{ x: -14.5, z: -21.0, width: 0.5, depth: 0.34, rotation: 0.1 },
			{ x: -14.7, z: -20.1, width: 0.48, depth: 0.32, rotation: -0.1 },
			{ x: -15.3, z: -19.8, width: 0.52, depth: 0.34, rotation: 0.15 },
			{ x: -16.0, z: -19.5, width: 0.48, depth: 0.32, rotation: -0.08 },
			{ x: -16.8, z: -19.2, width: 0.45, depth: 0.3, rotation: 0.1 }
		];

		const smallStones = [
			{ x: -14.1, z: -21.8, size: 0.16, rotation: 0.2 },
			{ x: -14.8, z: -21.5, size: 0.12, rotation: -0.3 },
			{ x: -14.2, z: -20.7, size: 0.14, rotation: 0.1 },
			{ x: -15.0, z: -20.4, size: 0.17, rotation: -0.2 },
			{ x: -14.9, z: -19.9, size: 0.12, rotation: 0.4 },
			{ x: -15.6, z: -19.7, size: 0.15, rotation: -0.1 },
			{ x: -15.7, z: -19.2, size: 0.12, rotation: 0.3 },
			{ x: -16.4, z: -19.4, size: 0.16, rotation: -0.2 },
			{ x: -16.5, z: -18.9, size: 0.11, rotation: 0.2 }
		];

		const stoneGeometry = new THREE.BoxGeometry(1, 1, 1);

		for (let i = 0; i < stones.length; i++) {
			const item = stones[i];

			// Batu utama.
			const stone = new THREE.Mesh(stoneGeometry, stoneMaterial);

			stone.position.set(item.x, 0.4, item.z);
			stone.scale.set(item.width, 0.1, item.depth);
			stone.rotation.y = item.rotation;
			stone.castShadow = false;
			stone.receiveShadow = true;
			path.add(stone);

			// Lapisan tipis salju di atas batu.
			const snow = new THREE.Mesh(stoneGeometry, snowMaterial);

			snow.position.set(item.x, 0.455, item.z);
			snow.scale.set(item.width * 0.96, 0.025, item.depth * 0.92);
			snow.rotation.y = item.rotation;
			snow.castShadow = false;
			snow.receiveShadow = true;
			path.add(snow);
		}

		const smallStoneGeometry = new THREE.DodecahedronGeometry(0.5, 0);

		const smallStoneMaterial = new THREE.MeshStandardMaterial({
			color: '#777b7e',
			roughness: 1,
			flatShading: true
		});

		for (const item of smallStones) {
			const stone = new THREE.Mesh(smallStoneGeometry, smallStoneMaterial);

			stone.position.set(item.x, 0.39, item.z);
			stone.scale.set(item.size, item.size * 0.35, item.size * 0.8);
			stone.rotation.y = item.rotation;
			stone.castShadow = false;
			stone.receiveShadow = true;
			stone.name = 'CabinPathSmallStone';

			path.add(stone);
		}

		scene.add(path);

		return () => {
			scene.remove(path);
			stoneGeometry.dispose();
			stoneMaterial.dispose();
			smallStoneGeometry.dispose();
			smallStoneMaterial.dispose();
			snowMaterial.dispose();
		};
	});
</script>
