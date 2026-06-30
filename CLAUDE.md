# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**As of now, this is an unmodified `mattermost-plugin-starter-template` scaffold, not yet a working image-resize plugin.** Despite the repo's name, there is no image-resizing code anywhere: `plugin.json`'s `id` is still `com.mattermost.plugin-starter-template`, `go.mod`'s module path is still `github.com/mattermost/mattermost-plugin-starter-template`, the README is the template's own generic "how to use this template" doc, and the git history is a single "Initial commit". Treat this repo as a greenfield starter when reasoning about what exists today — don't assume any image-resize behavior is implemented unless you've actually found it in the diff/commits you're working from.

Both `server/` (Go) and `webapp/` (TypeScript/React) are present and active (`plugin.json` declares both a `server.executables` block and a `webapp.bundle_path`), unlike some other Mattermost plugins in this workspace that are webapp-only.

## Current scaffold behavior (until real feature work replaces it)

- `server/plugin.go` — `Plugin` struct embeds `plugin.MattermostPlugin`; `ServeHTTP` just writes `"Hello, world!"` for any request — no real HTTP handlers yet.
- `server/configuration.go` — empty `configuration` struct (no fields) with the standard lock-guarded `getConfiguration`/`setConfiguration`/`OnConfigurationChange` pattern. Add real settings fields here when the plugin needs configuration, and update `plugin.json`'s `settings_schema.settings` (currently `[]`) to match.
- `server/main.go` — entry point, `plugin.ClientMain(&Plugin{})`.
- `plugin.go` (repo root, package `root`) — `go:embed`s `plugin.json` into a `model.Manifest`. This is a different mechanism from the webapp side's generated `webapp/src/manifest.ts` (see below) — both exist independently and both read from `plugin.json`.
- `webapp/src/index.tsx` — `Plugin.initialize()` is an empty async stub; no UI is registered with the Mattermost webapp plugin registry yet.
- `webapp/src/types/mattermost-webapp/index.d.ts` — minimal local typings, currently just a `PluginRegistry` interface with one stub method (`registerPostTypeComponent`). Extend this (or sync the fuller upstream version) when registering real UI components.

## Commands

Same `make`-based workflow as the upstream starter template:
- `make check-style` — golangci-lint (server) + eslint/tsc (webapp)
- `make test` — `go test ./server/...` + `npm test` in `webapp/`
- `make dist` — builds both sides, bundles into `dist/<plugin-id>-<version>.tar.gz`
- `make deploy` — build and push to a running Mattermost server (`MM_SERVICESETTINGS_SITEURL` + local-mode socket or admin credentials)
- `make watch` — rebuild webapp on change and deploy automatically

Webapp-only commands run from `webapp/`: `npm run lint`, `npm run check-types` (tsc, no emit), `npm test` (jest), `npm run build`.

## CI

`.circleci/config.yml` (orb `mattermost/plugin-ci@0.1.6`) is the only CI config present today — there's no `.github/workflows/` yet. A `ci/pr-checks` branch exists with a single untested draft GitHub Actions workflow; treat it as a first draft, not a working baseline, until it's actually been exercised by a real PR.

## Notes for future work in this repo

- Don't rename `plugin.json`'s `id`/`name`/`description` or the Go module path as a side effect of unrelated changes (build tooling sync, dependency bumps, etc.) — that's a deliberate branding decision the user should make explicitly, not something to fold into incidental work.
- Because this is still a stock scaffold, a "sync with upstream starter-template" task here is closer to "replace the scaffold with a newer version of itself" than to adapting custom logic — there's no feature code that needs careful preservation yet. That changes once real image-resize logic lands in `server/`/`webapp/src/`.
