# Design

## Context

This is a greenfield repository with no code or specs yet. See proposal.md (Why) for the motivation and the specs under `specs/` for the requirements. The development machine is Windows 11 with Visual Studio 2022 17.14 (MSVC 14.44), the .NET 9 SDK (needed by Sharpmake), Python 3.12, Blender 4.2 through 5.3 and an NVIDIA RTX 5070. The project requires Visual Studio 2026 (MSVC 14.52 or later), which must be installed before implementation starts (see D15). Windows x64 is the only target for this change, but nothing in the core may assume it.

Three constraints shape most of the decisions below:
- **Determinism and testability**: the pipeline spec requires bit-identical output for any thread count.
- **Guidance in the solver**: every guidance kind must enter the field or meshing optimisation as a term or constraint, never as a fix-up afterwards.
- **Host independence**: the core must be embeddable through a C ABI without Blender or Python.
- **One core, many operations**: retopology is the first of several operations (UVs, normals control, assisted texturing). Shared parts (mesh, analysis, symmetry, guidance, maths, backends, pipeline, C API, addon shell) must not assume retopology.

## Goals / Non-Goals

**Goals:**
- A single default strategy chain that is robust on the whole reference corpus. Peak quality on every input is not required yet.
- Clear seams for later strategies (global parametrisation, flow-based singularity repair) and for AI analysis providers, without restructuring.
- Interactive turnaround: field preview in seconds and a full run in under a minute for large sculpts.
- A future operation (for example UV unwrapping) can be added by registering stages, strategies, objectives and layer kinds, with no edits to shared code, the C API or the addon shell.

**Non-Goals:**
- GPU compute kernels (only the backend layer from D9 is built), compilers other than MSVC, Linux and macOS builds, and multi-plane or radial symmetry. Only one mirror plane is supported in this change.
- Running ML models in-process. Providers deliver guidance layers through the existing API.
- Editing the output mesh interactively after a run. Users re-run with changed guidance instead.

## Decisions

### D1. Repository layout and build
```
sharpmake/        main.sharpmake.cs + one .sharpmake.cs per project
engine/core/      static library  (namespace mobius)
  math/ mesh/ analysis/ symmetry/ guidance/ backend/ pipeline/ io/ eval/   shared
  ops/retopo/     retopology operation (field, meshing, objectives, metrics)
engine/capi/      shared library  mobius.dll + include/mobius/mobius.h
engine/cli/       mobius-cli.exe  (links mobius.dll only)
tests/unit/       doctest unit tests against core
tests/regression/ corpus runner against the C API + tests/corpus/
third_party/      vendored, pinned dependencies
blender/mobius/   extension package (blender_manifest.toml); ops/retopo.py per operation
tools/            bootstrap + build.ps1
```
Sharpmake targets `DevEnv.vs2026`, which produces a `.slnx` solution (pin Sharpmake 1.11.0 or later, whose samples include vs2026 targets). `tools/build.ps1 -Config <Debug|Release|Baseline> [-Test]` downloads a pinned Sharpmake release into `tools/sharpmake/` (if missing, after checking its SHA-256), generates projects into `build/` (git-ignored), builds with MSBuild and runs the tests. *Alternative:* adding Sharpmake as a git submodule and building it from source. Rejected: it is slower to bootstrap and gives no benefit until the build itself needs patching.

### D2. Third-party dependencies
- **simdjson 5.2** (Apache-2.0/MIT, used under MIT): JSON parsing for presets, guidance files and test-data bounds. The amalgamated `simdjson.h`/`simdjson.cpp` are vendored and compiled into `MobiusCore`.
- **doctest** (MIT): unit tests only. It never ships in the engine or the DLL.

Linear algebra is in-house (D16), and so is JSON writing (D12). Each geometry algorithm is implemented in-house from its published paper. *Alternatives:* Eigen. Rejected: its sparse direct solvers cannot move onto the compute backend (D9), and the fixed-size types we need are small enough to own. nlohmann/json. Rejected: it is heavy to compile and slow to parse, and simdjson covers parsing. CGAL, rejected because its GPL components conflict with the commercial-licence policy. libigl, rejected because its MPL core is usable but pulls in a large surface area we don't need; individual routines may be ported with attribution. Instant Meshes and QuadriFlow source, rejected as dependencies because they bring global state, Boost/TBB and non-deterministic parallelism. They are used as algorithmic references only.

