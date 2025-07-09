<template>
  <div class="w-full h-full relative overflow-hidden bg-gradient-to-br from-gray-900 via-gray-800 to-gray-900">
    <!-- Background avec effet de texture -->
    <div class="absolute inset-0 opacity-20">
      <div class="w-full h-full bg-[radial-gradient(circle_at_50%_50%,rgba(120,119,198,0.3),rgba(255,255,255,0))]"></div>
    </div>
    
    <!-- Three.js Container -->
    <div ref="threeContainer" class="w-full h-full"></div>
    
    <!-- UI Overlay -->
    <div class="absolute top-6 left-6 z-10">
      <h2 class="text-2xl font-bold text-white bg-black/30 backdrop-blur-xl px-6 py-3 rounded-2xl shadow-2xl border border-white/10">
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
            'px-4 py-2 rounded-2xl text-white font-bold shadow-2xl border border-white/30 transition-all duration-300 hover:scale-110 cursor-pointer backdrop-blur-xl',
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
    <div class="absolute bottom-6 right-6 z-10 space-y-3">
      <button 
        @click="resetCamera"
        class="bg-black/30 backdrop-blur-xl hover:bg-black/50 text-white px-6 py-3 rounded-2xl shadow-2xl border border-white/10 transition-all duration-300 font-medium"
      >
        Vue Standard
      </button>
      <button 
        @click="toggleLighting"
        class="bg-black/30 backdrop-blur-xl hover:bg-black/50 text-white px-6 py-3 rounded-2xl shadow-2xl border border-white/10 transition-all duration-300 font-medium"
      >
        {{ nightMode ? 'Mode Jour' : 'Mode Nuit' }}
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
const nightMode = ref(false)

// Three.js variables
let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let raycaster: THREE.Raycaster
let mouse: THREE.Vector2
let roomMeshes: THREE.Group[] = []
let selectedRoomMesh: THREE.Group | null = null
let animationId: number

// Room data avec des prix variés
const roomsToDisplay = ref<Room[]>([
  { id: '101', number: '101', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor: 1 },
  { id: '102', number: '102', status: 'occupied', type: 'deluxe', price: 180, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor: 1 },
  { id: '103', number: '103', status: 'maintenance', type: 'suite', price: 280, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], floor: 1 },
  { id: '104', number: '104', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor: 1 },
  { id: '105', number: '105', status: 'cleaning', type: 'deluxe', price: 180, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor: 1 },
  { id: '106', number: '106', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor: 1 },
  { id: '107', number: '107', status: 'occupied', type: 'deluxe', price: 180, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], floor: 1 },
  { id: '108', number: '108', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], floor: 1 },
  { id: '109', number: '109', status: 'occupied', type: 'suite', price: 280, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], floor: 1 },
])

// Layout des chambres en 3D (x, z, largeur, profondeur)
const roomPositions = [
  [-12, -10, 5, 6], // Chambre 101 - rangée arrière gauche
  [-5, -10, 5, 6],  // Chambre 102 - rangée arrière centre-gauche
  [2, -10, 5, 6],   // Chambre 103 - rangée arrière centre-droite  
  [9, -10, 5, 6],   // Chambre 104 - rangée arrière droite
  [-12, -2, 5, 6],  // Chambre 105 - rangée milieu gauche
  [-5, -2, 5, 6],   // Chambre 106 - rangée milieu centre-gauche
  [2, -2, 5, 6],    // Chambre 107 - rangée milieu centre-droite
  [9, -2, 5, 6],    // Chambre 108 - rangée milieu droite
  [-5, 6, 5, 6],    // Chambre 109 - rangée avant centre
]

