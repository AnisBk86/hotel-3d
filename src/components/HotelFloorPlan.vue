<template>
  <div class="w-full h-full relative overflow-hidden bg-gray-100">
    <!-- Three.js Container -->
    <div ref="threeContainer" class="w-full h-full"></div>
    
    <!-- UI Overlay -->
    <div class="absolute top-6 left-6 z-10">
      <h2 class="text-2xl font-bold text-gray-800 bg-white/90 backdrop-blur-sm px-4 py-2 rounded-lg shadow-lg">
        Hotel Floor Plan
      </h2>
    </div>
    
    <!-- Controls -->
    <div class="absolute bottom-6 right-6 z-10 space-y-2">
      <button 
        @click="zoomIn"
        class="bg-white/90 backdrop-blur-sm hover:bg-white text-gray-700 px-3 py-2 rounded-lg shadow-lg transition-all duration-300 text-xl font-bold"
        title="Zoom In"
      >
        +
      </button>
      <button 
        @click="zoomOut"
        class="bg-white/90 backdrop-blur-sm hover:bg-white text-gray-700 px-3 py-2 rounded-lg shad// Expose methods and data to parent component
defineExpose({
  updateRoomStatus,
  getRoomData: (roomId: string) => hotelRooms.find(room => room.id === roomId),
  getAllRoomsData: () => hotelRooms
})g transition-all duration-300 text-xl font-bold"
        title="Zoom Out"
      >
        −
      </button>
      <button 
        @click="resetCamera"
        class="bg-white/90 backdrop-blur-sm hover:bg-white text-gray-700 px-4 py-2 rounded-lg shadow-lg transition-all duration-300"
      >
        Reset View
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import * as THREE from 'three'

// Props and Emits
interface Room {
  id: string
  number: string
  status: 'available' | 'occupied' | 'maintenance' | 'cleaning'
  type: 'standard' | 'deluxe' | 'suite' | 'presidential'
  price: number
  capacity: number
  amenities: string[]
}

defineProps<{
  selectedRoom: Room | null
}>()

const emit = defineEmits<{
  selectRoom: [room: Room | null]
}>()

// Refs
const threeContainer = ref<HTMLDivElement>()

// Three.js variables
let scene: THREE.Scene
let camera: THREE.OrthographicCamera
let renderer: THREE.WebGLRenderer
let animationId: number | null = null
let raycaster: THREE.Raycaster
let mouse: THREE.Vector2
let roomMeshes: Map<string, THREE.Group> = new Map()
let hoveredRoom: THREE.Group | null = null
let selectedRoom: THREE.Group | null = null

// Animation variables
// let targetHoverScale = 1  // Temporairement commenté
// let currentHoverScale = 1  // Temporairement commenté
const hoverAnimationSpeed = 0.15 // Increased speed for more responsiveness
let roomAnimations: Map<string, { targetScale: number, currentScale: number }> = new Map()

// Label management
let roomLabels: Map<string, THREE.Mesh> = new Map()
let labelAnimations: Map<string, { targetOpacity: number, currentOpacity: number }> = new Map()
const labelAnimationSpeed = 0.12 // Speed for label fade in/out animations

// Zoom settings
let zoomLevel = 1
const minZoom = 0.5
const maxZoom = 3
const zoomStep = 0.2

