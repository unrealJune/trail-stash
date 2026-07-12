# Publishing trail-stash as its own public repo

`trail-stash` currently lives inside the streetCryptid monorepo at `infra/trail-stash/`, but the
whole `infra/` tree is **gitignored** there (it's deliberately excluded from the public app repo).
That has one important consequence:

- These files are **not tracked** by the app repo's git history, so `git subtree split` /
  `git filter-repo` can't extract them — there's nothing to extract.
- The included CI (`.github/workflows/publish.yml`) therefore **never runs from the app repo**. It
  only runs once this directory is the **root of its own GitHub repo**.

So "publicize it" = give this directory its own repository and push. Everything here is already
self-contained and rooted at this directory for exactly that.

> **Status:** the canonical instance is already live at
> <https://github.com/unrealJune/trail-stash>. It was initialized **in-place** (`git init` inside
> `infra/trail-stash/`, which is clean because the monorepo gitignores all of `infra/`), so you can
> keep developing here and `git push` from this directory. The copy-out flow below is the
> alternative for anyone forking a fresh self-hosted instance.

## 1. Create the standalone repo

```bash
# From the monorepo root, copy the directory out (no git history to preserve).
cp -r infra/trail-stash /tmp/trail-stash
cd /tmp/trail-stash

git init -b main
git add .
git commit -m "Initial import: trail-stash server, Helm chart, and CI"

gh repo create trail-stash --public --source=. --remote=origin --push
# or: create the repo in the GitHub UI, then `git remote add origin … && git push -u origin main`
```

The push to `main` triggers `.github/workflows/publish.yml`: it runs the pure-core tests, builds
and pushes the image to `ghcr.io/<owner>/trail-stash`, and packages + pushes the Helm chart to
`oci://ghcr.io/<owner>/charts/trail-stash`. `<owner>` is derived automatically from the repo owner
(lowercased) — nothing to hardcode in the workflow.

## 2. Adjust the owner placeholders (one-time)

A few files carry a default owner (`unrealJune` / `unrealjune`) or `<owner>` placeholders. Update
them to match your GitHub owner if different:

| File | What to change |
| --- | --- |
| `charts/trail-stash/values.yaml` | `image.repository: ghcr.io/<owner>/trail-stash` |
| `charts/trail-stash/Chart.yaml` | `home` / `sources` URLs, `maintainers` |
| `rust/Dockerfile` | `org.opencontainers.image.source` label |
| `README.md`, `INSTALL.md`, `charts/trail-stash/README.md` | `<owner>` in the example commands |
| `LICENSE`, `NOTICE` | copyright holder, if not "June Philip" |

The CI workflow itself needs no edits — it uses `${{ github.repository_owner }}`.

## 3. Make the GHCR packages public (so others can `docker pull` / `helm install`)

GHCR packages are **private by default**, even in a public repo. After the first successful
publish, flip both packages to public — one time:

1. GitHub → your profile/org → **Packages** → `trail-stash` (and `charts/trail-stash`).
2. **Package settings** → **Change visibility** → **Public**.
3. While there, under **Manage Actions access**, confirm the repo has `write` (the workflow's
   `packages: write` permission already grants this on push).

If you'd rather keep them private, skip this and have consumers add an `imagePullSecret`
(`--set imagePullSecrets[0].name=ghcr-creds`) and `helm registry login ghcr.io` before installing.

## 4. Cut a versioned release (optional but recommended)

`main` pushes publish moving `:latest` + `:sha-<short>` tags. For an immutable, semver-pinned
image + chart, push a tag:

```bash
git tag v0.1.0 && git push origin v0.1.0
```

That publishes `ghcr.io/<owner>/trail-stash:{0.1.0,0.1,latest}` and chart `0.1.0`. Consumers then:

```bash
helm install trail-stash oci://ghcr.io/<owner>/charts/trail-stash --version 0.1.0 \
  --set secret.existingSecret=trail-stash
```

## What an operator must still supply

The image/chart are generic; each deployment provides its own secrets (never in the repo):

- `TRAIL_STASH_SECRET_KEY` — 64 hex chars (`openssl rand -hex 32`); the stable node identity.
- `TRAIL_STASH_PSK` — control-API bearer (`openssl rand -hex 32`); must match the app's
  `EXPO_PUBLIC_TRAIL_STASH_PSK`.

See `INSTALL.md` for the full deploy + app-wiring runbook.

## Keeping the fork in sync (optional)

If you keep developing the service in the monorepo, re-syncing is a plain copy back out (the two
trees share no git history):

```bash
rsync -a --delete --exclude '.git' --exclude 'rust/target' \
  /path/to/streetCryptid/infra/trail-stash/ /tmp/trail-stash/
```

Review the diff, commit, and push.
