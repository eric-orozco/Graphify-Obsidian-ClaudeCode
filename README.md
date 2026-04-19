# GRAPHIFY + Obsidian <br> (Setup & Reference for Claude Code)

> Complete guide for setting up the graphify knowledge graph pipeline on any project. Covers what each piece does, how they work together, and how to rebuild from scratch on a new machine.

---

## Table of Contents

- [What Is graphify?](#what-is-graphify)
  - [Outputs](#outputs)
- [The Three Moving Parts](#the-three-moving-parts)
  - [1. `.graphifyignore` — Corpus Filter](#1-graphifyignore--corpus-filter)
  - [2. Git Post-Commit Hook — Free AST Updates on Every Commit](#2-git-post-commit-hook--free-ast-updates-on-every-commit)
  - [3. Claude Code Stop Hook — Automatic Obsidian Refresh](#3-claude-code-stop-hook--automatic-obsidian-refresh)
- [What's Automatic vs Manual](#whats-automatic-vs-manual)
- [How graphify Works as Memory for Claude Code](#how-graphify-works-as-memory-for-claude-code)
  - [The core problem graphify solves](#the-core-problem-graphify-solves)
  - [What "memory" means here](#what-memory-means-here)
  - [Token impact](#token-impact)
  - [Example: refactoring a monolithic React component](#example-refactoring-a-monolithic-react-component)
  - [Querying the graph directly](#querying-the-graph-directly)
- [Environment & PATH Requirements](#environment--path-requirements)
  - [Confirm what's currently on PATH](#confirm-whats-currently-on-path)
  - [Configure PATH — Windows PowerShell](#configure-path--windows-powershell)
  - [Configure PATH — macOS](#configure-path--macos)
  - [Environment variables](#environment-variables)
- [Full Setup From Scratch](#full-setup-from-scratch)
  - [Prerequisites: install graphify](#prerequisites-install-graphify)
  - [Step 1: Confirm `.graphifyignore` is present](#step-1-confirm-graphifyignore-is-present)
  - [Step 2: Run the initial full pipeline](#step-2-run-the-initial-full-pipeline)
  - [Step 3: Install the git hook](#step-3-install-the-git-hook)
  - [Step 4: Configure the Claude Code Stop hook](#step-4-configure-the-claude-code-stop-hook)
  - [Step 5: Confirm everything is working end-to-end](#step-5-confirm-everything-is-working-end-to-end)
- [Incremental Updates](#incremental-updates)
- [Troubleshooting](#troubleshooting)

---

## What Is graphify?

graphify reads a folder of source files and builds a **knowledge graph** — a network of nodes (functions, components, concepts, files) connected by edges (imports, calls, references, shared data). Once the graph exists, you can query it by concept, trace a path between two ideas, or browse it visually in Obsidian.

It does this in two passes:

| Pass | What it does | Cost |
|------|-------------|------|
| **AST extraction** | Parses code files (`.ts`, `.js`, `.py`, etc.) using static analysis to find imports, exports, and function calls. No AI involved — purely deterministic. | Free — runs offline |
| **Semantic extraction** | Sends non-code files (docs, images, markdown) to Claude to extract concepts and relationships that can't be found by parsing alone. | Uses Claude tokens |

The two passes are complementary. AST gives you the precise structural skeleton of the codebase. Semantic extraction gives you the meaning layer on top — what things are *for*, not just what they *call*.

### Outputs

Everything graphify writes goes into `graphify-out/` inside the project root:

| File / folder | What it is |
|---------------|-----------|
| `graph.json` | The full graph: every node and edge. This is the source of truth — all other outputs are generated from it. |
| `graph.html` | A self-contained interactive visualization. Open it in any browser — no server needed. |
| `GRAPH_REPORT.md` | A plain-English audit report covering: the most-connected nodes ("god nodes"), surprising cross-community connections, and suggested questions to explore. |
| `cache/` | A file-by-file cache of semantic extraction results, keyed by content hash. Prevents re-spending Claude tokens on files that haven't changed since the last run. |

The **Obsidian vault** is a separate export — graphify reads `graph.json` and writes one `.md` note per graph node, plus a `graph.canvas` file that lays out all communities as named groups. The vault is regenerated from scratch on each export, so it always reflects the current graph.

---

## The Three Moving Parts

This project uses three components working in sequence to keep the graph and Obsidian vault current without manual intervention.

### 1. `.graphifyignore` — Corpus Filter

**File:** `.graphifyignore` in the project root

**What it does:** Tells graphify which paths to skip when scanning for files to extract. Works identically to `.gitignore` — glob patterns, one per line, comments with `#`. Without this file, graphify counts every file it can find, including third-party vendor assets, build artifacts, and test fixtures that have no semantic value for graph extraction.

**Why it matters:** graphify has a built-in safety threshold — if it detects more than 200 files in the corpus, it prints a large-corpus warning and pauses to ask which subfolder to run on instead. This is intentional: large unfiltered corpora waste tokens on vendored or generated files. The `.graphifyignore` prevents those files from being counted in the first place, so the pipeline runs to completion without interruption.

**Rule for deciding what to ignore:** exclude anything that isn't source code or documentation *you wrote*. Common candidates:
- Third-party vendor directories (`node_modules/`, `public/vendor/`, any pre-built asset bundle)
- Build output (`.next/`, `dist/`, `build/`, `out/`)
- graphify's own output (`graphify-out/`) — otherwise it would try to extract its own reports
- Test artifacts (`e2e/test-results/`, `playwright-report/`, snapshot directories)

**Example `.graphifyignore` for a Next.js project with ArcGIS vendor assets:**

```
# Third-party vendor bundle — pre-built, no semantic value
public/arcgis-assets/

# Build and tool output
.next/
graphify-out/
node_modules/

# Test artifacts
e2e/snapshots/
e2e/test-results/
playwright-report/
```

Adapt the paths to match your project structure. Once the file exists in the project root, it is committed to git and travels with the repository to any new machine automatically.

---

### 2. Git Post-Commit Hook — Free AST Updates on Every Commit

**Installed by:** running `graphify hook install` once from the project root

**What it does:** After every `git commit`, this hook asks git which files changed in that commit, runs AST extraction on only those files, and merges the results back into `graphify-out/graph.json`. Because it only processes changed code files and never calls the LLM, it costs nothing and finishes in a few seconds.

**Where it lives:** graphify appends its hook block to the project's git hook file (typically `.husky/_/post-commit` for projects using Husky, or `.git/hooks/post-commit` otherwise). It does not replace the file — any hooks that were already there continue to work.

**What it cannot do:** it only handles code files. Markdown files, images, and other non-code files that were added or changed in a commit are not picked up — those need a manual `/graphify . --update` run to re-extract semantically.

**Management commands:**

```bash
graphify hook status      # prints whether the hook is installed
graphify hook install     # installs (safe to run more than once)
graphify hook uninstall   # removes the graphify block from the hook file
```

---

### 3. Claude Code Stop Hook — Automatic Obsidian Refresh

**File:** `.claude/settings.json` inside the project

**What it does:** Every time a Claude Code session ends — whether you typed something, ran a command, or just opened and closed the project — this hook runs a Python script that reads the current `graph.json`, re-runs community detection (clustering), and rewrites the entire Obsidian vault. The script takes roughly 1–2 seconds regardless of graph size.

**Importantly, this hook does not re-extract anything.** It only reshapes and re-exports whatever is already in `graph.json`. Think of it as a "sync to Obsidian" step, not a "rebuild the graph" step. The graph content itself is only ever changed by the AST hook (on commit) or a manual `/graphify` run.

**What you see when the session ends:**

```
graphify: Obsidian vault refreshed (N notes)
```

**The Python script the hook runs (shown here for clarity — it is already configured in `.claude/settings.json`):**

```python
import json, sys
from graphify.export import to_obsidian, to_canvas
from graphify.cluster import cluster
from networkx.readwrite import json_graph
from pathlib import Path

graph_path = Path('graphify-out/graph.json')
if not graph_path.exists():
    sys.exit(0)  # no graph yet — skip silently

data = json.loads(graph_path.read_text())
G = json_graph.node_link_graph(data, edges='links')
communities = cluster(G)
labels = {cid: f'Community {cid}' for cid in communities}

obsidian_dir = 'YOUR_VAULT_PATH_HERE'  # absolute path to your Obsidian vault folder
n = to_obsidian(G, communities, obsidian_dir, community_labels=labels)
to_canvas(G, communities, f'{obsidian_dir}/graph.canvas', community_labels=labels)
print(json.dumps({'systemMessage': f'graphify: Obsidian vault refreshed ({n} notes)'}))
```

---

## What's Automatic vs Manual

| What happened | What updates automatically | What you need to do manually |
|---------------|---------------------------|------------------------------|
| You committed code changes | `graph.json` is updated (AST re-extraction of changed files) | Nothing |
| A Claude Code session ended | Obsidian vault is rewritten from current `graph.json` | Nothing |
| You added new `.md` or image files | Nothing — the git hook skips non-code files | Run `/graphify . --update --obsidian --obsidian-dir <path>` |
| You want to pick up new semantic relationships | Nothing automatic | Run `/graphify . --update --obsidian --obsidian-dir <path>` |
| First time setup on a new machine | Nothing | Follow all five steps in the setup section below |

---

## Environment & PATH Requirements

graphify needs two programs available on your terminal's PATH: the `python` (or `python3`) executable, and the Python Scripts directory where `pip` installs command-line tools like `graphify`. On Windows these are not added to PATH by default during Python installation unless you check the option explicitly.

### Confirm what's currently on PATH

Run these in a new terminal window to see what your machine resolves:

```powershell
# Windows PowerShell
python --version          # should print Python 3.x.x
graphify --help           # should print graphify command list
```

```bash
# macOS terminal
python3 --version
graphify --help
```

If either command prints "not found" or "not recognized", follow the instructions for your OS below.

---

### Configure PATH — Windows PowerShell

Python installs in two places on Windows: the Python base directory (contains `python.exe`) and a user-level Scripts directory (where `pip install` puts CLI tools like `graphify.exe`). Both need to be on PATH.

First, find your actual paths:

```powershell
# Find where python.exe lives
(Get-Command python).Source

# Find where graphify.exe was installed after running pip install
pip show graphifyy | Select-String "Location"
# The Scripts directory is the same folder but named "Scripts" instead of "site-packages"
# Example: if Location is C:\Users\YourName\AppData\Roaming\Python\Python3xx\site-packages
#          then Scripts is C:\Users\YourName\AppData\Roaming\Python\Python3xx\Scripts
```

Then add both paths permanently (survives reboots):

```powershell
# Replace these two values with the actual paths from the commands above
# Replace these with the actual paths from the commands above
$pyBase    = "C:\Program Files\PythonXXX"           # e.g. C:\Program Files\Python312
$pyScripts = "$env:APPDATA\Python\PythonXXX\Scripts" # e.g. ...\Python312\Scripts

# Read the current user PATH, add any missing entries, write it back
$current = [Environment]::GetEnvironmentVariable("PATH", "User")
$additions = @($pyBase, $pyScripts) | Where-Object { $current -notlike "*$_*" }
if ($additions) {
    [Environment]::SetEnvironmentVariable(
        "PATH",
        ($current + ";" + ($additions -join ";")),
        "User"
    )
    Write-Host "Added to PATH: $($additions -join ', ')"
} else {
    Write-Host "Both paths are already in PATH."
}

# Apply to the current terminal session without restarting
$env:PATH = [Environment]::GetEnvironmentVariable("PATH","Machine") + ";" +
            [Environment]::GetEnvironmentVariable("PATH","User")

# Confirm it worked
python --version
graphify --help
```

After running the two discovery commands above, note down your paths here for reference — they will differ by machine and Python version.

---

### Configure PATH — macOS

Python and pip install locations vary depending on how Python was installed. Find which case applies to your machine:

```bash
# Step 1: find out where python3 comes from
which python3

# Step 2: find where pip installs CLI tools
python3 -c "import sysconfig; print(sysconfig.get_path('scripts'))"
```

Common results and what they mean:

| `which python3` output | Installation method | Scripts directory |
|------------------------|--------------------|--------------------|
| `/opt/homebrew/bin/python3` | Homebrew (Apple Silicon) | `$(brew --prefix)/bin` |
| `/usr/local/bin/python3` | Homebrew (Intel) | `/usr/local/bin` |
| `/usr/bin/python3` | macOS system Python | `~/Library/Python/3.x/bin` |
| `~/.pyenv/shims/python3` | pyenv | depends on active version |

Add the Scripts directory to your shell config. Open `~/.zshrc` (or `~/.bash_profile` if using bash) and add the appropriate block:

```bash
# ── Homebrew Python (Apple Silicon M1/M2/M3) ─────────────────────────────────
export PATH="/opt/homebrew/bin:$PATH"

# ── Homebrew Python (Intel Mac) ──────────────────────────────────────────────
# export PATH="/usr/local/bin:$PATH"

# ── macOS system Python or pip --user installs ───────────────────────────────
# Replace 3.x with your actual Python version (e.g. 3.11, 3.12)
# export PATH="$HOME/Library/Python/3.x/bin:$PATH"

# ── pyenv ────────────────────────────────────────────────────────────────────
# export PYENV_ROOT="$HOME/.pyenv"
# export PATH="$PYENV_ROOT/bin:$PATH"
# eval "$(pyenv init -)"
```

Apply without restarting your terminal:

```bash
source ~/.zshrc     # or source ~/.bash_profile

# Confirm both work
python3 --version
graphify --help
```

---

### Environment variables

graphify reads a small set of environment variables. None are required for normal use — they only apply in specific situations.

| Variable | When it applies | How to set it |
|----------|----------------|---------------|
| `GRAPHIFY_PYTHON` | Written automatically to `.graphify_python` during Step 1 of the pipeline. The git hook reads this file to call the same Python interpreter that built the graph. You do not set this manually — deleting `.graphify_python` and re-running `/graphify .` regenerates it. | Automatic |
| `GRAPHIFY_CHANGED` | Set internally by the git hook to pass the list of changed files to the Python rebuild script. Never set this yourself. | Internal only |
| `GRAPHIFY_WHISPER_MODEL` | Only relevant if you have video or audio files in the project. Controls which Whisper model is used for transcription. Larger models are more accurate but slower. Default: `base`. Options: `tiny`, `base`, `small`, `medium`, `large`. | Optional |
| `GRAPHIFY_WHISPER_PROMPT` | Only relevant for video/audio transcription. A short sentence that seeds Whisper with domain context, improving punctuation and terminology in the transcript. | Optional |

To set an optional variable for the current terminal session only:

```powershell
# Windows PowerShell — applies until you close this terminal window
$env:GRAPHIFY_WHISPER_MODEL = "small"
```

```bash
# macOS — applies until you close this terminal window
export GRAPHIFY_WHISPER_MODEL="small"
```

---

## Full Setup From Scratch

Follow these steps in order when setting up on a new machine or after the project has been wiped.

### Prerequisites: install graphify

The package name on PyPI is `graphifyy` — note the two y's. The name `graphify` on PyPI is an unrelated package and will not work.

```powershell
# Windows PowerShell
pip install graphifyy

# Confirm the import works and the CLI is reachable
python -c "import graphify; print('import OK')"
graphify --help
```

```bash
# macOS
pip3 install graphifyy

python3 -c "import graphify; print('import OK')"
graphify --help
```

If `graphify --help` fails with "command not found" even though `pip install` succeeded, the Scripts directory is not on PATH. Follow the PATH setup section above before continuing.

---

### Step 1: Confirm `.graphifyignore` is present

This file is committed to the repository so it will exist on any fresh clone. Its job is to exclude vendor assets and build artifacts from the corpus count so graphify doesn't trigger the large-corpus warning.

To confirm it is working, run graphify's detection step and check the output:

```powershell
# Windows PowerShell — run from the project root
python -c "
from graphify.detect import detect
from pathlib import Path
r = detect(Path('.'))
print('Total files detected:', r['total_files'])
print('Warning:', r.get('warning') or 'None')
"
```

```bash
# macOS — run from the project root
python3 -c "
from graphify.detect import detect
from pathlib import Path
r = detect(Path('.'))
print('Total files detected:', r['total_files'])
print('Warning:', r.get('warning') or 'None')
"
```

**Reading the output:**
- `Warning: None` — the ignore rules are working, proceed to Step 2.
- `Warning: Large corpus: N files...` — the `.graphifyignore` is either missing or not in the right location. The file must be in the project root (the same folder as `package.json`). If the count is unexpectedly high, look at which directories dominate and add them to `.graphifyignore`.

---

### Step 2: Run the initial full pipeline

This is the one step that costs Claude tokens. It runs both extraction passes and builds the Obsidian vault for the first time. You only need to do this once per machine — after this, the hooks keep things current automatically.

Open the project in Claude Code (make sure you are in the project root directory), then run:

```
/graphify . --obsidian --obsidian-dir <path-to-obsidian-vault>
```

Replace `<path-to-obsidian-vault>` with the absolute path to wherever Obsidian stores your vaults:

```
# Windows example — use forward slashes even on Windows
/graphify . --obsidian --obsidian-dir C:/Users/YourName/Projects/Obsidian/your-vault

# macOS example
/graphify . --obsidian --obsidian-dir ~/Documents/Obsidian/your-vault
```

What happens during this run:
1. graphify scans the project and counts files (respecting `.graphifyignore`)
2. AST extraction runs on all code files — free, takes seconds
3. Semantic extraction runs via parallel Claude subagents on docs and images — this is where token cost occurs and where most of the time is spent (typically 2–5 minutes depending on corpus size)
4. The graph is built, communities are detected, and `GRAPH_REPORT.md` is written
5. The Obsidian vault is written: one `.md` note per graph node, plus `graph.canvas`

When it finishes, `graphify-out/graph.json` contains the full graph and the Obsidian vault is ready to open.

---

### Step 3: Install the git hook

Run once from the project root:

```powershell
# Windows PowerShell
graphify hook install
graphify hook status    # confirm output reads: post-commit: installed
```

```bash
# macOS
graphify hook install
graphify hook status
```

This appends a block to the project's git hook file. If that file already has other hooks inside it, they are preserved — graphify only adds its own block and does not touch the rest.

After installation, every `git commit` automatically triggers an AST re-extraction of whichever code files changed in that commit and merges the results into `graph.json`. No action required from you.

---

### Step 4: Configure the Claude Code Stop hook

This hook tells Claude Code to refresh the Obsidian vault at the end of every session. It is already configured in `.claude/settings.json` for this project — if you cloned the repo, it is already there.

**If the hook is missing** (e.g. you are setting up a different project), add this `Stop` block to the `hooks` section of `.claude/settings.json`. Merge it with any existing hooks — do not replace the whole file:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python -c \"\nimport json, sys\nfrom graphify.export import to_obsidian, to_canvas\nfrom graphify.cluster import cluster\nfrom networkx.readwrite import json_graph\nfrom pathlib import Path\ngraph_path = Path('graphify-out/graph.json')\nif not graph_path.exists(): sys.exit(0)\ndata = json.loads(graph_path.read_text())\nG = json_graph.node_link_graph(data, edges='links')\ncommunities = cluster(G)\nlabels = {cid: f'Community {cid}' for cid in communities}\nobsidian_dir = 'YOUR_VAULT_PATH_HERE'\nn = to_obsidian(G, communities, obsidian_dir, community_labels=labels)\nto_canvas(G, communities, f'{obsidian_dir}/graph.canvas', community_labels=labels)\nprint(json.dumps({'systemMessage': f'graphify: Obsidian vault refreshed ({n} notes)'}))\n\"",
            "timeout": 60,
            "statusMessage": "Refreshing Obsidian vault..."
          }
        ]
      }
    ]
  }
}
```

Replace `YOUR_VAULT_PATH_HERE` with the absolute path to your Obsidian vault folder. Use forward slashes on both platforms (e.g. `C:/Users/you/Projects/Obsidian/my-vault` on Windows, `/Users/you/Documents/Obsidian/my-vault` on macOS).

After editing, confirm the JSON is still valid (a single misplaced comma or bracket breaks the entire settings file silently):

```powershell
# Windows PowerShell
python -c "import json; json.load(open('.claude/settings.json')); print('JSON is valid')"
```

```bash
# macOS
python3 -c "import json; json.load(open('.claude/settings.json')); print('JSON is valid')"
```

---

### Step 5: Confirm everything is working end-to-end

**Check the graph loaded correctly:**

```powershell
# Windows PowerShell
python -c "
from networkx.readwrite import json_graph
import json
from pathlib import Path
data = json.loads(Path('graphify-out/graph.json').read_text())
G = json_graph.node_link_graph(data, edges='links')
print(f'Graph loaded: {G.number_of_nodes()} nodes, {G.number_of_edges()} edges')
"
```

```bash
# macOS
python3 -c "
from networkx.readwrite import json_graph
import json
from pathlib import Path
data = json.loads(Path('graphify-out/graph.json').read_text())
G = json_graph.node_link_graph(data, edges='links')
print(f'Graph loaded: {G.number_of_nodes()} nodes, {G.number_of_edges()} edges')
"
```

A non-zero node and edge count means the graph is intact.

**Check Obsidian:**
1. Open Obsidian → click "Open folder as vault" → select your vault directory
2. Open `graph.canvas` — you should see nodes laid out in community groups
3. Individual note files appear in the file list on the left

**Check the git hook:**
Make a trivial change to any source file (e.g. add a blank comment), commit it, and watch the terminal output. You should see: `[graphify hook] N file(s) changed - rebuilding graph...`

**Check the Stop hook:**
End the current Claude Code session. After Claude finishes, look for the message: `graphify: Obsidian vault refreshed (N notes)`

---

## How graphify Works as Memory for Claude Code

### The core problem graphify solves

Claude Code has a context window — a fixed amount of text it can hold in working memory at once. On a large project, that budget gets spent fast. Every file Claude reads to understand the codebase is text that displaces something else. If a task requires understanding how many files relate to each other, Claude may spend the majority of its context budget just *reading* before it can start *thinking*.

Without graphify, Claude's default strategy for any architectural question is to search broadly and read deeply: run a Glob to find candidate files, open several of them, read imports, follow those to more files, and piece together the structure incrementally. This works, but it is expensive in tokens and slow — and because Claude does not retain anything between sessions, it starts from scratch every time.

**graphify gives Claude a pre-computed map of the entire codebase.** Instead of exploring the territory to answer a question, Claude reads the map first, identifies exactly which files are relevant, and reads only those. The graph encodes relationships that would otherwise cost tokens to rediscover: which components import which, which functions share data structures, which files are central enough to affect everything else.

### What "memory" means here

graphify is not conversational memory — it does not remember what you said last session. It is *structural memory*: a persistent, queryable record of how every part of the codebase connects to every other part. That record survives across sessions, is updated incrementally on each commit, and is always available for Claude to read without re-scanning any source files.

Three artifacts make up this memory layer:

| Artifact | What it tells Claude |
|----------|---------------------|
| `graphify-out/GRAPH_REPORT.md` | Which nodes are most connected ("god nodes"), which communities exist, what surprising cross-cutting connections were found, and suggested questions to explore. This is the executive summary — Claude reads it first. |
| `graphify-out/graph.json` | The full graph with every node, edge, confidence score, and source location. Claude uses this for precise queries: "what calls this function", "what does this module export", "shortest path between these two concepts". |
| Obsidian vault notes | One note per node, with backlinks to connected nodes. Useful for browsing the graph manually or for following a trail of connections across community boundaries. |

The `PreToolUse` hook configured in `.claude/settings.json` puts this memory to work automatically: whenever Claude is about to run a file search (Glob or Grep), the hook injects a reminder to check `GRAPH_REPORT.md` first. Claude sees the graph structure before it reaches for the filesystem.

### Token impact

The savings come from reading fewer files. File reads are the primary source of context consumption on large codebases — every file opened adds its full content to the window.

| Question Claude is answering | Without graphify | With graphify |
|-----------------------------|-----------------|---------------|
| "What imports this component?" | Search for the filename, open each match, scan for the import line — 5–10 files read | One graph query — 0 files read |
| "What will break if I change this function's signature?" | Read the file, trace imports manually, read each caller — potentially 15+ files | Read the god-nodes section of GRAPH_REPORT.md, identify callers from graph edges, read only the callers that matter |
| "How does data flow from the API to the UI?" | Open the API route, follow imports, open each intermediate file — 8–12 files, often circling back | BFS traversal from the API node — returns a subgraph with source locations, Claude reads only the 2–3 pivotal files |
| Understanding a module's purpose before editing | Read the whole file, read its tests, read its callers | Read the node's entry in GRAPH_REPORT.md plus its edge list — one small, structured read |

The practical effect is that Claude can address more complex tasks within a single session without exhausting the context budget. A refactor that would previously require multiple sessions — because the context filled up during the exploration phase — can often be completed in one.

---

### Example: refactoring a monolithic React component

This is a concrete walkthrough of how the token economics play out on a realistic task.

**The situation:** `Dashboard.tsx` has grown to 850 lines. It fetches its own data, manages a dozen pieces of local state, renders four distinct panel sections, and handles its own event logic. It was written as one file because it started small — but it is now difficult to test, slow to read, and impossible to reuse any part of independently. You want to break it into focused, composable components.

---

**Without graphify — what Claude does:**

Before writing anything, Claude needs to understand the full blast radius. That requires reading:

1. `Dashboard.tsx` in full — 850 lines, roughly 17,000 tokens of context consumed
2. Every file that imports Dashboard — to understand what the outside world expects from its API (props, exports, forwarded refs)
3. Every file Dashboard imports — to understand which dependencies are truly internal vs. shared
4. The test file — to understand which behaviors are under test and must not change
5. Any context providers or global state Dashboard reads — to understand what it silently depends on

After all that reading, Claude has consumed somewhere between 40,000 and 80,000 tokens of its context window *before writing a single line*. On a session with complex follow-up questions, this is the difference between finishing in one session and running out of room halfway through.

Claude also has no persistent record of what it discovered. The next session starts from scratch.

---

**With graphify — what Claude does:**

Before opening any source file, Claude reads `GRAPH_REPORT.md`. That report contains a section called "God Nodes" — the most-connected nodes in the codebase. It might read:

```
God Nodes (sorted by degree):
  Dashboard (degree 31) — connected to: useAuth, useFeatureFlags, ApiClient,
    ReportingPanel, MetricsPanel, ActivityFeed, SettingsPanel, useLocalStorage,
    chartConfig, exportUtils ... and 21 more
```

In one small read, Claude knows:
- Dashboard has 31 connections — it is a central node, changes will ripple outward
- It connects to four named panel components — those are the natural split boundaries
- It connects to `useAuth` and `useFeatureFlags` — shared context, not local state to be moved
- It connects to `exportUtils` and `chartConfig` — utility imports, straightforward to keep or redistribute

Claude then runs one graph query to get Dashboard's full edge list with confidence scores and source locations. This returns a structured list — no source files opened yet — that shows exactly which imports are used where inside the file and which external components reference Dashboard.

Now Claude opens `Dashboard.tsx` once, with a specific plan already formed from the graph data. It knows the split points, the shared dependencies, and the external interface before reading the first line of source code.

**Total file reads before starting the refactor: 1.** Total context consumed for the exploration phase: roughly 3,000–5,000 tokens — the graph report and the edge list, both compact and structured.

---

**During the refactor — how Claude uses the graph:**

As Claude extracts `ReportingPanel` into its own file, it queries the graph to confirm exactly what that panel needs:

```
/graphify query "ReportingPanel state dependencies"
```

The graph returns which nodes the panel's code connects to — the data shapes it reads, the callbacks it fires. Claude uses this to write the correct props interface without opening every file that touches reporting data.

When the extraction is done, Claude verifies nothing leaked:

```
/graphify path "Dashboard" "ApiClient"
```

This returns the shortest connection path between the two nodes — confirming whether ApiClient access is now properly encapsulated inside a specific child component or still passing through Dashboard directly.

---

**Net result:**

| Phase | Without graphify | With graphify |
|-------|-----------------|---------------|
| Exploration (understanding structure) | 40k–80k tokens, 10–20 file reads | 3k–5k tokens, 0–1 file reads |
| Implementation (writing code) | Same | Same |
| Verification (checking blast radius) | Another 5–10 file reads | Graph queries, 0 additional file reads |
| Next session (continuing the work) | Starts from scratch — re-reads everything | Graph is still there — reads report, resumes immediately |

The graph does not make Claude smarter. It makes Claude faster to reach the point where it can be smart, by replacing exploratory file reading with structured lookups against a pre-computed index.

---

### Querying the graph directly

Once `graphify-out/graph.json` exists, you can ask graphify questions directly in Claude Code without re-running the pipeline:

```
# Broad question — BFS traversal, surfaces all nearby connected nodes
/graphify query "how does the data fetching layer work"

# Trace a specific path — DFS traversal, follows one dependency chain deep
/graphify query "authentication flow" --dfs

# Find the shortest connection between two named concepts
/graphify path "LoginForm" "ApiClient"

# Plain-English explanation of one node and all its connections
/graphify explain "Dashboard"
```

---

## Incremental Updates

After the initial full pipeline run, use `--update` instead of running the full pipeline again. `--update` checks which files have changed since the last run (using the content-hash cache in `graphify-out/cache/`) and only re-extracts those files. For code-only changes it skips the LLM entirely.

```
# Windows — use forward slashes
/graphify . --update --obsidian --obsidian-dir C:/Users/YourName/Projects/Obsidian/your-vault

# macOS
/graphify . --update --obsidian --obsidian-dir ~/Documents/Obsidian/your-vault
```

Use `--update` when:
- You added or significantly edited markdown documentation files
- You added new images that should be in the graph
- You want the semantic layer to reflect recent changes to non-code files

You do **not** need `--update` after committing code changes — the git hook already handles that for free.

---

## Troubleshooting

| Symptom | What it means | Fix |
|---------|--------------|-----|
| `graphify: command not found` after `pip install graphifyy` | The Python Scripts directory (where pip puts CLI tools) is not on your PATH | Follow the PATH setup section for your OS above |
| `import graphify` fails or imports the wrong package | Either graphify is not installed, or the wrong `graphify` package (without the extra y) was installed | Run `pip install graphifyy` — note the **two y's**. The package named `graphify` (one y) on PyPI is an unrelated project. |
| Detection shows far more files than expected | `.graphifyignore` is missing or not in the right location | The file must be in the project root — the same directory as `package.json`. It must not be in a subdirectory. |
| Large corpus warning appears during pipeline run | Vendor or build files are being counted, pushing the total past graphify's 200-file threshold | Add the offending directories to `.graphifyignore`. Run the detection check from Step 1 to confirm the count drops. |
| Stop hook produces no message at session end | The hook is either missing from `.claude/settings.json` or the settings file has a JSON syntax error | Validate with `python -c "import json; json.load(open('.claude/settings.json'))"`. A syntax error causes Claude Code to silently ignore the entire file. Also try opening `/hooks` in Claude Code to force a reload. |
| Obsidian vault notes are out of date | The Stop hook ran but `graph.json` itself wasn't updated (the hook re-exports existing data — it does not re-extract) | If you added non-code files, run `/graphify . --update --obsidian --obsidian-dir <path>` to re-extract and refresh |
| Git hook does not run after commit | The hook was not installed or was removed | Run `graphify hook status`. If it shows uninstalled, run `graphify hook install` from the project root. |
| Git hook uses the wrong Python interpreter | `.graphify_python` in the project root stores the interpreter path from when the pipeline last ran — if you switched Python versions, it is stale | Delete `.graphify_python` and re-run `/graphify .` to write a fresh one with the current interpreter |
| `networkx` or `graspologic` import errors | These are graphify dependencies that failed to install | Run `pip install networkx graspologic` (or `pip3` on macOS) to install them directly |
| `graph.json` is missing or zero bytes | The full pipeline has not completed successfully yet | Run the full `/graphify . --obsidian --obsidian-dir <path>` command from Step 2 |
