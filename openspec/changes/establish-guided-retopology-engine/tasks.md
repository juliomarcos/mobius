# Tasks

Tasks numbered `N.0` are **learning gates**. They belong to the user: only the user checks them, after working through the matching module in `docs/learning/README.md` and recording decisions in `docs/decisions/`. Implementation of a gated group waits for its gate.

## 1. Repository and build scaffolding

- [ ] 1.1 Install Visual Studio 2026 with the "Desktop development with C++" workload (MSVC 14.52 or later). Verify that `vswhere -latest` reports version 18.x and that `cl` reports 19.52 or later.
- [ ] 1.2 Create the repository layout from design D1 (`sharpmake/`, `engine/core|capi|cli/` with the shared and `ops/retopo/` core folders, `tests/unit|regression|corpus/`, `third_party/`, `blender/mobius/`, `tools/`), plus a `.gitignore` excluding `build/`, `tools/sharpmake/` and generated `*.sln`/`*.slnx`/`*.vcxproj`. Verify with `git init`, then `git status`: no generated files are tracked.
- [ ] 1.3 Vendor pinned simdjson 5.2 (amalgamated `simdjson.h`/`simdjson.cpp`, compiled into `MobiusCore`) and doctest under `third_party/`, with a `third_party/VERSIONS.md` that lists version, source URL, SHA-256 and licence. Verify that every listed licence permits closed-source commercial use.
- [ ] 1.4 Write `tools/bootstrap.ps1`, which downloads the pinned Sharpmake release (1.11.0 or later) into `tools/sharpmake/` and checks its SHA-256. Verify that a second run is a no-op and that a tampered archive fails the hash check.
- [ ] 1.5 Write Sharpmake scripts for `MobiusCore` (static lib), `MobiusCApi` (DLL), `MobiusCli` (exe), `MobiusUnitTests` and `MobiusRegressionTests` (exe), targeting `DevEnv.vs2026`, x64, with Debug/Release on `/std:c++latest` and Baseline on `/std:c++23`, plus the D15 compiler flags. Verify that generation produces `build/Mobius.slnx` containing all five projects and the three configurations.
- [ ] 1.6 Write `tools/build.ps1 -Config <Debug|Release|Baseline> [-Test]` (bootstrap → generate → MSBuild → run tests, non-zero exit on failure). Verify with a placeholder failing test that the command exits non-zero and names the test, then remove the placeholder.
- [ ] 1.7 Add the toolset check (a `build.ps1` pre-check and a forced-include `_MSC_FULL_VER` static check) and `mobius/config/features.h` with feature-test-macro wrappers. Verify that pointing the build at toolset 14.44 fails with a message naming 14.52 and 14.44, and that a deliberately unguarded C++26 use breaks the Baseline build.
- [ ] 1.8 Add a symbol-export check step that lists `mobius.dll` exports and fails on any symbol without the `mobius_` prefix. Verify it passes on the stub DLL and fails when a non-prefixed function is exported temporarily.
- [ ] 1.9 Document build prerequisites (Visual Studio 2026 / MSVC 14.52+) and commands in `README.md`. Verify by following the README on a clean clone to a passing `build.ps1 -Config Release -Test` and a passing `build.ps1 -Config Baseline -Test`.
- [ ] 1.10 Prove C++20 named modules with Sharpmake and MSVC (coding-style decision 003): a throwaway proof project with an `.ixx` module interface and a `.cpp` implementation unit, `import std;`, a module that wraps simdjson and doctest through a global module fragment, and the MSBuild module settings injected from Sharpmake (`.ixx` as source files, module dependency scanning, standard-library modules). Build it in Debug, Release and Baseline (`/std:c++23`), and check in Visual Studio that IntelliSense gives completions and go-to-definition across modules. Also run the repository `.clang-format` on the proof's `.ixx` and `.cpp` files, and confirm it leaves `module;`, `export module` and `import` lines intact and groups includes as decision 003 specifies. Verify by showing the user the results; if any part fails, record why in `docs/decisions/` and fall back to headers with `#pragma once` before group 2 starts.

## 2. Core infrastructure, maths types and JSON

