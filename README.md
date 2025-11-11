# myblog

This is my personal portfolio & blog. Built with Hugo and the PaperMod theme, the site publishes content from Markdown and serves a fast, static portfolio and blog at bugaddr.tech.

## Quick overview

- Static site generator: Hugo (extended recommended)
- Theme: PaperMod (kept as a git submodule in `themes/PaperMod`)
- Content: Markdown files under `content/` (posts live in `content/posts/`)
- Custom domain: `bugaddr.tech` (CNAME present in repo)

## Getting started (local development)

Prerequisites:

- Hugo (extended) installed: <https://gohugo.io/getting-started/installing/>
- Git

Clone and initialize submodules:
  
  ```bash
  git clone https://github.com/Bugaddr/myblog.git
  cd myblog
  git submodule update --init --recursive
  ```

Run local dev server with drafts:
  hugo server -D

Create a new post scaffold:
  hugo new posts/my-new-post.md

Notes:

- Keep `baseURL` in `hugo.yaml` set appropriately for local vs production builds.
- Images and static files placed in `static/` are served from the site root.

## Build & deploy

Production build (example used by CI):

  `hugo --minify --baseURL "https://bugaddr.tech/"`

This repository is configured to publish the generated `public/` directory to GitHub Pages via a GitHub Actions workflow in `.github/workflows/`. Ensure CI fetches git submodules before building.

## Theme & customization

- PaperMod is included as a git submodule at `themes/PaperMod`. Update with:
  git submodule update --remote --merge --recursive

- Prefer adding custom templates, shortcodes, and partials under `layouts/` to avoid editing theme files directly.

- Common configurable params live in `hugo.yaml` (e.g., `menu.main`, `params.profileMode`, social icons, and toggles like `ShowToc`, `ShowReadingTime`).

## Content guidelines

- Posts: `content/posts/YYYY-MM-DD-title.md` (use `hugo new` to scaffold with front matter).
- Pages: `content/_index.md`, `content/about.md`, etc., use YAML front matter.
- Use shortcodes for consistent embeds and formatting.

## Contributing

Contributions are welcome:

1. Fork or create a branch from `main`.
2. Run the site locally and verify changes with `hugo server -D`.
3. Open a pull request with a clear description of changes.

Please include preview screenshots or a link to a deploy preview when updating UI or layout.

## License

This project is open source. Add your chosen license file (e.g., `LICENSE` with MIT text) at the repo root.

## Contact

Maintainer: Bugaddr — <https://github.com/Bugaddr>
