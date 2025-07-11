<template>
  <div class="w-full h-full bg-white shadow-xl flex flex-col">
    <!-- Header -->
    <div class="bg-gradient-to-r from-blue-600 to-blue-700 text-white p-6">
      <div class="flex justify-between items-center">
        <div>
          <h2 class="text-2xl font-bold">Room {{ room.number }}</h2>
          <div class="flex items-center space-x-2 mt-1">
            <span 
              :class="[
                'px-2 py-1 text-xs font-medium rounded-full',
                getStatusColor(room.status)
              ]"
            >
              {{ room.status.toUpperCase() }}
            </span>
            <span class="text-blue-100 text-sm">{{ room.type }}</span>
          </div>
        </div>
        <button 
          @click="$emit('close')"
          class="text-white hover:text-blue-200 transition-colors p-1"
        >
          <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
          </svg>
        </button>
      </div>
    </div>

    <!-- Content -->
    <div class="flex-1 p-6 space-y-4 overflow-y-auto">
      <!-- Basic Info -->
      <div class="grid grid-cols-2 gap-4">
        <div class="text-center p-4 bg-green-50 rounded-lg">
          <div class="text-2xl font-bold text-green-600">${{ room.price }}</div>
          <div class="text-sm text-gray-600">per night</div>
        </div>
        <div class="text-center p-4 bg-blue-50 rounded-lg">
          <div class="text-2xl font-bold text-blue-600">{{ room.capacity }}</div>
          <div class="text-sm text-gray-600">guests</div>
        </div>
      </div>

      <!-- Amenities -->
      <div>
        <h3 class="font-semibold text-gray-800 mb-2">Amenities</h3>
        <div class="flex flex-wrap gap-2">
          <span 
            v-for="amenity in room.amenities" 
            :key="amenity"
            class="px-2 py-1 bg-gray-100 text-gray-700 text-sm rounded"
          >
            {{ formatAmenity(amenity) }}
          </span>
        </div>
      </div>

      <!-- Actions -->
      <div v-if="room.status === 'occupied'" class="pt-4">
        <button 
          @click="$emit('markDirty', room.id)"
          class="w-full bg-orange-500 hover:bg-orange-600 text-white px-4 py-2 rounded-lg transition-colors"
        >
          Mark as Needs Cleaning
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Room {
  id: string
  number: string
  status: 'available' | 'occupied' | 'maintenance' | 'cleaning'
  type: 'standard' | 'deluxe' | 'suite' | 'presidential'
  price: number
  capacity: number
  amenities: string[]
}

// Props
defineProps<{
  room: Room
}>()

// Emits
defineEmits<{
  close: []
  markDirty: [roomId: string]
}>()

// Methods
function getStatusColor(status: string): string {
  const colors = {
    available: 'bg-green-100 text-green-800',
    occupied: 'bg-red-100 text-red-800',
    maintenance: 'bg-orange-100 text-orange-800',
    cleaning: 'bg-blue-100 text-blue-800'
  }
  return colors[status as keyof typeof colors] || 'bg-gray-100 text-gray-800'
}

function formatAmenity(amenity: string): string {
  const amenityMap: Record<string, string> = {
    wifi: 'Wi-Fi',
    tv: 'TV',
    minibar: 'Mini Bar',
    balcony: 'Balcony',
    ac: 'A/C',
    safe: 'Safe'
  }
  return amenityMap[amenity] || amenity.charAt(0).toUpperCase() + amenity.slice(1)
}
</script>

<style scoped>
.overflow-y-auto::-webkit-scrollbar {
  width: 4px;
}

.overflow-y-auto::-webkit-scrollbar-track {
  background: #f1f5f9;
}

.overflow-y-auto::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 2px;
}
</style>
