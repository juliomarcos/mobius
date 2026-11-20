# Proposal

## Why

Production teams spend a large share of asset time hand-retopologising sculpts, scans and AI-generated meshes, because existing automatic tools either just decimate or produce generic quad meshes that ignore features, silhouette and how the asset will be used. The same gap exists for UV unwrapping, normals control and texturing. Mobius is planned as a suite of guided geometry-processing operations that share one deterministic C++ core. Artist guidance is built into every solver, and AI (local models) helps with understanding and generation without replacing the geometry. This change lays that shared foundation and delivers its first operation, retopology, plus a Blender addon to drive it. UVs, normals control, local-model providers and assisted texturing follow as separate changes on the same core.

## What Changes

- Add a Sharpmake-generated build for a standalone C++ engine (static core library, shared library with a C API, command-line tool, unit and regression tests). The engine has no Blender dependency.
- Add an in-house maths module: fixed-size vectors and matrices, small dense decompositions (least squares, symmetric eigen), complex/tangent-frame helpers, sparse matrices and an iterative sparse solver whose kernels run on the compute backend. It replaces a third-party linear-algebra library.
- Add a core mesh model with validation, repair diagnostics and file I/O (OBJ, PLY).
- Add surface analysis: per-vertex curvature, principal directions, feature-edge detection and a local length scale.
- Add a smooth 4-RoSy cross field that aligns to features, curvature and artist strokes, with explicit singularity detection, reporting and placement control.
- Add field-aligned quad meshing: an orientation-aligned position field, quad-dominant extraction, conversion to an all-quad mesh, boundary and feature snapping, and projection back onto the source surface.
- Add a unified guidance constraint model shared by all operations (direction strokes, density maps, feature/crease locks, region constraints, surface curves), which operations can extend with layer kinds of their own. Solvers consume these inputs directly, whether an artist or an external analysis provider supplies them. Applying them as post-processing is not allowed.
- Add retopology objectives (static game asset, deformable character, organic sculpt, hard surface). Each objective is a named, tunable set of optimisation priorities.
- Add a shared symmetry service: an opt-in mirror plane, a measured symmetry map, symmetric/asymmetric regions, mirror correspondence and mirrored guidance. Retopology produces mirrored topology where the model is symmetric and free topology elsewhere.
- Add an operation-based processing pipeline: named operations, shared analysis stages, pluggable strategies, per-operation objectives, deterministic results, progress reporting and cancellation. Retopology is the first operation.
- Add a compute-backend layer with a CPU reference backend, so GPU acceleration can be added later without changing pipeline stages. No GPU kernels ship in this change.
- Add quality evaluation (valence distribution, quad ratio, element quality, surface deviation, feature and silhouette preservation) so strategies can be compared objectively.
- Add a stable, versioned, operation-generic C ABI (runs by operation name, named parameters and outputs), so the engine can be embedded in other applications and new operations need no new API functions.
- Add a Blender addon (Blender 4.2 LTS and later) that calls the C API through ctypes. It has shared guidance tools (edge-flow strokes, surface curves, density painting, crease/feature marking, regions, symmetry with map preview) and one panel per operation, starting with Retopology.
- AI and computer-vision analysis is **not** implemented in this change. It is supported through the guidance-constraint model, which any external provider can populate.

## Capabilities

### New Capabilities
- `engine-build`: Sharpmake project generation, build configurations and produced artifacts (core library, shared library, CLI, tests).
- `mesh-core`: Input/output mesh representation, topology validation, repair diagnostics and file I/O.
- `surface-analysis`: Curvature, principal directions, feature-edge detection and local scale estimation on the source mesh.
- `direction-field`: Cross-field generation, constraint alignment, smoothness optimisation and singularity management.
- `quad-meshing`: Field-aligned position field, mesh extraction, quad conversion, feature/boundary snapping and surface projection.
- `guidance-constraints`: Unified, operation-extensible model for artist- and provider-supplied guidance (direction, density, features, regions, surface curves) and how solvers must use each kind.
- `surface-symmetry`: Shared symmetry service: opt-in plane, symmetry map, symmetric/asymmetric regions, mirror correspondence, mirrored guidance and coverage reporting.
- `retopology-objectives`: Named retopology objectives and their tunable optimisation priorities.
- `processing-pipeline`: Operations, shared and operation-specific stages, pluggable strategies, per-operation objectives, compute backends, determinism, progress, cancellation and run reports.
- `retopology-evaluation`: Quantitative quality metrics and reports for comparing results and strategies.
- `engine-c-api`: Stable, versioned, operation-generic C ABI for embedding the engine in host applications.
- `blender-addon`: Blender integration: shared guidance authoring, per-operation panels (retopology first), previews and bringing results into the scene.

### Modified Capabilities
- None. This is the first change in the project.

## Impact

- **Code**: New repository layout: `sharpmake/` (build scripts), `engine/` (C++ core, C API, CLI), `tests/` (unit and regression tests plus a reference mesh corpus) and `blender/mobius/` (addon).
- **Toolchain**: MSVC from Visual Studio 2026 (toolset 14.52 or later) is the only supported compiler. Code compiles with `/std:c++latest`, and C++23 is the guaranteed language baseline.
- **Dependencies**: Sharpmake (.NET, build only), simdjson (JSON parsing for presets, guidance and test data; Apache-2.0/MIT) and doctest (tests only). Linear algebra and JSON writing are in-house. Any further third-party geometry library must be license-compatible with commercial distribution. The addon uses only Blender's bundled Python.
- **Platforms**: Windows x64 first. The core avoids platform-specific code so that Linux and macOS targets can be added later.
- **APIs**: Introduces the public C API (`mobius_*`), which becomes a compatibility surface from its first release.
- **Out of scope**: GPU compute kernels (only the backend layer is built), deformation simulation and LOD generation. The planned follow-up operations are also out of scope, each as its own change on this core: guided UV unwrapping, normals control (sharp edges, custom/weighted normals, bake cages), local-model providers, and assisted texturing. Retopology results must stay suitable for those operations.
