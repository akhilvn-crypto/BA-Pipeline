# ba-requirements

A Claude Code plugin that turns Markdown business requirements into validated atomic requirements, a triaged gap register and development stories with acceptance criteria, plus an offline HTML report.

## Quick start

**1. Install** (requires Claude Code and Python 3; no packages are installed):

```text
/plugin marketplace add akhilvn-crypto/BA-Pipeline
/plugin install ba-requirements@ba-plugin-marketplace
```

**2. Open Claude Code in your project folder** (the one that holds, or will hold, your requirements `.md` files). All output is written under this folder, never into the plugin.

**3. Run the pipeline:**

```text
/ba-requirements:run
```

It guides you through the whole flow and asks before it overwrites anything:

| Step | What happens | What you do |
|---|---|---|
| 1. Source | Finds existing artifacts, asks where the requirements are. | Give a file, folder or path; confirm the match. |
| 2. Phase 1: atomic requirements | Splits the requirements into atomic, traceable `REQ-xxx` items, validated and saved. | Nothing. |
| 3. Phase 2: gap analysis | Finds missing, ambiguous or conflicting information, prioritises it (Critical to Low) and builds the gap register and dashboard. | Nothing. |
| 4. Phase 2: triage | Shows gaps one at a time, Critical first. Every Critical and High gap must be resolved before stories are generated; if any is left open or skipped, you are returned to triage for those gaps. Medium and Low gaps may stay open. | Give a decision, skip the gap, skip all, or stop (progress is saved). |
| 5. Phase 3: stories | Appends a `# Development Stories` section to your requirements document, then enriches only the stories your decisions affect. | Pick the target document if there are several; confirm the preview. |
| 6. Report | Writes `report/phase3-report.html` and prints a summary with real counts. | Open the report in a browser. |

**Resume at any time.** `/ba-requirements:run` records a checkpoint after every operation (in `.ba-requirements/pipeline-state.json`). Run it again after an interruption and it shows the last completed operation and asks whether to **Resume** or **Start a new run**. A resume re-checks what was completed, skips what is still valid and continues from the first incomplete or invalid operation; stories keep their IDs and answered gaps stay answered. Edit your requirements, or settle skipped gaps later, and the affected operations (and the ones after them) are rerun.

Prefer to go step by step? Every phase is also a standalone command, see below.

### Typical workflows

| I want to... | Run |
|---|---|
| Do everything in one go | `/ba-requirements:run` |
| Only break requirements down | `/ba-requirements:start-analysis` |
| Find and review gaps | `/ba-requirements:analyse-gaps`, then `/ba-requirements:triage-gaps` |
| Settle skipped gaps later and update the stories | `/ba-requirements:triage-gaps`, then `/ba-requirements:enrich-stories` |
| Accept my manual edits to the stories | `/ba-requirements:sync-stories` |
| Refresh the HTML report only | `/ba-requirements:generate-report` |

### What you get

```text
<your-project>/
├── <your requirements>.md      original content untouched; '# Development Stories' appended (the only part written)
├── atomic_requirements/        atomic_requirements.json
├── gaps/                       gaps.json (register), gap-analysis.html (dashboard)
├── stories/                    story-generation-state.json, backups/
└── report/                     phase3-report.html (offline, read-only)
```

## Overview

- **Phase 1** reads Markdown requirement documents, decomposes them into atomic requirements, validates the result and saves it as JSON in your project.
- **Phase 2** analyses those atomic requirements for gaps, prioritises them, keeps a JSON gap register and an HTML dashboard, and guides you through triaging the gaps one at a time.
- **Phase 3** writes development stories into a `# Development Stories` section at the end of your original Markdown requirements document, enriches only the affected stories when gaps are resolved, detects and reconciles manual edits, and keeps a consolidated HTML report of stories, gaps and business decisions.

Out of scope: Jira integration, story approval workflows, test cases, writing gap decisions back into the atomic requirements, and any change to your documents outside the `Development Stories` section or to `atomic_requirements.json`.

