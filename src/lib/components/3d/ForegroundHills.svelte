<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	let { scene }: Props = $props();

	onMount(() => {
		const hills: THREE.Mesh[] = [];

		interface ForegroundHillConfig {
			width: number;
			depth: number;
			height: number;
			x: number;
			y: number;
			z: number;
			color: string;
			seed: number;
			ridgeOffset: number;
			ridgeWidth: number;
		}

		function createForegroundHill(config: ForegroundHillConfig) {
			const { width, depth, height, x, y, z, color, seed } = config;

			const segmentsX = 100;
			const segmentsZ = 48;

			const geometry = new THREE.PlaneGeometry(width, depth, segmentsX, segmentsZ);

			geometry.rotateX(-Math.PI / 2);

			const positions = geometry.attributes.position;

			for (let i = 0; i < positions.count; i++) {
				const px = positions.getX(i);
				const pz = positions.getZ(i);

				const nx = px / (width / 2);

				let terrainHeight: number;

				if (seed === 42) {
					// Lereng diagonal: rendah di kiri, tinggi di kanan.
					const slope = px * 0.24;

					// Punggung bukit yang lebih tinggi di bagian kanan.
					const ridge = Math.exp(-Math.pow((nx - 0.55) * 1.3, 2)) * 1.4;

					// Variasi kecil agar permukaan tidak terlalu geometris.
					const gentleCurve = Math.sin(px * 0.045 + seed) * 0.1 + Math.sin(pz * 0.08 + seed) * 0.06;

					terrainHeight = (slope + ridge + gentleCurve) * (height / 4.2);
				} else if (seed === 28) {
					// Bukit 3: lereng foreground kanan, naik menuju tepi kanan.
					const slope = px * 0.28;

					// Membuat punggung bukit sedikit membulat.
					const ridge = Math.exp(-Math.pow((nx - 0.45) * 1.5, 2)) * 0.65;

					// Variasi halus agar permukaan tidak terlalu lurus.
					const gentleCurve = Math.sin(px * 0.06 + seed) * 0.08 + Math.sin(pz * 0.09 + seed) * 0.05;

					terrainHeight = (slope + ridge + gentleCurve) * (height / 4.2);
				} else {
					const diagonalSlope = px * -0.22;

					const gentleCurve =
						Math.sin(px * 0.075 + seed) * 0.35 + Math.sin(px * 0.035 + seed * 0.7) * 0.25;

					const foregroundCurve = Math.exp(-Math.pow((nx + 0.15) * 1.8, 2)) * 0.45;

					const depthVariation = Math.sin(pz * 0.12 + seed) * 0.12;

					terrainHeight =
						(diagonalSlope + gentleCurve + foregroundCurve + depthVariation) * (height / 4.2);
				}

				positions.setY(i, terrainHeight);
			}

			geometry.computeVertexNormals();

			const isForeground = seed === 57;

			const material = new THREE.MeshStandardMaterial({
				color: isForeground ? '#242b36' : color,
				roughness: 1,
				metalness: 0,
				side: THREE.DoubleSide,
				flatShading: false
			});

			material.shadowSide = THREE.DoubleSide;

			const hill = new THREE.Mesh(geometry, material);

			hill.position.set(x, y, z);
			hill.receiveShadow = true;
			hill.castShadow = true;

			scene.add(hill);
			hills.push(hill);
		}

		// 1. Bukit 1
		createForegroundHill({
			width: 38,
			depth: 28,
			height: 4.2,
			x: -18,
			y: -0.5,
			z: 2,
			color: '#a4adba',
			seed: 10,
			ridgeOffset: -1.5,
			ridgeWidth: 5
		});

		// 2. Bukit 2
		createForegroundHill({
			width: 18,
			depth: 19,
			height: 4.2,
			x: -6,
			y: -4,
			z: -9,
			color: '#737f91',
			seed: 100,
			ridgeOffset: -1.5,
			ridgeWidth: 5
		});

		// 3. Bukit 3
		createForegroundHill({
			width: 25,
			depth: 22,
			height: 3.2,
			x: 7,
			y: -2.5,
			z: 5,
			color: '#b6bfcc',
			seed: 28,
			ridgeOffset: 1.5,
			ridgeWidth: 6
		});

		// 4. Bukit 4
		createForegroundHill({
			width: 30,
			depth: 20,
			height: 4.5,
			x: 24,
			y: 2,
			z: -5,
			color: '#c5cbd3',
			seed: 42,
			ridgeOffset: 1,
			ridgeWidth: 6
		});

		return () => {
			for (const hill of hills) {
				scene.remove(hill);
				hill.geometry.dispose();

				const material = hill.material;

				if (material instanceof THREE.Material) {
					material.dispose();
				}
			}
		};
	});
</script>
