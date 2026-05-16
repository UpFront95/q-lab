# Quantum Lab Workbench

Version: 0.1  
Status: Draft project specification  
Primary stack: Python, Qiskit, IBM Quantum Runtime, FastAPI, SQLite, plain HTML/CSS/JavaScript  
Project type: Research-oriented quantum programming workbench  
Working CLI name: `qlab`

## 1. Purpose

Quantum Lab Workbench is a multi-year quantum computing project that begins with small educational circuits and grows into a reproducible research platform for near-term quantum algorithms.

The project is not just a quantum dice roller. The dice roller is the first visible toy. The larger system is a research cockpit for building, running, comparing, logging, and interpreting quantum experiments across local simulators and IBM Quantum cloud hardware.

The project should let a user run foundational quantum experiments, compare simulator and hardware behavior, track noise and backend drift, visualize circuits and measurement distributions, implement small optimization problems, and eventually explore frontier topics in quantum programming.

The intended result is a durable codebase and research archive: part CLI tool, part local web console, part lab notebook, part algorithm playground.

## 2. Core concept

Every experiment should follow the same lifecycle.

A user defines an experiment. The system generates or loads a quantum circuit. The circuit is run locally first. The circuit is optionally run on IBM Quantum hardware. Results are stored with metadata. The web frontend visualizes the result. A notebook-style report interprets the result. The experiment can be replayed later.

This structure matters because quantum work is highly context-sensitive. Hardware calibration changes. Backend availability changes. Transpilation choices change the actual circuit. Simulator method matters. Shot count matters. Noise matters. Version drift matters.

A useful quantum project should not merely print an answer. It should preserve the path by which the answer was produced.

## 3. Major project surfaces

The project has three main user surfaces.

The CLI is the fast interface for running experiments, checking backend status, exporting results, and iterating quickly from the terminal.

The web frontend is the visual research console. It should show circuits, histograms, backend comparisons, optimizer traces, run history, parameter landscapes, and written experiment notes.

The GitHub repository is the permanent artifact. It should contain source code, tests, notebooks, experiment docs, generated reports, sample datasets, and a long-term research roadmap.

## 4. Technical stack

The core language should be Python.

The quantum layer should use Qiskit, `qiskit-ibm-runtime`, and Qiskit Aer. Qiskit Aer should provide local simulation. IBM Quantum Runtime should provide access to IBM Quantum cloud hardware.

The backend application should use FastAPI. FastAPI should expose JSON endpoints to the browser frontend and should own all interaction with IBM credentials.

The storage layer should begin with SQLite. SQLModel or SQLAlchemy can be used for schema management. SQLite is enough for a single-user research console and keeps the project easy to run locally. Postgres can be added later only if the project becomes multi-user or the experiment archive grows large.

The frontend should use plain HTML, CSS, and JavaScript. Avoid React, Vite, JSX, TypeScript, and bundler complexity at first. The frontend should be a research console, not a frontend framework terrarium.

The project should use `uv` or Poetry for Python dependency management. It should use `ruff`, `pytest`, and optionally `mypy` or Pyright. GitHub Actions should run linting and tests.

## 5. Frontend specification

The frontend should be static browser-native code served by FastAPI or any static host.

The frontend should not execute quantum workloads directly. The browser should be the lab notebook and instrument panel. Python should remain the quantum engine.

Recommended frontend structure:

```text
q-lab/
  web/
    index.html
    app.js
    api.js
    state.js
    router.js
    styles.css
    views/
      home.js
      runs.js
      experiment.js
      circuit-studio.js
      backend-observatory.js
      notebook.js
    components/
      histogram.js
      run-table.js
      circuit-viewer.js
      backend-card.js
      parameter-plot.js
```

The frontend should communicate with FastAPI through JSON endpoints.

```text
GET  /api/experiments
GET  /api/runs
GET  /api/runs/{run_id}
POST /api/run/coin
POST /api/run/dice
POST /api/run/bell
POST /api/run/teleport
POST /api/run/deutsch-jozsa
POST /api/run/grover
POST /api/run/qaoa/maxcut
POST /api/run/portfolio
GET  /api/backends
GET  /api/artifacts/{run_id}/histogram
GET  /api/artifacts/{run_id}/circuit
```

Histograms should initially be rendered with plain SVG or Canvas. Circuit diagrams can initially be generated server-side by Qiskit as SVGs and displayed in the browser. Parameter landscapes can be rendered with Canvas.

