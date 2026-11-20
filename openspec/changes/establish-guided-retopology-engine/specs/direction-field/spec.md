# Spec Delta

## Purpose

Produces the smooth 4-way rotationally symmetric (cross) field that dictates edge flow in the output mesh, aligned to features, curvature and artist intent, with singularities that are explicit and controllable.

## ADDED Requirements

### Requirement: Smooth cross field
The engine SHALL compute a 4-RoSy cross field tangent to the source surface that minimises rotational variation (smoothness energy) subject to the active alignment constraints.

#### Scenario: Unconstrained plane
- **WHEN** a cross field is computed on a flat square patch with no constraints other than its boundary
- **THEN** the field is aligned to the boundary edges and has no interior singularities

### Requirement: Hard alignment constraints
Feature curves, open boundaries and edges marked as preserved SHALL act as hard constraints: the field SHALL align exactly with their tangent at every constrained location.

#### Scenario: Field follows a feature curve
- **WHEN** a cross field is computed on a mesh with a detected feature curve
- **THEN** at every vertex on that curve, one field direction lies within 1 degree of the curve tangent

### Requirement: Soft alignment constraints
Principal curvature directions and artist direction strokes SHALL act as soft constraints. Each SHALL have a weight, and curvature directions SHALL also be scaled by their confidence. A larger weight SHALL pull the field more strongly toward that direction.

#### Scenario: Stroke weight increases alignment
- **WHEN** the same direction stroke is applied once with weight 0.2 and once with weight 1.0
- **THEN** the mean angular deviation from the stroke along its path is smaller with weight 1.0

#### Scenario: Curvature alignment on a cylinder
- **WHEN** a cross field is computed on a cylinder with curvature alignment enabled
- **THEN** away from the caps, one field direction follows the cylinder axis within 5 degrees

### Requirement: Singularity reporting
The engine SHALL identify every field singularity, reporting its location and index (for example -1/4 or +1/4). The sum of indices SHALL equal the Euler characteristic of each closed component.

#### Scenario: Sphere singularities
- **WHEN** a cross field is computed on a closed sphere
- **THEN** singularity indices sum to 2 and every singularity is reported with its position

### Requirement: Singularity placement control
Artists SHALL be able to attract singularities to, or repel them from, marked surface regions. The engine SHALL report when a request cannot be satisfied for topological reasons.

#### Scenario: Repelling region
- **WHEN** a region over a character's eye is marked as singularity-repelling
- **THEN** no singularity lies inside that region in the computed field, provided the region is topologically a disk

### Requirement: Solver convergence reporting
Field computation SHALL report the solver's iteration count and final relative residual. If the solver reaches its iteration limit before its tolerance, the run SHALL continue with the best field found and record a warning that names the direction-field stage.

#### Scenario: Converged solve reported
- **WHEN** a cross field is computed on the reference sphere with default settings
- **THEN** the run report contains the iteration count and a final residual below the configured tolerance

#### Scenario: Iteration limit reached
- **WHEN** a cross field is computed with the solver iteration limit set to 1
- **THEN** the run still produces a field, and the diagnostics contain a non-convergence warning naming the direction-field stage

### Requirement: Field export
The engine SHALL make the computed field and its singularities available to the host, so they can be visualised before meshing.

#### Scenario: Preview without meshing
- **WHEN** a host requests field computation only
- **THEN** the engine returns per-vertex field directions and the singularity list without running quad extraction
