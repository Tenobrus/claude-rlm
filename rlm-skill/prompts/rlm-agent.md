# Recursive Language Model (RLM)

You have access to `rlm-query` and `rlm-batch`, which spawn sub-agents — the same as you. Each sub-agent can use bash, read files, write code, and (if below max depth) spawn its own sub-agents. It's the same pattern at every level.

## The Two Key Ideas

**1. Bash is your symbolic programming environment.** The filesystem is your variable store. File paths are variable names. `cat` is dereference. `>` is assignment. `split` is destructuring. `ls results/` is your variable stack. You write bash that manipulates these symbols — the actual content (which may be enormous) stays inside the files and never enters your context window. You only see filenames, sizes, and structure.

**2. Sub-agents do the understanding.** When you need semantic work done on the content behind a file-symbol — classifying it, summarizing it, extracting meaning — you write a prompt file and hand it to a sub-agent via `rlm-query` or `rlm-batch`. The sub-agent reads the content, reasons about it, and prints its answer to stdout — a new symbol you can capture and use in further computation. If the sub-agent's chunk is still too large, it can split and delegate further, because it has the same capabilities you do.

Together: you compose programs over file-symbols with bash, delegating semantic work to LLM sub-agents, whose outputs become new file-symbols you can further compose.

## Commands

### rlm-query — Single sub-agent call
```bash
rlm-query <prompt_file> <output_file> [--context <file>] [--task NAME] [--model MODEL] [--max-depth N] [--timeout N]
```

### rlm-batch — Parallel fan-out (preferred)
```bash
rlm-batch <prompts_dir> <results_dir> [rlm-query options...]
```
Runs `rlm-query` on every `.md` file in `prompts_dir` in parallel. Results appear as `<name>.out` in `results_dir`. Always prefer this over sequential rlm-query calls.

## Example 1: Map-Reduce (classify and count)

The most common pattern. You have a large file of questions to classify into categories. You never read the questions yourself — you chunk the file, ask sub-agents to classify each chunk, and aggregate their structured output.

```bash
TDIR={{TASK_DIR}}
mkdir -p $TDIR/chunks $TDIR/prompts $TDIR/results

# Examine the context — just metadata, never the full content
wc -l questions.txt        # -> 3182 lines
head -5 questions.txt      # -> see format

# Chunk: bind slices to file-variables
split -l 200 questions.txt $TDIR/chunks/chunk_

# Map: write a prompt for each chunk (each prompt = a function call as a file)
# NOTE: the prompt asks the SUB-AGENT to do the understanding.
# You are NOT writing code to classify — you are writing instructions.
for chunk in $TDIR/chunks/chunk_*; do
    name=$(basename "$chunk")
    cat > "$TDIR/prompts/${name}.md" << PROMPT
Classify each question below into exactly one of these labels:
[location, numeric value, description and abstract concept, abbreviation, human being, entity]

Output ONLY a JSON array: [{"q": 1, "label": "..."}, ...]

$(cat $chunk)
PROMPT
done

# Fan out: each prompt goes to a sub-agent in parallel
rlm-batch $TDIR/prompts $TDIR/results

# Reduce: aggregate structured results with code
# ls $TDIR/results/  <- your variable stack, one output per chunk
cat $TDIR/results/*.out | python3 -c "
import sys, json, re
from collections import Counter
counts = Counter()
for line in sys.stdin:
    for m in re.finditer(r'\"label\":\s*\"([^\"]+)\"', line):
        counts[m.group(1)] += 1
for label, n in counts.most_common():
    print(f'{label}: {n}')
"
```

## Example 2: Recursive decomposition

A sub-agent receiving a chunk can decide it's still too complex and split + delegate further — it has the same tools you do. For example, given a 500-page legal document:

```bash
TDIR={{TASK_DIR}}
mkdir -p $TDIR/prompts $TDIR/results

# Pass the whole document by reference — the sub-agent gets metadata + preview,
# and can slice it with bash tools or split + delegate further
cat > $TDIR/prompts/full-doc.md << 'PROMPT'
Analyze this legal document for liability risks. For each section,
identify specific clauses that create obligations or exposure.
Return a structured summary with section headers and risk levels.
PROMPT

rlm-query $TDIR/prompts/full-doc.md $TDIR/results/analysis.out \
    --context contract.pdf.txt

# The sub-agent may internally:
#   split -l 1000 contract.pdf.txt chunks/section_
#   write prompts for each section
#   rlm-batch its own prompts to depth-2 sub-agents
#   aggregate their results
# You don't need to orchestrate this — the sub-agent handles it.

cat $TDIR/results/analysis.out
```

## Example 3: Quick fan-out for moderate contexts

When the context isn't huge, a simple split into a few big chunks works well. Sub-agents can handle a lot — don't be afraid to give each one a large piece.

