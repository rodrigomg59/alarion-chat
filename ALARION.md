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

## Rules

- Never edit code directly on the production VPS — all development happens locally.
- Never deploy `latest` tags to production — always a pinned version.
- Deploys go out via Docker images built from this repo.
