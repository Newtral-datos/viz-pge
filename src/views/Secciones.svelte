<script>
  import { onMount } from 'svelte';
  import { secciones, years, yearMeta } from '../lib/data.js';

  const displayYears = years.filter(y => y !== '2023');
  const sep     = n => { const r=Math.round(n); return (r<0?'-':'')+Math.abs(r).toString().replace(/\B(?=(\d{3})+(?!\d))/g,'.'); };
  const fmtMill = n => sep(n/1000);
  const fmtK    = n => sep(n);
  const pct = (a,b) => b ? Math.round(a/b*100) : 0;

  let sortCol = $state('acu');
  let sortDir = $state(-1);
  let expanded = $state(null);
  let detYear  = $state('2026');

  let rows = $derived(
    secciones
      .map(s => {
        const acuMod = displayYears.reduce((sum, y) => sum + (s.years[y]?.mod || 0), 0);
        const acuIni = displayYears.reduce((sum, y) => sum + (s.years[y]?.credito_inicial || 0), 0);
        return { ...s, acuMod, acuIni };
      })
      .sort((a, b) => {
        const va = sortCol === 'acu' ? a.acuMod : (a.years[sortCol]?.mod || 0);
        const vb = sortCol === 'acu' ? b.acuMod : (b.years[sortCol]?.mod || 0);
        return sortDir * (vb - va);
      })
  );

  function sort(col) {
    if (sortCol === col) sortDir = -sortDir;
    else { sortCol = col; sortDir = -1; }
  }

  function toggle(codigo) {
    expanded = expanded === codigo ? null : codigo;
  }

  function modColor(m) {
    return m > 500 ? '#007A5A' : m < -500 ? '#dc2626' : 'rgba(73,73,73,.35)';
  }

  onMount(() => {
    const hl = sessionStorage.getItem('highlightSec');
    if (hl) {
      expanded = hl;
      sessionStorage.removeItem('highlightSec');
      setTimeout(() => {
        document.getElementById('sec-' + hl)?.scrollIntoView({ behavior: 'smooth', block: 'center' });
      }, 150);
    }
  });
</script>

