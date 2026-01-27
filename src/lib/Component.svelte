<svelte:options customElement={{ tag: "lantern-wc", shadow: "open" }} />

<script>
  import { onMount } from "svelte";
  import * as THREE from "three";
  import { gsap } from "gsap";
  import { ScrollTrigger } from "gsap/ScrollTrigger";

  const browser = typeof window !== "undefined";

  export let foregroundSrc = "static/lanternCover.png";
  export let lanternOuterSrc = "static/zodiac1.png";
  export let innerImageSrc = "static/zodiac2.png";
  export let clockBackgroundSrc = "static/clockBg.png";

  export let backgroundSrc = [
    "static/1.png",
    "static/2.png",
    "static/3.png",
    "static/4.png",
    "static/5.png",
    "static/6.png",
    "static/7.png",
    "static/8.png",
    "static/9.png",
    "static/10.png",
    "static/11.png",
    "static/12.png",
    "static/13.png",
  ];

  export let textSrc = [
    "Lantern Text 1",
    "Lantern Text 2",
    "Lantern Text 3",
    "Lantern Text 4",
    "Lantern Text 5",
    "Lantern Text 6",
    "Lantern Text 7",
    "Lantern Text 8",
    "Lantern Text 9",
    "Lantern Text 10",
    "Lantern Text 11",
    "Lantern Text 12",
    "Lantern Text 13",
  ];

  export let clockTextSrc = [
    "Clock Item 1",
    "Clock Item 2",
    "Clock Item 3",
    "Clock Item 4",
    "Clock Item 5",
    "Clock Item 6",
    "Clock Item 7",
    "Clock Item 8",
    "Clock Item 9",
    "Clock Item 10",
    "Clock Item 11",
    "Clock Item 12",
    "Ignored",
  ];

  export let transitionText = "In traditional Chinese timekeeping,";
  export let screenHeight = 1200;

  const dayColors = [
    new THREE.Color("#1a1a2e"),
    new THREE.Color("#4a5e6d"),
    new THREE.Color("#536b6a"),
    new THREE.Color("#8c5e35"),
    new THREE.Color("#2e2e4a"),
  ];

  const loadTexture = (loader, url) => {
    if (!loader || !url) return null;
    const tex = loader.load(url);
    tex.colorSpace = THREE.SRGBColorSpace;
    return tex;
  };

  const parseInput = (input, maxLen) => {
    let items = [];
    if (Array.isArray(input)) items = input;
    else if (typeof input === "string") {
      try {
        items = JSON.parse(input);
      } catch (e) {
        items = input
          .split(",")
          .map((s) => s.trim())
          .filter((s) => s.length);
      }
    }
    if (!maxLen) return items;
    const result = [];
    for (let i = 0; i < maxLen; i++) {
      result.push(items[i % items.length]);
    }
    return result;
  };

  class EnvironmentManager {
    constructor(scene) {
      this.scene = scene;
      this.group = new THREE.Group();
      this.scene.add(this.group);
      this.init();
    }

    init() {
      const skyGeo = new THREE.PlaneGeometry(600, 600);
      this.skyMat = new THREE.MeshBasicMaterial({
        color: 0x000000,
        transparent: true,
        opacity: 0,
        depthWrite: false,
      });
      const skyMesh = new THREE.Mesh(skyGeo, this.skyMat);
      skyMesh.position.z = -200;
      this.group.add(skyMesh);

      const canvas = document.createElement("canvas");
      canvas.width = 512;
      canvas.height = 512;
      const ctx = canvas.getContext("2d");
      const grad = ctx.createRadialGradient(256, 256, 20, 256, 256, 256);
      grad.addColorStop(0, "rgba(255, 200, 120, 0.9)");
      grad.addColorStop(0.4, "rgba(100, 50, 0, 0.7)");
      grad.addColorStop(1, "rgba(0, 0, 0, 0)");
      ctx.fillStyle = grad;
      ctx.fillRect(0, 0, 512, 512);
      const groundTex = new THREE.CanvasTexture(canvas);

      this.groundMat = new THREE.MeshBasicMaterial({
        map: groundTex,
        transparent: true,
        opacity: 1,
        depthWrite: false,
        side: THREE.DoubleSide,
      });
      const groundMesh = new THREE.Mesh(
        new THREE.PlaneGeometry(80, 80),
        this.groundMat,
      );
      groundMesh.rotation.x = -Math.PI / 2;
      groundMesh.position.y = -8;
      this.group.add(groundMesh);

      this.wallMat = new THREE.MeshStandardMaterial({
        color: 0x220a00,
        roughness: 1.0,
        metalness: 0.0,
        side: THREE.FrontSide,
        transparent: true,
        opacity: 1,
      });
      const wallMesh = new THREE.Mesh(
        new THREE.PlaneGeometry(400, 400),
        this.wallMat,
      );
      wallMesh.position.z = -80;
      this.group.add(wallMesh);
    }

    update(transitionFactor, highlightProgress) {
      const envOpacity = Math.max(0, 1 - transitionFactor * 1.5);
      this.groundMat.opacity = envOpacity;
      this.wallMat.opacity = envOpacity;
      this.skyMat.opacity = transitionFactor;
      if (transitionFactor > 0.01) {
        if (highlightProgress <= 0) {
          this.skyMat.color.copy(dayColors[0]);
        } else {
          const count = dayColors.length - 1;
          const progress = highlightProgress * count;
          const idx = Math.floor(progress);
          const alpha = progress - idx;
          const c1 = dayColors[Math.min(idx, count)];
          const c2 = dayColors[Math.min(idx + 1, count)];
          this.skyMat.color.copy(c1).lerp(c2, alpha);
        }
      }
    }

    dispose() {
      this.scene.remove(this.group);
      if (this.skyMat) this.skyMat.dispose();
      if (this.groundMat) this.groundMat.dispose();
      if (this.wallMat) this.wallMat.dispose();
    }
  }

  class LanternManager {
    constructor(scene, loader, config) {
      this.scene = scene;
      this.loader = loader;
      this.config = config;
      this.group = new THREE.Group();
      this.scene.add(this.group);
      this.init();
    }

    createLayer(radius, imgUrl, height, opacity, emIntensity) {
      const tex = loadTexture(this.loader, imgUrl);
      if (tex) {
        tex.wrapS = THREE.RepeatWrapping;
        tex.repeat.set(1, 1);
      }
      const geo = new THREE.CylinderGeometry(
        radius,
        radius,
        height,
        64,
        1,
        true,
      );
      const mat = new THREE.MeshStandardMaterial({
        map: tex,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: opacity,
        emissive: 0xffaa00,
        emissiveMap: tex,
        emissiveIntensity: emIntensity,
        roughness: 0.5,
        metalness: 0.1,
        depthWrite: false,
      });
      return new THREE.Mesh(geo, mat);
    }

    init() {
      const h = 14;
      this.outerMesh = this.createLayer(
        12.5,
        this.config.outerSrc,
        h,
        1.0,
        1.5,
      );
      this.group.add(this.outerMesh);

      this.coreMesh = this.createLayer(
        11.5,
        this.config.innerSrc,
        h * 0.8,
        0.5,
        2.0,
      );
      this.group.add(this.coreMesh);

      const tex = loadTexture(this.loader, this.config.frameSrc);
      if (tex) {
        tex.wrapS = THREE.RepeatWrapping;
        tex.repeat.set(1, 1);
      }

      const fgGeo = new THREE.CylinderGeometry(16, 16, 17, 6, 1, false);
      const fgSideMat = new THREE.MeshStandardMaterial({
        map: tex,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.3,
        emissive: 0xff5500,
        emissiveIntensity: 0.5,
        alphaTest: 0.05,
        depthWrite: false,
      });
      const fgCapMat = new THREE.MeshStandardMaterial({
        color: 0x2a0d00,
        side: THREE.DoubleSide,
        roughness: 0.9,
        transparent: true,
        opacity: 1,
      });
      this.frameMesh = new THREE.Mesh(fgGeo, [fgSideMat, fgCapMat, fgCapMat]);
      this.frameMesh.renderOrder = 20;

      const strutGeo = new THREE.CylinderGeometry(0.2, 0.2, 18, 8);
      const strutMat = new THREE.MeshStandardMaterial({
        color: 0x3d1c00,
        transparent: true,
        opacity: 1,
      });
      for (let i = 0; i < 6; i++) {
        const angle = (i / 6) * Math.PI * 2;
        const strut = new THREE.Mesh(strutGeo, strutMat);
        strut.position.set(
          Math.cos(angle + Math.PI / 6) * 16,
          0,
          Math.sin(angle + Math.PI / 6) * 16,
        );
        this.frameMesh.add(strut);
      }
      this.scene.add(this.frameMesh);
    }

    update(time, rotationY, fgRotationY, transitionFactor, isMobile) {
      const lanternOpacity = 1 - transitionFactor;
      const envOpacity = Math.max(0, 1 - transitionFactor * 1.5); 

      this.group.visible = lanternOpacity > 0.01;
      this.frameMesh.visible = envOpacity > 0.01;

      if (this.group.visible) {
        const expScale = 1 + transitionFactor * 0.5;
        const floatY = Math.sin(time * 1.5) * 0.3;
        const hAdj = isMobile ? 3 : 1.5;

        this.outerMesh.rotation.y = rotationY;
        this.outerMesh.position.y = hAdj + floatY;
        this.outerMesh.scale.setScalar(expScale);
        this.outerMesh.material.opacity = lanternOpacity; 
        this.outerMesh.material.emissiveIntensity = 1.5 * lanternOpacity;

        this.coreMesh.rotation.y = rotationY * 0.45;
        this.coreMesh.position.y = -hAdj + Math.sin(time * 0.8 + 10) * 0.25;
        this.coreMesh.scale.setScalar(expScale);
        this.coreMesh.material.opacity = lanternOpacity * 0.2;
        this.coreMesh.material.emissiveIntensity = 2.0 * lanternOpacity;
      }

      if (this.frameMesh.visible) {
        this.frameMesh.rotation.y = fgRotationY;
        const fgOp = 0.3 * envOpacity;
        if (Array.isArray(this.frameMesh.material)) {
          this.frameMesh.material.forEach((m) => {
            if (m.transparent) m.opacity = fgOp;
            if (m.emissive) m.emissiveIntensity = 0.5 * envOpacity;
          });
        }
        this.frameMesh.children.forEach(
          (c) => (c.material.opacity = 0.9 * envOpacity),
        );
      }
    }

    dispose() {
      this.scene.remove(this.group);
      this.scene.remove(this.frameMesh);
      const safeDispose = (m) => {
        if (Array.isArray(m)) m.forEach((x) => x.dispose());
        else if (m) m.dispose();
      };
      safeDispose(this.outerMesh.material);
      safeDispose(this.coreMesh.material);
      safeDispose(this.frameMesh.material);
      this.frameMesh.children.forEach((c) => safeDispose(c.material));
    }
  }

  class ClockManager {
    constructor(scene, loader, images, bgSrc) {
      this.scene = scene;
      this.loader = loader;
      this.images = images;
      this.bgSrc = bgSrc;
      this.group = new THREE.Group();
      this.group.visible = false;
      this.scene.add(this.group);
      this.init();
    }

    init() {
      if (this.bgSrc) {
        const tex = loadTexture(this.loader, this.bgSrc);
        const geo = new THREE.PlaneGeometry(1, 1);
        this.bgMesh = new THREE.Mesh(
          geo,
          new THREE.MeshBasicMaterial({
            map: tex,
            transparent: true,
            opacity: 0,
            depthWrite: false,
          }),
        );
        this.bgMesh.position.z = -1;
        this.group.add(this.bgMesh);
      }

      const count = Math.min(this.images.length, 12);
      for (let i = 0; i < count; i++) {
        const container = new THREE.Group();
        const tex = loadTexture(this.loader, this.images[i]);
        const mesh = new THREE.Mesh(
          new THREE.PlaneGeometry(1, 1),
          new THREE.MeshBasicMaterial({
            map: tex,
            side: THREE.DoubleSide,
            transparent: true,
            color: 0xffffff,
          }),
        );
        container.add(mesh);
        container.userData = {
          id: i,
          angle: Math.PI / 2 - (i / 12) * Math.PI * 2,
          startRot: (i / 12) * Math.PI * 2,
          floatOffset: Math.random() * 10,
        };
        this.group.add(container);
      }
    }

    update(
      time,
      rotationY,
      transitionFactor,
      highlightProgress,
      activeIndex,
      isMobile,
    ) {
      this.group.visible = transitionFactor > 0.01;
      this.group.rotation.y = rotationY * (1 - transitionFactor);

      if (!this.group.visible) return;

      const radius = isMobile ? 16 : 24;
      const bgSize = isMobile ? 28 : 40;
      const pHeight = isMobile ? 6 : 5;
      const startRad = 13.5;

      if (this.bgMesh) {
        this.bgMesh.scale.set(bgSize, bgSize, 1);
        this.bgMesh.material.opacity = transitionFactor;
      }

      this.group.children.forEach((child) => {
        if (child === this.bgMesh) return;
        const u = child.userData;
        const mesh = child.children[0];
        const tx = Math.cos(u.angle) * radius;
        const ty = Math.sin(u.angle) * radius;
        const sx = Math.sin(u.startRot) * startRad;
        const sz = Math.cos(u.startRot) * startRad;

        child.position.set(
          sx + (tx - sx) * transitionFactor,
          0 +
            (ty - 0) * transitionFactor +
            Math.sin(time * 2 + u.floatOffset) * 0.2,
          sz + (0.5 - sz) * transitionFactor,
        );

        child.rotation.y = u.startRot * (1 - transitionFactor);

        let scale = 1;
        let opacity = transitionFactor;
        let renderOrder = 5;

        if (highlightProgress > 0) {
          const isActive = u.id === activeIndex;
          const targetScale = isActive ? 1.2 : 1.0;
          const targetOp = isActive ? 1.0 : 0.2;
          scale = child.scale.x + (targetScale - child.scale.x) * 0.1;
          opacity = targetOp;
          renderOrder = isActive ? 10 : 5;
        }

        child.scale.setScalar(scale);
        child.renderOrder = renderOrder;

        if (mesh.material.map && mesh.material.map.image) {
          const img = mesh.material.map.image;
          const aspect = img.width / img.height;
          if (aspect > 1) mesh.scale.set(pHeight * aspect, pHeight, 1);
          else mesh.scale.set(pHeight * aspect, pHeight, 1);
        } else {
          mesh.scale.set(pHeight, pHeight, 1);
        }

        mesh.material.opacity =
          mesh.material.opacity + (opacity - mesh.material.opacity) * 0.1;
      });
    }

    dispose() {
      this.scene.remove(this.group);
      this.group.children.forEach((c) => {
        if (c.material) c.material.dispose();
        if (c.children)
          c.children.forEach((gc) => {
            if (gc.material) gc.material.dispose();
          });
      });
    }
  }

  let stickyWrapper, stickyInner, canvasContainer, candleOverlay;
  let scene, camera, renderer, candleLight, loader;

  let envManager, lanternManager, clockManager;

  let animationId;
  let ctx;
  let isInitialized = false;
  let observer;

  let userScrolled = false;
  let activeIndex = 0;
  let isClockMode = false;
  let textVisible = false;
  let showTransitionText = false;

  let cameraZStart = 40;
  let cameraZEnd = 65;

  const scrollProgress = { value: 0 };

  $: parsedLanternTexts = parseInput(textSrc, 13);
  $: parsedClockTexts = parseInput(clockTextSrc, 12);
  $: currentTextList = isClockMode ? parsedClockTexts : parsedLanternTexts;
  $: parsedBackgrounds = parseInput(backgroundSrc, 13);

  const init = () => {
    if (isInitialized || !canvasContainer) return;

    loader = new THREE.TextureLoader();
    loader.setCrossOrigin("anonymous");

    scene = new THREE.Scene();

    const width = canvasContainer.clientWidth;
    const height = canvasContainer.clientHeight || 1;
    camera = new THREE.PerspectiveCamera(50, width / height, 0.1, 500);
    camera.position.set(0, 0, 40);
    camera.lookAt(0, 0, 0);

    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(width, height);
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    canvasContainer.innerHTML = "";
    canvasContainer.appendChild(renderer.domElement);

    envManager = new EnvironmentManager(scene);

    lanternManager = new LanternManager(scene, loader, {
      outerSrc: lanternOuterSrc,
      innerSrc: innerImageSrc,
      frameSrc: foregroundSrc,
    });

    clockManager = new ClockManager(
      scene,
      loader,
      parsedBackgrounds,
      clockBackgroundSrc,
    );

    candleLight = new THREE.PointLight(0xff9933, 1200, 300);
    scene.add(candleLight);
    scene.add(new THREE.AmbientLight(0xff6600, 0.3));

    onResize();
    animate();
    setupScrollTrigger();

    isInitialized = true;
  };

  const setupScrollTrigger = () => {
    gsap.registerPlugin(ScrollTrigger);
    ctx = gsap.context(() => {
      let h = Number(screenHeight);
      const isMobile = window.innerWidth < 768;
      if (isMobile) h = h * 0.7;
      if (stickyWrapper) stickyWrapper.style.height = `${h * 25}px`;

      gsap.to(scrollProgress, {
        value: 1,
        ease: "none",
        scrollTrigger: {
          trigger: stickyWrapper,
          start: "top top",
          end: "bottom bottom",
          pin: stickyInner,
          scrub: 0.1,
          onUpdate: (self) => {
            if (!userScrolled && self.progress > 0.005) {
              userScrolled = true;
            }
          },
        },
      });
    }, stickyWrapper);
  };

  const animate = () => {
    animationId = requestAnimationFrame(animate);
    const time = performance.now() / 1000;
    const isMobile = window.innerWidth < 768;

    const spinEnd = 0.45;
    const explodeEnd = 0.55;
    const clockPauseEnd = 0.6;

    let spinPhaseProgress = 0;
    let transitionFactor = 0;
    let highlightProgress = 0;
    let currentRotationY = 0;
    let foregroundRotationY = 0;
    let localActiveIndex = -1;

    const fullRotation = Math.PI * 2;

    if (!userScrolled) {
      currentRotationY = -time * 0.4;
      foregroundRotationY = 0;
      isClockMode = false;
      textVisible = false;
      showTransitionText = false;
      transitionFactor = 0;
      activeIndex = -1;
    } else {
      if (scrollProgress.value < spinEnd) {
        spinPhaseProgress = scrollProgress.value / spinEnd;
        isClockMode = false;
        textVisible = true;
        showTransitionText = false;
        transitionFactor = 0;
        highlightProgress = 0;
        currentRotationY = -(spinPhaseProgress * fullRotation);
        foregroundRotationY = currentRotationY;
        const totalTexts = parsedLanternTexts.length; 
        const raw = Math.floor(spinPhaseProgress * totalTexts);
        localActiveIndex = Math.min(raw, totalTexts - 1);

      } else if (scrollProgress.value < explodeEnd) {
        isClockMode = false;
        textVisible = false;
        showTransitionText = true;
        const p = (scrollProgress.value - spinEnd) / (explodeEnd - spinEnd);
        transitionFactor = p * p * (3 - 2 * p);
        currentRotationY = -fullRotation;
        foregroundRotationY = -fullRotation;
        localActiveIndex = -1;
      } else if (scrollProgress.value < clockPauseEnd) {
        isClockMode = true;
        textVisible = false;
        showTransitionText = false;
        transitionFactor = 1;
        currentRotationY = -fullRotation;
        foregroundRotationY = -fullRotation;
        localActiveIndex = -1;
      } else {
        isClockMode = true;
        textVisible = true;
        showTransitionText = false;
        transitionFactor = 1;
        const extraRot = (scrollProgress.value - clockPauseEnd) * 0.5;
        currentRotationY = -fullRotation - extraRot;
        foregroundRotationY = currentRotationY;

        const dur = 1.0 - clockPauseEnd;
        highlightProgress = (scrollProgress.value - clockPauseEnd) / dur;
        const count = 12;
        const raw = Math.floor(highlightProgress * count);
        localActiveIndex = Math.min(raw, count - 1);
        if (localActiveIndex < 0) localActiveIndex = 0;
      }
      activeIndex = localActiveIndex;
    }

    const currentZ =
      cameraZStart + (cameraZEnd - cameraZStart) * transitionFactor;
    camera.position.z = currentZ;
    camera.lookAt(0, 0, 0);

    if (envManager) envManager.update(transitionFactor, highlightProgress);
    if (lanternManager)
      lanternManager.update(
        time,
        currentRotationY,
        foregroundRotationY,
        transitionFactor,
        isMobile,
      );
    if (clockManager)
      clockManager.update(
        time,
        currentRotationY,
        transitionFactor,
        highlightProgress,
        activeIndex,
        isMobile,
      );

    renderer.render(scene, camera);
  };

  const kill = () => {
    if (!isInitialized) return;
    cancelAnimationFrame(animationId);
    if (ctx) {
      ctx.revert();
      ctx = null;
    }

    if (lanternManager) lanternManager.dispose();
    if (clockManager) clockManager.dispose();
    if (envManager) envManager.dispose();

    if (renderer) {
      renderer.dispose();
      if (canvasContainer) canvasContainer.innerHTML = "";
    }
    isInitialized = false;
  };

  const onResize = () => {
    if (isInitialized && camera && renderer) {
      const width = canvasContainer.clientWidth;
      const height = canvasContainer.clientHeight || 1;
      const isMobile = width < 768;
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height);
      if (isMobile) {
        cameraZStart = 60;
        cameraZEnd = 110;
      } else {
        cameraZStart = 40;
        cameraZEnd = 70;
      }
    }
  };

  onMount(() => {
    if (!browser) return;
    observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            if (!isInitialized) init();
          } else {
            if (isInitialized) kill();
          }
        });
      },
      { rootMargin: "100px 0px" },
    );
    if (stickyWrapper) observer.observe(stickyWrapper);
    window.addEventListener("resize", onResize);
    return () => {
      window.removeEventListener("resize", onResize);
      if (observer) observer.disconnect();
      kill();
    };
  });
