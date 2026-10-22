<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	interface ProceduralTreeConfig {
		x: number;
		y: number;
		z: number;
		height: number;
		rotationY?: number;
		seed: number;
	}

	let { scene }: Props = $props();

	onMount(() => {
		const proceduralGroups: THREE.Group[] = [];
		const geometries: THREE.BufferGeometry[] = [];
		const materials: THREE.Material[] = [];

		const proceduralConfigs: ProceduralTreeConfig[] = [
			{ x: -20, y: -1.5, z: -15, height: 30, seed: 33 },
			{ x: -11, y: -2, z: -7, height: 10, seed: 11 },
			{ x: 11, y: -1.5, z: -15, height: 5.3, seed: 44 },
			{ x: 12, y: -1.5, z: -10, height: 15.4, seed: 55 },
			{ x: 17, y: -1.5, z: -12, height: 13.6, seed: 66 }
		];

		function addBranch(
			group: THREE.Group,
			start: THREE.Vector3,
			end: THREE.Vector3,
			radius: number,
			material: THREE.Material
		) {
			const direction = new THREE.Vector3().subVectors(end, start);
			const length = direction.length();

			if (length <= 0) return;

			const geometry = new THREE.CylinderGeometry(radius * 0.15, radius, length, 5);

			const branch = new THREE.Mesh(geometry, material);

			branch.position.copy(start).add(end).multiplyScalar(0.5);
			branch.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), direction.normalize());

			branch.castShadow = true;
			branch.receiveShadow = true;

			group.add(branch);
			geometries.push(geometry);
		}

		function createProceduralTree(config: ProceduralTreeConfig) {
			const group = new THREE.Group();
			const height = config.height;

			const barkMaterial = new THREE.MeshStandardMaterial({
				color: '#343943',
				roughness: 1,
				metalness: 0
			});

			materials.push(barkMaterial);

			const trunkGeometry = new THREE.CylinderGeometry(0, height * 0.035, height, 7);
			geometries.push(trunkGeometry);

			const trunk = new THREE.Mesh(trunkGeometry, barkMaterial);
			trunk.position.y = height / 2;
			trunk.rotation.z = Math.sin(config.seed) * 0.055;
			trunk.castShadow = true;
			trunk.receiveShadow = true;

			group.add(trunk);

			// Variasi deterministik agar bentuk konsisten setiap refresh.
			let randomSeed = config.seed;

			function random() {
				randomSeed = (randomSeed * 16807) % 2147483647;
				return (randomSeed - 1) / 2147483646;
			}

			// Cabang tumbuh dari bawah ke atas.
			const branchCount = 7;

			for (let i = 0; i < branchCount; i++) {
				const t = 0.3 + (i / branchCount) * 0.62;
				const startY = height * t;

				const angle = i * 2.399 + random() * 0.45;
				const branchLength = height * (0.24 + random() * 0.12);

				const start = new THREE.Vector3(0, startY, 0);

				const end = new THREE.Vector3(
					Math.cos(angle) * branchLength,
					startY + branchLength * (0.3 + random() * 0.25),
					Math.sin(angle) * branchLength
				);

				addBranch(group, start, end, height * 0.018 * (1 - t * 0.55), barkMaterial);

				// Cabang kecil pada sebagian cabang utama.
				if (i % 2 === 0) {
					const middle = start.clone().lerp(end, 0.65);
					const twigLength = branchLength * 0.48;
					const twigAngle = angle + (random() > 0.5 ? 1 : -1) * 0.9;

					const twigEnd = middle
						.clone()
						.add(
							new THREE.Vector3(
								Math.cos(twigAngle) * twigLength,
								twigLength * 0.55,
								Math.sin(twigAngle) * twigLength
							)
						);

					addBranch(group, middle, twigEnd, height * 0.009, barkMaterial);
				}
			}

			group.position.set(config.x, config.y, config.z);
			group.rotation.y = config.rotationY ?? 0;

			scene.add(group);
			proceduralGroups.push(group);
		}

		for (const config of proceduralConfigs) {
			createProceduralTree(config);
		}

		return () => {
			for (const group of proceduralGroups) {
				scene.remove(group);
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
