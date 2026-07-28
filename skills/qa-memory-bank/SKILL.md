---
name: qa-memory-bank
description: Run structured, persistent QA sessions against a web/mobile app using a `qa-memory-bank/` workspace that survives across Claude sessions. Triggers on "run qa", "continue qa", "qa session", "qa pass", "smoke test", "retest", "file a bug", "qa memory bank", "set up qa workspace", or whenever the user has a `qa-memory-bank/` folder in the project root. Enforces a folder-as-status issue lifecycle (open → ready-for-retest → verified-fixed | wont-fix), monotonic ISSUE-NNNN numbering, tracker-driven test passes, retest-first session ordering, and an append-only session log so future sessions can pick up without losing context. Project-agnostic — reads project-specific config (URL, viewport, accounts, roles, browser driver) from `qa-memory-bank/test-environment.md`.
---

# qa-memory-bank

A persistent, file-based QA workspace that survives session boundaries. The folder layout *is* the state — no out-of-band tracking. Every project gets the same bones; project-specific facts live in `test-environment.md`.

## When to use

- User says: "run qa", "continue qa", "qa pass", "qa session", "smoke test the app", "retest", "file a bug for <feature>", "set up qa workspace", "qa memory bank".
- The current project has a `qa-memory-bank/` folder — assume the user wants you to operate inside it.
- User describes wanting structured, repeatable QA across sessions (not a one-shot test).

Don't use this skill for: one-off test-plan generation (use `qa-plan`), code-level test writing (pytest/jest), security pentesting beyond surface-level observation, or load testing.

## Two modes

### Mode A — Bootstrap (no `qa-memory-bank/` in project)

Scaffold the workspace from the templates next to this skill (`templates/` directory).

1. Confirm with the user before creating files. Ask: "I don't see a `qa-memory-bank/` here. Set one up at `<project>/qa-memory-bank/`? I'll need a few facts: app URL, viewport target, browser driver, test accounts, and the roles in scope."
2. Copy `templates/*` recursively into `<project>/qa-memory-bank/`.
3. Edit `test-environment.md` to fill in the project-specific blanks the user gave you. Leave a `TODO` marker on anything they couldn't answer — don't guess URLs, credentials, or compliance posture.
4. Edit `test-plan.md` to list the actual screens/areas in scope (you may need to scan the codebase). Keep the layered-passes structure as-is.
5. Create empty trackers for each pass under `trackers/`. The defaults (`00-smoke`, `01-auth`, then per-role, then `06-security` / `07-ux-gaps` / `08-cross-cutting`) are conventions, not commandments — adapt to the project's actual roles.
6. Append a kickoff entry to `session-log.md`. Stop and hand off — don't start testing in the same turn you scaffolded.
7. **Show the user the companion slash commands** (see "Companion slash commands" below) so they know how to drive future sessions.

### Mode B — Resume (workspace exists)

This is the common case. Follow the orientation flow on every session before any test action.

## Companion slash commands

Remind the user of these after bootstrap (and whenever helpful). They live in `~/.claude/commands/`:

- **`/qa`** — status check: loads the memory bank and reports open issues and issues marked for retest. If multiple `qa-memory-bank/` folders exist, it asks which environment to focus on.
- **`/qa-run`** — proceed with testing: lists pending passes per tracker, prioritizes the retest queue (asks the user whether to clear it first), then runs tests strictly per this workspace's instructions (declared driver, issue lifecycle). Running it explicitly delegates server startup to the agent (see hard rule 3).
- **`/investigate <issues or screenshot>`** — verify reported issues against the code, explain valid ones in simple terms with recommended fixes; no fixes applied until the user approves.

## Orientation flow (Mode B — every session, in order)