### D3. Mesh representation
Input polygons are kept as given (n-gons and attributes) for "keep existing topology" regions and for output transfer. For analysis and solving, the engine builds a **working triangle mesh**: each polygon is fan- or ear-triangulated, and a compact half-edge structure is stored as SoA index arrays (`uint32_t`, with sentinels for boundaries). Each working triangle keeps a map back to its source face, so guidance anchored on source faces stays valid. Surface boundary curves and artist feature curves are **embedded** into the working mesh by splitting triangles along them, which makes every hard constraint a sequence of mesh edges. *Alternative:* a polygonal half-edge mesh throughout. Rejected: every numerical operator (cotan Laplacian, connection Laplacian, BVH) is simpler and faster on triangles.

### D4. Surface analysis
- **Curvature:** per-vertex shape operator from least-squares quadric fitting over a geodesic-radius neighbourhood (Rusinkiewicz-style tensor averaging as a fast fallback). The fitting radius gives the noise robustness required by the spec.
- **Direction confidence:** `c = |k1−k2| / (|k1|+|k2|+ε)`.
- **Features:** dihedral-angle threshold → edge chains traced through vertices of feature-degree 2 → curves filtered by minimum length. A corner is a vertex of feature-degree ≠ 2, or one where the turning angle exceeds the corner threshold.
- **Local scale:** `h(v) = clamp(α / sqrt(k1² + k2²), h_min, h_max)`, then smoothed with a few Laplacian iterations so it varies gradually.

### D5. Default direction-field strategy: `complex-smooth`
The 4-RoSy field is represented per vertex as a complex number `z = e^{4iθ}` in a local tangent frame. Its smoothness energy is the connection Laplacian, following the "globally optimal direction fields" formulation. Soft constraints (curvature directions weighted by confidence × objective weight, and rasterised strokes weighted by stroke weight × falloff) add a diagonal penalty `w_i |z_i − z_i*|²`. Hard constraints (features, boundaries, artist-marked edges) remove those vertices from the system with fixed values. The complex Hermitian system is rewritten as a real symmetric positive-definite system of size 2n and solved with the in-house preconditioned conjugate gradient (D16), then normalised. The solve is warm-started from the curvature directions, and its iteration count and final residual go into the run report. A few optional nonlinear smoothing iterations follow, each using a fixed graph-colouring order.

Singularity control is a per-edge weight on the connection Laplacian. Raising it inside a repel region makes rotation there expensive, which pushes singularities out. Lowering it inside an attract region does the reverse. Singularities are found by summing the period jumps around each triangle. A repel request is reported as unsatisfiable when its region is not a disk or would have to contain non-zero total index.

*Alternatives:* Instant Meshes' extrinsic local smoothing. Rejected as the default: it is iterative and order-dependent, and its quality depends on the hierarchy. MIQ (mixed-integer). Rejected: too slow for interactive previews. It may become a later strategy.

### D6. Default meshing strategy: `field-aligned-extract`
1. **Position field:** a multiresolution 4-PoSy position field (Instant-Meshes-style), aligned to the orientation field from D5. Unlike the original, the target edge length is **spatially varying**: `h_eff(v) = h_local(v) · objective_scale / density(v)`. Position-field updates compute their lattice offsets in units of `h_eff` averaged per edge.
2. **Constraint handling:** vertices on embedded feature curves have their position-field target restricted to the curve's 1-D lattice. Feature corners are pinned to integer lattice points. Boundary curves are handled the same way.
3. **Extraction:** collapse the graph of vertex pairs that share a lattice cell, then extract faces from the resulting cycles. This gives a quad-dominant mesh with triangle and pentagon fixes.
4. **All-quad mode:** extract at twice the target edge length, then apply one Catmull-Clark-topology split (positions re-projected). This guarantees all quads.
5. **Post-process:** tangential smoothing constrained to the source surface, then closest-point projection through a BVH. Feature vertices slide only along their curves.

*Alternative:* global seamless parametrisation plus QEx extraction. Rejected as the default: it produces higher-quality layouts, but integer rounding is fragile on noisy scans and it is slower. It is the first planned additional strategy behind the same interface.

