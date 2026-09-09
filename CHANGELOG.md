# Changelog

All notable changes to this marketplace are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this marketplace adheres to [Semantic Versioning](https://semver.org/).
The version below is the marketplace's own `metadata.version` in
[`marketplace.json`](.claude-plugin/marketplace.json) — bumped **minor** when
a plugin is added, **patch** for any other change (renames, source
swaps, or a listed plugin's own version bump).

## [1.4.1] - 2026-09-08

### Removed
- **Per-plugin `version` fields**, from all five entries in
  [`marketplace.json`](.claude-plugin/marketplace.json). Each plugin's version is owned by
  its own `.claude-plugin/plugin.json`, and Claude Code reads it from there: when both
  places declare a version, `plugin.json` wins **without warning**, so the copies here
  could only ever mask the real value. Four of the five had already drifted — this
  marketplace listed `ono-project-inspector` at 0.7.0 (actual 0.9.0),
  `ono-mobile-dev-plugin` at 0.2.0 (actual 0.5.0), `ono-plugin-qa` at 0.3.0 (actual
  0.7.0), and `spec-team-toolkit` at 0.1.0 (actual 0.2.0). Nothing detected it, because a
  marketplace version that loses to `plugin.json` has no observable effect. Omitting the
  field also restores update-on-`ref`-change for these git sources.
- The **Version column** from the README's *Available plugins* table, and the sentence
  claiming those numbers tracked the manifest, for the same reason: both could only ever
  be stale copies. The README now says where to read a version instead.

### Changed
- `ono-mobile-dev-plugin`'s marketplace description, to match its own `plugin.json`: the
  single "dev planning" stage became "detailed design, task breakdown" when that plugin
  split `/create-dev-plan` into `/dev-design-start` and `/dev-feature-start`.
- `ono-plugin-qa`'s marketplace description, to match its own `plugin.json` at 0.7.0,
  which added Appium automation generation and live locator verification.
- The README's `ono-mobile-dev-plugin` command table: removed `/create-dev-plan`, which no
  longer exists, and added `/dev-design-start` and `/dev-feature-start`. Corrected
  "seven-stage" to "eight-stage" to match the plugin's current pipeline.
- *How it's wired* now records the metadata ownership boundary: this manifest owns `name`,
  `source` and its own `metadata.version`; each plugin owns everything else, its `version`
  above all. Marketplace-facing `description` stays a deliberate,
  hand-maintained summary of the plugin's own.

## [1.4.0] - 2026-07-20

### Added
- `matrix-studio` — Figma design-system generator plugin: color palette,
  spacing/scale tokens, and a two-platform typography system as Figma
  Variables and text styles via the Figma MCP.

## [1.3.3] - 2026-07-08

### Changed
- Reworked the README so users can find and understand every listed plugin:
  added the previously-missing `spec-team-toolkit` entry, a "which plugin do
  I need?" quick-pick table, and a per-plugin command/skill breakdown.

## [1.3.2] - 2026-07-07

### Changed
- Renamed `ono-plugin-QA-ReactNative` to `ono-plugin-qa` (source repo,
  description, and version updated to reflect broader QA scope across
  React, React Native, iOS, and Android).

## [1.3.1] - 2026-07-07

### Changed
- Renamed `ono-react-native-dev-plugin` to `ono-mobile-dev-plugin` (source
  repo and description updated to reflect its broader mobile scope).

## [1.3.0] - 2026-07-07

### Added
- `spec-team-toolkit` — HLD and LLD document builders for the spec team.

## [1.2.0] - 2026-07-06

### Added
- `ono-plugin-QA-ReactNative` — Figma-grounded QA test planning and dev/QA
  coverage gap analysis for React Native features.

## [1.1.0] - 2026-07-05

### Added
- `ono-react-native-dev-plugin` — React Native SDLC workflow plugin (feature
  analysis, dev planning, implementation, code review, debugging, QA
  handoff, release readiness).

## [1.0.7] - 2026-07-05

### Changed
- Renamed the marketplace identifier from `ono-claude-marketplace` to
  `ono-plugin-marketplace` and updated the `ono-project-inspector` source URL
  and README instructions to match.

## [1.0.6] - 2026-07-04

### Changed
- Bumped `ono-project-inspector` to `0.7.0`.

## [1.0.5] - 2026-07-04

### Changed
- Bumped `ono-project-inspector` to `0.6.0`.

## [1.0.4] - 2026-07-04

### Changed
- Switched `ono-project-inspector`'s source from a `github` source to an
  HTTPS `url` source so installs work on any machine without SSH keys
  configured.

## [1.0.3] - 2026-07-04

### Changed
- Bumped `ono-project-inspector` to `0.5.0`.

## [1.0.2] - 2026-07-03

### Changed
- Bumped `ono-project-inspector` to `0.4.0`.

## [1.0.1] - 2026-07-03

### Changed
- Switched `ono-project-inspector` from a local symlink to a GitHub source.

## [1.0.0] - 2026-07-02

### Added
- Initial marketplace with the `ono-project-inspector` plugin (referenced via
  a local symlink).
