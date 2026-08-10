# Mess Ledger — Implementation Plan

An offline-first PWA for three housemates sharing three rooms at different rents, splitting
utilities equally, and settling bazar by meal rate. Replaces the notebook.

**Plan v1 — 10 August 2026**
Shareable version: https://claude.ai/code/artifact/e5177273-b19e-45db-acd1-4fd749def454
Deployment and PWA runbook: [DEPLOYMENT.md](./DEPLOYMENT.md)

---

## Decisions locked

| | |
|---|---|
| Household | 3 people, 3 rooms (one each) |
| Stack | React PWA + Cloudflare (Pages / Workers / D1 / R2) |
| Auth | Phone number + 4-digit PIN, no SMS |
| Money | BDT, stored as integer paisa |
| Period | 1st to last of month, Asia/Dhaka |
| Bazar | Meal-rate split |
| Offline | Local-first, outbox sync |
| Sharing | WhatsApp share sheet + revocable link previews |
| Running cost | ৳0 / month |

---

## 1. Architecture

Local-first, because the free-tier and offline requirements both point the same way. The device
holds the truth it needs — screens never wait on a network call, so the app feels instant on a bad
connection or none at all. The cloud side exists only to move rows between three phones.

```
ON DEVICE                                      CLOUDFLARE · FREE TIER
┌──────────┐   ┌───────────────┐   ┌─────────┐   ┌──────────────┐   ┌─────────────┐
│ Screens  │<->│   Local DB    │-->│ Outbox  │-->│     API      │-->│  Database   │
│ React 19 │   │ Dexie/IndexDB │   │ queued  │   │ Worker+Hono  │   │ D1 (SQLite) │
└──────────┘   └───────────────┘   │ writes  │   └──────┬───────┘   └──────┬──────┘
                      ^            └─────────┘          │                  │
                      │                                 v                  │
                      │                          ┌──────────────┐          │
                      │                          │ R2 · bills   │          │
                      │                          └──────────────┘          │
                      └──────────────────────────────────────────────────────
                         pull everything changed since last cursor
```

Reads come straight from IndexedDB, so every screen paints on the first frame. Writes apply
optimistically, queue in the outbox, and flush when a connection appears. Sync pulls back rows
newer than the stored `updated_at` cursor.

| Layer | Choice | Why this one |
|---|---|---|
| UI | React 19 + Vite + TypeScript, Tailwind 4, shadcn/ui | Same components as `modern-admin`, minus Inertia |
| Local store | Dexie (IndexedDB) + `dexie-react-hooks` | Live queries re-render on local writes, no fetch layer |
| API | Hono on Cloudflare Workers | Routes and middleware map cleanly onto Laravel habits |
| Database | D1 + Drizzle ORM & migrations | SQLite semantics, 5 GB free, no idle pausing |
| Files | R2 + presigned `PUT` | 10 GB free, no egress charge, bytes never touch the Worker |
| Hosting | Cloudflare Pages, single `wrangler deploy` | One vendor, one dashboard, one bill of zero |
| Validation | Zod schemas shared client and server | One definition of an expense payload, not two |

**Why not Supabase:** its free projects pause after about a week of inactivity. A rent app gets
opened in bursts around the 1st, so you would routinely return to a cold, paused database. Workers
and D1 have no idle timer.

---

## 2. Data model

Eight tables, all synced. Every table carries `id` (UUIDv7, generated on the client),
`updated_at`, and `deleted_at`. Those three columns are what make offline sync tractable — nothing
is ever hard-deleted, and nothing waits on the server for an identity.

| Table | Columns that matter | Notes |
|---|---|---|
| `members` | name, phone, room_label, base_rent_paisa, pin_hash, active_from, active_to | Rent lives on the member, so a room rate change is just a new period value |
| `periods` | month (`2026-08`), status `open\|closed`, closed_at, meal_rate_paisa | Closing freezes the meal rate and locks edits |
| `expenses` | period_id, category_id, title, amount_paisa, paid_by, spent_on, split_type, note, shared_at | `split_type`: `room_rent` · `equal` · `meal` · `custom` · `exclude`. `shared_at` prevents double-posting |
| `expense_shares` | expense_id, member_id, share_paisa | Written at save time, not recomputed on read — this is the audit trail |
| `meal_entries` | period_id, member_id, on_date, count | One row per person per day, unique on the triple |
| `settlements` | from_member, to_member, amount_paisa, paid_on, method | Method: cash, bKash, Nagad, bank |
| `attachments` | expense_id, r2_key, mime, bytes, width, height | Bill photos, brochures, PDFs — uploaded separately from the row |
| `share_links` | token, target_type, target_id, created_by, expires_at, revoked_at | Public read-only pages for WhatsApp; token is random and non-enumerable |

