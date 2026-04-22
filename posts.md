---
layout: page
title: Posts
description: All blog posts.
permalink: /posts/
---

<div class="home-nav">
  <a href="{{ '/' | relative_url }}">Home</a>
  <a href="{{ '/about/' | relative_url }}">About</a>
</div>

<div class="post-list">
{% for post in site.posts %}
  <article class="post-card">
    <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
    <p class="post-card-meta">
      {{ post.date | date: "%B %-d, %Y" }}
      {% if post.categories and post.categories.size > 0 %}
        <span class="meta-separator">·</span>{{ post.categories | join: " / " }}
      {% endif %}
    </p>
    {% if post.excerpt %}
    <p>{{ post.excerpt | strip_html | truncate: 220 }}</p>
    {% endif %}
  </article>
{% endfor %}
</div>
