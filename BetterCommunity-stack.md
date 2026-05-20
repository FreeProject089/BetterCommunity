# BetterCommunity — Stack & Architecture (source de vérité)

> Lire `PROMPT.md` pour le périmètre fonctionnel et `theme.md` pour l'identité visuelle.

---

## 1. Stack

### Frontend / Web
- **Next.js 15** (App Router, RSC, PPR) + **React 19** + **TypeScript 5**
- **Tailwind CSS v4** (CSS-first `@theme`, container queries, no `tailwind.config.ts`)
- **GSAP 3.13** (latest as of Q4 2025 — `gsap`, `@gsap/react`, `ScrollTrigger`, `SplitText`, `Flip`)
- **Motion 12** (Framer Motion rebrandé) — micro-interactions
- **shadcn/ui** (style "new-york") + **Radix UI** + **lucide-react** + **Sonner** (toasts) + **cmdk** (palette) + **Vaul** (drawer)
- **TanStack Query v5** (server state) + **Zustand v5** (UI state) + **nuqs v2** (URL state)
- **React Hook Form v7** + **Zod v3** (forms)
- **Velite** + **MDX v3** (blog) + **shiki** (syntax highlight)

### Backend / DB
- **better-auth** (Discord OAuth, twoFactor TOTP, admin plugin) — *pas* next-auth
- **Drizzle ORM** + **postgres-js** (driver) + **PostgreSQL 16**
- **Redis (Upstash)** — cache repo.json, rate-limit, sessions volatiles
- **React Email v3** + Postal (email service layer centralisé via SMTP)
- **Stripe** (abonnement : 7€ tous les 3mois ou paiement unique à 20€ — partenaire) + **webhook Ko-fi**
- **S3-compatible** (Cloudflare R2) pour uploads de plugins / screenshots

### Bot Discord
- **discord.js v14** + **Hono 4** (API interne port 4000)

### Outils
- **Biome** (lint + format) — *pas* ESLint + Prettier
- **Vitest** (unit) + **Playwright** (E2E)
- **pnpm 9** workspaces + **Turborepo 2**
- **tsup** (build packages)

### Déploiement
- Docker multi-stage → GHCR → SSH deploy (VPS)
- Nginx reverse-proxy + Cloudflare (TLS, WAF, R2)
- GitHub Actions CI (typecheck, lint, test, build, push image)

---

## 2. Monorepo layout

```
BetterCommunity/
├── apps/
│   ├── web/                    # Next.js 15 (le site)
│   │   ├── src/
│   │   │   ├── app/            # App Router
│   │   │   ├── components/
│   │   │   ├── lib/
│   │   │   └── styles/
│   │   ├── content/blog/       # MDX (Velite source)
│   │   ├── public/
│   │   └── next.config.ts
│   └── bot/                    # Discord bot
│       └── src/
├── packages/
│   ├── db/                     # Drizzle schema + client
│   │   ├── src/schema/
│   │   └── drizzle.config.ts
│   ├── ui/                     # shadcn shared (optionnel — peut rester dans web V1)
│   ├── emails/                 # React Email templates
│   └── config/                 # tsconfig base, biome base
├── pnpm-workspace.yaml
├── turbo.json
├── biome.json
├── tsconfig.base.json
└── .env.example
```

---

## 3. Schéma Drizzle (PostgreSQL)

### 3.1 Auth (better-auth natif + extensions)

```ts
// users
id, discordId (unique), email, username, displayName, avatarUrl,
role: 'USER' | 'PARTNER' | 'ADMIN',
suspended: boolean,
twoFactorEnabled: boolean,
kofiOptInLeaderboard: boolean,
createdAt, updatedAt

// sessions, accounts, verifications, twoFactor — gérés par better-auth
```

### 3.2 Ko-fi

```ts
// kofi_donations
id, kofiMessageId (unique, idempotency), userId? (matched via email),
amount (numeric), currency, message (text?), public (bool),
type: 'Donation' | 'Subscription' | 'Shop Order',
createdAt

// kofi_goals
id, month (date, unique), targetAmount, currency, createdAt, updatedAt
```

