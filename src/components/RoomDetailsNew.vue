<template>
  <div class="h-full bg-white shadow-xl border-l border-gray-200 overflow-y-auto">
    <!-- Header -->
    <div class="p-4 border-b border-gray-200 flex items-center justify-between">
      <h2 class="text-xl font-bold text-gray-800">Room Details</h2>
      <button 
        @click="$emit('close')"
        class="p-2 hover:bg-gray-100 rounded-full transition-colors text-gray-500 hover:text-gray-700"
        title="Fermer les détails"
      >
        ✕
      </button>
    </div>

    <!-- Room Image -->
    <div class="p-4">
      <div class="aspect-video bg-gray-200 rounded-lg overflow-hidden">
        <img 
          src="https://images.unsplash.com/photo-1631049307264-da0ec9d70304?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=2070&q=80"
          alt="Hotel Room"
          class="w-full h-full object-cover"
        />
      </div>
    </div>

    <!-- Room Details - Compact Layout -->
    <div class="p-4 space-y-4">
      <!-- Price -->
      <div class="flex items-center justify-between">
        <span class="text-sm font-medium text-gray-600">Price</span>
        <span class="text-lg font-bold text-gray-900">€{{ room.price }}/ni</span>
      </div>

      <!-- Capacity -->
      <div class="flex items-center justify-between">
        <span class="text-sm font-medium text-gray-600">Capacity</span>
        <span class="text-lg font-semibold text-gray-900">{{ room.capacity }}</span>
      </div>

      <!-- Status -->
      <div class="flex items-center justify-between">
        <span class="text-sm font-medium text-gray-600">Status</span>
        <span 
          class="text-lg font-semibold capitalize"
          :class="getStatusTextColor(room.status)"
        >
          {{ getStatusText(room.status) }}
        </span>
      </div>

      <!-- Mark as Dirty Button -->
      <button 
        @click="markAsDirty"
        class="w-full bg-gray-200 hover:bg-gray-300 text-gray-800 font-medium py-2 px-4 rounded-lg transition-colors text-sm"
      >
        Mark as Dirty
      </button>

      <!-- Divider -->
      <hr class="border-gray-200 my-3">

      <!-- Bottom Section - More Compact -->
      <div class="space-y-3">
        <div class="flex items-center justify-between">
          <span class="text-xs text-gray-500">Price</span>
          <span class="text-sm font-medium text-gray-900">{{ room.price }} €</span>
        </div>

        <div class="flex items-center justify-between">
          <span class="text-xs text-gray-500">Capacity</span>
          <span class="text-sm font-medium text-gray-900">{{ room.capacity }}</span>
        </div>

        <div class="flex items-center justify-between">
          <span class="text-xs text-gray-500">Status</span>
          <span 
            class="text-sm font-medium capitalize"
            :class="getStatusTextColor(room.status)"
          >
            {{ getStatusText(room.status) }}
          </span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { defineProps, defineEmits, defineOptions } from 'vue'

// Define component options
defineOptions({
  name: 'RoomDetailsNew'
})

// Types
interface Room {
  id: string
  number: string
  x: number
  y: number
  width: number
  height: number
  status: 'available' | 'occupied' | 'maintenance' | 'cleaning'
  type: 'standard' | 'deluxe' | 'suite' | 'presidential'
  price: number
  capacity: number
  amenities: string[]
  floor: number
}

interface Props {
  room: Room
}

const props = defineProps<Props>()

// Emits
const emit = defineEmits<{
  close: []
  markDirty: [roomId: string]
}>()

// Methods
function getStatusText(status: string): string {
  const texts = {
    available: 'Available',
    occupied: 'Occupied',
    maintenance: 'Maintenance', 
    cleaning: 'Cleaning'
  }
  return texts[status as keyof typeof texts] || status
}

function getStatusTextColor(status: string): string {
  const colors = {
    available: 'text-green-600',
    occupied: 'text-red-600',
    maintenance: 'text-orange-600',
    cleaning: 'text-blue-600'
  }
  return colors[status as keyof typeof colors] || 'text-gray-600'
}

function markAsDirty() {
  emit('markDirty', props.room.id)
}
</script>

<style scoped>
/* Custom scrollbar */
div::-webkit-scrollbar {
  width: 6px;
}

div::-webkit-scrollbar-track {
  background: #f1f5f9;
}

div::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 3px;
}

div::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}
</style>
