# Resync with mattermost-plugin-starter-template Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring this repo's build tooling, CI, server, and webapp up to date with the current `mattermost-plugin-starter-template` (pinned at commit `22ef315a11a1219d604d2a6eff6968b202d6023b`, the same commit used for the equivalent migration in `mattermost-plugin-gpx-preview`).

**Architecture:** Unlike `mattermost-plugin-gpx-preview` (webapp-only, with real custom feature code to preserve), **this repo is an unmodified `mattermost-plugin-starter-template` clone** — `server/` and `webapp/src/` contain only the template's own placeholder scaffold (a `ServeHTTP` that prints "Hello, world!", an empty `configuration` struct, an empty webapp `Plugin.initialize()`). There is no custom logic to carefully adapt. That makes this migration simpler in one respect (most files can be replaced wholesale rather than diffed line-by-line) but larger in another: this repo has a real `server/` directory, so it picks up the full Go API migration from `mattermost-server/v6` to `mattermost/mattermost/server/public` (router via `gorilla/mux`, `pluginapi.Client` wrapper, slash-command scaffold, KV store scaffold, scheduled background job) that gpx-preview never had to deal with. golangci-lint also actually *runs* here (gpx-preview skipped it — no `server/`).

**Tech Stack:** Go 1.25 (real server-side dependency this time, not just build tooling), Node (version pinned via `.nvmrc`), webpack 5, React 18, TypeScript, mattermost-redux 11 / `@mattermost/types` 11 / `@mattermost/client` 11, Jest 27, `gorilla/mux`, `pluginapi`.

**Key verified facts:**
- `plugin.json`'s `id`/`name`/`description`/`homepage_url`/`support_url`/`release_notes_url` are still the upstream template's own values (e.g. `homepage_url` points at `github.com/mattermost/mattermost-plugin-starter-template`, not this fork) — this was never customized, same as gpx-preview's never-renamed Go module path. **Do not fix this as a side effect of the resync** — it's a deliberate, separate branding decision for the user to make (consistent with how the gpx-preview migration treated its own leftover `go.mod` module path).
- The new template has **no root-level `plugin.go`** (the `go:embed plugin.json` → `model.Manifest` pattern this repo currently has at the repo root, package `root`) — manifest handling moved entirely to the webapp side's generated `webapp/src/manifest.ts` plus the `build/bin/manifest` CLI. This file should be deleted, not migrated.
- New template's `server/` gained: `api.go` (gorilla/mux router + auth middleware + `/api/v1/hello`), `command/` (slash-command scaffold + mocks), `store/kvstore/` (KV store scaffold), `job.go` (scheduled background job). None of this exists in the current repo.
- New template's `go.mod` requires `github.com/gorilla/mux`, `github.com/mattermost/mattermost/server/public`, `github.com/pkg/errors`, `go.uber.org/mock` — unlike gpx-preview, **do not trim these via `go mod tidy`** — they're genuinely used by the new `server/` code this plan installs.
- `.circleci/config.yml` (orb `mattermost/plugin-ci@0.1.6`) is the only CI config on `master`. An unmerged `ci/pr-checks` branch has an untested draft GitHub Actions workflow (Node 14, Go 1.16) — this plan does not build on it; it's superseded outright by the official reusable workflow, same as gpx-preview's resync did.
- An unmerged `chore/dependabot-config` branch exists with a `.github/dependabot.yml` (gomod at `/`, npm at `/webapp`) — out of scope for this plan; the user was already informed it's still structurally accurate and asked separately whether to merge it.
- The `mattermost-plugin-starter-template` reference checkout from the gpx-preview migration is reusable here: `/tmp/mattermost-plugin-starter-template-ref`, pinned at `22ef315a11a1219d604d2a6eff6968b202d6023b`. Re-clone only if that path no longer exists.

---

### Task 1: Ensure the pinned reference checkout exists

**Files:** none (local reference checkout, not committed)

- [ ] **Step 1: Reuse or recreate the reference checkout**

```bash
if [ ! -d /tmp/mattermost-plugin-starter-template-ref ]; then
  git clone https://github.com/mattermost/mattermost-plugin-starter-template /tmp/mattermost-plugin-starter-template-ref
  git -C /tmp/mattermost-plugin-starter-template-ref checkout 22ef315a11a1219d604d2a6eff6968b202d6023b
fi
git -C /tmp/mattermost-plugin-starter-template-ref rev-parse HEAD
```

