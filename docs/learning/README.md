# Learning path

Mobius is built so that its owner understands every algorithm before it is implemented. Each module below unlocks one group of tasks in `openspec/changes/establish-guided-retopology-engine/tasks.md` through a **learning gate** (task `N.0`) that only the owner checks.

For the full outline of what to learn, including the maths prerequisites, see [syllabus.md](syllabus.md).

## How a module works

1. **Explainer:** an interactive page that cites its sources. Play with it, and don't just read it.
2. **Hands-on:** try the idea on real models with existing tools or the Mobius preview tool.
3. **Decisions:** the choices the plan currently makes for you, each with a recommendation and its trade-off. Confirm or change each one. Every answer is recorded in `docs/decisions/`.
4. **Sign-off:** check the gate in `tasks.md` (or say so in chat). Implementation of that group starts only after that.

After each algorithmic group is built, there is a walkthrough of what it produces on your own models.

## Modules

| # | Module | Gate | Status |
|---|---|---|---|
| 0 | What good topology is | 3.0 | in progress: [explainer](https://claude.ai/artifact/X2YhDyrNGJWAkYyUx6x9HS) (source: `module-00-good-topology.html`) |
| – | Coding style (decisions only) | 2.0 | awaiting your answers: [decision page](https://claude.ai/artifact/UWGKBiPGRLmcSvPLHiaSzJ) (source: `coding-style-decisions.html`) |
| – | Coding style, part 2: design rules (decisions only) | 2.0 | answered, recorded in [decision 004](../decisions/004-coding-design-rules.md): [decision page](https://claude.ai/artifact/GGweDr3FRyRRL3zDX37J21) (source: `coding-style-decisions-2.html`) |
| – | Artist workflows: hand-made input, re-runs, fused parts, Blender marks (decisions only) | – | awaiting your answers: [decision page](https://claude.ai/artifact/9BAkFwqxeW13UALt6QMAEe) (source: `artist-workflow-decisions.html`) |
| 1 | Meshes as data | 3.0 | not started |
| 2 | Curvature and features | 4.0 | not started |
| 3 | Measuring quality | 5.0 | not started |
| 4 | Direction fields (4-RoSy) and guidance | 6.0 | not started |
| 5 | Solving at scale | 7.0 | not started |
| 6 | From field to quads | 8.0 | not started |
| 7 | Symmetry, seams and preserved regions | 9.0 | not started |

### 0. What good topology is
- **Learn:** edge loops and edge flow; poles (vertices where 3 or 5 edges meet instead of 4) and where artists put them; why topology matters for deformation, UVs, baking and LODs; quad-dominant vs all-quad meshes.
- **Hands-on:** run Blender's built-in QuadriFlow remesher (Properties → Object Data → Remesh → Quad → QuadriFlow Remesh, or F3 search "QuadriFlow Remesh") and Instant Meshes on one of your sculpts. Compare both with a hand-made game character's topology.
- **Decisions:** default output mode (all-quad or quad-dominant); which asset types matter most to you first.

### 1. Meshes as data
- **Learn:** vertices, faces and connectivity; manifold vs non-manifold; boundaries and holes; genus and the Euler characteristic (V − E + F); why the engine triangulates internally.
- **Hands-on:** inspect broken meshes in Blender (non-manifold selection) and see what validation and repair report.
- **Decisions:** should repair be on by default; tolerance defaults; which file formats matter beyond OBJ and PLY.

### 2. Curvature and features
- **Learn:** principal curvatures and directions; mean and Gaussian curvature; umbilic (direction-less) regions; sharp features from dihedral angles; noise vs real features.
- **Hands-on:** preview curvature and feature curves on a cylinder, a cube and your own scan or sculpt.
- **Decisions:** default feature angle; minimum feature length on noisy scans; how strongly curvature should steer edge flow.

### 3. Measuring quality
- **Learn:** valence and irregular vertices; element shape quality; Hausdorff distance (how far the result strays from the original); silhouette and feature preservation; how to read a run report.
- **Hands-on:** compare the metrics of the QuadriFlow and Instant Meshes results from Module 0 against what your eye says.
- **Decisions:** which metrics define "good" for you; the regression bounds; whether the 60-second large-sculpt target is right.

### 4. Direction fields (4-RoSy) and guidance
- **Learn:** vector fields vs line fields vs cross fields; why quads need 4-fold rotational symmetry (4-RoSy); singularities are poles; the Poincaré–Hopf theorem (why every closed surface needs poles, for example their indices sum to 2 on a sphere); smoothness vs alignment; strokes and features as constraints.
- **Hands-on:** in the interactive explainer, draw strokes and watch the field and its singularities move. Then draw orientation strokes in Instant Meshes.
- **Decisions:** stroke falloff and default weight; how singularity attract/repel should feel to an artist; conflict precedence.

### 5. Solving at scale
- **Learn:** why "find the smoothest field" becomes a sparse linear system; what conjugate gradients do, intuitively; preconditioning; residuals and convergence; why this runs on a GPU later.
- **Hands-on:** a small interactive solver that shows the residual falling as iterations run on a mesh.
- **Decisions:** accept an iterative solver over a direct one; tolerance and iteration cap; the action when the solver hits the cap (the plan currently warns and continues).

### 6. From field to quads
- **Learn:** the position field and its lattice; how Instant Meshes extracts faces; why some triangles or pentagons appear; all-quad via subdivision; the alternatives (global parametrisation, QuadCover/MIQ, QuadriFlow) and why the plan picks the Instant-Meshes-style method first.
- **Hands-on:** step through extraction on a simple mesh in the explainer, then compare against Blender's QuadriFlow.
- **Decisions:** the meshing strategy for version 1; face-count tolerance; how hard features should snap.

### 7. Symmetry, seams and preserved regions
- **Learn:** measuring symmetry; partial symmetry; mirror correspondence; stitching patches without T-junctions; the parity problem (quads need even vertex counts along seams).
- **Hands-on:** preview the symmetry map on the sword-arm test character and on one of your own models.
- **Decisions:** default symmetry tolerance; the low-symmetry threshold; what happens when a seam can't be closed with quads.

## Reading list

Cited throughout the explainers so you can check claims rather than trust them. Numbering is continuous across groups so the syllabus can refer to entries.

Local copies in `science/` are not tracked by git; `science/papers.txt` has the download link for each. `science/source_code.txt` links libigl's code page and quadmesh.cloud. Individual libigl routines may be ported with attribution, as decided in D2.

### General and foundations

1. Keenan Crane, *Discrete Differential Geometry: An Applied Introduction*, CMU course notes and videos. The best starting point for Modules 1, 2 and 4.
2. Botsch, Kobbelt, Pauly, Alliez, Lévy, *Polygon Mesh Processing*, 2010. The general reference.
3. Bommes et al., "Quad-Mesh Generation and Processing: A Survey", Computer Graphics Forum, 2013. A map of the whole field.
4. David Goldberg, "What Every Computer Scientist Should Know About Floating-Point Arithmetic", ACM Computing Surveys, 1991. Why summation order matters for determinism.
5. Christer Ericson, *Real-Time Collision Detection*, 2004. Bounding-volume hierarchies and closest-point queries.

### Module 2 – Curvature and features

6. Rusinkiewicz, "Estimating Curvatures and Their Derivatives on Triangle Meshes", 3DPVT 2004. The fast fallback estimator in D4.
7. Meyer, Desbrun, Schröder, Barr, "Discrete Differential-Geometry Operators for Triangulated 2-Manifolds", 2003. The cotan Laplacian and angle-defect curvature.

### Module 3 – Measuring quality

8. Cignoni, Rocchini, Scopigno, "Metro: Measuring Error on Simplified Surfaces", Computer Graphics Forum, 1998. Surface deviation as in D13.

### Module 4 – Direction fields

9. Vaxman et al., "Directional Field Synthesis, Design, and Processing", Eurographics State of the Art Report, 2016. The overview of direction fields for Module 4.
10. Ray, Vallet, Li, Lévy, "N-Symmetry Direction Field Design", ACM TOG 2008. Singularity index and N-RoSy design.
11. Knöppel, Crane, Pinkall, Schröder, "Globally Optimal Direction Fields", SIGGRAPH 2013. Design decision D5.
12. Crane, Desbrun, Schröder, "Trivial Connections on Discrete Surfaces", SGP 2010 (Computer Graphics Forum 29(5)). Fields with singularities placed where you ask; background for attract/repel.
13. Marcias, Pietroni, Panozzo, Puppo, Sorkine-Hornung, "Animation-Aware Quadrangulation", SGP 2013 (local copy: `science/animation-aware-quadrangulation-2013.pdf`). A cross field from how the surface stretches across poses; a candidate guidance input for the deformable-character objective.

### Module 5 – Solving at scale

14. Shewchuk, "An Introduction to the Conjugate Gradient Method Without the Agonizing Pain", CMU tech report, 1994. The classic intuitive explanation of CG and preconditioning.
15. Yousef Saad, *Iterative Methods for Sparse Linear Systems*, 2nd ed., SIAM 2003. A reference; dip in only as needed.
16. Wiersma, Nasikun, Eisemann, Hildebrandt, "A Fast Geometric Multigrid Method for Curved Surfaces" (Gravo MG), SIGGRAPH 2023. The leading candidate for the multigrid preconditioner in D16 and task 11.9.
17. Padilla, Wiersma, Huisman, Campolattaro, Sorkine-Hornung, Hildebrandt, "GravoTet: A Fast Multigrid Hierarchy Construction for Tetrahedral Meshes", Computers & Graphics 2026 (local copy: `science/gravotet-a-fast-multigrid-hierarchy-construction-for-tetrahedral-meshes-smi-2026-padilla-et-al.pdf`). The volumetric follow-up; read it for its boundary-preservation ideas, since Mobius itself works on surfaces.

### Module 6 – From field to quads

18. Jakob, Tarini, Panozzo, Sorkine-Hornung, "Instant Field-Aligned Meshes", SIGGRAPH Asia 2015. Design decision D6.
19. Huang et al., "QuadriFlow: A Scalable and Robust Method for Quadrangulation", SGP 2018. The method behind Blender's QuadriFlow remesher.
20. Kälberer, Nieser, Polthier, "QuadCover – Surface Parameterization using Branched Coverings", Eurographics 2007. An early global-parametrisation approach to quad meshing.
21. Bommes, Zimmer, Kobbelt, "Mixed-Integer Quadrangulation", SIGGRAPH 2009. The main alternative to D6.
22. Bommes, Campen, Ebke, Alliez, Kobbelt, "Integer-Grid Maps for Reliable Quad Meshing", SIGGRAPH 2013. Integer-grid maps for more reliable global parametrisation.
23. Ebke, Bommes, Campen, Kobbelt, "QEx: Robust Quad Mesh Extraction", SIGGRAPH Asia 2013. How to read a quad mesh out of an integer-grid map.
24. Campen, Bommes, Kobbelt, "Quantized Global Parametrization", SIGGRAPH Asia 2015. Handling the integer rounding step of global parametrisation.
25. Pietroni, Nuvoli, Alderighi, Cignoni, Tarini, "Reliable Feature-Line Driven Quad-Remeshing" (QuadWild), SIGGRAPH 2021. Strong on hard-surface and CAD-like inputs; its code needs Boost and Gurobi, so it is a reference and not a dependency.
26. Lyon, Campen, Kobbelt, "Simpler Quad Layouts using Relaxed Singularities", SGP 2021. A layout-based method that relaxes where singularities must sit.
27. Takayama, Panozzo, Sorkine-Hornung, Sorkine-Hornung, "Sketch-Based Generation and Editing of Quad Meshes" (Sketch-Retopo), SIGGRAPH 2013. Artist-drawn patches with edge counts; a model for direct control.
28. Takayama, Panozzo, Sorkine-Hornung, "Pattern-Based Quadrangulation for N-Sided Patches", SGP 2014. Filling patches of any side count with quads.

### Module 7 – Symmetry, seams and preserved regions

29. Takayama, Panozzo, Sorkine-Hornung, Sorkine-Hornung, "Robust and Controllable Quadrangulation of Triangular and Rectangular Regions", technical report (local copy: `science/robust-and-controllable-quadrangulation-techreport.pdf`). Proves quad boundaries need an even edge count and fills 3- and 4-sided patches with any valid counts; directly relevant to the seam and parity fix in D7 and D8.
30. Mitra, Guibas, Pauly, "Partial and Approximate Symmetry Detection for 3D Geometry", SIGGRAPH 2006. Background for partial symmetry in D8.
31. Podolak, Shilane, Golovinskiy, Rusinkiewicz, Funkhouser, "A Planar-Reflective Symmetry Transform for 3D Shapes", SIGGRAPH 2006. Finding and scoring mirror planes.
32. Kobsik, Lim, Kobbelt, "Partial Symmetry Detection for 3D Geometry using Contrastive Learning with Geodesic Point Cloud Patches" (SymCL), VMV 2026 (local copy: `science/SymCL_-_Partial_Symmetry_Detection_for_3D_Geometry.pdf`). Learned detection of mirrored and repeated parts; a future out-of-process provider, not v1.

### Later operations and AI providers (not needed for v1)

33. Sorkine, Alexa, "As-Rigid-As-Possible Surface Modeling", SGP 2007. The basis of ARAP UV unwrapping later.
34. Oehri, Herholz, Sorkine-Hornung, "Higher-Order Continuity for Smooth As-Rigid-As-Possible Shape Modeling", JCGT 2025 (local copy: `science/smooth_arap_2025.pdf`). A smoother variant of ARAP, for later UV work.
35. Dong et al., "NeurCross: A Neural Approach to Computing Cross Fields for Quad Mesh Generation", SIGGRAPH 2025. Learned cross fields; a possible provider.
36. "QuadGPT: Native Quadrilateral Mesh Generation with Autoregressive Models", ICLR 2026 (arXiv 2509.21420). Generative quad meshes; watch list only.
37. Binninger, Wiersma, Herholz, Sorkine-Hornung, "TetWeave: Isosurface Extraction using On-The-Fly Delaunay Tetrahedral Grids for Gradient-Based Mesh Optimization", SIGGRAPH 2025 (local copy: `science/TetWeave_SIGGRAPH_2025_compressed_10MB.pdf`). Kept for reference; low value for Mobius (triangle output, PyTorch and GPU optimisation, weak on thin parts).
