<script setup>
import '../assets/main.css'
import * as THREE from 'three'
import { onMounted, ref } from 'vue'
import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js'
import PoissonDiskSampling from 'poisson-disk-sampling'
import { gsap } from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";

gsap.registerPlugin(ScrollTrigger);

const canvasRef = ref(null);
let camera, scene, renderer, labelRenderer;
let isDragging = false;
let previousMousePosition = { x: 0, y: 0 };
let zoomFactor = 1;
let initialDistance = 0;
let isPinching = false;
const names = ref([]);

async function loadNames() {
  try {
    const response = await fetch('/names.json');
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
    names.value = await response.json();
  } catch (error) {
    console.error('Failed to load names:', error);
  }
}

function latLonToXY(lat, lon, scale) {
  return { x: lon * (scale / 180), y: lat * (scale / 90) };
}

function createTextLabel(text, x, y, z) {
  const div = document.createElement('div');
  div.className = 'label';
  div.style.fontSize = `${Math.max(10, Math.min(24, 12 * zoomFactor))}px`;
  div.textContent = text;
  div.style.whiteSpace = 'nowrap';
  const label = new CSS2DObject(div);
  label.position.set(x, y, z);
  scene.add(label);
}

function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
  labelRenderer.render(scene, camera);
}

onMounted(async () => {
  await loadNames();

  scene = new THREE.Scene();
  scene.background = new THREE.Color(0xE9E9E9);
  
  camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 10;

  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  canvasRef.value.appendChild(renderer.domElement);

  labelRenderer = new CSS2DRenderer();
  labelRenderer.setSize(window.innerWidth, window.innerHeight);
  labelRenderer.domElement.style.position = 'absolute';
  labelRenderer.domElement.style.top = '0px';
  canvasRef.value.appendChild(labelRenderer.domElement);

  try {
    const response = await fetch('/map.geojson');
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
    const worldGeoJSON = await response.json();

    const scale = 8;
    worldGeoJSON.features?.forEach(feature => {
      if (feature.geometry.type === "Polygon" || feature.geometry.type === "MultiPolygon") {
        (feature.geometry.type === "MultiPolygon" ? feature.geometry.coordinates.flat() : feature.geometry.coordinates).forEach(ring => {
          const numPoints = 10;
          ring.forEach(([lon, lat], index) => {
            const { x, y } = latLonToXY(lat, lon, scale);
            if (names.value.length > 0) createTextLabel(names.value[index % names.value.length], x, y, -Math.random() * 6);
          });
        });
      }
    });
  } catch (error) {
    console.error('Failed to load GeoJSON:', error);
  }

  gsap.utils.toArray('.par').forEach(img => {
    gsap.to(img, {
      y: '-100%',
      ease: 'none',
      scrollTrigger: {
        trigger: img,
        start: 'top bottom',
        end: 'bottom top',
        scrub: 2,
      }
    });
  });

  animate();

  window.addEventListener('resize', onWindowResize);
  canvasRef.value.addEventListener('mousedown', onMouseDown);
  canvasRef.value.addEventListener('mousemove', onMouseMove);
  canvasRef.value.addEventListener('mouseup', onMouseUp);
  canvasRef.value.addEventListener('wheel', onMouseWheel);
  canvasRef.value.addEventListener('touchstart', onTouchStart);
  canvasRef.value.addEventListener('touchmove', onTouchMove);
  canvasRef.value.addEventListener('touchend', onTouchEnd);
});

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  labelRenderer.setSize(window.innerWidth, window.innerHeight);
}

function onMouseDown(event) {
  isDragging = true;
  previousMousePosition = { x: event.clientX, y: event.clientY };
}

function onMouseMove(event) {
  if (isDragging) {
    const moveSpeed = 0.1 / zoomFactor; // Justér bevægelse efter zoom
    camera.position.x -= (event.clientX - previousMousePosition.x) * moveSpeed;
    camera.position.y += (event.clientY - previousMousePosition.y) * moveSpeed;
    previousMousePosition = { x: event.clientX, y: event.clientY };
  }
}

function onMouseUp() { isDragging = false; }

