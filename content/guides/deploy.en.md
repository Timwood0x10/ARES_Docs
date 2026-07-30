---
title: "Deploy to GitHub Pages"
description: "Build and deploy the ARES documentation site to GitHub Pages with Hugo."
weight: 1
---

This site is built with [Hugo](https://gohugo.io) and deployed to GitHub Pages
via GitHub Actions. No theme submodule is needed — all layouts and styles live
directly under `Ares_docs/`.

## Prerequisites

- Hugo extended 0.164.0 or later (`brew install hugo`).

## Local preview

```bash
cd Ares_docs
hugo server --port 9090
```

Open http://localhost:9090/ARES/ in your browser.

## Production build

```bash
cd Ares_docs
hugo --minify --baseURL "https://timwood0x10.github.io/ARES/"
```

The generated static files land in `public/`.

## GitHub Pages CI

The workflow at `.github/workflows/hugo.yml` builds and deploys automatically on
push to `main`, `master`, or `dev`. It uses
`peaceiris/actions-hugo@v3` to install Hugo, runs `hugo --minify`, and deploys
the `public/` artifact via `actions/deploy-pages@v4`.

Make sure the repository Pages source is set to **GitHub Actions** in
Settings → Pages.

## Configuration

All site configuration lives in `config.yaml`. The key fields:

| Field | Value |
|-------|-------|
| `baseURL` | `https://timwood0x10.github.io/ARES/` |
| `defaultContentLanguage` | `en` |
| `markup.highlight.style` | `github-dark` (high-contrast code blocks) |
| `markup.highlight.noClasses` | `true` (colors baked in, no extra CSS needed) |
