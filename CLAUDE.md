# Lessons learned: VQE / quantum emulator pitfalls

Context: T1 blue-copper VQE validation project (arXiv 2609.20439, v2 correction).
These are hard-won lessons from debugging a spin-contamination bug that affected
the original paper's headline results. Read this before starting new quantum
chemistry / VQE work in this repo or a related one.

## 1. Spin symmetry (S²) is the #1 silent trap in VQE on open-shell systems

- Restricting to a fixed particle number (N) and fixed Sz is **not** the same as
  restricting to a fixed total spin S. A sector with correct N and Sz can still
  contain multiple spin manifolds (e.g. a doublet AND a quartet) mixed together.
- Unconstrained energy minimization (exact diagonalization, or ADAPT-VQE /
  HEA gradient descent) will preferentially drift toward the *lower-energy but
  unphysical* higher-spin state unless S² is explicitly controlled.
- This is a textbook quantum-mechanics/quantum-chemistry issue ("spin
  contamination"), well known in classical UHF/DFT circles — not a
  quantum-computing-specific bug. Don't assume expertise in circuits/qubits
  is sufficient to catch it; it needs the physics lens (angular momentum /
  term symbols), not just the software lens.
- **Always compute ⟨Ŝ²⟩ explicitly** on any reference energy, VQE result, or
  ansatz state before trusting it as "the" ground state — never assume a
  low-energy solution is automatically the physically correct spin state.
- Two fix strategies, different tradeoffs:
  - **Exact projection** (build ker(Ŝ+) basis, project H and generators into
    it): guarantees ⟨S²⟩ exact to numerical precision. Works well for
    classical/matrix-based ADAPT-VQE-style optimization where the state lives
    in an abstract determinant basis.
  - **Soft S²-penalty** added to the objective: necessary when the ansatz is a
    real hardware circuit (HEA) that can't be trivially restricted to an
    abstract subspace. Gives only approximate spin purity (residual ~0.76
    instead of exact 0.75) — report this honestly, don't claim exactness.
- HEA / hardware-efficient ansätze are just as susceptible to this as ADAPT-VQE
  — don't assume a "safer," more generic ansatz is automatically spin-safe.
  In this project, HEA's apparent accuracy (167 mEh) turned out to be ~33%
  quartet-contaminated once checked; the honest re-optimized number was
  370 mEh.

## 2. Numerical hygiene: distinguish real bugs from harmless noise

- `RuntimeWarning: divide by zero / overflow / invalid value encountered in
  matmul` on macOS is often a known, benign Apple Accelerate BLAS artifact
  (especially with matrices containing many exact zeros, e.g. from
  null-space/projection constructions) — NOT necessarily evidence of bad data.
- Don't trust the warning text alone either way. Verify directly:
  `np.all(np.isfinite(X))` on the actual arrays involved, both inputs and
  outputs, right at the suspect line.
- `np.seterr(all='raise')` is a useful diagnostic to convert warnings into a
  hard traceback pointing at the exact offending line — but it can behave
  inconsistently across BLAS backends (was unreliable here on macOS/Accelerate
  in one case, worked in another). Don't rely on it alone; combine with direct
  `isfinite` checks.
- **Silent guard-bypass gotcha:** a check like `if defect > 1e-9: raise ...`
  does **not** catch `defect = nan`, because `nan > 1e-9` evaluates to `False`
  in Python/NumPy. Any safety assertion comparing a possibly-NaN quantity
  must explicitly check `np.isfinite(...)` first, not just bound it with `<`/`>`.
- When re-verifying a computation with harder assertions, add them inline at
  *every* iteration of an optimization loop (not just a final check) — a bug
  can be transient and only show up mid-run.

## 3. Optimizer pitfalls found here

- Starting an optimizer at a highly symmetric point (e.g. all-zero rotation
  angles, which for `ExcitationPreserving`-type ansätze often reduces exactly
  to the identity / reference state) can produce a spuriously "flat" numerical
  gradient and cause immediate false convergence (`nit=0`). Perturb the start
  point (small random init) rather than starting exactly at zero.
- A hard discontinuity in the objective (e.g. `if leaked: return 1e3`) breaks
  gradient-based optimizers badly (finite-difference gradients near the cliff
  are meaningless) even though the optimizer may report "success." Prefer a
  smooth penalty (`+ mu * (1 - norm)**2`) over a hard cutoff.
- Always print/log optimizer diagnostics (`nit`, `nfev`, `message`) — "success"
  alone doesn't mean real work was done; check `nfev` is consistent with
  actually having explored the parameter space.

## 4. Reproducibility and cross-checks

- Re-run with the same random seed and confirm bit-identical results before
  trusting a number for publication — this caught nothing wrong here, but
  it's cheap insurance.
- When re-implementing a circuit (e.g. to recompute CNOT/depth), sanity-check
  the parameter count against the previously published value (`nparams`) as a
  structural consistency check before trusting the new depth/CNOT numbers.
- CNOT count / circuit depth are structural properties of an ansatz (which
  gates exist) and are **unaffected** by re-optimizing to different parameter
  *values* — don't recompute them unnecessarily if only the angles changed,
  but do explain this reasoning explicitly rather than silently assuming it.
- A noise-sensitivity benchmark run at "representative" (non-converged) circuit
  parameters is a real methodological weakness reviewers will (rightly) flag —
  redo it at the actual converged optimum before publishing degradation
  numbers.

## 5. Device/hardware facts specific to this project

- Euro-Q-Exa (LRZ) is a 54-qubit IQM Radiance system with a square-lattice
  "Crystal" topology (tunable couplers) — **not** 53-qubit heavy-hex, despite
  that being the label shown in some EuroHPC portal dropdowns/forms (that's
  just a catalog naming quirk, not the real hardware spec). Verify device
  specs from primary/vendor sources, not from an older draft or a portal label.
- Transpiling against an abstract fully-connected gate set (rather than the
  device's real coupling map) means reported CNOT/depth are a **lower bound**
  — don't claim "no SWAP overhead" without transpiling against the actual
  native coupling map.