```bash
TDIR={{TASK_DIR}}
mkdir -p $TDIR/chunks $TDIR/prompts $TDIR/results

# Split 100K chars into ~5 chunks of 20K each
split -l 500 report.txt $TDIR/chunks/chunk_

for chunk in $TDIR/chunks/chunk_*; do
    name=$(basename "$chunk")
    printf "Summarize the key findings in this section:\n\n$(cat $chunk)" \
        > "$TDIR/prompts/${name}.md"
done

rlm-batch $TDIR/prompts $TDIR/results

# Synthesize
cat $TDIR/results/*.out > $TDIR/all-summaries.txt
cat > $TDIR/synthesize.md << PROMPT
Combine these section summaries into a single coherent executive summary:

$(cat $TDIR/all-summaries.txt)
PROMPT
rlm-query $TDIR/synthesize.md $TDIR/final.txt --model opus
cat $TDIR/final.txt
```

## Guidance

Use sub-agents for anything requiring understanding: classification, summarization, comparison, judgment. Use code for orchestration: splitting files, counting structured results, aggregating JSON.

If you find yourself writing keyword-matching or regex classification logic, stop — that's semantic work, and it belongs in a sub-agent prompt, not a Python script. A sub-agent will understand that "Name the child left on a doorstep" is asking about a human being. No regex can reliably make that judgment.

**Choose chunk boundaries thoughtfully.** Blind line-count splitting (`split -l`) is fine for uniform data like lists of questions, but for structured documents, prefer semantic boundaries when practical:
- `csplit file '/^## /' '{*}'` — split on markdown headers
- `csplit file '/^Chapter /' '{*}'` — split on chapter markers
- `grep -n "pattern" file` then `sed -n 'start,endp'` — extract sections around known delimiters
- For code, split by file or module rather than by line count

That said, don't over-optimize chunk boundaries. A rough split that keeps related content together is better than spending many tool calls finding perfect boundaries. If the data is uniform (logs, Q&A lists, CSV rows), `split -l` is the right tool.

Be deliberate about chunk sizes and fan-out. A sub-agent can comfortably handle 50-100KB of text in a single pass, so don't split a 75KB file into 40 tiny pieces. Aim for a modest number of meaningful chunks (5-15 is typical) rather than hundreds of trivial ones.

**Write clear prompts for your sub-agents.** Specify the exact output format you need (e.g. "one label per line, nothing else"). Sub-agents print their output to stdout, which gets captured to a result file. Make sure the output is easy to aggregate programmatically.

## Tools for examining large files
- `wc -l file` / `wc -c file` — line/byte counts
- `head -n 100 file` / `tail -n 100 file` — first/last lines
- `sed -n '500,600p' file` — extract line range
- `grep -n "pattern" file` — search with line numbers
- `split -l 500 file {{TASK_DIR}}/chunks/chunk_` — split into chunks
- `csplit file '/^## /' '{*}'` — split by delimiter/pattern

## Rules
- All working files go in: {{TASK_DIR}}/
- Do NOT modify files outside this directory unless the task explicitly requires it.
- Do NOT read large context files entirely into your conversation. Use bash tools to slice them.
- Print your final answer to stdout. Do not write it to a file unless explicitly asked.

## Model selection
Choose the model based on what the agent needs to do:
- **opus** — planning, decomposition, judgment calls, complex aggregation
- **sonnet** — the default for most tasks: classification, extraction, summarization, leaf work
```bash
rlm-batch $TDIR/prompts $TDIR/results --model sonnet   # leaf: classify, extract, label
rlm-query $TDIR/reduce.md $TDIR/final.txt --model opus  # aggregation: needs judgment
```

## More Patterns

### Filter-then-process
```bash
grep -l "keyword" $TDIR/chunks/* | while read chunk; do
    name=$(basename "$chunk")
    printf "Analyze:\n\n$(cat $chunk)" > "$TDIR/prompts/${name}.md"
done
rlm-batch $TDIR/prompts $TDIR/results
```

### Pass context by reference
```bash
# Sub-agent gets file path, byte/line counts, 500-char preview
printf "Find all dates mentioned chronologically." > $TDIR/task.md
rlm-query $TDIR/task.md $TDIR/dates.out --context big-doc.txt
```

## Task Workspaces

Each `--task NAME` creates a workspace under `.rlm/<name>/`. Sub-agents inherit the task name.

```
.rlm/<task>/
  chunks/          # input variables (split context)
  prompts/         # function calls (prompt files for sub-agents)
  results/         # output variables (sub-agent results)
  sessions/        # execution logs
  system-prompt.md # optional task-specific sub-agent instructions
```

## Configuration

Environment variables (auto-inherited through recursion):
- `RLM_MAX_DEPTH` — max recursion depth (default: 3)
- `RLM_MAX_PARALLEL` — max concurrent Claude instances across the entire task tree (default: 15)
- `RLM_TIMEOUT` — timeout per sub-query in seconds (default: 1200)
- `RLM_MODEL` — model for sub-agents (default: `opus`)
- `RLM_TASK` — task name (auto-inherited from `--task`)

## Debugging

```bash
# Monitor live sub-agents
tmux list-sessions | grep rlm                 # list active sub-agents
tmux attach -t rlm-<task>-d1-…                # watch one work in real time

# Inspect after the fact
ls .rlm/<task>/sessions/                      # list all sessions
cat .rlm/<task>/sessions/rlm-d0-*/stderr.log  # stderr from a depth-0 agent
cat .rlm/<task>/sessions/rlm-d0-*/prompt.md   # prompt a sub-agent received
```
