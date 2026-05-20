# BetterCommunity — Brief produit

> Hub communautaire central de l'écosystème **Better\*** (BMM, BSM, et apps futures).
> URL cible : `https://bettercommunity.ch`
> Thème visuel : voir `theme.md` — palette jaune doré (`#f4b63d`) sur fond charbon (`#0f1115`), glassmorphism léger, Inter + JetBrains Mono.
> Animations : **GSAP** pour scènes complexes (hero, scroll, page transitions), Motion 12 pour micro-interactions, Tailwind transitions pour le reste.

---

## 1. Vision

Une plateforme unique qui :
- Présente proprement chaque app de l'écosystème (BMM, BSM, …) avec ses téléchargements officiels.
- Centralise la communauté Discord (auth, contributeurs Ko-fi, supporters).
- Fait vivre les apps (blog d'updates multi-projet, propositions de mods, catalogue de plugins).
- Permet aux serveurs repo tiers de devenir **partenaires officiels** (paiement : 7€ tous les 3 mois ou 20€ en paiement unique), validation manuelle, vitrine dans le browser BMM).
- Donne à l'admin tous les leviers (utilisateurs, repos, plugins, propositions, analytics).

Look & feel : *workspace premium*, ambiance Linear / Arc Browser / Discord, le jaune doré sert d'accent et jamais de couleur dominante agressive.

---

## 2. Auth

- **Connexion Discord uniquement** (OAuth) — on récupère email, pseudo, avatar, ID Discord.
- **2FA TOTP obligatoire pour l'admin**, optionnel pour les autres.
- Rôles : `USER`, `PARTNER`, `ADMIN`.
- Champ `suspended` pour bloquer un utilisateur sans le supprimer.
- Session côté cookie httpOnly + Secure, refresh long.

---

## 3. Apps de l'écosystème (BMM, BSM, …)

Chaque app a :
- Une page de présentation dédiée (`/bmm`, `/bsm`, …) : hero animé GSAP, features, screenshots, vidéos (`assets/videos/`), specs techniques, lien Reddit/Discord.
- Un **bouton download** qui sert de fallback officiel : redirige vers la GitHub release la plus récente *et* logue le download dans `download_stats` (compteur journalier par app + plateforme).
- Une section blog filtrée sur l'app.

Le site se généralise facilement à de nouvelles apps via une config (`apps.config.ts`) — pas de hard-code par app pour les éléments répétitifs.

---

## 4. Ko-fi — soutien BMM

- **Objectif mensuel** de don affiché en grand sur la homepage et sur `/support` (barre de progression GSAP).
- **Leaderboard** : top donateurs du mois (pseudo + montant cumulé) — opt-in pour l'affichage du pseudo.
- **Liste contributeurs** complète (paginated), avec badge spécial si donateur récurrent.
- Webhook Ko-fi sécurisé (vérification du `verification_token` en `timingSafeEqual`), idempotent (clé sur `message_id` Ko-fi).
- L'admin règle le montant cible du mois (CRUD `kofi_goals`).

---

## 5. Downloads BMM / BSM

- Route serveur `/api/downloads/[app]` :
  1. Lit la dernière release GitHub via API (cached 5 min Redis ou ISR Next).
  2. Insert dans `download_stats` (app, platform, country via header CF, day).
  3. `302` vers l'asset GitHub.
- Page `/downloads` agrège toutes les apps + un tableau des stats publiques (totaux par app, courbe 30j).

---

## 6. Blog BetterCommunity

- **MDX via Velite** (build-time, type-safe).
- Chaque post a un champ `app` (`bmm` | `bsm` | `general`) → permet de filtrer.
- Page `/blog` : liste paginée, filtres par app via **nuqs** (URL synced).
- Page `/blog/[slug]` : MDX rendu, table des matières sticky, composants custom (callout, code block JetBrains Mono, image GSAP reveal).
- Quand un post est publié, le bot Discord poste une embed dans `#blog-updates`.

---

## 7. BMM — Propositions de mods

Forum dédié `/bmm/proposals` :
- Deux **types** :
  - `CREATOR` — l'auteur du mod veut le faire entrer dans le repo officiel BMM.
  - `COMMUNITY_SUGGESTION` — un user suggère un mod qu'il aimerait voir ajouté.
- Champs : titre, description (MDX léger), lien mod, screenshots, jeu cible, type, auteur (Discord).
- **Status** : `PENDING` → `UNDER_REVIEW` → `APPROVED` / `REJECTED` (avec raison).
- Upvotes utilisateurs (1 par user, anti-spam).
- Bot Discord poste les nouvelles propositions dans `#mod-proposals`.
- Admin gère depuis `/admin/proposals`.

---

## 8. Serveurs Repo (officiels + partenaires)

Page `/bmm/repos` :
- Liste tous les **serveurs repo** (officiels + partenaires actifs).
- Badge `OFFICIAL` ou `PARTNER`.
- Filtres : jeu, status, recherche.
- Page détail `/bmm/repos/[id]` :
  - Fetch côté serveur du `repo.json` distant (cache 10 min, validation Zod stricte).
  - Liste des mods, descriptions, versions.
  - Liens site officiel + Discord du partenaire.
  - Bouton "Ajouter à BMM" (deep-link `bmm://add-repo?url=…`).

### Devenir partenaire — 7€ tous les 3 mois ou 20€ en paiement unique

