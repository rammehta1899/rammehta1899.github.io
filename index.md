---
layout: default
title: Home
---

<link rel="stylesheet" href="/assets/css/custom.css">
<div class="hero">
  <div class="hero-inner">
    <img src="/assets/profile.jpg" alt="Ram Mehta">
    <div>
      <div class="kicker">● Available for interesting problems</div>
      <h1 class="hero-title">Ram Mehta</h1>
      <p class="hero-sub">Engineering Director — I like hard problems, clean systems, and shipping. I build where product, platform, and a little AI/ML overlap.</p>
      <div class="hero-cta">
        <a class="btn primary" href="/blog/">Read the blog →</a>
        <a class="btn" href="https://github.com/rammehta1899">GitHub</a>
        <a class="btn" href="mailto:ram.mehta.1899@gmail.com">Email</a>
      </div>
    </div>
  </div>
</div>

<div class="section">
  <div class="grid">
    <div class="card">
      <h3>Systems & Platform</h3>
      <p>Reliable foundations, thoughtful APIs, and tooling that makes teams faster.</p>
    </div>
    <div class="card">
      <h3>Product Engineering</h3>
      <p>UI engineering with taste — fast, accessible, and easy to maintain.</p>
    </div>
    <div class="card">
      <h3>AI / ML in Practice</h3>
      <p>Using grounded models where they actually help, with citations and measurement.</p>
    </div>
  </div>

  <div class="blog-wrap">
    <h2 style="margin:0 0 8px; letter-spacing:-0.01em">Latest notes</h2>
    <p class="muted" style="margin-top:0">Short posts from the blog live at <a href="/blog/">/blog</a></p>
    {% for post in site.posts limit:5 %}
    <div class="post-row">
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <span class="muted">{{ post.date | date: "%b %d, %Y" }}</span>
    </div>
    {% endfor %}
    {% if site.posts.size == 0 %}
      <p class="muted">No posts yet — check back soon.</p>
    {% endif %}
  </div>
</div>
