# AGENTS.md

## Project mission

Build an open-source web application that lets users explore how simple interatomic potentials give rise to three-dimensional crystal structures.

The first scientific target is a Lennard-Jones solid, with Argon as the motivating example. The application should support both:

1. **Ideal lattice analysis** — evaluate and minimize the energy of candidate periodic lattices such as FCC, HCP, BCC, and simple cubic.
2. **Particle relaxation / molecular simulation** — start from an arbitrary set of particles and numerically evolve or minimize the system while visualizing the resulting structure.

Authoritative scientific computation should run **server-side in `f64`**, with CUDA acceleration where useful. The browser is primarily responsible for visualization, interaction, parameter editing, and job orchestration.

The long-term goal is not merely an Argon demo. The repository should evolve into a clean computational-physics platform whose first application is crystal formation.

---

## Core principles

### 1. Correctness before acceleration

The CPU `f64` implementation is the reference implementation and correctness oracle.

CUDA implementations must reproduce the same physical model and should be tested against the CPU backend on small systems.

Do not create separate, subtly different CPU and GPU physics models.

### 2. `f64` is authoritative

Scientific calculations that determine energies, forces, equilibrium lattice constants, minimization trajectories, or reported numerical results should use double precision unless a specific algorithm has been validated with lower precision.

Client-side `f32` computation may eventually be used for previews, rendering, or non-authoritative experiments, but browser GPU limitations must never silently reduce the precision of scientific results.

### 3. Separate physics from execution backend

Potential functions, boundary conditions, integrators, minimizers, simulation state, and analysis code should not depend directly on CUDA.

The compute backend should be replaceable.

Target architecture:

```text
Web UI
  |
  v
API / simulation service
  |
  v
Simulation + analysis layer
  |
  +------------------+
  |                  |
  v                  v
CPU f64 backend   CUDA f64 backend
(reference)       (accelerated)
```

### 4. Build the simplest correct version first

Start with `O(N^2)` pair-force evaluation.

Do not implement neighbor lists, spatial sorting, elaborate GPU memory layouts, distributed computation, or a general GPU compiler until the end-to-end application works.

### 5. Keep the first physical model simple

The first model is the 12-6 Lennard-Jones potential:

\[
V(r)=4\epsilon\left[\left(\frac{\sigma}{r}\right)^{12}-\left(\frac{\sigma}{r}\right)^6\right].
\]

Use reduced Lennard-Jones units initially:

- \(\epsilon = 1\)
- \(\sigma = 1\)
- particle mass \(m = 1\)

Do not hard-code physical Argon parameters without a cited and versioned source. Add real-unit presets only after the dimensionless implementation is tested.

### 6. Avoid overclaiming physical realism

Lennard-Jones Argon is an educational/model system, not a complete first-principles description of solid Argon.

The UI and documentation should distinguish:

- model predictions,
- numerical approximations,
- experimentally measured quantities,
- and first-principles/electronic-structure calculations.

A finite cluster is not guaranteed to relax into the same structure as an infinite bulk crystal. Boundary conditions, system size, initialization, temperature, pressure, and minimization procedure matter.

---

# Proposed repository layout

Prefer a Rust workspace for the server and scientific code.

```text
/
├── AGENTS.md
├── README.md
├── Cargo.toml
├── crates/
│   ├── model/              # shared physical types, units, simulation state
│   ├── potentials/         # Lennard-Jones and future pair potentials
│   ├── simulation/         # minimizers, integrators, PBC, orchestration
│   ├── compute/            # backend traits and shared compute interfaces
│   ├── compute-cpu/        # authoritative f64 CPU backend
│   ├── compute-cuda/       # CUDA backend wrapper
│   ├── analysis/           # RDF, lattice metrics, structural analysis
│   └── api-server/         # HTTP/WebSocket server
├── native/
│   └── cuda/               # CUDA kernels if compiled outside Rust
├── web/
│   ├── src/
│   └── ...
├── fixtures/
│   ├── lattices/
│   └── simulations/
└── docs/
    ├── architecture.md
    ├── physics.md
    └── benchmarks.md
```

