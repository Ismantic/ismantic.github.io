# Isma 是语

Personal site built with [Hugo](https://gohugo.io/) and the `PaperMod` theme.

Site URL: <https://ismantic.github.io/>

## Local Development

Requirements:

- Hugo extended
- Git submodules

Run locally:

```bash
git submodule update --init --recursive
hugo server
```

Default local preview:

- <http://localhost:1313/>

## Content Structure

- `content/posts/`: blog posts
- `themes/PaperMod/`: theme submodule
- `.github/workflows/hugo.yml`: GitHub Pages deployment workflow
- `public/`: generated site output

## Deployment

This repository is deployed through GitHub Pages with `GitHub Actions` as the source.

On each push to `main`, the workflow:

```bash
hugo --minify
```

builds the site and publishes the `public/` directory to GitHub Pages.