- [ ] 2.0 **Learning gate (user sign-off): Coding style.** The style-decisions page is published, the user's choices are recorded in `docs/decisions/`, and the agreed rules are written to `docs/coding-style.md` together with `.editorconfig` and `.clang-format`, adjusted to the outcome of the modules proof (1.10). Verify: the user confirms in chat and marks this box themself. No other task in this group starts before it is checked.
- [ ] 2.1 Implement the per-module error enums (`io_error`, `mesh_error`, … for `std::expected<T, E>`) with their mapping to C API status codes, the `diagnostic` type (warnings, repairs, conflicts with locations) returned in stage outputs and collected in stage order, and a developer-only singleton logger that never feeds results or the report (decision 004). Verify with unit tests for error propagation, status-code mapping, diagnostic ordering, and that logger output never appears in a report.
- [ ] 2.2 Implement a thread pool and a `parallel_for`/`parallel_reduce` with fixed chunk boundaries that do not depend on thread count (design D9). Verify with a unit test showing a float reduction is bit-identical with 1, 2, 8 and 16 threads.
- [ ] 2.3 Implement a seeded PCG32 with per-chunk streams. Verify with a unit test that a parallel sampling run is identical for any thread count.
- [ ] 2.4 Implement a cancellation token and progress reporter (stage name plus a monotonic fraction) checked inside `parallel_for` chunks. Verify that a unit test cancelling a long loop returns `Cancelled` within 100 ms.
- [ ] 2.5 Implement the two assertion levels (decision 004): `MOBIUS_ASSERT` for cheap checks, on in every build, and `MOBIUS_ASSERT_AUDIT` for expensive checks, on in Debug and tests only. A failure breaks into an attached debugger, then throws `mobius::base::contract_violation` with file, line and expression. Verify with unit tests: a failed `MOBIUS_ASSERT` throws in Debug and Release, a failed `MOBIUS_ASSERT_AUDIT` throws in Debug and compiles out in Release, and a C API call that hits a violation returns the internal-error status without terminating the process.
- [ ] 2.6 Implement the `compute_backend` (named kernels, typed `std::mdspan` buffers, device info), a backend name table over a closed set of backends, and the CPU reference backend built on `parallel_for` (design D9). Verify with unit tests: the CPU backend is listed as the default, requesting an unknown backend fails with the available names, and a sample kernel is bit-identical across thread counts.
- [ ] 2.7 Add a backend-equivalence test harness that runs a kernel or a full run on two backends and compares the results within per-metric tolerances. Verify by registering a test-only backend that perturbs results: the harness fails, naming the kernel and metric.
- [ ] 2.8 Implement the fixed-size maths types in `engine/core/math` (`vec<T,N>`, `mat<T,R,C>`, `frame`, aliases), following design D16. Verify with unit tests against hand-computed values, plus `static_assert` checks that core operations are `constexpr`.
- [ ] 2.9 Implement the JSON layer: a simdjson On-Demand parsing wrapper returning `std::expected` errors with file, line and column, plus an in-house writer (fixed key order, `std::to_chars` shortest round-trip floats). Verify with unit tests: a malformed document reports the correct line and column, doubles round-trip exactly, and the same value is written byte-identically on every run.
- [ ] 2.10 Vendor jbcoe/value_types (MIT, header-only) pinned under `third_party/`, with an entry in `third_party/VERSIONS.md` (version, source URL, SHA-256, licence), and expose only its `indirect` as `mobius::base::indirect<T>`, switching to `std::indirect` behind a `__cpp_lib_indirect` guard in `features.h` (design D2, D15). Verify with a unit test that `indirect<T>` copies deeply and keeps value semantics, that a variant holding an `indirect` alternative is smaller than one holding the large alternative inline, and that the Baseline build (`/std:c++23`) uses the vendored version.
- [ ] 2.11 Implement the `mobius.base` module in `engine/core/base` (design D1, decision 004): `base::strong_type<T, Tag, Modifiers...>` with the modifiers `regular`, `ordered`, `hashable`, `formattable`, `arithmetic`, `difference`, `affine_point<D>` and `default_value<V>`, `value_of(x)`, the `index<Tag>` alias with `vertex_index`, `face_index`, `edge_index` and `halfedge_index` (invalid default 0xFFFFFFFF), `contract_violation`, and the generic `base::attribute<Index, T>`. Verify with unit tests for each modifier, `static_assert(sizeof(S) == sizeof(T))` for every strong type, a compile-fail check that a `face_index` is rejected where a `vertex_index` is expected, and attribute lookup by strong index.

