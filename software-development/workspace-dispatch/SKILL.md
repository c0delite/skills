---
name: workspace-dispatch
description: |
  Single-agent mission orchestrator. Decomposes a mission into tasks, spawns one worker per task using the default model, verifies exit criteria, and chains tasks with retry. No critic pattern — each worker self-verifies. Simple, fast, works with any model config.
---

# Workspace Dispatch (Single Agent)

You are an autonomous mission orchestrator. Decompose work into tasks, spawn one worker per task, verify output, chain to the next — no user intervention needed.

## Quickstart: New Project

```
1. Save project context → memory tool (target=memory) + ~/.hermes/projects/<slug>/project.memory.md
2. Clone repo using github_token from ~/.hermes/.env
3. Discover project structure (find, ls, package.json, docker-compose.yml)
4. Write initial plan to .hermes/plans/<feature>.md
5. Set up todo list for Phase 1
6. Build, commit every meaningful block
7. If context breaks → session_search → read plan → continue from last incomplete item
```

## Flow

1. **Decompose** the goal into 2-6 tasks with machine-checkable exit criteria
2. **For each task**: spawn a worker → wait → verify exit criteria → approve or retry
3. **Report** summary when all tasks complete

## Decomposition Rules

- **Max 6 tasks** — keep it focused
- **Every task needs exit criteria** verifiable with shell commands:
  - `test -f /path` — file exists
  - `npx tsc --noEmit` — compiles
  - `grep -q "keyword" /path` — contains expected content
  - `wc -c < /path | awk '$1 > 100'` — file has real content
- **No vague criteria** — must be machine-checkable
- **Include working directory** (`cwd`) for each task
- **Each task is independent** — worker gets full context, no shared state between workers

## Preflight Checks

Before declaring a task complete, verify the build chain works:

- **Dependency versions pinned** — `package.json` has exact versions (no `latest`, no `*`), especially for ORM/DB clients, auth libraries, and frameworks. Run `npm ls <package>` to check what actually got installed.
- **Docker build succeeds** — `docker compose build <service>` exits 0 before declaring a service task done. A TypeScript-local build passing is insufficient if Docker fails.
- **Container starts and stays running** — `docker compose up -d <service>` + `docker compose ps` shows status "Up". Containers that crash on startup must be diagnosed before the task is marked complete.
- **Environment variables documented** — All required env vars appear in `.env.example` with placeholder values. Services that fail due to missing env vars are a pre-flight gap, not a runtime surprise.

When sub-agents scaffold dependencies (ORMs, auth, payments), treat version compatibility as a first-class exit criterion.

> **Reference:** `software-development/systematic-debugging/references/prisma-version-migration.md` — Prisma 7 / Docker generate issue caught via pre-flight in this session.

## Task Types

| Type | Worker Does | Verify With |
|------|-----------|-------------|
| coding | Write code, create files | file exists, tsc passes |
| research | Search, read, synthesize | output file exists with content |
| review | Read code, check behavior | reviewer outputs PASS verdict |

## Dispatch Loop

```
For each task (in dependency order):
  1. Spawn worker:
     sessions_spawn(
       task: <worker prompt>,
       label: "worker-<task-slug>",
       mode: "run",
       runTimeoutSeconds: 600
     )
  2. sessions_yield() — wait for worker
  3. Verify exit criteria via exec commands
  4. If ALL pass → mark complete, next task
  5. If ANY fail → retry (max 3) with error context, then fail + skip dependents
```

## Worker Prompt

Give each worker everything it needs in one prompt:

```
## Mission: {goal}
## Your Task: {task.title}
{task.description}

Working directory: {cwd}

## Exit Criteria (you MUST satisfy ALL):
- {criterion_1}
- {criterion_2}

## Rules
- Do NOT start servers or long-running processes
- Do NOT modify files outside your working directory
- Do NOT create scaffolding files unless explicitly requested — read existing files first and extend them
- Verify your own work before finishing — run the exit criteria commands yourself
- Commit only if the mission explicitly allows commits; otherwise leave changes uncommitted and report them
```

## Pitfalls

### Subagent File Write Conflicts

When dispatching multiple workers in parallel, each gets a fresh context. If two workers write to the same file (e.g., `package.json`, `src/index.ts`), the second write wins and the first is silently lost. Signs of this:

- `package.json` has unexpected `dependencies` from another worker's scope
- `src/index.ts` has duplicate imports or routes from two different workers
- Files that should have been created are missing

**Fix:** Assign ONE file per worker scope. If two tasks need to touch the same file, do them sequentially (same worker, same task).

### Long Tasks Exhaust Context

Workers doing scaffolding + fixing + testing in one task blow through context. Keep worker scope tight: scaffold OR fix OR test — never all three.

### Exit Criteria Must Be Self-Verifiable

If exit criteria require checking file content, give the exact grep/awk command. Don't rely on the worker to "report what it did" — the orchestrator must be able to verify independently.

On retry, append:
```
## ⚠️ Previous attempt failed (attempt {n}/3)
Error: {what went wrong}
Fix this specific issue.
```

## Completion

When all tasks done, output:

```
✅ Mission complete: {goal}

Tasks:
- ✅ {title} — verified
- ✅ {title} — verified

Output: {project_path}
Duration: {elapsed}
```

## Failure Handling

| Failure | Action |
|---------|--------|
| Worker timeout | Retry with simpler scope |
| Exit criteria fail | Retry with specific error |
| 3 retries exhausted | Mark failed, skip dependents, continue |
| Docker build fails | Check dependency version conflicts (e.g., Prisma 7 vs schema v5 syntax); check env vars passed to build |
| Container crashes on startup | Check Prisma client initialization, env vars, port conflicts — do NOT skip to next task |

## Project Memory Pattern

When starting a new project, IMMEDIATELY write key context to disk before any work begins:

```
~/.hermes/projects/<project-slug>/project.memory.md
```

Required fields:
```
Project: <name>
Path: <absolute path>
Stack: <tech stack>
Repo: <git URL>
Branch: <naming convention>
Deliverables: <what "done" looks like>
Key files: <main config paths, discovered after scan>
```

Also save to `memory` tool with target=memory so it survives context resets.

For projects that live in an Obsidian vault at repo root, check for `project.memory.md.md` in the repo root — it may already contain project context from a previous session. Always read it on first contact with a repo.

## Session Recovery

If context breaks mid-mission:
1. `session_search` → find last session with this project
2. read_file `~/.hermes/projects/<slug>/project.memory.md`
3. read_file `.hermes/plans/<feature>.md` (the on-disk implementation plan)
4. Reconstruct todo state from the last session summary
5. Continue from the last incomplete item

## Context Window Rules

- **Write session summaries** to `~/.hermes/sessions/<timestamp>-<project>.md` after every meaningful block of work (completed feature, major refactor, failed attempt). Include: what was done, what's pending, current todo state.
- **Summarize outputs** — don't dump raw logs or API responses into chat. Condense to findings.
- **Write plans to disk** — never hold mid-work state only in memory.
- **Commit frequently** — every meaningful block gets a commit. Git history is the recovery trail.
- **Sub-agent parallelization** — for independent workstreams, use `sessions_spawn` to run workers in parallel instead of chaining through the orchestrator's own context.

## Rules

- One worker per task, default model, no critic
- Workers self-verify (exit criteria are the quality gate)
- Don't hardcode model names — use whatever's available
- Don't hold state in memory — be ready for context loss
- Don't start servers in tasks
