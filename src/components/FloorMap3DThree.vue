<template>
  <div class="w-full h-full relative overflow-hidden">
    <!-- Three.js Container -->
    <div ref="threeContainer" class="w-full h-full bg-gradient-to-b from-slate-900 via-slate-800 to-slate-900"></div>
    
    <!-- UI Overlay -->
    <div class="absolute top-6 left-6 z-10">
      <h2 class="text-2xl font-bold text-white bg-black/40 backdrop-blur-md px-4 py-2 rounded-xl shadow-2xl border border-white/10">
        Étage {{ selectedFloor }}
      </h2>
    </div>
    
    <!-- Floating Price Badges -->
    <div class="absolute inset-0 pointer-events-none z-20">
      <div 
        v-for="(room, index) in roomsToDisplay" 
        :key="`badge-${room.id}`"
        :style="getBadgeStyle(index)"
        class="absolute transform -translate-x-1/2 -translate-y-1/2 pointer-events-auto"
      >
        <div 
          :class="[
            'px-4 py-2 rounded-2xl text-white font-bold shadow-2xl border border-white/20 transition-all duration-300 hover:scale-110 cursor-pointer backdrop-blur-md',
            getBadgeColorClass(room.status)
          ]"
          @click="selectRoom(room)"
        >
          <div class="text-sm font-bold">€{{ room.price }}</div>
          <div class="text-xs opacity-90">{{ getStatusText(room.status) }}</div>
        </div>
      </div>
    </div>
    
    <!-- Controls -->
    <div class="absolute bottom-6 right-6 z-10 space-y-2">
      <button 
        @click="resetCamera"
        class="bg-black/40 backdrop-blur-md hover:bg-black/60 text-white px-4 py-2 rounded-xl shadow-2xl border border-white/10 transition-all duration-300"
      >
        Reset View
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, nextTick, withDefaults, defineProps, defineEmits } from 'vue'
import * as THREE from 'three'

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

// Refs
const threeContainer = ref<HTMLDivElement>()

// Three.js variables
let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let raycaster: THREE.Raycaster
let mouse: THREE.Vector2
let roomMeshes: THREE.Group[] = []
let selectedRoomMesh: THREE.Group | null = null

// Room data with varied pricing
const roomsToDisplay = ref<Room[]>([
  { id: '101', number: '101', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor: 1 },
  { id: '102', number: '102', status: 'occupied', type: 'deluxe', price: 105, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor: 1 },
  { id: '103', number: '103', status: 'maintenance', type: 'suite', price: 140, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], floor: 1 },
  { id: '104', number: '104', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor: 1 },
  { id: '105', number: '105', status: 'cleaning', type: 'deluxe', price: 105, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor: 1 },
  { id: '106', number: '106', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor: 1 },
  { id: '107', number: '107', status: 'occupied', type: 'deluxe', price: 105, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor: 1 },
  { id: '108', number: '108', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor: 1 },
  { id: '109', number: '109', status: 'occupied', type: 'suite', price: 140, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], floor: 1 },
])

// Room positions in 3D space (x, z, width, depth) - Better layout like mockup
const roomPositions = [
  [-10, -8, 4.5, 5], // Room 101 - back row left
  [-4, -8, 4.5, 5],  // Room 102 - back row center-left
  [2, -8, 4.5, 5],   // Room 103 - back row center-right  
  [8, -8, 4.5, 5],   // Room 104 - back row right
  [-10, -1, 4.5, 5], // Room 105 - middle row left
  [-4, -1, 4.5, 5],  // Room 106 - middle row center-left
  [2, -1, 4.5, 5],   // Room 107 - middle row center-right
  [8, -1, 4.5, 5],   // Room 108 - middle row right
  [-4, 6, 4.5, 5],   // Room 109 - front row
]

