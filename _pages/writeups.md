---
layout: default
title: Writeups
permalink: /writeups/
---

<div class="wrapper">

  <section class="hero">
    <div class="hero__tag">Writeups</div>
    <h1 class="hero__title">ghostvyle<br><em>writeups</em></h1>
    <p class="hero__subtitle">Documentación completa de máquinas comprometidas en HackTheBox. Desde reconocimiento hasta escalada de privilegios.</p>
  </section>

  <div class="site-search">
    <svg class="site-search__icon" xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></svg>
    <input class="site-search__input" id="writeup-search" type="text" placeholder="Buscar máquina, OS o técnica..." autocomplete="off" spellcheck="false">
    <span class="site-search__count" id="writeup-count"></span>
  </div>

  <section class="writeups-index">

    <div class="writeup-os-section">
      <div class="writeup-os-header">
        <span class="writeup-os-icon">🐧</span>
        <h2>Linux</h2>
        {% assign linux_writeups = site.writeups | where: "os", "Linux" %}
        <span class="writeup-os-count">{{ linux_writeups.size }} máquina{% if linux_writeups.size != 1 %}s{% endif %}</span>
      </div>
      <div class="writeup-list" id="writeup-list">
        {% assign linux_writeups = site.writeups | where: "os", "Linux" %}
        {% for post in linux_writeups %}
        <a href="{{ post.url | relative_url }}" class="writeup-entry" data-title="{{ post.title | downcase }}" data-tags="{{ post.tags | join: ',' | downcase }}" data-os="linux">
          <div class="writeup-entry__left">
            <span class="writeup-entry__name">{{ post.title | remove: "HTB Writeup: " }}</span>
            <div class="writeup-entry__tags">
              <span class="badge badge--platform">{{ post.platform | default: "HackTheBox" }}</span>
              <span class="badge badge--difficulty badge--{{ post.difficulty | downcase | default: 'easy' }}">{{ post.difficulty | default: "Easy" }}</span>
              {% for tag in post.tags %}
                {% unless tag == "htb" or tag == "linux" or tag == "windows" or tag == "easy" or tag == "medium" or tag == "hard" %}
                <span class="badge badge--vuln">{{ tag }}</span>
                {% endunless %}
              {% endfor %}
            </div>
          </div>
          <span class="writeup-entry__arrow">→</span>
        </a>
        {% endfor %}
      </div>
    </div>

    <div class="writeup-os-section">
      <div class="writeup-os-header">
        <span class="writeup-os-icon">🪟</span>
        <h2>Windows</h2>
        {% assign windows_writeups = site.writeups | where: "os", "Windows" %}
        <span class="writeup-os-count">{{ windows_writeups.size }} máquina{% if windows_writeups.size != 1 %}s{% endif %}</span>
      </div>
      {% if windows_writeups.size > 0 %}
      <div class="writeup-list">
        {% for post in windows_writeups %}
        <a href="{{ post.url | relative_url }}" class="writeup-entry" data-title="{{ post.title | downcase }}" data-tags="{{ post.tags | join: ',' | downcase }}" data-os="windows">
          <div class="writeup-entry__left">
            <span class="writeup-entry__name">{{ post.title | remove: "HTB Writeup: " }}</span>
            <div class="writeup-entry__tags">
              <span class="badge badge--platform">{{ post.platform | default: "HackTheBox" }}</span>
              <span class="badge badge--difficulty badge--{{ post.difficulty | downcase | default: 'easy' }}">{{ post.difficulty | default: "Easy" }}</span>
              {% for tag in post.tags %}
                {% unless tag == "htb" or tag == "linux" or tag == "windows" or tag == "easy" or tag == "medium" or tag == "hard" %}
                <span class="badge badge--vuln">{{ tag }}</span>
                {% endunless %}
              {% endfor %}
            </div>
          </div>
          <span class="writeup-entry__arrow">→</span>
        </a>
        {% endfor %}
      </div>
      {% else %}
      <p class="writeup-empty">Las máquinas Windows están en camino.</p>
      {% endif %}
    </div>

  </section>

</div>

<script>
  const input = document.getElementById('writeup-search');
  const countEl = document.getElementById('writeup-count');
  const entries = document.querySelectorAll('.writeup-entry');

  function updateCount(visible) {
    countEl.textContent = visible < entries.length ? visible + ' resultado' + (visible !== 1 ? 's' : '') : '';
  }

  input.addEventListener('input', () => {
    const q = input.value.trim().toLowerCase();
    let visible = 0;
    entries.forEach(entry => {
      const haystack = (entry.dataset.title || '') + ' ' + (entry.dataset.tags || '') + ' ' + (entry.dataset.os || '');
      const match = !q || haystack.includes(q);
      entry.style.display = match ? '' : 'none';
      if (match) visible++;
    });
    updateCount(visible);
  });
</script>
