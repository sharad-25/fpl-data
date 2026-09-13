# From a fresh GitHub account to a published gameweek

Six steps, about fifteen minutes. I ran the whole sequence against a real git
repo with your GW4 output first, so the commands below are tested rather than
plausible: 11 files, 1,069 KB, committed clean.

---

## 1. Create the repository

github.com → **+** (top right) → **New repository**

| field | value | why |
|---|---|---|
| Repository name | `fpl-data` | it appears in every URL the app fetches |
| Description | *Weekly FPL model output* | |
| Visibility | **Public** | see below |
| **Add a README file** | **✅ tick it** | **this one matters** |
| .gitignore / licence | skip | |

**Tick the README.** A repository with no commits has no branch, and
`git clone` on one fails — which is the first thing the Colab cell does. One
tick now saves a confusing error later.

**Public, not private.** The contents are derived football numbers, nothing of
yours. Public means the app needs no credentials to read, which means **no
token ships inside the APK** — and a token in an APK is a token anyone you send
it to can extract in a minute. Your write token stays in Colab; readers need
nothing.

---

## 2. Add the three starter files

In the repo: **Add file → Create new file**, once per file. Contents are in
`repo_starter/` alongside this document.

- **`README.md`** — replace what GitHub generated. Describes the layout and how
  to read the draws array, so future-you (or a collaborator) isn't reverse
  engineering a binary blob.
- **`.gitattributes`** — one line, `*.i8 binary`. Without it git treats the
  draws as text and tries to diff them; a one-gameweek change then reads as a
  600 KB rewrite and bloats the history.
- **`.gitignore`** — Colab checkpoints and `__pycache__`.

---

## 3. Make a write token

**Settings** (your account menu, not the repo) → **Developer settings** →
**Personal access tokens** → **Fine-grained tokens** → **Generate new token**

| field | value |
|---|---|
| Token name | `colab-fpl-push` |
| Expiration | 90 days |
| Repository access | **Only select repositories** → `fpl-data` |
| Permissions → Repository → **Contents** | **Read and write** |

Everything else stays "No access". Copy the token — **GitHub shows it once**.

Fine-grained and scoped to one repo is the point: if it leaks, the blast radius
is one repository of public football numbers.

---

## 4. Put the token in Colab Secrets

Colab → **🔑** in the left sidebar → **Add new secret**

- Name: `GH_TOKEN`
- Value: the token
- **Notebook access: ON** (the toggle — it is off by default and the cell will
  fail without it)

**Not in a notebook cell.** Same rule as the odds key, same reason: a notebook
gets shared, screenshotted and version-controlled.

---

## 5. Add the push cell

Last cell of the weekly notebook, after the sheet write:

```python
# ---- publish the app payload -------------------------------------------
from pathlib import Path
from google.colab import userdata
from fplmodel import export_app

GH_USER = "<your-github-username>"
TOKEN   = userdata.get('GH_TOKEN')
OUT     = Path('/content/fpl-data')

!git config --global user.email "you@example.com"
!git config --global user.name  "fpl-model"
!rm -rf {OUT}
!git clone --depth 1 https://{TOKEN}@github.com/{GH_USER}/fpl-data.git {OUT}

m = export_app.write(board, extras, cfg, OUT)

!cd {OUT} && git add -A \
  && git commit -m "GW{m['gw']} · model {m['model']}" \
  && git push

print(f"published GW{m['gw']}:",
      sum(f['bytes'] for f in m['files'].values()) // 1024, "KB")
```

`export_app.write` needs `board`, `extras` and `cfg`, which the weekly run
already has in scope. It writes `latest/` and `gw/<NN>/`, and returns the
manifest.

**Clone fresh each week** rather than keeping a working copy. Colab discards
`/content` between sessions anyway, and a fresh clone can never push a stale
branch.

---

## 6. Run it, then check three things

1. The cell prints `published GW4: 1069 KB`.
2. The repo shows a commit named `GW4 · model 1.37.1` with `latest/`,
   `gw/04/` and `manifest.json`.
3. This URL returns JSON in a browser:

```
https://raw.githubusercontent.com/<you>/fpl-data/main/manifest.json
```

That third one is the real test — it is exactly what the phone will call.

```json
{ "schema": 1, "season": "2026-27", "gw": 4, "gws": [4,5,6,7,8,9],
  "built_at": "...", "model": "1.37.1",
  "files": { "players.json": { "sha": "3b4dc965a022963a", "bytes": 408943 }, ... } }
```

---

## What the APK does with it

On launch, fetch `manifest.json` — a few hundred bytes. Compare each `sha`
against what is cached on disk; download only what differs. Most launches cost
one small request and the app opens instantly, offline.

Everything personal — leagues, squads, rivals, live scores — comes from FPL's
public API on the device. This repo is the only thing you publish, and it is
identical for every user.

---

## Two things worth doing this week

**Push GW4 now, today, before the app can read anything.** The `gw/NN/` archive
only has value if it starts early, and a gameweek you don't push is history you
cannot reconstruct. GW4 is settling now; get it committed.

**Then push again between Friday's deadline and Saturday's first kickoff.**
That same window is when you need a live refresh for a clean drift baseline, so
it is one trip for two jobs.

## When it goes wrong

| symptom | cause |
|---|---|
| `repository not found` on clone | token lacks Contents:Write, or `GH_USER` is wrong |
| `SecretNotFoundError` | Notebook access toggle is off in Colab Secrets |
| `fatal: couldn't find remote ref main` | you skipped the README tick, so the repo has no branch |
| `nothing to commit` | the export produced identical bytes — a rerun of the same gameweek, which is fine |
| 404 on the raw URL | the repo is private, or the branch is `master` not `main` |
