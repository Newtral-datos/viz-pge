<script>
  import { onMount } from 'svelte';
  import { years, yearMeta, yearTotals, secciones } from '../lib/data.js';

  const displayYears = years.filter(y => y !== '2023');
  const sep     = n => { const r=Math.round(n); return (r<0?'-':'')+Math.abs(r).toString().replace(/\B(?=(\d{3})+(?!\d))/g,'.'); };
  const fmtMill = n => sep(n/1000);
  const pct   = (a,b) => b ? Math.round(a/b*100) : 0;
  const sign  = n => n >= 0 ? '+' : '';

  let selectedYear = $state('2026');
  let compareYear  = $state(null);

  // Accumulated totals (2024–2026 only)
  let acuMod  = $derived(displayYears.reduce((s,y) => s + yearTotals[y].credito_total - yearTotals[y].credito_inicial, 0));
  let acuSegs = $derived(displayYears.map(y => {
    const m = yearTotals[y].credito_total - yearTotals[y].credito_inicial;
    return { y, label: yearMeta[y].label, mod: m, pct: m / acuMod * 100 };
  }));

  let totales  = $derived(yearTotals[selectedYear]);
  let MOD      = $derived(totales.credito_total - totales.credito_inicial);
  let sections = $derived(
    secciones
      .map(s => ({ ...s, ...s.years[selectedYear] }))
      .filter(s => s.credito_total > 0)
      .sort((a,b) => b.credito_total - a.credito_total)
  );
  // Ranking — propio selector de año (null = acumulado 2024–2026)
  let rankYear = $state(null);

  let rankSections = $derived(
    rankYear
      ? secciones.map(s => ({ ...s, ...s.years[rankYear] })).filter(s => s.credito_total > 0)
      : secciones.map(s => {
          const mod           = displayYears.reduce((sum, y) => sum + (s.years[y]?.mod || 0), 0);
          const credito_inicial = displayYears.reduce((sum, y) => sum + (s.years[y]?.credito_inicial || 0), 0);
          return { ...s, mod, credito_inicial, credito_total: credito_inicial + mod };
        })
  );
  let topMods   = $derived([...rankSections].sort((a,b) => b.mod - a.mod).filter(s => s.mod > 0).slice(0, 10));
  let maxTopMod = $derived(topMods[0]?.mod || 1);

  let compareSecs = $derived(
    compareYear
      ? secciones.map(s => ({
          codigo: s.codigo, nombre: s.nombre,
          modA: Math.round(s.years[selectedYear].mod / 1000),
          modB: Math.round(s.years[compareYear].mod  / 1000),
          delta: Math.round((s.years[selectedYear].mod - s.years[compareYear].mod) / 1000),
        })).sort((a,b) => Math.abs(b.delta) - Math.abs(a.delta)).slice(0,10)
      : []
  );

  // Skyline — smaller, responsive
  const BW = 14, GAP = 3, SVG_H = 170, AX = 20, MBH = SVG_H - AX - 8;
  let sqMax = $derived(Math.sqrt(sections[0]?.credito_total || 1));
  let bars = $derived(sections.map((s,i) => {
    const _bh = v => v>0?(Math.sqrt(v)/sqMax)*MBH:0;
    return { ...s, i, x:i*(BW+GAP),
      baseH: _bh(Math.max(0,s.credito_inicial)),
      totalH: _bh(s.credito_total),
      modH:  Math.max(0, _bh(s.credito_total)-_bh(Math.max(0,s.credito_inicial))),
      negH:  Math.max(0, _bh(Math.max(0,s.credito_inicial))-_bh(s.credito_total)),
    };
  }));
  let SVG_W    = $derived(sections.length*(BW+GAP)-GAP);
  let econBar  = $derived(bars.find(s=>s.codigo==='27'));
  let gridLines= $derived([50_000_000,100_000_000].map(v=>{
    const _bh = x => x>0?(Math.sqrt(x)/sqMax)*MBH:0;
    return { y: SVG_H-AX-_bh(v), label:`${(v/1e3)|0} M` };
  }));

  let cityVis  = $state(false);
  let barsIn   = $state(false);
  let scrollPct= $state(0);
  let tooltip  = $state(null);
  let ttX=$state(0), ttY=$state(0);

  $effect(() => {
    selectedYear;
    cityVis=false; barsIn=false;
    setTimeout(()=>cityVis=true, 150);
    setTimeout(()=>barsIn=true, 300);
  });

  $effect(() => {
    rankYear;
    barsIn=false;
    setTimeout(()=>barsIn=true, 80);
  });

  function goToSection(s) {
    sessionStorage.setItem('highlightSec', s.codigo);
    window.location.hash = 'secciones';
  }

  onMount(()=>{
    window.addEventListener('scroll',()=>{
      const d=document.documentElement;
      scrollPct=d.scrollTop/(d.scrollHeight-d.clientHeight)*100;
    },{passive:true});
    const obs=(sel,cb)=>{
      const el=document.querySelector(sel);
      if(!el)return;
      const io=new IntersectionObserver(([e])=>{if(e.isIntersecting){cb();io.disconnect();}},{threshold:.05});
      io.observe(el);
    };
    obs('#sky',()=>cityVis=true);
    obs('#rank',()=>barsIn=true);
  });
