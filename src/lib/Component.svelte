<svelte:options customElement={{ tag: "kaileidoscope-wc", shadow: "open" }} />

<script>
	import { onMount } from 'svelte';
	import { fade, fly } from 'svelte/transition';

	let canvasContainer;
	let showAnswer = false;
	let currentAnswer = "";

	export let imageUrls = [
		'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/1.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/2.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/3.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/4.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/5.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/6.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/7.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/8.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/9.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/10.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/11.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/12.png',
    'https://multimedia.scmp.com/components/2026/kaileidoscope-wc/13.png'
  ];

	const answers = [
		"Yes, definitely.",
		"The outcome is promising.",
		"Trust your intuition.",
		"Wait for a clearer sign.",
		"Let it go.",
		"The answer lies in the past.",
		"Focus on the present.",
		"Unexpected joy is coming.",
		"Look closer.",
		"It is certain.",
		"Ask again later.",
		"Follow your heart."
	];

	const SYMMETRY = 6;
	const SLICE_ANGLE = 360 / SYMMETRY;
	const SHARD_COUNT = 12; 
	const GRAVITY_STRENGTH = 1.5;
	const FRICTION = 0.85;
	const SLEEP_THRESHOLD = 0.2;
	const BOUNCE_DAMPING = 0.3; 
	const IMPACT_THRESHOLD = 5; 

	let shakeIntensity = 0;
	let isSpinning = false;

	function triggerSpinStart() {
		showAnswer = false; 
	}

	function triggerSpinEnd() {
		const randomIndex = Math.floor(Math.random() * answers.length);
		currentAnswer = answers[randomIndex];
		showAnswer = true;
	}

	class Shard {
		constructor(p, maxR, img) {
			this.p = p;
			this.img = img; 
			this.r = p.random(40, 60); 
			
			let d = p.random(maxR * 0.2, maxR * 0.6);
			let a = p.random(0, SLICE_ANGLE);
			this.pos = p.createVector(d * p.cos(a), d * p.sin(a));
			this.vel = p.createVector(0, 0);
			this.acc = p.createVector(0, 0);

			this.vertices = [];
			let sides = p.floor(p.random(4, 6)); 
			for (let i = 0; i < sides; i++) {
				let angle = p.map(i, 0, sides, 0, 360) + p.random(-20, 20);
				let dist = this.r * p.random(0.85, 1.15); 
				this.vertices.push({x: dist * p.cos(angle), y: dist * p.sin(angle)});
			}
			
			this.paperColor = p.color('#f4e4bc'); 
		}

		applyForce(fx, fy) {
			this.acc.add(fx, fy);
		}

		update() {
			this.vel.add(this.acc);
			this.pos.add(this.vel);
			this.acc.mult(0);
			this.vel.mult(FRICTION); 
			if (this.vel.mag() < SLEEP_THRESHOLD) this.vel.set(0, 0);
		}

		collide(other) {
			let d = this.p.dist(this.pos.x, this.pos.y, other.pos.x, other.pos.y);
			let minDist = (this.r + other.r) * 0.85; 

			if (d < minDist) {
				let angle = this.p.atan2(other.pos.y - this.pos.y, other.pos.x - this.pos.x);
				let overlap = minDist - d;
				let fx = this.p.cos(angle) * overlap * 0.5;
				let fy = this.p.sin(angle) * overlap * 0.5;
				
				this.pos.sub(fx, fy);
				other.pos.add(fx, fy);

				let avgVelX = (this.vel.x + other.vel.x) / 2;
				let avgVelY = (this.vel.y + other.vel.y) / 2;
				this.vel.set(avgVelX * 0.6, avgVelY * 0.6);
				other.vel.set(avgVelX * 0.6, avgVelY * 0.6);
			}
		}

		checkBoundaries(maxR) {
			if (this.pos.mag() > maxR - this.r) {
				let speed = this.vel.mag();
				if (speed > IMPACT_THRESHOLD) {
					shakeIntensity = Math.min(shakeIntensity + speed * 0.8, 20);
				}
				this.pos.setMag(maxR - this.r);
				this.vel.mult(BOUNCE_DAMPING);
			}
			if (this.pos.y < this.r) {
				this.pos.y = this.r;
				this.vel.y = -Math.abs(this.vel.y) * BOUNCE_DAMPING;
			}
			let nx = -this.p.sin(SLICE_ANGLE);
			let ny = this.p.cos(SLICE_ANGLE);
			let distToWall = this.pos.x * nx + this.pos.y * ny;

			if (distToWall < this.r) {
				let overlap = this.r - distToWall;
				this.pos.x += nx * overlap;
				this.pos.y += ny * overlap;
				let dot = this.vel.x * nx + this.vel.y * ny;
				this.vel.x -= 2 * dot * nx;
				this.vel.y -= 2 * dot * ny;
				this.vel.mult(BOUNCE_DAMPING);
			}
		}

		show() {
			this.p.push();
			this.p.translate(this.pos.x, this.pos.y);
			
			this.p.fill(this.paperColor);
			this.p.noStroke();
			this.p.beginShape();
			for(let v of this.vertices) this.p.vertex(v.x, v.y);
			this.p.endShape(this.p.CLOSE);

			let ctx = this.p.drawingContext;
			ctx.save(); 
			ctx.beginPath();
			if (this.vertices.length > 0) {
				ctx.moveTo(this.vertices[0].x, this.vertices[0].y);
				for (let i = 1; i < this.vertices.length; i++) {
					ctx.lineTo(this.vertices[i].x, this.vertices[i].y);
				}
			}
			ctx.closePath();
			ctx.clip(); 

			if (this.img) {
				this.p.imageMode(this.p.CENTER);
				this.p.image(this.img, 0, 0, this.r * 1.5, this.r * 1.5);
			}
			
			ctx.restore(); 

			this.p.noFill();
			this.p.stroke(255, 60); 
			this.p.strokeWeight(1.5); 
			this.p.beginShape();
			for(let v of this.vertices) this.p.vertex(v.x, v.y);
			this.p.endShape(this.p.CLOSE);

			this.p.stroke(255, 100);
			this.p.strokeWeight(2);
			this.p.line(this.vertices[0].x, this.vertices[0].y, this.vertices[2].x, this.vertices[2].y);

			this.p.pop();
		}
	}

	onMount(() => {
		let p5Instance;

		const loadP5 = async () => {
			const p5Module = await import('p5');
			const p5 = p5Module.default;

			const sketch = (p) => {
				let shards = [];
				let loadedImages = [];
				let scopeRotation = 0;
				let scopeRadius = 0;
				let hexPoints = []; 
				let previousMouseAngle = 0;
				let isLoaded = false;

				p.setup = async () => {
					p.createCanvas(window.innerWidth, window.innerHeight);
					p.angleMode(p.DEGREES);
					p.noStroke();
					
					p.background('#f9f4e8'); 
					p.fill(50);
					p.textAlign(p.CENTER, p.CENTER);
					p.text("Loading Book of Answers...", p.width/2, p.height/2);

					scopeRadius = Math.min(p.width, p.height) * 0.4;
					
					for (let i = 0; i < 6; i++) {
						let angle = 60 * i - 30; 
						let rx = p.cos(angle) * (scopeRadius * 0.98);
						let ry = p.sin(angle) * (scopeRadius * 0.98);
						hexPoints.push({x: rx, y: ry});
					}

					try {
						for (let i = 0; i < imageUrls.length; i++) {
							const img = await p.loadImage(imageUrls[i]);
							loadedImages.push(img);
						}
					} catch (e) {
						console.error("Error loading images:", e);
					}

					for (let i = 0; i < SHARD_COUNT; i++) {
						let img = loadedImages[i % loadedImages.length];
						shards.push(new Shard(p, scopeRadius, img));
					}
					
					isLoaded = true;
				};

				p.mousePressed = () => {
					isSpinning = true;
					triggerSpinStart();
					let dx = p.mouseX - p.width / 2;
					let dy = p.mouseY - p.height / 2;
					previousMouseAngle = p.degrees(p.atan2(dy, dx));
				}

				p.mouseReleased = () => {
					isSpinning = false;
					triggerSpinEnd(); 
				}

				p.draw = () => {
					if (!isLoaded) return;

					p.background('#f9f4e8'); 

					if (isSpinning) {
						let dx = p.mouseX - p.width / 2;
						let dy = p.mouseY - p.height / 2;
						let currentMouseAngle = p.degrees(p.atan2(dy, dx));
						let delta = currentMouseAngle - previousMouseAngle;
						if (delta > 180) delta -= 360;
						if (delta < -180) delta += 360;
						scopeRotation += delta;
						previousMouseAngle = currentMouseAngle;
					}

					let gravityAngle = 90 - scopeRotation;
					let gx = p.cos(gravityAngle) * GRAVITY_STRENGTH;
					let gy = p.sin(gravityAngle) * GRAVITY_STRENGTH;

					for(let step=0; step<4; step++) {
						for (let shard of shards) {
							shard.applyForce(gx, gy);
							shard.update();
							shard.checkBoundaries(scopeRadius);
						}
						for (let i = 0; i < shards.length; i++) {
							for (let j = i + 1; j < shards.length; j++) {
								shards[i].collide(shards[j]);
							}
						}
					}

					p.push();
					p.translate(p.width / 2, p.height / 2);

					if (shakeIntensity > 0) {
						let shakeX = p.random(-shakeIntensity, shakeIntensity);
						let shakeY = p.random(-shakeIntensity, shakeIntensity);
						p.translate(shakeX, shakeY);
						shakeIntensity *= 0.85;
						if(shakeIntensity < 0.5) shakeIntensity = 0;
					}

					p.rotate(scopeRotation);

					p.fill(0); 
					p.noStroke();
					p.beginShape();
					for(let pt of hexPoints) p.vertex(pt.x, pt.y);
					p.endShape(p.CLOSE);

					for (let i = 0; i < SYMMETRY; i++) {
						p.push();
						p.rotate(i * SLICE_ANGLE);
						if (i % 2 === 1) {
							p.scale(1, -1);
							p.rotate(-SLICE_ANGLE);
						}
						for (let shard of shards) shard.show();
						p.pop();
					}

					p.fill('#f9f4e8'); 
					p.noStroke();
					p.beginShape();
					let w = p.width * 2; 
					let h = p.height * 2;
					p.vertex(-w, -h);
					p.vertex(w, -h);
					p.vertex(w, h);
					p.vertex(-w, h);
					
					p.beginContour();
					for (let i = 5; i >= 0; i--) {
						p.vertex(hexPoints[i].x, hexPoints[i].y);
					}
					p.endContour();
					p.endShape(p.CLOSE);

					p.noFill();
					p.stroke(20); 
					p.strokeWeight(10);
					p.beginShape();
					for (let pt of hexPoints) p.vertex(pt.x, pt.y);
					p.endShape(p.CLOSE);
					
					p.pop();
				};
				
				p.windowResized = () => {
					p.resizeCanvas(window.innerWidth, window.innerHeight);
					scopeRadius = Math.min(p.width, p.height) * 0.4;
					
					hexPoints = [];
					for (let i = 0; i < 6; i++) {
						let angle = 60 * i - 30;
						let rx = p.cos(angle) * (scopeRadius * 0.98);
						let ry = p.sin(angle) * (scopeRadius * 0.98);
						hexPoints.push({x: rx, y: ry});
					}
				};
			};

			if (canvasContainer) {
				p5Instance = new p5(sketch, canvasContainer);
			}
		};

		loadP5();

		return () => {
			if (p5Instance) p5Instance.remove();
		};
	});
