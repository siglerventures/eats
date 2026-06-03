# Eats rollout — spec & kickoff (for the session that hardens Eats access)

> **Where the shared governance lives:** the Firebase rules are ONE document for
> ALL Philinity apps, and they (plus the access-model standards) live in the
> **`autoflag-installs`** repo under `access-model/` —
> `RULES-COORDINATION.md`, `ADOPTED-STANDARD.md`, `SIGNIN-STANDARD.md`, and the
> deployable `PIECE3-rules-to-deploy.json`. **Read `RULES-COORDINATION.md` and
> `ADOPTED-STANDARD.md` there before touching rules.** Start from the CURRENT live
> rules, change ONLY the `eats` block (plus the new `eats_access` roster key, see
> below), and don't run two app rollouts in parallel.
>
> This file is the EATS-specific plan; it lives in the eats repo so it sits with
> the app it describes.
>
> **Use email-key-in-rules** (the adopted standard; matches AutoFlag's live rules
> and Veritas). Do **NOT** use the syncAccess / UID-mirror design (`not-adopted/`
> in autoflag-installs) — it was never deployed.

Eats ("Philinity Eats") lives at `siglerventures/eats` (GitHub Pages →
https://siglerventures.github.io/eats/). Same Firebase project as AutoFlag
(`philinity-893d2`, same `databaseURL`).

## What Eats is (from its source)
- Single `index.html`, **Google-only** sign-in. A restaurant/dining tracker.
- Currently gated by a hardcoded client array (same as Taskboard's old gate):
  `ALLOWED_UIDS = ['CuJigrzCbRfFFsM7uqe0mix46xH3', 'g4stabGPKiWAE23NTw6oUsqawWf1']`
  (checked in both `signInWithGoogle` and `onAuthStateChanged`). Rules are
  **root-UID only**.
- **Data model (matters for the rules):** the `eats` node is a **flat map of
  restaurant entries keyed by slug** — `eats/{entryId}` — with ONE special
  sibling child `eats/categories` (a slug→name object). An entry =
  `{ id, name, city, state, category, cuisine, rating, status, server, notes,
  mains/appetizers/desserts/drinks/phil_picks (lists), photos, imageUrl,
  lastUpdated, … }`. `categories` is already special-cased by the client; every
  OTHER child of `eats` is treated as a restaurant entry.
- The client loads the **whole `eats` node wholesale** via
  `onValue(ref(db,'eats'))` / `dbGet('eats')`, and the Data modal **Restore**
  writes the whole node (`dbSet('eats', data)`). Import writes per-entry
  (`update` of `eats/{id}`). Rating/edit writes per-field (`eats/{id}/…`).
- Shared data, **no per-user isolation** (everyone sees all restaurants — like
  Veritas, not like AutoFlag tenants).
- **Version is HARD-CODED in two spots** (login screen `Rev 11.12` and footer
  `Rev 11.12`) — there is **no `<meta name="app-rev">` single source** and no
  cache-bust bootstrap yet. Refactoring to the meta-tag + `DOMContentLoaded`
  injection pattern (and bumping the rev) is part of this work.

## Locked design (decided with Phil)
Roles: **admin / moderator / user** (mirror Veritas — shared data, roles gate
*who can edit*, not *who can see*). No customer/installer (no tenant split).
- **admin** (Phil / root): sees everything; has the Data modal (Backup / CSV /
  **Restore**); manages the access roster.
- **moderator**: sees everything; can edit content (entries + categories); NO
  Data modal; no roster management.
- **user**: sees everything; **can add / edit / rate restaurant entries**
  (collaborative list — Phil chose "all members can edit"); cannot edit the
  category list; NO Data modal; no roster management.
- Both root UIDs (`CuJigrzCbRfFFsM7uqe0mix46xH3`, `g4stabGPKiWAE23NTw6oUsqawWf1`)
  are always admin (lockout insurance) — they're already the Eats `ALLOWED_UIDS`,
  so keep both.

## ⚠️ The roster must live OUTSIDE the `eats` node
This is the one place Eats differs from Veritas/Taskboard and it drives the whole
rules shape:

- The client reads the **whole `eats` node** for every signed-in user. In RTDB a
  `.read` grant **cascades down** and **cannot be revoked by a child**. So if the
  roster were a child of `eats` (e.g. `eats/people`), giving members top-level
  `eats` read would **leak the roster** (emails + roles) to every member — the
  exact AutoFlag "parent-grant cascade" hole. (It would also pollute the
  entry-render loop, which treats every non-`categories` child as a restaurant.)
- Taskboard dodged this by reading **per-subtree** (top-level `taskboard` read is
  NOT member-granted). Eats reads wholesale, so that escape isn't available
  without a data migration.

**Decision: put the roster at a NEW sibling top-level key, `eats_access/{emailKey}`,
NOT under `eats`.** It gets its own admin-only rules and stays private no matter
how `eats` is read. Record = `{ email, role, name, addedAt, addedBy }`,
`role ∈ admin | moderator | user`. `emailKey` = the same transform as AutoFlag/
Veritas (client global-regex `emailToKey`; rules 6× `.replace('.','_')` chain).

> Orthodox alternative (more work, NOT recommended now): migrate entries to
> `eats/entries/{id}`, keep the roster at `eats/people`, make top-level `eats`
> read admin-only, and change the client to read `eats/entries` + `eats/categories`
> separately. That's a data migration + a bigger client rewrite for no functional
> gain over the `eats_access` sibling. Use the sibling node.

## Client changes (eats `index.html`)
1. **Refactor versioning FIRST** to the single-source pattern: add
   `<meta name="app-rev" content="…">`, fill the login-screen rev and footer rev
   from it at `DOMContentLoaded`, and add the cache-bust bootstrap
   (localStorage key `eats_lastRev`, one-time `?v={rev}` reload). Bump the rev —
   that's how we confirm the new `index.html` is live. (Building the gate is a
   client-code change, so the rev MUST bump regardless.)
2. **Replace the `ALLOWED_UIDS`-only gate with a roster gate**: after sign-in,
   allow if root UID **OR** `eats_access/{emailKey}` has a valid role
   (admin/moderator/user); else sign out with a clean "not authorized" message
   (reuse the existing `authError` element). Resolve and store the role.
   **Bounce null-role users** (sign out anyone not root and not on the roster) —
   the rules will deny them anyway, but the UX should be graceful.
3. **Role-gated UI:**
   - Data modal (Backup / CSV / **Restore**) → **admin only** (hide for
     moderator/user).
   - Category manager (add/remove/rename categories) → **admin or moderator**.
   - Add / edit / rate restaurant entries → **any roster member** (admin/mod/user).
   - New admin-only **"Manage Access" modal**: list / add / remove roster entries
     (email + role), writing `eats_access/{emailKey}`. (Separate concept from the
     food data — there's no existing `people` UI in Eats to collide with.)
4. Keep the existing mobile token-refresh fix (`getIdToken(true)` before `init()`)
   — it's unrelated to access and must stay.

## Rules (the `eats` block + the new `eats_access` block, in autoflag-installs)
Edit `access-model/PIECE3-rules-to-deploy.json` **in the autoflag-installs repo**.
Replace the root-UID-only `eats` block. Grant at the LEAF; do NOT grant member
read/write at a parent that would cascade. Use Eats's own `emailKey` transform and
both root UIDs. Roster reads point at **`eats_access`** (the sibling), not `eats`.

```
eats: {
  ".read":  member  (root | admin | moderator | user),   // whole-node read OK —
                                                          // no roster lives here
  ".write": admin-only,                                   // destructive Restore
                                                          //   = dbSet('eats', …)
  "categories": { ".write": admin | moderator },          // read inherited
  "$entryId":   { ".write": member }                      // add/edit/rate entries
}

eats_access: {                       // NEW sibling top-level roster (private)
  ".read":  admin-only,
  ".write": admin-only,
  "$emailKey": { ".read": own-record (==  caller's email key) }   // role self-read
}
```
Why this shape works (RTDB precedence + cascade rules):
- **Member whole-node read** is safe because the roster is NOT under `eats`.
- **Member entry writes** are granted at the `eats/$entryId` leaf, so they're
  allowed even though the top-level `eats` `.write` is admin-only.
- **Destructive Restore** writes the `eats` parent → governed by the top-level
  `eats` `.write` = admin-only (a child leaf grant can't authorize a parent write).
  An admin-only top-level write does NOT loosen children for non-admins.
- **`categories`** is a named child, so its explicit `.write` (admin|moderator)
  **overrides** the `$entryId` wildcard for that path — categories stay mod/admin
  even though plain entries are member-writable.
- **Roster privacy**: `eats_access` is admin read/write, with each user allowed to
  read only their OWN `$emailKey` record for role resolution (own-record read).
- **Keep every OTHER app's block byte-for-byte unchanged** (autoflag, taskboard,
  veritas, command_center, philinity, foodswipe, users, config). You are adding
  ONE new top-level key (`eats_access`) and rewriting ONE existing one (`eats`).

### Open decision for build time
The destructive **Restore** (`dbSet('eats', data)`) is admin-only in rules
(above). Import (`update` of `eats/{id}`) and per-field edits land on
`eats/$entryId`, so they're member-writable by design. If any of those per-entry
write paths should be tighter (e.g. delete-entry mod-only), decide at build time;
default is the collaborative "all members edit entries" Phil chose.

## Rollout (matched-pair, like AutoFlag / Veritas / Taskboard)
1. Branch on the eats repo.
2. Ship the **CLIENT first** (version refactor + roster gate + role-gated UI +
   Manage Access modal + rev bump). Safe under the CURRENT root-UID rules —
   building the gate doesn't lock anyone out (root still gets in; non-root can't
   yet, which is the status quo).
3. **Seed the roster:** add Phil (root, auto-admin) + any moderators/users to
   `eats_access` (admin adds via the Manage Access modal once rules allow; until
   then seed via the Firebase console).
4. Edit ONLY the `eats` block (+ add `eats_access`) in
   `access-model/PIECE3-rules-to-deploy.json` (autoflag-installs); open ONE PR on
   the **eats** repo for the client + note the rules change; share the link.
   **You (the bot) cannot deploy rules** — Phil pastes the whole PIECE3 file into
   the Firebase console.
5. **Deploy order is CLIENT → verify → RULES** (never rules first — lockout).
   Then **probe-verify each role** (token probes, like AutoFlag):
   - A logged-in account **NOT** in `eats_access` → read of `eats` = Permission
     denied (gate works).
   - A **user** → can read all entries, add/edit/rate an entry; **cannot** write
     `eats/categories`, write `eats_access`, or run Restore (`dbSet('eats',…)`).
   - A **moderator** → can edit categories; cannot open Manage Access / Data modal.
   - An **admin** → everything, including Restore and roster management.
   - Confirm no member can read `eats_access` beyond their own record.
6. Update `RULES-COORDINATION.md` (autoflag-installs) — flip the `eats` row from
   `🔒 root-UID only` to deployed/role-based — once Phil confirms the console
   publish.

## Kickoff prompt to paste into the new (eats) session
> Roll out role-based access to Philinity Eats (siglerventures/eats) per
> EATS-rollout-spec.md in the eats repo (shared rules/governance live in
> autoflag-installs/access-model). Roles admin/moderator/user over shared data
> (all members see + edit restaurant entries; categories = admin/moderator;
> Data-modal Restore + roster = admin). The roster lives at a NEW sibling key
> `eats_access/{emailKey}` (NOT under `eats`) because the client reads the whole
> `eats` node and a roster child would cascade-leak. First refactor versioning to
> the `<meta app-rev>` single-source + cache-bust pattern and bump the rev.
> Replace the root-UID-only `eats` rules block (member read; admin-only top-level
> write for Restore; `eats/$entryId` member write; `eats/categories` admin/mod
> write) and add the admin-only `eats_access` block (own-record self-read).
> Branch + PR + staged rollout (CLIENT → verify → RULES) + probe verification
> like AutoFlag. Same Firebase project (philinity-893d2); change only the eats +
> eats_access blocks.

## Also required: email + password sign-in (sign-in standard)
Per `SIGNIN-STANDARD.md` (autoflag-installs/access-model), Eats must offer
**email+password in addition to Google** (it's currently Google-only). Mirror
AutoFlag's flow: import `createUserWithEmailAndPassword` /
`signInWithEmailAndPassword` / `sendPasswordResetEmail`; add email+password
fields + a Sign-In/Create-Account toggle + "forgot password" to the auth gate;
friendly error mapping (`auth/invalid-credential`, `auth/email-already-in-use`,
`auth/weak-password`, `auth/too-many-requests`, …). No rules/roster change — role
resolution by email (`eats_access/{emailKey}`) is unchanged, and the
Email/Password provider is already enabled on `philinity-893d2`.

> Kickoff: "Add email+password sign-in to Eats alongside Google, mirroring
> AutoFlag's flow, per SIGNIN-STANDARD.md. Client-only; the email-keyed
> roster/role logic (eats_access/{emailKey}) is unchanged. Branch + PR; bump the
> rev."
