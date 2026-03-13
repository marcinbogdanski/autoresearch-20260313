# autoresearch

The idea: give an AI agent a small but real LLM training setup and let it experiment autonomously overnight. It modifies the code, trains for 5 minutes, checks if the result improved, keeps or discards, and repeats. You wake up in the morning to a log of experiments and (hopefully) a better model. The training code here is a simplified single-GPU implementation of [nanochat](https://github.com/karpathy/nanochat). The core idea is that you're not touching any of the Python files like you normally would as a researcher. Instead, you are programming the `program.md` Markdown files that provide context to the AI agents and set up your autonomous research org. The default `program.md` in this repo is intentionally kept as a bare bones baseline, though it's obvious how one would iterate on it over time to find the "research org code" that achieves the fastest research progress, how you'd add more agents to the mix, etc. A bit more context on this project is here in this [tweet](https://x.com/karpathy/status/2029701092347630069).

## How it works

The repo has three files that matter. To run an overnight series of experiments, fork this repo, squash history to a single root commit, run `prepare.py`, then let agents loose. Agents modify `train.py` and push experiment branches/commits to the fork.

- **`prepare.py`** — fixed constants, one-time data prep (downloads training data, trains a BPE tokenizer), and runtime utilities (dataloader, evaluation). Not modified.
- **`train.py`** — the single file the agent edits. Contains the full GPT model, optimizer (Muon + AdamW), and training loop. Everything is fair game: architecture, hyperparameters, optimizer, batch size, etc. **This file is edited and iterated on by the agent**.
- **`program.md`** — baseline instructions for one agent. **This file is edited and iterated on by the human**.

Before an overnight run, copy `program.md` into a gitignored `CLAUDE.md` (or `AGENTS.md`). This way, when the agent checks out different commits, the instructions file stays constant.

By design, training runs for a **fixed 5-minute time budget** (wall clock, excluding startup/compilation), regardless of the details of your compute. The metric is **val_bpb** (validation bits per byte) — lower is better, and vocab-size-independent so architectural changes are fairly compared.

## Quick start

```bash
# 1. Install dependencies
uv sync

# 2. Download data and train tokenizer (one-time, ~2 min)
uv run prepare.py

# 3. Manually run a single training experiment (~5 min)
uv run train.py
```

If the above commands all work ok, your setup is working and you can go into autonomous research mode.

## Running the agent

Copy `program.md` into `CLAUDE.md` (or `AGENTS.md` depending on agent). Then spin up your Claude/Codex or whatever you want in this repo (and disable all permissions), and prompt something like:

```
Hi have a look at CLAUDE.md and let's kick off a new experiment! let's do the setup first.
```

## Project structure

```
prepare.py      — constants, data prep + runtime utilities (do not modify)
train.py        — model, optimizer, training loop (agent modifies this)
program.md      — agent instructions (human iterates on this)
pyproject.toml  — dependencies
```

## Design choices

- **Single file to modify.** The agent only touches `train.py`. This keeps the scope manageable and diffs reviewable.
- **Fixed time budget.** Training always runs for exactly 5 minutes, regardless of your specific platform. This means you can expect approx 12 experiments/hour and approx 100 experiments while you sleep. There are two upsides of this design decision. First, this makes experiments directly comparable regardless of what the agent changes (model size, batch size, architecture, etc). Second, this means that autoresearch will find the most optimal model for your platform in that time budget. The downside is that your runs (and results) become not comparable to other people running on other compute platforms.
- **Self-contained.** No external dependencies beyond PyTorch and a few small packages. No distributed training, no complex configs. One GPU, one file, one metric.

## Platform support

This code currently requires that you have one or more NVIDIA GPUs. Currently assume one agent per GPU, but in principle an agent could command multiple GPUs and run parallel experiments in the background. This is something that should be defined in `program.md`. On multi-GPU systems be sure to instruct an agent clearly which GPU they should use, so multiple agents on the same machine don't interfere with each other.
