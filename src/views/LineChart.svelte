<script>
  import { onMount } from 'svelte';
  import { years, yearMeta, yearTotals, secciones } from '../lib/data.js';

  // ── formato: "32.188 millones" ──────────────────────────────
  // los valores están en miles de euros (k€); /1000 da millones
  const sep         = n => { const r=Math.round(n); return (r<0?'-':'')+Math.abs(r).toString().replace(/\B(?=(\d{3})+(?!\d))/g,'.'); };
  const fmtMil      = n => sep(n/1000) + ' millones';
  const fmtMilShort = n => sep(n/1000);
  const pct = (a, b) => b ? Math.round(a / b * 100) : 0;

  // ── línea total ─────────────────────────────────────────────
  const totalLine = years.map(y => ({
    year:  y,
    label: yearMeta[y].label,
    mod:   yearTotals[y].credito_total - yearTotals[y].credito_inicial,  // en k€
    modPct: pct(yearTotals[y].credito_total - yearTotals[y].credito_inicial, yearTotals[y].credito_inicial),
  }));

  // ── datos por sección ───────────────────────────────────────
  const allSecOptions = secciones
    .map(s => ({
      codigo: s.codigo,
      nombre: s.nombre,
      points: years.map(y => ({ year: y, mod: s.years[y].mod })),  // k€
      maxMod: Math.max(...years.map(y => s.years[y].mod)),
    }))
    .sort((a, b) => b.maxMod - a.maxMod);

  // ── secciones añadidas por el usuario ───────────────────────
  let extraCodigos = $state([]);
  let selectValue  = $state('');

  const COLORS = ['#3B82F6','#F59E0B','#8B5CF6','#EF4444','#10B981','#F97316','#EC4899','#6366F1'];

  function addSection(codigo) {
    if (codigo && !extraCodigos.includes(codigo)) {
      extraCodigos = [...extraCodigos, codigo];
    }
    selectValue = '';
  }

  function removeSection(codigo) {
    extraCodigos = extraCodigos.filter(c => c !== codigo);
  }

  let extraLines = $derived(
    extraCodigos.map((codigo, i) => ({
      ...allSecOptions.find(s => s.codigo === codigo),
      color: COLORS[i % COLORS.length],
    }))
  );

  // secciones disponibles en el desplegable (excluye ya añadidas)
  let availableOptions = $derived(
    allSecOptions.filter(s => !extraCodigos.includes(s.codigo))
  );

  // ── SVG layout ──────────────────────────────────────────────
  const W = 760, H = 260, PL = 60, PB = 38, PT = 36, PR = 24;
  const CW = W - PL - PR;
  const CH = H - PT - PB;
  const N  = years.length;

  // Y scale: basado en el max de todas las líneas visibles
  let maxVal = $derived(() => {
    const vals = [
      ...totalLine.map(t => t.mod),
      ...extraLines.flatMap(l => l.points.map(p => p.mod)),
    ];
    return Math.max(...vals) * 1.12;
  });

  function xOf(i)  { return PL + (i / (N - 1)) * CW; }
  function yOf(v)  { return PT + CH - (v / maxVal()) * CH; }

  // path suave entre puntos
  function makePath(pts) {
    const coords = pts.map((p, i) => ({ x: xOf(i), y: yOf(p.mod) }));
    let d = `M ${coords[0].x},${coords[0].y}`;
    for (let i = 1; i < coords.length; i++) {
      const p = coords[i-1], c = coords[i];
      const cx = (p.x + c.x) / 2;
      d += ` C ${cx},${p.y} ${cx},${c.y} ${c.x},${c.y}`;
    }
    return d;
  }

  // gridlines
  const GRID_N = 4;
  let gridLines = $derived(
    Array.from({ length: GRID_N + 1 }, (_, i) => {
      const v = (maxVal() / GRID_N) * i;
      return { v, y: yOf(v), label: fmtMilShort(v) };
    })
  );

  // ── hover ───────────────────────────────────────────────────
  let hoverIdx = $state(null);
  let ttX = $state(0), ttY = $state(0);
  let animated = $state(false);

  function onMove(e) {
    const rect = e.currentTarget.getBoundingClientRect();
    const mx   = (e.clientX - rect.left) * (W / rect.width);
    const best = years.reduce((b, _, i) => {
      const d = Math.abs(xOf(i) - mx);
      return d < b.d ? { i, d } : b;
    }, { i: 0, d: Infinity });
    hoverIdx = best.d < 45 ? best.i : null;
    if (hoverIdx !== null) { ttX = e.clientX; ttY = e.clientY; }
  }

  onMount(() => setTimeout(() => animated = true, 80));
