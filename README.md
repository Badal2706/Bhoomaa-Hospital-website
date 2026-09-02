# BHOOMAA Medicare Hospital & I.C.U. — Website

Static website for BHOOMAA Medicare Hospital & I.C.U., Palanpur, Banaskantha, Gujarat.

No build step, no framework, no backend. Plain HTML/CSS/JS.

## Structure

```
public/     Web root — this is the only folder the server serves
  index.html
  robots.txt
  sitemap.xml
  site.webmanifest
  assets/           images, icons (WebP + JPEG/PNG fallbacks)
docs/       Launch checklist and deployment guide
source-photos/  Full-resolution originals (gitignored, local only)
```

## Local preview

Open `public/index.html` in a browser, or:

```bash
cd public && python3 -m http.server 8080
```

## Deployment

Nginx `root` must point at `public/`, not the repository root. See
`docs/DEPLOY-DIGITALOCEAN.md`.

## Editing

`public/index.html` is the single source of truth — HTML, CSS and JS are all
inline, with English/Gujarati/Hindi copy in the `translations` object near the
bottom. Images live in `public/assets/`.

If you replace an image, give it a new filename — assets are cached for a year.
