# Inspection Workflow

**Scope:** what to type, in what order, to inspect a repository — and what each command does.
**Plugin repository:** [`OnOAppsDev/ono-plugin-project-inspector`](https://github.com/OnOAppsDev/ono-plugin-project-inspector)
**See also:** [`plugin-architecture.md`](plugin-architecture.md) for *how* the plugin is built, and [`docs/architecture/ecosystem-overview.html`](../../architecture/ecosystem-overview.html) for how this fits the wider feature pipeline.

> Command names and file paths below are relative to the **plugin repository**.

---

## Commands

Five commands, all thin wrappers around `agents/project-inspector.md`, which dispatches on the mode each one declares.

| Command | Mode | Responsibility |
|---|---|---|
| `/inspect [repo]` | `full` | Run or resume the complete guided workflow. Default entry point. |
| `/inspect-status [repo]` | `status` | Read-only progress report. Invokes no skill and writes nothing. |
| `/inspect-topic [topic] [repo]` | `targeted` | Break down one audit topic — a named one, or the next pending — skipping the general narrative. |
| `/inspect-approve [repo]` | `resume` | Approve the current gate: finalize the reviewed Draft and continue, or advance one stage. |
| `/inspect-sync [repo]` | `maintenance` | Documentation maintenance, on demand. Not part of the linear workflow. |

You never need to know a skill's name. Every command already knows which skills it needs and lets the agent sequence them.

---

## Starting an inspection

```text
/inspect /path/to/repository
```

Or just `/inspect`, and the agent will ask for the path.

**`/inspect` is state-aware — it does not blindly start from the beginning.** It first detects where the repository stands, then presents a status summary and a tailored set of choices, and waits:

- **No inspection exists** → offer to start one, or leave everything unchanged.
- **In progress** → show completed stages, the current stage, topic counts and the recommended next action; offer to continue, review and approve the open Draft, break down the next topic, run maintenance, or leave unchanged.
- **Complete** → report completion and offer maintenance only.

**Nothing is written until you choose to start or continue**, so a curious `/inspect` never changes the repository.

---

## The stages

### Stage 1 — `project-analysis`

Asks its own intake questions (repository path, scope, how to handle existing artifacts), confirms, then writes **only** `CLAUDE.md` and `AUDIT.md`. It does not create `audits/`.

The agent then verifies both artifacts exist at the real repository root, refreshes the knowledge manifest, reports the audit topic count, and **stops for approval**.

**You review** `CLAUDE.md` and `AUDIT.md`, then approve.

### Stage 2 — `project-docs`

Requires `CLAUDE.md` to exist. Reads the existing context first, inspects source read-only to fill the gaps, and writes the four `docs/project/` files: `overview.md`, `components.md`, `patterns.md`, `integrations.md`. It never touches `CLAUDE.md`, `AUDIT.md` or `audits/`.

The agent verifies, refreshes the manifest, and **stops for approval**.

**You review** the four files — `components.md` accuracy especially, since downstream feature work reads it to avoid rebuilding what already exists.

### Stage 3 — the breakdown-approve loop

Not a single stage but a loop over `Pending Breakdown` topics, one at a time.

1. **`audit-breakdown`** asks which pending topic to process (default: the first), confirms, then writes exactly one `audits/<slug>/<slug>-audit.md` and sets that row to `Draft`.
2. The agent runs a deterministic slug and status check, then **stops at the Draft's review gate**.
3. **You review the Draft.** On approval — `/inspect-approve`, or saying so in conversation — **`audit-approve`** validates the topic is `Draft`, flips its row to `Approved`, and confirms the permanent file reference. Running it *is* the approval, so the agent does not stop again here.
4. The agent immediately breaks down the **next** pending topic and stops at that new Draft's gate.

**No second Draft is ever generated without a review in between.** When no pending topics and no open Drafts remain, the agent reports Stage 3 complete.

---

## Resume

Throughout, the agent keeps `<repo>/.ono/state.json` current via the internal `inspection-state` skill. Because of it, restarting `/inspect` or running `/inspect-approve` after an interruption continues exactly where you left off — the reviewed Draft still awaiting approval, or the next topic to break down — instead of re-deriving progress from files.

`AUDIT.md` stays the source of truth for topic status; the state file only mirrors it. Commit `.ono/state.json` so progress is shared with the team.

---

## Repository knowledge for other plugins

Alongside the state file, the agent keeps `<repo>/.ono/repo-knowledge.json` current — after `project-analysis`, after `project-docs`, after each `audit-approve`, and after `audit-sync`.

This is the canonical index other OnO plugins read instead of re-analyzing the repository: what the stack is, how to build and test it, where the modules and entry points are, and where to find the component inventory, the conventions and the integrations. It carries pointers into the approved artifacts rather than copies, and reports any category it could not determine as `unknown` so a consumer derives that part itself.

You never invoke it directly. A repository inspected by an earlier plugin version gets its manifest on the next `/inspect-sync` — no re-inspection needed. Commit it alongside `.ono/state.json`.

See the plugin repository's `docs/repo-knowledge-contract.md` for the schema. It ships inside the plugin, alongside the code that implements it.

---

## Maintenance, outside the workflow

`audit-sync` is a documentation-maintenance tool, not a workflow stage. Run it with `/inspect-sync` once one or more topics are `Approved`, or any time later, to:

- regenerate the two managed blocks inside `CLAUDE.md` from all currently-Approved topics,
- verify `AUDIT.md` consistency and detect documentation drift,
- repair broken audit references inside its own managed blocks,
- refresh the knowledge manifest.

It never marks anything `Approved` — that is `audit-approve`'s job — and treats `AUDIT.md` as read-only. Re-runs replace the managed blocks rather than duplicating them, so it is safe to run repeatedly.

---

## Shortcuts onto the same steps

`/inspect-status` can be run at any point and never changes anything. `/inspect-approve` performs whichever gate is currently open. `/inspect-topic <name>` performs one breakdown cycle directly for a specific topic without running `/inspect` first. All of them read the same registry and the same `AUDIT.md` the guided flow uses, so state never diverges between commands.

---

## What never happens

- The agent never reads or writes repository source files directly.
- No stage runs without the developer having approved the previous one.
- No second `audit-breakdown` Draft is generated without a review of the previous Draft in between.
- Only `audit-approve` ever changes a topic to `Approved`, and only on explicit developer approval.
- `audit-sync` never writes `AUDIT.md` and never runs as part of the normal workflow.
- No skill creates implementation plans, tickets, or source-code patches — that is out of scope for the entire plugin by design.
