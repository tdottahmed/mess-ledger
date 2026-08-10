# Accounts & Signups

Companion to [PLAN.md](./PLAN.md) and [DEPLOYMENT.md](./DEPLOYMENT.md). Everything you need to
register before the first `wrangler deploy`, and — just as usefully — everything you can skip.

**Verified 10 August 2026.**

---

## The short answer

| # | Account | Cost | Card required | What it gives you |
|---|---|---|---|---|
| 1 | [Cloudflare](https://dash.cloudflare.com/sign-up) | Free | Only to switch on R2 | Workers, D1, R2, the `.workers.dev` URL |
| 2 | [GitHub](https://github.com/signup) | Free | No | Private repo, CI/CD, weekly D1 backup |

**Two accounts, ৳0/month.** One card on file, used only as a guardrail on R2 — see §1.4 if you want
to defer even that.

---

## 1. Cloudflare

The whole backend. Do these six steps in order; the last one is only needed when you wire up CI.

### 1.1 Create the account

<https://dash.cloudflare.com/sign-up>

Email and password, then confirm the verification mail. No card, no plan selection — the Workers
Free plan is the default and needs no opt-in. You do **not** need to add a domain or change any
nameservers; that prompt is for people putting Cloudflare in front of an existing site.

Turn on two-factor immediately — this account holds the household's money data:
<https://dash.cloudflare.com/profile/authentication>

### 1.2 Claim your `workers.dev` subdomain

Dashboard → **Compute (Workers) & Pages** → *Change* next to the subdomain.

Pick carefully: this becomes the app's permanent address,
`mess-ledger.<your-subdomain>.workers.dev`, and it goes in the WhatsApp group. It is free, HTTPS is
automatic, and that is all a PWA needs — service workers, `navigator.share()`, and installability
all work on it.

> Cloudflare describes `workers.dev` as "intended for personal or hobby projects" and steers
> production traffic to custom domains. Three housemates and a rent ledger is squarely inside that
> intent.

### 1.3 Note your Account ID

Dashboard → **Compute (Workers) & Pages** → right-hand sidebar. Or, once Wrangler is installed:

```bash
npx wrangler whoami
```

You need it for the GitHub secret in §2.3.

### 1.4 Switch on R2 — the one card step

<https://dash.cloudflare.com/?to=/:account/r2>

**R2 requires a payment method on file before it will activate, even on the free tier.** Workers and
D1 do not. You are not charged inside 10 GB-month / 1M Class A ops / 10M Class B ops, and this app
uses roughly 5 MB a year of compressed bill photos — but the card is a hard prerequisite for the
bucket to exist.

Two consequences worth understanding:

- R2 is the **only** service here that can produce a bill. Workers Free hard-stops at error 1027
  instead of billing you; D1 refuses writes instead of billing you. R2 bills on overage.
- So set a usage notification once: <https://dash.cloudflare.com/?to=/:account/notifications>

**If you want Phase 01 to be genuinely card-free:** skip this step. Bill photos are a Phase 02
feature. Keep the compressed WebP in IndexedDB on the device, ship the ledger, and activate R2 when
you actually start uploading. Nothing else in Phase 01 touches it.

### 1.5 D1 needs no signup

No activation step, no separate console. `npx wrangler d1 create mess-ledger` provisions it against
the same account. Same for Cron Triggers and for the observability logs — `"observability": { "enabled": true }`
in `wrangler.jsonc` is the entire setup, which is why there is no Sentry or analytics account on this list.

### 1.6 Create the CI API token

Only needed when you set up GitHub Actions. Local development uses `npx wrangler login`, which does
a browser OAuth handshake against the account you just made — no token involved.

<https://dash.cloudflare.com/profile/api-tokens> → **Create Token** → **Custom token**

| Scope | Permission | Level |
|---|---|---|
| Account | Workers Scripts | Edit |
| Account | D1 | Edit |
| Account | Workers R2 Storage | Edit |
| Account | Account Settings | Read |

Restrict *Account Resources* to your one account. There is a ready-made **Edit Cloudflare Workers**
template, but it also grants zone-level Workers Routes across every zone on the account — the four
rows above are the smaller, correct set for this project.

Copy the token once; the dashboard never shows it again. It goes straight into GitHub as a secret and
nowhere else — never into a file in the repo.

---

## 2. GitHub

Source of truth for the code, plus the two automations from DEPLOYMENT.md §1.9 and §1.10.

### 2.1 Create the account

<https://github.com/signup> — then enable 2FA at <https://github.com/settings/security>.

### 2.2 Create the repository

<https://github.com/new> — name it `mess-ledger`, **Private**.

Private matters here: the seed SQL carries three real phone numbers, and `wrangler.jsonc` carries
your D1 database IDs.

Then point this working copy at it:

```bash
git remote add origin git@github.com:<you>/mess-ledger.git
git push -u origin main
```

### 2.3 Add the two Actions secrets

Repo → **Settings** → **Secrets and variables** → **Actions** → *New repository secret*
(`https://github.com/<you>/mess-ledger/settings/secrets/actions`)

| Secret | Value |
|---|---|
| `CLOUDFLARE_API_TOKEN` | The token from §1.6 |
| `CLOUDFLARE_ACCOUNT_ID` | The ID from §1.3 |

### 2.4 The minutes budget

GitHub Free gives **2,000 Actions minutes/month on private repositories** (public repos are
unlimited). Our two workflows:

| Workflow | Frequency | Duration |
|---|---|---|
| typecheck → test → migrate → deploy | per push to `main` | <1 min |
| weekly `d1 export` backup to R2 | 4×/month | <1 min |

Call it 40 minutes a month against 2,000. Not a constraint.

---

## 3. What you do *not* need an account for

This list is longer than the one above, and each line is a signup the architecture deliberately
avoids.

| Not needed | Why not |
|---|---|
| **Apple Developer** ($99/yr) | It's a PWA. iOS installs it via Share → Add to Home Screen. No App Store, no review, no annual fee |
| **Google Play Console** ($25) | Same — Android installs it straight from Chrome |
| **WhatsApp Business / Meta Developer** | Sharing is `navigator.share()` from the device. No API touches WhatsApp — see PLAN.md §8.1 for why an API route doesn't exist for groups anyway |
| **Twilio / any SMS gateway** | Auth is phone + 4-digit PIN. No OTP, so no per-message cost to Bangladeshi numbers |
| **Supabase / Firebase / PlanetScale** | D1 replaces all of them, and doesn't pause when idle |
| **A push notification vendor** | Web Push (Phase 03) uses VAPID keys you generate yourself. The browser's push service is free and needs no account |
| **Sentry / LogRocket / Plausible** | `observability.enabled` gives request logs and errors in the Cloudflare dashboard |
| **A domain registrar** | `workers.dev` is free and HTTPS. See §4 if you want a prettier link |
| **npm** | Only needed to *publish* packages. Installing needs nothing |

---

## 4. Optional, later

Neither is needed to ship. Both are listed because they show up in the plan and it should be clear
they are opt-in.

**A custom domain** — purely cosmetic, and the only thing on this page that costs money. Cloudflare
Registrar sells at wholesale with no markup (~$10/yr for a `.com`):
<https://dash.cloudflare.com/?to=/:account/registrar>. Adding it later is a DNS record and one line
of config; nothing about the app changes.

**A Telegram bot** — free, via [@BotFather](https://t.me/BotFather). Relevant only if you ever want
*automatic* posting to a group chat, which WhatsApp cannot do legitimately (PLAN.md §8.1). You
already have Telegram or you don't; there is no separate developer account.

---

## 5. Local prerequisites — not accounts

| Tool | Get it |
|---|---|
| Node.js (LTS) | <https://nodejs.org> — Wrangler, Vite and Drizzle all run on it |
| git | Already installed |

Wrangler itself is not a global install — `npx wrangler` uses the dev dependency from §1.6 of
DEPLOYMENT.md.

---

## 6. Signup checklist

- [ ] Cloudflare account created, email verified, 2FA on
- [ ] `workers.dev` subdomain claimed — app URL is now known and can go in the group
- [ ] Account ID copied
- [ ] R2 activated with a payment method *(or consciously deferred to Phase 02)*
- [ ] R2 usage notification configured
- [ ] `npx wrangler login` succeeds locally
- [ ] GitHub account created, 2FA on
- [ ] Private `mess-ledger` repo created and `origin` pushed
- [ ] Cloudflare API token created with the four scopes, account-restricted
- [ ] `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` set as Actions secrets
- [ ] Node.js LTS installed

Next: the provisioning runbook in [DEPLOYMENT.md §1.6](./DEPLOYMENT.md).

---

## Sources

- [Cloudflare — Sign up](https://dash.cloudflare.com/sign-up)
- [Cloudflare Workers — workers.dev subdomain](https://developers.cloudflare.com/workers/configuration/routing/workers-dev/)
- [Cloudflare Workers — GitHub Actions & API token scopes](https://developers.cloudflare.com/workers/ci-cd/external-cicd/github-actions/)
- [Cloudflare R2 — Pricing and free tier](https://developers.cloudflare.com/r2/pricing/)
- [Cloudflare D1 — Limits](https://developers.cloudflare.com/d1/platform/limits/)
- [GitHub — Billing for GitHub Actions](https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions)
