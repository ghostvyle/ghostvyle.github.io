---
layout: post
permalink: /projects/unburden/
card_url: /projects/
title: "Unburden"
date: 2026-01-15
category: projects
tags: [llm, mcp, pentesting, automation, osint]
resumen: "Asistente de pentesting autónomo que integra un LLM local con servidores MCP para automatizar reconocimiento, análisis y flujos de trabajo de seguridad ofensiva."
---

Esta es la memoria de **Unburden**, una plataforma de pentesting automatizado que conecta un **LLM ejecutado en local** con herramientas como Nmap o Metasploit a través del **Model Context Protocol (MCP)**.

El documento recoge todo el proyecto: la motivación, el estado del arte, el diseño de la arquitectura, el desarrollo del sistema y un caso real de uso de principio a fin. Puedes leerlo completo aquí mismo o descargarlo en PDF.

<style>
.ub-doc{margin:2rem 0;border:1px solid var(--border);border-radius:var(--radius);overflow:hidden;background:var(--bg-card);box-shadow:0 8px 30px rgba(0,0,0,.4);}
.ub-doc__bar{display:flex;align-items:center;gap:1rem;padding:.85rem 1.25rem;border-bottom:1px solid var(--border);background:var(--bg-secondary);flex-wrap:wrap;}
.ub-doc__icon{display:inline-flex;align-items:center;justify-content:center;width:38px;height:38px;border-radius:var(--radius-sm);background:var(--accent-glow);color:var(--accent);flex-shrink:0;}
.ub-doc__title{font-family:var(--font-display);font-weight:600;color:var(--text-primary);line-height:1.2;}
.ub-doc__meta{font-family:var(--font-mono);font-size:.78rem;color:var(--text-muted);margin-top:.15rem;}
.ub-doc__tools{display:inline-flex;align-items:center;gap:.4rem;}
.ub-doc__btn{display:inline-flex;align-items:center;justify-content:center;gap:.4rem;height:36px;min-width:36px;padding:0 .55rem;border:1px solid var(--border-hover);border-radius:var(--radius-sm);background:var(--bg-card);color:var(--text-secondary);font-family:var(--font-mono);font-size:1rem;line-height:1;cursor:pointer;transition:var(--transition);text-decoration:none;}
.ub-doc__btn:hover{color:var(--accent);border-color:var(--accent);background:var(--bg-card-hover);}
.ub-doc__btn:disabled{opacity:.4;cursor:default;}
.ub-doc__btn:disabled:hover{color:var(--text-secondary);border-color:var(--border-hover);background:var(--bg-card);}
.ub-doc__pageinfo{font-family:var(--font-mono);font-size:.8rem;color:var(--text-secondary);min-width:62px;text-align:center;}
.ub-doc__download{padding:0 1rem;font-family:var(--font-display);font-weight:600;font-size:.85rem;}
.ub-doc__pages{position:relative;height:85vh;min-height:560px;overflow:auto;background:var(--bg-code);padding:1.5rem 1rem;}
.ub-doc__pages::-webkit-scrollbar{width:10px;}
.ub-doc__pages::-webkit-scrollbar-track{background:var(--bg-code);}
.ub-doc__pages::-webkit-scrollbar-thumb{background:var(--border-hover);border-radius:5px;}
.ub-doc__pages::-webkit-scrollbar-thumb:hover{background:var(--accent-dim);}
.ub-doc__page{margin:0 auto 1.25rem;background:#fff;border-radius:4px;box-shadow:0 4px 20px rgba(0,0,0,.5);overflow:hidden;}
.ub-doc__page:last-child{margin-bottom:0;}
.ub-doc__page canvas{display:block;width:100%;height:auto;}
.ub-doc__loading{color:var(--text-muted);font-family:var(--font-mono);font-size:.85rem;text-align:center;padding:3rem 1rem;}
@media (max-width:600px){.ub-doc__title{font-size:.92rem;}.ub-doc__bar{gap:.6rem;}.ub-doc__pages{height:75vh;}}
</style>

<div class="ub-doc">
  <div class="ub-doc__bar">
    <span class="ub-doc__icon">
      <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/>
      </svg>
    </span>
    <div style="flex:1 1 auto;min-width:0;">
      <div class="ub-doc__title">Memoria del Trabajo de Fin de Grado</div>
      <div class="ub-doc__meta">Unburden_TFG.pdf · 95 páginas · 6,8 MB</div>
    </div>
    <div class="ub-doc__tools">
      <button class="ub-doc__btn" id="ub-zoom-out" type="button" aria-label="Reducir zoom" title="Reducir zoom">−</button>
      <span class="ub-doc__pageinfo"><span id="ub-page">1</span> / <span id="ub-total">97</span></span>
      <button class="ub-doc__btn" id="ub-zoom-in" type="button" aria-label="Aumentar zoom" title="Aumentar zoom">+</button>
      <a class="ub-doc__btn ub-doc__download" href="{{ '/projects/unburden/assets/Unburden_TFG.pdf' | relative_url }}" download>
        <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">
          <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" y1="15" x2="12" y2="3"/>
        </svg>
        Descargar
      </a>
    </div>
  </div>

  <div class="ub-doc__pages" id="ub-pages">
    <div class="ub-doc__loading" id="ub-loading">Cargando memoria…</div>
    <iframe id="ub-fallback" src="{{ '/projects/unburden/assets/Unburden_TFG.pdf' | relative_url }}#view=FitH"
            title="Memoria del TFG — Unburden" style="display:none;width:100%;height:100%;min-height:540px;border:0;"></iframe>
  </div>
</div>

<script>
(function () {
  var PDF_URL = '{{ "/projects/unburden/assets/Unburden_TFG.pdf" | relative_url }}';
  var CDN = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/';
  var pagesEl = document.getElementById('ub-pages');
  var loadingEl = document.getElementById('ub-loading');

  function useFallback() {
    if (loadingEl) loadingEl.remove();
    var f = document.getElementById('ub-fallback');
    if (f) { f.style.display = 'block'; pagesEl.style.padding = '0'; }
  }

  function loadScript(src) {
    return new Promise(function (res, rej) {
      var s = document.createElement('script');
      s.src = src; s.onload = res; s.onerror = rej;
      document.head.appendChild(s);
    });
  }

  loadScript(CDN + 'pdf.min.js').then(init).catch(useFallback);

  function init() {
    var pdfjsLib = window['pdfjs-dist/build/pdf'] || window.pdfjsLib;
    if (!pdfjsLib) return useFallback();
    pdfjsLib.GlobalWorkerOptions.workerSrc = CDN + 'pdf.worker.min.js';

    var ZOOM_STEP = 0.25, ZOOM_MIN = 0.6, ZOOM_MAX = 2.5, MAX_W = 880;
    var pdfDoc = null, base = null, zoom = 1, lastWidth = 0;
    var holders = [], rendered = {}, current = 1, io = null, building = false;
    var pageEl = document.getElementById('ub-page');
    var totalEl = document.getElementById('ub-total');
    var zi = document.getElementById('ub-zoom-in');
    var zo = document.getElementById('ub-zoom-out');

    pdfjsLib.getDocument(PDF_URL).promise.then(function (pdf) {
      pdfDoc = pdf;
      totalEl.textContent = pdf.numPages;
      if (loadingEl) loadingEl.remove();
      return pdf.getPage(1);
    }).then(function (page) {
      base = page.getViewport({ scale: 1 });
      build();
      watch();
    }).catch(useFallback);

    // Ajuste al ANCHO del visor (sin huecos laterales) multiplicado por el zoom del usuario.
    function fitScale() {
      var avail = Math.min(pagesEl.clientWidth - 32, MAX_W);
      return (avail / base.width) * zoom;
    }

    function build() {
      building = true;
      var anchor = current;
      if (io) io.disconnect();
      pagesEl.innerHTML = '';
      holders = []; rendered = {};
      lastWidth = pagesEl.clientWidth;
      var css = fitScale();
      var w = Math.round(base.width * css);
      var h = Math.round(base.height * css);
      for (var n = 1; n <= pdfDoc.numPages; n++) {
        var el = document.createElement('div');
        el.className = 'ub-doc__page';
        el.dataset.page = n;
        el.style.width = w + 'px';
        el.style.height = h + 'px';
        pagesEl.appendChild(el);
        holders.push(el);
      }
      observe();
      // Mantener la página que se estaba viendo tras un re-render.
      var t = holders[anchor - 1];
      if (t) t.scrollIntoView({ block: 'start' });
      building = false;
    }

    function observe() {
      io = new IntersectionObserver(function (entries) {
        entries.forEach(function (e) {
          if (!e.isIntersecting) return;
          var n = +e.target.dataset.page;
          render(n, e.target);
          current = n;
          if (pageEl) pageEl.textContent = n;
        });
      }, { root: pagesEl, rootMargin: '400px 0px', threshold: 0.01 });
      holders.forEach(function (el) { io.observe(el); });
    }

    function render(n, holder) {
      if (rendered[n]) return;
      rendered[n] = true;
      pdfDoc.getPage(n).then(function (page) {
        // Nitidez real: resolución del lienzo = tamaño CSS x densidad de pantalla actual.
        var dpr = Math.min(window.devicePixelRatio || 1, 3);
        var css = fitScale();
        var vp = page.getViewport({ scale: css * dpr });
        var canvas = document.createElement('canvas');
        canvas.width = Math.round(vp.width);
        canvas.height = Math.round(vp.height);
        holder.innerHTML = '';
        holder.appendChild(canvas);
        page.render({ canvasContext: canvas.getContext('2d', { alpha: false }), viewport: vp });
      }).catch(function () { rendered[n] = false; });
    }

    function setZoom(z) {
      var nz = Math.max(ZOOM_MIN, Math.min(ZOOM_MAX, z));
      if (nz === zoom) return;
      zoom = nz;
      if (zo) zo.disabled = zoom <= ZOOM_MIN + 0.001;
      if (zi) zi.disabled = zoom >= ZOOM_MAX - 0.001;
      build();
    }
    if (zi) zi.addEventListener('click', function () { setZoom(zoom + ZOOM_STEP); });
    if (zo) zo.addEventListener('click', function () { setZoom(zoom - ZOOM_STEP); });

    // Re-render fiable ante cambios de tamaño O zoom del navegador (Ctrl +/−),
    // que alteran el ancho disponible y/o la densidad de pantalla.
    function watch() {
      var t;
      function relayout() {
        if (building) return;
        if (Math.abs(pagesEl.clientWidth - lastWidth) < 2) return; // evita bucles por scrollbar
        build();
      }
      if (window.ResizeObserver) {
        new ResizeObserver(function () { clearTimeout(t); t = setTimeout(relayout, 150); })
          .observe(pagesEl);
      } else {
        window.addEventListener('resize', function () { clearTimeout(t); t = setTimeout(relayout, 150); });
      }
    }
  }
})();
</script>
