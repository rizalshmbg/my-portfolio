<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

	interface Props {
		scene: THREE.Scene;
		candleEnabled?: boolean;
	}

	let { scene, candleEnabled = true }: Props = $props();

	onMount(() => {
		let disposed = false;
		let signboard: THREE.Group | null = null;
		let candleLight: THREE.PointLight | null = null;
		let lowerLight: THREE.PointLight | null = null;

		const loader = new GLTFLoader();

		const targetPosition = new THREE.Vector3(3.5, -1.2, -3.2);
		const targetHeight = 5.5;

		loader.load(
			'/models/shared-scene/signpost.glb',
			(gltf) => {
				if (disposed) {
					return;
				}

				signboard = gltf.scene;
				signboard.name = 'InteractiveSignboard';

				const box = new THREE.Box3().setFromObject(signboard);
				const size = box.getSize(new THREE.Vector3());
				const center = box.getCenter(new THREE.Vector3());

				if (size.y > 0) {
					const scale = targetHeight / size.y;

					signboard.scale.setScalar(scale);

					signboard.position.set(
						targetPosition.x - center.x * scale,
						targetPosition.y - box.min.y * scale,
						targetPosition.z - center.z * scale
					);
					signboard.rotation.y = THREE.MathUtils.degToRad(-15);
				} else {
					signboard.position.copy(targetPosition);
				}

				signboard.traverse((object) => {
					if (object instanceof THREE.Mesh) {
						object.castShadow = true;
						object.receiveShadow = true;
					}
				});

				scene.add(signboard);

				let lanternObject: THREE.Object3D | null = null;

				signboard.traverse((object) => {
					if (object.name === 'harvest_lantern004') {
						lanternObject = object;
					}
				});

				if (lanternObject  && candleEnabled) {
					signboard.updateMatrixWorld(true);

					const lanternBox = new THREE.Box3().setFromObject(lanternObject);
					const lanternCenter = lanternBox.getCenter(new THREE.Vector3());

					candleLight = new THREE.PointLight(0xffb45c, 6, 7, 1.5);
					candleLight.position.set(lanternCenter.x, lanternCenter.y, lanternCenter.z);
					candleLight.castShadow = false;
					scene.add(candleLight);

					lowerLight = new THREE.PointLight(0xffa64d, 4, 7, 1.5);
					lowerLight.position.set(lanternCenter.x, lanternCenter.y - 0.8, lanternCenter.z);
					lowerLight.castShadow = false;
					scene.add(lowerLight);
				}

				console.info('[Signboard] Model berhasil dimuat.');
			},
			undefined,
			(error) => {
				if (!disposed) {
					console.error('[Signboard] Gagal memuat model:', error);
				}
			}
		);

		return () => {
			disposed = true;

			if (!signboard) {
				return;
			}

			if (candleLight) {
				scene.remove(candleLight);
				candleLight.dispose();
				candleLight = null;
			}

			if (lowerLight) {
				scene.remove(lowerLight);
				lowerLight.dispose();
				lowerLight = null;
			}

			// Lepaskan model dari scene.
			scene.remove(signboard);

			// Jangan dispose resource GLB di sini.
			// Geometri, material, dan tekstur mungkin digunakan bersama
			// oleh komponen atau objek lain.
			signboard = null;
		};
	});
</script>
