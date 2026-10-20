# Coding-agent harness from scratch · Roadmap

A learning project. I'm building the harness for a coding agent in TypeScript, from scratch and without an agent framework, and measuring it against an established open-source agent on the same tasks.

The harness is everything around the model: the tool loop, the tools and their sandbox, budgets, context management, session resume, and the eval runner that scores the result. The model is off the shelf; the harness around it is hand-written.

**Current milestone:** 0 (groundwork) · **Last updated:** 2026-10-19

## The question

What makes a coding-agent harness good at a fixed cost? The headline metric is **pass^3 per dollar**: how often the agent solves a task in all three of three independent trials, relative to what those runs cost. The problem brief (`docs/brief.md`) sets out the targets and the reasoning.

## What gets built

- Model clients for the Anthropic Messages API and for an OpenAI-compatible local endpoint (llama.cpp), written on plain HTTP behind one interface.
- The tool-use loop.
- Tools: bash in a per-task Docker container with CPU, memory and process limits; read_file; str_replace editing; grep/glob.
- Step, cost and time budgets.
- JSONL transcripts that replay deterministically.
- Context management: tool-output truncation and clearing, compaction at a threshold, an aider-style repo map and a notes file.
- Checkpoint and resume of sessions.
- An MCP client and an MCP server for repo search.
- An eval runner: a clean environment per trial, test-based grading, three trials per task.
- A React supervision console: streaming transcript, interrupt and approve, and per-action provenance.

**Off the shelf:** the model API, Docker, a tree-sitter parser and benchmark task data. The point is harness engineering, so the loop, the context policy, the sandbox policy and the statistics are all hand-written.

**Non-goals:** training models, building on an agent framework, and SWE-bench Verified claims.

## How the work is done

- **Tests first** for pure logic. Unit tests run against a scripted fake model, so no test calls a live API.
- **Replayable runs.** Every session is logged as JSONL and must replay from its log with zero differences.
- **Threat model before sandbox.** The escape tests are written before the sandbox they test.
- **Measured, not asserted.** Results come from three trials per task, with 95% intervals and a flaky-task report. The external baseline is mini-SWE-agent with the same model, budgets, container image and limits.
- **Pinned infrastructure.** Image digests, container limits, the model version and the harness commit are printed at the top of every run report.
- **Recorded decisions.** Architecture decisions are written up as ADRs in `docs/adr/`.
- **AI assistance, logged.** I write the code, tests, design docs and write-ups myself. An AI assistant is my tutor and reviewer, not the implementer; it helped draft this roadmap from my working plan. Any change an AI tool shapes is logged in `docs/verification-log.md`, with how I verified it.

## Milestones

| # | Milestone | Status |
|---|---|---|
| 0 | Groundwork: problem brief and repository setup | In progress |
| 1 | Agent loop, three tools and replayable logs | Not started |
| 2 | Sandbox, budgets, resume and a supervision console | Not started |
| 3 | Eval harness on 20–50 real bug-fix tasks | Not started |
| 4 | Context features, measured by ablation | Not started |
| 5 | Head-to-head with mini-SWE-agent | Not started |
| 6 | SWE-bench Pro subset on pinned infrastructure | Not started |

### Milestone 0 · Groundwork

**Done when:** the problem brief is written, before any harness code.

- `docs/brief.md`: the question, the metric, the targets (the done-when lines below) and the non-goals.
- The repository layout, and a `.gitignore` that keeps secrets out. API keys live only in environment variables.

### Milestone 1 · Agent loop, three tools and replayable logs

**Done when:** the agent solves 5 of 5 hand-made smoke tasks, and every trajectory replays from its log.

The design follows Thorsten Ball's *How to Build an Agent*, ported from Go to TypeScript.

- Five smoke tasks, written before any harness code. Each has a small fixture repo, a prompt and an automatic pass/fail check, and each tool is needed by at least one task.
- A transcript format documented in `docs/design.md`: one JSON object per line, with events for session start, model request, model response, tool call, tool result and session end. The doc also says what replay needs and what it must ignore.
- Loop tests against a scripted fake model:
  - a text-only reply ends the session;
  - a tool result goes back with its matching tool-call ID;
  - an unknown tool or invalid input becomes an error the model can read;
  - API failures are retried a bounded number of times, then logged.
