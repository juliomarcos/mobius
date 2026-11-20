# Spec Delta

## Purpose

Provides one shared symmetry service for every operation. It measures how symmetric a model actually is about a declared plane, splits the surface into symmetric and asymmetric regions, and mirrors guidance between sides, so that retopology now (and UVs, normals and texturing later) can be symmetric where the model is, and free where it is not.

## ADDED Requirements

### Requirement: Opt-in symmetry plane
Symmetry SHALL be off unless the user declares a mirror plane, given as a point and a normal in object space. With symmetry off, no symmetry analysis SHALL run and results SHALL be identical to a run that never had a plane.

#### Scenario: Asymmetric model without a plane
- **WHEN** an asymmetric model is processed without declaring a symmetry plane
- **THEN** no symmetry analysis appears in the run report, and the whole surface is processed as one

### Requirement: Symmetry map
When a plane is declared, the engine SHALL compute for every source vertex the distance between its mirror image and the source surface. Vertices within the symmetry tolerance, which is relative to the bounding-box diagonal, SHALL be classed as symmetric, and the rest as asymmetric. The map SHALL be available to hosts.

#### Scenario: Perfectly symmetric mesh
- **WHEN** the symmetry map is computed for a mesh that is exactly mirror-symmetric about the declared plane
- **THEN** every vertex is classed as symmetric, with a distance of zero within floating-point tolerance

#### Scenario: Map available before running
- **WHEN** a host requests symmetry analysis only
- **THEN** it receives per-vertex distances and classes, and no operation output is produced

### Requirement: Symmetric and asymmetric regions
The engine SHALL group asymmetric vertices into connected asymmetric regions, dropping regions smaller than a configurable minimum area. Artists SHALL be able to force regions to be treated as symmetric or asymmetric, and those overrides SHALL take precedence over the measured classes.

#### Scenario: Asymmetric prop detected
- **WHEN** the symmetry map is computed for a symmetric character holding a sword in the right hand
- **THEN** the right hand and sword form an asymmetric region, and the rest of the body is symmetric

#### Scenario: Artist override
- **WHEN** an artist forces a slightly asymmetric face region to be symmetric
- **THEN** that region is treated as symmetric regardless of its measured distances

### Requirement: Mirror correspondence
For symmetric regions, the engine SHALL provide each operation with a correspondence that maps surface points on one side to surface points on the other side, and maps tangent directions accordingly. Operations SHALL use this correspondence rather than a plain reflection, so that small differences between sides follow each side's own surface.

#### Scenario: Correspondence lands on the surface
- **WHEN** a point on the left cheek of an approximately symmetric head is mapped to the right side
- **THEN** the mapped point lies on the source surface within 1e-6 of the bounding-box diagonal

### Requirement: Mirrored guidance
Inside symmetric regions, guidance placed on either side SHALL be mirrored to the other side and combined with any guidance already there. Disagreements between the two sides SHALL be reported as conflicts with their locations. Guidance in asymmetric regions SHALL NOT be mirrored.

#### Scenario: One-sided strokes apply to both sides
- **WHEN** an artist draws direction strokes only on the right side of a symmetric face
- **THEN** the resolved guidance contains the mirrored strokes on the left side, and the result follows them on both sides

#### Scenario: Opposing strokes
- **WHEN** matching strokes on the two sides of a symmetric region disagree in direction by more than 30 degrees
- **THEN** a conflict is reported at that location

### Requirement: Symmetry coverage report
The run report SHALL state the fraction of surface area classed as symmetric. When that fraction is below a configurable threshold, it SHALL also give a warning that symmetry brings little benefit for this model.

#### Scenario: Mostly asymmetric model
- **WHEN** a plane is declared for a model where only 20% of the surface area is symmetric and the threshold is 50%
- **THEN** the run completes, and the report shows 20% coverage along with a low-symmetry warning
