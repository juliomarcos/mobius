# Syllabus

This is the complete outline of what you need to grasp to solve guided retopology with confidence, from the prerequisites up to the engine's own algorithms. It complements [README.md](README.md), which stays the gated module list: the syllabus adds no new gates. Use it as a map. Dip into a topic when a module needs it, and don't try to finish a part before starting the next.

The topics are grouped into parts. **A** is the mathematical foundations (no gate). **B** is geometry-processing basics (modules 0–3). **C** is the retopology core (modules 4–7). **D** is engineering the engine (no gate, learned alongside the build). **E** lists later topics that v1 does not need.

Every topic says *why Mobius needs it*, *what to understand*, a *"You're ready when you can…"* self-check, and *Read*, which points to the numbered reading list in README.md by author or short title. Depth is stated honestly with three words: **intuition** (you can explain it in your own words and recognise it in a picture), **can derive** (you can work out the key formula on paper), and **can implement** (you could write and test it yourself). Nothing here asks for more than the plan needs.

## Depth at a glance

| Topic | Module | Target depth |
|---|---|---|
| A1 Linear algebra | prerequisite | can derive |
| A2 Calculus and energy minimisation | prerequisite | can derive |
| A3 Complex numbers as rotations | prerequisite | can derive |
| A4 Differential geometry of surfaces | prerequisite | intuition |
| A5 Discrete differential geometry | prerequisite | can derive (Laplacian, angle defect), intuition (the rest) |
| A6 Numerical computing | prerequisite | intuition |
| B1 What good topology is | 0 | intuition |
| B2 Meshes as data | 1 | can implement |
| B3 Spatial queries | 1 / engineering | intuition, can implement a simple BVH |
| B4 Curvature and features | 2 | can derive, can implement |
| B5 Measuring quality | 3 | can implement |
| C1 Direction fields | 4 | can derive the energy, intuition for the rest |
| C2 Solving at scale | 5 | can implement CG, intuition for multigrid |
| C3 From field to quads | 6 | intuition for alternatives, can follow Instant Meshes step by step |
| C4 Symmetry, seams and preserved regions | 7 | intuition, can reason about parity |
| D1–D5 Engineering | engineering | working knowledge |
| E Later topics | later | awareness only |

---

## Part A – Mathematical foundations (prerequisite, no gate)

These are tools, not a course. Learn them just in time, and stop when the self-check passes.

### A1. Linear algebra

**Why Mobius needs it.** Curvature is a 2×2 matrix (the shape operator) whose eigenvectors are the curvature directions (D4). Each vertex has a tangent frame, which is a change of basis (D16). The quadric fit is a least-squares problem.

**What to understand.**
- Vectors, dot product (length, angle, projection) and cross product (normals, areas).
- A matrix as a map that moves vectors, and a product as doing one map after another.
- A basis, and how to write the same vector in two different bases. An orthonormal frame is a basis of unit, mutually perpendicular vectors.
- Eigenvalues and eigenvectors of symmetric 2×2 and 3×3 matrices: why they are real, why the eigenvectors are perpendicular, and what they mean as "the directions the matrix only stretches".
- Least squares: why you solve the normal equations `AᵀA x = Aᵀb`, and what they mean geometrically (projecting `b` onto the column space of `A`).
- Conditioning: when a tiny change in the input swings the answer, and why centring and scaling the inputs helps.

**You're ready when you can…**
- find the eigenvalues and eigenvectors of `[[2,1],[1,2]]` by hand and say what the matrix does to the plane;
- write a vector given in the world basis in a rotated orthonormal basis, and back again;
- fit a straight line to five points by writing out and solving the normal equations;
- explain why fitting a quadric to points that are far from the origin and tightly clustered is badly conditioned.

**Read.** For a refresher, 3Blue1Brown's *Essence of Linear Algebra* video series builds the geometric picture fast, and Gilbert Strang's MIT 18.06 lectures go deeper where you need it. Crane (DDG notes) recaps what the geometry chapters need. Botsch et al. (Polygon Mesh Processing) has a compact least-squares appendix.

### A2. Multivariable calculus and minimising energy

**Why Mobius needs it.** The cross field is "the smoothest field that respects the constraints", which is an energy minimisation (D5). Understanding why that is a linear system is the key to Module 5.

