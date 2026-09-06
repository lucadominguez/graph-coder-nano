---
name: graph-coder-nano
description: Use when a software change should be planned once and then implemented by parallel subagents under a single review gate. One markdown plan file, no tooling, no state store.
---
# Graph Coder Nano

Write a complete plan, dispatch bounded units to lower-cost workers, and have
each worker's manager review the result before dependent work starts.

This file describes the workflow. It has no mechanical checks, so the Director
and managers must apply the rules and record the evidence themselves.

Nothing here needs a CLI, a state file, or an installed package. `PLAN.md` is the
graph, the ledger, and the status board, and you keep it current by editing it.
If you want the same method with a checker that mechanically refuses a bad plan,
use Graph Coder Lite instead.

```text
1. PLAN      ground yourself in the code, then write PLAN.md
2. APPROVE   render it in full, get a yes, freeze the contracts
3. EXECUTE   dispatch, review, escalate, finish
```

## Role boundaries

| Role | May do | May never do |
| --- | --- | --- |
| Director (you, the root session) | Plan, spawn every worker, route, advise, review, record | Edit implementation files during EXECUTE; implement a unit instead of spawning it |
| Manager | Advise its children, supply bounded context, review submissions, escalate | Edit files; run a repair itself; complete work without evidence |
| Worker | Implement its unit, run its commands, submit a report | Read or write outside its scope; review its own work |

A manager review is a verdict, not a task. There are no reviewer agents and no
review units: a worker's own manager owns its review, and that is the only review
in the system.

The Director never becomes the implementer of last resort. When a branch cannot
proceed after bounded advice, retry, and fallback, it is `human_required`.

## 1. PLAN

Ground first, in one pass:

- Read the code the change will touch: entry points, interfaces, tests,
  conventions.
- Run the build, lint, typecheck, and tests, and **record the pre-existing
  failure baseline**. Without it you cannot tell a new failure from an old one.
- Settle what only the user can decide: intent, scope, non-goals, anything
  irreversible. Inspect before asking, and ask only what the repository cannot
  answer.

Facts cite a file, symbol, or command result. Anything else is an assumption, and
an assumption becomes a bounded exploration unit or a question, never a plan.

Then write one `PLAN.md`. Never a second plan, a summary plan, or a parallel
contract document. If it is long, it is long.

```markdown
# <change>

Goal, non-goals, and what done means.

Baseline: <the failing tests that were already failing>
Never touch: .env, .git/, secrets, anything not named in a write scope
Budget: <director tokens> / <worker tokens total> / control plane under <n>%
        protected: <provider whose weekly quota is the scarce resource>
Attempts: 2 per unit, then escalate
Managers: M-STORAGE owns schema and persistence; M-API owns the HTTP surface

## IU-STORE

state: pending
objective: <one measurable outcome>
dependencies: [IU-MIGRATION]
read_scope: [src/store/tokens.py, migrations/002_token_revocation.sql]
write_scope: [src/store/tokens.py, tests/test_token_store.py]
procedure:
  - <step>
  - <step>
commands:
  red: pytest tests/test_token_store.py     # fails before the work
  green: pytest tests/test_token_store.py   # passes after it
artifacts: [src/store/tokens.py, tests/test_token_store.py]
output_contract:
  - revoke_token and is_revoked are importable from src/store/tokens.py.
  - A second revoke on the same token leaves exactly one audit row.
  - The audit rows read back through is_revoked, not only through raw SQL.
progress: writes incrementally, a checkpoint per function, no command over 180s
route: <model> / fallback <model>
manager: M-STORAGE
stop_conditions:
  - The migration did not create the columns this unit was told to use.
```

Twelve contract fields plus `state`. A unit missing any of them is not ready.

Write every unit so a **fresh agent with no chat history** can execute it from
that block alone. That bar is the entire reason planning gets the expensive
model: detailed contracts are what make cheap workers viable.

### Output, progress and routing

- **`output_contract` is a gate, not a description.** `artifacts` names the file;
  the contract says what has to be inside it, as assertions someone else can
  check without trusting the worker. A unit that says "scrape the listings and
  submit a report" is satisfied by a scraper that returns nothing: the code ran,
  the file exists, and no criterion described the contents. Prefer an assertion a
  command can decide, and give every contract the minimum below.
- **`progress` defines observable checkpoints.** Not every harness exposes a
  running worker's transcript. Specify what progress looks like on disk, and use
  live status or transcripts when available. An agent 900 items into a long job
  and an agent wedged in a dead loop look the same on disk if neither has written
  a checkpoint. Say the checkpoint
  cadence in the unit's own terms, say whether output accumulates or lands at the
  end, and bound every command in seconds. A worker inside an unbounded blocking
  call cannot report, cannot be told from a hung one, and quietly breaks the whole
  review pattern.
- **`route` is a real model, never a placeholder.** Dispatching a unit whose
  route says `local` or `default` runs the graph on whatever the harness happens
  to supply, which is not the run that was approved.

### Output-contract minimums