### 3.3 Mod proposals (BMM)

```ts
// mod_proposals
id, authorId,
type: 'CREATOR' | 'COMMUNITY_SUGGESTION',
status: 'PENDING' | 'UNDER_REVIEW' | 'APPROVED' | 'REJECTED',
title, description (mdx string), modUrl?, screenshots (text[]),
targetGame, rejectionReason?,
createdAt, updatedAt

// mod_proposal_upvotes
proposalId, userId  -- pk(proposalId, userId)
```

### 3.4 Server repos (browse BMM)

```ts
// server_repos
id, ownerId,
name, slug (unique), description, logoUrl?,
repoJsonUrl (unique),
websiteUrl?, discordInviteUrl?,
isOfficial: boolean,
status: 'PENDING' | 'UNDER_REVIEW' | 'ACTIVE' | 'SUSPENDED' | 'REJECTED',
rejectionReason?,
lastRepoJsonHash?,           -- détection changement critique
stripeSubscriptionId?,        -- null si officiel
createdAt, updatedAt, lastFetchedAt?
```

### 3.5 Plugins (catalogue BMM)

```ts
// plugins
id, authorId,
slug (unique), name, description (mdx),
type: 'C_PLUGIN' | 'core_plugin',
source: 'OFFICIAL' | 'COMMUNITY',
version, compatibleBmmVersion,
assetUrl (R2 key), screenshots (text[]),
status: 'PENDING' | 'APPROVED' | 'REJECTED' | 'SUSPENDED',
rejectionReason?,
downloadsCount (int, denormalized),
viewsCount (int, denormalized),
createdAt, updatedAt

// plugin_download_events  -- pour les stats fines
id, pluginId, userId?, ipHash, country?, day (date), createdAt
```

### 3.6 Download stats (apps)

```ts
// app_download_stats   -- une ligne par (app, platform, day)
id, app: 'bmm' | 'bsm' | ...,
platform: 'windows' | 'mac' | 'linux',
day (date),
count (int),
unique pk(app, platform, day)
```

### 3.7 Partner applications (paiement Stripe)

```ts
// partner_applications
id, userId, serverRepoId,
stripeCheckoutSessionId, stripeSubscriptionId?,
status: 'AWAITING_PAYMENT' | 'PENDING_REVIEW' | 'APPROVED' | 'REJECTED' | 'REFUNDED',
adminNotes?,
createdAt, updatedAt
```

### 3.8 Blog (Velite est source — DB ne cache que les compteurs)

```ts
// blog_post_stats
slug (pk), views, lastViewedAt
```

### 3.9 Audit log

```ts
// audit_logs
id, actorId, action (text), targetType, targetId, metadata (jsonb), ip, ua, createdAt
```

---

## 4. Routes (App Router)

### Public
- `/` — homepage GSAP
- `/bmm`, `/bsm`, `/downloads` — apps + downloads
- `/blog`, `/blog/[slug]` — Velite MDX
- `/bmm/repos`, `/bmm/repos/[id]` — browse repos
- `/bmm/proposals`, `/bmm/proposals/[id]` — forum
- `/plugins`, `/plugins/[slug]` — catalogue
- `/support` — Ko-fi (goal, leaderboard)
- `/login` — Discord OAuth (better-auth)

### Authentifié
- `/dashboard` — vue user (mes propositions, mes plugins, mes dons)
- `/dashboard/plugins`, `/plugins/new`
- `/partner/apply`, `/partner/dashboard`

### Admin (`ADMIN` + 2FA)
- `/admin` — vue d'ensemble + Google Analytics embed
- `/admin/users`, `/admin/repos`, `/admin/proposals`, `/admin/plugins`, `/admin/kofi`, `/admin/blog`, `/admin/audit`

### API
- `/api/auth/[...all]` — better-auth handler
- `/api/downloads/[app]` — redirect + log
- `/api/webhooks/stripe` — raw body + `constructEvent`
- `/api/webhooks/kofi` — `timingSafeEqual` verification token
- `/api/internal/*` — appelé par le bot Discord (IP allowlist + `X-Internal-Secret`)

