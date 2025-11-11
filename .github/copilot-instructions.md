**Overview**
- Hugo static blog configured via `hugo.yaml` with metadata, profile tiles, and navigation menus.
- PaperMod theme lives at `themes/PaperMod` as a submodule; keep it synced with `git submodule update --init --recursive` and override via local `layouts/` instead of editing upstream files.
- Domain configuration relies on `CNAME` (`bugaddr.tech`) aligning with `baseURL` in `hugo.yaml`.

**Content Model**
- Markdown content resides in `content/`; posts sit under `content/posts/` (see `content/posts/init.md`), pages like About live directly in `content/`.
- All pages use YAML front matter; reuse toggles such as `ShowToc`, `ShowBreadCrumbs`, and `ShowReadingTime` from `content/about.md` for consistent UX.
- `archetypes/default.md` seeds new entries; run `hugo new posts/<slug>.md` to scaffold posts with the default front matter.

**Assets**
- Static assets go in `static/`; files mirror their published path (e.g., `static/images/profile.jpg` referenced by `params.profileMode.imageUrl`).
- Add shared resources under `static/` so they survive theme updates and deploy cleanly.

**Build & Preview**
- Local preview: from repo root run `hugo server -D` to serve drafts using the YAML config.
- Production build: `.github/workflows/hugo.yaml` runs `hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"`; keep CLI flags aligned if build behavior changes.
- CI assumes the PaperMod submodule is present; ensure submodules are fetched before running Hugo locally or in new environments.

**Customization Tips**
- Update nav links via `menu.main` in `hugo.yaml`; include trailing slashes to match Hugo-generated URLs.
- Profile tiles and social buttons come from `params.profileMode` and `params.socialIcons`; replicate that structure when adding new buttons or platforms.
- Prefer adding custom templates or shortcodes under a new `layouts/` directory rather than editing `themes/PaperMod` directly to ease theme upgrades.
- To create new sections, add folders under `content/` and register them in `menu.main` or `params.profileMode.buttons` as needed.

**CI/CD**
- GitHub Pages deploys on every push to `main` through the workflow in `.github/workflows/hugo.yaml`; artifacts are published from `public/`.
- Any changes to the deployment flow should preserve the `pages` concurrency group and the two-stage build/deploy job structure.
