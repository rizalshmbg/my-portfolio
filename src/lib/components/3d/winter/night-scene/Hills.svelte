<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	type HillShape = 'peak' | 'rolling' | 'ridge';
	type GradientDirection = 'vertical' | 'horizontal' | 'diagonal';

	let { scene }: Props = $props();

	onMount(() => {
		const hills: THREE.Mesh[] = [];
		const hillTextures: THREE.CanvasTexture[] = [];

		function createSnowFacet(
			points: Array<[number, number]>,
			width: number,
			height: number,
			position: THREE.Vector3,
			color: string,
			depthOffset = 0.08
		) {
			const shape = new THREE.Shape();

			shape.moveTo(points[0][0] * width, points[0][1] * height);

			for (let i = 1; i < points.length; i++) {
				shape.lineTo(points[i][0] * width, points[i][1] * height);
			}

			shape.closePath();

			const geometry = new THREE.ShapeGeometry(shape);

			const material = new THREE.MeshBasicMaterial({
				color,
				side: THREE.DoubleSide,
				depthTest: true,
				depthWrite: false
			});

			const snow = new THREE.Mesh(geometry, material);

			snow.position.copy(position);
			snow.position.z += depthOffset;

			scene.add(snow);
			hills.push(snow);
		}

		function createPeakSnow(width: number, height: number, position: THREE.Vector3) {
			// Salju utama di puncak.
			createSnowFacet(
				[
					[-0.18, 0.55],
					[-0.04, 1.0],
					[0.08, 0.62],
					[0.015, 0.72],
					[-0.025, 0.66],
					[-0.08, 0.58]
				],
				width,
				height,
				position,
				'#46566F'
			);

			// Bayangan biru muda di sisi kanan.
			createSnowFacet(
				[
					[-0.04, 1.0],
					[0.08, 0.62],
					[0.015, 0.72]
				],
				width,
				height,
				position,
				'#35445C',
				0.09
			);

			// Ujung salju di lereng kiri.
			createSnowFacet(
				[
					[-0.18, 0.55],
					[-0.08, 0.58],
					[-0.25, 0.36]
				],
				width,
				height,
				position,
				'#252F42',
				0.12
			);

			// Ujung salju di lereng kanan.
			createSnowFacet(
				[
					[0.015, 0.72],
					[0.08, 0.62],
					[0.22, 0.35],
					[0.1, 0.54]
				],
				width,
				height,
				position,
				'#252F42',
				0.12
			);
		}

		function createRidgeSnow(width: number, height: number, position: THREE.Vector3) {
			// Salju pada puncak kiri.
			createSnowFacet(
				[
					[-0.38, 0.25],
					[-0.23, 0.48],
					[-0.08, 0.34],
					[-0.18, 0.4],
					[-0.23, 0.39]
				],
				width,
				height,
				position,
				'#46566F'
			);

			// Ujung salju pada lereng kiri.
			createSnowFacet(
				[
					[-0.23, 0.39],
					[-0.08, 0.34],
					[-0.035, 0.27],
					[-0.14, 0.35]
				],
				width,
				height,
				position,
				'#252F42',
				0.12
			);

			// Salju pada puncak utama.
			createSnowFacet(
				[
					[-0.08, 0.34],
					[0.08, 0.68],
					[0.22, 0.42],
					[0.13, 0.51],
					[0.015, 0.53]
				],
				width,
				height,
				position,
				'#46566F'
			);

			// Bayangan pada sisi kanan puncak utama.
			createSnowFacet(
				[
					[0.08, 0.68],
					[0.22, 0.42],
					[0.13, 0.51]
				],
				width,
				height,
				position,
				'#35445C',
				0.09
			);
		}

		function createHill(
			width: number,
			height: number,
			color: string,
			position: THREE.Vector3,
			shapeType: HillShape = 'peak',
			gradientDirection: GradientDirection = 'vertical'
		) {
			const shape = new THREE.Shape();

			shape.moveTo(-width / 2, -height * 0.15);

			if (shapeType === 'peak') {
				// Gunung lancip
				shape.lineTo(-width * 0.38, height * 0.2);
				shape.lineTo(-width * 0.18, height * 0.55);
				shape.lineTo(-width * 0.04, height);
				shape.lineTo(width * 0.08, height * 0.62);
				shape.lineTo(width * 0.22, height * 0.35);
				shape.lineTo(width * 0.38, height * 0.18);
			} else if (shapeType === 'rolling') {
				// Bukit landai dengan puncak membulat
				shape.lineTo(-width * 0.38, height * 0.15);

				shape.quadraticCurveTo(-width * 0.2, height * 0.85, -width * 0.02, height * 0.65);

				shape.quadraticCurveTo(width * 0.2, height * 0.48, width * 0.36, height * 0.12);
			} else {
				// Punggung bukit dengan beberapa puncak
				shape.lineTo(-width * 0.38, height * 0.25);
				shape.lineTo(-width * 0.23, height * 0.48);
				shape.lineTo(-width * 0.08, height * 0.34);
				shape.lineTo(width * 0.08, height * 0.68);
				shape.lineTo(width * 0.22, height * 0.42);
				shape.lineTo(width * 0.38, height * 0.2);
			}

			shape.lineTo(width / 2, -height * 0.15);
			shape.closePath();

			const geometry = new THREE.ShapeGeometry(shape, 16);

			// Normalisasi UV berdasarkan ketinggian gunung
			const positions = geometry.attributes.position;
			const uvs = geometry.attributes.uv;

			let minY = Infinity;
			let maxY = -Infinity;

			for (let i = 0; i < positions.count; i++) {
				const y = positions.getY(i);

				minY = Math.min(minY, y);
				maxY = Math.max(maxY, y);
			}

			const rangeY = maxY - minY || 1;

			for (let i = 0; i < positions.count; i++) {
				const y = positions.getY(i);
				const v = (y - minY) / rangeY;

				uvs.setXY(i, uvs.getX(i), v);
			}

			uvs.needsUpdate = true;

			// Canvas untuk gradient
			const gradientCanvas = document.createElement('canvas');
			gradientCanvas.width = 512;
			gradientCanvas.height = 512;

			const context = gradientCanvas.getContext('2d');

			if (!context) {
				geometry.dispose();
				return;
			}

			const baseColor = new THREE.Color(color);
			const middleColor = baseColor.clone().lerp(new THREE.Color('#17191e'), 0.2);
			const bottomColor = baseColor.clone().lerp(new THREE.Color('#050608'), 0.5);

			if (gradientDirection === 'diagonal') {
				// Lapisan pertama: gradasi horizontal
				const horizontalGradient = context.createLinearGradient(0, 0, 512, 0);

				horizontalGradient.addColorStop(0, baseColor.getStyle());
				horizontalGradient.addColorStop(0.45, middleColor.getStyle());
				horizontalGradient.addColorStop(1, bottomColor.getStyle());

				context.fillStyle = horizontalGradient;
				context.fillRect(0, 0, 512, 512);

				// Lapisan kedua: gradasi vertikal transparan
				const verticalCanvas = document.createElement('canvas');
				verticalCanvas.width = 512;
				verticalCanvas.height = 512;

				const verticalContext = verticalCanvas.getContext('2d');

				if (verticalContext) {
					const verticalGradient = verticalContext.createLinearGradient(0, 0, 0, 512);

					verticalGradient.addColorStop(0, 'rgba(0, 0, 0, 0.2)');
					verticalGradient.addColorStop(0.5, 'rgba(0, 0, 0, 0.04)');
					verticalGradient.addColorStop(1, 'rgba(0, 0, 0, 0.3)');

					verticalContext.fillStyle = verticalGradient;
					verticalContext.fillRect(0, 0, 512, 512);

					context.drawImage(verticalCanvas, 0, 0);
				}
			} else {
				const gradient =
					gradientDirection === 'horizontal'
						? context.createLinearGradient(0, 0, 512, 0)
						: context.createLinearGradient(0, 0, 0, 512);

				if (gradientDirection === 'horizontal') {
					const horizontalMiddle = baseColor.clone().lerp(new THREE.Color('#17191e'), 0.1);
					const horizontalEnd = baseColor.clone().lerp(new THREE.Color('#111318'), 0.25);

					gradient.addColorStop(0, baseColor.getStyle());
					gradient.addColorStop(0.5, horizontalMiddle.getStyle());
					gradient.addColorStop(1, horizontalEnd.getStyle());
				} else {
					gradient.addColorStop(0, bottomColor.getStyle());
					gradient.addColorStop(0.45, middleColor.getStyle());
					gradient.addColorStop(1, baseColor.getStyle());
				}

				context.fillStyle = gradient;
				context.fillRect(0, 0, 512, 512);
			}

			const gradientTexture = new THREE.CanvasTexture(gradientCanvas);
			gradientTexture.colorSpace = THREE.SRGBColorSpace;

			const material = new THREE.MeshStandardMaterial({
				map: gradientTexture,
				color: '#c2cad5',
				roughness: 1,
				metalness: 0,
				side: THREE.DoubleSide
			});

			const hill = new THREE.Mesh(geometry, material);

			hill.position.copy(position);

			scene.add(hill);
			hills.push(hill);
			hillTextures.push(gradientTexture);

			if (shapeType === 'peak') {
				createPeakSnow(width, height, position);
			} else if (shapeType === 'ridge') {
				createRidgeSnow(width, height, position);
			}
		}

		createHill(90, 12, '#343d4d', new THREE.Vector3(-60, 0, -48), 'rolling', 'diagonal');
		createHill(30, 9, '#394353', new THREE.Vector3(-28, -0.5, -47), 'peak', 'horizontal');
		createHill(35, 10, '#465165', new THREE.Vector3(0, 0, -46), 'ridge', 'vertical');
		createHill(50, 5, '#394353', new THREE.Vector3(29, 0, -47), 'rolling', 'diagonal');
		createHill(50, 7, '#3b4557', new THREE.Vector3(45, 0, -48), 'peak', 'vertical');

		return () => {
			for (const hill of hills) {
				scene.remove(hill);
				hill.geometry.dispose();

				const material = hill.material;
				if (material instanceof THREE.Material) {
					material.dispose();
				}
			}

			for (const texture of hillTextures) {
				texture.dispose();
			}
		};
	});
</script>
