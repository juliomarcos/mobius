# Spec Delta

## Purpose

Lets users choose what the retopology is for (static game asset, deformable character, organic sculpt or hard surface), so the engine applies optimisation priorities suited to that use while staying fully tunable.

## ADDED Requirements

### Requirement: Built-in objectives
The engine SHALL provide four built-in objectives: static game asset, deformable character, organic sculpt and hard surface. Exactly one objective SHALL be active per run, and static game asset SHALL be the default.

#### Scenario: Default objective
- **WHEN** a run is started without naming an objective
- **THEN** the static game asset objective is used and is named in the run report

### Requirement: Optimisation priorities
Each objective SHALL be defined as a set of named priority weights covering at least: polygon budget efficiency, silhouette preservation, edge-flow continuity, surface approximation, feature preservation, deformation readiness and singularity minimisation. Each weight SHALL measurably affect the stage it governs.

#### Scenario: Hard-surface prioritises features
- **WHEN** the same bevelled mechanical part is retopologised with the hard-surface objective and with the organic sculpt objective
- **THEN** the hard-surface result has a higher feature-preservation score in the evaluation report

#### Scenario: Static asset spends budget on silhouette
- **WHEN** a static prop is retopologised at a fixed face budget with the static game asset objective
- **THEN** its silhouette deviation is lower than with the organic sculpt objective at the same budget

### Requirement: Deformation readiness
The deformable character objective SHALL favour continuous edge loops around regions marked as deformation zones, and SHALL keep field singularities out of those zones whenever the topology allows.

#### Scenario: Loops around a marked joint
- **WHEN** an elbow region is marked as a deformation zone and the deformable character objective is active
- **THEN** the output contains at least two closed edge loops encircling the limb within the marked zone, and no singularity lies inside the zone

### Requirement: Objective overrides
Users SHALL be able to override any individual priority weight of the active objective for a run without redefining the objective. The run report SHALL list the effective weights.

#### Scenario: Single weight override
- **WHEN** a user runs the organic sculpt objective with the silhouette weight overridden to 1.0
- **THEN** the run report shows silhouette = 1.0 and the objective's defaults for every other weight

### Requirement: Custom objective presets
Users SHALL be able to save and load named objective presets that hold a full set of weights and default parameters, in a human-readable file format.

#### Scenario: Preset round-trip
- **WHEN** a user saves a custom preset and loads it in a later session
- **THEN** the loaded weights and parameters are identical to the saved ones