function initThreeJS() {
  if (!threeContainer.value) return

  // Scene with professional setup
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x0f172a) // Dark background like mockup
  scene.fog = new THREE.Fog(0x0f172a, 50, 200)

  // Camera - Perfect isometric view with better positioning
  camera = new THREE.PerspectiveCamera(35, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(30, 40, 30)
  camera.lookAt(0, 0, 0)

  // Renderer with professional settings
  renderer = new THREE.WebGLRenderer({ 
    antialias: true, 
    alpha: true,
    powerPreference: "high-performance"
  })
  renderer.setSize(threeContainer.value.clientWidth, threeContainer.value.clientHeight)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.setClearColor(0x0f172a, 1)
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.0
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.physicallyCorrectLights = true
  
  threeContainer.value.appendChild(renderer.domElement)

  // Enhanced lighting setup
  setupAdvancedLighting()
  
  // Create the hotel environment
  createHotelEnvironment()
    // Create rooms
  createAdvancedRooms()

  // Raycaster for mouse interaction
  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()

  // Event listeners
  renderer.domElement.addEventListener('click', onMouseClick)
  renderer.domElement.addEventListener('mousemove', onMouseMove)
  window.addEventListener('resize', onWindowResize)

  // Start render loop
  animate()
}

function setupAdvancedLighting() {
  // Ambient light for overall scene illumination
  const ambientLight = new THREE.AmbientLight(0x404040, 0.3)
  scene.add(ambientLight)

  // Main key light (warm hotel lighting)
  const keyLight = new THREE.DirectionalLight(0xfff8dc, 2.0)
  keyLight.position.set(25, 35, 20)
  keyLight.castShadow = true
  keyLight.shadow.mapSize.width = 4096
  keyLight.shadow.mapSize.height = 4096
  keyLight.shadow.camera.near = 0.5
  keyLight.shadow.camera.far = 150
  keyLight.shadow.camera.left = -50
  keyLight.shadow.camera.right = 50
  keyLight.shadow.camera.top = 50
  keyLight.shadow.camera.bottom = -50
  keyLight.shadow.bias = -0.0001
  keyLight.shadow.radius = 8
  scene.add(keyLight)

  // Rim light from opposite side (cooler tone)
  const rimLight = new THREE.DirectionalLight(0x87ceeb, 1.0)
  rimLight.position.set(-20, 30, -15)
  rimLight.castShadow = true
  rimLight.shadow.mapSize.width = 2048
  rimLight.shadow.mapSize.height = 2048
  scene.add(rimLight)

  // Interior warm lighting
  const interiorLight = new THREE.PointLight(0xffdb8a, 1.5, 100)
  interiorLight.position.set(0, 8, 0)
  interiorLight.castShadow = true
  scene.add(interiorLight)

  // Accent lighting for depth
  const accentLight1 = new THREE.SpotLight(0xff7f50, 0.8, 50, Math.PI / 6, 0.3)
  accentLight1.position.set(15, 25, 15)
  accentLight1.target.position.set(5, 0, 5)
  scene.add(accentLight1)
  scene.add(accentLight1.target)

  const accentLight2 = new THREE.SpotLight(0x87ceeb, 0.6, 50, Math.PI / 6, 0.3)
  accentLight2.position.set(-15, 25, -15)
  accentLight2.target.position.set(-5, 0, -5)
  scene.add(accentLight2)
  scene.add(accentLight2.target)
}

function createHotelEnvironment() {
  // Create realistic hotel floor with marble-like pattern
  const floorGeometry = new THREE.PlaneGeometry(50, 40)
  const floorMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x2a2a2a,
    metalness: 0.1,
    roughness: 0.8,
    transparent: false
  })
  const floor = new THREE.Mesh(floorGeometry, floorMaterial)
  floor.rotation.x = -Math.PI / 2
  floor.position.y = 0
  floor.receiveShadow = true
  scene.add(floor)

  // Building base platform with elegant design
  const platformGeometry = new THREE.BoxGeometry(36, 0.8, 28)
  const platformMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x3a3a3a,
    metalness: 0.2,
    roughness: 0.6
  })
  const platform = new THREE.Mesh(platformGeometry, platformMaterial)
  platform.position.set(0, 0.4, 0)
  platform.receiveShadow = true
  platform.castShadow = true
  scene.add(platform)
  
  // Create modern building structure
  createModernBuildingStructure()
  
  // Add environmental details
  createEnvironmentalDetails()
}

