<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
	import { clone as cloneSkeleton } from 'three/addons/utils/SkeletonUtils.js';

	interface Props {
		scene: THREE.Scene;
		registerUpdate?: (callback: ((time: number) => void) | undefined) => void;
	}

	let { scene, registerUpdate }: Props = $props();

	onMount(() => {
		let disposed = false;
		let previousTime = 0;

		const loader = new GLTFLoader();

		const wolfGroups: THREE.Group[] = [];
		const mixers: THREE.AnimationMixer[] = [];
		const pawPrintGroups: THREE.Object3D[] = [];
		const sourceGeometries: THREE.BufferGeometry[] = [];
		const sourceMaterials: THREE.Material[] = [];

		const pawShape = new THREE.Shape();

		// Mulai dari sisi kiri bantalan.
		pawShape.moveTo(-0.13, 0.08);

		// Jari paling kiri.
		pawShape.bezierCurveTo(-0.18, 0.04, -0.17, -0.04, -0.14, -0.08);
		pawShape.bezierCurveTo(-0.11, -0.11, -0.085, -0.06, -0.08, -0.025);

		// Jari kedua.
		pawShape.bezierCurveTo(-0.075, 0.005, -0.09, -0.12, -0.045, -0.16);
		pawShape.bezierCurveTo(-0.015, -0.18, 0.005, -0.12, 0.005, -0.075);

		// Jari ketiga.
		pawShape.bezierCurveTo(0.015, -0.025, 0.025, -0.15, 0.065, -0.15);
		pawShape.bezierCurveTo(0.1, -0.14, 0.1, -0.08, 0.085, -0.025);

		// Jari paling kanan.
		pawShape.bezierCurveTo(0.1, 0.005, 0.12, -0.08, 0.15, -0.055);
		pawShape.bezierCurveTo(0.19, -0.02, 0.17, 0.045, 0.13, 0.08);

		// Bantalan utama.
		pawShape.bezierCurveTo(0.12, 0.15, 0.07, 0.19, 0, 0.19);
		pawShape.bezierCurveTo(-0.07, 0.19, -0.12, 0.15, -0.13, 0.08);

		const pawGeometry = new THREE.ShapeGeometry(pawShape, 8);
		pawGeometry.scale(0.6, 0.6, 1);
		pawGeometry.rotateX(Math.PI / 2);

		const pawMaterial = new THREE.MeshBasicMaterial({
			color: '#35445b',
			transparent: true,
			opacity: 0.95,
			depthWrite: false,
			side: THREE.DoubleSide,
			toneMapped: false
		});

		// =========================================
		// KONFIGURASI KAWANAN
		// =========================================

		const WOLF_COUNT = 5;

		// Tinggi target setiap serigala dalam unit Three.js.
		const TARGET_HEIGHT = 1.15;

		const TERRAIN = {
			centerX: 8,
			centerY: -0.5,
			centerZ: -18,
			width: 70,
			depth: 38
		} as const;

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

		function createPawPrint(x: number, z: number) {
			const paw = new THREE.Mesh(pawGeometry, pawMaterial);

			const localX = x - TERRAIN.centerX;
			const localZ = z - TERRAIN.centerZ;

			const groundY = TERRAIN.centerY + terrainHeight(localX, localZ);

			paw.position.set(x, groundY + 0.025, z);

			scene.add(paw);
			pawPrintGroups.push(paw);
		}

		// Arah hadap model.
		// Jika model tidak menghadap kanan, sesuaikan nilai ini.
		const MODEL_ROTATION_Y = -Math.PI / 2.3;

		// Posisi kawanan di sisi kiri.
		// Tidak berubah selama animasi berjalan.
		const WOLF_POSITIONS = [
			{ x: 2.8, z: -22.6 },
			{ x: 4.2, z: -20.6 },
			{ x: 5.7, z: -21.4 },
			{ x: 7.2, z: -20.2 },
			{ x: 9.0, z: -20.8 }
		];

		// Sedikit perbedaan fase agar tidak serempak.
		const ANIMATION_OFFSETS = [0, 0.16, 0.31, 0.47, 0.62];

		function removeGroundMesh(root: THREE.Object3D) {
			const ground = root.getObjectByName('Plane_unnamed_0');

			if (!ground) {
				console.warn('[Wolf] Mesh alas tidak ditemukan.');
				return;
			}

			if (!(ground instanceof THREE.Mesh)) {
				console.warn('[Wolf] Object alas bukan mesh.');
				return;
			}

			ground.parent?.remove(ground);
			ground.geometry.dispose();

			const materials = Array.isArray(ground.material) ? ground.material : [ground.material];

			for (const material of materials) {
				material.dispose();
			}

			console.info('[Wolf] Mesh alas berhasil dihapus:', ground.name);
		}

		// =========================================
		// LOAD MODEL
		// =========================================

		loader.load(
			'/models/winter-night/wolf.glb',

			(gltf) => {
				if (disposed) {
					disposeSource(gltf.scene);
					return;
				}

				const sourceWolf = gltf.scene;
				removeGroundMesh(sourceWolf);

				if (gltf.animations.length === 0) {
					console.error('[Wolf] Model tidak memiliki animasi.');
					return;
				}

				// Animasi pertama adalah animasi berlari.
				const runAnimation = gltf.animations[1];

				console.info(
					'[Wolf] Animasi tersedia:',
					gltf.animations.map((clip, index) => ({
						index,
						name: clip.name,
						duration: clip.duration
					}))
				);

				// Kumpulkan resource model asli untuk cleanup.
				sourceWolf.traverse((object) => {
					if (!(object instanceof THREE.Mesh)) return;

					if (!sourceGeometries.includes(object.geometry)) {
						sourceGeometries.push(object.geometry);
					}

					const meshMaterials = Array.isArray(object.material)
						? object.material
						: [object.material];

					meshMaterials.forEach((material) => {
						if (!sourceMaterials.includes(material)) {
							sourceMaterials.push(material);
						}
					});

					object.castShadow = true;
					object.receiveShadow = true;
				});

				// Hitung bounding box model untuk normalisasi skala.
				sourceWolf.updateMatrixWorld(true);

				const sourceBounds = new THREE.Box3().setFromObject(sourceWolf);

				const sourceSize = new THREE.Vector3();
				sourceBounds.getSize(sourceSize);

				if (sourceSize.y <= 0) {
					console.error('[Wolf] Tinggi model tidak valid.');
					return;
				}

				const modelScale = TARGET_HEIGHT / sourceSize.y;

				// Posisi lokal agar kaki model berada di dasar root.
				const modelGroundOffset = -sourceBounds.min.y * modelScale;

				for (const material of sourceMaterials) {
					if (
						material instanceof THREE.MeshStandardMaterial ||
						material instanceof THREE.MeshPhysicalMaterial
					) {
						// material.color.multiplyScalar(0.55);
						material.color.set('#454545');
						material.roughness = 1;
						material.metalness = 0;
					}
				}

				// =========================================
				// JEJAK KAKI SERIGALA
				// =========================================

				const STEP_SPACING = 0.65;
				const PAW_OFFSET_X = 0.12;
				const TRACK_OFFSET_X = 0.12;

				for (let i = 0; i < WOLF_COUNT; i++) {
					const { x, z } = WOLF_POSITIONS[i];

					const startZ = 0;
					const endZ = z + 0.5;
					const distance = Math.abs(startZ - endZ);

					// Jumlah langkah disesuaikan dengan panjang jalur.
					const steps = Math.max(2, Math.floor(distance / STEP_SPACING) + 1);

					const trackOffset = (i - 2) * TRACK_OFFSET_X;

					for (let step = 0; step < steps; step++) {
						const printZ = startZ - step * STEP_SPACING;

						// Jangan melewati titik akhir jalur.
						if (printZ < endZ) break;

						const t = step / Math.max(steps - 1, 1);

						const curve = Math.sin(t * Math.PI) * Math.sin(t * Math.PI * 2) * 0.08;

						const printX = x + trackOffset + curve;

						// Posisi kiri dan kanan bergantian.
						const side = step % 2 === 0 ? -1 : 1;

						createPawPrint(printX + side * PAW_OFFSET_X, printZ);
					}
				}

				// =========================================
				// BUAT LIMA SERIGALA
				// =========================================

				for (let i = 0; i < WOLF_COUNT; i++) {
					const group = new THREE.Group();
					group.name = `Wolf-${i + 1}`;

					// Clone skeleton agar setiap serigala
					// mempunyai tulang dan animasi independen.
					const wolfModel = cloneSkeleton(sourceWolf);

					wolfModel.scale.setScalar(modelScale);
					wolfModel.position.y = modelGroundOffset;

					group.add(wolfModel);

					const { x, z } = WOLF_POSITIONS[i];

					// Konversi koordinat dunia ke koordinat lokal terrain.
					const localX = x - TERRAIN.centerX;
					const localZ = z - TERRAIN.centerZ;

					// Ketinggian permukaan salju dalam koordinat dunia.
					const groundY = TERRAIN.centerY + terrainHeight(localX, localZ);

					// Sedikit offset untuk mencegah kaki tenggelam akibat
					// perbedaan kecil antara model dan permukaan terrain.
					group.position.set(x, groundY + 0.015, z);

					group.rotation.y = MODEL_ROTATION_Y;

					// Material gelap untuk tampilan siluet.
					// Warna asli model tetap menjadi dasar.
					wolfModel.traverse((object) => {
						if (!(object instanceof THREE.Mesh)) return;

						object.castShadow = true;
						object.receiveShadow = true;
					});

					scene.add(group);
					wolfGroups.push(group);

					// Mixer independen untuk setiap serigala.
					const mixer = new THREE.AnimationMixer(wolfModel);

					const action = mixer.clipAction(runAnimation);

					action.reset();
					action.setLoop(THREE.LoopRepeat, Infinity);
					action.clampWhenFinished = false;
					action.play();

					// Offset fase agar animasi tidak identik.
					mixer.setTime(ANIMATION_OFFSETS[i]);

					mixers.push(mixer);
				}

				console.info(`[Wolf] ${wolfGroups.length} serigala berhasil dimuat.`);
			},

			undefined,

			(error) => {
				console.error('[Wolf] Gagal memuat /models/wolf.glb:', error);
			}
		);

		// =========================================
		// UPDATE ANIMASI
		// =========================================

		const updateWolves = (time: number) => {
			const delta = Math.max(0, Math.min(time - previousTime, 0.1));

			previousTime = time;

			// Hanya animasi model yang bergerak.
			// Posisi kelima serigala tidak berubah.
			for (const mixer of mixers) {
				mixer.update(delta);
			}
		};

		registerUpdate?.(updateWolves);

		// =========================================
		// CLEANUP
		// =========================================

		function disposeSource(root: THREE.Object3D) {
			const geometries: THREE.BufferGeometry[] = [];
			const materials: THREE.Material[] = [];

			root.traverse((object) => {
				if (!(object instanceof THREE.Mesh)) return;

				if (!geometries.includes(object.geometry)) {
					geometries.push(object.geometry);
				}

				const meshMaterials = Array.isArray(object.material) ? object.material : [object.material];

				meshMaterials.forEach((material) => {
					if (!materials.includes(material)) {
						materials.push(material);
					}
				});
			});

			geometries.forEach((geometry) => geometry.dispose());
			materials.forEach((material) => material.dispose());
		}

		return () => {
			disposed = true;

			registerUpdate?.(undefined);

			for (const mixer of mixers) {
				mixer.stopAllAction();
				mixer.uncacheRoot(mixer.getRoot());
			}

			for (const group of wolfGroups) {
				scene.remove(group);
			}

			for (const paw of pawPrintGroups) {
				scene.remove(paw);
			}

			pawGeometry.dispose();
			pawMaterial.dispose();

			pawPrintGroups.length = 0;

			sourceGeometries.forEach((geometry) => geometry.dispose());
			sourceMaterials.forEach((material) => material.dispose());

			wolfGroups.length = 0;
			mixers.length = 0;
		};
	});
</script>
