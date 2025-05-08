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
import { GammaCorrectionShader } from 'three/examples/jsm/shaders/GammaCorrectionShader.js'

const canvasRef = ref(null)

onMounted(() => {
  const scene = new THREE.Scene()
  const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000)
  // const aspect = window.innerWidth / window.innerHeight
  // const frustumSize = 20 // اینو می‌تونی به دلخواه تنظیم کنی (بسته به سایز مدل)
  // const camera = new THREE.OrthographicCamera(
  //   (-frustumSize * aspect) / 2, // left
  //   (frustumSize * aspect) / 2, // right
  //   frustumSize / 2, // top
  //   -frustumSize / 2, // bottom
  //   0.1, // near
  //   1000, // far
  // )
  camera.position.set(10, 10, 100)
  camera.lookAt(0, 190, 100)

  const renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setPixelRatio(window.devicePixelRatio)

  renderer.setSize(window.innerWidth, window.innerHeight)
  // renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMapping = THREE.LinearToneMapping
  // renderer.toneMappingExposure = 3.5
  renderer.toneMappingExposure = 1.5
  renderer.outputEncoding = THREE.sRGBEncoding
  renderer.outputColorSpace = THREE.SRGBColorSpace

  renderer.shadowMap.enabled = true // فعال کردن سیستم سایه در رندرر
  renderer.shadowMap.type = THREE.PCFSoftShadowMap // استفاده از PCF برای سایه‌های نرم‌تر و با کیفیت‌تر
  renderer.shadowMap.needsUpdate = true // اطمینان از به‌روزرسانی سایه‌ها

  renderer.physicallyCorrectLights = true
  canvasRef.value.appendChild(renderer.domElement)

  // تنظیمات OrbitControls با damping نرم‌تر
  const controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05 // شدت damping اینجا تنظیم شده

  // نور محیطی برای روشنایی کلی صحنه
  const ambientLight = new THREE.AmbientLight(0xfff2cc, 0.5)
  scene.add(ambientLight)

  // نور اصلی با تنظیمات دقیق سایه
  const dirLight = new THREE.DirectionalLight(0xfff2cc, 3)
  dirLight.position.set(50, 50, 50) // موقعیت نور اصلی
  dirLight.castShadow = true // فعال کردن سایه برای این نور
  dirLight.shadow.mapSize.width = 4096 // افزایش کیفیت سایه در محور X
  dirLight.shadow.mapSize.height = 4096 // افزایش کیفیت سایه در محور Y
  dirLight.shadow.radius = 8 // کاهش نرمی لبه‌های سایه برای جزئیات بیشتر
  dirLight.shadow.bias = -0.001 // تنظیم دقیق‌تر بایاس سایه
  // تنظیمات دوربین سایه برای کنترل دقیق‌تر ناحیه سایه
  dirLight.shadow.camera.near = 0.1 // نزدیک‌ترین فاصله رندر سایه
  dirLight.shadow.camera.far = 500 // دورترین فاصله رندر سایه
  dirLight.shadow.camera.left = -100 // محدوده سمت چپ ناحیه سایه
  dirLight.shadow.camera.right = 100 // محدوده سمت راست ناحیه سایه
  dirLight.shadow.camera.top = 100 // محدوده بالای ناحیه سایه
  dirLight.shadow.camera.bottom = -100 // محدوده پایین ناحیه سایه
  scene.add(dirLight)

  // نور حاشیه‌ای برای ایجاد کنتراست بیشتر
  const rimLight = new THREE.DirectionalLight(0xf7fbff, 1.5)
  rimLight.position.set(-50, 30, -50)
  rimLight.castShadow = true
  scene.add(rimLight)

  // نور پرکننده برای کاهش سایه‌های خیلی تیره
  const fillLight = new THREE.DirectionalLight(0xf7fbff, 3)
  fillLight.position.set(-50, 20, 50)
  fillLight.castShadow = true
  scene.add(fillLight)

  // بارگذاری HDR برای محیط
  new RGBELoader().setPath('/hdr/').load('lakeside_dawn_1k.hdr', (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping
    scene.environment = texture
    scene.background = new THREE.Color(0x000000)

    const loader = new GLTFLoader()
    loader.load('/models/scene2.glb', (gltf) => {
      const maxAnisotropy = renderer.capabilities.getMaxAnisotropy()

      // تنظیمات سایه و متریال برای همه مش‌های مدل
      gltf.scene.traverse((child) => {
        if (child.isMesh) {
          child.castShadow = true // فعال کردن سایه برای این مش
          child.receiveShadow = true // فعال کردن دریافت سایه برای این مش

          // تنظیمات متریال برای سایه‌های بهتر
          if (child.material) {
            // تبدیل به MeshStandardMaterial اگر نیست (این متریال برای سایه‌های واقعی‌تر ضروری است)
            if (!(child.material instanceof THREE.MeshStandardMaterial)) {
              const standardMaterial = new THREE.MeshStandardMaterial({
                map: child.material.map, // حفظ تکسچر اصلی
                color: child.material.color, // حفظ رنگ اصلی
                roughness: 0.7, // میزان زبری سطح (تاثیر مستقیم روی کیفیت سایه)
                metalness: 0.2, // میزان متالیک بودن سطح
              })
              child.material = standardMaterial
            }

            child.material.needsUpdate = true // اطمینان از به‌روزرسانی متریال

            // اگر متریال آرایه‌ای است (چند متریال)
            if (Array.isArray(child.material)) {
              child.material.forEach((mat) => {
                if (!(mat instanceof THREE.MeshStandardMaterial)) {
                  const standardMaterial = new THREE.MeshStandardMaterial({
                    map: mat.map,
                    color: mat.color,
                    roughness: 0.7,
                    metalness: 0.2,
                  })
                  mat = standardMaterial
                }
                mat.needsUpdate = true
              })
            }
          }

          // تنظیمات تکسچر برای کیفیت بهتر
          if (child.material.map) {
            child.material.map.generateMipmaps = true // تولید مپ‌های مختلف برای کیفیت بهتر در فاصله‌های مختلف
            child.material.map.minFilter = THREE.LinearMipmapLinearFilter // فیلتر مناسب برای مینیمم
            child.material.map.encoding = THREE.sRGBEncoding // تنظیم رنگ‌ها
            child.material.map.anisotropy = maxAnisotropy // بهبود کیفیت تکسچر در زوایای مختلف
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

  const gammaPass = new ShaderPass(GammaCorrectionShader)
  composer.addPass(gammaPass)
  // // افکت vignette
  // const vignetteShader = {
  //   uniforms: {
  //     tDiffuse: { value: null },
  //     amount: { value: 0.12 },
  //     center: { value: new THREE.Vector2(0.5, 0.5) },
  //     radius: { value: 0.7 },
  //     blur: { value: 0.18 },
  //   },
  //   vertexShader: `
  //     varying vec2 vUv;
  //     void main() {
  //       vUv = uv;
  //       gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  //     }
  //   `,
  //   fragmentShader: `
  //     uniform sampler2D tDiffuse;
  //     uniform float amount;
  //     uniform vec2 center;
  //     uniform float radius;
  //     uniform float blur;
  //     varying vec2 vUv;

  //     void main() {
  //       vec2 uv = vUv;
  //       float dist = distance(uv, center);
  //       float vignette = smoothstep(radius, radius - blur, dist);
  //       vec4 color = texture2D(tDiffuse, uv);
  //       gl_FragColor = color * (1.0 - vignette * amount);
  //     }
  //   `,
  // }

  // const vignettePass = new ShaderPass(vignetteShader)
  // composer.addPass(vignettePass)

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
  const raycaster = new THREE.Raycaster()
  const mouse = new THREE.Vector2()

  function onClick(event) {
    // محاسبه مختصات ماوس نسبت به صفحه (بین -1 و 1)
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1

    // ایجاد ray از دوربین به نقطه کلیک
    raycaster.setFromCamera(mouse, camera)

    // بررسی برخورد با کل آبجکت‌های صحنه
    const intersects = raycaster.intersectObjects(scene.children, true)

    if (intersects.length > 0) {
      const clickedObject = intersects[0].object
      console.log('✅ کلیک شد روی:', {
        name: clickedObject.name,
        type: clickedObject.type,
        material: clickedObject.material,
        geometry: clickedObject.geometry,
        position: clickedObject.position,
        scale: clickedObject.scale,
        rotation: clickedObject.rotation,
      })
    } else {
      console.log('❌ هیچ آبجکتی کلیک نشد.')
    }
  }

  window.addEventListener('click', onClick)
  // مدیریت تغییر سایز صفحه
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    // camera.left = (-frustumSize * aspect) / 2
    // camera.right = (frustumSize * aspect) / 2
    // camera.top = frustumSize / 2
    // camera.bottom = -frustumSize / 2
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
