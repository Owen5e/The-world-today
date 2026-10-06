# The World Today

A news reader: today's top headlines on the home page, filterable categories, saved articles kept
in the browser, and a reading view per story. React 19 on Vite with a serverless function in front
of NewsAPI so the key never reaches the browser.

| | |
| --- | --- |
| **Live site** | https://my-blog-nu-blush.vercel.app |
| **Stack** | React 19 · Vite 6 · Tailwind CSS 4 (`@tailwindcss/vite`) · React Router 7 · NewsAPI |
| **Node** | 20+ |
| **Repo** | https://github.com/Owen5e/The-world-today |

## What it is

`/` fetches top headlines and renders them as cards; each card opens `/post/:id` with the full story
and actions to save or share it. `/categories` narrows the feed to general, business, technology,
sports, health, science or entertainment. `/saved` lists the articles you bookmarked — kept in
`localStorage`, so they survive a refresh and stay on your device. `/about` explains the project.
A header toggle switches the whole app between light and dark mode, also persisted locally.
`ArticleButtons` is the shared save/share control behind those actions.

## Why it exists

A reading app is the honest way to exercise the parts of React that toy counters never touch: a
real network boundary, loading and error states, cache-vs-fresh trade-offs, routing with a dynamic
segment, and persistence without a database. It also had to solve the problem every front-end
developer hits the first time they use a keyed third-party API — a key in a `VITE_` variable is
compiled into the bundle and is public, so the browser cannot be trusted with it. That is what the
`api/` function below is for.

## The news data flow

The app never calls NewsAPI from the browser in production:

| Context | Path |
| --- | --- |
| Home feed, development | `src/App.jsx` calls NewsAPI directly with `import.meta.env.VITE_NEWS_API_KEY` — convenient, and explicitly a dev-only branch |
| Home feed, production | the app calls its own `/api/news`, handled by the serverless function in `api/news.js`: it reads `NEWS_API_KEY` from the **server** environment, requests `https://newsapi.org/v2/top-headlines?country=us`, and returns the JSON with CORS headers |
| Categories page, **all builds** | `src/pages/Categories.jsx` calls NewsAPI directly with `VITE_NEWS_API_KEY` — there is no dev/production branch here, so this one still goes out from the browser (see Known gaps) |
| Netlify | `netlify/functions/news.js` is the same handler in Netlify's function signature, for a Netlify deploy |

If `NEWS_API_KEY` is missing the function answers `500 {"error":"API key not configured"}` — a
readable failure rather than an empty feed on a key that silently is not there.

## Getting started

```bash
git clone https://github.com/Owen5e/The-world-today.git
cd The-world-today
npm ci
echo "VITE_NEWS_API_KEY=your_key_from_newsapi.org" > .env.local
npm run dev           # http://localhost:5173
```

Create the key at https://newsapi.org/register — the free tier is enough for local development.
Without it the dev build has nothing to fetch, so start there.

| Script | What it does |
| --- | --- |
| `npm run dev` | Vite dev server with HMR |
| `npm run build` | Production build → `dist/` |
| `npm run preview` | Serve the built `dist/` locally |
| `npm run lint` | ESLint |

## Environment variables

| Variable | Where | Purpose |
| --- | --- | --- |
| `VITE_NEWS_API_KEY` | `.env.local`, dev only | Lets the dev server call NewsAPI directly. **Public** — Vite inlines it into the bundle |
| `NEWS_API_KEY` | host environment (Vercel/Netlify), production | Used by `api/news.js` / the Netlify function, server-side, never shipped to the client |

## Routes

Defined once in `src/routes.js`: `/` (home), `/categories`, `/saved`, `/post/:id`,
`/about`.

## Deploy

A static SPA plus one serverless function, which is exactly a Vercel project: build `npm run
build`, output `dist/`, and `api/news.js` becomes `/api/news` automatically. Set `NEWS_API_KEY` in
the project's environment variables. On Netlify, point the build at `dist/` and let
`netlify/functions/news.js` serve the same route. Either host needs unknown paths rewritten to
`/index.html` so deep links work on refresh.

## Known gaps

- **The Categories page ships the API key to the browser.** `src/pages/Categories.jsx` reads
  `import.meta.env.VITE_NEWS_API_KEY` with no dev/production branch, so in a deployed build that
  key is inlined into the bundle and every category request goes straight to NewsAPI from the
  client. It needs the same treatment the home feed already has — call `/api/news` with a
  `category` parameter. (A `VITE_` variable is public by design; the fix is not to hide it but to
  stop using it client-side.)
- `src/pages/CreatePost.jsx` exists but is **not routed** — there is no entry for it in
  `src/routes.js`, so the page is unreachable. Either route it or delete it.
- Articles are shown but never cached beyond React state; each navigation back to `/` refetches.
- `/saved` is device-local by design (no accounts), so bookmarks do not follow you between browsers.
- No tests, and no error boundary — a render error blanks the page.
- The free NewsAPI tier only serves `top-headlines` for development use; a production deploy needs
  a paid plan or a different feed.
