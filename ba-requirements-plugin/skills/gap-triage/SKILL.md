---
name: gap-triage
description: Guided one-at-a-time triage of requirement gaps - presenting a gap, capturing and confirming a business decision, skipping one gap or all remaining gaps, reopening a resolved gap and resuming later. Use when running /ba-requirements:triage-gaps.
---

# Gap Triage

Follow `rules/gap-triage-rules.md` for statuses, transitions and decision history. This skill is the conversation method. All reads and writes of the register go through `scripts/gap-register-store.py`; use `python`, `python3` or `py`, whichever works.

Commands used below (each takes `--project-root <project root>`):

```text
python <plugin-root>/scripts/gap-register-store.py list
python <plugin-root>/scripts/gap-register-store.py resolve --gap-id GAP-001 --decision-file <file> [--revise]
python <plugin-root>/scripts/gap-register-store.py skip --gap-id GAP-001
python <plugin-root>/scripts/gap-register-store.py reopen --gap-id GAP-001
python <plugin-root>/scripts/gap-register-store.py summary
python <plugin-root>/scripts/gap-register-store.py regenerate-report
```

Each prints JSON with `ok`. Exit 0 = done, 1 = refused (nothing changed), 2 = I/O failure (nothing changed), 3 = register saved but the HTML report failed.

## Presenting a gap

Show, in plain readable text, the gap's ID, priority and one concise gap statement (shorten `gap_definition` to a single sentence without changing its meaning), then the choices for its priority:

```text
Gap 2 of 12 — GAP-005 — Critical (skipped earlier)

<one concise gap statement>

Previous decisions: <only if the gap was reopened; show them with dates>

Choices:
- Skip for now — skip this gap and continue.
- Other — enter a description or decision in free text.
```

**Critical and High gaps** offer exactly these choices:

- **Skip for now** — skip this gap and continue.
- **Other** — the user enters a description or decision in free text.

**Medium and Low gaps** offer exactly these choices:

- **Skip all remaining gaps** — skip all remaining Medium/Low gaps in the current triage run.
- **Skip this gap** — skip only the current gap and continue.
- **Other** — the user enters a description or decision in free text.

Present the choices as text and take the user's reply; do not use AskUserQuestion for them (a Critical/High gap has only one fixed choice besides free text). Choose the list by the gap's `priority`. "Skip all remaining gaps" is never offered for a Critical or High gap.

Reopening a resolved gap, revising a recorded decision and stopping stay available when the user asks for them, but they are not listed as choices.

## Capturing a decision

1. Take the text the user entered under **Other** (a decision or a description) as the starting point. If it is only a description or question and not a decision, answer or clarify as needed and show the choices again.
2. Check it against the gap definition. If the gap asks "whether, how many, how long" and the answer covers only part, ask one focused follow-up for the missing part. Ask again only if still unclear.
3. Treat these as not yet decisions, and offer to skip instead of recording them: "I don't know", "TBD", "ask someone else", "whatever is standard".
4. Do not propose a business answer, fill gaps with your own assumptions or choose between alternatives for the user. You may restate ambiguity and ask which meaning is intended.
5. Write the captured decision as the user's decision, in clear language and without added detail. Show it back and ask for confirmation with AskUserQuestion: "Confirm and record", "Change the wording", "Do not record". Only "Confirm and record" resolves the gap.
6. After confirmation, write the decision text to a file in the session's temporary directory with the Write tool, then run `resolve --gap-id ... --decision-file <file>`. Passing a file avoids shell quoting problems.
7. Read the JSON result. On success say that the decision was recorded and the gap is resolved, and that the report was regenerated. On any failure say what failed and that nothing was recorded; offer to retry.
8. Move to the next gap.

## Skip for now / Skip this gap

Run `skip --gap-id` at once. Say it is marked skipped, remains unresolved and is kept for later triage; for a Critical or High gap add that it still blocks story generation until resolved. Move on.

## Skip all remaining gaps (Medium and Low only)

Only offered on a Medium or Low gap. Record each skip immediately: for the current gap and every later queue entry that is Medium or Low and still `open`, run `skip --gap-id`, one gap at a time (a gap already `skipped` stays skipped; no call is needed). Read each JSON result; if one fails, stop, say which gap failed and that the earlier skips are saved. Say how many gaps were marked skipped, that they remain unresolved and are kept for later triage, and that nothing was resolved by this. Go to the session summary. Critical and High gaps are never touched by this choice.

## Reopen a resolved gap

Only when the user asks. If no ID was given, ask which gap. Run `reopen --gap-id`. Say that its decision history is kept and that it is open again. Ask whether to revisit it now; if yes, present it next, otherwise continue.

## Revise a decision

Only when the user asks to change a recorded decision. Capture and confirm the new decision as above, then run `resolve --revise`. The old entry stays in the history; the new one becomes current.

## Resume

When a session starts and the queue already contains skipped gaps, say how many are open and how many were skipped before. Resolved gaps are not shown. If the queue is empty, say that no open or skipped gaps remain and show the summary counts.

## Session tally

Keep a count in the conversation of: gaps presented (each at most once per session), gaps resolved, gaps skipped (by this session's actions). The end summary also uses `summary` for the register's real totals.
