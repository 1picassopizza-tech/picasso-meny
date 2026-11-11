# Picasso Meny

Digital menu screens for Picasso Pizzeria & Grill – https://picasso-meny.web.app

- `app.html` – the whole menu site: TV screens (`/pizza`, `/mat`, `/bar`, `/tips`, `/veckanstips`, `/afterwork`) and the admin page (`/`).
  Screens load this file from GitHub and pick up a new version by themselves within a few minutes.
- `loader.html` – the small page that runs on https://picasso-meny.web.app and loads `app.html`.
- `CLAUDE.md` – how everything fits together (read this before changing code).

Prices, dishes, texts and offer photos are edited in the admin page, not here.
