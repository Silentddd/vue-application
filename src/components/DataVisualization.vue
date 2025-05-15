<template>
  <div class="visualization-wrapper">
    <div class="controls">
      <button @click="changeVisualizationType('point')">散点图</button>
      <button @click="changeVisualizationType('surface')">曲面图</button>
      <button @click="changeVisualizationType('bar')">柱状图</button>
      <button @click="changeVisualizationType('product')">产品展示</button>
    </div>
    <div ref="container" class="visualization-container"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const container = ref(null)
let scene, camera, renderer, controls, labelRenderer

function changeVisualizationType(type) {
  addDataVisualization(type)
}

onMounted(() => {
  // 初始化3D场景
  initThreeJS()
  // 添加3D数据可视化
  addDataVisualization()
  // 开始渲染循环
  animate()
})

onUnmounted(() => {
  // 清理资源
  if (renderer) {
    renderer.dispose()
  }
})

function initThreeJS() {
  // 创建场景
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x222222)
  
  // 创建相机
  camera = new THREE.PerspectiveCamera(
    75,
    container.value.clientWidth / container.value.clientHeight,
    0.1,
    1000
  )
  camera.position.z = 5
  
  // 创建WebGL渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(container.value.clientWidth, container.value.clientHeight)
  container.value.appendChild(renderer.domElement)
  
  // 创建CSS2D标签渲染器
  labelRenderer = new CSS2DRenderer()
  labelRenderer.setSize(container.value.clientWidth, container.value.clientHeight)
  labelRenderer.domElement.style.position = 'absolute'
  labelRenderer.domElement.style.top = '0'
  labelRenderer.domElement.style.pointerEvents = 'none'
  container.value.appendChild(labelRenderer.domElement)
  
  // 添加轨道控制器
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  
  // 处理窗口大小变化
  window.addEventListener('resize', onWindowResize)
}

function generateRandomData(count = 1000) {
  const data = []
  for (let i = 0; i < count; i++) {
    data.push({
      x: Math.random() * 10 - 5,
      y: Math.random() * 10 - 5,
      z: Math.random() * 10 - 5,
      value: Math.random()
    })
  }
  return data
}

function generateSurfaceData() {
  const data = []
  const size = 20
  for (let x = -5; x < 5; x += 0.5) {
    for (let z = -5; z < 5; z += 0.5) {
      const y = Math.sin(x) * Math.cos(z)
      data.push({
        x: x,
        y: y,
        z: z,
        value: (y + 1) / 2
      })
    }
  }
  return data
}

function generateBarData() {
  const data = []
  for (let x = -4; x <= 4; x += 1) {
    for (let z = -4; z <= 4; z += 1) {
      const height = Math.random() * 2
      data.push({
        x: x,
        y: height / 2,
        z: z,
        value: height
      })
    }
  }
  return data
}

function createPointCloud(data) {
  const geometry = new THREE.BufferGeometry()
  const positions = []
  const colors = []
  const group = new THREE.Group()
  
  data.forEach(point => {
    positions.push(point.x, point.y, point.z)
    const color = new THREE.Color()
    color.setHSL(0.6 * point.value, 1.0, 0.5)
    colors.push(color.r, color.g, color.b)
    
    // 创建数据标签
    const labelDiv = document.createElement('div')
    labelDiv.className = 'data-label'
    labelDiv.textContent = point.value.toFixed(2)
    labelDiv.style.color = `rgb(${Math.floor(color.r*255)}, ${Math.floor(color.g*255)}, ${Math.floor(color.b*255)})`
    const label = new CSS2DObject(labelDiv)
    label.position.set(point.x, point.y, point.z)
    group.add(label)
  })
  
  geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
  geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3))
  
  const material = new THREE.PointsMaterial({
    size: 0.1,
    vertexColors: true,
    transparent: true,
    opacity: 0.8
  })
  
  const points = new THREE.Points(geometry, material)
  group.add(points)
  return group
}

