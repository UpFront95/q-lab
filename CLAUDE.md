# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Quantum Lab Workbench (`qlab`) is a research-oriented quantum computing platform. It is not a demo toy — it is a reproducible experiment platform for near-term quantum algorithms spanning local simulation, noisy simulation, and IBM Quantum hardware.

Full specification: `quantum-lab-workbench-spec.md`

## Stack

- **Language**: Python, with `uv` or Poetry for dependency management
- **Quantum**: Qiskit, `qiskit-aer` (local simulation), `qiskit-ibm-runtime` (cloud hardware)
- **Backend**: FastAPI (owns all IBM credential interaction — never expose to browser)
- **Storage**: SQLite via SQLModel or SQLAlchemy
- **Frontend**: Plain HTML/CSS/JS — no React, no bundlers, no TypeScript
- **Linting/testing**: `ruff`, `pytest`, optionally `mypy`/Pyright
- **CI**: GitHub Actions (`test.yml`, `lint.yml`)

## Common Commands

```bash
# Install dependencies
uv sync  # or: poetry install

# Lint
ruff check .
ruff format --check .

# Type check (optional)
mypy qlab/  # or: pyright

# Run all tests
pytest

# Run a single test file
pytest tests/test_dice.py

# Run the CLI
python -m qlab.cli <command>
# or if installed:
qlab doctor
qlab coin --shots 1024 --mode local
qlab bell --shots 1024 --mode local
qlab web

# Start the FastAPI backend
uvicorn qlab.api.main:app --reload

# IBM auth check (requires credentials configured)
qlab auth status
qlab backends list
```

## Architecture

### Experiment lifecycle

Every experiment follows: define → build circuit → run locally → optionally run on IBM → store result with full metadata → display in web UI → export as reproducible report. The data model in `qlab/storage/schema.py` captures `Experiment`, `Run`, `CircuitArtifact`, `ResultArtifact`, and `Conclusion`. **Never store just the answer** — always store circuit, backend, transpilation settings, shot count, software versions, raw result, and interpretation.

### Key module boundaries

| Path | Responsibility |
|---|---|
| `qlab/cli.py` | Typer/Click CLI entrypoint, `qlab` command |
| `qlab/circuits/` | Pure circuit builders — no IBM calls, no DB |
| `qlab/experiments/runner.py` | Dispatches local vs IBM execution |
| `qlab/ibm/` | All IBM Quantum Runtime interaction; credentials never leave this layer |
| `qlab/api/` | FastAPI app; all browser-facing JSON endpoints |
| `qlab/storage/` | SQLite schema, migrations, read/write helpers |
| `qlab/noise/` | Aer noise models, hardware comparison, mitigation |
| `qlab/optimization/` | Max-Cut, portfolio QUBO, classical baselines |
| `qlab/research/` | Advanced tracks: transpiler lab, circuit cutting, dynamic circuits |
| `web/` | Static frontend — no build step, served by FastAPI or any static host |

### Execution modes

- **local**: Qiskit Aer CPU simulator (default for all dev/test)
- **ibm**: IBM Quantum Runtime via `qlab/ibm/`
- **GPU** (optional): Aer GPU device, falls back to CPU gracefully unless `--require-gpu` is set

Use **Sampler** primitives for bitstring experiments (coin, dice, Bell, Grover, Deutsch-Jozsa).  
Use **Estimator** primitives for expectation-value experiments (QAOA, VQE, portfolio).  
IBM sessions are used only for iterative variational algorithms (QAOA, VQE) where the classical optimizer loops over the quantum backend.

### Frontend

Plain JS modules in `web/`. No bundler. API calls go through `web/api.js`. State lives in `web/state.js`. Routing in `web/router.js`. Views in `web/views/`, reusable components in `web/components/`. Histograms rendered with SVG/Canvas; circuit diagrams come from server-side Qiskit SVG output.

## Testing Rules

- All circuit tests must run without IBM credentials (local Aer only).
- IBM-dependent tests must be skipped unless credentials are present (use a pytest marker or env check).
- Dice tests must verify rejection sampling and absence of modulo bias.
- Classical baseline tests (Max-Cut brute force, portfolio brute force) must be deterministic and fast.

## Operating Rules (from spec)

- Never spend IBM QPU time before local tests pass.
- Never compare quantum results to classical baselines vaguely — compute exact classical baselines when the problem is small enough.
- Never hide failed runs — they are part of the research record.
- The portfolio optimizer is an algorithm mapping lab, not a financial product.

## Build Order

The spec defines a sequential arc. Current first target:

```bash
qlab init
qlab doctor
qlab coin --shots 1024 --mode local
qlab dice 1d20 --shots 128 --mode local
qlab bell --shots 1024 --mode local
qlab web
```

Then add IBM mode. Do not skip ahead — each arc depends on the previous one being stable.
