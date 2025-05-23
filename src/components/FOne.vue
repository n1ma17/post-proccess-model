<script setup>
import { onMounted, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer'
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass'
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass'
import { FXAAShader } from 'three/examples/jsm/shaders/FXAAShader'

const canvasRef = ref(null)

onMounted(() => {
  const scene = new THREE.Scene()
  const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(10, 10, 100)
  camera.lookAt(0, 190, 100)

  const renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 3.5
  renderer.outputEncoding = THREE.sRGBEncoding

  renderer.shadowMap.enabled = true
  // بالاترین کیفیت سایه با امکان تنظیمات خاص (VSM)
  renderer.shadowMap.type = THREE.VSMShadowMap // اینجا می‌تونی THREE.PCFSoftShadowMap رو هم تست کنی

  renderer.physicallyCorrectLights = true
  canvasRef.value.appendChild(renderer.domElement)

  // تنظیمات OrbitControls با damping نرم‌تر
  const controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05 // شدت damping اینجا تنظیم شده

  // نور محیطی
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.3)
  scene.add(ambientLight)

  // نور اصلی با سایه‌های خیلی دقیق
  const dirLight = new THREE.DirectionalLight(0xfff2cc, 9)
  dirLight.position.set(-11.464, 56.948, 18.799)
  dirLight.castShadow = true
  dirLight.shadow.mapSize.width = 16384
  dirLight.shadow.mapSize.height = 16384
  dirLight.shadow.radius = 10
  dirLight.shadow.bias = 1
  scene.add(dirLight)

  // نور حاشیه‌ای
  const rimLight = new THREE.DirectionalLight(0xf7fbff, 1.5)
  rimLight.position.set(-30, 20, -30)
  scene.add(rimLight)

  // نور پرکننده
  const fillLight = new THREE.DirectionalLight(0xf7fbff, 3)
  fillLight.position.set(-30, 5, 10)
  scene.add(fillLight)

  // بارگذاری HDR برای محیط
  new RGBELoader().setPath('/hdr/').load('lakeside_dawn_1k.hdr', (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping
    scene.environment = texture
    scene.background = new THREE.Color(0x000000)

    const loader = new GLTFLoader()
    loader.load('/models/scene2.glb', (gltf) => {
      const maxAnisotropy = renderer.capabilities.getMaxAnisotropy()

      // فعال‌سازی سایه و بهبود کیفیت تکسچرها روی همه Meshها
      gltf.scene.traverse((child) => {
        if (child.isMesh) {
          child.castShadow = true
          child.receiveShadow = true

          if (child.material.map) {
            child.material.map.generateMipmaps = true
            child.material.map.minFilter = THREE.LinearMipmapLinearFilter
            child.material.map.encoding = THREE.sRGBEncoding
            child.material.map.anisotropy = maxAnisotropy
          }
        }
      })
      scene.add(gltf.scene)
    })
  })

  // Postprocessing - Bloom
  const composer = new EffectComposer(renderer)
  const renderPass = new RenderPass(scene, camera)
  composer.addPass(renderPass)

  const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    0.2,
    0.2,
    0.7,
  )
  composer.addPass(bloomPass)

  // افکت vignette
  const vignetteShader = {
    uniforms: {
      tDiffuse: { value: null },
      amount: { value: 0.12 },
      center: { value: new THREE.Vector2(0.5, 0.5) },
      radius: { value: 0.7 },
      blur: { value: 0.18 },
    },
    vertexShader: `
      varying vec2 vUv;
      void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      uniform sampler2D tDiffuse;
      uniform float amount;
      uniform vec2 center;
      uniform float radius;
      uniform float blur;
      varying vec2 vUv;

      void main() {
        vec2 uv = vUv;
        float dist = distance(uv, center);
        float vignette = smoothstep(radius, radius - blur, dist);
        vec4 color = texture2D(tDiffuse, uv);
        gl_FragColor = color * (1.0 - vignette * amount);
      }
    `,
  }

  const vignettePass = new ShaderPass(vignetteShader)
  composer.addPass(vignettePass)

  // FXAA برای آنتی‌آلیاس بهتر
  const fxaaPass = new ShaderPass(FXAAShader)
  let pixelRatio = renderer.getPixelRatio()
  fxaaPass.material.uniforms['resolution'].value.x = 1 / (window.innerWidth * pixelRatio)
  fxaaPass.material.uniforms['resolution'].value.y = 1 / (window.innerHeight * pixelRatio)
  composer.addPass(fxaaPass)

  // حلقه انیمیشن
  const animate = () => {
    requestAnimationFrame(animate)
    controls.update()
    composer.render()
  }
  animate()

  // مدیریت تغییر سایز صفحه
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
    composer.setSize(window.innerWidth, window.innerHeight)
    pixelRatio = renderer.getPixelRatio()
    fxaaPass.material.uniforms['resolution'].value.x = 1 / (window.innerWidth * pixelRatio)
    fxaaPass.material.uniforms['resolution'].value.y = 1 / (window.innerHeight * pixelRatio)
  })
})
</script>

<template>
  <div ref="canvasRef"></div>
</template>

<style scoped>
div {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  margin: 0;
  padding: 0;
}
</style>
