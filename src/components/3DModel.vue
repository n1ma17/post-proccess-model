<script setup>
import { onMounted, ref, onUnmounted } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader'
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer'
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass'
// import { SSAOPass } from 'three/examples/jsm/postprocessing/SSAOPass'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass'
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass'

const container = ref(null)
let scene, camera, renderer, controls, composer

onMounted(() => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color('#bdd4e5')

  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(30, 10, 20)
  camera.lookAt(0, 0, 0)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.toneMapping = THREE.LinearToneMapping
  renderer.toneMappingExposure = 1.278

  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.outputColorSpace = THREE.SRGBColorSpace

  container.value.appendChild(renderer.domElement)

  const dirLight = new THREE.DirectionalLight(0xffffff, 3)
  dirLight.position.set(-100, 190, 40)
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
  const targetSadowObject = new THREE.Object3D()
  targetSadowObject.position.set(30, -10, 20)
  dirLight.target = targetSadowObject
  scene.add(dirLight)

  const light = new THREE.DirectionalLight('#ffe0b5', 1)
  light.position.set(0, 20, -20)
  light.shadow.bias = 0.2659
  const targetObject = new THREE.Object3D()
  targetObject.position.set(0, -10, -20)
  scene.add(targetObject)
  light.target = targetObject
  scene.add(light)

  const light2 = new THREE.DirectionalLight('#96b2b7', 0.6675)
  light2.position.set(25, 10, 0)
  light2.shadow.bias = 0.2659
  const targetObject2 = new THREE.Object3D()
  targetObject2.position.set(10, -10, 10)
  scene.add(targetObject2)
  light2.target = targetObject2
  scene.add(light2)

  const light3 = new THREE.DirectionalLight('#ffffff', 8)
  light3.castShadow = false
  light3.position.set(20, 2, 25)

  const targetObject3 = new THREE.Object3D()
  targetObject3.position.set(0, 2, 0)
  scene.add(targetObject3)
  light3.target = targetObject3
  scene.add(light3)

  const light4 = new THREE.DirectionalLight('#ffffff', 2)
  light4.position.set(200, 100, 0)
  light4.castShadow = false
  const targetObject4 = new THREE.Object3D()
  targetObject4.position.set(0, 0, 10)
  scene.add(targetObject4)
  light4.target = targetObject4
  scene.add(light4)

  const light4Helper = new THREE.DirectionalLightHelper(light4, 6)
  scene.add(light4Helper)
  const light3Helper = new THREE.DirectionalLightHelper(light3, 6)
  scene.add(light3Helper)
  const dirLightHelper = new THREE.DirectionalLightHelper(light, 6)
  scene.add(dirLightHelper)
  const dirLightHelper2 = new THREE.DirectionalLightHelper(light2, 6)
  scene.add(dirLightHelper2)
  const dirLightHelperShadow = new THREE.DirectionalLightHelper(dirLight, 6)
  scene.add(dirLightHelperShadow)

  const axesHelper = new THREE.AxesHelper(50)
  scene.add(axesHelper)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.enabled = true

  // // اضافه کردن event listener برای نمایش موقعیت دوربین
  // controls.addEventListener('change', () => {
  //   console.log('موقعیت دوربین:', {
  //     x: camera.position.x.toFixed(2),
  //     y: camera.position.y.toFixed(2),
  //     z: camera.position.z.toFixed(2),
  //   })
  // })

  // // اضافه کردن event listener برای نمایش موقعیت lookAt
  // controls.addEventListener('change', () => {
  //   const target = controls.target
  //   console.log('نقطه نگاه دوربین:', {
  //     x: target.x.toFixed(2),
  //     y: target.y.toFixed(2),
  //     z: target.z.toFixed(2),
  //   })
  // })

  // تنظیم EffectComposer و SSAO
  composer = new EffectComposer(renderer)
  const renderPass = new RenderPass(scene, camera)
  composer.addPass(renderPass)

  // const ssaoPass = new SSAOPass(scene, camera, window.innerWidth, window.innerHeight)
  // ssaoPass.minDistance = 0
  // ssaoPass.maxDistance = 0
  // ssaoPass.output = SSAOPass.OUTPUT.Default
  // ssaoPass.bias = 0.7
  // ssaoPass.intensity = 0.306
  // ssaoPass.radius = 9
  // composer.addPass(ssaoPass)

  const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    0.167,
    0.489,
    0.442,
  )
  composer.addPass(bloomPass)

  const VignetteShader = {
    uniforms: {
      tDiffuse: { value: null },
      offset: { value: 1.0 }, // سختی (vignette hardness)
      darkness: { value: 1.0 }, // شدت (vignette amount)
    },
    vertexShader: /* glsl */ `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
    fragmentShader: /* glsl */ `
    uniform sampler2D tDiffuse;
    uniform float offset;
    uniform float darkness;
    varying vec2 vUv;

    void main() {
      vec4 texel = texture2D(tDiffuse, vUv);
      float dist = distance(vUv, vec2(0.5, 0.5));
      texel.rgb *= smoothstep(0.8, offset * 0.799, dist * (darkness + offset));
      gl_FragColor = texel;
    }
  `,
  }
  const vignettePass = new ShaderPass(VignetteShader)
  vignettePass.uniforms['darkness'].value = 0.15 // به جای 0.228
  vignettePass.uniforms['offset'].value = 0.5
  composer.addPass(vignettePass)
  new RGBELoader().setPath('/hdr/').load('photo_studio_01_2k.hdr', (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping
    scene.environment = texture
    scene.background = new THREE.Color('#bdd4e5')
    loadModel()
  })
  const ColorCorrectionShader = {
    uniforms: {
      tDiffuse: { value: null },
      brightness: { value: 0.078 },
      contrast: { value: 0.033 },
      saturation: { value: 1.0 },
    },
    vertexShader: /* glsl */ `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position,1.0);
    }
  `,
    fragmentShader: /* glsl */ `
    uniform sampler2D tDiffuse;
    uniform float brightness;
    uniform float contrast;
    uniform float saturation;
    varying vec2 vUv;

    vec3 applySaturation(vec3 color, float sat) {
      float luma = dot(color, vec3(0.2126, 0.7152, 0.0722));
      return mix(vec3(luma), color, sat);
    }

    void main() {
      vec4 texel = texture2D(tDiffuse, vUv);
      texel.rgb += brightness;
      texel.rgb = (texel.rgb - 0.5) * (contrast + 1.0) + 0.5;
      texel.rgb = applySaturation(texel.rgb, saturation);
      gl_FragColor = texel;
    }
  `,
  }

  const colorCorrectionPass = new ShaderPass(ColorCorrectionShader)
  colorCorrectionPass.uniforms.brightness.value = 0.088
  colorCorrectionPass.uniforms.contrast.value = 0.063
  colorCorrectionPass.uniforms.saturation.value = 1.0

  composer.addPass(colorCorrectionPass)
  animate()

  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
    composer.setSize(window.innerWidth, window.innerHeight)
  })
})

function animate() {
  requestAnimationFrame(animate)
  controls.update()
  composer.render()
}

function loadModel() {
  const loader = new GLTFLoader()
  const dracoLoader = new DRACOLoader()
  dracoLoader.setDecoderPath('/draco/')
  loader.setDRACOLoader(dracoLoader)

  loader.load(
    '/models/scene2.glb',
    (gltf) => {
      console.log('✅ مدل لود شد')
      const model = gltf.scene

      const maxAnisotropy = renderer.capabilities.getMaxAnisotropy()
      model.traverse((child) => {
        if (child.isMesh) {
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

      const box = new THREE.Box3().setFromObject(model)
      // const center = box.getCenter(new THREE.Vector3())
      const size = box.getSize(new THREE.Vector3())
      // model.position.sub(center)
      model.position.set(0, 0, 0)
      const maxDim = Math.max(size.x, size.y, size.z)
      const scale = 100 / maxDim
      model.scale.multiplyScalar(scale)

      scene.add(model)
    },
    (xhr) => {
      console.log('پیشرفت لود:', ((xhr.loaded / xhr.total) * 100).toFixed(2) + '%')
    },
    (error) => {
      console.error('خطا در لود مدل:', error)
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
</style>
