---
layout: page
title: Irakli Gabisonia
description: A blog about software architecture, engineering leadership, and building practical systems.
---

<div class="home-nav">
  <a href="{{ '/posts/' | relative_url }}">All posts</a>
  <a href="{{ '/dotnet/' | relative_url }}">.NET</a>
  <a href="{{ '/ai/' | relative_url }}">AI</a>
  <a href="{{ '/dsa/' | relative_url }}">DSA</a>
  <a href="{{ '/notes/' | relative_url }}">Notes</a>
  <a href="{{ '/about/' | relative_url }}">About</a>
</div>

## Latest posts

<div class="post-list">
{% for post in site.posts limit:5 %}
  <article class="post-card">
    <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
    <p class="post-card-meta">{{ post.date | date: "%B %-d, %Y" }}</p>
    {% include category-tags.html categories=post.categories %}
    {% if post.excerpt %}
    <p>{{ post.excerpt | strip_html | truncate: 180 }}</p>
    {% endif %}
  </article>
{% endfor %}
</div>

## What to expect

- Architecture notes from real projects
- Delivery and quality practices that scale with teams
- Lessons from debugging, migrations, and system redesigns
- Opinions about tradeoffs, not just tools
