# How to Write and Publish Blog Posts

## Quick Start

```bash
# 1. Create a new post
hugo new posts/my-new-post.md

# 2. Edit it
open content/posts/my-new-post.md

# 3. Preview locally
hugo server -D

# 4. Push to publish
git add .
git commit -m "Add post: my new post"
git push origin main
```

## Step by Step

### 1. Create a new post

From the project root (`/Users/abhijit/programming/abhijitmohanty.com/`):

```bash
hugo new posts/my-new-post.md
```

This creates `content/posts/my-new-post.md` with default frontmatter.

Or create the file manually. Either way, it goes in `content/posts/`.

### 2. Write the frontmatter

Every post starts with YAML frontmatter between `---` fences:

```markdown
---
title: "My New Post"
date: 2026-02-28
draft: false
tags: ["go", "systems"]
description: "A short summary that appears in the post list."
---
```

| Field | Required | Notes |
|-------|----------|-------|
| `title` | Yes | Post title |
| `date` | Yes | Publish date (YYYY-MM-DD). Posts are sorted by this. |
| `draft` | No | Set to `true` to hide from production. Visible locally with `hugo server -D`. |
| `tags` | No | List of tags for categorization |
| `description` | No | Shows in the blog list and meta tags |

### 3. Write the content

Standard Markdown below the frontmatter:

```markdown
---
title: "Go Error Handling Patterns"
date: 2026-03-01
draft: false
tags: ["go"]
description: "Practical patterns for handling errors in Go."
---

Start writing here. Regular markdown works.

## Subheadings

Use `##` for sections, `###` for subsections.

### Code blocks

\```go
func main() {
    fmt.Println("hello")
}
\```

### Images

Put images in `static/images/` and reference them:

![Alt text](/images/my-diagram.png)

### Links

[Link text](https://example.com)
```

### 4. Preview locally

```bash
hugo server -D
```

Open **http://localhost:1313** in your browser. The `-D` flag shows draft posts too.

The server live-reloads — save a file and the browser updates automatically.

Kill it with `Ctrl+C` when done.

### 5. Publish

```bash
git add .
git commit -m "Add post: go error handling patterns"
git push origin main
```

This triggers the pipeline:

1. **Automatic**: Builds and deploys to **https://mohantyabhijit.github.io** (staging preview)
2. **Manual**: Go to [Actions](https://github.com/mohantyabhijit/mohantyabhijit.github.io/actions), click the run, and approve the `production` deployment to push to **https://abhijitmohanty.com**

### 6. Approve for production

1. Go to https://github.com/mohantyabhijit/mohantyabhijit.github.io/actions
2. Click the latest workflow run
3. The `deploy-production` job shows "Waiting for review"
4. Click **Review deployments** > check **production** > **Approve and deploy**

## File Structure

```
content/
  posts/
    my-new-post.md        <-- your posts go here
    another-post.md
  about.md                <-- standalone pages
  projects.md
  talks.md
  sponsors.md
static/
  images/                 <-- put images here
    my-diagram.png
```

## Tips

- **File naming**: Use lowercase with hyphens: `my-post-title.md`. The filename becomes the URL slug (`/posts/my-post-title/`).
- **Drafts**: Set `draft: true` to work on a post without publishing it. It will only show locally with `hugo server -D`.
- **Future dates**: Posts with dates in the future are hidden by default. Use `hugo server -F` to preview them.
- **Images**: Place them in `static/images/` and reference with `/images/filename.png` (leading slash, no `static` in the path).
- **Kill old servers**: If `hugo server` shows a blank page, kill any old process on port 1313 first: `lsof -ti:1313 | xargs kill`.
