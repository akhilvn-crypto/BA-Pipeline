---
description: Run the complete BA Requirements pipeline - atomic requirements, gap analysis and triage, development stories, enrichment and the consolidated HTML report (BA Requirements, master command)
allowed-tools: Read, Write, Glob, Grep, Agent, AskUserQuestion, Bash(python:*), Bash(python3:*), Bash(py:*)
---

# BA Requirements — Run (master pipeline)

You orchestrate the whole pipeline. This command contains **no phase logic of its own**: every phase is carried out by following the existing phase command file, with the hand-offs, gates and overrides below. Do not improvise methodology, hand-write any artifact, or bypass the scripts those commands use.

Plugin root: `${CLAUDE_PLUGIN_ROOT}`
Project root: the current working directory of this session. Every generated artifact goes under the project root, never into the plugin directory.

If a path below still contains the literal text `${CLAUDE_PLUGIN_ROOT}`, find the plugin directory by searching for `commands/run.md` under the directory that contains this plugin's `.claude-plugin/plugin.json` (name `ba-requirements`) and use that.

Use whichever of `python`, `python3`, `py` works. If none does, tell the user Python 3 is required and ask before installing or doing anything else. Never install dependencies automatically. Do not introduce a database or vector store.

## How to run a phase

For each phase, `Read` the phase command file in full and follow its steps and rules exactly, including every validation, confirmation and safe-write step. The files are the single source of truth:

| Phase | Command file | Role here |
|---|---|---|
| 1 | `${CLAUDE_PLUGIN_ROOT}/commands/start-analysis.md` | atomic requirements |
| 2a | `${CLAUDE_PLUGIN_ROOT}/commands/analyse-gaps.md` | gap register and gap report |
| 2b | `${CLAUDE_PLUGIN_ROOT}/commands/triage-gaps.md` | interactive triage |
| 3a | `${CLAUDE_PLUGIN_ROOT}/commands/generate-stories.md` | development stories |
| 3b | `${CLAUDE_PLUGIN_ROOT}/commands/enrich-stories.md` | enrichment from resolved decisions |
| 3c | `${CLAUDE_PLUGIN_ROOT}/commands/generate-report.md` | consolidated HTML report |

Overrides that apply to every phase file when it runs under this command:

- **Skip its closing "Execution summary".** Print one-line phase progress instead (see "Progress lines"). The only full summary is the final one in Step 8.
- **"Stop" means stop the pipeline.** Where a phase file says to stop, explain why, run Step 9 (pipeline stopped) and do not start a later phase.
- **Do not repeat a question already answered here** (the source location, the target document).
- Ignore any "Next step: run /ba-requirements:..." suggestions in the phase files; this command is the next step.
- Every counts, status and path reported must come from the scripts' JSON output or the agents' reports. Never use illustrative or estimated numbers.

Keep a short in-conversation ledger as you go: source files confirmed and their count, atomic requirement count, per-phase validation results, gap counts before and after triage, gaps presented / resolved / skipped in this run, stories created and updated, decisions incorporated, report status. The final summary is built from it plus a fresh `summary` call.

### Progress lines

After each phase gate print one line, for example `Phase 1 — Atomic requirements: PASSED (<n> requirements)`, using real values. Use `PASSED`, `REUSED`, `FAILED`, `PAUSED` or `SKIPPED`.

## Step 1 — Ask for the source (always first)

Your first message must be this question, and nothing else happens before the answer. Stop and wait for the reply.

> Where does your requirements document reside? You can provide a folder name, directory path, or individual filename.

## Step 2 — Detect existing work

Do this before any phase runs, and never overwrite anything silently.

1. Resolve and confirm the source exactly as Phase 1 Steps 2–3 describe (`requirement-source-discovery` skill: Markdown only, ambiguity asked, multi-file sets confirmed, outside-project sources confirmed). Keep the confirmed absolute `.md` paths.
2. Inspect the three artifact areas with the read-only inspectors:

```text
python ${CLAUDE_PLUGIN_ROOT}/scripts/gap-register-store.py inspect --project-root <project root>
python ${CLAUDE_PLUGIN_ROOT}/scripts/sync-development-stories.py inspect --project-root <project root>
```

   Both are read-only and may report missing or invalid inputs; that is information, not an error, at this point. Also check whether `report/phase3-report.html` exists.

3. Decide the starting point for Phase 1:

| Finding | Action |
|---|---|
| `atomic_requirements/atomic_requirements.json` absent | Run Phase 1. |
| Present but fails `validate-atomic-requirements.py` | Tell the user the existing file is invalid and why. Phase 1 will not overwrite it silently; its Step 7 asks replace / save under another name / cancel. Run Phase 1. |
| Present and valid | Ask with AskUserQuestion: **reuse it** or **re-run Phase 1**. Show its requirement count and date, and the evidence about staleness (below). Recommend reuse only when no source file is newer. Re-running goes through Phase 1 Step 7, which asks before replacing, and warns that IDs can change and gap and story mappings then need reconciliation. |

