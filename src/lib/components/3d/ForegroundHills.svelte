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
					// Bukit ketiga: lereng lebar seperti garis putih pada referensi.
					const slope = px * 0.16;

					// Membuat permukaan naik secara halus menuju kanan.
					const broadCurve = Math.exp(-Math.pow((nx + 0.25) * 1.2, 2)) * 1.1;

					// Variasi kecil agar lereng tidak terlihat benar-benar datar.
					const gentleCurve = Math.sin(px * 0.06 + seed) * 0.15 + Math.sin(pz * 0.1 + seed) * 0.08;

					terrainHeight = (slope + broadCurve + gentleCurve) * (height / 4.2);
				} else {
					const diagonalSlope = seed === 28 ? px * 0.08 : px * -0.22;

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
				color: isForeground ? '#080a0d' : color,
				roughness: 1,
				metalness: 0,
				side: THREE.DoubleSide,
				flatShading: false
			});

			const hill = new THREE.Mesh(geometry, material);

			hill.position.set(x, y, z);
			hill.receiveShadow = true;

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
			color: '#111619',
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
			color: '#0b0d11',
			seed: 100,
			ridgeOffset: -1.5,
			ridgeWidth: 5
		});

		// 3. Bukit kanan: punggung landai yang naik ke kanan.
		createForegroundHill({
			width: 25,
			depth: 22,
			height: 4,
			x: 19,
			y: -2.5,
			z: 2,
			color: '#101216',
			seed: 28,
			ridgeOffset: 1.5,
			ridgeWidth: 6
		});

		// 4. Bukit belakang kanan: rendah dan memanjang.
		createForegroundHill({
			width: 30,
			depth: 20,
			height: 3.5,
			x: 25,
			y: 1,
			z: -4,
			color: '#191d24',
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