> **Non-negotiable:** money is stored as **integer paisa** everywhere — database, API, client state.
> Floats will silently break a settlement by one paisa and nobody will find it. Format to `৳` only
> at the moment of render.

---

## 3. Split engine — one month, worked end to end

If the implementation reproduces these numbers, the engine is correct. Use this as the test fixture.

### 3.1 Rent, by room

| Room | Rent |
|---|---:|
| Room A | 6,500.00 |
| Room B | 5,500.00 |
| Room C | 4,000.00 |
| **Landlord** | **16,000.00** |

### 3.2 Shared costs, split equally

| Item | Amount | Each |
|---|---:|---:|
| Electricity | 3,600.00 | 1,200.00 |
| Gas | 1,080.00 | 360.00 |
| Water | 600.00 | 200.00 |
| Internet | 1,200.00 | 400.00 |
| Maid | 1,500.00 | 500.00 |
| **Subtotal** | **7,980.00** | **2,660.00** |

### 3.3 Bazar, split by meal rate

```
meal_rate    = total bazar ÷ total meals = 1,240,000p ÷ 224 = 5,535.71p  (৳55.36 / meal)
member_share = their meals × meal_rate, remainder by largest fraction
```

| Member | Meals | Bazar share |
|---|---:|---:|
| Room A | 78 | 4,317.86 |
| Room B | 84 | 4,650.00 |
| Room C | 62 | 3,432.14 |
| **Total** | **224** | **12,400.00** |

Exact division gives 431,785.71p / 465,000p / 343,214.29p, which floors to one paisa short of the
bazar total. The stray paisa goes to the largest fraction — Room A. Shares always sum to the amount
actually spent.

### 3.4 Who owes, who paid, who settles

| Member | Rent | Shared | Bazar | Owes | Paid | Balance |
|---|---:|---:|---:|---:|---:|---:|
| Room A | 6,500.00 | 2,660.00 | 4,317.86 | 13,477.86 | 16,000.00 | +2,522.14 |
| Room B | 5,500.00 | 2,660.00 | 4,650.00 | 12,810.00 | 7,980.00 | −4,830.00 |
| Room C | 4,000.00 | 2,660.00 | 3,432.14 | 10,092.14 | 12,400.00 | +2,307.86 |
| **Total** | **16,000.00** | **7,980.00** | **12,400.00** | **36,380.00** | **36,380.00** | **0.00** |

A positive balance means the person fronted more than their share and should receive money. With
three people the settlement is at most two transfers, so the screen says one plain sentence per
transfer rather than a grid of numbers:

```
Room B → Room A   ৳2,522.14
Room B → Room C   ৳2,307.86
```

**Two invariants, one unit test each:**

1. Balances across all members always sum to zero.
2. The shares of any single expense always sum to that expense's amount.

---

## 4. Meal & bazar module

Three people × thirty days = ninety numbers a month. If entry is tedious it stops happening in week
two and the module is dead. The interaction, not the arithmetic, is the hard part here.

- **Today-first entry** — home screen shows today's three counters with big +/− targets. One tap per
  meal, nothing to submit.
- **Defaults that fit reality** — an "everyone ate 3" button fills the day. Guest meals go on the
  host's count. Backfill fixes a missed week from a month grid.
- **Provisional rate, labelled** — the rate only becomes true at month close. Show it as
  *provisional* all month so nobody settles against a number that will move.
- **Off-meal days** — a zero count is a real answer, not missing data. Travelling for a week should
  visibly cut your bazar share.

---

## 5. Auth & session

1. **First run** — seed three members with their phone numbers. No public sign-up: an unknown number
   simply cannot get in.
2. **Set PIN** — first login on a device sets a 4-digit PIN, hashed with bcrypt or scrypt.
   Rate-limited to 5 attempts, then a 15-minute lockout keyed on the phone number.
3. **Session** — server returns an opaque 32-byte device token in an `httpOnly`, `Secure`,
   `SameSite=Lax` cookie with a 365-day life, renewed on each use. Nobody logs in twice.
4. **Re-open lock (optional)** — the PIN can also gate app open locally, verified against a hash in
   IndexedDB, so it works with no connection and no round trip.
5. **Later** — passkeys layer on top in v3 for fingerprint unlock, PIN as fallback.

