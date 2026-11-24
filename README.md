# RAG-EvalKit

<div align="center">

[![Typing SVG](https://readme-typing-svg.demolab.com?font=Sora&weight=700&size=22&duration=2800&pause=1000&color=6C63FF&center=true&vCenter=true&width=700&lines=Release+Gates+and+Capacity+Planning.;Retrieval+%C2%B7+Paired+Significance+%C2%B7+Judge+%C2%B7+Splits;Exact+Fractions+%C2%B7+Pydantic+%C2%B7+McNemar+Test;Fail+the+Build+Before+You+Ship.)](https://git.io/typing-svg)

</div>

<div align="center">

![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Pydantic](https://img.shields.io/badge/Pydantic-E92063?style=for-the-badge&logo=pydantic&logoColor=white)
![pytest](https://img.shields.io/badge/pytest-0A9EDC?style=for-the-badge&logo=pytest&logoColor=white)
![Ruff](https://img.shields.io/badge/Ruff-D7FF64?style=for-the-badge&logo=ruff&logoColor=black)

</div>

---

## Overview

**RAG-EvalKit** is a production-grade evaluation and capacity planning library for Retrieval-Augmented Generation (RAG) systems. It answers the hard questions: *Does the retriever actually clear our quality gates? Is Candidate A statistically better than Candidate B? At production scale, how many LLM judge flags will humans actually have to review?*

RAG systems are notoriously hard to evaluate because small changes in retrieval metrics can look like noise, and human review of LLM outputs doesn't scale. This toolkit calculates exact retrieval metrics using standard library `Fractions` (so a threshold hit exactly at 0.8 is never lost to floating-point error), runs McNemar's test for statistical significance, and applies Bayesian prevalence modeling to judge accuracy. 

The toolkit runs as a CLI designed to be used directly as a CI/CD gate—if a metric drops below your threshold, it exits with code `1` and blocks the release.

**Run it** → `evalkit retrieval -i examples/retrieval.json` (see [Quick Start](#quick-start))

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Stack](#stack)
- [Tool Registry](#tool-registry)
- [Decision Logic](#decision-logic)
- [Key Modelling Assumptions](#key-modelling-assumptions)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Extend](#extend)
- [Author](#author)

---

## Features

| Feature | Detail |
|---------|--------|
| Exact Metric Math | Uses Python's `Fraction` type for Recall@k and MRR, guaranteeing exact threshold checks with zero float drift |
| Statistical Significance | Uses McNemar's test and Wald Confidence Intervals to prove if Candidate A is *actually* better than B |
| Production Judge Scaling | Models LLM judge accuracy at real-world prevalence to tell you exactly how many human reviews you need |
| Leakage-Free Splitting | Splits QA sets by document ID, preventing the same document from appearing in both train and test sets |
| Fails as a CI Gate | Exit codes `0` (pass), `1` (gate failed), `2` (bad input) mean it drops straight into your pipeline |
| Strict Validation | Uses `pydantic` with `extra="forbid"` to catch typos in user JSON before the math even starts |

---

## Architecture

```
CLI (cli.py)
        │
        ├── argparse command dispatch (retrieval / paired / judge / split)
        │       exit 0 pass · exit 1 gate failed · exit 2 bad input
        │
        ├── pydantic models — JSON -> validated typed objects
        │       (catches typos, missing fields, wrong types)
        │
        └── pure core modules (no I/O, no printing)
                │
                ├── retrieval.py  Recall@k, MRR, gate checks
                ├── paired.py     McNemar test, Wald CI
                ├── judge.py      Prevalence, human review load
                └── splits.py     Document-level grouping
```

### Request Flow (CLI)

```
Input JSON (examples/*.json)
        │
        ├── Pydantic Validation -> typed domain objects
        │       ValidationError -> exit 2
        ├── Pure function computes metric / statistic
        ├── Gate evaluated (pass/fail vs threshold)
        │       Fail -> exit 1
        └── Result rendered (text or --json) -> exit 0
```

---

## Stack

| Layer | Technology |
|-------|------------|
| Language | Python 3.10+ |
| Exact Arithmetic | `fractions.Fraction` (stdlib) — no float drift on 0.8 |
| Validation | `pydantic` — strict, field-level errors, forbids typos |
| CLI | `argparse` (stdlib) — enough for 4 subcommands |
| Testing | `pytest` — parametrized tests; `scipy` used *only* in tests to cross-check chi-square |
| Lint / Format | `ruff` + `mypy` — style, imports, and type checking |
| CI | GitHub Actions — blocks merges if any gate fails |

---

## Tool Registry

`evalkit` maps every command to a specific RAG evaluation question.

| Command | What it answers | Module |
|---------|-----------------|--------|
| `evalkit retrieval` | Does the retriever clear Recall@k and MRR gates? | `retrieval.py` |
| `evalkit paired` | Should candidate A replace B (McNemar + Wald CI)? | `paired.py` |
| `evalkit judge` | What do an LLM judge's flags mean at production prevalence, and how much can humans review? | `judge.py` |
| `evalkit split` | How do I split a QA set by document without leakage? | `splits.py` |

---

## Decision Logic

| Command | Outcome | How It's Reached | What Happens |
|---------|---------|------------------|--------------|
| `retrieval` | Pass / Fail | Exact fractions compare Recall@k and MRR against your gates. | Exact metrics printed; exit 1 if below gate. |
| `paired` | Better / Not Better | McNemar's test checks if the discordant pairs are significant; Wald CI bounds the difference. | p-value and CI printed; tells you if A is a true upgrade. |
| `judge` | Review Load | Bayesian math maps precision/recall to production prevalence. | Flags per 1,000 queries and human hours needed. |
| `split` | Train/Test Split | Groups records by document ID, then allocates whole groups. | Writes `assignment.json` with zero document overlap. |

**Expected headline results from the bundled data:**
*(If you add sample expected numbers to your data folder, document them here. E.g., "Retrieval passes at 0.85 Recall, McNemar p=0.03")*

---

## Key Modelling Assumptions

*Read these before trusting a verdict.*
- **Wald CI:** Approximate for small discordant counts. Consider an exact or Newcombe interval then.
- **Sample Size:** `queries_needed` assumes the observed effect and discordance rate stay constant.
- **Splitting:** Document-level splitting does not remove near-duplicate documents. Deduplicate first.

---

## Quick Start

```bash
git clone https://github.com/<your-username>/rag-evalkit
cd rag-evalkit
python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\Activate.ps1
pip install -e ".[dev]"
```

Run the tools from the project root (examples are in `examples/`):

```bash
evalkit retrieval -i examples/retrieval.json
evalkit paired    -i examples/paired.json
evalkit judge     -i examples/judge.json
evalkit split     -i examples/split.json --out assignment.json
evalkit paired    -i examples/paired.json --json
python -m rag_evalkit paired -i examples/paired.json     # same thing
```

Test, lint, and type-check:

```bash
pytest                               # all tests
pytest --cov=rag_evalkit --cov-report=term-missing
pytest -k mcnemar -x                 # one test group, stop at first failure
ruff check src tests && ruff format src tests
mypy src
```

### VS Code Setup

1. **File > Open Folder...** and choose `rag-evalkit`.
2. **Ctrl+Shift+P > Python: Select Interpreter** and choose `.venv`.
3. Use the Testing sidebar (beaker icon) to run or debug single tests.
4. To debug, press **F5** (configs are in `.vscode/launch.json`), set a breakpoint, or right-click a test → "Debug Test".

---

## Project Structure

```
rag-evalkit/
├── src/rag_evalkit/
│   ├── retrieval.py  # Recall@k, MRR, gates
│   ├── paired.py     # McNemar, Wald CI
│   ├── judge.py      # Prevalence, review load
│   ├── splits.py     # Leakage-free splits
│   ├── gates.py      # Threshold helpers (at_least, below)
│   ├── cli.py        # argparse, exit codes
│   └── __main__.py   # python -m entrypoint
├── tests/
├── examples/         # Bundled example JSON
├── pyproject.toml
└── README.md
```

---

## Extend

- **New metric (e.g. nDCG@k):** Add a function to `retrieval.py`, test it, and include it in `evaluate`.
- **New gate rule:** Build it using `at_least` / `below` from `gates.py`.
- **Real QA data:** Use `splits.groups_from_records` + `split_records` on your JSONL records.
- **Load runs from CSV/JSONL:** Add a loader in `cli.py`, keeping `retrieval.py` pure.

---

## Author

Built by [Fuad Haque](https://fuadhaque.com)

[fuadhaque.dev@gmail.com](mailto:fuadhaque.dev@gmail.com) · [Book a Call](https://cal.com/fuad-haque) · [GitHub](https://github.com/Fuad-Haque)
