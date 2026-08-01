# Repository Knowledge Encapsulation MVP — Implementation Plan

> **Status: executed.** All 12 tasks were implemented and independently reviewed. This document was amended in place as defects were found, so it records what was actually built rather than the original intent — which is why it is retained rather than archived.

> **Note on paths.** Task instructions below name documentation files as they existed during implementation. Two have since been merged: `docs/architecture.md` and `docs/architecture/PLATFORM_ARCHITECTURE.md` are now `docs/architecture/plugin-architecture.md`; `docs/plugin-workflow.md` is now `docs/workflows/inspection-workflow.md`. All code, script and skill paths are unchanged. Start at `docs/architecture/ecosystem-overview.html` for the current documentation layout.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `ono-plugin-project-inspector` the single producer of repository knowledge via a deterministic, versioned `.ono/repo-knowledge.json`, and make `ono-mobile-dev-plugin` consume it instead of re-deriving repository inventory — with byte-identical behavior when the manifest is absent.

**Architecture:** The inspector gains one internal skill (`repo-knowledge`) and one deterministic helper (`scripts/repo-knowledge.ts`) that *derives* a manifest from artifacts already approved by the existing workflow — it performs no new repository analysis. The mobile-dev plugin gains one deterministic reader (`scripts/read-repo-knowledge.ts`) and one skill (`repo-knowledge-consumer`) that is the single component aware of the manifest format. `repo-analyst` narrows to feature-scoped detection only. Templates carry a *reference* to repository knowledge instead of a verbatim copy.

**Tech Stack:** TypeScript run directly under Bun (inspector) / Node ≥23.6 or Bun (mobile-dev). No external dependencies, no test framework — self-contained test scripts matching each repo's existing convention. Markdown skills/commands/agents/templates.

---

## Scope reconciliation — read this first

The proposal document defines six phases. **Your "Phase 1" spans proposal Phases 1–4**, because scope points 3–6 require the consumer side, the `repo-analyst` narrowing, and the template change — not just the producer. That is a coherent unit of work and this plan implements exactly your seven scope points. Naming it so there is no ambiguity later:

| Your scope point | Proposal phase | Tasks here |
|---|---|---|
| 1, 2 — inspector produces a deterministic versioned manifest | Phase 1 | T1–T5 |
| 3 — mobile-dev consumes it | Phase 2 + 3 | T6–T8 |
| 4 — `repo-analyst` narrowed to feature-specific detection | Phase 3 | T9 |
| 5 — `/analyze-feature` references instead of embeds | Phase 4 | T10 |
| 6 — `/dev-design-start` references instead of re-scans | Phase 4 | T11 |
| 7 — 100% backward compatible when manifest absent | invariant of all phases | T7, T12 |

Everything in your out-of-scope list stays out. Specifically excluded and **not** built here, even though the proposal describes them:

- **No `cautions` in the manifest.** Feeding approved audit findings to reviewers is *review inheritance* — explicitly out of scope. The manifest carries an audit-topic *index* only (topic, status, file path), because the AUDIT.md parser produces it for free and `coverage` needs it. No audit file bodies are parsed. `cautions` can be added additively in schema v2.
- **No `.ono/feature-state.json`**, no task-status storage.
- **No hook redesign.** The four inspector `hooks/*.md` files gain one verification line each (they are agent-read checklists, not `hooks.json` events). No new hook is added to either plugin — the proposal's `block-canonical-knowledge-writes` hook is **not** built.
- **No registry redesign.** One additive `registry.json` entry, using the existing `type: internal` shape that `inspection-state` already established. No new fields, no schema change.
- **No release, QA, review, or lifecycle changes.** `/review-code`, `/review-security`, `/prepare-mobile-release`, `/implement-task`, `/dev-feature-start`, `/create-dev-qa-notes`, `/fix-review-comments` are untouched.

## Decisions taken (confirm or correct before T1)

You did not answer the four open questions in §11 of the proposal, but scope point 4 answers two of them by implication. I have designed to the narrow, low-risk reading. **If either of the first two is wrong, T1 and T9 change materially.**

**D1 — The inspector does NOT own platform authority.** Scope point 4 keeps "platform routing, device_type, feature context" in `repo-analyst`. Therefore the manifest carries `stack.platformHints` — advisory strings lifted from `CLAUDE.md`'s existing `- Platform(s):` bullet — and nothing more. No RN-shell linkage detection, no monorepo workspace map, no `deviceTargets`. `repo-analyst` keeps its full 6-step platform algorithm and may use `platformHints` only as a prior to *corroborate* its own finding. It must never skip detection because of it. The human confirmation gate in `/analyze-feature` is untouched.

**D2 — `device_type` stays 100% in mobile-dev.** Not represented in the manifest at all.

**D3 — `.ono/repo-knowledge.json` is committed to git**, consistent with `.ono/state.json`. It is fingerprinted, so staleness is detectable; committing means a teammate who has not run `/inspect` still benefits. Neither plugin modifies the target repo's `.gitignore`.

**D4 — The contract spec is duplicated into both repos**, at `docs/repo-knowledge-contract.md`, version-pinned to schema v1. Claude Code has no cross-plugin dependency mechanism, so a single shared file is impossible; the mobile-dev plugin already set this precedent when it vendored `resolve-target-repo-root.ts`. Both copies must stay byte-identical below the title line, and T5/T8 each assert their own copy's `Schema version: 1` line.

**D5 — The manifest is derived, never authored.** `scripts/repo-knowledge.ts` reads only `CLAUDE.md`, `AUDIT.md`, and `docs/project/*.md` — all already produced and human-approved by the existing workflow. It never reads repository source. This is what keeps the inspector's "never inspects source outside a skill" constraint intact and means the manifest introduces no new approval gate.

**D6 — Structured facts come from a marker block, with prose fallback.** `project-analysis` gains a `<!-- repo-knowledge:facts:start/end -->` block in its `CLAUDE.md` template, following the existing `audit-sync` managed-block precedent. The parser prefers that block; when it is absent — which is the case for **every already-inspected repository** — it falls back to extracting the template's fixed `## Tech Stack` bullet labels and `## Build, Run, and Test Commands` fenced block. Both paths feed `coverage`, so an unparseable field is reported as `unknown` and the consumer derives it live rather than trusting a guess.

## Global Constraints

Every task's requirements implicitly include these.

- **Backward compatibility is the acceptance criterion, not a nice-to-have.** With no `.ono/repo-knowledge.json` present, `/analyze-feature` and `/dev-design-start` must behave exactly as they do today. T12 verifies this explicitly.
- **`REPO_KNOWLEDGE_SCHEMA_VERSION = 1`** — declared in both `scripts/repo-knowledge.ts` and `scripts/read-repo-knowledge.ts`. The consumer treats any manifest whose version exceeds its maximum supported version as absent.
- **No external dependencies.** Node stdlib only (`fs`, `path`, `crypto`, `child_process`). No YAML library — the facts block uses a strict `key: value` / `key: [a, b]` subset parsed by hand.
- **Runtime conventions differ per repo and must be matched, not unified.** Inspector scripts use `__dirname` and `if (require.main === module)` and are invoked with `bun`. Mobile-dev scripts use `import.meta.dirname ?? __dirname` and a `realpathSync(process.argv[1])` regex guard, and are invoked with `node` or `bun`.
- **The reader never fails a command.** `scripts/read-repo-knowledge.ts` always exits 0 and always emits valid JSON, including for absent, malformed, and too-new manifests. A missing manifest is a normal state, not an error.
- **Deterministic means deterministic.** Same inputs → byte-identical manifest, except `generatedAt`. Tests assert this by emitting twice and diffing with `generatedAt` stripped.
- **The inspector never writes source; the consumer never writes inspector artifacts.** Enforced by each skill's Output Contract prose (no new hook in this MVP).
- **`TARGET_ROOT` discipline is unchanged.** Both new scripts refuse to operate on any path containing `.claude/worktrees/`, matching `inspection-state.ts:451-458`.
- **Version bumps:** inspector `0.8.0 → 0.9.0`, mobile-dev `0.3.0 → 0.4.0`.

---

## File inventory — every file that changes, and why

### `ono-plugin-project-inspector` — 15 files (4 new, 11 modified)

| File | Action | Why this change is required |
|---|---|---|
| `scripts/repo-knowledge.ts` | **create** | The deterministic emitter. Scope point 2 requires the manifest be deterministic and versioned; that rules out having an LLM author it. Mirrors `inspection-state.ts`'s command/exit-code shape so the agent invokes it identically. |
| `scripts/repo-knowledge.test.ts` | **create** | The manifest is a cross-plugin contract; a parser regression silently degrades every consumer. Covers fixture-based extraction, the marker-block override, the prose fallback, absent-artifact coverage, and byte-determinism. Follows the mobile repo's framework-free test style, which the inspector currently lacks entirely. |
| `skills/repo-knowledge/SKILL.md` | **create** | The registry requires a skill folder per entry, and the agent invokes skills by id, not scripts directly. Declares the Output Contract (`.ono/repo-knowledge.json` only) and the "derived, never authored — reads no source" constraint that keeps the inspector's guarantees intact. |
| `docs/repo-knowledge-contract.md` | **create** | The schema is a contract between two independently released plugins with no dependency mechanism. Without a written spec pinned to a version, the two scripts drift. Producer-side copy (D4). |
| `skills/registry.json` | modify | The agent, and `inspection-state.ts`'s stage derivation, both read this file exclusively. An unregistered skill is invisible to orchestration. One additive `type: internal` entry — `inspection-state`'s existing shape, so no consumer of the registry needs to change. |
| `agents/project-inspector.md` | modify | The agent is the only thing that invokes skills. Without a checkpoint the manifest is never emitted. Added alongside the existing `inspection-state (sync)` calls, since the manifest must refresh at exactly the same moments knowledge changes. |
| `skills/project-analysis/SKILL.md` | modify | Two reasons: (a) emit the `repo-knowledge:facts` marker block so structured extraction is reliable rather than prose-parsed (D6); (b) replace `CLAUDE.md`'s duplicated *External Integrations* body with a pointer to `docs/project/integrations.md`, which is duplication finding D4 in the review and the one internal duplication cheap enough to fix here. |
| `hooks/after-project-analysis.md` | modify | Existing checklists independently verify produced artifacts rather than trusting the skill's report. The manifest needs the same treatment, plus a `validate` call that surfaces a missing/malformed facts block to the developer. |
| `hooks/after-project-docs.md` | modify | `docs/project/*` are the pointer targets for the `inventory`/`conventions`/`integrations` categories. The manifest must refresh once they exist or those categories stay `unknown`. |
| `hooks/after-audit-approve.md` | modify | Approving a topic changes the audit-topic index the manifest carries. Refresh keeps it accurate. |
| `hooks/after-audit-sync.md` | modify | `audit-sync` rewrites `CLAUDE.md`'s managed blocks, which changes its hash and therefore the manifest fingerprint. Without a refresh, consumers see a false `stale-artifacts` verdict. |
| `commands/inspect-sync.md` | modify | Gives the developer an existing, discoverable way to regenerate the manifest on demand. Deliberately reuses `/inspect-sync` rather than adding a command, to keep the command surface unchanged. |
| `.claude-plugin/plugin.json` | modify | `skills[]` is explicitly enumerated in this plugin (unlike mobile-dev), so an unlisted skill folder is a manifest inconsistency. Version bump to `0.9.0`. |
| `.claude-plugin/marketplace.json` | modify | Currently pins `0.7.0` while `plugin.json` says `0.8.0` — pre-existing drift. Corrected to `0.9.0` as part of the bump so the two stop diverging further. |
| `README.md`, `docs/architecture.md`, `docs/plugin-workflow.md`, `CHANGELOG.md` | modify | These document the artifact set and the skill roster; both change. `docs/architecture.md` gains the outbound-contract section — the "why" a future maintainer needs. Counted as one row; four files. |

### `ono-mobile-dev-plugin` — 12 files (4 new, 8 modified)

| File | Action | Why this change is required |
|---|---|---|
| `scripts/read-repo-knowledge.ts` | **create** | Constraint C2: exactly one component knows the manifest format, and structured facts must not be parsed by an LLM. Also the only place freshness (git HEAD + artifact hashes) and the degradation matrix are computed. Always exits 0 so a missing manifest can never fail a command — the mechanism behind scope point 7. |
| `scripts/read-repo-knowledge.test.ts` | **create** | The degradation matrix *is* the backward-compatibility guarantee. Each row needs a test: absent, malformed, schema-too-new, fresh, stale-head, stale-artifacts, partial coverage. |
| `skills/repo-knowledge-consumer/SKILL.md` | **create** | The commands need a shared, single definition of "resolve repository knowledge and decide what may be trusted." Without it, `/analyze-feature` and `/dev-design-start` each restate the rules in prose and drift (review finding R7). |
| `docs/repo-knowledge-contract.md` | **create** | Consumer-side copy of the pinned schema (D4). |
| `agents/repo-analyst.md` | modify | **The core change of scope point 4.** Step 6 (navigation/state/data-fetching/testing/folder/lint inventory) is what duplicates `docs/project/patterns.md`. It becomes conditional on `coverage`. Steps 1–5 (platform), Step 6.5 (`device_type`), and the file-attribution rule stay fully live — they are feature- and diff-scoped. |
| `skills/mobile-repo-analysis/SKILL.md` | modify | This skill's methodology tells `repo-analyst` what to do; leaving it unchanged would contradict the narrowed agent. Restructured to consume-then-delta. |
| `commands/analyze-feature.md` | modify | Scope point 5. Adds knowledge resolution as the new step 1 and changes what step 7 writes into the template. |
| `templates/feature-analysis-template.md` | modify | Scope point 5's substance: `## Repo Conventions Detected` currently mandates *"repo-analyst's structured findings, verbatim"* — the embedding that must become a reference. Adds `repo_knowledge_*` frontmatter so downstream stages can resolve what was used. |
| `commands/dev-design-start.md` | modify | Scope point 6. Adds knowledge resolution and reference carry-forward. |
| `skills/dev-design-start/SKILL.md` | modify | Scope point 6's substance: Step 3.4 currently instructs *"browse the relevant source directories, existing components, services, and API routes"* — an unconditional re-scan. Becomes gap-filling only. |
| `templates/dd-template.md` | modify | Must carry the `repo_knowledge_*` reference forward and cite pointers in §19/§20 instead of restating conventions. Minimal edit — no section is added or removed. |
| `agents/rn-architect.md` | modify | Its `Inputs` name only *"repo-analyst's structured findings summary"*, which after T9 is a narrower object. Left stale, the architect would ask for data the agent no longer produces. Adds the component-inventory input so it stops proposing components that already exist. |
| `README.md`, `CHANGELOG.md`, `.claude-plugin/plugin.json` | modify | README's platform-detection and pipeline sections describe the old flow; CHANGELOG records the change; version bump to `0.4.0`. Counted as one row; three files. |

**Not touched, deliberately:** `ono-plugin-qa`, `spec-team-toolkit`, `ono-plugin-marketplace`, all `standards/**`, all `hooks/*.json` and `hooks/*.sh`, `scripts/resolve-target-repo-root.ts` and its test, and the seven mobile-dev commands outside scope points 5–6.

**Totals: 27 files — 8 created, 19 modified.**

---

## Execution order and rationale

Producer first, then reader, then consumer wiring, then templates, then verification. Each task ends at a committable, independently verifiable state, and the ecosystem is fully functional after every one.

```
T1  inspector  scripts/repo-knowledge.ts — skeleton, fingerprint, documents, auditTopics
T2  inspector  scripts/repo-knowledge.ts — fact extraction (marker block + prose fallback) + coverage
T3  inspector  skills/repo-knowledge/SKILL.md + registry entry + plugin.json
T4  inspector  agent checkpoints + 4 hook checklists + /inspect-sync
T5  inspector  project-analysis facts block + integrations pointer + contract doc + docs/CHANGELOG/version
        ── manifest now produced; nothing consumes it; inspector fully working ──
T6  mobile     scripts/read-repo-knowledge.ts + tests (degradation matrix)
T7  mobile     docs/repo-knowledge-contract.md + schema-parity assertion
        ── reader exists and is proven; no command uses it yet ──
T8  mobile     skills/repo-knowledge-consumer/SKILL.md
T9  mobile     agents/repo-analyst.md narrowing + mobile-repo-analysis rewrite
T10 mobile     /analyze-feature + feature-analysis-template
T11 mobile     /dev-design-start + dev-design-start skill + dd-template + rn-architect
T12 both       end-to-end verification across three repository states + README/CHANGELOG/version
```

Why this order specifically:

- **T1 before T2** because fingerprint/documents/auditTopics are pure structure with no ambiguity, while extraction is the fragile part. Getting the deterministic skeleton green first means T2's failures are unambiguously extraction failures.
- **T5 (the facts block) after T1–T4** because the parser must already handle its absence. Building the fallback first proves already-inspected repositories keep working before the new emission path exists.
- **T6–T7 before T8** because the reader is the only component that can be unit-tested. Every later task depends on its verdicts, so it must be proven before any prose consumes it.
- **T9 after T8** because narrowing `repo-analyst` without a consumer skill to define "what may be trusted" would leave a gap where neither component owns the decision.
- **T10 and T11 last** because they are the only tasks that change what a developer sees, and they must land on top of a proven reader.
- **T12 last and mandatory.** Scope point 7 is a claim about behavior, not code, and only an end-to-end run across three repository states can substantiate it.

Tasks are strictly sequential. T1→T2 and T6→T7 share files; T9→T10→T11 share the knowledge-resolution contract. Do not parallelize.

---

## Risks and breaking changes

### Breaking changes: none, by construction

| Surface | Compatibility |
|---|---|
| Existing `.ono/state.json` files | Untouched. No schema change, no migration. |
| Already-inspected repositories | Keep working. Their `CLAUDE.md` has no facts block; the prose fallback handles it and marks unparseable fields `unknown`. Re-running `/inspect-sync` produces a manifest without re-inspecting. |
| Repositories with no inspection | Behave exactly as today. Reader returns `available: false`; both commands take the full-derivation branch. |
| In-flight feature documents | Old-shape feature analyses (with an embedded `## Repo Conventions Detected` body) remain valid input to `/dev-design-start`. T11 requires accepting both shapes. |
| Commands, command names, arguments | Unchanged. No command added or removed in either plugin. |
| The eight-stage pipeline and every approval gate | Unchanged. |
| `AUDIT.md` as human source of truth; `Draft → Approved` single-writer | Unchanged. |

### Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| K1 | **Prose extraction is fragile.** `CLAUDE.md` is LLM-authored; a reworded `- Language(s):` bullet breaks the fallback. | High | Low | Extraction failure is never an error — the field is omitted and `coverage` becomes `unknown`, so the consumer derives live. Failure mode is "no better than today," never "wrong." T2 tests a deliberately malformed fixture. The facts block (T5) makes new inspections reliable. |
| K2 | **Silent staleness** — a feature planned against outdated conventions. | Medium | High | `fingerprint.gitHead` + per-artifact SHA-256. `stale-artifacts` forces live re-derivation of affected categories. Every consumption surfaces the freshness verdict to the developer. |
| K3 | **The two schema constants drift** across independently released plugins. | Medium | High | Pinned contract doc in both repos (D4); T7 asserts the consumer's supported version matches the doc; consumer treats a too-new manifest as absent rather than mis-parsing. |
| K4 | **Narrowing `repo-analyst` loses standards-conformance judgment.** Its Step 6 does something `patterns.md` does not: compare folder structure against `ARCH-LAYERS-*`/`ARCH-FOLDERS-*`. | Medium | High | T9 explicitly retains the standards comparison and narrows only the neutral inventory. Observation is inspector-owned; judgment against org standards stays mobile-dev-owned. Called out as a step-level requirement, not left to the implementer. |
| K5 | **Platform authority accidentally migrates** to the manifest, weakening the human confirmation gate. | Low | High | D1. The manifest has no singular `platform` field — only advisory `platformHints`. T9 requires `repo-analyst` to run its full algorithm regardless, and T10 leaves the confirmation prompt untouched. |
| K6 | **A worktree-resolved root produces a manifest in the wrong place.** | Low | Medium | Both scripts refuse `.claude/worktrees/` paths, matching `inspection-state.ts`. The inspector's hooks additionally run `verify-artifacts.ts`. |
| K7 | **Anchor drift** — `docs/project/*` headings change and stored anchors 404. | Low | Low | Anchors are re-derived on every emit from live headings, never hand-written. A changed heading changes the artifact hash, which triggers `stale-artifacts`. |
| K8 | **Scope creep into review inheritance**, the most tempting adjacent feature. | Medium | Medium | `cautions` is excluded from schema v1 and no audit body is ever read. T1's test asserts the emitted manifest has no `cautions` key. |
| K9 | **Determinism regression** — an implementer adds `Date.now()` or unordered object/array iteration. | Medium | Medium | T1 includes a double-emit byte-comparison test with `generatedAt` stripped. All arrays are explicitly sorted. |
| K10 | **`.ono/repo-knowledge.json` committed with machine-specific data.** | Low | Medium | Only repo-relative paths, hashes, and the git SHA are persisted — same portability rule as `state.json`. T1 asserts no absolute path appears in the output. |

---

## The contract — `.ono/repo-knowledge.json` schema v1

This is the authoritative shape for this MVP. Both `docs/repo-knowledge-contract.md` copies reproduce it verbatim.

```jsonc
{
  "repoKnowledgeSchemaVersion": 1,
  "producedBy": { "plugin": "ono-project-inspector", "version": "0.9.0" },
  "generatedAt": "2026-07-28T09:12:44.000Z",

  // Freshness. gitHead is the repo's HEAD at emit time; artifacts maps each
  // source document to its SHA-256, so a consumer detects a hand-edit that
  // happened outside an inspection run.
  "fingerprint": {
    "gitHead": "a1b2c3d4e5f6...",
    "artifacts": {
      "CLAUDE.md": "sha256hex",
      "AUDIT.md": "sha256hex",
      "docs/project/overview.md": "sha256hex",
      "docs/project/components.md": "sha256hex",
      "docs/project/patterns.md": "sha256hex",
      "docs/project/integrations.md": null
    }
  },

  // Per-category trust. "unknown" means the consumer MUST derive it live.
  "coverage": {
    "stack": "populated",
    "commands": "partial",
    "structure": "populated",
    "inventory": "populated",
    "conventions": "populated",
    "integrations": "unknown",
    "auditTopics": "populated"
  },

  // Structured facts. platformHints is ADVISORY ONLY (decision D1) — never
  // authoritative for platform routing.
  "stack": {
    "languages": ["TypeScript"],
    "frameworks": ["React Native"],
    "platformHints": ["iOS", "Android"],
    "runtimeTooling": ["Metro"],
    "packageManagers": ["yarn"]
  },

  "commands": { "install": "yarn", "run": "yarn ios", "test": "yarn test", "build": null },

  // Pointers, not copies. Prose stays in the approved artifact.
  "structure": {
    "repositoryTree": "CLAUDE.md#repository-structure",
    "keyModules": "CLAUDE.md#key-modules",
    "entryPoints": "CLAUDE.md#entry-points"
  },

  "documents": {
    "claudeMd":     { "path": "CLAUDE.md", "exists": true, "anchors": [] },
    "auditMd":      { "path": "AUDIT.md", "exists": true, "anchors": [] },
    "overview":     { "path": "docs/project/overview.md", "exists": true, "anchors": ["#what-this-project-is"] },
    "inventory":    { "path": "docs/project/components.md", "exists": true, "anchors": ["#screens", "#reusable-ui-components", "#shared-hooks-utilities", "#navigation-map", "#design-system-theming", "#known-duplicates-or-deprecated-components"] },
    "conventions":  { "path": "docs/project/patterns.md", "exists": true, "anchors": ["#state-management", "#data-fetching-api-conventions", "#navigation-patterns", "#styling-conventions", "#error-handling-patterns", "#localization-rtl", "#naming-and-folder-conventions", "#testing-patterns", "#patterns-to-avoid-observed-anti-patterns"] },
    "integrations": { "path": "docs/project/integrations.md", "exists": false, "anchors": [] }
  },

  // Index only — topic, status, file. No audit body is ever read (K8).
  "auditTopics": [
    { "topic": "Architecture", "slug": "architecture", "status": "Approved", "file": "audits/architecture/architecture-audit.md" }
  ]
}
```