Expected: prints `22ef315a11a1219d604d2a6eff6968b202d6023b`.

All `cp` commands below reference `$REF=/tmp/mattermost-plugin-starter-template-ref`.

---

### Task 2: Replace build tooling and root config with upstream

**Files:**
- Modify: `Makefile`, `.golangci.yml`, `.editorconfig`, `.gitignore`, `go.mod`, `go.sum`
- Replace: `build/manifest/`, `build/pluginctl/`, `build/setup.mk`, `build/custom.mk`
- Create: `build/.gitignore`, `.vscode/settings.json`, `.nvmrc`
- Delete: `build/go.mod`, `build/go.sum`, `build/sync/`, `build/legacy.mk` (all removed upstream — same pattern as the gpx-preview migration: old template split build tooling into its own Go module and had a `make sync` mechanism; the new template folds build tooling into the single root `go.mod` and has no `sync`/`legacy.mk`)

- [ ] **Step 1: Remove obsolete build-only Go module and legacy/sync files**

```bash
git rm -r build/go.mod build/go.sum build/sync build/legacy.mk
```

- [ ] **Step 2: Copy upstream build tooling and root config verbatim**

```bash
REF=/tmp/mattermost-plugin-starter-template-ref
cp "$REF/Makefile" Makefile
cp "$REF/.golangci.yml" .golangci.yml
cp "$REF/.editorconfig" .editorconfig
cp "$REF/.gitignore" .gitignore
cp "$REF/go.mod" go.mod
cp "$REF/go.sum" go.sum
rm -rf build/manifest build/pluginctl
cp -r "$REF/build/manifest" build/manifest
cp -r "$REF/build/pluginctl" build/pluginctl
cp "$REF/build/setup.mk" build/setup.mk
cp "$REF/build/custom.mk" build/custom.mk
cp "$REF/build/.gitignore" build/.gitignore
mkdir -p .vscode
cp "$REF/.vscode/settings.json" .vscode/settings.json
cp "$REF/.nvmrc" .nvmrc
```

Expected: all files copied; `git status --short` shows the changes.

- [ ] **Step 3: Leave the Go module path and plugin.json identity fields untouched**

Same reasoning as the gpx-preview migration: don't rename `github.com/mattermost/mattermost-plugin-starter-template` in `go.mod`, and don't fix `plugin.json`'s stale `homepage_url`/etc. as a side effect of this plan. That's a separate, deliberate decision.

