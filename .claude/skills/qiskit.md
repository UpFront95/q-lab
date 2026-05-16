---
name: qiskit
description: Write, run, and reason about quantum programs with current Qiskit 2.x using the modern primitives workflow (SamplerV2 / EstimatorV2, ISA circuits, generate_preset_pass_manager, SparsePauliOp). Use whenever the user touches anything quantum in Python — Qiskit, IBM Quantum, quantum circuits, qubits, Hadamard / CNOT / Bell / GHZ states, transpilation, primitives, Sampler, Estimator, Aer simulators, noise models, error mitigation, dynamical decoupling, VQE, QAOA, QPE, Grover, Shor, quantum chemistry / machine learning — even when the word Qiskit is not used. Pins to Qiskit 2.x and refuses legacy patterns (BackendV1, qiskit.opflow, qiskit.utils.QuantumInstance, qiskit.algorithms, qiskit.execute, backend.run, V1 Sampler/Estimator, pulse) because those imports no longer exist or silently produce wrong results. Use for new code, porting old code, debugging ImportErrors in quantum scripts, and explaining what idiomatic Qiskit looks like today.
---

# Qiskit (2.x) skill

## Why this skill exists

Qiskit has gone through unusually heavy API churn:

- `qiskit` 0.x → 1.x → 2.x, with 2.0 (Mar 2025) deleting a lot of long-deprecated surface area, and 1.x reaching end-of-life in 2025.
- The provider, simulator, and algorithms layers were split out into separate packages (`qiskit-ibm-runtime`, `qiskit-aer`, `qiskit-algorithms`).
- The execution model moved from `backend.run(circuit)` → V1 primitives → V2 primitives (PUBs). `backend.run` is **gone** on Runtime.
- Pulse-level access and `qiskit.opflow` were removed entirely.

The model's training data is full of old patterns. Code that "looks like Qiskit" — `from qiskit import execute`, `QuantumInstance`, `PauliSumOp`, `Sampler` (V1), `BackendV1.configuration()` — is the most common failure mode. Treat anything that doesn't match the four-step pattern below as a red flag.

## The four-step Qiskit pattern

Every well-formed Qiskit program follows the same shape. Anchor on this; everything else is detail.

1. **Map** the problem to circuits and observables. Use `QuantumCircuit` and `SparsePauliOp`.
2. **Optimize for execution.** Transpile to an *ISA circuit* (instruction-set architecture) targeting the backend, using `generate_preset_pass_manager`. Apply layout to observables.
3. **Execute** with a V2 primitive — `SamplerV2` (returns bitstrings / counts) or `EstimatorV2` (returns expectation values). Inputs are *PUBs* (primitive unified blocs).
4. **Analyze** the result. PUB results expose `.data.<register_name>` (Sampler) or `.data.evs` and `.data.stds` (Estimator).

Even on a local statevector simulator, follow the same shape. The transpile step is a no-op for the reference primitives but keeps code portable.

## Package landscape (what lives where)

This is the single most common source of import errors.

- `qiskit` — the SDK core: `QuantumCircuit`, `QuantumRegister`, `ClassicalRegister`, `qiskit.quantum_info` (`SparsePauliOp`, `Statevector`, `Operator`, `Pauli`), `qiskit.transpiler` (`generate_preset_pass_manager`), `qiskit.circuit.library` (parameterized circuits like `real_amplitudes`, `efficient_su2`, `n_local`, `iqp`, `qaoa_ansatz`), `qiskit.primitives` (reference implementations: `StatevectorSampler`, `StatevectorEstimator`), `qiskit.visualization`.
- `qiskit-ibm-runtime` — IBM hardware access: `QiskitRuntimeService`, `SamplerV2`, `EstimatorV2`, `Session`, `Batch`, `qiskit_ibm_runtime.fake_provider` (e.g. `FakeManilaV2` for noise-model testing without an account).
- `qiskit-aer` — high-performance local simulator with noise: `AerSimulator`, `qiskit_aer.primitives.SamplerV2`, `qiskit_aer.primitives.EstimatorV2`, `qiskit_aer.noise.NoiseModel`.
- `qiskit-algorithms` — community-maintained, separate-install package containing packaged versions of VQE, QAOA, AmplitudeEstimation, time evolvers, gradients, etc. Lags the SDK and emits deprecation warnings. For plain VQE/QAOA prefer the hand-rolled `scipy.optimize` + `EstimatorV2` pattern (template D); reach for `qiskit-algorithms` only when you specifically want a packaged amplitude estimator, Trotter evolver, or gradient class. **Not** importable from `qiskit.algorithms` anymore.
- Domain packages — `qiskit-nature` (chemistry), `qiskit-optimization`, `qiskit-machine-learning`, `qiskit-finance`. Install as needed; do not assume they are present.

