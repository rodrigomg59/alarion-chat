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

## Local development (Docker)

Chatwoot already ships a working `docker-compose.yaml` (Rails, Sidekiq, Vite,
Postgres/pgvector, Redis, Mailhog). We don't edit that file — Alarion-specific
overrides go in `docker-compose.override.yaml`, which Docker Compose merges in
automatically. Today that file only fixes one thing: newer `pgvector/pgvector`
Postgres images refuse to boot with an empty `POSTGRES_PASSWORD` (the value
shipped in upstream's `docker-compose.yaml`), so we set one for local dev.

```bash
cp .env.example .env
# set SECRET_KEY_BASE (e.g. `ruby -rsecurerandom -e 'print SecureRandom.hex(64)'`)
# set POSTGRES_PASSWORD=postgres to match docker-compose.override.yaml

docker compose build base
docker compose build rails vite
docker compose up -d postgres redis mailhog
docker compose run --rm rails bundle exec rails db:create db:schema:load db:seed
docker compose up -d rails sidekiq vite
```

App at `http://localhost:3000`. Seeded login: `john@acme.inc` / `Password1!`
(change/remove before this ever goes near production). Mailhog UI at
`http://localhost:8025`.

Note: enabling `custom/` (see above) makes `ChatwootApp.custom?` true, which
makes Chatwoot's enterprise-injection code look up a top-level `Custom`
module. `custom/lib/custom.rb` (`module Custom; end`) must always exist, the
same way `enterprise/lib/enterprise.rb` does — without it, boot fails with
`NoMethodError: undefined method 'const_defined?' for false`.

## Host-side git hooks (rubocop, eslint)

`.husky/pre-commit` runs `eslint`/`lint-staged` on staged JS/Vue files and
`rubocop` on staged `.rb` files, **on the host**, not inside Docker. This
needs a host-side JS and Ruby toolchain even though the app itself runs in
containers:

```bash
# JS (Node 24.x / pnpm 10.x, pinned in package.json "engines")
brew install fnm
fnm install 24 && fnm use 24
corepack prepare pnpm@10.2.0 --activate
pnpm install

# Ruby (3.4.4, pinned in .ruby-version / Gemfile)
brew install rbenv ruby-build libpq
rbenv install 3.4.4   # picked up automatically via .ruby-version
gem install bundler -v 2.5.16
bundle config set --local build.pg --with-pg-config=/opt/homebrew/opt/libpq/bin/pg_config
bundle install        # installs into ./vendor/bundle_host, gitignored
```

Without this, `rubocop` silently fails inside the pre-commit hook (it's
wrapped in `|| true`) and Ruby changes never actually get linted before commit.

## Rules

- Never edit code directly on the production VPS — all development happens locally.
- Never deploy `latest` tags to production — always a pinned version.
- Deploys go out via Docker images built from this repo.