## 3. Mesh core

- [ ] 3.0 **Learning gate (user sign-off): Module 0, What good topology is, and Module 1, Meshes as data.** The explainer is published, the hands-on exercise is done, and the module's decisions are recorded in `docs/decisions/`. Verify: the user confirms in chat and marks this box themself. No other task in this group starts before it is checked.
- [ ] 3.1 Implement the polygon mesh container (positions, face sizes, indices, named per-vertex and per-face attributes). Verify with unit tests for mixed-arity input and attribute lookup by name.
- [ ] 3.2 Implement input validation for every kind in the mesh-core spec (indices, degenerate faces, non-finite values, non-manifold edges and vertices, orientation, components), each reported with kind, count and indices. Verify with one unit test per defect kind.
- [ ] 3.3 Implement opt-in repair (vertex merge within tolerance, degenerate removal, orientation unification, non-manifold vertex split), with each action recorded in diagnostics. Verify with unit tests for both repair-enabled and repair-disabled behaviour.
- [ ] 3.4 Implement working-mesh construction: triangulation, half-edge SoA arrays, the source-face map, boundary loops and components (design D3). Verify with unit tests for the Euler characteristic, boundary loop count and source mapping on the sphere, torus and open-disk fixtures.
- [ ] 3.5 Implement constraint-curve embedding: split triangles along a polyline given as surface points so that it becomes a mesh edge chain. Verify with a unit test that an embedded curve's edges match the input polyline within 1e-6 relative and the mesh stays manifold.
- [ ] 3.6 Implement a BVH with closest-point and ray queries over the working mesh. Verify with unit tests comparing against brute force on random queries.
- [ ] 3.7 Implement OBJ and PLY (ASCII and binary) read/write with path and line/offset errors. Verify with round-trip tests and a malformed-file test that checks the reported line.
- [ ] 3.8 Implement bounding-box-relative parameter resolution. Verify with a unit test that relative parameters scale with the mesh (feeds the scale-independence check in 8.9).
- [ ] 3.9 Implement the developer preview path: a dev-only `MobiusPreview` console project (Sharpmake, linked to `MobiusCore`) that runs the stages implemented so far on a mesh and writes a PLY with per-vertex scalar, vector and cross-field attributes. Add `tools/preview/mobius_preview.py`, a Blender script that loads it and shows scalars as colours and directions as line overlays. Verify by previewing the torus fixture in Blender with its Euler characteristic and boundary data shown.

## 4. Surface analysis

- [ ] 4.0 **Learning gate (user sign-off): Module 2, Curvature and features.** The explainer is published, the hands-on exercise is done, and the module's decisions are recorded in `docs/decisions/`. Verify: the user confirms in chat and marks this box themself. No other task in this group starts before it is checked.
- [ ] 4.1 Implement the small dense solvers: 2×2/3×3 symmetric eigen decomposition, and Cholesky least squares up to 6×6 with centring and scaling. Verify against analytic cases, and against a brute-force reference on 1,000 random SPD systems (relative error below 1e-10).
- [ ] 4.2 Implement quadric-fit curvature (k1, k2, principal directions, H, K) over a geodesic-radius neighbourhood (design D4), as a compute-backend kernel. Verify with unit tests: sphere principal curvatures within 5% of 1/r, and cylinder minimum direction within 5° of the axis.
- [ ] 4.3 Implement principal-direction confidence. Verify with a unit test that it is below the umbilic threshold on a plane and a sphere and high on a cylinder.
- [ ] 4.4 Implement dihedral feature detection, curve tracing, corner detection and minimum-length filtering. Verify with unit tests: cube gives 12 curves and 8 corners, a smooth sphere gives 0, and a noisy sphere gives 0 with the configured minimum length.
- [ ] 4.5 Implement the clamped, smoothed local scale field. Verify with a unit test that the curved region is smaller than the flat region and that every value lies within the clamps.
- [ ] 4.6 Add curvature, principal directions, confidence, features and local scale to the preview tool. Verify by previewing the cube, cylinder and one of the user's own models in Blender.

