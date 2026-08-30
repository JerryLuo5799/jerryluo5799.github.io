# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Jekyll blog (jerryluo.com / jerryluo5799.github.io) written in Chinese, mostly about .NET, Azure, Azure DevOps and AI. The theme is forked from https://github.com/oukohou — a fair amount of the template (`server/`, `tools/`, some widgets) is upstream leftovers that this site does not use.

Nearly all work in this repo is **adding or editing a post under `_posts/`**, occasionally with an image under `assets/imgs/`.

## Build & deploy

There is no Gemfile, no CI workflow, and no test suite. GitHub Pages builds `master` directly with its own Jekyll version; **pushing to `master` is the deploy**.

Local preview (needs a system Jekyll with `jekyll-paginate` and `jemoji`):

```bash
jekyll serve            # http://localhost:4000
jekyll build            # renders into _site/ (gitignored)
```

`tools/runserver.sh` and `server/` are an upstream Tornado static server hard-coded to someone else's machine — ignore them. `tools/create-post.py` emits front matter (`keywords`/`description`/`category`) that does **not** match the convention actually used below; prefer copying an existing recent post's header.

## Post conventions

File name: `_posts/YYYY-MM-DD-SomeTitle.md`. Permalink is `/:year/:month/:day/:title/`, so **renaming a published post's file or date breaks its URL and its LeanCloud read counter** (the counter is keyed on `page.url`).

Front matter every recent post uses:

```yaml
---
layout: post
title:  ".NET Aspire实战指南"
date:   2026-03-15 09:30:00 +0800--
categories: [.NET]
tags: [.NET Aspire]
---
```

- The trailing `--` after the timezone is present in all 93 posts; Jekyll tolerates it. Keep it for consistency rather than "fixing" one post.
- `categories` is effectively a single-value enum driving `/category/` and the sidebar widget. Existing values: `.NET` (most), `Azure`, `Azure DevOps`, `Tools`, `Activities`, `AI`, `Other`. Don't invent a new one casually — each new value gets its own heading on the category page.
- `tags` are free-form and render on `/tags/`.
- Body is Markdown (kramdown/GFM, Rouge highlighting). MathJax is loaded site-wide with `$...$` inline math, so **a bare `$` in prose or outside a code fence can be swallowed as math**.
- Images go in `assets/imgs/` and are referenced with a site-absolute path: `![alt](/assets/imgs/name.png)`.
- All link addresses with Microsoft-related domain names need to append the parameter: `wt.mc_id=MVP_324329`, for example: microsoft.com, github.com, etc.

## Layout & template structure

- `_layouts/base.html` is the real shell (head, nav from `site.menu`, sidebar, footer). `post.html` and `page.html` both wrap it; `keynote.html` references a nonexistent `default` layout and is dead.
- The sidebar is a fixed list of `{% include widgets/... %}` calls at the bottom of `base.html`. `widgets/star_post_list` is entirely commented out (upstream's links), and `widgets/search_box` is superseded by `search_box_new`.
- `assets/data/posts.json` is **a Liquid template**, not data — Jekyll renders it into the search index that `widgets/search_box_new` fetches client-side. Edit the template, never a generated file.
- Comments (Valine) and the read-count widget (LeanCloud) are configured in `_config.yml` under `valine:` and wired inline in `_layouts/post.html` and `index.html`. Analytics (Google Analytics / GTM) blocks in `base.html` are commented out but the GTM `<noscript>` iframe is still live.
- `_config.yml` `exclude:` keeps `server/`, `tools/`, `README.md` and `changelog.md` out of the build. Root-level oddities (`BingSiteAuth.xml`, `google*.html`, `baidu_verify_*.html`, `*.txt`) are search-engine verification files — leave them alone.
