# Hugo + PaperMod Blog (Auto Publish)

This is a simple Hugo blog setup using the PaperMod theme.

## Local development

1. Install Hugo Extended.
2. Start the dev server:

```bash
hugo server -D
```

3. Open `http://localhost:1313`.

## Theme

PaperMod is vendored in `themes/PaperMod`.

To update the theme later:

```bash
rm -rf themes/PaperMod
git clone --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

## Writing posts

Create markdown files inside `content/posts/`.

Required frontmatter:

```toml
title = "Your Post Title"
date = 2026-02-28T10:00:00+05:30
description = "A short summary"
tags = ["tag1", "tag2"]
draft = false
```

You can also use:

```toml
cover = "/images/cover.jpg"
```

## Build

```bash
hugo --minify
```

Generated site is in `public/`.

## Auto deploy with GitHub Actions

Workflow file: `.github/workflows/deploy.yml`

Set these GitHub repository secrets:

- `DO_HOST` (example: `143.110.253.218`)
- `DO_USER` (deploy user on your VPS)
- `DO_PORT` (usually `22`)
- `DO_SSH_KEY` (private key for deploy user)

Server path used by workflow:

- Releases: `/srv/blog/releases/<timestamp>`
- Current symlink: `/srv/blog/current`

You still need to configure your web server (Nginx/Caddy) to serve `/srv/blog/current`.