Every contract carries a minimum, so a unit whose artifact is empty or ambiguous
**fails loudly instead of passing quietly**. Existence is the weakest evidence
there is: a file with the right name and a plausible size proves that something
ran, not that anything usable came out.

An ingestion artifact can look valid and still fail when its consumer loads it.
Include the consumer's expected schema and a real load check in the contract,
so the failure is found before the next stage depends on the artifact.

A minimum has three parts, and a contract that omits any of them is a
description again:

```text
floor      the least an acceptable result contains: rows, records, symbols,
           test cases, and where useful an upper bound too
shape      the keys, columns, or signatures the next stage requires by name
load       the consumer itself accepts it: import it, parse it, open it,
           run it through the reader that will read it in production
```

The strongest minimum is the consumer. If the artifact exists to be loaded, the
unit's green command loads it, and the contract asserts on what came back.

**Ambiguous counts as empty.** If two readers can disagree about whether an
artifact satisfies a line of the contract, that line has no minimum, and the
review has no reliable basis for a verdict. Write
each line so a command can decide it, and a unit whose minimum cannot be
expressed that way is one to stop on, not to dispatch and hope about.

### Splitting, scopes, and managers

Split where a unit can be independently owned and verified. Avoid oversized units
carrying hidden context, and avoid coordination-heavy oversplitting.

**Two units that can run at the same time must not write the same file.** Check
it yourself against the dependency graph, and check it including parent
directories and case: on Windows `SRC/Store.py` and `src/store.py` are one file.
Two agents in one write scope is the corruption the whole structure exists to
prevent.

One manager per meaningful branch or failure domain, never one per worker. Every
unit names one, and that manager owns its review.

### Route cheapest-that-passes, not cheapest

```text
Director   the frontier model, pinned, never silently downgraded
Manager    capable enough to review and advise across its whole branch
Worker     the cheapest model that will pass first time
```

A worker that fails twice and escalates costs more than a capable one that passes
once: a failed attempt pays for its context twice, its output twice, and a review
it did not need.

### Record usage and stop at the budget

Write the numbers in the plan header and stop when they are hit.

Nano does not collect usage or stop workers automatically. Take usage from the
harness, keep a running total, and stop new dispatches at the agreed limits. If
usage is unavailable, disclose the gap rather than reporting the run as within
budget.

- **Dollars are not the scarce resource.** A subscription route has no marginal
  dollar price, which is exactly why a router that scores dollars spends it
  freely. A weekly quota is finite, and running out costs the user their week
  rather than a few cents. Name that provider as protected and keep it off worker
  routes: workers are the many, and the many are what exhaust an allowance.
- **The control plane is overhead, not work.** Directing, reviewing, and
  monitoring produce no artifact. Past the share you wrote down, the run has
  stopped being worth its own supervision.
- **Keep a running total in the plan as you go.** A run that cannot measure its
  spend cannot be stopped by any number, however carefully chosen.

At a breach, stop dispatching and take one of three paths with the user: simplify
what is left, raise the budget deliberately, or stop here and finish by hand.
Raising the number yourself to clear the stop is the failure this exists to catch.

## 2. APPROVE

Render the complete plan: every unit with its full contract, the routes, the
budget and what it rests on, and every unresolved risk. **A summary is not an
approval view.** If the plan is long, say so and render it anyway.

Approval binds to the unit contracts, not to the prose. Rewording is free.
Changing a scope, a command, an output contract, or a route voids it, and you go
back to the user before dispatching.

## 3. EXECUTE

**Execution means spawning subagents.** Every unit runs inside its own freshly
spawned agent. Not a section of your reply. Not a file you edit because by now
you know what the code should say. If this phase ends and you never called your
harness's subagent tool, the run failed, however good the diff looks.

You spawn, route, review, advise, and record. **You do not implement.** If you
are about to open an implementation file during this phase, stop: you have
skipped dispatch.

If the harness has no subagent tool, it cannot run this method. Say so and stop.
Do not silently degrade into implementing the plan yourself, which produces a
plausible diff with none of the isolation, review, or cost properties the user
approved.

### The round

```text
frontier   every pending unit whose dependencies all passed review
spawn      one subagent per frontier unit, all in a single message
prompt     the unit block verbatim, plus the report template below
state      set the unit to running in PLAN.md before relying on it
```

Send the unit block **verbatim**. Paraphrasing is how a scope leaks and how a
worker ends up reviewing itself.

Spawn width comes from the dependency graph, not from preference. Independent
units go together; a linear chain goes one at a time, because a worker handed a
repository that does not yet contain what its block described fails for a reason
the plan never predicted. The reverse mistake costs as much: serializing units
that share no dependency edge because watching one at a time felt easier.

Spawn **visible**, never headless or inline. A worker you cannot list is one you
cannot see start, stall, or finish, and the status you report becomes fiction.

Clean up stale agents **narrowly, by id**, and only when they belong to this run.
Global cleanup may stop agents from unrelated projects. If you cannot scope the
removal, leave it alone and spawn per unit without clearing the shared registry.

