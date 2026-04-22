# gabisonia.github.io

Personal blog powered by Jekyll and the Cayman theme.

## Local preview

Local preview requires Ruby 3.x.

1. Install dependencies:

   ```bash
   bundle install
   ```

2. Start the local server:

   ```bash
   bundle exec jekyll serve
   ```

3. Open `http://127.0.0.1:4000`.

## Writing a new post

Create a Markdown file in `_posts` using this format:

```text
YYYY-MM-DD-post-title.md
```

Use front matter like:

```yaml
---
layout: post
title: "Post title"
date: YYYY-MM-DD 09:00:00 +0400
categories: engineering architecture
excerpt: "Short summary."
---
```
