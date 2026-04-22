---
layout: page
title: Irakli Gabisonia
description: A blog about software architecture, engineering leadership, and building practical systems.
---

<div class="home-nav">
  <a href="{{ '/posts/' | relative_url }}">All posts</a>
  <a href="{{ '/about/' | relative_url }}">About</a>
</div>

This site is now a blog instead of a resume.

I use it to write about software architecture, .NET, distributed systems, delivery practices, and the engineering decisions that usually matter more than frameworks.

## Latest posts

<div class="post-list">
{% for post in site.posts limit:5 %}
  <article class="post-card">
    <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
    <p class="post-card-meta">{{ post.date | date: "%B %-d, %Y" }}</p>
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
