# Free Deployment & PWA Plan

Companion to [PLAN.md](./PLAN.md). How Mess Ledger gets onto three phones at ৳0/month, and what
makes it a real installable app rather than a website. The accounts this assumes you already have
are listed in [ACCOUNTS.md](./ACCOUNTS.md).

**Plan v1 — 10 August 2026.** Free-tier numbers below were verified against Cloudflare docs on this
date. Re-check before relying on any of them.

---

## Part 1 — Free deployment

### 1.1 Verified free-tier ceilings

| Resource | Free allowance | Our realistic use |
|---|---|---|
| Worker requests | 100,000 / day (resets midnight UTC, then error 1027) | ~200 / day |
| **Worker CPU** | **10 ms per request, 10 ms per cron** | See §1.2 — this is the one that binds |
| Subrequests | 50 / request (D1, R2, KV all count) | <10 |
| Cron triggers | 5 per account | 1 (see §1.8) |
| D1 databases | 10 per account | 2 (production + preview) |
| D1 size | **500 MB per database**, 5 GB per account | <10 MB after years |
| D1 rows | 5,000,000 read/day · 100,000 written/day | ~2,000 read/day |
| D1 queries | 50 per Worker invocation | See §1.2 |
| D1 Time Travel | **7 days** on free | See §1.9 — not enough, do your own exports |
| R2 storage | 10 GB-month, Standard class only | ~5 MB/year of compressed bills |
| R2 operations | 1M Class A + 10M Class B / month | Hundreds |
| R2 egress | Free, always | — |

Headroom is three orders of magnitude on everything except CPU time.

### 1.2 The two limits that actually constrain the design

**10 ms CPU per request.** Plenty for SQL and JSON, nowhere near enough to rasterise an image — and
the cron handler gets the same 10 ms, so there is no server-side path, scheduled or on demand. This
kills the shared-link preview image from §8.3 of PLAN.md as a Worker job entirely. Instead: **the
closing device renders the PNG to a canvas at month close and uploads it to R2** like any other
attachment, and the share page references it as a static URL. `ctx.waitUntil()` is not a way out —
its 30 s is wall-clock, the CPU budget stays at 10 ms.

The same ceiling rules out bcrypt and scrypt for the PIN, which is why PLAN.md §5 stores an
HMAC under a Worker-secret pepper instead.

**50 D1 queries per Worker invocation.** A free-plan ceiling specifically — paid gets 1,000. The
sync endpoint touches eight tables, so never loop per-row: `db.batch([...])` keeps a pull at eight
statements regardless of how many rows come back. A batch still counts each statement toward the
50, so the outbox flush is the endpoint actually at risk — **cap it at 40 mutations per request**
and let the loop drain across several calls. A phone that was offline for a week will need this.

### 1.3 Shape: one Worker, not Pages + Worker

Serve the SPA and the API from a **single Worker using Static Assets**. This is the decision that
removes the most work:

- **One origin**, so the `httpOnly` session cookie from PLAN.md §5 simply works — no CORS config, no
  preflight requests, no `SameSite=None` compromise
- **One deploy** (`wrangler deploy`) and one rollback surface
- Static asset requests are served from Cloudflare's edge without invoking your Worker code, so the
  app shell costs no CPU at all

`run_worker_first` decides what the Worker handles; everything else is a file.

### 1.4 Repo layout

```
mess-ledger/
├─ src/
│  ├─ client/           # React PWA — screens, Dexie, outbox
│  ├─ worker/           # Hono app: /api/*, /s/:token, cron handler
│  └─ shared/           # Zod schemas + split engine (imported by BOTH sides)
├─ drizzle/
│  ├─ schema.ts
│  └─ migrations/
├─ public/
│  ├─ manifest.webmanifest
│  └─ icons/
├─ wrangler.jsonc
├─ vite.config.ts
└─ .github/workflows/deploy.yml
```

`src/shared/` is the important one: the split engine and its Zod schemas live there and are imported
by the client *and* the Worker. One definition of the money math, so the optimistic local result and
the server's canonical result can never disagree.