// Hotel rooms data
const hotelRooms = [
  // Première série de chambres (101-110)
  { id: '101', number: '101', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], position: { x: 6, z: -3 } },
  { id: '102', number: '102', status: 'occupied', type: 'deluxe', price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: 6, z: 0 } },
  { id: '103', number: '103', status: 'available', type: 'suite', price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: 3, z: -6 } },
  { id: '104', number: '104', status: 'occupied', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], position: { x: 0, z: -6 } },
  { id: '105', number: '105', status: 'maintenance', type: 'deluxe', price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: -3, z: -6 } },
  { id: '106', number: '106', status: 'cleaning', type: 'suite', price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: -6, z: -3 } },
  { id: '107', number: '107', status: 'occupied', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], position: { x: -6, z: 0 } },
  { id: '108', number: '108', status: 'occupied', type: 'deluxe', price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: -6, z: 3 } },
  { id: '109', number: '109', status: 'occupied', type: 'suite', price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: -3, z: 6 } },
  { id: '110', number: '110', status: 'cleaning', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], position: { x: 0, z: 6 } },
  
  // Nouvelle série de chambres (111-112) - en parallèle avec les chambres 101 et 102
  { id: '111', number: '111', status: 'available', type: 'deluxe', price: 160, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: 10, z: -3 } },
  { id: '112', number: '112', status: 'occupied', type: 'suite', price: 220, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: 10, z: 0 } },
  
  // Nouvelles chambres adjacentes aux chambres 111 et 112
  { id: '125', number: '125', status: 'available', type: 'standard', price: 150, capacity: 2, amenities: ['wifi', 'tv'], position: { x: 10, z: -6 } }, // À côté de la chambre 111
  { id: '126', number: '126', status: 'cleaning', type: 'deluxe', price: 180, capacity: 3, amenities: ['wifi', 'tv', 'minibar'], position: { x: 10, z: 3 } }, // À côté de la chambre 112
  
  // Chambres 113-116 repositionnées en parallèle avec les chambres 109 et 110
  { id: '113', number: '113', status: 'maintenance', type: 'deluxe', price: 160, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: 3, z: 10 } },
  { id: '114', number: '114', status: 'cleaning', type: 'suite', price: 220, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: 0, z: 10 } },
  { id: '115', number: '115', status: 'available', type: 'presidential', price: 400, capacity: 6, amenities: ['wifi', 'tv', 'minibar', 'balcony', 'jacuzzi', 'butler'], position: { x: -3, z: 10 } },
  { id: '116', number: '116', status: 'occupied', type: 'deluxe', price: 160, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: -6, z: 10 } },
  
  // Chambres 117-119 repositionnées en parallèle avec les chambres 106, 107, 108 + 2 nouvelles chambres
  { id: '117', number: '117', status: 'available', type: 'standard', price: 130, capacity: 2, amenities: ['wifi', 'tv'], position: { x: -10, z: -3 } }, // En parallèle avec chambre 106
  { id: '118', number: '118', status: 'occupied', type: 'suite', price: 220, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: -10, z: 0 } }, // En parallèle avec chambre 107
  { id: '119', number: '119', status: 'cleaning', type: 'deluxe', price: 160, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: -10, z: 3 } }, // En parallèle avec chambre 108
  
  // Nouvelles chambres 127 et 128 pour compléter la ligne
  { id: '127', number: '127', status: 'maintenance', type: 'standard', price: 135, capacity: 2, amenities: ['wifi', 'tv'], position: { x: -10, z: -6 } }, // Nouvelle chambre au sud
  { id: '128', number: '128', status: 'available', type: 'deluxe', price: 175, capacity: 3, amenities: ['wifi', 'tv', 'minibar'], position: { x: -10, z: 6 } }, // Nouvelle chambre au nord
  
  // Nouvelles chambres (120-124) en parallèle aux chambres 103, 104, 105
  { id: '120', number: '120', status: 'available', type: 'deluxe', price: 170, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: 6, z: -10 } },
  { id: '121', number: '121', status: 'occupied', type: 'standard', price: 140, capacity: 2, amenities: ['wifi', 'tv'], position: { x: 3, z: -10 } },
  { id: '122', number: '122', status: 'cleaning', type: 'suite', price: 230, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: 0, z: -10 } },
  { id: '123', number: '123', status: 'available', type: 'deluxe', price: 170, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: -3, z: -10 } },
  { id: '124', number: '124', status: 'maintenance', type: 'standard', price: 140, capacity: 2, amenities: ['wifi', 'tv'], position: { x: -6, z: -10 } },
]

function resetCamera() {
  zoomLevel = 1
  updateCameraZoom()
  camera.position.set(35, 35, 35) // Augmenté pour voir la zone étendue vers l'ouest (x=-10)
  camera.lookAt(0, 0, 0) // Centré sur l'origine pour voir l'ensemble du plan
}

function zoomIn() {
  if (zoomLevel < maxZoom) {
    zoomLevel += zoomStep
    updateCameraZoom()
  }
}

function zoomOut() {
  if (zoomLevel > minZoom) {
    zoomLevel -= zoomStep
    updateCameraZoom()
  }
}

function updateCameraZoom() {
  if (!camera || !threeContainer.value) return
  const aspect = threeContainer.value.clientWidth / threeContainer.value.clientHeight
  const frustumSize = 50 / zoomLevel // Augmenté de 45 à 50 pour voir les nouvelles chambres à l'ouest (x=-10)
  camera.left = -frustumSize * aspect / 2
  camera.right = frustumSize * aspect / 2
  camera.top = frustumSize / 2
  camera.bottom = -frustumSize / 2
  camera.updateProjectionMatrix()
}

function getStatusColorHex(status: string): number {
  const colors = {
    available: 0x4ade80,  // Vert
    occupied: 0xef4444,   // Rouge
    maintenance: 0x60a5fa, // Bleu clair
    cleaning: 0xfbbf24    // Jaune beurre
  }
  return colors[status as keyof typeof colors] || 0x6b7280
}