function initThreeJS() {
  if (!threeContainer.value) return

  // Scene avec arrière-plan sombre sophistiqué
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x0a0a0a)
  scene.fog = new THREE.Fog(0x0a0a0a, 80, 200)

  // Caméra avec perspective parfaite
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(25, 30, 25)
  camera.lookAt(0, 0, 0)

  // Renderer ultra-haute qualité
  renderer = new THREE.WebGLRenderer({ 
    antialias: true, 
    alpha: true,
    powerPreference: "high-performance"
  })
  renderer.setSize(threeContainer.value.clientWidth, threeContainer.value.clientHeight)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.setClearColor(0x0a0a0a, 1)
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.2
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.physicallyCorrectLights = true
  
  threeContainer.value.appendChild(renderer.domElement)

  // Configuration de l'éclairage cinématographique
  setupCinematicLighting()
  
  // Création de l'environnement hôtelier
  createLuxuryHotelEnvironment()
  
  // Création des chambres réalistes
  createRealisticRooms()

  // Raycaster pour les interactions
  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()

  // Event listeners
  renderer.domElement.addEventListener('click', onMouseClick)
  renderer.domElement.addEventListener('mousemove', onMouseMove)
  window.addEventListener('resize', onWindowResize)

  // Démarrage de la boucle de rendu
  animate()
}

function setupCinematicLighting() {
  // Éclairage ambiant très subtil
  const ambientLight = new THREE.AmbientLight(0x1a1a2e, 0.1)
  scene.add(ambientLight)

  // Éclairage principal dramatique
  const mainLight = new THREE.DirectionalLight(0xfff4e6, 3.0)
  mainLight.position.set(30, 50, 20)
  mainLight.castShadow = true
  mainLight.shadow.mapSize.width = 8192
  mainLight.shadow.mapSize.height = 8192
  mainLight.shadow.camera.near = 0.5
  mainLight.shadow.camera.far = 200
  mainLight.shadow.camera.left = -60
  mainLight.shadow.camera.right = 60
  mainLight.shadow.camera.top = 60
  mainLight.shadow.camera.bottom = -60
  mainLight.shadow.bias = -0.0005
  mainLight.shadow.normalBias = 0.02
  scene.add(mainLight)

  // Éclairage de contre-jour bleu froid
  const rimLight = new THREE.DirectionalLight(0x4a90e2, 2.0)
  rimLight.position.set(-25, 40, -20)
  rimLight.castShadow = true
  rimLight.shadow.mapSize.width = 4096
  rimLight.shadow.mapSize.height = 4096
  scene.add(rimLight)

  // Éclairage d'accent doré chaud
  const accentLight = new THREE.SpotLight(0xffd700, 1.5, 100, Math.PI / 8, 0.5)
  accentLight.position.set(0, 40, 0)
  accentLight.target.position.set(0, 0, 0)
  accentLight.castShadow = true
  scene.add(accentLight)
  scene.add(accentLight.target)

  // Éclairages ponctuels pour créer de la profondeur
  const pointLight1 = new THREE.PointLight(0xff6b6b, 0.8, 50)
  pointLight1.position.set(-20, 15, 20)
  scene.add(pointLight1)

  const pointLight2 = new THREE.PointLight(0x4ecdc4, 0.8, 50)
  pointLight2.position.set(20, 15, -20)
  scene.add(pointLight2)
}

function createLuxuryHotelEnvironment() {
  // Sol principal avec effet de marbre noir
  const floorGeometry = new THREE.PlaneGeometry(60, 50)
  const floorMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x0f0f0f,
    metalness: 0.8,
    roughness: 0.2,
    envMapIntensity: 1.0
  })
  const floor = new THREE.Mesh(floorGeometry, floorMaterial)
  floor.rotation.x = -Math.PI / 2
  floor.position.y = 0
  floor.receiveShadow = true
  scene.add(floor)

  // Plateforme de l'hôtel avec des matériaux luxueux
  const platformGeometry = new THREE.BoxGeometry(40, 1.0, 32)
  const platformMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x2c2c2c,
    metalness: 0.4,
    roughness: 0.3,
    envMapIntensity: 0.8
  })
  const platform = new THREE.Mesh(platformGeometry, platformMaterial)
  platform.position.set(0, 0.5, 0)
  platform.receiveShadow = true
  platform.castShadow = true
  scene.add(platform)
  
  // Création des murs extérieurs modernes
  createModernExteriorWalls()
  
  // Ajout d'éléments décoratifs
  createDecorativeElements()
}

