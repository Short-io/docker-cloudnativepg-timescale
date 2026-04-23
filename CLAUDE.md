# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Dockerfile + GitHub Actions pipeline that layers TimescaleDB and TimescaleDB Toolkit onto the upstream `ghcr.io/cloudnative-pg/postgresql` image and publishes it to `ghcr.io/clevyr/cloudnativepg-timescale`. There is no application code, no tests, and no local build scripts — the source of truth is the workflow.

## How versions are resolved

Versions are **not pinned** in this repo. The `Resolve latest versions` step in `.github/workflows/build.yaml` fetches them at build time:

- TimescaleDB + Toolkit: GitHub `releases/latest` of `timescale/timescaledb` and `timescale/timescaledb-toolkit` (leading `v` stripped).
- PostgreSQL: `https://www.postgresql.org/versions.json` is queried for `latestMinor` of PG major `16` (`PG_MAJOR` env in the step). We then HEAD-probe `ghcr.io/v2/cloudnative-pg/postgresql/manifests/16.<minor>` and walk back one minor at a time until we get a `200` — this tolerates the lag between an upstream PG release and CNPG re-publishing. The tags/list endpoint is intentionally **not** used: it's capped at 1000 entries per page and CNPG has enough historical tags that a plain-minor tag like `16.13` can fall off the first page.

Two build args derive from different halves of the resolved version:

- `CLOUDNATIVEPG_VERSION` = the full probed tag (e.g. `16.13`) — used as the `FROM` tag.
- `POSTGRES_VERSION` = the PG major only (e.g. `16`) — used inside the Timescale apt package name `timescaledb-2-postgresql-<major>`. Passing the full `16.13` here will fail (`E: Unable to locate package timescaledb-2-postgresql-16.13`) — Timescale's Debian packages are keyed on the PG major, not the minor.

Changing the tracked PG major means updating `PG_MAJOR` in the resolver step; there is no matrix anymore.

Renovate (`renovate.json`) no longer drives these versions; its regex managers now match nothing in `build.yaml` but are left in place for future use. Don't reintroduce the old Renovate comment pragmas — they'd fight the runtime resolver.

## Build cadence and triggers

`build.yaml` runs on `push`, on `workflow_dispatch` (manual), and weekly via `schedule: '0 0 * * 0'` (Sun 00:00 UTC). Only runs on `main` actually push images (`push: ${{ github.ref_name == 'main' }}`); branch pushes are validation-only. Because versions resolve at runtime, two runs minutes apart can produce different images if upstream cut a release in between — that's intentional.

## Build contract

`Dockerfile` consumes four build args: `CLOUDNATIVEPG_VERSION`, `POSTGRES_VERSION`, `TIMESCALE_VERSION`, `TIMESCALE_TOOLKIT_VERSION`. The Toolkit installs as `=1:$TIMESCALE_TOOLKIT_VERSION~debian$VERSION_ID`.

TimescaleDB's package version is **resolved via `apt-cache madison`** at build time rather than pinned as `$TIMESCALE_VERSION~debian$VERSION_ID`. Starting with TimescaleDB 2.23.1, Timescale's Debian packages embed the target PG minor as a trailing suffix (e.g. `2.26.3~debian11-1613` for PG 16.13); the plain `~debianNN` form is no longer published. The resolver accepts either the plain form or the first `~debianNN-…` match, so the build works whether upstream keeps the new scheme or reverts.

The image ends with `USER 26` (the `postgres` user id CNPG expects). Do not add layers after that as root without resetting the user.

## Tag scheme

`docker/metadata-action` emits seven tags per build, in priority order: `latest`, `<pg>-ts<ts>` (full), `<pg-minor>-ts<ts-minor>`, `<pg-major>-ts<ts-major>`, then the same three without the `-ts…` suffix. The README example `16-ts2` is the major/major form. Toolkit version does not appear in any tag.

## Local iteration

`docker build --build-arg ...` against the four args above; no scheduled/remote infra needed. Branches are safe to experiment on — pushes are gated to `main`.