function createHotelFloor() {
  // Sol commun en carrelage blanc cassé couvrant toute la zone
  const floorGeom = new THREE.PlaneGeometry(24, 24) // Taille étendue pour couvrir de x=-12 à x=12 et z=-12 à z=12
  
  // Création d'une texture de carrelage
  const canvas = document.createElement('canvas')
  canvas.width = 512
  canvas.height = 512
  const context = canvas.getContext('2d')!
  
  // Couleur de base blanc cassé
  const baseColor = '#f8f6f0'
  const groutColor = '#e8e6e0'
  
  // Remplir le fond
  context.fillStyle = baseColor
  context.fillRect(0, 0, 512, 512)
  
  // Dessiner les joints de carrelage
  const tileSize = 64 // Taille d'un carreau en pixels
  context.strokeStyle = groutColor
  context.lineWidth = 2
  
  // Lignes verticales
  for (let x = 0; x <= 512; x += tileSize) {
    context.beginPath()
    context.moveTo(x, 0)
    context.lineTo(x, 512)
    context.stroke()
  }
  
  // Lignes horizontales
  for (let y = 0; y <= 512; y += tileSize) {
    context.beginPath()
    context.moveTo(0, y)
    context.lineTo(512, y)
    context.stroke()
  }
  
  // Ajouter des variations subtiles dans les carreaux
  for (let x = 0; x < 8; x++) {
    for (let y = 0; y < 8; y++) {
      // Variation aléatoire très subtile de la couleur
      const variation = Math.random() * 0.02 - 0.01
      const tileX = x * tileSize + 1
      const tileY = y * tileSize + 1
      const tileW = tileSize - 2
      const tileH = tileSize - 2
      
      context.fillStyle = `hsl(60, 5%, ${92 + variation * 100}%)`
      context.fillRect(tileX, tileY, tileW, tileH)
    }
  }
  
  const floorTexture = new THREE.CanvasTexture(canvas)
  floorTexture.wrapS = THREE.RepeatWrapping
  floorTexture.wrapT = THREE.RepeatWrapping
  floorTexture.repeat.set(4, 4) // Répéter la texture pour couvrir la zone
  
  const floorMat = new THREE.MeshLambertMaterial({ 
    map: floorTexture,
    color: 0xffffff
  })
  
  const floor = new THREE.Mesh(floorGeom, floorMat)
  floor.rotation.x = -Math.PI / 2
  floor.position.set(0, -0.01, 0) // Légèrement en dessous des autres éléments
  scene.add(floor)

  // Central courtyard/lobby
  const courtyardGeom = new THREE.PlaneGeometry(8, 8)
  const courtyardMat = new THREE.MeshLambertMaterial({ color: 0xf0f0f0 })
  const courtyard = new THREE.Mesh(courtyardGeom, courtyardMat)
  courtyard.rotation.x = -Math.PI / 2
  courtyard.position.y = 0
  scene.add(courtyard)

  // Corridors
  const corridorMat = new THREE.MeshLambertMaterial({ color: 0xe5e5e5 })
  
  // Horizontal corridors
  const hCorridorGeom = new THREE.PlaneGeometry(14, 2)
  const hCorridor1 = new THREE.Mesh(hCorridorGeom, corridorMat)
  hCorridor1.rotation.x = -Math.PI / 2
  hCorridor1.position.set(0, 0.01, -4.5)
  scene.add(hCorridor1)
  
  const hCorridor2 = new THREE.Mesh(hCorridorGeom, corridorMat)
  hCorridor2.rotation.x = -Math.PI / 2
  hCorridor2.position.set(0, 0.01, 4.5)
  scene.add(hCorridor2)

  // Vertical corridors
  const vCorridorGeom = new THREE.PlaneGeometry(2, 14)
  const vCorridor1 = new THREE.Mesh(vCorridorGeom, corridorMat)
  vCorridor1.rotation.x = -Math.PI / 2
  vCorridor1.position.set(-4.5, 0.01, 0)
  scene.add(vCorridor1)
  
  const vCorridor2 = new THREE.Mesh(vCorridorGeom, corridorMat)
  vCorridor2.rotation.x = -Math.PI / 2
  vCorridor2.position.set(4.5, 0.01, 0)
  scene.add(vCorridor2)

  // Nouveau couloir reliant la chambre 11 (6,0) à la chambre 19 (0,6)
  // Ce couloir suit une diagonale de la chambre 11 vers la chambre 19
  const diagonalCorridorGeom = new THREE.PlaneGeometry(1.5, 9) // Largeur 1.5, longueur 9
  const diagonalCorridor = new THREE.Mesh(diagonalCorridorGeom, corridorMat)
  diagonalCorridor.rotation.x = -Math.PI / 2
  diagonalCorridor.rotation.z = Math.PI / 4 // Rotation de 45 degrés pour la diagonale
  diagonalCorridor.position.set(3, 0.01, 3) // Position centrale entre les deux chambres
  scene.add(diagonalCorridor)

  // Couloirs d'extension pour connecter aux nouvelles chambres (20-29)
  // Couloir horizontal élargi entre les chambres 10,11 et 20,21
  const eastCorridorGeom = new THREE.PlaneGeometry(8, 2.5) // Élargi de 6x1.5 à 8x2.5
  const eastCorridor = new THREE.Mesh(eastCorridorGeom, corridorMat)
  eastCorridor.rotation.x = -Math.PI / 2
  eastCorridor.position.set(8, 0.01, -1.5)
  scene.add(eastCorridor)

  // Couloir vertical pour connecter aux chambres du haut
  const eastVerticalCorridorGeom = new THREE.PlaneGeometry(2.5, 6)
  const eastVerticalCorridor = new THREE.Mesh(eastVerticalCorridorGeom, corridorMat)
  eastVerticalCorridor.rotation.x = -Math.PI / 2
  eastVerticalCorridor.position.set(8, 0.01, 3)
  scene.add(eastVerticalCorridor)

  // Couloirs supplémentaires pour les nouvelles chambres 34 et 35
  // Couloir vertical étendu pour couvrir les chambres 34, 20, 21, 35
  const eastExtendedCorridorGeom = new THREE.PlaneGeometry(2.5, 12) // Étendu pour couvrir de z=-6 à z=6
  const eastExtendedCorridor = new THREE.Mesh(eastExtendedCorridorGeom, corridorMat)
  eastExtendedCorridor.rotation.x = -Math.PI / 2
  eastExtendedCorridor.position.set(8, 0.01, -1.5) // Centré entre les chambres
  scene.add(eastExtendedCorridor)

  // Couloir horizontal entre les chambres 12,13,14 et les nouvelles chambres 29,30,31,32,33
  const southCorridorGeom = new THREE.PlaneGeometry(14, 2) // Largeur 14 pour couvrir toutes les chambres, hauteur 2
  const southCorridor = new THREE.Mesh(southCorridorGeom, corridorMat)
  southCorridor.rotation.x = -Math.PI / 2
  southCorridor.position.set(0, 0.01, -8) // Positionné entre z=-6 et z=-10
  scene.add(southCorridor)

  // Couloir horizontal entre les chambres 18,19 et les nouvelles chambres 22,23,24,25
  const northCorridorGeom = new THREE.PlaneGeometry(12, 2) // Largeur 12 pour couvrir les chambres, hauteur 2
  const northCorridor = new THREE.Mesh(northCorridorGeom, corridorMat)
  northCorridor.rotation.x = -Math.PI / 2
  northCorridor.position.set(-1.5, 0.01, 8) // Positionné entre z=6 et z=10, centré sur les chambres
  scene.add(northCorridor)

  // Couloir horizontal entre les chambres 15,16,17 et les nouvelles chambres 26,27,28,36,37
  const westCorridorGeom = new THREE.PlaneGeometry(8, 2.5) // Élargi pour un meilleur passage
  const westCorridor = new THREE.Mesh(westCorridorGeom, corridorMat)
  westCorridor.rotation.x = -Math.PI / 2
  westCorridor.position.set(-8, 0.01, 0) // Positionné entre x=-6 et x=-10, centré verticalement
  scene.add(westCorridor)

  // Couloir vertical pour connecter les nouvelles chambres ouest (36, 26, 27, 28, 37)
  const westVerticalCorridorGeom = new THREE.PlaneGeometry(2.5, 14) // Couvre de z=-6 à z=6
  const westVerticalCorridor = new THREE.Mesh(westVerticalCorridorGeom, corridorMat)
  westVerticalCorridor.rotation.x = -Math.PI / 2
  westVerticalCorridor.position.set(-8, 0.01, 0) // Même position que le couloir horizontal principal
  scene.add(westVerticalCorridor)

  // Create rooms
  hotelRooms.forEach(room => {
    createRoom(room)
  })
}

