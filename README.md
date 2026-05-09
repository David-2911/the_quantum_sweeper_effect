# Quantum Sweeper Effect Simulation

A computational investigation of the Quantum Sweeper Effect (QSE) grounded in the
superclassical thermodynamic framework of Grössing et al., implemented as a
publication-quality Jupyter notebook in Python.

---

## Overview

This project numerically simulates the Quantum Sweeper Effect: a predicted anomalous
beam behaviour in which extreme attenuation of one slit in a double-slit experiment
causes the surviving particles from that slit to be laterally compressed and deflected
by the dominant beam, rather than simply fading into the background. The simulation
pursues two goals in parallel. First, it verifies that the Bohmian guidance equation
and the superclassical current algebra of Grössing et al. produce identical velocity
fields, to machine precision, across five orders of magnitude of slit attenuation.
Second, it integrates Bohmian streamlines across the full attenuation range to
visualise and quantify the sweeper morphology.

The core research question motivating this work is:

> Can the pilot wave in de Broglie-Bohm theory be interpreted as a superclassical
> sub-quantum thermodynamic medium, and does that interpretation accurately predict
> anomalous beam deflection under extreme attenuation?

---

## Physical Setup

The simulation follows the geometry specified in Grössing et al. (arXiv:1406.1346,
arXiv:1502.04034). Two Gaussian wavepackets are emitted from adjacent slits and
propagate freely to a screen 5 metres away.

| Parameter | Symbol | Value |
|---|---|---|
| Initial beam waist | σ₀ | 22 µm |
| Slit separation | d | 200 µm |
| Propagation distance | L | 5 m |
| de Broglie wavelength | λ | 1.8 nm |
| Beam velocity | v₀ | ~220 m/s |
| Rayleigh range | y_R | ~3.4 m |

The transmission factor `a` of slit 2 is swept across five values:

```python
a_values = [1, 1e-2, 1e-4, 1e-6, 1e-10]
```

At `a = 1` the experiment is symmetric and produces standard double-slit interference.
At `a = 1e-10` the weak beam is attenuated by five orders of magnitude, placing the
simulation in the extreme sweeper regime.

---

## Theory

### The Two Velocity Formulations

The simulation computes the transverse velocity field via two independent routes and
verifies their agreement.

**Superclassical formulation** (Grössing et al. Eq. 2.10):

```
v_x,SC = J_tot,x / P_tot
```

**Bohmian guidance equation:**

```
v_x,Bohm = (ℏ/m) · Im(∂ₓψ / ψ)
```

### The Four-Term Current Decomposition

The total probability current `Jx` is decomposed into four physically distinct
contributions:

| Term | Expression | Physical meaning |
|---|---|---|
| `term1` | `R1² · v1x` | Slit-1 self-current (full amplitude) |
| `term2` | `R2² · v2x` | Slit-2 self-current (attenuated by √a) |
| `term3` | `R1·R2·(v1x + v2x)·cos(φ)` | Conventional interference |
| `term4` | `R1·R2·(u1x - u2x)·sin(φ)` | Diffusive osmotic interference (sweeper driver) |

The osmotic velocity difference driving `term4` is:

```
u1x - u2x = ℏd / (2mσ²)
```

This quantity is strictly independent of the attenuation parameter `a`. As the
conventional interference term decays with √a, the osmotic driver maintains
full geometric strength. This asymmetry is the physical mechanism behind the Quantum
Sweeper Effect.

### Trajectory Integration

Streamlines are integrated using the **spatial ODE formulation**:

```
dx/dy = vx / vy
```

rather than the temporal form `dQ/dt = vx`. Because `vy = v0` exactly under the
`y = v0·t` coordinate convention adopted by Grössing et al., the denominator never
vanishes regardless of how small `P_tot` becomes at extreme attenuation.
This eliminates the numerical singularities that make the temporal form unstable in
the sweeper regime.

---

## Repository Structure

```
quantum_sweeper_effect/
├── 01_quantum_sweeper.ipynb    # Main simulation notebook (all sections)
├── outputs/                    # Generated figures (created at runtime)
│   ├── fig_density_trajectory_heatmap_4panel.png
│   └── ...
└── README.md                   # This file
```

---

## Notebook Structure

The notebook is organised into fourteen numbered sections with a clear progression
from setup through physics through validation.

| Sections | Content |
|---|---|
| 1 | Introduction, research question, terminology, geometry conventions |
| 2 to 4 | Physical constants, computational grid, probability density `P_tot` |
| 5 | Equivalence verification: superclassical vs. Bohmian velocity fields |
| 6 to 10 | Sweeper signature: heat-flow driver, streamlines, attenuation sweep |
| 11 to 14 | Validation suite, regime maps, interpretation and conclusion |

---

## Computational Details

### Integrator

Trajectories are integrated with `scipy.integrate.solve_ivp` using the DOP853
solver (explicit 8th-order Runge-Kutta with adaptive step size).

