---
layout: default
title: Home
---

<link rel="stylesheet" href="/assets/css/custom.css">
<div class="wrap">
  <nav class="topnav">
    <div class="brand">Ram Mehta</div>
    <div class="navlinks">
      <a href="/blog/">Writing</a>
      <a href="https://github.com/rammehta1899">GitHub</a>
      <a href="mailto:ram.mehta.1899@gmail.com">Email</a>
    </div>
  </nav>

  <div class="hero">
    <img src="/assets/profile.jpg" alt="Ram Mehta" />
    <div>
      <div class="kicker"><i></i> Available for interesting problems</div>
      <h1>Ram Mehta</h1>
      <p class="role">Engineering Leader</p>
      <p class="bio">I’m Ram — I like hard problems, clean systems, and shipping. I’ve spent time leading product & platform teams where reliable foundations and a little practical AI actually helps people.</p>
      <div class="ctas">
        <a class="btn primary" href="/blog/">Read writing →</a>
        <a class="btn" href="https://github.com/rammehta1899">GitHub</a>
        <a class="btn" href="https://www.linkedin.com/in/rammehta1899/">LinkedIn</a>
      </div>
    </div>
  </div>

  <div class="section">
    <h2>Now</h2>
    <div class="grid3">
      <div class="card">
        <h3>Systems & Platform</h3>
        <p>Reliable foundations, thoughtful APIs, and tooling that makes teams faster without the ceremony.</p>
      </div>
      <div class="card">
        <h3>Product Engineering</h3>
        <p>UI that feels good to use — fast, accessible, maintainable. I like the craft part.</p>
      </div>
      <div class="card">
        <h3>AI in Practice</h3>
        <p>Grounded models with citations and measurement, only where they earn their keep.</p>
      </div>
    </div>
  </div>

  <div class="section">
    <h2>Writing</h2>
    {% if site.posts.size > 0 %}
      <ul class="posts">
      {% for post in site.posts limit:7 %}
        <li class="postrow">
          <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
          <span class="date">{{ post.date | date: "%b %d, %Y" }}</span>
        </li>
      {% endfor %}
      </ul>
      <div style="margin-top:12px"><a class="muted" href="/blog/">All posts →</a></div>
    {% else %}
      <div class="empty">No posts yet — I deleted the placeholder. The next real post will show here once it’s published from the queue.</div>
    {% endif %}
  </div>

  <div class="footer">
    <span>© {{ "now" | date: "%Y" }} Ram Mehta</span>
    <span style="display:flex;gap:12px">
      <a href="/blog/">/blog</a>
      <a href="/feed.xml">RSS</a>
      <a href="/sitemap.xml">Sitemap</a>
    </span>
  </div>
</div>
