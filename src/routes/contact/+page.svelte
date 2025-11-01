<script lang="ts">
  import { resolve } from '$app/paths';
</script>

<svelte:head>
  <title>Contact Promethix</title>
  <meta
    name="description"
    content="Connect with Promethix to explore fluid, immersive experiences for your next product launch."
  />
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
  <link
    href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Space+Grotesk:wght@400;500;600;700&family=Noto+Sans+JP:wght@300;400;500;600;700&display=swap"
    rel="stylesheet"
  />
</svelte:head>

<section class="contact">
  <div class="contact__background">
    <span class="orb orb--one"></span>
    <span class="orb orb--two"></span>
    <span class="orb orb--three"></span>
    <span class="grain"></span>
  </div>

  <div class="contact__card">
    <span class="contact__eyebrow">Contact</span>
    <h1>次の波を、一緒に創り出しましょう。</h1>
    <p>
      Promethixと対話を始め、流体シミュレーションやリアルタイムグラフィックス、インタラクティブなストーリーテリングがローンチ体験をどのように高められるかをご体感ください。
    </p>
    <a class="contact__email" href="mailto:hello@promethix.jp">hello@promethix.jp</a>
    <a class="contact__home" href={resolve('/')}>← Return to home</a>
  </div>
</section>

<style>
  :global(body) {
    margin: 0;
    font-family: 'Noto Sans JP', 'Inter', system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI',
      sans-serif;
    background: #04040b;
    color: rgba(240, 245, 255, 0.92);
  }

  .contact {
    position: relative;
    min-height: 100dvh;
    display: grid;
    place-items: center;
    padding: clamp(4rem, 12vh, 7rem) clamp(1.5rem, 6vw, 6rem);
    overflow: hidden;
  }

  .contact__background {
    position: absolute;
    inset: 0;
    overflow: hidden;
    z-index: 0;
    background: radial-gradient(circle at 20% 20%, rgba(85, 120, 255, 0.25), transparent 55%),
      radial-gradient(circle at 80% 10%, rgba(255, 133, 238, 0.35), transparent 60%),
      radial-gradient(circle at 50% 80%, rgba(70, 255, 216, 0.22), transparent 60%);
  }

  .grain {
    position: absolute;
    inset: -200%;
    background-image: url('data:image/svg+xml,%3Csvg xmlns="http://www.w3.org/2000/svg" width="400" height="400" viewBox="0 0 400 400"%3E%3Cfilter id="n" x="0" y="0" width="100%25" height="100%25"%3E%3CfeTurbulence type="fractalNoise" baseFrequency="2.8" numOctaves="3" stitchTiles="stitch"/%3E%3C/filter%3E%3Crect width="400" height="400" filter="url(%23n)" opacity="0.12"/%3E%3C/svg%3E');
    opacity: 0.35;
    mix-blend-mode: screen;
    animation: drift 18s linear infinite;
  }

  .orb {
    position: absolute;
    border-radius: 50%;
    filter: blur(80px);
    opacity: 0.55;
    transform: translate3d(0, 0, 0);
    animation: float 16s ease-in-out infinite;
  }

  .orb--one {
    width: 36rem;
    height: 36rem;
    background: radial-gradient(circle, rgba(111, 132, 255, 0.8), rgba(31, 43, 120, 0));
    top: -10rem;
    left: -10rem;
  }

  .orb--two {
    width: 28rem;
    height: 28rem;
    background: radial-gradient(circle, rgba(255, 133, 238, 0.8), rgba(100, 39, 120, 0));
    bottom: -12rem;
    right: -12rem;
    animation-duration: 20s;
  }

  .orb--three {
    width: 22rem;
    height: 22rem;
    background: radial-gradient(circle, rgba(70, 255, 216, 0.7), rgba(12, 88, 88, 0));
    top: 35%;
    right: 10%;
    animation-duration: 22s;
  }

  .contact__card {
    position: relative;
    z-index: 1;
    max-width: 38rem;
    padding: clamp(2.5rem, 8vw, 3.5rem);
    border-radius: 1.75rem;
    background: rgba(11, 13, 32, 0.82);
    border: 1px solid rgba(123, 138, 255, 0.25);
    box-shadow: 0 20px 60px rgba(11, 14, 40, 0.45);
    backdrop-filter: blur(18px);
    display: grid;
    gap: 1.5rem;
    text-align: center;
  }

  .contact__eyebrow {
    letter-spacing: 0.42em;
    text-transform: uppercase;
    font-size: 0.75rem;
    color: rgba(191, 199, 255, 0.65);
    font-weight: 600;
    font-family: 'Space Grotesk', 'Noto Sans JP', 'Inter', sans-serif;
  }

  h1 {
    margin: 0;
    font-size: clamp(2rem, 4vw, 2.8rem);
    line-height: 1.15;
    font-family: 'Space Grotesk', 'Noto Sans JP', 'Inter', sans-serif;
    color: #f8f9ff;
  }

  p {
    margin: 0;
    line-height: 1.7;
    color: rgba(214, 220, 255, 0.8);
    font-size: 1.05rem;
  }

  .contact__email {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0.85rem 2rem;
    border-radius: 999px;
    font-weight: 600;
    text-decoration: none;
    color: #060613;
    background: linear-gradient(120deg, #66f7ff 0%, #7a5cff 100%);
    box-shadow: 0 18px 45px rgba(123, 92, 255, 0.32);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
  }

  .contact__email:hover {
    transform: translateY(-3px);
    box-shadow: 0 24px 60px rgba(123, 92, 255, 0.42);
  }

  .contact__email:focus-visible {
    outline: 2px solid rgba(102, 247, 255, 0.65);
    outline-offset: 3px;
  }

  .contact__home {
    text-decoration: none;
    color: rgba(200, 208, 255, 0.7);
    font-weight: 500;
    transition: color 0.3s ease;
  }

  .contact__home:hover {
    color: rgba(195, 233, 255, 0.9);
  }

  @keyframes float {
    0%,
    100% {
      transform: translate3d(0, 0, 0) scale(1);
    }
    50% {
      transform: translate3d(2rem, -1.5rem, 0) scale(1.08);
    }
  }

  @keyframes drift {
    0% {
      transform: translate3d(0, 0, 0);
    }
    100% {
      transform: translate3d(-20%, -20%, 0);
    }
  }

  @media (max-width: 600px) {
    .orb {
      filter: blur(60px);
    }

    .orb--one,
    .orb--two,
    .orb--three {
      width: 24rem;
      height: 24rem;
    }

    .contact__card {
      padding: 2.25rem;
      gap: 1.25rem;
    }
  }
</style>
