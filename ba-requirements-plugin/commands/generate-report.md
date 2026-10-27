---
description: Rebuild the Phase 3 HTML report from the current stories, gaps and decisions without changing any story (BA Requirements, Phase 3)
allowed-tools: Read, Bash(python:*), Bash(python3:*), Bash(py:*)
---

# BA Requirements — Generate Report (Phase 3)

Plugin root: `${CLAUDE_PLUGIN_ROOT}`
Project root: the current working directory of this session.

Rebuild `report/phase3-report.html` only. This command never rewrites the requirements document, renumbers or changes stories, edits `stories/story-generation-state.json`, or changes the gap register or any recorded decision. Use whichever of `python`, `python3`, `py` works; if none does, tell the user Python 3 is required and ask before doing anything else.

If a path below still contains the literal text `${CLAUDE_PLUGIN_ROOT}`, find the plugin directory by searching for `scripts/generate-phase3-report.py` under the directory that contains this plugin's `.claude-plugin/plugin.json` (name `ba-requirements`).

## Step 1 — Regenerate

```text
python ${CLAUDE_PLUGIN_ROOT}/scripts/sync-development-stories.py regenerate-report --project-root <project root>
```

The report is built from the saved `# Development Stories` section of the recorded target document, from `gaps/gaps.json` and from the story state (traceability only). It is written to a temporary file, replaced, read back and checked. A failed run leaves the previous report untouched.

## Step 2 — Check the result

Read the JSON. Do not report success unless `ok` is true and `report.generated` is true.

- Exit `1`: the report cannot be built. Typical causes: no story state yet (run `/ba-requirements:generate-stories`), the recorded document is missing or its section boundary is ambiguous (see `/ba-requirements:sync-stories`). Say exactly which, and that any previous report is out of date.
- Exit `2`: a file could not be read or written. Say what failed and that the previous report, if any, is stale.

## Step 3 — Summarise

Run `summary --project-root <project root>` for the figures, and report real values only:

```text
Phase 3 report regenerated.

Report: report/phase3-report.html
Stories: <summary.stories>
Gaps: <open / skipped / resolved from summary.gaps.register_counts, or "gap data unavailable">
Story validation: <Passed | Failed>
Synchronisation: <In sync | Edited since the last sync: run /ba-requirements:sync-stories>

Stories, gaps and decisions were not changed.
```

If `summary` says the document was edited since the last sync, point out that the report shows the document as it is now and the section is flagged "Out of sync". If the report is missing sections because gap data is unavailable, say so rather than implying a complete view. Remind the user the report is a read-only view: stories are edited in the requirements document and decisions are recorded through `/ba-requirements:triage-gaps`.