The frontend should add third-party charting only if hand-rendered charts become a bottleneck. The first rule should be: no frontend framework until the frontend has earned one.

## 6. Simulator hardware model

Most local simulation should be assumed to be CPU by default.

Qiskit Aer simulation normally runs on CPU unless GPU support has been installed and configured. GPU support is available only for some simulator methods and requires compatible hardware, drivers, and the GPU-enabled Aer package.

For early project stages, CPU simulation is sufficient. Quantum coin, dice, Bell pairs, teleportation, Deutsch-Jozsa, Bernstein-Vazirani, small Grover examples, and tiny QAOA examples do not require GPU acceleration.

GPU support becomes relevant for larger statevector simulations, density-matrix simulations, tensor-network experiments, noisy simulations, and broad parameter sweeps. GPU acceleration can make some simulations faster, but it does not remove the exponential memory growth that comes with exact classical simulation of quantum systems.

The CLI should expose simulator choice but treat GPU as optional.

```bash
qlab bell --mode local --sim cpu
qlab bell --mode local --sim gpu
qlab grover --qubits 20 --mode local --sim gpu
qlab qaoa maxcut --nodes 10 --mode local --sim cpu
```

The simulator configuration should support method and device.

```python
SIMULATOR_DEVICE = "CPU"      # or "GPU"
SIMULATOR_METHOD = "automatic"  # or "statevector", "density_matrix", etc.
```

If GPU is requested but unavailable, the system should fail gracefully.

```text
GPU simulator requested, but GPU-enabled Qiskit Aer support was not detected.
Falling back to CPU simulator unless --require-gpu is set.
```

## 7. IBM Quantum execution model

The project should support three execution modes.

Local mode runs circuits with local simulators. This should be the default for development and testing.

IBM single-job mode submits individual workloads to IBM Quantum hardware through Qiskit Runtime.

IBM session mode groups iterative calls. This is useful for variational algorithms such as QAOA and VQE, where the classical optimizer repeatedly asks the quantum backend to evaluate parameterized circuits.

Use Sampler-style execution for bitstring-producing experiments such as quantum coin, dice, Bell pairs, Deutsch-Jozsa, Bernstein-Vazirani, and Grover.

Use Estimator-style execution for expectation-value experiments such as QAOA, VQE, Hamiltonian simulation, and portfolio optimization mapped into operator form.

IBM credentials should never be exposed to the browser. Credentials should be stored locally through the IBM/Qiskit account mechanism or in local environment variables. The FastAPI backend should be the only layer that interacts with IBM credentials.

## 8. Repository structure

Recommended monorepo structure:

```text
q-lab/
  README.md
  LICENSE
  pyproject.toml
  .env.example
  .gitignore
  .github/
    workflows/
      test.yml
      lint.yml
  docs/
    000_project_spec.md
    010_ibm_setup.md
    020_experiment_model.md
    030_research_roadmap.md
    glossary.md
  qlab/
    __init__.py
    cli.py
    config.py
    ibm/
      service.py
      backends.py
      runtime.py
      sessions.py
    circuits/
      coin.py
      dice.py
      bell.py
      teleportation.py
      deutsch_jozsa.py
      bernstein_vazirani.py
      grover.py
      qaoa.py
      portfolio.py
      vqe.py
    experiments/
      model.py
      registry.py
      runner.py
      analysis.py
      artifacts.py
    noise/
      aer_models.py
      hardware_compare.py
      mitigation.py
    optimization/
      maxcut.py
      portfolio_qubo.py
      classical_baselines.py
    research/
      transpiler_lab.py
      cutting_lab.py
      dynamic_circuits.py
      error_mitigation.py
      vqe_lab.py
      resource_estimation.py
    api/
      main.py
      routes_experiments.py
      routes_runs.py
      routes_backends.py
      routes_artifacts.py
    storage/
      db.py
      schema.py
      migrations/
  web/
    index.html
    app.js
    api.js
    state.js
    router.js
    styles.css
    views/
      home.js
      runs.js
      experiment.js
      circuit-studio.js
      backend-observatory.js
      notebook.js
    components/
      histogram.js
      run-table.js
      circuit-viewer.js
      backend-card.js
      parameter-plot.js
  notebooks/
    001_quantum_coin.ipynb
    002_bell_pair_noise.ipynb
    003_teleportation.ipynb
    004_deutsch_jozsa.ipynb
    005_bernstein_vazirani.ipynb
    006_grover.ipynb
    007_qaoa_maxcut.ipynb
    008_portfolio_optimization.ipynb
    009_vqe_spin_system.ipynb
  experiments/
    .gitkeep
  data/
    sample_assets.csv
    sample_covariance.csv
  tests/
    test_dice.py
    test_circuits.py
    test_experiment_schema.py
    test_classical_baselines.py
```