**Why no SMS OTP:** there is no free SMS route to Bangladeshi numbers — Twilio and friends charge
per message, and both Supabase and Firebase make you bring your own provider. For three known
housemates, a PIN gives the same practical security at zero cost. If you want the OTP feel later, a
Telegram bot delivers codes for free.

---

## 6. Offline rules

Full conflict-free replication is overkill here. Expenses are append-only rows created by one person
at a time, so genuine conflicts are close to nonexistent. These six rules are enough:

1. **Client-generated UUIDv7 ids** — a row created offline has its final identity immediately, so
   shares and attachments can reference it before any sync.
2. **Outbox, not retries** — every mutation is appended to an outbox table with its payload, then
   applied to Dexie. A single flush loop drains it in order, idempotent by mutation id.
3. **Pull by cursor** — sync asks for rows where `updated_at > cursor` and stores the server's new
   high-water mark. No full table scans.
4. **Last write wins on `updated_at`** — ties break on member id. Document it and move on.
5. **Soft delete only** — a hard delete cannot propagate to a device that never saw the row.
6. **Closed periods are immutable** — the server rejects writes to a closed month, which also stops
   a stale phone from resurrecting settled history.

Files follow the same shape: the Blob is compressed to ~1600px WebP on device, stashed in IndexedDB,
and uploaded to R2 via presigned URL when the connection returns. The expense is usable with a local
preview the whole time.

---

## 7. Screens (mobile-first)

| Screen | What it answers | Key interaction |
|---|---|---|
| **Home** | What do I owe this month? | Your balance up top, today's meal counters, month total, recent entries |
| **Add** | — | Numeric keypad first, then category, payer, split type. Photo optional. Two taps to save |
| **Ledger** | Where did the money go? | Month switcher, category filter, grouped by day, tap for detail and bill photo |
| **Settle** | Who pays whom? | One sentence per transfer, mark paid, method picker, history below |
| **Setup** | — | Members, room rents, categories, recurring templates, close month |

Bottom tab bar with a centre **Add** button. Sheets instead of pages for anything modal. No spinner
anywhere — local reads mean there is nothing to wait for. A small offline chip appears only when the
outbox is non-empty.

---

## 8. Sharing to WhatsApp

### 8.1 The constraint that shapes this

**No official API can post to a WhatsApp group.** Meta's Cloud API sends to individual numbers only.
The unofficial libraries (`whatsapp-web.js`, Baileys) can post to groups, but they drive a real
WhatsApp Web session — needing an always-on host with Chromium, breaking both the ৳0 and the
serverless parts of this plan, and risking a ban on the number. So sharing is **user-initiated**, not
automatic. If automatic group posting is ever wanted, Telegram's official Bot API is the correct
tool, not a WhatsApp bridge.

### 8.2 Scope A — Share sheet

The user taps **Share**, the OS share sheet opens, they pick the group. `navigator.share()` from an
installed PWA carries **text + files together**, so the bill photo goes with the message.

Fallback chain, so the button is never dead:

```
navigator.share({ text, files })  →  navigator.share({ text })
                                  →  https://wa.me/?text=…
                                  →  copy to clipboard
```

Four things are shareable:

| Target | When | Contents |
|---|---|---|
| **Month summary + settle-up** | At month close | Totals by group, meal rate, who pays whom, link |
| **Day digest** | End of day, one message | Everything logged today, running month total, link |
| **Single expense** | Right after logging | Title, amount, payer, split, bill photo attached |
| **Due reminder** | Around the 25th | Pre-written rent/bill reminder with each person's amount |

Message format — WhatsApp markdown (`*bold*`) renders in-chat:

```
*Mess Ledger — August 2026*
Total ৳36,380.00

*Rent* ৳16,000.00
*Shared* ৳7,980.00 (৳2,660.00 each)
*Bazar* ৳12,400.00 · 224 meals · ৳55.36/meal (provisional)

*Settle up*
Room B → Room A  ৳2,522.14
Room B → Room C  ৳2,307.86

Details: https://…/s/x7k2m9
```

Two platform gotchas to handle explicitly:

- **iOS drops shared text when a file is attached.** Android uses it as the image caption; iOS often
  discards it. So never let the only copy of critical info live in text-only alongside an image, and
  always offer *Copy message*.
- **Stored bills are WebP, which WhatsApp treats inconsistently** (a 512×512 WebP can arrive as a
  sticker). Generate a **JPEG copy for sharing**, separate from the stored original.

### 8.3 Scope B — Link previews

Worker route `/s/:token` serves a public read-only page so the link renders as a rich card in the
chat instead of a bare URL, and anyone in the group can open the detail without logging in.

