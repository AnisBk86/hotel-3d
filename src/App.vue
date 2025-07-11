// hotel-3d-app/src/App.vue
<template>
  <div class="flex h-screen bg-gray-100 overflow-hidden">
    <!-- Sidebar -->
    <div class="w-64 bg-slate-700 text-white flex flex-col">
      <!-- Logo Header -->
      <div class="p-6 border-b border-slate-600">
        <div class="flex items-center space-x-3">
          <div class="w-8 h-8 bg-white rounded-lg flex items-center justify-center">
            <svg class="w-5 h-5 text-slate-700" fill="currentColor" viewBox="0 0 20 20">
              <path d="M10.707 2.293a1 1 0 00-1.414 0l-7 7a1 1 0 001.414 1.414L4 10.414V17a1 1 0 001 1h2a1 1 0 001-1v-2a1 1 0 011-1h2a1 1 0 011 1v2a1 1 0 001 1h2a1 1 0 001-1v-6.586l.293.293a1 1 0 001.414-1.414l-7-7z"></path>
            </svg>
          </div>
          <h1 class="text-xl font-bold">HOTEL</h1>
        </div>
      </div>

      <!-- Navigation -->
      <nav class="flex-1 p-4">
        <div class="space-y-2">
          <button 
            @click="activeSection = 'dashboard'"
            :class="['w-full text-left px-4 py-3 rounded-lg flex items-center space-x-3 transition-colors', 
                     activeSection === 'dashboard' ? 'bg-slate-600 text-white' : 'text-slate-300 hover:bg-slate-600 hover:text-white']"
          >
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2H6a2 2 0 01-2-2V6zM14 6a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2V6zM4 16a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2H6a2 2 0 01-2-2v-2zM14 16a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2v-2z"></path>
            </svg>
            <span>Dashboard</span>
          </button>
          
          <button 
            @click="activeSection = 'reservations'"
            :class="['w-full text-left px-4 py-3 rounded-lg flex items-center space-x-3 transition-colors', 
                     activeSection === 'reservations' ? 'bg-slate-600 text-white' : 'text-slate-300 hover:bg-slate-600 hover:text-white']"
          >
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"></path>
            </svg>
            <span>Reservations</span>
          </button>
          
          <button 
            @click="activeSection = 'guests'"
            :class="['w-full text-left px-4 py-3 rounded-lg flex items-center space-x-3 transition-colors', 
                     activeSection === 'guests' ? 'bg-slate-600 text-white' : 'text-slate-300 hover:bg-slate-600 hover:text-white']"
          >
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4.354a4 4 0 110 5.292M15 21H3v-1a6 6 0 0112 0v1zm0 0h6v-1a6 6 0 00-9-5.197m13.5-9a2.5 2.5 0 11-5 0 2.5 2.5 0 015 0z"></path>
            </svg>
            <span>Guests</span>
          </button>
        </div>
      </nav>

      <!-- Add Room Button -->
      <div class="p-4 border-t border-slate-600">
        <button 
          @click="showAddRoom = !showAddRoom"
          class="w-full bg-slate-600 hover:bg-slate-500 text-white px-4 py-3 rounded-lg flex items-center justify-center space-x-2 transition-colors"
        >
          <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6v6m0 0v6m0-6h6m-6 0H6"></path>
          </svg>
          <span>Add Room</span>
        </button>
      </div>
    </div>

    <!-- Main Content -->
    <main class="flex-1 flex">
      <!-- 3D Floor View -->
      <div class="flex-1 bg-gray-100 relative overflow-hidden">
        <HotelFloorPlan 
          :selectedRoom="selectedRoom"
          @selectRoom="handleRoomSelection"
          class="w-full h-full"
        />
        
        <!-- Room Selection Buttons -->
        <div class="absolute bottom-6 left-1/2 transform -translate-x-1/2 flex space-x-3">
          <button 
            v-for="room in quickSelectRooms" 
            :key="room.id"
            @click="selectRoom(room)"
            :class="['px-6 py-3 rounded-lg text-white font-medium text-lg shadow-lg transition-all transform hover:scale-105', 
                     getRoomButtonColor(room.status)]"
          >
            {{ room.number }}
          </button>
        </div>
      </div>

      <!-- Room Details Panel -->
      <Transition name="slide-left" mode="out-in">
        <RoomDetails 
          v-if="selectedRoom" 
          :room="selectedRoom" 
          @close="selectedRoom = null"
          @markDirty="markRoomDirty"
          class="bg-white shadow-xl"
          style="width: 30vw; min-width: 300px; max-width: 500px;"
        />
      </Transition>
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import HotelFloorPlan from './components/HotelFloorPlan.vue'
import RoomDetails from './components/RoomDetails.vue'

// Types
interface Room {
  id: string
  number: string
  status: 'available' | 'occupied' | 'maintenance' | 'cleaning'
  type: 'standard' | 'deluxe' | 'suite' | 'presidential'
  price: number
  capacity: number
  amenities: string[]
}

// State
const selectedRoom = ref<Room | null>(null)
const activeSection = ref('dashboard')
const showAddRoom = ref(false)

// Sample rooms data for quick selection
const allRooms = ref<Room[]>([
  { id: '10', number: '10', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'] },
  { id: '11', number: '11', status: 'occupied', type: 'deluxe', price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'] },
  { id: '12', number: '12', status: 'available', type: 'suite', price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'] },
  { id: '13', number: '13', status: 'occupied', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'] },
  { id: '14', number: '14', status: 'maintenance', type: 'deluxe', price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'] },
  { id: '15', number: '15', status: 'cleaning', type: 'suite', price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'] },
  { id: '16', number: '16', status: 'occupied', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'] },
  { id: '17', number: '17', status: 'occupied', type: 'deluxe', price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'] },
  { id: '18', number: '18', status: 'occupied', type: 'suite', price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'] },
  { id: '19', number: '19', status: 'cleaning', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'] },
])

// Computed
const quickSelectRooms = computed(() => {
  return allRooms.value.slice(0, 3) // Show first 3 rooms as quick select
})

// Methods
function selectRoom(room: Room | null) {
  selectedRoom.value = room
}

function handleRoomSelection(room: Room | null) {
  selectRoom(room)
}

function getRoomButtonColor(status: string): string {
  const colors = {
    available: 'bg-green-500 hover:bg-green-600',
    occupied: 'bg-red-500 hover:bg-red-600', 
    maintenance: 'bg-orange-500 hover:bg-orange-600',
    cleaning: 'bg-blue-500 hover:bg-blue-600'
  }
  return colors[status as keyof typeof colors] || 'bg-gray-500 hover:bg-gray-600'
}

function markRoomDirty(roomId: string) {
  const room = allRooms.value.find(r => r.id === roomId)
  if (room) {
    room.status = 'cleaning'
  }
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

#app {
  height: 100vh;
  overflow: hidden;
}

/* Enhanced transition animations */
.slide-left-enter-active,
.slide-left-leave-active {
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.slide-left-enter-from {
  transform: translateX(100%);
  opacity: 0;
}

.slide-left-leave-to {
  transform: translateX(100%);
  opacity: 0;
}

/* Global improvements */
* {
  scrollbar-width: thin;
  scrollbar-color: #6b7280 #374151;
}

*::-webkit-scrollbar {
  width: 8px;
}

*::-webkit-scrollbar-track {
  background: #374151;
  border-radius: 4px;
}

*::-webkit-scrollbar-thumb {
  background: #6b7280;
  border-radius: 4px;
}

*::-webkit-scrollbar-thumb:hover {
  background: #9ca3af;
}
</style>