function createRoom(roomData: any) {
  const roomSize = 2.5
  const wallHeight = 2

  // Create a group for the entire room
  const roomGroup = new THREE.Group()
  roomGroup.userData = { roomId: roomData.id, roomData }
  roomGroup.position.set(roomData.position.x, 0, roomData.position.z)
  roomGroup.name = `room-${roomData.id}`

  // Room base (floor) - Make it larger for better click detection
  const roomGeom = new THREE.BoxGeometry(roomSize * 1.1, 0.1, roomSize * 1.1)
  const roomMat = new THREE.MeshLambertMaterial({ color: getStatusColorHex(roomData.status) })
  const roomMesh = new THREE.Mesh(roomGeom, roomMat)
  roomMesh.position.set(0, 0.05, 0)
  roomMesh.userData = { roomId: roomData.id, roomData }
  roomMesh.name = `room-floor-${roomData.id}`
  roomGroup.add(roomMesh)

  // Walls with 70% opacity and status color
  const wallColor = getStatusColorHex(roomData.status)
  const wallMat = new THREE.MeshLambertMaterial({ 
    color: wallColor,
    transparent: true,
    opacity: 0.7
  })
  
  // Back wall
  const backWallGeom = new THREE.BoxGeometry(roomSize, wallHeight, 0.1)
  const backWall = new THREE.Mesh(backWallGeom, wallMat)
  backWall.position.set(0, wallHeight / 2, -roomSize / 2)
  backWall.userData = { roomId: roomData.id, roomData }
  backWall.name = `room-wall-back-${roomData.id}`
  roomGroup.add(backWall)

  // Side walls
  const sideWallGeom = new THREE.BoxGeometry(0.1, wallHeight, roomSize)
  const leftWall = new THREE.Mesh(sideWallGeom, wallMat)
  leftWall.position.set(-roomSize / 2, wallHeight / 2, 0)
  leftWall.userData = { roomId: roomData.id, roomData }
  leftWall.name = `room-wall-left-${roomData.id}`
  roomGroup.add(leftWall)

  const rightWall = new THREE.Mesh(sideWallGeom, wallMat)
  rightWall.position.set(roomSize / 2, wallHeight / 2, 0)
  rightWall.userData = { roomId: roomData.id, roomData }
  rightWall.name = `room-wall-right-${roomData.id}`
  roomGroup.add(rightWall)

  // Add room furniture to make more area clickable
  const furnitureMat = new THREE.MeshLambertMaterial({ color: 0x8b4513 })
  
  // Bed
  const bedGeom = new THREE.BoxGeometry(1.5, 0.3, 1)
  const bed = new THREE.Mesh(bedGeom, furnitureMat)
  bed.position.set(0, 0.35, -0.5)
  bed.userData = { roomId: roomData.id, roomData }
  bed.name = `room-bed-${roomData.id}`
  roomGroup.add(bed)

  // Nightstand
  const nightstandGeom = new THREE.BoxGeometry(0.4, 0.4, 0.4)
  const nightstand = new THREE.Mesh(nightstandGeom, furnitureMat)
  nightstand.position.set(1, 0.4, -0.5)
  nightstand.userData = { roomId: roomData.id, roomData }
  nightstand.name = `room-nightstand-${roomData.id}`
  roomGroup.add(nightstand)

  // Large invisible clickable area covering the entire room space and beyond
  const clickAreaGeom = new THREE.BoxGeometry(roomSize * 1.5, wallHeight * 1.5, roomSize * 1.5)
  const clickAreaMat = new THREE.MeshBasicMaterial({ 
    transparent: true, 
    opacity: 0,
    visible: false 
  })
  const clickArea = new THREE.Mesh(clickAreaGeom, clickAreaMat)
  clickArea.position.set(0, wallHeight / 2, 0)
  clickArea.userData = { roomId: roomData.id, roomData }
  clickArea.name = `room-clickarea-${roomData.id}`
  roomGroup.add(clickArea)

  // Add userData to the room group for animation tracking
  roomGroup.userData = { roomId: roomData.id, roomData }

  scene.add(roomGroup)
  roomMeshes.set(roomData.id, roomGroup)

  // Room number label
  createRoomLabel(roomData)
}

