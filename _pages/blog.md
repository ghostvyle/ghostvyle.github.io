---
layout: default
title: Blog
permalink: /blog/
---

<div class="wrapper">

  <section class="hero">
    <div class="hero__tag">Blog</div>
    <h1 class="hero__title">ghostvyle<br><em>blog</em></h1>
    <p class="hero__subtitle">Técnicas ofensivas, herramientas y análisis técnico.</p>
  </section>

  <div class="site-search">
    <svg class="site-search__icon" xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></svg>
    <input class="site-search__input" id="blog-search" type="text" placeholder="Buscar por título o tag..." autocomplete="off" spellcheck="false">
    <span class="site-search__count" id="search-count"></span>
  </div>

  <section class="posts" id="posts">
    {% assign all_posts = site.blog | sort: "date" | reverse %}
    {% for post in all_posts %}
      <a href="{{ post.url | relative_url }}" class="post-card" data-title="{{ post.title | downcase }}" data-tags="{{ post.tags | join: ',' | downcase }}">
        <div class="post-card__body">
          <div class="post-card__meta">
            <span class="post-card__category">{{ post.category }}</span>
            <span class="post-card__date">{{ post.date | date: "%d %b %Y" }}</span>
          </div>
          <h2 class="post-card__title">{{ post.title }}</h2>
          <p class="post-card__excerpt">{{ post.excerpt | strip_html | truncatewords: 28 }}</p>
          <div class="post-card__tags">
            {% for tag in post.tags %}
            <span class="tag">{{ tag }}</span>
            {% endfor %}
          </div>
        </div>
        <span class="post-card__arrow">→</span>
      </a>
    {% endfor %}
  </section>

</div>

<script>
  const input = document.getElementById('blog-search');
  const countEl = document.getElementById('search-count');
  const cards = document.querySelectorAll('.post-card');

  function updateCount(visible) {
    countEl.textContent = visible < cards.length ? visible + ' resultado' + (visible !== 1 ? 's' : '') : '';
  }

  input.addEventListener('input', () => {
    const q = input.value.trim().toLowerCase();
    let visible = 0;
    cards.forEach(card => {
      const haystack = (card.dataset.title || '') + ' ' + (card.dataset.tags || '');
      const match = !q || haystack.includes(q);
      card.style.display = match ? '' : 'none';
      if (match) visible++;
    });
    updateCount(visible);
  });
</script>
