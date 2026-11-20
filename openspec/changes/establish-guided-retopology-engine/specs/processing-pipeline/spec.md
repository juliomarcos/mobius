# Spec Delta

## Purpose

Runs every Mobius operation (retopology first, then UV unwrapping, normals control and texturing) as an ordered set of replaceable stages. Results are deterministic, and every run gets progress, cancellation and a report, so operations and strategies can be developed and compared independently and hosts can drive long runs interactively.

## ADDED Requirements

### Requirement: Operations
Processing SHALL be organised into named operations. Each operation declares its ordered stages, its strategies, its objectives and any guidance layer kinds of its own. Adding an operation SHALL require no changes to existing operations. Hosts SHALL be able to list the available operations.

#### Scenario: Operation listing
- **WHEN** a host lists operations on this release
- **THEN** it receives "retopology" with its stages, strategies and objectives

#### Scenario: Unknown operation
- **WHEN** a run requests an operation name that is not registered
- **THEN** the run fails before processing with an error listing the available operations

### Requirement: Shared stages
Every operation SHALL begin with the shared stages validation/repair, surface analysis, symmetry analysis (when symmetry is enabled) and guidance resolution, and SHALL end with evaluation. These stages SHALL behave identically whichever operation runs them.

#### Scenario: Shared analysis is consistent
- **WHEN** the same mesh and guidance are run through two operations that both stop after guidance resolution
- **THEN** both runs return identical analysis and resolved guidance

### Requirement: Retopology operation stages
The retopology operation SHALL run, after the shared stages, the direction field, quad meshing and post-processing stages, then evaluation. A run SHALL be able to stop after any named stage and return that stage's outputs.

#### Scenario: Stop after field
- **WHEN** a host requests a retopology run that ends after the direction-field stage
- **THEN** the run returns the analysis and field outputs and does not produce an output mesh

### Requirement: Pluggable strategies
Each operation-specific stage SHALL be implemented by a named strategy chosen per run. Adding a strategy SHALL require no changes to other stages, and the strategies of each operation SHALL be listable at runtime.

#### Scenario: Strategy listing
- **WHEN** a host queries the strategies of the retopology operation
- **THEN** it receives each stage's strategy names, along with the default strategy for each stage

#### Scenario: Unknown strategy
- **WHEN** a run requests a strategy name that is not registered for that stage
- **THEN** the run fails before processing with an error listing the valid names for that stage

### Requirement: Per-operation objectives
Each operation SHALL define its own named objectives using one shared preset format. Selecting an objective that belongs to another operation SHALL fail before processing.

#### Scenario: Objective from the wrong operation
- **WHEN** a retopology run names an objective that is not a retopology objective
- **THEN** the run fails with an error listing the retopology objectives

### Requirement: Determinism
Given identical operation, input mesh, guidance, objective, parameters, engine version and seed, a run SHALL produce bit-identical output on the same platform, build, compute backend and device, regardless of thread count.

#### Scenario: Repeat run
- **WHEN** the same retopology run is executed twice on the CPU reference backend, once with 1 thread and once with 8 threads
- **THEN** both output meshes are bit-identical

### Requirement: Compute backends
Data-parallel stage work SHALL run on a compute backend chosen per run. A CPU reference backend SHALL always be available and be the default. Hosts SHALL be able to list available backends and devices, and the run report SHALL name the backend and device used.

#### Scenario: Backend listing
- **WHEN** a host queries available compute backends on a machine without any GPU backend installed
- **THEN** the list contains the CPU reference backend, marked as the default

#### Scenario: Unavailable backend
- **WHEN** a run requests a compute backend that is not available
- **THEN** the run fails before processing, with an error listing the available backends

### Requirement: Backend equivalence
Any compute backend other than the CPU reference SHALL produce results whose evaluation metrics on the reference corpus are within documented tolerances of the CPU reference results. A backend SHALL NOT become selectable by default unless it meets those tolerances.

#### Scenario: Non-reference backend checked against CPU
- **WHEN** the regression suite runs with a non-reference backend available
- **THEN** it runs every corpus mesh on both backends and fails, naming the mesh and metric, if any metric differs beyond its tolerance

### Requirement: Progress reporting
The engine SHALL report progress as the operation name, the current stage name and a fraction in [0, 1], at least once per second during long stages.

#### Scenario: Progress updates
- **WHEN** a run takes longer than five seconds
- **THEN** the host receives progress updates naming each stage, and the fraction never decreases within a stage

### Requirement: Cancellation
A host SHALL be able to cancel a run at any time. The engine SHALL stop within one second, release the run's resources and report the run as cancelled rather than failed.

#### Scenario: Cancel during meshing
- **WHEN** a host cancels a retopology run during the quad-meshing stage
- **THEN** the run ends within one second with status "cancelled" and returns no output mesh

### Requirement: Run report
Every completed, failed or cancelled run SHALL produce a structured, machine-readable report. It SHALL contain operation, engine version, effective parameters and weights, strategies, compute backend and device, per-stage timings, diagnostics (warnings, conflicts, repairs) and evaluation results when available.

#### Scenario: Failed run still reports
- **WHEN** a retopology run fails during field computation
- **THEN** the report names the operation, includes the stages that completed with their timings, and has an error naming the failing stage

### Requirement: Retopology performance baseline
On the reference hardware profile documented with the test corpus, a 500k-triangle input retopologised to 10k faces SHALL complete in under 60 seconds in a Release build.

#### Scenario: Large sculpt
- **WHEN** the 500k-triangle reference sculpt is retopologised in Release with default settings to a 10k-face target
- **THEN** the total run time reported is under 60 seconds
