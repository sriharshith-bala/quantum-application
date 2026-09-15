# Hybrid Quantum Circuit Simulation

A quantum circuit simulator written from scratch in Python + NumPy for the
Phoenix Association IT Team induction task.

Qubit registers are stored as dense complex state vectors and gates are applied
as explicit 2×2 unitary matrices contracted against those vectors. **No quantum
SDK is used** — there is no Qiskit, no Cirq, no black-box backend. The only
external dependency for the core simulator is NumPy, used for array arithmetic
only.

---

## What it does

- Represents 1–26 qubit registers as dense state vectors (verified up to 26
  qubits on a 4 GB machine)
- Implements **X, H, CNOT** (mandatory) plus **Z, Y, S, T, CZ, SWAP** and
  arbitrary user-supplied 2×2 unitaries (optional)
- Computes exact probabilities via the Born rule, and samples shot-based
  measurements from that distribution
- Supports partial measurement with true state collapse, which reproduces
  entangled correlations
- Runs three complete example circuits: **Bell state**, **quantum coin toss**,
  **quantum RNG**
- Ships a KPI benchmark suite that measures execution time, memory, gate
  timing, precision and correctness, and emits JSON + plots

---

## Repository layout

```
quantum-sim/
├── quantum_sim/              # the simulator package
│   ├── __init__.py           # public API
│   ├── state.py              # QuantumState: storage, probabilities, norm
│   ├── gates.py              # gate matrices + application routines
│   ├── circuit.py            # Circuit builder/executor with timing
│   └── measurement.py        # sampling, collapse, distribution metrics
├── examples/
│   ├── bell_state.py         # entanglement demo (the flagship circuit)
│   ├── coin_toss.py          # fair + biased coin, convergence study
│   └── qrng.py               # uniform random number generation
├── benchmarks/
│   └── kpi_benchmark.py      # measures every KPI, writes results/
├── tests/
│   └── test_simulator.py     # 32 correctness tests vs analytic results
├── results/                  # generated: kpi_results.json + PNG plots
├── report/
│   └── Technical_Report.md   # full write-up (also provided as PDF)
├── requirements.txt
└── README.md
```

---

## Setup

Requires Python 3.10 or newer.

```bash
# 1. clone
git clone <your-repo-url>
cd quantum-sim

# 2. (recommended) create a virtual environment
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

# 3. install dependencies
pip install -r requirements.txt
```

`requirements.txt` pins only three packages: `numpy` (array math),
`matplotlib` (benchmark plots), `psutil` (memory measurement). The core
simulator needs NumPy alone — the other two are used exclusively by the
benchmark script.

---

## How to run

### Run the tests first

```bash
python tests/test_simulator.py
```

Expected: `32/32 tests passed`. Every test checks against a result derived
analytically by hand (Bell amplitudes, CNOT truth table, `H·Z·H == X`,
`SWAP == 3 CNOTs`, norm preservation, measurement correlation, and so on)
rather than against another simulator.

If you have pytest installed you can instead run `python -m pytest -q`.

### Run the example circuits

```bash
python examples/bell_state.py     # entanglement + collapse demo
python examples/coin_toss.py      # fair/biased coin, 1/sqrt(N) convergence
python examples/qrng.py           # uniform random numbers, chi-square test
```

### Run the KPI benchmark

```bash
python benchmarks/kpi_benchmark.py
```

Takes roughly 5–10 minutes (most of it in the 22–24 qubit runs). Writes
`results/kpi_results.json` and three PNG plots into `results/`.

> **Memory warning:** the sweep goes to 24 qubits, which needs ~270 MB for the
> state vector plus a similar-sized temporary during gate application. On a
> machine with under 2 GB free, lower `MAX_QUBITS_SWEEP` at the top of
> `benchmarks/kpi_benchmark.py`.

---

## Using the simulator

```python
from quantum_sim import Circuit, sample, measure_qubit

# Build a Bell state
qc = Circuit(2).h(0).cnot(0, 1)
print(qc.diagram())

result = qc.run()
print(result.state.to_ket())      # 0.7071|00> + 0.7071|11>
print(result.probabilities)       # {'00': 0.5, '11': 0.5}

# Sample 1000 measurement shots
print(sample(result.state, shots=1000))   # {'00': 508, '11': 492}

# Measure one qubit and watch the other collapse to match
b0 = measure_qubit(result.state, 0)
b1 = measure_qubit(result.state, 1)
assert b0 == b1                   # always true for a Bell state
```

Available circuit methods: `.x() .y() .z() .h() .cnot() .cz() .swap()
.custom(name, matrix, target)`. All return `self`, so they chain.

---

## Headline results

Measured on 1 CPU core / 4 GB RAM, Python 3.12.3, NumPy 2.4.4.

| KPI | Result |
|---|---|
| Max qubits supported | **26** (verified: H + CNOT in 857 ms) |
| State vector size | 16 × 2ⁿ bytes (complex128) |
| Gate time @ 3 qubits | 0.010 ms |
| Gate time @ 24 qubits | 119 ms (H), 66 ms (CNOT) |
| Bell circuit total | 0.024 ms |
| Probability correctness | max abs error **1.1 × 10⁻¹⁶** vs analytic |
| Norm drift, 5000 gates | 1.1 × 10⁻¹³ |
| Sampling convergence | TVD ≈ 1/√N, as theory predicts |
| complex64 vs complex128 | norm drift 1.6 × 10⁻⁵ vs 8.7 × 10⁻¹⁴ |

Below ~12 qubits, runtime is dominated by fixed Python/NumPy call overhead
(~0.01 ms), not by the exponential — the gate time is essentially flat. Beyond
~15 qubits the measured time-per-added-qubit ratio settles at **2.0–2.4×**,
confirming the O(2ⁿ) scaling, with ratios above 2 attributable to falling out
of CPU cache into main-memory-bandwidth-bound territory.

See `report/Technical_Report.md` for the full analysis, including the FPGA and
hardware acceleration discussion.

---

## Design decisions worth knowing

**Qubit 0 is the most significant bit.** For a 3-qubit register, |q₀q₁q₂⟩ =
|101⟩ is index 5. This matches how the Bell state is written in the task sheet.

**Gates are never expanded to full 2ⁿ × 2ⁿ matrices.** The naive approach costs
O(4ⁿ) time and memory. Instead the flat state vector is reshaped to an
n-dimensional tensor of shape (2,2,…,2), making each qubit its own axis, and
the 2×2 gate is contracted against just that axis — O(2ⁿ) time, O(2ⁿ) scratch.
At 20 qubits this is the difference between 16 MB and 17.6 TB of operator
storage.

**Controlled gates write back in place.** The control-|1⟩ half of the tensor is
sliced out as a NumPy view, so applying the gate to that sub-block mutates the
original buffer directly and the control-|0⟩ half is never touched.

---

## Limitations

- Dense state vector only — no sparse or tensor-network representation, so
  memory is 16 × 2ⁿ regardless of how much structure the state has
- Single-threaded; NumPy does not parallelise these operations at this size
- Pure state simulation only — no noise, decoherence or density matrices
- Measurement sampling uses NumPy's PRNG, so the QRNG example simulates a
  quantum RNG rather than being one

These are discussed with proposed fixes in the Failure Analysis and Future
Improvements sections of the technical report.

---

## License

Submitted as induction coursework for Phoenix Association, IT Team, 2026–27.