### 1.5 `wrangler.jsonc`

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "mess-ledger",
  "main": "src/worker/index.ts",
  "compatibility_date": "2026-08-10",

  "assets": {
    "directory": "./dist/client",
    "binding": "ASSETS",
    "not_found_handling": "single-page-application",
    "run_worker_first": ["/api/*", "/s/*", "/files/*"]
  },

  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "mess-ledger",
      "database_id": "<from wrangler d1 create>",
      "migrations_dir": "drizzle/migrations"
    }
  ],

  "r2_buckets": [{ "binding": "FILES", "bucket_name": "mess-ledger-files" }],

  "triggers": { "crons": ["0 19 * * *"] },
  "observability": { "enabled": true },

  "env": {
    "preview": {
      "d1_databases": [
        { "binding": "DB", "database_name": "mess-ledger-preview", "database_id": "<id>" }
      ],
      "r2_buckets": [{ "binding": "FILES", "bucket_name": "mess-ledger-files-preview" }]
    }
  }
}
```

`/s/*` **must** be in `run_worker_first` — share pages are server-rendered for their OG tags. If they
fell through to the SPA fallback, WhatsApp would get an empty shell and render no preview card.

### 1.6 Provisioning runbook

Assumes the Cloudflare account, `workers.dev` subdomain and R2 activation from
[ACCOUNTS.md](./ACCOUNTS.md) are already done. Then, once, from the project root:

```bash
npm i -D wrangler drizzle-kit vite-plugin-pwa
npx wrangler login

npx wrangler d1 create mess-ledger
npx wrangler d1 create mess-ledger-preview       # copy both database_ids into wrangler.jsonc
npx wrangler r2 bucket create mess-ledger-files
npx wrangler r2 bucket create mess-ledger-files-preview

npx wrangler secret put SESSION_SECRET           # 32 random bytes, for signing device tokens
npx wrangler secret put SHARE_SECRET             # for signing /s/ tokens and R2 URLs
```

Then per deploy:

```bash
npx drizzle-kit generate                         # SQL into drizzle/migrations
npx wrangler d1 migrations apply mess-ledger --remote
npm run build                                    # Vite → dist/client
npx wrangler deploy
```

Seed the three members once, from a local SQL file, never from a public endpoint:

```bash
npx wrangler d1 execute mess-ledger --remote --file=./seed/members.sql
```

### 1.7 Domain and HTTPS

`mess-ledger.<your-subdomain>.workers.dev` is free and served over HTTPS, which is all a PWA needs —
service workers, `navigator.share`, and installability all work there. A custom domain is cosmetic;
add it later via Cloudflare DNS if you want a nicer link in the WhatsApp group.

### 1.8 Cron: one trigger, not two

Cron triggers fire on **UTC**, and cron syntax cannot express "1st of the month in Asia/Dhaka" (that
instant is 18:00 UTC on the *previous* month's last day, which varies). So use **one daily trigger at
`0 19 * * *`** (01:00 Dhaka) and branch inside the handler on the Dhaka-local date:

| Dhaka date | Action |
|---|---|
| 1st | Create recurring expenses (rent, internet, maid) for the new period |
| 25th | Prepare the due-reminder message and mark it ready to share |
| otherwise | Return immediately |

One trigger, no timezone bugs, 4 of your 5 free triggers still spare.

### 1.9 Backups — do not rely on Time Travel

D1's point-in-time recovery is **7 days on the free plan**. A bug you notice at month-end could
easily be older than that. So:

- A GitHub Action on a weekly cron runs `wrangler d1 export mess-ledger --remote --output=dump.sql`
- The dump is uploaded to R2 under `backups/YYYY-WW.sql` (keeps 52, still kilobytes)
- Restore drill: run the import into `mess-ledger-preview` **once** so you know the path works before
  you ever need it

### 1.10 CI/CD

`.github/workflows/deploy.yml` — push to `main` runs typecheck, tests (both split invariants),
migrations, then deploy via `cloudflare/wrangler-action`. Needs two repo secrets,
`CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` — see [ACCOUNTS.md §1.6 and §2.3](./ACCOUNTS.md)
for the exact token scopes. Free for a private repo within the 2,000 Actions minutes/month; this
pipeline is under a minute.

Migrations run **before** deploy, so new code never meets an old schema.

### 1.11 Rollback and monitoring

- `npx wrangler rollback` reverts to the previous version immediately; `wrangler versions list` shows
  what's deployed
- `observability.enabled` gives request logs and errors in the dashboard without extra setup
- If the 100,000 requests/day ceiling is ever hit, the Worker returns **error 1027** until midnight
  UTC. At ~200/day the only realistic cause is a sync loop bug — so cap outbox flush retries with
  exponential backoff, and make the flush idempotent by mutation id

---

## Part 2 — Building the PWA

### 2.1 `public/manifest.webmanifest`

```json
{
  "id": "/",
  "name": "Mess Ledger",
  "short_name": "Mess",
  "description": "Rent, utilities and bazar for the house.",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "display_override": ["standalone", "minimal-ui"],
  "orientation": "portrait",
  "background_color": "#11150F",
  "theme_color": "#0B6B4F",
  "lang": "en",
  "categories": ["finance", "utilities"],
  "icons": [
    { "src": "/icons/192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ],
  "screenshots": [
    { "src": "/icons/shot-home.png", "sizes": "1080x1920", "type": "image/png", "form_factor": "narrow" }
  ],
  "shortcuts": [
    { "name": "Add expense", "url": "/add" },
    { "name": "Settle up", "url": "/settle" }
  ]
}
```

Three details that are easy to skip and visibly wrong when you do:

- **`purpose: "maskable"` needs its own artwork** with the logo inside the safe circle. Reusing the
  square icon gets it cropped on Android.
- **`screenshots` unlock the rich install dialog** on Android. Without them you get the plain
  one-line prompt.
- **`shortcuts`** put *Add expense* on the icon's long-press menu — the fastest path to the app's
  primary action.

### 2.2 iOS-specific head tags

iOS ignores parts of the manifest, so the HTML still needs:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
<link rel="apple-touch-icon" href="/icons/apple-touch-180.png" />
<meta name="theme-color" media="(prefers-color-scheme: light)" content="#F6F8F5" />
<meta name="theme-color" media="(prefers-color-scheme: dark)"  content="#11150F" />
```

`viewport-fit=cover` plus `black-translucent` means you must pad with `env(safe-area-inset-bottom)`
on the tab bar and `env(safe-area-inset-top)` on the header, or content hides under the notch and the
home indicator.

### 2.3 Service worker strategy

The rule that keeps this sane: **the service worker caches the app, Dexie caches the data.** Two
caches over the same rows is how you get stale numbers nobody can explain.

| Request | Strategy | Why |
|---|---|---|
| App shell — JS, CSS, HTML, fonts | Precache, versioned | Instant cold start, works offline |
| `/api/*` | **Never cached** | Dexie is the offline store; the SW must stay out of it |
| `/files/*` (bill photos) | CacheFirst, 300 entries / 180 days | Immutable objects, cheap to keep |
| `/s/*` (share pages) | **Never cached, never fallback** | Server-rendered per token |

```ts
// vite.config.ts
VitePWA({
  registerType: 'prompt',
  manifest: false,                       // hand-written, see 2.1
  workbox: {
    globPatterns: ['**/*.{js,css,html,svg,woff2}'],
    navigateFallback: '/index.html',
    navigateFallbackDenylist: [/^\/api\//, /^\/s\//, /^\/files\//],
    cleanupOutdatedCaches: true,
    runtimeCaching: [
      {
        urlPattern: /^\/files\//,
        handler: 'CacheFirst',
        options: {
          cacheName: 'bills',
          expiration: { maxEntries: 300, maxAgeSeconds: 60 * 60 * 24 * 180 },
          cacheableResponse: { statuses: [0, 200] }
        }
      }
    ]
  }
})
```

The `navigateFallbackDenylist` entry for `/s/` matters in practice: without it, a housemate who has
the app installed taps a share link and gets the SPA shell instead of the share page.

### 2.4 Update flow

`registerType: 'prompt'`, not `'autoUpdate'`. This app is opened, used for ten seconds, and left —
sessions are long and tabs are never closed, so a silent swap can reload the screen mid-entry. Show a
small **"New version — reload"** toast and call `skipWaiting()` only when tapped.

Never auto-reload while the outbox is non-empty.

### 2.5 Storage durability

```ts
if (navigator.storage?.persist) await navigator.storage.persist();
```

Ask on first successful login, when the grant is most likely. Two things to know:

- Safari evicts IndexedDB after ~7 days of non-use for **websites**, but **home-screen apps are
  exempt** — which makes "install it properly" a data-integrity requirement on iOS, not a nicety
- Check `navigator.storage.estimate()` on startup; if usage approaches quota, prune cached bill images
  before touching any ledger rows

### 2.6 Install UX

- **Android:** capture `beforeinstallprompt`, stash the event, show your own *Install* row in Setup
- **iOS:** there is no prompt. Detect iOS Safari in a browser tab and show a one-time card with the
  actual gesture — Share → Add to Home Screen — because on iOS installing is what protects the data
- Detect success with `matchMedia('(display-mode: standalone)')` and never nag again

### 2.7 Performance budget

| Metric | Target | How |
|---|---|---|
| Initial JS | < 100 KB gzipped | Route-level splitting; keep Dexie + Zod in the main chunk, charts lazy |
| First paint on repeat visit | < 300 ms | Precached shell, no network on the critical path |
| Any screen's data | 0 network requests | Every read is a Dexie live query |
| Fonts | 0 external requests | System stacks, or self-hosted `woff2` — CDN fonts are a blocked request and a silent fallback |
| Spinners | 0 | If a screen can show a spinner, its data should have come from Dexie |

Numbers in tables get `font-variant-numeric: tabular-nums` so columns of taka line up.

### 2.8 Offline UX rules

1. Every action succeeds immediately against Dexie — nothing is disabled for being offline
2. A small chip appears **only** when the outbox is non-empty: "2 changes waiting"
3. Flush on `visibilitychange`, on `online`, and via Background Sync where supported (Chromium);
   never a polling interval
4. Sync failures are silent and retried — a failed background sync is not an error the user caused
5. Closed-period writes rejected by the server surface once, with the server's canonical value shown

### 2.9 Acceptance checklist

Deployment:

- [ ] `wrangler deploy` serves the SPA and `/api/health` from one origin
- [ ] Login cookie survives a browser restart and a week of inactivity
- [ ] `/s/:token` returns server-rendered OG tags — verify by pasting a link into the WhatsApp group
- [ ] Daily cron fires and no-ops on a day that is neither the 1st nor the 25th Dhaka time
- [ ] Weekly backup export lands in R2, and a restore into preview has been done once
- [ ] `wrangler rollback` tested before you need it

PWA:

- [ ] Installs on Android Chrome with the rich dialog (screenshots present)
- [ ] Installs on iOS via Add to Home Screen, with correct icon and no notch clipping
- [ ] Airplane mode: app opens, all months readable, a new expense can be added and appears in the ledger
- [ ] Back online: outbox drains, other two phones see the row after sync
- [ ] Update toast appears on a new deploy and does not reload with a pending outbox
- [ ] Lighthouse PWA + Performance pass on a throttled 3G profile
- [ ] Tested on the actual three phones, not just devtools emulation

---

## Corrections to PLAN.md

Four things verified today differ from the earlier estimate and are fixed in PLAN.md:

1. **D1 free storage is 500 MB per database** (5 GB per account), not 5 GB per database. Still ~50×
   more than this app will ever need.
2. **The 10 ms CPU limit rules out preview-image generation on the Worker altogether** — cron
   handlers get the same 10 ms, so "pre-generate it on a schedule" was never an option either.
   Phase 03's shared-link image is rendered on the closing device and uploaded to R2.
3. **The same limit rules out bcrypt and scrypt**, which §5 had specified for the PIN. It now stores
   `HMAC-SHA-256(pin, salt)` under a pepper kept in a Worker secret. For a 10,000-candidate PIN
   space this costs nothing real; the lockout and the phone allowlist were always doing the work.
4. **50 D1 queries per invocation is a free-plan ceiling** (paid gets 1,000), and batched statements
   each count. §6's outbox flush is now capped at 40 mutations per request.

---

## Sources

- [Cloudflare Workers — Static assets](https://developers.cloudflare.com/workers/static-assets/)
- [Cloudflare Workers — Platform limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Cloudflare D1 — Limits](https://developers.cloudflare.com/d1/platform/limits/)
- [Cloudflare D1 — Pricing](https://developers.cloudflare.com/d1/platform/pricing/)
- [Cloudflare R2 — Pricing](https://developers.cloudflare.com/r2/pricing/)
