<script setup lang="ts">
import { ref } from 'vue'
import LeafletMap from './components/LeafletMap.vue'
import CustomMap from './components/CustomMap.vue'

const center = ref<[number, number]>([21.0285, 105.8542])
const zoom = ref(13)
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100">
    <div class="container mx-auto px-4 py-8">
      <header class="mb-8 text-center">
        <h1 class="text-4xl font-bold text-gray-800 mb-2">
          Leaflet.js + Vue 3 Demo
        </h1>
        <p class="text-gray-600">
          Interactive map với Leaflet, Vue 3, TypeScript, Pinia & Tailwind 4
        </p>
      </header>

      <main class="max-w-6xl mx-auto space-y-8">
        <!-- OpenStreetMap Demo -->
        <div class="bg-white rounded-xl shadow-xl p-6">
          <div class="mb-4">
            <h2 class="text-2xl font-semibold text-gray-700 mb-2">
              Bản đồ Hà Nội
            </h2>
            <p class="text-gray-500 text-sm">
              Click vào markers để xem thông tin các địa điểm nổi tiếng
            </p>
          </div>

          <LeafletMap
            :center="center"
            :zoom="zoom"
            height="600px"
          />

          <div class="mt-6 grid grid-cols-1 md:grid-cols-3 gap-4">
            <div class="bg-blue-50 p-4 rounded-lg">
              <h3 class="font-semibold text-blue-900 mb-1">Markers</h3>
              <p class="text-sm text-blue-700">3 địa điểm du lịch nổi tiếng</p>
            </div>
            <div class="bg-green-50 p-4 rounded-lg">
              <h3 class="font-semibold text-green-900 mb-1">Tiles</h3>
              <p class="text-sm text-green-700">OpenStreetMap tiles</p>
            </div>
            <div class="bg-purple-50 p-4 rounded-lg">
              <h3 class="font-semibold text-purple-900 mb-1">Features</h3>
              <p class="text-sm text-purple-700">Circle overlay & popups</p>
            </div>
          </div>
        </div>

        <!-- Custom Fantasy Map -->
        <div class="bg-gradient-to-br from-amber-50 to-orange-50 rounded-xl shadow-xl p-6">
          <div class="mb-4">
            <h2 class="text-2xl font-semibold text-amber-900 mb-2 flex items-center gap-2">
              🗺️ Ragnasia Fantasy Map
            </h2>
            <p class="text-amber-700 text-sm">
              Custom map tiles từ ảnh 2048x2048, 4 zoom levels (0-3), 85 tiles
            </p>
          </div>

          <CustomMap height="400px" />

          <!-- <div class="mt-6 grid grid-cols-1 md:grid-cols-3 gap-4">
            <div class="bg-amber-100 p-4 rounded-lg border-2 border-amber-300">
              <h3 class="font-semibold text-amber-900 mb-1">Custom Tiles</h3>
              <p class="text-sm text-amber-800">Local tiles trong public/map/</p>
            </div>
            <div class="bg-orange-100 p-4 rounded-lg border-2 border-orange-300">
              <h3 class="font-semibold text-orange-900 mb-1">CRS.Simple</h3>
              <p class="text-sm text-orange-800">Pixel coordinates, không dùng lat/lng</p>
            </div>
            <div class="bg-red-100 p-4 rounded-lg border-2 border-red-300">
              <h3 class="font-semibold text-red-900 mb-1">Zoom 0-3</h3>
              <p class="text-sm text-red-800">85 tiles, 2048×2048px</p>
            </div>
          </div> -->
        </div>
      </main>
    </div>
  </div>
</template>
