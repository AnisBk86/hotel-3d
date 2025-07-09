<template>
  <div class="w-full h-full bg-gradient-to-br from-gray-50 to-gray-200 flex items-center justify-center relative overflow-hidden">
    <!-- 3D Isometric Floor Plan -->
    <svg 
      viewBox="0 0 1200 800" 
      class="max-w-7xl max-h-full"
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
        </filter>
      </defs>

      <!-- Large Floor Base - Isometric perspective -->
      <path 
        d="M 100 500 L 250 300 L 1000 300 L 850 500 Z" 
        fill="url(#floorGradient)" 
        stroke="#d6d3d1" 
        stroke-width="2"
        filter="url(#roomShadow)"
        class="floor-base"
      />
      
      <!-- Building Back Wall -->
      <path 
        d="M 250 300 L 1000 300 L 1000 180 L 250 180 Z" 
        fill="url(#wallGradient)" 
        stroke="#a8a29e" 
        stroke-width="1"
        opacity="0.9"
      />
      
      <!-- Building Left Wall -->
      <path 
        d="M 100 500 L 250 300 L 250 180 L 100 380 Z" 
        fill="#f5f5f4" 
        stroke="#a8a29e" 
        stroke-width="1"
        opacity="0.8"
      />

      <!-- 3D Isometric Rooms -->
      <g class="rooms">
        <!-- Floor Indicator -->
        <text 
          x="80" 
          y="80" 
          fill="#57534e" 
          font-size="32" 
          font-weight="bold" 
          text-anchor="start"
          class="floor-indicator"
        >
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
            class="room-floor cursor-pointer transition-all duration-300 hover:brightness-105"
            :class="{ 'ring-2 ring-blue-400': props.selectedRoom?.id === room.id }"
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
              r="8" 
              fill="#fef3c7"
              stroke="#f59e0b"
              stroke-width="1"
            />
            <circle 
              :cx="getBedPillowX(...getRoomPosition(index)) + 16" 
              :cy="getBedPillowY(...getRoomPosition(index))" 
              r="8" 
              fill="#fef3c7"
              stroke="#f59e0b"
              stroke-width="1"
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
              width="12" 
              height="20" 
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
            font-size="14" 
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
              :x="getBadgeX(...getRoomPosition(index)) - 35" 
              :y="getBadgeY(...getRoomPosition(index)) - 18" 
              width="70" 
              height="36" 
              :fill="getBadgeColor(room.status)"
              stroke="white" 
              stroke-width="2"
              rx="18"
              filter="url(#badgeShadow)"
              class="badge-bg transition-all duration-300 hover:scale-110"
            />
            
            <!-- Price Text -->
            <text 
              :x="getBadgeX(...getRoomPosition(index))" 
              :y="getBadgeY(...getRoomPosition(index)) - 6" 
              fill="white" 
              font-size="14" 
              font-weight="bold" 
              text-anchor="middle"
              class="price-text"
            >
              €{{ room.price }}
            </text>
            
            <!-- Status Text -->
            <text 
              :x="getBadgeX(...getRoomPosition(index))" 
              :y="getBadgeY(...getRoomPosition(index)) + 8" 
              fill="white" 
              font-size="10" 
              font-weight="normal" 
              text-anchor="middle"
              class="status-text"
            >
              {{ getStatusText(room.status) }}
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
  name: 'FloorMap3DRealistic'
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

