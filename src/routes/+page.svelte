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
				vec3 base = vec3(0.80, 0.88, 0.98);
				vec3 wave = vec3(0.30, 0.40, 0.65);
				vec3 mod = vec3(0.18, 0.24, 0.30);
				vec3 shift = vec3(0.00, 0.35, 0.72);
				return base + wave * cos(PI * (mod * t + shift));
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
				vec3 color = palette(intensity + dist * 0.28);

				color = mix(color, vec3(0.97, 0.95, 0.99), smoothstep(0.0, 0.85, dist));
				color += 0.06 * vec3(cos((centeredUv.x + time) * 2.6), sin((centeredUv.y - time) * 3.1), sin((centeredUv.x + centeredUv.y + time) * 2.1));

				gl_FragColor = vec4(color, 0.9);
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
		href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Space+Grotesk:wght@400;500;600;700&family=Noto+Sans+JP:wght@300;400;500;600;700&display=swap"
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
			<h1>Promethix</h1>
			<p class="hero__lead">
				私たちは、最先端の技術を駆使し、お客様のビジネスを加速させるWEBアプリケーションを開発します。企画から設計、実装、運用までワンストップでサポートします。
			</p>
			<div class="hero__actions">
				<a class="btn btn--primary" href="#about">Discover Promethix</a>
				<a class="btn btn--outline" href={resolve('/contact')}>Contact</a>
			</div>
		</div>
	</section>

	<section id="about" class="about">
		<div class="about__intro">
			<h2>高い技術力で、確かなビジネス価値を創造します。</h2>
			<p>
				私たちは、WEBアプリケーション開発に特化した専門家チームです。React,
				Vue,
				Svelteなどのモダンなフレームワークから、サーバーサイド、インフラ構築まで、幅広い技術領域をカバーしています。
            </p>
		</div>

		<div class="about__grid">
			<article class="card">
				<h3 class="card__title">WEBシステム開発</h3>
				<p class="card__body">
					業務の効率化から新規事業の立ち上げまで、お客様固有の課題に合わせたオーダーメイドのWEBアプリケーションを構築します。
				</p>
			</article>
			<article class="card">
				<h3 class="card__title">高速なフロントエンド開発</h3>
				<p class="card__body">
					ユーザーに最高の体験を提供するため、パフォーマンスを追求したSPA（シングルページアプリケーション）を構築。UI/UXデザインも考慮し、使いやすいインターフェースを実現します。
				</p>
			</article>
			<article class="card">
				<h3 class="card__title">堅牢なバックエンド・API開発</h3>
				<p class="card__body">
					スケーラビリティとセキュリティを両立させた、安定稼働するサーバーサイドシステムを設計・開発。REST/GraphQL
					APIの構築も得意としています。
				</p>
			</article>
			<article class="card">
				<h3 class="card__title">クラウドインフラ構築・運用</h3>
				<p class="card__body">
					AWS, GCP,
					Azureを活用し、サービスの成長に合わせて柔軟にスケールするインフラを設計。CI/CDの導入や保守・運用サポートも行います。
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
		font-family: 'Noto Sans JP', 'Inter', system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI',
			sans-serif;
		color: #1e1f36;
		background: radial-gradient(circle at 18% 10%, #f5f7ff 0%, #e6f3ff 45%, #dbe8ff 100%);
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
		color: rgba(108, 116, 172, 0.7);
		font-weight: 600;
		font-family: 'Space Grotesk', 'Noto Sans JP', 'Inter', sans-serif;
	}

	h1 {
		margin: 0;
		font-size: clamp(2.9rem, 6vw, 5rem);
		line-height: 1.05;
		font-family: 'Space Grotesk', 'Noto Sans JP', 'Inter', sans-serif;
		color: #262453;
	}

	h2 {
		font-size: clamp(2rem, 3.5vw, 3.1rem);
		line-height: 1.1;
		margin: 0;
		font-family: 'Space Grotesk', 'Noto Sans JP', 'Inter', sans-serif;
		color: #2e2b63;
	}

	.hero__lead {
		margin: 0;
		font-size: clamp(1.05rem, 2.1vw, 1.35rem);
		max-width: 40rem;
		color: rgba(70, 76, 122, 0.95);
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
		background: linear-gradient(120deg, #9ed9ff 0%, #b4a4ff 100%);
		color: #0f1430;
		box-shadow: 0 12px 35px rgba(158, 217, 255, 0.35);
	}

	.btn--primary:hover {
		box-shadow: 0 18px 45px rgba(180, 164, 255, 0.38);
	}

	.btn--outline {
		border: 1px solid rgba(150, 160, 235, 0.6);
		color: rgba(55, 60, 105, 0.9);
		background: rgba(255, 255, 255, 0.65);
		backdrop-filter: blur(10px);
	}

	.about {
		position: relative;
		padding: clamp(6rem, 12vw, 9rem) clamp(1.5rem, 8vw, 8rem);
		background: linear-gradient(180deg, rgba(248, 247, 255, 0.9) 0%, rgba(232, 240, 255, 0.9) 100%);
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
		color: rgba(74, 83, 118, 0.9);
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
		background: rgba(255, 255, 255, 0.78);
		border: 1px solid rgba(159, 170, 235, 0.35);
		display: grid;
		gap: 1rem;
		box-shadow: 0 20px 45px rgba(173, 187, 255, 0.28);
		backdrop-filter: blur(14px);
	}

	.card__title {
		margin: 0;
		font-size: 1.1rem;
		font-weight: 600;
		color: #2c2d64;
		font-family: 'Space Grotesk', 'Noto Sans JP', 'Inter', sans-serif;
	}

	.card__body {
		margin: 0;
		color: rgba(76, 85, 120, 0.85);
		line-height: 1.65;
		font-size: 0.98rem;
	}

	.footer {
		margin-top: auto;
		padding: 4rem 1.5rem;
		text-align: center;
		color: rgba(80, 86, 130, 0.9);
		font-size: 0.9rem;
		background: rgba(242, 243, 255, 0.95);
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
