<template>
  <div class="w-full h-full bg-gradient-to-br from-gray-50 to-gray-200 flex items-center justify-center relative overflow-hidden">
    <!-- 3D Isometric Floor Plan -->
    <svg 
      viewBox="0 0 1000 700" 
      class="max-w-6xl max-h-full"
      @click="handleBackgroundClick"
    >
      <!-- Defs for gradients, shadows and patterns -->
      <defs>
        <!-- Floor gradient -->
        <linearGradient id="floorGradient" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#f5f5f4;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#e7e5e4;stop-opacity:1" />
        </linearGradient>
        
        <!-- Wall gradients -->
        <linearGradient id="wallGradient" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#fafaf9;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#f5f5f4;stop-opacity:1" />
        </linearGradient>
        
        <!-- Bed gradient -->
        <linearGradient id="bedGradient" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#fef7ed;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#fed7aa;stop-opacity:1" />
        </linearGradient>
        
        <!-- Wood texture for doors -->
        <linearGradient id="woodGradient" x1="0%" y1="0%" x2="100%" y2="100%">
          <stop offset="0%" style="stop-color:#d4a574;stop-opacity:1" />
          <stop offset="100%" style="stop-color:#b8936d;stop-opacity:1" />
        </linearGradient>

        <!-- Enhanced shadow filter -->
        <filter id="roomShadow" x="-50%" y="-50%" width="200%" height="200%">
          <feDropShadow dx="4" dy="8" flood-color="rgba(0,0,0,0.2)" flood-opacity="0.6"/>
        </filter>
        
        <!-- Badge shadow -->
        <filter id="badgeShadow" x="-50%" y="-50%" width="200%" height="200%">
          <feDropShadow dx="2" dy="4" flood-color="rgba(0,0,0,0.3)" flood-opacity="0.8"/>
        </filter>      </defs>

      <!-- Large Floor Base - Isometric perspective -->
      <path 
        d="M 50 400 L 200 250 L 850 250 L 700 400 Z" 
        fill="url(#floorGradient)" 
        stroke="#d6d3d1" 
        stroke-width="2"
        filter="url(#roomShadow)"
        class="floor-base"
      />
      
      <!-- Building Back Wall -->
      <path 
        d="M 200 250 L 850 250 L 850 150 L 200 150 Z" 
        fill="url(#wallGradient)" 
        stroke="#a8a29e" 
        stroke-width="1"
        opacity="0.9"
      />
      
      <!-- Building Left Wall -->
      <path 
        d="M 50 400 L 200 250 L 200 150 L 50 300 Z" 
        fill="#f5f5f4" 
        stroke="#a8a29e" 
        stroke-width="1"
        opacity="0.8"
      />

      <!-- 3D Isometric Rooms -->
      <g class="rooms">
        <!-- Floor Indicator -->
        <text 
          x="50" 
          y="50" 
          fill="#57534e" 
          font-size="28" 
          font-weight="bold" 
          text-anchor="start"
          class="floor-indicator"        >
          Étage {{ selectedFloor }}
        </text>
        
        <!-- Render realistic rooms dynamically -->
        <g v-for="(room, index) in roomsToDisplay" :key="room.id" class="room-group" @click="selectRoom(room, $event)">
          <!-- Room Floor -->
          <path 
            :d="getRoomFloorPath(...getRoomPosition(index))" 
            fill="url(#floorGradient)"
            stroke="#d6d3d1" 
            stroke-width="1"
            class="room-floor cursor-pointer"
          />
          
          <!-- Room Walls -->
          <g class="room-walls">
            <!-- Back wall -->
            <path 
              :d="getRoomBackWallPath(...getRoomPosition(index))" 
              fill="url(#wallGradient)"
              stroke="#a8a29e" 
              stroke-width="1"
            />
            <!-- Left wall -->
            <path 
              :d="getRoomLeftWallPath(...getRoomPosition(index))" 
              fill="#f5f5f4"
              stroke="#a8a29e" 
              stroke-width="1"
            />
            <!-- Right wall -->
            <path 
              :d="getRoomRightWallPath(...getRoomPosition(index))" 
              fill="#e7e5e4"
              stroke="#a8a29e" 
              stroke-width="1"
            />
          </g>
          
          <!-- Room Furniture -->
          <g class="room-furniture">
            <!-- Bed -->
            <path 
              :d="getBedPath(...getRoomPosition(index))" 
              fill="url(#bedGradient)"
              stroke="#d97706" 
              stroke-width="1"
            />
            <!-- Pillows -->
            <circle 
              :cx="getBedPillowX(...getRoomPosition(index))" 
              :cy="getBedPillowY(...getRoomPosition(index))" 
              r="6" 
              fill="#fef3c7"
              stroke="#f59e0b"
            />
            <circle 
              :cx="getBedPillowX(...getRoomPosition(index)) + 12" 
              :cy="getBedPillowY(...getRoomPosition(index))" 
              r="6" 
              fill="#fef3c7"
              stroke="#f59e0b"
            />
            
            <!-- Door -->
            <path 
              :d="getDoorPath(...getRoomPosition(index))" 
              fill="url(#woodGradient)"
              stroke="#92400e" 
              stroke-width="1"
            />
            
            <!-- Window -->
            <rect 
              :x="getWindowX(...getRoomPosition(index))" 
              :y="getWindowY(...getRoomPosition(index))" 
              width="8" 
              height="16" 
              fill="#e0f2fe"
              stroke="#0369a1" 
              stroke-width="1"
            />
          </g>
          
          <!-- Room Number on Door -->
          <text 
            :x="getDoorNumberX(...getRoomPosition(index))" 
            :y="getDoorNumberY(...getRoomPosition(index))" 
            fill="#57534e" 
            font-size="12" 
            font-weight="bold" 
            text-anchor="middle"
            class="room-number"
          >
            {{ room.number }}
          </text>
        </g>
        
        <!-- Floating Price Badges -->
        <g class="price-badges">
          <g v-for="(room, index) in roomsToDisplay" :key="`badge-${room.id}`" class="price-badge">
            <!-- Badge Background -->
            <rect 
              :x="getBadgeX(...getRoomPosition(index)) - 25" 
              :y="getBadgeY(...getRoomPosition(index)) - 12" 
              width="50" 
              height="24" 
              :fill="getBadgeColor(room.status)"
              stroke="white" 
              stroke-width="1"
              rx="12"
              filter="url(#badgeShadow)"
              class="badge-bg transition-all duration-300"
            />
            
            <!-- Price Text -->
            <text 
              :x="getBadgeX(...getRoomPosition(index))" 
              :y="getBadgeY(...getRoomPosition(index)) - 4" 
              fill="white" 
              font-size="10" 
              font-weight="bold" 
              text-anchor="middle"
              class="price-text"
            >
              €{{ room.price }}
            </text>
            
            <!-- Status Text -->
            <text 
              :x="getBadgeX(...getRoomPosition(index))" 
              :y="getBadgeY(...getRoomPosition(index)) + 6" 
              fill="white" 
              font-size="8" 
              font-weight="normal" 
              text-anchor="middle"
              class="status-text"
            >              {{ getStatusText(room.status) }}
            </text>
          </g>
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