function createModernExteriorWalls() {
  const wallHeight = 6
  const wallMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x1a1a1a,
    metalness: 0.1,
    roughness: 0.8,
    envMapIntensity: 0.5
  })
  
  const wallThickness = 0.5
  
  // Mur arrière avec texture sophistiquée
  const backWallGeometry = new THREE.BoxGeometry(40, wallHeight, wallThickness)
  const backWall = new THREE.Mesh(backWallGeometry, wallMaterial)
  backWall.position.set(0, wallHeight/2, -16)
  backWall.castShadow = true
  backWall.receiveShadow = true
  scene.add(backWall)
  
  // Mur gauche
  const leftWallGeometry = new THREE.BoxGeometry(wallThickness, wallHeight, 32)
  const leftWall = new THREE.Mesh(leftWallGeometry, wallMaterial)
  leftWall.position.set(-20, wallHeight/2, 0)
  leftWall.castShadow = true
  leftWall.receiveShadow = true
  scene.add(leftWall)
  
  // Mur droit
  const rightWallGeometry = new THREE.BoxGeometry(wallThickness, wallHeight, 32)
  const rightWall = new THREE.Mesh(rightWallGeometry, wallMaterial)
  rightWall.position.set(20, wallHeight/2, 0)
  rightWall.castShadow = true
  rightWall.receiveShadow = true
  scene.add(rightWall)
  
  // Murs avant partiels pour l'entrée
  const frontWallLeftGeometry = new THREE.BoxGeometry(14, wallHeight, wallThickness)
  const frontWallLeft = new THREE.Mesh(frontWallLeftGeometry, wallMaterial)
  frontWallLeft.position.set(-13, wallHeight/2, 16)
  frontWallLeft.castShadow = true
  scene.add(frontWallLeft)
  
  const frontWallRightGeometry = new THREE.BoxGeometry(14, wallHeight, wallThickness)
  const frontWallRight = new THREE.Mesh(frontWallRightGeometry, wallMaterial)
  frontWallRight.position.set(13, wallHeight/2, 16)
  frontWallRight.castShadow = true
  scene.add(frontWallRight)
}

function createDecorativeElements() {
  // Colonnes décoratives modernes
  for (let i = 0; i < 3; i++) {
    const columnGeometry = new THREE.CylinderGeometry(0.4, 0.4, 5.5, 12)
    const columnMaterial = new THREE.MeshStandardMaterial({ 
      color: 0x333333,
      metalness: 0.6,
      roughness: 0.2
    })
    const column = new THREE.Mesh(columnGeometry, columnMaterial)
    column.position.set(-15 + i * 15, 2.75, 15)
    column.castShadow = true
    column.receiveShadow = true
    scene.add(column)
  }
  
  // Éclairages décoratifs sur les murs
  for (let i = 0; i < 5; i++) {
    const lightFixtureGeometry = new THREE.CylinderGeometry(0.3, 0.3, 0.1, 16)
    const lightFixtureMaterial = new THREE.MeshStandardMaterial({ 
      color: 0xffffff,
      emissive: 0x444444,
      metalness: 0.9,
      roughness: 0.1
    })
    const lightFixture = new THREE.Mesh(lightFixtureGeometry, lightFixtureMaterial)
    lightFixture.position.set(-16 + i * 8, 4.5, -15.7)
    lightFixture.castShadow = true
    scene.add(lightFixture)
    
    // Point light pour chaque fixture
    const wallLight = new THREE.PointLight(0xfff8dc, 0.5, 20)
    wallLight.position.set(-16 + i * 8, 4.5, -15)
    scene.add(wallLight)
  }
}