The public GitHub repo should not contain IBM credentials, private datasets, or large generated artifacts. Experiment summaries can be committed as markdown. Larger run artifacts can be excluded or stored separately.

## 9. CLI specification

The CLI should be the fastest way to run experiments and inspect results.

```bash
qlab doctor
qlab init
qlab auth status
qlab backends list
qlab backends least-busy

qlab coin --shots 1024 --mode local
qlab dice 1d20 --mode local
qlab dice 2d6 --mode ibm --backend least-busy

qlab bell --shots 4096 --mode local
qlab bell --shots 4096 --mode ibm --compare

qlab noise bell --ideal --noisy --hardware
qlab teleport --state plus --mode local
qlab deutsch-jozsa --n 4 --oracle balanced --mode local
qlab bernstein-vazirani --secret 101101 --mode local
qlab grover --qubits 4 --marked 1011 --iterations auto

qlab qaoa maxcut --nodes 6 --p 1 --optimizer cobyla
qlab portfolio optimize --assets data/sample_assets.csv --budget 3 --risk 0.5

qlab runs list
qlab runs show <run_id>
qlab runs compare <run_id_a> <run_id_b>
qlab runs export <run_id> --format markdown

qlab web
```

The `doctor` command should check Python version, installed packages, Qiskit availability, Aer availability, optional GPU support, IBM Runtime package, configured IBM credentials, database status, and frontend availability.

The `backends` command should show available IBM backends, simulator status, qubit count, operational status, pending jobs if available, basis gates, coupling map summary, and last known calibration metadata if exposed.

The `runs export` command should generate a reproducible markdown report containing circuit summary, backend, shots, histogram, metadata, analysis, conclusion, and caveats.

## 10. Experiment data model

Every run should be treated as a scientific object.

```text
Experiment
  id
  slug
  title
  track
  description
  hypothesis
  created_at

Run
  id
  experiment_id
  mode
  backend_name
  backend_kind
  qiskit_version
  qiskit_ibm_runtime_version
  shots
  seed
  simulator_method
  simulator_device
  transpilation_level
  circuit_hash
  runtime_options_json
  status
  created_at
  completed_at

CircuitArtifact
  id
  run_id
  qasm
  circuit_diagram_svg
  depth
  width
  size
  two_qubit_gate_count
  basis_gates_json

ResultArtifact
  id
  run_id
  counts_json
  quasi_distribution_json
  expectation_values_json
  raw_memory_path
  analysis_json
  plot_paths_json

Conclusion
  id
  run_id
  markdown
  tags
```

The system should never save just the answer. It should save the circuit, execution mode, backend, simulator method, device type, transpilation settings, shot count, software versions, raw result, and interpretation.

## 11. Sequential project arc

### Foundation arc

Start with the project skeleton. Build the Python package, CLI, FastAPI backend, static HTML/CSS/JS frontend, SQLite experiment database, and local simulator path.

The first goal is not advanced quantum theory. The first goal is a clean pipeline: define experiment, build circuit, run circuit, store result, display result.

The first working commands should be:

```bash
qlab doctor
qlab coin --shots 1024 --mode local
qlab dice 1d20 --mode local
qlab bell --shots 1024 --mode local
qlab web
```

The first web console should show run history, experiment detail, circuit artifact, counts histogram, and markdown notes.

### Randomness arc

Build the quantum coin and dice roller properly. This introduces qubits, Hadamard gates, measurement, bitstrings, shot counts, histograms, and rejection sampling.

A quantum coin prepares one qubit in `|0⟩`, applies a Hadamard gate, and measures. A quantum dice roller generalizes this by generating enough random bits to cover the die range and then using rejection sampling to avoid modulo bias.

For a d20, generate five measured bits. Five bits represent values from 0 through 31. Values 0 through 19 map to die faces 1 through 20. Values 20 through 31 are rejected and redrawn.

The web UI should show accepted bitstrings, rejected bitstrings, final die values, frequency charts, and simple randomness diagnostics.