## 5. Evaluation metrics and reference corpus

- [ ] 5.0 **Learning gate (user sign-off): Module 3, Measuring quality.** The explainer is published, the hands-on exercise is done, and the module's decisions are recorded in `docs/decisions/`. Verify: the user confirms in chat and marks this box themself. No other task in this group starts before it is checked.
- [ ] 5.1 Write procedural corpus generators (sphere, torus, genus-3, cube, bevelled box, cylinder, open disk, noisy sphere, approximately symmetric head proxy, exactly symmetric mesh, symmetric character proxy holding a sword in the right hand, 500k displaced sculpt). Each writes OBJ plus a default `.guidance.json`. Verify that the generators are deterministic: repeated generation gives identical file hashes.
- [ ] 5.2 Implement topology metrics (arity counts, quad ratio, valence histogram, irregular vertices, singularity comparison). Verify with a unit test that a regular grid gives quad ratio 1.0 and zero interior irregular vertices.
- [ ] 5.3 Implement element-quality metrics (angle deviation, aspect ratio, edge-length deviation, with min/mean/max/p5 and per-face values). Verify with a unit test that a 30° quad is flagged with deviation ≥ 60°.
- [ ] 5.4 Implement surface deviation (one-sided and symmetric Hausdorff, mean, absolute and relative) using seeded area sampling and the BVH, with sampling as a compute-backend kernel. Verify with a unit test that self-comparison gives zero.
- [ ] 5.5 Implement feature-preservation and 26-view silhouette metrics (design D13), with rasterisation as a compute-backend kernel. Verify with a unit test that a cube missing one feature curve lists that curve.
- [ ] 5.6 Implement guidance-adherence metrics (per-stroke angular deviation, achieved/requested density per layer and region). Verify with a unit test on a synthetic grid with a known stroke.
- [ ] 5.7 Implement symmetry metrics (mirrored-face fraction, maximum mirrored-pair deviation). Verify with a unit test that a mirrored grid gives fraction 1.0 and zero deviation.
- [ ] 5.8 Implement the regression runner, which loads corpus meshes, runs the pipeline and checks `.bounds.json`, failing with the mesh and metric named. Verify that it fails when a bound is tightened artificially.
- [ ] 5.9 Document the metrics and the corpus format in `docs/evaluation.md`. Verify that each metric in the retopology-evaluation spec is documented.

## 6. Guidance model

- [ ] 6.0 **Learning gate (user sign-off): Module 4, Direction fields (4-RoSy) and guidance.** The explainer is published, the hands-on exercise is done, and the module's decisions are recorded in `docs/decisions/`. Verify: the user confirms in chat and marks this box themself. No other task in this group starts before it is checked.
- [ ] 6.1 Implement guidance layers (kind, source label, weight, enabled), a closed set of layer kinds with one container per kind (operations declare which kinds they accept), and surface-point anchors, plus reference validation that rejects out-of-range faces and names the layer. Verify with unit tests for disabled-layer identity, stale-reference rejection, and rejection of a kind the operation does not accept (error lists accepted kinds).
- [ ] 6.2 Implement direction-stroke rasterisation to per-vertex soft targets with radius falloff. Verify with a unit test that target weight falls off with distance and is zero outside the radius.
- [ ] 6.3 Implement density-map and region-multiplier combination (log-space weighted blending into the effective density). Verify with unit tests: an all-ones map is the identity, and artist and provider layers blend by weight.
- [ ] 6.4 Implement feature lock/exclude merging with detected features, and surface curves via embedding (3.5). Verify with unit tests: a forced loop on a sphere becomes a feature, and an excluded cube edge is removed.
- [ ] 6.5 Implement region definitions with overrides (density, singularity attract/repel, smoothness weight, deformation zone, keep existing topology). Verify with unit tests for override resolution on overlapping regions.
- [ ] 6.6 Implement conflict detection and precedence (hard → region → weighted soft), with located diagnostics. Verify with a unit test that a stroke crossing a feature at 45° reports a conflict at the crossing.
- [ ] 6.7 Implement guidance JSON (de)serialisation for the corpus and CLI. Verify with a round-trip unit test covering every layer kind.