**What to understand.**
- The gradient (direction of steepest increase) and the Hessian (how the gradient itself changes).
- A quadratic energy `E(x) = ½ xᵀAx − bᵀx + c` and why its minimum satisfies `Ax = b` when `A` is symmetric positive definite.
- Soft constraints as penalties (`w |x − x*|²`, which pulls towards a target) versus hard constraints (fix the value and remove the unknown from the system).
- Lagrange multipliers at intuition level: a constraint pushes back on the energy with a force, and the multiplier is the size of that force.

**You're ready when you can…**
- minimise `(x − 3)² + 2(x − 1)²` by hand and then solve it again as a 1×1 linear system;
- explain why a larger penalty weight makes a soft constraint behave more like a hard one, and what goes wrong numerically if you make it huge;
- say what "positive definite" buys you (a single minimum, and CG works) in one sentence.

**Read.** 3Blue1Brown's *Essence of Calculus* series if derivatives feel rusty. Shewchuk (CG without the agonizing pain) starts with exactly this picture and is the best first read. Botsch et al. covers energy minimisation for meshes.

### A3. Complex numbers as rotations

**Why Mobius needs it.** The field is stored as `z = e^{4iθ}` per vertex (D5). This is the one trick that makes the whole field solvable as a linear system.

**What to understand.**
- A complex number as a point or arrow in the plane, and `e^{iθ} = cos θ + i sin θ`.
- Multiplication adds angles and multiplies lengths, so multiplying by `e^{iθ}` rotates by `θ`.
- A cross is four directions at 90° steps, so it looks identical after rotating by 90°. Raising the direction `e^{iθ}` to the fourth power sends all four directions to the same number `e^{4iθ}`. The ambiguity vanishes, and "the same cross" is "the same complex number".
- Going back: the cross direction is `θ = arg(z) / 4`, which has four solutions, as it should.

**You're ready when you can…**
- compute `(1+i)⁴` and say which direction it corresponds to;
- show that directions at 0°, 90°, 180° and 270° all give `z = 1`;
- explain why averaging two crosses as angles is wrong (try 5° and 85°) but averaging their `z` values is sensible.

**Read.** Knöppel et al. (Globally Optimal Direction Fields), the first sections. Vaxman et al. (Directional Field Synthesis) explains the representation in general terms.

### A4. Differential geometry of surfaces (intuition)

**Why Mobius needs it.** This is the vocabulary of Modules 2 and 4: curvature, singularities and why every closed surface needs poles.