<div class="sc-page">
  <div class="sc-header">
    <div>
      <h1 class="sc-title">Secciones ministeriales</h1>
      <p class="sc-sub">37 secciones presupuestarias · Modificaciones de crédito 2024–2026</p>
    </div>
    <span class="sc-note">2026 hasta Abril</span>
  </div>

  <div class="sc-table-wrap">
    <table class="sc-table">
      <thead>
        <tr>
          <th class="th-sec">Sección</th>
          {#each displayYears as y}
            <th class="th-yr th-click" class:th-on={sortCol===y} onclick={()=>sort(y)}>
              {yearMeta[y].label}{#if sortCol===y}<span class="sort-ind">{sortDir<0?'↓':'↑'}</span>{/if}
            </th>
          {/each}
          <th class="th-acu th-click" class:th-on={sortCol==='acu'} onclick={()=>sort('acu')}>
            Acumulado{#if sortCol==='acu'}<span class="sort-ind">{sortDir<0?'↓':'↑'}</span>{/if}
          </th>
          <th class="th-pct">% acum.</th>
          <th class="th-exp"></th>
        </tr>
      </thead>
      <tbody>
        {#each rows as s}
          {@const isOpen = expanded === s.codigo}
          <tr id="sec-{s.codigo}" class="tr-row" class:tr-open={isOpen} onclick={()=>toggle(s.codigo)}>
            <td class="td-sec">
              <span class="td-cod">{s.codigo}</span>
              <span class="td-nom">{s.nombre.replace(/\.$/, '')}</span>
            </td>
            {#each displayYears as y}
              {@const m = s.years[y]?.mod ?? 0}
              <td class="td-yr" style="color:{modColor(m)}">{Math.abs(m) > 500 ? (m>0?'+':'')+fmtMill(m) : '—'}</td>
            {/each}
            <td class="td-acu" style="color:{modColor(s.acuMod)}">
              {Math.abs(s.acuMod) > 500 ? (s.acuMod>0?'+':'')+fmtMill(s.acuMod) : '—'}
            </td>
            <td class="td-pct" style="color:{modColor(s.acuMod)}">
              {s.acuIni > 0 && Math.abs(s.acuMod) > 500 ? '+'+pct(s.acuMod, s.acuIni)+'%' : '—'}
            </td>
            <td class="td-exp">{isOpen ? '▲' : '▼'}</td>
          </tr>
          {#if isOpen}
            <tr class="tr-detail">
              <td colspan="7">
                <div class="det-inner">
                  <div class="det-ytabs">
                    {#each displayYears as y}
                      <button class="det-ytab" class:det-ytab-on={detYear===y} onclick={()=>detYear=y}>
                        {yearMeta[y].label}
                      </button>
                    {/each}
                  </div>

                  {#if s.years[detYear]}
                    {@const dy = s.years[detYear]}
                    <div class="det-kpis">
                      <div class="dk"><span class="dk-v">{fmtMill(dy.credito_inicial)}</span><span class="dk-u">mill.</span><span class="dk-l">Inicial</span></div>
                      <div class="dk dk-a"><span class="dk-v">{dy.mod>=0?'+':''}{fmtMill(dy.mod)}</span><span class="dk-u">mill.</span><span class="dk-l">Modificación</span></div>
                      <div class="dk"><span class="dk-v">{fmtMill(dy.credito_total)}</span><span class="dk-u">mill.</span><span class="dk-l">Total</span></div>
                      <div class="dk dk-e"><span class="dk-v">{dy.mod>=0?'+':''}{pct(dy.mod,dy.credito_inicial)}<small>%</small></span><span class="dk-l">% Modificado</span></div>
                    </div>
                    {#if dy.credito_inicial > 0}
                      <div class="det-bar">
                        <div class="det-bar-i" style="flex:{dy.credito_inicial};min-width:4px" title="Inicial"></div>
                        {#if dy.mod>0}<div class="det-bar-m" style="flex:{dy.mod};min-width:4px" title="Modificación"></div>{/if}
                      </div>
                    {/if}
                    {#if dy.organismos?.length >= 1}
                      {@const orgs = [...dy.organismos]
                        .map(o => ({ ...o, mod: o.credito_total - o.credito_inicial }))
                        .sort((a,b) => b.credito_total - a.credito_total)}
                      {@const maxOrg = orgs[0]?.credito_total || 1}
                      <p class="det-org-lbl">Subcategorías — {orgs.length} organismo{orgs.length!==1?'s':''}</p>
                      <div class="org-table-wrap">
                        <table class="org-table">
                          <thead>
                            <tr>
                              <th class="oth-name">Organismo</th>
                              <th class="oth-num">Inicial (miles €)</th>
                              <th class="oth-num">Modificación (miles €)</th>
                              <th class="oth-num">Total (miles €)</th>
                              <th class="oth-pct">% mod.</th>
                            </tr>
                          </thead>
                          <tbody>
                            {#each orgs as o}
                              {@const omod = o.credito_total - o.credito_inicial}
                              {@const ocode = o.organica.match(/^(\d+)/)?.[1] ?? ''}
                              {@const oname = o.organica.replace(/^\d+\s*[-–]\s*/,'').trim()}
                              {@const barW = (o.credito_total / maxOrg * 100).toFixed(1)}
                              <tr class="org-row">
                                <td class="otd-name">
                                  <span class="otd-code">{ocode}</span>
                                  <div class="otd-nameblock">
                                    <span class="otd-label">{oname}</span>
                                    <div class="otd-bar" style="width:{barW}%"></div>
                                  </div>
                                </td>
                                <td class="otd-num">{fmtK(o.credito_inicial)}</td>
                                <td class="otd-num otd-mod" style="color:{modColor(omod)}">
                                  {omod !== 0 ? (omod>0?'+':'')+fmtK(omod) : '—'}
                                </td>
                                <td class="otd-num otd-total">{fmtK(o.credito_total)}</td>
                                <td class="otd-pct" style="color:{modColor(omod)}">
                                  {o.credito_inicial > 0 && omod !== 0 ? (omod>=0?'+':'')+pct(omod,o.credito_inicial)+'%' : '—'}
                                </td>
                              </tr>
                            {/each}
                          </tbody>
                        </table>
                      </div>
                    {/if}
                  {:else}
                    <p class="det-nodata">Sin datos disponibles para {yearMeta[detYear].label}</p>
                  {/if}
                </div>
              </td>
            </tr>
          {/if}
        {/each}
      </tbody>
    </table>
  </div>

  <footer class="sc-footer">
    <span>Ministerio de Hacienda · Modificaciones de Crédito 2024–2026</span>
    <span>Cifras en millones de euros · 2026 hasta Abril</span>
  </footer>
</div>

<style>
  .sc-page { max-width:1100px;margin:0 auto;padding:.75rem 1.5rem 2rem;width:100%; }

  .sc-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    margin-bottom: .75rem;
    flex-wrap: wrap;
    gap: .4rem;
  }
  .sc-title { font-size:clamp(1rem,2vw,1.3rem);font-weight:700;letter-spacing:-.04em;color:#494949;margin-bottom:.15rem; }
  .sc-sub   { font-size:.78rem;color:rgba(73,73,73,.45); }
  .sc-note  { font-size:.65rem;font-weight:700;color:#007A5A;background:rgba(1,243,179,.12);padding:.25rem .65rem;border-radius:4px;white-space:nowrap;flex-shrink:0; }

  /* table */
  .sc-table-wrap { width:100%;overflow-x:auto; }
  .sc-table { width:100%;border-collapse:collapse;font-size:.8rem; }

  thead th {
    padding: .32rem .55rem;
    text-align: right;
    font-size: .6rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: .08em;
    color: rgba(73,73,73,.4);
    border-bottom: 2px solid rgba(73,73,73,.1);
    white-space: nowrap;
    background: #fff;
  }
  .th-sec  { text-align:left; }
  .th-click { cursor:pointer; }
  .th-click:hover { color:#007A5A; }
  .th-on   { color:#007A5A; }
  .th-acu  { font-weight:800; }
  .sort-ind { margin-left:.2rem;font-size:.7em; }

  /* rows */
  .tr-row {
    cursor: pointer;
    border-bottom: 1px solid rgba(73,73,73,.06);
    transition: background .1s;
  }
  .tr-row:hover { background: rgba(73,73,73,.025); }
  .tr-open      { background: rgba(1,243,179,.04);border-bottom:none; }
  .tr-open:hover{ background: rgba(1,243,179,.07); }

  td { padding:.32rem .55rem;vertical-align:middle; }
  .td-sec { text-align:left;min-width:170px; }
  .td-cod { font-size:.55rem;font-weight:700;letter-spacing:.1em;color:rgba(73,73,73,.3);margin-right:.35rem; }
  .td-nom { font-size:.75rem;font-weight:500;color:#494949;line-height:1.3; }
  .td-yr, .td-acu, .td-pct { text-align:right;font-variant-numeric:tabular-nums;white-space:nowrap; }
  .td-acu { font-weight:700; }
  .td-pct { font-size:.72rem; }
  .td-exp { text-align:center;font-size:.55rem;color:rgba(73,73,73,.28);padding-right:.5rem;width:28px; }

  /* expansion row */
  .tr-detail { background:rgba(1,243,179,.025); }
  .tr-detail td { padding:0;border-bottom:1px solid rgba(73,73,73,.08); }
  .det-inner { padding:.75rem 1rem 1rem; }

  /* year tabs */
  .det-ytabs { display:flex;gap:.3rem;margin-bottom:.6rem; }
  .det-ytab {
    all: unset;
    cursor: pointer;
    padding: .25rem .65rem;
    border-radius: 20px;
    font-size: .7rem;
    font-weight: 500;
    border: 1px solid rgba(73,73,73,.12);
    color: rgba(73,73,73,.45);
    transition: all .15s;
    white-space: nowrap;
  }
  .det-ytab:hover    { border-color:rgba(1,243,179,.45);color:#007A5A; }
  .det-ytab-on       { background:#007A5A;color:#fff;border-color:#007A5A; }

  /* KPI cards */
  .det-kpis {
    display: grid;
    grid-template-columns: repeat(4,1fr);
    gap: 1px;
    background: rgba(73,73,73,.07);
    border-radius: 9px;
    overflow: hidden;
    margin-bottom: .55rem;
    border: 1px solid rgba(73,73,73,.07);
  }
  .dk   { background:#fff;padding:.5rem .7rem;display:flex;flex-direction:column;gap:.12rem; }
  .dk-a { background:rgba(1,243,179,.06); }
  .dk-e { background:rgba(5,150,105,.04); }
  .dk-v { font-size:.82rem;font-weight:700;letter-spacing:-.02em;font-variant-numeric:tabular-nums;color:#494949; }
  .dk-a .dk-v { color:#007A5A; }
  .dk-e .dk-v { color:#059669; }
  .dk-v small { font-size:.6em;font-weight:500; }
  .dk-u { font-size:.58rem;color:rgba(73,73,73,.35);margin-top:-.1rem; }
  .dk-l { font-size:.58rem;text-transform:uppercase;letter-spacing:.08em;color:rgba(73,73,73,.38); }

  /* composition bar */
  .det-bar   { height:6px;border-radius:4px;overflow:hidden;display:flex;margin-bottom:.65rem; }
  .det-bar-i { background:#494949; }
  .det-bar-m { background:#01f3b3; }

  /* organisms subcategory table */
  .det-org-lbl { font-size:.6rem;text-transform:uppercase;letter-spacing:.09em;color:rgba(73,73,73,.38);margin-bottom:.6rem; }
  .det-nodata  { font-size:.78rem;color:rgba(73,73,73,.35);padding:.5rem 0; }

  .org-table-wrap { width:100%;overflow-x:auto; }
  .org-table { width:100%;border-collapse:collapse;font-size:.75rem; }
  .org-table thead th {
    padding: .35rem .6rem;
    font-size: .58rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: .07em;
    color: rgba(73,73,73,.35);
    border-bottom: 1px solid rgba(73,73,73,.1);
    background: transparent;
  }
  .oth-name { text-align:left; }
  .oth-num  { text-align:right;white-space:nowrap; }
  .oth-pct  { text-align:right;white-space:nowrap; }

  .org-row { border-bottom:1px solid rgba(73,73,73,.04); }
  .org-row:last-child { border-bottom:none; }
  .org-row:hover { background:rgba(73,73,73,.018); }

  .otd-name {
    display: flex;
    align-items: flex-start;
    gap: .4rem;
    padding: .35rem .5rem;
    max-width: 320px;
  }
  .otd-code {
    font-size: .58rem;
    font-weight: 700;
    color: rgba(73,73,73,.3);
    letter-spacing: .08em;
    flex-shrink: 0;
    padding-top: .1rem;
    min-width: 28px;
  }
  .otd-nameblock { display:flex;flex-direction:column;gap:.22rem;flex:1;min-width:0; }
  .otd-label { font-size:.75rem;color:#494949;line-height:1.3; }
  .otd-bar {
    height: 3px;
    background: rgba(1,243,179,.5);
    border-radius: 2px;
    max-width: 100%;
    transition: width .4s ease;
  }

  .otd-num {
    text-align: right;
    padding: .35rem .5rem;
    font-variant-numeric: tabular-nums;
    white-space: nowrap;
    color: rgba(73,73,73,.5);
  }
  .otd-mod   { font-weight:600; }
  .otd-total { font-weight:700;color:#494949; }
  .otd-pct   { text-align:right;padding:.35rem .5rem;font-size:.72rem;font-weight:600;white-space:nowrap; }

  .sc-footer {
    border-top: 1px solid rgba(73,73,73,.07);
    padding-top: 1rem;
    margin-top: 1.25rem;
    display: flex;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: .5rem;
    font-size: .63rem;
    color: rgba(73,73,73,.28);
    letter-spacing: .04em;
  }

  @media (max-width:640px) {
    .sc-page { padding:1.25rem 1rem 3rem; }
    .th-yr { display:none; }
    .td-yr { display:none; }
    .det-kpis { grid-template-columns:repeat(2,1fr); }
    .sc-table { font-size:.75rem; }
  }
</style>