- [ ] **Step 4: Verify the build tooling compiles (server code not migrated yet, so full build will fail — that's expected until Task 3)**

```bash
go build ./build/... 2>&1
```

Expected: `build/manifest` and `build/pluginctl` compile cleanly. (Don't run `go build ./...` yet — `server/` and root `plugin.go` still reference the old `mattermost-server/v6` API at this point and will fail; that's resolved in Task 3.)

- [ ] **Step 5: Commit**

```bash
git add -A -- Makefile .golangci.yml .editorconfig .gitignore go.mod go.sum build .vscode .nvmrc
git commit -m "$(cat <<'EOF'
chore: sync build tooling with mattermost-plugin-starter-template

Pull in the current Makefile, golangci-lint v2 config, and build/
tooling (manifest/pluginctl) from upstream. Drop the old per-folder
build/go.mod, the build/sync machinery, and build/legacy.mk, all
removed upstream in favor of a single root go.mod. Unlike the
equivalent gpx-preview migration, go.mod's dependencies are NOT
trimmed via `go mod tidy` here -- this repo has real server/ code
that needs gorilla/mux, mattermost/server/public, etc.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Replace server/ with the upstream scaffold

**Files:**
- Delete: `plugin.go` (repo root — the `go:embed` manifest pattern, removed upstream)
- Replace: `server/plugin.go`, `server/main.go`, `server/configuration.go`, `server/plugin_test.go`
- Create: `server/api.go`, `server/job.go`, `server/command/command.go`, `server/command/command_test.go`, `server/command/mocks/mock_commands.go`, `server/store/kvstore/kvstore.go`, `server/store/kvstore/startertemplate.go`

This repo has no custom server logic (confirmed: `server/plugin.go`'s only behavior is `ServeHTTP` writing `"Hello, world!"`, `server/configuration.go` has zero fields). There is nothing to preserve — copy upstream's scaffold wholesale.

- [ ] **Step 1: Remove the obsolete root-level manifest embed**

```bash
git rm plugin.go
```

- [ ] **Step 2: Replace server/ wholesale with upstream's**

```bash
REF=/tmp/mattermost-plugin-starter-template-ref
rm -f server/plugin.go server/main.go server/configuration.go server/plugin_test.go
cp -r "$REF/server/." server/
```

Expected: `server/` now contains `api.go`, `command/` (with `command.go`, `command_test.go`, `mocks/mock_commands.go`), `configuration.go`, `job.go`, `main.go`, `plugin.go`, `plugin_test.go`, `store/kvstore/` (with `kvstore.go`, `startertemplate.go`), `.gitignore`.

- [ ] **Step 3: Verify it builds**

```bash
go build ./... 2>&1
```

Expected: exits 0. If it doesn't, the error will name the exact missing import/symbol — likely a `go.mod` entry that Task 2 didn't pick up; re-copy `go.mod`/`go.sum` from `$REF` rather than hand-editing.

- [ ] **Step 4: Verify `go mod tidy` produces no diff**

```bash
go mod tidy -v
git --no-pager diff --exit-code go.mod go.sum
```

Expected: clean exit (0) — unlike gpx-preview, this should need **no trimming**, since the new `server/` code now genuinely uses everything upstream's `go.mod` declares. If it does find something unused, that's worth a second look before accepting the diff (it may mean a copy step above missed a file).

- [ ] **Step 5: Run the Go tests**

```bash
go test ./... 2>&1
```

Expected: `server` (TestServeHTTP via the new router), `server/command` (TestHelloCommand), `build/manifest`, `build/pluginctl` all pass (or report "no test files" where applicable).

- [ ] **Step 6: Run golangci-lint** (this actually executes here, unlike gpx-preview which has no `server/`)

```bash
go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.9.0
$(go env GOPATH)/bin/golangci-lint run ./...
```

Expected: exits 0 — this is upstream's own maintained scaffold, so it should already satisfy upstream's own lint config. If it doesn't, read the specific finding; don't suppress it blindly.

- [ ] **Step 7: Commit**

```bash
git add -A -- plugin.go server
git commit -m "$(cat <<'EOF'
chore: replace server/ scaffold with current starter-template

This repo had no custom server logic (ServeHTTP just printed "Hello,
world!", configuration had zero fields), so there's nothing to adapt
-- replace the whole server/ tree wholesale with upstream's current
scaffold: gorilla/mux router (api.go), pluginapi.Client wrapper,
slash-command scaffold (command/), KV store scaffold (store/kvstore/),
and a scheduled background job (job.go). Also removes the root-level
plugin.go go:embed manifest pattern, dropped upstream in favor of
manifest handling living entirely on the webapp side.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Bump plugin.json minimum server version

**Files:**
- Modify: `plugin.json`

- [ ] **Step 1: Update `min_server_version`**

Change `"min_server_version": "5.37.0",` to `"min_server_version": "6.2.1",`. Leave every other field untouched (see "Key verified facts" above — the identity fields are a separate, deliberate decision, not part of this plan).

- [ ] **Step 2: Commit**

```bash
git add plugin.json
git commit -m "$(cat <<'EOF'
chore: bump min_server_version to 6.2.1

The new server/ scaffold uses mattermost/server/public APIs (pluginapi,
cluster.Schedule for the background job) that target a newer minimum
server version than 5.37.0 declared.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Switch CI to the official reusable workflow

**Files:**
- Delete: `.circleci/config.yml` (and the now-empty `.circleci/` directory)
- Create: `.github/workflows/ci.yml`

This repo has no existing `.github/workflows/` on `master` to rename (unlike gpx-preview, which had a `pr-checks.yml` to convert). The `ci/pr-checks` branch has a draft workflow, but it's untested and being superseded outright — don't merge or build on it.

- [ ] **Step 1: Remove the legacy CircleCI config**

```bash
git rm -r .circleci
```

- [ ] **Step 2: Create the official reusable workflow**

Write `.github/workflows/ci.yml`:
```yaml
name: ci
on:
  schedule:
    - cron: "0 0 * * *"
  push:
    branches:
      - master
    tags:
      - "v*"
  pull_request:

permissions:
  contents: read
  id-token: write

jobs:
  plugin-ci:
    uses: mattermost/actions-workflows/.github/workflows/plugin-ci.yml@main
    secrets: inherit
```

Its `delivery` job (S3 upload, the only one needing AWS secrets) only runs when `github.repository_owner == 'mattermost'`, so it's a no-op on this fork.

- [ ] **Step 3: Commit**

```bash
git add -A -- .circleci .github/workflows
git commit -m "$(cat <<'EOF'
ci: switch to the official mattermost plugin-ci reusable workflow

Replaces the unused legacy CircleCI config with
mattermost/actions-workflows/.github/workflows/plugin-ci.yml, which
reads tool versions from .nvmrc/go.mod and runs make check-style /
make test / make dist. Its delivery (S3 upload) job only runs for
github.repository_owner == 'mattermost', so it's a no-op here. The
existing untested draft on the ci/pr-checks branch is superseded by
this, not merged.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: Replace webapp/ with the upstream scaffold

**Files:**
- Replace: `webapp/package.json`, `webapp/webpack.config.js`, `webapp/babel.config.js`, `webapp/tsconfig.json`, `webapp/tests/setup.tsx`, `webapp/.eslintrc.json`, `webapp/src/index.tsx`, `webapp/src/manifest.test.tsx`, `webapp/src/types/mattermost-webapp/index.d.ts`
- Create: `webapp/.eslintignore`
- Delete: `webapp/src/manifest.ts` (becomes generated, same as gpx-preview)
- Regenerate: `webapp/package-lock.json`

This repo's `webapp/src/index.tsx` is confirmed to be the empty stock stub (no custom UI registered) and `webapp/src/types/mattermost-webapp/index.d.ts` is the minimal stock stub (one `registerPostTypeComponent` method) — there's nothing custom to preserve, so replace wholesale rather than hand-editing import paths like the gpx-preview migration had to.

- [ ] **Step 1: Replace package.json and regenerate the lockfile**

```bash
REF=/tmp/mattermost-plugin-starter-template-ref
cp "$REF/webapp/package.json" webapp/package.json
cd webapp
rm -f package-lock.json
rm -rf node_modules
npm install
cd ..
```

Expected: completes without needing git/SSH credentials (no more git-based `mattermost-webapp` devDependency).

- [ ] **Step 2: Replace the rest of the webapp tooling and source wholesale**

```bash
REF=/tmp/mattermost-plugin-starter-template-ref
cp "$REF/webapp/webpack.config.js" webapp/webpack.config.js
cp "$REF/webapp/babel.config.js" webapp/babel.config.js
cp "$REF/webapp/tsconfig.json" webapp/tsconfig.json
cp "$REF/webapp/tests/setup.tsx" webapp/tests/setup.tsx
cp "$REF/webapp/.eslintrc.json" webapp/.eslintrc.json
cp "$REF/webapp/.eslintignore" webapp/.eslintignore
cp "$REF/webapp/src/index.tsx" webapp/src/index.tsx
cp "$REF/webapp/src/manifest.test.tsx" webapp/src/manifest.test.tsx
cp "$REF/webapp/src/react_fragment.test.tsx" webapp/src/react_fragment.test.tsx
cp "$REF/webapp/src/types/mattermost-webapp/index.d.ts" webapp/src/types/mattermost-webapp/index.d.ts
git rm webapp/src/manifest.ts
```

- [ ] **Step 3: Generate manifest.ts and verify it**

```bash
make apply
cat webapp/src/manifest.ts
```

Expected: a JSON.parse-embedded manifest matching this repo's `plugin.json` (default export only — no named `id`/`version` exports). `git status --short` should show nothing for `webapp/src/manifest.ts` (gitignored from Task 2's `.gitignore` copy).

- [ ] **Step 4: Lint, type-check, test, build**

```bash
cd webapp
npm run lint
npm run check-types
npm test
npm run build
cd ..
```

Expected: all four pass. If lint reports anything, it'll be in the freshly-copied upstream files — that would be surprising (it's upstream's own maintained code) and worth investigating rather than auto-fixing blindly, since it could mean a stale cached `.eslintrc.json`/`eslintcache` from before this task. Clear `webapp/.eslintcache` and retry if so.

- [ ] **Step 5: Commit**

```bash
git add webapp .nvmrc
git status --short
git commit -m "$(cat <<'EOF'
chore: replace webapp/ scaffold with current starter-template

This repo's webapp/src had no custom UI (index.tsx's
Plugin.initialize() was an empty stub, types/mattermost-webapp had
only a placeholder PluginRegistry interface), so there's nothing to
adapt -- replace package.json (React 16->18, webpack 4->5, drops the
git-based mattermost-webapp devDependency), build/test tooling, and
the stock source files wholesale with upstream's current versions.
webapp/src/manifest.ts is now generated via `make apply` instead of
committed, matching the gpx-preview migration's approach.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 7: Full validation pass

**Files:** none (verification only — fix forward in the relevant file from earlier tasks if something fails)

- [ ] **Step 1: Clean reinstall and full Go verification**

```bash
go build ./...
go test ./...
$(go env GOPATH)/bin/golangci-lint run ./...
```

Expected: all exit 0.

- [ ] **Step 2: Clean webapp reinstall**

```bash
rm -rf webapp/node_modules
make webapp/node_modules
```

Expected: completes without errors, no SSH/git credential prompts.

- [ ] **Step 3: Full `make` pipeline**

```bash
make check-style
make test
make dist
ls dist/
```

Expected: `make check-style` runs golangci-lint AND eslint/tsc this time (unlike gpx-preview — `HAS_SERVER` is set here) and exits 0; `make test` exits 0; `make dist` produces `dist/com.mattermost.plugin-starter-template-0.1.0.tar.gz` (still the un-rebranded id, per Task 2 Step 3 / Task 4's deliberate scope limit).

- [ ] **Step 4: If everything passes, no commit needed for this task** — pure verification. Any fix discovered here should already be committed at the point it was fixed, not batched into one "fix everything" commit.

---

### Task 8: Refresh CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update the "Current scaffold behavior" and "Commands"/"CI" sections**

The `CLAUDE.md` committed earlier this session describes the *pre-migration* scaffold (`mattermost-server/v6`, CircleCI-only, the root `plugin.go` go:embed pattern). After this plan, that's stale. Update it to describe:
- `server/` now uses `mattermost/mattermost/server/public`, `gorilla/mux` routing (`api.go`), `pluginapi.Client`, a slash-command scaffold (`server/command/`), a KV store scaffold (`server/store/kvstore/`), and a scheduled background job (`job.go`).
- The root-level `plugin.go` (go:embed manifest) no longer exists — manifest handling is entirely `webapp/src/manifest.ts`, generated by `make apply`.
- CI is `.github/workflows/ci.yml` delegating to the official reusable workflow (same wording pattern as gpx-preview's `CLAUDE.md` CI section).
- `make check-style`/`make test` now actually exercise golangci-lint and `go test ./server/...` (this repo has `HAS_SERVER` set, unlike gpx-preview) — call this out explicitly since it's a real difference from the other plugin in this workspace.

Keep the "Notes for future work" section's guidance about not renaming identity fields — that's still true and still relevant.

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "$(cat <<'EOF'
docs: update CLAUDE.md after starter-template resync

Describes the post-migration scaffold: mattermost/server/public APIs,
gorilla/mux routing, the command/kvstore/job scaffold additions, the
removed root plugin.go go:embed pattern, and the new CI workflow.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

## Self-review notes

- **Spec coverage:** build tooling (Task 2), server scaffold replacement (Task 3), `plugin.json` version bump (Task 4), CI (Task 5), webapp scaffold replacement (Task 6), end-to-end validation (Task 7), docs (Task 8).
- **Deliberately out of scope:** renaming `plugin.json`'s identity fields or the Go module path (flagged in "Key verified facts" — same precedent as the gpx-preview migration); merging the unmerged `chore/dependabot-config` branch (user asked about it separately, hasn't decided); building on the untested `ci/pr-checks` branch (superseded, not merged from).
- **Biggest real risk:** Task 3's Go API migration (`mattermost-server/v6` → `mattermost/mattermost/server/public`) is new territory this plan didn't need for gpx-preview. Mitigated by wholesale-copying upstream's own maintained scaffold rather than hand-translating the old API calls — there's no custom logic to mistranslate.
