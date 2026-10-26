<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

	interface Props {
		scene: THREE.Scene;
	}

	let { scene }: Props = $props();

	onMount(() => {
		const loader = new GLTFLoader();

		let sun: THREE.Group | null = null;
		let isDisposed = false;

		// Menyimpan seluruh resource glow agar mudah dibersihkan.
		const glowSprites: THREE.Sprite[] = [];
		const glowMaterials: THREE.SpriteMaterial[] = [];
		const glowTextures: THREE.CanvasTexture[] = [];

		// Posisi matahari mengikuti Moon.svelte.
		const sunPosition = new THREE.Vector3(-7, 20, -50);

		/**
		 * Membuat tekstur radial glow.
		 */
		function createGlowTexture(
			stops: Array<{ offset: number; color: string }>
		): THREE.CanvasTexture {
			const canvas = document.createElement('canvas');
			canvas.width = 512;
			canvas.height = 512;

			const context = canvas.getContext('2d');

			if (!context) {
				throw new Error('Gagal membuat canvas untuk glow matahari.');
			}

			const gradient = context.createRadialGradient(256, 256, 35, 256, 256, 256);

			for (const stop of stops) {
				gradient.addColorStop(stop.offset, stop.color);
			}

			context.fillStyle = gradient;
			context.fillRect(0, 0, 512, 512);

			const texture = new THREE.CanvasTexture(canvas);
			texture.colorSpace = THREE.SRGBColorSpace;

			glowTextures.push(texture);

			return texture;
		}

		/**
		 * Membuat sprite glow dan menambahkannya ke scene.
		 */
		function createGlow(
			texture: THREE.CanvasTexture,
			size: number,
			opacity: number,
			positionZ: number,
			renderOrder: number
		): THREE.Sprite {
			const material = new THREE.SpriteMaterial({
				map: texture,
				color: '#ffffff',
				transparent: true,
				opacity,
				blending: THREE.AdditiveBlending,
				depthWrite: false,
				depthTest: false,
				toneMapped: false
			});

			const sprite = new THREE.Sprite(material);

			sprite.position.set(sunPosition.x, sunPosition.y, positionZ);

			sprite.scale.set(size, size, 1);
			sprite.renderOrder = renderOrder;

			scene.add(sprite);

			glowSprites.push(sprite);
			glowMaterials.push(material);

			return sprite;
		}

		/**
		 * Memuat model matahari.
		 */
		loader.load(
			'/models/winter-day/sun.glb',
			(gltf) => {
				const loadedSun = gltf.scene;

				// Jika komponen sudah dihancurkan sebelum model selesai dimuat.
				if (isDisposed) {
					disposeSun(loadedSun);
					return;
				}

				sun = loadedSun;

				// Mempertahankan model, tetapi membuat warna permukaannya
				// lebih merata dan hangat.
				sun.traverse((child) => {
					if (!(child instanceof THREE.Mesh)) return;

					const originalMaterials = Array.isArray(child.material)
						? child.material
						: [child.material];

					const updatedMaterials = originalMaterials.map((originalMaterial) => {
						const material = new THREE.MeshBasicMaterial({
							color: '#FFD36A',
							toneMapped: false
						});

						// Mempertahankan transparansi jika material asal menggunakannya.
						material.transparent = originalMaterial.transparent;
						material.opacity = originalMaterial.opacity;
						material.side = originalMaterial.side;

						return material;
					});

					child.material = Array.isArray(child.material) ? updatedMaterials : updatedMaterials[0];
				});

				sun.position.copy(sunPosition);
				sun.scale.setScalar(3);

				scene.add(sun);

				// --------------------------------------------------
				// 1. INNER GLOW
				// --------------------------------------------------

				const innerGlowTexture = createGlowTexture([
					{ offset: 0, color: 'rgba(255, 220, 120, 0.25)' },
					{ offset: 0.2, color: 'rgba(255, 205, 90, 0.28)' },
					{ offset: 0.4, color: 'rgba(255, 185, 65, 0.22)' },
					{ offset: 0.65, color: 'rgba(255, 165, 50, 0.10)' },
					{ offset: 1, color: 'rgba(255, 165, 50, 0)' }
				]);

				createGlow(innerGlowTexture, 7, 0.8, -49.7, 2);

				// --------------------------------------------------
				// 2. MID GLOW
				// --------------------------------------------------

				const midGlowTexture = createGlowTexture([
					{ offset: 0, color: 'rgba(255, 210, 110, 0.15)' },
					{ offset: 0.15, color: 'rgba(255, 190, 75, 0.25)' },
					{ offset: 0.3, color: 'rgba(255, 165, 55, 0.25)' },
					{ offset: 0.45, color: 'rgba(255, 145, 45, 0.18)' },
					{ offset: 0.6, color: 'rgba(255, 155, 65, 0.10)' },
					{ offset: 0.8, color: 'rgba(255, 175, 100, 0.04)' },
					{ offset: 1, color: 'rgba(255, 175, 100, 0)' }
				]);

				createGlow(midGlowTexture, 20, 0.2, -49.5, 3);

				// --------------------------------------------------
				// 3. OUTER GLOW
				// --------------------------------------------------

				const outerGlowTexture = createGlowTexture([
					{ offset: 0, color: 'rgba(255, 250, 230, 0.3)' },
					{ offset: 0.15, color: 'rgba(255, 235, 190, 0.32)' },
					{ offset: 0.3, color: 'rgba(255, 211, 155, 0.3)' },
					{ offset: 0.45, color: 'rgba(255, 190, 130, 0.25)' },
					{ offset: 0.6, color: 'rgba(255, 173, 120, 0.18)' },
					{ offset: 0.75, color: 'rgba(255, 188, 160, 0.1)' },
					{ offset: 0.9, color: 'rgba(245, 220, 220, 0.035)' },
					{ offset: 1, color: 'rgba(235, 225, 240, 0)' }
				]);

				createGlow(outerGlowTexture, 10, 0.5, -49.2, 4);
			},
			undefined,
			(error) => {
				if (!isDisposed) {
					console.error('Gagal memuat model sun.glb:', error);
				}
			}
		);

		/**
		 * Membersihkan geometri dan material model.
		 */
		function disposeSun(object: THREE.Object3D) {
			object.traverse((child) => {
				if (!(child instanceof THREE.Mesh)) return;

				child.geometry.dispose();

				const materials = Array.isArray(child.material) ? child.material : [child.material];

				for (const material of materials) {
					material.dispose();
				}
			});
		}

		/**
		 * Membersihkan semua resource saat komponen dihancurkan.
		 */
		return () => {
			isDisposed = true;

			if (sun) {
				scene.remove(sun);
				disposeSun(sun);
				sun = null;
			}

			for (const sprite of glowSprites) {
				scene.remove(sprite);
			}

			for (const material of glowMaterials) {
				material.dispose();
			}

			for (const texture of glowTextures) {
				texture.dispose();
			}

			glowSprites.length = 0;
			glowMaterials.length = 0;
			glowTextures.length = 0;
		};
	});
</script>