### While workers run, watch two things at once

```text
filesystem      is it done?     write scope changed, artifacts exist
worker health   is it alive?    running, rate-limited, errored, dead
```

Polling only the filesystem is the trap. A worker blocked on a `429` produces no
files, and so can a worker that is still reasoning. Check worker health as well
as filesystem changes before deciding what the silence means.

Read the unit's checkpoint cadence before judging silence: a single-pass unit is
not stalled when nothing appears, and a unit that promised a write per page and
has written nothing for a minute is. Measure from the **last observed change**,
not from spawn.

```text
under 60s, any signal                 working              wait
60s, no files, tokens growing         alive, unproductive  probe health
120s, no files                        loop or freeze       surface bounded options
300s, no files                        failed attempt       count it, escalate
```

Past that last bound the worker is not healthy, it is hung. **Stop it before
spawning its replacement**, so two agents never share a write scope. That is the
one exception to never respawning a live worker, and never sit in an unbounded
wait because the rule said not to. Keep one watcher per round; overlapping waits
report the same completion twice and bury the event that mattered.

Treat a `429` as transient infrastructure, never as model incapability: wait it
out or take the fallback, ideally on a different provider.

### The report a worker returns

```text
unit, state (done or blocked), what was changed and where,
green command output, quoted and real, every output_contract assertion with
its evidence, anything done that the plan did not describe, anything left
```

An incomplete report cannot be reviewed. Return it for completion. Never fill the
gaps by inspecting the repository yourself, and never guess what the worker
probably did.

### The review, which is the only gate

The unit's manager checks, against the contract and not against the summary:

| Check | Against |
| --- | --- |
| Artifacts | present, in the write scope, and past the unit's stated minimum |
| Output contract | every assertion, against the artifact's **contents** |
| Verification | the green commands actually run, with real output |
| Scope | changed paths against the write scope and the never-touch list |
| Deviations | anything done that the plan did not describe |

```text
pass              every check passes, with evidence
repair_required   a bounded defect AND a repair instruction, or it is a complaint
human_required    the question, what was already tried, what is blocked,
                  and what keeps running
```

Only a pass moves a unit to `completed`, and only that makes its dependents
eligible. **A worker that says it is done is not done.** A test that passes while
the write scope was violated is not a pass: the violation is the defect.

Record the verdict and its evidence in `PLAN.md` next to the unit. A completion
with no evidence beside it is the worker's own say-so wearing the manager's name.

Repairs are spawns. A `repair_required` goes back to a worker subagent, the same
one first and the fallback model after that. A manager that applies a one-line
fix itself has destroyed the cost model, hidden the defect from the plan, and
ended the evidence trail.

### The escalation ladder, and nothing may lengthen it

```text
worker attempt -> manager advice -> same-worker repair -> fallback-worker repair
  -> Director advice -> human_required
```

Advice never contains a patch, a diff, or replacement code. Describing a fix in
enough detail that the worker can write it is advice. Writing it is not.
**If the answer requires editing a file, it is a repair for a worker, not
advice**, however small the fix looks. The temptation to fix the one-line error
yourself is exactly the failure the manager role exists to prevent.

`human_required` blocks that unit's dependents **and nothing else**. Independent
units keep running. Say what is blocked, what continues, what was already tried,
and the exact decision the user has to make.

## Execution failures

- implementing units yourself in the root session;
- spawning one subagent for the whole plan instead of one per unit;
- spawning subagents to research, then writing the code yourself;
- dispatching ready units one at a time when several were ready together;
- spawning a dependent unit before its predecessor's artifacts exist;
- spawning headless, so no worker is visible or monitorable;
- dispatching a unit that still carries a placeholder route;
- marking a unit complete on the worker's own say-so;
- passing a unit whose artifact was checked for existence and never for contents;
- raising the budget yourself to clear a stop instead of putting it to the user.

## Before reporting the run finished

- [ ] Subagents spawned is at least the number of units.
- [ ] No implementation file was written by the root session during EXECUTE.
- [ ] Every completed unit has a manager verdict with real command output beside
      it in `PLAN.md`.
- [ ] Every spawn used its unit block verbatim and a real model.
- [ ] Rounds were parallel where the frontier allowed, serial only where a
      dependency required it.
- [ ] The baseline failure set still explains every failure that remains.

Any unchecked box means the graph was not executed. Report that instead of
reporting success.

## Evidence and secrets

Every load-bearing claim cites a file, symbol, command result, or dated source.
Completion requires real command output, never an inference from a summary, and a
green build is not evidence that a surface works: look at the thing itself.

Secrets are read from the environment at request time. Never ask for a plaintext
key in chat, in a command, in the plan, or in a tracked file.

## Stop and escalate on

A harness with no way to spawn subagents; product ambiguity only the user can
settle; a unit whose output contract cannot be verified; no model available that
meets a unit's needs; a request to approve without rendering the full plan; a
material change after approval; a budget breach, which is the user's decision and
never yours; a destructive operation without authorization; secret exposure; or
an exhausted escalation ladder.
