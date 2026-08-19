---
layout: page
title: Blog
permalink: /blog/
---

# Blog

Posts live here. Latest first.

<ul>
{% for post in site.posts %}
  <li>
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a> — {{ post.date | date: "%B %d, %Y at %I:%M %p %Z" }}
    <p style="margin-top:4px;color:#555">{{ post.description | default: post.excerpt }}</p>
  </li>
{% endfor %}
</ul>
