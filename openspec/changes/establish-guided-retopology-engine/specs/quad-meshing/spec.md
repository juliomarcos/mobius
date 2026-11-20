# Spec Delta

## Purpose

Builds the output polygon mesh from the direction field and target density, producing quad-dominant or all-quad topology that follows the field and stays faithful to the source surface and its features.

## ADDED Requirements

### Requirement: Target resolution
The user SHALL be able to set target resolution as an approximate face count or a target edge length. The resulting face count SHALL be within 10% of a requested face count on the reference corpus.

#### Scenario: Face-count target
- **WHEN** a sphere is retopologised with a target of 2000 faces
- **THEN** the output has between 1800 and 2200 faces

### Requirement: Field-aligned edges
Output edges SHALL follow the direction field: away from singularities and constrained features, each edge direction SHALL lie close to one of the field directions at that location.

#### Scenario: Alignment on reference corpus
- **WHEN** a reference mesh is retopologised
- **THEN** the median angle between output edges and the nearest field direction is below 10 degrees

### Requirement: Output topology modes
The engine SHALL support two output modes: quad-dominant (quads plus a minimal number of triangles and pentagons) and all-quad. All-quad output SHALL contain only four-sided faces.

#### Scenario: All-quad mode
- **WHEN** a closed mesh is retopologised in all-quad mode
- **THEN** every output face has exactly four vertices

#### Scenario: Quad-dominant mode
- **WHEN** a smooth closed mesh is retopologised in quad-dominant mode
- **THEN** at least 95% of output faces are quads

### Requirement: Valid output topology
The output mesh SHALL be 2-manifold with consistent orientation, have no degenerate or zero-area faces, and keep the same number of boundary loops and components as the input. On closed components it SHALL preserve genus.

#### Scenario: Torus genus preserved
- **WHEN** a torus is retopologised
- **THEN** the output is a closed, consistently oriented manifold with genus 1

### Requirement: Feature and boundary snapping
Output vertices SHALL lie on preserved feature curves and boundaries, with chains of output edges following each such curve. Feature corners SHALL coincide with output vertices.

#### Scenario: Hard-surface edges preserved
- **WHEN** a bevelled box is retopologised with feature preservation enabled
- **THEN** each detected feature curve is represented by a continuous chain of output edges, and each feature corner is an output vertex

### Requirement: Surface fidelity
Output vertices SHALL be projected onto the source surface. The engine SHALL report the maximum and mean deviation between the output and the source surface.

#### Scenario: Vertices lie on the source
- **WHEN** retopology completes
- **THEN** each output vertex lies on the source surface within 0.1% of the bounding-box diagonal

### Requirement: Symmetric output
When a symmetry plane is declared, output topology in symmetric regions SHALL be mirror-symmetric, with mirrored vertices placed on their own side's surface and vertices on the plane lying exactly on it. Asymmetric regions SHALL be meshed freely and joined to the symmetric part without gaps or T-junctions.

#### Scenario: Symmetric head
- **WHEN** an approximately symmetric head is retopologised with an X-axis symmetry plane
- **THEN** every output vertex in the symmetric region has a mirror counterpart in the output topology, and vertices on the plane lie exactly on it

#### Scenario: Character with an asymmetric prop
- **WHEN** a symmetric character holding a sword in the right hand is retopologised with symmetry enabled
- **THEN** the body topology is mirror-symmetric, the sword arm is meshed freely, and the seam between them has no gaps or T-junctions

#### Scenario: Low-symmetry model
- **WHEN** a model with symmetry coverage below the threshold is retopologised with a plane declared
- **THEN** the run completes, the asymmetric parts are meshed freely, and the report contains the low-symmetry warning

### Requirement: Density adherence
Local output edge length SHALL follow the effective density field, which combines curvature-based local scale, artist density painting and objective settings.

#### Scenario: Painted high density
- **WHEN** one half of a sphere is painted with double density and the other half is left at the default
- **THEN** the mean edge length on the painted half is between 40% and 65% of that on the unpainted half
