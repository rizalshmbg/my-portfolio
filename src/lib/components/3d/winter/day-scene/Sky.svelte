<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';

	interface Props {
		scene: THREE.Scene;
	}

	let { scene }: Props = $props();

	onMount(() => {
		// Shader untuk membuat gradasi warna langit.
		const geometry = new THREE.SphereGeometry(90, 32, 24);

		const material = new THREE.ShaderMaterial({
			vertexShader: `
				varying vec3 vWorldPosition;

				void main() {
					vec4 worldPosition = modelMatrix * vec4(position, 1.0);
					vWorldPosition = worldPosition.xyz;

					gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
				}
			`,
			fragmentShader: `
				varying vec3 vWorldPosition;

				void main() {
					float height = normalize(vWorldPosition).y;

					vec3 horizonColor = vec3(0.35, 0.65, 0.90);
					vec3 middleColor = vec3(0.16, 0.45, 0.82);
					vec3 topColor = vec3(0.02, 0.22, 0.65);

					float horizonToMiddle = smoothstep(-0.12, 0.18, height);
					float middleToTop = smoothstep(0.12, 0.85, height);

					vec3 color = mix(horizonColor, middleColor, horizonToMiddle);
					color = mix(color, topColor, middleToTop);

					gl_FragColor = vec4(color, 1.0);

					#include <tonemapping_fragment>
					#include <colorspace_fragment>
				}
			`,
			side: THREE.BackSide,
			depthWrite: false,
			depthTest: true,
			toneMapped: false
		});

		const sky = new THREE.Mesh(geometry, material);
		sky.name = 'WinterDaySky';
		sky.renderOrder = -1000;

		scene.add(sky);

		return () => {
			scene.remove(sky);
			geometry.dispose();
			material.dispose();
		};
	});
</script>
