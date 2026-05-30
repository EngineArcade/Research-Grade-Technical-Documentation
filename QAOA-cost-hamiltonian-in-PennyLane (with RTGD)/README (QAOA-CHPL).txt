# PSO-QAOA Topological Superconductor Pipeline

[![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Tests: pytest](https://img.shields.io/badge/tests-pytest-green.svg)](tests/)

A research-ready Python package implementing the full pipeline:

```
Algebraic Layer (ℍ/𝕆)
  → DFT / Wannier90 → Tight-Binding H(k)
      → BdG Construction (Kagome / 8D hyperlattice)
          → GPE Toroidal BEC Vortex Lattice
              → PSO-seeded QAOA VQC Loop
                  → Tensor Network / Classical Simulators
                      → Validation Metrics
```

---

## Repository Structure

```
pso_qaoa_pipeline/
├── diagram/
│   ├── pipeline.svg              ← vector block diagram (all pipeline stages)
│   └── pipeline_diagram.pdf      ← high-resolution PDF version
├── src/
│   ├── tb_bdg/
│   │   ├── __init__.py
│   │   ├── kagome_data.py        ← Kagome geometry & example TB parameters
│   │   └── bdg_solver.py         ← BdG builder, Pfaffian Z2, edge spectrum
│   └── pso_qaoa/
│       ├── __init__.py
│       ├── pso_engine.py         ← PSO optimiser (encoding, update rules)
│       └── qaoa_circuit.py       ← QAOA VQC (Qiskit / PennyLane / Classical)
├── tests/
│   ├── conftest.py               ← sys.path setup
│   ├── test_bdg_solver.py        ← 30+ TB/BdG/Pfaffian unit tests
│   └── test_pso_qaoa.py          ← 30+ PSO/encoding/QAOA unit tests
├── LICENSE
├── README.md
└── requirements.txt
```

---

## Installation

```bash
# 1. Core dependencies (required)
pip install -r requirements.txt

# 2. Optional — Qiskit backend
pip install qiskit>=1.0.0 qiskit-aer>=0.13.0

# 3. Optional — PennyLane backend
pip install pennylane>=0.35.0

# 4. Optional — topological transport
pip install kwant>=1.4.3

# 5. Optional — tensor network simulators
pip install quimb>=1.6.0 cotengra>=0.5.0
```

---

## Quick Start

### 1 — Kagome TB → BdG (classical, zero extra install)

```python
from src.tb_bdg import get_example_kagome_strip_params, BdGSolver

data   = get_example_kagome_strip_params(N_cells=6)
solver = BdGSolver(data["H_TB"], delta=data["Delta"])
solver.solve()

print(solver.summary())
# BdGSolver | 18 sites → 36 BdG modes | gap=0.245839 | Z2=-1 | edge_wt=0.7231
```

### 2 — Plug in your own DFT / Wannier90 output

```python
from src.tb_bdg.bdg_solver import load_wannier_hr, BdGSolver

H_TB   = load_wannier_hr("path/to/wannier90_hr.dat")   # ← your file here
solver = BdGSolver(H_TB, delta=0.2, pairing_type="swave", mu=-0.5)
solver.solve()
print(solver.summary())
```

### 3 — PSO-seeded QAOA (classical backend)

```python
from src.pso_qaoa import run_pso_qaoa

result = run_pso_qaoa(
    n_sites     = 6,          # Kagome sites
    p_layers    = 2,          # QAOA depth
    backend     = "classical",
    delta_seed  = 0.3,        # BdG pairing seed
    n_particles = 30,
    n_iter      = 100,
    seed        = 42,
)

gate_types, thetas = result["decoded_params"]
print(f"Best energy : {result['g_best_cost']:.6f}")
print(f"Converged   : {result['converged']}")
```

### 4 — Full end-to-end pipeline

```python
from src.tb_bdg import get_example_kagome_strip_params, BdGSolver
from src.pso_qaoa import run_pso_qaoa

# Step 1 — BdG
data   = get_example_kagome_strip_params(N_cells=6)
solver = BdGSolver(data["H_TB"], delta=data["Delta"])
solver.solve()

# Step 2 — PSO-QAOA seeded with BdG gap
result = run_pso_qaoa(
    delta_seed  = solver.gap(),
    backend     = "classical",
    n_particles = 30,
    n_iter      = 100,
)
print("Pipeline complete. Best VQC energy:", result["g_best_cost"])
```

### 5 — Run the test suite

```bash
pytest tests/ -v
# Expected: 60+ tests, all passing
```

---

## Module Reference

### `tb_bdg`

| Symbol | Description |
|---|---|
| `build_kagome_strip(N_cells, t, mu, t2)` | Finite strip H_TB (3N×3N) |
| `kagome_bulk_hamiltonian(kx, ky, t, mu)` | Bulk Bloch H(k) (3×3) |
| `get_example_kagome_strip_params(N_cells)` | Ready-to-use data dict |
| `load_wannier_hr(filepath)` | Parse Wannier90 hr.dat → H(R=0) |
| `BdGSolver(H_TB, delta, pairing_type, mu)` | Full BdG solver |
| `BdGSolver.solve()` | Diagonalise; returns (eigenvalues, eigenvectors) |
| `BdGSolver.gap()` | Minimum positive quasi-particle energy |
| `BdGSolver.z2_invariant()` | Z2 Pfaffian invariant (+1 trivial / -1 topo.) |
| `BdGSolver.edge_spectral_weight(n_edge, window)` | Edge-localised spectral weight |
| `BdGSolver.density_of_states(E_grid, sigma)` | Gaussian-broadened DOS |
| `pfaffian(A)` | Pfaffian of real skew-symmetric matrix |

### `pso_qaoa`

| Symbol | Description |
|---|---|
| `PSOConfig(...)` | Hyperparameter dataclass |
| `PSOEngine(fitness_fn, config, dim, lb, ub)` | PSO optimiser |
| `PSOEngine.run()` | Full optimisation loop → result dict |
| `encode_particle(gate_types, thetas)` | Pack into flat PSO vector |
| `decode_particle(x)` | Unpack → (gate_types, thetas) |
| `particle_bounds(n_gates)` | lb / ub arrays |
| `KagomePairingCircuit(...)` | QAOA circuit for 6-site Kagome BdG |
| `KagomePairingCircuit.fitness(x)` | PSO hook → ⟨H_cost⟩ |
| `ClassicalBackend(n_qubits)` | Numpy statevector (no extra install) |
| `QiskitBackend(n_qubits, shots)` | Qiskit Aer evaluator |
| `PennyLaneBackend(n_qubits, shots)` | PennyLane default.qubit |
| `run_pso_qaoa(...)` | One-call convenience pipeline |

---

## PSO Hyperparameter Guide

| Parameter | Symbol | Recommended | Effect |
|---|---|---|---|
| Swarm size | N | 30 | ↑ → better exploration, slower |
| Inertia weight | ω | 0.73 (decaying 0.9→0.4) | ↑ → more exploration |
| Cognitive coeff. | c₁ | 1.496 | Trust personal best |
| Social coeff. | c₂ | 1.496 | Trust swarm best |
| Max velocity | v_max | 20 % of range | ↑ → larger jumps |
| Iterations | T | 100 | ↑ → finer refinement |

---

## QAOA Particle Encoding

```
x = [g₀, θ₀, g₁, θ₁, ..., g_{G-1}, θ_{G-1}]
     │    │
     │    └─ rotation angle θ ∈ [0, 2π]  (continuous)
     └──── gate type g ∈ {0=RX, 1=RY, 2=RZ, 3=CX, 4=CZ}  (discrete, rounded)

QAOA layer:
  |+>^n  →  [Cost: RZ(2γJ_ij) + CZ on bonds]  →  [Mixer: gate(g, θ)]  →  measure
```

---

## Extending the Package

### Custom p-wave pairing

```python
import numpy as np
from src.tb_bdg import BdGSolver, build_kagome_strip

H_TB  = build_kagome_strip(N_cells=8, mu=-1.5)
n     = H_TB.shape[0]
Delta = np.zeros((n, n), dtype=complex)
for i in range(n - 1):
    Delta[i, i+1] =  0.2j   # chiral p-wave
    Delta[i+1, i] = -0.2j

solver = BdGSolver(H_TB, delta=Delta, pairing_type="custom")
solver.solve()
print("Z2 =", solver.z2_invariant())
```

### Qiskit backend with noise model

```python
from qiskit_aer.noise import NoiseModel
from qiskit_ibm_runtime.fake_provider import FakeNairobi

noise_model = NoiseModel.from_backend(FakeNairobi())
from src.pso_qaoa.qaoa_circuit import KagomePairingCircuit, QiskitBackend
circuit = KagomePairingCircuit(backend="qiskit", shots=1024)
circuit._backend.noise_model = noise_model
```

---

## Citation

```bibtex
@software{pso_qaoa_topo_2026,
  title   = {PSO-QAOA Topological Superconductor Pipeline},
  author  = {Quinn, Robert},
  year    = {2026},
  license = {MIT},
}
```

---

## License

MIT License — Copyright (c) 2026 Robert Quinn — see [LICENSE](LICENSE).