function createRealisticRooms() {
  roomMeshes = []
  
  roomsToDisplay.value.forEach((room, index) => {
    if (index >= roomPositions.length) return
    
    const [x, z, width, depth] = roomPositions[index]
    const roomGroup = createLuxuryRoom(room, x, z, width, depth)
    roomGroup.userData = { room, index }
    roomMeshes.push(roomGroup)
    scene.add(roomGroup)
  })
}

function createLuxuryRoom(room: Room, x: number, z: number, width: number, depth: number): THREE.Group {
  const roomGroup = new THREE.Group()
  
  // Dimensions de chambre luxueuse
  const wallHeight = 4.5
  const wallThickness = 0.25
  
  // Matériaux premium avec PBR
  const wallMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xf5f3f0,
    roughness: 0.7,
    metalness: 0.0,
    envMapIntensity: 0.3
  })
  
  const roomFloorMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x3d2914,
    roughness: 0.8,
    metalness: 0.1,
    envMapIntensity: 0.4
  })

  // Sol de chambre avec parquet de luxe
  const floorGeometry = new THREE.BoxGeometry(width, 0.2, depth)
  const roomFloor = new THREE.Mesh(floorGeometry, roomFloorMaterial)
  roomFloor.position.set(0, 0.6, 0)
  roomFloor.receiveShadow = true
  roomFloor.castShadow = true
  roomGroup.add(roomFloor)
  
  // Murs intérieurs avec matériaux réalistes
  const walls = [
    { geometry: new THREE.BoxGeometry(width, wallHeight, wallThickness), position: [0, wallHeight/2 + 0.5, -depth/2] },
    { geometry: new THREE.BoxGeometry(wallThickness, wallHeight, depth), position: [-width/2, wallHeight/2 + 0.5, 0] },
    { geometry: new THREE.BoxGeometry(wallThickness, wallHeight, depth), position: [width/2, wallHeight/2 + 0.5, 0] }
  ]
  
  walls.forEach(wall => {
    const wallMesh = new THREE.Mesh(wall.geometry, wallMaterial)
    wallMesh.position.set(...wall.position)
    wallMesh.castShadow = true
    wallMesh.receiveShadow = true
    roomGroup.add(wallMesh)
  })
  
  // Mobilier de luxe ultra-détaillé
  createUltraLuxuryFurniture(roomGroup, room, width, depth)
  
  // Porte d'entrée sophistiquée
  createSophisticatedDoor(roomGroup, room, width, depth)
  
  // Fenêtres panoramiques
  createPanoramicWindows(roomGroup, width, depth)
  
  // Éclairage intérieur ambiant
  createAmbientRoomLighting(roomGroup, width, depth)
  
  // Positionnement de la chambre
  roomGroup.position.set(x, 0, z)
  
  return roomGroup
}