This remains a friendly demo, but the real purpose is to teach sampling, measurement, and statistical interpretation.

### Entanglement arc

Build the Bell pair lab. Prepare a Bell state with a Hadamard gate and a CNOT gate, then measure both qubits.

On an ideal simulator, results should mostly be `00` and `11`. On real hardware, results such as `01` and `10` appear because of noise.

The UI should show the raw histogram, correlation metrics, and ideal-versus-hardware comparison.

This is the first experiment where quantum behavior is visibly different from classical independent randomness.

### IBM hardware arc

Add IBM Quantum authentication, backend discovery, backend selection, Runtime execution, job metadata capture, and result import.

The goal is to run the same Bell experiment locally and on IBM hardware, then show the two histograms side by side.

The CLI should support:

```bash
qlab auth status
qlab backends list
qlab bell --shots 1024 --mode ibm --backend least-busy
qlab runs compare <local_run_id> <ibm_run_id>
```

This is the first moment the project touches real quantum hardware.

### Noise and drift arc

Build the noise comparison layer. Every foundational experiment should support ideal local simulation, noisy local simulation, and real hardware execution.

The web UI should include a Backend Observatory. This page should show backend name, run date, circuit depth, two-qubit gate count, transpilation settings, output distribution, deviation from ideal, and notes.

Over time, this becomes one of the most valuable parts of the project. The workbench is not only learning quantum algorithms. It is watching quantum hardware behave like weather.

### Teleportation arc

Build quantum teleportation. Start with a static implementation first, then add dynamic-circuit support where available.

The static version can use separate circuits or post-processing. The dynamic version can use mid-circuit measurement and classical feedforward when supported by the backend.

This arc teaches entanglement as a resource, classical correction, state transfer, and the difference between quantum state transfer and ordinary information transfer.

### Oracle algorithms arc

Add Deutsch-Jozsa and Bernstein-Vazirani.

Deutsch-Jozsa distinguishes constant and balanced functions under a promise. Bernstein-Vazirani recovers a hidden bitstring. Together, they form the first oracle construction lab.

The web UI should show the hidden function or hidden string, generated oracle circuit, final measurement, predicted result, and classical comparison.

This arc teaches interference as computation rather than just quantum weirdness as spectacle.

### Search arc

Add Grover search. Start with tiny marked-bitstring search. Then add multiple marked states. Later add Boolean expression oracles.

The web UI should show the marked state, search-space size, number of Grover iterations, success probability, and histogram.

The research view should plot success probability versus iteration count. This becomes especially interesting when comparing ideal simulation to noisy simulation and real hardware.

This arc teaches amplitude amplification, oracle design, iteration tuning, and the problem of depth on noisy machines.

### Hybrid optimization arc

Add QAOA Max-Cut. This is where the project becomes a serious near-term quantum algorithms workbench.

The system should generate small graphs, solve them exactly with brute force, construct QAOA circuits, run optimizer loops, store optimizer traces, and compare approximation ratios.

The web UI should show the graph, exact optimum, sampled QAOA solutions, parameter trace, approximation ratio, and parameter landscape.

This arc introduces parameterized circuits, classical optimizers, expectation values, approximation quality, and hardware-aware limitations.

### Portfolio optimization arc

Add portfolio optimization as a QUBO and Ising mapping lab.

Use tiny asset sets, expected return vectors, covariance matrices, budget constraints, and risk-aversion parameters.

The system should solve the portfolio problem classically by brute force, then compare QAOA or SamplingVQE-style approaches.

The web UI should show assets, covariance, feasible portfolios, exact best portfolios, sampled quantum results, constraint violations, and solution quality.

This should not be presented as a trading system or financial product. It is an algorithm mapping and research exercise.

### Simulation and VQE arc

Add VQE and Hamiltonian simulation.

Start with small spin systems before chemistry. Then add a standard molecular hydrogen demo once the machinery is stable.

This arc teaches Hamiltonians, ansatz design, expectation estimation, optimizer sensitivity, measurement grouping, and why useful quantum simulation is difficult.

### Transpiler arc

Build a transpiler lab.

Let the user compare untranspiled circuits, transpiled circuits, depth, gate count, two-qubit gate count, layout, routing, and backend-specific changes.

This should apply to Bell states, teleportation, Grover, QAOA, and VQE circuits.

This arc teaches the hard engineering layer between elegant textbook circuits and real machine instructions.

