<script setup lang="ts">
import { onMounted, onUnmounted, ref, reactive } from 'vue'
import L from 'leaflet'
import 'leaflet-rotate'
import type { Map as LeafletMap } from 'leaflet'
import {
  CRANE_TYPES, makeCraneIcon, makeAnimatedCraneIcon, makeLandmarkIcon,
  styleBoundary, styleBlock, styleArea, styleRoad, styleBuilding,
  popupBlock, popupArea, popupRoad, popupBuilding, popupLandmark, 
  popupActiveCrane, tooltipActiveCrane,
} from '@/layers/dvp-layers'

interface Props {
  height?: string
}
const props = withDefaults(defineProps<Props>(), { height: '600px' })

const mapContainer = ref<HTMLElement | null>(null)
const initialBearing = -36.4
const bearing = ref(initialBearing)
let map: LeafletMap | null = null

const layerVisibility = reactive<Record<string, boolean>>({
  yard_boundary: true,
  yard_block: true,
  yard_area: true,
  yard_road: true,
  building: true,
  landmarks: true,
  active_cranes: true,
})

const layerLabels: Record<string, string> = {
  yard_boundary: 'Ranh giới bãi',
  yard_block:    'Block container',
  yard_area:     'Khu vực chức năng',
  yard_road:     'Đường nội bộ',
  building:      'Tòa nhà',
  landmarks:     'Điểm mốc',
  active_cranes: 'Xe cẩu đang hoạt động',
}

const featureCounts = reactive<Record<string, number>>({})
const layerInstances = new Map<string, L.GeoJSON>()

// --- Active Cranes Management ---
interface ActiveCrane {
  deviceId: string
  deviceName?: string
  device_name?: string
  lat: number
  lng: number
  speed?: number
  location?: string
  area?: string
  block?: string
  lastUpdate: string
  [key: string]: unknown
}

const activeCranes = reactive<Map<string, ActiveCrane>>(new Map())
const activeCraneMarkers = new Map<string, L.Marker>()

// --- Highlight ---
let highlightedLayer: L.Path | null = null
let highlightedParent: L.GeoJSON | null = null

function highlightFeature(parent: L.GeoJSON, layer: L.Path) {
  resetHighlight()
  layer.setStyle({ color: '#00e5ff', weight: 4 })
  layer.bringToFront()
  highlightedLayer = layer
  highlightedParent = parent
}

function resetHighlight() {
  if (highlightedLayer && highlightedParent) {
    highlightedParent.resetStyle(highlightedLayer)
  }
  highlightedLayer = null
  highlightedParent = null
}

// --- Layer loading ---
async function loadGeoJSON(url: string): Promise<GeoJSON.FeatureCollection> {
  const res = await fetch(url)
  if (!res.ok) throw new Error(`Failed to load ${url}: HTTP ${res.status}`)
  return res.json()
}

async function loadLayer(name: string, url: string, options: L.GeoJSONOptions) {
  try {
    const data = await loadGeoJSON(url)
    featureCounts[name] = data.features.length

    const origOnEach = options.onEachFeature
    options.onEachFeature = (feature, layer) => {
      origOnEach?.(feature, layer)
      layer.on('click', () => {
        if ('setStyle' in layer) highlightFeature(geoLayer, layer as L.Path)
      })
      layer.on('popupclose', () => resetHighlight())
    }

    const geoLayer = L.geoJSON(data, options)
    layerInstances.set(name, geoLayer)
    if (map && layerVisibility[name]) geoLayer.addTo(map)
  } catch (err) {
    console.error(`Error loading ${name}:`, err)
  }
}

function toggleLayer(name: string) {
  layerVisibility[name] = !layerVisibility[name]
  
  // Xử lý riêng cho active_cranes (không phải GeoJSON layer)
  if (name === 'active_cranes') {
    activeCraneMarkers.forEach(marker => {
      if (!map) return
      layerVisibility[name] ? marker.addTo(map) : map.removeLayer(marker)
    })
    return
  }
  
  // Xử lý các GeoJSON layers thông thường
  const layer = layerInstances.get(name)
  if (!layer || !map) return
  layerVisibility[name] ? layer.addTo(map) : map.removeLayer(layer)
}