function createModernBuildingStructure() {
  const wallHeight = 5
  const wallMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x4a4a4a,
    metalness: 0.1,
    roughness: 0.7
  })
  
  // Main building walls with modern design
  const wallThickness = 0.4
  
  // Back wall
  const backWallGeometry = new THREE.BoxGeometry(36, wallHeight, wallThickness)
  const backWall = new THREE.Mesh(backWallGeometry, wallMaterial)
  backWall.position.set(0, wallHeight/2, -14)
  backWall.castShadow = true
  backWall.receiveShadow = true
  scene.add(backWall)
  
  // Left wall  
  const leftWallGeometry = new THREE.BoxGeometry(wallThickness, wallHeight, 28)
  const leftWall = new THREE.Mesh(leftWallGeometry, wallMaterial)
  leftWall.position.set(-18, wallHeight/2, 0)
  leftWall.castShadow = true
  leftWall.receiveShadow = true
  scene.add(leftWall)
  
  // Right wall
  const rightWallGeometry = new THREE.BoxGeometry(wallThickness, wallHeight, 28)
  const rightWall = new THREE.Mesh(rightWallGeometry, wallMaterial)
  rightWall.position.set(18, wallHeight/2, 0)
  rightWall.castShadow = true
  rightWall.receiveShadow = true
  scene.add(rightWall)
  
  // Front partial walls for entrance
  const frontWallLeftGeometry = new THREE.BoxGeometry(12, wallHeight, wallThickness)
  const frontWallLeft = new THREE.Mesh(frontWallLeftGeometry, wallMaterial)
  frontWallLeft.position.set(-12, wallHeight/2, 14)
  frontWallLeft.castShadow = true
  scene.add(frontWallLeft)
  
  const frontWallRightGeometry = new THREE.BoxGeometry(12, wallHeight, wallThickness)
  const frontWallRight = new THREE.Mesh(frontWallRightGeometry, wallMaterial)
  frontWallRight.position.set(12, wallHeight/2, 14)
  frontWallRight.castShadow = true
  scene.add(frontWallRight)
}

function createEnvironmentalDetails() {
  // Add subtle lighting fixtures on walls
  for (let i = 0; i < 4; i++) {
    const fixture = new THREE.Mesh(
      new THREE.CylinderGeometry(0.3, 0.3, 0.2, 8),
      new THREE.MeshStandardMaterial({ 
        color: 0xffd700,
        emissive: 0x332200,
        metalness: 0.8,
        roughness: 0.2
      })
    )
    fixture.position.set(-15 + i * 10, 3.5, -13.8)
    fixture.castShadow = true
    scene.add(fixture)
  }
  
  // Add subtle floor patterns
  const patternGeometry = new THREE.RingGeometry(8, 12, 32)
  const patternMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x333333,
    transparent: true,
    opacity: 0.3,
    roughness: 0.9
  })
  const pattern = new THREE.Mesh(patternGeometry, patternMaterial)
  pattern.rotation.x = -Math.PI / 2
  pattern.position.y = 0.02
  pattern.receiveShadow = true
  scene.add(pattern)
function createAdvancedRooms() {
  roomMeshes = []
  
  roomsToDisplay.value.forEach((room, index) => {
    if (index >= roomPositions.length) return
    
    const [x, z, width, depth] = roomPositions[index]
    const roomGroup = createPremiumRoom(room, x, z, width, depth)
    roomGroup.userData = { room, index }
    roomMeshes.push(roomGroup)
    scene.add(roomGroup)
  })
}

