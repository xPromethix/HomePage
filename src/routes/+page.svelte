<script lang="ts">
		import { onMount, onDestroy } from 'svelte';
		import { resolve } from '$app/paths';
	import type {
		WebGLRenderer,
		Scene,
		OrthographicCamera,
		ShaderMaterial,
		PlaneGeometry,
		Mesh,
		Vector2
	} from 'three';

	let canvasWrapper: HTMLDivElement | null = null;
	let canvasEl: HTMLCanvasElement | null = null;

	let renderer: WebGLRenderer | undefined;
	let scene: Scene | undefined;
	let camera: OrthographicCamera | undefined;
	let material: ShaderMaterial | undefined;
	let geometry: PlaneGeometry | undefined;
	let plane: Mesh | undefined;
	let animationId = 0;
	let cleanup: (() => void) | undefined;

	onMount(async () => {
		const THREE = await import('three');

		if (!canvasWrapper || !canvasEl) return;

		scene = new THREE.Scene();
		camera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0.01, 10);
		camera.position.z = 1;

		renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, canvas: canvasEl });
		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
		renderer.setClearColor(0x000000, 0);

		const uniforms = {
			u_time: { value: 0 },
			u_mouse: { value: new THREE.Vector2(0.5, 0.5) as Vector2 },
			u_resolution: {
				value: new THREE.Vector2(canvasWrapper.clientWidth || 1, canvasWrapper.clientHeight || 1) as Vector2
			}
		} satisfies Record<string, { value: number | Vector2 }>;

		geometry = new THREE.PlaneGeometry(2, 2);

		const vertexShader = /* glsl */ `
			varying vec2 vUv;

			void main() {
				vUv = uv;
				gl_Position = vec4(position, 1.0);
			}
		`;

		const fragmentShader = /* glsl */ `
			precision highp float;

			uniform float u_time;
			uniform vec2 u_mouse;
			uniform vec2 u_resolution;
			varying vec2 vUv;

			#define PI 3.141592653589793

			float hash(vec2 p) {
				return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453123);
			}

			float noise(vec2 p) {
				vec2 i = floor(p);
				vec2 f = fract(p);

				float a = hash(i);
				float b = hash(i + vec2(1.0, 0.0));
				float c = hash(i + vec2(0.0, 1.0));
				float d = hash(i + vec2(1.0, 1.0));

				vec2 u = f * f * (3.0 - 2.0 * f);

				return mix(a, b, u.x) + (c - a) * u.y * (1.0 - u.x) + (d - b) * u.x * u.y;
			}

			vec3 palette(float t) {
				vec3 a = vec3(0.10, 0.20, 0.70);
				vec3 b = vec3(0.90, 0.30, 0.20);
				vec3 c = vec3(0.20, 0.80, 0.60);
				vec3 d = vec3(0.50, 0.20, 0.90);
				return a + b * cos(PI * (c * t + d));
			}

			void main() {
				vec2 uv = vUv;
				vec2 centeredUv = uv * 2.0 - 1.0;

				float time = u_time * 0.45;

				vec2 dir = normalize(u_mouse - uv + 0.0001);
				float dist = distance(uv, u_mouse);
				float ripple = sin(dist * 22.0 - time * 3.0) * exp(-dist * 6.0);

				vec2 flow = centeredUv * 1.6;
				float n = 0.0;
				float amplitude = 0.6;
				float frequency = 1.0;
				for (int i = 0; i < 4; i++) {
					n += amplitude * noise(flow * frequency + time * 0.22);
					flow += dir * 0.15;
					frequency *= 2.0;
					amplitude *= 0.5;
				}

				float intensity = n + ripple * 0.45;
				vec3 color = palette(intensity + dist * 0.35);

				color = mix(color, vec3(0.08, 0.06, 0.14), smoothstep(0.0, 0.9, dist));
				color += 0.08 * vec3(cos((centeredUv.x + time) * 3.0), sin((centeredUv.y - time) * 3.5), sin((centeredUv.x + centeredUv.y + time) * 2.5));

				gl_FragColor = vec4(color, 0.92);
			}
		`;

		material = new THREE.ShaderMaterial({
			uniforms,
			vertexShader,
			fragmentShader,
			transparent: true
		});

		plane = new THREE.Mesh(geometry, material);
		scene.add(plane);

			const resize = () => {
				if (!canvasWrapper || !renderer) return;
				const { clientWidth, clientHeight } = canvasWrapper;
				renderer.setSize(clientWidth, clientHeight, false);
				uniforms.u_resolution.value.set(clientWidth, clientHeight);
			};

			const updateMouse = (event: PointerEvent) => {
				if (!canvasWrapper) return;
				const rect = canvasWrapper.getBoundingClientRect();
				const x = (event.clientX - rect.left) / rect.width;
				const y = 1 - (event.clientY - rect.top) / rect.height;
				uniforms.u_mouse.value.set(x, y);
			};

		const resetMouse = () => {
			uniforms.u_mouse.value.set(0.5, 0.5);
		};

		resize();

		let prefersReducedMotion = false;
		const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
		prefersReducedMotion = mediaQuery.matches;

		const handleMotionChange = (event: MediaQueryListEvent) => {
			prefersReducedMotion = event.matches;
			if (prefersReducedMotion) {
				if (renderer && scene && camera) {
					renderer.render(scene, camera);
				}
			} else {
				cancelAnimationFrame(animationId);
				animationId = requestAnimationFrame(animate);
			}
		};

		mediaQuery.addEventListener('change', handleMotionChange);

		const animate = (time: number) => {
			uniforms.u_time.value = time * 0.001;
			if (renderer && scene && camera) {
				renderer.render(scene, camera);
			}
			if (!prefersReducedMotion) {
				animationId = requestAnimationFrame(animate);
			}
		};

		if (prefersReducedMotion) {
			if (renderer && scene && camera) {
				renderer.render(scene, camera);
			}
		} else {
			animationId = requestAnimationFrame(animate);
		}

		window.addEventListener('resize', resize);
		window.addEventListener('pointermove', updateMouse);
		window.addEventListener('pointerleave', resetMouse);

		cleanup = () => {
			mediaQuery.removeEventListener('change', handleMotionChange);
			window.removeEventListener('resize', resize);
			window.removeEventListener('pointermove', updateMouse);
			window.removeEventListener('pointerleave', resetMouse);
			cancelAnimationFrame(animationId);
			if (scene && plane) {
				scene.remove(plane);
			}
			geometry?.dispose();
			material?.dispose();
			renderer?.dispose();
			renderer = undefined;
			scene = undefined;
			camera = undefined;
			material = undefined;
			geometry = undefined;
			plane = undefined;
		};
	});

	onDestroy(() => {
		cleanup?.();
	});
