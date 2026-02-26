<script setup lang="ts">
import { onMounted, onUnmounted, ref, reactive } from 'vue'
import L from 'leaflet'
import 'leaflet-rotate'
import type { Map as LeafletMap } from 'leaflet'

interface Props {
  height?: string
}

const props = withDefaults(defineProps<Props>(), {
  height: '600px'
})

const mapContainer = ref<HTMLElement | null>(null)
const initialBearing = -36.4
const bearing = ref(initialBearing)
let map: LeafletMap | null = null

// Layer visibility toggles
const layerVisibility = reactive<Record<string, boolean>>({
  yard_boundary: true,
  yard_block: true,
  yard_area: true,
  yard_road: true,
  building: true,
  landmarks: true,
})

const layerLabels: Record<string, string> = {
  yard_boundary: 'Ranh giới bãi',
  yard_block: 'Block container',
  yard_area: 'Khu vực chức năng',
  yard_road: 'Đường nội bộ',
  building: 'Tòa nhà',
  landmarks: 'Điểm mốc',
}

const featureCounts = reactive<Record<string, number>>({})
const layerInstances = new Map<string, L.GeoJSON>()

// --- Highlight state ---
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

// --- Landmark icon colors by type ---
const LANDMARK_COLORS: Record<string, string> = {
  'crane': '#e65100',
  'qc-crane': '#d50000',
  'customs-office': '#1565C0',
  '5f-office': '#1565C0',
  'parking': '#6A1B9A',
  'power': '#F9A825',
  'equipment': '#795548',
  'warehouse': '#2E7D32',
  'weightstation': '#455A64',
  'label': '#757575',
}

// --- Material Symbol per landmark type ---
const LANDMARK_SYMBOLS: Record<string, string> = {
  'customs-office': 'gavel',
  '5f-office':      'apartment',
  'parking':        'local_parking',
  'power':          'bolt',
  'equipment':      'precision_manufacturing',
  'warehouse':      'warehouse',
  'weightstation':  'scale',
  'label':          'location_on',
}

function makeLandmarkIcon(type: string): L.DivIcon {
  const color  = LANDMARK_COLORS[type] ?? '#757575'
  const symbol = LANDMARK_SYMBOLS[type] ?? 'place'
  return L.divIcon({
    html: `<div style="width:28px;height:28px;background:${color};border-radius:50%;
                       border:2px solid #fff;display:flex;align-items:center;
                       justify-content:center;box-shadow:0 1px 4px rgba(0,0,0,.45)">
             <span class="material-symbols-outlined"
                   style="font-size:15px;color:#fff;line-height:1;user-select:none">${symbol}</span>
           </div>`,
    className: '',
    iconSize:   [28, 28],
    iconAnchor: [14, 14],
    popupAnchor:[0, -16],
  })
}

const CRANE_TYPES = new Set(['crane', 'qc-crane'])
const CRANE_W = 18
const CRANE_H = Math.round(CRANE_W * 402 / 144) // giữ tỷ lệ SVG gốc 144×402

function makeCraneIcon(counterRotateDeg: number, scale = 1): L.DivIcon {
  const [width, height] = [CRANE_W * scale, CRANE_H * scale]
  return L.divIcon({
    html: `<img src="/icons/sts_crane_icon.svg" class="crane-icon"
               style="width:${width}px;height:${height}px;
                      transform:rotate(${counterRotateDeg}deg);
                      transform-origin:50% 100%;display:block">`,
    className: '',
    iconSize: [width, height],
    iconAnchor: [width / 2, height / 1.6],
    popupAnchor: [0, -height],
  })
}

// ---------------------------------------------------------------------------
// --- Style functions ---
function styleBoundary(): L.PathOptions {
  return { color: '#d32f2f', weight: 3, dashArray: '8 4', fillOpacity: 0.03 }
}