This is a suggested organization, not a requirement. Prefer clarity over preserving this exact tree.

If CUDA kernels are initially written in CUDA C++ and called from Rust through FFI, keep that boundary narrow and explicit. Experimental Rust-to-PTX approaches may be investigated later, but they must not block the scientific application.

---

# Technology direction

## Server

Preferred baseline:

- Rust
- Tokio
- Axum or another minimal async HTTP framework
- Serde for request/result schemas
- WebSocket for streaming simulation snapshots

The server should own:

- validation,
- job lifecycle,
- simulation execution,
- GPU selection,
- numerical results,
- progress reporting,
- and persistence/export where added.

## Compute

Initial backends:

1. `CpuBackend`
2. `CudaBackend`

The backend interface should expose domain operations rather than generic arbitrary kernels.

Prefer interfaces conceptually similar to:

```rust
trait ComputeBackend {
    fn compute_forces(
        &self,
        state: &ParticleState,
        model: &InteractionModel,
        boundary: &BoundaryCondition,
        out: &mut ForceBuffer,
    ) -> Result<EnergySummary>;

    fn backend_info(&self) -> BackendInfo;
}
```

Do not prematurely create a large generic GPU framework.

## CUDA

For the first accelerated implementation, prioritize reliability and tooling over ideological purity.

Acceptable approaches include:

- Rust host code calling a small CUDA library through FFI,
- a mature Rust CUDA binding,
- or a Rust CUDA compiler path if it proves stable enough.

The repository must not depend on an experimental compiler solely to satisfy a "100% Rust" goal.

CUDA kernel implementations should use `f64` for authoritative calculations.

## Browser

Suggested baseline:

- TypeScript
- React
- Three.js for atom/lattice visualization

Keep rendering isolated from scientific state.

The visualization should consume simulation snapshots supplied by the server rather than reconstructing scientific calculations independently.

---

# Scientific model

## Lennard-Jones potential

Implement the pair potential and radial derivative from a single well-tested module.

Required analytic checks:

At

\[
r_\mathrm{min}=2^{1/6}\sigma,
\]

the potential satisfies

\[
V(r_\mathrm{min})=-\epsilon
\]

and the radial force is zero.

The implementation must also guard against invalid or near-zero pair separation.

## Force convention

Document the force sign convention once and test it.

The implementation should satisfy:

\[
\mathbf F_i=-\nabla_i U.
\]

For each isolated pair:

\[
\mathbf F_{ij}=-\mathbf F_{ji}.
\]

## Reduced units

Use reduced units through the initial milestones.

Avoid mixing SI units, Ångström, Kelvin, electron volts, or reduced units inside the same low-level API.

If physical-unit presets are added later, conversion must occur at a clear boundary.

## Boundary conditions

Support these incrementally:

1. finite/no-boundary cluster,
2. cubic periodic boundary conditions,
3. generalized simulation boxes only if later required.

Periodic calculations should use the minimum-image convention where valid.

## Cutoffs

The earliest implementation may evaluate all particle pairs.

When a cutoff is introduced, make the truncation policy explicit.

Supported policies may eventually include:

- hard truncation,
- energy-shifted potential,
- force-shifted potential.

Never change cutoff behavior silently because it changes the physical model.

---

# Ideal lattice analysis

The first application mode should analyze predefined candidate lattices.

Initial structures:

- FCC
- HCP
- BCC
- simple cubic

For each structure:

1. generate lattice sites,
2. evaluate energy per particle for a periodic/infinite approximation,
3. vary lattice scale,
4. locate the minimum,
5. report equilibrium reduced lattice parameter and minimum energy,
6. visualize the lattice and relevant neighbor shells.

Where the Lennard-Jones lattice energy can be expressed through precomputed lattice sums, provide both:

- a direct numerical summation implementation,
- and an analytic/minimized formulation used as a cross-check.

Do not hard-code "FCC wins" into the algorithm. Candidate structures must be evaluated by the model.

---

# Particle simulation modes

## Energy minimization

Implement minimization before full molecular dynamics.