**What to understand.**
- Tangent plane and normal at a point. The shape operator describes how the normal tilts as you move; its eigenvalues are the principal curvatures `k1`, `k2` and its eigenvectors are the principal directions.
- Mean curvature `(k1 + k2)/2` and Gaussian curvature `k1·k2`. A sphere has positive Gaussian curvature, a cylinder zero, a saddle negative.
- Umbilics: points where `k1 = k2`, so the principal directions are undefined. This is why direction confidence exists (D4).
- Geodesics (straightest paths on the surface) and parallel transport (carrying a vector along a path without turning it). Carry a vector around a loop on a curved surface and it comes back rotated. That rotation is the holonomy, and it equals the total Gaussian curvature enclosed.
- The index of a singularity: how many turns the field makes around it, in units of a quarter turn for a cross field.
- Gauss–Bonnet (total Gaussian curvature is `2π · χ`) and Poincaré–Hopf (the indices of a field's singularities sum to the Euler characteristic). Genus is the number of handles.

**You're ready when you can…**
- say the signs of `k1` and `k2` for a sphere, a cylinder, a saddle and a flat plane;
- explain with a picture why you cannot comb a hairy sphere flat, and why a torus can be combed;
- show that a quad mesh of a sphere whose only poles are 3- and 5-valent must have exactly eight more 3-poles than 5-poles, and check it on a cube (hint: a vertex of valence `v` has index `(4 − v)/4` in a 4-fold field, and the indices must sum to 2);
- explain why indices sum to 0 on a torus.

**Read.** Crane (DDG notes), the early lectures on curvature and Gauss–Bonnet. Ray et al. (N-Symmetry Direction Field Design) for index in the N-fold setting.

### A5. Discrete differential geometry

**Why Mobius needs it.** The engine works on triangle meshes (D3), so every concept above needs a discrete version.

**What to understand.**
- The cotangent Laplacian: why the weights are `½(cot α + cot β)`, what it does to a function (smooths it), and why it is symmetric and positive semi-definite even though single weights turn negative on obtuse triangles.
- Angle defect: `2π` minus the sum of the triangle angles at a vertex is the discrete Gaussian curvature. A flat vertex has defect 0.
- A discrete connection: each vertex has its own tangent frame, so comparing vectors at neighbouring vertices needs a rotation between the two frames. That rotation is the parallel transport along the edge. The connection Laplacian is the Laplacian that includes it, and it is what the field energy uses.
- The period jump on an edge, and how summing the jumps around a triangle finds singularities (D5).
- Discrete exterior calculus at intuition level: functions live on vertices, edge quantities on edges, area quantities on faces, and operators connect them. You don't need the machinery, only to recognise it in the papers.

**You're ready when you can…**
- compute the angle defect at the apex of a cone made from a square pyramid and at a flat vertex;
- write the cotan weight for an edge given the two opposite angles;
- explain in words why the connection Laplacian has complex entries and an ordinary Laplacian does not;
- say why summing angle defects over a closed mesh gives `2π χ` (a discrete Gauss–Bonnet).

**Read.** Crane (DDG notes), Meyer et al. (Discrete Differential-Geometry Operators), Knöppel et al. (Globally Optimal Direction Fields), and Botsch et al. (Polygon Mesh Processing) for the Laplacian.

### A6. Numerical computing

**Why Mobius needs it.** The pipeline must give bit-identical output for any thread count (D9, D16). That is only possible if you know why floating-point maths is not associative.

**What to understand.**
- Floating-point numbers: a sign, an exponent and a limited number of significant digits, so every operation rounds.
- Why `(a + b) + c` can differ from `a + (b + c)`, and therefore why summing in a different order, as parallel threads do, changes the result.
- Cancellation (subtracting nearly equal numbers loses digits) and the condition number of a problem.
- The remedy Mobius uses: fixed chunk boundaries and a fixed combination order (D9).

**You're ready when you can…**
- find, with a calculator or a few lines of code, three numbers where addition order changes the result;
- explain why a sum of a million small numbers added to one large number first can lose them all;
- explain how fixed-chunk reductions make a parallel sum repeatable.

**Read.** Goldberg (What Every Computer Scientist Should Know About Floating-Point). Intuition level is enough, so read the first third and skim the rest.

---

## Part B – Geometry-processing basics (modules 0–3)

### B1. What good topology is (Module 0)

**Why Mobius needs it.** This defines the target. Every objective and metric in the engine is a stand-in for what an artist would call good flow.

**What to understand.**
- Edge loops and edge flow: rings of edges that follow the form and let a mesh bend, shade and subdivide cleanly.
- Valence, and poles: a vertex with 3 or 5 edges instead of 4. Poles are unavoidable (A4) but their placement matters.
- Quad-dominant versus all-quad: what you gain from allowing a few triangles and what you lose.
- What each downstream use needs: deformation wants loops around joints; UVs want few, long, clean seams; baking wants even density; LODs want a structure that reduces gracefully.

**You're ready when you can…**
- point at a hand-made character head and name where the loops run and where the poles sit;
- explain why a pole on a flat cheek is less harmful than one across an elbow crease;
- explain what is wrong with a mesh that is "all quads" but has long skinny faces.

**Read.** Bommes et al. (Quad-Mesh Generation and Processing: A Survey), the introduction and requirements sections.

### B2. Meshes as data (Module 1)

**Why Mobius needs it.** The engine's first job is to accept messy meshes and turn them into something the numerical code can trust (D3).

**What to understand.**
- A face list (vertices plus index arrays) versus a half-edge structure (each edge stored as two directed halves, with next, twin and face links) and what each makes cheap.
- Manifold versus non-manifold: an edge shared by more than two faces, a vertex whose faces don't form one fan, and so on.
- Boundaries and holes, and how orientation (consistent winding) decides which way is outside.
- The Euler characteristic `V − E + F`, and how it relates to genus (`χ = 2 − 2g` for a closed orientable surface).
- Triangulating n-gons (fan versus ear clipping) and keeping a map back to the source face (D3).
- Validation and repair: what to report, what to fix and what to leave.
- Attributes: data attached to vertices, faces or corners, and what happens to them when topology changes.

**You're ready when you can…**
- compute `V − E + F` for a torus mesh by hand and explain why it is 0;
- list three different ways a mesh can be non-manifold and how Blender highlights each;
- draw a half-edge structure for two triangles and trace `next` and `twin` around one;
- say how many triangles any pentagon splits into, and draw a concave pentagon where a fan from the wrong corner fails (which is why ear clipping exists).

**Read.** Botsch et al. (Polygon Mesh Processing), the data-structures chapter. Crane (DDG notes), the early lectures on meshes and topology.

### B3. Spatial queries

**Why Mobius needs it.** Closest-point projection is how the output is snapped back to the source surface (D6.5), how symmetry is measured (D8) and how deviation is evaluated (D13). Ray casts are used in the addon for strokes (D14).

**What to understand.**
- A bounding-volume hierarchy (BVH): a tree of boxes that lets a query skip most of the mesh.
- Closest-point-on-triangle, and how a BVH query prunes branches whose box is farther than the best answer so far.
- Ray–triangle intersection and ray casts against a BVH.
- Why these queries are the workhorse of almost every step: projection, sampling, evaluation and picking.

**You're ready when you can…**
- explain how a BVH query avoids testing every triangle, using a picture of nested boxes;
- compute the closest point on a triangle to a given point, including the edge and vertex cases;
- describe how you would build a simple BVH over 1,000 triangles and what makes a split good.

**Read.** Ericson (Real-Time Collision Detection), the bounding-volume and closest-point chapters.

### B4. Curvature and features (Module 2)

**Why Mobius needs it.** Curvature directions are the soft target for the cross field (D5), and features are the hard constraints (D3, D4). Local length scale sets the face size (D4, D6).

**What to understand.**
- The estimators: quadric fitting over a neighbourhood (the default in D4), Rusinkiewicz tensor averaging (the fast fallback) and the Meyer et al. operators (angle defect, mean-curvature normal).
- Noise versus neighbourhood radius: a bigger radius suppresses noise but blurs small features. This is why the radius is a parameter.
- Direction confidence `c = |k1 − k2| / (|k1| + |k2| + ε)`: why a sphere is low-confidence and a cylinder high-confidence.
- Dihedral-angle features: the angle between adjacent face normals, a threshold, chains traced through feature-degree-2 vertices, minimum-length filtering and corners.
- Local length scale `h(v) = clamp(α / sqrt(k1² + k2²), h_min, h_max)`: small faces where curvature is high, large where it is low, then smoothed.

**You're ready when you can…**
- explain why the principal directions on a perfect sphere are meaningless and what the engine does about it;
- predict where feature chains and corners appear on a bevelled box at two different angle thresholds;
- derive the length-scale formula's behaviour for a sphere of radius `r` (what `h` is) and for a plane;
- implement a quadric fit on a small neighbourhood and compare its curvature with a known analytic surface.

**Read.** Rusinkiewicz (Estimating Curvatures and Their Derivatives), Meyer et al. (Discrete Differential-Geometry Operators), and Botsch et al. (Polygon Mesh Processing) for the curvature chapter.

### B5. Measuring quality (Module 3)

**Why Mobius needs it.** Every claim the engine makes about a result is a number in the run report, and the regression suite compares them (D13).

**What to understand.**
- Valence statistics: the fraction of regular (valence 4) vertices and the number and spread of irregular ones.
- Element quality: how square a quad is (angles, aspect ratio, Jacobian) and why skinny elements hurt.
- Hausdorff and surface deviation: how far the result strays from the source, measured both ways by sampling points and finding the closest point on the other surface (Metro).
- Silhouette IoU: render both from many directions and compare the overlap.
- Feature preservation: do the output edges run along the source's feature curves?
- Reading a run report: which numbers are warnings and which are failures.

**You're ready when you can…**
- explain why deviation must be measured in both directions (give a failure case for one direction alone);
- compute the aspect ratio and the angle quality of a given quad;
- implement uniform area-weighted sampling of a triangle mesh with a fixed seed;
- read a run report and say in plain words whether the result is acceptable.

**Read.** Cignoni et al. (Metro), and Bommes et al. (Quad-Mesh Generation and Processing: A Survey) for quality criteria.

---

## Part C – The retopology core (modules 4–7)

### C1. Direction fields (Module 4)

**Why Mobius needs it.** The field is the skeleton of the output mesh: the edges will follow it, and the poles will sit at its singularities (D5).

**What to understand.**
- Vector fields (one arrow), line fields (an arrow up to sign) and cross fields (four arrows at 90° steps). N-RoSy is the general idea: N-fold rotational symmetry.
- The trade-off between smoothness and alignment: a smooth field ignores the shape, an aligned field is noisy, and the energy balances them.
- The connection Laplacian formulation (Knöppel et al.): `z = e^{4iθ}` per vertex, a Hermitian energy, the smoothest field as the eigenvector or, with constraints, the solution of a linear system.
- Singularities and their index, found by summing period jumps around each triangle, and why they are the poles of the final mesh.
- Controlling singularity placement: trivial connections (you choose where the singularities are, and the field follows), and the cheaper per-edge weighting used by Mobius (D5), which makes rotation cost more or less in a region.
- Guidance as soft constraints (a penalty towards a target) and hard constraints (values fixed). Features and boundaries are hard, strokes and curvature are soft.
- Deformation-aware fields (Marcias et al.) as a later input: a field computed from how the surface stretches across poses, which could feed the deformable-character objective as a guidance layer.

**You're ready when you can…**
- explain why the energy is quadratic in `z`, and so why a linear solve gives the minimum;
- find the singularities on a small triangulated patch by summing period jumps;
- say what happens to a field when you raise the edge weight inside a repel region, and why singularities leave it;
- explain why a repel region that must contain a non-zero total index cannot be satisfied (D5), using Poincaré–Hopf;
- explain the difference between a soft stroke and a hard feature in one sentence each.

**Read.** Vaxman et al. (Directional Field Synthesis), Knöppel et al. (Globally Optimal Direction Fields), Ray et al. (N-Symmetry Direction Field Design), Crane et al. (Trivial Connections), and Marcias et al. (Animation-Aware Quadrangulation) for the later input.

### C2. Solving at scale (Module 5)

**Why Mobius needs it.** A 500,000-vertex field is a sparse system of a million unknowns. How fast and how repeatably it is solved decides whether previews feel interactive (D16).

**What to understand.**
- Sparse matrices and CSR storage: only the non-zeros are kept, in three arrays, and a matrix-vector product costs time proportional to them.
- Symmetric positive-definite systems, and why the field system, rewritten as a real system of size `2n`, is one (D5).
- Conjugate gradients (CG): each step picks a new search direction that does not undo the progress of earlier ones. Shewchuk's picture of ellipses is the intuition.
- Preconditioning: change the problem so that CG converges faster. Jacobi (divide by the diagonal) is the cheapest. Multigrid solves a coarse version of the problem to remove the smooth part of the error, which is what makes convergence nearly independent of mesh size.
- Mesh hierarchies for multigrid (Gravo MG, D16 / task 11.9): how to coarsen a curved surface so the coarse problem still means something.
- Residuals and stopping rules: relative residual tolerance, an iteration cap, and what the plan does when the cap is hit (warn and continue with the best iterate).
- Warm starts: begin from the curvature directions, not from zero.
- Why direct solvers (sparse Cholesky and friends) are hard to put on a GPU, and why the plan chooses an iterative solver.

**You're ready when you can…**
- store a small sparse symmetric matrix in CSR by hand and multiply it by a vector;
- implement CG for a small dense SPD system and plot the residual;
- explain what Jacobi preconditioning changes and when it is not enough;
- explain in one paragraph what a multigrid V-cycle does and why it helps with smooth error;
- explain why the solve must use fixed-chunk reductions to stay bit-identical (links back to A6).

**Read.** Shewchuk (CG without the agonizing pain) first. Then Gravo MG for the hierarchy idea. Saad (Iterative Methods for Sparse Linear Systems) as a reference only, and GravoTet for its boundary-preservation ideas, since Mobius works on surfaces and not volumes.

### C3. From field to quads (Module 6)

**Why Mobius needs it.** The field only says which way edges should run. Turning it into actual vertices and faces is the hardest step, and the choice of method decides robustness (D6).

**What to understand.**
- The Instant Meshes approach that D6 follows: a position field (each vertex also carries a point on a lattice aligned to the cross field), then extraction by collapsing vertex pairs that share a lattice cell and reading faces off the cycles.
- Why extraction produces some triangles and pentagons: lattices from neighbouring regions may not meet cleanly at singularities, or where the surface cannot be tiled exactly.
- All-quad by subdivision: extract at twice the edge length and split each face once (D6.4).
- The alternatives and what each buys:
  - *Global seamless parametrisation and integer-grid maps* (MIQ, QuadCover, Integer-Grid Maps, QEx, Quantized Global Parametrization): a map from the surface to the plane in which the grid lines are the integer lines. Higher quality layouts, but the integer rounding is fragile on noisy scans and slower.
  - *QuadriFlow*: the method behind Blender's remesher, a scalable and robust alternative to the Instant Meshes extraction. Read the paper for how it differs.
  - *Quad layouts and patch-based methods* (QuadWild and feature-line-driven methods, relaxed singularities): partition the surface into a few large quad patches first, then fill them. Strong on hard-surface input.
  - *Sketch and patch-based control* (Sketch-Retopo, pattern-based quadrangulation): the artist draws patches with edge counts and the engine fills them. A model for direct control.

**You're ready when you can…**
- step through extraction on a small mesh and say where a triangle or pentagon appears and why;
- explain how the position field's lattice spacing relates to the target edge length, and how D6 varies it (`h_eff`);
- explain in your own words why integer-grid maps give better layouts and why they are more fragile;
- say what Blender's QuadriFlow does differently from Instant Meshes, and which input types would make you pick a patch-based method instead.

**Read.** Instant Meshes (Instant Field-Aligned Meshes) first. Bommes et al. (Quad-Mesh Generation and Processing: A Survey) for the map. Then, as needed: MIQ, QuadCover, Integer-Grid Maps, QEx, Quantized Global Parametrization, QuadriFlow, QuadWild (a reference, not a dependency), Lyon et al. (Simpler Quad Layouts), Sketch-Retopo and Pattern-Based Quadrangulation.

### C4. Symmetry, seams and preserved regions (Module 7)

**Why Mobius needs it.** Artists expect a mirrored character to get a mirrored mesh, and expect to keep hand-made parts of the topology. Both need seams that close without T-junctions (D7, D8).

**What to understand.**
- Measuring reflective symmetry: reflect each vertex across the plane and take the closest-point distance to the surface (D8). The result is a per-vertex map of "symmetric here, not there".
- Partial and approximate symmetry: real models are mirrored in most places and not in others (a sword arm, a pose). Mitra et al. and the planar reflective symmetry transform are the background.
- Correspondence: mapping a point to its mirror partner on the surface, and carrying tangents through the reflection.
- Stitching patches without T-junctions: two meshes meet along a seam only if their boundary vertices line up one to one.
- The parity theorem: the boundary of a quad mesh patch has an even number of edges. The Takayama et al. report proves it by induction. This is why a seam with an odd number of vertices cannot be closed with quads, and why D7 splits one boundary edge to fix the parity.
- Filling patches with prescribed edge counts: the Takayama report shows how to fill 3- and 4-sided regions for any valid counts.
- Learned symmetry detection (SymCL) only as a future out-of-process provider. It is not part of v1.

**You're ready when you can…**
- explain why a mirror plane cut produces a boundary curve for the field, and why the field must cross it at right angles or run along it;
- show with a small picture why an odd boundary count cannot be tiled with quads;
- say what the "primary side" is and why the engine meshes one half and mirrors it;
- explain what a T-junction is and why it breaks a quad mesh.

**Read.** Takayama et al. (Robust and Controllable Quadrangulation), Mitra et al. (Partial and Approximate Symmetry Detection), Podolak et al. (Planar-Reflective Symmetry Transform), and SymCL for the later provider.

---

## Part D – Engineering the engine (no gate, learned alongside the build)

These are working skills for the build, not gated theory. The self-checks are lighter.

### D1. C++23 and modules

Used as described in `docs/coding-style.md`: named modules in `.ixx` interface units, `import std;`, `std::expected`, snake_case and strong index types. Read that file and follow it. You don't need to learn C++ from scratch to review the code, but you should be able to read a function and say what it does.
- *You're ready when you can…* read one short module interface and name what it exports.

### D2. Deterministic parallelism

Fixed chunk boundaries that do not depend on the thread count, reductions combined in chunk order, no iteration over hash maps, and randomness from a seeded PCG32 with per-chunk streams (D9). This is A6 applied.
- *You're ready when you can…* explain why a parallel sum with dynamic work stealing can differ between runs, and how fixed chunks prevent that.

### D3. C ABI design and ctypes from Blender

The public header is plain C99 (D11) with opaque handles, copied inputs and named settings. Runs are asynchronous and the host polls, because a callback from a foreign thread into Python risks deadlocks with Blender's GIL. Ctypes in the addon loads the DLL, checks the API version and passes NumPy buffers by pointer.
- *You're ready when you can…* explain why polling is safer than callbacks here, and why the handles are opaque.

### D4. Blender data access (intuition only)

`bmesh` versus mesh attributes, fast reads with `foreach_get`, `BVHTree` for ray casts, and `draw_handler_add` with the `gpu` module for overlays (D14). Guidance is stored as object properties and attributes, then re-anchored to the evaluated mesh at run time.
- *You're ready when you can…* say where a painted density map is stored and how it reaches the engine.

### D5. Testing numerical code

Analytic cases (a sphere has curvature `1/r`), brute-force references (a dense solve on a small random system), and regression bounds on corpus metrics (D13). Each in-house maths routine gets one of each.
- *You're ready when you can…* name an analytic test for a curvature estimator and a brute-force check for a CG solver.

---

## Part E – Later, not needed for retopology v1

**UV parametrisation.** Flattening a surface to the plane with low distortion is the next operation the architecture is built for. ARAP (as-rigid-as-possible) and LSCM are the usual starting points. The Smooth ARAP paper (local copy in `science/`) is a higher-order-continuity variant. Read when UV work begins.

**Learned meshing and cross fields.** NeurCross (a neural approach to cross fields) and QuadGPT (generative quad meshes) are research directions, as is other learned cross-field work such as CrossGen. If they are ever used, they would only supply guidance layers through the existing API. The project rule is that AI assists but deterministic code makes the geometry. Treat them as a watch list.

**Isosurface and remeshing tools for broken inputs.** TetWeave extracts isosurfaces from on-the-fly tetrahedral grids for gradient-based optimisation. It is kept for reference and judged low value for Mobius: its output is triangles, it relies on PyTorch and GPU optimisation, and it is weak on thin parts.

---

## Suggested order

A rough path for part-time study. The time estimates are guesses, not promises. They assume a few hours a week and will vary a lot with your background.

1. **A1 → A3 → A2** (linear algebra, complex numbers, energy minimisation). About 2–3 weeks. Do A3 early: it is short and unlocks the field ideas.
2. **B1 → B2** (good topology, meshes as data). About 1–2 weeks, and this is the gate for Modules 0 and 1.
3. **A4 → A5** (surface geometry, discrete versions). About 3–4 weeks. This is the longest stretch. Aim for intuition, and revisit once Module 2 needs it.
4. **B3 → B4** (spatial queries, curvature and features). About 2–3 weeks, with the Module 2 hands-on.
5. **A6 → B5** (floating point, quality metrics). About 1–2 weeks.
6. **C1** (direction fields). About 3–4 weeks. This is the conceptual core, so don't rush it.
7. **C2** (solving at scale). About 2–3 weeks, mostly reading Shewchuk and writing a small CG.
8. **C3** (field to quads). About 3–4 weeks. Learn Instant Meshes well and skim the alternatives.
9. **C4** (symmetry and seams). About 2 weeks.
10. **Part D** alongside the build. **Part E** only when a later operation needs it.

As a whole, expect roughly four to six months at a part-time pace. Overlap is fine: the modules in README.md gate the implementation, not your reading, so you can read ahead on any topic that interests you.