If the user imports something and gets `ModuleNotFoundError`, almost always the answer is "that's now in a separate package; pip install it."

## Canonical templates

Start from these, do not invent shapes.

### A. Local statevector simulation (no IBM account needed)

```python
from qiskit import QuantumCircuit
from qiskit.quantum_info import SparsePauliOp
from qiskit.transpiler import generate_preset_pass_manager
from qiskit.primitives import StatevectorSampler, StatevectorEstimator

# 1. Map: Bell state
qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)

# Sampling path: need measurements
qc_meas = qc.copy()
qc_meas.measure_all()

# 2. Optimize (reference primitives accept abstract gates, but transpiling keeps code portable)
pm = generate_preset_pass_manager(optimization_level=1)
isa_qc_meas = pm.run(qc_meas)

# 3a. Execute with Sampler
sampler = StatevectorSampler()
result = sampler.run([isa_qc_meas], shots=4096).result()
counts = result[0].data.meas.get_counts()   # "meas" = default measure_all() register name
print(counts)

# 3b. Execute with Estimator (no measurements on the circuit!)
isa_qc = pm.run(qc)
observable = SparsePauliOp.from_list([("ZZ", 1.0), ("XX", 1.0)])
isa_obs = observable.apply_layout(isa_qc.layout)   # observable must follow circuit's layout

estimator = StatevectorEstimator()
ev = estimator.run([(isa_qc, isa_obs)]).result()[0].data.evs
print(ev)
```

Two things that trip people up here:

- **Sampler circuits need measurements, Estimator circuits must not.** Estimator measures the observable itself.
- After transpiling, **always** call `observable.apply_layout(isa_qc.layout)` before passing the observable to Estimator. Otherwise the operator addresses logical qubits while the circuit addresses physical ones, and the answer is silently wrong.

### B. Aer simulator with a noise model

Use Aer when the user wants realistic shot noise or wants to simulate a specific device.

```python
from qiskit_aer import AerSimulator
from qiskit_aer.primitives import SamplerV2, EstimatorV2
from qiskit_ibm_runtime.fake_provider import FakeManilaV2

# Build an AerSimulator from a fake backend's noise model
fake_backend = FakeManilaV2()
sim = AerSimulator.from_backend(fake_backend)

pm = generate_preset_pass_manager(backend=sim, optimization_level=1)
isa_qc = pm.run(qc_meas)

sampler = SamplerV2.from_backend(sim)   # also: SamplerV2(default_shots=4096)
result = sampler.run([isa_qc]).result()
```

### C. Real IBM Quantum hardware

```python
from qiskit_ibm_runtime import QiskitRuntimeService, SamplerV2 as Sampler

service = QiskitRuntimeService()           # uses saved account; see "Authentication" below
backend = service.least_busy(operational=True, simulator=False, min_num_qubits=5)

pm = generate_preset_pass_manager(backend=backend, optimization_level=2)
isa_qc = pm.run(qc_meas)

sampler = Sampler(mode=backend)
sampler.options.default_shots = 4096
# Optional: error suppression
sampler.options.dynamical_decoupling.enable = True
sampler.options.dynamical_decoupling.sequence_type = "XpXm"

job = sampler.run([isa_qc])
print(f"Job ID: {job.job_id()}")
result = job.result()
counts = result[0].data.meas.get_counts()
```

For Estimator on hardware, also configure resilience (error mitigation):

```python
from qiskit_ibm_runtime import EstimatorV2 as Estimator

estimator = Estimator(mode=backend)
estimator.options.resilience_level = 1            # 0=none, 1=TREX, 2=ZNE, ...
estimator.options.resilience.measure_mitigation = True
job = estimator.run([(isa_qc, isa_obs)])
evs = job.result()[0].data.evs
```

Mode argument: pass a `backend` (job mode), a `Session(backend=...)` (low-latency sequence), or a `Batch(backend=...)` (queue several jobs). Pick `Batch` for many independent jobs, `Session` when each job depends on the previous result (e.g. variational loops).

**Dry-run a hardware script without an IBM account.** `qiskit_ibm_runtime.fake_provider` ships snapshots of real devices (`FakeManilaV2`, `FakeBrisbaneV2`, `FakeKyivV2`, etc.) that implement the same `BackendV2` interface. Drop one in where `service.least_busy(...)` would go and the rest of the script — transpile, sampler options, PUB submission — runs end-to-end locally. Use this to validate ISA construction, register names, and option settings before burning queue time.

