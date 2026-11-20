# Spec Delta

## Purpose

Defines one model for every kind of guidance (direction, density, features, regions, surface curves), whether an artist or an external analysis provider such as a local AI model supplies it, and how each kind must shape an operation's solver rather than post-process its output. Every operation shares this model.

## ADDED Requirements

### Requirement: Operation-specific layer kinds
Shared layer kinds (direction strokes, density maps, feature locks, regions, surface curves) SHALL be usable by any operation. An operation SHALL be able to add layer kinds of its own to the shared set of layer kinds. A run that includes a layer kind its operation does not accept SHALL fail before processing, with an error naming the layer and listing the accepted kinds.

#### Scenario: Shared curve reused
- **WHEN** a surface curve layer is supplied to the retopology operation
- **THEN** it is accepted, and acts as a patch boundary as defined for retopology

#### Scenario: Unsupported layer kind
- **WHEN** a retopology run includes a layer of a kind that only another operation accepts
- **THEN** the run fails before processing, with an error naming that layer and listing the kinds retopology accepts

### Requirement: Guidance layers
Guidance SHALL be supplied as a set of typed layers. Each layer has a kind, a source label (for example "artist" or a provider name), a weight in [0, 1] and an enabled flag. Several layers of the same kind SHALL be allowed and combined by weight.

#### Scenario: Disabled layer has no effect
- **WHEN** a direction layer is disabled
- **THEN** the result is identical to a run without that layer

#### Scenario: Provider and artist layers combined
- **WHEN** an artist density layer and a provider density layer cover the same region
- **THEN** the effective density in that region is their weight-blended combination

### Requirement: Direction strokes
A direction stroke SHALL be a polyline on the source surface, given as surface points (face index plus barycentric coordinates). It SHALL act as a soft cross-field alignment constraint along its path, with influence falling off over a configurable radius.

#### Scenario: Stroke steers edge flow
- **WHEN** an artist draws a stroke around a character's mouth and retopologises
- **THEN** the output has a continuous edge loop within one target edge length of the stroke path

### Requirement: Density maps
A density map SHALL be a per-vertex scalar on the source mesh that multiplies the local target resolution. A value of 1 means default, greater than 1 means denser, and less than 1 means sparser.

#### Scenario: Zero-change density map
- **WHEN** a density map with every value set to 1 is supplied
- **THEN** the result is identical to a run without a density map

### Requirement: Feature and crease locks
Artists SHALL be able to mark source edges as preserved features or as excluded from automatic detection. Marked edges SHALL be treated as hard constraints, and excluded edges SHALL never be treated as features.

#### Scenario: Forced feature on a smooth surface
- **WHEN** an artist marks an edge loop on a smooth sphere as a preserved feature
- **THEN** the output contains an edge loop that follows the marked loop

#### Scenario: Excluded sharp edge
- **WHEN** an artist excludes a sharp edge that automatic detection would report
- **THEN** that edge is not treated as a feature and does not constrain the field

### Requirement: Region constraints
Artists SHALL be able to define face regions on the source mesh and give each region overrides: density multiplier, singularity attract or repel, field smoothness weight, a deformation-zone flag, and "keep existing topology" (copy the source faces unchanged and stitch them conformingly to the surrounding output).

#### Scenario: Keep existing topology
- **WHEN** a region covering an already clean hand is marked "keep existing topology"
- **THEN** the output contains the hand's original faces unchanged and connects them to neighbouring output faces without gaps or T-junctions

### Requirement: Surface curves
Artists SHALL be able to draw closed or open curves on the source surface, given as surface points. In retopology a surface curve is a patch boundary: the output SHALL contain an edge chain along each curve, so the surface is partitioned as the artist intended.

#### Scenario: Seam-like boundary
- **WHEN** an artist draws a closed curve separating a character's head from its neck
- **THEN** the output contains a closed edge loop following that curve

### Requirement: Conflict reporting
When guidance layers conflict, the engine SHALL resolve them using this precedence: hard constraints, then region overrides, then soft layers weighted by weight. It SHALL report each conflict it detects, with its location.

#### Scenario: Stroke crossing a hard feature
- **WHEN** a direction stroke crosses a preserved feature curve at 45 degrees
- **THEN** the feature constraint wins at the crossing, and the diagnostics report the conflict location

### Requirement: Guidance reference validation
Guidance anchored with surface points SHALL remain valid for the source mesh it was authored on. If the host submits guidance referring to faces that do not exist, the engine SHALL reject it with an error naming the layer.

#### Scenario: Stale guidance
- **WHEN** a stroke references a face index beyond the mesh face count
- **THEN** the run fails before processing with an error naming the stroke's layer