## 7. Direction field (`complex-smooth`)

- [ ] 7.0 **Learning gate (user sign-off): Module 5, Solving at scale.** The explainer is published, the hands-on exercise is done, and the module's decisions are recorded in `docs/decisions/`. Verify: the user confirms in chat and marks this box themself. No other task in this group starts before it is checked.
- [ ] 7.1 Implement the 4-RoSy complex helpers (encode/decode, frame transport, period jumps). Verify with unit tests: transport around a closed loop on a plane has zero holonomy, and around a cone vertex it equals the angle defect.
- [ ] 7.2 Implement the CSR sparse matrix (built from triplets with deterministic duplicate summation, values in backend buffers), plus SpMV, dot and axpy as CPU-backend kernels. Verify that SpMV matches a dense reference on random matrices and that repeated builds are bit-identical.
- [ ] 7.3 Implement PCG with a Jacobi preconditioner, relative-residual stopping, an iteration cap and a non-convergence warning. Verify with unit tests: a grid Poisson problem reaches 1e-8 relative residual and matches a dense solve, results are bit-identical for 1 and 8 threads, and an iteration cap of 1 yields the best iterate plus a warning.
- [ ] 7.4 Build tangent frames and the connection Laplacian for 4-RoSy, with per-edge weights (design D5). Verify with a unit test that the energy of a parallel field on a plane is zero.
- [ ] 7.5 Implement the linear solve with hard constraints (features, boundaries) and soft constraints (curvature × confidence × weight, strokes), using the PCG solver from 7.3 on the real 2n form (design D5), with iteration count and residual reported. Verify with unit tests: an unconstrained square has no interior singularities, features align within 1°, the cylinder axis is within 5°, the sphere report shows a residual below tolerance, and an iteration limit of 1 gives a field plus a non-convergence warning.
- [ ] 7.6 Implement stroke-weight influence. Verify with a unit test that mean deviation along a stroke is lower at weight 1.0 than at 0.2.
- [ ] 7.7 Implement singularity detection by period jumps, with index reporting. Verify with unit tests that the index sum is 2 on a sphere and 0 on a torus.
- [ ] 7.8 Implement singularity attract/repel via edge weights, with unsatisfiable-request reporting. Verify with unit tests: a disk repel region ends up singularity-free, and a non-disk region reports unsatisfiable.
- [ ] 7.9 Implement optional nonlinear smoothing as a compute-backend kernel with a fixed colouring order, and register the strategy as `complex-smooth`. Verify that a determinism test gives a bit-identical field for 1 and 8 threads.
- [ ] 7.10 Add the cross field, singularities (coloured by index) and solver residual to the preview tool. Verify by previewing the sphere, torus and a stroke-guided user model in Blender.

## 8. Quad meshing (`field-aligned-extract`)