function createPremiumRoom(room: Room, x: number, z: number, width: number, depth: number): THREE.Group {
  const roomGroup = new THREE.Group()
  
  // Enhanced room dimensions
  const wallHeight = 4.0
  const wallThickness = 0.2
  
  // Premium materials with realistic textures
  const wallMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xf8f6f0,
    roughness: 0.8,
    metalness: 0.0
  })
  
  const roomFloorMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x8b7355,
    roughness: 0.9,
    metalness: 0.1
  })

  // Elevated room floor with texture
  const floorGeometry = new THREE.BoxGeometry(width, 0.15, depth)
  const roomFloor = new THREE.Mesh(floorGeometry, roomFloorMaterial)
  roomFloor.position.set(0, 0.475, 0)
  roomFloor.receiveShadow = true
  roomFloor.castShadow = true
  roomGroup.add(roomFloor)
  
  // Interior walls with proper materials
  // Back wall
  const backWallGeometry = new THREE.BoxGeometry(width, wallHeight, wallThickness)
  const backWall = new THREE.Mesh(backWallGeometry, wallMaterial)
  backWall.position.set(0, wallHeight/2 + 0.4, -depth/2)
  backWall.castShadow = true
  backWall.receiveShadow = true
  roomGroup.add(backWall)
  
  // Left wall
  const sideWallGeometry = new THREE.BoxGeometry(wallThickness, wallHeight, depth)
  const leftWall = new THREE.Mesh(sideWallGeometry, wallMaterial)
  leftWall.position.set(-width/2, wallHeight/2 + 0.4, 0)
  leftWall.castShadow = true
  leftWall.receiveShadow = true
  roomGroup.add(leftWall)
  
  // Right wall
  const rightWall = new THREE.Mesh(sideWallGeometry, wallMaterial)
  rightWall.position.set(width/2, wallHeight/2 + 0.4, 0)
  rightWall.castShadow = true
  rightWall.receiveShadow = true
  roomGroup.add(rightWall)
  
  // Create luxury furniture
  createLuxuryFurniture(roomGroup, room, width, depth)
  
  // Create elegant door
  createElegantDoor(roomGroup, room, width, depth)
  
  // Add premium windows
  createPremiumWindow(roomGroup, width, depth)
  
  // Add room lighting
  createRoomLighting(roomGroup, width, depth)
  
  // Position the room
  roomGroup.position.set(x, 0, z)
  
  return roomGroup
}