`/partner/apply` :
1. Formulaire : nom du repo, `repo.json` URL, site, Discord d'invite, description, logo.
2. **Stripe Checkout** subscription 7€/mois.
3. Au paiement réussi → status `PENDING` côté DB, embed Discord dans `#partner-requests`.
4. **Validation manuelle 24–48h (ou +) ** par l'admin :
   - **Approuvé** → status `ACTIVE`, visible dans le browse, mail "ton repo est en ligne".
   - **Rejeté avec corrections** → mail détaillé des changements demandés (React Email + Resend), status reste `PENDING`, l'user peut renvoyer.
   - **Refusé définitivement** → status `REJECTED`, remboursement Stripe automatique.
5. **Auto-suspend** si changements critiques détectés dans `repo.json` (hash diff sur les URLs de DL des mods) → status `SUSPENDED`, mail admin, re-validation requise.

`/partner/dashboard` :
- Liste des repos du partenaire, status en temps réel, bouton "Modifier" (relance validation), bouton "Désactiver" (annule sub Stripe).
- Stats : vues du repo, clics "Ajouter à BMM".

---

## 9. Admin

`/admin` (gated **role ADMIN + 2FA TOTP activée**, refusé sinon — pas seulement caché) :
- **Utilisateurs** : recherche, voir profil Discord, changer rôle, suspendre/réactiver, voir leurs server repos.
- **Server Repos** : queue de validation, approuver/rejeter avec commentaire, voir historique des modifs, forcer re-fetch du `repo.json`.
- **Propositions de mods** : modérer (status, raison).
- **Plugins** : valider / rejeter / suspendre un plugin uploadé par user.
- **Ko-fi** : régler l'objectif du mois, voir les dons en détail.
- **Blog** : créer / éditer / publier des posts MDX (live preview).
- **Analytics** : embed **Google Analytics** (GA4 measurement ID configurable), + nos propres compteurs (downloads, vues blog, vues repos).
- **Audit log** : toutes les actions admin loguées (qui, quoi, quand, IP).

---

## 10. Catalogue de plugins BMM (C_PLUGIN, core_plugin)

`/plugins` :
- Liste catalogue officiel (`source = OFFICIAL`) + catalogue user (`source = COMMUNITY`).
- Filtres : type (`C_PLUGIN` / `core_plugin`), version BMM compatible, recherche.
- Page détail `/plugins/[slug]` : description MDX, screenshots, changelog, **download** (avec compteur), **vues** (compteur sur la page), lien repo source.
- **Upload user** : `/plugins/new` (auth requise) → submit avec asset (S3-compatible), status `PENDING` → validation admin.
- **Dashboard user** `/dashboard/plugins` : voir mes plugins, leurs status, **stats downloads/vues**, éditer / retirer.

---

## 11. Discord Bot

App séparée `apps/bot` (discord.js v14) + **Hono internal API** côté Next.js (port 4000, IP allowlist Nginx + header `X-Internal-Secret`).

Le bot poste dans :
- `#blog-updates` — chaque nouveau post.
- `#mod-proposals` — chaque nouvelle proposition + change de status.
- `#partner-requests` — nouvelle demande + validation/refus.
- `#supporters` — chaque don Ko-fi (avec pseudo si opt-in).
- `#download-stats` — résumé quotidien (cron 09:00 UTC).
- `#repo-status` — chaque auto-suspend de repo partenaire.

---

## 12. Contraintes techniques transverses

- **Sécurité** :
  - Zod sur toutes les entrées externes (forms, webhooks, repo.json distant).
  - Webhook Stripe : raw body + `constructEvent`.
  - Webhook Ko-fi : `timingSafeEqual` sur le token.
  - Drizzle uniquement, aucune chaîne SQL crafted.
  - Rate-limit (upstash) sur les endpoints publics (`/api/*`).
  - Headers stricts (CSP, HSTS, X-Frame-Options).
- **Perf** :
  - Server Components par défaut, RSC streaming.
  - Images via `next/image` + AVIF.
  - GSAP côté client uniquement, lazy via dynamic import.
- **i18n** : FR / EN (next-intl), structure prête même si on lance avec FR seul.
- **Accessibilité** : focus visible jaune doré sur fond sombre, ARIA sur les composants Radix, contrastes WCAG AA minimum.

---

## 13. Hors scope (V1)

- Hosting des fichiers de repos (juste le browse / vitrine).
- Marketplace payante de mods.
- App mobile.
- Multi-langue au-delà de FR/EN.

---

## 14. Assets disponibles (déjà dans le repo)

- `assets/BC.webp` — logo BetterCommunity
- `assets/BMm.png`, `assets/BetterMM.png` — logos BMM
- `assets/Tasky/Tasky*.png` — mascotte (à utiliser sur les 404, vides, callouts)
- `assets/userpfp/*` — avatars de fallback / démo
- `assets/Other/kofi_symbol.svg`, `reddit-svg.svg`, `rust-logo.svg`, `ED.webp`
- `assets/videos/credits_bg.mp4` — vidéo loop de fond pour section credits

Repos référencés (`repo/repolist.txt`) :
- `BetterDCS/Better_ModManager_ServerBrowse`
- `BetterDCS/BMM_Contributors`
- `FreeProject089/BetterModsManager` (branche `Tdev`)
