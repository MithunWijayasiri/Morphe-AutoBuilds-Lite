# Operations runbook

How-tos and pipeline/release detail. Field formats live in `config-reference.md`.

## Add a new app

1. Create `apps/<platform>/<app>.json` for at least one store (see config-reference → `apps/`).
2. Optionally create `patches/<app>-<source>.txt` for include/exclude (may be empty).
3. Add `{ "app_name": "<app>", "source": "<source>" }` to `patch-config.json`.
4. Optionally add an `arch-config.json` override (default is `universal` only).

`source` must already exist as `sources/<source>.json` with exact casing.

## Fix a failing build

1. Find the failing `(app, source)` in the build-apps matrix logs.
2. **`source` not found** → check `sources/<source>.json` exists with exact casing (`Hoo.json` ≠ `hoo.json` on the Linux runner).
3. **Patch failure** (fingerprint mismatch, "patching aborted") → the app version is incompatible with current patches. The build already auto-retries older candidate versions (`_should_retry_with_older_version`); if all fail, the patch source likely needs a newer release — bump `tag` in `sources/<source>.json` or wait for upstream.
4. **No download** → all four stores failed for that package; verify `package`/`name` fields in the app config.
5. Empty `patches/<app>-<source>.txt` is **not** the cause — empty means "all patches".

## Update a patch source

Edit `sources/<source>.json` (`tag`, `user`/`repo`/`project`, or `bundle_url`). Next run, `check_app_updates.py` detects the new source signature and rebuilds every app on that source.

## Trigger / force a build

- **Daily**: cron `0 6 * * *` (06:00 UTC) on `patch.yml`.
- **Force full rebuild**: `patch.yml` → Run workflow → `force_full_rebuild = true` (sets `FORCE_FULL_REBUILD`, ignores incremental check).
- **Single app**: `manual-patch.yml` → Run workflow (app, source, arch, optional pinned version, replace-in-release toggle). Pins version into the app config for the run, then resets it.

## Pipeline stages (`patch.yml`)

```
download-tools → check-updates → patch-apps → build-apps (matrix) → create-single-release → cleanup
```

- **download-tools**: pre-fetches ReVanced CLI/patches/integrations into `tools/`; cached on `hashFiles('patch-config.json', 'arch-config.json')`.
- **check-updates**: runs the incremental planner; emits `build_matrix.json`, `carry_over.json`, `new_manifest.json` (build-plan artifact). Skips the rest when `has_updates=false`.
- **patch-apps**: reads `build_matrix` into the job matrix (falls back to full `patch-config.json` if empty).
- **build-apps**: `fail-fast: false` matrix, one cell per `(app, source)`; runs `python -m src`, then `record_build.py` writes a per-APK record (`build_records/*.json`). Uploads APK + record artifacts.
- **create-single-release**: gathers rebuilt APKs + records, `merge_manifest.py` folds records into `new_manifest.json` → `manifest.json`, uploads APKs + manifest to `latest` via `gh release upload --clobber` (creates `latest` if missing). Carried-over APKs already on the release stay in place.
- **cleanup**: `delete-workflow-runs` keeps 3 days / min 5 runs.

Live scripts: `check_app_updates.py`, `record_build.py`, `merge_manifest.py`, `validate_github_auth.py`.

## Release / manifest strategy

- **Single rolling `latest` tag.** No per-day tags. Assets updated in place daily.
- `manifest.json` (release asset) is the incremental state of record — `(app, source, arch)` → `{config_version, source_sig, apk}`.
- Rebuild triggers per entry: app `version` changed, source signature changed, APK missing from the release, or no APK recorded.
- Build matrix is deduped to `(app, source)` — if any arch of a pair needs a rebuild, the whole pair rebuilds (one `python -m src` run emits all its arches) and its stale carry-overs are dropped.
- Total payload is large (~8 GB / 100+ APKs) — watch GitHub storage limits.