Suggested progression:

1. steepest descent / damped gradient descent,
2. FIRE minimization,
3. optional conjugate-gradient or L-BFGS only if useful.

The minimizer must report:

- step number,
- total potential energy,
- maximum force magnitude,
- RMS force,
- convergence status.

Convergence criteria must be configurable and recorded with results.

## Molecular dynamics

Add MD only after minimization is reliable.

Initial integrator:

- velocity Verlet.

Later optional features:

- simple thermostat,
- annealing schedule,
- pressure/barostat,
- restart state.

Energy conservation should be tested in an appropriate microcanonical configuration.

---

# Server API direction

Keep the first API minimal.

Possible shape:

```text
POST /api/v1/lattice/analyze
POST /api/v1/simulations
GET  /api/v1/simulations/{id}
POST /api/v1/simulations/{id}/cancel
WS   /api/v1/simulations/{id}/stream
GET  /api/v1/backends
```

A simulation request should include explicit values for:

- model/potential,
- particle count or initial coordinates,
- simulation box,
- boundary condition,
- minimizer/integrator,
- timestep where relevant,
- convergence criteria,
- cutoff policy,
- random seed,
- requested backend.

The server should return all configuration needed to reproduce the run.

Do not make hidden defaults part of the scientific result. Defaults may exist in the UI, but the resolved configuration should be serialized into each job/result.

---

# Streaming and visualization

Do not stream every integration step to the browser.

Scientific timesteps and visualization frames are different concepts.

The server should support configurable snapshot intervals, for example every N minimizer or MD steps.

A streamed frame should contain only what the visualizer needs, typically:

- step/time,
- particle positions,
- box dimensions,
- energy summary,
- convergence/progress values.

Use binary framing later if JSON becomes a bottleneck. JSON is acceptable for the first prototype.

The frontend should support:

- orbit/pan/zoom,
- atom radius scaling,
- periodic box visibility,
- energy plot,
- start/pause/cancel controls,
- potential/model controls,
- structure presets,
- progress/convergence display.

Do not let rendering FPS control simulation timestep.

---

# Numerical validation requirements

Every optimization must be validated against a simpler implementation.

## Unit tests

Required early tests include:

- Lennard-Jones value at known points,
- zero force at the LJ minimum,
- force direction for particles closer/farther than equilibrium,
- equal and opposite pair forces,
- finite-difference gradient check,
- lattice generator neighbor distances,
- periodic minimum-image calculations.

## CPU reference tests

Maintain tiny fixtures with known coordinates.

For each fixture compare:

- total energy,
- per-particle or global force,
- minimizer behavior,
- periodic vs non-periodic results.

## CPU vs CUDA

For small deterministic systems, CUDA results should agree with the CPU reference within documented tolerances.

Initial target in reduced units:

- component-wise force `rtol` around `1e-10`,
- `atol` around `1e-12`,
- energy tolerances chosen to account for reduction order.

Do not force bitwise equality across CPU and GPU reductions.

If tolerances need to be relaxed, document why.

## Conservation / invariants

Where applicable test:

- total pair force approximately zero for isolated systems,
- symmetry under translation,
- symmetry under permutation of particle ordering,
- periodic translation invariance,
- energy behavior during minimization,
- energy conservation for stable NVE MD runs.

---

# Reproducibility

Every stochastic run must accept a random seed.

A stored result should include at minimum:

- application version / git revision if available,
- backend,
- GPU/device metadata,
- precision,
- resolved simulation configuration,
- seed,
- start state or reproducible initializer,
- convergence reason,
- final energy,
- final coordinates.

GPU reductions may introduce minor floating-point ordering differences. Reproducibility means scientifically equivalent and tolerance-tested results, not necessarily bit-for-bit identical trajectories.

---

# Performance strategy

Optimize only after profiling.

## Phase 1: brute force

Use pairwise `O(N^2)` interactions.

This is the required baseline because it is:

- easy to inspect,
- easy to test,
- appropriate for small visual demos,
- and ideal for CPU/GPU cross-validation.

## Phase 2: cutoff

