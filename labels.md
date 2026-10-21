---
layout: default
title: Labels
permalink: /labels/
description: Browse essays by label — one controlled vocabulary for the whole blog. Each post stores its labels in its own front matter; this page is generated from them at build time.
---

{% assign all_labels = "" | split: "," %}
{% for post in site.posts %}
  {% for label in post.labels %}
    {% unless all_labels contains label %}
      {% assign all_labels = all_labels | push: label %}
    {% endunless %}
  {% endfor %}
{% endfor %}
{% assign all_labels = all_labels | sort %}

<div style="margin-top:12px">
  <h1 class="page-title">Labels</h1>
  <p class="muted" style="margin:4px 0 18px">Every essay is filed under labels from one controlled vocabulary. Pick a label to see everything grouped under it.</p>

  {% for label in all_labels %}
    {% assign count = 0 %}
    {% for post in site.posts %}
      {% if post.labels contains label %}{% assign count = count | plus: 1 %}{% endif %}
    {% endfor %}
    <div class="labelgroup" data-surface="labels-page">
      <h2 id="{{ label }}">{{ label }} <span class="count">({{ count }})</span></h2>
      <ul class="posts">
        {% for post in site.posts %}
          {% if post.labels contains label %}
            <li class="postrow">
              <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
              <span class="date">{{ post.date | date: "%b %d, %Y" }}</span>
            </li>
          {% endif %}
        {% endfor %}
      </ul>
    </div>
  {% endfor %}
</div>