</script>

<svelte:head>
	<title>Promethix | Fluid Futures in Motion</title>
	<meta
		name="description"
		content="Promethix crafts immersive, fluid-inspired digital experiences for ambitious brands."
	/>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Space+Grotesk:wght@400;500;600;700&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div class="page">
	<section class="hero">
		<div class="hero__canvas" bind:this={canvasWrapper}>
			<canvas bind:this={canvasEl}></canvas>
		</div>
		<div class="hero__content">
			<span class="eyebrow">Promethix</span>
			<h1>We orchestrate fluid, immersive worlds for ambitious brands.</h1>
			<p class="hero__lead">
				Promethix blends advanced simulation, spatial storytelling, and adaptive interfaces to launch
				experiences that feel alive. We help visionary teams visualize the future before it arrives.
			</p>
			<div class="hero__actions">
				<a class="btn btn--primary" href="#about">Discover Promethix</a>
			<a class="btn btn--outline" href={resolve('/contact')}>Contact</a>
			</div>
		</div>
	</section>

	<section id="about" class="about">
		<div class="about__intro">
			<h2>Designing the interface between innovation and emotion.</h2>
			<p>
				We are a multidisciplinary collective of technologists, artists, and strategists. With Three.js
				in our toolkit, we prototype atmospheres, interactions, and visual narratives that distill your
				boldest ideas into unforgettable digital touchpoints.
			</p>
		</div>

		<div class="about__grid">
			<article class="card">
				<h3 class="card__title">Immersive Brand Launches</h3>
				<p class="card__body">
					From experiential microsites to flagship portal experiences, we choreograph every pixel with
					cinematic pacing and fluid dynamics, revealing your story in motion.
				</p>
			</article>
			<article class="card">
				<h3 class="card__title">Realtime Visualization</h3>
				<p class="card__body">
					Our team fuses realtime rendering with data-driven narrative systems, translating complexity
					into clarity for executive briefings and investor demos.
				</p>
			</article>
			<article class="card">
				<h3 class="card__title">Experience Prototyping</h3>
				<p class="card__body">
					We iterate rapidly with interactive prototypes, ensuring the final experience balances
					awe-inspiring aesthetics with measurable business outcomes.
				</p>
			</article>
			<article class="card">
				<h3 class="card__title">Partnerships &amp; Labs</h3>
				<p class="card__body">
					Promethix embeds alongside your team, creating knowledge transfer programs and innovation labs
					that uplift internal capabilities long after launch.
				</p>
			</article>
		</div>
	</section>

	<footer class="footer">
		© {new Date().getFullYear()} Promethix. Fluid experiences, responsibly crafted.
	</footer>
</div>