- OG / Twitter meta tags plus a **generated OG image** — month total and per-person balances
- Images inside the page come from **short-lived signed R2 URLs**; the bucket is never public
- Tokens are random and non-enumerable, default **30-day expiry**, revocable from Setup
- Room labels and amounts only — no phone numbers, no PINs, nothing that identifies a person beyond
  what the housemates already know

### 8.4 Noise discipline

Three people logging groceries daily will flood the group and get the chat muted, which kills the
feature. **Day digest and month summary are the primary actions**; single-expense sharing exists but
is secondary. `expenses.shared_at` drives an "already shared" state so nothing gets posted twice.

---

## 9. Delivery phases

### Phase 01 — Usable ledger (~2 weekends)

- [ ] Wrangler project, D1 schema, Drizzle migrations, seed the three members
- [ ] Phone + PIN login, long-lived device cookie
- [ ] Dexie schema mirroring the server, outbox table, flush loop, cursor sync
- [ ] Add expense with payer and split type — `room_rent` and `equal` only
- [ ] Home balance, ledger list, settle-up with minimal transfers
- [ ] Installable PWA, offline app shell, unit tests for both split invariants

### Phase 02 — The parts you actually asked for (~1 weekend)

- [ ] Meal entry (today counters + month backfill), provisional meal rate, `meal` split type
- [ ] R2 presigned uploads, client-side compression, offline queued files
- [ ] Recurring templates — rent, internet, maid auto-created on the 1st
- [ ] Close month: freeze rate, lock edits, roll unsettled balances forward
- [ ] Share sheet (Scope A) — message composer for all four targets, fallback chain, JPEG copy for sharing
- [ ] `share_links` table, `/s/:token` public page with OG meta, revoke from Setup

### Phase 03 — Polish

- [ ] Preview image for shared links, generated at month close and stored in R2 (not on demand — 10 ms CPU limit)
- [ ] Spend-by-category trend, per-member monthly history
- [ ] CSV and PDF export of a closed month
- [ ] Web push for rent due and new-expense notifications
- [ ] Passkey / biometric unlock, PIN as fallback

---

## 10. Free-tier budget

| Service | Free allowance | Realistic use |
|---|---|---|
Verified against Cloudflare docs on 10 Aug 2026 — see [DEPLOYMENT.md](./DEPLOYMENT.md) for the full
table and the two limits that actually constrain the design.

| Service | Free allowance | Realistic use |
|---|---|---|
| Workers | 100,000 requests/day; **10 ms CPU per request** | ~200/day across three phones |
| D1 | 500 MB per database, 5 GB per account, 5M rows read/day | Under 10 MB after years |
| R2 | 10 GB-month, 1M Class A ops, free egress | ~5 MB/year of compressed bills |
| Cron triggers | 5 per account | 1 daily trigger, branching on Dhaka date |
| **Monthly cost** | **৳0.00** | Three orders of magnitude of headroom, except CPU |

The 10 ms CPU ceiling is the only tight one: it rules out rasterising images on demand, so the shared
link's preview image is **pre-generated at month close** and stored in R2.

---

## 11. Risks and mitigations

| Trap | Mitigation |
|---|---|
| Meal entry gets abandoned in week two | Today-first counters and a fill-the-day button; treat entry friction as a bug |
| Meal rate moves after someone settles | Label it provisional until close; settlement screen warns while the month is open |
| Rounding drift on thirds | Integer paisa plus largest-remainder distribution; invariant test on every save |
| Month boundary off by a day | Store instants in UTC, derive the period from Asia/Dhaka local date — never from the device clock's offset |
| R2 objects publicly guessable | Serve through the Worker with short-lived signed URLs; no public bucket |
| A stale phone overwrites settled history | Server rejects writes to closed periods and returns the canonical row |
| Someone leaves or a rent changes | `active_from`/`active_to` on members; rent read per period, never globally |
| Group gets flooded and muted | Day digest and month summary are the primary share actions, not per-expense |
| Shared text vanishes on iOS | Never put critical info only in text alongside an image; always offer *Copy message* |
| WebP arrives as a sticker in WhatsApp | Generate a JPEG copy for sharing, separate from the stored original |
| A share link leaks outside the group | Non-enumerable tokens, 30-day expiry, revocable from Setup, room labels only |

**Sanity check:** Splitwise already handles unequal splits and settle-up for free. What it does not
do is per-room recurring rent, meal-rate bazar, bill photos tied to a month, or a locked monthly
close. Those four are the reason to build this — keep them in scope and resist rebuilding the rest
of Splitwise.

---

## Next step

Phase 01, first task: scaffold and schema.
