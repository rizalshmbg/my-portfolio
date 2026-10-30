<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
		registerUpdate?: (callback: ((time: number) => void) | undefined) => void;
	}

	let { scene, registerUpdate }: Props = $props();

	onMount(() => {
		const COUNT = 200;

		const positions = new Float32Array(COUNT * 3);
		const fallSpeeds = new Float32Array(COUNT);
		const drift = new Float32Array(COUNT);
		const sizes = new Float32Array(COUNT);

		const bounds = {
			x: 32,
			yMin: -1,
			yMax: 20,
			zMin: -30,
			zMax: 8
		};

		for (let i = 0; i < COUNT; i++) {
			positions[i * 3] = THREE.MathUtils.randFloatSpread(bounds.x * 2);
			positions[i * 3 + 1] = THREE.MathUtils.randFloat(bounds.yMin, bounds.yMax);
			positions[i * 3 + 2] = THREE.MathUtils.randFloat(bounds.zMin, bounds.zMax);

      fallSpeeds[i] = THREE.MathUtils.randFloat(0.3, 0.8);
			// fallSpeeds[i] = THREE.MathUtils.randFloat(1.2, 3.0);
			drift[i] = THREE.MathUtils.randFloat(-0.35, 0.35);
			sizes[i] = THREE.MathUtils.randFloat(0.035, 0.075);
      // sizes[i] = THREE.MathUtils.randFloat(0.07, 0.12);
		}

		const geometry = new THREE.BufferGeometry();
		geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
		geometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

		const material = new THREE.PointsMaterial({
			color: '#e6edf7',
			size: 0.09,
			transparent: true,
			opacity: 0.72,
			depthWrite: false,
			sizeAttenuation: true
		});

		const snow = new THREE.Points(geometry, material);
		snow.name = 'Snowfall';
		snow.frustumCulled = false;
		scene.add(snow);

		let previousTime = 0;

		const updateSnow = (time: number) => {
			const delta = previousTime === 0 ? 0 : Math.min((time - previousTime) / 1000, 0.05);

			previousTime = time;

			const attribute = geometry.getAttribute('position');
			const array = attribute.array as Float32Array;

			for (let i = 0; i < COUNT; i++) {
				const idx = i * 3;

				array[idx + 1] -= fallSpeeds[i] * delta;
				array[idx] += drift[i] * delta;

				// Angin lembut agar salju tidak jatuh lurus.
				array[idx] += Math.sin(time * 0.001 + i) * 0.002;

				if (array[idx + 1] < bounds.yMin) {
					array[idx + 1] = bounds.yMax;
					array[idx] = THREE.MathUtils.randFloatSpread(bounds.x * 2);
					array[idx + 2] = THREE.MathUtils.randFloat(bounds.zMin, bounds.zMax);
				}

				if (array[idx] < -bounds.x) array[idx] = bounds.x;
				if (array[idx] > bounds.x) array[idx] = -bounds.x;
			}

			attribute.needsUpdate = true;
		};

		registerUpdate?.(updateSnow);

		return () => {
			registerUpdate?.(undefined);
			scene.remove(snow);
			geometry.dispose();
			material.dispose();
		};
	});
</script>
