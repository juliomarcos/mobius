# Spec Delta

## Purpose

Defines how the engine accepts, validates, diagnoses and emits polygon meshes, so that every later stage works on a well-defined surface and hosts receive clear feedback about input they cannot process.

## ADDED Requirements

### Requirement: Mesh input
The engine SHALL accept a polygon mesh given as vertex positions and polygon index lists of arbitrary arity (triangles, quads and n-gons). Optional per-vertex and per-face attributes supplied by the host SHALL be preserved and remain addressable by name.

#### Scenario: Mixed polygon input
- **WHEN** a mesh containing triangles, quads and a hexagon is provided
- **THEN** the engine accepts it and treats all faces as part of a single surface

#### Scenario: Named attribute round-trip
- **WHEN** a host supplies a per-vertex scalar attribute named "density"
- **THEN** later stages can look up that attribute by the name "density"

### Requirement: Input validation
Before processing, the engine SHALL validate the input and report each problem with its kind, count and the indices involved. Kinds: out-of-range indices, degenerate faces, non-finite coordinates, non-manifold edges, non-manifold vertices, inconsistent face orientation and disconnected components.

#### Scenario: Non-finite coordinate is rejected
- **WHEN** a vertex position contains NaN or infinity
- **THEN** validation fails with an error naming that vertex index, and no processing stage runs

#### Scenario: Non-manifold edge is reported
- **WHEN** an edge is shared by three faces
- **THEN** validation reports a non-manifold edge with the indices of its two vertices

### Requirement: Recoverable defect repair
The engine SHALL offer opt-in repair of recoverable defects: merging coincident vertices within a tolerance, removing degenerate faces, unifying face orientation per component and splitting non-manifold vertices. Each repair action SHALL be listed in the diagnostics.

#### Scenario: Coincident vertices merged when enabled
- **WHEN** repair is enabled and two vertices lie closer together than the merge tolerance
- **THEN** they are merged into one vertex and the diagnostics record the merge

#### Scenario: Repair disabled leaves input untouched
- **WHEN** repair is disabled and the input contains degenerate faces
- **THEN** the engine reports them and processes the mesh without modifying its topology

### Requirement: Surfaces with boundaries and multiple components
The engine SHALL process meshes that have open boundaries and meshes with several disconnected components. Each component SHALL be retopologised independently within a single run.

#### Scenario: Open boundary is preserved
- **WHEN** a mesh with an open boundary loop is processed
- **THEN** the output keeps an open boundary loop that follows the input boundary

#### Scenario: Disconnected components
- **WHEN** a mesh consisting of two disconnected spheres is processed
- **THEN** the output contains two disconnected components, one for each sphere

### Requirement: Mesh file I/O
The engine SHALL read and write Wavefront OBJ and PLY (ASCII and binary), including polygon faces of arbitrary arity. Reading SHALL report the file path and line or offset of any parse error.

#### Scenario: OBJ round-trip preserves geometry
- **WHEN** a mesh is written to OBJ and read back
- **THEN** vertex positions match within float precision and face connectivity is identical

#### Scenario: Malformed file
- **WHEN** an OBJ file contains a face that references a missing vertex
- **THEN** reading fails with an error giving the file path and line number

### Requirement: Scale independence
Processing results SHALL NOT depend on the absolute scale or position of the input. Size parameters SHALL be interpreted either in the input's units or relative to the mesh's bounding-box diagonal, and every such parameter SHALL state which.

#### Scenario: Uniformly scaled input
- **WHEN** the same mesh is processed at scale 1 and at scale 100 with relative size parameters
- **THEN** both outputs have the same face count and connectivity, and their positions differ only by the scale factor within tolerance
