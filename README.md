# coding-agent-harness

A coding-agent harness written from scratch in TypeScript, with sandboxed tools and evals that measure it against an established open-source agent.

> [!NOTE]
> Early-stage learning project: nothing runs yet. I'm working on Milestone 0 (problem brief and setup). The [roadmap](ROADMAP.md) has the full plan and current status.

## What it is

A *harness* is the code around a language model that lets it act: it gives the model a task, runs the tools the model asks for, returns the results, and decides when to stop. This one is built without an agent framework. When finished, it will include:

- **An agent loop** that reads, searches and edits code and runs commands, logging every step so any session can be replayed exactly.
- **Sandboxed tools.** Commands run in a per-task Docker container with no network, writes limited to the task's workspace, and CPU, memory and process limits.
- **Context management:** output truncation, compaction, a repo map and a notes file, each behind a flag so its effect on pass rate and cost can be measured.
- **An eval runner** that scores the agent on real bug-fix tasks, three trials per task with 95% intervals, and compares it with [mini-SWE-agent](https://github.com/SWE-agent/mini-swe-agent) under the same model and budgets.
- **A supervision console** (React) for watching a session live, interrupting it, and approving or denying risky actions.

The roadmap's [full list](ROADMAP.md#what-gets-built) also covers session resume and an MCP client and server.

## Why from scratch

Agent frameworks are a good way to ship, but they make the decisions this project is about for you: what the loop does after an error, which commands a model may run, what stays in context, and how results are measured. Building those parts by hand is the point.

The question driving it: **what makes a harness good at a fixed cost?** The headline metric is pass^3 per dollar: how often the agent solves a task in all three of three trials, relative to what those runs cost.

## How it's built

- Tests come first. Unit tests run against a scripted fake model, so none of them call a live API.
- The sandbox's escape tests are written before the sandbox they test.
- Every run report starts with its pinned infrastructure: image digests, container limits, model version and harness commit.
- Design decisions are recorded as ADRs.

## Getting started

Nothing to run yet. A quick start arrives with Milestone 1, when the agent loop solves its first smoke tasks. It will need Node.js, Docker and an Anthropic API key.

## Acknowledgements

The loop's design follows Thorsten Ball's [How to Build an Agent](https://ampcode.com/notes/how-to-build-an-agent). [mini-SWE-agent](https://github.com/SWE-agent/mini-swe-agent) is the baseline the harness is measured against.

## License

[MIT](LICENSE). Questions and suggestions are welcome: please open an issue.
