<p align="center">
  <img src="https://raw.githubusercontent.com/lucadominguez/graph-coder/main/assets/banner-nano.png" alt="Graph Coder Nano" width="900">
</p>

# Graph Coder Nano

Nano is the [Graph Coder](https://github.com/lucadominguez/graph-coder) method
packaged as one skill file. It tells a coding agent how to plan a change, hand
bounded tasks to workers, and review their results. There is no CLI, package or
state store to install.

That makes it easy to try, but there is a trade-off: the checks are instructions,
not code. You and your agent have to follow them.

## Install

Clone the repository, then copy `SKILL.md` into your harness's skill directory.
For JCode on Linux or macOS:

```sh
git clone https://github.com/lucadominguez/graph-coder-nano.git
cd graph-coder-nano
mkdir -p ~/.jcode/skills/graph-coder-nano
cp SKILL.md ~/.jcode/skills/graph-coder-nano/SKILL.md
```

For JCode on Windows, open PowerShell in the cloned repository:

```powershell
$dest = "$env:USERPROFILE\.jcode\skills\graph-coder-nano"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item SKILL.md (Join-Path $dest "SKILL.md")
```

For Claude Code, use `~/.claude/skills/graph-coder-nano/` instead. Other harnesses
have their own skill locations. Start a new session if the harness loads skills
only at startup, then ask it to use `graph-coder-nano` for your change.

You can also paste the file as a prompt. The method still requires a harness
that can spawn workers and expose their status; pasting it does not add those
capabilities to a model.

## Which edition to use

| | Graph Coder | Lite | Nano |
| --- | --- | --- | --- |
| Phases | 10 | 4 | 3 |
| Skill files | 8 | 3 | 1 |
| Unit fields | ~35 | 19 | 13 |
| Tooling | Python CLI, SQLite ledger, compiled graph | `gcl` CLI, JSON state | none |
| Plan checks | Code validates the plan | Code validates the plan | You and the agent check it |
| Install | pip + skills | pip + skills | copy one file |

Use Nano for a run you can supervise, a repository without Python, or a harness
where installing a CLI is inconvenient. Choose
[Lite](https://github.com/lucadominguez/graph-coder-lite) if you want checks for
missing fields, conflicting scopes and review evidence. Use
[Full](https://github.com/lucadominguez/graph-coder) for longer runs that need a
compiled graph, routing receipts and durable recovery records.

## How it works

The Director writes one plan before dispatching work. Each unit names the files
it may touch, the steps to take, the expected output and the commands that will
verify it. The user approves the full plan, not a summary.

Workers implement their assigned units. A worker's report is a submission, not
a completion verdict: its manager must check the artifacts and command output
first. Managers can advise or request repair, but they do not edit the files
for the worker. Only reviewed work can unblock dependent units.

A few rules are worth understanding before the first run:

- Specify what must be **inside** each artifact. A scraper that returns an empty
  file can still satisfy a task that only asks for a file.
- Define progress checkpoints and command timeouts. Use worker status or
  transcripts when the harness exposes them, alongside filesystem changes.
  Silence alone does not distinguish useful work from a stall or a rate limit.
- Give concurrent units separate write scopes. Check parent directories and
  case-insensitive collisions, such as `SRC/Store.py` versus `src/store.py` on a
  case-insensitive filesystem.
- Spawn one visible worker per ready unit and send its packet unchanged. Group
  independent spawns in one round. Clean up only stale workers that belong to
  this run, never unrelated agents on the machine.
- Keep retries and escalation bounded. A `human_required` unit blocks its
  dependents, not independent branches.
- Track usage against the plan's budget, including protected provider quotas.
  A subscription may have no per-request price and still have a limited weekly
  allowance. Nano has no automatic accounting or budget breaker.

## What you give up

| Missing tooling | What you need to handle |
| --- | --- |
| `gcl` CLI checks | Read the plan for scope collisions, missing fields and dangling dependencies. |
| JSON state and `gcl recover` | Reconstruct the frontier from `PLAN.md` after an interruption; verify that recorded reviews actually happened. |
| Approval bound to a contract hash | Notice when an edit changes the approved contract and obtain approval again. |
| Recorded per-turn usage | Record spending and stop when a budget is reached. |
| Separate planner and reviewer skills | Keep the Director, Manager and Worker responsibilities separate within one skill. |

If you need these checks to be enforced by code, use Lite rather than relying
on Nano to behave as if the tooling were present. There has not been a live
behavioral evaluation showing that agents reliably follow every rule.

MIT licensed. See [NOTICE](NOTICE) for provenance.