function createLuxuryFurniture(roomGroup: THREE.Group, room: Room, width: number, depth: number) {
  // Luxury bed with detailed design
  const bedWidth = width * 0.6
  const bedDepth = depth * 0.5
  const bedHeight = 0.8
  
  // Premium bed frame (dark walnut)
  const bedFrameGeometry = new THREE.BoxGeometry(bedWidth, bedHeight * 0.4, bedDepth)
  const bedFrameMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x3d2914,
    roughness: 0.3,
    metalness: 0.1
  })
  const bedFrame = new THREE.Mesh(bedFrameGeometry, bedFrameMaterial)
  bedFrame.position.set(-width * 0.08, bedHeight * 0.2 + 0.4, -depth * 0.1)
  bedFrame.castShadow = true
  bedFrame.receiveShadow = true
  roomGroup.add(bedFrame)
  
  // Luxury mattress with proper proportions
  const mattressGeometry = new THREE.BoxGeometry(bedWidth * 0.96, bedHeight * 0.25, bedDepth * 0.96)
  const mattressMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xf5f2ed,
    roughness: 0.7,
    metalness: 0.0
  })
  const mattress = new THREE.Mesh(mattressGeometry, mattressMaterial)
  mattress.position.set(-width * 0.08, bedHeight * 0.525 + 0.4, -depth * 0.1)
  mattress.castShadow = true
  mattress.receiveShadow = true
  roomGroup.add(mattress)
  
  // Elegant headboard
  const headboardGeometry = new THREE.BoxGeometry(bedWidth * 1.1, bedHeight * 0.8, 0.1)
  const headboardMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x2a1f14,
    roughness: 0.4,
    metalness: 0.1
  })
  const headboard = new THREE.Mesh(headboardGeometry, headboardMaterial)
  headboard.position.set(-width * 0.08, bedHeight * 0.8 + 0.4, -depth * 0.35)
  headboard.castShadow = true
  roomGroup.add(headboard)
  
  // Premium pillows
  const pillowGeometry = new THREE.BoxGeometry(bedWidth * 0.2, bedHeight * 0.15, bedDepth * 0.2)
  const pillowMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xfefbf6,
    roughness: 0.8,
    metalness: 0.0
  })
  
  for (let i = 0; i < 2; i++) {
    const pillow = new THREE.Mesh(pillowGeometry, pillowMaterial)
    pillow.position.set(-width * 0.2 + i * bedWidth * 0.25, bedHeight * 0.7 + 0.4, -depth * 0.25)
    pillow.castShadow = true
    roomGroup.add(pillow)
  }
  
  // Luxury bedside tables
  for (let i = 0; i < 2; i++) {
    const tableGeometry = new THREE.BoxGeometry(width * 0.18, bedHeight * 0.6, depth * 0.18)
    const tableMaterial = new THREE.MeshStandardMaterial({ 
      color: 0x4a3426,
      roughness: 0.2,
      metalness: 0.15
    })
    const table = new THREE.Mesh(tableGeometry, tableMaterial)
    table.position.set(-width * 0.08 + (i === 0 ? -bedWidth * 0.65 : bedWidth * 0.65), bedHeight * 0.3 + 0.4, -depth * 0.1)
    table.castShadow = true
    table.receiveShadow = true
    roomGroup.add(table)
    
    // Table lamps
    const lampBaseGeometry = new THREE.CylinderGeometry(0.04, 0.06, 0.25, 12)
    const lampMaterial = new THREE.MeshStandardMaterial({ 
      color: 0x8b6f14,
      roughness: 0.1,
      metalness: 0.8
    })
    const lampBase = new THREE.Mesh(lampBaseGeometry, lampMaterial)
    lampBase.position.set(-width * 0.08 + (i === 0 ? -bedWidth * 0.65 : bedWidth * 0.65), bedHeight * 0.725 + 0.4, -depth * 0.1)
    lampBase.castShadow = true
    roomGroup.add(lampBase)
    
    // Lamp shade
    const lampShadeGeometry = new THREE.ConeGeometry(0.08, 0.12, 8)
    const lampShadeMaterial = new THREE.MeshStandardMaterial({ 
      color: 0xf5f0e8,
      transparent: true,
      opacity: 0.9
    })
    const lampShade = new THREE.Mesh(lampShadeGeometry, lampShadeMaterial)
    lampShade.position.set(-width * 0.08 + (i === 0 ? -bedWidth * 0.65 : bedWidth * 0.65), bedHeight * 0.9 + 0.4, -depth * 0.1)
    lampShade.castShadow = true
    roomGroup.add(lampShade)
  }
  
  // Luxury armchair
  const chairSeatGeometry = new THREE.BoxGeometry(width * 0.22, 0.12, depth * 0.22)
  const chairMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x6b4423,
    roughness: 0.6,
    metalness: 0.1
  })
  const chairSeat = new THREE.Mesh(chairSeatGeometry, chairMaterial)
  chairSeat.position.set(width * 0.28, 0.86, depth * 0.25)
  chairSeat.castShadow = true
  chairSeat.receiveShadow = true
  roomGroup.add(chairSeat)
  
  // Chair back with elegant curve
  const chairBackGeometry = new THREE.BoxGeometry(width * 0.22, 0.7, 0.08)
  const chairBack = new THREE.Mesh(chairBackGeometry, chairMaterial)
  chairBack.position.set(width * 0.28, 1.25, depth * 0.36)
  chairBack.castShadow = true
  roomGroup.add(chairBack)
  
  // Chair arms
  for (let i = 0; i < 2; i++) {
    const armGeometry = new THREE.BoxGeometry(0.05, 0.4, depth * 0.18)
    const arm = new THREE.Mesh(armGeometry, chairMaterial)
    arm.position.set(width * 0.28 + (i === 0 ? -width * 0.085 : width * 0.085), 1.05, depth * 0.25)
    arm.castShadow = true
    roomGroup.add(arm)
  }
  
  // Small coffee table
  const coffeeTableGeometry = new THREE.CylinderGeometry(width * 0.12, width * 0.12, 0.08, 16)
  const coffeeTableMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x8b6914,
    roughness: 0.1,
    metalness: 0.3
  })
  const coffeeTable = new THREE.Mesh(coffeeTableGeometry, coffeeTableMaterial)
  coffeeTable.position.set(width * 0.15, 0.84, depth * 0.1)
  coffeeTable.castShadow = true
  coffeeTable.receiveShadow = true
  roomGroup.add(coffeeTable)
}