</script>

<div class="prog" style="transform:scaleX({scrollPct/100})"></div>

<div class="ds-page">

  <!-- ── OVERVIEW CARD ──────────────────────────────────────── -->
  <div class="overview">

    <!-- header -->
    <div class="ov-header">
      <span class="ov-eyebrow">Modificaciones del crédito</span>
      <span class="ov-range">2024 – 2026</span>
    </div>

    <!-- hero total acumulado -->
    <div class="ov-hero">
      <div class="ov-stat">
        <span class="ov-pre">+</span>
        <span class="ov-fig">{fmtMill(acuMod)}</span>
        <span class="ov-suf"> millones</span>
      </div>
      <p class="ov-desc">de euros acumulados en 3 ejercicios presupuestarios</p>
    </div>

    <!-- stacked bar por año -->
    <div class="ov-segbar">
      {#each acuSegs as seg, i}
        <div class="ov-seg" style="flex:{seg.pct};--i:{i}" title="{seg.label}: +{fmtMill(seg.mod)} millones"></div>
      {/each}
    </div>
    <div class="ov-seglabels">
      {#each acuSegs as seg}
        <span class="ov-seglbl">{seg.label}</span>
      {/each}
    </div>

    <!-- year cards — selectors -->
    <div class="ov-years">
      {#each acuSegs as seg}
        <button class="oy" class:oy-on={selectedYear===seg.y} onclick={()=>selectedYear=seg.y}>
          <span class="oy-yr">{seg.label}</span>
          <span class="oy-mod">+{fmtMill(seg.mod)}<small> mill.</small></span>
          <span class="oy-share">{seg.pct.toFixed(0)}% del total</span>
        </button>
      {/each}
    </div>

  </div>

  <!-- ── COMPARE ────────────────────────────────────────────── -->
  <div class="cmp-ctrl">
    <span class="cmp-ctrl-lbl">Comparar {yearMeta[selectedYear].label} con</span>
    <div class="ctrl-pills">
      {#each displayYears.filter(y=>y!==selectedYear) as y}
        <button class="pill pill-cmp"
          class:pill-on={compareYear===y}
          onclick={()=>compareYear=compareYear===y?null:y}>
          {yearMeta[y].label}
        </button>
      {/each}
      {#if compareYear}
        <button class="pill-x" onclick={()=>compareYear=null}>✕</button>
      {/if}
    </div>
  </div>

  {#if compareYear}
    <div class="cmp-block">
      <div class="cmp-block-head">
        <span class="cmp-block-title">Diferencia por sección</span>
        <span class="cmp-block-sub">{yearMeta[selectedYear].label} vs {yearMeta[compareYear].label} · top 10 variaciones</span>
      </div>
      <div class="cmp-list">
        {#each compareSecs as s}
          {@const col = s.delta > 0 ? '#007A5A' : '#dc2626'}
          <button class="cmp-row" onclick={()=>selectSection(s)}>
            <span class="cmp-name">{s.nombre.replace(/\.$/, '')}</span>
            <span class="cmp-val">{s.modA>=0?'+':''}{sep(s.modA)}</span>
            <span class="cmp-arr">→</span>
            <span class="cmp-val">{s.modB>=0?'+':''}{sep(s.modB)}</span>
            <span class="cmp-delta" style="color:{col}">{s.delta>=0?'▲':'▼'} {sep(Math.abs(s.delta))} mill.</span>
          </button>
        {/each}
      </div>
    </div>
  {/if}

  <!-- ── SKYLINE ─────────────────────────────────────────────── -->
  <div class="sec-wrap" id="sky">
    <div class="sec-head">
      <div class="sec-head-left">
        <h2 class="sec-title">Sección a sección</h2>
        <span class="sec-badge">{yearMeta[selectedYear].label}</span>
      </div>
      <p class="sec-desc">Altura = crédito total (escala √) · <strong style="color:#007A5A">verde = modificación</strong> · pulsa una barra para ver el detalle completo</p>
    </div>

    <div class="sky-outer">
      <svg viewBox="0 0 {SVG_W} {SVG_H}" class="sky-svg">
        {#each gridLines as gl}
          <line x1="0" y1={gl.y} x2={SVG_W} y2={gl.y} stroke="rgba(73,73,73,.08)" stroke-width="1"/>
          <text x="2" y={gl.y-3} font-size="7" fill="rgba(73,73,73,.32)" font-family="Helvetica Neue,sans-serif">{gl.label}</text>
        {/each}
        <line x1="0" y1={SVG_H-AX} x2={SVG_W} y2={SVG_H-AX} stroke="rgba(73,73,73,.15)" stroke-width="1"/>
        {#each bars as s}
          <g transform="translate({s.x},{SVG_H-AX})" class="bg" class:bg-vis={cityVis}
            style="--di:{s.i}" onclick={()=>goToSection(s)}
            onkeydown={e=>e.key==='Enter'&&goToSection(s)}
            onmouseenter={e=>{tooltip=s;ttX=e.clientX;ttY=e.clientY;}}
            onmousemove={e=>{ttX=e.clientX;ttY=e.clientY;}}
            onmouseleave={()=>tooltip=null}
            role="button" tabindex={s.i+1} aria-label={s.nombre}>
            <rect x="-3" y={-(Math.max(s.baseH,s.totalH)+8)} width={BW+6} height={Math.max(s.baseH,s.totalH)+8} fill="transparent"/>
            {#if s.baseH>0.5}<rect x="0" y={-s.baseH} width={BW} height={s.baseH} fill="#494949" rx="2" class="br br-b"/>{/if}
            {#if s.modH>0.5}<rect x="0" y={-s.totalH} width={BW} height={s.modH} fill="#01f3b3" rx="2" class="br br-m"/>{/if}
            {#if s.negH>1}<rect x="0" y={-s.baseH} width={BW} height={s.negH} fill="rgba(220,38,38,.07)" stroke="#dc2626" stroke-width="1.5" stroke-dasharray="3,2" rx="1"/>{/if}
            <text x={BW/2} y={14} text-anchor="middle" font-size="7" font-family="Helvetica Neue,sans-serif"
              font-weight={s.mod>0?'700':'400'} fill={s.mod>0?'#007A5A':'rgba(73,73,73,0.38)'}>{s.codigo}</text>
          </g>
        {/each}
        {#if econBar && cityVis && econBar.modH>5}
          <g class="ann">
            <line x1={econBar.x+BW/2} y1={SVG_H-AX-econBar.totalH-6} x2={econBar.x+BW/2} y2={18} stroke="#007A5A" stroke-width="1" stroke-dasharray="3,3"/>
            <rect x={econBar.x-12} y="4" width={BW+24} height="15" fill="#e8fff8" rx="3"/>
            <text x={econBar.x+BW/2} y="15" text-anchor="middle" font-size="8" font-weight="700" fill="#007A5A" font-family="Helvetica Neue,sans-serif">+{Math.round(econBar.modPct*100)}%</text>
          </g>
        {/if}
      </svg>
    </div>

    <div class="sky-legend">
      <span class="sl"><span class="sl-sq sl-sq-b"></span>Crédito inicial</span>
      <span class="sl"><span class="sl-sq sl-sq-m"></span>Modificación</span>
    </div>

  </div>

  <!-- ── RANKING ─────────────────────────────────────────────── -->
  <div class="sec-wrap" id="rank">
    <div class="sec-head">
      <div class="sec-head-left">
        <h2 class="sec-title">Mayores modificaciones</h2>
        <span class="sec-badge">{rankYear ? yearMeta[rankYear].label : '2024–2026'}</span>
      </div>
      <p class="sec-desc">{rankYear ? 'Las diez secciones con mayor incremento sobre el presupuesto inicial.' : 'Las diez secciones con mayor modificación acumulada en los tres ejercicios.'}</p>
    </div>
    <div class="rk-filter">
      <button class="rk-yr" class:rk-yr-on={rankYear===null} onclick={()=>rankYear=null}>General</button>
      {#each displayYears as y}
        <button class="rk-yr" class:rk-yr-on={rankYear===y} onclick={()=>rankYear=y}>{yearMeta[y].label}</button>
      {/each}
    </div>
    <ol class="rk-list">
      {#each topMods as s, i}
        {@const w = barsIn ? s.mod/maxTopMod*100 : 0}
        <li>
          <button class="rk-row" onclick={()=>goToSection(s)}>
            <span class="rk-n">#{i+1}</span>
            <div class="rk-body">
              <div class="rk-top">
                <span class="rk-name">{s.nombre.replace(/\.$/, '')}</span>
                <span class="rk-pct">+{pct(s.mod,s.credito_inicial)}%</span>
              </div>
              <div class="rk-track"><div class="rk-fill" style="width:{w}%;transition-delay:{barsIn?i*65:0}ms"><span class="rk-label">+{fmtMill(s.mod)} mill.</span></div></div>
              <div class="rk-meta"><span>Inicial: {fmtMill(s.credito_inicial)} mill.</span><span>Total: {fmtMill(s.credito_total)} mill.</span><span>Modificado: +{pct(s.mod,s.credito_inicial)}%</span></div>
            </div>
            <span class="rk-goto">→</span>
          </button>
        </li>
      {/each}
    </ol>
  </div>

  <footer class="ds-footer">
    <span>Ministerio de Hacienda · Modificaciones de Crédito 2024–2026</span>
    <span>Miles de euros · 2026 hasta Abril</span>
  </footer>
</div>

<!-- tooltip -->
{#if tooltip}
  <div class="tt" style="left:{ttX+14}px;top:{ttY-68}px">
    <strong>{tooltip.nombre?.replace(/\.$/, '').slice(0,44)}</strong>
    <span>{fmtMill(tooltip.credito_total)} mill. total</span>
    {#if tooltip.mod>0}<span class="tt-g">+{fmtMill(tooltip.mod)} mill. · +{pct(tooltip.mod,tooltip.credito_inicial)}%</span>
    {:else if tooltip.mod<0}<span class="tt-r">{fmtMill(tooltip.mod)} mill.</span>
    {:else}<span class="tt-d">Sin modificación</span>{/if}
  </div>
{/if}

<style>
  .prog { position:fixed;top:0;left:0;width:100%;height:2px;background:#01f3b3;transform-origin:left;z-index:9999;transition:transform .08s linear; }
  .ds-page { max-width:1100px;margin:0 auto;padding:.75rem 1.5rem 2rem;width:100%; }

  /* ── overview card ── */
  .overview {
    border: 1px solid rgba(73,73,73,.1);
    border-radius: 10px;
    padding: .75rem 1rem;
    margin-bottom: .75rem;
  }

  .ov-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: .5rem;
  }
  .ov-eyebrow {
    font-size: .6rem;
    text-transform: uppercase;
    letter-spacing: .13em;
    color: rgba(73,73,73,.45);
    font-weight: 600;
  }
  .ov-range {
    font-size: .68rem;
    font-weight: 700;
    color: rgba(73,73,73,.55);
    background: rgba(73,73,73,.06);
    padding: .2rem .6rem;
    border-radius: 4px;
  }

  /* hero stat */
  .ov-hero { margin-bottom: .5rem; }
  .ov-stat {
    display: flex;
    align-items: flex-end;
    gap: .12rem;
    line-height: 1;
    margin-bottom: .2rem;
  }
  .ov-pre  { font-size:clamp(.8rem,1.6vw,1rem);font-weight:700;color:#01f3b3;padding-bottom:.07em; }
  .ov-fig  { font-size:clamp(1.35rem,3vw,2rem);font-weight:700;color:#494949;letter-spacing:-.04em;font-variant-numeric:tabular-nums; }
  .ov-suf  { font-size:clamp(.65rem,1.2vw,.88rem);font-weight:600;color:rgba(73,73,73,.35);padding-bottom:.18em;margin-left:.05rem; }
  .ov-desc { font-size:.7rem;color:rgba(73,73,73,.45);margin:0; }

  /* stacked segment bar */
  .ov-segbar {
    display: flex;
    height: 6px;
    border-radius: 3px;
    overflow: hidden;
    gap: 2px;
    margin-bottom: .35rem;
  }
  .ov-seg {
    border-radius: 2px;
    background: rgba(1,243,179, calc(0.3 + var(--i) * 0.23));
    transition: opacity .2s;
  }
  .ov-seglabels {
    display: flex;
    margin-bottom: .5rem;
  }
  .ov-seglbl {
    flex: 1;
    font-size: .58rem;
    color: rgba(73,73,73,.38);
    letter-spacing: .04em;
  }
  .ov-seglbl:last-child { text-align: right; }
  .ov-seglbl:nth-child(2),
  .ov-seglbl:nth-child(3) { text-align: center; }

  /* year selector cards */
  .ov-years {
    display: grid;
    grid-template-columns: repeat(4,1fr);
    gap: .35rem;
    margin-bottom: 0;
  }
  .oy {
    all: unset;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    gap: .1rem;
    padding: .4rem .55rem;
    border: 1px solid rgba(73,73,73,.1);
    border-radius: 8px;
    transition: all .15s;
    background: transparent;
  }
  .oy:hover { border-color:rgba(1,243,179,.4);background:rgba(1,243,179,.04); }
  .oy-on    { border-color:#01f3b3;background:rgba(1,243,179,.07); }
  .oy-yr    { font-size:.62rem;font-weight:700;color:rgba(73,73,73,.45);text-transform:uppercase;letter-spacing:.08em; }
  .oy-on .oy-yr { color:#007A5A; }
  .oy-mod   { font-size:clamp(.78rem,1.5vw,.95rem);font-weight:700;color:#494949;font-variant-numeric:tabular-nums; }
  .oy-on .oy-mod { color:#007A5A; }
  .oy-mod small { font-size:.55em;font-weight:500;color:rgba(73,73,73,.38); }
  .oy-on .oy-mod small { color:rgba(0,122,90,.5); }
  .oy-share { font-size:.6rem;color:rgba(73,73,73,.35); }
  .oy-on .oy-share { color:rgba(0,122,90,.6); }

  /* ── compare control ── */
  .cmp-ctrl {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: .5rem;
    margin-bottom: .5rem;
    padding: .35rem 0;
    border-top: 1px solid rgba(73,73,73,.07);
  }
  .cmp-ctrl-lbl {
    font-size: .62rem;
    text-transform: uppercase;
    letter-spacing: .1em;
    color: rgba(73,73,73,.38);
    white-space: nowrap;
  }
  .ctrl-pills { display:flex;gap:.25rem;flex-wrap:wrap; }
  .pill {
    all: unset;
    cursor: pointer;
    padding: .28rem .7rem;
    border-radius: 20px;
    font-size: .72rem;
    font-weight: 500;
    border: 1px solid rgba(73,73,73,.12);
    color: rgba(73,73,73,.45);
    transition: all .15s;
    white-space: nowrap;
  }
  .pill:hover { border-color:rgba(1,243,179,.45);color:#007A5A; }
  .pill-on  { background:#007A5A;color:#fff;border-color:#007A5A; }
  .pill-x {
    all: unset;
    cursor: pointer;
    font-size: .68rem;
    color: rgba(73,73,73,.35);
    padding: .22rem .5rem;
    border-radius: 4px;
    border: 1px solid rgba(73,73,73,.1);
    transition: all .15s;
  }
  .pill-x:hover { color:#dc2626;border-color:#dc2626; }

  /* ── compare block ── */
  .cmp-block {
    border: 1px solid rgba(73,73,73,.09);
    border-radius: 10px;
    overflow: hidden;
    margin-bottom: 1.25rem;
  }
  .cmp-block-head {
    display: flex;
    align-items: baseline;
    gap: .75rem;
    padding: .75rem 1.1rem;
    background: rgba(73,73,73,.02);
    border-bottom: 1px solid rgba(73,73,73,.07);
    flex-wrap: wrap;
  }
  .cmp-block-title { font-size:.8rem;font-weight:700;color:#494949; }
  .cmp-block-sub   { font-size:.66rem;color:rgba(73,73,73,.4); }
  .cmp-list { display:flex;flex-direction:column; }
  .cmp-row {
    all: unset;
    display: flex;
    align-items: center;
    gap: .65rem;
    padding: .55rem 1.1rem;
    cursor: pointer;
    transition: background .12s;
    font-size: .78rem;
    border-bottom: 1px solid rgba(73,73,73,.04);
  }
  .cmp-row:last-child { border-bottom:none; }
  .cmp-row:hover { background:rgba(73,73,73,.025); }
  .cmp-name  { flex:1;font-weight:500;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;color:#494949; }
  .cmp-val   { font-variant-numeric:tabular-nums;font-size:.74rem;color:rgba(73,73,73,.45);white-space:nowrap; }
  .cmp-arr   { color:rgba(73,73,73,.22);font-size:.7rem; }
  .cmp-delta { font-weight:700;white-space:nowrap;font-size:.78rem; }

  /* ── section wraps ── */
  .sec-wrap { padding:1.1rem 0;border-top:1px solid rgba(73,73,73,.07); }
  .sec-head { margin-bottom:.6rem; }
  .sec-head-left { display:flex;align-items:baseline;gap:.55rem;margin-bottom:.25rem; }
  .sec-title { font-size:clamp(.88rem,1.6vw,1.1rem);font-weight:700;letter-spacing:-.03em;color:#494949; }
  .sec-badge {
    font-size: .65rem;
    font-weight: 700;
    color: #007A5A;
    background: rgba(1,243,179,.12);
    padding: .15rem .45rem;
    border-radius: 3px;
  }
  .sec-desc { font-size:.68rem;color:rgba(73,73,73,.45);max-width:560px;line-height:1.4;margin:0; }

  /* ── skyline ── */
  .sky-outer { width:100%; }
  .sky-svg   { display:block;width:100%;height:auto; }
  .bg { cursor:pointer; }
  .bg:hover .br-b { fill:#666; }
  .bg:hover .br-m { fill:#5ff7d6; }
  .br { transform-box:fill-box;transform-origin:center bottom;transform:scaleY(0); }
  .bg-vis .br-b { animation:barUp .55s cubic-bezier(.16,1,.3,1) both;animation-delay:calc(var(--di)*12ms); }
  .bg-vis .br-m { animation:barUp .45s cubic-bezier(.16,1,.3,1) both;animation-delay:calc(var(--di)*12ms+180ms); }
  @keyframes barUp { from{transform:scaleY(0)}to{transform:scaleY(1)} }
  .ann { opacity:0;animation:fadeIn .4s 1.1s ease forwards; }
  @keyframes fadeIn { to{opacity:1} }
  .sky-legend { display:flex;align-items:center;gap:1.25rem;flex-wrap:wrap;margin-top:.75rem; }
  .sl { display:flex;align-items:center;gap:.35rem;font-size:.68rem;color:rgba(73,73,73,.45); }
  .sl-sq { width:9px;height:9px;border-radius:2px; }
  .sl-sq-b { background:#494949; }
  .sl-sq-m { background:#01f3b3; }

  /* ── ranking filter ── */
  .rk-filter { display:flex;gap:.3rem;flex-wrap:wrap;margin-bottom:.7rem; }
  .rk-yr {
    all: unset;
    cursor: pointer;
    padding: .28rem .7rem;
    border-radius: 20px;
    font-size: .72rem;
    font-weight: 500;
    border: 1px solid rgba(73,73,73,.12);
    color: rgba(73,73,73,.45);
    transition: all .15s;
    white-space: nowrap;
  }
  .rk-yr:hover { border-color:rgba(1,243,179,.45);color:#007A5A; }
  .rk-yr-on  { background:#0B1D3A;color:#01f3b3;border-color:#0B1D3A; }

  /* ── ranking ── */
  .rk-list { list-style:none;display:flex;flex-direction:column;gap:1px; }
  .rk-row {
    all: unset;
    display: flex;
    align-items: flex-start;
    gap: .65rem;
    padding: .38rem .45rem;
    border-radius: 8px;
    cursor: pointer;
    width: 100%;
    box-sizing: border-box;
    border: 1px solid transparent;
    transition: background .12s, border-color .12s;
  }
  .rk-row:hover { background:rgba(1,243,179,.05);border-color:rgba(1,243,179,.18); }
  .rk-n   { font-size:.6rem;font-weight:700;color:rgba(73,73,73,.3);width:20px;flex-shrink:0;padding-top:.12rem; }
  .rk-body{ flex:1;min-width:0; }
  .rk-top { display:flex;justify-content:space-between;align-items:baseline;margin-bottom:.18rem; }
  .rk-name{ font-size:.78rem;font-weight:600;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;flex:1;padding-right:.75rem;color:#494949; }
  .rk-pct { font-size:.68rem;font-weight:700;color:#007A5A;white-space:nowrap;background:rgba(1,243,179,.12);padding:.1rem .38rem;border-radius:3px; }
  .rk-track{ height:18px;background:rgba(73,73,73,.05);border-radius:5px;overflow:hidden;margin-bottom:.28rem; }
  .rk-fill { height:100%;background:linear-gradient(90deg,#01f3b3,#00c49a);border-radius:5px;transition:width .95s cubic-bezier(.16,1,.3,1);display:flex;align-items:center;overflow:hidden; }
  .rk-label{ font-size:.65rem;font-weight:700;color:#494949;padding:0 .6rem;white-space:nowrap; }
  .rk-meta { display:flex;gap:1rem;font-size:.62rem;color:rgba(73,73,73,.35);flex-wrap:wrap; }

  /* ── footer ── */
  .ds-footer { border-top:1px solid rgba(73,73,73,.07);padding-top:1rem;margin-top:1.5rem;display:flex;justify-content:space-between;flex-wrap:wrap;gap:.5rem;font-size:.63rem;color:rgba(73,73,73,.28);letter-spacing:.04em; }

  /* ── tooltip ── */
  .tt { position:fixed;background:#494949;color:#fff;padding:.5rem .8rem;border-radius:8px;font-size:.73rem;pointer-events:none;z-index:500;display:flex;flex-direction:column;gap:.12rem;box-shadow:0 6px 24px rgba(73,73,73,.2);max-width:240px; }
  .tt strong { font-weight:600;font-size:.78rem;line-height:1.3; }
  .tt span   { color:rgba(255,255,255,.5); }
  .tt .tt-g  { color:#01f3b3; }
  .tt .tt-r  { color:#fb7185; }
  .tt .tt-d  { color:rgba(255,255,255,.25); }
  @media (hover:none) { .tt { display:none; } }

  /* ranking goto arrow */
  .rk-goto { font-size:.9rem;color:rgba(73,73,73,.2);transition:color .15s,transform .15s;flex-shrink:0; }
  .rk-row:hover .rk-goto { color:#007A5A;transform:translateX(3px); }

  @media (max-width:640px) {
    .ds-page { padding:1.25rem 1rem 3rem; }
    .overview { padding:1.25rem 1rem; }
    .ov-years { grid-template-columns:repeat(2,1fr); }
    .sec-wrap { padding:2rem 0; }
  }
</style>