## Commands

| Command | Purpose |
|---|---|
| `/ba-requirements:run` | **Master.** Runs all three phases in order: source, atomic requirements, gap analysis, gap triage, stories, enrichment and the HTML report. |
| `/ba-requirements:start-analysis` | Phase 1. Decompose Markdown requirements into atomic requirements. |
| `/ba-requirements:analyse-gaps` | Phase 2. Identify and prioritise gaps; create or reconcile the gap register and HTML report. |
| `/ba-requirements:triage-gaps` | Phase 2. Review open and skipped gaps one at a time and record decisions. |
| `/ba-requirements:generate-stories` | Phase 3. Generate development stories (open or skipped gaps do not block it). |
| `/ba-requirements:enrich-stories` | Phase 3. Apply decisions from resolved gaps to the stories they affect. |
| `/ba-requirements:sync-stories` | Phase 3. Detect and reconcile manual changes to the stories section and its mappings. |
| `/ba-requirements:generate-report` | Phase 3. Rebuild the HTML report only; no story changes. |

## Master pipeline — `/ba-requirements:run`

One command for the whole workflow. It is an orchestration layer only: each phase is carried out by following the existing phase command (`start-analysis`, `analyse-gaps`, `triage-gaps`, `generate-stories`, `enrich-stories`, `generate-report`), so all validation, confirmation and safe-write behaviour is the same as running them one by one, and every phase command stays usable on its own.

0. Reads the checkpoint first (see "Checkpoint and resume" below) and offers to resume an unfinished run.
1. Asks where the requirements are, then detects existing artifacts (atomic requirements, gap register, stories section, report). Nothing is overwritten silently; a valid existing `atomic_requirements.json` can be reused or regenerated, with a warning if a source file is newer than it.
2. **Phase 1.** Atomic requirements, validated and saved. A failure stops the pipeline.
3. **Phase 2.** Gap analysis, saved register and gap report (validation gate), then interactive triage one gap at a time: give a decision, skip this gap, skip all remaining gaps, or stop (the pipeline pauses; every answer already given is saved and `/ba-requirements:run` resumes).
4. **Story-generation gate.** Phase 3 starts only when every **Critical and High** gap is `resolved`. If any is open or skipped, including after "skip" or "skip all remaining", the pipeline names them and returns you to triage for those gaps; you can also exit and resume later. Medium and Low gaps may stay open or skipped and do not block.
5. **Phase 3.** Stories are generated, then enriched using only the decisions of resolved gaps. Stories keep exactly four fields; mappings stay in `stories/story-generation-state.json`.
6. Refreshes `report/phase3-report.html`. If that fails, the document and state stay valid, the previous report is kept, the run is recorded as stopped at the report (not completed), and the summary says the report is stale or unavailable; resuming reruns only the report.
7. Runs a final validation, marks the run completed only if it succeeds, and prints a summary of actual counts and statuses (run ID, last completed, skipped and rerun operations, sources, requirements, gaps by priority and status, decisions, stories generated and enriched, validation per phase, report path and status, remaining follow-up).

### Checkpoint and resume

The pipeline has nine operations, each checkpointed separately: resolve the source, generate atomic requirements, gap analysis, gap triage, the Critical/High gate, story generation, enrichment, the HTML report, and final validation. The state is written before an operation starts (`in_progress`) and immediately after it completes, through a temporary file and an atomic replace, so a crash never leaves a half-written checkpoint. During triage every answer, skip and status change is recorded as it happens. The checkpoint holds the run ID, status (`in_progress`, `completed`, `blocked`, `failed`), source files, the current and last completed operation, per-operation status, timestamps, input and output fingerprints, output paths and concise errors, and never copies requirements, gaps, stories or report content.

On resume the completed operations are re-validated before they are trusted:

- **Changed input.** Source files are fingerprinted outside the managed `# Development Stories` section (so the stories Phase 3 writes do not count as a change). If a source file or an upstream artifact changed, that operation and everything after it are reset and rerun; existing decisions, gap IDs and story IDs are preserved by the phases' own reconciliation.
- **Missing or invalid output.** A missing, corrupted or no-longer-valid artifact reruns the operation that produces it and what follows.
- **Interrupted operation.** An operation that was `in_progress` is simply rerun; the phase commands are safe to rerun.
- **Source changed mid-run.** The run stops before applying stale results, names the change and asks whether to restart from source analysis.
- **Gate.** Unresolved Critical or High gaps keep `gate_validation` blocked and refuse to start story generation, on a first run and after any resume. Open or skipped Medium and Low gaps never block.
- **Damaged checkpoint.** A missing, corrupt or incompatible file is reported and a fresh run is offered; the old file is archived under `.ba-requirements/history/`, never silently discarded. A completed run is never reused as a new execution.

The individual phase commands keep working on their own; when they change an artifact they refresh the checkpoint so a later `/ba-requirements:run` does not trust a stale completion record.

The gate applies only to the master command. `/ba-requirements:generate-stories` still works on its own and does not require resolved gaps. A failed gate, a pause or an exit prints a "stopped" or "paused" summary that names the phase, the reason (including the blocking gaps), what was preserved and how to continue. Dependencies are never installed automatically.

## Phase 1 — `/ba-requirements:start-analysis`

1. The command asks where your requirements are: a folder name, directory path or filename.
2. It resolves the location. Ambiguous matches and multi-file directories are shown for you to confirm.
3. The Requirement Analyst reads every confirmed file completely and decomposes it.
4. The output is validated, then saved to `<project-root>/atomic_requirements/atomic_requirements.json`.
5. A summary reports the real counts, paths and any limitations.

An existing output file is never replaced without asking. Input is Markdown (`.md`) only; directories are searched recursively, hidden and generated directories (including `atomic_requirements/`) are skipped.

Output is a JSON array; every object has exactly three properties:

```json
[
  {
    "requirement_id": "REQ-001",
    "title": "User Login Authentication",
    "description": "The system shall authenticate users using their registered email address and password. Access shall be granted only when the supplied credentials are valid."
  }
]
```

IDs are unique and sequential from `REQ-001` on initial generation. Schema: `schemas/atomic-requirement.schema.json`.

## Phase 2 — gap analysis and triage

Run Phase 1 first. Then, from the same project directory:

```text
/ba-requirements:analyse-gaps
/ba-requirements:triage-gaps
```

### `/ba-requirements:analyse-gaps`

1. Validates `atomic_requirements/atomic_requirements.json`. A missing, unreadable, invalid or empty file stops the command and nothing is created.
2. The Gap Analysis Agent examines every requirement for missing, ambiguous, incomplete or conflicting information that needs a business decision. It does not invent rules or choose answers.
3. Each gap gets one priority (Critical, High, Medium, Low), a clear definition, and every affected requirement with its ID and description.
4. The draft is validated, then saved safely (temp file, atomic replace, read-back, re-validation) to `gaps/gaps.json`. `gaps/gap-analysis.html` is generated from the saved JSON.
5. A summary reports the real counts.

Run it again at any time:

- **Input unchanged** (SHA-256 of the atomic requirements matches `gaps/gap-analysis-state.json`): the existing register is reused. No duplicates, no changes to decisions or statuses.
- **Input changed**: the requirements are diffed by ID and content, re-analysed, and reconciled with the register. Matched gaps keep their IDs, statuses and decisions; new gaps get new IDs. You approve every material change before it is saved: resolved gaps whose requirements changed (keep resolved or reopen, never silently), and gaps whose requirements were all removed (re-point them, or retire them to `gaps/retired-gaps.json` with your approval).
- A fresh analysis never overwrites an existing register. A reset needs your explicit confirmation and backs the old files up to `gaps/backup-<timestamp>/`.

### `/ba-requirements:triage-gaps`