```python
from qiskit_ibm_runtime.fake_provider import FakeManilaV2
backend = FakeManilaV2()                     # <-- swap in for service.least_busy(...)
# Everything below stays identical.
pm = generate_preset_pass_manager(backend=backend, optimization_level=2)
# ...
```

For shot-noise realism with this fake backend, route execution through Aer (template B): `AerSimulator.from_backend(fake_backend)` builds a simulator that uses the snapshot's calibration data.

### D. Variational algorithms — scipy.optimize over EstimatorV2 directly

For VQE, QAOA, and any variational loop, the modern idiomatic shape is **not** to reach for `qiskit-algorithms`. That package exists, but it lags the SDK, carries deprecation warnings, and hides the very loop you typically need to instrument. Build the cost function by hand on top of `EstimatorV2` and minimize it with `scipy.optimize.minimize`. The whole loop is ~20 lines and the structure transfers verbatim from simulator to hardware.

```python
import numpy as np
from scipy.optimize import minimize

from qiskit.circuit.library import efficient_su2
from qiskit.quantum_info import SparsePauliOp
from qiskit.transpiler import generate_preset_pass_manager
from qiskit.primitives import StatevectorEstimator

# 1. MAP
hamiltonian = SparsePauliOp.from_list(
    [("II", -1.05), ("IZ", 0.40), ("ZI", -0.40), ("ZZ", 0.18), ("XX", 0.18)]
)
ansatz = efficient_su2(hamiltonian.num_qubits, reps=2)

# 2. OPTIMIZE FOR EXECUTION — transpile the ansatz once, reuse every iteration.
# On hardware, set backend=...; here we transpile abstractly.
pm = generate_preset_pass_manager(optimization_level=1)
isa_ansatz = pm.run(ansatz)
isa_hamiltonian = hamiltonian.apply_layout(isa_ansatz.layout)

estimator = StatevectorEstimator(seed=42)

# 3. EXECUTE — cost is one Estimator job per call. PUB carries the bound parameters.
history = []
def cost(params, ansatz, observable, estimator):
    pub = (ansatz, observable, params)
    energy = float(estimator.run([pub]).result()[0].data.evs)
    history.append(energy)
    return energy

x0 = np.random.default_rng(42).uniform(-np.pi, np.pi, isa_ansatz.num_parameters)
result = minimize(
    cost, x0,
    args=(isa_ansatz, isa_hamiltonian, estimator),
    method="COBYLA",            # gradient-free, robust on noisy cost; SLSQP/L-BFGS-B if smooth
    options={"maxiter": 200, "rhobeg": 0.5},
)

# 4. ANALYZE
print(f"VQE energy:  {result.fun:+.6f}")
print(f"Iterations:  {result.nfev}")
print(f"Converged:   {result.success}")
```

For **hardware VQE**, the diff is small:

```python
from qiskit_ibm_runtime import QiskitRuntimeService, Session, EstimatorV2 as Estimator

service = QiskitRuntimeService()
backend = service.least_busy(operational=True, simulator=False)

pm = generate_preset_pass_manager(backend=backend, optimization_level=2)
isa_ansatz = pm.run(ansatz)
isa_hamiltonian = hamiltonian.apply_layout(isa_ansatz.layout)

with Session(backend=backend) as session:
    estimator = Estimator(mode=session)
    estimator.options.resilience_level = 1
    estimator.options.default_shots = 4096
    result = minimize(
        cost, x0, args=(isa_ansatz, isa_hamiltonian, estimator),
        method="COBYLA", options={"maxiter": 100},
    )
```

Use a `Session`, not job mode, so the iterations stay co-scheduled and you don't re-queue between every cost evaluation.

**Why this shape and not `qiskit-algorithms.VQE`:**

- One place to add logging, batching, gradient tricks, parameter clipping, or early stopping. The packaged class buries all of these.
- PUB broadcasting falls out naturally — if you want SPSA or parameter-shift gradients, stack the parameter vectors into a 2D array and submit one PUB; the cost evaluation parallelizes server-side.
- The cost function is also the thing you want to plot, save, and resume from. Direct ownership of it is worth the 10 extra lines.

`qiskit-algorithms` still has useful pieces (`AmplitudeEstimation`, `TrotterQRTE`, gradient classes), so reach for it when those specifically help. For plain VQE/QAOA, skip it.

## ISA circuits, explained once

