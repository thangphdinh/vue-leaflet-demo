<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import L from 'leaflet'
import type { Map, Marker } from 'leaflet'

// Props
interface Props {
  center?: [number, number]
  zoom?: number
  height?: string
}

const props = withDefaults(defineProps<Props>(), {
  center: () => [21.0285, 105.8542], // Hà Nội, Việt Nam
  zoom: 13,
  height: '600px'
})

// Refs
const mapContainer = ref<HTMLElement | null>(null)
let map: Map | null = null
const markers: Marker[] = []

// Initialize map
onMounted(() => {
  if (!mapContainer.value) return

  // Create map instance
  map = L.map(mapContainer.value).setView(props.center, props.zoom)

  // Add OpenStreetMap tile layer
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    maxZoom: 19,
  }).addTo(map)

  // Add some example markers
  const locations = [
    { name: 'Hồ Hoàn Kiếm', coords: [21.0285, 105.8542] as [number, number] },
    { name: 'Văn Miếu Quốc Tử Giám', coords: [21.0277, 105.8355] as [number, number] },
    { name: 'Lăng Chủ tịch Hồ Chí Minh', coords: [21.0369, 105.8345] as [number, number] },
  ]

  locations.forEach(location => {
    const marker = L.marker(location.coords)
      .addTo(map!)
      .bindPopup(`<strong>${location.name}</strong>`)
    markers.push(marker)
  })

  // Add a circle
  L.circle(props.center, {
    color: 'red',
    fillColor: '#f03',
    fillOpacity: 0.3,
    radius: 1000
  }).addTo(map)
    .bindPopup('Khu vực trung tâm Hà Nội')
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
      class="w-full rounded-lg shadow-lg"
      :style="{ height: props.height }"
    ></div>
  </div>
</template>