---

## 5. Flux clés

### 5.1 Login Discord
```
User clique "Login with Discord"
→ better-auth /api/auth/sign-in/discord
→ Discord OAuth (scope: identify email)
→ callback: fetch user, upsert users (discordId, email, username, avatarUrl)
→ session cookie httpOnly
→ redirect /dashboard
```

### 5.2 Devenir partenaire
```
1. /partner/apply form → Zod validate → insert partner_applications (AWAITING_PAYMENT)
   + insert server_repos (status PENDING, stripeSubscriptionId null)
2. Stripe Checkout Session 7€/mois → redirect Stripe
3. Webhook checkout.session.completed → update sub id, status PENDING_REVIEW
   → bot post #partner-requests embed
4. Admin valide /admin/repos :
   APPROVED → server_repos.status ACTIVE, user role PARTNER, mail "live"
   REJECTED_FIX → mail React Email avec changements demandés
   REJECTED_FINAL → status REJECTED, Stripe refund, mail
5. Cron: pour chaque repo ACTIVE, GET repoJsonUrl, hash, si change critique → SUSPENDED + mail admin
```

### 5.3 Webhook Ko-fi
```
POST /api/webhooks/kofi avec data Ko-fi (form-encoded, champ `data` JSON)
→ parse JSON, Zod schema strict
→ timingSafeEqual(verification_token, env.KOFI_VERIFICATION_TOKEN)
→ idempotency: ON CONFLICT (kofiMessageId) DO NOTHING
→ try match user via email (lowercase)
→ insert kofi_donations
→ bot post #supporters (si data.is_public)
→ update kofi_goals progress (calculé live, pas stocké)
```

### 5.4 Download app
```
GET /api/downloads/bmm?platform=windows
→ cache (5min) lookup GitHub latest release
→ insert app_download_stats (UPSERT count + 1 for today)
→ 302 vers asset GitHub
```

---

## 6. Sécurité

| Surface              | Contrôle                                                              |
|----------------------|-----------------------------------------------------------------------|
| Webhook Stripe       | raw body required + `stripe.webhooks.constructEvent`                  |
| Webhook Ko-fi        | `crypto.timingSafeEqual` sur verification_token                       |
| Admin                | middleware: `session.user.role === 'ADMIN' && twoFactorVerified`      |
| Bot ↔ web API        | IP allowlist Nginx + header `X-Internal-Secret` constant-time compare |
| Forms                | Zod strict, schemas exportés depuis `packages/db`                     |
| SQL                  | Drizzle uniquement, zéro template string                              |
| Repo.json distant    | Zod parse, taille max 5 MB, timeout 8s, AbortController                |
| Uploads R2           | présigne POST, content-type whitelist, taille max, virusscan futur    |
| Rate-limit           | Upstash sliding window, IP + userId, 30 req / 10s sur `/api/*`        |
| CSP                  | strict, GSAP via `'self'`, pas d'inline sauf nonce                    |
| Session              | httpOnly + Secure + SameSite=Lax, rotation, 30j                       |
| Logs                 | tout admin action → audit_logs                                        |

---

## 7. Variables d'environnement

