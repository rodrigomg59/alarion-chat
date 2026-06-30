# Alarion Chat — Developer Notes

This repository is the Alarion Chat product, built on top of [Chatwoot](https://github.com/chatwoot/chatwoot).
It keeps the full upstream Git history so new Chatwoot releases can be merged in over time.

This file (not `CONTRIBUTING.md`, which stays close to upstream to minimize merge conflicts)
documents everything specific to working on this fork.

## Remotes

- `origin` → `rodrigomg59/alarion-chat` — this repo, where we develop and deploy from.
- `upstream` → `chatwoot/chatwoot` — official Chatwoot, read-only, used only to pull in new releases.

This is **not** a GitHub fork. It's an independent repository that shares Git history with
Chatwoot, so `git merge upstream/<tag>` works like it would on a fork, without the
limitations GitHub forks impose (visibility, PR assumptions, etc).

## Required tooling

- **Git LFS** is required to clone this repo correctly. Large binary files (`*.onnx`, `*.pdf`)
  are stored in LFS, not in the regular Git history.

  ```bash
  brew install git-lfs
  git lfs install
  git clone git@github.com:rodrigomg59/alarion-chat.git
  ```

  If you already cloned without LFS installed, run `git lfs pull` afterwards to fetch the
  real file contents.

## Pulling in new Chatwoot releases

`main` tracks a specific Chatwoot release tag (currently based on `v4.15.1`), never
`upstream/develop` and never `latest`. To upgrade:

```bash
git fetch upstream --tags
git checkout -b upgrade/vX.Y.Z main
git merge vX.Y.Z          # resolve conflicts, run tests
git checkout main
git merge upgrade/vX.Y.Z
git push origin main
```

## Where Alarion-specific code goes

Chatwoot already has a built-in extension mechanism for white-label products,
the same one used for the `enterprise/` edition: `ChatwootApp.custom?` detects
a `custom/` directory at the repo root, and `prepend_mod_with` / `include_mod_with`
/ `extend_mod_with` calls already scattered through core classes pick up
`Custom::X` modules automatically. We wired the matching autoload/view paths
into `config/application.rb` (mirroring the existing `enterprise/` block) and
documented the convention in [custom/README.md](./custom/README.md).

Rule of thumb:

- **New backend feature or behavior override** → `custom/app/...`, `Custom::` namespace. See `custom/README.md`.
- **Branding** (name, logo, colors, terms/privacy URLs) → `config/installation_config.yml` values, set via env vars or Super Admin UI. No code change, no merge risk.
- **Frontend customization** → no overlay mechanism exists upstream for this (EE frontend features are gated via API flags, not a directory split). Until we need more, keep frontend branding to CSS variables / the installation_config-driven logo & name, and revisit a proper isolation pattern if/when real custom UI is needed.

Avoid editing core Chatwoot files for behavior changes — every line changed
there is a future merge conflict. The one acceptable exception is adding a
single `prepend_mod_with('ClassName')`-style hook line to a class that doesn't
have one yet, which is small and rarely conflicts.

## Rules

- Never edit code directly on the production VPS — all development happens locally.
- Never deploy `latest` tags to production — always a pinned version.
- Deploys go out via Docker images built from this repo.
