# Repo Task Proof Loop

This skill was built from [OpenClaw-RL: Train Any Agent Simply by Talking](https://arxiv.org/html/2603.10165v1) and applies its proven approach to agentic flows in a repo-local workflow.

> "next-state signals are universal, and policy can learn from all of them simultaneously."

Repo Task Proof Loop is a repo-local workflow skill for non-trivial coding tasks in Claude Code.

It creates a durable task folder under `.agent/tasks/<TASK_ID>/`, installs project-scoped Claude Code subagents, updates `CLAUDE.md` with the workflow, and drives a strict implementation loop:

`spec freeze → build → evidence → fresh verify → minimal fix → fresh verify`

The goal is simple: keep all proof of work inside the repository, separate implementation from verification, and make it easy to resume or audit a task later.

## What problem this solves

Large coding-agent tasks often fail in predictable ways:

- the agent claims the job is done without durable proof
- the same session both implements and judges its own work
- acceptance criteria drift during the task
- a later session cannot tell what was actually verified
- repo guidance in `CLAUDE.md` is ignored or not reused

This skill addresses that by enforcing a repo-local artifact set and role-separated subagents.

## What this skill installs and manages

Inside the target repository:

```text
.agent/tasks/<TASK_ID>/
  spec.md
  evidence.md
  evidence.json
  raw/
    build.txt
    test-unit.txt
    test-integration.txt
    lint.txt
    screenshot-1.png
  verdict.json
  problems.md
```

Also inside the target repository:

```text
.claude/agents/
  task-spec-freezer.md
  task-builder.md
  task-verifier.md
  task-fixer.md
```

And a managed workflow block in `CLAUDE.md`.

The managed block is updated in place and is designed to preserve unrelated user content outside the managed section.

## Quick prompts to begin work with this skill

Pick the prompt that matches the current task state.

### Init

```text
Use $repo-task-proof-loop to initialize this repository for the repo-local
spec -> build -> evidence -> verify -> fix workflow. Install or refresh
the project-scoped subagents, update the managed workflow guidance, and set
this repo up to follow the proof-loop philosophy for future tasks.
```

### Status

```text
Use $repo-task-proof-loop to find the existing repo-local task that matches
the task described below, inspect its artifacts, and report the matched task ID,
current status, and next recommended step.
...
```

### Build

```text
Use $repo-task-proof-loop to continue the task described below in this repository.
Reuse the matching repo-local task if it already exists; if not, stop after
explaining that init should be run first.
...
```

For `Status` and `Build`, replace `...` with either `Task file: <path/to/task-file.md>` on the next line or the task text pasted on following lines.

## Промпты на русском / Russian prompts

Skill понимает промпты на любом языке. Ниже — русские аналоги.

### Инициализация

```text
Используй $repo-task-proof-loop чтобы инициализировать этот репозиторий
для workflow spec -> build -> evidence -> verify -> fix. Установи субагенты
и обнови CLAUDE.md.
```

### Статус

```text
Используй $repo-task-proof-loop, найди существующую задачу, соответствующую
описанию ниже, проверь её артефакты и покажи ID задачи, текущий статус
и рекомендуемый следующий шаг.
...
```

### Сборка / продолжение работы

```text
Используй $repo-task-proof-loop, продолжи задачу описанную ниже.
Если задача уже существует в репозитории — используй её.
Если нет — останови и объясни что сначала нужен init.
...
```

### Полный цикл

```text
Используй $repo-task-proof-loop, запусти полный цикл для задачи ниже:
spec freeze -> build -> evidence -> verify -> fix до PASS.
...
```

Вместо `...` подставь описание задачи текстом или `Файл задачи: <путь/к/файлу.md>`.

## Installation

Install the skill as a project skill in Claude Code.

Copy this directory to:

```text
.claude/skills/repo-task-proof-loop/
```

## Quick start

1. Install the skill in your repository.
2. Start Claude Code with one of the prompts above that matches your task state.
3. Let the skill initialize the repo-local task structure.
4. Let the agent execute the workflow using role-separated subagents.
5. Validate before sign-off.

## Manual helper commands

The bundled helper script ships three CLI commands:

- `init`
- `validate`
- `status`

The workflow phases `freeze`, `build`, `evidence`, `verify`, `fix`, and `run` are skill-level commands for the agent, not direct CLI subcommands in the current package.

Set `SKILL_PATH` to the installed skill directory:

```bash
SKILL_PATH=.claude/skills/repo-task-proof-loop
```

### Initialize a task

```bash
python3 "$SKILL_PATH/scripts/task_loop.py" init \
  --task-id feature-auth-hardening \
  --task-file docs/tasks/auth-hardening.md
```

You can also seed the task from inline text:

```bash
python3 "$SKILL_PATH/scripts/task_loop.py" init \
  --task-id feature-auth-hardening \
  --task-text "Implement auth hardening for session refresh and logout."
```

Useful options:

- `--guides auto|claude|none`
- `--install-subagents claude|yes|none`
- `--force`

### Validate a task bundle

```bash
python3 "$SKILL_PATH/scripts/task_loop.py" validate \
  --task-id feature-auth-hardening
```

### Show current task status

```bash
python3 "$SKILL_PATH/scripts/task_loop.py" status \
  --task-id feature-auth-hardening
```

## Workflow model

This skill is built around six roles and states.

### 1. Spec freeze

Create or refine `.agent/tasks/<TASK_ID>/spec.md`.

The spec must contain at least:

- original task statement
- explicit acceptance criteria labeled `AC1`, `AC2`, ...
- constraints
- non-goals

It may also include assumptions and a concise verification plan.

### 2. Build

A builder subagent implements the task against the frozen spec.

The builder should make the smallest safe change set that satisfies the acceptance criteria.

### 3. Evidence

The same builder session should switch into evidence mode when possible.

It writes:

- `evidence.md`
- `evidence.json`
- raw artifacts under `raw/`

Evidence may conclude `PASS`, `FAIL`, or `UNKNOWN` for each acceptance criterion. It must not keep changing production code.

### 4. Fresh verify

A fresh verifier session inspects the current repository state and reruns checks.

It writes:

- `verdict.json`
- `problems.md` when the verdict is not `PASS`

The verifier must not edit production code.

### 5. Fix

A fresh fixer session reads:

- `spec.md`
- `verdict.json`
- `problems.md`

It applies the smallest safe fix set, regenerates the evidence bundle, and stops without writing final sign-off.

### 6. Verify again

A fresh verifier session reruns verification.

If the task is still not `PASS`, the workflow loops:

`fix → verify → fix → verify`

## Subagent roles

This skill installs four role-specific subagents for Claude Code.

### `task-spec-freezer`

Purpose:
- freeze the task into `spec.md`

Boundaries:
- may read repo guidance and relevant code
- must not change production code
- must not write `verdict.json` or `problems.md`

### `task-builder`

Purpose:
- implement the task
- later switch into evidence mode

Boundaries:
- in `BUILD`, implement against the spec
- in `EVIDENCE`, do not change production code

### `task-verifier`

Purpose:
- perform fresh-session verification against the current codebase

Boundaries:
- must not edit production code
- must not patch the evidence bundle to make it look complete
- must write `verdict.json`
- must write `problems.md` only when the verdict is not `PASS`

### `task-fixer`

Purpose:
- repair only what the verifier identified

Boundaries:
- must reread the spec and verifier output
- must reconfirm the problem before editing
- must regenerate evidence after the fix
- must not write final sign-off

## How the agent should use subagents

Use the installed project agents under `.claude/agents/`.

Example shape:

```text
Use the `task-verifier` agent for TASK_ID <TASK_ID>. It must be a fresh verifier
pass against the current codebase and must write verdict.json and, if needed,
problems.md.
```

For large tasks, prefer one child per role rather than a single general-purpose child.

## Guardrails

- All task artifacts must stay inside the repository.
- Do not claim task completion unless every acceptance criterion is `PASS`.
- Keep implementer, verifier, and fixer roles separate.
- Keep verifier passes fresh.
- Prefer minimal diffs during repairs.
- Preserve unrelated user guidance outside the managed block in `CLAUDE.md`.

## Validation and smoke testing

The package includes a smoke test:

```bash
python3 "$SKILL_PATH/scripts/verify_package.py"
```

It checks the skill structure, initializes a temporary git repository, installs the task artifacts and subagents, and validates the generated task bundle.

## Repository contents in this package

```text
repo-task-proof-loop/
  README.md
  SKILL.md
  VERIFICATION.md
  assets/
  references/
  scripts/
```