```bash
# Database
DATABASE_URL=postgres://...

# Redis (Upstash)
REDIS_URL=...

# better-auth
BETTER_AUTH_SECRET=...
BETTER_AUTH_URL=https://bettercommunity.ch
DISCORD_CLIENT_ID=...
DISCORD_CLIENT_SECRET=...

# Stripe
STRIPE_SECRET_KEY=...
STRIPE_WEBHOOK_SECRET=...
STRIPE_PARTNER_PRICE_ID=...

# Ko-fi
KOFI_VERIFICATION_TOKEN=...

# Email (Postal self-host)
POSTAL_API_URL=https://mail.bettercommunity.ch
POSTAL_SMTP_HOST=mail.bettercommunity.ch
POSTAL_SMTP_PORT=587
POSTAL_USERNAME=...
POSTAL_PASSWORD=...
POSTAL_FROM_EMAIL=hello@bettercommunity.ch

# R2 / S3
S3_ENDPOINT=...
S3_BUCKET=...
S3_ACCESS_KEY_ID=...
S3_SECRET_ACCESS_KEY=...
S3_PUBLIC_URL=...

# Bot
DISCORD_BOT_TOKEN=...
DISCORD_GUILD_ID=...
INTERNAL_BOT_SECRET=...
DISCORD_CHANNEL_BLOG_UPDATES=...
DISCORD_CHANNEL_MOD_PROPOSALS=...
DISCORD_CHANNEL_PARTNER_REQUESTS=...
DISCORD_CHANNEL_SUPPORTERS=...
DISCORD_CHANNEL_DOWNLOAD_STATS=...
DISCORD_CHANNEL_REPO_STATUS=...

# Analytics
NEXT_PUBLIC_GA_MEASUREMENT_ID=G-...

# GitHub (downloads)
GITHUB_TOKEN=...                  # bump rate-limit
```

---

## 8. Modules de code clés (cartographie)

| Module                                | Rôle                                                |
|---------------------------------------|-----------------------------------------------------|
| `apps/web/src/lib/auth.ts`            | better-auth instance + helpers `getSession`         |
| `apps/web/src/lib/db.ts`              | Drizzle client (re-export `packages/db`)            |
| `apps/web/src/lib/stripe.ts`          | Stripe SDK + helpers checkout / refund              |
| `apps/web/src/lib/kofi.ts`            | parse Ko-fi payload + verify token                  |
| `apps/web/src/lib/github.ts`          | fetch latest release + cache Redis                  |
| `apps/web/src/lib/repo-json.ts`       | fetch + Zod parse `repo.json`, hash compare         |
| `apps/web/src/lib/ratelimit.ts`       | Upstash ratelimit wrappers                          |
| `apps/web/src/lib/gsap.ts`            | client-only GSAP registry (`useGSAP`, plugins)      |
| `apps/web/src/lib/audit.ts`           | `logAdminAction()` helper                           |
| `apps/web/src/components/gsap/*`      | composants animés (HeroIntro, Reveal, Marquee, …)   |
| `apps/web/src/components/ui/*`        | shadcn re-exports thémés                            |
| `apps/web/src/config/apps.ts`         | métadonnées par app (slug, repo GitHub, logo, …)    |
| `apps/bot/src/index.ts`               | bootstrap discord.js                                |
| `apps/bot/src/api/server.ts`          | Hono internal API (auth header, IP allowlist)       |
| `apps/web/src/lib/mail.ts` | abstraction unique email (React Email + Postal SMTP wrapper)   |
| `packages/db/src/schema/*.ts`         | tables Drizzle + types exportés                     |
| `packages/db/src/index.ts`            | client + re-export schema                           |
| `packages/emails/*.tsx`               | templates React Email (partner approve/reject, …)   |

---

## 9. Politique d'animation

- **GSAP** = animations **complexes** (hero, scroll timelines, SplitText, page transitions, sequences).
- **Motion 12** = **micro-interactions** composants (hover, tap, `AnimatePresence`).
- **Tailwind transitions** = simples (opacity, scale, translate sans timeline).
- Toujours `prefers-reduced-motion` respecté (helper `useReducedMotion`).

---

## 10. Déploiement (résumé)

```
git push main
→ GitHub Actions: typecheck (turbo), biome check, vitest, build images web+bot
→ push GHCR
→ SSH deploy: docker compose pull && up -d
→ Nginx reverse-proxy + Cloudflare TLS
→ Migrations Drizzle: `pnpm db:migrate` exécuté côté serveur avant up
```

Compose services :
- `web` (Next.js standalone)
- `bot` (Node 22)
- `postgres` (image officielle 16)
- `nginx` (TLS + IP allowlist sur :4000)

---

## 11. Hors V1

- Hosting de fichiers de repos partenaires (juste le browse).
- App mobile.
- Marketplace de mods payants.
- ML / recos personnalisées.
