<script setup>
import { onMounted, ref, onUnmounted } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader'

const container = ref(null)
const loadingProgress = ref(0)
let scene, camera, renderer, controls
let currentModel = null
let isHighQualityLoaded = false
let isTransitioning = false
let transitionProgress = 0
const TRANSITION_DURATION = 8000 // افزایش مدت زمان transition به 8 ثانیه
let startTime = 0
let initialCameraPosition = new THREE.Vector3(0, 50, 1600) // موقعیت اولیه دوربین
let finalCameraPosition = new THREE.Vector3(0, 50, 100) // موقعیت نهایی دوربین

onMounted(() => {
  // ایجاد صحنه
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x000000)

  // تنظیم دوربین
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.copy(initialCameraPosition)
  camera.lookAt(0, 0, 0)

  // تنظیم رندرر
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.outputColorSpace = THREE.SRGBColorSpace
  container.value.appendChild(renderer.domElement)

  // نور محیطی برای روشنایی کلی صحنه
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4)
  scene.add(ambientLight)

  // نور جهت‌دار اصلی
  const dirLight = new THREE.DirectionalLight(0xfff2cc, 2.5)
  dirLight.position.set(-100, 120, 0)
  dirLight.castShadow = true
  dirLight.shadow.mapSize.width = 2048
  dirLight.shadow.mapSize.height = 2048
  dirLight.shadow.radius = 8
  dirLight.shadow.bias = -0.001
  dirLight.shadow.camera.near = 0.1
  dirLight.shadow.camera.far = 500
  dirLight.shadow.camera.left = -100
  dirLight.shadow.camera.right = 100
  dirLight.shadow.camera.top = 100
  dirLight.shadow.camera.bottom = -100
  scene.add(dirLight)

  // نور حاشیه‌ای برای ایجاد کنتراست بیشتر
  const rimLight = new THREE.DirectionalLight(0x000000, 1.2)
  rimLight.position.set(50, 30, 50)
  scene.add(rimLight)

  // تنظیمات OrbitControls با damping نرم‌تر
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.enabled = false // غیرفعال کردن کنترل‌ها در ابتدا

  // لود HDR
  new RGBELoader().setPath('/hdr/').load('lakeside_dawn_1k.hdr', (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping
    scene.environment = texture
    scene.background = new THREE.Color(0x000000)

    // لود اولیه مدل با کیفیت پایین
    loadLowQualityModel()
  })

  // شروع انیمیشن
  animate()

  // اضافه کردن event listener برای تغییر سایز پنجره
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
  })
})

// تابع easing برای حرکت نرم‌تر
function smoothEase(t) {
  // استفاده از منحنی Bezier برای حرکت نرم‌تر
  return t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2
}

// تابع انیمیشن
function animate() {
  requestAnimationFrame(animate)
  controls.update()

  if (isTransitioning) {
    const currentTime = Date.now()
    const elapsed = currentTime - startTime
    transitionProgress = Math.min(elapsed / TRANSITION_DURATION, 1)

    // محاسبه موقعیت جدید با easing نرم
    const easedProgress = smoothEase(transitionProgress)

    // به‌روزرسانی موقعیت دوربین به صورت مستقیم
    camera.position.x =
      initialCameraPosition.x + (finalCameraPosition.x - initialCameraPosition.x) * easedProgress
    camera.position.y =
      initialCameraPosition.y + (finalCameraPosition.y - initialCameraPosition.y) * easedProgress
    camera.position.z =
      initialCameraPosition.z + (finalCameraPosition.z - initialCameraPosition.z) * easedProgress

    camera.lookAt(0, 0, 0)

    if (transitionProgress >= 1) {
      isTransitioning = false
      controls.enabled = true
    }
  }

  renderer.render(scene, camera)
}

// تابع لود مدل با کیفیت پایین
function loadLowQualityModel() {
  const loader = new GLTFLoader()
  const dracoLoader = new DRACOLoader()
  dracoLoader.setDecoderPath('/draco/')
  loader.setDRACOLoader(dracoLoader)

  // تنظیمات برای کیفیت پایین
  dracoLoader.setDecoderConfig({ type: 'js' })
  dracoLoader.setWorkerLimit(2)

  // غیرفعال کردن سایه‌ها در مرحله اول
  renderer.shadowMap.enabled = false

  loader.load(
    '/model/scene.gltf',
    (gltf) => {
      console.log('✅ مدل با کیفیت پایین لود شد')
      const model = gltf.scene
      const maxAnisotropy = renderer.capabilities.getMaxAnisotropy()

      // تنظیمات اولیه برای مدل با کیفیت پایین
      model.traverse((child) => {
        if (child.isMesh) {
          child.castShadow = false
          child.receiveShadow = false

          if (child.material) {
            if (child.material.map) {
              child.material.map.minFilter = THREE.LinearMipmapLinearFilter
              child.material.map.magFilter = THREE.LinearFilter
              child.material.map.generateMipmaps = true
              child.material.map.anisotropy = Math.min(4, maxAnisotropy)
            }

            const standardMaterial = new THREE.MeshStandardMaterial({
              map: child.material.map,
              color: child.material.color,
              roughness: 0.7,
              metalness: 0.2,
              envMapIntensity: 1.0,
              transparent: child.material.transparent,
              opacity: child.material.opacity,
            })
            child.material = standardMaterial

            if (Array.isArray(child.material)) {
              child.material = child.material.map((mat) => {
                const standardMaterial = new THREE.MeshStandardMaterial({
                  map: mat.map,
                  color: mat.color,
                  roughness: 0.7,
                  metalness: 0.2,
                  envMapIntensity: 1.0,
                  transparent: mat.transparent,
                  opacity: mat.opacity,
                })
                return standardMaterial
              })
            }
          }
        }
      })

      const box = new THREE.Box3().setFromObject(model)
      const center = box.getCenter(new THREE.Vector3())
      const size = box.getSize(new THREE.Vector3())
      model.position.sub(center)
      const maxDim = Math.max(size.x, size.y, size.z)
      const scale = 100 / maxDim
      model.scale.multiplyScalar(scale)

      scene.add(model)
      currentModel = model
      loadingProgress.value = 50

      // شروع انیمیشن دوربین
      isTransitioning = true
      startTime = Date.now()
      transitionProgress = 0

      // شروع لود مدل با کیفیت بالا
      setTimeout(loadHighQualityModel, 3000)
    },
    (xhr) => {
      loadingProgress.value = (xhr.loaded / xhr.total) * 50
    },
    (error) => {
      console.error('خطا در لود مدل با کیفیت پایین:', error)
    },
  )
}

