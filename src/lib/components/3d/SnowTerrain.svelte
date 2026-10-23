<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	let { scene }: Props = $props();

	const TERRAIN = {
		centerX: 8,
		centerY: -0.5,
		centerZ: -18,
		width: 70,
		depth: 38,
		segmentsX: 280,
		segmentsZ: 180
	} as const;

	// Jalur sungai dalam koordinat lokal terrain.
	function riverCenterNX(nz: number): number {
		return -0.35 - 0.22 * nz + 0.07 * Math.sin(nz * 3);
	}

	// Ketinggian permukaan tanah dan cekungan sungai.
	function terrainHeight(x: number, z: number): number {
		const nx = x / (TERRAIN.width / 2);
		const nz = z / (TERRAIN.depth / 2);

		const ridgeCenter = -0.28 + 0.2 * nx * nx + 0.035 * Math.sin(nx * 3);
		const distance = nz - ridgeCenter;

		const ridge = 0.9 * Math.exp(-Math.pow(distance / 0.58, 2));
		const shoulder = 0.38 * Math.exp(-Math.pow(distance / 1.0, 2));
		const frontSlope = -0.4 * THREE.MathUtils.smoothstep(distance, 0.05, 1.0);

		// Cekungan sungai tetap mengikuti jalur yang sudah ada.
		const riverDistance = nx - riverCenterNX(nz);

		const riverMask = Math.exp(-Math.pow(riverDistance / 0.25, 4));
		const riverBed = -0.72 * riverMask;
		const riverBanks = 0.3 * Math.exp(-Math.pow((Math.abs(riverDistance) - 0.3) / 0.12, 2));

		// Gundukan salju pada sisi kiri dan kanan foreground.
		const sideMask = THREE.MathUtils.smoothstep(Math.abs(nx), 0.3, 0.95);
		const frontMask = THREE.MathUtils.smoothstep(nz, -0.05, 0.85);
		const sideRise = 0.28 * sideMask * frontMask;

		// Gelombang dasar yang sudah ada.
		const rolling = 0.025 * Math.sin(nx * 2.4 + nz * 0.8) + 0.018 * Math.sin(nx * 4.0 - nz * 1.1);

		// Gundukan salju lebar dengan pola yang tidak terlalu teratur.
		const broadSnow =
			0.052 * Math.sin(x * 0.15 + z * 0.08) * Math.sin(z * 0.12 - x * 0.045) +
			0.022 * Math.sin(x * 0.095 - z * 0.13 + 1.4);

		// Detail kecil untuk memecah permukaan yang terlalu rata.
		const fineSnow = 0.007 * Math.sin(x * 0.34 + z * 0.19) * Math.sin(z * 0.25 - x * 0.14);

		// Efek kontur lebih terasa di foreground.
		const foregroundMask = THREE.MathUtils.smoothstep(nz, -0.05, 0.8);

		// Kurangi perubahan bentuk tepat di dekat tepi sungai.
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

	onMount(() => {
		// ================================================
		// LEFT AND RIGHT SNOW BANKS
		// ================================================

		const terrainSegments = 220;
		const depthSegments = TERRAIN.segmentsZ;

		// Batas sungai dibuat sedikit lebih lebar daripada air
		// agar tepian salju tidak menutupi permukaan air.
		const bankHalfWidth = 2.6;

		function createSnowBank(isLeft: boolean): THREE.Mesh {
			const vertices: number[] = [];
			const colors: number[] = [];
			const indices: number[] = [];

			const rows = depthSegments + 1;
			const columns = terrainSegments + 1;

			for (let iz = 0; iz < rows; iz++) {
				const t = iz / depthSegments;
				const localZ = THREE.MathUtils.lerp(-TERRAIN.depth / 2, TERRAIN.depth / 2, t);

				const nz = localZ / (TERRAIN.depth / 2);
				const centerX = riverCenterNX(nz) * (TERRAIN.width / 2);

				for (let ix = 0; ix < columns; ix++) {
					const u = ix / terrainSegments;

					// Sisi kiri berakhir di tepi kiri sungai;
					// sisi kanan dimulai di tepi kanan sungai.
					const localX = isLeft
						? THREE.MathUtils.lerp(-TERRAIN.width / 2, centerX - bankHalfWidth, u)
						: THREE.MathUtils.lerp(centerX + bankHalfWidth, TERRAIN.width / 2, u);

					const height = terrainHeight(localX, localZ);

					vertices.push(localX, height, localZ);

					const normalizedHeight = THREE.MathUtils.clamp((height + 0.85) / 1.8, 0, 1);

					const color = new THREE.Color();

					if (normalizedHeight < 0.48) {
						color.set('#414a5b').lerp(new THREE.Color('#929bad'), normalizedHeight / 0.48);
					} else {
						color.set('#929bad').lerp(new THREE.Color('#d8d5cf'), (normalizedHeight - 0.48) / 0.52);
					}

					// Variasi warna halus untuk menghindari permukaan salju yang monoton.
					const nx = localX / (TERRAIN.width / 2);

					// Gunakan nz yang sudah tersedia dari loop iz.
					const broadVariation = Math.sin(nx * 5.5 + nz * 2.2) * Math.sin(nz * 3.8 - nx * 2.7);

					const fineVariation = Math.sin(nx * 13.0 + nz * 7.0) * Math.sin(nz * 10.0 - nx * 6.0);

					const variation = broadVariation * 0.018 + fineVariation * 0.004;

					const snowTint = new THREE.Color(variation >= 0 ? '#e3e8f0' : '#8995a8');

					color.lerp(snowTint, Math.abs(variation));

					colors.push(color.r, color.g, color.b);

					if (iz < rows - 1 && ix < columns - 1) {
						const a = iz * columns + ix;
						const b = a + 1;
						const c = a + columns;
						const d = c + 1;

						indices.push(a, c, b, b, c, d);
					}
				}
			}

			const geometry = new THREE.BufferGeometry();

			geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
			geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));
			geometry.setIndex(indices);
			geometry.computeVertexNormals();

			const material = new THREE.MeshStandardMaterial({
				color: '#ffffff',
				vertexColors: true,
				roughness: 1,
				metalness: 0,
				side: THREE.DoubleSide
			});

			const bank = new THREE.Mesh(geometry, material);

			bank.position.set(TERRAIN.centerX, TERRAIN.centerY, TERRAIN.centerZ);

			bank.receiveShadow = true;
			bank.castShadow = true;
			bank.name = isLeft ? 'LeftSnowBank' : 'RightSnowBank';

			return bank;
		}

		const leftSnow = createSnowBank(true);
		const rightSnow = createSnowBank(false);

		scene.add(leftSnow, rightSnow);

		// ================================================
		// RIVER WATER
		// ================================================

		const riverSegments = 240;
		const halfWidth = 3.8;

		const riverVertices: number[] = [];
		const riverIndices: number[] = [];

		for (let i = 0; i <= riverSegments; i++) {
			const t = i / riverSegments;

			// Membentang dari belakang sampai foreground.
			const localZ = THREE.MathUtils.lerp(-TERRAIN.depth / 2 + 0.5, TERRAIN.depth / 2 - 0.5, t);

			const nz = localZ / (TERRAIN.depth / 2);
			const centerNX = riverCenterNX(nz);
			const centerLocalX = centerNX * (TERRAIN.width / 2);

			// Ketinggian air ditentukan dari dasar cekungan
			// di tengah jalur, bukan dari kedua tepi sungai.
			const waterY = TERRAIN.centerY + terrainHeight(centerLocalX, localZ) + 0.055;

			const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

			const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0.0, 0.12);

			const riverWidth = halfWidth * widthVariation * endFade;

			for (const side of [-1, 1]) {
				const localX = centerLocalX + side * riverWidth;

				riverVertices.push(TERRAIN.centerX + localX, waterY, TERRAIN.centerZ + localZ);
			}

			if (i < riverSegments) {
				const a = i * 2;
				const b = a + 1;
				const c = a + 2;
				const d = a + 3;

				riverIndices.push(a, c, b, b, c, d);
			}
		}

		const riverGeometry = new THREE.BufferGeometry();

		riverGeometry.setAttribute('position', new THREE.Float32BufferAttribute(riverVertices, 3));

		riverGeometry.setIndex(riverIndices);
		riverGeometry.computeVertexNormals();

		const waterMaterial = new THREE.MeshStandardMaterial({
			color: '#31596a',
			roughness: 0.28,
			metalness: 0.08,
			emissive: '#07151b',
			emissiveIntensity: 0.15,
			side: THREE.DoubleSide
		});

		const river = new THREE.Mesh(riverGeometry, waterMaterial);

		river.name = 'RiverWater';
		river.receiveShadow = false;
		river.castShadow = false;
		river.renderOrder = 2;

		scene.add(river);


		return () => {
			scene.remove(river);
			scene.remove(leftSnow);
			scene.remove(rightSnow);

			riverGeometry.dispose();
			waterMaterial.dispose();

			for (const bank of [leftSnow, rightSnow]) {
				bank.geometry.dispose();

				const material = bank.material;
				if (material instanceof THREE.Material) {
					material.dispose();
				}
			}
		};
	});
</script>