Introduce an interaction cutoff only after the baseline is correct.

Measure the model change separately from the performance improvement.

## Phase 3: cell lists / neighbor lists

For larger systems implement:

1. cell assignment,
2. particle bin counts,
3. prefix sum / offsets,
4. particle reordering or indexed bins,
5. neighboring-cell traversal,
6. optional Verlet neighbor lists with a skin distance.

Keep a brute-force validation path for small systems.

## Phase 4: GPU reductions and memory optimization

Profile:

- global memory traffic,
- occupancy,
- divergent branches,
- reductions,
- host-device transfers,
- snapshot copies.

Avoid premature use of shared-memory tiling or exotic layouts unless benchmarks show a meaningful gain.

---

# Milestones

## Milestone 0 — Repository and correctness scaffold

### Goal

Create a clean workspace that can support numerical code, CUDA, a server, and a browser client.

### Deliverables

- Rust workspace.
- Basic web application.
- CI for CPU-only builds/tests.
- formatting and lint checks.
- shared error-handling conventions.
- `docs/physics.md` describing reduced units and LJ conventions.
- test fixture infrastructure.
- backend trait with CPU backend only.

### Acceptance criteria

- repository builds on a machine without CUDA,
- CPU unit tests run in CI,
- web app launches,
- CUDA is optional at build/run time.

---

## Milestone 1 — Lennard-Jones and ideal lattice explorer

### Goal

Produce the first scientifically meaningful result without GPU dependency.

### Deliverables

- `f64` Lennard-Jones implementation,
- FCC/HCP/BCC/simple-cubic lattice generation,
- energy-per-particle evaluation,
- lattice-parameter sweep,
- scalar minimization of lattice scale,
- basic web visualization,
- energy-vs-lattice-parameter plot,
- exported resolved calculation configuration.

### Acceptance criteria

- potential passes analytic unit tests,
- candidate lattices give stable converged energy curves,
- numerical and analytic/lattice-sum approaches cross-check where implemented,
- UI can switch structures and show their equilibrium geometry.

---

## Milestone 2 — CPU particle relaxation

### Goal

Animate arbitrary particles relaxing under the Lennard-Jones potential.

### Deliverables

- particle state representation,
- brute-force `O(N^2)` CPU forces,
- finite and cubic periodic boundaries,
- steepest-descent minimizer,
- FIRE minimizer,
- deterministic random initialization,
- simulation snapshots,
- convergence metrics.

### Acceptance criteria

- finite-difference force checks pass,
- total energy decreases appropriately during minimization,
- fixed-seed runs are reproducible within the chosen model,
- small systems can be rendered as an animation.

---

## Milestone 3 — Simulation service and interactive web UI

### Goal

Make server-side computation the authoritative execution path.

### Deliverables

- HTTP job creation,
- job status endpoint,
- cancellation,
- WebSocket snapshot stream,
- frontend controls for model/run parameters,
- 3D atom visualization,
- live energy/convergence plot,
- result metadata panel.

### Acceptance criteria

- browser performs no authoritative force/energy calculation,
- reconnecting to a running job is possible or cleanly reported as unsupported,
- invalid or excessive inputs are rejected,
- UI remains responsive while simulations run.

---

## Milestone 4 — CUDA brute-force backend

### Goal

Accelerate the same `O(N^2)` calculation with server-side `f64` CUDA.

### Deliverables

- CUDA backend discovery,
- pair-force kernel,
- energy reduction,
- device buffers,
- backend selection,
- CPU/CUDA comparison tests,
- simple benchmark suite.

### Acceptance criteria

- CUDA is optional,
- CPU and CUDA agree within documented tolerances,
- correctness tests run automatically on CUDA-enabled development machines,
- GPU path demonstrates a measurable speedup for sufficiently large N,
- scientific API does not change when switching backend.

---

## Milestone 5 — Cutoffs and scalable neighbor search

### Goal

Move beyond brute-force pair enumeration.

### Deliverables

