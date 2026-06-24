<svelte:options customElement={{ tag: "flashingbackground-wgc", shadow: "open" }} />

<script>
  import { onMount } from 'svelte';

  let starsCanvas;
  let animationFrameId;
  let resizeHandler;

  const PARTICLE_COUNT = 120;
  const BASE_COLOR = [255, 255, 255]; 
  const MIN_SIZE = 0.5;
  const MAX_SIZE = 1;
  const MIN_SPEED = 0.006;
  const MAX_SPEED = 0.002;

  class Particle {
    constructor(ctx) {
      this.ctx = ctx;
      this.canvas = ctx.canvas;
      this.x = Math.random() * this.canvas.width;
      this.y = Math.random() * this.canvas.height;
      this.size = Math.random() * (MAX_SIZE - MIN_SIZE) + MIN_SIZE;

      this.baseColor = [
        Math.max(0, Math.min(255, BASE_COLOR[0] + Math.random() * 40 - 20)),
        Math.max(0, Math.min(255, BASE_COLOR[1] + Math.random() * 40 - 20)),
        Math.max(0, Math.min(255, BASE_COLOR[2] + Math.random() * 40 - 20)),
      ];

      this.maxAlpha = 1;
      this.alpha = 0;
      this.speed = Math.random() * (MAX_SPEED - MIN_SPEED) + MIN_SPEED;
      this.fadingIn = true;
      this.blurAmount = Math.random() * 2;
    }

    update() {
      if (this.fadingIn) {
        this.alpha += this.speed;
        if (this.alpha >= this.maxAlpha) {
          this.fadingIn = false;
        }
      } else {
        this.alpha -= this.speed;
        if (this.alpha <= 0) {
          this.alpha = 0;
          this.fadingIn = true;
          this.x = Math.random() * this.canvas.width;
          this.y = Math.random() * this.canvas.height;
          this.blurAmount = Math.random() * 2;
        }
      }
    }

    draw() {
      const ctx = this.ctx;
      ctx.beginPath();
      ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
      const colorString = `rgba(${this.baseColor[0]}, ${this.baseColor[1]}, ${this.baseColor[2]}, ${this.alpha})`;

      ctx.fillStyle = colorString;
      ctx.shadowBlur = this.blurAmount;
      ctx.shadowColor = colorString;
      ctx.fill();
      ctx.shadowBlur = 0;
    }
  }

  onMount(() => {
    const ctx = starsCanvas.getContext("2d");

    function resizeCanvas() {
      if (starsCanvas) {
        starsCanvas.width = window.innerWidth;
        starsCanvas.height = window.innerHeight;
      }
    }
    
    resizeHandler = resizeCanvas;
    window.addEventListener("resize", resizeHandler);
    resizeCanvas();

    const particlesArray = [];

    function init() {
      for (let i = 0; i < PARTICLE_COUNT; i++) {
        particlesArray.push(new Particle(ctx));
      }
    }

    function animate() {
      if (!starsCanvas) return;
      ctx.clearRect(0, 0, starsCanvas.width, starsCanvas.height);
      for (let i = 0; i < particlesArray.length; i++) {
        particlesArray[i].update();
        particlesArray[i].draw();
      }
      animationFrameId = requestAnimationFrame(animate);
    }

    init();
    animate();

    return () => {
      window.removeEventListener("resize", resizeHandler);
      cancelAnimationFrame(animationFrameId);
    };
  });
</script>

<canvas bind:this={starsCanvas} id="starsCanvas"></canvas>

<style>
  :host {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    display: block;
    pointer-events: none;
    z-index: -1; 
    background-color: #000000; 
  }
  canvas {
    display: block;
    width: 100%;
    height: 100%;
  }
</style>