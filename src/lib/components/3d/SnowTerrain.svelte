<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
		registerUpdate?: (callback: ((time: number) => void) | undefined) => void;
	}

	let { scene, registerUpdate }: Props = $props();

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

		// ================================================
		// SOFT MOONLIGHT REFLECTION
		// ================================================

		const reflectionGroup = new THREE.Group();
		reflectionGroup.name = 'RiverMoonReflection';

		const reflectionCount = 200;

		// Simpan data untuk animasi kilauan.
		const shimmerMeshes: {
			mesh: THREE.Mesh;
			baseOpacity: number;
			phase: number;
			speed: number;
			distanceFactor: number;
		}[] = [];

		for (let i = 0; i < reflectionCount; i++) {
			// Posisi tersebar di sepanjang sungai.
			// Gunakan distribusi nonlinier agar pantulan lebih
			// terkonsentrasi di bagian tengah hingga foreground.
			const t = 0.22 + Math.pow(Math.random(), 0.9) * 0.75;

			const localZ = THREE.MathUtils.lerp(-TERRAIN.depth / 2 + 0.5, TERRAIN.depth / 2 - 0.5, t);

			const nz = localZ / (TERRAIN.depth / 2);
			const centerNX = riverCenterNX(nz);
			const centerLocalX = centerNX * (TERRAIN.width / 2);

			const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

			const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0, 0.12);

			const riverWidth = halfWidth * widthVariation * endFade;

			// Sebarkan kilauan di dalam area tengah sungai.
			const spread = Math.random() < 0.75 ? 0.32 : 0.75;

			const offsetX = THREE.MathUtils.randFloat(-riverWidth * spread, riverWidth * spread);

			const localX = centerLocalX + offsetX;

			const middleFactor = THREE.MathUtils.smoothstep(t, 0.35, 0.65);
			const foregroundFactor = THREE.MathUtils.smoothstep(t, 0.58, 0.92);
			const waterY = TERRAIN.centerY + terrainHeight(localX, localZ) + 0.085;

			// Kilauan pendek dengan ukuran bervariasi.
			const lineWidth =
				THREE.MathUtils.randFloat(0.08, 0.2) * (1 + 0.2 * middleFactor + 0.35 * foregroundFactor);

			const lineLength =
				(Math.random() < 0.18
					? THREE.MathUtils.randFloat(0.05, 0.09)
					: THREE.MathUtils.randFloat(0.02, 0.05)) *
				(1 + 0.15 * middleFactor + 0.3 * foregroundFactor);

			const reflectionGeometry = new THREE.PlaneGeometry(lineWidth, lineLength);

			reflectionGeometry.rotateX(-Math.PI / 2);

			const brightChance = 0.2 + 0.08 * middleFactor + 0.08 * foregroundFactor;

			const isBright = Math.random() < brightChance;

			const baseOpacity = isBright
				? THREE.MathUtils.randFloat(0.55, 0.78)
				: THREE.MathUtils.randFloat(0.22, 0.42);

			const reflectionMaterial = new THREE.MeshBasicMaterial({
				color: isBright ? '#ffe8b5' : '#d4dce0',
				transparent: true,
				opacity: baseOpacity,
				side: THREE.DoubleSide,
				depthWrite: false,
				toneMapped: false
			});

			const reflection = new THREE.Mesh(reflectionGeometry, reflectionMaterial);

			reflection.position.set(TERRAIN.centerX + localX, waterY, TERRAIN.centerZ + localZ);

			reflection.rotation.y = THREE.MathUtils.randFloat(-0.22, 0.22);
			reflection.renderOrder = 3;

			reflectionGroup.add(reflection);

			shimmerMeshes.push({
				mesh: reflection,
				baseOpacity,
				phase: Math.random() * Math.PI * 2,
				speed: THREE.MathUtils.randFloat(0.6, 1.8),
				distanceFactor: Math.max(middleFactor, foregroundFactor)
			});
		}

		// Garis pantulan utama dibuat sangat samar.
		// Kilauan kecil menjadi fokus visual.
		const reflectionRibbonGeometry = new THREE.BufferGeometry();

		const ribbonVertices: number[] = [];
		const ribbonIndices: number[] = [];

		const segmentCount = 34;

		for (let i = 0; i < segmentCount; i++) {
			// Sisakan celah di antara setiap pantulan.
			const startT = 0.32 + (i / segmentCount) * 0.66;

			const endT = Math.min(
				startT + (0.66 / segmentCount) * THREE.MathUtils.randFloat(0.35, 0.8),
				0.98
			);

			for (let j = 0; j <= 2; j++) {
				const t = THREE.MathUtils.lerp(startT, endT, j / 2);

				const localZ = THREE.MathUtils.lerp(-TERRAIN.depth / 2 + 0.5, TERRAIN.depth / 2 - 0.5, t);

				const nz = localZ / (TERRAIN.depth / 2);
				const centerNX = riverCenterNX(nz);
				const centerLocalX = centerNX * (TERRAIN.width / 2);

				const widthVariation = 0.88 + 0.18 * Math.sin(nz * 3.2) + 0.06 * Math.sin(nz * 6.0);

				const endFade = THREE.MathUtils.smoothstep(Math.min(t, 1 - t), 0, 0.12);

				// Pantulan lebih lebar di bagian tengah segmen.
				const localSegmentT = j / 2;
				const shape = Math.sin(Math.PI * localSegmentT);

				const ribbonWidth =
					THREE.MathUtils.randFloat(0.09, 0.17) * widthVariation * (0.45 + 0.55 * shape) * endFade;

				const waterY = TERRAIN.centerY + terrainHeight(centerLocalX, localZ) + 0.14;

				ribbonVertices.push(
					TERRAIN.centerX + centerLocalX - ribbonWidth,
					waterY,
					TERRAIN.centerZ + localZ,

					TERRAIN.centerX + centerLocalX + ribbonWidth,
					waterY,
					TERRAIN.centerZ + localZ
				);

				if (j < 2) {
					const base = i * 6 + j * 2;

					ribbonIndices.push(base, base + 2, base + 1, base + 1, base + 2, base + 3);
				}
			}
		}

		reflectionRibbonGeometry.setAttribute(
			'position',
			new THREE.Float32BufferAttribute(ribbonVertices, 3)
		);

		reflectionRibbonGeometry.setIndex(ribbonIndices);
		reflectionRibbonGeometry.computeVertexNormals();

		const reflectionRibbonMaterial = new THREE.MeshBasicMaterial({
			color: '#f4d99c',
			transparent: true,
			opacity: 0.035,
			side: THREE.DoubleSide,
			depthWrite: false,
			toneMapped: false
		});

		const reflectionRibbon = new THREE.Mesh(reflectionRibbonGeometry, reflectionRibbonMaterial);

		reflectionRibbon.name = 'RiverMoonlightRibbon';
		reflectionRibbon.renderOrder = 3;

		reflectionGroup.add(reflectionRibbon);

		scene.add(river);
		scene.add(reflectionGroup);

		const updateRiverReflection = (time: number) => {
			for (const shimmer of shimmerMeshes) {
				const material = shimmer.mesh.material;

				if (!(material instanceof THREE.MeshBasicMaterial)) {
					continue;
				}

				const foreground = shimmer.distanceFactor;

				// Perubahan terang-redup lebih kuat di foreground.
				const waveStrength = 0.35 + 0.4 * foreground;

				const wave =
					1 -
					waveStrength +
					waveStrength * (0.5 + 0.5 * Math.sin(time * shimmer.speed + shimmer.phase));

				const secondaryWave =
					0.9 + 0.1 * Math.sin(time * shimmer.speed * 0.65 + shimmer.phase * 1.8);

				material.opacity = shimmer.baseOpacity * wave * secondaryWave;
			}
		};

		registerUpdate?.(updateRiverReflection);

		// ================================================
		// CLEANUP
		// ================================================

		return () => {
			// Hapus callback animasi dari NightScene
			registerUpdate?.(undefined);

			scene.remove(river);
			scene.remove(leftSnow);
			scene.remove(rightSnow);
			scene.remove(reflectionGroup);

			reflectionGroup.children.forEach((child) => {
				if (child instanceof THREE.Mesh) {
					child.geometry.dispose();

					const material = child.material;

					if (Array.isArray(material)) {
						material.forEach((item) => item.dispose());
					} else {
						material.dispose();
					}
				}
			});

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
