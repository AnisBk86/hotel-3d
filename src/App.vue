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
    <main class="flex-1 flex relative">
      <!-- 3D Floor View -->
      <div class="w-full bg-gray-100 relative overflow-hidden">
        <HotelFloorPlan 
          ref="floorPlanRef"
          :selectedRoom="selectedRoom"
          @selectRoom="handleRoomSelection"
          class="w-full h-full"
        />
        
        <!-- Room Selection Buttons -->
        <div class="absolute bottom-6 left-1/2 transform -translate-x-1/2 flex space-x-3">
          <!-- Test button -->
          <button 
            @click="testRoomSelection"
            class="px-6 py-3 rounded-lg text-white font-medium text-lg shadow-lg bg-purple-500 hover:bg-purple-600 transition-all transform hover:scale-105"
          >
            TEST PANEL
          </button>
          
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

      <!-- Room Details Panel - FORCÉ À DROITE DE L'ÉCRAN -->
      <Transition name="fade">
        <div v-if="selectedRoom" 
             :key="selectedRoom.id"
             class="room-details-panel room-update-animation" 
             style="position: fixed !important; top: 0 !important; right: 0 !important; width: 400px !important; height: 100vh !important; z-index: 1000 !important; background: white !important; box-shadow: -4px 0 15px rgba(0,0,0,0.1) !important; border-left: 1px solid #e5e7eb !important; overflow: hidden !important;">
          <RoomDetails 
            :room="selectedRoom"
            @close="selectedRoom = null"
            @markDirty="markRoomDirty"
            @updateStatus="updateRoomStatus"
          />
        </div>
      </Transition>
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
// Import component
import HotelFloorPlan from './components/HotelFloorPlan.vue'
import RoomDetails from './components/RoomDetails.vue'

// Référence au composant HotelFloorPlan
const floorPlanRef = ref()

// Types
interface Room {
  id: string
  number: string
  status: 'available' | 'occupied' | 'maintenance' | 'cleaning'
  type: 'standard' | 'deluxe' | 'suite' | 'presidential'
  price: number
  capacity: number
  amenities: string[]
  position: { x: number, z: number } // Position for 3D rooms
}

// State
const selectedRoom = ref<Room | null>(null)
const activeSection = ref('dashboard')
const showAddRoom = ref(false)

// Computed
const quickSelectRooms = computed(() => {
  // Get rooms from HotelFloorPlan component if available
  if (floorPlanRef.value && floorPlanRef.value.getAllRoomsData) {
    return floorPlanRef.value.getAllRoomsData().slice(0, 3)
  }
  return []
})

// Methods
function selectRoom(room: Room | null) {
  selectedRoom.value = room
}

function handleRoomSelection(room: Room | null) {
  selectedRoom.value = room
}

function testRoomSelection() {
  const testRoom: Room = {
    id: 'test-123',
    number: 'TEST',
    status: 'available',
    type: 'deluxe',
    price: 150,
    capacity: 2,
    amenities: ['wifi', 'tv', 'minibar', 'balcony'],
    position: { x: 0, z: 0 }
  }
  selectedRoom.value = testRoom
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
  // Update the 3D scene directly through the component reference
  if (floorPlanRef.value && floorPlanRef.value.updateRoomStatus) {
    floorPlanRef.value.updateRoomStatus(roomId, 'cleaning')
  }
  // Update selectedRoom if it's the same room
  if (selectedRoom.value?.id === roomId) {
    if (floorPlanRef.value && floorPlanRef.value.getRoomData) {
      const updatedRoom = floorPlanRef.value.getRoomData(roomId)
      if (updatedRoom) {
        selectedRoom.value = { ...updatedRoom }
      }
    }
  }
}

function updateRoomStatus(roomId: string, newStatus: string) {
  console.log('Updating room status:', roomId, 'to', newStatus) // Debug log
  
  // Update the 3D scene directly through the component reference
  if (floorPlanRef.value && floorPlanRef.value.updateRoomStatus) {
    floorPlanRef.value.updateRoomStatus(roomId, newStatus)
    
    // Update selectedRoom if it's the same room
    if (selectedRoom.value?.id === roomId) {
      // Get the updated room data from HotelFloorPlan
      if (floorPlanRef.value.getRoomData) {
        const updatedRoom = floorPlanRef.value.getRoomData(roomId)
        if (updatedRoom) {
          selectedRoom.value = { ...updatedRoom }
        }
      }
    }
  } else {
    console.error('FloorPlan reference not available or updateRoomStatus method missing')
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

/* Animation simple fade pour le panneau - reste fixé à droite */
.fade-enter-active {
  transition: opacity 0.3s ease-out;
}

.fade-leave-active {
  transition: opacity 0.2s ease-in;
}

.fade-enter-from {
  opacity: 0;
}

.fade-leave-to {
  opacity: 0;
}

/* Animation de mise à jour quand on change de chambre */
.room-update-animation {
  animation: roomUpdate 0.5s ease-out;
}

@keyframes roomUpdate {
  0% {
    transform: scale(0.95);
    box-shadow: 0 25px 50px -12px rgba(59, 130, 246, 0.5);
    border-left-color: #3b82f6;
  }
  50% {
    transform: scale(1.02);
    box-shadow: 0 25px 50px -12px rgba(59, 130, 246, 0.3);
    border-left-color: #1d4ed8;
  }
  100% {
    transform: scale(1);
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
    border-left-color: #e5e7eb;
  }
}

/* Responsive design pour le panneau RoomDetails - FORCÉ À DROITE */
.room-details-panel {
  position: fixed !important;
  top: 0 !important;
  right: 0 !important;
  width: 400px !important;
  height: 100vh !important;
  z-index: 1000 !important;
}

@media (max-width: 1200px) {
  .room-details-panel {
    width: 350px !important;
    right: 0 !important;
  }
}

@media (max-width: 768px) {
  .room-details-panel {
    width: 300px !important;
    right: 0 !important;
  }
}

@media (max-width: 480px) {
  .room-details-panel {
    width: 100vw !important;
    right: 0 !important;
  }
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