// --- Active Cranes ---
function addOrUpdateActiveCrane(crane: ActiveCrane) {
  if (!map) return

  const existingMarker = activeCraneMarkers.get(crane.deviceId)
  
  if (existingMarker) {
    // Update existing marker position
    existingMarker.setLatLng([crane.lat, crane.lng])
    existingMarker.setPopupContent(popupActiveCrane(crane))
    existingMarker.setTooltipContent(tooltipActiveCrane(crane))
    activeCranes.set(crane.deviceId, crane)
  } else {
    // Create new marker with animated icon
    const iconScale = 1.8
    const marker = L.marker([crane.lat, crane.lng], {
      icon: makeAnimatedCraneIcon(0, iconScale),
      zIndexOffset: 1000, // Đảm bảo hiển thị trên các layer khác
    })
    
    // Bind popup (click để xem chi tiết)
    marker.bindPopup(popupActiveCrane(crane))
    
    // Bind tooltip (hover để xem thông tin nhanh)
    marker.bindTooltip(tooltipActiveCrane(crane), {
      permanent: false,
      direction: 'top',
      offset: [0, -10],
      className: 'active-crane-tooltip-wrapper',
      opacity: 0.95,
    })
    
    marker.addTo(map)
    
    activeCraneMarkers.set(crane.deviceId, marker)
    activeCranes.set(crane.deviceId, crane)
    featureCounts.active_cranes = activeCranes.size
  }
}

function removeActiveCrane(deviceId: string) {
  const marker = activeCraneMarkers.get(deviceId)
  if (marker && map) {
    map.removeLayer(marker)
    activeCraneMarkers.delete(deviceId)
    activeCranes.delete(deviceId)
    featureCounts.active_cranes = activeCranes.size
  }
}

function updateActiveCranePosition(deviceId: string, lat: number, lng: number, speed?: number) {
  const crane = activeCranes.get(deviceId)
  if (crane) {
    crane.lat = lat
    crane.lng = lng
    if (speed !== undefined) crane.speed = speed
    crane.lastUpdate = new Date().toISOString()
    addOrUpdateActiveCrane(crane)
  }
}

// Function để nhận dữ liệu từ server (WebSocket hoặc polling)
// Bạn có thể thay thế bằng WebSocket hoặc API call thực tế
function handleCraneDataFromServer(data: ActiveCrane | ActiveCrane[]) {
  const cranes = Array.isArray(data) ? data : [data]
  cranes.forEach(crane => {
    addOrUpdateActiveCrane(crane)
  })
}

// --- Rotation ---
function updateCraneRotations(delta: number) {
  document.querySelectorAll<HTMLElement>('.crane-icon').forEach(el => {
    const cur = parseFloat(el.getAttribute('data-rot') ?? '0')
    const next = cur + delta
    el.setAttribute('data-rot', String(next))
    el.style.transform = `rotate(${next}deg)`
  })
}

function setBearing(deg: number) {
  const prevDeg = bearing.value
  bearing.value = (deg + 360) % 360
  map?.setBearing(bearing.value)
  let delta = bearing.value - prevDeg
  if (delta > 180) delta -= 360
  if (delta < -180) delta += 360
  updateCraneRotations(delta)
}
function rotateCW()  { setBearing(bearing.value + 5) }
function rotateCCW() { setBearing(bearing.value - 5) }
function resetBearing() {
  setBearing(initialBearing)
  document.querySelectorAll<HTMLElement>('.crane-icon').forEach(el => {
    el.setAttribute('data-rot', '0')
    el.style.transform = 'rotate(0deg)'
  })
}

