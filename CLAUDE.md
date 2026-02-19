# prtk.in — Personal Website

Praateek Mahajan's personal site built with **Hugo** (v0.156.0), deployed to **GitHub Pages** at `prtk.in`.

## Structure

```text
hugo.toml                         # Site config — baseURL, social links, tagline, bio
assets/css/style.css              # All CSS — theme variables, layout, homepage, blog, prose
layouts/
  _default/baseof.html            # HTML shell, inline theme-init JS (default: dark)
  _default/list.html              # Blog listing (date desc, summary)
  _default/single.html            # Individual post
  partials/head.html              # <head> — meta, CSS via Hugo Pipes (minify + fingerprint)
  partials/header.html            # Nav: site name, "Notes / Blogs" link, theme toggle
  partials/footer.html            # Copyright footer
  index.html                      # Homepage: avatar, tagline, bio, social icons, blog blurb
content/
  _index.md                       # Homepage (empty frontmatter, content in template)
  blog/_index.md                  # Blog section listing
  blog/*.md                       # Blog posts
archetypes/default.md             # Template for `hugo new content blog/my-post.md`
static/CNAME                      # Custom domain: prtk.in
.github/workflows/hugo.yaml       # GH Actions: install Hugo, build --gc --minify, deploy-pages
```

## Key Decisions

- **No theme folder** — all layouts live in `layouts/` directly
- **Hugo Pipes** for CSS — `resources.Get | minify | fingerprint` gives cache busting with zero build tools
- **Dark theme default** — `localStorage` override, no flash (inline script in `<head>`)
- **Accent color** — indigo/purple via `--accent` CSS variable, used for links, icon hover, avatar ring
- **GitHub avatar by URL** — `https://github.com/praateekmahajan.png`, always current
- **Social icons** — inline SVGs using `fill: currentColor` (black/white, not brand colors)
- **RSS + sitemap enabled**, taxonomy/term disabled

## Local Dev

```bash
hugo server --bind 0.0.0.0  # dev server at localhost:1313
hugo --gc --minify          # production build to public/
```

## Adding a Blog Post

```bash
hugo new content blog/my-post.md
# Edit content/blog/my-post.md, set draft: false when ready
```
