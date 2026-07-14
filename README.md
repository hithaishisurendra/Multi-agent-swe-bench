# SWE-bench-C Multi-Agent Bug Fixer

Multi-agent pipeline for automated bug fixing on SWE-bench-C (C repositories: jq, redis, zstd). Single-shot LLMs fail on this benchmark because they cannot localize bugs without repository access. They generate syntactically valid patches with correct diff headers and hunk markers, but the context lines do not match the actual repository state at base commit. The patches are structurally convincing fiction. This system addresses that failure mode through graph-augmented retrieval and evidence-routed feedback loops.

## Architecture

![SWE-bench-C Multi-Agent Pipeline](swe%20architecture.png)

```
Issue Text
    │
    ▼
┌─────────┐       PlannerOutput
│ Planner │ ──────────────────────────────────────────────────────────────────┐
└─────────┘                                                                   │
    │                                                                         │
    │ keywords, suspected_modules, priority_functions                         │
    ▼                                                                         │
┌───────────┐      ContextBundle                                              │
│ Localizer │ ─────────────────────────────────────────────────────────────┐ │
└───────────┘                                                               │ │
    │ ranked files, snippets, confidence                                    │ │
    ▼                                                                       │ │
┌──────────────┐   FixPlan                                                  │ │
│ Diagnostician│ ──────────────────────────────────────────────────────┐   │ │
└──────────────┘                                                        │   │ │
    │ root cause, affected regions, fix description                     │   │ │
    ▼                                                                   │   │ │
┌────────┐         PatchOutput                                          │   │ │
│ Patcher│ ─────────────────────────────────────────────────────────┐  │   │ │
└────────┘                                                           │  │   │ │
    │ unified diff (git-apply ready)                                 │  │   │ │
    ▼                                                                │  │   │ │
┌──────────┐      ValidationResult                                   │  │   │ │
│ Validator│ ── apply → compile → test ──────► SUCCESS              │  │   │ │
└──────────┘                    │                                    │  │   │ │
                        FAILURE │  FeedbackMessage                   │  │   │ │
                                ├── apply_failed   ──────────────────┘  │   │ │
                                ├── compile_failed ──────────────────────┘   │ │
                                ├── test_failed    ──────────────────────────┘ │
                                ├── regression     ──────────────────────────────┘ (Diagnostician)
                                └── low_confidence ──────────────────────────────── Planner (replan)
```

### Agents

**Planner**: Receives the issue text and produces a search strategy. Classifies the issue type (bug fix, feature request, refactor), extracts keywords, and generates priority functions for the Localizer. No repository access.

**Localizer**: Three-pass retrieval engine. Pass 1 is graph keyword search across function and file nodes. Pass 2 is grep for keyword density. Pass 3 expands via graph neighborhood (callers, callees, shared headers). Composite scoring produces a ranked context bundle with file contents and function snippets. Confidence below 0.4 triggers replan.

**Diagnostician**: Reads localized source files and test metadata to identify the root cause. Produces a structured fix plan with affected lines, expected behavior, and test constraints. Does not generate code.

**Patcher**: Generates a unified diff from the fix plan and real source file contents. Context lines are copied from actual files, not hallucinated. This is the critical difference from single-shot approaches.

**Validator**: Runs git apply, then the repository build system (make/gcc for C), then FAIL_TO_PASS tests. Stops at first failure and returns typed error messages.

### Tree-Sitter Dependency Graph

Built offline before agents run. Parses C source files without requiring compilation. Nodes are files and functions (with line ranges). Edges are #include dependencies, function calls, type/struct usage, and test mappings. Stored in-memory as Python dictionaries. Construction takes seconds per repository.

This graph enables structural retrieval. A fix in compress.c may require zstd_internal.h and compress_impl.c, neither of which mentions issue keywords. The graph captures these relationships explicitly.

## Evidence Routing

Each failure type routes back to the one agent that can fix it, capped at 2 retries per slot.

| Failure Type   | Detected At | Routed To     | Action                                     |
| -------------- | ----------- | ------------- | ------------------------------------------ |
| apply_failed   | Validator   | Localizer     | Re-search with expanded graph neighborhood |
| compile_failed | Validator   | Patcher       | Retry with compiler error message          |
| test_failed    | Validator   | Diagnostician | Re-analyze with test output                |
| regression     | Validator   | Diagnostician | Revise fix plan                            |
| low_confidence | Localizer   | Planner       | Replan with alternative keywords           |

No re-running the whole pipeline on every retry. A compilation error goes to the Patcher with the error message. A test failure goes to the Diagnostician with test output. An apply failure goes to the Localizer for better retrieval.

## Results

Evaluated on 174 instances from SWE-bench-C (jq: 45, redis: 90, zstd: 44).

### Localization

