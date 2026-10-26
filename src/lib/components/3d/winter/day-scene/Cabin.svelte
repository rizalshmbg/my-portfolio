<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

	let { scene }: { scene: THREE.Scene } = $props();

	onMount(() => {
		let disposed = false;
		let cabin: THREE.Group | null = null;

		const loader = new GLTFLoader();

		function disposeModel(root: THREE.Object3D) {
			const geometries: THREE.BufferGeometry[] = [];
			const materials: THREE.Material[] = [];
			const textures: THREE.Texture[] = [];

			root.traverse((object) => {
				if (!(object instanceof THREE.Mesh)) return;

				geometries.push(object.geometry);

				const meshMaterials = Array.isArray(object.material) ? object.material : [object.material];

				for (const material of meshMaterials) {
					materials.push(material);

					for (const value of Object.values(material)) {
						if (value instanceof THREE.Texture) {
							textures.push(value);
						}
					}
				}
			});

			for (const texture of textures) texture.dispose();
			for (const material of materials) material.dispose();
			for (const geometry of geometries) geometry.dispose();
		}

		loader.load(
			'/models/winter-day/cabin-winter.glb',
			(gltf) => {
				if (disposed) {
					disposeModel(gltf.scene);
					return;
				}

				cabin = gltf.scene;
				cabin.name = 'WinterDayCabin';

				// Periksa ukuran dan pusat model.
				const box = new THREE.Box3().setFromObject(cabin);
				const size = box.getSize(new THREE.Vector3());
				const center = box.getCenter(new THREE.Vector3());

				console.log('[WinterDay Cabin]', {
					size: size.toArray(),
					center: center.toArray(),
					min: box.min.toArray(),
					max: box.max.toArray()
				});

				// Aktifkan bayangan.
				cabin.traverse((object) => {
					if (object instanceof THREE.Mesh) {
						object.castShadow = true;
						object.receiveShadow = true;
					}
				});

				// Pusatkan model berdasarkan bounding box lokal.
				const localBox = new THREE.Box3().setFromObject(cabin);
				const localCenter = localBox.getCenter(new THREE.Vector3());

				// Pertahankan transformasi cabin saat ini.
				const scale = 0.4;
				cabin.scale.setScalar(scale);
				cabin.rotation.y = 0.3;

				const target = new THREE.Vector3(-19, 2, -25);

				cabin.position.set(
					target.x - localCenter.x * scale,
					target.y - localCenter.y * scale,
					target.z - localCenter.z * scale
				);

				scene.add(cabin);

				console.info('[WinterDay Cabin] Model berhasil dimuat.');
			},
			undefined,
			(error) => {
				if (!disposed) {
					console.error('[WinterDay Cabin] Gagal memuat model:', error);
				}
			}
		);

		return () => {
			disposed = true;

			if (cabin) {
				scene.remove(cabin);
				disposeModel(cabin);
				cabin = null;
			}
		};
	});
</script>