function onMouseWheel(event) {
  event.preventDefault(); // Forhindrer standard scroll-adfærd
  zoomFactor += event.deltaY * 0.001;
  zoomFactor = Math.max(0.001, Math.min(100, zoomFactor));
  camera.zoom = zoomFactor;
  camera.updateProjectionMatrix();

  // Re-create labels with adjusted font size
  scene.children.forEach(child => {
    if (child instanceof CSS2DObject) {
      const div = child.element;
      const baseFontSize = 12;
      const adjustedFontSize = baseFontSize * zoomFactor;
      const fontSize = Math.max(10, Math.min(24, adjustedFontSize));
      div.style.fontSize = `${fontSize}px`;
    }
  });
}

// Touch event functions for spread zoom
function onTouchStart(event) {
  if (event.touches.length === 2) {
    isPinching = true;
    initialDistance = getDistanceBetweenTouches(event);
  }
}

function onTouchMove(event) {
  if (isPinching && event.touches.length === 2) {
    const currentDistance = getDistanceBetweenTouches(event);
    const zoomChange = (currentDistance - initialDistance) * 0.05; // Adjust multiplier for speed
    zoomFactor -= zoomChange; // Invert the zoom change for spread zoom
    zoomFactor = Math.max(0.001, Math.min(100, zoomFactor));

    // Update zoom by adjusting the camera's position.z property
    camera.position.z = 10 / zoomFactor;
    camera.updateProjectionMatrix();

    // Re-create labels with adjusted font size
    scene.children.forEach(child => {
      if (child instanceof CSS2DObject) {
        const div = child.element;
        const baseFontSize = 12;
        const adjustedFontSize = baseFontSize * zoomFactor;
        const fontSize = Math.max(10, Math.min(24, adjustedFontSize));
        div.style.fontSize = `${fontSize}px`;
      }
    });

    initialDistance = currentDistance;
    event.preventDefault(); // Prevent default pinch-zoom behavior
  }
}

function onTouchEnd() {
  isPinching = false;
}

function getDistanceBetweenTouches(event) {
  const dx = event.touches[0].clientX - event.touches[1].clientX;
  const dy = event.touches[0].clientY - event.touches[1].clientY;
  return Math.sqrt(dx * dx + dy * dy);
}
</script>

<template>
  <div class="intro">
    <h1 class="title">COMMON</h1>
    <h1 class="title">GROUND</h1>
  </div>

  <div class="section1">
    <h1>In a world where people are called “<span class="bad">Aliens</span>” and “<span class="bad">Rapists & criminals</span>” 
      We need to find common ground.</h1>
      <img class="blurr one par" src="/public/image 2.png" alt="">
      <img class="blurr two par" src="/public/image 3.png" alt="">
      <img class="blurr three par" src="/public/image 4.png" alt="">
      <img class="blurr four par" src="/public/image 5.png" alt="">
  </div>
  <div class="section1">
    <h1>We need to remember we are all descended from immgrants.
      We need to find common ground.</h1>
      <img class="us six par" src="/public/image 6.png" alt="">
      <img class="us seven par" src="/public/image 7.png" alt="">
      <img class="us eight par" src="/public/image 8.png" alt="">
      <img class="us nine par" src="/public/image 9.png" alt="">
      <img class="us ten par" src="/public/image 10.png" alt="">
  </div>
  <div class="section1">
    <h1>All of us.</h1>
  </div>
  <div class="relative" ref="canvasRef" style="width: 100vw; height: 100vh;"></div>
</template>

<style scoped>
@font-face {
  font-family: 'MyFontReg';
  src: url('/fonts/SansControlRegular.otf') format('truetype');
}
@font-face {
  font-family: 'MyFontScrib';
  src: url('/fonts/SansControlScribbled.otf') format('truetype');
}

.label {
  position: absolute;
  font-family: 'MyFontReg', sans-serif;  
  padding: 2px 5px;
  border-radius: 3px;
  font-size: 12px;
  transition: color 0.3s; /* Tilføj en transition for en glidende effekt */
}

.label:hover {
  color: red; /* Ændr denne farve til den ønskede hover-farve */
}
</style>