- explicit cutoff configuration,
- selected cutoff/shift policy,
- cell-list implementation on CPU,
- GPU cell binning,
- prefix-sum/offset construction,
- neighbor-cell traversal,
- optional Verlet neighbor list,
- brute-force cross-validation mode.

### Acceptance criteria

- optimized and brute-force implementations agree for equivalent models,
- cutoff behavior is documented and visible in run metadata,
- benchmark documents crossover sizes,
- no regression in small-system correctness.

---

## Milestone 6 — Molecular dynamics and annealing

### Goal

Allow users to explore temperature-dependent evolution and crystallization.

### Deliverables

- velocity Verlet,
- kinetic energy and temperature calculation,
- NVE mode,
- simple thermostat,
- annealing schedule,
- restartable simulation state,
- trajectory controls.

### Acceptance criteria

- energy conservation is characterized for NVE,
- timestep stability is tested,
- annealing parameters are stored with results,
- UI clearly distinguishes minimization from dynamics.

---

## Milestone 7 — Structural analysis

### Goal

Explain what structure formed rather than only drawing it.

### Deliverables

- radial distribution function \(g(r)\),
- coordination-number analysis,
- nearest-neighbor shell metrics,
- lattice comparison metrics,
- optional common-neighbor analysis or bond-order parameters later.

### Acceptance criteria

- analysis routines have synthetic lattice fixtures,
- known ideal lattices produce expected qualitative signatures,
- UI can display analysis alongside the geometry,
- classification is presented with uncertainty/limitations rather than as an infallible label.

---

## Milestone 8 — Physical Argon preset and scientific documentation

### Goal

Move from generic LJ reduced units to an educational Argon mode.

### Deliverables

- documented source for Argon LJ parameters,
- unit conversion layer,
- real-unit display,
- explanation of model limitations,
- comparison against selected reference values where appropriate,
- citation metadata.

### Acceptance criteria

- reduced-unit core remains unchanged,
- conversions are tested,
- every physical preset is sourced,
- UI clearly labels calculated vs reference/experimental values.

---

## Milestone 9 — Performance hardening

### Goal

Make larger interactive simulations practical.

### Deliverables

- benchmark corpus,
- GPU profiling notes,
- optimized data layout,
- asynchronous snapshot transfer where useful,
- binary stream protocol if justified,
- device-memory budgeting,
- server-side quotas.

### Acceptance criteria

- performance claims are backed by reproducible benchmarks,
- profiling identifies the reason for each significant optimization,
- no optimization bypasses correctness tests.

---

## Milestone 10 — Portable compute research track

### Goal

Only after the application is established, investigate whether a reusable Rust-native GPU abstraction is worth extracting.

Possible directions:

- CubeCL backend experiments,
- Rust-to-PTX approaches,
- WebGPU preview backend,
- shared scientific-kernel DSL,
- portable reduction/scan primitives.

This milestone is intentionally non-critical.

Do not rewrite the working CUDA path merely to pursue backend elegance.

A separate reusable compute library should be extracted only when at least two real application components require the abstraction.

---

# Definition of done for scientific features

A scientific feature is not done merely because it renders correctly.

It should include, as appropriate:

1. mathematical convention documented,
2. implementation,
3. unit tests,
4. reference/cross-check implementation,
5. numerical tolerance,
6. representative fixture,
7. API serialization,
8. UI exposure,
9. reproducibility metadata,
10. documentation of limitations.

---

# Security and resource limits

A public CUDA-backed compute server is also a denial-of-service target.

The API must eventually enforce limits on:

- maximum particle count,
- maximum steps,
- wall-clock duration,
- concurrent jobs,
- GPU memory use,
- snapshot frequency,
- request payload size.

Do not allow clients to upload arbitrary CUDA kernels or execute arbitrary code.

Treat compute requests as structured simulation specifications only.

Cancellation should release GPU and host resources promptly.

---

# Error handling

Prefer explicit typed errors.

Separate:

- invalid scientific configuration,
- numerical failure,
- convergence failure,
- CUDA unavailable,
- CUDA execution failure,
- resource limit exceeded,
- user cancellation,
- internal server failure.

