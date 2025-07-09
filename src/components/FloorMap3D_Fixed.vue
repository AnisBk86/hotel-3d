<template>
  <div class="w-full h-full bg-gray-100 flex items-center justify-center relative overflow-hidden">
    <!-- 3D Isometric Floor Plan -->
    <svg 
      viewBox="0 0 800 600" 
      class="max-w-4xl max-h-full"
      @click="handleBackgroundClick"
    >
      <!-- Defs for gradients and shadows -->
      <defs>
        <!-- Room gradients for different statuses -->
        <linearGradient id="availableGradient" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#10b981;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#059669;stop-opacity:1" />
        </linearGradient>
        
        <linearGradient id="occupiedGradient" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#ef4444;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#dc2626;stop-opacity:1" />
        </linearGradient>
        
        <linearGradient id="maintenanceGradient" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#f59e0b;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#d97706;stop-opacity:1" />
        </linearGradient>
        
        <linearGradient id="cleaningGradient" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#3b82f6;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#2563eb;stop-opacity:1" />
        </linearGradient>

        <!-- Shadow filter -->
        <filter id="roomShadow" x="-50%" y="-50%" width="200%" height="200%">
          <feDropShadow dx="3" dy="6" flood-color="rgba(0,0,0,0.3)" flood-opacity="0.5"/>
        </filter>
      </defs>

      <!-- Floor Base -->
      <path 
        d="M 100 200 L 350 100 L 650 100 L 400 200 Z" 
        fill="#f8f9fa" 
        stroke="#e5e7eb" 
        stroke-width="2"
        class="floor-base"
      />
      
      <!-- Back Wall -->
      <path 
        d="M 350 100 L 650 100 L 650 50 L 350 50 Z" 
        fill="#d1d5db" 
        stroke="#9ca3af" 
        stroke-width="1"
      />
      
      <!-- Left Wall -->
      <path 
        d="M 100 200 L 350 100 L 350 50 L 100 150 Z" 
        fill="#e5e7eb" 
        stroke="#9ca3af" 
        stroke-width="1"
      />

      <!-- 3D Isometric Rooms -->
      <g class="rooms">
        <!-- Floor Indicator -->
        <text 
          x="70" 
          y="40" 
          fill="#64748b" 
          font-size="24" 
          font-weight="bold" 
          text-anchor="middle"
          class="floor-indicator"
        >
          Floor {{ selectedFloor }}
        </text>
        
        <!-- Render rooms dynamically -->
        <g v-for="(room, index) in roomsToDisplay" :key="room.id" class="room" @click="selectRoom(room, $event)">
          <path 
            :d="getRoomPath(...getRoomPosition(index))" 
            :fill="getRoomGradient(room.status)"
            stroke="#fff" 
            stroke-width="2"
            filter="url(#roomShadow)"
            class="room-face transition-all duration-300 hover:brightness-110 cursor-pointer"
            :class="{ 'ring-4 ring-blue-400': props.selectedRoom?.id === room.id }"
          />
          <path 
            :d="getRoomTopPath(...getRoomPosition(index))" 
            :fill="getRoomTopColor(room.status)"
            stroke="#fff" 
            stroke-width="1"
            class="room-top"
          />
          <path 
            :d="getRoomSidePath(...getRoomPosition(index))" 
            :fill="getRoomSideColor(room.status)"
            stroke="#fff" 
            stroke-width="1"
            class="room-side"
          />
          <text 
            :x="getRoomTextX(...getRoomPosition(index).slice(0, 3))" 
            :y="getRoomTextY(...getRoomPosition(index).slice(0, 3))" 
            fill="white" 
            font-size="18" 
            font-weight="bold" 
            text-anchor="middle"
            class="room-number"
          >
            {{ room.number }}
          </text>
        </g>
      </g>
    </svg>
  </div>
</template>

<script setup lang="ts">
import { computed, withDefaults, defineProps, defineEmits, defineOptions } from 'vue'

