---
description: Investigate reported issues, verify validity, explain in simple terms with recommended fixes — no fixes until approved
---

Investigate the issue(s) described below (or in the attached screenshot/image).

Operate as orchestrator/architect: delegate the digging to agents (via the Agent tool) and audit what they find.

## Batching — cluster before you spawn

Before spawning any agents, read all reported issues and group the ones that likely touch the same surface, files, or flow (same page, same endpoint, same model, same feature area). **One agent per cluster, not one agent per issue** — isolated agents re-reading the same files wastes tokens. Only give an issue its own agent when it genuinely shares nothing with the others. Tell each agent every issue in its cluster up front so it reads the shared code once.

## Model selection — Sonnet gathers, the orchestrator decides

- **Default worker: `model: sonnet`.** Investigator agents do legwork — find files, trace the chain, pin `file:line` evidence, draft a verdict. Sonnet 5 handles that; it's ~5x cheaper than Opus/Fable and has the same 1M context.
- **The orchestrator (you — Opus or Fable, whichever this session runs) owns the verdict.** Treat the agent's *evidence* as data and its *verdict* as a draft. Re-derive real/not-real yourself from the cited lines. Sonnet's failure mode is closing an issue confidently after the first plausible explanation — not fetching the wrong files.
- **Required in every agent report:** (1) the raw grep output for callers/consumers of any shared function the root cause touches — so sibling coverage is auditable, not asserted; (2) a closing "Layers I did not read" line (write "None" if all were read) — so an omitted layer is an explicit claim, not a silent gap. Put both requirements in every agent prompt.
- **Escalation tell:** an agent returns a verdict without `file:line` evidence at every layer it was told to traverse, or omits either required item above. Re-spawn that cluster with `model: opus` (or omit `model` to inherit the session model) — don't argue with the report.
- **Skip delegation entirely** for high-stakes clusters — payments/Stripe, consent/COPPA/age transitions, the matching algorithm, anything cross-module where a missed sibling caller costs real money. Traverse those yourself at the orchestrator's model.

## Investigation standard — no stone unturned

Each investigation must be exhaustive *within the scope of its issues*:

- **Traverse the full chain** relevant to the issue before reaching a verdict: frontend component → API function → TS types → DRF view/serializer → model/DB → and any Celery task, webhook, signal, or cache layer in the path. Read the actual code at every layer touched by the issue; do not stop at the first plausible explanation.
- **Never assume.** If you are not sure about a field, a contract, a behavior, or a config value — open the file, run the query, or curl the endpoint. Anything unverified must be labeled as unverified in the report, never stated as fact. Banned: "probably", "likely", "should be", "typically" as load-bearing reasoning.
- **Check the siblings.** If the root cause lives in shared code, identify every other caller/consumer affected — the verdict must say whether the bug is isolated or systemic.
- **Verdict gate:** an issue only gets a verdict (valid / invalid) once the full related code path has been read and the root cause (or the proof of non-existence) is pinned to specific `file:line` evidence. If evidence is incomplete, say what's missing instead of guessing.

## Report format — plain English, one block per issue

The reader is the product owner, not an engineer. Do the exhaustive `file:line` traversal above, but **present** the outcome in this fixed shape, one block per issue, no code dumps. Keep `file:line` references out of the main block — put them in a single collapsed/footnote line at the end of each block if they are needed at all.

```
## ISSUE-NNNN — <what it is, in one line a non-engineer understands>
**Status:** ✅ Valid | ❌ Not valid (why: not a bug / already fixed in <commit> / by design)
**Severity:** S0–S4 — one clause on why it earns that level (the harm, in plain words)
**Cause:** 1–3 sentences. Root cause, not symptom. No jargon, no file paths. "The system decides X based on Y, but it should be based on Z."
**Example:** A concrete story with a named role: "Coach A does X. Coach B does Y. Result: Z happens and nobody is told."
**Fix:** 1–3 sentences. What changes, in which one place, and whether it fixes just this path or every sibling path at once.
**Your decision:** None — <one clause on why the fix is clear> | ✅ Required — <the exact question, the options, and your recommendation>
```

Rules for the block:
- Every field present, every time, in that order. If an issue is already fixed, still fill Cause (past tense) and Fix ("none needed — move the ticket").
- **Cause** and **Example** are the two fields the reader actually uses to judge severity. Spend the words there.
- **Your decision** is the only place a question goes. If the answer would not change the fix, write "None". Never bury a product question inside Fix.
- Anything you could **not** verify goes in one line at the end of the block: "Not verified: …". Never state an unverified claim as fact anywhere in the block.
- Side findings (bugs you found that were not in the ask) get their own block at the end, marked **Side finding (not filed)**, and end with "Want me to file it?".

After all blocks, close with a 4-column table — Issue | Status | Severity | Next step — and one sentence stating you are waiting for the go-ahead.

**Do NOT apply any fixes.** Report your findings and wait — I'll tell you whether or not to proceed with the fixes.

$ARGUMENTS