function createRoomLabel(roomData: any) {
  const canvas = document.createElement('canvas')
  const context = canvas.getContext('2d')!
  canvas.width = 200
  canvas.height = 160  // Augmenté pour accommoder le texte du statut
  
  // Polyfill for roundRect if not available
  if (!context.roundRect) {
    (context as any).roundRect = function(x: number, y: number, width: number, height: number, radius: number) {
      this.beginPath()
      this.moveTo(x + radius, y)
      this.lineTo(x + width - radius, y)
      this.quadraticCurveTo(x + width, y, x + width, y + radius)
      this.lineTo(x + width, y + height - radius)
      this.quadraticCurveTo(x + width, y + height, x + width - radius, y + height)
      this.lineTo(x + radius, y + height)
      this.quadraticCurveTo(x, y + height, x, y + height - radius)
      this.lineTo(x, y + radius)
      this.quadraticCurveTo(x, y, x + radius, y)
    }
  }
  
  // Effacer le canvas
  context.clearRect(0, 0, 200, 160)
  
  // Dimensions du rectangle (plus grand)
  const centerX = 100
  const centerY = 80
  const rectWidth = 180
  const rectHeight = 140  // Augmenté pour le texte du statut
  const cornerRadius = 8
  
  // Ombre portée plus prononcée
  context.beginPath()
  ;(context as any).roundRect(centerX - rectWidth/2 + 3, centerY - rectHeight/2 + 3, rectWidth, rectHeight, cornerRadius)
  context.fillStyle = 'rgba(0, 0, 0, 0.4)'
  context.fill()
  
  // Fond du rectangle avec gradient linéaire plus moderne et opacité réduite à 70%
  const gradient = context.createLinearGradient(centerX - rectWidth/2, centerY - rectHeight/2, centerX + rectWidth/2, centerY + rectHeight/2)
  gradient.addColorStop(0, 'rgba(255, 255, 255, 0.7)')
  gradient.addColorStop(0.5, 'rgba(248, 250, 252, 0.7)')
  gradient.addColorStop(1, 'rgba(226, 232, 240, 0.7)')
  
  context.beginPath()
  ;(context as any).roundRect(centerX - rectWidth/2, centerY - rectHeight/2, rectWidth, rectHeight, cornerRadius)
  context.fillStyle = gradient
  context.fill()
  
  // Bordure extérieure avec couleur selon le statut de la chambre
  const statusColor = getStatusColor(roomData.status)
  context.beginPath()
  ;(context as any).roundRect(centerX - rectWidth/2, centerY - rectHeight/2, rectWidth, rectHeight, cornerRadius)
  context.strokeStyle = statusColor
  context.lineWidth = 6
  context.stroke()
  
  // Bordure intérieure pour plus de définition (plus sombre)
  context.beginPath()
  ;(context as any).roundRect(centerX - rectWidth/2 + 3, centerY - rectHeight/2 + 3, rectWidth - 6, rectHeight - 6, cornerRadius - 3)
  context.strokeStyle = '#475569'
  context.lineWidth = 2
  context.stroke()
  
  // Texte du numéro avec police Orbitron et ombre portée
  // Ombre du texte plus légère (blanche) pour contraster avec le texte noir
  context.fillStyle = 'rgba(255, 255, 255, 0.4)'
  context.font = 'bold 48px Orbitron, Arial, sans-serif'  // Taille réduite pour faire place au statut
  context.textAlign = 'center'
  context.textBaseline = 'middle'
  context.fillText(roomData.number, centerX + 2, centerY - 25 + 2)  // Décalé vers le haut
  
  // Texte principal du numéro avec couleur unifiée (noir pour plus de clarté)
  context.fillStyle = '#000000'
  context.font = 'bold 48px Orbitron, Arial, sans-serif'
  context.fillText(roomData.number, centerX, centerY - 25)  // Décalé vers le haut
  
  // Ajouter le statut de disponibilité sous le numéro
  const statusText = getStatusText(roomData.status)
  
  // Ombre du texte du statut
  context.fillStyle = 'rgba(255, 255, 255, 0.3)'
  context.font = 'bold 24px Orbitron, Arial, sans-serif'
  context.fillText(statusText, centerX + 1, centerY + 25 + 1)  // Décalé vers le bas
  
  // Texte du statut principal avec couleur unifiée (noir pour plus de clarté)
  context.fillStyle = '#000000'
  context.font = 'bold 24px Orbitron, Arial, sans-serif'
  context.fillText(statusText, centerX, centerY + 25)  // Décalé vers le bas
  
  const texture = new THREE.CanvasTexture(canvas)
  const labelMat = new THREE.MeshBasicMaterial({ 
    map: texture, 
    transparent: true,
    opacity: 0 // Start invisible, will animate to 1.0 on hover
  })
  const labelGeom = new THREE.PlaneGeometry(2.0, 1.2)
  const label = new THREE.Mesh(labelGeom, labelMat)
  
  // Positionner le label parfaitement centré au milieu de chaque chambre
  const offsetX = 0  // Centré horizontalement
  const offsetY = 2.8 // Hauteur constante au-dessus de la chambre
  const offsetZ = 0   // Centré en profondeur
  
  label.position.set(
    roomData.position.x + offsetX, 
    offsetY, 
    roomData.position.z + offsetZ
  )
  
  label.lookAt(camera.position)
  label.userData = { roomId: roomData.id, roomData }
  scene.add(label)
  
  // Store the label for animation management
  roomLabels.set(roomData.id, label)
  
  // Initialize label animation state
  labelAnimations.set(roomData.id, { targetOpacity: 0, currentOpacity: 0 })
}

