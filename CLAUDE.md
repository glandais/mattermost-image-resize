# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**This is still an unmodified `mattermost-plugin-starter-template` scaffold, not yet a working image-resize plugin** — only the build tooling, CI, server framework, and webapp framework have been refreshed to match the current upstream template (see "Resync history" below); no actual image-resizing feature exists yet. `plugin.json`'s `id` is still `com.mattermost.plugin-starter-template`, `go.mod`'s module path is still `github.com/mattermost/mattermost-plugin-starter-template`, and the README is the template's own generic "how to use this template" doc. Treat this repo as a greenfield starter when reasoning about what exists today — don't assume any image-resize behavior is implemented unless you've actually found it in the diff/commits you're working from.

Both `server/` (Go) and `webapp/` (TypeScript/React) are present and active (`plugin.json` declares both a `server.executables` block and a `webapp.bundle_path`), unlike `mattermost-plugin-gpx-preview` elsewhere in this workspace, which is webapp-only. That matters for tooling: `make check-style` here actually runs `golangci-lint` (gpx-preview's doesn't, since it has no `server/`), and `make test` here runs Go tests via `gotestsum` in addition to the webapp's `npm test`.

## Current scaffold behavior (until real feature work replaces it)

- `server/plugin.go` — `Plugin` struct holds a `gorilla/mux` `router`, a `pluginapi.Client`, a `command.Command` handler, a `kvstore.KVStore`, and a scheduled `backgroundJob`. `OnActivate` wires all of these up; `ExecuteCommand` delegates to the command handler.
- `server/api.go` — sets up the HTTP router: a `MattermostAuthorizationRequired` middleware (rejects requests with no `Mattermost-User-ID` header) wrapping a `GET /api/v1/hello` route that returns `"Hello, world!"`.
- `server/command/command.go` — registers a `/hello [@username]` slash command; `server/command/mocks/` has a generated `gomock` mock of the `Command` interface for tests.
- `server/store/kvstore/kvstore.go` — `KVStore` interface with one sample method (`GetTemplateData`), backed by `pluginapi`'s KV client in `startertemplate.go`.
- `server/job.go` — a `cluster.Schedule`d background job (hourly) that currently just logs that it ran.
- `server/configuration.go` — empty `configuration` struct (no fields) with the standard lock-guarded `getConfiguration`/`setConfiguration`/`OnConfigurationChange` pattern. Add real settings fields here when the plugin needs configuration, and update `plugin.json`'s `settings_schema.settings` (currently `[]`) to match.
- `server/main.go` — entry point, `plugin.ClientMain(&Plugin{})`. Uses `github.com/mattermost/mattermost/server/public/plugin`, not the old `mattermost-server/v6` API — see "Resync history".
- `webapp/src/index.tsx` — `Plugin.initialize()` is an empty async stub; no UI is registered with the Mattermost webapp plugin registry yet.
- `webapp/src/manifest.ts` — **generated, not committed** (gitignored). Produced by `build/bin/manifest apply` (run via `make apply`) from `plugin.json`.
- `webapp/src/types/mattermost-webapp/index.d.ts` — full ambient typings for `PluginRegistry`, synced verbatim from upstream (not hand-maintained here).

There is no longer a root-level `plugin.go` (it used to `go:embed plugin.json` into a `model.Manifest` — that pattern was dropped upstream; manifest handling now lives entirely in the generated `webapp/src/manifest.ts` plus the `build/bin/manifest` CLI used by the Makefile).

## Commands

Same `make`-based workflow as the upstream starter template:
- `make check-style` — golangci-lint (server) + eslint/tsc (webapp) — both actually run here, unlike gpx-preview
- `make test` — `gotestsum -- -v ./...` (server + build tooling) + `npm test` in `webapp/`
- `make dist` — builds both sides for all platforms (`linux/darwin` amd64+arm64, `windows-amd64`), bundles into `dist/<plugin-id>-<version>.tar.gz`
- `make deploy` — build and push to a running Mattermost server (`MM_SERVICESETTINGS_SITEURL` + local-mode socket or admin credentials)
- `make watch` — rebuild webapp on change and deploy automatically
- `make apply` — regenerates `webapp/src/manifest.ts` from `plugin.json` (gitignored, not committed)

Webapp-only commands run from `webapp/`: `npm run lint`, `npm run check-types` (tsc, no emit), `npm test` (jest), `npm run build`.

## CI

`.github/workflows/ci.yml` delegates entirely to Mattermost's official reusable workflow
(`mattermost/actions-workflows/.github/workflows/plugin-ci.yml@main`), which runs
`make check-style`, `make test`, and `make dist`. Node and Go versions come from
`.nvmrc` and `go.mod` respectively. The workflow's `delivery` job (S3 upload) only runs
when `github.repository_owner == 'mattermost'`, so it's a no-op on this fork. There is no
git-based `mattermost-webapp` devDependency, so `npm ci`/`npm install` never needs git
credentials. (An unmerged `chore/dependabot-config` branch and an unmerged, now-superseded
draft `ci/pr-checks` branch also exist — neither has been merged into this work.)

## Resync history

This repo was synced from a much older snapshot of `mattermost-plugin-starter-template`
(Go 1.16, `mattermost-server/v6`, React 16, webpack 4) up to the current template
(Go 1.25, `mattermost/mattermost/server/public`, React 18, webpack 5). Because this repo
had no custom feature code at the time (everything was still the template's own
placeholder scaffold), that migration replaced `server/`, `webapp/`, and the build
tooling **wholesale** with upstream's current versions rather than adapting custom logic
file-by-file. The full task-by-task plan is in
`docs/superpowers/plans/2026-07-01-sync-starter-template.md`.

The Go server API changed meaningfully: `mattermost-server/v6` → `mattermost/mattermost/server/public`,
plus the addition of a `pluginapi.Client` wrapper, `gorilla/mux` routing, a slash-command
scaffold, a KV store scaffold, and a scheduled background job — none of which existed
before. If you're debugging something that looks like a server API mismatch, check
which Mattermost Go module is actually imported (`go.mod` should only reference
`github.com/mattermost/mattermost/server/public`, not `mattermost-server/v6`).

## Notes for future work in this repo

- Don't rename `plugin.json`'s `id`/`name`/`description`/`homepage_url`/`support_url`/`release_notes_url` or the Go module path as a side effect of unrelated changes (build tooling sync, dependency bumps, etc.) — that's a deliberate branding decision the user should make explicitly, not something to fold into incidental work. `make dist` prints a warning about the still-default plugin ID; that's expected and not a bug to silently fix.
- Once real image-resize logic lands in `server/`/`webapp/src/`, this file's "Current scaffold behavior" section will go stale — update it to describe the actual feature rather than the placeholder scaffold.