- [ ] 8.0 **Learning gate (user sign-off): Module 6, From field to quads.** The explainer is published, the hands-on exercise is done, and the module's decisions are recorded in `docs/decisions/`. Verify: the user confirms in chat and marks this box themself. No other task in this group starts before it is checked.
- [ ] 8.1 Build the multiresolution hierarchy over the working mesh, with deterministic graph colouring. Verify with a unit test that the hierarchy is identical across thread counts.
- [ ] 8.2 Implement the position-field optimisation with spatially varying `h_eff` (design D6.1), with sweeps as compute-backend kernels. Verify with a unit test that the lattice spacing on a plane matches the target within 5%.
- [ ] 8.3 Implement feature-curve lattice restriction and corner pinning (D6.2). Verify with a unit test that corner positions land on lattice points.
- [ ] 8.4 Implement graph collapse and face extraction into a quad-dominant mesh. Verify with tests: smooth closed corpus meshes give ≥ 95% quads, and the output is manifold and consistently oriented.
- [ ] 8.5 Implement all-quad mode (2× extraction plus a split step). Verify with a test that every face is a quad on all closed corpus meshes.
- [ ] 8.6 Implement target face count and edge length resolution, with face count within 10%. Verify with a test that a sphere at 2000 faces lands in [1800, 2200] across the corpus.
- [ ] 8.7 Implement tangential smoothing and BVH projection with feature-sliding vertices. Verify with a test that every output vertex lies within 0.1% of the diagonal from the source.
- [ ] 8.8 Verify topology guarantees on the corpus: genus preserved on the torus and genus-3, boundary loops preserved on the open disk, components preserved on two spheres, and the bevelled box keeps feature chains and corners.
- [ ] 8.9 Verify density adherence and scale independence. The painted half-sphere edge-length ratio must be in [0.40, 0.65], and scale 1 vs 100 must give identical connectivity.
- [ ] 8.10 Register the strategy as `field-aligned-extract`, and record initial `.bounds.json` for every corpus mesh. Verify that the regression runner passes.
- [ ] 8.11 Add the output mesh with per-face quality colours to the preview tool, shown next to the source. Verify by previewing the bevelled box and a user model in Blender.

## 9. Symmetry service, preserved regions and deformation zones

- [ ] 9.0 **Learning gate (user sign-off): Module 7, Symmetry, seams and preserved regions.** The explainer is published, the hands-on exercise is done, and the module's decisions are recorded in `docs/decisions/`. Verify: the user confirms in chat and marks this box themself. No other task in this group starts before it is checked.
- [ ] 9.1 Implement "keep existing topology": region cut-out, pinned boundary, source-face copy and conforming stitching with the parity fix (design D7 and Risks). Verify with a test that the region faces are unchanged and that there are no gaps or T-junctions at the seam.
- [ ] 9.2 Implement the symmetry map and regions (reflection distance via BVH, tolerance classes, flood-fill, minimum area, dilation, artist overrides), following design D8. Verify with unit tests: the exactly symmetric mesh is all symmetric, the sword-arm character isolates the arm and sword, and an override flips a region's class.
- [ ] 9.3 Implement the mirror correspondence for points and tangents, restricted to symmetric regions. Verify with a unit test that mapped points on the head proxy lie on the source within 1e-6 of the diagonal, and mapping twice returns near the start point.
- [ ] 9.4 Implement guidance mirroring and merging in symmetric regions, with conflict reports and no mirroring in asymmetric regions. Verify with unit tests: one-sided strokes appear on both sides, opposing strokes beyond 30° report a conflict, and a stroke on the sword is not mirrored.
- [ ] 9.5 Implement the symmetry coverage report and low-symmetry warning, and expose the `symmetry_map` output. Verify with a unit test that a 20%-symmetric model with a 50% threshold reports 20% plus the warning.
- [ ] 9.6 Implement partial-symmetric retopology: primary-side meshing with the plane constraint, mirroring through the correspondence, plane snapping and welding, free meshing of asymmetric regions, and stitching via 9.1. Verify on the corpus: the head proxy has mirror counterparts and exact plane vertices, the sword-arm character is symmetric on the body with a gap-free seam, and the symmetry metrics meet their bounds.
- [ ] 9.7 Implement the deformation-zone behaviour (local repel, loop-preferring smoothness, density floor). Verify with a test that a limb proxy with a marked elbow zone has ≥ 2 encircling loops in the zone and no singularity inside it.
- [ ] 9.8 Verify that the guidance acceptance scenarios hold end to end: a mouth stroke produces a loop within one edge length, a head/neck surface curve produces a closed loop, and a forced feature loop on a sphere is preserved.
- [ ] 9.9 Add the symmetry map, asymmetric regions and stitched seams to the preview tool. Verify by previewing the sword-arm character in Blender.

## 10. Retopology objectives and presets