// Room positions layout (x, y, width, height) - Updated for larger, more realistic rooms
const roomPositions = [
  [220, 180, 120, 80], // Room 0 - top left
  [380, 180, 120, 80], // Room 1 - top center left
  [540, 180, 120, 80], // Room 2 - top center right
  [700, 180, 120, 80], // Room 3 - top right
  [220, 290, 120, 80], // Room 4 - middle left
  [380, 290, 120, 80], // Room 5 - middle center left
  [540, 290, 120, 80], // Room 6 - middle center right
  [700, 290, 120, 80], // Room 7 - middle right
  [220, 400, 120, 80], // Room 8 - bottom left
  [380, 400, 120, 80], // Room 9 - bottom center left
  [540, 400, 120, 80], // Room 10 - bottom center right
  [700, 400, 120, 80], // Room 11 - bottom right
]

function getRoomPosition(index: number): [number, number, number, number] {
  if (index < roomPositions.length) {
    return roomPositions[index] as [number, number, number, number]
  }
  // Default position if index out of bounds
  return [400, 400, 120, 80]
}

// Methods for realistic 3D room rendering
function getRoomFloorPath(x: number, y: number, width: number, height: number): string {
  // Isometric floor with perspective
  const depth = 25
  return `M ${x} ${y + height} L ${x + width} ${y + height} L ${x + width - depth} ${y + height - depth} L ${x - depth} ${y + height - depth} Z`
}