</script>

<div class="lc-page">
  <div class="lc-head">
    <h1>Evolución de las modificaciones de crédito</h1>
    <p>Incremento total sobre el presupuesto inicial aprobado en cada ejercicio.
      <strong>2026 solo incluye datos hasta Abril.</strong></p>
  </div>

  <!-- Totals summary strip -->
  <div class="lc-strip">
    {#each totalLine as t, i}
      <div class="lc-strip-item" class:lc-strip-item--hover={hoverIdx === i}>
        <span class="lcs-year">{t.label}{t.year === '2026' ? '*' : ''}</span>
        <span class="lcs-val">+{fmtMil(t.mod)}</span>
        <span class="lcs-exec">+{t.modPct}% modificado</span>
      </div>
    {/each}
  </div>

  <!-- Chart -->
  <div class="lc-chart-wrap">
    <svg
      viewBox="0 0 {W} {H}"
      width="100%"
      class="lc-svg"
      onmousemove={onMove}
      onmouseleave={() => hoverIdx = null}
      role="img"
      aria-label="Gráfico de líneas de modificaciones presupuestarias 2023-2026"
    >
      <!-- grid -->
      {#each gridLines as gl}
        {#if gl.v > 0}
          <line x1={PL} y1={gl.y} x2={W - PR} y2={gl.y}
            stroke="rgba(11,29,58,.06)" stroke-width="1"/>
          <text x={PL - 8} y={gl.y + 4} text-anchor="end"
            font-size="10" fill="rgba(11,29,58,.3)"
            font-family="Helvetica Neue,sans-serif">{gl.label}</text>
        {/if}
      {/each}

      <!-- baseline -->
      <line x1={PL} y1={H - PB} x2={W - PR} y2={H - PB}
        stroke="rgba(11,29,58,.1)" stroke-width="1"/>

      <!-- X labels -->
      {#each years as y, i}
        <text x={xOf(i)} y={H - PB + 18} text-anchor="middle"
          font-size="12" font-weight={y === '2026' ? '700' : '500'}
          fill={y === '2026' ? '#007A5A' : 'rgba(11,29,58,.55)'}
          font-family="Helvetica Neue,sans-serif">
          {yearMeta[y].label}{y === '2026' ? '*' : ''}
        </text>
        <text x={xOf(i)} y={H - PB + 32} text-anchor="middle"
          font-size="9" fill="rgba(11,29,58,.3)"
          font-family="Helvetica Neue,sans-serif">
          {yearMeta[y].periodo}
        </text>
      {/each}

      <!-- sección lines (debajo del total) -->
      {#each extraLines as sl}
        {#if animated}
          <path d={makePath(sl.points)} fill="none"
            stroke={sl.color} stroke-width="2"
            stroke-linecap="round" stroke-linejoin="round"
            opacity="0.7" class="anim-path"/>
        {/if}
        {#each sl.points as p, i}
          <circle cx={xOf(i)} cy={yOf(p.mod)} r="4"
            fill={sl.color} opacity="0.8"/>
        {/each}
      {/each}

      <!-- total line -->
      {#if animated}
        <path d={makePath(totalLine)} fill="none"
          stroke="#01f3b3" stroke-width="3.5"
          stroke-linecap="round" stroke-linejoin="round"
          class="anim-path total-path"/>
      {/if}

      <!-- value labels above total points -->
      {#each totalLine as t, i}
        {@const cx = xOf(i)}
        {@const cy = yOf(t.mod)}
        <!-- Dot -->
        <circle {cx} {cy} r="5.5" fill="#01f3b3" stroke="#fff" stroke-width="2"/>
        <!-- Label -->
        {#if animated}
          <text x={cx} y={cy - 14} text-anchor="middle"
            font-size="11" font-weight="700" fill="#007A5A"
            font-family="Helvetica Neue,sans-serif"
            class="val-label">
            +{fmtMil(t.mod)}
          </text>
        {/if}
      {/each}

      <!-- hover guide -->
      {#if hoverIdx !== null}
        <line
          x1={xOf(hoverIdx)} y1={PT}
          x2={xOf(hoverIdx)} y2={H - PB}
          stroke="rgba(11,29,58,.15)" stroke-width="1" stroke-dasharray="4,3"/>
        <!-- rehighlight dots -->
        <circle cx={xOf(hoverIdx)} cy={yOf(totalLine[hoverIdx].mod)}
          r="7" fill="#01f3b3" stroke="#fff" stroke-width="2.5"/>
        {#each extraLines as sl}
          <circle cx={xOf(hoverIdx)} cy={yOf(sl.points[hoverIdx].mod)}
            r="5.5" fill={sl.color} stroke="#fff" stroke-width="2"/>
        {/each}
      {/if}

      <!-- Y axis label -->
      <text x="12" y={H / 2} text-anchor="middle"
        transform="rotate(-90,12,{H/2})"
        font-size="9" fill="rgba(11,29,58,.3)"
        font-family="Helvetica Neue,sans-serif">Millones €</text>
    </svg>

    <!-- hover tooltip -->
    {#if hoverIdx !== null}
      <div class="lc-tt" style="left:{ttX + 14}px; top:{ttY - 90}px">
        <strong>{yearMeta[years[hoverIdx]].label}{years[hoverIdx] === '2026' ? '*' : ''}</strong>
        <div class="lc-tt-total">
          <span class="lc-tt-dot" style="background:#01f3b3"></span>
          Total: <strong>+{fmtMil(totalLine[hoverIdx].mod)}</strong>
        </div>
        {#each extraLines as sl}
          <div class="lc-tt-row">
            <span class="lc-tt-dot" style="background:{sl.color}"></span>
            <span class="lc-tt-name">{sl.nombre.slice(0, 28)}</span>
            <span class="lc-tt-val" style="color:{sl.color}">
              {sl.points[hoverIdx].mod >= 0 ? '+' : ''}{fmtMil(sl.points[hoverIdx].mod)}
            </span>
          </div>
        {/each}
      </div>
    {/if}
  </div>

  <!-- Section picker + active lines -->
  <div class="lc-picker-area">
    <div class="lc-picker">
      <label class="lc-picker-lbl" for="sec-select">Añadir sección al gráfico</label>
      <div class="lc-picker-row">
        <select
          id="sec-select"
          class="lc-select"
          bind:value={selectValue}
          onchange={() => addSection(selectValue)}
        >
          <option value="">Selecciona una sección…</option>
          {#each availableOptions as s}
            <option value={s.codigo}>
              {s.codigo} — {s.nombre.replace(/\.$/, '').slice(0, 48)}
            </option>
          {/each}
        </select>
      </div>
    </div>

    {#if extraLines.length > 0}
      <div class="lc-active-lines">
        {#each extraLines as sl}
          <div class="lc-line-tag" style="border-color:{sl.color}">
            <span class="lc-line-dot" style="background:{sl.color}"></span>
            <span class="lc-line-name">{sl.nombre.replace(/\.$/, '').slice(0, 36)}</span>
            <button class="lc-line-rm" onclick={() => removeSection(sl.codigo)}
              aria-label="Quitar {sl.nombre}">✕</button>
          </div>
        {/each}
      </div>
    {/if}
  </div>

  <p class="lc-footnote">* 2026: datos hasta Abril (primer trimestre) — no comparable directamente con ejercicios completos.</p>
</div>

<style>
  .lc-page { max-width:960px; margin:0 auto; padding:.75rem 1.5rem 2rem; width:100%; }

  .lc-head { margin-bottom:.75rem; }
  .lc-head h1 { font-size:clamp(1rem,2.2vw,1.35rem); font-weight:700; letter-spacing:-.03em; margin-bottom:.25rem; }
  .lc-head p  { font-size:.78rem; color:rgba(11,29,58,.5); max-width:580px; line-height:1.5; }
  .lc-head p strong { color:#0B1D3A; }

  /* strip */
  .lc-strip {
    display:grid; grid-template-columns:repeat(4,1fr);
    gap:1px; background:rgba(11,29,58,.08);
    border:1px solid rgba(11,29,58,.08); border-radius:10px;
    overflow:hidden; margin-bottom:1rem;
  }
  .lc-strip-item {
    background:#fff; padding:.55rem .85rem;
    display:flex; flex-direction:column; gap:.12rem;
    transition:background .15s;
  }
  .lc-strip-item--hover { background:rgba(1,243,179,.07); }
  .lcs-year { font-size:.65rem; text-transform:uppercase; letter-spacing:.08em; color:rgba(11,29,58,.4); }
  .lcs-val  { font-size:1.1rem; font-weight:700; color:#0B1D3A; letter-spacing:-.03em; font-variant-numeric:tabular-nums; line-height:1.1; }
  .lcs-exec { font-size:.68rem; color:#007A5A; margin-top:.05rem; }

  /* chart */
  .lc-chart-wrap { position:relative; margin-bottom:1rem; }
  .lc-svg { display:block; overflow:visible; cursor:crosshair; }

  .anim-path { stroke-dasharray:4000; stroke-dashoffset:4000; animation:drawLine 1.2s cubic-bezier(.4,0,.2,1) forwards; }
  .total-path { animation-duration:.9s; }
  @keyframes drawLine { to { stroke-dashoffset:0; } }

  .val-label { opacity:0; animation:fadeUp .4s ease forwards; animation-delay:.8s; }
  @keyframes fadeUp { from{opacity:0;transform:translateY(4px)} to{opacity:1;transform:translateY(0)} }

  /* tooltip */
  .lc-tt {
    position:fixed; background:#0B1D3A; color:#fff;
    padding:.7rem 1rem; border-radius:8px; font-size:.75rem;
    pointer-events:none; z-index:500; min-width:180px;
    box-shadow:0 6px 24px rgba(11,29,58,.25);
    display:flex; flex-direction:column; gap:.3rem;
  }
  .lc-tt strong { font-size:.82rem; display:block; margin-bottom:.1rem; }
  .lc-tt-total  { display:flex; align-items:center; gap:.45rem; color:#fff; }
  .lc-tt-total strong { display:inline; }
  .lc-tt-row    { display:flex; align-items:center; gap:.45rem; }
  .lc-tt-dot    { width:8px; height:8px; border-radius:50%; flex-shrink:0; }
  .lc-tt-name   { color:rgba(255,255,255,.55); flex:1; }
  .lc-tt-val    { font-weight:600; }
  @media (hover:none) { .lc-tt { display:none; } }

  /* section picker */
  .lc-picker-area { margin-top:.5rem; display:flex; flex-direction:column; gap:1rem; }
  .lc-picker { display:flex; flex-direction:column; gap:.5rem; }
  .lc-picker-lbl { font-size:.68rem; text-transform:uppercase; letter-spacing:.1em; color:rgba(11,29,58,.4); }
  .lc-picker-row { display:flex; gap:.75rem; flex-wrap:wrap; }
  .lc-select {
    flex:1; min-width:220px; max-width:520px;
    padding:.5rem .75rem; border-radius:6px;
    border:1px solid rgba(11,29,58,.15); font-size:.85rem;
    color:#0B1D3A; background:#fff; cursor:pointer;
    font-family:'Helvetica Neue',sans-serif;
    transition:border-color .15s;
    appearance:none;
    background-image:url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8' viewBox='0 0 12 8'%3E%3Cpath d='M1 1l5 5 5-5' stroke='%230B1D3A' stroke-width='1.5' fill='none' stroke-linecap='round'/%3E%3C/svg%3E");
    background-repeat:no-repeat;
    background-position:right .75rem center;
    padding-right:2rem;
  }
  .lc-select:focus { outline:none; border-color:#01f3b3; }

  /* active section tags */
  .lc-active-lines { display:flex; flex-wrap:wrap; gap:.5rem; }
  .lc-line-tag {
    display:flex; align-items:center; gap:.45rem;
    padding:.3rem .6rem .3rem .5rem;
    border-radius:20px; border:1.5px solid;
    background:#fff; font-size:.78rem;
  }
  .lc-line-dot  { width:8px; height:8px; border-radius:50%; flex-shrink:0; }
  .lc-line-name { color:#0B1D3A; font-weight:500; }
  .lc-line-rm   {
    all:unset; cursor:pointer; font-size:.7rem;
    color:rgba(11,29,58,.3); line-height:1;
    padding:0 .1rem; transition:color .15s;
  }
  .lc-line-rm:hover { color:#dc2626; }

  .lc-footnote { margin-top:1.5rem; font-size:.72rem; color:rgba(11,29,58,.35); font-style:italic; }

  @media (max-width:640px) {
    .lc-page  { padding:2rem 1.25rem 3rem; }
    .lc-strip { grid-template-columns:repeat(2,1fr); }
    .lcs-val  { font-size:1.15rem; }
    .lc-select { min-width:0; width:100%; }
  }
</style>
