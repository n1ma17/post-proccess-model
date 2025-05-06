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
import { SSAOPass } from 'three/examples/jsm/postprocessing/SSAOPass'
import { FXAAShader } from 'three/examples/jsm/shaders/FXAAShader'

const canvasRef = ref(null)

onMounted(() => {
  const scene = new THREE.Scene()
  const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(10, 10, 10)
  camera.lookAt(0, 0, 0)

  const renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 3.5
  renderer.outputEncoding = THREE.sRGBEncoding
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap

  canvasRef.value.appendChild(renderer.domElement)

  const controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true

  // نور محیطی خیلی کم
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.3)
  scene.add(ambientLight)

  // نور اصلی قوی و گرم
  const dirLight = new THREE.DirectionalLight(0xfff2cc, 6)
  dirLight.position.set(20, 40, 20)
  dirLight.castShadow = true
  dirLight.shadow.mapSize.width = 4096
  dirLight.shadow.mapSize.height = 4096
  dirLight.shadow.radius = 10
  dirLight.shadow.bias = -0.0008
  scene.add(dirLight)

  // نور Rim آبی برای درخشش لبه‌ها
  const rimLight = new THREE.DirectionalLight(0x6699ff, 2.5)
  rimLight.position.set(-30, 20, -30)
  rimLight.castShadow = false
  scene.add(rimLight)

  // نور Fill خیلی ضعیف
  const fillLight = new THREE.DirectionalLight(0xffffff, 0.5)
  fillLight.position.set(-10, 5, 10)
  fillLight.castShadow = false
  scene.add(fillLight)

  new RGBELoader().setPath('/hdr/').load('belfast_sunset_puresky_1k.hdr', (texture) => {
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

  // SSAO pass with optimized settings
  const ssaoPass = new SSAOPass(scene, camera, window.innerWidth, window.innerHeight)
  ssaoPass.kernelRadius = 80
  ssaoPass.minDistance = 0.005
  ssaoPass.maxDistance = 0.2
  composer.addPass(ssaoPass)

  const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    0.2, // strength
    0.2, // radius
    0.7, // threshold
  )
  composer.addPass(bloomPass)

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

  // FXAA Pass
  const fxaaPass = new ShaderPass(FXAAShader)
  let pixelRatio = renderer.getPixelRatio()
  fxaaPass.material.uniforms['resolution'].value.x = 1 / (window.innerWidth * pixelRatio)
  fxaaPass.material.uniforms['resolution'].value.y = 1 / (window.innerHeight * pixelRatio)
  composer.addPass(fxaaPass)

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
    ssaoPass.setSize(window.innerWidth, window.innerHeight)
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