Invariants: no absolute paths; every array sorted deterministically; `generatedAt` is the only field that varies between two emits over identical inputs; **no `cautions` key in v1.**

---

## Task 1: Deterministic manifest skeleton — fingerprint, documents, audit index

**Files:**
- Create: `ono-plugin-project-inspector/scripts/repo-knowledge.ts`
- Create: `ono-plugin-project-inspector/scripts/repo-knowledge.test.ts`

**Interfaces:**
- Consumes: `slugify` from `./slugify` (existing, `scripts/slugify.ts:18`).
- Produces: `REPO_KNOWLEDGE_SCHEMA_VERSION = 1`; `buildManifest(repoRoot: string): RepoKnowledge`; CLI `emit|validate|show <repo-root>`. Exit codes: `0` success, `1` usage/missing root/worktree refusal, `2` existing manifest is invalid JSON or fails validation. T2 extends `buildManifest`; T4's hooks call `emit` and `validate`.

- [ ] **Step 1: Write the failing test**

Create `ono-plugin-project-inspector/scripts/repo-knowledge.test.ts`:

```ts
/**
 * repo-knowledge.test.ts
 *
 * Self-contained tests for scripts/repo-knowledge.ts. Builds throwaway
 * repositories in a temp dir with fixture CLAUDE.md / AUDIT.md / docs/project
 * files and exercises the CLI end-to-end (stdout JSON + exit code).
 *
 * No external test framework. Run with:
 *   bun scripts/repo-knowledge.test.ts
 */

import { execFileSync } from "child_process";
import { mkdtempSync, mkdirSync, writeFileSync, readFileSync, realpathSync, rmSync, existsSync } from "fs";
import { tmpdir } from "os";
import { join, dirname } from "path";

const HERE = typeof __dirname !== "undefined" ? __dirname : ".";
const HELPER = join(HERE, "repo-knowledge.ts");
const RUNTIME = process.execPath;

let failures = 0;
function check(name: string, cond: boolean, detail = ""): void {
  if (cond) console.log(`PASS  ${name}`);
  else { failures++; console.log(`FAIL  ${name}${detail ? `  — ${detail}` : ""}`); }
}

function run(args: string[]): { code: number; stdout: string; stderr: string } {
  try {
    const stdout = execFileSync(RUNTIME, [HELPER, ...args], { encoding: "utf-8", stdio: ["ignore", "pipe", "pipe"] });
    return { code: 0, stdout, stderr: "" };
  } catch (err: any) {
    return { code: typeof err.status === "number" ? err.status : 1, stdout: err.stdout?.toString() ?? "", stderr: err.stderr?.toString() ?? "" };
  }
}

function git(cwd: string, ...args: string[]): void {
  execFileSync("git", args, { cwd, stdio: "ignore" });
}

function write(root: string, rel: string, body: string): void {
  const p = join(root, rel);
  mkdirSync(dirname(p), { recursive: true });
  writeFileSync(p, body, "utf-8");
}

function initRepo(dir: string): void {
  mkdirSync(dir, { recursive: true });
  git(dir, "init", "-q");
  git(dir, "config", "user.email", "test@example.com");
  git(dir, "config", "user.name", "Test");
  write(dir, "README.md", "# test\n");
  git(dir, "add", "-A");
  git(dir, "commit", "-qm", "init");
}

const AUDIT_MD = `# AUDIT.md — Demo

## Audit Topics

| # | Status | Topic | Priority | File | Notes |
|---|--------|-------|----------|------|-------|
| 1 | Approved | Architecture | High | audits/architecture/architecture-audit.md | ok |
| 2 | Pending Breakdown | Player / Media | Medium | Not created yet | later |
`;

const PATTERNS_MD = `# Patterns and Conventions — Demo

## State Management
Redux Toolkit.

## Testing Patterns
Jest.
`;

const root = mkdtempSync(join(realpathSync(tmpdir()), "rk-"));
try {
  // --- Scenario 1: full artifact set -> manifest with fingerprint, documents, audit index ---
  const repo = join(root, "full");
  initRepo(repo);
  write(repo, "CLAUDE.md", "# CLAUDE.md — Demo\n");
  write(repo, "AUDIT.md", AUDIT_MD);
  write(repo, "docs/project/patterns.md", PATTERNS_MD);
  git(repo, "add", "-A");
  git(repo, "commit", "-qm", "artifacts");

  {
    const r = run(["emit", repo]);
    check("1 emit: exit 0", r.code === 0, `code=${r.code} stderr=${r.stderr}`);
    const p = join(repo, ".ono", "repo-knowledge.json");
    check("1 emit: manifest written", existsSync(p));
    const m = JSON.parse(readFileSync(p, "utf-8"));

    check("1 schema version is 1", m.repoKnowledgeSchemaVersion === 1);
    check("1 producedBy names the inspector", m.producedBy?.plugin === "ono-project-inspector");
    check("1 gitHead is a 40-char sha", typeof m.fingerprint?.gitHead === "string" && m.fingerprint.gitHead.length === 40);
    check("1 CLAUDE.md hashed", typeof m.fingerprint?.artifacts?.["CLAUDE.md"] === "string");
    check("1 missing artifact hashes to null", m.fingerprint?.artifacts?.["docs/project/components.md"] === null);

    check("1 documents: patterns exists", m.documents?.conventions?.exists === true);
    check("1 documents: components missing", m.documents?.inventory?.exists === false);
    check("1 anchors derived from headings",
      Array.isArray(m.documents?.conventions?.anchors) &&
      m.documents.conventions.anchors.includes("#state-management") &&
      m.documents.conventions.anchors.includes("#testing-patterns"),
      JSON.stringify(m.documents?.conventions?.anchors));

    check("1 auditTopics: 2 rows", m.auditTopics?.length === 2, JSON.stringify(m.auditTopics));
    check("1 auditTopics: slug via slugify", m.auditTopics?.[1]?.slug === "player-media", m.auditTopics?.[1]?.slug);
    check("1 auditTopics: status carried", m.auditTopics?.[0]?.status === "Approved");
    check("1 coverage.auditTopics populated", m.coverage?.auditTopics === "populated");

    // Out-of-scope guard (K8): review inheritance is NOT part of schema v1.
    check("1 no cautions key in v1", !("cautions" in m));
    // Portability (K10).
    check("1 no absolute paths persisted", !JSON.stringify(m).includes(realpathSync(repo)));
  }

  // --- Scenario 2: determinism — two emits differ only in generatedAt (K9) ---
  {
    const p = join(repo, ".ono", "repo-knowledge.json");
    const first = JSON.parse(readFileSync(p, "utf-8"));
    run(["emit", repo]);
    const second = JSON.parse(readFileSync(p, "utf-8"));
    delete first.generatedAt;
    delete second.generatedAt;
    check("2 determinism: byte-identical ignoring generatedAt",
      JSON.stringify(first) === JSON.stringify(second));
  }

  // --- Scenario 3: no artifacts at all -> emits, everything unknown ---
  {
    const bare = join(root, "bare");
    initRepo(bare);
    const r = run(["emit", bare]);
    check("3 bare: exit 0", r.code === 0, `code=${r.code} stderr=${r.stderr}`);
    const m = JSON.parse(readFileSync(join(bare, ".ono", "repo-knowledge.json"), "utf-8"));
    check("3 bare: coverage.stack unknown", m.coverage?.stack === "unknown");
    check("3 bare: coverage.auditTopics unknown", m.coverage?.auditTopics === "unknown");
    check("3 bare: auditTopics empty", Array.isArray(m.auditTopics) && m.auditTopics.length === 0);
    check("3 bare: claudeMd exists false", m.documents?.claudeMd?.exists === false);
  }

  // --- Scenario 4: validate ---
  {
    const ok = run(["validate", repo]);
    check("4 validate: exit 0 on good manifest", ok.code === 0, `code=${ok.code} stderr=${ok.stderr}`);

    const broken = join(root, "broken");
    initRepo(broken);
    write(broken, ".ono/repo-knowledge.json", "{ not json");
    const bad = run(["validate", broken]);
    check("4 validate: exit 2 on malformed", bad.code === 2, `code=${bad.code}`);

    const none = join(root, "none");
    initRepo(none);
    check("4 validate: exit 2 when absent", run(["validate", none]).code === 2);
  }

  // --- Scenario 5: worktree refusal (K6) ---
  {
    const wt = join(repo, ".claude", "worktrees", "agent-1");
    git(repo, "worktree", "add", "-q", wt);
    const r = run(["emit", wt]);
    check("5 worktree: exit 1", r.code === 1, `code=${r.code}`);
    check("5 worktree: no manifest written there", !existsSync(join(wt, ".ono", "repo-knowledge.json")));
  }

  // --- Scenario 6: usage errors ---
  {
    check("6 usage: no args -> exit 1", run([]).code === 1);
    check("6 usage: unknown command -> exit 1", run(["frobnicate", repo]).code === 1);
    check("6 usage: missing root -> exit 1", run(["emit", join(root, "nope")]).code === 1);
  }
} finally {
  rmSync(root, { recursive: true, force: true });
}

console.log(failures === 0 ? "\nALL TESTS PASSED" : `\n${failures} TEST(S) FAILED`);
process.exit(failures > 0 ? 1 : 0);
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd ono-plugin-project-inspector && bun scripts/repo-knowledge.test.ts
```

Expected: every scenario FAILs — `repo-knowledge.ts` does not exist, so each `run()` returns a non-zero code.

- [ ] **Step 3: Write the implementation**

Create `ono-plugin-project-inspector/scripts/repo-knowledge.ts`:

```ts
/**
 * repo-knowledge.ts
 *
 * Deterministic helper that owns the repository-knowledge manifest at
 * `<repo>/.ono/repo-knowledge.json`. Invoked by the internal `repo-knowledge`
 * skill (skills/repo-knowledge/SKILL.md), which the project-inspector agent
 * calls automatically whenever repository knowledge changes.
 *
 * Design contract:
 * - DERIVED, NEVER AUTHORED. It reads only artifacts the workflow has already
 *   produced and a human has already approved: CLAUDE.md, AUDIT.md, and
 *   docs/project/*.md. It never reads repository source, so it introduces no
 *   new analysis and no new approval gate.
 * - POINTERS, NOT COPIES. Prose stays in the approved artifact; the manifest
 *   carries paths plus heading anchors so consumers can cite a section.
 * - PORTABLE. Only repo-relative paths, hashes, and the git SHA are persisted.
 * - DETERMINISTIC. Same inputs produce a byte-identical file except for
 *   `generatedAt`. Every array is explicitly sorted.
 * - The `.ono/` directory is the shared Ono infrastructure directory; this
 *   plugin owns `state.json` and `repo-knowledge.json` within it.
 *
 * Usage:
 *   bun scripts/repo-knowledge.ts <emit|validate|show> <repo-root>
 *
 * Exit codes:
 *   0 - success
 *   1 - usage error, missing repo root, or a .claude/worktrees path
 *   2 - manifest absent (validate) or present but invalid
 */

import { readFileSync, writeFileSync, existsSync, mkdirSync, realpathSync } from "fs";
import { createHash } from "crypto";
import { execFileSync } from "child_process";
import { join, sep } from "path";
import { slugify } from "./slugify";

/** Bump only for a breaking shape change; additive fields keep version 1. */
export const REPO_KNOWLEDGE_SCHEMA_VERSION = 1;

const WORKTREE_MARKER = `${sep}.claude${sep}worktrees${sep}`;

/** Every artifact the manifest derives from, in fingerprint order. */
const SOURCE_ARTIFACTS = [
  "CLAUDE.md",
  "AUDIT.md",
  "docs/project/overview.md",
  "docs/project/components.md",
  "docs/project/patterns.md",
  "docs/project/integrations.md",
] as const;

/** documents[] key -> repo-relative path. */
const DOCUMENT_MAP: Array<[string, string]> = [
  ["claudeMd", "CLAUDE.md"],
  ["auditMd", "AUDIT.md"],
  ["overview", "docs/project/overview.md"],
  ["inventory", "docs/project/components.md"],
  ["conventions", "docs/project/patterns.md"],
  ["integrations", "docs/project/integrations.md"],
];

type Coverage = "populated" | "partial" | "unknown";

interface DocumentRef {
  path: string;
  exists: boolean;
  anchors: string[];
}

interface AuditTopicRef {
  topic: string;
  slug: string;
  status: string;
  file: string;
}

export interface RepoKnowledge {
  repoKnowledgeSchemaVersion: number;
  producedBy: { plugin: string; version: string };
  generatedAt: string;
  fingerprint: { gitHead: string | null; artifacts: Record<string, string | null> };
  coverage: Record<string, Coverage>;
  stack: {
    languages: string[];
    frameworks: string[];
    platformHints: string[];
    runtimeTooling: string[];
    packageManagers: string[];
  };
  commands: { install: string | null; run: string | null; test: string | null; build: string | null };
  structure: { repositoryTree: string | null; keyModules: string | null; entryPoints: string | null };
  documents: Record<string, DocumentRef>;
  auditTopics: AuditTopicRef[];
}

function manifestPath(repoRoot: string): string {
  return join(repoRoot, ".ono", "repo-knowledge.json");
}

function readIfExists(repoRoot: string, rel: string): string | null {
  const p = join(repoRoot, rel);
  return existsSync(p) ? readFileSync(p, "utf-8") : null;
}

function sha256(text: string): string {
  return createHash("sha256").update(text, "utf-8").digest("hex");
}

function gitHead(repoRoot: string): string | null {
  try {
    return execFileSync("git", ["rev-parse", "HEAD"], {
      cwd: repoRoot,
      encoding: "utf-8",
      stdio: ["ignore", "pipe", "ignore"],
    }).trim() || null;
  } catch {
    return null;
  }
}

/** Current plugin identity, read from the plugin's own manifest (portable, relative to this script). */
function currentPlugin(): { plugin: string; version: string } {
  try {
    const m = JSON.parse(readFileSync(join(__dirname, "..", ".claude-plugin", "plugin.json"), "utf-8"));
    return { plugin: m.name ?? "ono-project-inspector", version: m.version ?? "0.0.0" };
  } catch {
    return { plugin: "ono-project-inspector", version: "0.0.0" };
  }
}

/**
 * GitHub-style anchors for every `##`/`###` heading. Re-derived on every emit
 * from live headings so an anchor can never be stale relative to the document.
 */
