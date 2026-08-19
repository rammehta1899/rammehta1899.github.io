---
layout: default
title: Home
description: Ram Mehta — Engineering Leader exploring interesting problems
---

<div class="hero">
  <img src="/assets/profile.jpg" alt="Ram Mehta" />
  <div>
    <div class="kicker"><i></i> Available for interesting problems</div>
    <h1>Ram Mehta</h1>
    <p class="role">Engineering Leader · Systems & Product</p>
    <p class="bio">
      I build reliable platforms and product engineering teams that ship. 
      Most of my work is about <strong>foundations that scale</strong>, thoughtful APIs, and practical AI that actually helps people — not demos.
    </p>
    <div class="ctas">
      <a class="btn primary" href="/blog/">Read writing →</a>
      <a class="btn" href="mailto:ram.mehta.1899@gmail.com">Email</a>
      <a class="btn" href="https://github.com/rammehta1899">GitHub</a>
    </div>
    <div class="meta-row">
      <span class="pill">Platform & Infrastructure</span>
      <span class="pill">Product Engineering</span>
      <span class="pill">Grounded AI</span>
      <span class="pill mono">Potomac, MD · Remote</span>
    </div>
  </div>
</div>

<div class="section">
  <h2>Focus</h2>
  <div class="grid3">
    <div class="card">
      <h3>Platform as product</h3>
      <p>Internal platforms that feel like a good product: clear contracts, fast to adopt, boring to run.</p>
    </div>
    <div class="card">
      <h3>Product craft</h3>
      <p>UI engineering where performance, accessibility, and maintainability aren’t afterthoughts.</p>
    </div>
    <div class="card">
      <h3>AI where it fits</h3>
      <p>Search-grounded, citation-first, measured. If it doesn’t earn its place, it doesn’t ship.</p>
    </div>
  </div>
</div>

<div class="section">
  <h2>Selected work</h2>
  <div class="posts" style="margin-top:8px">
    <div class="postrow">
      <div>
        <div style="font-weight:620;font-size:14.2px">Bird-feeder recognition pipeline</div>
        <div class="muted" style="font-size:13px">YOLO → BioCLIP tracklet voting → zero-noise dataset · RTX 4090</div>
      </div>
      <span class="mono" style="color:var(--muted-2)">2026</span>
    </div>
    <div class="postrow">
      <div>
        <div style="font-weight:620;font-size:14.2px">Local LLM desktop build</div>
        <div class="muted" style="font-size:13px">Qwen2.5-Coder, live price-tracked, local-first</div>
      </div>
      <span class="mono" style="color:var(--muted-2)">2026</span>
    </div>
    <div class="postrow">
      <div>
        <div style="font-weight:620;font-size:14.2px">Home / auto automation</div>
        <div class="muted" style="font-size:13px">Reliable glue code > ceremony</div>
      </div>
      <span class="mono" style="color:var(--muted-2)">—</span>
    </div>
  </div>
</div>

<div class="section">
  <h2>Writing</h2>
  {% if site.posts.size > 0 %}
    <ul class="posts">
    {% for post in site.posts limit:6 %}
      <li class="postrow">
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
        <span class="date">{{ post.date | date: "%b %d, %Y" }}</span>
      </li>
    {% endfor %}
    </ul>
    <div style="margin-top:14px"><a class="muted" href="/blog/">All posts →</a></div>
  {% else %}
    <div class="empty">
      No posts published yet. The first real post lands here once the queue runs — currently wired to GitHub Actions at 10:30 AM PT.
    </div>
  {% endif %}
</div>
