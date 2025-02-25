<script setup>
import * as THREE from 'three'
import { onMounted, ref } from 'vue'

const canvasRef = ref(null)
let camera, scene, renderer
let planes = []
let isDragging = false
let previousMousePosition = { x: 0, y: 0 }
let zoomFactor = 1

onMounted(() => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xE9E9E9)

  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.z = 5

  renderer = new THREE.WebGLRenderer()
  renderer.setSize(window.innerWidth, window.innerHeight)
  canvasRef.value.appendChild(renderer.domElement)

  // Create 10 planes
  for (let i = 0; i < 10; i++) {
    const geometry = new THREE.PlaneGeometry(1, 1)
    const material = new THREE.MeshBasicMaterial({ color: Math.random() * 0xffffff })
    const plane = new THREE.Mesh(geometry, material)
    plane.position.set(Math.random() * 10 - 5, Math.random() * 10 - 5, Math.random() * 10 - 5)
    planes.push(plane)
    scene.add(plane)
  }

  const animate = function () {
    requestAnimationFrame(animate)
    renderer.render(scene, camera)
  }

  animate()

  // Add event listeners for drag and zoom
  canvasRef.value.addEventListener('mousedown', onMouseDown, false)
  canvasRef.value.addEventListener('mousemove', onMouseMove, false)
  canvasRef.value.addEventListener('mouseup', onMouseUp, false)
  canvasRef.value.addEventListener('wheel', onMouseWheel, false)
  canvasRef.value.addEventListener('touchstart', onTouchStart, false)
  canvasRef.value.addEventListener('touchmove', onTouchMove, false)
  canvasRef.value.addEventListener('touchend', onTouchEnd, false)
})

function onMouseDown(event) {
  isDragging = true
  previousMousePosition = {
    x: event.clientX,
    y: event.clientY
  }
}

function onMouseMove(event) {
  if (isDragging) {
    const deltaMove = {
      x: event.clientX - previousMousePosition.x,
      y: event.clientY - previousMousePosition.y
    }

    camera.position.x -= deltaMove.x * 0.01
    camera.position.y += deltaMove.y * 0.01

    previousMousePosition = {
      x: event.clientX,
      y: event.clientY
    }
  }
}

function onMouseUp() {
  isDragging = false
}

function onMouseWheel(event) {
  zoomFactor += event.deltaY * 0.001 // Juster denne værdi for at ændre zoomhastigheden
  zoomFactor = Math.max(0.1, Math.min(10, zoomFactor)) // Juster grænserne for zoomfaktoren
  camera.zoom = zoomFactor
  camera.updateProjectionMatrix()
}

function onTouchStart(event) {
  if (event.touches.length === 1) {
    isDragging = true
    previousMousePosition = {
      x: event.touches[0].clientX,
      y: event.touches[0].clientY
    }
  }
}

function onTouchMove(event) {
  if (isDragging && event.touches.length === 1) {
    const deltaMove = {
      x: event.touches[0].clientX - previousMousePosition.x,
      y: event.touches[0].clientY - previousMousePosition.y
    }

    camera.position.x -= deltaMove.x * 0.01
    camera.position.y += deltaMove.y * 0.01

    previousMousePosition = {
      x: event.touches[0].clientX,
      y: event.touches[0].clientY
    }
  }
}

function onTouchEnd() {
  isDragging = false
}
</script>

<template>
  <div ref="canvasRef" style="width: 100vw; height: 100vh;"></div>
</template>

<style scoped>
/* Tilføj eventuelle nødvendige stilarter her */
</style>
