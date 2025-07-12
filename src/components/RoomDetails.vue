<template>
  <div class="w-full h-full bg-white shadow-xl flex flex-col">
    <!-- Header -->
    <div class="border-b border-gray-200 p-4">
      <div class="flex justify-between items-center">
        <h2 class="text-xl font-bold text-gray-900">Room Details</h2>
        <button 
          @click="$emit('close')"
          class="text-gray-400 hover:text-gray-600 transition-colors p-1"
        >
          <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
          </svg>
        </button>
      </div>
    </div>

    <!-- Content -->
    <div class="flex-1 p-4 space-y-4 overflow-y-auto">
      <!-- Room Image -->
      <div class="w-full h-32 bg-gradient-to-br from-amber-200 to-amber-400 rounded-lg flex items-center justify-center">
        <img 
          src="https://images.unsplash.com/photo-1590490360182-c33d57733427?w=400&h=200&fit=crop&crop=center" 
          alt="Room Image"
          class="w-full h-full object-cover rounded-lg"
          @error="handleImageError"
        />
      </div>

      <!-- Room Info Cards -->
      <div class="space-y-3">
        <div class="flex justify-between items-center">
          <span class="text-sm font-medium text-gray-600">Price</span>
          <span class="text-lg font-bold text-gray-900">€{{ room.price }}/night</span>
        </div>
        
        <div class="flex justify-between items-center">
          <span class="text-sm font-medium text-gray-600">Capacity</span>
          <span class="text-lg font-bold text-gray-900">{{ room.capacity }} guests</span>
        </div>
        
        <div class="flex justify-between items-center">
          <span class="text-sm font-medium text-gray-600">Status</span>
          <div class="relative">
            <select 
              v-model="selectedStatus"
              @change="updateRoomStatus"
              class="appearance-none bg-white border border-gray-300 rounded-lg px-3 py-2 pr-8 text-sm font-medium cursor-pointer hover:border-gray-400 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              :class="getStatusBorderColor(selectedStatus)"
            >
              <option 
                v-for="status in statusOptions" 
                :key="status.value" 
                :value="status.value"
                class="py-2"
              >
                {{ status.label }}
              </option>
            </select>
            <div class="absolute inset-y-0 right-0 flex items-center pr-2 pointer-events-none">
              <div 
                class="w-3 h-3 rounded-full mr-1" 
                :style="{ backgroundColor: getStatusColorCSS(selectedStatus) }"
              ></div>
              <svg class="w-4 h-4 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path>
              </svg>
            </div>
          </div>
        </div>
      </div>

      <!-- Action Button -->
      <div class="pt-2">
        <button 
          v-if="room.status === 'occupied'"
          @click="$emit('markDirty', room.id)"
          class="w-full bg-gray-200 hover:bg-gray-300 text-gray-800 px-4 py-3 rounded-lg transition-colors font-medium"
        >
          Mark as Dirty
        </button>
        <button 
          v-else
          class="w-full bg-blue-400 hover:bg-blue-500 text-white px-4 py-3 rounded-lg transition-colors font-medium"
        >
          Book Room
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue'

interface Room {
  id: string
  number: string
  status: 'available' | 'occupied' | 'maintenance' | 'cleaning'
  type: 'standard' | 'deluxe' | 'suite' | 'presidential'
  price: number
  capacity: number
  amenities: string[]
  position?: { x: number, z: number } // Optional position for 3D rooms
}

// Props
const { room } = defineProps<{
  room: Room
}>()

// Emits
const emit = defineEmits<{
  close: []
  markDirty: [roomId: string]
  updateStatus: [roomId: string, newStatus: string]
}>()

// Reactive state
const selectedStatus = ref(room.status)

// Status options with labels
const statusOptions = [
  { value: 'available', label: 'Available' },
  { value: 'occupied', label: 'Occupied' },
  { value: 'maintenance', label: 'Maintenance' },
  { value: 'cleaning', label: 'Cleaning' }
]

// Watch for room status changes to update selected status
watch(() => room.status, (newStatus) => {
  selectedStatus.value = newStatus
})

// Methods
function updateRoomStatus() {
  emit('updateStatus', room.id, selectedStatus.value)
}

function getStatusColorCSS(status: string): string {
  // Utilise les mêmes couleurs que Three.js (converties en CSS)
  const colors = {
    available: '#72b043',  // Vert naturel
    occupied: '#ff6361',   // Rouge corail
    maintenance: '#a2d2ff', // Bleu clair
    cleaning: '#f8cc1b'    // Jaune doré
  }
  return colors[status as keyof typeof colors] || '#6b7280'
}

function getStatusBorderColor(status: string): string {
  const borderColors = {
    available: 'border-green-400',
    occupied: 'border-red-400',
    maintenance: 'border-blue-400',
    cleaning: 'border-yellow-400'
  }
  return borderColors[status as keyof typeof borderColors] || 'border-gray-300'
}

function handleImageError(event: Event) {
  const target = event.target as HTMLImageElement
  target.style.display = 'none'
  const parent = target.parentElement
  if (parent) {
    parent.innerHTML = `
      <div class="w-full h-full bg-gradient-to-br from-amber-200 to-amber-400 rounded-lg flex items-center justify-center">
        <svg class="w-12 h-12 text-amber-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 7v10a2 2 0 002 2h14a2 2 0 002-2V9a2 2 0 00-2-2H5a2 2 0 00-2-2zM3 7l9 6 9-6"></path>
        </svg>
      </div>
    `
  }
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

/* Style personnalisé pour le select dropdown */
select {
  background-image: none;
  -webkit-appearance: none;
  -moz-appearance: none;
  appearance: none;
}

select:focus {
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

/* Animation pour les transitions de couleur */
select, .w-3.h-3.rounded-full {
  transition: all 0.2s ease-in-out;
}
</style>