</script>

<section class="revolvingLatternCtn">
  <div class="revolvingLattern">
    <div class="stickyWrapper" bind:this={stickyWrapper}>
      <div class="stickyInner" bind:this={stickyInner}>
        <div class="stickyBackground">
          <div class="canvasContainer" bind:this={canvasContainer}></div>
        </div>

        <div class="foregroundTextContainer">
          {#each currentTextList as text, i}
            <div
              class="textBox {i === activeIndex && textVisible
                ? 'active'
                : ''} {isClockMode ? 'clockMode' : 'lanternMode'}"
            >
              <p>{text}</p>
            </div>
          {/each}

          <div
            class="transitionTextBox"
            style="opacity: {showTransitionText ? 1 : 0}"
          >
            <p>{transitionText}</p>
          </div>
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
    font-family: "Roboto", sans-serif;
  }

  .revolvingLattern {
    width: 100%;
    margin: 0 auto;
    background-color: #220500;
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
    pointer-events: none;
  }

  .stickyBackground {
    z-index: 5;
  }
  .stickyForeground {
    z-index: 10;
  }

  .canvasContainer {
    width: 100%;
    height: 100%;
  }

  .foregroundTextContainer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 20;
    display: flex;
    justify-content: center;
    align-items: center;
    pointer-events: none;
  }

  .textBox {
    position: absolute;
    width: 60%;
    max-width: 400px;
    text-align: center;
    color: #ffbf51;
    font-size: 16px;
    opacity: 0;
    transform: translateY(20px);
    transition:
      opacity 0.3s ease,
      transform 0.3s ease;
    padding: 20px;
    border-radius: 10px;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.8);
  }

  .lanternMode {
    top: 40px;
  }

  .textBox.active {
    opacity: 1;
    transform: translateY(0);
  }

  .transitionTextBox {
    position: absolute;
    width: 50%;
    max-width: 600px;
    text-align: center;
    color: #ffffff;
    font-size: 16px;
    line-height: 25px;
    transition: opacity 0.5s ease;
    padding: 20px;
    border-radius: 10px;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.8);
    z-index: 25;
    background-color: rgba(34, 5, 0, 0.8);
  }

  /* UPDATED CSS for softer vignette */
  .candleOverlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    background: radial-gradient(
      circle at center,
      rgba(255, 220, 150, 0.05) 0%,
      rgba(50, 20, 0, 0.2) 60%,
      rgba(30, 10, 5, 0.5) 100%
    );
    mix-blend-mode: multiply;
  }

  @media (max-width: 768px) {
    .textBox {
      width: 90%;
      font-size: 14px;
      padding: 10px;
    }
    .lanternMode {
      top: 60px;
    }
    .transitionTextBox {
      width: 85%;
      font-size: 14px;
    }
  }
</style>
