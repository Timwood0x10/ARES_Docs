# ARES Docs

[![Deploy Docs to GitHub Pages](https://github.com/Timwood0x10/ARES_Docs/actions/workflows/hugo.yml/badge.svg)](https://github.com/Timwood0x10/ARES_Docs/actions/workflows/hugo.yml)

**ARES Docs** is the official documentation site for [ARES](https://github.com/Timwood0x10/ARES), a Go-native, multi-agent runtime with an adaptive knowledge graph. The site is built with Hugo, deployed to GitHub Pages, and available in both English and Chinese.

## Quick Start

### Prerequisites

- [Hugo extended](https://gohugo.io/installation/) v0.164.0 or later

```bash
# macOS
brew install hugo

# Verify
hugo version
# → hugo v0.164.0+extended ...
```

### Local Preview

```bash
git clone https://github.com/Timwood0x10/ARES_Docs.git
cd ARES_Docs
hugo server --port 9090
```

Open http://localhost:9090/ARES/ in your browser.

Hugo supports live reload — editing Markdown files refreshes the browser automatically.

### Production Build

```bash
hugo --minify
```

Static files are output to the `public/` directory.

## Project Structure

```
ARES_Docs/
├── content/                  # Documentation content (Markdown)
│   ├── _index.en.md          # Home page (English)
│   ├── _index.zh.md          # Home page (Chinese)
│   ├── architecture/         # System architecture docs
│   ├── modules/              # Per-package module reference
│   │   ├── core.en.md / core.zh.md
│   │   ├── llm.en.md / llm.zh.md
│   │   ├── ares_runtime.en.md / ares_runtime.zh.md
│   │   ├── ares_observability.en.md / ares_observability.zh.md
│   │   └── ...               # 30+ modules total
│   └── guides/               # Usage guides
│       ├── deploy.en.md / deploy.zh.md
│       └── extend.en.md / extend.zh.md
├── layouts/                  # Hugo templates
│   ├── _default/             # Base layouts (baseof, single, list)
│   ├── partials/             # Partial templates (head, header, sidebar, footer)
│   ├── shortcodes/           # Custom shortcodes (maturity badge)
│   └── index.html            # Home page template
├── assets/
│   └── css/
│       └── main.css          # Stylesheet (light/dark theme)
├── i18n/
│   ├── en.yaml               # English translations
│   └── zh.yaml               # Chinese translations
├── config.yaml               # Hugo configuration
└── .github/workflows/
    └── hugo.yml              # GitHub Pages deployment workflow
```

## Documentation

| Section | Description |
|---------|-------------|
| **Architecture** | Layered system architecture, request flow, and module collaboration |
| **Modules** | Per-package reference: responsibility, Mermaid architecture diagram, exported interfaces, key types/methods, extension points, maturity annotation |
| **Guides** | Deployment walkthrough, extension guide (LLM providers, custom tools, knowledge stores, strategy sources) |

Each module page includes:
- **Mermaid diagram** — visual map of internal component relationships
- **External interfaces** — exported Go function signatures
- **Key types & methods table** — type/method to purpose mapping
- **Extension points** — pluggable interfaces and implementation guidance
- **Maturity badge** — Production / Beta / Experimental

## Tech Stack

- **Static site generator**: [Hugo](https://gohugo.io) (extended, v0.164+)
- **Diagram rendering**: [Mermaid](https://mermaid.js.org) (v11, client-side)
- **Syntax highlighting**: Hugo Chroma (github-dark style)
- **Deployment**: GitHub Pages + GitHub Actions
- **Languages**: English (source), Chinese (translation), config-driven switching

## Maintenance

### Adding a New Module

1. Create `content/modules/{name}.en.md` and `content/modules/{name}.zh.md`
2. Set front matter: `title`, `description`, `weight`, `maturity`
3. Follow the page structure: Responsibility → Architecture (mermaid) → External interfaces (Go signatures) → Key types → Module collaboration → Extension points → Maturity
4. Register in `content/modules/_index.{en,zh}.md` under `listed_packages` if applicable

### Maturity Annotations

```
{{< maturity "Production" >}}
{{< maturity "Beta" >}}
{{< maturity "Experimental" >}}
```

## CI / CD

The GitHub Actions workflow at `.github/workflows/hugo.yml`:

1. Installs Hugo extended directly from GitHub releases (avoids the deprecated third-party action)
2. `actions/configure-pages@v5` configures the Pages environment
3. `hugo --minify` builds the site
4. `actions/upload-pages-artifact@v3` uploads the build artifact
5. `actions/deploy-pages@v4` deploys to GitHub Pages

> **Note**: Before deployment, ensure the repository's Pages source is set to **GitHub Actions** in Settings → Pages.

## Related Repositories

- [ARES](https://github.com/Timwood0x10/ARES) — Go-native multi-agent runtime (main project)
- [ARES_Docs](https://github.com/Timwood0x10/ARES_Docs) — This documentation site

## License

The documentation content is licensed under the same terms as the ARES main project. See the [ARES repository](https://github.com/Timwood0x10/ARES) for details.
