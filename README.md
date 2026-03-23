# zhukong-orchestrator

`zhukong-orchestrator` is a Codex skill repository for running a strict "orchestrator" workflow: plan, dispatch work to a subagent, wait safely for long-running jobs, verify against explicit acceptance criteria, and keep an audit trail on disk.

This repository is not a typical application. Its source of truth is the skill prompt in `SKILL.md`, plus a bundled local `gxd-subagent-shim` wrapper and Python implementation under `assets/`.

## What This Skill Does

The skill is designed for tasks where a main agent should act like a controller rather than an implementer.

Core behavior:

- The orchestrator does not directly edit project files.
- The orchestrator does not directly run build, test, or other side-effecting project commands.
- All execution must go through the bundled local `gxd-subagent-shim`.
- The orchestrator should avoid exposing raw subagent logs or half-finished intermediate work to the end user.
- Every run is traced into `.artifacts/agent_runs/<RUN_ID>/...`.

In practice, the skill is useful when you want:

- strict plan -> execute -> verify sequencing
- explicit acceptance criteria per step
- controlled rework loops instead of vague retrying
- long-running safety rules for subagent jobs
- filesystem-backed traceability for what was requested and what was returned

## Repository Layout

```text
.
├── README.md
├── README_ZH.md
├── SKILL.md
├── references/
│   ├── agent_v1.1.md
│   └── gxd-subagent-shim.md
└── assets/
    ├── bin/
    │   └── gxd-subagent-shim
    └── gxd-subagent-shim-0.2.3/
        ├── pyproject.toml
        ├── README.md
        └── gxd_subagent_shim/
```

Key files:

- `SKILL.md`: the main prompt and policy document for the orchestrator.
- `references/agent_v1.1.md`: a reference version of the orchestrator prompt and output contract.
- `references/gxd-subagent-shim.md`: quick reference for invoking the bundled shim correctly.
- `assets/bin/gxd-subagent-shim`: the preferred wrapper that forces use of the bundled shim source.
- `assets/gxd-subagent-shim-0.2.3/`: the local stdlib-only Python shim implementation.

## How the Bundled Shim Works

The wrapper script at `assets/bin/gxd-subagent-shim` prepends the bundled shim source directory to `PYTHONPATH` and runs:

```bash
python -m gxd_subagent_shim ...
```

This avoids accidental PATH drift and prevents the skill from silently using a different globally installed `gxd-subagent-shim`.

The shim itself:

- supports `create` and `resume`
- resolves step metadata from JSON input and CLI flags
- captures backend stdout/stderr
- records artifacts and events under `.artifacts/agent_runs`
- extracts `thread_id`, model, output text, and request metadata
- can compact Codex event streams for more readable stdout

According to `pyproject.toml`, the bundled shim requires Python `>=3.9` and has no runtime dependencies beyond the standard library.

## Execution Model

The orchestrator prompt in `SKILL.md` enforces a strict pipeline:

1. Extract the goal, scope, constraints, and success criteria.
2. Build a small serial plan, usually starting with a single step `S1`.
3. Dispatch that step through the local shim.
4. Wait without prematurely interrupting long-running work.
5. Verify the result using explicit acceptance criteria and artifact evidence.
6. If verification fails, send structured rework feedback and resume the same thread.
7. Report to the user only once the task is complete or definitively failed.

The default design bias is to keep plans small and to put implementation plus self-verification in the same subagent step unless there is a real isolation reason to split the work.

## Long-Running Safety Policy

This repository is built around the idea that a silent or long job is not automatically a failed job.

The policy encoded in `SKILL.md` is:

- do not interrupt a running shim call just because it takes a long time
- only treat the run as stalled after at least 20 minutes with no new stdout and no new stderr
- treat 60 minutes as the hard upper bound for a single `create` or `resume` call
- if the execution platform itself kills the process early, treat that as runner/platform timeout rather than proof that the subagent failed

The subagent prompt template also requires heartbeat output for commands that may run longer than 5 minutes.

## Audit Artifacts

The bundled shim writes an append-oriented audit trail below:

```text
.artifacts/agent_runs/<RUN_ID>/
├── meta.json
├── events.jsonl
├── index.md
└── steps/
    └── <STEP_ID>/
        └── rounds/
            └── R0/
```

Typical round-level files include:

- `request_raw.txt`
- `create_request.json`, `resume_request.json`, or `rework_request.json`
- `shim_response.json`
- `subagent_output.md`
- `shim_stdout.txt`
- `shim_stderr.txt`
- `shim_error.txt`
- `shim_abort.txt`

This means the audit trail comes from the shim's persisted files, not from a later human summary of what supposedly happened.

## Requirements

To use this skill effectively, the surrounding environment should provide:

- Python 3.9 or newer
- a working `codex` CLI in `PATH` for the default backend
- optionally a working `claude` CLI plus `ANTHROPIC_API_KEY` or `ANTHROPIC_AUTH_TOKEN` for Claude backend usage
- permission for the parent agent environment to run the bundled wrapper script

Relevant environment variables implemented by the shim:

- `SUBAGENT_BACKEND`
- `SUBAGENT_RUN_ID`
- `SUBAGENT_ARTIFACTS_ROOT`
- `SUBAGENT_SAVE_RAW_IO`
- `SUBAGENT_LEGACY_LOG`
- `SUBAGENT_VALIDATE_INPUT`
- `SUBAGENT_VALIDATE_OUTPUT`
- `SUBAGENT_STDOUT_MODE`
- `SUBAGENT_COMPACT_PROFILE`
- `CODEX_BIN`
- `CLAUDE_BIN`

## Recommended Usage

This repository is intended to be used as a skill, not as a standalone end-user program.

Typical trigger cases:

- the user explicitly asks for "主控", "主控模式", or "按主控流程"
- the task needs strict delegation with acceptance gates
- the task is long-running and you need resumable, auditable execution

The skill instructs the orchestrator to prefer the bundled wrapper:

```bash
"<SKILL_ROOT>/assets/bin/gxd-subagent-shim" create "<JSON>" --backend codex --run-id <RUN_ID> --task-id <TASK_ID> --step-id S1
"<SKILL_ROOT>/assets/bin/gxd-subagent-shim" resume "<JSON>" <thread_id> --backend codex --run-id <RUN_ID> --task-id <TASK_ID> --step-id S1
```

Fallback, only if the wrapper is unavailable:

```bash
PYTHONPATH="<SKILL_ROOT>/assets/gxd-subagent-shim-0.2.3" \
python -m gxd_subagent_shim create "<JSON>" --backend codex --run-id <RUN_ID> --task-id <TASK_ID> --step-id S1
```

## What Makes This Repository Different

Compared with a simple "spawn subagent and hope for the best" setup, this repository adds four concrete controls:

- execution must flow through a local pinned shim
- every step has explicit acceptance criteria
- rework is structured as machine-readable feedback
- audit evidence is written to disk by default

That combination makes it better suited to high-discipline, high-traceability workflows than an ad hoc prompt alone.

## Development Notes

The bundled shim under `assets/gxd-subagent-shim-0.2.3/` is a separate Python package with its own `README.md` and `pyproject.toml`.

If you need to modify the orchestrator behavior:

- change `SKILL.md` for policy or prompt behavior
- change `references/` only if the reference docs should stay aligned
- change `assets/bin/gxd-subagent-shim` only if the local wrapper behavior must change
- change `assets/gxd-subagent-shim-0.2.3/` only if the shim implementation itself must change

## Language

- English documentation: `README.md`
- Chinese documentation: `README_ZH.md`