- [ ] 10.1 Define the shared objective preset format (operation name, weights, parameter defaults) and the four built-in retopology objectives as embedded JSON presets, with the weight → solver-parameter mapping table from design D12 documented in `docs/objectives.md`. Verify with a unit test that every built-in loads and has every named weight.
- [ ] 10.2 Implement objective selection (default static game asset), per-weight overrides and effective-weight reporting. Verify with unit tests that an override changes only the targeted weight, and that an objective from another operation is rejected with the retopology objectives listed.
- [ ] 10.3 Implement custom preset save/load (JSON). Verify with a round-trip unit test.
- [ ] 10.4 Verify that objectives are distinguishable on the corpus: hard surface scores higher than organic on feature preservation for the bevelled part, and static game asset scores lower than organic on silhouette deviation at equal budget.

## 11. Processing pipeline and operations

- [ ] 11.1 Implement the operation list as a closed variant with a name table (name, stages, parameter schema, accepted layer kinds, objectives, named outputs), following design D9 and decision 004. Verify with unit tests for operation listing and unknown-operation errors that list the available operations.
- [ ] 11.2 Implement the shared stages (validate/repair → analysis → symmetry → guidance resolution) and shared evaluation around operation stages, with stop-after-stage. Verify with tests that two operations stopping after guidance resolution return identical analysis and guidance.
- [ ] 11.3 Register retopology as the `retopology` operation (field → meshing → post-process, with its strategies, objectives, layer kinds and outputs). Verify with a test that stop-after-field returns the field and no mesh.
- [ ] 11.4 Add a test-only dummy operation with its own stage, parameter, layer kind and output. Verify that it runs end to end through the pipeline when added by one entry in the operation list (and one in the layer-kind set), with no other change to shared code (the C API check is in 12.3).
- [ ] 11.5 Implement the strategy name table per operation stage over a closed variant of strategies, with listing, defaults, and unknown-strategy errors listing valid names. Verify with unit tests.
- [ ] 11.6 Wire progress (operation, stage, fraction) and cancellation through every stage. Verify with tests that progress is monotonic within each stage and that cancelling during meshing ends within 1 s with status `cancelled`.
- [ ] 11.7 Implement the JSON run report (operation, version, effective parameters, strategies, compute backend and device, timings, diagnostics, evaluation) for completed, failed and cancelled runs. Verify with tests for each outcome, including a forced failure in the field stage.
- [ ] 11.8 Verify end-to-end determinism with a corpus test on the CPU reference backend: 1 vs 8 threads gives bit-identical output meshes.
- [ ] 11.9 Benchmark the 500k sculpt to 10k faces in Release and record the hardware profile in `tests/corpus/BENCHMARK.md`. Verify that the run is under 60 s. If not, profile it, and add the multigrid V-cycle preconditioner (design D16) if the field solve dominates.

## 12. C API

- [ ] 12.1 Write `mobius.h` (C99, opaque handles, status codes, version and API check, thread-local last error), following design D11. Verify with a C99 compile-only test that builds cleanly at warning level 4.
- [ ] 12.2 Implement mesh, guidance, settings and preset handles that copy caller-owned buffers. Verify with a test that frees host buffers right after submission and the run still succeeds.
- [ ] 12.3 Implement operation-generic runs: `mobius_operation_list`/`describe`, typed named settings, asynchronous runs (start by operation name, poll, cancel, wait), and named-output accessors (mesh, arrays, report JSON). Verify with tests: describing `retopology` lists its parameters, layer kinds, objectives and outputs; the start call returns immediately and polling reaches completion; an unknown output name returns invalid-argument with the available outputs listed; the test-only dummy operation from 11.4 runs through the API with no API changes.
- [ ] 12.4 Implement catch-all exception translation, invalid-handle detection after release, and null-argument errors that name the parameter. Verify with unit tests for each.
- [ ] 12.5 Implement strategy listing, compute-backend listing and selection, symmetry-analysis requests, and objective/preset functions, so the API offers full feature parity. Verify with a test that enumerates every pipeline option through the C API.
- [ ] 12.6 Port the regression runner to call only the C API. Verify that the regression suite passes through `mobius.dll`.
- [ ] 12.7 Document the API in `docs/c-api.md` with a minimal C example. Verify that the example compiles and runs against the built DLL.

## 13. Command-line tool

