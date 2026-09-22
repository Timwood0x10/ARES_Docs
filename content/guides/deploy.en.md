---
title: "Deploy to GitHub Pages"
description: "Build the ARES documentation site with Hugo and deploy it to GitHub Pages."
weight: 1
---

This site is built with [Hugo](https://gohugo.io) and deployed to GitHub Pages
via GitHub Actions. There is no theme submodule — all layouts and styles live
in the repository root.

## Prerequisites

- Hugo extended 0.164.0 or later (`brew install hugo`).

## Local preview

```bash
cd ARES_Docs
hugo server --port 9090
```

Open http://localhost:9090/ARES/ in a browser.

## Production build

```bash
cd ARES_Docs
hugo --minify --baseURL "https://timwood0x10.github.io/ARES/"
```

Static files are written to `public/`.

## GitHub Pages CI

The workflow at `.github/workflows/hugo.yml` builds and deploys on pushes to
`main`, `master`, or `dev`. It installs Hugo extended from GitHub releases,
runs `actions/configure-pages@v5`, executes `hugo --minify`, then publishes
the `public/` artifact with `actions/upload-pages-artifact@v3` and
`actions/deploy-pages@v4`.

Ensure the repository Pages source is set to **GitHub Actions**
(Settings → Pages).

## Configuration

All site configuration is in `config.yaml`. Key fields:

| Field | Value |
|-------|-------|
| `baseURL` | `https://timwood0x10.github.io/ARES/` |
| `defaultContentLanguage` | `en` |
| `markup.highlight.style` | `github-dark` |
| `markup.highlight.noClasses` | `true` |
