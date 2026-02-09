---
name: rlm
description: "Use when processing very long contexts, large documents, or extensive codebases that exceed practical context window limits. Activate when user mentions 'rlm', 'recursive language model', or needs to process files too large to read into context."
---

# Recursive Language Model (RLM)

You have access to `rlm-query` and `rlm-batch` for delegating long-context work to sub-agents. **Do not try to orchestrate the work yourself** — your job is to set up the workspace and immediately hand off to a depth-0 orchestrator.

## Setup

```bash
SKILL_DIR="$(dirname "$(readlink -f ~/.claude/skills/rlm/SKILL.md 2>/dev/null || echo ~/.claude/skills/rlm/SKILL.md)")"
export PATH="${SKILL_DIR}/scripts:$PATH"
```

## What to do

1. **Write a prompt file** describing what needs to be done, including paths to any input files:
   ```bash
   TASK="my-task"
   mkdir -p .rlm/$TASK
   cat > .rlm/$TASK/task.md << 'EOF'
   Analyze /path/to/big-document.txt for liability risks...
   EOF
   ```

2. **Launch the root orchestrator in the background** so you can give the user progress updates while it runs:
   ```bash
   rlm-query .rlm/$TASK/task.md .rlm/$TASK/result.out \
       --task $TASK --model opus --max-depth 2 &
   RLM_PID=$!
   ```

3. **Poll for progress** and keep the user informed. Check active sub-agents, completed results, and whether the orchestrator has finished:
   ```bash
   # How many sub-agents are running?
   tmux list-sessions 2>/dev/null | grep -c "rlm-$TASK" || echo 0
   # How many results are in so far?
   ls .rlm/$TASK/results/*.out 2>/dev/null | wc -l
   # Is the orchestrator done?
   kill -0 $RLM_PID 2>/dev/null && echo "still running" || echo "done"
   ```
   Tell the user things like "3 sub-agents active, 5/16 chunks processed so far" while waiting.

4. **Read the result** once the orchestrator finishes:
   ```bash
   wait $RLM_PID
   cat .rlm/$TASK/result.out
   ```

The orchestrator receives the full RLM instructions (`rlm-agent.md`) automatically and knows how to split, delegate to sub-agents, and aggregate. You don't need to read those instructions yourself.

## Configuration

Set these via flags on `rlm-query` or environment variables. Choose based on the task:

- **`--model` / `RLM_MODEL`** (default: `opus`) — Model for the orchestrator and sub-agents. Use `opus` for the root orchestrator (it needs to plan and decompose). Sub-agents doing straightforward work (classification, extraction) can use `sonnet` — the orchestrator can pass `--model sonnet` when it calls `rlm-batch`.

- **`--max-depth` / `RLM_MAX_DEPTH`** (default: `3`) — How many levels of recursion are allowed. Depth 1 means the orchestrator can spawn leaf agents but they can't delegate further. Depth 2 is usually enough — go higher only if chunks are still too large for a single agent after one level of splitting.

- **`--timeout` / `RLM_TIMEOUT`** (default: `1200`) — Seconds before a sub-agent is killed. If agents are timing out, either increase this or split into smaller chunks.

- **`RLM_MAX_PARALLEL`** (default: `15`) — Max concurrent Claude instances across the entire task tree. All depths share the same pool. Lower this if you're hitting rate limits or resource constraints. Set via environment variable only (not a flag).

Example for a large task:
```bash
RLM_MAX_PARALLEL=10 rlm-query .rlm/$TASK/task.md .rlm/$TASK/result.out \
    --task $TASK --model opus --max-depth 2 --timeout 1200
```

## Monitoring

Sub-agents run in named tmux sessions. You (or the user) can watch them live:

```bash
tmux list-sessions | grep rlm   # list all active sub-agents
tmux attach -t rlm-<task>-d1-…  # watch a specific sub-agent work
```