export function headingAnchors(md: string): string[] {
  const out: string[] = [];
  for (const line of md.split("\n")) {
    if (!/^#{2,3}\s+/.test(line)) continue;
    const anchor =
      "#" +
      line
        .replace(/^#{2,3}\s+/, "")
        .trim()
        .toLowerCase()
        .replace(/[^a-z0-9\s-]/g, "")
        .trim()
        .replace(/\s+/g, "-");
    if (anchor.length > 1 && !out.includes(anchor)) out.push(anchor);
  }
  return out;
}

/**
 * Parse AUDIT.md's `## Audit Topics` table. Identical row-shape logic to
 * scripts/inspection-state.ts so the two never disagree about the table.
 */
export function parseAuditTopics(md: string | null): AuditTopicRef[] {
  if (!md) return [];
  const rows: AuditTopicRef[] = [];
  let inTable = false;
  for (const line of md.split("\n")) {
    if (/^\|\s*#\s*\|\s*Status\s*\|\s*Topic\s*\|/i.test(line)) {
      inTable = true;
      continue;
    }
    if (!inTable) continue;
    if (!line.trim().startsWith("|")) break;
    if (/^\|\s*-+\s*\|/.test(line)) continue;
    const cells = line.split("|").slice(1, -1).map((c) => c.trim());
    if (cells.length < 6) continue;
    rows.push({ topic: cells[2], slug: slugify(cells[2]), status: cells[1], file: cells[4] });
  }
  return rows;
}

/** T2 replaces this stub with real extraction. */
function extractFacts(_claudeMd: string | null): {
  stack: RepoKnowledge["stack"];
  commands: RepoKnowledge["commands"];
} {
  return {
    stack: { languages: [], frameworks: [], platformHints: [], runtimeTooling: [], packageManagers: [] },
    commands: { install: null, run: null, test: null, build: null },
  };
}

function coverageForList(values: string[][]): Coverage {
  const filled = values.filter((v) => v.length > 0).length;
  if (filled === 0) return "unknown";
  return filled === values.length ? "populated" : "partial";
}

function coverageForNullable(values: Array<string | null>): Coverage {
  const filled = values.filter((v) => v !== null && v !== "").length;
  if (filled === 0) return "unknown";
  return filled === values.length ? "populated" : "partial";
}

export function buildManifest(repoRoot: string): RepoKnowledge {
  const claudeMd = readIfExists(repoRoot, "CLAUDE.md");
  const auditMd = readIfExists(repoRoot, "AUDIT.md");

  const artifacts: Record<string, string | null> = {};
  for (const rel of SOURCE_ARTIFACTS) {
    const body = readIfExists(repoRoot, rel);
    artifacts[rel] = body === null ? null : sha256(body);
  }

  const documents: Record<string, DocumentRef> = {};
  for (const [key, rel] of DOCUMENT_MAP) {
    const body = readIfExists(repoRoot, rel);
    documents[key] = {
      path: rel,
      exists: body !== null,
      // Anchors are only useful for the docs/project knowledge base; CLAUDE.md
      // and AUDIT.md are referenced by fixed section pointers instead.
      anchors: body !== null && rel.startsWith("docs/project/") ? headingAnchors(body) : [],
    };
  }

  const auditTopics = parseAuditTopics(auditMd);
  const { stack, commands } = extractFacts(claudeMd);

  const structure = claudeMd
    ? {
        repositoryTree: "CLAUDE.md#repository-structure",
        keyModules: "CLAUDE.md#key-modules",
        entryPoints: "CLAUDE.md#entry-points",
      }
    : { repositoryTree: null, keyModules: null, entryPoints: null };

  const coverage: Record<string, Coverage> = {
    stack: coverageForList([
      stack.languages,
      stack.frameworks,
      stack.platformHints,
      stack.runtimeTooling,
      stack.packageManagers,
    ]),
    commands: coverageForNullable([commands.install, commands.run, commands.test, commands.build]),
    structure: claudeMd ? "populated" : "unknown",
    inventory: documents.inventory.exists && documents.inventory.anchors.length > 0 ? "populated" : "unknown",
    conventions: documents.conventions.exists && documents.conventions.anchors.length > 0 ? "populated" : "unknown",
    integrations: documents.integrations.exists && documents.integrations.anchors.length > 0 ? "populated" : "unknown",
    auditTopics: auditTopics.length > 0 ? "populated" : "unknown",
  };

  return {
    repoKnowledgeSchemaVersion: REPO_KNOWLEDGE_SCHEMA_VERSION,
    producedBy: currentPlugin(),
    generatedAt: new Date().toISOString(),
    fingerprint: { gitHead: gitHead(repoRoot), artifacts },
    coverage,
    stack,
    commands,
    structure,
    documents,
    auditTopics,
  };
}

/** Structural validation. Deliberately shallow: this guards shape, not content quality. */
export function validateManifest(value: unknown): string[] {
  const errors: string[] = [];
  const m = value as Partial<RepoKnowledge> | null;
  if (!m || typeof m !== "object") return ["manifest is not an object"];
  if (m.repoKnowledgeSchemaVersion !== REPO_KNOWLEDGE_SCHEMA_VERSION) {
    errors.push(`repoKnowledgeSchemaVersion is ${String(m.repoKnowledgeSchemaVersion)}, expected ${REPO_KNOWLEDGE_SCHEMA_VERSION}`);
  }
  if (!m.producedBy?.plugin) errors.push("producedBy.plugin missing");
  if (typeof m.generatedAt !== "string") errors.push("generatedAt missing");
  if (!m.fingerprint || typeof m.fingerprint.artifacts !== "object") errors.push("fingerprint.artifacts missing");
  if (!m.coverage || typeof m.coverage !== "object") errors.push("coverage missing");
  if (!m.documents || typeof m.documents !== "object") errors.push("documents missing");
  if (!Array.isArray(m.auditTopics)) errors.push("auditTopics is not an array");
  return errors;
}

function writeManifest(repoRoot: string, manifest: RepoKnowledge): void {
  const dir = join(repoRoot, ".ono");
  if (!existsSync(dir)) mkdirSync(dir, { recursive: true });
  writeFileSync(manifestPath(repoRoot), JSON.stringify(manifest, null, 2) + "\n", "utf-8");
}

function cmdEmit(repoRoot: string): void {
  const manifest = buildManifest(repoRoot);
  writeManifest(repoRoot, manifest);
  const unknown = Object.entries(manifest.coverage)
    .filter(([, v]) => v === "unknown")
    .map(([k]) => k);
  console.log(
    `Emitted ${join(".ono", "repo-knowledge.json")} (schema v${REPO_KNOWLEDGE_SCHEMA_VERSION}, ` +
      `${manifest.auditTopics.length} audit topic(s))` +
      (unknown.length ? `; coverage unknown: ${unknown.join(", ")}` : "; full coverage")
  );
  process.exit(0);
}

function cmdValidate(repoRoot: string): void {
  const p = manifestPath(repoRoot);
  if (!existsSync(p)) {
    console.error(`No manifest at ${join(".ono", "repo-knowledge.json")}. Run this skill's emit step.`);
    process.exit(2);
  }
  let parsed: unknown;
  try {
    parsed = JSON.parse(readFileSync(p, "utf-8"));
  } catch (err) {
    console.error(`repo-knowledge.json is not valid JSON: ${(err as Error).message}`);
    process.exit(2);
  }
  const errors = validateManifest(parsed);
  if (errors.length) {
    console.error(`repo-knowledge.json failed validation:\n- ${errors.join("\n- ")}`);
    process.exit(2);
  }
  const m = parsed as RepoKnowledge;
  const unknown = Object.entries(m.coverage).filter(([, v]) => v === "unknown").map(([k]) => k);
  console.log(
    `repo-knowledge.json is valid (schema v${m.repoKnowledgeSchemaVersion}, produced by ${m.producedBy.plugin} ${m.producedBy.version})` +
      (unknown.length ? `; coverage unknown: ${unknown.join(", ")}` : "; full coverage")
  );
  process.exit(0);
}

function cmdShow(repoRoot: string): void {
  const p = manifestPath(repoRoot);
  if (!existsSync(p)) {
    console.error(`No manifest at ${join(".ono", "repo-knowledge.json")}.`);
    process.exit(2);
  }
  console.log(readFileSync(p, "utf-8").trimEnd());
  process.exit(0);
}

function main(): void {
  const [, , command, repoRoot] = process.argv;
  if (!command || !repoRoot) {
    console.error("Usage: repo-knowledge.ts <emit|validate|show> <repo-root>");
    process.exit(1);
  }
  if (!existsSync(repoRoot)) {
    console.error(`Repository root not found: ${repoRoot}`);
    process.exit(1);
  }
  // Worktree safety: a Claude agent worktree is an ephemeral execution copy,
  // not the developer's repository. Callers must pass the resolved main-tree
  // root (scripts/resolve-repo-root.ts). Guard reads too, since a manifest
  // read from a worktree would mislead every consumer.
  if (realpathSync(repoRoot).includes(WORKTREE_MARKER)) {
    console.error(
      `Refusing to operate on a Claude agent worktree: ${repoRoot}\n` +
        `Pass the resolved main repository root (scripts/resolve-repo-root.ts).`
    );
    process.exit(1);
  }
  switch (command) {
    case "emit":
      return cmdEmit(repoRoot);
    case "validate":
      return cmdValidate(repoRoot);
    case "show":
      return cmdShow(repoRoot);
    default:
      console.error(`Unknown command "${command}".`);
      process.exit(1);
  }
}

if (require.main === module) {
  main();
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
cd ono-plugin-project-inspector && bun scripts/repo-knowledge.test.ts
```

Expected: `ALL TESTS PASSED`. Scenario 1's `coverage.stack unknown` is not asserted here because `extractFacts` is still a stub — scenario 3 covers the unknown path and T2 adds the populated assertions.

- [ ] **Step 5: Commit**

```bash
git add scripts/repo-knowledge.ts scripts/repo-knowledge.test.ts
git commit -m "feat(repo-knowledge): deterministic manifest skeleton — fingerprint, documents, audit index"
```

---

## Task 2: Fact extraction — marker block with prose fallback, and coverage

**Files:**
- Modify: `ono-plugin-project-inspector/scripts/repo-knowledge.ts` (replace the `extractFacts` stub)
- Modify: `ono-plugin-project-inspector/scripts/repo-knowledge.test.ts` (append scenarios 7–9)

**Interfaces:**
- Consumes: `buildManifest` and the `RepoKnowledge` types from T1.
- Produces: `parseFactsBlock(md: string): Record<string, string | string[]> | null`, `extractStackFromProse(md: string): Partial<RepoKnowledge["stack"]>`, `extractCommandsFromProse(md: string): RepoKnowledge["commands"]`, and a real `extractFacts(claudeMd: string | null)`. T5's `CLAUDE.md` template emits the block these parse.

- [ ] **Step 1: Write the failing test**

Append to `ono-plugin-project-inspector/scripts/repo-knowledge.test.ts`, immediately before the closing `} finally {`:

```ts
  // --- Scenario 7: prose fallback (an already-inspected repo, no facts block) ---
  {
    const legacy = join(root, "legacy");
    initRepo(legacy);
    write(legacy, "CLAUDE.md", `# CLAUDE.md — Legacy

## Tech Stack

- Language(s): TypeScript, Kotlin
- Framework(s): React Native
- Platform(s): iOS, Android
- Runtime / Tooling: Metro
- Package manager(s): yarn

## Build, Run, and Test Commands

\`\`\`bash
# Install dependencies
yarn

# Run / develop
yarn ios

# Test
yarn test

# Build
Unknown
\`\`\`
`);
    run(["emit", legacy]);
    const m = JSON.parse(readFileSync(join(legacy, ".ono", "repo-knowledge.json"), "utf-8"));
    check("7 prose: languages extracted", JSON.stringify(m.stack.languages) === JSON.stringify(["Kotlin", "TypeScript"]), JSON.stringify(m.stack.languages));
    check("7 prose: frameworks extracted", JSON.stringify(m.stack.frameworks) === JSON.stringify(["React Native"]));
    check("7 prose: platformHints extracted", JSON.stringify(m.stack.platformHints) === JSON.stringify(["Android", "iOS"]));
    check("7 prose: packageManagers extracted", JSON.stringify(m.stack.packageManagers) === JSON.stringify(["yarn"]));
    check("7 prose: coverage.stack populated", m.coverage.stack === "populated", m.coverage.stack);
    check("7 prose: install command", m.commands.install === "yarn", String(m.commands.install));
    check("7 prose: run command", m.commands.run === "yarn ios", String(m.commands.run));
    check("7 prose: test command", m.commands.test === "yarn test", String(m.commands.test));
    check("7 prose: 'Unknown' build is null", m.commands.build === null, String(m.commands.build));
    check("7 prose: coverage.commands partial", m.coverage.commands === "partial", m.coverage.commands);
  }

  // --- Scenario 8: facts block overrides prose ---
  {
    const marked = join(root, "marked");
    initRepo(marked);
    write(marked, "CLAUDE.md", `# CLAUDE.md — Marked

## Tech Stack

- Language(s): WRONG
- Framework(s): WRONG

<!-- repo-knowledge:facts:start -->
\`\`\`yaml
languages: [Swift]
frameworks: [SwiftUI]
platform_hints: [iOS]
runtime_tooling: [Xcode]
package_managers: [SPM]
install_command: xcodebuild -resolvePackageDependencies
run_command: xcodebuild -scheme App
test_command: xcodebuild test
build_command: xcodebuild archive
\`\`\`
<!-- repo-knowledge:facts:end -->
`);
    run(["emit", marked]);
    const m = JSON.parse(readFileSync(join(marked, ".ono", "repo-knowledge.json"), "utf-8"));
    check("8 block: overrides prose languages", JSON.stringify(m.stack.languages) === JSON.stringify(["Swift"]), JSON.stringify(m.stack.languages));
    check("8 block: frameworks from block", JSON.stringify(m.stack.frameworks) === JSON.stringify(["SwiftUI"]));
    check("8 block: snake_case key maps to platformHints", JSON.stringify(m.stack.platformHints) === JSON.stringify(["iOS"]));
    check("8 block: scalar command parsed", m.commands.run === "xcodebuild -scheme App", String(m.commands.run));
    check("8 block: coverage.commands populated", m.coverage.commands === "populated", m.coverage.commands);
    check("8 block: coverage.stack populated", m.coverage.stack === "populated");
  }

  // --- Scenario 9: malformed input degrades to unknown, never to a wrong value (K1) ---
  {
    const messy = join(root, "messy");
    initRepo(messy);
    write(messy, "CLAUDE.md", `# CLAUDE.md — Messy

## Tech Stack

The stack is described in prose rather than the template bullets.

<!-- repo-knowledge:facts:start -->
this is not parseable at all
<!-- repo-knowledge:facts:end -->
`);
    const r = run(["emit", messy]);
    check("9 messy: still exits 0", r.code === 0, `code=${r.code} stderr=${r.stderr}`);
    const m = JSON.parse(readFileSync(join(messy, ".ono", "repo-knowledge.json"), "utf-8"));
    check("9 messy: coverage.stack unknown", m.coverage.stack === "unknown", m.coverage.stack);
    check("9 messy: coverage.commands unknown", m.coverage.commands === "unknown", m.coverage.commands);
    check("9 messy: languages empty, not guessed", JSON.stringify(m.stack.languages) === JSON.stringify([]));
    check("9 messy: unparsed placeholder not persisted", !JSON.stringify(m).includes("{{"));
  }

  // --- Scenario 10: unreplaced template placeholders are ignored ---
  {
    const tmpl = join(root, "tmpl");
    initRepo(tmpl);
    write(tmpl, "CLAUDE.md", `# CLAUDE.md

## Tech Stack

- Language(s): {{LANGUAGES}}
- Framework(s): {{FRAMEWORKS}}
`);
    run(["emit", tmpl]);
    const m = JSON.parse(readFileSync(join(tmpl, ".ono", "repo-knowledge.json"), "utf-8"));
    check("10 placeholder: languages empty", JSON.stringify(m.stack.languages) === JSON.stringify([]));
    check("10 placeholder: coverage.stack unknown", m.coverage.stack === "unknown");
  }

  // --- Scenario 11: an empty-valued bullet must not swallow the following line ---
  // JS `\s` matches newlines, so a naive `:\s*(.+)$` fabricates a value for a
  // label that has none. Honest coverage requires the label be left absent.
  {
    const empty = join(root, "emptybullet");
    initRepo(empty);
    write(empty, "CLAUDE.md", `# CLAUDE.md

## Tech Stack

- Language(s):
- Framework(s): React Native
`);
    run(["emit", empty]);
    const m = JSON.parse(readFileSync(join(empty, ".ono", "repo-knowledge.json"), "utf-8"));
    check("11 empty bullet: languages empty, next line not swallowed", JSON.stringify(m.stack.languages) === JSON.stringify([]), JSON.stringify(m.stack.languages));
    check("11 empty bullet: following label still parsed", JSON.stringify(m.stack.frameworks) === JSON.stringify(["React Native"]), JSON.stringify(m.stack.frameworks));
    check("11 empty bullet: no label text leaked into the manifest", !JSON.stringify(m).includes("Framework(s)"));
  }

  // --- Scenario 12: an unterminated list must be skipped, letting prose survive ---
  // Falling through to the scalar branch would comma-split the fragment AND
  // displace the correct prose value, since a non-empty block value wins.
  {
    const badlist = join(root, "badlist");
    initRepo(badlist);
    write(badlist, "CLAUDE.md", `# CLAUDE.md

## Tech Stack

- Language(s): TypeScript

<!-- repo-knowledge:facts:start -->
languages: [Swift, Kotlin
<!-- repo-knowledge:facts:end -->
`);
    run(["emit", badlist]);
    const m = JSON.parse(readFileSync(join(badlist, ".ono", "repo-knowledge.json"), "utf-8"));
    check("12 bad list: malformed key skipped, prose fallback survives", JSON.stringify(m.stack.languages) === JSON.stringify(["TypeScript"]), JSON.stringify(m.stack.languages));
    check("12 bad list: no fragment persisted", !JSON.stringify(m).includes("[Swift"));
    check("12 bad list: no partial list item persisted", !JSON.stringify(m).includes("Kotlin"));
  }
```

- [ ] **Step 2: Run the test to verify the new scenarios fail**

```bash
cd ono-plugin-project-inspector && bun scripts/repo-knowledge.test.ts
```

Expected: scenarios 1–6 PASS (from T1); scenarios 7, 8, 10 FAIL because `extractFacts` returns empty values. Scenario 9 passes already — that is correct and expected, since the stub's behavior for malformed input is the same as the final behavior.

- [ ] **Step 3: Write the implementation**

In `ono-plugin-project-inspector/scripts/repo-knowledge.ts`, replace the entire `extractFacts` stub with:

```ts
/**
 * Parse the `<!-- repo-knowledge:facts:start/end -->` block that
 * project-analysis emits in CLAUDE.md. Deliberately a strict, tiny subset of
 * YAML — `key: scalar` and `key: [a, b, c]` — so no dependency is needed and
 * so anything unexpected yields no value rather than a wrong one.
 * Returns null when the block is absent (every pre-0.9.0 CLAUDE.md).
 */
export function parseFactsBlock(md: string): Record<string, string | string[]> | null {
  const m = md.match(
    /<!--\s*repo-knowledge:facts:start\s*-->([\s\S]*?)<!--\s*repo-knowledge:facts:end\s*-->/
  );
  if (!m) return null;
  const body = m[1].replace(/```[a-z]*/gi, "");
  const out: Record<string, string | string[]> = {};
  for (const raw of body.split("\n")) {
    const line = raw.trim();
    if (!line || line.startsWith("#")) continue;
    const kv = line.match(/^([a-z0-9_]+)\s*:\s*(.*)$/i);
    if (!kv) continue;
    const key = kv[1].toLowerCase();
    const value = kv[2].trim();
    if (!value || value.startsWith("{{")) continue;
    if (value.startsWith("[")) {
      // A value that opens a list but never closes it is malformed. Skip the
      // key entirely rather than falling through to the scalar branch, where
      // toList() would comma-split it into fabricated fragments AND displace a
      // correct prose-fallback value (the block wins whenever it is non-empty).
      if (!value.endsWith("]")) continue;
      const items = value
        .slice(1, -1)
        .split(",")
        .map((s) => s.trim().replace(/^["']|["']$/g, ""))
        .filter((s) => s.length > 0 && !/^unknown$/i.test(s));
      if (items.length) out[key] = items;
    } else {
      const scalar = value.replace(/^["']|["']$/g, "");
      if (!/^unknown$/i.test(scalar)) out[key] = scalar;
    }
  }
  return out;
}

/** Normalize an extracted value to a sorted, de-duplicated string list. */
function toList(value: string | string[] | undefined): string[] {
  if (!value) return [];
  const items = Array.isArray(value) ? value : value.split(/,\s*/);
  return Array.from(
    new Set(
      items
        .map((s) => s.trim())
        .filter((s) => s.length > 0 && !/^unknown$/i.test(s) && !s.startsWith("{{"))
    )
  ).sort();
}

/**
 * Fallback extraction from the fixed `## Tech Stack` bullet labels in
 * project-analysis's CLAUDE.md template. This is what lets an
 * already-inspected repository produce a useful manifest without re-running
 * the analysis stage.
 */
export function extractStackFromProse(md: string): Partial<RepoKnowledge["stack"]> {
  const labels: Array<[string, keyof RepoKnowledge["stack"]]> = [
    ["Language\\(s\\)", "languages"],
    ["Framework\\(s\\)", "frameworks"],
    ["Platform\\(s\\)", "platformHints"],
    ["Runtime / Tooling", "runtimeTooling"],
    ["Package manager\\(s\\)", "packageManagers"],
  ];
  const out: Partial<RepoKnowledge["stack"]> = {};
  for (const [label, key] of labels) {
    // `[ \t]*` and `[^\n]+` deliberately, NOT `\s*` and `(.+)`: JS `\s` matches
    // newlines, so `\s*:\s*(.+)$` on an empty-valued bullet swallows the NEXT
    // line and fabricates a value for this label — violating honest coverage.
    const m = md.match(new RegExp(`^-\\s*${label}\\s*:[ \\t]*([^\\n]+)$`, "im"));
    if (!m) continue;
    const list = toList(m[1]);
    if (list.length) out[key] = list;
  }
  return out;
}

/**
 * Fallback extraction from the fixed fenced bash block under
 * `## Build, Run, and Test Commands`. Each command sits on the line after a
 * known comment. "Unknown" is treated as absent, per the template's own rule.
 */
export function extractCommandsFromProse(md: string): RepoKnowledge["commands"] {
  const result: RepoKnowledge["commands"] = { install: null, run: null, test: null, build: null };
  const block = md.match(/##\s*Build, Run, and Test Commands[\s\S]*?```bash\n([\s\S]*?)```/i);
  if (!block) return result;
  const pairs: Array<[RegExp, keyof RepoKnowledge["commands"]]> = [
    [/#\s*Install dependencies\s*\n([^\n]*)/i, "install"],
    [/#\s*Run \/ develop\s*\n([^\n]*)/i, "run"],
    [/#\s*Test\s*\n([^\n]*)/i, "test"],
    [/#\s*Build\s*\n([^\n]*)/i, "build"],
  ];
  for (const [re, key] of pairs) {
    const found = block[1].match(re);
    const value = found?.[1]?.trim();
    if (!value || /^unknown$/i.test(value) || value.startsWith("{{") || value.startsWith("#")) continue;
    result[key] = value;
  }
  return result;
}

/**
 * Resolve structured facts from CLAUDE.md. The facts block wins where present;
 * the prose fallback fills the rest. A field that cannot be resolved is left
 * empty, which drives its `coverage` to "unknown" so the consumer derives it
 * live rather than trusting a guess.
 */
function extractFacts(claudeMd: string | null): {
  stack: RepoKnowledge["stack"];
  commands: RepoKnowledge["commands"];
} {
  const empty = {
    stack: { languages: [], frameworks: [], platformHints: [], runtimeTooling: [], packageManagers: [] },
    commands: { install: null, run: null, test: null, build: null },
  };
  if (!claudeMd) return empty;

  const prose = extractStackFromProse(claudeMd);
  const block = parseFactsBlock(claudeMd) ?? {};

  // Block keys are snake_case; map them onto the manifest's camelCase fields.
  const pick = (blockKey: string, proseValue: string[] | undefined): string[] => {
    const fromBlock = toList(block[blockKey]);
    return fromBlock.length ? fromBlock : proseValue ?? [];
  };

  const stack: RepoKnowledge["stack"] = {
    languages: pick("languages", prose.languages),
    frameworks: pick("frameworks", prose.frameworks),
    platformHints: pick("platform_hints", prose.platformHints),
    runtimeTooling: pick("runtime_tooling", prose.runtimeTooling),
    packageManagers: pick("package_managers", prose.packageManagers),
  };

  const proseCommands = extractCommandsFromProse(claudeMd);
  const scalar = (blockKey: string, fallback: string | null): string | null => {
    const v = block[blockKey];
    if (typeof v === "string" && v.length) return v;
    return fallback;
  };

  const commands: RepoKnowledge["commands"] = {
    install: scalar("install_command", proseCommands.install),
    run: scalar("run_command", proseCommands.run),
    test: scalar("test_command", proseCommands.test),
    build: scalar("build_command", proseCommands.build),
  };

  return { stack, commands };
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
cd ono-plugin-project-inspector && bun scripts/repo-knowledge.test.ts
```

Expected: `ALL TESTS PASSED`, all ten scenarios.

- [ ] **Step 5: Verify determinism explicitly**

```bash
cd ono-plugin-project-inspector && bun scripts/repo-knowledge.test.ts 2>&1 | grep "determinism"
```

Expected: `PASS  2 determinism: byte-identical ignoring generatedAt`

- [ ] **Step 6: Commit**

```bash
git add scripts/repo-knowledge.ts scripts/repo-knowledge.test.ts
git commit -m "feat(repo-knowledge): extract stack and commands from facts block with prose fallback"
```

---

## Task 3: The `repo-knowledge` skill, registry entry, and plugin manifest

**Files:**
- Create: `ono-plugin-project-inspector/skills/repo-knowledge/SKILL.md`
- Modify: `ono-plugin-project-inspector/skills/registry.json` (append one entry to `skills[]`)
- Modify: `ono-plugin-project-inspector/.claude-plugin/plugin.json` (`skills[]`)

**Interfaces:**
- Consumes: `scripts/repo-knowledge.ts` CLI from T1/T2.
- Produces: a registry entry with `id: "repo-knowledge"`, `type: "internal"`, `autoInvoke: true`, `produces: [".ono/repo-knowledge.json"]`. T4's agent and hook edits reference this id.

**Why `type: internal`:** `inspection-state` already established this shape for auto-invoked infrastructure that is not a workflow stage, has no approval gate, and has no command. `scripts/inspection-state.ts:102` filters stages on `type === "workflow" && workflowRole === "inspection"`, so an `internal` entry is automatically excluded from stage derivation, completion, and resume — meaning **no change to `inspection-state.ts` is needed**. This is the "no registry redesign" constraint being honored.

- [ ] **Step 1: Create the skill**

Create `ono-plugin-project-inspector/skills/repo-knowledge/SKILL.md`:

```markdown
---
name: repo-knowledge
description: >-
  Internal infrastructure skill (not user-facing). Emits and refreshes the
  repository-knowledge manifest at <repo>/.ono/repo-knowledge.json via the
  deterministic scripts/repo-knowledge.ts helper, so downstream Ono plugins
  consume canonical repository knowledge instead of re-deriving it. The
  manifest is DERIVED from artifacts the workflow has already produced and a
  human has already approved — CLAUDE.md, AUDIT.md, and docs/project/*.md. It
  performs no repository analysis, reads no source files, and never modifies
  CLAUDE.md, AUDIT.md, audits/, docs/, or source code.
---

# Repo Knowledge Skill

## Type

**Internal.** This skill has no command and is never invoked directly by a developer. The `project-inspector` agent invokes it automatically whenever repository knowledge changes (see "When the Agent Invokes This"). In `skills/registry.json` it is `type: internal` with `autoInvoke: true`, so it is not a workflow stage, has no approval gate, and is excluded from both the linear inspection loop and on-demand maintenance.

## Purpose

Give downstream Ono plugins one deterministic, versioned, fingerprinted entry point to this repository's approved knowledge, so they stop re-deriving it. Without this manifest, every consumer re-scans the repository and reaches its own conclusions, which then diverge from the human-approved artifacts.

## Source of Truth

The approved artifacts remain the source of truth. This manifest is an **index over them**:

- `CLAUDE.md` — stack, commands, structure pointers.
- `AUDIT.md` — the audit-topic index (topic, status, file). **Bodies of audit files are never read.**
- `docs/project/*.md` — pointers plus heading anchors for the inventory, conventions, and integrations knowledge bases.

Prose is never copied into the manifest. Consumers receive a path and an anchor and read the artifact themselves.

## Output Contract

This skill may create or modify only:

```text
<repository-root>/.ono/repo-knowledge.json
```

It never writes `CLAUDE.md`, `AUDIT.md`, `audits/`, `docs/`, `.ono/state.json`, or any source file. All reads/writes go through the deterministic helper — never hand-edit the JSON.

## The Deterministic Helper

All logic lives in `scripts/repo-knowledge.ts` (run with a TypeScript runner, e.g. `bun scripts/repo-knowledge.ts <command> <repo-root>`). Commands:

| Command | Purpose |
|---------|---------|
| `emit <repo-root>` | Rebuild the manifest from the current approved artifacts and write it. Idempotent. |
| `validate <repo-root>` | Structural validation of an existing manifest. Exit 2 if absent or invalid. |
| `show <repo-root>` | Print the manifest (read-only). |

Always pass the orchestrator's resolved absolute `TARGET_ROOT`. The helper refuses to operate on any path containing `.claude/worktrees/` and exits non-zero.

## Coverage and Graceful Degradation

Every knowledge category carries a `coverage` value of `populated`, `partial`, or `unknown`. A category is `unknown` when its source artifact is missing or its content could not be parsed deterministically. **This is a normal state, not an error.** A repository inspected before this skill existed has no `repo-knowledge:facts` block in `CLAUDE.md`, so some fields resolve from the template's prose bullets and others report `unknown`. Consumers are contractually required to derive `unknown` categories themselves.

Never fabricate a value to fill a category. An honest `unknown` leaves a consumer no worse off than before this manifest existed; a guessed value makes it worse.

## When the Agent Invokes This

The agent calls this skill automatically — the developer never asks for it. Refresh at every point where the approved knowledge changes:

1. **After `project-analysis`** — `CLAUDE.md` and `AUDIT.md` now exist.
2. **After `project-docs`** — the `docs/project/` pointers and anchors now resolve.
3. **After each `audit-approve`** — the audit-topic index changed.
4. **After `audit-sync` maintenance** — `audit-sync` rewrites `CLAUDE.md`'s managed blocks, changing its hash; without a refresh consumers would see a false staleness verdict.

Invoking this skill is bookkeeping, not workflow advancement, and never needs developer approval.

## Portability

The manifest is committed to Git alongside `.ono/state.json`, so a teammate who has not run `/inspect` still benefits from the approved knowledge. It stores only repo-relative paths, content hashes, and the git HEAD SHA — never an absolute filesystem path. This skill does not modify the repository's `.gitignore`.

## Hard Constraints

- Only create or modify `<repository-root>/.ono/repo-knowledge.json`.
- Never read repository source files. Only `CLAUDE.md`, `AUDIT.md`, and `docs/project/*.md`.
- Never read the body of an `audits/*.md` file — the manifest carries the topic index only.
- Never copy prose into the manifest; emit a path and an anchor instead.
- Never fabricate a value for a category that could not be parsed — report `unknown`.
- Never treat the manifest as authoritative over the artifacts it indexes.
- Always go through `scripts/repo-knowledge.ts`; never hand-edit the JSON.
- Never assume the current working directory is the target repository — the root is always the orchestrator's resolved `TARGET_ROOT`, and never a `.claude/worktrees/` path.
```

- [ ] **Step 2: Add the registry entry**

In `ono-plugin-project-inspector/skills/registry.json`, insert this object into `skills[]` immediately after the `inspection-state` entry (keeping internal infrastructure grouped before the workflow stages):

```json
    {
      "id": "repo-knowledge",
      "enabled": true,
      "type": "internal",
      "autoInvoke": true,
      "stage": 0,
      "path": "skills/repo-knowledge",
      "produces": [".ono/repo-knowledge.json"],
      "requires": [],
      "repeatable": true,
      "requiresApproval": false,
      "hooks": {},
      "notes": "Internal infrastructure skill (not user-facing, no command). Owns <repo>/.ono/repo-knowledge.json via the deterministic scripts/repo-knowledge.ts helper. The manifest is DERIVED from already-approved artifacts (CLAUDE.md, AUDIT.md, docs/project/*.md) so downstream Ono plugins consume canonical repository knowledge instead of re-deriving it; it reads no source files and never reads audit bodies. The agent invokes it automatically after project-analysis, after project-docs, after each audit-approve, and after audit-sync maintenance. Excluded from the linear inspection loop and from maintenance because type is internal."
    },
```

- [ ] **Step 3: Register the skill in the plugin manifest**

In `ono-plugin-project-inspector/.claude-plugin/plugin.json`, add `"./skills/repo-knowledge"` to `skills[]` immediately after `"./skills/inspection-state"`:

```json
  "skills": [
    "./skills/inspection-state",
    "./skills/repo-knowledge",
    "./skills/project-analysis",
    "./skills/project-docs",
    "./skills/audit-breakdown",
    "./skills/audit-approve",
    "./skills/audit-sync"
  ],
```

- [ ] **Step 4: Verify the registry still parses and stage derivation is unaffected**

```bash
cd ono-plugin-project-inspector
node -e 'const r=require("./skills/registry.json"); const ids=r.skills.map(s=>s.id); console.log("ids:",ids.join(",")); const stages=r.skills.filter(s=>s.enabled!==false&&s.type==="workflow"&&s.workflowRole==="inspection").map(s=>s.id); console.log("inspection stages:",stages.join(",")); if(!ids.includes("repo-knowledge")) process.exit(1); if(stages.length!==4) process.exit(1); console.log("OK");'
node -e 'JSON.parse(require("fs").readFileSync(".claude-plugin/plugin.json","utf8")); console.log("plugin.json OK");'
```

Expected: `inspection stages: project-analysis,project-docs,audit-breakdown,audit-approve` — unchanged, four entries — then `OK` and `plugin.json OK`. This is the assertion that the new entry did not alter stage derivation, completion, or resume.

- [ ] **Step 5: Confirm the existing test suite still passes**

```bash
cd ono-plugin-project-inspector && bun scripts/repo-knowledge.test.ts && bun scripts/slugify.ts
```

Expected: `ALL TESTS PASSED`, then four `PASS` lines from `slugify`.

- [ ] **Step 6: Commit**

```bash
git add skills/repo-knowledge/SKILL.md skills/registry.json .claude-plugin/plugin.json
git commit -m "feat(repo-knowledge): register internal repo-knowledge skill"
```

---

## Task 4: Agent checkpoints, hook verification, and `/inspect-sync`

**Files:**
- Modify: `ono-plugin-project-inspector/agents/project-inspector.md`
- Modify: `ono-plugin-project-inspector/hooks/after-project-analysis.md`
- Modify: `ono-plugin-project-inspector/hooks/after-project-docs.md`
- Modify: `ono-plugin-project-inspector/hooks/after-audit-approve.md`
- Modify: `ono-plugin-project-inspector/hooks/after-audit-sync.md`
- Modify: `ono-plugin-project-inspector/commands/inspect-sync.md`

**Interfaces:**
- Consumes: the `repo-knowledge` skill id from T3; `scripts/repo-knowledge.ts emit|validate`.
- Produces: the manifest is now actually emitted during a real workflow run. Nothing consumes it yet.

**Why these five files and no others:** the agent is the only component that invokes skills, so without it the skill never runs. The four hooks are the existing checkpoints for exactly the four moments knowledge changes. `before-inspect.md` is deliberately **not** modified — the manifest must not be written during read-only startup detection, preserving the "choosing *leave everything unchanged* writes nothing" guarantee from v0.7.0.

- [ ] **Step 1: Add the agent's invocation contract**

In `ono-plugin-project-inspector/agents/project-inspector.md`, insert a new section immediately after the `## Inspection State (internal, auto-invoked)` section (which currently ends with the line beginning `` `AUDIT.md` remains the source of truth ``):

```markdown
## Repo Knowledge (internal, auto-invoked)

`repo-knowledge` (`type: internal`, `autoInvoke: true`) maintains `<repo>/.ono/repo-knowledge.json` through the deterministic `scripts/repo-knowledge.ts` helper. It is the canonical, versioned index over the knowledge this workflow has already produced and the developer has already approved, so downstream Ono plugins consume it instead of re-deriving repository knowledge. Always invoke it with `TARGET_ROOT`, never a raw CWD — the helper refuses to operate on a `.claude/worktrees/` path.

You invoke it automatically — never on developer request, never as a stage:

- **After `project-analysis`** — `CLAUDE.md` and `AUDIT.md` now exist.
- **After `project-docs`** — the `docs/project/` pointers now resolve.
- **After each `audit-approve`** — the audit-topic index changed.
- **After `audit-sync`** — the `CLAUDE.md` managed blocks changed, so its fingerprint changed.

Invoke it **alongside** `inspection-state` (`sync`) at those checkpoints, not instead of it: the two own different files and neither replaces the other. Invoking it is bookkeeping, not workflow advancement, and never needs developer approval. Do **not** invoke it during startup detection — startup must remain read-only.

The approved artifacts remain the source of truth; the manifest only indexes them. A category the helper reports as `unknown` is a normal state and must never be presented to the developer as a failure.
```

- [ ] **Step 2: Add the emit + verify step to the four hooks**

In `hooks/after-project-analysis.md`, insert as a new item immediately after item 3 (the AUDIT.md topic-count item), renumbering the subsequent items to 5 and 6:

```markdown
4. **Refresh the repository-knowledge manifest.** Invoke the internal `repo-knowledge` skill (`emit`) with `TARGET_ROOT`, then verify it landed at the real root: `bun scripts/verify-artifacts.ts <TARGET_ROOT> .ono/repo-knowledge.json`. On exit 2 STOP (the root is a worktree). Then run `bun scripts/repo-knowledge.ts validate <TARGET_ROOT>`; on exit 2 report that the manifest is missing or malformed and do not report the stage complete. If `validate` reports any category as `coverage unknown`, mention which ones once in your stage-transition report — that is informational, not a failure, and it tells the developer which knowledge downstream plugins will still derive themselves.
```

In `hooks/after-project-docs.md`, insert after item 3 (the `CLAUDE.md`-must-exist item), renumbering the rest:

```markdown
4. **Refresh the repository-knowledge manifest.** Invoke `repo-knowledge` (`emit`) with `TARGET_ROOT`. As a location backstop, confirm the manifest landed at the real root — run `bun scripts/verify-artifacts.ts <TARGET_ROOT> .ono/repo-knowledge.json`; on exit 2 STOP (the emit ran against a `.claude/worktrees/` path, not the repository). Then run `bun scripts/repo-knowledge.ts validate <TARGET_ROOT>`; on exit 2 report that the manifest is missing or malformed and do not report this step complete. Three of the four `docs/project/` files back a manifest coverage category — `components.md` → `inventory`, `patterns.md` → `conventions`, `integrations.md` → `integrations` — so those categories only become usable to downstream plugins after this refresh. `overview.md` is indexed as a pointer but has no coverage category of its own. Report any category still `unknown`.
```

In `hooks/after-audit-approve.md`, insert after item 4 (the `File`-reference item), renumbering the rest:

```markdown
5. **Refresh the repository-knowledge manifest.** Invoke `repo-knowledge` (`emit`) with `TARGET_ROOT` so the manifest's audit-topic index reflects the new `Approved` status. As a location backstop, confirm the manifest landed at the real root — run `bun scripts/verify-artifacts.ts <TARGET_ROOT> .ono/repo-knowledge.json`; on exit 2 STOP (the emit ran against a `.claude/worktrees/` path, not the repository). Then run `bun scripts/repo-knowledge.ts validate <TARGET_ROOT>`; on exit 2 report that the manifest is missing or malformed and do not report this step complete. The manifest carries the topic index only — topic, status, and file path — never the audit's findings, so this is a cheap index refresh and not a re-read of the audit document.
```

In `hooks/after-audit-sync.md`, insert after item 3 (the full-regeneration item), renumbering the rest:

```markdown
4. **Refresh the repository-knowledge manifest.** `audit-sync` rewrites the two managed blocks inside `CLAUDE.md`, which changes that file's content hash. Invoke `repo-knowledge` (`emit`) with `TARGET_ROOT` so the manifest's fingerprint matches the file on disk — without this, every downstream consumer would report a false `stale-artifacts` verdict for `CLAUDE.md` after a routine sync. As a location backstop, confirm the manifest landed at the real root — run `bun scripts/verify-artifacts.ts <TARGET_ROOT> .ono/repo-knowledge.json`; on exit 2 STOP (the emit ran against a `.claude/worktrees/` path, not the repository). Then run `bun scripts/repo-knowledge.ts validate <TARGET_ROOT>`; on exit 2 report that the manifest is missing or malformed and do not report this step complete.
```

- [ ] **Step 3: Document the on-demand refresh path**

In `ono-plugin-project-inspector/commands/inspect-sync.md`, append this paragraph immediately before the final `$ARGUMENTS` line:

```markdown
This command is also the on-demand way to refresh `<repo>/.ono/repo-knowledge.json` — the canonical repository-knowledge manifest that downstream Ono plugins read instead of re-analyzing the repository. The agent refreshes it automatically after every stage that changes repository knowledge; running `/inspect-sync` regenerates it for a repository that was inspected by an earlier plugin version and therefore has no manifest yet. Regeneration is idempotent and derives only from artifacts already approved — it re-reads no source code and re-runs no inspection stage.
```

- [ ] **Step 4: Verify the hook and agent edits are consistent**

```bash
cd ono-plugin-project-inspector
grep -c "repo-knowledge" agents/project-inspector.md hooks/after-project-analysis.md hooks/after-project-docs.md hooks/after-audit-approve.md hooks/after-audit-sync.md commands/inspect-sync.md
grep -c "repo-knowledge" hooks/before-inspect.md
```

Expected: a non-zero count for each of the six files, then `0` for `before-inspect.md` — the assertion that startup remains read-only.

- [ ] **Step 5: Confirm hook item numbering is sequential**

```bash
cd ono-plugin-project-inspector
for f in hooks/after-project-analysis.md hooks/after-project-docs.md hooks/after-audit-approve.md hooks/after-audit-sync.md; do echo "== $f"; grep -oE "^[0-9]+\." "$f" | tr -d '.' | tr '\n' ' '; echo; done
```

Expected: each file prints a contiguous ascending run starting at 1 with no repeats — `1 2 3 4 5 6` for `after-project-analysis.md`, and similarly for the others.

- [ ] **Step 6: Commit**

```bash
git add agents/project-inspector.md hooks/ commands/inspect-sync.md
git commit -m "feat(repo-knowledge): emit and verify the manifest at every knowledge checkpoint"
```

---

## Task 5: `project-analysis` facts block, integrations pointer, contract doc, and release

**Files:**
- Modify: `ono-plugin-project-inspector/skills/project-analysis/SKILL.md`
- Create: `ono-plugin-project-inspector/docs/repo-knowledge-contract.md`
- Modify: `ono-plugin-project-inspector/docs/architecture.md`
- Modify: `ono-plugin-project-inspector/docs/plugin-workflow.md`
- Modify: `ono-plugin-project-inspector/README.md`
- Modify: `ono-plugin-project-inspector/CHANGELOG.md`
- Modify: `ono-plugin-project-inspector/.claude-plugin/plugin.json` (version)
- Modify: `ono-plugin-project-inspector/.claude-plugin/marketplace.json` (version)

**Interfaces:**
- Consumes: `parseFactsBlock` from T2 — the block written here must parse under it.
- Produces: `docs/repo-knowledge-contract.md` at schema v1, which T7 duplicates into the mobile-dev plugin verbatim below the title line.

**Why the integrations pointer is included here:** `CLAUDE.md`'s `## External Integrations` section and `docs/project/integrations.md` state the same facts at different depths with no link between them — duplication finding D4. It is the one duplication *internal to the inspector* that this MVP can fix in a two-line template change, and it directly serves scope point 1 (single producer). Nothing else in the review's D1–D14 list is touched here.

- [ ] **Step 1: Add the facts block to the `CLAUDE.md` template**

In `ono-plugin-project-inspector/skills/project-analysis/SKILL.md`, inside the `### CLAUDE.md Template` fenced block, replace the `## Tech Stack` section:

```markdown
## Tech Stack

- Language(s): {{LANGUAGES}}
- Framework(s): {{FRAMEWORKS}}
- Platform(s): {{PLATFORMS}}
- Runtime / Tooling: {{RUNTIME_TOOLING}}
- Package manager(s): {{PACKAGE_MANAGERS}}
```

with:

```markdown
## Tech Stack

- Language(s): {{LANGUAGES}}
- Framework(s): {{FRAMEWORKS}}
- Platform(s): {{PLATFORMS}}
- Runtime / Tooling: {{RUNTIME_TOOLING}}
- Package manager(s): {{PACKAGE_MANAGERS}}

<!-- repo-knowledge:facts:start -->
```yaml
languages: [{{LANGUAGES}}]
frameworks: [{{FRAMEWORKS}}]
platform_hints: [{{PLATFORMS}}]
runtime_tooling: [{{RUNTIME_TOOLING}}]
package_managers: [{{PACKAGE_MANAGERS}}]
install_command: {{INSTALL_COMMAND}}
run_command: {{RUN_COMMAND}}
test_command: {{TEST_COMMAND}}
build_command: {{BUILD_COMMAND}}
```
<!-- repo-knowledge:facts:end -->
```

- [ ] **Step 2: Add the authoring rules for the block**

In the same file, in `## Step 6: Generate CLAUDE.md`, append to the `Rules:` list, immediately after the existing bullet about preserving the `audit-sync` markers:

```markdown
- Preserve the `<!-- repo-knowledge:facts:start -->` / `<!-- repo-knowledge:facts:end -->` marker pair exactly as written, and populate the block with the **same values** you wrote into the prose sections above it. This block is the machine-readable form of the Tech Stack and Commands sections — it exists so the `repo-knowledge` skill can index them deterministically instead of parsing prose. Rules for it:
  - Use only `key: value` and `key: [a, b, c]` — no nesting, no multi-line values, no comments.
  - List values are comma-separated inside square brackets. A single value is still a valid list: `languages: [Swift]`.
  - Write `Unknown` for any value you could not determine confidently. Never guess — `Unknown` is read as "not known" and the consumer derives it itself, whereas a wrong value silently misleads every downstream plugin.
  - Never write a secret, credential, or environment-variable **value** here. Variable names only, consistent with this skill's other constraints.
  - Do not add keys beyond the nine shown in the template.
```

- [ ] **Step 3: Replace the duplicated integrations body with a pointer**

In the same `### CLAUDE.md Template` block, replace:

```markdown
## External Integrations

{{EXTERNAL_INTEGRATIONS}}
```

with:

```markdown
## External Integrations

{{EXTERNAL_INTEGRATIONS_SUMMARY}}

Full inventory: [`docs/project/integrations.md`](docs/project/integrations.md) — generated by the `project-docs` stage. That file is the authoritative list of backend services, SDKs, auth, analytics, push, payments, and environment-variable names; this section is a two-to-four-line summary that points at it rather than restating it.
```

And in `## Step 6: Generate CLAUDE.md`'s `Rules:` list, append:

```markdown
- Keep `## External Integrations` to a two-to-four-line summary plus the pointer to `docs/project/integrations.md`. The full inventory belongs to `project-docs`; restating it here creates two sources of truth for the same facts that then drift apart.
```

- [ ] **Step 4: Write the contract document**

Create `ono-plugin-project-inspector/docs/repo-knowledge-contract.md`:

```markdown
# Repository Knowledge Contract

**Schema version: 1**
Producer: `ono-project-inspector` (this plugin), via `skills/repo-knowledge` and `scripts/repo-knowledge.ts`.
Consumers: `ono-mobile-dev-plugin` (and future Ono plugins), via their own deterministic reader.

> This file is duplicated verbatim (below the title line) in every plugin that participates in the contract. Claude Code has no cross-plugin dependency mechanism, so the specification is vendored the same way `resolve-target-repo-root.ts` is. **When this file changes, change every copy in the same release.**

## The file

`<repository-root>/.ono/repo-knowledge.json` — committed to Git, portable, machine-generated.

## Guarantees the producer makes

1. **Derived, never authored.** Built only from `CLAUDE.md`, `AUDIT.md`, and `docs/project/*.md` — artifacts this plugin's workflow already produced and a human already approved. No repository source is read. No audit file body is read.
2. **Deterministic.** Identical inputs produce a byte-identical file except `generatedAt`. All arrays are sorted and de-duplicated.
3. **Portable.** Only repo-relative paths, content hashes, and the git HEAD SHA. Never an absolute filesystem path.
4. **Pointers, not copies.** Prose stays in the artifact. The manifest carries a path and heading anchors.
5. **Honest coverage.** A category that could not be parsed is reported `unknown`, never guessed.
6. **Additive versioning.** New optional fields keep `repoKnowledgeSchemaVersion: 1`. A breaking shape change bumps it.

## Obligations the consumer accepts

1. **Read-only.** Never write `.ono/repo-knowledge.json`, `CLAUDE.md`, `AUDIT.md`, `docs/project/**`, `audits/**`, or `.ono/state.json`.
2. **One reader.** Exactly one component per consumer plugin parses the manifest. Commands and agents receive its normalized output.
3. **Cite, do not copy.** Downstream documents record `{ path, anchor, fingerprint }`, never a verbatim paste of repository knowledge.
4. **Degrade, never fail.** An absent, malformed, stale, or too-new manifest is never fatal. Fall back to live derivation.
5. **Do not re-derive a covered, fresh category.**
6. **Derive any `unknown` category yourself.**
7. **Report drift, never repair it.** Recommend `/inspect` or `/inspect-sync`; never regenerate a producer-owned artifact.
8. **`platformHints` is advisory only.** It must never be used as the authoritative platform for routing or for a feature decision. The consumer runs its own platform detection and its own human confirmation gate regardless.

## Schema v1

See the `RepoKnowledge` interface in `scripts/repo-knowledge.ts` for the authoritative type. Field reference:

| Field | Type | Meaning |
|---|---|---|
| `repoKnowledgeSchemaVersion` | `1` | Contract version. A consumer supporting max version N treats `> N` as absent. |
| `producedBy` | `{plugin, version}` | Which plugin and version wrote it. |
| `generatedAt` | ISO-8601 | Emit time. The only field that varies between two emits over identical inputs. |
| `fingerprint.gitHead` | sha \| null | HEAD at emit time. Differs from current HEAD → `stale-head`. |
| `fingerprint.artifacts` | `{relpath: sha256 \| null}` | Per-source-document hash. `null` = the document did not exist. A mismatch → `stale-artifacts` for the categories that document backs. |
| `coverage.<category>` | `populated \| partial \| unknown` | Per-category trust for `stack`, `commands`, `structure`, `inventory`, `conventions`, `integrations`, `auditTopics`. |
| `stack` | `{languages, frameworks, platformHints, runtimeTooling, packageManagers}` | Sorted string lists. `platformHints` is **advisory only**. |
| `commands` | `{install, run, test, build}` | Strings or `null`. `null` means not known. |
| `structure` | `{repositoryTree, keyModules, entryPoints}` | `CLAUDE.md#anchor` pointers, or `null` when `CLAUDE.md` is absent. |
| `documents.<key>` | `{path, exists, anchors}` | Keys: `claudeMd`, `auditMd`, `overview`, `inventory`, `conventions`, `integrations`. `anchors` holds GitHub-style heading anchors, re-derived on every emit, and is populated **only** for the four `docs/project/*` documents — `claudeMd` and `auditMd` always carry `[]` and are cited through `structure`'s fixed pointers instead. |
| `auditTopics[]` | `{topic, slug, status, file}` | Index over `AUDIT.md`'s topic table. **Index only — no findings.** |

**Not in v1:** audit findings / cautions, platform capability beyond `platformHints`, device targets, task lifecycle state. These are deliberate exclusions, not omissions; each is a separate project.

## Category → backing document

| Category | Backed by | Consumer uses it for |
|---|---|---|
| `stack` | `CLAUDE.md` facts block, else Tech Stack bullets | languages/frameworks/tooling context |
| `commands` | `CLAUDE.md` facts block, else the Commands fenced block | install/run/test/build |
| `structure` | `CLAUDE.md` | repository tree, key modules, entry points |
| `inventory` | `docs/project/components.md` | existing screens, components, hooks — reuse before creating |
| `conventions` | `docs/project/patterns.md` | state management, API, navigation, styling, errors, i18n, naming, testing |
| `integrations` | `docs/project/integrations.md` | services, SDKs, env-var names |
| `auditTopics` | `AUDIT.md` | which topics exist and their approval status |

## Freshness verdicts

| Verdict | Condition | Consumer behavior |
|---|---|---|
| `fresh` | `gitHead` equals current HEAD and every recorded hash matches | Trust all non-`unknown` categories. |
| `stale-head` | HEAD moved, all recorded hashes still match | Trust the manifest; report how far behind. |
| `stale-artifacts` | A recorded hash no longer matches | Trust unaffected categories; derive the affected ones live; report it. |
| `unknown` | `gitHead` is null, or git is unavailable | Trust the manifest; report that freshness could not be established. |
```

- [ ] **Step 5: Update the three documentation files**

In `docs/architecture.md`, append a new section at the end:

```markdown
## The outbound repository-knowledge contract

The plugin's knowledge is consumed by other Ono plugins, not just by future Claude Code sessions. Before `.ono/repo-knowledge.json` existed, every downstream plugin re-derived repository knowledge from source — `ono-mobile-dev-plugin`'s `repo-analyst` re-detected the stack and conventions on every command, then snapshotted the result verbatim into per-feature documents that went stale silently. The same facts were derived up to five times in two vocabularies with no reconciliation.

`skills/repo-knowledge` closes that gap by publishing one deterministic, versioned index over what this workflow has already produced and the developer has already approved. Three design rules keep it safe:

- **Derived, never authored.** It reads only `CLAUDE.md`, `AUDIT.md`, and `docs/project/*.md`. It never reads source, so it adds no analysis and no approval gate — the human review that gated those artifacts is the same review that gates the manifest.
- **Pointers, not copies.** Prose stays in the artifact; the manifest carries paths and heading anchors. The artifacts remain the source of truth, so there is no second place for the same fact to drift.
- **Honest coverage.** Every category reports `populated`, `partial`, or `unknown`. Consumers are contractually required to derive `unknown` categories themselves, which is what makes a repository inspected by an older plugin version — and a repository never inspected at all — keep working unchanged.

The structured half of the manifest comes from a `<!-- repo-knowledge:facts:start/end -->` block in `CLAUDE.md`, using the same managed-block idea as `audit-sync` but in the opposite direction: the skill *writes* prose plus a machine-readable mirror, and the deterministic helper *reads* the mirror. When the block is absent, the helper falls back to the template's fixed Tech Stack bullet labels and Commands fenced block. See `docs/repo-knowledge-contract.md` for the schema and the obligations a consumer accepts.
```

In `docs/plugin-workflow.md`, append after the `## Inspection state and resume` section:

```markdown
## Repository knowledge for downstream plugins

Alongside `.ono/state.json`, the agent keeps `<repo>/.ono/repo-knowledge.json` current via the internal `repo-knowledge` skill — after `project-analysis`, after `project-docs`, after each `audit-approve`, and after `audit-sync`. This is the canonical, versioned index other Ono plugins read instead of re-analyzing the repository: what the stack is, how to build and test it, where the modules and entry points are, and where to find the component inventory, the conventions, and the integrations. It carries pointers into the approved artifacts rather than copies of them, and reports any category it could not determine as `unknown` so a consumer derives that part itself.

The developer never invokes it directly. A repository inspected by an earlier plugin version gets its manifest on the next `/inspect-sync` — no re-inspection needed. Commit the file alongside `.ono/state.json` so the whole team benefits. See `docs/repo-knowledge-contract.md`.
```

In `README.md`, in the `## Status` list, add after the `inspection-state` line:

```markdown
- `repo-knowledge` — enabled (internal infrastructure, auto-invoked; not user-facing)
```

And in `README.md`'s `## Structure` block, change the `scripts/` line to:

```text
scripts/                     deterministic helpers (slug rules, AUDIT.md consistency, .ono/state.json state, .ono/repo-knowledge.json manifest)
```

- [ ] **Step 6: Bump versions and write the changelog**

In `.claude-plugin/plugin.json` set `"version": "0.9.0"`. In `.claude-plugin/marketplace.json` set the `ono-project-inspector` entry's `"version": "0.9.0"` — it currently reads `0.7.0` while `plugin.json` read `0.8.0`, so this also corrects pre-existing drift.

Prepend to `CHANGELOG.md`:

```markdown
## 0.9.0 — 2026-07-28

Made the plugin the single producer of repository knowledge for the Ono plugin ecosystem, by publishing a deterministic, versioned manifest downstream plugins consume instead of re-deriving repository facts from source.

- Added the internal `repo-knowledge` skill (registry `type: internal`, `autoInvoke: true`) and `scripts/repo-knowledge.ts` (`emit` / `validate` / `show`). It owns one file, `<repo>/.ono/repo-knowledge.json`.
- The manifest is **derived, never authored**: it indexes `CLAUDE.md`, `AUDIT.md`, and `docs/project/*.md` — artifacts the workflow already produced and a developer already approved. It reads no repository source and never reads an audit file's body, so it introduces no new analysis and no new approval gate.
- It carries **pointers, not copies** (paths plus heading anchors), so the approved artifacts remain the source of truth, plus a fingerprint (git HEAD and per-artifact SHA-256) and per-category `coverage` (`populated` / `partial` / `unknown`) so a consumer can tell what it may trust and what it must still derive itself.
- `project-analysis`'s `CLAUDE.md` template now emits a `<!-- repo-knowledge:facts:start/end -->` block mirroring the Tech Stack and Commands sections in a strict machine-readable subset, so those facts are indexed deterministically rather than parsed from prose. The helper falls back to the template's fixed bullet labels when the block is absent, so repositories inspected by earlier versions produce a manifest on the next `/inspect-sync` with no re-inspection.
- `project-analysis`'s `CLAUDE.md` template no longer restates the full external-integrations inventory; it now carries a short summary plus a pointer to `docs/project/integrations.md`, removing a duplication internal to this plugin.
- The agent invokes `repo-knowledge` (`emit`) alongside `inspection-state` (`sync`) after `project-analysis`, after `project-docs`, after each `audit-approve`, and after `audit-sync`. Startup detection remains read-only — `before-inspect` does not emit.
- All four affected `after-*` hooks verify the manifest at the real root via `verify-artifacts.ts` and run `repo-knowledge.ts validate`, reporting any `unknown` category as information rather than failure.
- `/inspect-sync` documents the on-demand refresh path. No new command.
- Added `docs/repo-knowledge-contract.md` — the schema and the obligations a consumer accepts, pinned to schema v1 and duplicated verbatim in every participating plugin.
- Added `scripts/repo-knowledge.test.ts`, the plugin's first automated test suite: fixture-based extraction, marker-block override, prose fallback, malformed-input degradation, byte-determinism, portability, and worktree refusal.
- Corrected `.claude-plugin/marketplace.json`, which pinned `0.7.0` against `plugin.json`'s `0.8.0`.
- No changes to the workflow shape, stage order, approval gates, the `Draft` → `Approved` single-writer rule, `AUDIT.md` as human source of truth, `inspection-state`, or `.ono/state.json`. No changes to any target repository's source code.
```

- [ ] **Step 7: Verify the facts block parses under the real parser**

```bash
cd ono-plugin-project-inspector
bun -e '
const fs = require("fs");
const md = fs.readFileSync("skills/project-analysis/SKILL.md","utf8");
// IMPORTANT: take the LAST start-marker occurrence. The first one is inside the
// Step 6 rules prose, which mentions both markers inline on a single line — a
// naive .match() grabs that zero-key span and passes vacuously.
const i = md.lastIndexOf("<!-- repo-knowledge:facts:start -->");
if (i < 0) { console.error("FAIL: template has no facts block"); process.exit(1); }
const block = md.slice(i, md.indexOf("<!-- repo-knowledge:facts:end -->", i) + 34);
const keys = block.match(/^[a-z_]+(?=:)/gm) || [];
const expected = ["languages","frameworks","platform_hints","runtime_tooling","package_managers","install_command","run_command","test_command","build_command"];
console.log("template block declares", keys.length, "keys:", keys.join(","));
if (JSON.stringify(keys) !== JSON.stringify(expected)) { console.error("FAIL: key set/order differs from the contract"); process.exit(1); }
console.log("OK — markers well-formed, all nine keys present in contract order");
'
```

Expected: `template block declares 9 keys: languages,frameworks,platform_hints,…` then `OK`.

Then prove the end-to-end behavior that actually matters — an unfilled template must never fabricate a manifest value:

```bash
cd /tmp && rm -rf t5check && mkdir t5check && cd t5check
printf '# CLAUDE.md\n\n<!-- repo-knowledge:facts:start -->\n```yaml\nlanguages: [{{LANGUAGES}}]\ninstall_command: {{INSTALL_COMMAND}}\n```\n<!-- repo-knowledge:facts:end -->\n' > CLAUDE.md
bun /path/to/ono-plugin-project-inspector/scripts/repo-knowledge.ts emit "$(pwd)" >/dev/null
node -e 'const m=require("/tmp/t5check/.ono/repo-knowledge.json");
if (JSON.stringify(m).includes("{{")) { console.error("FAIL: placeholder leaked into the manifest"); process.exit(1); }
if (m.coverage.stack !== "unknown") { console.error("FAIL: coverage should be unknown"); process.exit(1); }
console.log("OK — unfilled template yields no fabricated value, coverage honestly unknown");'
```

Expected: `OK`. Note `parseFactsBlock` itself does return the raw `{{…}}` string for a *bracketed* placeholder — the guard only catches a bare one — but `toList()` in `extractFacts` filters it before it can reach the manifest. Recorded as a known Minor: the filtering belongs in the parser too, for defense in depth, since `parseFactsBlock` is exported.

- [ ] **Step 8: Run the full suite and confirm nothing regressed**

```bash
cd ono-plugin-project-inspector && bun scripts/repo-knowledge.test.ts && bun scripts/slugify.ts
node -e 'const p=require("./.claude-plugin/plugin.json"), m=require("./.claude-plugin/marketplace.json");
if (p.version!=="0.9.0") process.exit(1);
if (m.plugins[0].version!==p.version) process.exit(1);
console.log("versions aligned at", p.version);'
```

Expected: `ALL TESTS PASSED`, four `PASS` lines, then `versions aligned at 0.9.0`.

- [ ] **Step 9: Commit**

```bash
git add skills/project-analysis/SKILL.md docs/ README.md CHANGELOG.md .claude-plugin/
git commit -m "feat(repo-knowledge): emit machine-readable facts block, publish contract, release 0.9.0"
```

**Producer complete.** The inspector emits a validated manifest during a real workflow run; nothing consumes it yet; every existing behavior is unchanged.

---

## Task 6: The deterministic reader — freshness and the degradation matrix

**Files:**
- Create: `ono-mobile-dev-plugin/scripts/read-repo-knowledge.ts`
- Create: `ono-mobile-dev-plugin/scripts/read-repo-knowledge.test.ts`

**Interfaces:**
- Consumes: `.ono/repo-knowledge.json` as specified by the contract.
- Produces: `MAX_SUPPORTED_SCHEMA_VERSION = 1`; `readRepoKnowledge(targetRoot: string): KnowledgeResult`; CLI `node scripts/read-repo-knowledge.ts <target-root>` printing a `KnowledgeResult` as JSON and **always exiting 0**. `KnowledgeResult` fields consumed by T8–T11: `available`, `reason`, `freshness`, `staleDetail`, `schemaVersion`, `producedBy`, `usableCategories`, `deriveLive`, `knowledge`, `summary`.

**Why it always exits 0:** this is the mechanism behind scope point 7. If the reader could exit non-zero, a command would have to decide whether a missing manifest is an error, and some path would eventually treat it as one. An always-successful reader whose payload says `available: false` makes "no manifest" structurally indistinguishable from any other normal state.

- [ ] **Step 1: Write the failing test**

Create `ono-mobile-dev-plugin/scripts/read-repo-knowledge.test.ts`:

```ts
/**
 * read-repo-knowledge.test.ts
 *
 * Self-contained tests for scripts/read-repo-knowledge.ts. Builds throwaway
 * git repositories with fixture .ono/repo-knowledge.json files and exercises
 * the CLI end-to-end (stdout JSON + exit code).
 *
 * The degradation matrix IS the backward-compatibility guarantee, so every row
 * of it has a test here: absent, malformed, schema-too-new, fresh, stale-head,
 * stale-artifacts, and partial/unknown coverage.
 *
 * No external test framework. Run with:
 *   node scripts/read-repo-knowledge.test.ts
 *   bun  scripts/read-repo-knowledge.test.ts
 */

import { execFileSync } from "child_process";
import { mkdtempSync, mkdirSync, writeFileSync, realpathSync, rmSync } from "fs";
import { createHash } from "crypto";
import { tmpdir } from "os";
import { join, dirname } from "path";

const HELPER = join(import.meta.dirname ?? __dirname, "read-repo-knowledge.ts");
const RUNTIME = process.execPath;

let failures = 0;
function check(name: string, cond: boolean, detail = ""): void {
  if (cond) console.log(`PASS  ${name}`);
  else { failures++; console.log(`FAIL  ${name}${detail ? `  — ${detail}` : ""}`); }
}

function run(targetRoot: string): { code: number; json: any; stderr: string } {
  try {
    const stdout = execFileSync(RUNTIME, [HELPER, targetRoot], { encoding: "utf-8", stdio: ["ignore", "pipe", "pipe"] });
    return { code: 0, json: JSON.parse(stdout), stderr: "" };
  } catch (err: any) {
    let json: any = null;
    try { json = JSON.parse(err.stdout?.toString() ?? ""); } catch { /* leave null */ }
    return { code: typeof err.status === "number" ? err.status : 1, json, stderr: err.stderr?.toString() ?? "" };
  }
}

function git(cwd: string, ...args: string[]): string {
  return execFileSync("git", args, { cwd, encoding: "utf-8", stdio: ["ignore", "pipe", "ignore"] }).trim();
}

function write(root: string, rel: string, body: string): void {
  const p = join(root, rel);
  mkdirSync(dirname(p), { recursive: true });
  writeFileSync(p, body, "utf-8");
}

function initRepo(dir: string): void {
  mkdirSync(dir, { recursive: true });
  git(dir, "init", "-q");
  git(dir, "config", "user.email", "test@example.com");
  git(dir, "config", "user.name", "Test");
  write(dir, "README.md", "# test\n");
  git(dir, "add", "-A");
  git(dir, "commit", "-qm", "init");
}

const sha = (s: string) => createHash("sha256").update(s, "utf-8").digest("hex");

const CLAUDE_BODY = "# CLAUDE.md — Demo\n";
const PATTERNS_BODY = "# Patterns\n\n## State Management\nRedux Toolkit.\n";

/** A schema-v1 manifest matching the fixture artifacts written by seedRepo(). */
function manifest(head: string | null, overrides: Record<string, any> = {}): any {
  return {
    repoKnowledgeSchemaVersion: 1,
    producedBy: { plugin: "ono-project-inspector", version: "0.9.0" },
    generatedAt: "2026-07-28T00:00:00.000Z",
    fingerprint: {
      gitHead: head,
      artifacts: {
        "CLAUDE.md": sha(CLAUDE_BODY),
        "AUDIT.md": null,
        "docs/project/overview.md": null,
        "docs/project/components.md": null,
        "docs/project/patterns.md": sha(PATTERNS_BODY),
        "docs/project/integrations.md": null,
      },
    },
    coverage: {
      stack: "populated",
      commands: "partial",
      structure: "populated",
      inventory: "unknown",
      conventions: "populated",
      integrations: "unknown",
      auditTopics: "unknown",
    },
    stack: { languages: ["TypeScript"], frameworks: ["React Native"], platformHints: ["Android", "iOS"], runtimeTooling: ["Metro"], packageManagers: ["yarn"] },
    commands: { install: "yarn", run: "yarn ios", test: null, build: null },
    structure: { repositoryTree: "CLAUDE.md#repository-structure", keyModules: "CLAUDE.md#key-modules", entryPoints: "CLAUDE.md#entry-points" },
    documents: {
      claudeMd: { path: "CLAUDE.md", exists: true, anchors: [] },
      auditMd: { path: "AUDIT.md", exists: false, anchors: [] },
      overview: { path: "docs/project/overview.md", exists: false, anchors: [] },
      inventory: { path: "docs/project/components.md", exists: false, anchors: [] },
      conventions: { path: "docs/project/patterns.md", exists: true, anchors: ["#state-management"] },
      integrations: { path: "docs/project/integrations.md", exists: false, anchors: [] },
    },
    auditTopics: [],
    ...overrides,
  };
}

/** A repo with the fixture artifacts committed, ready for a manifest. */
function seedRepo(dir: string): string {
  initRepo(dir);
  write(dir, "CLAUDE.md", CLAUDE_BODY);
  write(dir, "docs/project/patterns.md", PATTERNS_BODY);
  git(dir, "add", "-A");
  git(dir, "commit", "-qm", "artifacts");
  return git(dir, "rev-parse", "HEAD");
}

const root = mkdtempSync(join(realpathSync(tmpdir()), "rrk-"));
try {
  // --- Row 1: absent manifest -> available false, exit 0 (scope point 7) ---
  {
    const repo = join(root, "absent");
    seedRepo(repo);
    const r = run(repo);
    check("1 absent: exit 0", r.code === 0, `code=${r.code} stderr=${r.stderr}`);
    check("1 absent: available false", r.json?.available === false);
    check("1 absent: reason absent", r.json?.reason === "absent", String(r.json?.reason));
    check("1 absent: usableCategories empty", Array.isArray(r.json?.usableCategories) && r.json.usableCategories.length === 0);
    check("1 absent: deriveLive lists every category", Array.isArray(r.json?.deriveLive) && r.json.deriveLive.length === 7, JSON.stringify(r.json?.deriveLive));
    check("1 absent: knowledge is null", r.json?.knowledge === null);
    check("1 absent: summary mentions no manifest", typeof r.json?.summary === "string" && /not available|no repository knowledge/i.test(r.json.summary), r.json?.summary);
  }

  // --- Row 2: fresh manifest -> usable categories exclude unknowns ---
  {
    const repo = join(root, "fresh");
    const head = seedRepo(repo);
    write(repo, ".ono/repo-knowledge.json", JSON.stringify(manifest(head), null, 2));
    const r = run(repo);
    check("2 fresh: exit 0", r.code === 0, `code=${r.code} stderr=${r.stderr}`);
    check("2 fresh: available true", r.json?.available === true);
    check("2 fresh: freshness fresh", r.json?.freshness === "fresh", String(r.json?.freshness));
    check("2 fresh: schemaVersion 1", r.json?.schemaVersion === 1);
    check("2 fresh: conventions usable", r.json?.usableCategories?.includes("conventions"));
    check("2 fresh: partial commands usable", r.json?.usableCategories?.includes("commands"));
    check("2 fresh: unknown inventory not usable", !r.json?.usableCategories?.includes("inventory"));
    check("2 fresh: unknown inventory in deriveLive", r.json?.deriveLive?.includes("inventory"));
    check("2 fresh: knowledge carries stack", JSON.stringify(r.json?.knowledge?.stack?.languages) === JSON.stringify(["TypeScript"]));
    check("2 fresh: conventions pointer exposed", r.json?.knowledge?.documents?.conventions?.path === "docs/project/patterns.md");
  }

  // --- Row 3: stale-head — HEAD moved but artifacts unchanged ---
  {
    const repo = join(root, "stale-head");
    const head = seedRepo(repo);
    write(repo, ".ono/repo-knowledge.json", JSON.stringify(manifest(head), null, 2));
    write(repo, "src/unrelated.ts", "export const x = 1;\n");
    git(repo, "add", "-A");
    git(repo, "commit", "-qm", "unrelated change");
    const r = run(repo);
    check("3 stale-head: available true", r.json?.available === true);
    check("3 stale-head: freshness stale-head", r.json?.freshness === "stale-head", String(r.json?.freshness));
    check("3 stale-head: conventions still usable", r.json?.usableCategories?.includes("conventions"));
    check("3 stale-head: staleDetail explains", typeof r.json?.staleDetail === "string" && r.json.staleDetail.length > 0);
  }

  // --- Row 4: stale-artifacts — a source document was hand-edited ---
  {
    const repo = join(root, "stale-art");
    const head = seedRepo(repo);
    write(repo, ".ono/repo-knowledge.json", JSON.stringify(manifest(head), null, 2));
    write(repo, "docs/project/patterns.md", PATTERNS_BODY + "\n## Testing Patterns\nJest.\n");
    const r = run(repo);
    check("4 stale-art: available true", r.json?.available === true);
    check("4 stale-art: freshness stale-artifacts", r.json?.freshness === "stale-artifacts", String(r.json?.freshness));
    check("4 stale-art: affected category NOT usable", !r.json?.usableCategories?.includes("conventions"), JSON.stringify(r.json?.usableCategories));
    check("4 stale-art: affected category in deriveLive", r.json?.deriveLive?.includes("conventions"));
    check("4 stale-art: unaffected category still usable", r.json?.usableCategories?.includes("stack"), JSON.stringify(r.json?.usableCategories));
    check("4 stale-art: staleDetail names the file", typeof r.json?.staleDetail === "string" && r.json.staleDetail.includes("docs/project/patterns.md"), r.json?.staleDetail);
  }

  // --- Row 5: malformed JSON -> treated as absent ---
  {
    const repo = join(root, "malformed");
    seedRepo(repo);
    write(repo, ".ono/repo-knowledge.json", "{ this is not json");
    const r = run(repo);
    check("5 malformed: exit 0", r.code === 0, `code=${r.code}`);
    check("5 malformed: available false", r.json?.available === false);
    check("5 malformed: reason unparseable", r.json?.reason === "unparseable", String(r.json?.reason));
    check("5 malformed: deriveLive is complete", r.json?.deriveLive?.length === 7);
  }

  // --- Row 6: schema too new -> treated as absent, never mis-parsed ---
  {
    const repo = join(root, "future");
    const head = seedRepo(repo);
    write(repo, ".ono/repo-knowledge.json", JSON.stringify(manifest(head, { repoKnowledgeSchemaVersion: 99 }), null, 2));
    const r = run(repo);
    check("6 future: exit 0", r.code === 0, `code=${r.code}`);
    check("6 future: available false", r.json?.available === false);
    check("6 future: reason schema-too-new", r.json?.reason === "schema-too-new", String(r.json?.reason));
    check("6 future: schemaVersion reported", r.json?.schemaVersion === 99);
    check("6 future: summary names the versions", typeof r.json?.summary === "string" && r.json.summary.includes("99"), r.json?.summary);
  }

  // --- Row 7: structurally invalid (right version, wrong shape) -> absent ---
  {
    const repo = join(root, "invalid");
    seedRepo(repo);
    write(repo, ".ono/repo-knowledge.json", JSON.stringify({ repoKnowledgeSchemaVersion: 1 }, null, 2));
    const r = run(repo);
    check("7 invalid: available false", r.json?.available === false);
    check("7 invalid: reason invalid", r.json?.reason === "invalid", String(r.json?.reason));
  }

  // --- Row 8: no git -> freshness unknown, still usable ---
  {
    const repo = join(root, "nogit");
    mkdirSync(repo, { recursive: true });
    write(repo, "CLAUDE.md", CLAUDE_BODY);
    write(repo, "docs/project/patterns.md", PATTERNS_BODY);
    write(repo, ".ono/repo-knowledge.json", JSON.stringify(manifest(null), null, 2));
    const r = run(repo);
    check("8 nogit: available true", r.json?.available === true);
    check("8 nogit: freshness unknown", r.json?.freshness === "unknown", String(r.json?.freshness));
    check("8 nogit: conventions usable", r.json?.usableCategories?.includes("conventions"));
  }

  // --- Row 9: worktree refusal is reported, not thrown (K6) ---
  {
    const repo = join(root, "wt");
    const head = seedRepo(repo);
    write(repo, ".ono/repo-knowledge.json", JSON.stringify(manifest(head), null, 2));
    const wt = join(repo, ".claude", "worktrees", "agent-1");
    git(repo, "worktree", "add", "-q", wt);
    const r = run(wt);
    check("9 worktree: exit 0", r.code === 0, `code=${r.code} stderr=${r.stderr}`);
    check("9 worktree: available false", r.json?.available === false);
    check("9 worktree: reason worktree", r.json?.reason === "worktree", String(r.json?.reason));
  }

  // --- Row 10: nonexistent target root -> reported, exit 0 ---
  {
    const r = run(join(root, "does-not-exist"));
    check("10 missing root: exit 0", r.code === 0, `code=${r.code}`);
    check("10 missing root: available false", r.json?.available === false);
    check("10 missing root: reason root-not-found", r.json?.reason === "root-not-found", String(r.json?.reason));
  }

  // --- Row 11: advisory-only platformHints is never surfaced as a decision (K5) ---
  {
    const repo = join(root, "hints");
    const head = seedRepo(repo);
    write(repo, ".ono/repo-knowledge.json", JSON.stringify(manifest(head), null, 2));
    const r = run(repo);
    check("11 hints: no top-level platform field", r.json?.knowledge?.platform === undefined);
    check("11 hints: platformHints marked advisory", r.json?.platformHintsAreAdvisory === true);
    check("11 hints: 'platform' is not a category", !r.json?.usableCategories?.includes("platform"));
  }
} finally {
  rmSync(root, { recursive: true, force: true });
}

console.log(failures === 0 ? "\nALL TESTS PASSED" : `\n${failures} TEST(S) FAILED`);
process.exit(failures > 0 ? 1 : 0);
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd ono-mobile-dev-plugin && node scripts/read-repo-knowledge.test.ts
```

Expected: every row FAILs — `read-repo-knowledge.ts` does not exist yet, so `run()` returns a non-zero code with `json: null`.

- [ ] **Step 3: Write the implementation**

Create `ono-mobile-dev-plugin/scripts/read-repo-knowledge.ts`:

```ts
/**
 * read-repo-knowledge.ts
 *
 * Deterministic reader for the repository-knowledge manifest at
 * `<target-root>/.ono/repo-knowledge.json`, produced by the sibling
 * ono-project-inspector plugin. See docs/repo-knowledge-contract.md for the
 * schema and the obligations this plugin accepts as a consumer.
 *
 * This is the ONLY component in this plugin that parses the manifest. Commands,
 * skills, and agents receive its normalized output via the
 * `repo-knowledge-consumer` skill and never read the file themselves — so the
 * contract lives in exactly one place and structured facts are never parsed by
 * an LLM.
 *
 * CRITICAL: this helper ALWAYS exits 0 and ALWAYS prints a valid JSON object,
 * including when the manifest is absent, malformed, or written by a newer
 * schema. A missing manifest is a normal state, not an error — that is what
 * keeps every command byte-for-byte backward compatible on a repository that
 * was never inspected. Callers branch on `available`, never on the exit code.
 *
 * Runtime: Node >= 23.6 (`node scripts/read-repo-knowledge.ts`) or Bun. No
 * external deps, and it does NOT rely on CLAUDE_PLUGIN_ROOT or
 * CLAUDE_PROJECT_DIR.
 *
 * Usage:
 *   node scripts/read-repo-knowledge.ts [target-root]   (default: CWD)
 */

import { readFileSync, existsSync, realpathSync } from "fs";
import { createHash } from "crypto";
import { execFileSync } from "child_process";
import { join, resolve, sep } from "path";

/** Highest contract version this plugin understands. A higher one is treated as absent. */
export const MAX_SUPPORTED_SCHEMA_VERSION = 1;

const CLAUDE_WORKTREE_MARKER = `${sep}.claude${sep}worktrees${sep}`;

/** Every category in contract v1, in a stable order. */
const ALL_CATEGORIES = [
  "stack",
  "commands",
  "structure",
  "inventory",
  "conventions",
  "integrations",
  "auditTopics",
] as const;

/** Which source document backs each category, for stale-artifact attribution. */
const CATEGORY_SOURCE: Record<string, string> = {
  stack: "CLAUDE.md",
  commands: "CLAUDE.md",
  structure: "CLAUDE.md",
  inventory: "docs/project/components.md",
  conventions: "docs/project/patterns.md",
  integrations: "docs/project/integrations.md",
  auditTopics: "AUDIT.md",
};

type Unavailable = "absent" | "unparseable" | "invalid" | "schema-too-new" | "worktree" | "root-not-found";
type Freshness = "fresh" | "stale-head" | "stale-artifacts" | "unknown";

export interface KnowledgeResult {
  available: boolean;
  reason: Unavailable | null;
  schemaVersion: number | null;
  producedBy: { plugin: string; version: string } | null;
  generatedAt: string | null;
  freshness: Freshness | null;
  staleDetail: string | null;
  /** Categories the consumer may trust as-is. */
  usableCategories: string[];
  /** Categories the consumer MUST derive itself. */
  deriveLive: string[];
  /** The manifest, verbatim, when available. Never partially rewritten. */
  knowledge: Record<string, any> | null;
  /** Always true — a reminder that platformHints is never authoritative (contract obligation 8). */
  platformHintsAreAdvisory: true;
  /** One line for a command to show the developer. */
  summary: string;
}

function unavailable(reason: Unavailable, summary: string, schemaVersion: number | null = null): KnowledgeResult {
  return {
    available: false,
    reason,
    schemaVersion,
    producedBy: null,
    generatedAt: null,
    freshness: null,
    staleDetail: null,
    usableCategories: [],
    deriveLive: [...ALL_CATEGORIES],
    knowledge: null,
    platformHintsAreAdvisory: true,
    summary,
  };
}

function currentHead(targetRoot: string): string | null {
  try {
    return execFileSync("git", ["rev-parse", "HEAD"], {
      cwd: targetRoot,
      encoding: "utf-8",
      stdio: ["ignore", "pipe", "ignore"],
    }).trim() || null;
  } catch {
    return null;
  }
}

function sha256OfFile(targetRoot: string, rel: string): string | null {
  const p = join(targetRoot, rel);
  if (!existsSync(p)) return null;
  return createHash("sha256").update(readFileSync(p, "utf-8"), "utf-8").digest("hex");
}

/** Structural validation. Mirrors the producer's validateManifest — shape only. */
function structuralErrors(m: any): string[] {
  const errors: string[] = [];
  if (!m || typeof m !== "object") return ["manifest is not an object"];
  if (!m.producedBy?.plugin) errors.push("producedBy.plugin missing");
  if (typeof m.generatedAt !== "string") errors.push("generatedAt missing");
  if (!m.fingerprint || typeof m.fingerprint.artifacts !== "object") errors.push("fingerprint.artifacts missing");
  if (!m.coverage || typeof m.coverage !== "object") errors.push("coverage missing");
  if (!m.documents || typeof m.documents !== "object") errors.push("documents missing");
  if (!Array.isArray(m.auditTopics)) errors.push("auditTopics is not an array");
  return errors;
}

export function readRepoKnowledge(targetRootInput: string): KnowledgeResult {
  const targetRootAbs = resolve(targetRootInput);
  if (!existsSync(targetRootAbs)) {
    return unavailable("root-not-found", `Target root not found: ${targetRootAbs}. Deriving all repository knowledge live.`);
  }
  const targetRoot = realpathSync(targetRootAbs);

  // A manifest read from an ephemeral agent worktree would mislead every
  // consumer, so refuse it the same way the producer does.
  if (targetRoot.includes(CLAUDE_WORKTREE_MARKER)) {
    return unavailable(
      "worktree",
      "Target root is inside .claude/worktrees — refusing to read repository knowledge from an agent worktree. Deriving all repository knowledge live."
    );
  }

  const manifestRel = join(".ono", "repo-knowledge.json");
  const manifestPath = join(targetRoot, manifestRel);
  if (!existsSync(manifestPath)) {
    return unavailable(
      "absent",
      "Repository knowledge is not available (no .ono/repo-knowledge.json). Deriving all repository knowledge live — running /inspect with the Ono Project Inspector would let this plugin reuse approved knowledge instead."
    );
  }

  let parsed: any;
  try {
    parsed = JSON.parse(readFileSync(manifestPath, "utf-8"));
  } catch (err) {
    return unavailable("unparseable", `.ono/repo-knowledge.json is not valid JSON (${(err as Error).message}). Deriving all repository knowledge live.`);
  }

  const schemaVersion = typeof parsed?.repoKnowledgeSchemaVersion === "number" ? parsed.repoKnowledgeSchemaVersion : null;
  if (schemaVersion === null) {
    return unavailable("invalid", ".ono/repo-knowledge.json has no repoKnowledgeSchemaVersion. Deriving all repository knowledge live.");
  }
  if (schemaVersion > MAX_SUPPORTED_SCHEMA_VERSION) {
    return unavailable(
      "schema-too-new",
      `.ono/repo-knowledge.json uses contract schema v${schemaVersion}; this plugin supports up to v${MAX_SUPPORTED_SCHEMA_VERSION}. Deriving all repository knowledge live — upgrade ono-mobile-dev-plugin to consume it.`,
      schemaVersion
    );
  }

  const errors = structuralErrors(parsed);
  if (errors.length) {
    return unavailable("invalid", `.ono/repo-knowledge.json failed structural validation (${errors.join("; ")}). Deriving all repository knowledge live.`, schemaVersion);
  }

  // --- Freshness ---
  const recordedHead: string | null = parsed.fingerprint.gitHead ?? null;
  const head = currentHead(targetRoot);

  const changedArtifacts: string[] = [];
  for (const [rel, recordedHash] of Object.entries(parsed.fingerprint.artifacts as Record<string, string | null>)) {
    if (sha256OfFile(targetRoot, rel) !== recordedHash) changedArtifacts.push(rel);
  }
  changedArtifacts.sort();

  let freshness: Freshness;
  let staleDetail: string | null = null;
  if (changedArtifacts.length > 0) {
    freshness = "stale-artifacts";
    staleDetail = `Changed since the manifest was written: ${changedArtifacts.join(", ")}. Categories backed by these documents are derived live. Run /inspect-sync (or /inspect) to refresh.`;
  } else if (recordedHead === null || head === null) {
    freshness = "unknown";
    staleDetail = "Freshness could not be established (no git HEAD recorded, or git unavailable). Using the manifest as-is.";
  } else if (recordedHead !== head) {
    freshness = "stale-head";
    staleDetail = `HEAD moved since the manifest was written (recorded ${recordedHead.slice(0, 8)}, current ${head.slice(0, 8)}), but every indexed document is unchanged. Using the manifest as-is.`;
  } else {
    freshness = "fresh";
  }

  // --- Usable vs derive-live, per contract obligations 5 and 6 ---
  const usableCategories: string[] = [];
  const deriveLive: string[] = [];
  for (const category of ALL_CATEGORIES) {
    const coverage = parsed.coverage?.[category];
    const source = CATEGORY_SOURCE[category];
    const sourceChanged = changedArtifacts.includes(source);
    if (coverage === "populated" || coverage === "partial") {
      if (sourceChanged) deriveLive.push(category);
      else usableCategories.push(category);
    } else {
      deriveLive.push(category);
    }
  }

  const summary =
    `Repository knowledge available (contract v${schemaVersion}, produced by ${parsed.producedBy.plugin} ${parsed.producedBy.version}, ${freshness}). ` +
    `Reusing: ${usableCategories.length ? usableCategories.join(", ") : "nothing"}. ` +
    `Deriving live: ${deriveLive.length ? deriveLive.join(", ") : "nothing"}.`;

  return {
    available: true,
    reason: null,
    schemaVersion,
    producedBy: parsed.producedBy,
    generatedAt: parsed.generatedAt,
    freshness,
    staleDetail,
    usableCategories,
    deriveLive,
    knowledge: parsed,
    platformHintsAreAdvisory: true,
    summary,
  };
}

function main(): void {
  const target = process.argv[2] ?? process.cwd();
  // Always exit 0 with valid JSON — see the header note.
  console.log(JSON.stringify(readRepoKnowledge(target), null, 2));
  process.exit(0);
}

// Only run the CLI when executed directly, so the test file can import the
// pure function without triggering process.exit.
const invokedDirectly =
  typeof process !== "undefined" &&
  process.argv[1] !== undefined &&
  /read-repo-knowledge\.ts$/.test(realpathSync(process.argv[1]));

if (invokedDirectly) {
  main();
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
cd ono-mobile-dev-plugin && node scripts/read-repo-knowledge.test.ts
```

Expected: `ALL TESTS PASSED`, all eleven rows.

- [ ] **Step 5: Confirm the existing resolver test still passes**

```bash
cd ono-mobile-dev-plugin && node scripts/resolve-target-repo-root.test.ts
```

Expected: `ALL TESTS PASSED` — unchanged. This asserts T6 touched nothing the resolver depends on.

- [ ] **Step 6: Verify the exit code is 0 in every failure mode**

```bash
cd ono-mobile-dev-plugin
for t in /tmp /nonexistent-path-xyz .; do node scripts/read-repo-knowledge.ts "$t" > /dev/null; echo "$t -> exit $?"; done
```

Expected: `exit 0` for all three. This is scope point 7's mechanism verified directly.

- [ ] **Step 7: Commit**

```bash
git add scripts/read-repo-knowledge.ts scripts/read-repo-knowledge.test.ts
git commit -m "feat(repo-knowledge): deterministic reader with freshness and degradation matrix"
```

---

## Task 7: Consumer-side contract document and schema-parity assertion

**Files:**
- Create: `ono-mobile-dev-plugin/docs/repo-knowledge-contract.md`

**Interfaces:**
- Consumes: `ono-plugin-project-inspector/docs/repo-knowledge-contract.md` from T5 — copied verbatim below the title line.
- Produces: the pinned reference every later task cites; the parity check that prevents K3 (schema drift between independently released plugins).

**Why a duplicated file rather than a shared one:** there is no cross-plugin dependency mechanism in Claude Code. The alternative — each plugin describing the contract in its own words — is precisely how the two would drift. Vendoring the specification matches the precedent already set by `resolve-target-repo-root.ts`.

- [ ] **Step 1: Copy the contract document and adjust the producer-relative references**

```bash
cd /Users/yagilcohen/Developer/AI
mkdir -p ono-mobile-dev-plugin/docs
cp ono-plugin-project-inspector/docs/repo-knowledge-contract.md \
   ono-mobile-dev-plugin/docs/repo-knowledge-contract.md
```

Then make exactly two edits to the copy, so the body below the title stays byte-identical apart from these:

Replace:

```markdown
Producer: `ono-project-inspector` (this plugin), via `skills/repo-knowledge` and `scripts/repo-knowledge.ts`.
Consumers: `ono-mobile-dev-plugin` (and future Ono plugins), via their own deterministic reader.
```

with:

```markdown
Producer: `ono-project-inspector`, via its `skills/repo-knowledge` and `scripts/repo-knowledge.ts`.
Consumers: `ono-mobile-dev-plugin` (this plugin), via `scripts/read-repo-knowledge.ts` and the `repo-knowledge-consumer` skill.
```

And replace the schema-section pointer:

```markdown
See the `RepoKnowledge` interface in `scripts/repo-knowledge.ts` for the authoritative type. Field reference:
```

with:

```markdown
The authoritative type is the `RepoKnowledge` interface in the producer's `scripts/repo-knowledge.ts`; this plugin's reader mirrors it in `scripts/read-repo-knowledge.ts` and validates the fields it depends on. Field reference:
```

- [ ] **Step 2: Assert the two copies agree on the schema version**

```bash
cd /Users/yagilcohen/Developer/AI
A=$(grep -m1 '^\*\*Schema version:' ono-plugin-project-inspector/docs/repo-knowledge-contract.md)
B=$(grep -m1 '^\*\*Schema version:' ono-mobile-dev-plugin/docs/repo-knowledge-contract.md)
C=$(grep -m1 'MAX_SUPPORTED_SCHEMA_VERSION = ' ono-mobile-dev-plugin/scripts/read-repo-knowledge.ts)
D=$(grep -m1 'REPO_KNOWLEDGE_SCHEMA_VERSION = ' ono-plugin-project-inspector/scripts/repo-knowledge.ts)
echo "producer doc:  $A"
echo "consumer doc:  $B"
echo "consumer code: $C"
echo "producer code: $D"
test "$A" = "$B" && echo "PASS docs agree" || { echo "FAIL docs disagree"; exit 1; }
# NOTE: the doc line is `**Schema version: 1**` — Markdown bold, so it ends in
# `1**`, not `1`. A naive `grep -q ': 1$'` is a FALSE NEGATIVE and fails on a
# perfectly correct file. Match the trailing bold markers explicitly.
echo "$A" | grep -qE ':[[:space:]]*1\*\*$' && echo "PASS docs pin v1" || { echo "FAIL docs do not pin v1"; exit 1; }
echo "$C" | grep -q '= 1;' && echo "PASS consumer code pins v1" || { echo "FAIL"; exit 1; }
echo "$D" | grep -q '= 1;' && echo "PASS producer code pins v1" || { echo "FAIL"; exit 1; }
```

Expected: four `PASS` lines.

- [ ] **Step 3: Assert the contract bodies have not diverged beyond the two intended edits**

```bash
cd /Users/yagilcohen/Developer/AI
diff <(tail -n +2 ono-plugin-project-inspector/docs/repo-knowledge-contract.md) \
     <(tail -n +2 ono-mobile-dev-plugin/docs/repo-knowledge-contract.md) | tee /tmp/contract-diff.txt
echo "--- changed lines: $(grep -c '^[<>]' /tmp/contract-diff.txt)"
```

Expected: exactly 6 changed lines — the 2 producer/consumer lines replaced by 2, and the 1 schema-pointer line replaced by 1. Anything more means an unintended divergence; reconcile before continuing.

- [ ] **Step 4: Commit**

```bash
cd ono-mobile-dev-plugin
git add docs/repo-knowledge-contract.md
git commit -m "docs(repo-knowledge): vendor the repository knowledge contract, pinned to schema v1"
```

---

## Task 8: The `repo-knowledge-consumer` skill

**Files:**
- Create: `ono-mobile-dev-plugin/skills/repo-knowledge-consumer/SKILL.md`

**Interfaces:**
- Consumes: `scripts/read-repo-knowledge.ts` from T6; `scripts/resolve-target-repo-root.ts` (existing) for `TARGET_ROOT`.
- Produces: the **Repository Knowledge Resolution** procedure and the **Repo Knowledge Reference block** shape, both cited verbatim by T9, T10, and T11. This is the single place either command learns what it may trust.

**Note on plugin.json:** unlike the inspector, `ono-mobile-dev-plugin/.claude-plugin/plugin.json` does **not** enumerate `skills[]`, `agents[]`, or `commands[]` — they are directory-discovered. So a new skill folder needs no manifest edit. Only the version bump in T12 touches that file.

- [ ] **Step 1: Create the skill**

Create `ono-mobile-dev-plugin/skills/repo-knowledge-consumer/SKILL.md`:

```markdown
---
name: repo-knowledge-consumer
description: Resolves the canonical repository knowledge published by the ono-project-inspector plugin at .ono/repo-knowledge.json, decides which knowledge categories may be reused and which must still be derived live, and defines the Repo Knowledge Reference block downstream documents record instead of embedding repository facts verbatim. Used by /analyze-feature and /dev-design-start. This is the only component in this plugin that understands the manifest format. It never writes any inspector-owned artifact and never blocks a command when the manifest is absent.
---

## Purpose

Before this plugin re-derives anything about the repository, check whether the repository already has approved knowledge and reuse it. The Ono Project Inspector derives repository knowledge once, gates it behind human review, and publishes a deterministic index at `.ono/repo-knowledge.json`. Without this step, every command re-scans the repository, reaches its own conclusions, and then freezes them into per-feature documents that go stale silently.

**This skill never blocks.** A repository with no manifest is the normal case, not an error — behavior falls back to full live derivation, exactly as before this skill existed.

See `docs/repo-knowledge-contract.md` for the schema and the obligations this plugin accepts.

## Step 1 — Resolve `TARGET_ROOT`

If the calling command has already resolved `TARGET_ROOT`, use it unchanged. Otherwise run the existing helper — do not reimplement worktree logic:

```
node "${CLAUDE_PLUGIN_ROOT}/scripts/resolve-target-repo-root.ts" "<candidate>"
```

Use `targetRoot` from its JSON on exit 0. On exit 1/3, `ok: false`, or `targetIsWorktree: true`, stop and report — that failure is about the repository root, not about repository knowledge.

## Step 2 — Read the manifest through the helper

```
node --no-warnings "${CLAUDE_PLUGIN_ROOT}/scripts/read-repo-knowledge.ts" "<TARGET_ROOT>"
```

The `--no-warnings` flag is part of the canonical invocation, not optional. Node emits an `ExperimentalWarning` on stderr when executing a `.ts` file directly; any caller that merges stderr into stdout would be handed non-JSON and fail to parse it, defeating the always-valid-JSON contract below.

**Never read or parse `.ono/repo-knowledge.json` yourself.** The helper is the only parser, so freshness and coverage are computed identically everywhere. It always exits 0 and always prints a JSON object — treat a non-zero exit as a broken installation, not as "no knowledge."

Read these fields:

| Field | Use |
|---|---|
| `available` | `false` → skip to Step 5 (full live derivation). |
| `reason` | Why it is unavailable: `absent`, `unparseable`, `invalid`, `schema-too-new`, `worktree`, `root-not-found`. |
| `freshness` | `fresh`, `stale-head`, `stale-artifacts`, or `unknown`. |
| `staleDetail` | Human-readable staleness explanation, when present. |
| `usableCategories` | Categories you may reuse. |
| `deriveLive` | Categories you **must** derive yourself. |
| `knowledge` | The manifest: `stack`, `commands`, `structure`, `documents`, `auditTopics`. |
| `knowledge.fingerprint.gitHead` | The recorded git HEAD — the value the citation block's `repo_knowledge_fingerprint` records. Note it is nested inside `knowledge`, not a top-level result field. |
| `summary` | The one line to show the developer. |

## Step 3 — Reuse the usable categories

For every category in `usableCategories`, use the manifest instead of deriving it:

| Category | What you get | Read it from |
|---|---|---|
| `stack` | languages, frameworks, runtime tooling, package managers | `knowledge.stack` |
| `commands` | install / run / test / build | `knowledge.commands` |
| `structure` | repository tree, key modules, entry points | the `CLAUDE.md#…` pointers in `knowledge.structure` |
| `inventory` | existing screens, components, hooks, navigation map, known duplicates | open `knowledge.documents.inventory.path` |
| `conventions` | state management, API, navigation, styling, errors, i18n/RTL, naming, testing | open `knowledge.documents.conventions.path` |
| `integrations` | services, SDKs, auth, analytics, push, payments, env-var names | open `knowledge.documents.integrations.path` |
| `auditTopics` | which audit topics exist and their approval status | `knowledge.auditTopics` |

For a pointer category, **open the document and read the relevant section** — the manifest deliberately carries no prose. Use `anchors` to cite the exact section. Do not re-derive a fact the document already states.

**Reuse means read, not copy.** Record a citation (path plus anchor), never a verbatim paste, per Step 6.

## Step 4 — Derive only what is in `deriveLive`

A category appears in `deriveLive` when its coverage is `unknown` (the producer could not determine it) or its backing document changed since the manifest was written. Derive exactly those, and no others. Do not re-derive a category you just reused because it "seems cheap to double-check" — that is the duplication this skill exists to remove.

## Step 5 — Full live derivation when knowledge is unavailable

When `available` is `false`, behave **exactly as this plugin did before the manifest existed**: derive everything live via `repo-analyst`'s full procedure. This is a supported, first-class path — most repositories will be here.

Tell the developer once, in one line, and then proceed without further comment:

```
Repository knowledge: not available (<reason>). Deriving repository context live.
Running /inspect with the Ono Project Inspector would let this plugin reuse approved
repository knowledge instead of re-deriving it each time.
```

Never stop, never ask permission to continue, and never treat this as a warning that needs resolving.

## Step 6 — The Repo Knowledge Reference block

Any document this plugin generates that used repository knowledge records **what it used**, not a copy of it, so a later reader can resolve the same sources and tell whether they have moved since.

Frontmatter fields:

```yaml
repo_knowledge_status:      # available | unavailable
repo_knowledge_schema:      # the contract schema version, or null
repo_knowledge_fingerprint: # fingerprint.gitHead from the manifest, or null
repo_knowledge_freshness:   # fresh | stale-head | stale-artifacts | unknown | null
repo_knowledge_reused:      # comma-separated usableCategories actually used, or none
repo_knowledge_derived:     # comma-separated categories derived live, or none
```

Write the bare YAML keyword `null` (not the string `"null"`, and not an empty value) for any field that has no value. `repo_knowledge_reused` and `repo_knowledge_derived` **mirror the reader's `usableCategories` and `deriveLive` verbatim** — do not summarize, reorder, or abbreviate them; write `none` only when the array is genuinely empty. Three different commands populate this block, so an unspecified encoding here becomes divergence across three documents.

When knowledge is unavailable the reader returns `usableCategories: []` and a `deriveLive` containing all seven categories, so the block records exactly that:

```yaml
repo_knowledge_status: unavailable
repo_knowledge_schema: null
repo_knowledge_fingerprint: null
repo_knowledge_freshness: null
repo_knowledge_reused: none
repo_knowledge_derived: stack, commands, structure, inventory, conventions, integrations, auditTopics
```

Body section:

```markdown
## Repo Knowledge Reference

Source: `.ono/repo-knowledge.json` (contract v<schema>, produced by <plugin> <version>, <freshness> @ <gitHead short>)

| Category | Reused from | Section |
|---|---|---|
| conventions | `docs/project/patterns.md` | `#state-management`, `#navigation-patterns` |
| inventory | `docs/project/components.md` | `#screens`, `#reusable-ui-components` |

Derived live for this feature: <categories, or "none">
```

When knowledge is unavailable, the section reads:

```markdown
## Repo Knowledge Reference

Repository knowledge was not available (<reason>). All repository context in this document was derived live at authoring time and is a point-in-time observation.
```

That last sentence matters: it marks the content as a snapshot rather than a citation, which is the honest label for it.

## Step 7 — Report staleness, never repair it

When `freshness` is `stale-head` or `stale-artifacts`, state it in one line and recommend `/inspect-sync` (or `/inspect`). Then continue — the helper has already moved affected categories into `deriveLive`, so the work is already correct.

**Never** write, regenerate, or "fix" `.ono/repo-knowledge.json`, `CLAUDE.md`, `AUDIT.md`, `docs/project/**`, `audits/**`, or `.ono/state.json`. Those belong to the inspector. Repairing them from here would create two writers for the same file.

## Hard Constraints

- Never parse `.ono/repo-knowledge.json` directly — always go through `scripts/read-repo-knowledge.ts`.
- Never write any inspector-owned artifact.
- Never block, warn repeatedly, or ask the developer to run an inspection before continuing.
- Never re-derive a category listed in `usableCategories`.
- Never skip deriving a category listed in `deriveLive`.
- Never copy repository knowledge verbatim into a generated document — cite it per Step 6.
- **Never treat `knowledge.stack.platformHints` as the platform.** It is advisory corroboration only. The authoritative platform is whatever `repo-analyst` detects and the human confirms in `/analyze-feature`, every time, regardless of what the manifest says.
- Never use the manifest to resolve `device_type` — it carries no device information.
```

- [ ] **Step 2: Verify the skill's frontmatter parses and the name matches its folder**

```bash
cd ono-mobile-dev-plugin
head -4 skills/repo-knowledge-consumer/SKILL.md
grep -c "^name: repo-knowledge-consumer$" skills/repo-knowledge-consumer/SKILL.md
grep -c "platformHints" skills/repo-knowledge-consumer/SKILL.md
```

Expected: frontmatter opens with `---` then `name: repo-knowledge-consumer`; the second command prints `1`; the third prints a non-zero count (the K5 guard is present).

- [ ] **Step 3: Commit**

```bash
git add skills/repo-knowledge-consumer/SKILL.md
git commit -m "feat(repo-knowledge): add repo-knowledge-consumer skill as the single contract reader"
```

---

## Task 9: Narrow `repo-analyst` to feature-specific detection

**Files:**
- Modify: `ono-mobile-dev-plugin/agents/repo-analyst.md`
- Modify: `ono-mobile-dev-plugin/skills/mobile-repo-analysis/SKILL.md`

**Interfaces:**
- Consumes: the `repo-knowledge-consumer` output contract from T8.
- Produces: a `repo-analyst` whose findings summary now has a **Repository Knowledge** section first, and whose stack-detection section may legitimately read "reused from canonical repository knowledge." T10's template edit depends on that shape.

**This is scope point 4, and the highest-risk task in the plan.** What changes and what must not:

| `repo-analyst` element | Change | Why |
|---|---|---|
| Step 0 workspace scoping | **unchanged** | Feature/diff-scoped. |
| Steps 1–5 platform detection + verdicts + RN-shell linkage + routing | **unchanged, still always runs** | Platform is a feature decision confirmed by a human (D1, K5). |
| Step 6 neutral stack inventory (nav / state / data-fetching / test-runner / monorepo / lint-format) | **conditional** on `conventions` being in `deriveLive` | This is what duplicates `docs/project/patterns.md`. |
| Step 6 folder-structure comparison against `ARCH-LAYERS-*` / `ARCH-FOLDERS-*` | **unchanged, still always runs** | **K4.** This is judgment against org standards, not neutral inventory. `patterns.md` describes conventions; it does not evaluate them against Ono's standards. Losing this would silently drop standards enforcement. |
| Step 6.5 device-type detection | **unchanged** | Feature-scoped (D2); the manifest carries no device data. |
| File attribution rule | **unchanged** | Diff-scoped and ephemeral. |
| Output format | **extended** with a Repository Knowledge section | Downstream needs to know what was reused vs derived. |

- [ ] **Step 1: Add the knowledge-resolution step to the agent's process**

In `ono-mobile-dev-plugin/agents/repo-analyst.md`, insert a new step immediately before `### Step 0 — Workspace scoping for monorepos`:

```markdown
### Step −1 — Resolve canonical repository knowledge first

Before any detection, apply the `repo-knowledge-consumer` skill to resolve the repository knowledge the Ono Project Inspector may already have published at `.ono/repo-knowledge.json`. Record its `available`, `freshness`, `usableCategories`, and `deriveLive` values — every later step branches on them.

This exists so this agent stops re-deriving repository facts that already exist in an approved form. It changes what you *inventory*, never what you *decide*:

- **Platform detection (Steps 1–5) always runs in full**, regardless of what the manifest says. `knowledge.stack.platformHints` is advisory corroboration only — mention it as supporting evidence if it agrees with your finding, and if it disagrees, report the disagreement and trust your own evidence. There is no manifest field that can substitute for platform detection or for the human confirmation that follows it.
- **Device-type resolution (Step 6.5) always runs in full.** The manifest carries no device information.
- **Only the neutral stack inventory in Step 6 becomes conditional.**

If knowledge is unavailable, proceed exactly as this agent always has — full detection for every step.
```

- [ ] **Step 2: Make the neutral inventory conditional, and protect the standards comparison**

In the same file, replace the React Native bullet of `### Step 6 — Platform-specific stack detection (only once platform is confirmed)`:

```markdown
- **React Native**: read `package.json` dependencies to identify the navigation library, state-management library, data-fetching layer, and test runner actually in use — don't assume any of these; detect them. Check for monorepo tooling and note package boundaries. Scan the folder structure and compare it against `standards/react-native/rn-architecture.md`'s expected layering (`ARCH-LAYERS-*`, `ARCH-FOLDERS-*`). Note lint/format tooling and any custom rule configuration.
```

with:

```markdown
- **React Native**:
  - **Neutral stack inventory — reuse when available.** The navigation library, state-management library, data-fetching layer, test runner, monorepo tooling/package boundaries, and lint/format tooling are exactly what `docs/project/patterns.md` already records. If `conventions` is in `usableCategories`, **read that document instead of re-detecting**, cite the sections you used, and report this section as reused. Only when `conventions` is in `deriveLive` (coverage `unknown`, or the document changed since the manifest was written) do you detect these from `package.json` and the folder tree yourself — and then detect them, never assume them.
  - **Standards conformance — always runs, never reused.** Independently of the manifest, scan the folder structure and compare it against `standards/react-native/rn-architecture.md`'s expected layering (`ARCH-LAYERS-*`, `ARCH-FOLDERS-*`). This is a judgment about whether the repository conforms to Ono's standards, which is this plugin's responsibility and is **not** something `docs/project/patterns.md` contains — that document describes what the conventions *are*, not whether they comply. Never skip this step because `conventions` was reused.
```

- [ ] **Step 3: Extend the output format**

In the same file, replace the `## Output format` section body with:

```markdown
A structured findings summary (not free-form prose) with these sections, in this order. Every section is present even if the answer is "not detected" or "reused" — downstream agents rely on the summary's shape being consistent.

1. **Repository Knowledge** — whether canonical knowledge was available and, if so: contract version, producing plugin and version, freshness, the git fingerprint, which categories were reused, and which were derived live. When unavailable, state the reason in one line.
2. **Platform Detection** — raw signals found, candidate platform(s), confidence (High/Medium/Low), and — if Low — the exact question put to the human. Note whether `platformHints` corroborated or contradicted the finding. **Always derived fresh.**
3. **Device Type** — the resolved `mobile` or `tv` value and its confidence, or the exact mobile-vs-TV question put to the human. **Always derived fresh.**
4. **Stack Detection** — for React Native: Navigation, State Management, Data Fetching, Testing, Monorepo/Workspace, Lint/Format. Each entry is tagged either `[reused: docs/project/patterns.md#anchor]` or `[derived live]` so a reader can tell a citation from an observation. For iOS/Android/React: the lighter existence-check findings.
5. **Standards Conformance** — the folder-structure comparison against the platform's `ARCH-*` expectations. **Always derived fresh**, never reused.
```

- [ ] **Step 4: Update the agent's inputs and constraints**

In the same file, add to `## Inputs`:

```markdown
- Canonical repository knowledge from `.ono/repo-knowledge.json`, resolved via the `repo-knowledge-consumer` skill (may be unavailable — that is a normal state).
```

And append to `## Constraints`:

```markdown
- Resolve canonical repository knowledge before detecting anything, and do not re-derive a category the `repo-knowledge-consumer` skill reported as reusable. Re-deriving it is the duplication this step exists to remove.
- Never let canonical knowledge substitute for platform detection or device-type resolution. `platformHints` is advisory; the platform is decided by your own evidence plus the human's confirmation in `/analyze-feature`, every time.
- Never skip the standards-conformance comparison because the neutral inventory was reused — they answer different questions.
- Report every stack finding as either `[reused: <path>#<anchor>]` or `[derived live]`. An unlabelled finding is indistinguishable from a guess.
- Never write to `.ono/repo-knowledge.json`, `CLAUDE.md`, `AUDIT.md`, `docs/project/**`, `audits/**`, or `.ono/state.json`.
```

- [ ] **Step 5: Rewrite the skill's methodology as consume-then-delta**

In `ono-mobile-dev-plugin/skills/mobile-repo-analysis/SKILL.md`, replace step 1 and step 3 of `## Methodology`.

Replace step 1:

```markdown
1. **Detect platform first, before anything else.** Run `repo-analyst`'s platform-detection algorithm (raw signal checks → RN/iOS/Android/React verdicts → RN-native-shell linkage check → final routing). See `agents/repo-analyst.md` for the full decision procedure. If monorepo tooling is present, run detection per touched workspace/package, not once for the whole repo.
```

with:

```markdown
1. **Resolve canonical repository knowledge, then detect the platform.** Apply the `repo-knowledge-consumer` skill first, so what the repository already knows about itself is in hand before anything is re-derived. Then run `repo-analyst`'s platform-detection algorithm in full (raw signal checks → RN/iOS/Android/React verdicts → RN-native-shell linkage check → final routing) — see `agents/repo-analyst.md`. Platform detection always runs regardless of what the manifest contains; `stack.platformHints` is advisory corroboration, never a substitute. If monorepo tooling is present, run detection per touched workspace/package, not once for the whole repo. When no canonical knowledge is available, this step behaves exactly as it always has.
```

Replace step 3:

```markdown
3. **Once platform is confirmed, run platform-specific stack detection.** For React Native: enumerate `package.json` dependencies to identify the navigation library, state-management library, data-fetching layer, and test runner in use; detect monorepo/workspace tooling; scan the folder structure against `standards/react-native/rn-architecture.md`'s expected layering (`ARCH-LAYERS-*`, `ARCH-FOLDERS-*`); note lint/format tooling. For iOS/Android/React (web): lightweight existence checks only for now (see `agents/repo-analyst.md` — deep convention detection is deferred until each platform's standards are authored). **Also resolve the device type for this workflow** — exactly one of `mobile` or `tv` (the mobile-vs-TV context signal, not a new platform value, with no `mixed` value) — using `repo-analyst`'s device-type markers (tvOS device family / `TVUIKit`, Android Leanback, `react-native-tvos`, Tizen/webOS packaging). If a repo has both targets, resolve the one this feature is about, asking the human when unclear.
```

with:

```markdown
3. **Once platform is confirmed, fill only the gaps in the stack picture.** Split this into what can be reused and what must always be derived:

   - **Reuse the neutral stack inventory when available.** If `conventions` is in `usableCategories`, read `docs/project/patterns.md` for the navigation library, state-management library, data-fetching layer, test runner, monorepo/workspace tooling, and lint/format tooling, and cite the sections used. Do not re-enumerate `package.json` for facts that document already states. Only when `conventions` is in `deriveLive` do you detect these yourself — and then detect, never assume.
   - **Reuse the component inventory when available.** If `inventory` is in `usableCategories`, read `docs/project/components.md` so the architect can reuse existing screens, components, and hooks instead of proposing duplicates. This is the single highest-value reuse in the pipeline: that document exists specifically to be read before speccing a feature.
   - **Always run the standards-conformance comparison.** Scan the folder structure against `standards/react-native/rn-architecture.md`'s expected layering (`ARCH-LAYERS-*`, `ARCH-FOLDERS-*`) regardless of what was reused. `patterns.md` states what the conventions are; it does not judge them against Ono's standards, and that judgment is this plugin's job.
   - For iOS/Android/React (web): lightweight existence checks only for now (see `agents/repo-analyst.md` — deep convention detection is deferred until each platform's standards are authored).
   - **Always resolve the device type for this workflow** — exactly one of `mobile` or `tv` (the mobile-vs-TV context signal, not a new platform value, with no `mixed` value) — using `repo-analyst`'s device-type markers (tvOS device family / `TVUIKit`, Android Leanback, `react-native-tvos`, Tizen/webOS packaging). The manifest carries no device information, so this is never reused. If a repo has both targets, resolve the one this feature is about, asking the human when unclear.
```

- [ ] **Step 6: Verify the invariants survived the edit**

```bash
cd ono-mobile-dev-plugin
echo "— standards comparison retained (K4):"
grep -c "ARCH-LAYERS" agents/repo-analyst.md skills/mobile-repo-analysis/SKILL.md
echo "— platform detection still unconditional (K5):"
grep -c "always runs" agents/repo-analyst.md
echo "— device type never reused (D2):"
grep -c "no device information\|Always resolve the device type" agents/repo-analyst.md skills/mobile-repo-analysis/SKILL.md
echo "— Steps 1-5 untouched:"
grep -c "^### Step [1-5] —" agents/repo-analyst.md
echo "— file attribution rule intact:"
grep -c "File attribution rule" agents/repo-analyst.md
```

Expected: non-zero counts for the standards comparison in both files, `always runs` present, device-type language present, `5` step headings for Steps 1–5, and `1` for the file-attribution rule. Any zero means a required invariant was dropped — fix before committing.

- [ ] **Step 7: Commit**

```bash
git add agents/repo-analyst.md skills/mobile-repo-analysis/SKILL.md
git commit -m "refactor(repo-analyst): reuse canonical repo knowledge, keep platform/device/standards detection live"
```

---

## Task 10: `/analyze-feature` references repository knowledge instead of embedding it

**Files:**
- Modify: `ono-mobile-dev-plugin/commands/analyze-feature.md`
- Modify: `ono-mobile-dev-plugin/templates/feature-analysis-template.md`

**Interfaces:**
- Consumes: the Repo Knowledge Reference block shape from T8; `repo-analyst`'s extended output from T9.
- Produces: a feature analysis carrying `repo_knowledge_*` frontmatter and a `## Repo Knowledge Reference` section. T11 reads those fields.

**This is scope point 5.** The template currently mandates *"repo-analyst's structured findings, verbatim"* — a point-in-time snapshot of a moving repository, stored permanently in an approved document that three downstream stages are told to trust and not re-derive (review finding D13).

- [ ] **Step 1: Add knowledge resolution as the command's first step**

In `ono-mobile-dev-plugin/commands/analyze-feature.md`, replace step 1:

```markdown
1. Apply the `mobile-repo-analysis` skill methodology. Invoke the `repo-analyst` agent first to **detect the platform** (React Native, native iOS, native Android, React web, or a mix) using its platform-detection algorithm, before anything else.
```

with:

```markdown
1. **Resolve canonical repository knowledge, then detect the platform.** Apply the `repo-knowledge-consumer` skill first to resolve `.ono/repo-knowledge.json` — the approved repository knowledge published by the Ono Project Inspector — so this command reuses it instead of re-deriving repository facts. Then apply the `mobile-repo-analysis` skill methodology and invoke the `repo-analyst` agent to **detect the platform** (React Native, native iOS, native Android, React web, or a mix) using its platform-detection algorithm.

   - If knowledge is **available**, `repo-analyst` reuses the categories reported reusable (typically the neutral stack inventory from `docs/project/patterns.md` and the component inventory from `docs/project/components.md`) and derives only the rest. Show the developer the one-line summary the skill produces, including the freshness verdict.
   - If knowledge is **unavailable**, say so in one line and continue with full live detection — **behavior identical to before this step existed.** Do not stop, do not ask permission, and do not ask the developer to run an inspection first.
   - **Platform detection always runs in full either way.** `stack.platformHints` is advisory corroboration only; it never substitutes for detection and never for the confirmation gate in step 2.
```

- [ ] **Step 2: Change what step 7 writes into the template**

In the same file, replace step 7:

```markdown
7. Populate `templates/feature-analysis-template.md` in full: the feature request, `repo-analyst`'s findings verbatim (including the Platform Detection section), the **confirmed** `platform` (a single value — never `mixed`) and the **confirmed** `device_type` (`mobile` or `tv`) frontmatter fields, the confirmed platform architect's proposed approach as a single flat "Proposed Technical Approach" section, the four design-reference fields exactly as resolved in step 5 (`design_reference_status` — `provided` or `not_required`, never left `pending`; `design_reference_type`; `design_reference`; `figma_link`), and any open questions/risks. When the status is `not_required`, record in "Open Questions & Risks" why the feature has no user-facing UI change. Every downstream stage reads these fields rather than re-asking.
```

with:

```markdown
7. Populate `templates/feature-analysis-template.md` in full:

   - The feature request.
   - The **confirmed** `platform` (a single value — never `mixed`) and the **confirmed** `device_type` (`mobile` or `tv`) frontmatter fields.
   - The six `repo_knowledge_*` frontmatter fields and the `## Repo Knowledge Reference` section, per the `repo-knowledge-consumer` skill's Step 6 block shape.
   - `repo-analyst`'s findings in the `## Repo Context` section — **cite reused knowledge, embed only what was derived live.** A fact that came from canonical knowledge is recorded as a path plus anchor, never pasted; a fact derived live is recorded inline and labelled as a point-in-time observation. The Platform Detection, Device Type, and Standards Conformance findings are always derived live, so they are always embedded. This is the change that stops repository facts from being frozen into an approved document that outlives them.
   - The confirmed platform architect's proposed approach as a single flat "Proposed Technical Approach" section.
   - The four design-reference fields exactly as resolved in step 5 (`design_reference_status` — `provided` or `not_required`, never left `pending`; `design_reference_type`; `design_reference`; `figma_link`).
   - Any open questions/risks. When the status is `not_required`, record in "Open Questions & Risks" why the feature has no user-facing UI change.

   Every downstream stage reads these fields rather than re-asking or re-detecting.
```

- [ ] **Step 3: Update the template's frontmatter**

In `ono-mobile-dev-plugin/templates/feature-analysis-template.md`, add these fields to the YAML block immediately after `device_type`:

```yaml
repo_knowledge_status: # available | unavailable — whether .ono/repo-knowledge.json was resolved when this analysis was written
repo_knowledge_schema: # the repository-knowledge contract schema version, or null when unavailable
repo_knowledge_fingerprint: # fingerprint.gitHead from the manifest, or null — lets a later reader tell whether the cited knowledge has moved
repo_knowledge_freshness: # fresh | stale-head | stale-artifacts | unknown | null
repo_knowledge_reused: # comma-separated categories reused from canonical knowledge (e.g. conventions, inventory), or none
repo_knowledge_derived: # comma-separated categories derived live for this feature, or none
```

- [ ] **Step 4: Replace the verbatim-embedding section**

In the same file, replace this section:

```markdown
<!-- repo-analyst's structured findings, verbatim: Platform Detection (raw signals, candidate platform(s), confidence) first, then the platform-specific stack section (for react-native: Navigation, State Management, Data Fetching, Testing, Monorepo/Workspace, Folder Structure, Lint/Format; for ios/android/react: lightweight existence-check findings). Every section present even if "not detected". -->
## Repo Conventions Detected
```

with:

```markdown
<!--
Where this feature's repository context came from. Produced by the repo-knowledge-consumer skill (Step 6 block shape). When canonical knowledge was unavailable, this section says so and everything below is a point-in-time observation instead of a citation.
-->
## Repo Knowledge Reference

<!--
repo-analyst's findings. CITE reused knowledge, EMBED only what was derived live — repository facts pasted verbatim into this document go stale the moment the repository changes, and three downstream stages are instructed to trust this document without re-deriving.

Required sections, always present even when the answer is "not detected":
- Platform Detection — raw signals, candidate platform(s), confidence. ALWAYS derived live, so always embedded.
- Device Type — resolved mobile|tv and confidence. ALWAYS derived live, so always embedded.
- Stack Detection — Navigation, State Management, Data Fetching, Testing, Monorepo/Workspace, Lint/Format (react-native), or the lightweight existence-check findings (ios/android/react). Tag EVERY entry either `[reused: docs/project/patterns.md#anchor]` or `[derived live]`.
- Standards Conformance — folder structure vs the platform's ARCH-* expectations. ALWAYS derived live, so always embedded.
-->
## Repo Context
```

- [ ] **Step 5: Verify the embedding is gone and the reference is in place**

```bash
cd ono-mobile-dev-plugin
echo "— old verbatim section removed (must be 0):"
grep -c "Repo Conventions Detected" templates/feature-analysis-template.md commands/analyze-feature.md
echo "— 'verbatim' instruction removed from the command (must be 0):"
grep -c "findings verbatim" commands/analyze-feature.md
echo "— new sections present (must be 1 each):"
grep -c "^## Repo Knowledge Reference$" templates/feature-analysis-template.md
grep -c "^## Repo Context$" templates/feature-analysis-template.md
echo "— all six frontmatter fields present (must be 6):"
grep -c "^repo_knowledge_" templates/feature-analysis-template.md
echo "— backward-compatible path stated in the command:"
grep -c "identical to before this step existed" commands/analyze-feature.md
```

Expected: `0` and `0` for the removed items, `1` and `1` for the new sections, `6` frontmatter fields, and `1` for the backward-compatibility sentence.

- [ ] **Step 6: Confirm the confirmation gate was not weakened (K5)**

```bash
cd ono-mobile-dev-plugin
grep -c "Is the current feature intended for this context?" commands/analyze-feature.md
grep -c "exactly one" commands/analyze-feature.md
```

Expected: `1` for the confirmation prompt — it must still be present and unchanged — and a non-zero count for the single-platform rule.

- [ ] **Step 7: Commit**

```bash
git add commands/analyze-feature.md templates/feature-analysis-template.md
git commit -m "feat(analyze-feature): cite canonical repo knowledge instead of embedding it verbatim"
```

---

## Task 11: `/dev-design-start` references knowledge instead of re-scanning

**Files:**
- Modify: `ono-mobile-dev-plugin/commands/dev-design-start.md`
- Modify: `ono-mobile-dev-plugin/skills/dev-design-start/SKILL.md`
- Modify: `ono-mobile-dev-plugin/templates/dd-template.md`
- Modify: `ono-mobile-dev-plugin/agents/rn-architect.md`

**Interfaces:**
- Consumes: `repo_knowledge_*` frontmatter from T10; the reference block shape from T8.
- Produces: a DD carrying the knowledge reference forward. No later stage's contract changes — `/dev-feature-start` and `/implement-task` keep reading the DD exactly as they do today.

**This is scope point 6.** `dev-design-start/SKILL.md` Step 3.4 currently instructs an unconditional *"Repository inspection — browse the relevant source directories, existing components, services, and API routes"* — a full re-scan of ground the inspector has already covered.

**Backward compatibility requirement for this task:** a feature analysis written **before** T10 has a `## Repo Conventions Detected` section and no `repo_knowledge_*` fields. `/dev-design-start` must accept both shapes. An in-flight feature must not be invalidated by this change.

- [ ] **Step 1: Add knowledge resolution and legacy tolerance to the command**

In `ono-mobile-dev-plugin/commands/dev-design-start.md`, replace step 2:

```markdown
2. Read the `platform`, `device_type`, `design_reference_status`, `design_reference_type`, `design_reference`, and `figma_link` fields from the approved feature analysis — do not re-run platform or device-type detection, and **never re-ask for design input that `/analyze-feature` already recorded.**
```

with:

```markdown
2. Read the `platform`, `device_type`, `design_reference_status`, `design_reference_type`, `design_reference`, and `figma_link` fields from the approved feature analysis — do not re-run platform or device-type detection, and **never re-ask for design input that `/analyze-feature` already recorded.**

   Also read the six `repo_knowledge_*` fields and the analysis's `## Repo Knowledge Reference` section, then re-resolve current knowledge with the `repo-knowledge-consumer` skill so the DD is built against what the repository knows **now**, not only what it knew when the analysis was approved. If the fingerprint has moved since, note it in one line and continue — the consumer skill has already routed affected categories to live derivation.

   **Accept a feature analysis written before these fields existed.** An older analysis has a `## Repo Conventions Detected` section and no `repo_knowledge_*` frontmatter. That is valid input: treat its embedded findings as a point-in-time observation, resolve current knowledge yourself, and prefer canonical knowledge over the embedded snapshot where the two disagree — noting the disagreement. Never stop or ask for the analysis to be regenerated.
```

- [ ] **Step 2: Turn the unconditional re-scan into gap-filling**

In `ono-mobile-dev-plugin/skills/dev-design-start/SKILL.md`, replace item 4 of `### Step 3 — Read all available context`:

```markdown
4. **Repository inspection** — browse the relevant source directories, existing components, services, and API routes to understand the actual implementation landscape.
```

with:

```markdown
4. **Canonical repository knowledge first, source inspection only for the gaps.** Apply the `repo-knowledge-consumer` skill, then:
   - Read the documents it reports reusable rather than re-deriving them: `docs/project/components.md` for existing screens, components, and hooks the design should reuse; `docs/project/patterns.md` for the state-management, API, navigation, styling, error-handling, and i18n conventions the design must follow; `docs/project/integrations.md` for the services and SDKs §11 and §21 will reference; and the `CLAUDE.md` structure pointers for the module map §20 builds on.
   - **Then inspect source only for what canonical knowledge does not cover**: any category in `deriveLive`, plus the feature-specific detail no repository-wide document could contain — the actual signatures, props, state shape, and call sites of the specific components and services *this feature* touches. That feature-specific reading is required and is not duplication.
   - When knowledge is unavailable, browse the relevant source directories, existing components, services, and API routes directly — **exactly as this step always did.**

   The distinction to hold onto: repository-wide facts are read from the approved documents; feature-specific detail is read from source. Re-deriving a repository-wide fact that `patterns.md` already states is the duplication this step exists to remove.
```

- [ ] **Step 3: Add the frontmatter and citation rules to the skill's generation step**

In the same file, in `### Step 6 — Generate the DD`, replace the `**Frontmatter:**` bullet:

```markdown
- **Frontmatter:** carry `platform`, `device_type`, and all four design-reference fields (`design_reference_status`, `design_reference_type`, `design_reference`, `figma_link`) from the feature analysis unchanged; set `feature_analysis_link` to the analysis path, `status: draft`, `detail_level`, `date`.
```

with:

```markdown
- **Frontmatter:** carry `platform`, `device_type`, and all four design-reference fields (`design_reference_status`, `design_reference_type`, `design_reference`, `figma_link`) from the feature analysis unchanged; set `feature_analysis_link` to the analysis path, `status: draft`, `detail_level`, `date`. Also set the six `repo_knowledge_*` fields from **this run's** knowledge resolution — not copied from the analysis, since the repository may have moved since it was approved. When the analysis predates those fields, fill them from this run and note in §23 Assumptions that the analysis carried an embedded snapshot instead of a citation.
- **Repository knowledge citations:** include the `## Repo Knowledge Reference` section per the `repo-knowledge-consumer` skill's Step 6 shape. In §19 and §20, cite the canonical documents for repository-wide facts (`docs/project/patterns.md#<anchor>`, `docs/project/components.md#<anchor>`, the `CLAUDE.md` structure pointers) rather than restating their contents. Reserve inline description for feature-specific detail read from source.
```

- [ ] **Step 4: Update the DD template**

In `ono-mobile-dev-plugin/templates/dd-template.md`, add to the YAML block immediately after `device_type`:

```yaml
repo_knowledge_status: # available | unavailable — resolved when this DD was written, not copied from the feature analysis
repo_knowledge_schema: # the repository-knowledge contract schema version, or null
repo_knowledge_fingerprint: # fingerprint.gitHead at DD authoring time, or null
repo_knowledge_freshness: # fresh | stale-head | stale-artifacts | unknown | null
repo_knowledge_reused: # comma-separated categories reused from canonical knowledge, or none
repo_knowledge_derived: # comma-separated categories derived live while writing this DD, or none
```

Immediately after the closing ``` of that YAML block and before `## 1. Feature Overview`, insert:

```markdown
<!-- Produced by the repo-knowledge-consumer skill. Records which repository knowledge this design was built on, so a reviewer can resolve the same sources and tell whether they have moved since. When canonical knowledge was unavailable, says so — and everything repository-wide below is then a point-in-time observation rather than a citation. -->
## 0. Repo Knowledge Reference
```

Then extend the two section comments. Replace the §19 comment's final line:

```markdown
Exactly one confirmed platform applies, carried over from the feature analysis — write this as a single flat section from that one platform's architect, never split into per-platform subsections.
```

with:

```markdown
Exactly one confirmed platform applies, carried over from the feature analysis — write this as a single flat section from that one platform's architect, never split into per-platform subsections.
Cite canonical repository knowledge for repository-wide facts (`docs/project/patterns.md#<anchor>` for conventions, `docs/project/components.md#<anchor>` for existing components to reuse) instead of restating them. Describe inline only the feature-specific detail read from source.
-->
```

...replacing the existing closing `-->` of that comment block. And replace the §20 comment:

```markdown
<!-- Every file, module, component, or service that will need to change, with a brief note on what changes. Feeds /dev-feature-start's task decomposition. -->
```

with:

```markdown
<!-- Every file, module, component, or service that will need to change, with a brief note on what changes. Feeds /dev-feature-start's task decomposition. Build on the module map via the CLAUDE.md structure pointers rather than re-deriving it, and name reused components from docs/project/components.md by path — a "new" component that already exists is the most common failure this section can prevent. -->
```

- [ ] **Step 5: Update `rn-architect`'s inputs**

In `ono-mobile-dev-plugin/agents/rn-architect.md`, replace the first `## Inputs` bullet:

```markdown
- `repo-analyst`'s structured findings summary (navigation/state/data-fetching/testing/folder conventions actually in use), used only when the detected platform is react-native.
```

with:

```markdown
- `repo-analyst`'s structured findings summary (navigation/state/data-fetching/testing/folder conventions actually in use, each entry labelled `[reused: <path>#<anchor>]` or `[derived live]`), used only when the detected platform is react-native.
- The existing **component inventory** — `docs/project/components.md` — when canonical repository knowledge reports it reusable. Read it before proposing any new screen, component, or hook.
```

And insert a new step immediately before the existing step 4 (`Propose feature-folder placement…`), renumbering the steps that follow:

```markdown
4. **Check what already exists before proposing anything new.** When the component inventory is available, consult it for screens, reusable components, shared hooks, the navigation map, and the known-duplicates list. For each element the feature needs, state explicitly whether you are reusing an existing one (name it by path) or introducing a new one (say why nothing existing fits). Proposing a component that already exists is the failure this step prevents — that document exists specifically to be read before speccing a feature. When the inventory is unavailable, say so and proceed from `repo-analyst`'s findings as before.
```

- [ ] **Step 6: Verify the edits**

```bash
cd ono-mobile-dev-plugin
echo "— unconditional re-scan instruction removed (must be 0):"
grep -c "^4\. \*\*Repository inspection\*\* — browse" skills/dev-design-start/SKILL.md
echo "— gap-filling language present:"
grep -c "only for the gaps\|only for what canonical knowledge does not cover" skills/dev-design-start/SKILL.md
echo "— legacy analysis tolerated (backward compat):"
grep -c "Repo Conventions Detected" commands/dev-design-start.md
echo "— DD frontmatter fields (must be 6):"
grep -c "^repo_knowledge_" templates/dd-template.md
echo "— DD section 0 present (must be 1):"
grep -c "^## 0. Repo Knowledge Reference$" templates/dd-template.md
echo "— architect reuse-first step present:"
grep -c "Check what already exists" agents/rn-architect.md
echo "— architect step numbering contiguous:"
grep -oE "^[0-9]+\. " agents/rn-architect.md | tr -d '. ' | tr '\n' ' '
```

Expected: `0` for the removed instruction; non-zero for gap-filling; `1` for the legacy tolerance in the command; `6` DD frontmatter fields; `1` for section 0; non-zero for the architect step; and a contiguous ascending run for the architect's `## Process` numbering.

- [ ] **Step 7: Confirm no downstream stage contract changed**

```bash
cd ono-mobile-dev-plugin
git diff --name-only HEAD~4 -- commands/ | sort
```

Expected: exactly `commands/analyze-feature.md` and `commands/dev-design-start.md`. Any other command appearing here means scope creep — `/dev-feature-start`, `/implement-task`, `/review-code`, `/review-security`, `/fix-review-comments`, `/create-dev-qa-notes`, and `/prepare-mobile-release` are all out of scope.

- [ ] **Step 8: Commit**

```bash
git add commands/dev-design-start.md skills/dev-design-start/SKILL.md templates/dd-template.md agents/rn-architect.md
git commit -m "feat(dev-design-start): read canonical repo knowledge, inspect source only for gaps"
```

---

## Task 12: End-to-end verification across three repository states, and release

**Files:**
- Modify: `ono-mobile-dev-plugin/README.md`
- Modify: `ono-mobile-dev-plugin/CHANGELOG.md`
- Modify: `ono-mobile-dev-plugin/.claude-plugin/plugin.json` (version)

**Interfaces:**
- Consumes: everything from T1–T11.
- Produces: the evidence for scope point 7. **This task substantiates the plan's central claim and must not be skipped.**

- [ ] **Step 1: Build the three fixture repositories**

```bash
mkdir -p /tmp/rk-e2e && cd /tmp/rk-e2e && rm -rf never inspected stale

# A — never inspected: no .ono/ at all. The backward-compatibility case.
mkdir -p never/src && cd never && git init -q && git config user.email t@e.com && git config user.name T
printf '{\n  "name": "demo",\n  "dependencies": { "react-native": "0.74.0", "react": "18.2.0" }\n}\n' > package.json
printf 'module.exports = {};\n' > metro.config.js
git add -A && git commit -qm init && cd ..

# B — inspected and fresh: full artifact set + a manifest emitted by the real helper.
cp -r never inspected && cd inspected
# NOTE: this CLAUDE.md must include `## Repository Structure`, `## Key Modules`, and
# `## Entry Points`, because project-analysis's real template emits all three and
# `coverage.structure` is derived honestly from how many of them actually exist
# (populated / partial / unknown). A fixture omitting them yields
# `coverage.structure: unknown` and puts `structure` in deriveLive — a correct result
# for an unrepresentative fixture, but not the path this end-to-end check is testing.
printf '# CLAUDE.md — Demo\n\n## Tech Stack\n\n- Language(s): TypeScript\n- Framework(s): React Native\n- Platform(s): iOS, Android\n- Runtime / Tooling: Metro\n- Package manager(s): yarn\n\n## Repository Structure\n\n```text\nsrc/\n```\n\n## Key Modules\n\n| Module / Folder | Responsibility |\n|---|---|\n| src/features | feature modules |\n\n## Entry Points\n\nindex.js, src/App.tsx\n\n## Build, Run, and Test Commands\n\n```bash\n# Install dependencies\nyarn\n\n# Run / develop\nyarn ios\n\n# Test\nyarn test\n\n# Build\nUnknown\n```\n' > CLAUDE.md
printf '# AUDIT.md — Demo\n\n## Audit Topics\n\n| # | Status | Topic | Priority | File | Notes |\n|---|--------|-------|----------|------|-------|\n| 1 | Approved | Architecture | High | audits/architecture/architecture-audit.md | ok |\n' > AUDIT.md
mkdir -p docs/project audits/architecture
printf '# Patterns — Demo\n\n## State Management\nRedux Toolkit.\n\n## Navigation Patterns\nreact-navigation, typed routes.\n\n## Testing Patterns\nJest.\n' > docs/project/patterns.md
printf '# Component Inventory — Demo\n\n## Screens\n\n| Screen | Path | Purpose | Notes |\n|---|---|---|---|\n| LoginScreen | src/screens/LoginScreen.tsx | auth entry | reuse for biometric login |\n\n## Reusable UI Components\n\n| Component | Path | Purpose | Reuse notes |\n|---|---|---|---|\n| PrimaryButton | src/components/PrimaryButton.tsx | primary CTA | reuse, do not recreate |\n' > docs/project/components.md
printf '# Overview — Demo\n\n## What This Project Is\nA demo app.\n' > docs/project/overview.md
printf '# Architecture Audit\n\nStatus: Approved\n' > audits/architecture/architecture-audit.md
git add -A && git commit -qm artifacts
bun /Users/yagilcohen/Developer/AI/ono-plugin-project-inspector/scripts/repo-knowledge.ts emit "$(pwd)"
cd ..

# C — inspected then drifted: patterns.md hand-edited after the manifest was written.
cp -r inspected stale && cd stale
printf '\n## Styling Conventions\nStyleSheet only.\n' >> docs/project/patterns.md
cd ..
ls -d never inspected stale
```

Expected: the emit prints `Emitted .ono/repo-knowledge.json …`, and the three directories exist.

- [ ] **Step 2: Verify the reader's verdict for each state**

```bash
cd /Users/yagilcohen/Developer/AI/ono-mobile-dev-plugin
for state in never inspected stale; do
  echo "════ $state ════"
  node scripts/read-repo-knowledge.ts "/tmp/rk-e2e/$state" \
    | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{const j=JSON.parse(s);
      console.log("exit-safe   :", true);
      console.log("available   :", j.available);
      console.log("reason      :", j.reason);
      console.log("freshness   :", j.freshness);
      console.log("reuse       :", j.usableCategories.join(",") || "(none)");
      console.log("derive live :", j.deriveLive.join(",") || "(none)");})'
done
```

Expected exactly:

| State | `available` | `reason` | `freshness` | Reuse includes | `deriveLive` includes |
|---|---|---|---|---|---|
| `never` | `false` | `absent` | `null` | (none) | all seven |
| `inspected` | `true` | `null` | `fresh` | `stack`, `commands`, `structure`, `inventory`, `conventions`, `auditTopics` | `integrations` |
| `stale` | `true` | `null` | `stale-artifacts` | `stack`, `commands`, `structure`, `inventory`, `auditTopics` | `conventions`, `integrations` |

The `stale` row is the important one: `conventions` moves out of reuse and into live derivation because `patterns.md` changed, while `inventory` stays reusable because `components.md` did not. Per-document attribution, not all-or-nothing invalidation.

- [ ] **Step 3: Verify the producer is idempotent and unaffected by the drift**

```bash
cd /tmp/rk-e2e/inspected
cp .ono/repo-knowledge.json /tmp/rk-before.json
bun /Users/yagilcohen/Developer/AI/ono-plugin-project-inspector/scripts/repo-knowledge.ts emit "$(pwd)" >/dev/null
node -e 'const a=require("/tmp/rk-before.json"),b=require("/tmp/rk-e2e/inspected/.ono/repo-knowledge.json");
delete a.generatedAt; delete b.generatedAt;
console.log(JSON.stringify(a)===JSON.stringify(b) ? "PASS idempotent" : "FAIL not idempotent"); '
bun /Users/yagilcohen/Developer/AI/ono-plugin-project-inspector/scripts/repo-knowledge.ts validate "$(pwd)"
```

Expected: `PASS idempotent`, then `repo-knowledge.json is valid (schema v1, produced by ono-project-inspector 0.9.0); coverage unknown: integrations`.

- [ ] **Step 4: Run every automated test in both plugins**

```bash
cd /Users/yagilcohen/Developer/AI/ono-plugin-project-inspector && bun scripts/repo-knowledge.test.ts && bun scripts/slugify.ts
cd /Users/yagilcohen/Developer/AI/ono-mobile-dev-plugin && node scripts/read-repo-knowledge.test.ts && node scripts/resolve-target-repo-root.test.ts
```

Expected: `ALL TESTS PASSED` three times and four `PASS` lines from `slugify`, with no failures anywhere.

- [ ] **Step 5: Manually verify the backward-compatibility claim end to end**

This step is a human judgment call and cannot be scripted. In a Claude Code session with both plugins installed:

1. Run `/analyze-feature Add biometric login to the auth flow` against `/tmp/rk-e2e/never`.
   **Expect:** one line stating repository knowledge is unavailable, then the platform-confirmation prompt exactly as today, then full live detection, then a populated feature analysis. Nothing blocks; nothing asks you to run an inspection first. **This is the scope-point-7 acceptance test.**
2. Run the same command against `/tmp/rk-e2e/inspected`.
   **Expect:** the reuse summary naming `conventions` and `inventory`; the `Repo Context` section citing `docs/project/patterns.md#state-management` rather than pasting it; the architect naming `PrimaryButton` / `LoginScreen` as reuse candidates by path; `integrations` derived live; the platform-confirmation prompt still presented.
3. Run it against `/tmp/rk-e2e/stale`.
   **Expect:** a one-line staleness note naming `docs/project/patterns.md`, a `/inspect-sync` recommendation, `conventions` derived live, `inventory` still reused, and no attempt to modify any inspector-owned file.
4. Approve the analysis from run 2 and run `/dev-design-start biometric-login`.
   **Expect:** a DD with `## 0. Repo Knowledge Reference` populated, §19/§20 citing canonical paths, and source inspection confined to the specific components the feature touches.

Record the outcome of all four runs. If run 1 differs in any way from today's behavior beyond the single informational line, **stop and fix before releasing** — that is the one thing this MVP promises not to change.

- [ ] **Step 6: Clean up the fixtures**

```bash
rm -rf /tmp/rk-e2e /tmp/rk-before.json /tmp/contract-diff.txt
```

- [ ] **Step 7: Update the README**

In `ono-mobile-dev-plugin/README.md`, insert a new section immediately before `## How platform detection works`:

```markdown
## Repository knowledge comes from the inspector, not from re-analysis

When a repository has been inspected by [`ono-project-inspector`](https://github.com/OnOAppsDev/ono-plugin-project-inspector), it carries approved repository knowledge at `.ono/repo-knowledge.json` — a deterministic, versioned index over `CLAUDE.md`, `AUDIT.md`, and `docs/project/*.md`. `/analyze-feature` and `/dev-design-start` **read that instead of re-deriving it**: the stack, build/test commands, module map, component inventory, and coding conventions are consumed rather than re-scanned, and generated documents *cite* those sources by path and anchor instead of pasting a copy that goes stale.

Two properties matter:

- **It is optional.** A repository with no manifest behaves exactly as it always has — full live detection, no prompt, no warning beyond one informational line. Most repositories start here.
- **It never overrides a decision.** The manifest's `platformHints` is advisory corroboration only. Platform detection and the human confirmation gate run in full on every feature, and `device_type` is resolved live every time — the manifest carries no device information.

What is still derived live on every run, regardless of the manifest: platform detection, device type, folder-structure conformance against the platform's `ARCH-*` standards, per-diff file attribution, and any knowledge category the manifest reports as `unknown`.

See `docs/repo-knowledge-contract.md` for the schema and the obligations this plugin accepts as a consumer.
```

And in the `## Plugin internals` tree, add to the `skills/` block:

```text
  repo-knowledge-consumer/          (resolves canonical repository knowledge; the only reader of the contract)
```

- [ ] **Step 8: Bump the version and write the changelog**

Set `"version": "0.4.0"` in `ono-mobile-dev-plugin/.claude-plugin/plugin.json`. Prepend to `CHANGELOG.md`:

```markdown
## [0.4.0] - 2026-07-28

### Added
- `scripts/read-repo-knowledge.ts` — deterministic reader for the repository-knowledge
  manifest `ono-project-inspector` publishes at `.ono/repo-knowledge.json`. Computes
  freshness (git HEAD plus per-document SHA-256) and applies the degradation matrix,
  reporting which knowledge categories may be reused and which must still be derived
  live. **Always exits 0 with valid JSON**, including when the manifest is absent,
  malformed, or written by a newer contract version — so a missing manifest can never
  fail a command.
- `skills/repo-knowledge-consumer` — the single component in this plugin that
  understands the manifest format. Defines the resolution procedure and the
  `Repo Knowledge Reference` block that generated documents record.
- `docs/repo-knowledge-contract.md` — the contract schema and this plugin's obligations
  as a consumer, pinned to schema v1 and duplicated verbatim in the producer.
- `scripts/read-repo-knowledge.test.ts` — a test per degradation-matrix row.

### Changed
- `repo-analyst` resolves canonical repository knowledge before detecting anything, and
  no longer re-derives the neutral stack inventory (navigation, state management, data
  fetching, test runner, monorepo tooling, lint/format) when `docs/project/patterns.md`
  already records it. Its findings summary now leads with a Repository Knowledge section
  and labels every stack finding `[reused: <path>#<anchor>]` or `[derived live]`.
- `/analyze-feature` resolves repository knowledge as its first step, and the feature
  analysis **cites** repository knowledge instead of embedding `repo-analyst`'s findings
  verbatim. `templates/feature-analysis-template.md`'s `## Repo Conventions Detected`
  section becomes `## Repo Knowledge Reference` plus `## Repo Context`, with six new
  `repo_knowledge_*` frontmatter fields. Repository facts pasted into an approved
  document went stale the moment the repository changed, while three downstream stages
  were instructed to trust them.
- `/dev-design-start` re-resolves current knowledge and reads the approved documents
  first; `dev-design-start`'s Step 3.4 source inspection is now confined to gaps and to
  the feature-specific detail no repository-wide document can contain. The DD carries the
  same six frontmatter fields plus a `## 0. Repo Knowledge Reference` section.
- `rn-architect` consults `docs/project/components.md` before proposing new screens,
  components, or hooks, and states for each element whether it is reusing an existing one
  (by path) or introducing a new one (and why nothing existing fits).

### Unchanged (deliberately)
- **Behavior with no `.ono/repo-knowledge.json` is designed to be identical to 0.3.0** —
  the reader always exits 0 with `available: false`, and both changed commands fall back to
  full live derivation, adding only one informational line. The deterministic layer of this
  is covered by tests and by a fixture check against a never-inspected repository; the
  end-to-end confirmation is a live `/analyze-feature` run, recorded as a required manual
  verification step. Do not claim "verified end to end" in this entry until that run has
  actually been performed and its result recorded.
- Platform detection, the single-platform confirmation gate, and `device_type` resolution
  run in full on every feature. `platformHints` is advisory only and never authoritative.
- Folder-structure conformance against the platform's `ARCH-*` standards is still derived
  live on every run — `docs/project/patterns.md` describes what the conventions are, not
  whether they comply with Ono's standards.
- The eight-stage pipeline, every approval gate, all three safety hooks, all templates
  outside the two above, all `standards/**`, and the seven commands outside
  `/analyze-feature` and `/dev-design-start`.
```

- [ ] **Step 9: Final verification**

```bash
cd /Users/yagilcohen/Developer/AI/ono-mobile-dev-plugin
node -e 'const p=require("./.claude-plugin/plugin.json"); if(p.version!=="0.4.0")process.exit(1); console.log("mobile-dev at",p.version);'
git status --short
git diff --stat HEAD~5
```

Expected: `mobile-dev at 0.4.0`; a clean-but-for-this-task `git status`; and a diff touching only the files this plan lists.

- [ ] **Step 10: Commit**

```bash
git add README.md CHANGELOG.md .claude-plugin/plugin.json
git commit -m "docs(repo-knowledge): document canonical knowledge consumption, release 0.4.0"
```

---

## Self-review against scope

| Scope point | Where it is implemented | Verified by |
|---|---|---|
| 1. Inspector is the single producer | T3 (skill + registry), T4 (checkpoints), T5 (facts block, integrations pointer) | T3 step 4, T4 step 4, T5 step 7 |
| 2. Deterministic versioned `.ono/repo-knowledge.json` | T1 (skeleton), T2 (extraction) | T1 scenario 2 (byte-determinism), T2 all scenarios, T12 step 3 (idempotence) |
| 3. Mobile-dev consumes instead of re-deriving | T6 (reader), T8 (consumer skill), T9–T11 (wiring) | T6 rows 2–4, T12 step 2 |
| 4. `repo-analyst` narrowed to feature-specific detection | T9 | T9 step 6 (all five invariants) |
| 5. `/analyze-feature` references instead of embeds | T10 | T10 steps 5–6 |
| 6. `/dev-design-start` references instead of re-scans | T11 | T11 steps 6–7 |
| 7. 100% backward compatible when the manifest is absent | T6 (always exit 0), T8 Step 5, T10 step 1, T11 step 1 | T6 rows 1/5/6/10 + step 6, T12 step 2 (`never` row), **T12 step 5 run 1** |

Out-of-scope items confirmed absent: no `cautions` key (asserted in T1 scenario 1); no `.ono/feature-state.json`; no new hook in either plugin; no registry field added; no QA-plugin change; no change to the seven out-of-scope commands (asserted in T11 step 7).

---

## Before starting

Confirm or correct **D1** (the inspector does not own platform authority — `platformHints` is advisory only) and **D2** (`device_type` stays entirely in mobile-dev). Those two shape T1's schema and T9's narrowing; if either is wrong, both tasks change materially. D3–D6 are lower-stakes and can be adjusted during implementation.