function createUltraLuxuryFurniture(roomGroup: THREE.Group, room: Room, width: number, depth: number) {
  // Lit king-size avec design moderne
  const bedWidth = width * 0.7
  const bedDepth = depth * 0.55
  const bedHeight = 1.0
  
  // Cadre de lit en bois d'ébène
  const bedFrameGeometry = new THREE.BoxGeometry(bedWidth, bedHeight * 0.3, bedDepth)
  const bedFrameMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x1a1a1a,
    roughness: 0.1,
    metalness: 0.2,
    envMapIntensity: 0.8
  })
  const bedFrame = new THREE.Mesh(bedFrameGeometry, bedFrameMaterial)
  bedFrame.position.set(-width * 0.05, bedHeight * 0.15 + 0.5, -depth * 0.08)
  bedFrame.castShadow = true
  bedFrame.receiveShadow = true
  roomGroup.add(bedFrame)
  
  // Matelas premium
  const mattressGeometry = new THREE.BoxGeometry(bedWidth * 0.98, bedHeight * 0.2, bedDepth * 0.98)
  const mattressMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xfaf8f5,
    roughness: 0.8,
    metalness: 0.0
  })
  const mattress = new THREE.Mesh(mattressGeometry, mattressMaterial)
  mattress.position.set(-width * 0.05, bedHeight * 0.4 + 0.5, -depth * 0.08)
  mattress.castShadow = true
  mattress.receiveShadow = true
  roomGroup.add(mattress)
  
  // Tête de lit capitonnée luxueuse
  const headboardGeometry = new THREE.BoxGeometry(bedWidth * 1.2, bedHeight * 0.9, 0.15)
  const headboardMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x2c1810,
    roughness: 0.3,
    metalness: 0.1
  })
  const headboard = new THREE.Mesh(headboardGeometry, headboardMaterial)
  headboard.position.set(-width * 0.05, bedHeight * 0.7 + 0.5, -depth * 0.4)
  headboard.castShadow = true
  roomGroup.add(headboard)
  
  // Oreillers de luxe
  for (let i = 0; i < 4; i++) {
    const pillowGeometry = new THREE.BoxGeometry(bedWidth * 0.15, bedHeight * 0.12, bedDepth * 0.15)
    const pillowMaterial = new THREE.MeshStandardMaterial({ 
      color: 0xffffff,
      roughness: 0.9,
      metalness: 0.0
    })
    const pillow = new THREE.Mesh(pillowGeometry, pillowMaterial)
    pillow.position.set(-width * 0.2 + i * bedWidth * 0.13, bedHeight * 0.56 + 0.5, -depth * 0.25)
    pillow.castShadow = true
    roomGroup.add(pillow)
  }
  
  // Tables de chevet design
  for (let i = 0; i < 2; i++) {
    const tableGeometry = new THREE.BoxGeometry(width * 0.2, bedHeight * 0.5, depth * 0.2)
    const tableMaterial = new THREE.MeshStandardMaterial({ 
      color: 0x1a1a1a,
      roughness: 0.1,
      metalness: 0.3,
      envMapIntensity: 0.9
    })
    const table = new THREE.Mesh(tableGeometry, tableMaterial)
    table.position.set(-width * 0.05 + (i === 0 ? -bedWidth * 0.7 : bedWidth * 0.7), bedHeight * 0.25 + 0.5, -depth * 0.08)
    table.castShadow = true
    table.receiveShadow = true
    roomGroup.add(table)
    
    // Lampes design modernes
    const lampBaseGeometry = new THREE.CylinderGeometry(0.03, 0.05, 0.3, 12)
    const lampMaterial = new THREE.MeshStandardMaterial({ 
      color: 0xd4af37,
      roughness: 0.05,
      metalness: 0.95,
      envMapIntensity: 1.0
    })
    const lampBase = new THREE.Mesh(lampBaseGeometry, lampMaterial)
    lampBase.position.set(-width * 0.05 + (i === 0 ? -bedWidth * 0.7 : bedWidth * 0.7), bedHeight * 0.65 + 0.5, -depth * 0.08)
    lampBase.castShadow = true
    roomGroup.add(lampBase)
    
    // Abat-jour élégant
    const lampShadeGeometry = new THREE.ConeGeometry(0.1, 0.15, 12)
    const lampShadeMaterial = new THREE.MeshStandardMaterial({ 
      color: 0xf8f6f0,
      transparent: true,
      opacity: 0.95,
      roughness: 0.8
    })
    const lampShade = new THREE.Mesh(lampShadeGeometry, lampShadeMaterial)
    lampShade.position.set(-width * 0.05 + (i === 0 ? -bedWidth * 0.7 : bedWidth * 0.7), bedHeight * 0.85 + 0.5, -depth * 0.08)
    lampShade.castShadow = true
    roomGroup.add(lampShade)
    
    // Point light pour chaque lampe
    const lampLight = new THREE.PointLight(0xfff8dc, 0.3, 10)
    lampLight.position.set(-width * 0.05 + (i === 0 ? -bedWidth * 0.7 : bedWidth * 0.7), bedHeight * 0.85 + 0.5, -depth * 0.08)
    roomGroup.add(lampLight)
  }
  
  // Fauteuil lounge de luxe
  const chairSeatGeometry = new THREE.BoxGeometry(width * 0.25, 0.15, depth * 0.25)
  const chairMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x4a3426,
    roughness: 0.4,
    metalness: 0.1
  })
  const chairSeat = new THREE.Mesh(chairSeatGeometry, chairMaterial)
  chairSeat.position.set(width * 0.3, 0.75 + 0.5, depth * 0.28)
  chairSeat.castShadow = true
  chairSeat.receiveShadow = true
  roomGroup.add(chairSeat)
  
  // Dossier de fauteuil courbé
  const chairBackGeometry = new THREE.BoxGeometry(width * 0.25, 0.8, 0.1)
  const chairBack = new THREE.Mesh(chairBackGeometry, chairMaterial)
  chairBack.position.set(width * 0.3, 1.2 + 0.5, depth * 0.4)
  chairBack.castShadow = true
  roomGroup.add(chairBack)
  
  // Accoudoirs du fauteuil
  for (let i = 0; i < 2; i++) {
    const armGeometry = new THREE.BoxGeometry(0.06, 0.5, depth * 0.2)
    const arm = new THREE.Mesh(armGeometry, chairMaterial)
    arm.position.set(width * 0.3 + (i === 0 ? -width * 0.1 : width * 0.1), 1.0 + 0.5, depth * 0.28)
    arm.castShadow = true
    roomGroup.add(arm)
  }
  
  // Table basse design
  const coffeeTableGeometry = new THREE.CylinderGeometry(width * 0.15, width * 0.15, 0.1, 20)
  const coffeeTableMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x1a1a1a,
    roughness: 0.05,
    metalness: 0.8,
    envMapIntensity: 1.0
  })
  const coffeeTable = new THREE.Mesh(coffeeTableGeometry, coffeeTableMaterial)
  coffeeTable.position.set(width * 0.15, 0.75 + 0.5, depth * 0.1)
  coffeeTable.castShadow = true
  coffeeTable.receiveShadow = true
  roomGroup.add(coffeeTable)
}