function setupLighting() {
  // Ambient light
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
  scene.add(ambientLight)
  
  // Directional light
  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8)
  dirLight.position.set(10, 20, 10)
  dirLight.castShadow = true
  scene.add(dirLight)
}

function handleClick(event: MouseEvent) {
  if (!threeContainer.value) return

  const rect = threeContainer.value.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(mouse, camera)
  
  // Obtenir TOUS les objets de la scène pour le raycasting
  const allObjects: THREE.Object3D[] = []
  scene.traverse((child) => {
    if (child instanceof THREE.Mesh) {
      allObjects.push(child)
    }
  })
  
  const intersects = raycaster.intersectObjects(allObjects, false)
  
  let roomFound = false

  if (intersects.length > 0) {
    // Chercher dans les intersections s'il y a une chambre
    for (const intersect of intersects) {
      // Remonter dans la hiérarchie pour trouver un objet avec roomId
      let current = intersect.object
      while (current) {
        if (current.userData && current.userData.roomId) {
          const roomId = current.userData.roomId
          
          // Créer les données de la chambre
          const roomNumber = roomId.replace('room-', '')
          const roomTypes = ['standard', 'deluxe', 'suite'] as const
          const roomData = {
            id: roomId,
            number: roomNumber,
            status: 'available' as const,
            type: roomTypes[Math.floor(Math.random() * 3)],
            price: 100 + parseInt(roomNumber) * 10,
            capacity: 2,
            amenities: ['wifi', 'tv', 'minibar', 'climatisation', 'room service']
          }
          
          emit('selectRoom', roomData)
          roomFound = true
          break
        }
        if (current.parent) {
          current = current.parent
        } else {
          break
        }
        if (current === scene) break
      }
      if (roomFound) break
    }
  }
  
  // Si aucune chambre trouvée, fermer le panneau
  if (!roomFound) {
    emit('selectRoom', null)
  }
}