### D7. Guidance model and its entry points into the solver
| Guidance | Enters the solver as |
|---|---|
| Direction stroke | soft target `z*` on vertices within falloff radius (D5) |
| Density map / region multiplier | factor in `h_eff` (D6.1) |
| Feature lock / boundary curve | embedded edges (D3) → hard field constraint (D5) + lattice restriction (D6.2) |
| Feature exclude | removed from feature set before D5 |
| Singularity attract/repel | connection-Laplacian edge weights (D5) |
| Deformation zone | local repel + loop-preferring smoothness weight + density floor (from objective) |
| Keep existing topology | region cut out of working mesh; its boundary becomes a pinned boundary curve; source faces copied and stitched |
| Symmetry | mirrored guidance + correspondence from the symmetry service (D8) |

Layers are combined by kind: density layers multiply by weight in log space, and direction layers sum their weighted complex targets. Conflict detection runs during guidance resolution, and the precedence is the one given in the spec.

### D8. Symmetry service and partial symmetry
**Shared service (`core/symmetry`)**, run as a shared stage when a plane is declared:
1. **Symmetry map:** reflect every source vertex across the plane and take the BVH closest-point distance to the source. A vertex is symmetric if the distance is within the tolerance (relative to the diagonal).
2. **Regions:** flood-fill asymmetric vertices into connected regions, drop those below the minimum area, dilate by one ring for stability, then apply artist overrides.
3. **Correspondence:** for symmetric regions, map a surface point p to `closest_point(reflect(p))` restricted to the symmetric region on the other side. Tangents map through the reflection, then re-project into the target tangent plane. Because each side lands on its own surface, small left/right differences are followed.
4. **Guidance mirroring:** in symmetric regions, every layer is copied through the correspondence and merged with what is there (same rules as D7). Disagreements beyond thresholds are reported as conflicts.

**Retopology use:**
- Cut the working mesh along the plane and along the borders of the asymmetric regions.
- The **primary side** is the symmetric half on the plane's positive side. It is meshed with the plane cut as a boundary curve whose field is constrained to cross the plane at right angles or run along it.
- That mesh is mirrored through the correspondence. Plane vertices are snapped exactly onto the plane and welded.
- **Asymmetric regions** are meshed freely as separate patches. Their borders are pinned boundary curves, and they are stitched to the symmetric part with the same conforming-seam method as "keep existing topology" (D7), including its parity fix.
- If no symmetric area remains, the whole mesh is processed without symmetry and the low-coverage warning is emitted.

*Alternatives:* reflecting the mirrored half and then projecting it to the nearest surface, which was the original D8. Rejected: on asymmetric parts the projection lands on the wrong surface, and guidance on the discarded half is lost. A symmetric field over the whole mesh. Rejected: it gives no exact topological symmetry guarantee.

### D9. Pipeline, determinism and threading
- **Operations:** an `operation` declares its name, ordered operation-specific stages, parameter schema (name, type, default, range), accepted layer kinds, objectives and named outputs. Operations register in a static registry, and retopology is `ops/retopo`. The pipeline runs the shared stages (validate/repair → analysis → symmetry → guidance resolution), then the operation's stages, then shared evaluation plus the operation's metrics.
- Each operation-specific stage is an interface (for retopology `field_strategy` and `meshing_strategy`) with strategies registered by name. The pipeline passes an immutable `run_context` (mesh, analysis, symmetry, resolved guidance, objective, parameters), and each stage returns a typed output struct that is exposed under a named output.
- Parallelism uses an in-house `parallel_for` with **fixed chunk boundaries that do not depend on thread count**. Reductions are combined in chunk order, iteration order is never driven by hash maps, and randomness comes from a seeded PCG32 with per-chunk streams. This is what makes the 1-thread and 8-thread results bit-identical.
- Cancellation and progress use an atomic flag and counters, checked at cooperative checkpoints (every chunk and every solver iteration).
- **Compute backends:** data-parallel kernels (per-vertex fitting, field smoothing sweeps, position-field sweeps, evaluation sampling and rasterisation) are called through an `compute_backend` interface. Buffers are typed views (`std::mdspan` over backend-owned storage), and each kernel is a named operation with a CPU implementation. The **CPU reference backend** implements every kernel with the deterministic `parallel_for` above, and it is the only backend in this change. Sparse direct solves stay on the CPU in every backend. A future GPU backend (Vulkan compute is the likely choice, for cross-vendor reach; its host code is still compiled by MSVC) adds implementations per kernel, falls back to CPU for any it lacks, and is gated by the backend-equivalence tolerance test. Determinism is therefore promised per backend and device, not across backends: GPU float ordering differs.
- *Alternative:* TBB. Rejected: its work stealing makes deterministic reductions awkward and adds a runtime DLL.