Presents open and skipped gaps one at a time, Critical first, then High, Medium, Low, and by gap number within a priority:

- **Provide a decision** — it is captured, clarified if incomplete, read back to you, and recorded only after you confirm. The gap becomes `resolved`.
- **Skip this gap** — it becomes `skipped`; still unresolved and offered again next session.
- **Skip all remaining** — ends the session without changing the remaining gaps.
- **Reopen a resolved gap** (on request) — back to `open`; its decision history is kept.
- **Revise a decision** (on request) — a new decision is appended; earlier ones stay.

Every change is saved immediately and the HTML report is regenerated. Run the command again to resume. It does not re-run analysis; if the atomic requirements changed it tells you to run `analyse-gaps`.

### Artifacts

```text
<project-root>/
├── atomic_requirements/
│   └── atomic_requirements.json    Phase 1 output (read only in Phase 2)
└── gaps/
    ├── gaps.json                   authoritative gap register
    ├── gap-analysis.html           generated, read-only dashboard
    └── gap-analysis-state.json     internal control file (input hash, timestamps, versions)
```

`gaps/retired-gaps.json` and `gaps/backup-<timestamp>/` appear only after you approve a retirement or a reset.

### Gap register format

A JSON array. Every gap has exactly these six properties:

```json
[
  {
    "gap_id": "GAP-001",
    "priority": "Critical",
    "status": "open",
    "gap_definition": "Should the system lock an account after repeated failed login attempts? If so, what is the maximum number of failed attempts and how long should the account remain locked?",
    "decisions_recorded": [],
    "affected_requirements": [
      {
        "requirement_id": "REQ-001",
        "description": "The system shall authenticate users using their registered email address and password."
      }
    ]
  }
]
```

`status` is `open`, `skipped` or `resolved`. Each decision is `{"decision", "recorded_at"}` with an ISO 8601 timestamp; decisions are append-only and the latest is current. Schema: `schemas/gap-register.schema.json`.

### HTML dashboard

Self-contained (no server, no external resources). Summary cards, priority distribution, and separate Open, Resolved and Skipped sections. Search by gap ID, definition or requirement ID, filter by priority and status, expand or collapse details. It is a read-only view of `gaps.json`; decisions are made only through `/ba-requirements:triage-gaps`.

## Phase 3 — development stories

Run Phase 1 first. Phase 2 is optional for the first generation: stories are written from the requirements as they stand, and open or skipped gaps never block them. A missing gap register only means gap traceability is unavailable.

```text
/ba-requirements:generate-stories
/ba-requirements:triage-gaps          (Phase 2, whenever you want to settle gaps)
/ba-requirements:enrich-stories
/ba-requirements:sync-stories         (after you edit the stories by hand)
/ba-requirements:generate-report
```

### `/ba-requirements:generate-stories`

1. Validates `atomic_requirements/atomic_requirements.json` (missing or invalid stops the command) and reads `gaps/gaps.json` if it exists. An invalid register: you either correct it or explicitly proceed without gap context.
2. Asks which Markdown document should hold the stories. Phase 1 does not record the files it analysed, so this is asked once and remembered. For a folder with several files you select one primary document; stories are never appended to every file.
3. The Story Generation Agent groups and splits the requirements by business capability and writes each story. It never decides an open gap or turns a possible answer into a criterion.
4. The draft is validated, previewed, and written by `scripts/sync-development-stories.py`: manual edits and concurrent changes are detected, a backup is made, the document is written to a temporary file and replaced atomically, then read back and verified.
5. The state is saved, the HTML report is generated, and the summary reports real counts.

Run it again at any time: existing stories are matched and keep their IDs; only requirements without stories get new ones; nothing is duplicated.

### `/ba-requirements:enrich-stories`

