# Picasso Meny – notes for Claude

Digital menu boards for **Picasso Pizzeria och Grill** (Sweden). Fire TV sticks in the restaurant show web pages from
https://picasso-meny.web.app. The owner edits prices and dishes himself in the admin page. Code changes go through this repo.

Talk to the owner in the language he writes in (often Arabic). Menu text on the screens is Swedish.

## How it fits together

| Part | Where | Notes |
|---|---|---|
| `app.html` | this repo, `main` branch | The whole site in one file: all TV screens + the admin page. **A push to `main` is live**: every screen fetches it from `raw.githubusercontent.com` (cache max-age 300 s) and reloads itself within about 2–7 minutes. |
| `loader.html` | deployed as `index.html` on Firebase Hosting | Fetches the newest `app.html` from GitHub (falls back to the last copy in localStorage, then `/app.html` on Hosting) and `document.write`s it. Rarely changes; changing it needs a Firebase Hosting deploy. |
| Menu data | Firestore, project `picasso-meny`, document `menu/main` → `{menu, updatedAt}` | Read by everyone, written only by the admin account (uid `rXotLcntYJatDzIyKvslTdERgBk1`). |
| Offer photos | Firestore `menu/img_tips`, `menu/img_aw` → `{data: "data:image/jpeg;base64,…", at}` | Kept out of `menu/main` so the menu stays small (1 MB document limit). |
| Backups | Firestore `menu/backup_*` | Copies of `menu/main` taken before big data changes. |
| TV app | Not in this repo; on the restaurant Mac in `~/Documents/picasso-tv` (Amazon Vega / React Native WebView, package `com.picasso.meny`) | Opens `/tv`. The Back button calls `window.tvBack()` (screen chooser in `app.html`). Install scripts are in `~/Documents/picasso-meny` on that Mac (`Ny-stick.command`, `Uppdatera-stick.command`). Changing the app means reinstalling on every stick over USB, so prefer web changes. |

Firestore rules: `match /menu/{docId}` → `allow read: if true; allow write: if request.auth.uid == 'rXotLcntYJatDzIyKvslTdERgBk1'`. Everything else closed.

## Routes (path = screen key)

`/` admin · `/pizza` · `/mat` · `/bar` · `/tips` (Veckans tips + Afterwork) · `/veckanstips` · `/afterwork` · `/tv` (opens the screen saved in localStorage `tvscreen`; `/tv?d=bar&pick` saves a choice) · `/info` (screen size test).

## Data model (`state`, saved as `menu/main.menu`)

```
{ v:2,
  screens: { pizza|mat|bar|tips|veckanstips|afterwork: {theme, bg, video, power, outline, dim, dsize, split} },
  sections: { <sid>: { title, note, footer, numbered, labels:null|[col1,col2], items:[
      { id:'i123', no:'12', name, desc, price, price2, state:'on'|'slut'|'hidden' } ] } },
  promos: { tips|aw: { on, title, name, desc, price, when, img (example photo URL), imgOff } } }
```

- `\n` in note/footer/labels/desc/when = new line (the admin uses ` / ` for it).
- Prices are text; `fmtP` turns digits into `129:-` (`22/cl` → `22:-/cl`).
- `labels` set = two price columns (e.g. `Glas/Flaska`, `4 cl/6 cl`, `Endast burgare/Med pommes`).
- Numbered sections (`pizza`, `special`, `inbakad`) are numbered 1…N in list order across the sections by `renumber()`; typing a number in the admin moves the pizza there (`moveToNumber`). The pizza screen sorts by number.

## Where things are in `app.html`

- `SCREENS` – each screen: `order` (sections in the admin tab) and `cols` (which sections go in which column). Pizza uses `cols3`/`flow`. `promo:true` screens render offers instead (`only:'tips'|'aw'` = one offer).
- `SCREEN_KEYS` – list of all screens (admin cards, routing, TV chooser).
- `normalize()` – fills in anything an older saved menu lacks (new screens, sections, promos). **Add new sections/screens here** so the live data keeps working without a migration.
- `defaults()` / `barSections()` / `promoDefaults()` – the starting menu ("Återställ originalmenyn").
- `renderBoard()` → `fit()` – lays out a 1920×1080 board and scales text (`--k`) so every column fits; per-column grow on mat/bar/pizza.
- `renderTips()` – offer screens. `shrinkImg()` + `onPromoImg()` – photo upload.
- Admin: `cardHTML`/`rowHTML` (section editors), `promoHTML` (offers), `onEdit`, `onClick`, `queueSave` → `flush()` (writes the whole state to `menu/main`).
- `window.tvBack` – TV screen chooser (arrows + OK, closes after 8 s).
- `checkVersion` – screens compare a SHA-256 of the GitHub file with the running one and reload when it changed.

## Working rules

1. Small, careful changes. Keep `app.html` a single self-contained file.
2. Before pushing, render the changed screen(s) at 1920×1080 in a headless browser (Playwright) and look at the result; check the admin still loads. A broken push breaks every TV within minutes.
3. Never change `menu/main` from code without a backup copy to `menu/backup_<date>` first. Content changes (prices, dishes, texts) belong in the admin page, done by the owner, not in code.
4. New screen = add to `SCREENS`, `SCREEN_KEYS`, a `.scard` block in the admin HTML (copy an existing one, rename ids), `normalize()` defaults, and the TV chooser buttons.
5. Commit to `main` with a clear message. Tell the owner the change shows on the TVs in about 2–7 minutes (or after restarting the app).
6. No passwords or login details in the repo or in chat.