</script>

<div class="wrapper">
  {#if !showAnswer}
		<p class="ui-text" transition:fade>Spin the Scope to find your answer...</p>
	{/if}
	<div bind:this={canvasContainer} class="canvas-box"></div>
	{#if showAnswer}
		<div class="answer-overlay" transition:fly={{ y: 20, duration: 400 }}>
			<div class="answer-card">
				<div class="card-icon">✧</div>
				<h2>The Scope Says:</h2>
				<p class="answer-text">{currentAnswer}</p>
				<!-- <button class="close-btn" on:click={() => showAnswer = false}>Spin Again</button> -->
			</div>
		</div>
	{/if}
</div>

<style>
	.wrapper {
		width: 100%;
		height: 100vh;
		/* background-color: #f9f4e8; */
	}

	.canvas-box {
		width: 100%;
		height: 100%;
		display: block;
		cursor: grab;
	}
	
	.canvas-box:active {
		cursor: grabbing;
	}

	.ui-text {
		position: absolute;
		top: 80px;
		width: 100%;
		text-align: center;
		color: #555; 
		font-style: italic;
		pointer-events: none;
		user-select: none;
	}

	.answer-overlay {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		display: flex;
		justify-content: center;
		align-items: center;
		pointer-events: none;
		z-index: 100;
	}

	.answer-card {
		pointer-events: auto;
		background: #fffdf5cf;
		width: 300px;
		padding: 20px 10px;
		text-align: center;
		border: 1px solid #d4c5a3;
		box-shadow: 0 10px 40px rgba(88, 70, 40, 0.15);
		border-radius: 4px;
		/* background-image: linear-gradient(#f4e4bc 1px, transparent 1px);
		background-size: 100% 1.2em; */
	}

	.card-icon {
		font-size: 5rem;
		color: #bfa575;
		margin-bottom: 10px;
	}

	/* .answer-card h2 {
		font-size: 0.9rem;
		text-transform: uppercase;
		letter-spacing: 2px;
		color: #887;
		margin: 0 0 20px 0;
	} */

	.answer-text {
		font-size: 16px;
		color: #222;
		font-weight: bold;
		min-height: 3rem;
		display: flex;
		justify-content: center;
		align-items: center;
		margin-bottom: 30px;
	}

</style>