- Model clients on plain HTTP, with every response validated at the boundary. The no-SDK choice is recorded in an ADR.
- read_file, list_files and str_replace tools. Each declares its input schema and its errors (a missing file, a path outside the workspace, an old string that is absent or appears more than once), and every error is unit-tested.
- The loop, with JSONL logging and a replay mode. Replay reruns the tools on a fresh copy of the fixture and fails on any difference in tool output or in the final diff.
- Tokens and cost recorded per task.

### Milestone 2 · Sandbox, budgets, resume and a supervision console

**Done when:** writes outside the workspace and network egress are blocked in a documented escape test, and container limits are pinned and reported.

- `docs/escape-test.md`, written before the sandbox:
  - the threat: every command the model writes is untrusted;
  - what must be blocked: writes outside the workspace, network egress, reading host files or environment variables, and exhausting CPU, memory or processes;
  - one automated test per item, run through the bash tool.
- A bash tool on Docker:
  - one container per task, from an image pinned by digest;
  - only the task workspace mounted, a non-root user, no network, and CPU, memory and process limits;
  - it returns the exit code, truncated output and duration;
  - the container is removed on every exit path, including crashes;
  - grep/glob runs in the same container as a structured tool.
- Step, cost and time budgets. When one runs out, the session stops cleanly and the last log event names the budget and the totals. Each budget is tested exactly at its limit.
- Checkpoint and resume after every step. A test kills the process mid-session, resumes it, and checks that no edit is applied twice and that the outcome matches an uninterrupted run.
- A React console on the harness's event stream:
  - a streaming transcript, and an interrupt that stops after the current step;
  - approve or deny for actions the policy marks as risky, with both paths tested end to end;
  - per-action provenance linking each tool call to the model response that produced it and to its output;
  - a UI that meets WCAG 2.1 AA.
- A sandbox-policy ADR.

### Milestone 3 · Eval harness on real bug-fix tasks

**Done when:** pass@1 and pass^3 are reported with 95% intervals, alongside a flaky-task report.

- 20–50 tasks mined from bug-fix commits in public repositories (personal or open source), modeled on SWE-bench's task format. Each task has the base commit, a problem statement, the hidden tests the fix added or changed, and the fix as a reference patch.
- Each task is validated in a clean container before it enters the suite: the hidden tests fail on the base commit and pass with the reference patch, on repeated runs. Tasks that don't pass reliably go on a flaky list.
- Metric functions, written test-first:
  - pass@1 is mean success over trials;
  - pass^3 is the share of tasks where all three trials pass;
  - 95% intervals come from a bootstrap over tasks (or another method justified in an ADR), reproducible with a fixed seed.
- A runner. For each task and trial it starts a fresh container and checkout, runs the agent under fixed budgets, then runs the hidden tests. It writes one results line per trial: outcome, tokens, cost, wall time, transcript path and pinned config.
- A flaky-task report: reference patches that aren't stable, and tasks whose outcome varies across trials, each linked to its transcripts.
- `make eval` reproduces the baseline run with one command.

### Milestone 4 · Context features, measured by ablation

**Done when:** each feature's effect on pass rate and token cost is measured on the same tasks.

- Every context feature sits behind a flag, so a run config states exactly what's on.
- Tool-output truncation, with a marker that says how much was dropped, plus clearing of old tool results.
- Compaction when the context passes a threshold, with every compaction recorded in the transcript.
- A repo map ranked from a tree-sitter parse and rendered within a token budget, plus a notes file that survives compaction.
- An MCP client written from the specification (JSON-RPC over stdio), and an MCP server that exposes repo search. The client is also checked against an example server from an official MCP SDK.
- The ablation design and a prediction for each feature, written before any run.
- A results table: each feature's change in pass rate (with intervals) and in tokens and cost per task. Effects whose intervals include zero are marked.
- A context-policy ADR that decides keep, drop or default-off for each feature, citing the table.

### Milestone 5 · Head-to-head with mini-SWE-agent

**Done when (targets):** the harness is within ±10 points of mini-SWE-agent, and it solves at least 14 of 20 custom repo-fix tasks.

- A thin adapter, so mini-SWE-agent reads this task format and writes this results schema. Model, budgets, container image and limits are held constant.
- The comparison set is fixed before any run: at least 25 tasks, including the 20 custom repo-fix tasks.
- Both agents run three trials per task. Each reports pass@1, pass^3, 95% intervals, tokens, cost and pass^3 per dollar.
- Where the agents disagree, failures are sorted by cause: wrong file, misread test, lost context, tool misuse, or budget exhaustion.
- If a target is missed: one change based on that analysis, a rerun, and both runs reported. Any tuning on the comparison set is disclosed.

