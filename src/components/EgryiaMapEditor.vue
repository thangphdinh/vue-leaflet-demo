<script setup lang="ts">
import { onMounted, onUnmounted, ref, computed } from 'vue'
import L from 'leaflet'
import type { Map as LeafletMap } from 'leaflet'

// Props
interface Props {
  height?: string
}

const props = withDefaults(defineProps<Props>(), {
  height: '600px'
})

// Types - chỉ chứa plain data, không chứa Leaflet objects
interface MarkerData {
  id: string
  name: string
  lat: number
  lng: number
}

// Refs - chỉ chứa plain data, Vue có thể reactive an toàn
const mapContainer = ref<HTMLElement | null>(null)
let map: LeafletMap | null = null
const markers = ref<MarkerData[]>([])
const editingMarkerId = ref<string | null>(null)
const editingName = ref('')

// Leaflet marker instances - plain JS Map, KHÔNG reactive, tránh Vue proxy
const markerInstances = new Map<string, L.Marker>()

// Counter chỉ tăng, không bao giờ giảm khi xóa marker
let markerCounter = 0

// async function generateMarkerName(): Promise<string> {
//   try {
//     const res = await fetch('https://random-city-api.vercel.app/api/random-city')
//     if (!res.ok) throw new Error('API error')
//     const content: { city: string } = await res.json()
//     return content.city
//   } catch {
//     // markerCounter++
//     return `Location ${markerCounter}`
//   }
// }

// Computed
const markersJson = computed(() => {
  return JSON.stringify(
    markers.value.map(m => ({
      name: m.name,
      coords: [m.lat, m.lng]
    })),
    null,
    2
  )
})

// Initialize map
onMounted(() => {
  if (!mapContainer.value) return

  // Image dimensions
  const imageSize = 8192
  const tileSize = 256
  const tileMaxZoom = 5

  // Setup map
  map = L.map(mapContainer.value, {
    crs: L.CRS.Simple,
    // minZoom: 0
  })

  const bounds: L.LatLngBoundsExpression = [[0, 0], [-tileSize, tileSize]]

  // Add tile layer
  L.tileLayer('map/{z}/{x}/{y}.png', {
    attribution: 'Egryia Fantasy Map',
    noWrap: true,
    tileSize: tileSize,
    zoomOffset: 0,
    minZoom: 1,
    maxZoom: tileMaxZoom,
    // maxNativeZoom: 5,
    continuousWorld: false
  }).addTo(map)

  map.fitBounds(bounds)
  map.setMaxBounds(bounds)

  // Add click event to create markers
  map.on('click', (e) => {
    addMarker(e.latlng.lat, e.latlng.lng)
  })
})

// Add marker function
async function addMarker(lat: number, lng: number, name?: string) {
  if (!map) return

  const id = `marker-${Date.now()}`

  // Tên tạm trong lúc chờ API
  markerCounter++
  const tempName = `Location ${markerCounter}`

  // Test custom marker icon
  // const markerIcon = L.icon({
  //   iconUrl: `icons/icons8-shipping-container-96.png`,
  //   iconSize: [28, 40],
  //   iconAnchor: [22.5, 60],
  //   popupAnchor: [0, -62]
  // });

  const leafletMarker = L.marker([lat, lng], {
    // icon: markerIcon,
    draggable: true
  }).addTo(map)
  leafletMarker.bindPopup(`<strong>${tempName}</strong><br>Lat: ${lat.toFixed(4)}, Lng: ${lng.toFixed(4)}`)

  // Update position on drag
  leafletMarker.on('dragend', () => {
    const pos = leafletMarker.getLatLng()
    const markerData = markers.value.find(m => m.id === id)
    if (markerData) {
      markerData.lat = pos.lat
      markerData.lng = pos.lng
      leafletMarker.getPopup()?.setContent(`<strong>${markerData.name}</strong><br>Lat: ${pos.lat.toFixed(4)}, Lng: ${pos.lng.toFixed(4)}`)
    }
  })

  markerInstances.set(id, leafletMarker)
  markers.value.push({ id, name: tempName, lat, lng })

  // Fetch tên từ API
  // if (!name) {
  //   const generatedName = await generateMarkerName()
  //   const markerData = markers.value.find(m => m.id === id)
  //   if (markerData) {
  //     markerData.name = generatedName
  //     leafletMarker.getPopup()?.setContent(`<strong>${generatedName}</strong><br>Lat: ${lat.toFixed(4)}, Lng: ${lng.toFixed(4)}`)
  //   }
  // }
}

// Delete marker
function deleteMarker(id: string) {
  const index = markers.value.findIndex(m => m.id === id)
  if (index !== -1) {
    const leafletMarker = markerInstances.get(id)
    if (leafletMarker) {
      leafletMarker.off()
      if (map) map.removeLayer(leafletMarker)
    }
    markerInstances.delete(id)
    markers.value.splice(index, 1)
  }
}

// Start editing marker name
function startEdit(id: string, currentName: string) {
  editingMarkerId.value = id
  editingName.value = currentName
}

// Save edited name
function saveEdit(id: string) {
  const markerData = markers.value.find(m => m.id === id)
  if (markerData && editingName.value.trim()) {
    markerData.name = editingName.value.trim()
    markerInstances.get(id)?.getPopup()?.setContent(
      `<strong>${markerData.name}</strong><br>Lat: ${markerData.lat.toFixed(4)}, Lng: ${markerData.lng.toFixed(4)}`
    )
  }
  editingMarkerId.value = null
  editingName.value = ''
}