function createElegantDoor(roomGroup: THREE.Group, room: Room, width: number, depth: number) {
  // Premium door frame
  const doorFrameGeometry = new THREE.BoxGeometry(1.0, 2.6, 0.12)
  const doorFrameMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x2a1f14,
    roughness: 0.3,
    metalness: 0.1
  })
  const doorFrame = new THREE.Mesh(doorFrameGeometry, doorFrameMaterial)
  doorFrame.position.set(-width * 0.2, 1.7, depth/2 - 0.03)
  doorFrame.castShadow = true
  doorFrame.receiveShadow = true
  roomGroup.add(doorFrame)
  
  // Elegant door with panels
  const doorGeometry = new THREE.BoxGeometry(0.9, 2.4, 0.08)
  const doorMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x4a3426,
    roughness: 0.2,
    metalness: 0.1
  })
  const door = new THREE.Mesh(doorGeometry, doorMaterial)
  door.position.set(-width * 0.2, 1.6, depth/2 - 0.01)
  door.castShadow = true
  door.receiveShadow = true
  roomGroup.add(door)
  
  // Door panels (decorative)
  for (let i = 0; i < 2; i++) {
    const panelGeometry = new THREE.BoxGeometry(0.35, 0.8, 0.02)
    const panelMaterial = new THREE.MeshStandardMaterial({ 
      color: 0x3d2914,
      roughness: 0.4
    })
    const panel = new THREE.Mesh(panelGeometry, panelMaterial)
    panel.position.set(-width * 0.2, 1.2 + i * 0.8, depth/2 + 0.02)
    roomGroup.add(panel)
  }
  
  // Premium door handle
  const handleGeometry = new THREE.CylinderGeometry(0.02, 0.02, 0.15, 8)
  const handleMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xd4af37,
    roughness: 0.1,
    metalness: 0.9
  })
  const handle = new THREE.Mesh(handleGeometry, handleMaterial)
  handle.rotation.z = Math.PI / 2
  handle.position.set(-width * 0.4, 1.6, depth/2 + 0.05)
  handle.castShadow = true
  roomGroup.add(handle)
  
  // Elegant room number plaque
  const plaqueGeometry = new THREE.BoxGeometry(0.3, 0.18, 0.03)
  const plaqueMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x1a1a1a,
    roughness: 0.1,
    metalness: 0.8
  })
  const plaque = new THREE.Mesh(plaqueGeometry, plaqueMaterial)
  plaque.position.set(-width * 0.2, 2.2, depth/2 + 0.04)
  plaque.castShadow = true
  roomGroup.add(plaque)
  
  // Number text (golden)
  const numberGeometry = new THREE.PlaneGeometry(0.15, 0.08)
  const numberMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xd4af37,
    emissive: 0x332200,
    roughness: 0.2,
    metalness: 0.7
  })
  const numberText = new THREE.Mesh(numberGeometry, numberMaterial)
  numberText.position.set(-width * 0.2, 2.2, depth/2 + 0.05)
  roomGroup.add(numberText)
}

function createPremiumWindow(roomGroup: THREE.Group, width: number, depth: number) {
  // Large elegant window frame
  const windowFrameGeometry = new THREE.BoxGeometry(1.4, 1.6, 0.12)
  const windowFrameMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xf5f2ed,
    roughness: 0.4,
    metalness: 0.1
  })
  const windowFrame = new THREE.Mesh(windowFrameGeometry, windowFrameMaterial)
  windowFrame.position.set(width * 0.15, 2.4, -depth/2 + 0.03)
  windowFrame.castShadow = true
  windowFrame.receiveShadow = true
  roomGroup.add(windowFrame)
  
  // High-quality window glass with reflections
  const windowGlassGeometry = new THREE.BoxGeometry(1.3, 1.5, 0.04)
  const windowGlassMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x87ceeb,
    transparent: true,
    opacity: 0.4,
    roughness: 0.0,
    metalness: 0.1,
    transmission: 0.9,
    thickness: 0.1
  })
  const windowGlass = new THREE.Mesh(windowGlassGeometry, windowGlassMaterial)
  windowGlass.position.set(width * 0.15, 2.4, -depth/2 + 0.01)
  roomGroup.add(windowGlass)
  
  // Window dividers for realistic look
  const dividerMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xe0ddd8,
    roughness: 0.6
  })
  
  // Horizontal divider
  const hDividerGeometry = new THREE.BoxGeometry(1.3, 0.03, 0.02)
  const hDivider = new THREE.Mesh(hDividerGeometry, dividerMaterial)
  hDivider.position.set(width * 0.15, 2.4, -depth/2 + 0.02)
  roomGroup.add(hDivider)
  
  // Vertical divider
  const vDividerGeometry = new THREE.BoxGeometry(0.03, 1.5, 0.02)
  const vDivider = new THREE.Mesh(vDividerGeometry, dividerMaterial)
  vDivider.position.set(width * 0.15, 2.4, -depth/2 + 0.02)
  roomGroup.add(vDivider)
}