// Define component options
defineOptions({
  name: 'FloorMap3D'
})

// Types
interface Room {
  id: string
  number: string
  status: 'available' | 'occupied' | 'maintenance' | 'cleaning'
  type: 'standard' | 'deluxe' | 'suite' | 'presidential'
  price: number
  capacity: number
  amenities: string[]
  floor: number
}

// Props
interface Props {
  selectedRoom?: Room | null
  selectedFloor?: number
  loadingRoom?: string | null
}

const props = withDefaults(defineProps<Props>(), {
  selectedRoom: null,
  selectedFloor: 1,
  loadingRoom: null
})

// Emits
const emit = defineEmits<{
  selectRoom: [room: Room | null]
  clickOutside: []
}>()

// Sample rooms organized by floor - these will represent the visual layout
const allRooms = computed((): Room[] => {
  const floor = props.selectedFloor || 1
  const baseNumber = (floor - 1) * 100 + 1
  
  return [
    { id: `${baseNumber}`, number: `${baseNumber}`, status: (floor === 1 ? 'available' : floor === 2 ? 'occupied' : 'available') as Room['status'], type: 'standard' as Room['type'], price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
    { id: `${baseNumber + 1}`, number: `${baseNumber + 1}`, status: (floor === 1 ? 'occupied' : floor === 2 ? 'available' : 'available') as Room['status'], type: 'deluxe' as Room['type'], price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor },
    { id: `${baseNumber + 2}`, number: `${baseNumber + 2}`, status: (floor === 1 ? 'maintenance' : floor === 2 ? 'occupied' : 'available') as Room['status'], type: 'suite' as Room['type'], price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], floor },
    { id: `${baseNumber + 3}`, number: `${baseNumber + 3}`, status: (floor === 1 ? 'available' : floor === 2 ? 'available' : 'available') as Room['status'], type: 'standard' as Room['type'], price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
    { id: `${baseNumber + 4}`, number: `${baseNumber + 4}`, status: (floor === 1 ? 'occupied' : floor === 2 ? 'occupied' : 'available') as Room['status'], type: 'deluxe' as Room['type'], price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor },
    { id: `${baseNumber + 5}`, number: `${baseNumber + 5}`, status: (floor === 1 ? 'cleaning' : floor === 2 ? 'occupied' : 'available') as Room['status'], type: 'standard' as Room['type'], price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
    { id: `${baseNumber + 6}`, number: `${baseNumber + 6}`, status: (floor === 1 ? 'available' : floor === 2 ? 'cleaning' : 'occupied') as Room['status'], type: 'deluxe' as Room['type'], price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor },
    { id: `${baseNumber + 7}`, number: `${baseNumber + 7}`, status: (floor === 1 ? 'available' : floor === 2 ? 'occupied' : 'available') as Room['status'], type: 'standard' as Room['type'], price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
    { id: `${baseNumber + 8}`, number: `${baseNumber + 8}`, status: (floor === 1 ? 'occupied' : floor === 2 ? 'available' : 'available') as Room['status'], type: 'suite' as Room['type'], price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], floor },
    { id: `${baseNumber + 9}`, number: `${baseNumber + 9}`, status: (floor === 1 ? 'available' : floor === 2 ? 'occupied' : 'maintenance') as Room['status'], type: 'standard' as Room['type'], price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
    { id: `${baseNumber + 10}`, number: `${baseNumber + 10}`, status: (floor === 1 ? 'available' : floor === 2 ? 'available' : 'available') as Room['status'], type: 'deluxe' as Room['type'], price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor },
    { id: `${baseNumber + 11}`, number: `${baseNumber + 11}`, status: (floor === 1 ? 'available' : floor === 2 ? 'available' : 'available') as Room['status'], type: 'standard' as Room['type'], price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
  ]
})

// Use allRooms computed property
const roomsToDisplay = computed(() => allRooms.value)

// Room positions layout (x, y, width, height)
const roomPositions = [
  [150, 120, 80, 60], // Room 0 - top left
  [280, 120, 80, 60], // Room 1 - top
  [410, 120, 80, 60], // Room 2 - top
  [540, 120, 80, 60], // Room 3 - top right
  [150, 220, 80, 60], // Room 4 - middle left
  [280, 220, 80, 60], // Room 5 - middle
  [410, 220, 80, 60], // Room 6 - middle
  [540, 220, 80, 60], // Room 7 - middle right
  [410, 320, 80, 60], // Room 8 - bottom center
  [540, 320, 80, 60], // Room 9 - bottom
  [670, 320, 80, 60], // Room 10 - bottom right
  [670, 220, 80, 60], // Room 11 - far right
]

function getRoomPosition(index: number): [number, number, number, number] {
  if (index < roomPositions.length) {
    return roomPositions[index] as [number, number, number, number]
  }
  // Default position if index out of bounds
  return [400, 400, 80, 60]
}

// Methods for 3D room rendering
function getRoomPath(x: number, y: number, width: number, height: number): string {
  // Front face of the isometric room
  return `M ${x} ${y} L ${x + width} ${y} L ${x + width} ${y + height} L ${x} ${y + height} Z`
}

function getRoomTopPath(x: number, y: number, width: number, _height: number): string {
  // Top face of the isometric room
  const depth = 30
  return `M ${x} ${y} L ${x + width} ${y} L ${x + width - depth} ${y - depth} L ${x - depth} ${y - depth} Z`
}

function getRoomSidePath(x: number, y: number, width: number, height: number): string {
  // Right side face of the isometric room
  const depth = 30
  return `M ${x + width} ${y} L ${x + width - depth} ${y - depth} L ${x + width - depth} ${y + height - depth} L ${x + width} ${y + height} Z`
}

function getRoomTextX(x: number, y: number, width: number): number {
  return x + width / 2
}

function getRoomTextY(x: number, y: number, width: number, height: number): number {
  return y + height / 2 + 6
}

function getRoomGradient(status: string): string {
  const gradients = {
    available: 'url(#availableGradient)',
    occupied: 'url(#occupiedGradient)', 
    maintenance: 'url(#maintenanceGradient)',
    cleaning: 'url(#cleaningGradient)'
  }
  return gradients[status as keyof typeof gradients] || 'url(#availableGradient)'
}

function getRoomTopColor(status: string): string {
  const colors = {
    available: '#34d399',
    occupied: '#f87171',
    maintenance: '#fbbf24',
    cleaning: '#60a5fa'
  }
  return colors[status as keyof typeof colors] || '#34d399'
}

function getRoomSideColor(status: string): string {
  const colors = {
    available: '#059669',
    occupied: '#dc2626',
    maintenance: '#d97706', 
    cleaning: '#2563eb'
  }
  return colors[status as keyof typeof colors] || '#059669'
}

function selectRoom(room: Room, event: MouseEvent) {
  event.stopPropagation()
  
  // Add visual feedback for clicked room
  const target = event.currentTarget as SVGElement
  if (target) {
    target.style.transform = 'scale(0.95)'
    setTimeout(() => {
      target.style.transform = 'scale(1)'
    }, 150)
  }
  
  emit('selectRoom', room)
}

function handleBackgroundClick() {
  // Deselect room when clicking background
  emit('clickOutside')
}
</script>

<style scoped>
.room-face:hover {
  filter: brightness(1.1) url(#roomShadow);
  cursor: pointer;
  transform: translateY(-2px);
  transition: all 0.2s ease-in-out;
}

.room-number {
  pointer-events: none;
  text-shadow: 1px 1px 2px rgba(0,0,0,0.5);
}

.floor-base {
  opacity: 0.9;
}

svg {
  filter: drop-shadow(0 10px 25px rgba(0,0,0,0.1));
}
</style>