```python
solve_ivp(
    fun=lambda y, x: [trajectory_rhs(y, x[0], p, a_val)],
    method='DOP853',
    rtol=1e-10,
    atol=1e-12,
    max_step=L / 1000.0,
)
```

### Spatial Grid

The y-grid uses 1,350 points distributed non-uniformly: dense near the slit plane
where gradients are steepest, coarser in the far field.

```python
y_traj_arr = np.concatenate([
    np.linspace(1e-6, 0.005, 200),   # near-field (dense)
    np.linspace(0.005, 0.02,  150),
    np.linspace(0.02,  0.2,   300),
    np.linspace(0.2,   1.5,   400),
    np.linspace(1.5,   L,     300),   # far-field
])
```

### Trajectory Seeds

60 seeds per slit family are drawn from Born-rule Gaussian distributions
(σ = σ₀) centred on each slit position, then clipped to enforce
strict side separation.

```python
x0_slit1 = np.clip(sample_born(-d/2, sig0, N_TRAJ), -d/2 - 4*sig0, -1e-7)
x0_slit2 = np.clip(sample_born( d/2, sig0, N_TRAJ),  1e-7,  d/2 + 4*sig0)
```

### Custom Colour Maps

Two publication-quality colormaps are defined and registered at runtime.

`fig_warm`: a sequential pale-yellow-to-dark-red map used for probability density,
osmotic pressure, and all strictly non-negative scalars. The low end is a visible
pale yellow rather than white so that low-density structure does not vanish into
the white axis background.

`fig_warm_div`: a symmetric steel-blue-to-deep-red diverging map used for signed
scalars such as the heat-flow field and the transverse velocity. Zero maps to
near-white and is unambiguous against the white figure background.

---

## Validation

The notebook includes a structured validation suite of ten test groups covering
all core physical invariants. Every group prints a PASS/FAIL result and the final
summary reports total counts.

| Test | What it checks |
|---|---|
| T1 | Probability density is non-negative everywhere |
| T2 | Phase field φ is antisymmetric and zero on the axis |
| T3 | `vy = v0` exactly, derived from the structural Jy decomposition |
| T4 | Bohmian-superclassical equivalence below relative error `1e-10` at all test points |
| T5 | Contrast law `(1 + √a)² / 4` at the on-axis screen position across all five a-values |
| T6 | Heat-flow map Q_heat is antisymmetric and scales as √a |
| T7 | Seeds are correctly separated and the y-grid is strictly monotonic |
| T8 | Streamline families at a=1 fan outward symmetrically with correct sign |
| T9 | Slit-2 beam is spatially compressed at a=1e-10 relative to the a=1 baseline |
| T10 | Sweeper deflection ratio grows monotonically and exceeds 1000x across the attenuation range |

A clean run produces:

```
══════════════════════════════════════════════════════════
  Total: 33  |  Passed: 33  |  Failed: 0
  ✓  ALL TESTS PASSED : Simulation Verified.
══════════════════════════════════════════════════════════
```

---

## Key Results

The simulation reproduces the Quantum Sweeper Effect with strong quantitative
agreement with the predictions of Grössing et al.

At `a = 1e-10`, the attenuated slit-2 beam undergoes **4.5-fold spatial compression**
relative to the symmetric baseline, and the transverse deflection ratio at the
midfield point grows by over three orders of magnitude. The strict no-crossing
property of Bohmian trajectories is preserved across all five attenuation levels.
The superclassical and Bohmian velocity fields agree at every tested spatial point
and attenuation value to below `1e-10` relative error.

---

## Dependencies

The notebook runs on standard scientific Python. No custom packages are required.

| Package | Role |
|---|---|
| `numpy` | Array operations and numerical core |
| `scipy` | ODE integration (`solve_ivp`, DOP853) |
| `matplotlib` | All plotting and figure export |

Install with:

```bash
pip install numpy scipy matplotlib
```

The notebook was developed and tested under **Python 3.12.3** in a dedicated conda
environment. It is compatible with both local Jupyter and Google Colab; the setup
cell detects the runtime automatically and resolves the output directory accordingly.

---

## Running the Notebook

Clone or download the repository, then open the notebook in Jupyter:

```bash
jupyter notebook 01_quantum_sweeper.ipynb
```

Run all cells in order from Section 1 through Section 14. Each section includes
assertion guards that confirm required state from earlier sections is present before
proceeding, so out-of-order execution will raise an explicit error rather than
silently producing wrong results.

Generated figures are saved to the `outputs/` directory at publication quality
(300 DPI, tight bounding box, white background).

---

## References

**[P1]** Grössing, Fussy, Mesa Pascasio, Schwabl. *Extreme beam attenuation in
double-slit experiments.* arXiv:1406.1346 (2014).

**[P2]** Grössing, Fussy, Mesa Pascasio, Schwabl. *The Quantum Sweeper Effect.*
arXiv:1502.04034 (2015).

---

## Licence

This project was developed as part of the BeyondQuantum research programme.
Please contact the author before reproducing or adapting any portion of this work.
