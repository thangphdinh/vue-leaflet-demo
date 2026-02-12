<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import L from 'leaflet'
import type { Map } from 'leaflet'

// Props
interface Props {
  height?: string
}

const props = withDefaults(defineProps<Props>(), {
  height: '500px'
})

// Refs
const mapContainer = ref<HTMLElement | null>(null)
let map: Map | null = null

// Initialize map
onMounted(() => {
  if (!mapContainer.value) return

  // Image dimensions and tile info
  const imageSize = 2048
  const tileSize = 256

  // Setup map
  map = L.map(mapContainer.value, {
    crs: L.CRS.Simple,
    // minZoom: mapMinZoom,
    // maxZoom: mapMaxZoom,
  })
  //.setView([0, 0], 0)

  const boundsSize = tileSize

  const bounds: L.LatLngBoundsExpression = [
    [0, 0],
    [-boundsSize, boundsSize]
  ]

  L.tileLayer('tiles/{z}/{x}/{y}.png', {
    attribution: 'Ragnasia Fantasy Map',
    noWrap: true,
    tileSize: tileSize,
    minZoom: 1,
    maxZoom: 3,
  }).addTo(map)

  map.setView([-boundsSize/2, boundsSize/2], 1)

  // Fit bounds to show full map
  // map.fitBounds(bounds)

  // Set max bounds with padding
  map.setMaxBounds(bounds)
  // map.setMaxBounds([
  //   [0, 0],
  //   // [-boundsSize + 100, boundsSize + 100]
  //   [-boundsSize, boundsSize]
  // ])

  // Add example markers (scaled to bounds)
  // Original coords were based on imageSize (2048), now scale to boundsSize (256)
  const scaleFactor = boundsSize / imageSize  // 256 / 2048 = 0.125
  const locations = [
    { name: 'Northern Region', coords: [-400 * scaleFactor, 1024 * scaleFactor] as [number, number] },
    { name: 'Central Island', coords: [-1024 * scaleFactor, 1024 * scaleFactor] as [number, number] },
    { name: 'Southern Coast', coords: [-1600 * scaleFactor, 1024 * scaleFactor] as [number, number] },
  ]

  locations.forEach(location => {
    L.marker(location.coords)
      .addTo(map!)
      .bindPopup(`<strong>${location.name}</strong>`)
  })
})

// Cleanup
onUnmounted(() => {
  if (map) {
    map.remove()
    map = null
  }
})
</script>

<template>
  <div class="w-full">
    <div
      ref="mapContainer"
      class="w-full rounded-lg shadow-lg border-4 border-amber-900"
      :style="{ height: props.height }"
    ></div>
  </div>
</template>

<style scoped>
/* Add fantasy map styling */
:deep(.leaflet-container) {
  /* Inherit height from parent - don't set fixed height here! */
  background: #3d6b8a;
  font-family: 'Georgia', serif;
}

:deep(.leaflet-popup-content-wrapper) {
  background: #f4e4c1;
  color: #2c1810;
  border: 2px solid #8b6914;
}

:deep(.leaflet-popup-tip) {
  background: #f4e4c1;
}
</style>