function createSophisticatedDoor(roomGroup: THREE.Group, room: Room, width: number, depth: number) {
  // Cadre de porte en acier brossé
  const doorFrameGeometry = new THREE.BoxGeometry(1.1, 2.8, 0.15)
  const doorFrameMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x2c2c2c,
    roughness: 0.2,
    metalness: 0.8
  })
  const doorFrame = new THREE.Mesh(doorFrameGeometry, doorFrameMaterial)
  doorFrame.position.set(-width * 0.15, 1.9, depth/2 - 0.05)
  doorFrame.castShadow = true
  doorFrame.receiveShadow = true
  roomGroup.add(doorFrame)
  
  // Porte en bois laqué noir
  const doorGeometry = new THREE.BoxGeometry(1.0, 2.6, 0.1)
  const doorMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x0a0a0a,
    roughness: 0.1,
    metalness: 0.3,
    envMapIntensity: 0.9
  })
  const door = new THREE.Mesh(doorGeometry, doorMaterial)
  door.position.set(-width * 0.15, 1.8, depth/2 - 0.02)
  door.castShadow = true
  door.receiveShadow = true
  roomGroup.add(door)
  
  // Poignée design moderne
  const handleGeometry = new THREE.CylinderGeometry(0.015, 0.015, 0.2, 12)
  const handleMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xd4af37,
    roughness: 0.05,
    metalness: 0.95
  })
  const handle = new THREE.Mesh(handleGeometry, handleMaterial)
  handle.rotation.z = Math.PI / 2
  handle.position.set(-width * 0.45, 1.8, depth/2 + 0.06)
  handle.castShadow = true
  roomGroup.add(handle)
  
  // Plaque de numéro de chambre élégante
  const plaqueGeometry = new THREE.BoxGeometry(0.35, 0.2, 0.04)
  const plaqueMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x1a1a1a,
    roughness: 0.1,
    metalness: 0.9
  })
  const plaque = new THREE.Mesh(plaqueGeometry, plaqueMaterial)
  plaque.position.set(-width * 0.15, 2.4, depth/2 + 0.06)
  plaque.castShadow = true
  roomGroup.add(plaque)
  
  // Numéro doré
  const numberGeometry = new THREE.PlaneGeometry(0.2, 0.1)
  const numberMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xd4af37,
    emissive: 0x332200,
    roughness: 0.1,
    metalness: 0.8
  })
  const numberText = new THREE.Mesh(numberGeometry, numberMaterial)
  numberText.position.set(-width * 0.15, 2.4, depth/2 + 0.08)
  roomGroup.add(numberText)
}

