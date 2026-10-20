<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
	import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
	import { SvelteSet } from 'svelte/reactivity';

	interface Props {
		scene: THREE.Scene;
	}

	interface TreeConfig {
		model: string;
		position: THREE.Vector3;
		height: number;
		rotationY?: number;
	}

	let { scene }: Props = $props();

	onMount(() => {
		const dracoLoader = new DRACOLoader();
		dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.7/');

		const loader = new GLTFLoader();
		loader.setDRACOLoader(dracoLoader);
		const treeGroups: THREE.Group[] = [];
		const geometries = new SvelteSet<THREE.BufferGeometry>();
		const materials = new SvelteSet<THREE.Material>();

		let disposed = false;

		// Pohon ditempatkan di tepi lanskap agar area tengah
		// tetap tersedia untuk serigala dan konten Hero.
		const treeConfigs: TreeConfig[] = [
			{
				model: '/models/dead-tree.glb',
				position: new THREE.Vector3(-13, -1.5, -17),
				height: 7,
				rotationY: 0.2
			},
			{
				model: '/models/bare-tree.glb',
				position: new THREE.Vector3(13, -1.5, -18),
				height: 6,
				rotationY: -0.25
			},
			{
				model: '/models/bare-tree.glb',
				position: new THREE.Vector3(-19, -1.5, -24),
				height: 8,
				rotationY: 0.35
			},
			{
				model: '/models/dead-tree.glb',
				position: new THREE.Vector3(20, -1.5, -25),
				height: 7,
				rotationY: -0.3
			}
		];

		function trackResources(root: THREE.Object3D) {
			root.traverse((object) => {
				if (!(object instanceof THREE.Mesh)) return;

				geometries.add(object.geometry);

				const meshMaterials = Array.isArray(object.material) ? object.material : [object.material];

				for (const material of meshMaterials) {
					materials.add(material);
				}
			});
		}

		async function loadTree(config: TreeConfig) {
			try {
				const gltf = await loader.loadAsync(config.model);

				// Komponen mungkin sudah dilepas ketika model selesai dimuat.
				if (disposed) {
					gltf.scene.traverse((object) => {
						if (!(object instanceof THREE.Mesh)) return;

						object.geometry.dispose();

						const meshMaterials = Array.isArray(object.material)
							? object.material
							: [object.material];

						for (const material of meshMaterials) {
							material.dispose();
						}
					});

					return;
				}

				const tree = gltf.scene;

				// Hitung ukuran asli model.
				tree.updateMatrixWorld(true);

				const originalBounds = new THREE.Box3().setFromObject(tree);
				const originalSize = new THREE.Vector3();

				originalBounds.getSize(originalSize);

				if (originalSize.y <= 0) {
					console.warn(`Model pohon tidak memiliki tinggi valid: ${config.model}`);
					return;
				}

				// Samakan tinggi pohon tanpa mengubah proporsi model.
				const scale = config.height / originalSize.y;
				tree.scale.multiplyScalar(scale);

				// Hitung ulang bounds setelah perubahan skala.
				tree.updateMatrixWorld(true);

				const scaledBounds = new THREE.Box3().setFromObject(tree);

				// Tempatkan pangkal pohon tepat di permukaan Y konfigurasi.
				tree.position.set(
					config.position.x,
					config.position.y - scaledBounds.min.y,
					config.position.z
				);

				tree.rotation.y = config.rotationY ?? 0;

				// Wrapper memudahkan pengaturan posisi dan animasi nantinya.
				const group = new THREE.Group();
				group.add(tree);
				group.position.set(config.position.x, config.position.y, config.position.z);

				// Posisi pohon pada koordinat lokal group.
				tree.position.set(0, -scaledBounds.min.y, 0);

				trackResources(tree);

				scene.add(group);
				treeGroups.push(group);
			} catch (error) {
				if (!disposed) {
					console.error(`Gagal memuat model pohon: ${config.model}`, error);
				}
			}
		}

		void Promise.all(treeConfigs.map(loadTree));

		return () => {
			disposed = true;

			for (const group of treeGroups) {
				scene.remove(group);
			}

			for (const geometry of geometries) {
				geometry.dispose();
			}

			for (const material of materials) {
				material.dispose();
			}

			dracoLoader.dispose();
		};
	});
</script>
