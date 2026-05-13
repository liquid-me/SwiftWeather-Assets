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
`description` and (optional) `changelog`. Skycast ships UI in **four**
locales: **English (en), German (de), Spanish (es), French (fr)**.

**Hard rule (enforced by app-side CI fence):** Every `description`
dictionary MUST have keys for ALL FOUR locales. Every `changelog`
dictionary (when present) MUST also cover all four. Missing-locale
entries degrade ES/FR users to the English fallback — silently —
which is bad UX and considered a defect.

Adding a 5th language to Skycast triggers a coordinated update:

1. Add the locale to `AppLocales.supported` in the app (`Core/Helpers/AppLocales.swift`)
2. Add the language code to `knownRegions` in `project.pbxproj`
3. Add the `<lang>.lproj/` directories with `Localizable.strings` + `InfoPlist.strings`
4. **Update every `description` and `changelog` in this manifest** to include the new locale
5. The app's `BundledCatalogLocaleCompletenessFenceTests` +
   `LiveManifestLocaleCompletenessIntegrationTests` will catch
   incomplete entries at CI time

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
   - `description` — **all four locales required** (see contract above)
   - `version` — semver string
   - `releaseTag` — matches the GitHub Release tag exactly
   - `assetName` — matches the asset filename exactly
   - `sizeMB` — integer MB (decimal, rounded)
   - `minAppVersion` — null unless the entry requires a specific app build
   - `changelog` — optional but if present, **all four locales required**
4. Bump `lastUpdated` to current UTC ISO 8601 timestamp
5. Commit + push to `main` — the app will pick up the new entry on
   the next `WeatherWallsDownloader.fetchManifest()` call (typically
   at the next app launch or pull-to-refresh in Settings)

## JSON formatting

- 2-space indent
- UTF-8 without BOM
- Final newline at EOF
- Preserve key order: `id, type, name, description, version, releaseTag, assetName, sizeMB, minAppVersion, changelog`
- Inside `description` / `changelog` dicts: `en, de, es, fr` (alphabetical-after-en convention)