function createPanoramicWindows(roomGroup: THREE.Group, width: number, depth: number) {
  // Grande fenêtre panoramique
  const windowFrameGeometry = new THREE.BoxGeometry(1.8, 2.0, 0.15)
  const windowFrameMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xf8f6f0,
    roughness: 0.3,
    metalness: 0.1
  })
  const windowFrame = new THREE.Mesh(windowFrameGeometry, windowFrameMaterial)
  windowFrame.position.set(width * 0.1, 2.5, -depth/2 + 0.05)
  windowFrame.castShadow = true
  windowFrame.receiveShadow = true
  roomGroup.add(windowFrame)
  
  // Verre avec effet réaliste
  const windowGlassGeometry = new THREE.BoxGeometry(1.7, 1.9, 0.05)
  const windowGlassMaterial = new THREE.MeshStandardMaterial({ 
    color: 0x87ceeb,
    transparent: true,
    opacity: 0.3,
    roughness: 0.0,
    metalness: 0.0,
    transmission: 0.95,
    thickness: 0.1,
    envMapIntensity: 1.0
  })
  const windowGlass = new THREE.Mesh(windowGlassGeometry, windowGlassMaterial)
  windowGlass.position.set(width * 0.1, 2.5, -depth/2 + 0.02)
  roomGroup.add(windowGlass)
  
  // Séparateurs de fenêtre
  const dividerMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xe8e5dc,
    roughness: 0.5
  })
  
  // Séparateur horizontal
  const hDividerGeometry = new THREE.BoxGeometry(1.7, 0.04, 0.03)
  const hDivider = new THREE.Mesh(hDividerGeometry, dividerMaterial)
  hDivider.position.set(width * 0.1, 2.5, -depth/2 + 0.025)
  roomGroup.add(hDivider)
  
  // Séparateur vertical
  const vDividerGeometry = new THREE.BoxGeometry(0.04, 1.9, 0.03)
  const vDivider = new THREE.Mesh(vDividerGeometry, dividerMaterial)
  vDivider.position.set(width * 0.1, 2.5, -depth/2 + 0.025)
  roomGroup.add(vDivider)
}

function createAmbientRoomLighting(roomGroup: THREE.Group, width: number, depth: number) {
  // Plafonnier central design
  const lightFixtureGeometry = new THREE.CylinderGeometry(0.2, 0.2, 0.06, 20)
  const lightFixtureMaterial = new THREE.MeshStandardMaterial({ 
    color: 0xffffff,
    emissive: 0x666666,
    roughness: 0.1,
    metalness: 0.9
  })
  const lightFixture = new THREE.Mesh(lightFixtureGeometry, lightFixtureMaterial)
  lightFixture.position.set(0, 4.2, 0)
  lightFixture.castShadow = true
  roomGroup.add(lightFixture)
  
  // Éclairage ambiant de la chambre
  const roomLight = new THREE.PointLight(0xfff8dc, 0.6, 20)
  roomLight.position.set(0, 4.0, 0)
  roomLight.castShadow = true
  roomLight.shadow.mapSize.width = 2048
  roomLight.shadow.mapSize.height = 2048
  roomGroup.add(roomLight)
  
  // Éclairage LED d'accentuation
  const accentLight = new THREE.RectAreaLight(0x4a90e2, 2, width * 0.8, 0.1)
  accentLight.position.set(0, 3.8, -depth/2 + 0.1)
  accentLight.lookAt(0, 0, 0)
  roomGroup.add(accentLight)
}