A minimizer reaching its iteration limit is not necessarily an internal error; return a valid result with an explicit non-converged status.

Do not silently replace a requested CUDA backend with CPU unless the request explicitly allows fallback.

---

# Logging and observability

Record enough information to debug numerical problems without dumping entire trajectories.

Useful structured fields include:

- job id,
- backend,
- device,
- particle count,
- step,
- energy,
- max force,
- elapsed compute time,
- snapshot count,
- convergence reason.

Avoid logging complete user-provided coordinate arrays by default.

---

# Coding guidance for agents

## General

- Prefer small focused modules.
- Keep physics code deterministic and side-effect-light.
- Avoid unsafe Rust except at narrow FFI/CUDA boundaries.
- Document every `unsafe` block with its invariant.
- Avoid unnecessary dependencies.
- Do not introduce a framework when a small library is sufficient.
- Do not combine unrelated refactors with scientific changes.

## Numerical code

- Use descriptive names and explicit units.
- Avoid unexplained magic constants.
- Make tolerance choices visible.
- Be careful with catastrophic cancellation and reduction order.
- Never compare floats for exact equality unless the value is mathematically constructed to be exact in that representation and the reason is documented.
- Prefer numerically stable reductions when they materially improve accuracy.

## CUDA code

- Start from a readable kernel.
- Preserve a direct CPU analogue.
- Add performance complexity only with benchmark evidence.
- Check every CUDA API error.
- Keep allocation outside hot loops.
- Minimize host-device transfers.
- Never optimize by silently switching to `f32`.

## Frontend

- Keep simulation state separate from rendering state.
- Do not duplicate scientific formulas in the UI unless used only for labels/previews.
- Make resolved server configuration inspectable.
- Ensure visual interpolation does not modify actual simulation data.
- Prefer clear scientific controls over decorative UI complexity.

---

# Testing and CI

At minimum, CPU CI should run:

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Add frontend commands once the package manager is established, for example:

```bash
npm test
npm run lint
npm run build
```

CUDA tests should be a separate optional CI lane because ordinary contributors may not have NVIDIA hardware.

Never make the default repository test suite require CUDA.

---

# Benchmarking

Do not benchmark only one particle count.

Maintain representative sizes such as:

- tiny correctness case,
- small interactive case,
- medium GPU crossover case,
- large throughput case.

Record:

- CPU model,
- GPU model,
- CUDA/toolchain version,
- precision,
- particle count,
- boundary/cutoff mode,
- steps,
- elapsed time,
- interactions per second where meaningful.

Do not compare brute-force and cutoff algorithms as if they are identical workloads without stating the physical/algorithmic difference.

---

# Near-term non-goals

Do not prioritize these before Milestone 6 unless a prerequisite forces the issue:

- density-functional theory,
- ab initio electronic structure,
- quantum chemistry,
- multi-node/distributed GPU execution,
- arbitrary user-defined GPU kernels,
- a custom Rust GPU compiler,
- a universal tensor framework,
- WebGPU as the authoritative scientific backend,
- mobile optimization,
- elaborate authentication,
- persistent cloud job scheduling,
- every chemical element,
- every crystal space group,
- publication-grade material-property prediction.

The project should earn complexity gradually.

---

# First implementation target

The first impressive end-to-end demo should be:

1. choose a candidate lattice or random particle initialization,
2. select Lennard-Jones reduced-unit parameters,
3. submit the calculation to the Rust server,
4. compute authoritative `f64` forces/energies on CPU,
5. stream snapshots,
6. animate atoms in 3D,
7. display energy and maximum force over time,
8. converge to a local minimum,
9. inspect the final geometry,
10. rerun the identical configuration using CUDA once Milestone 4 is complete.

This establishes the full product loop before advanced optimization.

---

# Decision rule for future work

When choosing between:

- a more general abstraction,
- a more advanced GPU technique,
- or a working scientifically validated feature,

prefer the scientifically validated feature unless the abstraction is required by at least two concrete use cases.

The project succeeds first by making crystal physics understandable and interactive.

Performance engineering and reusable GPU infrastructure should grow from measured needs in that application.