A backend can only execute a specific set of gates on a specific qubit connectivity (its *target*). An ISA circuit is a circuit already rewritten in those terms — basis gates only, two-qubit gates only on connected pairs, logical-to-physical qubit mapping applied. The `generate_preset_pass_manager` helper builds a transpiler pipeline for a given `backend` and `optimization_level` (0–3); calling `.run(circuit)` produces the ISA circuit.

You must transpile before submitting to Runtime. The Runtime primitives **reject non-ISA circuits**. The reference primitives accept abstract gates, but transpiling anyway costs nothing and keeps code portable.

Optimization levels: 0 (none, layout only), 1 (light), 2 (medium, usually a good default), 3 (heavy, slow). Pick 2 for hardware jobs unless the circuit is enormous.

## Primitives in detail

A **PUB** (Primitive Unified Bloc) is a tuple bundling everything one execution needs:

- Sampler PUB: `(circuit, parameter_values?, shots?)`
- Estimator PUB: `(circuit, observables, parameter_values?, precision?)`

You always pass a *list* of PUBs to `.run([...])`. The result is a `PrimitiveResult` indexed by PUB position; each `pub_result` exposes:

- Sampler: `pub_result.data.<creg_name>` (a `BitArray`) with `.get_counts()`, `.get_bitstrings()`, `.array`.
- Estimator: `pub_result.data.evs` and `pub_result.data.stds`, broadcast over the PUB's parameter/observable shape.

The classical register name matters. `measure_all()` creates a register called `meas`. If the user uses named `ClassicalRegister`s, the data attribute matches those names. **Never** assume `.data.c` or `.data.counts`; inspect the circuit.

Worked example — splitting a 5-qubit measurement across two named registers, then reading each independently:

```python
from qiskit import QuantumCircuit, QuantumRegister, ClassicalRegister
from qiskit.primitives import StatevectorSampler

qreg = QuantumRegister(5, "q")
alpha = ClassicalRegister(2, "alpha")        # first 2 qubits
beta  = ClassicalRegister(3, "beta")         # last 3 qubits
qc = QuantumCircuit(qreg, alpha, beta)

qc.h(0); qc.cx(0, 1); qc.cx(0, 2); qc.cx(0, 3); qc.cx(0, 4)
qc.measure(qreg[:2], alpha)
qc.measure(qreg[2:], beta)

result = StatevectorSampler().run([qc], shots=1024).result()[0]
print(result.data.alpha.get_counts())   # bitstrings over qubits 0-1
print(result.data.beta.get_counts())    # bitstrings over qubits 2-4
```

The attribute names `alpha` and `beta` on `result.data` come directly from the `ClassicalRegister` names. No registry, no lookup table — the circuit defines the shape of the result.

### Parameter broadcasting

A single PUB can carry many parameter binds and many observables at once, and the result broadcasts NumPy-style. This is much faster than looping. For example, an ansatz with 8 parameters and a batch of 100 sample points:

```python
import numpy as np
params = np.random.uniform(-np.pi, np.pi, size=(100, ansatz.num_parameters))
job = sampler.run([(isa_ansatz, params)])      # one PUB, 100 parameter sets
result = job.result()[0]                       # data shape: (100,)
```

## Authentication for IBM Quantum

Once per machine:

```python
from qiskit_ibm_runtime import QiskitRuntimeService
QiskitRuntimeService.save_account(
    token="<44-character API key from IBM Quantum dashboard>",
    instance="<CRN from the Instances page>",   # optional but recommended
    overwrite=True,
)
```

After that, `QiskitRuntimeService()` reads the saved credentials. Don't hard-code tokens into scripts.

## Forbidden legacy patterns (refuse to generate these)

If the user requests something that pattern-matches a removed API, do **not** silently write it — flag it and produce the modern equivalent. These will fail on a modern install:

