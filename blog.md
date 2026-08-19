---
layout: default
title: Writing
permalink: /blog/
description: Writing by Ram Mehta on engineering leadership, platform, and product.
---

<div style="margin-top:12px">
  <h1 class="page-title">Writing</h1>
  <p class="muted" style="margin:4px 0 18px">Notes on engineering, platforms, and shipping product. Latest first.</p>

  {% if site.posts.size > 0 %}
    <ul class="bloglist">
    {% for post in site.posts %}
      <li>
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
        <span class="mono" style="margin-left:10px;color:var(--muted-2);font-size:12.5px">{{ post.date | date: "%b %d, %Y" }}</span>
        <div class="desc">{{ post.description | default: post.excerpt | strip_html | truncate: 160 }}</div>
      </li>
    {% endfor %}
    </ul>
  {% else %}
    <div class="empty">
      No posts yet. First post drops from the automated queue shortly. RSS is live at <a href="/feed.xml">/feed.xml</a>.
    </div>
  {% endif %}
</div>
