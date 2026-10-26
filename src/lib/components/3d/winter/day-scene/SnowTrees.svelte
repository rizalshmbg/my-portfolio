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

		const loader = new GLTFLoader();

		const treeGroups: THREE.Group[] = [];
		const sourceGeometries: THREE.BufferGeometry[] = [];
		const sourceMaterials: THREE.Material[] = [];

		// =========================================
		// CONFIGURATION TREE
		// =========================================
		function darkenTreeTrunk(root: THREE.Object3D) {
			const trunkMaterials = new Set(['Material.002', 'mat20']);

			root.traverse((object) => {
				if (!(object instanceof THREE.Mesh)) return;

				const materials = Array.isArray(object.material) ? object.material : [object.material];

				for (const material of materials) {
					if (!trunkMaterials.has(material.name)) continue;

					if (
						material instanceof THREE.MeshStandardMaterial ||
						material instanceof THREE.MeshPhongMaterial
					) {
						material.color.set('#827568');
						material.needsUpdate = true;
					}
				}
			});
		}

		// =========================================
		// snow-pine-tree.glb
		// =========================================
		const SNOW_PINE_TREE_POSITIONS = [
			// Kelompok belakang kiri
			{ x: 5, z: -38, height: 2.2, rotation: 0.8 },
			{ x: 11, z: -41, height: 3.1, rotation: 2.4 },
			{ x: 15, z: -35, height: 3.5, rotation: 1.2 },
			{ x: 9, z: -31, height: 2.8, rotation: 2.8 },

			// Kelompok belakang tengah
			{ x: 20, z: -40, height: 3.5, rotation: 2.1 },
			{ x: 24, z: -35, height: 4.8, rotation: 0.5 },
			{ x: 29, z: -39, height: 4.2, rotation: 2.5 },
			{ x: 34, z: -36, height: 4.5, rotation: 1.4 },
			{ x: 27, z: -33, height: 3.6, rotation: 3.1 },

			// Kelompok belakang kanan
			{ x: 39, z: -34, height: 4.8, rotation: 3.1 },
			{ x: 43, z: -29, height: 3.7, rotation: 1.7 },
			{ x: 36, z: -30, height: 4.1, rotation: 0.9 },

			// Kelompok tengah kiri
			{ x: 19, z: -30, height: 3.8, rotation: 2.0 },
			{ x: 24, z: -28, height: 4.5, rotation: 0.7 },
			{ x: 16, z: -27, height: 3.2, rotation: 1.3 },

			// Kelompok tengah kanan
			{ x: 33, z: -29, height: 4.2, rotation: 2.8 },
			{ x: 40, z: -25, height: 3.8, rotation: 1.1 },
			{ x: 36, z: -24, height: 3.3, rotation: 0.4 },

			// Beberapa pohon kecil sebagai transisi
			{ x: 21, z: -24, height: 2.8, rotation: 1.3 },
			{ x: 30, z: -25, height: 3.1, rotation: 3.0 }
		] as const;

		// =========================================
		// tree-with-snow.glb
		// =========================================
		const TREE_WITH_SNOW_POSITIONS = [
			// Kelompok belakang kiri
			{ x: 8, z: -35, height: 3.5, rotation: 3.2 },
			{ x: 13, z: -39, height: 3.2, rotation: 1.7 },
			{ x: 18, z: -37, height: 3.8, rotation: 1.1 },

			// Kelompok belakang tengah
			{ x: 22, z: -37, height: 3.5, rotation: 2.4 },
			{ x: 26, z: -41, height: 4.0, rotation: 0.4 },
			{ x: 32, z: -38, height: 3.6, rotation: 0.8 },
			{ x: 35, z: -34, height: 3.3, rotation: 2.6 },

			// Kelompok belakang kanan
			{ x: 41, z: -32, height: 4.0, rotation: 3.5 },
			{ x: 44, z: -25, height: 3.5, rotation: 1.6 },

			// Kelompok tengah
			{ x: 21, z: -32, height: 3.4, rotation: 0.3 },
			{ x: 28, z: -31, height: 3.8, rotation: 1.9 },
			{ x: 33, z: -32, height: 3.3, rotation: 2.7 },
			{ x: 38, z: -28, height: 3.7, rotation: 1.5 },

			// Transisi menuju area terbuka
			{ x: 25, z: -25, height: 2.9, rotation: 3.2 },
			{ x: 34, z: -23, height: 3.1, rotation: 2.1 }
		] as const;

		// =========================================
		// RESOURCE MANAGEMENT
		// =========================================

		function trackSourceResources(root: THREE.Object3D) {
			root.traverse((object) => {
				if (!(object instanceof THREE.Mesh)) return;

				if (!sourceGeometries.includes(object.geometry)) {
					sourceGeometries.push(object.geometry);
				}

				const materials = Array.isArray(object.material) ? object.material : [object.material];

				for (const material of materials) {
					if (!sourceMaterials.includes(material)) {
						sourceMaterials.push(material);
					}
				}
			});
		}

		// =========================================
		// PERSIAPAN MODEL POHON
		// =========================================

		function prepareTree(source: THREE.Object3D, targetHeight: number): THREE.Group {
			const tree = new THREE.Group();
			const model = cloneSkeleton(source);

			model.traverse((object) => {
				if (!(object instanceof THREE.Mesh)) return;

				object.castShadow = true;
				object.receiveShadow = true;
			});

			model.updateMatrixWorld(true);

			const originalBounds = new THREE.Box3().setFromObject(model);
			const originalHeight = originalBounds.max.y - originalBounds.min.y;

			if (originalHeight <= 0) {
				console.warn('[SnowTrees] Tinggi model tidak valid.');
				return tree;
			}

			// Sesuaikan tinggi model.
			const scale = targetHeight / originalHeight;
			model.scale.multiplyScalar(scale);

			model.updateMatrixWorld(true);

			// Posisikan bagian bawah model pada y = 0.
			const scaledBounds = new THREE.Box3().setFromObject(model);

			model.position.y -= scaledBounds.min.y;

			tree.add(model);

			return tree;
		}

		// =========================================
		// MEMUAT MODEL
		// =========================================

		async function loadTrees() {
			try {
				const [snowPineGLTF, treeWithSnowGLTF] = await Promise.all([
					loader.loadAsync('/models/winter-night/snow-pine-tree.glb'),
					loader.loadAsync('/models/winter-night/tree-with-snow.glb')
				]);

				if (disposed) return;

				const models = [
					{
						name: 'SnowPineTree',
						source: snowPineGLTF.scene,
						positions: SNOW_PINE_TREE_POSITIONS
					},
					{
						name: 'TreeWithSnow',
						source: treeWithSnowGLTF.scene,
						positions: TREE_WITH_SNOW_POSITIONS
					}
				];

				for (const modelConfig of models) {
					trackSourceResources(modelConfig.source);

					for (const config of modelConfig.positions) {
						if (disposed) return;

						const tree = prepareTree(modelConfig.source, config.height);

						darkenTreeTrunk(tree);

						tree.name = `${modelConfig.name}-${treeGroups.length + 1}`;

						tree.position.set(config.x, 0, config.z);
						tree.rotation.y = config.rotation;

						scene.add(tree);
						treeGroups.push(tree);
					}
				}

				console.info(`[SnowTrees] ${treeGroups.length} pohon berhasil ditambahkan.`);
			} catch (error) {
				if (!disposed) {
					console.error('[SnowTrees] Gagal memuat model pohon:', error);
				}
			}
		}

		void loadTrees();

		// =========================================
		// CLEANUP
		// =========================================

		return () => {
			disposed = true;

			registerUpdate?.(undefined);

			for (const tree of treeGroups) {
				scene.remove(tree);
			}

			sourceGeometries.forEach((geometry) => geometry.dispose());
			sourceMaterials.forEach((material) => material.dispose());

			treeGroups.length = 0;
			sourceGeometries.length = 0;
			sourceMaterials.length = 0;
		};
	});
</script>
