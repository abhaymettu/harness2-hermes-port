# Quick start

This file is added by The harness2-ports Authors. It holds the install and run steps that the
[README](../README.md) leaves out, for the Hermes executor (`--substrate hm`). For the
upstream benchmarks and their requirements, see the [setup guide](SETUP.md).

## Install

Use **Python 3.12 or 3.13 and [uv](https://docs.astral.sh/uv/getting-started/installation/)**.
From the repository root:

```bash
uv venv --python 3.12 .venv
source .venv/bin/activate
uv pip install -e '.[render]'
harness2-demo --output runs/demo --k 2
```

The local demo runs both recursion modes with deterministic editors and needs
no model credentials.

The Hermes executor needs the **`hermes` CLI** (tested on Hermes Agent
v0.21.5). The localbench read boundary uses macOS `sandbox-exec`; see
[localbench](#localbench) for running elsewhere. For the upstream benchmarks,
see the [setup guide](SETUP.md).

## First run

Point the executor at a model, check that every component reaches a rollout and
that the floor holds, then run and judge two localbench tasks:

```bash
export HARNESS2_HERMES_MODEL=claude-haiku-5-5 HARNESS2_HERMES_PROVIDER=anthropic \
  HARNESS2_HERMES_PASS_ENV=ANTHROPIC_API_KEY
harness2-check --substrate hm --conformance
harness2 --bench local --substrate hm --k 1 --only-domains data-cleaning --max-tasks 2
harness2-judge --bench local --substrate hm --k 1
```

The conformance probe runs two rollouts and prints one line per component (C1
to C8), the floor line and the override line. Judging must use the same
benchmark, subset, `--k` and seed as the refinement run. On a subscription
login Hermes reports no price, so costs are recorded as unknown with token
counts.

## The Hermes executor

`--substrate hm` runs rollouts on
[Hermes Agent](https://github.com/NousResearch/hermes-agent). Each rollout gets
a fresh `HERMES_HOME` built from the harness and a fixed operator floor; your
own Hermes home, its `config.yaml`, memories, skills and `state.db` are never
read. Settings come from the environment:

| Variable | Meaning |
|---|---|
| `HARNESS2_HERMES_MODEL` | Executor model, for example `claude-haiku-5-5` |
| `HARNESS2_HERMES_PROVIDER` | Hermes provider name, for example `anthropic` |
| `HARNESS2_HERMES_CMD` | The command that starts Hermes (default `hermes`) |
| `HARNESS2_HERMES_ROLLOUTS` | Where rollout homes are created (default `~/.cache/harness2/hermes-rollouts`); refused inside a `.hermes` directory or a git checkout, whose `AGENTS.md` and `.hermes.md` Hermes would load |
| `HARNESS2_HERMES_RUNNER` | Optional prefix that runs Hermes as another user, for example `sudo -n -u rollouts` |
| `HARNESS2_HERMES_PASS_ENV` | Names of variables passed to the rollout, such as `ANTHROPIC_API_KEY`; nothing else from your environment is |
| `HARNESS2_HERMES_PROVIDERS_FILE` | Optional JSON file with a `providers:` map, for a custom endpoint |

Credentials reach Hermes through `HARNESS2_HERMES_PASS_ENV` (sent on stdin to
the launcher, never on a command line), or through a wrapper named in
`HARNESS2_HERMES_CMD` that loads them itself.

Two gaps against the eight components:

* **Hermes has no file-defined subagents.** `delegate_task` children get
  Hermes's default identity, not the parent's `SOUL.md` or memory, and cannot
  be named. A harness subagent becomes a skill holding its brief; `SOUL.md`
  tells the agent to load it and pass the brief as the `context` of
  `delegate_task`. Whether the brief is used depends on the model following
  that instruction; the probe checks that it did.
* **Hermes caps memory at 2,200 characters by default.** The floor raises
  `memory.memory_char_limit` to 20,000, the always-on cap the surface enforces,
  so a harness memory loads whole.

The floor also denies any command that would need an approval, turns off
background review, title generation, memory and skill nudges, the curator and
bundled skills, and puts the operator's hooks before the harness's. That order
is not enforcement: Hermes applies a hook's argument rewrite after every hook
has seen the original call, and a hook command is shell.

On a subscription login Hermes reports no price, so `rollout.json` records
`cost_usd: null` and `cost_source: unknown` with token counts. Hermes passes
some credential variables on to the agent's terminal and every passed variable
on to hook commands; it redacts token-shaped tool output. Use a key you can
revoke, and a separate OS user (`HARNESS2_HERMES_RUNNER`) if rollouts must not
read your files.

### Where each component lands

Under `--substrate hm`, each one lands in a real Hermes surface:

| Component | In the rollout |
|---|---|
| System prompt, guardrails, tool guidance | `SOUL.md`, in that order |
| Memory | `memories/MEMORY.md`, one entry per paragraph |
| Skills | `work/skills/`, loaded through `skills.external_dirs` so helper paths resolve from the workspace |
| Scripts | `work/scripts/` |
| Subagents | A brief skill `agent-<name>` in the home's `skills/`, named in `SOUL.md` |
| Plugins | `hooks.json` with Hermes shell hooks (flat entries: `matcher`, `command`, `timeout`, `fail_closed`) |

## localbench

`--bench local` is a small suite for checking the executor end to end: 12 tasks
in 3 domains (data cleaning, document-to-table extraction, a small repository
bugfix), each with a deterministic checker, so it needs no judge model, Docker
or cloud project. The checkers live in `localbench/tasks/<task>/grader/`,
outside every workspace and model context.

On macOS every localbench session, rollouts and improvers alike, runs under
`sandbox-exec`, enforced by the kernel for the whole process tree. A session
cannot read, write or rename anything under `localbench/tasks/`, the checkout's
`.git` (which holds every checker as a blob) or the executor's session
transcripts, cannot read or write the run state under `runs/` (scores, other
rollouts' outputs) outside its own directory, and cannot reach a tmux server or
send Apple Events, through which it could ask a process outside the sandbox to
read for it. Links are dropped from a workspace before it is scored. Copies of
a checker made elsewhere on the machine are outside the boundary, so run
untrusted executors as a separate user or in a container. A sudoers rule for
`HARNESS2_HERMES_RUNNER` must also allow `/usr/bin/sandbox-exec`, which the
runner starts. Elsewhere set `HARNESS2_GRADER_BOUNDARY=off` and run where
`localbench/tasks/` is not mounted; the only guard left is then a floor hook
that refuses tool calls naming the tasks, and it matches strings.

## Results and resume

Results are saved under `runs/`, organized by benchmark and subset where
applicable. They include task summaries, deliverables, adopted harnesses,
reflection notes, and evaluation results.

Repeat the same command to resume. Use a new `HARNESS2_RUNS_DIR` when changing
models or task selection. See [results and resume](EXPERIMENTS.md#results-and-resume)
for details; each command also supports `--help`.

