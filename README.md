# Ono Claude Marketplace

Internal [Claude Code](https://code.claude.com) plugin marketplace maintained by **Ono Apps**.

Add this marketplace to Claude Code once, then install any Ono plugin from it.

## Documentation

This repository is the **single source of truth for all architecture and technical documentation** across the plugin ecosystem. Each plugin repository keeps only its README and whatever is needed to build, run or contribute to that plugin.

### Ecosystem

| Document | Answers |
| --- | --- |
| **[`docs/architecture/ecosystem-overview.html`](docs/architecture/ecosystem-overview.html)** | **Start here.** How the plugins cooperate, what Repository Knowledge is, the complete Claude Code workflow, the command chain, and how information flows from one command to the next. Bilingual (English / Hebrew). |
| [`docs/specs/repo-knowledge-encapsulation.md`](docs/specs/repo-knowledge-encapsulation.md) | Why it is built this way: the ecosystem review, every duplication found, the ownership model, and the alternatives rejected. |
| [`docs/plans/repo-knowledge-mvp-implementation.md`](docs/plans/repo-knowledge-mvp-implementation.md) | What was built, task by task, with verification commands. Amended as it was executed, so it matches reality. |

### Per plugin

| Document | Answers |
| --- | --- |
| [`docs/plugins/ono-plugin-project-inspector/plugin-architecture.md`](docs/plugins/ono-plugin-project-inspector/plugin-architecture.md) | How the Project Inspector is built: registry-driven orchestration, skill types, approval gates, hooks, deterministic scripts, state and resume, worktree safety. |
| [`docs/plugins/ono-plugin-project-inspector/inspection-workflow.md`](docs/plugins/ono-plugin-project-inspector/inspection-workflow.md) | What to type to inspect a repository, in what order, and what each of the five commands does. |

Paths inside a per-plugin document (`skills/`, `agents/`, `scripts/`, `commands/`) are relative to that **plugin's** repository, not to this one.

### Stays in the plugin repositories

`docs/repo-knowledge-contract.md` ships inside **both** `ono-plugin-project-inspector` and `ono-mobile-dev-plugin`, byte-identically. It is deliberately not centralised here: Claude Code has no cross-plugin dependency mechanism, so vendoring it alongside the code that implements it is the only way both ends can ship the same pinned specification. The producer's `scripts/repo-knowledge.ts` and the consumer's `scripts/read-repo-knowledge.ts` and `repo-knowledge-consumer` skill all cite it by that path.

## Which plugin do I need?

| I am a... | Use this plugin |
| --- | --- |
| Engineer shipping a mobile/web feature (React Native, iOS, Android, React) | `ono-mobile-dev-plugin` |
| QA engineer writing test plans or checking coverage | `ono-plugin-qa` |
| Anyone onboarding onto or auditing an unfamiliar repo | `ono-project-inspector` |
| Spec/design team writing HLDs or Figma-based LLDs | `spec-team-toolkit` |
| Designer building a Figma design system (colors, spacing, typography) | `matrix-studio` |

## Available plugins

| Plugin | Description |
| --- | --- |
| [`ono-project-inspector`](#ono-project-inspector) | Inspects an existing repository and gradually builds a structured, approval-gated AI knowledge base for it (`CLAUDE.md`, `AUDIT.md`, `docs/project/`, and per-topic audit documents), then syncs approved findings back into `CLAUDE.md`. **Never modifies source code.** |
| [`ono-mobile-dev-plugin`](#ono-mobile-dev-plugin) | Mobile-division SDLC workflow for Ono Apps: feature analysis, detailed design, task breakdown, implementation, code review, debugging, QA handoff, and release readiness across React Native, native iOS, native Android, and React web, with shared process and platform-aware routing. |
| [`ono-plugin-qa`](#ono-plugin-qa) | Figma- and spec/LLD-grounded QA test planning, approval, sync, dev/QA coverage gap analysis, Appium automation generation, and live locator verification for Ono Apps' features across React, React Native, iOS, and Android, run in parallel with `ono-mobile-dev-plugin`. |
| [`spec-team-toolkit`](#spec-team-toolkit) | HLD and LLD document builders for the spec team: discovers and scopes a High Level Design with self-QA before customer delivery, and turns a Figma component into a two-part LLD spec for developers/QA. |
| [`matrix-studio`](#matrix-studio) | Design-system generators for Figma: builds a complete two-tier color palette, spacing/scale tokens, and a two-platform typography system as Figma Variables and text styles via the Figma MCP, individually or as one orchestrated design-system run. |

**There is deliberately no version column.** A plugin's version is owned by its own
`.claude-plugin/plugin.json`, and Claude Code reads it from there — so a version listed
here could only ever be a copy that goes stale. To see what a plugin is at, read its
manifest in its own repository, or run `/plugin` after installing. The descriptions above
are marketplace-facing summaries of each plugin's own `description`, maintained by hand.

## Install (developers)

Nothing to clone. The marketplace and each plugin are fetched directly from GitHub.

### 1. Add the marketplace

```
/plugin marketplace add OnOAppsDev/ono-plugin-marketplace
```

### 2. Install a plugin

```
/plugin install ono-project-inspector@ono-plugin-marketplace
/plugin install ono-mobile-dev-plugin@ono-plugin-marketplace
/plugin install ono-plugin-qa@ono-plugin-marketplace
/plugin install spec-team-toolkit@ono-plugin-marketplace
/plugin install matrix-studio@ono-plugin-marketplace
```

Install only the ones you need — see [Which plugin do I need?](#which-plugin-do-i-need) above.

Claude Code restarts and the installed plugin's commands, agents, and skills become available.

### 3. Verify

```
/plugin marketplace list
/plugin
```

You should see `ono-plugin-marketplace` listed and your installed plugin(s) enabled.

## Plugin details

### `ono-project-inspector`

Builds a structured, approval-gated AI knowledge base for an existing repo without ever touching its source code.

| Command | What it does |
| --- | --- |
| `/inspect [repo-path]` | Runs the full guided workflow: analysis → docs → audit breakdown/approval loop |
| `/inspect-status [repo-path]` | Read-only progress report |
| `/inspect-topic [topic-name] [repo-path]` | Breaks down a single audit topic |
| `/inspect-approve [repo-path]` | Finalizes a reviewed audit and continues the loop |
| `/inspect-sync [repo-path]` | On-demand maintenance: syncs approved findings back into `CLAUDE.md` |

### `ono-mobile-dev-plugin`

Encodes Ono's eight-stage mobile SDLC, with an approval gate between stages and platform-aware routing for React Native, native iOS, native Android, and React web.

| Command | What it does |
| --- | --- |
| `/analyze-feature` | Detects platform and proposes a technical approach |
| `/dev-design-start` | Turns an approved analysis into a Detailed Design (DD) |
| `/dev-feature-start` | Turns an approved DD into a task breakdown + feature plan |
| `/implement-task` | Executes an individual task from the approved breakdown |
| `/review-code` | Reviews for correctness, style, standards, and performance |
| `/review-security` | Security-focused review with platform-specific concerns |
| `/fix-review-comments` | Addresses review findings, grouped by platform |
| `/create-dev-qa-notes` | Generates a QA handoff with build/install instructions |
| `/prepare-mobile-release` | Validates shippability for the target release |

### `ono-plugin-qa`

Lets QA author Figma- and spec/LLD-grounded test plans while dev is still implementing, then check coverage once a build lands — designed to run in parallel with `ono-mobile-dev-plugin`.

| Command | What it does |
| --- | --- |
| `/create-qa-test-plan` | Authors a test plan from Figma and/or spec/LLD documents |
| `/approve-qa-test-plan` | Marks a test plan approved and ready for coverage analysis |
| `/sync-qa-test-plan` | Updates a test plan when designs or specs change |
| `/check-qa-coverage` | Compares approved test plans against dev QA handoff notes to surface gaps |

### `spec-team-toolkit`

Two skills for the spec team's document workflow — invoked by describing what you need (no dedicated slash commands).

| Skill | What it does |
| --- | --- |
| `hld-builder` | Discovers and scopes a High Level Design through fixed question scripts, drafts it against the team's template, and self-reviews against a 10-part QA checklist before customer delivery |
| `lld-figma-spec` | Extracts design data from a Figma component via MCP tools and produces a two-part content-entry + display/behavior spec for developers and QA |

### `matrix-studio`

Generates a complete Figma design system — color palette, spacing/scale tokens, and a two-platform typography system — directly into a Figma file via the bundled Figma MCP connection. Requires a Full Figma seat and edit permission on the target file; a one-time Figma OAuth authorization is needed after install (see the plugin's own README for setup).

| Skill | Slash command | What it creates |
| --- | --- | --- |
| Color palette | `/matrix-studio:figma-color-palette-generator` | `Primitives` + `Semantic` (Light/Dark) COLOR variable collections + a visual reference frame |
| Space & scale | `/matrix-studio:figma-space-scale-generator` | `Scale` + `Tokens` FLOAT variable collections + a visual reference frame |
| Typography | `/matrix-studio:figma-type-system-generator` | `Type Primitives` + `Type Tokens` collections + variable-bound text styles for desktop/mobile + a visual specimen frame |
| Full design system | `/matrix-studio:generate-design-system` | Runs all three above, in order, with one shared preflight, input pass, conflict check, and completion report |

## Updating

After new changes are published to a plugin or to this marketplace repo:

```
/plugin marketplace update ono-plugin-marketplace
```

## How it's wired

- `.claude-plugin/marketplace.json` — the marketplace manifest. Each plugin entry uses an **HTTPS `url` source** pointing at the plugin's own repository (e.g. `https://github.com/OnOAppsDev/ono-plugin-project-inspector.git`). HTTPS is used (rather than a `github` source, which clones over SSH) so installs work on any machine without SSH keys configured.
- Plugin code is **not** duplicated or symlinked into this repo. It lives only in its own repository; Claude Code fetches it from GitHub on install.
- **Plugin versions are not declared here.** Each entry carries only `name`, `source`, and marketplace-facing `description`/`author`. A plugin's `version` lives in its own `.claude-plugin/plugin.json`, which is what Claude Code uses — the docs are explicit that when both declare a version, `plugin.json` wins *without warning*, so a copy here could silently mask the real one. Omitting it also means git-sourced plugins update whenever their tracked `ref` moves, rather than only when someone remembers to edit two repositories. `metadata.version` at the top of the manifest is this marketplace's own version and stays owned here.

Each source pins `ref: main`, so the marketplace tracks each plugin repo's default branch. To pin an install to a specific release instead, change `ref` to a tag (or add a `sha`):

```json
{
  "name": "ono-project-inspector",
  "source": {
    "source": "url",
    "url": "https://github.com/OnOAppsDev/ono-plugin-project-inspector.git",
    "ref": "v0.7.0"
  }
}
```