// Sample rooms organized by floor with varied pricing
const allRooms = computed((): Room[] => {
  const floor = props.selectedFloor || 1
  const baseNumber = (floor - 1) * 100 + 1
  
  return [
    { id: `${baseNumber}`, number: `${baseNumber}`, status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
    { id: `${baseNumber + 1}`, number: `${baseNumber + 1}`, status: 'occupied', type: 'deluxe', price: 105, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor },
    { id: `${baseNumber + 2}`, number: `${baseNumber + 2}`, status: 'maintenance', type: 'suite', price: 140, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], floor },
    { id: `${baseNumber + 3}`, number: `${baseNumber + 3}`, status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
    { id: `${baseNumber + 4}`, number: `${baseNumber + 4}`, status: 'cleaning', type: 'deluxe', price: 105, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor },
    { id: `${baseNumber + 5}`, number: `${baseNumber + 5}`, status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
    { id: `${baseNumber + 6}`, number: `${baseNumber + 6}`, status: 'occupied', type: 'deluxe', price: 105, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor },
    { id: `${baseNumber + 7}`, number: `${baseNumber + 7}`, status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
    { id: `${baseNumber + 8}`, number: `${baseNumber + 8}`, status: 'occupied', type: 'suite', price: 140, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], floor },
    { id: `${baseNumber + 9}`, number: `${baseNumber + 9}`, status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor },
  ]
})

// Use allRooms computed property
const roomsToDisplay = computed(() => allRooms.value)

// Room positions layout (x, y, width, height) - Realistic isometric layout
const roomPositions = [
  [280, 220, 140, 100], // Room 0 - top left
  [480, 220, 140, 100], // Room 1 - top center left
  [680, 220, 140, 100], // Room 2 - top center right
  [280, 350, 140, 100], // Room 3 - middle left
  [480, 350, 140, 100], // Room 4 - middle center left
  [680, 350, 140, 100], // Room 5 - middle center right
  [280, 480, 140, 100], // Room 6 - bottom left
  [480, 480, 140, 100], // Room 7 - bottom center left
  [680, 480, 140, 100], // Room 8 - bottom center right
  [880, 350, 140, 100], // Room 9 - right side
]

function getRoomPosition(index: number): [number, number, number, number] {
  if (index < roomPositions.length) {
    return roomPositions[index] as [number, number, number, number]
  }
  // Default position if index out of bounds
  return [400, 400, 140, 100]
}

// Methods for realistic 3D room rendering
function getRoomFloorPath(x: number, y: number, width: number, height: number): string {
  // Isometric floor with perspective
  const depth = 30
  return `M ${x} ${y + height} L ${x + width} ${y + height} L ${x + width - depth} ${y + height - depth} L ${x - depth} ${y + height - depth} Z`
}

function getRoomBackWallPath(x: number, y: number, width: number, height: number): string {
  // Back wall of the room
  const depth = 30
  return `M ${x} ${y} L ${x + width} ${y} L ${x + width - depth} ${y - depth} L ${x - depth} ${y - depth} Z`
}

function getRoomLeftWallPath(x: number, y: number, width: number, height: number): string {
  // Left wall of the room
  const depth = 30
  return `M ${x} ${y} L ${x - depth} ${y - depth} L ${x - depth} ${y + height - depth} L ${x} ${y + height} Z`
}

function getRoomRightWallPath(x: number, y: number, width: number, height: number): string {
  // Right wall of the room
  const depth = 30
  return `M ${x + width} ${y} L ${x + width - depth} ${y - depth} L ${x + width - depth} ${y + height - depth} L ${x + width} ${y + height} Z`
}

function getBedPath(x: number, y: number, width: number, height: number): string {
  // Bed positioned in the room
  const bedX = x + width * 0.25
  const bedY = y + height * 0.35
  const bedWidth = width * 0.5
  const bedHeight = height * 0.4
  const depth = 12
  
  // Isometric bed
  return `M ${bedX} ${bedY + bedHeight} L ${bedX + bedWidth} ${bedY + bedHeight} L ${bedX + bedWidth - depth} ${bedY + bedHeight - depth} L ${bedX - depth} ${bedY + bedHeight - depth} Z`
}

function getBedPillowX(x: number, y: number, width: number, height: number): number {
  return x + width * 0.3
}

function getBedPillowY(x: number, y: number, width: number, height: number): number {
  return y + height * 0.45
}

function getDoorPath(x: number, y: number, width: number, height: number): string {
  // Door on the front wall
  const doorX = x + width * 0.05
  const doorY = y + height * 0.75
  const doorWidth = width * 0.2
  const doorHeight = height * 0.25
  
  return `M ${doorX} ${doorY} L ${doorX + doorWidth} ${doorY} L ${doorX + doorWidth} ${doorY + doorHeight} L ${doorX} ${doorY + doorHeight} Z`
}

function getDoorNumberX(x: number, y: number, width: number, height: number): number {
  return x + width * 0.15
}

function getDoorNumberY(x: number, y: number, width: number, height: number): number {
  return y + height * 0.87
}

function getWindowX(x: number, y: number, width: number, height: number): number {
  return x + width * 0.75
}

function getWindowY(x: number, y: number, width: number, height: number): number {
  return y + height * 0.25
}

function getBadgeX(x: number, y: number, width: number, height: number): number {
  return x + width * 0.5
}

function getBadgeY(x: number, y: number, width: number, height: number): number {
  return y - 40
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

function selectRoom(room: Room, event: MouseEvent) {
  event.stopPropagation()
  
  // Add visual feedback for clicked room
  const target = event.currentTarget as SVGElement
  if (target) {
    target.style.transform = 'scale(0.98)'
    setTimeout(() => {
      target.style.transform = 'scale(1)'
    }, 200)
  }
  
  emit('selectRoom', room)
}

function handleBackgroundClick() {
  // Deselect room when clicking background
  emit('clickOutside')
}
</script>

<style scoped>
.room-floor:hover {
  filter: brightness(1.1);
  cursor: pointer;
  transform: translateY(-2px);
  transition: all 0.3s ease-in-out;
}

.room-group:hover .badge-bg {
  transform: scale(1.1);
  filter: brightness(1.1) url(#badgeShadow);
}

.room-number {
  pointer-events: none;
  text-shadow: 1px 1px 2px rgba(0,0,0,0.3);
}

.price-text, .status-text {
  pointer-events: none;
  text-shadow: 1px 1px 2px rgba(0,0,0,0.5);
}

.floor-base {
  opacity: 0.95;
}

.floor-indicator {
  text-shadow: 2px 2px 4px rgba(0,0,0,0.2);
}

svg {
  filter: drop-shadow(0 15px 35px rgba(0,0,0,0.1));
}

.badge-bg {
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.room-furniture {
  pointer-events: none;
}

.room-walls {
  pointer-events: none;
}
</style>
