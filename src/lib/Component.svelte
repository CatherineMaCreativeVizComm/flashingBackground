<svelte:options customElement={{ tag: "lantern-wc", shadow: "open" }} />

<script>
  import { onMount } from "svelte"
  import * as THREE from "three"
  import { gsap } from "gsap"
  import { ScrollTrigger } from "gsap/ScrollTrigger"

  const browser = typeof window !== "undefined"

  export let foregroundSrc =
    "https://multimedia.scmp.com/components/2026/lantern-wc/foregroundImg.png"
  export let backgroundSrc =
    "https://multimedia.scmp.com/components/2026/lantern-wc/backgroundImg.png"
  export let scrollspeed = 1
  export let screenHeight = 1200

  let stickyWrapper
  let stickyInner
  let canvasContainer
  let candleOverlay

  let scene
  let camera
  let renderer
  let bgMesh
  let fgMesh
  let candleLight
  let animationId
  let ctx
  let loader

  let prevForegroundSrc = foregroundSrc
  let prevBackgroundSrc = backgroundSrc

  const scrollProgress = { value: 0 }

  const loadTexture = (url) => {
    if (!loader || !url) return null
    const tex = loader.load(url)
    tex.wrapS = THREE.RepeatWrapping
    tex.wrapT = THREE.ClampToEdgeWrapping
    tex.colorSpace = THREE.SRGBColorSpace
    tex.repeat.set(2, 1)
    return tex
  }

  const initThreeJS = () => {
    if (!canvasContainer) return

    loader = new THREE.TextureLoader()
    scene = new THREE.Scene()
    scene.background = new THREE.Color(0x220500)
    scene.fog = new THREE.FogExp2(0x220500, 0.02)

    const width = canvasContainer.clientWidth
    const height = canvasContainer.clientHeight || 1

    camera = new THREE.PerspectiveCamera(50, width / height, 0.1, 100)
    camera.position.set(0, 0, 35)
    camera.lookAt(0, 0, 0)

    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true })
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
    renderer.setSize(width, height)
    renderer.toneMapping = THREE.ACESFilmicToneMapping
    renderer.toneMappingExposure = 2.3

    canvasContainer.innerHTML = ""
    canvasContainer.appendChild(renderer.domElement)

    const bgGeo = new THREE.CylinderGeometry(9, 9, 7, 64, 1, true)
    const bgMat = new THREE.MeshStandardMaterial({
      map: loadTexture(backgroundSrc),
      side: THREE.DoubleSide,
      transparent: true,
      opacity: 0.9,
      emissive: 0xff5500,
      emissiveIntensity: 0.8,
      roughness: 0.4,
    })
    bgMesh = new THREE.Mesh(bgGeo, bgMat)
    scene.add(bgMesh)

    const fgGeo = new THREE.CylinderGeometry(15, 15, 10, 64, 1, true)
    const fgMat = new THREE.MeshStandardMaterial({
      map: loadTexture(foregroundSrc),
      side: THREE.DoubleSide,
      transparent: true,
      opacity: 0.95,
      emissive: 0xff5500,
      emissiveIntensity: 0.4,
      alphaTest: 0.1,
    })
    fgMesh = new THREE.Mesh(fgGeo, fgMat)
    scene.add(fgMesh)

    candleLight = new THREE.PointLight(0xff6600, 1500, 45)
    scene.add(candleLight)

    const ambient = new THREE.AmbientLight(0xffaa33, 0.5)
    scene.add(ambient)

    const flameGeo = new THREE.SphereGeometry(0.3, 16, 16)
    const flameMat = new THREE.MeshBasicMaterial({ color: 0xffaa88 })
    const flameMesh = new THREE.Mesh(flameGeo, flameMat)
    scene.add(flameMesh)

    const clock = new THREE.Clock()

    const animate = () => {
      animationId = requestAnimationFrame(animate)
      const time = clock.getElapsedTime()

      candleLight.intensity =
        600 + Math.sin(time * 15) * 100 + Math.random() * 50

      const autoRot = time * (scrollspeed * 0.1)
      const scrollRot = scrollProgress.value * (Math.PI * 2)

      if (bgMesh && fgMesh) {
        bgMesh.rotation.y = autoRot * 1.5 + scrollRot * 1.5
        fgMesh.rotation.y = autoRot * 0.8 + scrollRot * 0.8
      }

      renderer.render(scene, camera)
    }

    animate()
  }

  const onResize = () => {
    if (camera && renderer && canvasContainer) {
      const width = canvasContainer.clientWidth
      const height = canvasContainer.clientHeight || 1
      camera.aspect = width / height
      camera.updateProjectionMatrix()
      renderer.setSize(width, height)
    }
  }

  const updateTextures = () => {
    if (!loader || !bgMesh || !fgMesh) return

    if (
      backgroundSrc &&
      backgroundSrc !== prevBackgroundSrc &&
      bgMesh.material
    ) {
      bgMesh.material.map = loadTexture(backgroundSrc)
      bgMesh.material.needsUpdate = true
      prevBackgroundSrc = backgroundSrc
    }

    if (
      foregroundSrc &&
      foregroundSrc !== prevForegroundSrc &&
      fgMesh.material
    ) {
      fgMesh.material.map = loadTexture(foregroundSrc)
      fgMesh.material.needsUpdate = true
      prevForegroundSrc = foregroundSrc
    }
  }

  const destroyThree = () => {
    cancelAnimationFrame(animationId)
    if (bgMesh) {
      bgMesh.geometry.dispose()
      bgMesh.material.dispose()
    }
    if (fgMesh) {
      fgMesh.geometry.dispose()
      fgMesh.material.dispose()
    }
    if (renderer) {
      renderer.dispose()
    }
    scene = null
    camera = null
    renderer = null
    bgMesh = null
    fgMesh = null
    candleLight = null
    loader = null
  }

  onMount(() => {
    if (!browser) return
    gsap.registerPlugin(ScrollTrigger)

    initThreeJS()
    window.addEventListener("resize", onResize)

    ctx = gsap.context(() => {
      let h = screenHeight
      if (window.innerWidth < 768) h = screenHeight * 0.8
      if (stickyWrapper) stickyWrapper.style.height = `${h * 3}px`

      if (candleOverlay) {
        gsap.to(candleOverlay, {
          opacity: 0.9,
          duration: 0.2,
          repeat: -1,
          yoyo: true,
          ease: "rough({ strength: 1, points: 20, randomize: true })",
        })
      }

      gsap.to(scrollProgress, {
        value: 1,
        ease: "none",
        scrollTrigger: {
          trigger: stickyWrapper,
          start: "top top",
          end: "bottom bottom",
          pin: stickyInner,
          scrub: 0.5,
        },
      })
    }, stickyWrapper)

    return () => {
      window.removeEventListener("resize", onResize)
      if (ctx) ctx.revert()
      destroyThree()
    }
  })

  $: if (browser && renderer && (foregroundSrc || backgroundSrc)) {
    updateTextures()
  }

  $: if (browser && stickyWrapper) {
    let h = screenHeight
    if (window.innerWidth < 768) h = screenHeight * 0.8
    stickyWrapper.style.height = `${h * 3}px`
  }