// تابع لود مدل با کیفیت بالا
function loadHighQualityModel() {
  if (isHighQualityLoaded) return

  // فعال کردن سایه‌ها در مرحله دوم
  renderer.shadowMap.enabled = true

  const loader = new GLTFLoader()
  const dracoLoader = new DRACOLoader()
  dracoLoader.setDecoderPath('/draco/')
  loader.setDRACOLoader(dracoLoader)

  // تنظیمات برای کیفیت بالا
  dracoLoader.setDecoderConfig({ type: 'js' })
  dracoLoader.setWorkerLimit(4)

  loader.load(
    '/model/scene.gltf',
    (gltf) => {
      console.log('✅ مدل با کیفیت بالا لود شد')
      const model = gltf.scene
      const maxAnisotropy = renderer.capabilities.getMaxAnisotropy()

      // تنظیمات پیشرفته برای مدل با کیفیت بالا
      model.traverse((child) => {
        if (child.isMesh) {
          // فعال کردن سایه‌ها برای همه مش‌ها
          child.castShadow = true
          child.receiveShadow = true

          if (child.material) {
            if (!(child.material instanceof THREE.MeshStandardMaterial)) {
              const standardMaterial = new THREE.MeshStandardMaterial({
                map: child.material.map,
                color: child.material.color,
                roughness: 0.5,
                metalness: 0.3,
                envMapIntensity: 1.2,
                aoMapIntensity: 1.0,
                normalScale: new THREE.Vector2(1, 1),
              })
              child.material = standardMaterial
            }

            child.material.needsUpdate = true

            if (child.material.map) {
              child.material.map.generateMipmaps = true
              child.material.map.minFilter = THREE.LinearMipmapLinearFilter
              child.material.map.magFilter = THREE.LinearFilter
              child.material.map.encoding = THREE.sRGBEncoding
              child.material.map.anisotropy = maxAnisotropy
            }
          }
        }
      })

      // تنظیم موقعیت و مقیاس مدل
      const box = new THREE.Box3().setFromObject(model)
      const center = box.getCenter(new THREE.Vector3())
      const size = box.getSize(new THREE.Vector3())
      model.position.sub(center)
      const maxDim = Math.max(size.x, size.y, size.z)
      const scale = 100 / maxDim
      model.scale.multiplyScalar(scale)

      // جایگزینی مدل با کیفیت پایین با مدل با کیفیت بالا
      if (currentModel) {
        scene.remove(currentModel)
      }
      scene.add(model)
      currentModel = model
      isHighQualityLoaded = true
      loadingProgress.value = 100

      // مخفی کردن progress bar بعد از 2 ثانیه
      setTimeout(() => {
        loadingProgress.value = 0
      }, 2000)
    },
    (xhr) => {
      loadingProgress.value = 50 + (xhr.loaded / xhr.total) * 50
    },
    (error) => {
      console.error('خطا در لود مدل با کیفیت بالا:', error)
    },
  )
}

onUnmounted(() => {
  if (renderer) {
    renderer.dispose()
  }
})
</script>

<template>
  <div class="container">
    <div ref="container" style="width: 100vw; height: 100vh"></div>
    <div v-if="loadingProgress > 0" class="loading-container">
      <div class="progress-bar">
        <div class="progress" :style="{ width: loadingProgress + '%' }"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  position: relative;
  width: 100vw;
  height: 100vh;
}

div {
  position: fixed;
  top: 0;
  left: 0;
  outline: none;
}

.loading-container {
  position: fixed;
  top: 5px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.8);
  padding: 20px;
  border-radius: 10px;
  color: white;
  z-index: 1000;
  min-width: 100%;
}

.loading-text {
  margin-bottom: 10px;
  text-align: center;
  font-size: 14px;
}

.progress-bar {
  width: 100%;
  height: 4px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 2px;
  overflow: hidden;
}

.progress {
  height: 4px;
  background: white;
  transition: width 0.5s cubic-bezier(0.4, 0, 0.2, 1);
  transform-origin: left;
  animation: progressAnimation 0.5s ease-out;
}

@keyframes progressAnimation {
  from {
    transform: scaleX(0);
  }
  to {
    transform: scaleX(1);
  }
}
</style>
