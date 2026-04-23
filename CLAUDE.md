# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Dockerfile + GitHub Actions pipeline that layers TimescaleDB and TimescaleDB Toolkit onto the upstream `ghcr.io/cloudnative-pg/postgresql` image and publishes it to `ghcr.io/clevyr/cloudnativepg-timescale`. There is no application code, no tests, and no local build scripts — the source of truth is the workflow.

## How versions are resolved

Versions are **not pinned** in this repo. The `Resolve latest versions` step in `.github/workflows/build.yaml` fetches them at build time:

- TimescaleDB + Toolkit: GitHub `releases/latest` of `timescale/timescaledb` and `timescale/timescaledb-toolkit` (leading `v` stripped).
- PostgreSQL / CNPG: `ghcr.io/v2/cloudnative-pg/postgresql/tags/list`, filtered to `^16\.[0-9]+$` and `sort -V | tail -1`. Only Postgres 16 is tracked — to add another major, add a parallel filter or matrix, don't change `16` in place.

`POSTGRES_VERSION` and `CLOUDNATIVEPG_VERSION` are fed the **same** resolved value (the CNPG tag like `16.11`). The Dockerfile uses `$POSTGRES_VERSION` inside the Timescale apt package name, so this relies on Timescale's Debian repo accepting the full `major.minor` (or on apt's loose matching for the major). If a scheduled build starts failing on `apt-get install` of `timescaledb-2-postgresql-<ver>`, verify the packagecloud triplet first — upstream may not have published yet for a brand-new CNPG minor.

Renovate (`renovate.json`) no longer drives these versions; its regex managers now match nothing in `build.yaml` but are left in place for future use. Don't reintroduce the old Renovate comment pragmas — they'd fight the runtime resolver.

## Build cadence and triggers

`build.yaml` runs on `push`, on `workflow_dispatch` (manual), and weekly via `schedule: '0 0 * * 0'` (Sun 00:00 UTC). Only runs on `main` actually push images (`push: ${{ github.ref_name == 'main' }}`); branch pushes are validation-only. Because versions resolve at runtime, two runs minutes apart can produce different images if upstream cut a release in between — that's intentional.

## Build contract

`Dockerfile` consumes four build args: `CLOUDNATIVEPG_VERSION`, `POSTGRES_VERSION`, `TIMESCALE_VERSION`, `TIMESCALE_TOOLKIT_VERSION`. It installs `timescaledb-2-postgresql-$POSTGRES_VERSION=$TIMESCALE_VERSION~debian$VERSION_ID` and the Toolkit as `=1:$TIMESCALE_TOOLKIT_VERSION~debian$VERSION_ID`, so upstream has to have published a matching `~debianNN` package before a build will succeed.

The image ends with `USER 26` (the `postgres` user id CNPG expects). Do not add layers after that as root without resetting the user.

## Tag scheme

`docker/metadata-action` emits seven tags per build, in priority order: `latest`, `<pg>-ts<ts>` (full), `<pg-minor>-ts<ts-minor>`, `<pg-major>-ts<ts-major>`, then the same three without the `-ts…` suffix. The README example `16-ts2` is the major/major form. Toolkit version does not appear in any tag.

## Local iteration

`docker build --build-arg ...` against the four args above; no scheduled/remote infra needed. Branches are safe to experiment on — pushes are gated to `main`.
