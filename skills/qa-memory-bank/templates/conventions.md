# Conventions

## Issue IDs

- Format: `ISSUE-NNNN` — zero-padded, monotonically increasing across all categories.
- Allocate the next ID by running `ls qa-memory-bank/issues/{open,ready-for-retest,verified-fixed,wont-fix}` and taking max + 1.
- File name: `ISSUE-NNNN-<short-kebab-slug>.md` (e.g. `ISSUE-0007-login-rejects-valid-mfa-code.md`).

## Severity

| Level    | When to use |
|----------|-------------|
| **S0 — Blocker** | App unusable for a role: cannot log in, primary navigation broken, data loss, security breach (auth bypass, sensitive-data leak). |
| **S1 — Critical** | Core feature broken: cannot create/read/update primary entity, feature throws unhandled error, broken navigation on a major flow. |
| **S2 — Major** | Significant degradation: secondary feature broken, data displayed incorrectly, validation missing on important field, slow page (>5s) on critical path. |
| **S3 — Minor** | Cosmetic, layout, copy, low-impact UX gap (e.g., missing confirmation dialog on delete, wrong icon, inconsistent spacing). |
| **S4 — Trivial** | Typo, small visual polish, suggestion for nice-to-have. |

When unsure, pick the higher severity — easier to downgrade than miss.

## Issue file structure

Use [`issues/ISSUE-TEMPLATE.md`](./issues/ISSUE-TEMPLATE.md) as a starting point. Required sections:

- Header: ID, title, severity, role, area, status, dates.
- **Summary** — one paragraph, plain English.
- **Steps to reproduce** — numbered, deterministic. Include test account used.
- **Expected vs Actual.**
- **Evidence** — screenshot path(s), console excerpt, network excerpt.
- **Environment** — URL, browser, viewport, build commit if known.
- **Notes** — hypothesis, related issues, suggested fix area (file path/line if obvious from codebase).

## Issue lifecycle

```
        ┌────── filed ──────┐
        ▼                   │
  issues/open/  ──fix──▶ issues/ready-for-retest/
                              │
                  ┌───────────┴────────────┐
                  ▼                        ▼
          retest passes              retest fails
                  │                        │
                  ▼                        ▼
       issues/verified-fixed/      back to issues/open/  (append `## Retest <date>` section)

  issues/wont-fix/   ← terminal (user/eng explicitly accepts)
```

**Moving a file = `git mv` semantics: just use `mv`.** Don't duplicate. The folder it lives in is the source of truth for status — never edit a status field in two places that can disagree.

## Tracker rows

Each tracker is a markdown table. Columns:

| Test ID | Area | Test case | Steps (short) | Expected | Status | Issue | Last run | Notes |

- **Test ID** — `<area>-NNN`, e.g. `AUTH-001`, `SMOKE-014`.
- **Status** — see legend in [`README.md`](./README.md). Update inline as you test.
- **Issue** — link to issue file if `fail` or `retest`, else blank.
- **Last run** — ISO date you last touched this row.

When you change a row's status, also update **Last run**. Don't silently pass over rows.

## Detection oracles (every pass)

`pass` means "UI correct **and** no silent errors":

- **Console + network monitoring stay on during every pass.** Any console error, unhandled rejection, or non-2xx observed during any test is filed/noted immediately — never deferred to the cross-cutting pass.
- **Computed values get one independent spot-check** (raw API response, second screen showing the same fact, or value after hard refresh). Two screens disagreeing about the same number is always an issue.
- **Every create/edit form gets three mutation probes:** submit twice fast (double-write?), edit-then-re-edit (effects stack instead of replace?), refresh after mutate (persisted or optimistic-only?).
- **Reproduce twice before filing.** Non-reproducing second run → file anyway, tagged `intermittent` with observed frequency.

## Retest workflow (THE important loop)

Engineering signals "fixed" by:
1. Moving issue file to `issues/ready-for-retest/`.
2. Appending a `## Fix notes` section with PR/commit and a summary of what changed.
3. Flipping the tracker row to `retest`.

QA's first job each session is to clear `ready-for-retest/`:
- Re-run the **original repro** (not a different path).
- If the point retest passes, run a **class retest**: 1–2 sibling variants of the same bug class (same bug on the edit path if found on create; another role/entity; the adjacent boundary value). Fixes are routinely too narrow.
- If pass (point + class): `mv` the file to `verified-fixed/`, append a `## Retest <YYYY-MM-DD>: PASS` block (note which variants you checked), set the tracker row to `pass`, update **Last run**, and **add a permanent `REG-<issue number>` regression row** to the relevant tracker with the issue's repro as steps — fixed bugs are the most likely to recur.
- If fail: `mv` back to `open/`, append a `## Retest <YYYY-MM-DD>: FAIL` block describing the new failure (it may be different from the original — note that), set the tracker row back to `fail`.
- Do additional regression around the changed area (smoke the surrounding screens) — many "fixed" patches break something nearby.

## Filing UX gaps (no-bug, but worth raising)

Missing affordances (no empty state, no delete button, no validation message, no confirmation on destructive action) are real findings even though they aren't bugs in code that exists. File these as normal issues with severity S3 (or S2 if it materially blocks a workflow). Use the area tag `ux-gap` in the title prefix:

`ISSUE-0042-ux-gap-medications-list-no-empty-state.md`

These also get tracked in [`trackers/07-ux-gaps.md`](./trackers/07-ux-gaps.md).

## When to NOT file

- A test you couldn't run because of environment (server down, no seed data, account locked) — log in `session-log.md`, mark tracker row `blocked`, **do not** open an issue.
- Behavior you're unsure about — note it in the tracker row's **Notes** column with a `?`, and continue. If still unsure after the area is done, file as S4 with the question explicit.
- Duplicates — if symptom matches an existing open issue, add a `## Additional repro <date>` block to that issue instead of filing a new one. **Re-evaluate severity upward when you do** — frequency is a severity input; a second independent repro by a different role or path usually bumps an S2 to S1.
- Findings that only repro outside the project's declared design viewport — downgrade to S4 or label with the platform (`web-only`, `desktop-only`) in the title.

## Escalation and sign-off

- **S0/S1 stops the line.** File immediately (don't batch to session end). If it contaminates further results, halt the tracker walk, mark remaining rows `blocked` against it, and lead the session report with it.
- **`wont-fix` requires explicit user sign-off.** Only the user can authorize the move; record who, when, and why in the issue file.
- **Traceability is bidirectional.** The issue header names the Test ID(s) that found it; the tracker row's Issue column names the file. An issue with no Test ID means coverage has a hole — add the row.
