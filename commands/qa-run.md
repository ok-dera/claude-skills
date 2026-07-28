---
description: Run a QA test pass — retest queue first, strictly per the qa-memory-bank instructions
---

Proceed with QA testing using the `qa-memory-bank` workspace (and the `qa-memory-bank` skill).

**Setup:**
- Locate the `qa-memory-bank` folder(s). If more than one exists, ask me which environment to focus on first.
- Run the full orientation flow from the QA instructions: read `test-environment.md`, `conventions.md`, and the recent `session-log.md` entries before any test action.

**Then:**
1. List the **pending passes** per tracker (each tracker with its count of `pending`/`blocked` rows).
2. Check `issues/ready-for-retest/`. If retests exist, **prioritize them**: ask me if I'd like to clear the retest queue first. If yes, run the retests exactly per the QA instructions (original repro + class retest, move files between issue folders, flip tracker rows, add regression rows).
3. If the queue is empty (or after it's cleared and I say continue), proceed with regular pending passes in tracker order.

**How to test — non-negotiable:**
- Follow the QA workspace's instructions for *how* tests are carried out. If it declares a driver (e.g. `/playwright-cli`, Maestro), use that — do not improvise your own testing method.
- Issue lifecycle (filing, reopening failed retests, marking passed ones verified) follows the QA conventions exactly — folder = status, move don't copy.

**Servers:** this command is your explicit delegation to start whatever server environments the tests need (backend, frontend, etc.) on available ports. Check what's already running first; if a port conflict or credentials question comes up, ask me.

**Parallel agents (use when the work partitions cleanly):**
When multiple pending passes or multiple retests are independent — different trackers, different roles, different surfaces — fan them out to concurrent agents instead of running them serially. Only parallelize work that can't collide; when in doubt, keep it serial.

Partitioning rules (all mandatory):
1. **One tracker = one agent, exclusive.** An agent may edit only its own tracker file. Never give two agents rows in the same tracker.
2. **One test account/role per agent.** Two agents logged in as the same account can invalidate each other's sessions and mutate the same records. If two passes need the same account, they run in the same agent (or serially).
3. **Each agent gets its own browser session/profile.** No shared driver state.
4. **Retests batch by surface, not one-per-agent.** Group `ready-for-retest/` issues by area/role; issues touching the same records or the same fix go to the same agent (the class retest needs that shared context).
5. **Issue numbering is orchestrator-owned.** Scan the max ISSUE-NNNN once, then pre-assign each agent a reserved ID block (e.g. agent A: 1570–1579, agent B: 1580–1589). Agents file issue files using only their block; unused IDs in a block are simply skipped — numbering stays monotonic, gaps are fine.
6. **Session log has one writer.** Agents return a structured report (rows flipped, issues filed/moved, retest verdicts, escalations, metrics); the orchestrator writes the single session-log entry after merging.
7. **File moves stay with the owner.** An agent moves only the `ready-for-retest/` files it was assigned. Folder = status still holds — move, don't copy.

Cross-agent escalation: if any agent reports an S0/S1 that contaminates shared state (auth broken, data corrupted, server erroring), stop dispatching further agents, let running ones finish or note their results as suspect, and lead the report with it.

Mutation caution still applies per hard rule 4: two agents concurrently mutating a shared dev DB is fine only when they operate on disjoint accounts/records. Anything bulk or destructive still requires asking me first.

$ARGUMENTS