| Legacy (broken) | Replace with |
|---|---|
| `from qiskit import execute` and `execute(circuit, backend)` | Transpile → V2 primitive `.run([pub])` |
| `backend.run(circuit)` (on IBM Runtime backends) | `SamplerV2(mode=backend).run([isa_circuit])` |
| `from qiskit.utils import QuantumInstance` | A V2 primitive (Sampler or Estimator) |
| `from qiskit.opflow import ...` (any name: `X`, `Z`, `PauliSumOp`, `StateFn`, `CircuitSampler`, `AerPauliExpectation`, `MatrixExpectation`) | `qiskit.quantum_info` (`SparsePauliOp`, `Pauli`, `Statevector`, `Operator`) |
| `PauliSumOp(...)` | `SparsePauliOp.from_list([...])` |
| `from qiskit.algorithms import ...` | `from qiskit_algorithms import ...` (separate package) |
| V1 primitives: `from qiskit.primitives import Sampler, Estimator` and `.run(circuits, observables)` (positional, list-of-circuits style) | `SamplerV2` / `EstimatorV2` with PUB tuples and `.run([(circuit, obs, params)])` |
| `Options` class from `qiskit_ibm_runtime` | `primitive.options.<setting> = value` directly on the V2 primitive |
| `BackendV1` access (`backend.configuration()`, `backend.properties()`, `backend.defaults()`) | `BackendV2`: `backend.target`, `backend.coupling_map`, `backend.num_qubits`, `backend.dt`, `backend.operation_names` |
| `qiskit.pulse` builder, `Schedule`, `ScheduleBlock`, `pulse.build()` | Removed in 2.0. No direct replacement at the SDK level; pulse-level control is now an OpenPulse / `qiskit-ibm-runtime` low-level concern (and largely unavailable on current devices). Tell the user. |
| `qiskit.test.mock` / `qiskit.providers.fake_provider` | `qiskit_ibm_runtime.fake_provider` |
| `assemble`, `Qobj` | Gone; primitives consume circuits directly. |
| `BasicAer`, `Aer.get_backend('...')` style (old Aer 0.x) | `from qiskit_aer import AerSimulator` |
| `circuit.bind_parameters({...})` (deprecated name) | `circuit.assign_parameters({...})` |
| PascalCase circuit-library classes `TwoLocal`, `RealAmplitudes`, `EfficientSU2`, `QAOAAnsatz` | New function-style: `n_local`, `real_amplitudes`, `efficient_su2`, `qaoa_ansatz`. The classes still work in 2.x but are scheduled for removal in 3.0; prefer the functions for new code. |

When the user pastes legacy code, do not "translate line by line." Re-derive what the program is *trying* to do, then write it in the four-step pattern. Line-by-line translation usually leaves stale assumptions intact (e.g. translating `QuantumInstance(backend, shots=1024)` into a primitive without realising the shots arg now lives on a PUB or on `options.default_shots`).

## Common gotchas

- **Estimator circuits must not contain measurements.** The primitive handles the basis change. A measured circuit handed to Estimator either errors out or returns garbage depending on version.
- **`apply_layout` on the observable after transpiling.** Easy to forget; produces wrong expectation values without any error.
- **Classical register names propagate to the result.** Use named registers when reading specific subsets of qubits; otherwise stick to `measure_all()` and read `data.meas`.
- **Endianness.** Qiskit prints bitstrings with qubit 0 on the **right** (little-endian). When comparing to a hand-derived expected value, reverse the string or use `result.get_int_counts()`.
- **Seeding.** For reproducibility, pass `seed=...` to `StatevectorSampler`/`StatevectorEstimator`, and set `algorithm_globals.random_seed` if using `qiskit-algorithms`. NumPy's `default_rng(seed)` covers parameter generation.
- **Drawing.** `circuit.draw("mpl")` needs matplotlib. `circuit.draw()` returns ASCII and is always safe.
- **Plotting histograms.** `from qiskit.visualization import plot_histogram`. Pass a counts dict, not a `BitArray`.
- **Don't iterate one circuit at a time.** If running N variants, build a list of PUBs and submit one job. Hardware queue time dominates.

## When the user's request is ambiguous

- "Run a circuit" with no further detail → default to the **local statevector** template (template A). Cheap, no auth, works offline. Note the path to hardware in a one-liner.
- "Simulate noise" → Aer template (template B).
- "Run on IBM / on a real quantum computer" → Runtime template (template C), and mention the auth step if it's the user's first time.
- Anything that says "find the ground state" / "minimize energy" / "VQE" / "QAOA" → algorithms template (template D). Default to the hand-rolled `scipy.optimize.minimize` + `EstimatorV2` shape; only suggest `qiskit-algorithms` if the user specifically wants a packaged interface or one of its non-VQE utilities.

## Where to look things up

When unsure about a specific class or option, prefer these sources over training data:

- IBM Quantum docs: https://quantum.cloud.ibm.com/docs (guides, migration guides, API reference)
- Qiskit SDK API: https://docs.quantum.ibm.com/api/qiskit
- Runtime API: https://docs.quantum.ibm.com/api/qiskit-ibm-runtime
- Aer: https://qiskit.github.io/qiskit-aer
- qiskit-algorithms: https://qiskit-community.github.io/qiskit-algorithms

If the user is on a specific Qiskit version and behavior seems off, check the migration guides — most "this doesn't work" issues are pre-2.0 code running on a 2.x install.