function handleMouseMove(event: MouseEvent) {
  if (!threeContainer.value) return

  const rect = threeContainer.value.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(mouse, camera)
  
  // Reset previous hover effect
  if (hoveredRoom && hoveredRoom !== selectedRoom) {
    hoveredRoom.children.forEach(child => {
      if (child instanceof THREE.Mesh && child.material && !child.name?.includes('clickarea')) {
        const material = child.material as THREE.MeshLambertMaterial
        if (child.name?.includes('wall')) {
          material.opacity = 0.7 // Reset wall opacity
        }
        material.emissive.setHex(0x000000) // Reset glow completely
      }
    })
    
    // Set target scale to 1 for smooth shrinking animation
    const roomId = hoveredRoom.userData.roomId
    if (roomId) {
      if (!roomAnimations.has(roomId)) {
        roomAnimations.set(roomId, { targetScale: 1, currentScale: hoveredRoom.scale.x })
      } else {
        roomAnimations.get(roomId)!.targetScale = 1
      }
      
      // Hide label for this room
      if (labelAnimations.has(roomId)) {
        labelAnimations.get(roomId)!.targetOpacity = 0
      }
    }
    hoveredRoom = null
  }

  // Reset target scale when not hovering
  if (!hoveredRoom) {
    // targetHoverScale = 1  // Temporairement commenté
  }

  // Set default cursor
  if (threeContainer.value) {
    threeContainer.value.style.cursor = 'default'
  }

  // Get all clickable objects in the scene
  const allClickableObjects: THREE.Object3D[] = []
  scene.traverse((child) => {
    if (child.userData && child.userData.roomId) {
      allClickableObjects.push(child)
    }
  })
  
  const intersects = raycaster.intersectObjects(allClickableObjects, true)

  if (intersects.length > 0) {
    // Find the room ID from the intersected object
    const intersectedObject = intersects[0].object
    const roomId = intersectedObject.userData.roomId
    
    if (roomId && roomMeshes.has(roomId)) {
      const roomGroup = roomMeshes.get(roomId)
      if (roomGroup && roomGroup !== selectedRoom) {
        hoveredRoom = roomGroup
        
        // Apply hover effect
        roomGroup.children.forEach(child => {
          if (child instanceof THREE.Mesh && child.material && !child.name?.includes('clickarea')) {
            const material = child.material as THREE.MeshLambertMaterial
            if (child.name?.includes('wall')) {
              material.opacity = 0.9 // Increase wall opacity on hover
            }
            material.emissive.setHex(0x333366) // Add blue-ish hover glow
          }
        })
        
        // Set target scale for smooth animation
        const roomId = roomGroup.userData.roomId
        if (roomId) {
          if (!roomAnimations.has(roomId)) {
            roomAnimations.set(roomId, { targetScale: 1.1, currentScale: roomGroup.scale.x })
          } else {
            roomAnimations.get(roomId)!.targetScale = 1.1
          }
          
          // Show label for this room with 100% opacity
          if (labelAnimations.has(roomId)) {
            labelAnimations.get(roomId)!.targetOpacity = 1.0
          }
        }
        
        // Change cursor to pointer
        if (threeContainer.value) {
          threeContainer.value.style.cursor = 'pointer'
        }
      }
    }
  }
}

function animate() {
  animationId = requestAnimationFrame(animate)
  
  // Animate all rooms that have active animations
  roomAnimations.forEach((animation, roomId) => {
    const roomGroup = roomMeshes.get(roomId)
    if (roomGroup) {
      // Smooth scale animation
      animation.currentScale += (animation.targetScale - animation.currentScale) * hoverAnimationSpeed
      roomGroup.scale.set(animation.currentScale, animation.currentScale, animation.currentScale)
      
      // Remove animation if it's close enough to target (avoid infinite micro-animations)
      if (Math.abs(animation.currentScale - animation.targetScale) < 0.01) {
        animation.currentScale = animation.targetScale
        roomGroup.scale.set(animation.targetScale, animation.targetScale, animation.targetScale)
        
        // Remove from animations map if back to normal size
        if (animation.targetScale === 1) {
          roomAnimations.delete(roomId)
        }
      }
    }
  })
  
  // Animate all room labels that have active animations
  labelAnimations.forEach((animation, roomId) => {
    const label = roomLabels.get(roomId)
    if (label && label.material instanceof THREE.MeshBasicMaterial) {
      // Smooth opacity animation
      animation.currentOpacity += (animation.targetOpacity - animation.currentOpacity) * labelAnimationSpeed
      label.material.opacity = animation.currentOpacity
      
      // Remove animation if it's close enough to target
      if (Math.abs(animation.currentOpacity - animation.targetOpacity) < 0.01) {
        animation.currentOpacity = animation.targetOpacity
        label.material.opacity = animation.targetOpacity
        
        // Keep the animation in the map but mark it as stable
        // This allows for quick re-activation on hover
      }
    }
  })
  
  renderer.render(scene, camera)
}

function handleResize() {
  if (!renderer || !camera || !threeContainer.value) return
  updateCameraZoom()
  renderer.setSize(threeContainer.value.clientWidth, threeContainer.value.clientHeight)
}

function handleWheel(event: WheelEvent) {
  event.preventDefault()
  
  if (!camera) return
  
  // Update zoom level
  if (event.deltaY < 0) {
    // Scroll up - zoom in
    if (zoomLevel < maxZoom) {
      zoomLevel += zoomStep
    }
  } else {
    // Scroll down - zoom out
    if (zoomLevel > minZoom) {
      zoomLevel -= zoomStep
    }
  }
  
  // Update camera projection
  updateCameraZoom()
}