### D10. Error handling inside the core
The core returns `std::expected<T, error>` (C++23) for every anticipated failure (invalid input, unknown strategy, unsatisfiable constraints). Exceptions are reserved for programmer errors and allocation failure. Every C API entry point wraps its body in a catch-all that converts exceptions to status codes plus a thread-local last-error message.

### D11. C API shape
- `mobius.h` exposes `mobius_get_version`, `mobius_check_api(major)`, and handle types `mobius_mesh*`, `mobius_guidance*`, `mobius_settings*`, `mobius_run*` and `mobius_result*`.
- **Operation-generic surface:** `mobius_operation_list`, and `mobius_operation_describe(name)`, which returns a JSON description of parameters, layer kinds, objectives and outputs. Settings use typed setters by name (`mobius_settings_set_f64/i64/str/bool(settings, "retopology.target_faces", …)`). Results are read as named outputs (`mobius_result_get_mesh(result, "mesh")`, `mobius_result_get_array(result, "field", …)`). A new operation adds names, not functions.
- Inputs are copied from caller-owned buffers: positions (float32 or float64), a face-size array, a flat index array, and named attributes.
- Runs are asynchronous (`mobius_run_start`, `mobius_run_poll`, `mobius_run_cancel`, `mobius_run_wait`, `mobius_run_result`). The engine owns the worker thread.
- **Polling instead of callbacks:** ctypes callbacks invoked from a foreign thread must take the Python GIL inside Blender, which risks deadlocks and stalls. Polling avoids that entirely.
- Reports and presets cross the boundary as UTF-8 JSON strings in engine-owned buffers.

### D12. Objectives
Objectives are data, not code: JSON preset files. Each one holds the named weights from the spec plus derived defaults (feature threshold, curvature-alignment weight, density floor and ceiling, singularity cost, smoothing iterations). A documented mapping table turns weights into solver parameters. Built-in presets are compiled in as embedded JSON, so the engine has no file-system dependency.

All JSON is **parsed with simdjson** (On-Demand API, which gives correctly rounded numbers and therefore identical values on every run). Parse errors are converted to `std::expected` errors carrying the file name and the line and column computed from the byte offset. All JSON is **written by a small in-house writer** that uses a fixed key order, `std::to_chars` shortest round-trip floats and no locale dependence, so reports and presets are byte-identical across runs and can be compared in tests. *Alternative:* simdjson's builder API for writing. Rejected for now: its most convenient form relies on C++26 reflection, which MSVC lacks, and the writer needs exact control over formatting.

### D13. Evaluation
- **Deviation:** uniform area-weighted sampling (fixed seed) plus BVH closest-point queries, in both directions.
- **Silhouette:** CPU-rasterise source and output from 26 fixed view directions (cube faces, edges and corners) at 256² and report the mean of (1 − IoU).
- **Feature preservation:** sample each feature curve and check the distance to the nearest output edge whose direction is within 15°.

The corpus lives in `tests/corpus/`: procedurally generated meshes (sphere, torus, genus-3, bevelled box, cylinder, noisy sphere, 500k displaced sculpt) plus a small number of CC0 scanned and sculpted assets. Each has a `.guidance.json` file and a `.bounds.json` file.