function createSurfaceMesh(data) {
  const geometry = new THREE.BufferGeometry()
  const positions = []
  const colors = []
  
  data.forEach(point => {
    positions.push(point.x, point.y, point.z)
    const color = new THREE.Color()
    color.setHSL(0.6 * point.value, 1.0, 0.5)
    colors.push(color.r, color.g, color.b)
  })
  
  geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
  geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3))
  
  const material = new THREE.MeshBasicMaterial({
    vertexColors: true,
    wireframe: false,
    side: THREE.DoubleSide
  })
  
  return new THREE.Mesh(geometry, material)
}

function createBarChart(data) {
  const group = new THREE.Group()
  
  data.forEach(point => {
    const geometry = new THREE.BoxGeometry(0.8, point.value, 0.8)
    const color = new THREE.Color().setHSL(0.6 * point.value, 1.0, 0.5)
    const material = new THREE.MeshBasicMaterial({ color: color })
    const bar = new THREE.Mesh(geometry, material)
    bar.position.set(point.x, point.y, point.z)
    group.add(bar)
    
    // 创建数据标签
    const labelDiv = document.createElement('div')
    labelDiv.className = 'data-label'
    labelDiv.textContent = point.value.toFixed(2)
    labelDiv.style.color = `rgb(${Math.floor(color.r*255)}, ${Math.floor(color.g*255)}, ${Math.floor(color.b*255)})`
    const label = new CSS2DObject(labelDiv)
    label.position.set(point.x, point.y + point.value/2 + 0.2, point.z)
    group.add(label)
  })
  
  return group
}

function createProductModel() {
  const group = new THREE.Group()
  const loader = new GLTFLoader()
  
  // 这里需要添加产品模型文件路径
  loader.load(
    'models/product.glb',
    (gltf) => {
      const model = gltf.scene
      model.scale.set(0.5, 0.5, 0.5)
      model.position.set(0, 0, 0)
      group.add(model)
      
      // 添加产品信息标签
      const labelDiv = document.createElement('div')
      labelDiv.className = 'product-label'
      labelDiv.textContent = '产品名称'
      labelDiv.style.color = 'white'
      const label = new CSS2DObject(labelDiv)
      label.position.set(0, 1.5, 0)
      group.add(label)
    },
    undefined,
    (error) => {
      console.error('加载模型出错:', error)
    }
  )
  
  return group
}

function addDataVisualization(type = 'point') {
  // 清除场景中的现有对象
  while(scene.children.length > 0){ 
    scene.remove(scene.children[0]); 
  }
  
  let visualization
  
  switch(type) {
    case 'point':
      const pointData = generateRandomData(2000)
      visualization = createPointCloud(pointData)
      break
    case 'surface':
      const surfaceData = generateSurfaceData()
      visualization = createSurfaceMesh(surfaceData)
      break
    case 'bar':
      const barData = generateBarData()
      visualization = createBarChart(barData)
      break
    case 'product':
      visualization = createProductModel()
      break
    default:
      const defaultData = generateRandomData(2000)
      visualization = createPointCloud(defaultData)
  }
  
  scene.add(visualization)
}

function onWindowResize() {
  camera.aspect = container.value.clientWidth / container.value.clientHeight
  camera.updateProjectionMatrix()
  renderer.setSize(container.value.clientWidth, container.value.clientHeight)
  labelRenderer.setSize(container.value.clientWidth, container.value.clientHeight)
}

function animate() {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
  labelRenderer.render(scene, camera)
}
</script>

<style scoped>
.visualization-wrapper {
  position: relative;
  width: 100%;
  height: 100vh;
}

.controls {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 100;
  display: flex;
  gap: 10px;
}

.controls button {
  padding: 8px 16px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.controls button:hover {
  background: rgba(0, 0, 0, 0.9);
}

.visualization-container {
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.data-label {
  font-size: 12px;
  font-weight: bold;
  text-shadow: 0 0 3px #000;
  transform: translateY(-50%);
  white-space: nowrap;
}

.product-label {
  font-size: 16px;
  font-weight: bold;
  text-shadow: 0 0 5px #000;
  transform: translateY(-50%);
  white-space: nowrap;
  padding: 5px 10px;
  background: rgba(0, 0, 0, 0.5);
  border-radius: 4px;
}
</style>