### Milestone 6 · SWE-bench Pro subset on pinned infrastructure

**Done when:** results are reported with the infrastructure pinned, and gaps under about 3 points are treated as noise ([Anthropic on infrastructure noise in agentic coding evals](https://www.anthropic.com/engineering/infrastructure-noise)).

- A compute choice (sb-cli, Modal or a rented x86 machine), recorded in an ADR with a cost estimate. Official SWE-bench images are x86 and need about 120 GB of disk, 16 GB or more of RAM and 8 CPUs.
- A fixed SWE-bench Pro subset, with its selection rule written down before any run.
- Pinned infrastructure, printed in every results header: machine type, CPU and memory, image digests, model version, budgets and harness commit.
- Repeated trials of this harness (and of mini-SWE-agent if the budget allows), reporting pass@1 and pass^3 with intervals.
- The public write-up of the results.

## Beyond the code

- **Problem brief** before Milestone 1.
- **Design doc** before Milestone 2, extended before Milestone 4. It covers goals and non-goals, plus rejected alternatives for three choices: a single loop vs orchestrator-workers, bash-only vs structured tools, and compaction vs retrieval.
- **ADRs:** model-client approach, sandbox policy, interval method, context policy, and Milestone 6 compute.
- **Baseline and reproduction:** mini-SWE-agent on the same tasks and model as the named baseline; three trials per task, medians and 95% intervals; `make eval` as the one-command reproduction.
- **Operations:**
  - SLOs for session completion and resume;
  - a runbook for stuck sessions, runaway cost and orphaned containers;
  - failure tests for an API outage, a container crash, and kill-and-resume.
- **Outside review in two stages:** the design doc before Milestone 2, then the results and write-up before publishing.
- **Outside users:** two people run it on their own repos. What broke for them, and what changed as a result, is recorded.
- **Upstream:** one issue or pull request to mini-SWE-agent or an MCP SDK.
- **Usable by strangers:** a README quick start that runs in under 15 minutes, CI, and a license.

## Stretch goals

- A long-running two-agent harness: an initializer agent, plus a coding agent working from a feature list and a progress file. An ablation tests whether the harness fails more often without them.
- Durable sessions that survive disconnects.
- Multi-agent orchestration.
- Rejection-sampled trajectories exported as training data.

## Compute and cost

- Milestones 0–5 run on an Apple Silicon Mac with Docker. Milestone 6 runs on x86 cloud machines (see above).
- API spend is capped by per-run budgets and reported per task. Cost is token usage times a per-model price table kept in config.

## Repository layout

```text
src/       agent loop, model clients, tools, context features, MCP client and server
console/   React supervision console
tasks/     smoke tasks now, eval tasks later
tests/
bench/     eval runs and results
docs/      brief.md, design.md, adr/, log.md, verification-log.md, escape-test.md
```

## References

- Anthropic Engineering, [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Thorsten Ball, [How to Build an Agent](https://ampcode.com/notes/how-to-build-an-agent)
- [mini-SWE-agent](https://github.com/SWE-agent/mini-swe-agent)
- Anthropic Engineering, [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Anthropic Engineering, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Anthropic Engineering, [Quantifying infrastructure noise in agentic coding evals](https://www.anthropic.com/engineering/infrastructure-noise)
- [SWE-bench](https://www.swebench.com/SWE-bench/): paper, harness and docs; [sb-cli](https://www.swebench.com/sb-cli/)
- [SWE-bench Pro](https://github.com/scaleapi/SWE-bench_Pro-os) (Scale AI)
- aider, [Building a better repository map with tree sitter](https://aider.chat/2023/10/22/repomap.html); [tree-sitter](https://tree-sitter.github.io/tree-sitter/)
- [Model Context Protocol specification (2026-07-28)](https://modelcontextprotocol.io/specification/2026-07-28); [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Anthropic Messages API](https://platform.claude.com/docs/en/api/messages); [llama.cpp server](https://github.com/ggml-org/llama.cpp/tree/master/tools/server)
- Docker: [resource constraints](https://docs.docker.com/engine/containers/resource_constraints/), [`docker container run`](https://docs.docker.com/reference/cli/docker/container/run/)
- [Modal documentation](https://modal.com/docs)
- Chip Huyen, [*AI Engineering*](https://huyenchip.com/books/) (O'Reilly)

---

Feedback and questions are welcome: please open an issue.