### D14. Blender addon
- Structured as an extension package (`blender_manifest.toml`, minimum 4.2.0) with `mobius.dll` under `lib/`.
- `engine.py` wraps ctypes and checks the API version at load time.
- Mesh data is read with `foreach_get` into NumPy arrays (bundled with Blender) and passed by pointer.
- **Shell plus operation modules:** a shared shell (`engine.py`, guidance tools, symmetry tools, run/poll machinery, report panel) and one module per operation (`ops/retopo.py`) that builds its panel from `mobius_operation_describe` plus hand-written layout hints. Guidance data lives once per object and is filtered by the operation's accepted layer kinds.
- **Symmetry UI:** a plane from axis presets or picked point and normal, a tolerance slider, region overrides (face attribute `mobius_symmetry_override`), and a symmetry-map overlay drawn from the engine's `symmetry_map` output.
- Runs are started from an operator. A modal timer operator (0.1 s) polls progress, updates a progress bar, and on completion builds the result mesh with `from_pydata`/`foreach_set`.
- **Strokes and boundaries** are stored in a `PropertyGroup` on the object as object-space points plus a face anchor. At run time they are **re-anchored** to the evaluated mesh by closest-point projection, so modifiers that change topology do not invalidate them. Drawing uses `bpy.types.SpaceView3D.draw_handler_add` + `gpu`, and raycasts use `BVHTree`.
- **Density** is a float colour attribute `mobius_density` painted in Sculpt/Vertex Paint mode. Grey value g maps to multiplier `4^(2g−1)`, which covers [0.25, 4].
- **Features and regions** are a boolean edge attribute `mobius_feature`, an exclusion attribute `mobius_feature_exclude`, and an int face attribute `mobius_region` indexing a region list in the `PropertyGroup`.
- If an attribute does not survive the modifier stack, the addon warns and falls back to the base mesh.

*Alternative:* storing strokes as Grease Pencil or curve objects. Rejected: the Grease Pencil API changed substantially in Blender 4.3, and separate objects clutter the scene.

### D15. Compiler and language standard
- **MSVC only:** Visual Studio 2026 with toolset 14.52 or later, `/std:c++latest /permissive- /W4 /WX /utf-8 /Zc:__cplusplus /Zc:preprocessor`. A Sharpmake-generated check (`_MSC_FULL_VER` static check in a forced-include header, plus a `build.ps1` pre-check) rejects older toolsets with a clear message.
- **C++23 is the guaranteed baseline.** As of MSVC 14.52, `/std:c++23` is stable and the STL is C++23 feature complete. The core uses `std::expected`, `std::mdspan`, `std::print`/`std::format`, `std::flat_map`, ranges and deducing `this` freely.
- **C++26 is opportunistic.** MSVC 14.52 implements almost no C++26 core-language features (no reflection, contracts, pack indexing or `#embed`) and few C++26 library features. Any post-C++23 use goes through `engine/core/include/mobius/config/features.h`, which wraps feature-test macros (`__cpp_*`, `__cpp_lib_*`) and provides a C++23 fallback. The **Baseline** configuration (Release with `/std:c++23`) builds and runs the full test suite, so an unguarded use fails the build. As MSVC ships C++26 features, they can be adopted behind the same guards without changing the baseline.
- **Public header stays C99** (D11), so the language standard never leaks into hosts.
- **Code organisation and style** follow decision 003 (`docs/coding-style.md`): C++20 named modules in `.ixx` interface units with `.cpp` implementation units, `import std;`, snake_case for types, functions and variables, and strong index types. Macros (feature tests, `MOBIUS_ASSERT`) cannot be exported from modules, so they live in small `.h` headers included from a global module fragment. Third-party headers (simdjson, doctest) are wrapped the same way. Task 1.10 must prove this works with Sharpmake and IntelliSense before group 2. The fallback is ordinary headers.
- *Alternatives:* clang-cl, which would give pack indexing, `#embed` and similar features today, and GCC 16, which has reflection and contracts. Rejected by project decision: one toolchain keeps the build, debugging and ABI story simple.

### D16. Maths module (`engine/core/math`)
- **Fixed-size types:** `vec<T,N>` and `mat<T,R,C>` value types (`constexpr`, `float`/`double`, aliases `vec3d`, `mat3d`, …) plus a `frame` (tangent basis) type, with only the operations the engine needs: arithmetic, dot and cross products, norms, transpose, and products. No expression templates: plain value semantics are fast enough at these sizes, and MSVC optimises them well.
- **Small dense solvers:** closed-form or Jacobi-iteration symmetric eigen decomposition for 2×2 and 3×3 matrices (curvature tensors), and normal-equation least squares with Cholesky for the quadric fits (up to 6×6, D4). The fit inputs are centred and scaled first, to keep the normal equations well conditioned.
- **Complex and rotational helpers:** representing 4-RoSy values as `std::complex<double>`, transporting them between tangent frames, and computing period jumps (D5).
- **Sparse matrices:** CSR storage built from triplets with deterministic duplicate summation (sorted by row, column and insertion index). The values live in backend buffers (D9).
- **Iterative solver:** preconditioned conjugate gradient on top of three backend kernels: SpMV, dot (deterministic fixed-chunk reduction) and axpy. The preconditioner is Jacobi first. A multigrid V-cycle reusing the D6 hierarchy follows if Jacobi misses the performance target (task 11.9 decides). Stopping is on relative residual with an iteration cap. Hitting the cap is reported as a warning naming the stage, and the best iterate is still used.
- **Determinism:** every reduction uses the fixed-chunk rule from D9, so a solve is bit-identical for any thread count on the CPU backend.
- *Alternatives:* a hand-written sparse Cholesky (supernodal, with AMD ordering). Rejected for now: it is several times the code with no route to the GPU. It can be added later as a second solver behind the same interface if PCG struggles on hard inputs.

