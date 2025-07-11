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
        class="bg-white/90 backdrop-blur-sm hover:bg-white text-gray-700 px-3 py-2 rounded-lg shadow-lg transition-all duration-300 text-xl font-bold"
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
let targetHoverScale = 1
let currentHoverScale = 1
const hoverAnimationSpeed = 0.15 // Increased speed for more responsiveness
let roomAnimations: Map<string, { targetScale: number, currentScale: number }> = new Map()

// Zoom settings
let zoomLevel = 1
const minZoom = 0.5
const maxZoom = 3
const zoomStep = 0.2

// Hotel rooms data
const hotelRooms = [
  { id: '10', number: '10', status: 'available', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], position: { x: 6, z: -3 } },
  { id: '11', number: '11', status: 'occupied', type: 'deluxe', price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: 6, z: 0 } },
  { id: '12', number: '12', status: 'available', type: 'suite', price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: 3, z: -6 } },
  { id: '13', number: '13', status: 'occupied', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], position: { x: 0, z: -6 } },
  { id: '14', number: '14', status: 'maintenance', type: 'deluxe', price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: -3, z: -6 } },
  { id: '15', number: '15', status: 'cleaning', type: 'suite', price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: -6, z: -3 } },
  { id: '16', number: '16', status: 'occupied', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], position: { x: -6, z: 0 } },
  { id: '17', number: '17', status: 'occupied', type: 'deluxe', price: 150, capacity: 2, amenities: ['wifi', 'tv', 'minibar'], position: { x: -6, z: 3 } },
  { id: '18', number: '18', status: 'occupied', type: 'suite', price: 200, capacity: 4, amenities: ['wifi', 'tv', 'minibar', 'balcony'], position: { x: -3, z: 6 } },
  { id: '19', number: '19', status: 'cleaning', type: 'standard', price: 120, capacity: 2, amenities: ['wifi', 'tv'], position: { x: 0, z: 6 } },
]

function resetCamera() {
  zoomLevel = 1
  updateCameraZoom()
  camera.position.set(20, 20, 20)
  camera.lookAt(0, 0, 0)
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
  const frustumSize = 25 / zoomLevel
  camera.left = -frustumSize * aspect / 2
  camera.right = frustumSize * aspect / 2
  camera.top = frustumSize / 2
  camera.bottom = -frustumSize / 2
  camera.updateProjectionMatrix()
}

function getStatusColor(status: string): number {
  const colors = {
    available: 0x4ade80,  // Green
    occupied: 0xef4444,   // Red
    maintenance: 0xf97316, // Orange
    cleaning: 0x3b82f6    // Blue
  }
  return colors[status as keyof typeof colors] || 0x6b7280
}

function createHotelFloor() {
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
  const roomMat = new THREE.MeshLambertMaterial({ color: getStatusColor(roomData.status) })
  const roomMesh = new THREE.Mesh(roomGeom, roomMat)
  roomMesh.position.set(0, 0.05, 0)
  roomMesh.userData = { roomId: roomData.id, roomData }
  roomMesh.name = `room-floor-${roomData.id}`
  roomGroup.add(roomMesh)

  // Walls with 70% opacity and status color
  const wallColor = getStatusColor(roomData.status)
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
  canvas.width = 128
  canvas.height = 64
  
  context.fillStyle = '#ffffff'
  context.fillRect(0, 0, 128, 64)
  context.fillStyle = '#000000'
  context.font = 'bold 24px Arial'
  context.textAlign = 'center'
  context.fillText(roomData.number, 64, 40)
  
  const texture = new THREE.CanvasTexture(canvas)
  const labelMat = new THREE.MeshBasicMaterial({ map: texture, transparent: true })
  const labelGeom = new THREE.PlaneGeometry(1, 0.5)
  const label = new THREE.Mesh(labelGeom, labelMat)
  label.position.set(roomData.position.x, 2.5, roomData.position.z)
  label.lookAt(camera.position)
  label.userData = { roomId: roomData.id, roomData }
  scene.add(label)
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
  
  // Get all objects in the scene that can be clicked
  const allClickableObjects: THREE.Object3D[] = []
  scene.traverse((child) => {
    if (child.userData && child.userData.roomId) {
      allClickableObjects.push(child)
    }
  })
  
  const intersects = raycaster.intersectObjects(allClickableObjects, true)

  // Reset previous selection
  if (selectedRoom) {
    selectedRoom.children.forEach(child => {
      if (child instanceof THREE.Mesh && child.material && !child.name?.includes('clickarea')) {
        const material = child.material as THREE.MeshLambertMaterial
        if (child.name?.includes('wall')) {
          material.opacity = 0.7 // Reset wall opacity
        }
        material.emissive.setHex(0x000000) // Reset glow
      }
    })
    selectedRoom = null
  }

  if (intersects.length > 0) {
    // Find the first intersected object with room data
    for (const intersect of intersects) {
      if (intersect.object.userData && intersect.object.userData.roomId) {
        const roomData = intersect.object.userData.roomData
        const roomId = intersect.object.userData.roomId
        
        if (roomMeshes.has(roomId)) {
          selectedRoom = roomMeshes.get(roomId)!
          
          // Apply selection effect
          selectedRoom.children.forEach(child => {
            if (child instanceof THREE.Mesh && child.material && !child.name?.includes('clickarea')) {
              const material = child.material as THREE.MeshLambertMaterial
              if (child.name?.includes('wall')) {
                material.opacity = 0.9 // Increase wall opacity for selected room
              }
              material.emissive.setHex(0x333333) // Add selection glow
            }
          })
          
          console.log('Room selected:', roomData)
          emit('selectRoom', roomData)
          break
        }
      }
    }
  } else {
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
    }
    hoveredRoom = null
  }

  // Reset target scale when not hovering
  if (!hoveredRoom) {
    targetHoverScale = 1
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
  
  renderer.render(scene, camera)
}

function handleResize() {
  if (!renderer || !camera || !threeContainer.value) return
  updateCameraZoom()
  renderer.setSize(threeContainer.value.clientWidth, threeContainer.value.clientHeight)
}

function handleWheel(event: WheelEvent) {
  event.preventDefault()
  
  if (event.deltaY < 0) {
    // Scroll up - zoom in
    zoomIn()
  } else {
    // Scroll down - zoom out
    zoomOut()
  }
}

// Lifecycle
onMounted(async () => {
  await nextTick()
  if (typeof window !== 'undefined' && threeContainer.value) {
    scene = new THREE.Scene()
    scene.background = new THREE.Color(0xf5f5f5)
    
    const aspect = window.innerWidth / window.innerHeight
    const frustumSize = 25
    camera = new THREE.OrthographicCamera(
      -frustumSize * aspect / 2, 
      frustumSize * aspect / 2,
      frustumSize / 2,
      -frustumSize / 2,
      1,
      1000
    )
    camera.position.set(20, 20, 20)
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