Staleness evidence for a valid existing file: Phase 1 does not record its source hashes, so compare file modification times of the confirmed source files with `atomic_requirements.json` (for example `python -c "import os,sys; ..."`) and report plainly whichever source files are newer. Say this is a heuristic, not proof.

4. If the gap register or stories section already exists, say so in one line (counts from the inspectors), so the user knows this is a resume and that Phases 2 and 3 will reconcile, not restart.

## Step 3 — Phase 1: atomic requirements

If reusing the existing file, skip to the gate. Otherwise follow `start-analysis.md` Steps 4–7 (draft location, Requirement Analyst, independent validation, safe persistence). Step 1 of that file (the source question) and the confirmation in Step 3 are already done.

**Gate — must pass to continue.** The saved `atomic_requirements/atomic_requirements.json` must pass:

```text
python ${CLAUDE_PLUGIN_ROOT}/scripts/validate-atomic-requirements.py <project root>/atomic_requirements/atomic_requirements.json
```

Stop the pipeline (Step 9) if the source cannot be read, was only partially processed, the file is empty, validation fails, the save or read-back check fails, or the user cancels. Never run Phase 2 after a Phase 1 failure. Record: source files analysed, requirement count, validation result.

## Step 4 — Phase 2a: gap analysis

Follow `analyse-gaps.md` Steps 1–7 and 9 (inspect, agent, validation, reconciliation review, safe save, verify). Reconciliation of an existing register, including keep/reopen and retire decisions, is asked of the user exactly as that file specifies. A destructive reset (its Step 8) happens only if the user asks for it.

**Gate — must pass to continue.** `validate-gap-register.py` passes on the saved `gaps/gaps.json`, `gap-register-store.py inspect` reports `UNCHANGED`, and `gaps/gap-analysis.html` exists (a failed HTML regeneration with a saved, valid register is a warning: run `regenerate-report` once, then continue and report it as stale if it still fails). Stop the pipeline (Step 9) on `INPUT_*`, `REGISTER_INVALID`, a failed save or a failed validation. If the register is empty (no gaps), say so and continue to Phase 3 with triage skipped.

Record: gaps by priority and status from `gap-register-store.py summary`.

## Step 5 — Phase 2b: interactive gap triage

Run `gap-register-store.py list`. If `eligible_count` is 0, report `Phase 2 triage: SKIPPED (no open or skipped gaps)` and go to Phase 3.

Otherwise follow `triage-gaps.md` Steps 1–3 (the `gap-triage` skill): one gap at a time, Critical first, each decision confirmed before it is recorded and saved immediately through the store. The user can:

- provide a decision, which resolves the gap once confirmed;
- **skip** the current gap (it becomes `skipped` and stays unresolved);
- **skip all remaining gaps**: nothing further is written, the remaining gaps keep their status, and the pipeline **continues to Phase 3**;
- **stop / exit**: the pipeline **pauses**. Everything already confirmed is already saved. Do not start Phase 3. Go to Step 9 with status `PAUSED` and tell the user that running `/ba-requirements:run` again resumes (the source and Phase 1 and 2 outputs are detected and reused, and triage returns to the remaining open and skipped gaps).

If the session is interrupted unexpectedly, no special handling is possible or needed: each answer was saved the moment it was confirmed, and the next run resumes from the artifacts.

Skipped and open gaps are never treated as resolved, and they never block Phase 3. A failed save of a decision is reported and offered a retry; it is not ignored.

After triage, run `gap-register-store.py summary` and record gaps presented / resolved / skipped in this run and the register totals by status. Print `Phase 2 — Gaps: <open> open, <skipped> skipped, <resolved> resolved`.

## Step 6 — Phase 3: development stories

Follow `generate-stories.md` Steps 1–8 with these hand-offs:

- **Target document.** If the confirmed source in Step 2 is a single `.md` file, use it as `--target`. If several files were confirmed, ask the user to select the single primary document that holds the `# Development Stories` section (Step 3 of that file). If `inspect` already shows a recorded target, reuse it and do not ask again. Never add the section to several files.
- **Gap register.** The register was validated in Phase 2, so pass no `--ignore-gaps`. If the register has become invalid or unreadable, stop the pipeline (Step 9); do not continue without gap context from this command. The user can run `/ba-requirements:generate-stories` on its own to make that choice explicitly.
- **Open and skipped gaps do not block story generation.** Generate stories from the validated atomic requirements regardless. Never decide an open or skipped gap and never invent answers or acceptance criteria.
- **Story contract.** Each story in the document holds exactly four fields: Story ID, Summary, Detailed Description, Acceptance Criteria. Requirement-to-story and gap-to-story mappings stay in `stories/story-generation-state.json` only. `validate-development-stories.py` enforces this; treat its failure as a gate failure.
- **Safe update.** Keep using the dry-run, conflict, `--expected-source-sha256`, backup and read-back protections. If the managed section or a story was edited by hand, the conflict process in the phase file asks the user before anything is changed. Existing story IDs and unrelated content are preserved.
- If the stories already cover all requirements and nothing changed, report that and keep the existing section untouched (Step 4 of that file), then continue to enrichment.

