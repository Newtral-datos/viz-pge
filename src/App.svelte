<script>
  import { onMount } from 'svelte';
  import LineChart  from './views/LineChart.svelte';
  import Desglose   from './views/Desglose.svelte';
  import Secciones  from './views/Secciones.svelte';
  import Mapa       from './views/Mapa.svelte';
  import logoNewtral from './assets/logo-newtral.png';

  let route = $state('lineas');

  onMount(() => {
    const read = () => window.location.hash.replace(/^#\/?/, '') || 'lineas';
    route = read();
    window.addEventListener('hashchange', () => { route = read(); });
  });
</script>

<header class="gnav">
  <div class="gnav-inner">
    <span class="gnav-brand">Modificaciones de los Presupuestos Generales del Estado</span>
    <nav class="gnav-links">
      <a href="#lineas"    class:act={route==='lineas'}>Tendencia</a>
      <a href="#desglose"  class:act={route==='desglose'}>Desglose</a>
      <a href="#secciones" class:act={route==='secciones'}>Secciones</a>
      <a href="#mapa"      class:act={route==='mapa'}>Mapa</a>
      <img src={logoNewtral} alt="Newtral" class="gnav-logo" />
    </nav>
  </div>
</header>

{#if route === 'lineas'}
  <LineChart />
{:else if route === 'desglose'}
  <Desglose />
{:else if route === 'secciones'}
  <Secciones />
{:else}
  <Mapa />
{/if}

<style>
  :global(*, *::before, *::after) { box-sizing: border-box; margin: 0; padding: 0; }
  :global(html)  { scroll-behavior: smooth; }
  :global(body)  {
    font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
    background: #fff; color: #0B1D3A; -webkit-font-smoothing: antialiased;
  }

  .gnav {
    position: sticky; top: 0; z-index: 200;
    background: rgba(255,255,255,.96);
    backdrop-filter: blur(12px); -webkit-backdrop-filter: blur(12px);
    border-bottom: 1px solid rgba(11,29,58,.08);
  }
  .gnav-inner {
    max-width: 1200px; margin: 0 auto;
    display: flex; align-items: center; gap: 1.25rem;
    padding: .4rem 1.5rem;
  }
  .gnav-brand {
    font-size: .72rem; font-weight: 700; letter-spacing: .01em;
    color: #494949; line-height: 1.3;
  }
  .gnav-links {
    display: flex; gap: .25rem; margin-left: auto; flex-wrap: wrap;
  }
  .gnav-links a {
    text-decoration: none; font-size: .8rem; font-weight: 500;
    color: rgba(11,29,58,.45); padding: .4rem .9rem; border-radius: 20px;
    border: 1px solid transparent; transition: all .15s;
  }
  .gnav-links a:hover { color: #007A5A; border-color: rgba(1,243,179,.35); }
  .gnav-links a.act   {
    background: #0B1D3A; color: #01f3b3; border-color: #0B1D3A;
  }
  .gnav-logo { height: 22px; width: auto; display: block; margin-left: 1.5rem; opacity: .85; }

  @media (max-width: 480px) {
    .gnav-inner { padding: .6rem 1rem; gap: 1rem; }
    .gnav-brand { display: none; }
  }
</style>