For each resolved gap whose latest decision has not been reviewed (new or revised), only the stories built from the affected requirements are considered. A story changes only if the decision changes its behaviour; IDs and unrelated criteria are kept; no criterion is duplicated. A decision that needs no story change is recorded as such (and still reaches the report). A resolved gap that was reopened never deletes criteria on its own: you choose to retain, revise or remove them. A decision that needs a new story is proposed to you, not created silently. Running it again with nothing new changes nothing.

### `/ba-requirements:sync-stories`

Detects edited, added, deleted and re-numbered stories, edits elsewhere in the document, and requirement or gap-register changes. You choose what to accept; accepted edits are recorded in the story state without touching the document. Sections that break the story contract are reported and left for you to fix.

### `/ba-requirements:generate-report`

Rebuilds `report/phase3-report.html` from the saved document, the gap register and the state. It never changes a story, an ID, the document or a recorded decision. Use it after a report failure, or whenever you want a fresh view.

### The story format

Each story in the `# Development Stories` section has exactly four fields. The section is always the last part of the document.

```markdown
# Development Stories

## US-001

**Summary:** User authentication using registered credentials

**Detailed Description:**
As a registered user, I want to log in using my registered email address and
password so that I can securely access my account.

**Acceptance Criteria:**

1. Given a registered user provides valid credentials, when the user submits
   the login form, then the system authenticates the user and grants access.

2. Given a user provides invalid credentials, when the user submits the login
   form, then the system rejects the login attempt.
```

No requirement IDs, gap IDs, priorities, statuses or decisions appear in a story. Those mappings live in `stories/story-generation-state.json`; gaps and decisions are in the report. Story IDs (`US-001`, ...) are unique, sorted numerically, kept when a story is updated, and never reused after a story is removed.

### Phase 3 artifacts

```text
<project-root>/
├── <your requirements>.md          '# Development Stories' section appended / updated (the only part Phase 3 writes)
├── stories/
│   ├── story-generation-state.json internal: IDs, requirement and gap mappings, hashes, decision ledger
│   └── backups/                    copy of the document before each change to it
└── report/
    └── phase3-report.html          generated, read-only, works offline
```

### The HTML report

Self-contained (no server, CDN or database). Sections: project overview (timestamp, target document, story and gap counts, validation and synchronisation status), Development Stories, Open Gaps, Resolved Gaps (with decisions and resolution time), Skipped Gaps, Business Decisions (history with the current decision marked, and whether it is reflected in the stories) and a separate Traceability section. Story entries show only ID, summary, description and acceptance criteria. All source text is HTML-escaped. Anything not recorded is shown as `Not recorded`; if there is no valid gap register the gap sections say so instead of showing anything. If a refresh fails, the previous report is kept and the command says it is stale.

### Safety rules in short

- Only the `# Development Stories` section is written. Everything else in the document is preserved byte for byte (encoding, line endings, BOM).
- If another top-level heading follows the section, the heading is duplicated or look-alike, or the boundary is otherwise unclear, nothing is written until you fix it.
- A story you edited by hand is never overwritten, a story is never removed, and a deleted story is never restored without your consent for that story.
- If the document changes while Phase 3 is working, nothing is overwritten.
- If a write or its read-back verification fails, the previous document is kept (or restored). The state is saved only after the document is verified.

## Install and test locally