1. **Read project facts** — `qa-memory-bank/test-environment.md` (URL, viewport, driver, accounts, what's in/out of scope, what cannot be tested via the available driver).
2. **Read conventions** — `qa-memory-bank/conventions.md` (issue IDs, severity scale, file naming, retest workflow, when NOT to file). These are the project's house rules; treat them as binding even if they differ slightly from this skill's defaults.
3. **Read recent context** — last 1–2 entries (top of file) of `qa-memory-bank/session-log.md`. The "Next session should:" line at the bottom of the most recent entry is your default starting point unless the user overrides.
4. **Check retest queue** — `ls qa-memory-bank/issues/ready-for-retest/`. **If non-empty, clear it before any new testing** (see Retest workflow). This is the single most-skipped step; do not skip it.
5. **Triage the open backlog** — `ls qa-memory-bank/issues/open/`, sorted by severity then filed date. Any S0/S1 open longer than 7 days gets surfaced in this session's summary to the user — the backlog must not silently age.
6. **Change intake** — if the user mentions new features, a release, or recent merges, add tracker rows for the changed surface *before* testing (use `qa-plan` on the diff if useful). New code is the highest-defect-density surface; walking only the stale tracker systematically misses the newest bugs.
7. **Pre-flight the env** — confirm the app URL is reachable. If it's not, mark the relevant tracker rows `blocked`, log it, and stop. **Never start the dev server yourself** — that's the user's responsibility unless they explicitly delegated it.
8. **Pick up the tracker** — open the relevant tracker file under `qa-memory-bank/trackers/` and resume from the first row whose **Status** is `pending` or `blocked` (re-checking blockers in case the precondition has cleared).

## Detection oracles (apply during every pass)

A row's `pass` means "UI correct **and** no silent errors" — not just "the screen looked right." Most escaped incidents are multi-oracle failures: the UI renders fine while the network or data layer is wrong. Run all of these continuously, in every pass, not just the cross-cutting one:

1. **Always-on console + network monitoring.** Keep both open from the first action. Any console error, unhandled rejection, or non-2xx response observed during *any* test — even one whose UI outcome passed — is filed (or noted in the row) immediately. Do not defer to the `08-cross-cutting` pass.
2. **Data-integrity spot-check.** When a screen displays computed or aggregated values (totals, counts, prices, balances), verify at least one against an independent source: the raw API response, a second screen showing the same fact, or the value after a hard refresh. Two screens disagreeing about the same number is always an issue, even if each looks plausible alone.
3. **Mutation-side checks per form.** Beyond the render/validation checklist, every create/edit form gets three cheap probes: **submit twice fast** (double-tap — does it write twice?), **edit-then-re-edit** (does the second save stack effects instead of replacing them?), and **refresh after mutate** (did the write persist, or was it optimistic-only?).
4. **Reproduce twice before filing.** Re-run a failure once before opening the issue. If it doesn't repro the second time, file it anyway tagged `intermittent` with the observed frequency — intermittent is often the worst class (races, sync queues), never ignorable.

## Retest workflow (the most important loop)

When `issues/ready-for-retest/` has files, those run first. The retest signal is cross-session; you cannot rely on memory or chat.

For each file in `ready-for-retest/`:

1. Re-run the **original repro** from the issue file, not a paraphrase or shortcut.
2. **PASS — but don't stop at the point retest.** Fixes are routinely too narrow, so run a **class retest**: 1–2 sibling variants of the same bug class (same bug on the edit path if found on create; same bug for another role/entity; the boundary value next to the one that failed). Only if the variants also pass:
   - `mv qa-memory-bank/issues/ready-for-retest/<file> qa-memory-bank/issues/verified-fixed/`
   - Append a `## Retest <YYYY-MM-DD>: PASS` block to the issue file (what you did, observation, which class variants you checked, regression spot-check around the area).
   - Flip the matching tracker row's status from `retest` → `pass`, update **Last run** to today.
   - **Add a permanent regression row**: append a row to the relevant tracker with Test ID `REG-<issue number>` whose steps are the issue's repro. Fixed bugs are the most likely to recur; every future pass re-runs them.
3. **FAIL:**
   - `mv qa-memory-bank/issues/ready-for-retest/<file> qa-memory-bank/issues/open/`
   - Append a `## Retest <YYYY-MM-DD>: FAIL` block describing how it still fails (and note explicitly if the failure mode has changed — that's a separate diagnostic signal).
   - Flip the tracker row from `retest` → `fail`.
4. After clearing the queue, do a quick regression sweep around the changed surface area — fixes often break something nearby.

**Hard rule**: the folder a file lives in *is* its status. Never duplicate status in a frontmatter field, the tracker row, and the folder name and let them disagree. Use `mv`, not copy.

## Test passes

Default ordering (override only if the user says so or `test-plan.md` documents otherwise):

1. **Smoke** (`00-smoke.md`) — every role logs in, hits each top-level screen, look for blockers. Goal: confirm the app is testable and find S0/S1 fast.
2. **Auth & onboarding** (`01-auth.md`) — register, login, verify, MFA, password reset, session/token, profile completion. Happy + negative paths.
3. **Per-role functional sweeps** — one tracker per role (`02-<role>.md`, `03-<role>.md`, …). Walk every screen × every CRUD path × every form. For each screen check: render / empty state / loading state / error state / primary action / secondary actions / back nav / refresh / validation.
4. **Security** (`06-security.md`) — IDOR (user A → user B's data via ID swap), cross-role access, session/token expiry, auth bypass on critical endpoints, sensitive-data display in DOM/console. Stay observational — do not weaponize.
5. **UX gaps** (`07-ux-gaps.md`) — focused walk asking "what's *missing*?": empty states, confirmations, undo, sort/filter/search/pagination, edit/delete affordances, breadcrumbs.
6. **Cross-cutting** (`08-cross-cutting.md`) — silent console errors, swallowed network 4xx/5xx, perf smell tests, accessibility smoke (tab order, focus rings, alt text, label association), responsive spot-checks, copy/i18n inconsistencies.

These are *cheap multi-pass*, not exhaustive single-pass. Don't try to be exhaustive in one tracker before moving on.

### Exploratory charter (every session)

Scripted trackers only catch what someone thought to write down. Reserve the last ~15% of each session for a one-line exploratory charter — e.g. "try weird inputs on X", "interleave role A and role B on the same record", "do the flow backwards". Record the charter and its findings in the session log. This is where unknown-unknowns get caught; skipping it turns the process into pure regression checking.

### Coverage audit (periodic)

A pass has exit criteria, and silence is not one of them. Every few sessions (or when a pass claims completion), cross-check tracker rows against `test-plan.md` scope and the app's *actual* surface (nav walk or route list from the codebase). Any screen × role × CRUD combo with zero rows gets a `pending` row added. A pass is "complete" only when every row is `pass`, `n/a`, or linked to an issue — never while rows are silently absent.

## Tracker rows

Each tracker is a markdown table. Columns:

| Test ID | Area | Test case | Steps (short) | Expected | Status | Issue | Last run | Notes |

**Test ID** format: `<AREA>-NNN`, e.g. `AUTH-001`, `SMOKE-014`.

**Status legend** (these go in the cell verbatim):

- `pending` — not yet attempted.
- `in-progress` — only set during an active session you're inside right now.
- `pass` — works as expected.
- `fail` — bug; the **Issue** column links to a file in `issues/open/`.
- `blocked` — cannot test (precondition missing, server down, role unavailable). Note why in **Notes**.
- `retest` — was failed, eng claims fixed; issue is in `issues/ready-for-retest/`.
- `n/a` — not applicable in current build (note why in **Notes**, e.g. "camera flow — driver cannot exercise").

Whenever you change a row's status, also update **Last run** to today's date (ISO). Never silently skip rows — set them to `blocked` or `n/a` with a reason.

## Filing an issue

### Numbering

Allocate the next ID by scanning all four issue folders for the current max:

```
ls qa-memory-bank/issues/{open,ready-for-retest,verified-fixed,wont-fix} 2>/dev/null \
  | grep -oE 'ISSUE-[0-9]+' | sort -u | tail -1
```

Increment, zero-pad to 4 digits. Numbering is monotonic across all categories — never reuse, never reset.

### File naming

`ISSUE-NNNN-<short-kebab-slug>.md` — slug describes the symptom in 3-6 words. Examples:
- `ISSUE-0007-login-rejects-valid-mfa-code.md`
- `ISSUE-0042-ux-gap-medications-list-no-empty-state.md` (UX gaps prefix the slug with `ux-gap`)

### File contents

Copy `qa-memory-bank/issues/ISSUE-TEMPLATE.md` verbatim and fill it in. Required sections: header (ID/severity/role/area/status/dates), Summary, Steps to reproduce, Expected, Actual, Evidence, Environment, Notes.

**Traceability is bidirectional and mandatory**: the issue header names the Test ID(s) that found it (`Found by: SMOKE-014`), and the tracker row's **Issue** column names the issue file. An issue with no Test ID means coverage has a hole — add the row.

### Escalation — stop the line on S0/S1

An S0/S1 is not filed like an S3 and walked past. On finding one:

1. File it immediately (don't batch to end of session).
2. If it contaminates further results (auth broken, data corrupted, server erroring), halt the current tracker walk and mark remaining rows `blocked` against the new issue.
3. Lead the end-of-session report with it, and fill the `**Escalations:**` line in the session-log entry.

### Duplicate repros escalate severity

When appending an `## Additional repro` block to an existing issue, re-evaluate its severity *upward* — frequency is a severity input. A second independent repro of an S2 by a different role or path is usually an S1.

### Severity scale

| Level | Use when |
|---|---|
| **S0 — Blocker** | App unusable for a role, primary nav broken, data loss, security breach (auth bypass, sensitive-data leak). |
| **S1 — Critical** | Core feature broken, cannot create/read/update primary entity, unhandled error on a major flow. |
| **S2 — Major** | Significant degradation: secondary feature broken, data shown wrong, validation missing on important field, page >5s on critical path. |
| **S3 — Minor** | Cosmetic, layout, copy, low-impact UX gap. |
| **S4 — Trivial** | Typo, polish, nice-to-have suggestion. |

When unsure, pick higher — easier to downgrade than miss.

### Evidence

- Screenshots go to `qa-memory-bank/screenshots/ISSUE-NNNN-<slug>.png`. Ambient evidence not tied to a bug: prefix `evidence-` and date.
- Console / network excerpts: paste only the relevant lines plus a few of context. Do not dump full logs.

## When NOT to file

- **Env outage** (server down, URL unreachable) — log in `session-log.md`, mark rows `blocked`, do not open an issue.
- **Missing seed data / locked accounts** — same: log + block, no issue.
- **Behavior you're unsure about** — note in the tracker row's **Notes** with a `?`, continue. If still unsure after the area is done, file as S4 with the question explicit.
- **Duplicates** — if symptoms match an existing open issue, append `## Additional repro <YYYY-MM-DD>` to that issue. Do not file a new one.
- **Findings that don't reproduce on the design viewport** — if the project's `test-environment.md` declares a primary viewport (e.g. mobile 390×844) and a layout finding only repros at desktop sizes, downgrade to S4 or label `<platform>-only` in the title. Read `test-environment.md` for what counts as the design target.

## Mobile/web/native caveat

If `test-environment.md` says the app is a mobile app being tested via a web build (or any cross-platform hybrid), respect what the driver can and cannot exercise. Common can't-test items: camera, biometrics, push notifications, native share/photo picker, OS permission prompts, background tasks. Mark these `n/a` with a note; do not invent mobile bugs from web symptoms (DOM warnings, browser-only CSS, URL routing). Do file underlying causes that are real on both surfaces (misclassified API errors, unnecessary network calls).

## Browser driver

Use whatever the project declares in `test-environment.md`. The skill is driver-agnostic, but most projects use Playwright (via the `playwright-cli` skill or similar). On every fresh launch:

1. Resize to the project's declared design viewport before doing anything else.
2. Log in with the seeded account for the role under test.
3. Capture console + network logs throughout — paste relevant excerpts when filing.

If the declared driver isn't available in the current environment, mark relevant rows `blocked` and log it. Do not silently substitute.

## Session log

`qa-memory-bank/session-log.md` is append-only, **newest entry on top** (reverse chronological). One entry per session. Write the entry at session end *before* you run out of context budget — future-you depends on it.

Template:

```
## YYYY-MM-DD — <short pass name or session id>

**Pass:** <which tracker(s) you worked on>
**Started from:** <last-run row in tracker, or "retest queue">
**Tested:** <bullet summary>
**Filed:** <issue IDs with one-line title>
**Retests cleared:** <issue IDs verified-fixed | none>
**Escalations:** <S0/S1 found this session, or aged >7 days in open/ | none>
**Exploratory charter:** <one-line charter + findings | skipped (why)>
**Metrics:** <per tracker touched: N pass / N fail / N pending / N blocked>
**Blockers:** <if any>
**Next session should:** <one concrete next step — make this load-bearing>
```

Keep entries short — the trackers and issues hold the detail. The "Next session should" line is the handoff; be specific (a tracker row to start at, a retest to verify, an environment fix to confirm).

## Hard rules

These are non-negotiable across projects:

1. **Folder = status.** Never edit a status field in two places. Move files; don't copy.
2. **Retest queue first.** Always clear `issues/ready-for-retest/` before new testing.
3. **Don't start the dev server** — unless explicitly delegated. The user invoking `/qa-run` counts as explicit delegation to start needed servers on available ports; otherwise the user owns server lifecycle. If unreachable and not delegated, block and stop.
4. **No destructive shared-resource writes** without explicit user permission. Bulk-creating data in a shared dev DB, deleting test users, etc. — ask first. UI-driven side-effects (a check-in click that writes one row) are fine but worth noting in the session log.
5. **Reproduce on the design viewport before filing.** If the project declares one, layout findings outside it get downgraded.
6. **Don't weaponize security findings.** Observe, file, stop. Note the vector; don't exploit it.
7. **Never silently skip a tracker row.** Set `pass`, `fail`, `blocked`, `n/a`, or leave it for the next session — but don't pretend you tested something you didn't.
8. **`pass` means UI correct AND no silent errors.** Console + network monitoring stay on during every pass; a clean screen over a swallowed 500 is a `fail`.
9. **`wont-fix` requires explicit user sign-off.** Only the user can authorize the move, and the file must record who, when, and why. Never bury an issue on your own judgment.
10. **S0/S1 stops the line.** File immediately, block contaminated rows, lead the session report with it.

## Templates

Reference templates ship next to this skill in `templates/`. On bootstrap, copy them in. On resume, the project's existing copies are authoritative — do not overwrite even if they've drifted from the templates here.

- `templates/README.md` — workspace overview (what every session reads first)
- `templates/test-plan.md` — pass strategy + coverage matrix skeleton
- `templates/test-environment.md` — project-specific facts (the file you'll customize most)
- `templates/conventions.md` — IDs, severity, lifecycle, retest rules
- `templates/session-log.md` — empty log with the entry template
- `templates/issues/README.md` — issue folder workflow
- `templates/issues/ISSUE-TEMPLATE.md` — bug report template
- `templates/trackers/00-smoke.md` — smoke tracker skeleton
- (Other tracker files are project-specific — scaffold them as empty tables during bootstrap.)
