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

const canvasRef = ref(null)

onMounted(() => {
  const scene = new THREE.Scene()
  const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(10, 10, 10)
  camera.lookAt(0, 0, 0)

  const renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 2.5 // Increases the overall brightness
  renderer.outputEncoding = THREE.sRGBEncoding
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap

  canvasRef.value.appendChild(renderer.domElement)

  const controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true

  const ambientLight = new THREE.AmbientLight(0xffffff, 2.5) // Increased intensity for more ambient light
  scene.add(ambientLight)

  const dirLight = new THREE.DirectionalLight(0xffffff, 4) // Increased intensity for the directional light
  dirLight.position.set(10, 20, 10)
  dirLight.castShadow = true
  dirLight.shadow.mapSize.width = 2048
  dirLight.shadow.mapSize.height = 2048
  dirLight.shadow.camera.near = 0.5
  dirLight.shadow.camera.far = 50
  scene.add(dirLight)

  new RGBELoader().setPath('/hdr/').load('rustig_koppie_puresky_1k.hdr', (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping
    scene.environment = texture
    scene.background = new THREE.Color(0x000000)

    const loader = new GLTFLoader()
    loader.load('/models/scene2.glb', (gltf) => {
      gltf.scene.traverse((child) => {
        if (child.isMesh) {
          child.castShadow = true
          child.receiveShadow = true
        }
      })
      scene.add(gltf.scene)
    })
  })

  const composer = new EffectComposer(renderer)
  const renderPass = new RenderPass(scene, camera)
  composer.addPass(renderPass)

  const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    0.2, // Strength: Increased for a more pronounced bloom effect
    0.2, // Radius: Larger bloom radius
    0.6, // Threshold: Keep bright areas to trigger bloom
  )
  composer.addPass(bloomPass)

  // Vignette Shader with subtle effect
  const vignetteShader = {
    uniforms: {
      tDiffuse: { value: null },
      amount: { value: 0.015 }, // مقدار کمتر برای Vignette
      center: { value: new THREE.Vector2(0.5, 0.5) }, // مرکز Vignette در وسط تصویر
      radius: { value: 0.25 }, // شعاع کمتر برای اثرگذاری فقط در اطراف
      blur: { value: 0.01 }, // مقدار محو کردن اطراف
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
      // محاسبه Vignette با استفاده از smoothstep برای ایجاد یک انتقال نرم‌تر
      float vignette = smoothstep(radius, radius - blur, dist);
      vec4 color = texture2D(tDiffuse, uv);
      // اعمال Vignette بدون تاثیر بر سایه‌ها
      gl_FragColor = color * (1.0 - vignette * amount);
    }
  `,
  }

  const vignettePass = new ShaderPass(vignetteShader)
  composer.addPass(vignettePass)

  const animate = () => {
    requestAnimationFrame(animate)
    controls.update()
    composer.render()
  }
  animate()

  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
    composer.setSize(window.innerWidth, window.innerHeight)
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