function toggleLighting() {
  nightMode.value = !nightMode.value
  
  // Modifier l'intensité des lumières selon le mode
  scene.traverse((object) => {
    if (object instanceof THREE.DirectionalLight || object instanceof THREE.PointLight || object instanceof THREE.SpotLight) {
      object.intensity = nightMode.value ? object.intensity * 0.3 : object.intensity / 0.3
    }
  })
  
  // Modifier la couleur d'arrière-plan
  scene.background = new THREE.Color(nightMode.value ? 0x040404 : 0x0a0a0a)
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

  // Reset des highlights
  roomMeshes.forEach(roomGroup => {
    roomGroup.children.forEach(child => {
      if (child instanceof THREE.Mesh && child.material instanceof THREE.MeshStandardMaterial) {
        child.material.emissive.setHex(0x000000)
      }
    })
  })

  // Highlight de la chambre survolée
  for (const intersect of intersects) {
    let object = intersect.object
    while (object.parent && !object.userData?.room) {
      object = object.parent
    }
    
    if (object.userData?.room) {
      const roomGroup = object as THREE.Group
      roomGroup.children.forEach(child => {
        if (child instanceof THREE.Mesh && child.material instanceof THREE.MeshStandardMaterial) {
          child.material.emissive.setHex(0x111111)
        }
      })
      threeContainer.value!.style.cursor = 'pointer'
      return
    }
  }
  
  threeContainer.value!.style.cursor = 'default'
}

function selectRoom(room: Room) {
  // Highlight de la chambre sélectionnée
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
  camera.position.set(25, 30, 25)
  camera.lookAt(0, 0, 0)
}

function onWindowResize() {
  if (!threeContainer.value) return
  
  camera.aspect = threeContainer.value.clientWidth / threeContainer.value.clientHeight
  camera.updateProjectionMatrix()
  renderer.setSize(threeContainer.value.clientWidth, threeContainer.value.clientHeight)
}

function animate() {
  animationId = requestAnimationFrame(animate)
  renderer.render(scene, camera)
}

// Fonctions utilitaires pour l'UI
function getBadgeStyle(index: number) {
  if (index >= roomPositions.length) return { display: 'none' }
  
  const [x, z] = roomPositions[index]
  
  // Conversion de la position 3D vers l'écran
  const vector = new THREE.Vector3(x, 5, z)
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
    available: 'bg-gradient-to-r from-emerald-500 to-teal-600 hover:from-emerald-600 hover:to-teal-700',
    occupied: 'bg-gradient-to-r from-red-500 to-rose-600 hover:from-red-600 hover:to-rose-700',
    maintenance: 'bg-gradient-to-r from-amber-500 to-orange-600 hover:from-amber-600 hover:to-orange-700',
    cleaning: 'bg-gradient-to-r from-blue-500 to-indigo-600 hover:from-blue-600 hover:to-indigo-700'
  }
  return classes[status as keyof typeof classes] || 'bg-gradient-to-r from-emerald-500 to-teal-600'
}

function getStatusText(status: string): string {
  const texts = {
    available: 'Disponible',
    occupied: 'Occupée',
    maintenance: 'Maintenance',
    cleaning: 'Nettoyage'
  }
  return texts[status as keyof typeof texts] || 'Disponible'
}

// Lifecycle
onMounted(async () => {
  await nextTick()
  initThreeJS()
})

onBeforeUnmount(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  
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