function styleBlock(feature?: GeoJSON.Feature): L.PathOptions {
  return {
    color: '#555',
    weight: 1.5,
    fillColor: feature?.properties?.fill_color ?? '#ffff00',
    fillOpacity: 0.5,
  }
}

function styleArea(): L.PathOptions {
  return { color: '#00897B', weight: 1, dashArray: '4 2', fillColor: '#B2DFDB', fillOpacity: 0.25 }
}

function styleRoad(): L.PathOptions {
  return { color: '#78909C', weight: 4, opacity: 0.7 }
}

function styleBuilding(): L.PathOptions {
  return { color: '#5D4037', weight: 1.5, fillColor: '#BCAAA4', fillOpacity: 0.6 }
}

// ---------------------------------------------------------------------------
// --- Popup builders ---
function popupBoundary(p: Record<string, unknown>): string {
  return `
    <div style="min-width:200px">
      <div style="font-weight:700;font-size:14px;margin-bottom:6px">${p.yard_name}</div>
      <div style="font-size:12px;color:#555">${p.operator ?? ''}</div>
      <hr style="margin:6px 0;border-color:#ddd">
      <table style="font-size:12px;width:100%">
        <tr><td style="color:#888">Trạng thái</td><td style="text-align:right"><b>${p.status}</b></td></tr>
        <tr><td style="color:#888">Diện tích</td><td style="text-align:right"><b>${Number(p.total_area_sqm).toLocaleString()} m²</b></td></tr>
        <tr><td style="color:#888">Sức chứa</td><td style="text-align:right"><b>${Number(p.capacity_teus).toLocaleString()} TEU</b></td></tr>
      </table>
    </div>`
}

function popupBlock(p: Record<string, unknown>): string {
  const rows = [
    ['Category', p.category],
    ['Max slots', p.max_slots],
    p.bay_count ? ['Bays', p.bay_count] : null,
    p.row_count ? ['Rows', p.row_count] : null,
    p.tier_count && Number(p.tier_count) > 1 ? ['Tiers', p.tier_count] : null,
    p.direction ? ['Direction', p.direction] : null,
    p.remark ? ['Ghi chú', p.remark] : null,
  ].filter(Boolean) as [string, unknown][]

  return `
    <div style="min-width:160px">
      <div style="font-weight:700;font-size:14px;margin-bottom:4px">${p.block_name}</div>
      <div style="font-size:11px;color:#888;margin-bottom:6px">ID: ${p.block_id}</div>
      <table style="font-size:12px;width:100%">
        ${rows.map(([k, v]) => `<tr><td style="color:#888;padding:1px 8px 1px 0">${k}</td><td style="text-align:right"><b>${v}</b></td></tr>`).join('')}
      </table>
    </div>`
}

function popupArea(p: Record<string, unknown>): string {
  return `
    <div style="min-width:150px">
      <div style="font-weight:700;font-size:13px;margin-bottom:4px">${p.name}</div>
      <div style="font-size:12px;color:#888">Loại: ${p.area_type ?? 'N/A'}</div>
      ${p.remark ? `<div style="font-size:12px;margin-top:4px">${p.remark}</div>` : ''}
    </div>`
}

function popupRoad(p: Record<string, unknown>): string {
  const info = [
    p.road_type ? `Loại: ${p.road_type}` : null,
    p.lane_count ? `Làn: ${p.lane_count}` : null,
  ].filter(Boolean).join(' | ')
  return `<div><b>${p.name ?? 'Road'}</b>${info ? `<div style="font-size:12px;color:#888;margin-top:2px">${info}</div>` : ''}</div>`
}

function popupBuilding(p: Record<string, unknown>): string {
  return `
    <div>
      <div style="font-weight:700;font-size:13px">${p.bld_name}</div>
      <div style="font-size:12px;color:#888">ID: ${p.bld_id}${p.bld_type ? ` | ${p.bld_type}` : ''}</div>
      ${p.remark ? `<div style="font-size:12px;margin-top:4px">${p.remark}</div>` : ''}
    </div>`
}

