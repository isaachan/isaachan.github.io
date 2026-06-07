# Blog Guide

This repository is a Jekyll-based GitHub Pages blog using the NexT theme.

## Repo Structure

- `_posts/`: published blog posts
- `_drafts/`: draft posts not published yet
- `images/`: image assets referenced by posts
- `videos/`: video assets referenced by posts
- `_config.yml`: site-wide Jekyll and theme configuration
- `.github/workflows/jekyll.yml`: GitHub Pages build and deploy workflow

## How Posts Work

Published posts live in `_posts/` and must use this filename format:

```text
YYYY-MM-DD-slug.md
```

Example:

```text
2026-06-05-my-first-post.md
```

Drafts live in `_drafts/`. They can follow the same naming style, but are only shown locally when you start Jekyll with `--drafts`.

## Post Front Matter

Each post starts with YAML front matter.

```md
---
title: My New Post
description: Short summary of the post
categories:
 - software
tags:
 - Jekyll
 - Blog
---
```

Notes:

- `layout: post` is already applied by default in `_config.yml`
- `comments: true` is also applied by default for posts
- Keep `description` short and specific
- `categories` is typically a short list such as `software` or `technology`
- `tags` should be a flat list of useful search labels

## Writing Style

Posts in this repo are standard Markdown with GitHub Flavored Markdown via `kramdown`.

You can use:

- headings with `#`, `##`, `###`
- fenced code blocks with language tags
- tables
- inline links
- images
- raw HTML when needed, such as `<video>`

## Images And Media

Store images in a post-specific folder under `images/`.

Example:

```text
images/2026-06-05-my-first-post/cover.png
```

Reference it in Markdown like this:

```md
![Cover image](/images/2026-06-05-my-first-post/cover.png)
```

Optional caption style used in this repo:

```md
![Cover image](/images/2026-06-05-my-first-post/cover.png)
*Cover image caption*
```

Videos can be placed under `videos/` and embedded with HTML:

```html
<video width="320" height="240" controls>
  <source src="/videos/example.mp4" type="video/mp4">
</video>
```

## Local Development

Install dependencies:

```bash
bundle install
```

Run the site locally:

```bash
bundle exec jekyll serve
```

Open:

```text
http://127.0.0.1:4000
```

Build the site once:

```bash
bundle exec jekyll build
```

Preview drafts:

```bash
bundle exec jekyll serve --drafts
```

## Publish Workflow

1. Create a draft in `_drafts/` or write directly in `_posts/`
2. Add front matter
3. Add images under `images/<post-folder>/`
4. Preview locally with Jekyll
5. Move the file into `_posts/` if it started as a draft
6. Commit and push to `main`

GitHub Actions will build and deploy the site automatically from `main`.

## Common Pitfall

Jekyll parses Liquid syntax inside Markdown. If a code block contains template text like:

{% raw %}
```text
{{ .System }}
{{ .Prompt }}
```
{% endraw %}

wrap that block with raw tags so Jekyll does not try to evaluate it:

```liquid
{% raw %}
```text
{{ .System }}
{{ .Prompt }}
```
{% endraw %}
```

## Starter Template

Use this when creating a new post:

```md
---
title: My New Post
description: Short summary of the post
categories:
 - software
tags:
 - Topic1
 - Topic2
---

Intro paragraph.

# Section 1

Write the main content here.

## Section 1.1

Add examples, code, screenshots, or notes.
```
