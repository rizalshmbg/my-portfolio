<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
	// import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';

	let { scene }: { scene: THREE.Scene } = $props();

	onMount(() => {
		let disposed = false;
		let cabin: THREE.Group | null = null;
		let cabinLight: THREE.PointLight | null = null;

		// const dracoLoader = new DRACOLoader();
		// dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.7/');

		const loader = new GLTFLoader();
		// loader.setDRACOLoader(dracoLoader);

		function disposeModel(root: THREE.Object3D) {
			const geometries: THREE.BufferGeometry[] = [];
			const materials: THREE.Material[] = [];
			const textures: THREE.Texture[] = [];

			root.traverse((object) => {
				if (!(object instanceof THREE.Mesh)) return;

				if (!geometries.includes(object.geometry)) {
					geometries.push(object.geometry);
				}

				const meshMaterials = Array.isArray(object.material) ? object.material : [object.material];

				for (const material of meshMaterials) {
					if (!materials.includes(material)) {
						materials.push(material);
					}

					for (const value of Object.values(material)) {
						if (value instanceof THREE.Texture && !textures.includes(value)) {
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
			'/models/cabin-winter.glb',
			(gltf) => {
				if (disposed) {
					disposeModel(gltf.scene);
					return;
				}

				cabin = gltf.scene;
				cabin.name = 'WinterCabin';

				const box = new THREE.Box3().setFromObject(cabin);
				const size = box.getSize(new THREE.Vector3());
				const center = box.getCenter(new THREE.Vector3());

				console.log('[Cabin New Model]', {
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

				// Atur posisi dan skala kabin.
				const scale = 0.4;
				cabin.scale.setScalar(scale);

				const target = new THREE.Vector3(-17, 2, -25);

				// Tempatkan pusat model di lokasi target.
				cabin.position.set(
					target.x - localCenter.x * scale,
					target.y - localCenter.y * scale,
					target.z - localCenter.z * scale
				);

				scene.add(cabin);

				// Cahaya hangat di sekitar kabin.
				cabinLight = new THREE.PointLight('#ffb45c', 20, 10, 1);
				cabinLight.position.set(-16, 2.8, -24);
				cabinLight.castShadow = false;

				scene.add(cabinLight);

				console.info('[Cabin] Model berhasil dimuat.');
			},
			undefined,
			(error) => {
				if (!disposed) {
					console.error('[Cabin] Gagal memuat model:', error);
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

			if (cabinLight) {
				scene.remove(cabinLight);
				cabinLight.dispose();
				cabinLight = null;
			}
		};
	});
</script>