</script>

<section class="revolvingLatternCtn">
  <div class="revolvingLattern">
    <div class="stickyWrapper" bind:this={stickyWrapper}>
      <div class="stickyInner" bind:this={stickyInner}>
        <div class="stickyBackground">
          <div class="canvasContainer" bind:this={canvasContainer}></div>
        </div>
        <div class="stickyForeground">
          <div class="candleOverlay" bind:this={candleOverlay}></div>
        </div>
      </div>
    </div>
  </div>
</section>

<style>
  :global(body) {
    overflow-x: hidden;
    min-width: 290px;
    width: 100%;
    height: 100%;
    -webkit-tap-highlight-color: transparent;
    -webkit-touch-callout: none;
    -webkit-font-smoothing: antialiased;
  }
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  .revolvingLattern {
    width: 100%;
    margin: 0 auto;
  }

  .stickyWrapper {
    position: relative;
    width: 100%;
  }

  .stickyInner {
    position: relative;
    width: 100%;
    height: 100vh;
    overflow: hidden;
  }

  .stickyBackground,
  .stickyForeground {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }

  .canvasContainer {
    width: 100%;
    height: 100%;
  }

  .candleOverlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 10;
    background: radial-gradient(
      circle at center,
      rgba(255, 200, 100, 0.1) 10%,
      rgba(50, 10, 0, 0.3) 50%,
      rgba(20, 5, 0, 1) 90%
    );
    mix-blend-mode: multiply;
  }
</style>
