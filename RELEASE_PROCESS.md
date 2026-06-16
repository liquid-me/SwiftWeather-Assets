# Release Process

This repository ships two runtime artifacts that Skycast fetches by raw GitHub
URL, each with its own release ritual:

- **Skycast Shortcut** — `shortcut-version.json` (the install-version pointer).
- **WeatherWalls / RandomWeatherWalls sets** — `manifest.json` + release ZIPs.

## Skycast Shortcut

`shortcut-version.json` is the version pointer Skycast fetches at runtime
(`ShortcutManager.fetchRemoteVersion()` reads the raw URL below) to drive the
**Settings → Kurzbefehl** status pill and its "Update available" nudge. Bump it
whenever you publish a new revision of the Skycast Shortcut so existing users
are told a new version exists.

```
https://raw.githubusercontent.com/liquid-me/SwiftWeather-Assets/main/shortcut-version.json
```

### Format

A flat JSON object — `ShortcutManager.parseRemoteVersion` is the app-side parser:

| Field         | Type         | Req. | Meaning |
|---------------|--------------|------|---------|
| `version`     | **integer**  | yes  | Monotonically increasing. The parser **rejects the whole file** if this is missing or not an integer (`2.0` and `"2"` are NOT integers → no pill, no nudge). |
| `note`        | string       | no   | Short release note shown under the install button. Defaults to empty. |
| `icloud_link` | string (URL) | no   | iCloud `…/shortcuts/<id>` install link. **Omit unless rotating the link** — when absent the app uses its built-in `fallbackICloudLink` (`ShortcutManager.swift`). Set it only to point users at a *different* iCloud share. |

Example (current shipping form):

```json
{"version": 2, "note": "Shortcut v.1.2"}
```

### How the "Update available" nudge works

iOS exposes **no** API to query whether a Shortcut is installed or its version,
so the app can't know ground truth. It only knows the latest `version` (this
file) and a best-effort marker of the version the user last *tapped install for*
(`installedShortcutVersion`, written at tap time). The nudge fires when
`version > installedShortcutVersion`. So: **+1 the `version` on every new
shortcut revision** → every prior installer sees "Update available"; on their
next install-tap the marker catches up and the pill goes neutral again. Users
who never installed (`marker = 0`) just see a neutral "v{version}" pill, never a
nudge — correct, since "update" doesn't apply to them.

### Bump procedure

```bash
# 1. Edit shortcut-version.json: increment `version` by 1, update `note`.
#    The LIVE file is authoritative for the current value (2 at this writing,
#    so the next bump is 3). Only add `icloud_link` if you are rotating the link.

# 2. Validate: JSON parses AND `version` is an integer (mirrors the app parser).
python3 -c 'import json; d=json.load(open("shortcut-version.json")); assert isinstance(d["version"], int), "version must be an integer"; print("OK", d)'

# 3. Commit + push to main.
git add shortcut-version.json && git commit -m "chore(shortcut): bump version N->N+1" && git push origin main

# 4. Verify the LIVE file — this is exactly what the app fetches.
curl -s "https://raw.githubusercontent.com/liquid-me/SwiftWeather-Assets/main/shortcut-version.json"
```

UTF-8 without BOM, final newline at EOF (same convention as `manifest.json`).

## WeatherWalls sets

How to add a new WeatherWalls set to the Skycast wizard. Follow this exactly — Skycast's `WeatherWallsDownloader` is manifest-driven, so app code never needs to change for new sets, but the ZIP layout and manifest contract are load-bearing.

### ZIP layout invariant

**Every WeatherWalls / RandomWeatherWalls ZIP must have its content files at the top level — no wrapper folder.**

```
✓ correct                           ✗ wrong
1d_1.png                            05_WeatherWallLib/
1n_1.png                            05_WeatherWallLib/1d_1.png
2d_1.png                            05_WeatherWallLib/1n_1.png
…                                   …
```

Reference: `01_random_portrait.zip` (the canonical layout — PNGs directly at the archive root).

#### Why this matters

Skycast's `ZIPExtractor.detectSingleTopLevelFolder` strips a single wrapper folder when it detects one, so a wrapper-form ZIP also "works" in the field. But the unwrapped form is canonical and what every other set in this repo follows. Mixing forms creates a maintenance burden and makes it harder to audit ZIP contents at a glance.

The 2026-05-05 audit caught a `05_WeatherWallLib.zip` shipped with a wrapper; it was rebuilt to the canonical form for consistency.

#### Correct build command

From inside the source folder containing the PNGs:

```bash
cd <SetName>
zip -q ../<SetName>.zip *.png
```

The `cd` + glob ensures PNGs land at the archive root. **Do not** run `zip -r <SetName>.zip <SetName>/` from the parent directory — that wraps everything in a `<SetName>/` prefix.

#### Junk to exclude

macOS Finder and editor tools sprinkle metadata files into source folders. Strip these before zipping:

```bash
find <SetName> -name "._*" -delete           # AppleDouble sidecars
find <SetName> -name ".DS_Store" -delete     # Finder window state
rm -f <SetName>/00_wwalls.json               # Legacy mapping (auto-deleted on launch since 2026-04-19)
rm -f <SetName>/00_wwalls_twc.json
```

The on-device extractor handles `__MACOSX/` automatically (`ZIPExtractor.shouldSkipEntry`), but excluding it at build time is cleaner and reduces archive size.

### RandomWeatherWalls naming convention

Random sets must use the OW30-code filename pattern that `RandomWeatherWallsIndex` parses:

```
{ow30code}{d|n}_{index}.{png|jpg|jpeg}
```

- `ow30code`: one of `1`–`14`, `50`, `51` — anything outside this range is silently excluded from the random index
- `d|n`: day or night variant
- `index`: 1-based variant counter for random selection

Plus an optional `dunno_*.png` fallback for unknown weather conditions.

Examples: `5d_1.png`, `12n_27.jpg`, `51d_3.png`, `dunno_1.png`.

Files that don't match this regex are filtered out by the index — they won't crash the app, but they also won't render. Verify your set's filenames pass the pattern before releasing.

### Manifest entry

Edit `manifest.json` and insert a new entry to `sets`. Required fields:

| Field | Value |
|---|---|
| `id` | Unique identifier (matches the ZIP filename without `.zip`, e.g. `05_WeatherWallLib`) |
| `type` | `randomWeatherWalls` (random selection per condition) or `weatherWalls` (single image per condition) |
| `name` | Human-readable display name shown in the wizard |
| `description` | Object with localized descriptions; minimum keys: `en`, `de`. Skycast falls back to `en` for any missing locale (es, fr) |
| `version` | SemVer string, e.g. `1.0.0` |
| `releaseTag` | GitHub release tag — typically `<id>-v<version>` |
| `assetName` | The ZIP file name attached to the release |
| `sizeMB` | Compressed size in MB (rounded down). Run `du -h <SetName>.zip` and use the integer megabytes |
| `minAppVersion` | Set to `null` unless a specific Skycast version is required |
| `changelog` | Object with localized changelog entries; same locale convention as `description` |

Also bump the top-level `lastUpdated` field to today's date in ISO 8601 format. Skycast doesn't read this for caching but it helps human auditors track manifest changes.

### "Recommended" tag

The orange `recommended` badge that appears next to a set in the onboarding wizard is **automatically applied** to every set with `type: "randomWeatherWalls"`. There is no explicit flag — see `WeatherWallSetRow` in `SwiftWeather/Views/ContentView.swift`. To opt out, use `type: "weatherWalls"` instead.

### Step-by-step procedure

```bash
# 1. Build the canonical ZIP
cd <SetName>
find . -name "._*" -delete
find . -name ".DS_Store" -delete
zip -q ../<SetName>.zip *.png
cd ..

# 2. Verify the layout — first entry must be a PNG, not a folder
unzip -l <SetName>.zip | head -5

# 3. Verify integrity
unzip -t <SetName>.zip | tail -2

# 4. Get the size for the manifest
du -h <SetName>.zip

# 5. Create the GitHub release with the ZIP attached
gh release create <SetName>-v<version> \
  <SetName>.zip \
  --repo liquid-me/SwiftWeather-Assets \
  --target main \
  --title "<SetName> v<version> — Initial release" \
  --notes "Initial release of the <SetName> WeatherWalls set."

# 6. Edit manifest.json — insert the new sets[] entry, bump lastUpdated, commit, push.

# 7. Verify the live manifest
curl -s https://raw.githubusercontent.com/liquid-me/SwiftWeather-Assets/main/manifest.json | python3 -m json.tool
```

### Replacing a release asset

If you need to update the ZIP for an existing release without bumping the version:

```bash
gh release upload <SetName>-v<version> <SetName>.zip --repo liquid-me/SwiftWeather-Assets --clobber
```

`--clobber` overwrites the existing asset. Skycast's local cache is keyed by `releaseTag`, so users who already downloaded the old version won't auto-refresh — typical for non-breaking ZIP-cleanup operations.

### Verification on device

After publishing, kill and reopen Skycast (forces a manifest refresh), then navigate to the WeatherWalls wizard step or `Settings → WeatherWalls`. The new set should appear in the list with the correct name, size, and (for random sets) the orange `recommended` badge.

If the set doesn't appear within 30 seconds:
1. Check the manifest is valid JSON: `python3 -m json.tool manifest.json`
2. Check the asset URL is reachable: `curl -ILf https://github.com/liquid-me/SwiftWeather-Assets/releases/download/<releaseTag>/<assetName>` (expect HTTP 302→200)
3. Check Skycast's network logs (Settings → Performance → Logs) for `WeatherWallsDownloader` errors
