---
name: kanban
description: Hermes Kanban multi-agent work queue — both orchestrator routing and worker lifecycle. Run as orchestrator (decompose, route, don't execute) or as a dispatched worker.
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [kanban, multi-agent, orchestrator, worker, routing, collaboration]
    related_skills: [hermes-agent, task-delegation]
---

# Kanban — Multi-Agent Work Queue

Hermes Kanban is a durable SQLite-based work queue for multi-profile / multi-worker collaboration. Two roles exist:

- **Orchestrator** — decomposes work into cards, routes to specialists, doesn't execute
- **Worker** — claims a card, does the work, completes or blocks

The core lifecycle is auto-injected into every kanban process via `KANBAN_GUIDANCE`. This skill provides the deeper playbook for both roles.

---

## Part 1: Orchestrator Playbook

Your job is to decompose, route, and summarize — never execute.

### Step 0: Discover Profiles

Before fanning out, you must ground in real profile names. Unknown assignees silently fail to spawn.

```bash
hermes profile list
# OR
kanban_list(assignee="<some-name>")  # sanity-check a name
```

Cache the result. Re-asking wastes tool calls.

### When to Use the Board (vs just doing the work)

Create Kanban tasks when any of these are true:
1. Multiple specialists needed
2. Work should survive a crash/restart
3. User may want to interject
4. Multiple subtasks can run in parallel
5. Review/iteration expected
6. Audit trail matters

If none apply, use `delegate_task` instead or answer directly.

### The Anti-Temptation Rules

- **Do not execute the work yourself.** Your restricted toolset usually lacks terminal/file/code/web.
- **For any concrete task, create a Kanban task and assign it.** Every time.
- **Split multi-lane requests before creating cards.** One card per independent workstream.
- **Run independent lanes in parallel.** No parent links between them.
- **Never create dependent work as independent ready cards.** Use `parents=[...]`.
- **If no specialist fits available profiles, ask the user.**

### Decomposition Playbook

#### Step 1 — Understand the goal
Ask clarifying questions if the goal is ambiguous.

#### Step 2 — Sketch the task graph
1. Extract lanes from the request
2. Map each lane to a profile from Step 0
3. Decide independence vs gating
4. Independent lanes → parallel cards with no parents
5. Synthesis/review cards → parent links to dependent lanes

Show the graph to the user before creating cards.

#### Step 3 — Create tasks with parent links

```python
t1 = kanban_create(title="research: cost analysis", assignee="<profile-A>", body="...")["task_id"]
t2 = kanban_create(title="research: performance", assignee="<profile-A>", body="...")["task_id"]
t3 = kanban_create(title="synthesize recommendation", assignee="<profile-B>", body="...", parents=[t1, t2])["task_id"]
```

Create parent cards first, capture their ids, and include those ids in the child's `parents` list.

#### Step 4 — Complete your own task

```python
kanban_complete(summary="decomposed into 3 tasks: T1 cost, T2 perf (parallel), T3 synthesis (gated on T1+T2)", metadata=...)
```

#### Step 5 — Report to user

Tell them what you created in plain prose, naming actual profiles.

### Goal-Mode Cards (Persistent Workers)

For open-ended cards where one turn rarely finishes:

```python
kanban_create(title="Translate full docs site to French", body="Acceptance: every page translated", assignee="<profile>", goal_mode=True, goal_max_turns=15)
```

After each turn, a judge evaluates against the title+body. Not done+budget remains → worker keeps going. Budget exceeded → blocked for human review.

### Recovering Stuck Workers

- **Reclaim** (`hermes kanban reclaim <task_id>`) — abort running worker, reset to ready
- **Reassign** (`hermes kanban reassign <task_id> <new-profile> --reclaim`) — switch to different profile
- **Change profile model** — dashboard prints `hermes -p <profile> model` hint

### Common Orchestrator Patterns

- **Fan-out + fan-in (research → synthesize):** N research cards, one synthesis card with all parents
- **Parallel implementation + validation:** Implementer + explorer in parallel, reviewer gated on both
- **Pipeline with gates:** planner → implementer → reviewer
- **Same-profile queue:** N tasks, same profile, no dependencies
- **Human-in-the-loop:** Worker calls `kanban_block()` → dispatcher respawns after `/unblock`

### Orchestrator Pitfalls

- **Inventing profile names that don't exist** — dispatcher silently fails, card sits in `ready` forever
- **Bundling independent lanes into one card** — create two cards, not one
- **Over-linking because of wording** — "finally check X" may still be parallel
- **Forgetting dependency links** — use `parents` so implement/review can't run before inputs exist
- **Reassignment vs new task** — create NEW task from blocked review, don't re-run same task
- **kanban_link argument order** — `parent_id` first, then `child_id`
- **Don't pre-create the whole graph** if shape depends on intermediate findings
- **Tenant inheritance** — pass `tenant=os.environ.get("HERMES_TENANT")` on every `kanban_create`

---

## Part 2: Worker Lifecycle & Pitfalls

You're dispatched with `--skills kanban-worker`. The basic lifecycle (6 steps: orient → work → heartbeat → block/complete) is auto-injected via `KANBAN_GUIDANCE`. This section covers deeper detail.

### Step 0: Check the board first

Between dispatch and boot, the task may have been blocked, reassigned, or archived. Always `kanban_show` first. If it reports `blocked` or `archived`, stop.

### Workspace Handling

| Kind | What it is | How to work |
|---|---|---|
| `scratch` | Fresh tmp dir, yours alone | Read/write freely; GC'd when task is archived |
| `dir:<path>` | Shared persistent directory | Treat like long-lived state |
| `worktree` | Git worktree | If `.git` doesn't exist, run `git worktree add <path> ${HERMES_KANBAN_BRANCH:-wt/$HERMES_KANBAN_TASK}` from main repo |

### Tenant Isolation

If `$HERMES_TENANT` is set, prefix memory entries with the tenant so context doesn't leak:
- Good: `business-a: Acme is our biggest customer`
- Bad: `Acme is our biggest customer`

### Good Summary + Metadata Shapes

**Coding task:**
```python
kanban_complete(summary="shipped rate limiter — 14 tests pass", metadata={"changed_files": [...], "tests_passed": 14, "decisions": [...]})
```

**Research task:**
```python
kanban_complete(summary="3 libs reviewed; vLLM wins", metadata={"recommendation": "vLLM", "benchmarks": {"vllm": 1.0, "sglang": 0.87}})
```

**Review task:**
```python
kanban_complete(summary="reviewed PR #123; 2 blocking issues", metadata={"pr_number": 123, "findings": [...], "approved": False})
```

### Review-Required Pattern

For code changes needing human eyes, block instead of complete:

```python
kanban_comment(body="review-required handoff:\n" + json.dumps({"changed_files": [...], "tests_run": 14, "tests_passed": 14}))
kanban_block(reason="review-required: rate limiter shipped — needs eyes before merging")
```

Use `kanban_complete` only when genuinely terminal (typo fix, docs change, research where writeup IS the artifact).

### Claiming Cards You Created

```python
c1 = kanban_create(title="remediate SQL injection", assignee="security-worker")
kanban_complete(summary="...", created_cards=[c1["task_id"]])
```

Only list ids from successful return values — never invent or paste.

### Block Reasons That Get Answered Fast

Bad: `"stuck"` (no context). Good: one sentence naming the specific decision needed. Leave longer context as a comment instead.

### Heartbeats Worth Sending

Good: `"epoch 12/50, loss 0.31"`, `"scanned 1.2M/2.4M rows"`. Bad: `"still working"`, sub-second intervals.

### Retry Scenarios

- `outcome: "timed_out"` — previous hit `max_runtime_seconds`. Chunk or shorten work.
- `outcome: "crashed"` — OOM or segfault. Reduce memory footprint.
- `outcome: "spawn_failed"` + `error` — profile config issue. Ask human via `kanban_block`.
- `outcome: "reclaimed"` — operator archived task out from under you. Check status carefully.

### Notification Routing

`notification_sources: ['*']` accepts from all profiles. `notification_sources: ['default', 'zilor-ppt']` restricts.

### Worker Do NOTs

- **Do NOT call `delegate_task`** as substitute for `kanban_create`
- **Do NOT call `clarify`** — there is no live user. Use `kanban_comment` + `kanban_block`
- **Do NOT modify files outside `$HERMES_KANBAN_WORKSPACE`** unless task body says to
- **Do NOT create follow-up tasks assigned to yourself** — assign to right specialist
- **Do NOT complete a task you didn't finish** — block it instead

### Worker Pitfalls

- **Task state can change between dispatch and startup** — always `kanban_show` first
- **Workspace may have stale artifacts** — read comment thread for context
- **Don't rely on CLI when tools available** — `kanban_*` tools work across all terminal backends
- **Hallucination protection** — gate blocks `kanban_complete(created_cards=...)` claiming non-existent ids
