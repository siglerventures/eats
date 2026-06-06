# CLAUDE.md — Philinity Eats

Guidance for working in this repo. Read this before making changes.

## What this is

**Philinity Eats** is a single-page restaurant / dining tracker. Users log
restaurants, tastings, and recipes, rate them, attach photos, track visit
history, and query an AI assistant. Shared data — everyone signed in sees the
same list; roles gate *who can edit*, not *who can see*.

- Live site: https://siglerventures.github.io/eats/ (also Firebase Hosting,
  project `philinity-893d2`).
- The whole app is **one file: `index.html`** (~5,600 lines). HTML, CSS
  (`<style>`), and JS (`<script type="module">`) are all inline. There is no
  build step, no bundler, no tests, no `package.json`.

## Architecture

| Concern | Implementation |
|---|---|
| Data | Firebase Realtime Database, node `eats` |
| Photos | Firebase Storage (data URLs uploaded, downloadURLs stored) |
| Auth | Firebase Auth — Google sign-in **and** email/password |
| AI assistant | Cloud Function `askAI` (`us-central1-philinity-893d2.cloudfunctions.net/askAI`) — **not in this repo** |
| Hosting | Firebase Hosting + GitHub Pages |

`index.html` structure (approximate line ranges drift as the file changes):

- `<style>` — all CSS, design tokens as CSS variables (`--amber`, `--surface`, …)
- HTML — auth gate, header, search/AI/filter bars, main grid, add/edit modal,
  detail modal, toast
- `<script type="module">` — Firebase SDK v10.8.0 imports, then all app logic,
  grouped by concern (auth/roles, DB/storage helpers, filters, rendering,
  AI commands, categories, visit history, import/export, junk-category repair).

### Data model

The `eats` node is a **flat map of entries keyed by slug** — `eats/{entryId}` —
with ONE special sibling child `eats/categories` (a slug→name object). Every
other child of `eats` is an entry. An entry looks like:

```
{ id, name, city, state, category, cuisine, rating, status, server, notes,
  mains/appetizers/desserts/drinks/phil_picks (lists), photos, imageUrl,
  visits, lastUpdated, … }
```

The client loads the **whole `eats` node at once** (`onValue(ref(db,'eats'))`).
Restore writes the whole node (`dbSet('eats', data)`); import/edit/rating write
per-field (`eats/{id}/…`).

### Access model (roles)

Roles: **admin / moderator / user** (see `EATS-rollout-spec.md` for the full
design).

- **admin**: sees Data modal (Backup / CSV / Restore) + Manage Access roster.
- **moderator**: can edit entries and categories; no Data modal / roster.
- **user**: can add / edit / rate entries; cannot edit categories.

Resolution (`resolveRole`): two hardcoded `ROOT_ADMIN_UIDS` are always admin
(lockout insurance); everyone else gets their role from the `eats_access/{emailKey}`
roster (a **sibling** of `eats`, not a child — a child would leak in the
wholesale read). `emailKey` = email lowercased with `.` → `_`.

> ⚠️ **Client-side role checks (`isAdmin()`, `applyRoleUI()`) are UI only.**
> The real enforcement is the **Realtime Database security rules**, which are
> **NOT in this repo** — they live in the `autoflag-installs` repo under
> `access-model/` (one shared rules document for all Philinity apps). Read
> `RULES-COORDINATION.md` and `ADOPTED-STANDARD.md` there before changing
> anything access-related. Changing role logic here without the matching rule
> change does nothing (or breaks access).

## Versioning — single source of truth

The rev lives in ONE place: `<meta name="app-rev" content="X.YZ">` (near the top
of `index.html`). A bootstrap script reads it, force-reloads with a `?v=<rev>`
cache-bust on change, and injects `Rev X.YZ` into every `[data-app-rev]` element
(login screen + footer).

**When you make a user-facing change, bump the `app-rev` meta value.** That's the
only edit needed — do not hardcode the rev anywhere else.

## Deploying

Automated via GitHub Actions:

- **Merge to `main`** → `firebase-hosting-merge.yml` deploys to the live channel.
- **Open a PR** → `firebase-hosting-pull-request.yml` deploys a preview channel.

So the workflow is: branch → edit `index.html` (bump rev) → PR (get a preview) →
merge (goes live). No local build/deploy needed.

## Conventions & gotchas

- **Inline event handlers.** Interactive elements use `onclick="fn()"`, so every
  handler must be re-exported at the bottom: `window.fn = fn;`. If you add a new
  handler and it errors with "fn is not defined", you forgot the export.
- **Hosting ignores** (`firebase.json`): `*.json`, `*.bak`, `index_eats_rev*.html`
  are not deployed. `.js`/`.css` get 1-year immutable cache; HTML is no-cache.
- **Security:** the Firebase `apiKey` in `index.html` is public by design (not a
  secret) — security depends entirely on the DB rules (see above). User input is
  sanitized (`escapeHTML`, `sanitizeValue`) and image URLs are allowlisted
  (`isSafeImageUrl`); keep that up when touching rendering.
- **No backup files in git.** Old revisions live in git history, not as
  `*.bak` / `index_eats_rev*.html` files (these are now `.gitignore`d).

## Files

- `index.html` — the app (the only file you'll usually edit).
- `404.html` — fallback page.
- `firebase.json` / `.firebaserc` — hosting config + project id.
- `EATS-rollout-spec.md` — design doc for the role-based access rollout.
- `.github/workflows/` — Firebase deploy actions (auto-generated, don't hand-edit).
