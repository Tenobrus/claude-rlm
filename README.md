# RLM: Recursive Language Models for Claude Code

A Claude Code skill that implements **Recursive Language Models (RLMs)** — enabling Claude to process arbitrarily long inputs by treating them as external data and recursively delegating analysis to sub-agents.

Based on the paper [Recursive Language Models](https://arxiv.org/abs/2512.24601) (Zhang, Kraska, Khattab; MIT CSAIL, 2026).

## What are Recursive Language Models?

LLMs have limited context windows, and their performance degrades as prompts get longer — a phenomenon called **context rot**. The standard solution is context compaction (summarization), but this loses fine-grained information.

RLMs take a fundamentally different approach: **treat the input as an external object in a programming environment, and let the LLM programmatically interact with it through code — including recursively calling itself on slices of the input.**

Three design choices distinguish RLMs from naive agent scaffolds:

1. **Symbolic handle**: The prompt is stored as a variable in the environment, not pasted into the LLM's context. The model gets metadata (length, preview) and manipulates it through code.
2. **Unbounded output via variables**: Results are built up in environment variables/files, not limited by the LLM's output length.
3. **Symbolic recursion**: The LLM can invoke sub-LM calls inside programmatic loops, enabling O(n) or O(n^2) semantic work over inputs of any length.

## Installation

Rather than installing this skill directly, we recommend one of these approaches:

1. **Paste this README** into a conversation with your agent and ask it to reimplement the system from scratch. The design decisions and architecture are all here — a capable agent can build it in one session.

2. **Ask your agent to summarize the repo** into a detailed design document, then reimplement from just that document without referencing the code or prompts directly. This gives you a clean-room implementation you fully control.

Installing scripts and prompts from a random GitHub repo as a Claude Code skill means giving that code influence over your agent's behavior. Even if the code looks fine today, a future update could change what the prompts instruct your agent to do. A clean-room reimplement sidesteps this entirely.

If you do want to install directly, read every file first — especially the prompts and skill definitions — and pin to a specific commit:

```bash
cp -r rlm-skill ~/.claude/skills/rlm
```

Once installed, Claude Code will activate the RLM skill when you need to process large contexts, or you can trigger it explicitly with `/rlm`.

## Architecture

```
User
  |
  v
Claude Code (interactive session, skill loaded)
  |
  |-- reads rlm-agent.md for instructions
  |-- examines context metadata (wc, head)
  |-- splits context with bash (split, sed)
  |-- writes prompt files, one per chunk
  |-- calls rlm-batch to fan out
  |       |
  |       |-- rlm-query chunk1 --> tmux --> claude -p --> result1.out
  |       |-- rlm-query chunk2 --> tmux --> claude -p --> result2.out
  |       |-- rlm-query chunk3 --> tmux --> claude -p --> result3.out
  |       |       |
  |       |       └── (can itself call rlm-query/rlm-batch recursively)
  |       |
  |       └── waits for all, collects results
  |
  |-- reads result files, aggregates with code
  |-- returns final answer
```

**Components:**
- **`rlm-query`** — spawns a Claude Code instance (`claude -p`) in a tmux session to execute a prompt file. Handles concurrency limiting, timeout, and depth tracking.
- **`rlm-batch`** — parallel fan-out wrapper. Runs `rlm-query` on every `.md` file in a directory concurrently.
- **`rlm-agent.md`** — single source of truth for RLM instructions, shared by the interactive skill and all sub-agents. Contains worked examples, tool reference, and configuration docs.
- **`SKILL.md`** — minimal skill trigger that points Claude to `rlm-agent.md`.

## Design Decisions

**Bash as the programming environment.** The paper uses a Python REPL with `lm_query()`. We use the filesystem as the variable store and bash as the manipulation layer — file paths are variable names, `cat` is dereference, `>` is assignment, `split` is destructuring. This works because bash already has all the primitives for chunking, filtering, slicing, and aggregation.

**True depth-N recursion.** The paper's implementation used max depth one — sub-calls were plain LLM API calls without their own REPL. Each of our sub-agents is a full Claude Code instance that can itself spawn sub-agents. Tested working at 4 levels deep.

**Concurrency limiting.** Recursive fan-out can spawn hundreds of Claude instances. All agents in a task tree share a `mkdir`-based slot pool (atomic on all Unix filesystems) with PID-based stale lock detection. Default 15 concurrent instances.

**tmux for observability.** Each sub-agent runs in a named tmux session. `tmux list-sessions | grep rlm` shows all active agents; `tmux attach` lets you watch one work in real time.

## Configuration

All configuration propagates through the recursion tree via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `RLM_DEPTH` | 0 | Current depth (auto-incremented) |
| `RLM_MAX_DEPTH` | 3 | Maximum recursion depth |
| `RLM_MAX_PARALLEL` | 15 | Max concurrent Claude instances |
| `RLM_TIMEOUT` | 1200 | Timeout per sub-query (seconds) |
| `RLM_MODEL` | opus | Model for sub-agents |
| `RLM_TASK` | auto | Task name (auto-generated if not set) |

## File Structure

```
rlm-skill/
  SKILL.md              # Skill trigger (minimal, points to rlm-agent.md)
  prompts/
    rlm-agent.md        # Shared RLM instructions (single source of truth)
  scripts/
    rlm-query           # Sub-agent spawner (tmux + claude -p)
    rlm-batch           # Parallel fan-out wrapper
```

## Example: Named Characters in Frankenstein

We asked both standard Claude Code and the RLM skill to find all named characters in the full text of *Frankenstein* (~78K tokens).

**Standard Claude Code** found 20 characters in ~1.5 minutes. It read the beginning of the file, hit the context limit, then searched for character names it already knew from training. This meant it found the major characters but missed anyone it didn't think to search for.

**RLM** found 29 characters. By splitting the text into chunks and having sub-agents read every section, it caught 9 additional minor characters that only appear once — most from a single gossipy letter where Elizabeth rattles off Geneva acquaintances (Miss Mansfield, John Melbourne, Manon, M. Duvillard, Louis Manoir, Madame Tavernier) and brief mentions elsewhere (Madame Moritz, M. Moritz, Uncle Thomas).

When we showed the RLM's list to the standard agent, it confirmed every additional character was real and present in the text. The standard agent wasn't wrong about any character it found — it was just less thorough, because it never actually read the full text.

This is the core value proposition: standard context-window approaches process what fits and guess the rest. RLM reads everything.

## References

- [Recursive Language Models](https://arxiv.org/abs/2512.24601) — Zhang, Kraska, Khattab (MIT CSAIL, 2026)
- [GitHub: alexzhang13/rlm](https://github.com/alexzhang13/rlm) — Official implementation
- [OOLONG Benchmark](https://arxiv.org/abs/2412.15204) — Long-context evaluation suite
