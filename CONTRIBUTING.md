# Contributing to SwiftWeather-Assets

This repository hosts the remote assets consumed by the Skycast iOS
app. Two artifacts are public-API for the app:

- `manifest.json` — Skycast fetches this at runtime to enumerate
  downloadable WeatherWalls sets. Schema is documented in
  `SwiftWeather/Services/WeatherWallsManifest.swift`.
- `shortcut-version.json` — version pointer for the bundled Skycast
  Shortcut. Format: see `RELEASE_PROCESS.md`.

## Locale-completeness contract

`manifest.json` carries two locale-keyed dictionaries per entry:
`description` and (optional) `changelog`. Skycast ships UI in
**eleven** locales as of 2026-05-14 (Phase 1 Tier-1 rollout):

| Code      | Language                            |
|-----------|-------------------------------------|
| `en`      | English (terminal fallback)         |
| `de`      | German                              |
| `es`      | Spanish                             |
| `fr`      | French                              |
| `it`      | Italian                             |
| `ja`      | Japanese                            |
| `pt-BR`   | Brazilian Portuguese                |
| `pt-PT`   | Continental Portuguese              |
| `ru`      | Russian                             |
| `zh-Hans` | Simplified Chinese                  |
| `zh-Hant` | Traditional Chinese                 |

The canonical SoT lives in
`SwiftWeather/Core/Helpers/AppLocales.swift` in the Skycast main repo.
Any future locale addition is reflected there first; this manifest
must then be extended to match.

**Hard rule (enforced by app-side CI fence
`LiveManifestLocaleCompletenessIntegrationTests`):** Every
`description` dictionary MUST have keys for ALL eleven locales.
Every `changelog` dictionary (when present) MUST also cover all
eleven. Missing-locale entries silently degrade users to the English
fallback — bad UX, considered a defect. Empty-string values are
valid (signals "intentionally blank" per the
`LocalizedDictionary` contract); the fence checks key presence, not
value content.

## Adding a Phase-N locale (cross-repo extension)

When Skycast's `AppLocales.supported` grows, this manifest must
extend in lock-step. The canonical tool is
`scripts/ensure_locale_coverage.py` in the Skycast main repo:

```bash
# From the Skycast main repo, with this assets repo cloned to /tmp/SwiftWeather-Assets:
python3 scripts/ensure_locale_coverage.py \
    --input /tmp/SwiftWeather-Assets/manifest.json \
    --locales-from-swift

# Or with explicit locale list (works without the Swift source):
python3 scripts/ensure_locale_coverage.py \
    --input /tmp/SwiftWeather-Assets/manifest.json \
    --locales de en es fr it ja pt-BR pt-PT ru zh-Hans zh-Hant
```

The tool is idempotent + verify-mode-aware. CI integration:

```bash
# Verify-only: exit 1 if any extension would be required (for pre-commit / CI)
python3 .../ensure_locale_coverage.py --input manifest.json --verify --locales ...
```

The Phase-1 cross-repo extension (84 keys across 6 entries × 2 fields
× 7 new locales) was applied via this tool on 2026-05-14.

## Adding a new WeatherWalls set

1. Prepare the asset ZIP per the format documented in
   `SwiftWeather/Core/Storage/IconSetStorage.swift` (OW30 filenames
   `{1-14,50,51}{d|n}.png` for `weatherWalls` type; index-suffixed
   `{ow30code}{d|n}_{index}.png` for `randomWeatherWalls` type)
2. Create a GitHub Release with the ZIP attached as an asset
3. Add a new entry to `manifest.json`'s `sets` array with:
   - `id` — folder name post-extract (case-sensitive)
   - `type` — `"weatherWalls"` OR `"randomWeatherWalls"`
   - `name` — user-facing display name
   - `description` — **all eleven locales required** (see contract above; use
     `ensure_locale_coverage.py` to auto-fill empty placeholders)
   - `version` — semver string
   - `releaseTag` — matches the GitHub Release tag exactly
   - `assetName` — matches the asset filename exactly
   - `sizeMB` — integer MB (decimal, rounded)
   - `minAppVersion` — null unless the entry requires a specific app build
   - `changelog` — optional but if present, **all eleven locales required**
4. Bump `lastUpdated` to current UTC ISO 8601 timestamp
5. Commit + push to `main` — the app will pick up the new entry on
   the next `WeatherWallsDownloader.fetchManifest()` call (typically
   at the next app launch or pull-to-refresh in Settings)

## JSON formatting

- 2-space indent
- UTF-8 without BOM
- Final newline at EOF
- Preserve key order: `id, type, name, description, version, releaseTag, assetName, sizeMB, minAppVersion, changelog`
- Inside `description` / `changelog` dicts: prefer `en, de, es, fr, it, ja, pt-BR, pt-PT, ru, zh-Hans, zh-Hant`
  ordering (English-first for translator readability, then by Skycast's
  `AppLocales.supported` order). The
  `ensure_locale_coverage.py` tool's append-order respects this.