// --- Lifecycle ---
onMounted(async () => {
  if (!mapContainer.value) return

  map = L.map(mapContainer.value, {
    rotate: true,
    bearing: 0,
    touchRotate: true,
    rotateControl: false,
  })
  map.on('click', () => resetHighlight())

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    maxZoom: 19,
  }).addTo(map)

  const base = '/geojson/dvp'

  // Load layers bottom to top (z-order)
  await loadLayer('yard_boundary', `${base}/yard_boundary.geojson`, {
    style: styleBoundary,
    interactive: false, // bao trùm toàn bãi — không nhận click để tránh chặn layer trên
  })
  await loadLayer('yard_area', `${base}/yard_area.geojson`, {
    style: styleArea,
    onEachFeature: (f, l) => l.bindPopup(popupArea(f.properties ?? {})),
  })
  await loadLayer('yard_road', `${base}/yard_road.geojson`, {
    style: styleRoad,
    renderer: L.canvas({ tolerance: 10 }),
    onEachFeature: (f, l) => l.bindPopup(popupRoad(f.properties ?? {})),
  })
  await loadLayer('building', `${base}/building.geojson`, {
    style: styleBuilding,
    onEachFeature: (f, l) => l.bindPopup(popupBuilding(f.properties ?? {})),
  })
  await loadLayer('yard_block', `${base}/yard_block.geojson`, {
    style: styleBlock,
    onEachFeature: (f, l) => {
      l.bindPopup(popupBlock(f.properties ?? {}))
      const blockId = f.properties?.block_id ?? ''
      if (blockId) {
        l.bindTooltip(blockId, { permanent: true, direction: 'center', className: 'block-label' })
      }
    },
  })
  await loadLayer('landmarks', `${base}/landmarks.geojson`, {
    pointToLayer(feature, latlng) {
      const lmType = feature.properties?.landmark_type ?? 'label'
      const iconScale = lmType === 'qc-crane' ? 1.8 : 1
      return CRANE_TYPES.has(lmType)
        ? L.marker(latlng, { icon: makeCraneIcon(0, iconScale) })
        : L.marker(latlng, { icon: makeLandmarkIcon(lmType) })
    },
    onEachFeature: (f, l) => l.bindPopup(popupLandmark(f.properties ?? {})),
  })

  const boundaryLayer = layerInstances.get('yard_boundary')
  if (boundaryLayer && map) {
    const bounds = boundaryLayer.getBounds()
    map.fitBounds(bounds, { padding: [20, 20] })
    map.setMinZoom(map.getBoundsZoom(bounds, false, [20, 20]) + 1)
    map.setMaxBounds(bounds.pad(0.3))
  }

  map.setBearing(initialBearing)

  // Demo: Thêm một số xe cẩu đang hoạt động (mock data)
  // Bạn có thể xóa phần này và thay bằng dữ liệu thực từ server
  setTimeout(() => {
    const boundaryLayer = layerInstances.get('yard_boundary')
    if (boundaryLayer && map) {
      const bounds = boundaryLayer.getBounds()
      const center = bounds.getCenter()
      
      // Thêm 2 xe cẩu demo với đầy đủ thông tin
      handleCraneDataFromServer([
        {
          deviceId: 'CRANE-001',
          deviceName: 'Xe cẩu STS-01',
          lat: center.lat + 0.0005,
          lng: center.lng + 0.0003,
          speed: 5.2,
          location: 'Khu vực A - Block 12',
          area: 'Khu vực A',
          block: 'Block 12',
          lastUpdate: new Date().toISOString(),
        },
        {
          deviceId: 'CRANE-002',
          deviceName: 'Xe cẩu RTG-05',
          lat: center.lat - 0.0004,
          lng: center.lng - 0.0002,
          speed: 3.8,
          location: 'Khu vực B - Block 8',
          area: 'Khu vực B',
          block: 'Block 8',
          lastUpdate: new Date().toISOString(),
        },
      ])
    }
  }, 1000)
})

onUnmounted(() => {
  activeCraneMarkers.forEach(marker => {
    if (map) map.removeLayer(marker)
  })
  activeCraneMarkers.clear()
  activeCranes.clear()
  layerInstances.forEach(layer => layer.off())
  layerInstances.clear()
  if (map) { map.off(); map.remove(); map = null }
})

const totalFeatures = () => Object.values(featureCounts).reduce((a, b) => a + b, 0)
</script>

