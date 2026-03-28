# autoresearch

This is a generic framework for autonomous AI-driven experimentation. The agent iterates on ideas, evaluates them against a fixed harness, and keeps what works.

## Architecture

The framework is built around two files:

| File | Role | Modifiable? |
|---|---|---|
| `agent_cannot_modify_this.py` | **Evaluation harness** — defines how to measure experiment quality. Exposes an `evaluate()` function that returns a numeric score. | ❌ Read-only |
| `agent_can_modify_this.py` | **Experiment code** — where you generate and implement ideas. Must `import agent_cannot_modify_this` and call its evaluation. | ✅ Freely editable |

**The contract**: `agent_can_modify_this.py` is the only file you edit. It must import and use the evaluation function from `agent_cannot_modify_this.py` to score each experiment. The evaluation harness is the source of truth — you cannot change how scoring works.

## Setup

To set up a new experiment run, work with the user to:

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `mar28`). The branch `autoresearch/<tag>` must not already exist — this is a fresh run.
2. **Create the branch**: `git checkout -b autoresearch/<tag>` from the current main branch.
3. **Read the in-scope files**: Read these files for full context:
   - `README.md` — repository context.
   - `agent_cannot_modify_this.py` — the fixed evaluation harness. Do not modify.
   - `agent_can_modify_this.py` — the file you modify. Your experiment code lives here.
4. **Initialize results.tsv**: Create `results.tsv` with just the header row. The baseline will be recorded after the first run.
5. **Confirm and go**: Confirm setup looks good.

Once you get confirmation, kick off the experimentation.

## Experimentation

Each experiment runs for a **fixed time budget of 5 minutes** (wall clock). You launch it as: `uv run agent_can_modify_this.py`.

**What you CAN do:**
- Modify `agent_can_modify_this.py` — this is the only file you edit. Everything is fair game: algorithms, parameters, data structures, approach, strategy, etc.

**What you CANNOT do:**
- Modify `agent_cannot_modify_this.py`. It is read-only. It contains the fixed evaluation logic.
- Create new files. You may only modify `agent_can_modify_this.py` — no additional scripts, modules, or data files.
- Install new packages or add dependencies. You can only use what's already in `pyproject.toml`.
- Modify the evaluation harness. The `evaluate()` function in `agent_cannot_modify_this.py` is the ground truth metric.

**The goal is simple: get the best score** (as defined by the evaluation harness). Since the time budget is fixed at 5 minutes, you don't need to worry about runtime — it's always capped. Everything else is fair game.

**Simplicity criterion**: All else being equal, simpler is better. A small improvement that adds ugly complexity is not worth it. Conversely, removing something and getting equal or better results is a great outcome — that's a simplification win. When evaluating whether to keep a change, weigh the complexity cost against the improvement magnitude.

**The first run**: Your very first run should always be to establish the baseline, so you will run the experiment code as-is.

## Output format

The experiment script (`agent_can_modify_this.py`) should print a summary when it finishes. At minimum it must print the score:

```
score: <value>
```

You can extract the key metric from the log file:

```
grep "^score:" run.log
```

## Logging results

When an experiment is done, log it to `results.tsv` (tab-separated, NOT comma-separated — commas break in descriptions).

The TSV has a header row and 4 columns:

```
commit	score	status	description
```

1. git commit hash (short, 7 chars)
2. score achieved (e.g. 0.1234) — use 0.0 for crashes
3. status: `keep`, `discard`, or `crash`
4. short text description of what this experiment tried

Example:

```
commit	score	status	description
a1b2c3d	0.8500	keep	baseline
b2c3d4e	0.8720	keep	increase learning rate
c3d4e5f	0.8400	discard	switch activation function
d4e5f6g	0.0000	crash	invalid approach (runtime error)
```

## The experiment loop

The experiment runs on a dedicated branch (e.g. `autoresearch/mar28`).

LOOP FOREVER:

1. Look at the git state: the current branch/commit we're on
2. Modify `agent_can_modify_this.py` with an experimental idea.
3. git commit
4. Run the experiment: `uv run agent_can_modify_this.py > run.log 2>&1` (redirect everything — do NOT use tee or let output flood your context)
5. Read out the results: `grep "^score:" run.log`
6. If the grep output is empty, the run crashed. Run `tail -n 50 run.log` to read the Python stack trace and attempt a fix. If you can't get things to work after more than a few attempts, give up.
7. Record the results in the tsv (NOTE: do not commit the results.tsv file, leave it untracked by git)
8. If score improved, you "advance" the branch, keeping the git commit
9. If score is equal or worse, you git reset back to where you started

The idea is that you are a completely autonomous researcher trying things out. If they work, keep. If they don't, discard. And you're advancing the branch so that you can iterate. If you feel like you're getting stuck in some way, you can rewind but you should probably do this very very sparingly (if ever).

**Timeout**: Each experiment should take ~5 minutes total (+ a few seconds for startup overhead). If a run exceeds 10 minutes, kill it and treat it as a failure (discard and revert).

**Crashes**: If a run crashes, use your judgment: If it's something dumb and easy to fix (e.g. a typo, a missing import), fix it and re-run. If the idea itself is fundamentally broken, just skip it, log "crash" as the status in the tsv, and move on.

**NEVER STOP**: Once the experiment loop has begun (after the initial setup), do NOT pause to ask the human if you should continue. Do NOT ask "should I keep going?" or "is this a good stopping point?". The human might be asleep, or gone from a computer and expects you to continue working *indefinitely* until you are manually stopped. You are autonomous. If you run out of ideas, think harder — re-read the in-scope files for new angles, try combining previous near-misses, try more radical changes. The loop runs until the human interrupts you, period.

As an example use case, a user might leave you running while they sleep. If each experiment takes you ~5 minutes then you can run approx 12/hour. The user then wakes up to experimental results, all completed by you while they slept!
