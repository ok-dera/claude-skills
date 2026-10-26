# claude-skills

Claude Code skills and commands I use to ship production software. Not prompt collections — each one encodes a
process with enforced invariants, so the model operates inside constraints instead of improvising.

Three things live here: a persistent QA harness (`qa-memory-bank`), a client-reporting pipeline
(`client-walkthrough`), and an orchestrator mode (`fable`).

---

## `qa-memory-bank` — QA that survives the session boundary

**The problem.** Agent-driven QA loses everything when the session ends. Next session re-tests what already
passed, forgets what was filed, and never re-checks the fix someone claimed to ship. Chat history is not a
tracking system.

**The design.** The filesystem *is* the state. An issue's status is the folder it sits in:

```
qa-memory-bank/
├── test-environment.md      project facts: URL, viewport, driver, accounts, roles, what can't be tested
├── conventions.md           house rules — IDs, severity, lifecycle
├── test-plan.md             pass strategy + coverage matrix
├── session-log.md           append-only, newest on top, ends with "Next session should:"
├── trackers/                one file per pass; rows are pending|in-progress|pass|fail|blocked|retest|n/a
├── issues/
│   ├── open/                 ← filed, unfixed
│   ├── ready-for-retest/     ← eng claims fixed; cleared FIRST next session
│   ├── verified-fixed/       ← retest passed, regression row added
│   └── wont-fix/             ← requires explicit human sign-off
└── screenshots/
```

There is no status field anywhere. Nothing can disagree with anything else, because there is only one place
the answer lives. Transitions are `mv`, never copy.

**The rules that make it work,** and the failure each one prevents:

| Rule | Prevents |
|---|---|
| Retest queue clears before any new testing | Fixes that were never actually verified |
| **Class retest** — after a fix passes, test 1–2 sibling variants of the same bug class | Fixes too narrow for the bug (fixed on create, still broken on edit) |
| Every verified fix leaves a permanent `REG-<id>` regression row | Fixed bugs silently recurring — the likeliest class of regression |
| `pass` = UI correct **AND** no console/network errors | A clean-looking screen sitting on a swallowed 500 |
| Data-integrity spot-check against an independent source | Two screens confidently disagreeing about the same number |
| Mutation probes on every form: submit-twice, edit-then-re-edit, refresh-after-mutate | Double writes, stacking updates, optimistic-only saves |
| Monotonic `ISSUE-NNNN` across all four folders, never reused | Collisions and dangling references |
| Never silently skip a row — `pass`, `fail`, `blocked`, or `n/a` with a reason | Untested work reported as tested |
| Exploratory charter in the last ~15% of every session | Process degrading into pure regression checking |
| Periodic coverage audit against the app's real surface | Passes "completing" while rows are quietly absent |
| S0/S1 stops the line; contaminated rows blocked immediately | Testing on top of broken auth or corrupted data |
| `wont-fix` needs explicit human sign-off | Issues buried on the agent's own judgment |

**Deliberately withheld from the agent:** starting dev servers (unless `/qa-run` explicitly delegates it),
destructive writes to shared resources, exploiting security findings beyond observation, and moving anything to
`wont-fix`. Those stay with the human.

Project-agnostic — everything project-specific lives in `test-environment.md`, and the templates scaffold a
fresh workspace on first run.

### Companion commands

- **`/qa`** — status check. Loads the memory bank, reports open issues and the retest queue.
- **`/qa-run`** — runs a pass. Retest queue first, then pending rows in tracker order. Also the explicit
  delegation that lets the agent start servers. Includes a parallel-agent partitioning scheme with hard
  ownership rules: one tracker per agent, one test account per agent, isolated browser sessions, and
  orchestrator-owned ID blocks pre-assigned so concurrent filing stays monotonic.
- **`/investigate`** — read-only triage. Clusters the reported issues, traces each one through every layer to
  `file:line` evidence, and reports one plain-English block per issue: status, severity, cause, a concrete
  example, the fix, and whether a human decision is needed. Applies no fixes.
