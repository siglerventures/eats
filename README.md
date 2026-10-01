# Philinity Eats 🍽

Our family restaurant tracker. Single-file web app (`index.html`), hosted on
GitHub Pages at https://siglerventures.github.io/eats/ , data in the shared
Firebase project `philinity-893d2`.

- **[BACKUP.md](BACKUP.md)** — what is backed up, where, and how to restore.
- **[EATS-rollout-spec.md](EATS-rollout-spec.md)** — access model, roles, rules, and design notes.
- Firebase rules live in `Autoflag/autoflag-installs` → `access-model/PIECE3-rules-to-deploy.json`
  (one shared ruleset for all apps; edit only the `eats` / `eats_access` blocks).

Merging to `main` deploys the app. Every client change bumps `<meta name="app-rev">` in `index.html`.