### Error mitigation arc

Add error mitigation experiments.

Start with readout-style post-processing and Runtime-supported mitigation settings where available. Then compare raw and mitigated results across repeated runs.

The web UI should make overhead visible. Error mitigation is not free. It trades extra work and assumptions for better estimates.

### Circuit cutting arc

Add circuit cutting as an advanced research track.

Show the original circuit, cut locations, subcircuits, reconstruction process, sampling overhead, and error versus uncut simulation.

This belongs after the user understands circuit depth, noise, and hardware limitations. Otherwise it is too easy to mistake it for a cheat code.

### Dynamic circuits arc

Return to teleportation, then expand into mid-circuit measurement, reset, feedforward, and tiny adaptive protocols.

This arc should be backend-capability-aware. The CLI and web UI should state clearly when a selected backend does not support the requested dynamic feature.

### Error correction primitives arc

Add repetition codes, bit-flip code, phase-flip code, syndrome measurement, and logical error-rate simulation.

This is not about building a useful fault-tolerant quantum computer. It is about understanding the programming shape of fault tolerance: physical qubits, logical qubits, stabilizers, syndrome bits, correction, and decoding.

### Frontier research arc

Once the workbench is stable, pick serious research lanes.

Strong candidates include QAOA parameter transfer, noise-aware circuit compilation, backend drift benchmarking, error mitigation benchmarking, circuit cutting economics, dynamic-circuit protocol design, oracle-generation tooling, quantum finance mappings, Hamiltonian simulation, and resource estimation.

The long-term goal is not to claim quantum advantage. The long-term goal is to produce a reproducible body of experiments that shows how algorithms, simulators, transpilers, noise, hardware, and classical baselines interact.

## 12. Experiment tracks

### Quantum coin

Prepare a single qubit in `|0⟩`, apply a Hadamard gate, measure, and collect counts.

Expected local simulator behavior is roughly 50/50 between `0` and `1`. Hardware behavior may show bias or noise.

The web UI should show counts, percentages, deviation from ideal, and confidence intervals.

### Quantum dice

Generate enough quantum random bits to cover a die range. Use rejection sampling to avoid modulo bias.

The CLI should support standard dice notation.

```bash
qlab dice 1d20 --mode local
qlab dice 2d6 --mode ibm
qlab dice 4d6 --mode local
```

A later version can support keep-highest, keep-lowest, advantage, disadvantage, roll history, and JSON output.

### Bell pair

Prepare an entangled two-qubit Bell state and measure both qubits.

This experiment should support simulator mode, noisy simulator mode, and IBM hardware mode.

The analysis should compute correlation strength and show unexpected states caused by noise.

### Noise comparison

Run the same circuit across ideal simulation, noisy simulation, and real hardware.

Track circuit depth, gate count, two-qubit gate count, backend, transpilation settings, and result divergence.

This track should eventually become a monthly backend drift benchmark.

### Teleportation

Prepare an input state, create an entangled pair, perform Bell measurement, apply classical correction, and verify the output state.

Support both static and dynamic implementations. Dynamic support should depend on backend capability.

### Deutsch-Jozsa

Generate constant and balanced oracle circuits under the Deutsch-Jozsa promise.

Run the algorithm and classify the oracle.

Compare quantum query behavior to classical deterministic query behavior.

### Bernstein-Vazirani

Generate an oracle encoding a hidden bitstring.

Run the circuit and recover the hidden string.

This is a useful bridge into oracle construction and phase kickback.

### Grover search

Search a tiny space for one or more marked states.

Support custom marked bitstrings, automatic iteration count, and manual iteration count.

Show success probability versus iteration count.

### QAOA Max-Cut

Generate small graphs, solve exactly by brute force, run QAOA, and compare approximation quality.

Track optimizer traces, parameter values, circuit depth, and sampled solution distribution.

### Portfolio optimization

Represent a tiny asset-selection problem as a QUBO or Ising problem.

Include expected returns, covariance matrix, risk-aversion parameter, and a budget constraint.

Compare exact brute force to quantum or hybrid methods.

### VQE and Hamiltonian simulation

Start with tiny spin Hamiltonians. Then add simple chemistry examples.

Compare VQE results against exact diagonalization.

Track ansatz, optimizer, measurement grouping, expectation values, and convergence.

### Transpiler lab

Compare pre-transpile and post-transpile circuits.

Track layout, routing, depth, basis gates, two-qubit gate count, and result quality.

