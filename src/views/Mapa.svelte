<script>
  import { secciones, years, yearMeta } from '../lib/data.js';

  const sep   = n => { const r=Math.round(n); return (r<0?'-':'')+Math.abs(r).toString().replace(/\B(?=(\d{3})+(?!\d))/g,'.'); };
  const fmtMM = n => sep(n/1e6);
  const fmtM  = n => sep(n/1e3);

  // ── Sección 38 organismos con coordenadas de tile ─────────
  const TILES = [
    { code:'3803', name:'Galicia',           abbr:'GAL', col:0, row:0 },
    { code:'3805', name:'Asturias',          abbr:'AST', col:1, row:0 },
    { code:'3806', name:'Cantabria',         abbr:'CNT', col:2, row:0 },
    { code:'3801', name:'País Vasco',        abbr:'PVA', col:3, row:0 },
    { code:'3813', name:'Navarra',           abbr:'NAV', col:5, row:0 },
    { code:'3807', name:'La Rioja',          abbr:'RIO', col:6, row:0 },
    { code:'3817', name:'Castilla y León',   abbr:'CYL', col:1, row:1 },
    { code:'3810', name:'Aragón',            abbr:'ARA', col:4, row:1 },
    { code:'3802', name:'Cataluña',          abbr:'CAT', col:6, row:1 },
    { code:null,   name:'Com. de Madrid',    abbr:'MAD', col:3, row:2, nodata:true },
    { code:'3814', name:'Extremadura',       abbr:'EXT', col:0, row:3 },
    { code:'3811', name:'C.-La Mancha',      abbr:'CLM', col:3, row:3 },
    { code:null,   name:'C. Valenciana',     abbr:'VAL', col:5, row:3, nodata:true },
    { code:'3804', name:'Andalucía',         abbr:'AND', col:1, row:4 },
    { code:null,   name:'Murcia',            abbr:'MUR', col:4, row:4, nodata:true },
    { code:'3815', name:'Illes Balears',     abbr:'IBA', col:6, row:4 },
    { code:'3819', name:'Ceuta',             abbr:'CEU', col:3, row:5 },
    { code:'3818', name:'Melilla',           abbr:'MEL', col:4, row:5 },
    { code:'3812', name:'Canarias',          abbr:'CAN', col:0, row:6 },
  ];

  const displayYears = years.filter(y => y !== '2023');
  const sec38 = secciones.find(s => s.codigo === '38');

  // ── estado ─────────────────────────────────────────────────
  let selectedYear = $state(null);   // null = acumulado general 2024–2026
  let hovered      = $state(null);
  let detail       = $state(null);

  // ── datos del año activo (null en modo general) ────────────
  let s38year = $derived(selectedYear ? sec38?.years[selectedYear] : null);

  // helper: credito_total de un organismo en un año concreto
  function orgYear(orgCode, y) {
    if (!orgCode) return 0;
    const orgs = sec38?.years[y]?.organismos || [];
    return orgs.find(o => o.organica.startsWith(orgCode))?.credito_total || 0;
  }
  function orgYearIni(orgCode, y) {
    if (!orgCode) return 0;
    const orgs = sec38?.years[y]?.organismos || [];
    return orgs.find(o => o.organica.startsWith(orgCode))?.credito_inicial || 0;
  }

  // modificación de un tile: un año o acumulado
  function orgValue(orgCode) {
    if (!orgCode) return null;
    if (selectedYear) {
      const v = orgYear(orgCode, selectedYear) - orgYearIni(orgCode, selectedYear);
      return v !== 0 ? v : null;
    }
    const total = displayYears.reduce((s, y) => s + orgYear(orgCode, y) - orgYearIni(orgCode, y), 0);
    return total !== 0 ? total : null;
  }

  // map tiles with values
  let tileData = $derived(
    TILES.map(t => {
      const val = t.nodata ? null : orgValue(t.code);
      return { ...t, val };
    })
  );

  // acumulado sección 38 completa
  let s38acu = $derived({
    credito_total: displayYears.reduce((s,y) => s + (sec38?.years[y]?.credito_total||0), 0),
    mod:           displayYears.reduce((s,y) => s + (sec38?.years[y]?.mod||0), 0),
  });

  // acumulado de la CCA seleccionada en el detalle
  let detailAcu = $derived(() => {
    if (!detail?.code) return null;
    const ct  = displayYears.reduce((s,y) => s + orgYear(detail.code, y), 0);
    const ini = displayYears.reduce((s,y) => s + orgYearIni(detail.code, y), 0);
    return { credito_total: ct, credito_inicial: ini, mod: ct - ini };
  });

  // aggregate partidas
  function aggPartida(code) {
    if (selectedYear) {
      return sec38?.years[selectedYear]?.organismos?.find(o=>o.organica.startsWith(code))?.credito_total || 0;
    }
    return displayYears.reduce((s,y) => s + (sec38?.years[y]?.organismos?.find(o=>o.organica.startsWith(code))?.credito_total||0), 0);
  }

  // max value for color scale (absolute, mods can be negative)
  let maxVal = $derived(() => {
    const vals = tileData.map(t=>t.val).filter(v=>v!==null).map(Math.abs);
    return vals.length ? Math.max(...vals) : 1;
  });

  // Color: white → dark teal → #01f3b3
  function tileColor(val) {
    if (val === null) return '#F0F4F8';
    const t = Math.sqrt(Math.abs(val) / maxVal());
    const r = Math.round(255 + t*(1-255));
    const g = Math.round(255 + t*(243-255));
    const b = Math.round(255 + t*(179-255));
    return `rgb(${r},${g},${b})`;
  }

  function textColor(val) {
    if (val === null) return 'rgba(11,29,58,0.3)';
    const t = Math.sqrt(val / maxVal());
    return t > 0.55 ? '#0B1D3A' : '#0B1D3A';
  }

  // ── Tile SVG layout ─────────────────────────────────────────
  const TS = 50;   // tile size px (SVG units)
  const PAD = 6;   // gap between tiles
  const COLS = 7, ROWS = 7;
  const SVG_W = COLS*(TS+PAD)+PAD;
  const SVG_H = ROWS*(TS+PAD)+PAD;

  function tileX(col) { return PAD + col*(TS+PAD); }
  function tileY(row) { return PAD + row*(TS+PAD); }