function createRoomLighting(roomGroup: THREE.Group, width: number, depth: number) {
  // Ceiling light fixture
  const lightFixtureGeometry = new THREE.CylinderGeometry(0.15, 0.15, 0.05, 16)
  const lightFixtureMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xffffff,
    emissive: 0x444444,
    roughness: 0.2,
    metalness: 0.8
  })
  const lightFixture = new THREE.Mesh(lightFixtureGeometry, lightFixtureMaterial)
  lightFixture.position.set(0, 3.8, 0)
  lightFixture.castShadow = true
  roomGroup.add(lightFixture)
  
  // Add point light for room illumination
  const roomLight = new THREE.PointLight(0xfff8dc, 0.8, 15)
  roomLight.position.set(0, 3.5, 0)
  roomLight.castShadow = true
  roomLight.shadow.mapSize.width = 1024
  roomLight.shadow.mapSize.height = 1024
  roomGroup.add(roomLight)
}
  
  // Pillows (white/cream)
  const pillowGeometry = new THREE.BoxGeometry(bedWidth * 0.25, bedHeight * 0.2, bedDepth * 0.25)
  const pillowMaterial = new THREE.MeshLambertMaterial({ color: 0xfefcf8 })
  
  const pillow1 = new THREE.Mesh(pillowGeometry, pillowMaterial)
  pillow1.position.set(-width * 0.25, bedHeight * 0.9 + 0.5, -depth * 0.3)
  pillow1.castShadow = true
  roomGroup.add(pillow1)
  
  const pillow2 = new THREE.Mesh(pillowGeometry, pillowMaterial)
  pillow2.position.set(width * 0.05, bedHeight * 0.9 + 0.5, -depth * 0.3)
  pillow2.castShadow = true
  roomGroup.add(pillow2)
  
  // Bedside table
  const tableGeometry = new THREE.BoxGeometry(width * 0.15, bedHeight * 0.8, depth * 0.15)
  const tableMaterial = new THREE.MeshLambertMaterial({ color: 0xc9985a })
  const table = new THREE.Mesh(tableGeometry, tableMaterial)
  table.position.set(width * 0.25, bedHeight * 0.4 + 0.5, -depth * 0.15)
  table.castShadow = true
  roomGroup.add(table)
  
  // Small lamp on table
  const lampBaseGeometry = new THREE.CylinderGeometry(0.05, 0.08, 0.2, 8)
  const lampMaterial = new THREE.MeshLambertMaterial({ color: 0x8b7355 })
  const lampBase = new THREE.Mesh(lampBaseGeometry, lampMaterial)
  lampBase.position.set(width * 0.25, bedHeight * 0.9 + 0.5, -depth * 0.15)
  lampBase.castShadow = true
  roomGroup.add(lampBase)
  
  // Chair
  const chairSeatGeometry = new THREE.BoxGeometry(width * 0.18, 0.1, depth * 0.18)
  const chairMaterial = new THREE.MeshLambertMaterial({ color: 0xb89968 })
  const chairSeat = new THREE.Mesh(chairSeatGeometry, chairMaterial)
  chairSeat.position.set(width * 0.25, 0.95, depth * 0.2)
  chairSeat.castShadow = true
  roomGroup.add(chairSeat)
  
  // Chair back
  const chairBackGeometry = new THREE.BoxGeometry(width * 0.18, 0.6, 0.05)
  const chairBack = new THREE.Mesh(chairBackGeometry, chairMaterial)
  chairBack.position.set(width * 0.25, 1.25, depth * 0.3)
  chairBack.castShadow = true
  roomGroup.add(chairBack)
}

