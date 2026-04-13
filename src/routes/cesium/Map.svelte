<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	import { CESIUM_INITIAL_OPTIONS } from './cesiumUtil';
	import type * as CesiumType from 'cesium';
	import type { EntityPopupAPI } from '$lib/types';
	import Entities from './Entities.svelte';
	import { EntityPopup as LibraryEntityPopup } from '$lib';

	let { popupOptions, popupApi = $bindable() }: { popupOptions: any; popupApi?: EntityPopupAPI } =
		$props();
	// Cesium module is dynamically imported, so use import type
	let cesium: typeof CesiumType | undefined = $state();
	let viewer: CesiumType.Viewer | undefined = $state();
	let viewerReady = $state(false); // Flag to track viewer readiness

	onMount(async (): Promise<void> => {
		if (!browser) return;

		try {
			// Import Cesium module only in browser
			cesium = (await import('cesium')) as typeof CesiumType;
			await import('cesium/Build/Cesium/Widgets/widgets.css');

			// Get necessary exports from Cesium module
			const {
				UrlTemplateImageryProvider,
				CesiumTerrainProvider,
				Viewer: CesiumViewer,
				Cartesian3,
				Math: CesiumMath,
				Ion
			} = cesium;

			Ion.defaultAccessToken = import.meta.env.VITE_CESIUM_ION_ACCESS_TOKEN;

			// Configure imagery provider (GSI)
			const gsiSeamless = new UrlTemplateImageryProvider({
				//url: 'https://cyberjapandata.gsi.go.jp/xyz/seamlessphoto/{z}/{x}/{y}.jpg'
				url: 'http://localhost:3000/satellite_0_8_world_v1.0/{z}/{x}/{y}'
			});

			const gsiSeamlessItaly = new UrlTemplateImageryProvider({
				url: 'http://localhost:3000/satellite_8_15_Italy_v1.1/{z}/{x}/{y}'
			});

			const gsiSeamlessSwiss = new UrlTemplateImageryProvider({
				url: 'http://localhost:3000/satellite_8_15_Switzerland_v1.0/{z}/{x}/{y}'
			});

			const gsiSeamlessInterlaken = new UrlTemplateImageryProvider({
				url: 'http://localhost:3000/satellite_15_19_Interlaken_v1.0/{z}/{x}/{y}'
			});

			// Configure CESIUM_BASE_URL
			const baseUrl = import.meta.env.DEV ? '/node_modules/cesium/Build/Cesium/' : '/Cesium/';
			// @ts-expect-error - Global CESIUM_BASE_URL setting for Cesium
			window.CESIUM_BASE_URL = baseUrl;

			// Initialize Viewer (assign to viewer variable)
			viewer = new CesiumViewer('cesiumContainer', {
				...CESIUM_INITIAL_OPTIONS
			});

			// Configure terrain provider
			//viewer.terrainProvider = await cesium.createWorldTerrainAsync();
			viewer.terrainProvider = await CesiumTerrainProvider.fromUrl('http://localhost:3003');

			// Add imagery provider
			viewer.imageryLayers.addImageryProvider(gsiSeamless);
			viewer.imageryLayers.addImageryProvider(gsiSeamlessItaly);
			viewer.imageryLayers.addImageryProvider(gsiSeamlessSwiss);
			viewer.imageryLayers.addImageryProvider(gsiSeamlessInterlaken);

			// Set initial camera position
			viewer.camera.setView({
				destination: Cartesian3.fromDegrees(
					8.649821675420293, // Longitude
					45.9262704067969, // Latitude
					2000 // Altitude (meters)
				),
				orientation: {
					heading: CesiumMath.toRadians(220), // View direction (heading)
					pitch: CesiumMath.toRadians(-20), // Pitch (tilt)
					roll: 0 // Roll (rotation)
				}
			});
			// Disable transparency against terrain
			viewer.scene.globe.depthTestAgainstTerrain = false;

			// Indicate that viewer is ready
			viewerReady = true;
		} catch (error) {
			console.error('Failed to initialize Cesium:', error);
		}
	});
</script>

<!-- Container for rendering Cesium -->
<div id="cesiumContainer" class="h-full w-full"></div>
<!-- Display Entities component and EntityPopup when viewer is ready -->
{#if viewerReady && viewer && cesium}
	<Entities {viewer} {cesium} />
	<!-- Use library's EntityPopup component -->
	<LibraryEntityPopup bind:this={popupApi} {viewer} {cesium} options={popupOptions} />
{/if}