// Cancel editing
function cancelEdit() {
  editingMarkerId.value = null
  editingName.value = ''
}

// Copy JSON to clipboard
function copyJson() {
  navigator.clipboard.writeText(markersJson.value)
  alert('JSON copied to clipboard!')
}

// Clear all markers
function clearAll() {
  if (confirm('Are you sure you want to delete all markers?')) {
    markerInstances.forEach((leafletMarker) => {
      leafletMarker.off()
      if (map) map.removeLayer(leafletMarker)
    })
    markerInstances.clear()
    markers.value = []
  }
}

// Zoom to marker
function zoomToMarker(markerData: MarkerData) {
  if (map) {
    map.setView([markerData.lat, markerData.lng], 3)
    markerInstances.get(markerData.id)?.openPopup()
  }
}

// Cleanup
onUnmounted(() => {
  markerInstances.forEach((leafletMarker) => leafletMarker.off())
  markerInstances.clear()
  if (map) {
    map.off()
    map.remove()
    map = null
  }
})
</script>

<template>
  <div class="w-full flex gap-4">
    <!-- Map Container -->
    <div class="flex-1">
      <div ref="mapContainer" class="w-full rounded-lg shadow-lg border-4 border-purple-900"
        :style="{ height: props.height }"></div>
      <div class="mt-2 text-sm text-gray-600">
        💡 Click vào map để thêm marker. Kéo marker để di chuyển vị trí.
      </div>
    </div>

    <!-- Sidebar -->
    <div class="w-80 bg-white rounded-lg shadow-lg p-4 overflow-y-auto" :style="{ maxHeight: props.height }">
      <div class="mb-4">
        <h3 class="text-lg font-bold text-purple-900 mb-2">Marker Editor</h3>
        <div class="flex gap-2">
          <button @click="copyJson" class="flex-1 bg-green-600 text-white px-3 py-2 rounded text-sm hover:bg-green-700"
            :disabled="markers.length === 0">
            📋 Copy JSON
          </button>
          <button @click="clearAll" class="flex-1 bg-red-600 text-white px-3 py-2 rounded text-sm hover:bg-red-700"
            :disabled="markers.length === 0">
            🗑️ Clear All
          </button>
        </div>
      </div>

      <!-- Markers List -->
      <div class="space-y-2">
        <div class="text-sm font-semibold text-gray-700 mb-2">
          Markers ({{ markers.length }})
        </div>

        <div v-if="markers.length === 0" class="text-sm text-gray-500 italic text-center py-8">
          Click vào map để thêm marker đầu tiên
        </div>

        <div v-for="marker in markers" :key="marker.id" class="bg-purple-50 border border-purple-200 rounded p-3">
          <!-- Editing Mode -->
          <div v-if="editingMarkerId === marker.id" class="space-y-2">
            <input v-model="editingName" type="text" class="w-full px-2 py-1 border border-purple-300 rounded text-sm"
              @keyup.enter="saveEdit(marker.id)" @keyup.esc="cancelEdit" autofocus />
            <div class="flex gap-1">
              <button @click="saveEdit(marker.id)"
                class="flex-1 bg-green-600 text-white px-2 py-1 rounded text-xs hover:bg-green-700">
                ✓ Save
              </button>
              <button @click="cancelEdit"
                class="flex-1 bg-gray-500 text-white px-2 py-1 rounded text-xs hover:bg-gray-600">
                ✗ Cancel
              </button>
            </div>
          </div>

          <!-- Display Mode -->
          <div v-else>
            <div class="font-semibold text-purple-900 text-sm mb-1">
              {{ marker.name }}
            </div>
            <div class="text-xs text-gray-600 font-mono mb-2">
              [{{ marker.lat.toFixed(4) }}, {{ marker.lng.toFixed(4) }}]
            </div>
            <div class="flex gap-1">
              <button @click="zoomToMarker(marker)"
                class="flex-1 bg-blue-500 text-white px-2 py-1 rounded text-xs hover:bg-blue-600">
                🔍 Zoom
              </button>
              <button @click="startEdit(marker.id, marker.name)"
                class="flex-1 bg-yellow-500 text-white px-2 py-1 rounded text-xs hover:bg-yellow-600">
                ✏️ Edit
              </button>
              <button @click="deleteMarker(marker.id)"
                class="flex-1 bg-red-500 text-white px-2 py-1 rounded text-xs hover:bg-red-600">
                🗑️ Delete
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- JSON Preview -->
      <div v-if="markers.length > 0" class="mt-4">
        <div class="text-sm font-semibold text-gray-700 mb-2">JSON Preview</div>
        <pre class="bg-gray-100 p-2 rounded text-xs overflow-x-auto">{{ markersJson }}</pre>
      </div>
    </div>
  </div>
</template>

<style scoped>
:deep(.leaflet-container) {
  background: #6B8ABA;
  font-family: 'Georgia', serif;
  cursor: crosshair;
}

:deep(.leaflet-popup-content-wrapper) {
  background: #f4e4c1;
  color: #2c1810;
  border: 2px solid #8b6914;
}

:deep(.leaflet-popup-tip) {
  background: #f4e4c1;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