<style>
	:global(body) {
		margin: 0;
		font-family: 'Inter', 'Noto Sans JP', system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI',
			sans-serif;
		color: rgba(240, 245, 255, 0.92);
		background: radial-gradient(circle at top, #0f1535 0%, #050510 50%, #020207 100%);
	}

	:global(*) {
		box-sizing: border-box;
	}

	:global(a) {
		color: inherit;
	}

	.page {
		display: flex;
		flex-direction: column;
		min-height: 100vh;
	}

	.hero {
		position: relative;
		min-height: 100vh;
		display: grid;
		place-items: center;
		padding: clamp(6rem, 10vh, 9rem) clamp(1.5rem, 6vw, 6rem);
		overflow: hidden;
	}

	.hero__canvas {
		position: absolute;
		inset: 0;
		z-index: 0;
	}

	.hero__canvas canvas {
		width: 100%;
		height: 100%;
		display: block;
		filter: saturate(130%) contrast(110%);
		pointer-events: none;
	}

	.hero__content {
		position: relative;
		z-index: 1;
		max-width: 62rem;
		display: grid;
		gap: 1.75rem;
		text-align: left;
	}

	.eyebrow {
		letter-spacing: 0.42em;
		text-transform: uppercase;
		font-size: 0.75rem;
		display: inline-block;
		color: rgba(255, 255, 255, 0.6);
		font-weight: 600;
		font-family: 'Space Grotesk', 'Inter', sans-serif;
	}

	h1 {
		margin: 0;
		font-size: clamp(2.9rem, 6vw, 5rem);
		line-height: 1.05;
		font-family: 'Space Grotesk', 'Inter', sans-serif;
		color: #f8f9ff;
	}

	h2 {
		font-size: clamp(2rem, 3.5vw, 3.1rem);
		line-height: 1.1;
		margin: 0;
		font-family: 'Space Grotesk', 'Inter', sans-serif;
	}

	.hero__lead {
		margin: 0;
		font-size: clamp(1.05rem, 2.1vw, 1.35rem);
		max-width: 40rem;
		color: rgba(220, 229, 255, 0.82);
		line-height: 1.65;
	}

	.hero__actions {
		display: flex;
		flex-wrap: wrap;
		gap: 1rem 1.5rem;
	}

	.btn {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		padding: 0.85rem 2.5rem;
		border-radius: 999px;
		font-weight: 600;
		letter-spacing: 0.02em;
		text-decoration: none;
		transition: transform 0.3s ease, box-shadow 0.3s ease, background 0.3s ease, border-color 0.3s ease;
	}

	.btn:hover {
		transform: translateY(-3px);
	}

	.btn:focus-visible {
		outline: 2px solid rgba(99, 173, 255, 0.75);
		outline-offset: 3px;
	}

	.btn--primary {
		background: linear-gradient(120deg, #66f7ff 0%, #7a5cff 100%);
		color: #04040b;
		box-shadow: 0 12px 35px rgba(102, 247, 255, 0.25);
	}

	.btn--primary:hover {
		box-shadow: 0 18px 45px rgba(123, 92, 255, 0.35);
	}

	.btn--outline {
		border: 1px solid rgba(120, 131, 255, 0.55);
		color: rgba(224, 229, 255, 0.9);
		background: rgba(12, 15, 40, 0.55);
		backdrop-filter: blur(8px);
	}

	.about {
		position: relative;
		padding: clamp(6rem, 12vw, 9rem) clamp(1.5rem, 8vw, 8rem);
		background: linear-gradient(180deg, rgba(12, 12, 24, 0.95) 0%, rgba(6, 6, 14, 1) 100%);
		display: grid;
		gap: 4rem;
	}

	.about__intro {
		max-width: 48rem;
		display: grid;
		gap: 1.6rem;
	}

	.about__intro p {
		margin: 0;
		color: rgba(210, 218, 255, 0.8);
		line-height: 1.7;
		font-size: 1.02rem;
	}

	.about__grid {
		display: grid;
		gap: 2rem;
		grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
	}

	.card {
		padding: 2rem;
		border-radius: 1.5rem;
		background: rgba(17, 20, 42, 0.78);
		border: 1px solid rgba(100, 120, 255, 0.12);
		display: grid;
		gap: 1rem;
		box-shadow: 0 20px 45px rgba(6, 12, 34, 0.35);
		backdrop-filter: blur(10px);
	}

	.card__title {
		margin: 0;
		font-size: 1.1rem;
		font-weight: 600;
		color: #f6f7ff;
	}

	.card__body {
		margin: 0;
		color: rgba(210, 218, 255, 0.78);
		line-height: 1.65;
		font-size: 0.98rem;
	}

	.footer {
		margin-top: auto;
		padding: 4rem 1.5rem;
		text-align: center;
		color: rgba(170, 180, 220, 0.7);
		font-size: 0.9rem;
		background: rgba(5, 6, 15, 0.9);
	}

	@media (max-width: 720px) {
		.hero__content {
			text-align: center;
			align-items: center;
		}

		.hero__lead {
			max-width: none;
		}

		.hero__actions {
			justify-content: center;
		}
	}
</style>