## Risks / Trade-offs

- [Spatially varying scale in the position field can produce tears or extra singularities where density changes sharply] → Smooth `h_eff` (bounded gradient), test on the density-painting corpus case, and keep a uniform-scale fallback switch.
- [Stitching "keep existing topology" regions in all-quad mode can create parity conflicts, because an odd boundary vertex count cannot be closed with quads] → Make pinned boundary counts even by splitting one boundary edge of the generated side, and report when that is not possible (quad-dominant fallback for that seam).
- [Generalising the pipeline, API and addon before the second operation exists can lead to wrong abstractions] → Keep the operation interface minimal (stages, parameter schema, layer kinds, objectives, outputs). Prove it with a test-only dummy operation that registers and runs end to end without touching shared code.
- [Stitching asymmetric regions into symmetric topology creates extra seams with the same parity problem as preserved regions] → Reuse the D7 stitching and parity fix, and include the partially symmetric sword-arm mesh in the regression corpus.
- [Fixed-chunk determinism costs some parallel efficiency] → Use chunk sizes tuned for 8–16 cores. The 60 s performance target is measured with this constraint in place.
- [C++20 modules: IntelliSense support is still labelled experimental in Visual Studio 2026, and Sharpmake has no built-in MSVC module option] → Prove the setup in task 1.10 first, injecting the MSBuild module settings from Sharpmake. Fall back to headers if the build or editor experience is not workable.
- [MSVC's C++26 support grows slowly, and preview features under `/std:c++latest` can change between toolset updates] → Keep C++23 as the contract (Baseline config), guard every newer feature, and pin the minimum toolset in the build check.
- [PCG converges slowly on large or badly conditioned field systems (strong soft weights, tiny triangles), compared with the direct solve the plan originally used] → Warm start, Jacobi then multigrid preconditioning, scale-normalised weights, and the iteration count and residual in every run report. The 60 s benchmark (task 11.9) includes the solve.
- [In-house maths code can harbour subtle numerical bugs that a mature library would not] → Test each routine against analytic results and against a brute-force dense reference on small random systems.
- [The compute-backend indirection could become an unused abstraction, or a bottleneck for a later GPU port] → Keep it to a small set of named kernels with typed buffers, and exercise it fully through the CPU reference backend in every test.
- [Blender Python API drift between 4.2 and 5.x (gpu shaders, attribute APIs)] → Add a compatibility module that isolates version branches, and run CI smoke tests headlessly (`blender -b --python`) against 4.2 LTS and the newest installed release.
- [Sharp density changes or tiny features below `h_min` cannot be represented] → Report under-resolved features in diagnostics instead of silently dropping them.
- [Hard constraints can make the field infeasible (for example, conflicting feature tangents at a corner)] → Corner vertices are left free of direction constraints, and conflicts are reported with their location.
- [A greedy default meshing strategy will fall short of hand topology on complex characters] → Accepted for this milestone. The strategy interface and the evaluation suite exist so better strategies can land and be measured.

## Migration Plan

Not applicable: this is the first implementation. The C API is version 0.x until the first tagged release. Breaking changes are allowed until 1.0, then they follow the semantic-versioning rule in the engine-c-api spec.

## Open Questions

- Reference hardware profile for the performance baseline (CPU model and core count). It will be recorded with the corpus once a benchmark machine is chosen.
- Which specific CC0 scanned and sculpted assets join the procedural corpus.
- Distribution channel for the addon (Blender Extensions platform, or direct download only).