function popupLandmark(p: Record<string, unknown>): string {
  return `
    <div>
      <div style="font-weight:700;font-size:13px">${p.bld_name}</div>
      <div style="font-size:12px;color:#888">${p.landmark_type ?? ''}</div>
      ${p.remark ? `<div style="font-size:12px;margin-top:4px">${p.remark}</div>` : ''}
    </div>`
}

// ---------------------------------------------------------------------------
// --- Layer loading ---
async function loadGeoJSON(url: string): Promise<GeoJSON.FeatureCollection> {
  const res = await fetch(url)
  if (!res.ok) throw new Error(`Failed to load ${url}: HTTP ${res.status}`)
  return res.json()
}

async function loadLayer(
  name: string,
  url: string,
  options: L.GeoJSONOptions,
) {
  try {
    const data = await loadGeoJSON(url)
    featureCounts[name] = data.features.length

    // Wrap onEachFeature to add highlight on click
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
  const layer = layerInstances.get(name)
  if (!layer || !map) return
  if (layerVisibility[name]) {
    layer.addTo(map)
  } else {
    map.removeLayer(layer)
  }
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
  // Delta counter-rotation: accumulate small deltas to keep cranes upright on screen
  let delta = bearing.value - prevDeg
  if (delta > 180) delta -= 360   // handle wrap-around 355→5
  if (delta < -180) delta += 360
  updateCraneRotations(delta)
}
function rotateCW() { setBearing(bearing.value + 5) }
function rotateCCW() { setBearing(bearing.value - 5) }
function resetBearing() {
  setBearing(initialBearing)
  // Reset crane icons back to 0° (initial upright position)
  document.querySelectorAll<HTMLElement>('.crane-icon').forEach(el => {
    el.setAttribute('data-rot', '0')
    el.style.transform = 'rotate(0deg)'
  })
}

// ---------------------------------------------------------------------------
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

  // Load layers sequentially — bottom to top z-order
  await loadLayer('yard_boundary', `${base}/yard_boundary.geojson`, {
    style: styleBoundary,
    interactive: false, // polygon bao trùm toàn bãi — không nhận click để tránh chặn layer trên
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
      // Show la
      const blockId = f.properties?.block_id ?? ''
      if (blockId) {
        l.bindTooltip(blockId, {
          permanent: true,
          direction: 'center',
          className: 'block-label',
        })
      }
    },
  })
  await loadLayer('landmarks', `${base}/landmarks.geojson`, {
    pointToLayer(feature, latlng) {
      const lmType = feature.properties?.landmark_type ?? 'label'
      const iconScale = lmType === 'qc-crane' ? 1.8 : 1
      if (CRANE_TYPES.has(lmType)) {
        return L.marker(latlng, { icon: makeCraneIcon(0, iconScale) })
      }

      return L.marker(latlng, { icon: makeLandmarkIcon(lmType) })
    },
    onEachFeature: (f, l) => l.bindPopup(popupLandmark(f.properties ?? {})),
  })

  // Fit to boundary
  const boundaryLayer = layerInstances.get('yard_boundary')
  if (boundaryLayer && map) {
    const bounds = boundaryLayer.getBounds()
    map.fitBounds(bounds, { padding: [20, 20] })
    const fitZoom = map.getBoundsZoom(bounds, false, [20, 20])
    map.setMinZoom(fitZoom + 1)
    map.setMaxBounds(bounds.pad(0.3))
  }

  map.setBearing(initialBearing)
})

onUnmounted(() => {
  layerInstances.forEach(layer => layer.off())
  layerInstances.clear()
  if (map) {
    map.off()
    map.remove()
    map = null
  }
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
      <div class="text-sm text-gray-500 ml-auto">
        {{ totalFeatures() }} features
      </div>
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
</style>