function getRoomBackWallPath(x: number, y: number, width: number, height: number): string {
  // Back wall of the room
  const depth = 25
  return `M ${x} ${y} L ${x + width} ${y} L ${x + width - depth} ${y - depth} L ${x - depth} ${y - depth} Z`
}

function getRoomLeftWallPath(x: number, y: number, width: number, height: number): string {
  // Left wall of the room
  const depth = 25
  return `M ${x} ${y} L ${x - depth} ${y - depth} L ${x - depth} ${y + height - depth} L ${x} ${y + height} Z`
}

function getRoomRightWallPath(x: number, y: number, width: number, height: number): string {
  // Right wall of the room
  const depth = 25
  return `M ${x + width} ${y} L ${x + width - depth} ${y - depth} L ${x + width - depth} ${y + height - depth} L ${x + width} ${y + height} Z`
}

function getBedPath(x: number, y: number, width: number, height: number): string {
  // Bed positioned in the room
  const bedX = x + width * 0.3
  const bedY = y + height * 0.4
  const bedWidth = width * 0.4
  const bedHeight = height * 0.35
  const depth = 8
  
  // Isometric bed
  return `M ${bedX} ${bedY + bedHeight} L ${bedX + bedWidth} ${bedY + bedHeight} L ${bedX + bedWidth - depth} ${bedY + bedHeight - depth} L ${bedX - depth} ${bedY + bedHeight - depth} Z`
}

function getBedPillowX(x: number, y: number, width: number, height: number): number {
  return x + width * 0.35
}

function getBedPillowY(x: number, y: number, width: number, height: number): number {
  return y + height * 0.5
}

function getDoorPath(x: number, y: number, width: number, height: number): string {
  // Door on the front wall
  const doorX = x + width * 0.1
  const doorY = y + height * 0.8
  const doorWidth = width * 0.15
  const doorHeight = height * 0.2
  
  return `M ${doorX} ${doorY} L ${doorX + doorWidth} ${doorY} L ${doorX + doorWidth} ${doorY + doorHeight} L ${doorX} ${doorY + doorHeight} Z`
}

function getDoorNumberX(x: number, y: number, width: number, height: number): number {
  return x + width * 0.175
}

function getDoorNumberY(x: number, y: number, width: number, height: number): number {
  return y + height * 0.9
}

function getWindowX(x: number, y: number, width: number, height: number): number {
  return x + width * 0.8
}

function getWindowY(x: number, y: number, width: number, height: number): number {
  return y + height * 0.3
}

function getBadgeX(x: number, y: number, width: number, height: number): number {
  return x + width * 0.5
}

function getBadgeY(x: number, y: number, width: number, height: number): number {
  return y - 30
}

function getBadgeColor(status: string): string {
  const colors = {
    available: '#10b981',    // Green
    occupied: '#ef4444',     // Red  
    maintenance: '#f59e0b',  // Orange
    cleaning: '#3b82f6'      // Blue
  }
  return colors[status as keyof typeof colors] || '#10b981'
}

function getStatusText(status: string): string {
  const texts = {
    available: 'Disponible',
    occupied: 'Occupée',
    maintenance: 'En nettoyage', 
    cleaning: 'Réservé'
  }
  return texts[status as keyof typeof texts] || 'Disponible'
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

function getRoomTextX(x: number, _y: number, width: number, _height: number): number {
  return x + width / 2
}

function getRoomTextY(_x: number, y: number, _width: number, height: number): number {
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
