# Repository Guidelines

## Project Structure & Module Organization

This repository is a Hugo site published at `ismantic.github.io`.

- `content/posts/` contains authored blog posts in Markdown.
- `archetypes/default.md` defines front matter for new content.
- `hugo.toml` holds site-wide configuration.
- `assets/css/` contains processed site CSS assets.
- `themes/PaperMod/` is the PaperMod Git submodule; avoid editing vendored theme files unless intentionally maintaining a local theme change.
- `.github/workflows/hugo.yml` builds and deploys the site to GitHub Pages.
- `public/` is generated output and is ignored by Git; do not commit it.

## Build, Test, and Development Commands

Install Hugo Extended (CI currently uses Hugo `0.157.0`) and initialize the theme before developing:

```bash
git submodule update --init --recursive
hugo server
```

`hugo server` starts a live preview at `http://localhost:1313/`; add `-D` when drafts must be visible. Use `hugo new content posts/my-post.md` to create a post from the repository archetype. Before submitting changes, run the same production-style build used in CI:

```bash
hugo --gc --minify
```

This writes the rendered site to `public/` and surfaces template, front-matter, or asset-pipeline errors.

## Coding Style & Naming Conventions

Use TOML front matter (`+++`) for posts. Include `date`, `draft`, and `title`; add a stable `url` only when a custom permalink is required. Name post files with short, lowercase, hyphen-separated slugs, such as `content/posts/data-notes.md`. Keep Markdown readable, use semantic headings, and avoid raw HTML unless Hugo Markdown cannot express the layout. Match the existing four-space indentation in nested `hugo.toml` settings. No formatter or linter is configured, so keep diffs focused and follow neighboring files.

## Testing Guidelines

There is no automated test suite or coverage target. A successful `hugo --gc --minify` build is the required baseline. Also preview changed pages locally and check headings, links, images, dates, and mobile layout. Confirm draft posts remain `draft = true` until publication.

## Commit & Pull Request Guidelines

Recent history uses brief, imperative, topic-specific subjects such as `Update book.md` and `Fix sime post publish date`. Follow that pattern and keep each commit scoped to one logical change. Pull requests should explain the content or configuration change, identify affected URLs, and note the local build result. Link relevant issues and include before/after screenshots for visible layout or theme changes. Do not include generated `public/` files or unrelated theme-submodule updates.
