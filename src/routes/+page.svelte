
<script lang="ts">
	import { onMount } from 'svelte';
	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D | null = null;
	let animationId: number;

	// 簡易流体シミュレーション（パーリンノイズ風の流れ）
	function drawFluid(time: number) {
		if (!ctx) return;
		const w = canvas.width;
		const h = canvas.height;
		ctx.clearRect(0, 0, w, h);
		for (let i = 0; i < 100; i++) {
			const x = w / 2 + Math.sin(time / 800 + i) * (w / 3) * Math.sin(i);
			const y = h / 2 + Math.cos(time / 1000 + i) * (h / 3) * Math.cos(i);
			const r = 60 + 40 * Math.sin(time / 500 + i);
			ctx.beginPath();
			ctx.arc(x, y, r * Math.abs(Math.sin(i + time / 2000)), 0, 2 * Math.PI);
			ctx.fillStyle = `hsla(${(i * 12 + time / 10) % 360}, 80%, 60%, 0.12)`;
			ctx.fill();
		}
		animationId = requestAnimationFrame(drawFluid);
	}

	onMount(() => {
		ctx = canvas.getContext('2d');
		function resize() {
			canvas.width = window.innerWidth;
			canvas.height = window.innerHeight;
		}
		resize();
		window.addEventListener('resize', resize);
		animationId = requestAnimationFrame(drawFluid);
		return () => {
			window.removeEventListener('resize', resize);
			cancelAnimationFrame(animationId);
		};
	});
</script>

<section class="relative flex items-center justify-center h-screen min-h-screen w-full overflow-hidden bg-black">
	<canvas
		bind:this={canvas}
		class="absolute inset-0 w-full h-full pointer-events-none z-0"
		style="display:block;"
	></canvas>
	<div class="relative z-10 flex flex-col items-center justify-center w-full h-full">
		<h1 class="text-6xl md:text-7xl font-extrabold text-white drop-shadow-lg tracking-widest select-none" style="letter-spacing:0.1em;">
			Promethix
		</h1>
		<p class="mt-6 text-lg text-white/80 font-medium">プロメティクス</p>
	</div>
</section>