<template>
  <div class="w-full">
    <div ref="mapContainer" class="w-full rounded-lg shadow-lg border-2 border-blue-300"
      :style="{ height: props.height }"></div>

    <!-- Controls -->
    <div class="mt-3 flex flex-wrap items-center gap-x-6 gap-y-2">
      <!-- Layer toggles -->
      <div class="flex flex-wrap items-center gap-x-3 gap-y-1">
        <span class="text-sm font-medium text-gray-700">Layers:</span>
        <label v-for="(label, name) in layerLabels" :key="name"
          class="inline-flex items-center gap-1 text-sm cursor-pointer select-none"
          :class="layerVisibility[name] ? 'text-gray-800' : 'text-gray-400'">
          <input type="checkbox" :checked="layerVisibility[name]" @change="toggleLayer(name)"
            class="accent-blue-600 w-3.5 h-3.5" />
          {{ label }}
          <span v-if="featureCounts[name]" class="text-xs text-gray-400">({{ featureCounts[name] }})</span>
        </label>
      </div>

      <!-- Rotation -->
      <div class="flex items-center gap-2">
        <span class="text-sm font-medium text-gray-700">Xoay:</span>
        <button @click="rotateCCW"
          class="w-7 h-7 bg-gray-100 hover:bg-gray-200 border border-gray-300 rounded text-sm font-bold">↺</button>
        <input type="range" min="0" max="360" step="1" :value="bearing"
          @input="setBearing(+($event.target as HTMLInputElement).value)" class="w-24 accent-blue-600" />
        <button @click="rotateCW"
          class="w-7 h-7 bg-gray-100 hover:bg-gray-200 border border-gray-300 rounded text-sm font-bold">↻</button>
        <span class="text-sm font-mono text-blue-700 w-10">{{ bearing }}°</span>
        <button v-if="bearing !== initialBearing" @click="resetBearing"
          class="text-xs bg-red-100 hover:bg-red-200 text-red-700 px-2 py-1 rounded border border-red-300">Reset</button>
      </div>

      <!-- Info -->
      <div class="text-sm text-gray-500 ml-auto">{{ totalFeatures() }} features</div>
    </div>
  </div>
</template>

<style>
.block-label {
  background: none !important;
  border: none !important;
  box-shadow: none !important;
  font-size: 11px !important;
  font-weight: 700 !important;
  color: #333 !important;
  text-shadow: 0 0 3px #fff, 0 0 3px #fff, 0 0 3px #fff;
  white-space: nowrap;
}

/* Animation cho xe cẩu đang hoạt động */
.animated-crane-container {
  position: relative;
  display: inline-block;
}

.animated-crane-pulse {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: rgba(76, 175, 80, 0.4);
  border: 3px solid rgba(76, 175, 80, 0.8);
  animation: cranePulse 2s ease-out infinite;
  z-index: 1;
}

.animated-crane-icon {
  animation: craneBlink 1.5s ease-in-out infinite;
  filter: drop-shadow(0 0 4px rgba(76, 175, 80, 0.8));
}

@keyframes cranePulse {
  0% {
    transform: translate(-50%, -50%) scale(0.8);
    opacity: 1;
  }
  50% {
    transform: translate(-50%, -50%) scale(1.2);
    opacity: 0.6;
  }
  100% {
    transform: translate(-50%, -50%) scale(1.5);
    opacity: 0;
  }
}

@keyframes craneBlink {
  0%, 100% {
    opacity: 1;
    filter: drop-shadow(0 0 4px rgba(76, 175, 80, 0.8)) brightness(1);
  }
  50% {
    opacity: 0.7;
    filter: drop-shadow(0 0 8px rgba(76, 175, 80, 1)) brightness(1.2);
  }
}

/* Styling cho tooltip của active crane */
.active-crane-tooltip-wrapper {
  background: rgba(255, 255, 255, 0.98) !important;
  border: 2px solid #4CAF50 !important;
  border-radius: 8px !important;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.25) !important;
  padding: 8px 10px !important;
  font-size: 12px !important;
  max-width: 220px !important;
  pointer-events: none !important;
  transition: opacity 0.2s ease-in-out !important;
}

.active-crane-tooltip-wrapper::before {
  border-top-color: #4CAF50 !important;
}

.active-crane-tooltip {
  line-height: 1.4;
}

.active-crane-tooltip b {
  color: #1976D2;
  font-weight: 600;
}
</style>