**22 of 174 instances** localized (gold-patch file recovered). Beats all three baselines:

- V1 Qwen3 Blind: 5/174
- V2 Qwen3 Enhanced: 17/174 (but this baseline consumed gold-patch metadata: file paths, affected functions, test names)
- V3 GPT-4.1-mini: 1/174

SWE-bench-AGENT does not consume any gold-patch metadata.

### Compilation

- jq subset: 9 patches compiled after localization
- redis subset: 1 apply-clean patch (redis_redis-8580), the only system besides the metadata-fed baseline to apply anything on redis
- zstd subset: 3 localized the correct file but failed git apply due to mismatched hunk context lines

### Resolution

**0 of 174 resolved.** Every system in the study (single-shot and agentic) resolved zero instances. The published Phase 1 baseline on SWE-bench-C is also 0%.

### What Blocks Resolution

Hunk context lines must match the repository state exactly at base commit. A single character difference causes git apply to fail. Even when the Patcher has access to real source files, generating context that matches line-for-line remains the wall. This is the central open challenge for the benchmark.

## Quick Start

### Install

```bash
pip install tree-sitter tree-sitter-c openai python-dotenv
```

### Environment

```bash
cp .env.example .env
# Edit .env and add your OPENAI_API_KEY (Voyager API key)
```

### Run

```bash
# Single instance
python run_pipeline.py --instance_id jq-493__jqlang__jq

# All jq instances
python run_pipeline.py --repo jqlang/jq

# All 174 SWE-bench-C instances
python run_pipeline.py --all
```

Logs are written to `logs/<instance_id>.jsonl`. Final validation results are in `results/<instance_id>.json`.

## Repository Structure

```
swe-bench-agent/
├── config.py                    # Environment configuration
├── run_pipeline.py              # CLI entry point
├── .env.example                 # API key template
│
├── graph/                       # Tree-sitter dependency graph
│   ├── model.py                 # DepGraph data structure
│   ├── builder.py               # C parser and graph builder
│   ├── query.py                 # Traversal API (callers, callees, etc)
│   ├── scoring.py               # Keyword search and evidence subgraph
│   └── repo_checkout.py         # Clone and checkout at base commit
│
├── pipeline/                    # Orchestration
│   ├── schema.py                # Inter-agent typed messages
│   ├── logger.py                # JSONL event logger
│   └── controller.py            # Evidence routing and retry logic
│
├── agents/                      # Agents
│   ├── planner.py               # Issue classification and search strategy
│   ├── localizer.py             # Three-pass retrieval engine
│   ├── diagnostician.py         # Root cause analysis
│   ├── patcher.py               # Unified diff generation
│   ├── validator.py             # git apply → compile → test
│   ├── llm.py                   # Shared OpenAI-compatible client
│   ├── stubs.py                 # Agent stubs for smoke testing
│   └── tools/                   # Localizer tools
│       ├── grep_tool.py         # Whole-word regex grep
│       ├── file_reader.py       # File reading and snippet extraction
│       └── graph_tools.py       # Scored candidate builder
│
├── scripts/
│   ├── build_all_graphs.py      # Batch graph builder
│   ├── test_graph.py            # Graph infrastructure tests
│   ├── test_person2.py          # Planner and controller tests
│   ├── test_person3.py          # Localizer tests
│   ├── test_person4.py          # Diagnostician and Patcher tests
│   └── test_person5.py          # Validator tests
│
├── graphs/                      # Pre-built graph JSON files
├── repos/                       # Cloned repository checkouts
├── results/                     # Final validation results
└── logs/                        # Per-instance JSONL logs
```

## Environment Variables

| Variable          | Default                        | Description                                     |
| ----------------- | ------------------------------ | ----------------------------------------------- |
| `OPENAI_API_KEY`  | required                       | Voyager API key                                 |
| `OPENAI_API_BASE` | `https://openai.rc.asu.edu/v1` | Voyager base URL                                |
| `MODEL_NAME`      | `qwen3-30b-a3b-instruct-2507`  | Model for Planner/Diagnostician/Patcher         |
| `CONF_THRESHOLD`  | `0.4`                          | Localizer confidence below this triggers replan |
| `MAX_RETRIES`     | `2`                            | Max retries per agent slot                      |
| `COMPILE_TIMEOUT` | `300`                          | Validator make timeout (seconds)                |
| `TEST_TIMEOUT`    | `120`                          | Validator per-test timeout (seconds)            |
| `LOG_DIR`         | `logs/`                        | JSONL log output directory                      |
| `GRAPHS_DIR`      | `graphs/`                      | Pre-built graph JSON files                      |
| `REPOS_DIR`       | `repos/`                       | Cloned repository checkouts                     |
| `RESULTS_DIR`     | `results/`                     | Final ValidationResult JSON files               |
