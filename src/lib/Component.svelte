<svelte:options customElement={{ tag: "lantern-wc", shadow: "open" }} />

<script>
  import { onMount } from "svelte";
  import * as THREE from "three";
  import { gsap } from "gsap";
  import { ScrollTrigger } from "gsap/ScrollTrigger";

  const browser = typeof window !== "undefined";

  export let foregroundSrc = "foregroundImg.png";
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
    "Inner Circle 1",
    "Inner Circle 2",
    "Inner Circle 3",
    "Inner Circle 4",
    "Inner Circle 5",
    "Inner Circle 6",
    "Outer Circle 1",
    "Outer Circle 2",
    "Outer Circle 3",
    "Outer Circle 4",
    "Outer Circle 5",
    "Outer Circle 6",
    "Ignored Item",
  ];

  export let annotationSrc = [
    "Zi Shi 子时",
    "Chou Shi 丑时",
    "Jin Shi 寅时",
    "Mu Shi 卯时",
    "Chen Shi 辰时",
    "Si Shi 巳时",
    "Wu Shi 午时",
    "Wei Shi 未时",
    "Shen Shi 申时",
    "You Shi 酉时",
    "Xu Shi 戌时",
    "Hai Shi 亥时",
  ];

  export let transitionText = "In traditional Chinese timekeeping,";
  export let scrollspeed = 1;
  export let screenHeight = 1200;

  let stickyWrapper, stickyInner, canvasContainer, candleOverlay;
  let scene, camera, renderer, bgGroup, fgMesh, candleLight, loader;
  let innerClockMesh, outerClockMesh;
  let clockTicksGroup;
  let animationId;
  let ctx;
  let isInitialized = false;
  let observer;

  let userScrolled = false;
  let autoRotationAngle = 0;
  let activeIndex = 0;
  let isClockMode = false;
  let textVisible = true;
  let showTransitionText = false;

  const scrollProgress = { value: 0 };

  const parseInput = (input, maxLen = 13) => {
    if (!input) return [];
    let items = [];
    if (Array.isArray(input)) {
      items = input;
    } else if (typeof input === "string") {
      try {
        const parsed = JSON.parse(input);
        if (Array.isArray(parsed)) items = parsed;
        else items = [input];
      } catch (e) {
        items = input
          .split(",")
          .map((s) => s.trim())
          .filter((s) => s.length > 0);
      }
    }
    if (items.length === 0) return [];
    const result = [];
    for (let i = 0; i < maxLen; i++) {
      result.push(items[i % items.length]);
    }
    return result;
  };

  const loadTexture = (url, isPanel = false) => {
    if (!loader || !url || typeof url !== "string") return null;
    const tex = loader.load(url);
    tex.colorSpace = THREE.SRGBColorSpace;
    if (isPanel) {
      tex.wrapS = THREE.ClampToEdgeWrapping;
      tex.wrapT = THREE.ClampToEdgeWrapping;
      tex.repeat.set(1, 1);
    } else {
      tex.wrapS = THREE.RepeatWrapping;
      tex.wrapT = THREE.ClampToEdgeWrapping;
      tex.repeat.set(2, 1);
    }
    return tex;
  };

  const createLabel = (text, options = {}) => {
    const fontSize = options.fontSize || 60;
    const color = options.color || "#ffaa33";
    const font = options.font || "serif";
    const align = options.align || "center";

    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");

    canvas.width = 256;
    canvas.height = 128;

    ctx.font = `bold ${fontSize}px ${font}`;
    ctx.fillStyle = color;
    ctx.textAlign = align;
    ctx.textBaseline = "middle";

    ctx.fillText(text, canvas.width / 2, canvas.height / 2);

    const texture = new THREE.CanvasTexture(canvas);
    texture.colorSpace = THREE.SRGBColorSpace;
    texture.minFilter = THREE.LinearFilter;

    const material = new THREE.MeshBasicMaterial({
      map: texture,
      transparent: true,
      side: THREE.DoubleSide,
      depthWrite: false,
    });

    const aspect = canvas.width / canvas.height;
    const height = options.worldScale || 2;
    const width = height * aspect;

    const geometry = new THREE.PlaneGeometry(width, height);
    const mesh = new THREE.Mesh(geometry, material);

    return mesh;
  };

  const createTickMesh = (width, height, color) => {
    const geometry = new THREE.PlaneGeometry(width, height);
    const material = new THREE.MeshBasicMaterial({
      color: color,
      side: THREE.DoubleSide,
      transparent: true,
      depthWrite: false,
    });
    return new THREE.Mesh(geometry, material);
  };

  const init = () => {
    if (isInitialized || !canvasContainer) return;

    loader = new THREE.TextureLoader();
    loader.setCrossOrigin("anonymous");

    scene = new THREE.Scene();
    scene.background = null;

    const width = canvasContainer.clientWidth;
    const height = canvasContainer.clientHeight || 1;

    camera = new THREE.PerspectiveCamera(50, width / height, 0.1, 100);
    camera.position.set(0, 0, 35);
    camera.lookAt(0, 0, 0);

    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(width, height);
    renderer.outputColorSpace = THREE.SRGBColorSpace;

    canvasContainer.innerHTML = "";
    canvasContainer.appendChild(renderer.domElement);

    bgGroup = new THREE.Group();
    scene.add(bgGroup);

    const bgUrls = parseInput(backgroundSrc);
    const annotationTexts = parseInput(annotationSrc);
    const totalCount = bgUrls.length;
    const clockCount = 12;
    const radius = 9;
    const panelHeight = 7;
    const panelWidth = 2 * radius * Math.sin(Math.PI / totalCount) * 1.02;
    const innerRadius = 14;
    const outerRadius = 24;
    const innerCount = 6;
    const thickness = 0.3;
    const ringMaterial = new THREE.MeshBasicMaterial({
      color: 0xffaa33,
      side: THREE.DoubleSide,
      transparent: true,
      opacity: 0,
      depthWrite: false,
    });

    const innerRingGeo = new THREE.RingGeometry(
      innerRadius - thickness / 2,
      innerRadius + thickness / 2,
      128,
    );
    innerClockMesh = new THREE.Mesh(innerRingGeo, ringMaterial);
    innerClockMesh.position.z = -0.5;
    innerClockMesh.visible = false;
    bgGroup.add(innerClockMesh);

    const outerRingGeo = new THREE.RingGeometry(
      outerRadius - thickness / 2,
      outerRadius + thickness / 2,
      128,
    );
    outerClockMesh = new THREE.Mesh(outerRingGeo, ringMaterial.clone());
    outerClockMesh.position.z = -0.5;
    outerClockMesh.visible = false;
    bgGroup.add(outerClockMesh);

    clockTicksGroup = new THREE.Group();
    clockTicksGroup.visible = false;
    bgGroup.add(clockTicksGroup);

    // --- INNER RING: 0 to 11 ---
    for (let i = 0; i < 12; i++) {
      const angle = Math.PI / 2 - (i / 12) * Math.PI * 2;
      const r = innerRadius + 2.5;

      // Even numbers display text, Odd numbers display a tick
      if (i % 2 === 0) {
        const tickMesh = createLabel(i.toString(), {
          fontSize: 60,
          worldScale: 2.5,
        });
        tickMesh.position.x = Math.cos(angle) * r;
        tickMesh.position.y = Math.sin(angle) * r;
        tickMesh.position.z = -0.5;
        clockTicksGroup.add(tickMesh);
      } else {
        const tickMesh = createTickMesh(0.15, 0.6, 0xffaa33);
        tickMesh.position.x = Math.cos(angle) * r;
        tickMesh.position.y = Math.sin(angle) * r;
        tickMesh.position.z = -0.5;
        // Rotate tick to point towards center
        tickMesh.rotation.z = angle + Math.PI / 2;
        clockTicksGroup.add(tickMesh);
      }
    }

    // --- OUTER RING: 12 to 23 ---
    for (let i = 12; i < 24; i++) {
      const angle = Math.PI / 2 - (i / 12) * Math.PI * 2;
      const r = outerRadius + 2.5;

      // Even numbers display text (12, 14...), Odd numbers display tick (13, 15...)
      if (i % 2 === 0) {
        const tickMesh = createLabel(i.toString(), {
          fontSize: 60,
          worldScale: 2.5,
        });
        tickMesh.position.x = Math.cos(angle) * r;
        tickMesh.position.y = Math.sin(angle) * r;
        tickMesh.position.z = -0.5;
        clockTicksGroup.add(tickMesh);
      } else {
        const tickMesh = createTickMesh(0.15, 0.6, 0xffaa33);
        tickMesh.position.x = Math.cos(angle) * r;
        tickMesh.position.y = Math.sin(angle) * r;
        tickMesh.position.z = -0.5;
        // Rotate tick to point towards center
        tickMesh.rotation.z = angle + Math.PI / 2;
        clockTicksGroup.add(tickMesh);
      }
    }

    for (let i = 0; i < totalCount; i++) {
      const url = bgUrls[i];
      const tex = loadTexture(url, true);
      const geo = new THREE.PlaneGeometry(panelWidth, panelHeight);
      const mat = new THREE.MeshBasicMaterial({
        map: tex,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 1,
        color: 0xffffff,
      });
      const mesh = new THREE.Mesh(geo, mat);

      const angleStart = (i / totalCount) * Math.PI * 2;
      mesh.position.x = Math.sin(angleStart) * radius;
      mesh.position.z = Math.cos(angleStart) * radius;
      mesh.rotation.y = angleStart;
      const labelText = annotationTexts[i] || `Image ${i}`;
      const labelMesh = createLabel(labelText, {
        fontSize: 50,
        worldScale: 1.5,
        color: "#ffffff",
      });
      labelMesh.position.y = -(panelHeight / 2) - 1.2;
      labelMesh.position.z = 0.1;
      mesh.add(labelMesh);
      mesh.userData.labelMesh = labelMesh;

      let clockAngle = 0;
      let clockX = 0;
      let clockY = 0;
      let isClockVisible = false;

      if (i < clockCount) {
        isClockVisible = true;
        let r, countInRing, indexInRing, offsetAngle;

        if (i < innerCount) {
          r = innerRadius;
          countInRing = innerCount;
          indexInRing = i;
          offsetAngle = 0;
        } else {
          r = outerRadius;
          countInRing = clockCount - innerCount;
          indexInRing = i - innerCount;
          offsetAngle = ((Math.PI * 2) / countInRing) * 0.5;
        }

        clockAngle =
          Math.PI / 2 - (indexInRing / countInRing) * Math.PI * 2 - offsetAngle;
        clockX = Math.cos(clockAngle) * r;
        clockY = Math.sin(clockAngle) * r;
      } else {
        isClockVisible = false;
      }

      mesh.userData = {
        id: i,
        cylAngle: angleStart,
        cylRadius: radius,
        clockX: clockX,
        clockY: clockY,
        isClockVisible: isClockVisible,
        labelMesh: labelMesh,

        initialY: mesh.position.y,
        bobSpeed: 1 + Math.random() * 1.5,
        bobOffset: Math.random() * Math.PI * 2,
        bobAmp: 0.15 + Math.random() * 0.1,
      };

      bgGroup.add(mesh);
    }

    const fgTex = loadTexture(foregroundSrc, false);
    const fgGeo = new THREE.CylinderGeometry(15, 15, 10, 64, 1, true);
    const fgMat = new THREE.MeshStandardMaterial({
      map: fgTex,
      side: THREE.DoubleSide,
      transparent: true,
      opacity: 0.95,
      emissive: 0xff5500,
      emissiveIntensity: 0.5,
      alphaTest: 0.1,
      depthWrite: false,
    });
    fgMesh = new THREE.Mesh(fgGeo, fgMat);
    fgMesh.renderOrder = 1;
    scene.add(fgMesh);

    candleLight = new THREE.PointLight(0xff6600, 100, 45);
    scene.add(candleLight);
    scene.add(new THREE.AmbientLight(0xffaa33, 0.8));

    const clock = new THREE.Clock();

    const animate = () => {
      animationId = requestAnimationFrame(animate);
      const time = clock.getElapsedTime();

      const unfoldStart = 0.5;
      const highlightStart = 0.6;
      const activeClockStart = 0.75;

      let spinPhaseProgress = 0;
      let transitionFactor = 0;
      let highlightProgress = 0;

      if (scrollProgress.value < unfoldStart) {
        isClockMode = false;
        textVisible = true;
        showTransitionText = false;
        spinPhaseProgress = scrollProgress.value / unfoldStart;
        transitionFactor = 0;
        highlightProgress = 0;
      } else if (scrollProgress.value < highlightStart) {
        isClockMode = false;
        textVisible = false;
        showTransitionText = true;
        spinPhaseProgress = 1;
        transitionFactor =
          (scrollProgress.value - unfoldStart) / (highlightStart - unfoldStart);
        transitionFactor =
          transitionFactor * transitionFactor * (3 - 2 * transitionFactor);
        highlightProgress = 0;
      } else if (scrollProgress.value < activeClockStart) {
        isClockMode = true;
        textVisible = false;
        showTransitionText = false;
        spinPhaseProgress = 1;
        transitionFactor = 1;
        highlightProgress = 0;
      } else {
        isClockMode = true;
        textVisible = true;
        showTransitionText = false;
        spinPhaseProgress = 1;
        transitionFactor = 1;
        highlightProgress =
          (scrollProgress.value - activeClockStart) / (1.0 - activeClockStart);
      }

      let currentRotationY = 0;
      if (!userScrolled) {
        currentRotationY = -time * 0.2 * Number(scrollspeed);
        autoRotationAngle = currentRotationY;
      } else {
        const totalRotation = Math.PI * 2;
        const scrollRot = -(spinPhaseProgress * totalRotation);
        currentRotationY = autoRotationAngle + scrollRot;
      }

      const startZ = 35;
      const endZ = 65;
      camera.position.z = startZ + (endZ - startZ) * transitionFactor;
      camera.lookAt(0, 0, 0);

      let clockActiveIndex = -1;

      if (
        isClockMode &&
        transitionFactor >= 1 &&
        scrollProgress.value >= activeClockStart
      ) {
        const rawIdx = Math.floor(highlightProgress * clockCount);
        clockActiveIndex = Math.min(rawIdx, clockCount - 1);
        if (clockActiveIndex < 0) clockActiveIndex = 0;
        activeIndex = clockActiveIndex;
      } else if (!isClockMode) {
        let normalizedRot = -currentRotationY % (Math.PI * 2);
        if (normalizedRot < 0) normalizedRot += Math.PI * 2;
        const anglePerPanel = (Math.PI * 2) / totalCount;
        let rawIndex = Math.round(normalizedRot / anglePerPanel);
        activeIndex = rawIndex % totalCount;
      } else {
        activeIndex = -1;
      }

      const clockElementsOpacity = Math.pow(transitionFactor, 2);

      if (innerClockMesh && outerClockMesh) {
        if (transitionFactor > 0) {
          innerClockMesh.visible = true;
          outerClockMesh.visible = true;
          innerClockMesh.material.opacity = clockElementsOpacity * 0.8;
          outerClockMesh.material.opacity = clockElementsOpacity * 0.8;
        } else {
          innerClockMesh.visible = false;
          outerClockMesh.visible = false;
        }
      }

      if (clockTicksGroup) {
        if (transitionFactor > 0) {
          clockTicksGroup.visible = true;
          clockTicksGroup.children.forEach((tick) => {
            tick.material.opacity = clockElementsOpacity * 0.7;
          });
        } else {
          clockTicksGroup.visible = false;
        }
      }

      if (bgGroup) {
        bgGroup.rotation.y = currentRotationY * (1 - transitionFactor);
        bgGroup.children.forEach((child) => {
          if (
            child === innerClockMesh ||
            child === outerClockMesh ||
            child === clockTicksGroup
          )
            return;

          const u = child.userData;
          const bobY = Math.sin(time * u.bobSpeed + u.bobOffset) * u.bobAmp;

          const cylX = Math.sin(u.cylAngle) * u.cylRadius;
          const cylZ = Math.cos(u.cylAngle) * u.cylRadius;
          const cylRotY = u.cylAngle;
          const cylY = u.initialY + bobY;

          const clockX = u.clockX;
          const clockY = u.clockY + bobY;
          const clockZ = 0;
          const clockRotY = 0;

          child.position.x =
            cylX * (1 - transitionFactor) + clockX * transitionFactor;
          child.position.y =
            cylY * (1 - transitionFactor) + clockY * transitionFactor;
          child.position.z =
            cylZ * (1 - transitionFactor) + clockZ * transitionFactor;
          child.rotation.y =
            cylRotY * (1 - transitionFactor) + clockRotY * transitionFactor;

          if (u.labelMesh) {
            if (transitionFactor > 0.5) {
              u.labelMesh.visible = true;
              u.labelMesh.material.opacity = (transitionFactor - 0.5) * 2;
            } else {
              u.labelMesh.visible = false;
            }
          }

          if (transitionFactor > 0.9) {
            if (!u.isClockVisible) {
              child.visible = false;
            } else {
              child.visible = true;
              if (
                highlightProgress > 0 &&
                scrollProgress.value >= activeClockStart
              ) {
                const isActive = u.id === clockActiveIndex;
                const targetScale = isActive ? 1.3 : 0.85;
                const targetOpacity = isActive ? 1.0 : 0.3;
                const lerpSpeed = 0.1;
                child.scale.setScalar(
                  child.scale.x + (targetScale - child.scale.x) * lerpSpeed,
                );
                child.material.opacity +=
                  (targetOpacity - child.material.opacity) * lerpSpeed;
                child.renderOrder = isActive ? 10 : 5;
              } else {
                child.scale.setScalar(1);
                child.material.opacity = 1;
                child.renderOrder = 5;
              }
            }
          } else {
            child.visible = true;
            child.scale.setScalar(1);
            child.material.opacity = 1;
            child.renderOrder = 0;
          }
        });
      }

      if (fgMesh) {
        fgMesh.rotation.y = currentRotationY * 0.8;
        if (fgMesh.material)
          fgMesh.material.opacity = 0.95 * (1 - transitionFactor);
      }

      renderer.render(scene, camera);
    };
    animate();

    gsap.registerPlugin(ScrollTrigger);
    ctx = gsap.context(() => {
      let h = Number(screenHeight);
      if (window.innerWidth < 768) h = h * 0.8;
      if (stickyWrapper)
        stickyWrapper.style.height = `${h * (totalCount * 0.9)}px`;

      gsap.to(scrollProgress, {
        value: 1,
        ease: "none",
        scrollTrigger: {
          trigger: stickyWrapper,
          start: "top top",
          end: "bottom bottom",
          pin: stickyInner,
          scrub: 0.5,
          onUpdate: (self) => {
            if (!userScrolled && self.progress > 0.01) {
              userScrolled = true;
            }
          },
        },
      });
    }, stickyWrapper);

    isInitialized = true;
  };

  const kill = () => {
    if (!isInitialized) return;
    cancelAnimationFrame(animationId);
    if (ctx) {
      ctx.revert();
      ctx = null;
    }
    if (bgGroup) {
      while (bgGroup.children.length > 0) {
        const child = bgGroup.children[0];
        bgGroup.remove(child);
        if (child.geometry) child.geometry.dispose();
        if (child.material) child.material.dispose();
      }
    }
    if (innerClockMesh) {
      innerClockMesh.geometry.dispose();
      innerClockMesh.material.dispose();
    }
    if (outerClockMesh) {
      outerClockMesh.geometry.dispose();
      outerClockMesh.material.dispose();
    }
    if (clockTicksGroup) {
      while (clockTicksGroup.children.length > 0) {
        const t = clockTicksGroup.children[0];
        clockTicksGroup.remove(t);
        if (t.material.map) t.material.map.dispose();
        if (t.material) t.material.dispose();
        if (t.geometry) t.geometry.dispose();
      }
    }

    if (fgMesh) {
      fgMesh.geometry.dispose();
      fgMesh.material.dispose();
    }
    if (renderer) {
      renderer.dispose();
      if (canvasContainer) canvasContainer.innerHTML = "";
    }
    isInitialized = false;
  };

  const onResize = () => {
    if (isInitialized && camera && renderer && canvasContainer) {
      const width = canvasContainer.clientWidth;
      const height = canvasContainer.clientHeight || 1;
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height);
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

  $: parsedLanternTexts = parseInput(textSrc, 13);
  $: parsedClockTexts = parseInput(clockTextSrc, 13);
  $: annotationTexts = parseInput(annotationSrc, 13);
  $: currentTextList = isClockMode ? parsedClockTexts : parsedLanternTexts;
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

  .clockMode {
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
    font-size: 16;
    line-height: 25px;
    transition: opacity 0.5s ease;
    padding: 20px;
    border-radius: 10px;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.8);
    z-index: 25;
    background-color: #220500;
  }

  .candleOverlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    background: radial-gradient(
      circle at center,
      rgba(255, 200, 100, 0.1) 10%,
      rgba(50, 10, 0, 0.3) 50%,
      rgba(20, 5, 0, 1) 90%
    );
    mix-blend-mode: multiply;
  }
</style>