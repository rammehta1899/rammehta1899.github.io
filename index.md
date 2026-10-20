---
layout: home
title: Home
description: Ram Mehta — Engineering Leader exploring interesting problems
---

{% assign profile = site.data.profile %}

<section class="hero">
  <div>
    <div class="kicker">{{ profile.role }}</div>
    <h1>{{ profile.first_name }}<br><em>{{ profile.last_name }}</em></h1>
    <p class="bio">{{ profile.bio_html }}</p>
    <div class="cta-row">
      <a class="btn" href="{{ '/blog/' | relative_url }}">Read blog</a>
      <div class="socials">
        {% for s in profile.socials %}<a href="{{ s.url }}">{{ s.label }}</a>{% endfor %}
      </div>
    </div>
    <div class="status"><i></i> {{ profile.status }}</div>
  </div>
  <div class="photo-frame">
    <img src="{{ profile.photo | relative_url }}" alt="{{ profile.name }}">
  </div>
</section>

{% assign pinned = site.posts | where: "featured", true | first %}
{% if pinned %}
  {% assign featured = pinned %}
{% else %}
  {% assign featured = site.posts.first %}
{% endif %}

<section class="featured">
  <div class="sec-label">
    <span class="l">Featured essay</span>
    <span class="r">From {{ site.posts.size }} essays on platforms, distributed systems &amp; applied AI</span>
  </div>
  <h2><a href="{{ featured.url | relative_url }}">{{ featured.title }}</a></h2>
  <p class="excerpt">{{ featured.description | default: featured.excerpt | strip_html }}</p>
  <p class="byline"><b>{{ site.author }}</b> &nbsp;·&nbsp; {{ featured.date | date: "%b %d, %Y" }}{% if featured.categories.size > 0 %} &nbsp;·&nbsp; {{ featured.categories | join: ", " }}{% endif %} &nbsp;·&nbsp; {{ featured.content | number_of_words | divided_by: 200 | plus: 1 }} min read &nbsp;&nbsp;
    <a class="readmore" href="{{ featured.url | relative_url }}">Read the essay</a></p>
</section>

<section class="latest">
  <h3 class="sec">Latest writing</h3>
  <div class="grid">
    {% assign shown = 0 %}
    {% for post in site.posts %}
      {% if post.url != featured.url and shown < 5 %}
        <article class="post">
          <div class="meta">{{ post.date | date: "%b %d, %Y" }}{% if post.categories.size > 0 %} · {{ post.categories.first }}{% endif %}</div>
          <h4><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h4>
          <p>{{ post.description | default: post.excerpt | strip_html | truncate: 170 }}</p>
          {% if post.tags.size > 0 %}<div class="tags">{{ post.tags | slice: 0, 4 | join: " · " }}</div>{% endif %}
        </article>
        {% assign shown = shown | plus: 1 %}
      {% endif %}
    {% endfor %}
    <article class="post all">
      <a href="{{ '/blog/' | relative_url }}">Browse all {{ site.posts.size }} essays →</a>
    </article>
  </div>
</section>

{% if site.data.popular.posts.size > 0 %}
<section class="popular">
  <div class="sec-label">
    <span class="l">Most read</span>
    <span class="r">Ranked by analytics · last {{ site.data.popular.window_days }} days</span>
  </div>
  <h3 class="sec">Popular essays</h3>
  <div class="pop-list">
    {% for entry in site.data.popular.posts limit: 6 %}
      {% assign p = site.posts | where: "url", entry.path | first %}
      {% if p %}
        <div class="poprow">
          <span class="rank">{{ forloop.index }}</span>
          <a href="{{ p.url | relative_url }}">{{ p.title }}</a>
          <span class="views">{{ entry.views }} views</span>
        </div>
      {% endif %}
    {% endfor %}
  </div>
</section>
{% endif %}

<section class="topics">
  {% for topic in site.data.topics %}
    <div class="topic">
      <div class="t-k">{{ topic.kicker }}</div>
      <h5>{{ topic.title }}</h5>
      <p>{{ topic.text }}</p>
    </div>
  {% endfor %}
</section>