function onMouseClick(event: MouseEvent) {
  if (!threeContainer.value) return
  
  const rect = threeContainer.value.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(mouse, camera)
  const intersects = raycaster.intersectObjects(scene.children, true)

  let clickedRoom: Room | null = null
  
  for (const intersect of intersects) {
    let object = intersect.object
    while (object.parent && !object.userData?.room) {
      object = object.parent
    }
    
    if (object.userData?.room) {
      clickedRoom = object.userData.room
      break
    }
  }

  if (clickedRoom) {
    selectRoom(clickedRoom)
  } else {
    emit('clickOutside')
  }
}

function onMouseMove(event: MouseEvent) {
  if (!threeContainer.value) return
  
  const rect = threeContainer.value.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(mouse, camera)
  const intersects = raycaster.intersectObjects(scene.children, true)
  // Reset all room highlights
  roomMeshes.forEach(roomGroup => {
    roomGroup.children.forEach(child => {
      if (child instanceof THREE.Mesh && child.material instanceof THREE.MeshStandardMaterial) {
        child.material.emissive.setHex(0x000000)
      }
    })
  })

  // Highlight hovered room
  for (const intersect of intersects) {
    let object = intersect.object
    while (object.parent && !object.userData?.room) {
      object = object.parent
    }
    
    if (object.userData?.room) {
      const roomGroup = object as THREE.Group
      roomGroup.children.forEach(child => {
        if (child instanceof THREE.Mesh && child.material instanceof THREE.MeshStandardMaterial) {
          child.material.emissive.setHex(0x222222)
        }
      })
      threeContainer.value!.style.cursor = 'pointer'
      return
    }
  }
  
  threeContainer.value!.style.cursor = 'default'
}

function selectRoom(room: Room) {  // Highlight selected room
  if (selectedRoomMesh) {
    selectedRoomMesh.children.forEach(child => {
      if (child instanceof THREE.Mesh && child.material instanceof THREE.MeshStandardMaterial) {
        child.material.emissive.setHex(0x000000)
      }
    })
  }
  
  const roomGroup = roomMeshes.find(group => group.userData.room.id === room.id)
  if (roomGroup) {
    selectedRoomMesh = roomGroup
    roomGroup.children.forEach(child => {
      if (child instanceof THREE.Mesh && child.material instanceof THREE.MeshStandardMaterial) {
        child.material.emissive.setHex(0x004488)
      }
    })
  }
  
  emit('selectRoom', room)
}

function resetCamera() {
  camera.position.set(30, 40, 30)
  camera.lookAt(0, 0, 0)
}

function onWindowResize() {
  if (!threeContainer.value) return
  
  camera.aspect = threeContainer.value.clientWidth / threeContainer.value.clientHeight
  camera.updateProjectionMatrix()
  renderer.setSize(threeContainer.value.clientWidth, threeContainer.value.clientHeight)
}

function animate() {
  requestAnimationFrame(animate)
  renderer.render(scene, camera)
}

// UI Helper functions
function getBadgeStyle(index: number) {
  if (index >= roomPositions.length) return { display: 'none' }
  
  const [x, z] = roomPositions[index]
  
  // Convert 3D world position to screen position
  const vector = new THREE.Vector3(x, 4, z)
  vector.project(camera)
  
  const widthHalf = threeContainer.value?.clientWidth / 2 || 400
  const heightHalf = threeContainer.value?.clientHeight / 2 || 300
  
  return {
    left: `${(vector.x * widthHalf) + widthHalf}px`,
    top: `${-(vector.y * heightHalf) + heightHalf}px`,
  }
}

function getBadgeColorClass(status: string): string {
  const classes = {
    available: 'bg-green-500 hover:bg-green-600',
    occupied: 'bg-red-500 hover:bg-red-600',
    maintenance: 'bg-orange-500 hover:bg-orange-600',
    cleaning: 'bg-blue-500 hover:bg-blue-600'
  }
  return classes[status as keyof typeof classes] || 'bg-gray-500'
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

// Lifecycle
onMounted(async () => {
  await nextTick()
  initThreeJS()
})

onBeforeUnmount(() => {
  if (renderer) {
    renderer.domElement.removeEventListener('click', onMouseClick)
    renderer.domElement.removeEventListener('mousemove', onMouseMove)
    window.removeEventListener('resize', onWindowResize)
    
    if (threeContainer.value && renderer.domElement) {
      threeContainer.value.removeChild(renderer.domElement)
    }
    
    renderer.dispose()
  }
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