- **`/qa-cycle`** — the closed loop: fix approved issues → test → retest → investigate → PR → code review,
  repeating on its own until a human decision is needed. Every fix starts from a failing reproducing test that
  may not be weakened afterwards. Auto-fixes are limited to verified issues outside high-stakes areas (money,
  auth, permissions, PII, migrations). Review rounds are capped at 3, scoped to what the PR introduced, and
  logged in a ledger, so a rejected finding stays visible and a failed review never counts as approval. Never
  merges.

---

## `client-walkthrough` — a PR the client can actually read

**The problem.** The people paying for the work can't read a diff. "What did we get this sprint?" deserves
better than a commit list, and hand-assembling a screenshot deck for every release doesn't scale.

**The design.** A seven-step pipeline from pull request to polished A4 PDF, screenshots taken from the real
running app:

1. **Read the PR** (`gh pr view` / `gh pr diff`) and split changes into client-visible vs. purely technical.
2. **Approval gate** — present the proposed section list and wait; nothing is shot or written until the human
   trims it. The agent decides nothing about what the client sees.
3. **Disposable demo database** — clone the dev DB (settings overlay points Django at the clone), then *curate
   by renaming*: QA fixture rows become plausible fictional people. Renaming instead of deleting matters —
   deletes cascade away the very rows that make dashboards worth photographing.
4. **Capture with playwright-cli** — fixed viewport, `NN-slug.png` ordering, per-role logins (public /
   customer / staff). Ships with the capture gotchas pre-learned: eval-scrolling silently resets (use
   `mousewheel`), element shots for forms on busy backgrounds, `label:has()` for sr-only toggles, cursor
   parked off-table so no hover state leaks into a frame, re-navigate past one-time toasts.
5. **Compress** PNG→JPG (`sips`, ~80 quality) so the PDF stays mailable.
6. **Render** — fill the bundled A4 print template (cover, TOC, numbered sections, figure/caption blocks,
   staff/public audience tags, before/after boxes), serve it over local HTTP (playwright-cli blocks `file://`),
   `playwright-cli pdf`.
7. **QA pass** over the rendered pages — no split figures, no junk data, no logged-in QA username in a header —
   then drop the demo database.

**The invariants:** every screenshot is the real app, never a mockup; sample data is declared as illustrative
on the cover; anything the walkthrough claims must be visible in a frame — if the shot contradicts the story
(a stale button label, a reverted fix), that's a finding to report back to engineering, not a thing to crop
around silently.

Comes with `references/writing-style.md` — the tone rules (outcome-first headings, banned jargon, caption
format) that keep the prose a client document instead of a changelog.

---

## `fable` — orchestrator mode

Flips the session's default: plan, supervise, audit, review — delegate implementation to sub-agents via specs,
and only touch code directly for trivial fixes. Useful when the work is broad enough that the expensive
resource is judgment rather than typing.

---

## Install

Clone anywhere, then symlink into your Claude Code config:

```bash
git clone https://github.com/ok-dera/claude-skills.git
ln -s "$PWD/claude-skills/skills/qa-memory-bank"     ~/.claude/skills/qa-memory-bank
ln -s "$PWD/claude-skills/skills/client-walkthrough" ~/.claude/skills/client-walkthrough
ln -s "$PWD/claude-skills/commands/qa.md"      ~/.claude/commands/qa.md
ln -s "$PWD/claude-skills/commands/qa-run.md"  ~/.claude/commands/qa-run.md
ln -s "$PWD/claude-skills/commands/fable.md"   ~/.claude/commands/fable.md
ln -s "$PWD/claude-skills/commands/investigate.md" ~/.claude/commands/investigate.md
ln -s "$PWD/claude-skills/commands/qa-cycle.md"    ~/.claude/commands/qa-cycle.md
```

`/qa-cycle` calls `/investigate`, `/qa-run` and Claude Code's built-in `/code-review`. Its fix-mode test guard
hook isn't included here; without it, the command applies the same rule by instruction.

Then `/qa` in any project. With no `qa-memory-bank/` present it offers to scaffold one and asks for the facts
it needs; it won't guess URLs or credentials.

`qa-memory-bank` is driver-agnostic but expects a browser driver declared in `test-environment.md` —
[playwright-cli](https://github.com/microsoft/playwright) in most of my projects. If the declared driver isn't
available it blocks the affected rows rather than silently substituting another.

## License

MIT — see [LICENSE](LICENSE).