// Fonctions utilitaires pour les statuts
function getStatusText(status: string): string {
  switch (status) {
    case 'available':
      return 'DISPONIBLE'
    case 'occupied':
      return 'OCCUPÉ'
    case 'maintenance':
      return 'MAINTENANCE'
    case 'cleaning':
      return 'NETTOYAGE'
    default:
      return 'INCONNU'
  }
}

function getStatusColor(status: string): string {
  switch (status) {
    case 'available':
      return '#16a34a'  // Vert
    case 'occupied':
      return '#dc2626'  // Rouge
    case 'maintenance':
      return '#60a5fa'  // Bleu clair
    case 'cleaning':
      return '#fbbf24'  // Jaune beurre
    default:
      return '#6b7280'  // Gris
  }
}

// Public method to update room status
function updateRoomStatus(roomId: string, newStatus: string) {
  // Find the room in hotelRooms data and update it
  const roomIndex = hotelRooms.findIndex(room => room.id === roomId)
  if (roomIndex !== -1) {
    hotelRooms[roomIndex].status = newStatus as any
    
    // Update the 3D room visual
    const roomGroup = roomMeshes.get(roomId)
    if (roomGroup) {
      const newColor = getStatusColorHex(newStatus)
      
      // Update room floor color
      const roomFloor = roomGroup.getObjectByName(`room-floor-${roomId}`) as THREE.Mesh
      if (roomFloor && roomFloor.material instanceof THREE.MeshLambertMaterial) {
        roomFloor.material.color.setHex(newColor)
      }
      
      // Update wall colors
      const wallNames = [`room-wall-back-${roomId}`, `room-wall-left-${roomId}`, `room-wall-right-${roomId}`]
      wallNames.forEach(wallName => {
        const wall = roomGroup.getObjectByName(wallName) as THREE.Mesh
        if (wall && wall.material instanceof THREE.MeshLambertMaterial) {
          wall.material.color.setHex(newColor)
        }
      })
      
      // Update room label with new status and color
      updateRoomLabel(roomId, hotelRooms[roomIndex])
      
      // Update userData for the room group and all its children
      roomGroup.userData.roomData = hotelRooms[roomIndex]
      roomGroup.traverse((child) => {
        if (child.userData.roomId === roomId) {
          child.userData.roomData = hotelRooms[roomIndex]
        }
      })
    }
  }
}

function updateRoomLabel(roomId: string, roomData: any) {
  // Remove existing label
  const existingLabel = roomLabels.get(roomId)
  if (existingLabel) {
    scene.remove(existingLabel)
    roomLabels.delete(roomId)
  }
  
  // Create new label with updated status
  createRoomLabel(roomData)
}

// Expose methods for parent component
defineExpose({
  updateRoomStatus
})

// Lifecycle
onMounted(async () => {
  await nextTick()
  if (typeof window !== 'undefined' && threeContainer.value) {
    scene = new THREE.Scene()
    scene.background = new THREE.Color(0xf5f5f5)
    
    const aspect = window.innerWidth / window.innerHeight
    const frustumSize = 35 // Augmenté de 25 à 35 pour voir la zone étendue
    camera = new THREE.OrthographicCamera(
      -frustumSize * aspect / 2, 
      frustumSize * aspect / 2,
      frustumSize / 2,
      -frustumSize / 2,
      1,
      1000
    )
    camera.position.set(25, 25, 25) // Augmenté pour voir la zone étendue
    camera.lookAt(0, 0, 0)
    
    renderer = new THREE.WebGLRenderer({ antialias: true })
    renderer.setSize(threeContainer.value.clientWidth, threeContainer.value.clientHeight)
    renderer.shadowMap.enabled = true
    renderer.shadowMap.type = THREE.PCFSoftShadowMap
    threeContainer.value.appendChild(renderer.domElement)
    
    raycaster = new THREE.Raycaster()
    mouse = new THREE.Vector2()
    
    setupLighting()
    createHotelFloor()
    animate()
    
    // Event listeners
    renderer.domElement.addEventListener('click', handleClick)
    renderer.domElement.addEventListener('mousemove', handleMouseMove)
    renderer.domElement.addEventListener('wheel', handleWheel, { passive: false })
    window.addEventListener('resize', handleResize)
  }
})

onBeforeUnmount(() => {
  if (renderer && threeContainer.value) {
    renderer.domElement.removeEventListener('click', handleClick)
    renderer.domElement.removeEventListener('mousemove', handleMouseMove)
    renderer.domElement.removeEventListener('wheel', handleWheel)
    threeContainer.value.removeChild(renderer.domElement)
  }
  window.removeEventListener('resize', handleResize)
  if (animationId) cancelAnimationFrame(animationId)
  if (renderer) renderer.dispose()
})
</script>

<style scoped>
.w-full {
  width: 100%;
}

.h-full {
  height: 100%;
}
</style>