**Gate — must pass to continue.** `apply` returned `ok` true (exit `0`, or `3` where only the report is stale), and `summary` reports story validation passed. Stop the pipeline (Step 9) on any other result. Exit `4` (document saved, state not) is not complete: follow that file's `accept` guidance and re-check before continuing.

Record: stories created (from `apply`), stories in the document, requirements covered, validation result.

## Step 7 — Phase 3: enrichment and report

**Enrichment.** Run `sync-development-stories.py inspect` again. Then follow `enrich-stories.md` Steps 2–9:

- Use only confirmed decisions from **resolved** gaps. Open and skipped gaps are not decisions.
- With `pending_decisions` empty and no reopened gap needing a change, that file's Step 4 applies: report that no enrichment was required, change no story.
- Reopened gaps, new-story proposals and conflicts are put to the user exactly as that file specifies.

Record: decisions incorporated, decisions needing no story change, stories enriched (from `apply`), stories unchanged, validation result.

**Report.** Follow `generate-report.md` to refresh `report/phase3-report.html` from the authoritative artifacts, unless an `apply` in this run already produced it and `summary` shows it current. The report shows development stories, open gaps, resolved gaps, skipped gaps, recorded business decisions, summary counts and a generation timestamp; stories, gaps and decisions stay in their own sections and no gap or requirement metadata appears inside a story entry. All of that is done by the script; you only check the result.

If report generation fails: the requirements document and the state are valid and stay as they are, the previous report is retained, and the final summary says clearly that the report is **STALE** (a previous report exists) or **UNAVAILABLE** (none exists), with the reason and `/ba-requirements:generate-report` as the fix. This is not a reason to undo anything and does not turn a completed pipeline into a failed one, but never print `GENERATED` for it.

## Step 8 — Final execution summary

Run `gap-register-store.py summary` and `sync-development-stories.py summary` once more and print this, replacing every placeholder with an actual value (counts and statuses from script output and the ledger; priority counts from the register, for example by reading `gaps/gaps.json` with `python -c` and counting `priority` against `status`; decisions recorded as the total number of entries in the `decisions_recorded` arrays). Omit nothing; write `0` or `not run` where that is the truth.

```text
BA REQUIREMENTS PIPELINE — <COMPLETED | COMPLETED WITH FOLLOW-UP | STOPPED | PAUSED>

Source files analysed: <n>
Atomic requirements generated: <n>   (<new | reused existing file>)

Gaps (<total>):
- By priority:  Critical <n>, High <n>, Medium <n>, Low <n>
- By status:    Open <n>, Skipped <n>, Resolved <n>
- This run:     presented <n>, resolved <n>, skipped <n>
Business decisions recorded: <n total>   (<n> this run)

Stories:
- Generated this run: <n>   (in document: <n>)
- Enriched this run: <n>    (decisions incorporated: <n>, needing no story change: <n>)
- Requirements covered: <n> of <n>

Validation:
- Phase 1 atomic requirements: <Passed | Failed | Not run>
- Phase 2 gap register:        <Passed | Failed | Not run>
- Phase 3 stories:             <Passed | Failed | Not run>

Artifacts:
- atomic_requirements/atomic_requirements.json
- gaps/gaps.json, gaps/gap-analysis.html
- <target document> (Development Stories section; backup: <path>)
- stories/story-generation-state.json
- HTML report: <project root>/report/phase3-report.html — <GENERATED <timestamp> | STALE: <reason> | UNAVAILABLE: <reason>>

Follow-up required:
- <n> open gaps and <n> skipped gaps remain unresolved; stories contain only what the requirements and resolved decisions support. Re-run /ba-requirements:run (or /ba-requirements:triage-gaps then /ba-requirements:enrich-stories) after they are decided.
- <any stale report, unreconciled manual edit, requirement not covered, or warning from earlier steps>
```

Use `COMPLETED` only when all phases passed their gates, the report is `GENERATED`, and no open or skipped gaps remain. Use `COMPLETED WITH FOLLOW-UP` when the pipeline finished but gaps remain open or skipped, or the report is stale or unavailable. Remind the user that original requirement text outside the `# Development Stories` section was not modified, and that the report is a read-only view.

## Step 9 — Pipeline stopped or paused

When a gate fails, the user cancels, or the user stops triage, print this instead of Step 8, with actual values:

```text
BA REQUIREMENTS PIPELINE — <STOPPED | PAUSED>

Stopped at: <phase and step>
Reason: <exact error or user choice, with the script's details>

Completed before the stop:
- <phase>: <result and artifact path>

Not started: <phases>

Preserved: <artifacts kept as they were, including all gap answers and triage status already saved>
To continue: <the fix, if any>, then run /ba-requirements:run again; completed phase outputs are detected and reused after their validity is checked.
```

Never claim a later phase ran, never call a stopped pipeline completed, and do not delete or roll back valid earlier outputs.
