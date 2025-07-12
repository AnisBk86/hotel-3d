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
          <span class="text-lg font-bold" :class="getStatusTextColor(room.status)">
            {{ formatStatus(room.status) }}
          </span>
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
const props = defineProps<{
  room: Room
}>()

// Emits
const emit = defineEmits<{
  close: []
  markDirty: [roomId: string]
}>()

// Methods
function getStatusTextColor(status: string): string {
  const colors = {
    available: 'text-green-600',
    occupied: 'text-red-600', 
    maintenance: 'text-blue-500',
    cleaning: 'text-yellow-600'
  }
  return colors[status as keyof typeof colors] || 'text-gray-600'
}

// Methods
function formatStatus(status: string): string {
  const statusMap: Record<string, string> = {
    available: 'Available',
    occupied: 'Occupied',
    maintenance: 'Maintenance',
    cleaning: 'Cleaning'
  }
  return statusMap[status] || status
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
</style>
