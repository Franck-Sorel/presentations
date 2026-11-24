# Presentations

A public repository that hosts each presentation in its **own directory**. Every talk is self-contained (source + its own `package.json`, styles, and a built `dist/`).

## Repository layout

```
presentations/
├── email-encryption/        # Presentation 1 · "Email Encryption, Inside Out"
│   ├── slides.md            #   Slidev source (one concept per slide)
│   ├── GO-DEEPER.md         #   companion deep-dive / blog post
│   ├── README.md            #   that deck's own readme
│   ├── styles/index.css     #   global CSS (TOC bar off, centered diagrams)
│   ├── package.json         #   Slidev + theme
│   └── dist/                #   built static export (hostable)
└── <your-next-presentation>/  # next talk goes here
```

## The first presentation

**`email-encryption/`** — "Email Encryption, Inside Out": a blog-style walkthrough of TLS vs E2EE, PGP/S-MIME internals, the metadata gap, and decentralized mail with DIDs + DIDComm. 39 slides, Mermaid diagrams, screen-fit verified.

## Adding a new presentation

1. Create a directory for it, e.g. `mkdir my-talk`.
2. Put its `slides.md` and any supporting files inside.
3. Give it its own `package.json` (a minimal one is enough):
   ```json
   { "name": "my-talk", "private": true, "type": "module",
     "scripts": { "dev": "slidev", "build": "slidev build" },
     "dependencies": { "@slidev/cli": "^0.50.0", "@slidev/theme-default": "^0.25.0" } }
   ```
4. Run and export from that directory: `npm install && npm run dev` (or `npm run build` → `dist/`).
5. Commit and push — the deck lives alongside the others.

> Each deck is independent: install, build, and serve it from its own folder.

## Hosting

Each deck's `dist/` is a static export — point GitHub Pages (or any static host) at a deck's `dist/` folder to serve it.