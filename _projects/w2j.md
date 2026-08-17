---
title: "W2J: Porting a 40-Year-Old Fortran Reservoir Model to Julia"
collection: projects
type: "Independent Software Engineering Project"
permalink: /projects/w2j
date: 2026-08-01
excerpt: "A Julia reimplementation of CE-QUAL-W2, USACE's Fortran hydrodynamic and water-quality reservoir model — rebuilt for multi-core parallelism and gradient-based calibration, validated line-by-line against real reservoir data and the original ~45,000-line source."
---

What does it take to modernize a scientific model that predates the engineers who maintain it? CE-QUAL-W2 is a ~45,000-line Fortran codebase USACE has used since the 1980s to simulate how reservoirs stratify, mix, and carry heat and pollutants — the kind of model a water district calls on before deciding whether it's safe to draw drinking water downstream of a dam. This project rebuilds it from scratch in Julia, module by module, each one validated against the original source and real reservoir data before it's trusted.

The Problem
======

Fortran scientific codebases like this one are everywhere in government and research infrastructure, and they share the same three limitations:

- **No real parallelism.** The physics is embarrassingly parallel — thousands of independent per-segment tridiagonal solves — but the original code runs single-threaded, `OMP` pragmas commented out.
- **No differentiability.** Calibrating a model like this against observed data means tuning dozens of rate coefficients by hand. A differentiable version could use gradient-based optimization instead.
- **Decades of accumulated coupling.** Water-quality kinetics for ~70 constituent processes are entangled with grid indexing throughout — you can't reuse the ecology math without dragging the `(K,I)` loop structure with it.

Rewriting isn't a matter of running a Fortran-to-Julia transpiler and hoping for the best. It's re-deriving *why* each subroutine does what it does, then deciding what to port faithfully, what to defer, and what to redesign — and being honest, in the code itself, about which is which.

Engineering Approach
======

### Read the source, don't guess

Every module here traces back to a specific Fortran file and line range, confirmed by reading the source directly — not reconstructed from documentation or memory. This discipline caught real mistakes early: an initial assumption that the transport module updates concentrations directly turned out to be wrong. Reading `transport.f90` showed it only computes explicit advective flux terms; the actual implicit solve lives in a different file entirely, three call sites away, and reuses coefficients computed by *yet another* module earlier in the same timestep — a real sequencing dependency that would have silently produced wrong physics if assumed instead of traced.

### Reduced physics, clearly flagged — not silently wrong

Faithfully porting every term in the original momentum and transport equations before anything runs end-to-end would mean months with nothing to validate. Instead, each physics module ports the real formulas for its core terms and explicitly zeroes out — and documents in the code, not just a wiki — the terms that need a not-yet-built dependency (turbulence closure, meteorology forcing). Nothing is silently approximated; every simplification is a flagged, temporary scope decision.

### Validate with checks that can actually fail

A test that can't fail isn't a test. Correctness checks throughout this project are built to be falsifiable by a real bug, not just a smoke test:

```
Zero-flow sanity check:  water surface must stay bit-stable to < 1e-9
                         over a full simulated year — a bug in the
                         tridiagonal assembly shows up as drift or NaN,
                         not a false pass.

Heat conservation check: sum(T x volume) before/after a diffusion step
                         must match to machine precision — this is what
                         caught two real bugs (see below), not a
                         hypothetical edge case.
```

Architecture
======

<img src='/images/w2j-architecture.png'>

Read bottom-to-top, like a water column: input files at the bed, the physics solve
through the middle depths, the timestepping driver at the surface. **Green** = validated
against real data or known reference values. **Amber** = real core formulas, with
specific terms deliberately stubbed and flagged rather than silently approximated.
**Grey** = not started. `Tridiagonal.jl` is drawn as a shared primitive rather than
another layer in the stack, since nearly every physics module reaches into it directly.

Bug Found: A Real Regression, Not a Hypothetical
======

Parallelizing the free-surface solve with `Threads.@threads` was correct on the first pass — bit-identical output confirmed at 1 and 4 threads. But wall-clock time hadn't been checked, and when it was, naive threading made the hot loop **3.6x slower**, not faster:

| Threads | Naive threading | After fix |
|---|---|---|
| 1 | 0.136 ms/step | 0.14 ms/step |
| 8 | 0.484 ms/step | 0.20 ms/step |

The grid is small enough that `Threads.@threads`'s own spawn overhead outweighed the actual per-column work. The fix — a `parallel_foreach` that only threads above an empirically benchmarked size threshold, picked serial vs. threaded per call — recovered the 1-thread baseline at every thread count, while still threading automatically once a grid is large enough to benefit. Nobody profiles a fix like this without first measuring that the "obvious" optimization made things worse.

Bugs Found: Heat Was Leaking Through the Boundaries
======

Adding temperature transport meant validating that heat is actually conserved under pure diffusion — no sources, no flow, so total heat should be exactly constant. It wasn't. Two rounds of debugging traced it to the same root cause at both ends of the water column: the raw bathymetry array is genuinely nonzero *above* the current water surface and *at* the lakebed (it reflects the reservoir's full physical shape, not where the model currently truncates it). A uniformly-filled diffusivity field was coupling active layers to phantom cells above the surface and below the bed.

| Check | Before fix | After fix |
|---|---|---|
| Single-step heat conservation | ~24% loss over 3,000 steps | ~1e-16 relative (machine precision) |
| Sharp temperature step profile | didn't reach expected equilibrium | smooths exactly as diffusion predicts |

Results
======

| Metric | Value |
|---|---|
| Test suite | 498/498 passing, at 1 and 4 threads |
| Full simulated year (26,208 timesteps) | ~2.7 seconds, drift < 1.2e-10 |
| Heat conservation (post-fix) | ~1e-16 single-step, ~1e-13 over 3,000 steps |
| Threading regression recovered | 3.6x slowdown → parity or better at every thread count |
| Fortran source read directly | ~45,000 lines across the modules ported so far |

Tech Stack
======

* **Language:** Julia 1.11
* **Parallelism:** `Threads.jl` with an empirically-tuned, size-adaptive dispatch (`parallel_foreach`)
* **Numerics:** Custom Thomas-algorithm tridiagonal solver, shared across every module that needs it
* **Planned:** `Enzyme.jl` for gradient-based calibration (drove the "no global mutable state" design constraint from day one)
* **Tooling:** Standalone Excel→YAML control-file converter, a ported subset of the reference model's own input-validation preprocessor
* **Reference:** Original CE-QUAL-W2 Fortran source (USACE), validated line-by-line

Key Takeaways
======

**Reading the source beats guessing every time.** Two of the most important findings in this project — a real cross-module sequencing dependency and two boundary-condition bugs — were only catchable by tracing the actual Fortran, not by working from documentation or a plausible mental model.

**"It's correct" and "it's fast" are different claims.** The threading regression shipped as *correct* — bit-identical output, passing tests — while quietly being slower than no threading at all. Only measuring wall-clock time surfaced it.

**A test that can't fail isn't validation.** The zero-drift and heat-conservation checks weren't chosen for convenience; they're the specific checks a real bug in the physics would visibly break.

---

*Ongoing project — currently validated against real Detroit Reservoir (Oregon) data, with water-quality kinetics, meteorology forcing, and a full Fortran reference-output comparison still ahead.*