### Error mitigation

Compare raw and mitigated results.

Track overhead, assumptions, and result improvement or degradation.

### Circuit cutting

Cut larger circuits into smaller subcircuits, reconstruct results, and compare with uncut simulations.

Track cut count, sampling overhead, reconstruction error, and practical usefulness.

### Dynamic circuits

Explore mid-circuit measurement, reset, feedforward, and adaptive protocols.

Use backend capability checks before attempting hardware execution.

### Error correction primitives

Simulate simple error-correction codes and syndrome extraction.

Visualize physical qubits, logical qubits, stabilizers, syndrome bits, and corrections.

## 13. Research-oriented web console

The web console should contain the following pages.

The Home page should show the project pathway, recent runs, next recommended experiments, and open research questions.

The Experiment Registry should list all available experiments and their status.

The Run Console should show queued, running, completed, and failed runs.

The Experiment Detail page should show inputs, circuit diagrams, backend metadata, result histograms, analysis, and notes.

The Circuit Studio should show generated circuits and later allow circuit editing or template-based generation.

The Backend Observatory should show backend comparisons, hardware drift, run history by backend, and noise fingerprints.

The Notebook page should store markdown-style hypotheses, observations, interpretations, and follow-up questions.

The Parameter Landscape page should visualize optimizer traces and parameter sweeps for QAOA, VQE, and portfolio optimization.

The Research Board should list frontier topics, experiments in progress, paper notes, failed attempts, and future directions.

## 14. Documentation plan

The README should explain the project, provide local setup instructions, and show the first three commands.

```bash
qlab doctor
qlab coin --mode local
qlab bell --mode local
```

The IBM setup doc should explain account setup, token handling, local testing, backend selection, quota expectations, and credential safety.

The experiment model doc should explain what metadata gets stored and why.

Each major lab should have a notebook and a markdown report.

The research roadmap should be a living document that records open questions, completed experiments, failed experiments, and future ideas.

## 15. Testing plan

Circuit construction should be testable without IBM access.

Dice tests should verify rejection sampling and absence of modulo bias.

Circuit tests should verify qubit count, classical bit count, measurement presence, and expected ideal behavior for small circuits.

Experiment schema tests should verify that runs can be stored, loaded, exported, and replayed.

Classical baseline tests should verify Max-Cut brute force and portfolio brute force.

IBM tests should be optional and skipped unless credentials are present.

## 16. Operating rules

Never spend IBM QPU time before local tests pass.

Never run large shot counts on hardware without a reason.

Never compare quantum results to classical baselines vaguely. Compute exact classical baselines whenever the problem is small enough.

Never report success without showing noise, approximation quality, rejected samples, fidelity, or other relevant diagnostics.

Never hide failed runs. Failed runs are part of the research record.

Never call the portfolio optimizer a financial product. It is an algorithm mapping lab.

## 17. First implementation target

The first build target should be:

```bash
qlab init
qlab doctor
qlab coin --shots 1024 --mode local
qlab dice 1d20 --shots 128 --mode local
qlab bell --shots 1024 --mode local
qlab web
```

The first web version should show only four pages: Home, Runs, Experiment Detail, and Backend Setup.

Then add IBM mode.

```bash
qlab auth status
qlab backends list
qlab bell --shots 1024 --mode ibm --backend least-busy
qlab runs export <run_id> --format markdown
```

This is the seed crystal. Everything else grows from that lattice.

## 18. Long-term identity of the project

At the toy level, this project rolls quantum dice.

At the learning level, it teaches superposition, entanglement, interference, measurement, teleportation, search, optimization, and noise.

At the engineering level, it becomes a reproducible Qiskit and IBM Runtime workbench.

At the research level, it becomes a living archive of experiments on noisy quantum hardware.

At the multi-year level, it becomes a personal quantum field station: codebase, lab notebook, experiment tracker, and probability engine.

## 19. Reference starting points

IBM Quantum Platform: https://quantum.cloud.ibm.com/

Qiskit documentation: https://docs.quantum.ibm.com/

Qiskit Aer documentation: https://qiskit.github.io/qiskit-aer/

Qiskit IBM Runtime repository: https://github.com/Qiskit/qiskit-ibm-runtime

Qiskit Finance tutorials: https://qiskit-community.github.io/qiskit-finance/

Qiskit Algorithms documentation: https://qiskit-community.github.io/qiskit-algorithms/
