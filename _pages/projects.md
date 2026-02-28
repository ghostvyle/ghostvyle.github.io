---
layout: default
title: Projects
permalink: /projects/
---

<div class="wrapper">

  <section class="hero">
    <div class="hero__tag">Projects</div>
    <h1 class="hero__title">ghostvyle<br><em>projects</em></h1>
    <p class="hero__subtitle">Herramientas de seguridad ofensiva desarrolladas y publicadas en GitHub.</p>
  </section>

  <section class="projects-grid" id="projects-grid">

    <article class="project-card">
      <div class="project-card__header">
        <div class="project-card__meta">
          <span class="project-lang project-lang--python">Python</span>
          <span class="project-stars"><svg xmlns="http://www.w3.org/2000/svg" width="13" height="13" viewBox="0 0 24 24" fill="currentColor"><path d="M12 .587l3.668 7.568 8.332 1.151-6.064 5.828 1.48 8.279L12 18.896l-7.416 4.517 1.48-8.279L0 9.306l8.332-1.151z"/></svg> <span id="stars-unburden">1</span></span>
        </div>
        <a href="https://github.com/ghostvyle/Unburden" target="_blank" rel="noopener" class="project-card__github">GitHub ↗</a>
      </div>
      <h2 class="project-card__name">Unburden</h2>
      <p class="project-card__desc">Asistente de pentesting autónomo que integra un LLM local con servidores MCP para automatizar reconocimiento, análisis y flujos de trabajo de seguridad ofensiva.</p>
      <div class="project-card__topics">
        <span class="tag">llm</span>
        <span class="tag">mcp</span>
        <span class="tag">pentesting</span>
        <span class="tag">automation</span>
        <span class="tag">osint</span>
      </div>
    </article>

    <article class="project-card">
      <div class="project-card__header">
        <div class="project-card__meta">
          <span class="project-lang project-lang--shell">Shell</span>
          <span class="project-stars"><svg xmlns="http://www.w3.org/2000/svg" width="13" height="13" viewBox="0 0 24 24" fill="currentColor"><path d="M12 .587l3.668 7.568 8.332 1.151-6.064 5.828 1.48 8.279L12 18.896l-7.416 4.517 1.48-8.279L0 9.306l8.332-1.151z"/></svg> <span id="stars-racoonx">3</span></span>
        </div>
        <a href="https://github.com/ghostvyle/RacoonX" target="_blank" rel="noopener" class="project-card__github">GitHub ↗</a>
      </div>
      <h2 class="project-card__name">RacoonX</h2>
      <p class="project-card__desc">Herramienta ofensiva en Bash para automatizar tareas de reconocimiento y auditoría: detección de hosts, escaneo de puertos, fingerprinting de servicios, fuerza bruta SSH y OSINT con Google Dorks.</p>
      <div class="project-card__topics">
        <span class="tag">bash</span>
        <span class="tag">nmap</span>
        <span class="tag">reconnaissance</span>
        <span class="tag">osint</span>
        <span class="tag">offensive-security</span>
      </div>
    </article>

  </section>

</div>

<script>
  const repos = [
    { id: 'stars-unburden', repo: 'Unburden' },
    { id: 'stars-racoonx', repo: 'RacoonX' },
  ];
  repos.forEach(({ id, repo }) => {
    fetch(`https://api.github.com/repos/ghostvyle/${repo}`)
      .then(r => r.json())
      .then(data => {
        if (data.stargazers_count !== undefined) {
          document.getElementById(id).textContent = data.stargazers_count;
        }
      })
      .catch(() => {});
  });
</script>
