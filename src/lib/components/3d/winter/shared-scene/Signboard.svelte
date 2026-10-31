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
		let lanternLight: THREE.PointLight | null = null;
		let lanternGroundLight: THREE.SpotLight | null = null;
		let lanternGlass: THREE.Mesh | null = null;
		let lanternGlassMaterials: THREE.MeshStandardMaterial[] = [];

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
					if (!(object instanceof THREE.Mesh)) return;

					object.castShadow = true;
					object.receiveShadow = true;

					const materials = Array.isArray(object.material) ? object.material : [object.material];

					if (materials.some((material) => material.name === 'LanternGlass')) {
						lanternGlass = object;

						lanternGlassMaterials = materials.filter(
							(material): material is THREE.MeshStandardMaterial =>
								material instanceof THREE.MeshStandardMaterial && material.name === 'LanternGlass'
						);

						for (const material of lanternGlassMaterials) {
							material.transparent = true;
							material.opacity = 0.4;
							material.roughness = 0.15;
							material.depthWrite = false;

							// Warna kaca ketika lampu mati.
							material.color.set('#ffffff');
							material.emissive.set('#000000');
							material.emissiveIntensity = 0;
						}
					}
				});

				scene.add(signboard);

				if (lanternGlass && candleEnabled) {
					signboard.updateMatrixWorld(true);

					const glassBox = new THREE.Box3().setFromObject(lanternGlass);
					const glassCenter = glassBox.getCenter(new THREE.Vector3());

					lanternLight = new THREE.PointLight(0xffb52e, 8, 4, 2);
					lanternLight.position.copy(glassCenter);
					lanternLight.castShadow = false;
					scene.add(lanternLight);

					lanternGroundLight = new THREE.SpotLight(0xffc45c, 15, 5, Math.PI / 7, 0.8, 2);
					lanternGroundLight.position.set(glassCenter.x, glassCenter.y - 0.3, glassCenter.z);
					lanternGroundLight.target.position.set(glassCenter.x, glassCenter.y - 3, glassCenter.z);
					lanternGroundLight.castShadow = false;
					scene.add(lanternGroundLight);
					scene.add(lanternGroundLight.target);

					// Kaca berubah warna ketika lampu menyala.
					for (const material of lanternGlassMaterials) {
						material.color.set('#ffdf57');
						material.emissive.set('#ffb300');
						material.emissiveIntensity = 4;

						material.transparent = true;
						material.opacity = 0.9;
						material.roughness = 0.2;
						material.depthWrite = false;
					}

					console.log('[Signboard] Lantern menyala:', {
						glassName: lanternGlass.name,
						position: lanternLight.position.toArray()
					});
				} else if (!lanternGlass) {
					console.warn('[Signboard] Mesh dengan material LanternGlass tidak ditemukan.');
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

			if (lanternLight) {
				scene.remove(lanternLight);
				lanternLight.dispose();
				lanternLight = null;
			}

			if (lanternGroundLight) {
				scene.remove(lanternGroundLight);
				scene.remove(lanternGroundLight.target);
				lanternGroundLight.dispose();
				lanternGroundLight = null;
			}

			if (signboard) {
				scene.remove(signboard);
				signboard = null;
			}
		};
	});
</script>