- [ ] 13.1 Implement `mobius-cli` (`--operation` defaulting to retopology, `--list-operations`, `--describe <op>`, `--set name=value`, input/output paths, symmetry plane, guidance JSON, objective and overrides, target, mode, strategy, stop-after, report path, thread count), linking only `mobius.dll`. Verify with a test run on the corpus sphere that writes OBJ plus a report.
- [ ] 13.2 Document CLI usage in `docs/cli.md`. Verify that every documented example command runs successfully.

## 14. Blender addon

- [ ] 14.1 Create the extension package `blender/mobius` (`blender_manifest.toml`, minimum version 4.2.0) with the shared shell and an `ops/` folder, a `compat.py` for version branches, and `engine.py` (ctypes loader plus API-version check). Verify that it installs and enables in Blender 4.2 and 5.3, and that a mismatched DLL shows both versions with the operators disabled.
- [ ] 14.2 Implement mesh extraction from the evaluated object (`foreach_get` → NumPy → C API) and result-object creation with the source transform. Verify with a headless test (`blender -b --python`) that the source is unchanged and the result object exists.
- [ ] 14.3 Implement the modal run operator (timer polling, progress bar, cancel). Verify with a manual check that the UI stays responsive and that cancelling creates no result, plus a headless test of the cancel path.
- [ ] 14.4 Implement the shared stroke and surface-curve tool (BVH raycast drawing, PropertyGroup storage, weight editing, delete, re-anchoring to the evaluated mesh, GPU overlay). Verify that strokes persist across save and reopen, and that a test confirms re-anchoring through a subdivision modifier.
- [ ] 14.5 Implement density painting setup (`mobius_density` colour attribute, grey → multiplier mapping). Verify with a headless test that painted values reach the engine as the expected multipliers.
- [ ] 14.6 Implement feature mark/exclude operators (edge attributes), region assignment with override UI (face attribute plus region list), and the attribute-loss warning. Verify with a headless test that the region constraint covers exactly the assigned faces with the set multiplier.
- [ ] 14.7 Implement operation panels: a module per operation that builds its panel from `mobius_operation_describe` plus layout hints, and the Retopology panel (objective, overrides, target, output mode, strategy, preset save/load). Panels show only accepted guidance kinds. Verify with headless tests that the chosen objective is named in the run report and that only the shared section and the Retopology panel are registered.
- [ ] 14.8 Implement the shared symmetry tools (enable, axis or picked plane, tolerance, region overrides via `mobius_symmetry_override`) and the symmetry-map overlay with coverage display. Verify on the sword-arm corpus character that the arm and sword show as asymmetric and the coverage percentage matches the engine report.
- [ ] 14.9 Implement the field/singularity preview overlay and the per-face quality overlay on results. Verify manually on the corpus sphere and torus, and confirm that preview creates no result object.
- [ ] 14.10 Implement the run report panel (key metrics, warnings, conflicts, full text view). Verify with a manual check that a run with a stroke crossing a feature shows the conflict warning.
- [ ] 14.11 Add a packaging step to `build.ps1` that bundles `mobius.dll` into an installable zip, and write `docs/blender-addon.md`. Verify by installing the zip in a fresh Blender profile and following the guide.

## 15. Integration checks

- [ ] 15.1 Run `tools/build.ps1 -Config Release -Test` and `tools/build.ps1 -Config Baseline -Test` from a clean clone. Verify that the unit, regression and C-API tests all pass in both configurations and the export check passes.
- [ ] 15.2 Run the headless Blender test suite against Blender 4.2 LTS and the newest installed release (5.3). Verify that both pass.
- [ ] 15.3 Do an artist walkthrough in Blender on the head proxy and bevelled box: strokes, density paint, region, symmetry, objective switch, preview and run. Verify that the results match the guidance acceptance scenarios and record screenshots in `docs/`.
- [ ] 15.4 Run `openspec validate establish-guided-retopology-engine --strict`. Verify that it reports the change as valid.

## Workflow follow-up

- Archive the change after implementation is reviewed and the integration checks pass.
- After archiving, confirm the main specs under `openspec/specs/` contain all twelve capabilities with their Purpose sections.