</script>

<div class="mp-page">
  <div class="mp-head">
    <h1>Financiación territorial</h1>
  </div>

  <!-- Year selector -->
  <div class="mp-ys">
    <span class="mp-ys-lbl">Ejercicio</span>
    <div class="mp-pills">
      <button class="mp-pill" class:mp-pill--on={selectedYear===null} onclick={()=>{selectedYear=null;detail=null;}}>
        General
      </button>
      {#each displayYears as y}
        <button class="mp-pill" class:mp-pill--on={selectedYear===y} onclick={()=>{selectedYear=y;detail=null;}}>
          {yearMeta[y].label}
        </button>
      {/each}
    </div>
    <span class="mp-period">{selectedYear ? yearMeta[selectedYear].periodo : '2024–2026 acumulado'}</span>
  </div>

  <!-- Main layout: map + detail -->
  <div class="mp-body">
    <!-- TILE MAP -->
    <div class="mp-map-wrap">
      <svg
        viewBox="0 0 {SVG_W} {SVG_H}"
        width="100%"
        class="mp-svg"
        aria-label="Mapa de financiación territorial por comunidad autónoma"
      >
        {#each tileData as t}
          {@const x = tileX(t.col)}
          {@const y = tileY(t.row)}
          {@const col = tileColor(t.val)}
          {@const tc  = textColor(t.val)}
          {@const isHov = hovered === t.abbr}
          {@const isSel = detail?.abbr === t.abbr}

          <g
            onclick={() => { detail = t.nodata ? null : t; }}
            onmouseenter={() => hovered = t.abbr}
            onmouseleave={() => hovered = null}
            onkeydown={e => e.key==='Enter' && !t.nodata && (detail = t)}
            role="button"
            tabindex={t.nodata ? -1 : 0}
            aria-label={t.name}
            style="cursor:{t.nodata?'default':'pointer'}"
          >
            <!-- Tile background -->
            <rect
              {x} {y}
              width={TS} height={TS}
              fill={col}
              rx="6"
              stroke={isSel ? '#007A5A' : isHov && !t.nodata ? '#01f3b3' : 'transparent'}
              stroke-width={isSel ? 2.5 : 2}
            />

            <!-- Abbreviation -->
            <text
              x={x + TS/2} y={y + TS/2 - 5}
              text-anchor="middle" dominant-baseline="middle"
              font-size="10" font-weight="700"
              fill={tc}
              font-family="Helvetica Neue,sans-serif"
            >{t.abbr}</text>

            <!-- Value or "sin datos" -->
            {#if t.val !== null}
              <text
                x={x + TS/2} y={y + TS/2 + 9}
                text-anchor="middle"
                font-size="7.5" font-weight="400"
                fill={tc}
                font-family="Helvetica Neue,sans-serif"
              >{t.val > 0 ? '+' : ''}{fmtM(t.val)} M</text>
            {:else}
              <text
                x={x + TS/2} y={y + TS/2 + 9}
                text-anchor="middle"
                font-size="6.5" fill="rgba(11,29,58,.3)"
                font-family="Helvetica Neue,sans-serif"
              >sin datos</text>
            {/if}
          </g>
        {/each}
      </svg>

      <!-- Color legend -->
      <div class="mp-legend">
        <span class="mp-leg-lbl">Menor modificación</span>
        <div class="mp-leg-grad"></div>
        <span class="mp-leg-lbl">Mayor modificación</span>
        <span class="mp-leg-nd"><span class="mp-leg-sq"></span>Sin desglose individual</span>
      </div>
    </div>

    <!-- Detail panel -->
    <div class="mp-detail">
      {#if detail}
        {@const acu = detailAcu()}
        {@const maxV = Math.max(...displayYears.map(y => orgYear(detail.code, y)))}
        <button class="mp-det-close" onclick={()=>detail=null}>✕</button>
        <p class="mp-det-abbr">{detail.abbr}</p>
        <h3 class="mp-det-name">{detail.name}</h3>

        {#if acu && acu.credito_total > 0}
          <!-- KPIs: muestra el año activo o el acumulado -->
          {#if selectedYear}
            {@const org = s38year?.organismos?.find(o=>o.organica.startsWith(detail.code))}
            {#if org}
              <div class="mp-det-kpis">
                <div class="mp-dk">
                  <span class="mp-dk-v">{fmtM(org.credito_total)} mill.</span>
                  <span class="mp-dk-l">Total {yearMeta[selectedYear].label}</span>
                </div>
                <div class="mp-dk">
                  <span class="mp-dk-v" style="color:{org.credito_total-org.credito_inicial>0?'#007A5A':'rgba(11,29,58,.4)'}">
                    {org.credito_total-org.credito_inicial>0?'+':''}{fmtM(org.credito_total-org.credito_inicial)} mill.
                  </span>
                  <span class="mp-dk-l">Modificación</span>
                </div>
                <div class="mp-dk">
                  <span class="mp-dk-v" style="color:#007A5A">+{org.credito_inicial>0?Math.round((org.credito_total-org.credito_inicial)/org.credito_inicial*100):0}%</span>
                  <span class="mp-dk-l">% Modificado</span>
                </div>
              </div>
            {:else}
              <p class="mp-nodata">Sin datos para {yearMeta[selectedYear].label}.</p>
            {/if}
          {:else}
            <div class="mp-det-kpis">
              <div class="mp-dk">
                <span class="mp-dk-v">{fmtM(acu.credito_total)} mill.</span>
                <span class="mp-dk-l">Total acumulado 2024–2026</span>
              </div>
              <div class="mp-dk">
                <span class="mp-dk-v" style="color:{acu.mod>0?'#007A5A':'rgba(11,29,58,.4)'}">
                  {acu.mod>0?'+':''}{fmtM(acu.mod)} mill.
                </span>
                <span class="mp-dk-l">Modificación acumulada</span>
              </div>
              <div class="mp-dk">
                <span class="mp-dk-v" style="color:#007A5A">+{acu.credito_inicial>0?Math.round(acu.mod/acu.credito_inicial*100):0}%</span>
                <span class="mp-dk-l">% Modificado</span>
              </div>
            </div>
          {/if}

          <!-- Evolución anual -->
          <p class="mp-det-lbl">Evolución anual</p>
          <div class="mp-evo">
            {#each displayYears as y}
              {@const v = orgYear(detail.code, y)}
              <div class="mp-evo-col" class:mp-evo-col--on={y===selectedYear}
                onclick={()=>selectedYear=y} role="button" tabindex="0"
                onkeydown={e=>e.key==='Enter'&&(selectedYear=y)}
              >
                <div class="mp-evo-bar-wrap">
                  <div class="mp-evo-bar" style="height:{maxV>0?v/maxV*100:0}%;background:{y===selectedYear?'#01f3b3':selectedYear===null?'rgba(1,243,179,.4)':'rgba(11,29,58,.15)'}"></div>
                </div>
                <span class="mp-evo-v">{fmtM(v)}</span>
                <span class="mp-evo-y">{yearMeta[y].label}</span>
              </div>
            {/each}
          </div>
        {:else}
          <p class="mp-nodata">Sin datos individuales para esta comunidad.</p>
        {/if}

      {:else}
        <!-- Resumen cuando no hay detalle -->
        <div class="mp-summary">
          <p class="mp-sum-title">Sección 38 · {selectedYear ? yearMeta[selectedYear].label : 'Acumulado 2024–2026'}</p>

          {#if selectedYear && s38year}
            <div class="mp-sum-kpi">
              <span class="mp-sum-v">{fmtM(s38year.credito_total)} millones</span>
              <span class="mp-sum-l">Crédito total</span>
            </div>
            <div class="mp-sum-kpi">
              <span class="mp-sum-v" style="color:#007A5A">+{fmtM(s38year.mod)} millones</span>
              <span class="mp-sum-l">Modificación</span>
            </div>
          {:else}
            <div class="mp-sum-kpi">
              <span class="mp-sum-v">{fmtM(s38acu.credito_total)} millones</span>
              <span class="mp-sum-l">Crédito total acumulado</span>
            </div>
            <div class="mp-sum-kpi">
              <span class="mp-sum-v" style="color:#007A5A">+{fmtM(s38acu.mod)} millones</span>
              <span class="mp-sum-l">Modificación acumulada</span>
            </div>
          {/if}

          <p class="mp-sum-note">
            <strong>Nota:</strong> Madrid, Valencia y Murcia se consolidan en
            <em>3820 — Varias CC.AA.</em>
            ({fmtM(aggPartida('3820'))} millones).
            Entidades locales: <em>3821</em>
            ({fmtM(aggPartida('3821'))} millones).
          </p>

          <p class="mp-sum-hint">← Toca un territorio para ver su detalle</p>
        </div>
      {/if}
    </div>
  </div>

  <p class="mp-desc">
    Transferencias del Estado a las Comunidades Autónomas registradas en la sección
    <strong>38 — Sistemas de Financiación de Entes Territoriales</strong>.
    Los importes en gris (<em>Varias CC.AA., Entidades Locales</em>) están agregados sin desglose autonómico en el presupuesto.
  </p>

  <footer class="mp-footer">
    <span>Ministerio de Hacienda · Sección 38 PGE 2023–2026</span>
    <span>Miles de euros · 2026 hasta Abril</span>
  </footer>
</div>

<style>
  .mp-page { max-width:1100px;margin:0 auto;padding:.75rem 1.5rem 2rem;width:100%; }
  .mp-head { margin-bottom:.75rem; }
  .mp-head h1 { font-size:clamp(1rem,2.2vw,1.35rem);font-weight:700;letter-spacing:-.03em;margin-bottom:.25rem; }

  /* year selector */
  .mp-ys { display:flex;align-items:center;gap:.6rem;flex-wrap:wrap;margin-bottom:.75rem; }
  .mp-ys-lbl { font-size:.65rem;text-transform:uppercase;letter-spacing:.1em;color:rgba(11,29,58,.4); }
  .mp-pills  { display:flex;gap:.3rem;flex-wrap:wrap; }
  .mp-pill   { all:unset;cursor:pointer;padding:.28rem .7rem;border-radius:20px;font-size:.78rem;font-weight:500;border:1px solid rgba(11,29,58,.12);color:rgba(11,29,58,.5);transition:all .15s; }
  .mp-pill:hover  { border-color:#01f3b3;color:#007A5A; }
  .mp-pill--on    { background:#0B1D3A;color:#01f3b3;border-color:#0B1D3A; }
  .mp-period { font-size:.68rem;color:rgba(11,29,58,.3);margin-left:auto; }

  /* body layout */
  .mp-body { display:grid;grid-template-columns:1fr 280px;gap:1.25rem;align-items:start; }
  @media (max-width:800px) { .mp-body { grid-template-columns:1fr;} }

  /* map */
  .mp-map-wrap { position:relative; max-width:560px; }
  .mp-svg { display:block;overflow:visible;width:100%; }

  /* legend */
  .mp-legend { display:flex;align-items:center;gap:.6rem;margin-top:.9rem;flex-wrap:wrap; }
  .mp-leg-lbl  { font-size:.68rem;color:rgba(11,29,58,.45); }
  .mp-leg-grad { flex:1;min-width:80px;height:8px;border-radius:4px;background:linear-gradient(to right,#F0F4F8,#007A5A,#01f3b3); }
  .mp-leg-nd   { display:flex;align-items:center;gap:.35rem;font-size:.68rem;color:rgba(11,29,58,.4);margin-left:.5rem; }
  .mp-leg-sq   { width:10px;height:10px;border-radius:2px;background:#F0F4F8;border:1px solid rgba(11,29,58,.15); }

  /* detail panel */
  .mp-detail {
    position:sticky;top:80px;
    background:#F8FAFC;border:1px solid rgba(11,29,58,.08);
    border-radius:12px;padding:1.25rem;
  }

  .mp-det-close { all:unset;cursor:pointer;float:right;font-size:.8rem;color:rgba(11,29,58,.35);padding:.2rem .4rem;border-radius:4px;transition:color .15s; }
  .mp-det-close:hover { color:#dc2626; }
  .mp-det-abbr  { font-size:.65rem;letter-spacing:.12em;text-transform:uppercase;color:#007A5A;font-weight:700;margin-bottom:.25rem;clear:right; }
  .mp-det-name  { font-size:1rem;font-weight:700;letter-spacing:-.02em;margin-bottom:.65rem; }

  .mp-det-kpis  { display:flex;flex-direction:column;gap:.4rem;margin-bottom:.65rem; }
  .mp-dk        { display:flex;flex-direction:column;gap:.12rem; }
  .mp-dk-v      { font-size:1rem;font-weight:700;letter-spacing:-.02em;font-variant-numeric:tabular-nums;color:#0B1D3A; }
  .mp-dk-l      { font-size:.62rem;text-transform:uppercase;letter-spacing:.08em;color:rgba(11,29,58,.4); }

  .mp-det-lbl   { font-size:.62rem;text-transform:uppercase;letter-spacing:.09em;color:rgba(11,29,58,.4);margin-bottom:.6rem; }
  .mp-evo       { display:flex;gap:.4rem; }
  .mp-evo-col   { flex:1;display:flex;flex-direction:column;align-items:center;gap:.2rem;cursor:pointer;border-radius:4px;padding:.3rem .1rem;border:1px solid transparent;transition:background .15s; }
  .mp-evo-col:hover  { background:rgba(11,29,58,.04); }
  .mp-evo-col--on    { background:rgba(1,243,179,.08);border-color:rgba(1,243,179,.3); }
  .mp-evo-bar-wrap   { width:100%;height:44px;display:flex;align-items:flex-end; }
  .mp-evo-bar        { width:100%;border-radius:2px 2px 0 0;transition:height .4s ease,background .2s; }
  .mp-evo-v   { font-size:.62rem;font-weight:600;color:rgba(11,29,58,.6);font-variant-numeric:tabular-nums; }
  .mp-evo-y   { font-size:.6rem;color:rgba(11,29,58,.35); }

  .mp-nodata { font-size:.82rem;color:rgba(11,29,58,.4);font-style:italic;padding:.5rem 0; }

  /* summary */
  .mp-summary { display:flex;flex-direction:column;gap:.65rem; }
  .mp-sum-title { font-size:.65rem;text-transform:uppercase;letter-spacing:.1em;color:rgba(11,29,58,.4); }
  .mp-sum-kpi   { display:flex;flex-direction:column;gap:.1rem; }
  .mp-sum-v     { font-size:1.15rem;font-weight:700;letter-spacing:-.03em;font-variant-numeric:tabular-nums;color:#0B1D3A; }
  .mp-sum-l     { font-size:.65rem;text-transform:uppercase;letter-spacing:.08em;color:rgba(11,29,58,.4); }
  .mp-sum-note  { font-size:.78rem;color:rgba(11,29,58,.5);line-height:1.6;padding:.75rem;background:rgba(11,29,58,.03);border-radius:6px;border-left:3px solid rgba(1,243,179,.5); }
  .mp-sum-note strong { color:#0B1D3A; }
  .mp-sum-note em     { color:#007A5A;font-style:normal; }
  .mp-sum-hint  { font-size:.75rem;color:rgba(11,29,58,.3);font-style:italic;text-align:center;padding-top:.5rem; }

  .mp-desc { font-size:.75rem;color:rgba(11,29,58,.45);max-width:700px;line-height:1.55;margin-bottom:1rem; }
  .mp-desc strong { color:rgba(11,29,58,.65); }
  .mp-desc em     { color:#007A5A;font-style:normal; }

  .mp-footer { border-top:1px solid rgba(11,29,58,.07);padding-top:1rem;margin-top:1rem;display:flex;justify-content:space-between;flex-wrap:wrap;gap:.5rem;font-size:.68rem;color:rgba(11,29,58,.3);letter-spacing:.04em; }

  @media (max-width:500px) {
    .mp-page { padding:2rem 1.25rem 3rem; }
  }
</style>
