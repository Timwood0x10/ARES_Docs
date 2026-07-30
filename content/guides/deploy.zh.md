---
title: "部署到 GitHub Pages"
description: "使用 Hugo 构建并将 ARES 文档站点部署到 GitHub Pages。"
weight: 1
---

本站使用 [Hugo](https://gohugo.io) 构建，通过 GitHub Actions 部署到 GitHub Pages。
无需主题子模块 —— 所有布局与样式直接放在 `Ares_docs/` 下。

## 前置条件

- Hugo extended 0.164.0 或更高版本（`brew install hugo`）。

## 本地预览

```bash
cd Ares_docs
hugo server --port 9090
```

在浏览器中打开 http://localhost:9090/ARES/。

## 生产构建

```bash
cd Ares_docs
hugo --minify --baseURL "https://timwood0x10.github.io/ARES/"
```

生成的静态文件输出到 `public/`。

## GitHub Pages CI

`.github/workflows/hugo.yml` 中的工作流在推送到 `main`、`master` 或 `dev`
分支时自动构建并部署。它使用 `peaceiris/actions-hugo@v3` 安装 Hugo，运行
`hugo --minify`，并通过 `actions/deploy-pages@v4` 部署 `public/` 产物。

确保仓库的 Pages 来源设置为 **GitHub Actions**（Settings → Pages）。

## 配置

所有站点配置在 `config.yaml` 中。关键字段：

| 字段 | 值 |
|-------|-------|
| `baseURL` | `https://timwood0x10.github.io/ARES/` |
| `defaultContentLanguage` | `en` |
| `markup.highlight.style` | `github-dark`（高对比度代码块） |
| `markup.highlight.noClasses` | `true`（颜色内联，无需额外 CSS） |
