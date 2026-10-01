# Philinity Eats — what is backed up, where, and how to restore

Eats is a single-file app (`index.html`) on GitHub Pages, with its data in the
shared Firebase project `philinity-893d2` (one Realtime Database and one Storage
bucket shared by every Philinity app). This page says where each piece lives,
what backs it up, and how to get it back.

## The short version

| Piece | Lives in | Backed up by | Automatic? |
|---|---|---|---|
| The program (`index.html`) | GitHub `siglerventures/eats` | GitHub itself (every version, every PR, forever) + a snapshot inside each App Backup ZIP | ✅ |
| Restaurant data (`eats`) | Realtime Database | Firebase nightly backup + App Backup ZIP | ✅ nightly |
| Roster / roles (`eats_access`) | Realtime Database | Firebase nightly backup + App Backup ZIP | ✅ nightly |
| Photos | Storage bucket `philinity-893d2.firebasestorage.app` | 📷 Download all photos ZIP (manual) + Storage soft delete (if turned on) | ⚠ manual |
| Firebase rules | GitHub `autoflag-installs/access-model/PIECE3-rules-to-deploy.json` | GitHub + the nightly `_rules.json.gz` | ✅ |
| askAI Cloud Function | GitHub `siglerventures/philinity-functions` (shared by all apps) | GitHub | ✅ |
| Google Maps key (`eats/_config`) | Realtime Database | Firebase nightly backup only (deliberately left OUT of the ZIP) | ✅ nightly |
| Sign-in accounts | Firebase Authentication | Not backed up — not needed (see below) | — |

## 1. Firebase nightly backup (automatic, all apps)

Firebase console → Realtime Database → **Backups** tab. Runs every night
(around 10:19 PM local time) and writes two files to the bucket
`philinity-893d2-default-rtdb-backups`:

- `<date>_philinity-893d2-default-rtdb_data.json.gz` — the **whole** database:
  accounting, autoflag, cellar, command_center, config, eats, eats_access,
  foodswipe, orbit, taskboard, users, veritas.
- `<date>_philinity-893d2-default-rtdb_rules.json.gz` — the rules as published.

This is the safety net. Check the bucket's **Lifecycle** setting now and then;
it should keep at least 30 days so a problem noticed late is still fixable.

The `eats/_config` key (the shared Google Maps key) is inside these files. The
bucket is private to project owners, so that's fine — just never share one of
these files outside.

## 2. 🗄️ App Backup (ZIP) — a copy you hold yourself

In the app: sign in as Admin → footer **📦 Data Manager** → **🗄️ Download app backup (.zip)**.
It downloads `philinity-eats-backup-<date>.zip` containing:

- `restore/eats.entries.json` — all restaurants (the `eats` node, minus `_config`).
- `restore/eats.categories.json` — the category list (`eats/categories`).
- `restore/eats.access.json` — the roster (`eats_access`).
- `firebase-eats-<date>-ALL.json` — everything in one file, for reading / searching only.
- `index.html` — the app exactly as served that day.
- `MANIFEST.txt` — counts per section.
- `RESTORE-README.txt` — the restore steps (same as section 5 below).

Run it after any big round of edits, and before using any Data Manager tool
that changes many records at once.

## 3. 📷 Photos (ZIP)

Data Manager → **📷 Download all photos (.zip)**. Walks every record, finds every Storage link, and
downloads the actual image files into one ZIP. Each file is stored under its real
Storage path (e.g. `eats/<entryId>/photo-….jpg`), so restoring = uploading back to
the same path. Also includes `photo-index.csv` (file → record → link) and
`MISSING.txt` (links that returned 403/404 — stale links to replaced/deleted files).

Needs the bucket's CORS rule to list `https://siglerventures.github.io`
(already done; the app shows the setup box if it ever stops working — and
remember that rule is shared with AutoFlag, so always ADD an origin, never replace).

Also recommended: Google Cloud console → Cloud Storage → the bucket →
**Protection** → turn on **Soft delete**. A deleted or overwritten photo then
stays recoverable for 7 days without needing a ZIP.

## 4. Other manual exports

- **Full JSON Backup** — just the `eats` node as one JSON file (older format; the
  ZIP replaces it, but it's still there).
- **CSV** — a spreadsheet of the restaurants for reading, not for restoring.

## 5. Restoring

### ⚠ The one rule
**Never "Import JSON" at the `eats` node from an ALL file, and never at the
database root.** Firebase REPLACES the node you import at with the file. At the
root that wipes every other app. At `eats` with the wrong file, `_config` and
anything added since the backup are erased.

### Restore one section from the App Backup ZIP
1. Unzip the backup.
2. Firebase console → Realtime Database → **Data**.
3. Click into the node you're restoring (`eats`, `eats/categories`, or `eats_access`).
4. ⋮ menu → **Import JSON** → pick the matching `restore/eats.<section>.json`.
5. If you imported `eats` (entries), the Maps key is gone: in the app, Data
   Manager → **Enable maps & drive time for everyone** → click once.

Or use the app: Data Manager → **Restore from Backup** → choose the entries file.
It asks for confirmation and spells out exactly what it will overwrite.

### Restore from a Firebase nightly backup
1. Backups tab → download the `_data.json.gz` for the night you want.
2. Unzip it and open the JSON.
3. Copy **only** the `eats` object into its own file (and `eats_access` into another, if needed).
4. Import each at its own node, as above. Never the whole file at the root.

### Restore photos
Firebase console → Storage → navigate to the folder in the path → **Upload file**.
Or, for many files, Cloud Shell: `gsutil cp -r eats gs://philinity-893d2.firebasestorage.app/`
from inside the unzipped photos folder. Links in the records keep working because
the paths match.

### Restore dates after a bulk tool
If a bulk tool ever stamps `lastUpdated` on everything (it shouldn't any more —
tools now use `infoUpdatedAt` / `geocodedAt`), Data Manager → **Restore dates
from a backup** reads an older backup and puts the real dates back without
touching anything else. The button is **🗓 Restore “last updated” dates from a backup**.

### The program itself
GitHub → `siglerventures/eats` → `index.html`. Any past version is in the
commit history. GitHub Pages serves `main`, so a merge is a deploy.

### Rules
GitHub → `Autoflag/autoflag-installs` → `access-model/PIECE3-rules-to-deploy.json`
→ **Copy raw file** (selecting the page text only grabs part of it) → Firebase →
Realtime Database → Rules → paste the whole thing → Publish. Edit only the `eats`
and `eats_access` blocks; every other app's block must stay byte-identical.

### Sign-in accounts
Not backed up and don't need to be. People sign in again (Google or
email+password) and the roster in `eats_access` gives them their role back.

## 6. What's still manual

- Photos ZIP — run it after adding a batch of photos, or turn on soft delete.
- App Backup ZIP — nice to have before big edits; the nightly backup is the real net.
- Checking the backup bucket's retention once in a while.
