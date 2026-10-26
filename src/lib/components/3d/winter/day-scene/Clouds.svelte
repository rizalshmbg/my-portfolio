<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
		registerUpdate?: (callback: (time: number) => void) => void;
	}

	let { scene, registerUpdate }: Props = $props();

	interface CloudLayer {
		width: number;
		height: number;
		position: THREE.Vector3;
		color: string;
		opacity: number;
		seed: number;
		speed: number;
		direction: number;
	}

	onMount(() => {
		let isDisposed = false;

		const cloudMeshes: THREE.Mesh[] = [];
		const cloudTextures: THREE.CanvasTexture[] = [];
		const cloudMaterials: THREE.MeshBasicMaterial[] = [];

		const animatedClouds: {
			mesh: THREE.Mesh;
			startX: number;
			speed: number;
			direction: number;
		}[] = [];

		/**
		 * Membentuk siluet awan dengan kurva Bézier.
		 * Seed menjaga bentuk awan tetap konsisten setiap refresh.
		 */
		function drawCloudSilhouette(
			context: CanvasRenderingContext2D,
			width: number,
			height: number,
			seed: number
		) {
			const variation = (index: number) => {
				const value = Math.sin((index + 1) * 127.1 + seed * 311.7) * 43758.5453;

				return (value - Math.floor(value)) * 2 - 1;
			};

			context.beginPath();
			context.moveTo(0, height * 0.72);

			const lobes = [
				{ x: 0.04, y: 0.61, r: 0.025 },
				{ x: 0.1, y: 0.48, r: 0.045 },
				{ x: 0.17, y: 0.57, r: 0.035 },
				{ x: 0.23, y: 0.39, r: 0.055 },
				{ x: 0.31, y: 0.52, r: 0.035 },
				{ x: 0.38, y: 0.43, r: 0.05 },
				{ x: 0.46, y: 0.56, r: 0.035 },
				{ x: 0.54, y: 0.37, r: 0.055 },
				{ x: 0.62, y: 0.5, r: 0.04 },
				{ x: 0.7, y: 0.43, r: 0.05 },
				{ x: 0.78, y: 0.55, r: 0.035 },
				{ x: 0.86, y: 0.41, r: 0.05 },
				{ x: 0.94, y: 0.53, r: 0.035 },
				{ x: 1, y: 0.62, r: 0.02 }
			];

			for (let i = 0; i < lobes.length; i++) {
				const current = lobes[i];
				const next = lobes[i + 1];

				const x = current.x * width;
				const y = (current.y + variation(i) * current.r * 0.3) * height;

				const radius = current.r * width;

				if (!next) {
					context.quadraticCurveTo(x, y, width, height * 0.66);

					break;
				}

				const nextX = next.x * width;
				const nextY = (next.y + variation(i + 1) * next.r * 0.3) * height;

				context.bezierCurveTo(
					x - radius,
					y - current.r * height * 0.5,
					nextX - radius * 0.4,
					nextY - next.r * height * 0.5,
					(nextX + x) / 2,
					(nextY + y) / 2
				);
			}

			context.lineTo(width, height * 0.79);

			// Kontur bawah awan.
			context.bezierCurveTo(
				width * 0.93,
				height * 0.75,
				width * 0.91,
				height * 0.84,
				width * 0.84,
				height * 0.79
			);

			context.bezierCurveTo(
				width * 0.76,
				height * 0.74,
				width * 0.72,
				height * 0.86,
				width * 0.64,
				height * 0.81
			);

			context.bezierCurveTo(
				width * 0.56,
				height * 0.77,
				width * 0.51,
				height * 0.85,
				width * 0.43,
				height * 0.81
			);

			context.bezierCurveTo(
				width * 0.34,
				height * 0.76,
				width * 0.29,
				height * 0.86,
				width * 0.21,
				height * 0.81
			);

			context.bezierCurveTo(
				width * 0.13,
				height * 0.76,
				width * 0.08,
				height * 0.82,
				0,
				height * 0.78
			);

			context.closePath();
		}

		/**
		 * Membuat tekstur awan dengan warna siang.
		 */
		function createCloud(layer: CloudLayer) {
			const canvas = document.createElement('canvas');

			canvas.width = 1024;
			canvas.height = 256;

			const context = canvas.getContext('2d');

			if (!context) return;

			// Menggambar bentuk awan.
			drawCloudSilhouette(context, canvas.width, canvas.height, layer.seed);

			// Gradasi putih pada bagian atas dan biru muda
			// pada bagian bawah.
			const gradient = context.createLinearGradient(0, 0, 0, canvas.height);

			gradient.addColorStop(0, '#FFFFFF');
			gradient.addColorStop(0.3, '#FFFFFF');
			gradient.addColorStop(0.58, '#F8FBFF');
			gradient.addColorStop(0.78, '#E8F1FB');
			gradient.addColorStop(1, '#D0E1F2');

			context.fillStyle = gradient;
			context.fill();

			// Bayangan biru lembut di bagian bawah awan.
			context.save();

			context.globalCompositeOperation = 'source-atop';

			const shadowGradient = context.createLinearGradient(
				0,
				canvas.height * 0.55,
				0,
				canvas.height
			);

			shadowGradient.addColorStop(0, 'rgba(210, 228, 247, 0)');
			shadowGradient.addColorStop(0.45, 'rgba(190, 215, 241, 0.10)');
			shadowGradient.addColorStop(1, 'rgba(165, 195, 226, 0.30)');

			context.fillStyle = shadowGradient;

			context.fillRect(0, canvas.height * 0.5, canvas.width, canvas.height * 0.5);

			context.restore();

			// Sedikit blur agar tepi awan menyatu dengan langit.
			context.save();

			context.globalCompositeOperation = 'source-in';
			context.filter = 'blur(2px)';
			context.drawImage(canvas, 0, 0);

			context.restore();

			// Mengubah Canvas menjadi tekstur Three.js.
			const texture = new THREE.CanvasTexture(canvas);

			texture.colorSpace = THREE.SRGBColorSpace;
			texture.minFilter = THREE.LinearFilter;
			texture.magFilter = THREE.LinearFilter;

			// PlaneGeometry mengikuti pendekatan Clouds.svelte malam.
			const geometry = new THREE.PlaneGeometry(layer.width, layer.height);

			const material = new THREE.MeshBasicMaterial({
				map: texture,
				color: '#FFFFFF',
				transparent: true,
				opacity: layer.opacity,
				side: THREE.DoubleSide,
				depthTest: true,
				depthWrite: false,
				alphaTest: 0.01,
				toneMapped: false
			});

			const mesh = new THREE.Mesh(geometry, material);

			mesh.position.copy(layer.position);

			scene.add(mesh);

			cloudMeshes.push(mesh);
			cloudTextures.push(texture);
			cloudMaterials.push(material);

			animatedClouds.push({
				mesh,
				startX: layer.position.x,
				speed: layer.speed,
				direction: layer.direction
			});
		}

		// --------------------------------------------------
		// MEMBUAT KELOMPOK AWAN SIANG
		// --------------------------------------------------

		// Kiri.
		createCloud({
			width: 15,
			height: 8,
			position: new THREE.Vector3(-45, 11, -49),
			color: '#FFFFFF',
			opacity: 0.98,
			seed: 20,
			speed: 0.3,
			direction: 1
		});

		createCloud({
			width: 13,
			height: 8,
			position: new THREE.Vector3(-28, 8, -49),
			color: '#F8FBFF',
			opacity: 0.88,
			seed: 35,
			speed: 0.08,
			direction: -1
		});

		createCloud({
			width: 17,
			height: 9,
			position: new THREE.Vector3(-10, 6.5, -49),
			color: '#FFFFFF',
			opacity: 0.96,
			seed: 120,
			speed: 0.06,
			direction: 1
		});

		// Tengah.
		createCloud({
			width: 10,
			height: 6,
			position: new THREE.Vector3(9, 7.5, -49),
			color: '#F8FBFF',
			opacity: 0.82,
			seed: 86,
			speed: 0.14,
			direction: -1
		});

		// Kanan.
		createCloud({
			width: 15,
			height: 10,
			position: new THREE.Vector3(26, 7, -49),
			color: '#FFFFFF',
			opacity: 0.96,
			seed: 62,
			speed: 0.25,
			direction: -1
		});

		createCloud({
			width: 10,
			height: 8,
			position: new THREE.Vector3(45, 8, -49),
			color: '#F8FBFF',
			opacity: 0.88,
			seed: 95,
			speed: 0.17,
			direction: 1
		});

		// --------------------------------------------------
		// ANIMASI HORIZONTAL
		// --------------------------------------------------

		function updateClouds(time: number) {
			if (isDisposed) return;

			for (const cloud of animatedClouds) {
				cloud.mesh.position.x = cloud.startX + Math.sin(time * cloud.speed) * 0.8 * cloud.direction;
			}
		}

		registerUpdate?.(updateClouds);

		// --------------------------------------------------
		// CLEANUP
		// --------------------------------------------------

		return () => {
			isDisposed = true;

			for (const mesh of cloudMeshes) {
				scene.remove(mesh);
				mesh.geometry.dispose();
			}

			for (const material of cloudMaterials) {
				material.dispose();
			}

			for (const texture of cloudTextures) {
				texture.dispose();
			}

			registerUpdate?.(() => {});

			cloudMeshes.length = 0;
			cloudTextures.length = 0;
			cloudMaterials.length = 0;
			animatedClouds.length = 0;
		};
	});
</script>