Requires Claude Code and Python 3 (standard library only; nothing is installed). To install from the marketplace, see [Quick start](#quick-start).

To test the plugin from a local checkout:

```text
claude --plugin-dir ./ba-requirements-plugin
```

Then run the commands from your project directory. Use `/reload-plugins` after editing the plugin.

## Scripts

| Script | Purpose |
|---|---|
| `scripts/validate-atomic-requirements.py` | Phase 1 validator. |
| `scripts/validate-gap-register.py` | Validates structure and semantics of `gaps.json`. Exit `0` valid, `1` invalid, `2` unreadable. |
| `scripts/generate-gap-report.py` | Validates `gaps.json` and writes `gap-analysis.html`. No report is written for an invalid register. |
| `scripts/gap-register-store.py` | Safe persistence used by both Phase 2 commands: `inspect`, `save`, `list`, `resolve`, `skip`, `reopen`, `summary`, `regenerate-report`. |
| `scripts/story_common.py` | Phase 3 library: section boundary detection, story parsing and rendering, hashing, paths, safe writes, input and state loading. |
| `scripts/validate-development-stories.py` | Validates a structured story draft (structure, IDs, references, duplicates, coverage, applied decisions) or the managed section of a document. Exit `0` valid, `1` invalid, `2` unreadable. |
| `scripts/sync-development-stories.py` | Safe persistence used by the Phase 3 commands: `inspect`, `export`, `apply`, `accept`, `summary`, `regenerate-report`. |
| `scripts/generate-phase3-report.py` | Builds `report/phase3-report.html`. |
| `scripts/pipeline-state.py` | Checkpoint and resume state for `/ba-requirements:run`: `inspect`, `begin`, `resume`, `start`, `check-inputs`, `complete`, `fail`, `block`, `triage-event`, `invalidate`, `refresh`, `gate`, `summary`. Exit `0` ok, `1` refused or blocked, `2` unreadable. |

```text
python scripts/validate-atomic-requirements.py atomic_requirements/atomic_requirements.json
python scripts/validate-gap-register.py gaps/gaps.json
python scripts/generate-gap-report.py --project-root .
python scripts/validate-development-stories.py draft.json --project-root .
python scripts/validate-development-stories.py --document requirements/business-requirements.md --project-root .
python scripts/generate-phase3-report.py --project-root .
```

The gap validator checks JSON syntax, the exact property sets, enumerations, ID format, unique IDs, valid and current requirement references (IDs and descriptions), no duplicate affected requirement, at least one affected requirement per gap, resolved gaps having decisions, valid ISO 8601 timestamps in chronological order, and no duplicate gap definitions. It never repairs anything.

## Tests

```text
python -m unittest discover -s tests -v
```

`tests/test_master_pipeline.py` checks that `commands/run.md` is well formed and refers only to files, scripts and commands that exist, that the phase commands do not depend on it, and drives the real scripts through a full run, all gaps skipped, partly resolved gaps, an interrupted and resumed triage and failed validation gates. `tests/test_pipeline_state.py` drives the checkpoint script through crash-safe writes, resume from every kind of interruption, change detection and downstream invalidation, corrupt, incompatible and inconsistent checkpoints, per-answer triage recovery and the Critical/High gate before and after a resume. The tests cover the validators, safe persistence (including simulated failed saves and concurrent edits), triage state changes, the HTML reports (statuses, decisions, escaping), unchanged-input detection and reconciliation safeguards, and for Phase 3 the section boundary rules, story ID handling, idempotent re-runs, enrichment bookkeeping, manual-edit detection and reconciliation, and the report. Judgements that depend on the model (quality of the gaps found, story decomposition and wording, whether a decision changes a story, asking for clarification) are specified in the agents, skills and rules and are not unit-testable.

## Layout

| Path | Purpose |
|---|---|
| `commands/` | `run` (master), `start-analysis`, `analyse-gaps`, `triage-gaps`, `generate-stories`, `enrich-stories`, `sync-stories`, `generate-report` |
| `agents/` | `requirement-analyst` (Phase 1), `gap-analysis-agent` (Phase 2), `story-generation-agent` (Phase 3) |
| `skills/` | Source discovery, decomposition, quality validation; gap identification, gap prioritisation, gap triage; story generation, acceptance-criteria design, story enrichment |
| `rules/` | Mandatory standards for atomic requirements, IDs, artifacts, gap analysis and triage, story writing, acceptance criteria and story management |
| `schemas/` | `atomic-requirement.schema.json`, `gap-register.schema.json`, `development-stories.schema.json` |
| `scripts/` | Validators, report generators, register store, story sync and persistence |
| `tests/` | Unit tests for the Phase 2 and Phase 3 scripts |

## License

See `LICENSE` (copyright holder placeholder to be completed by the owner).
