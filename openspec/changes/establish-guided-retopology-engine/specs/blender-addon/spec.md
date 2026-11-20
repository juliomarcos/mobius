# Spec Delta

## Purpose

Makes Blender the first interactive environment for the engine: artists author guidance directly on their models, preview intermediate results, run operations (retopology first) without blocking the UI, and get results as ordinary Blender objects.

## ADDED Requirements

### Requirement: Operation panels and shared tools
The addon SHALL present one panel per engine operation. Retopology is the only operation in this release. Guidance authoring tools (strokes, curves, density paint, feature marks, regions, symmetry) SHALL be shared across operations and store their data once per object, and an operation's panel SHALL show only the guidance kinds that operation accepts.

#### Scenario: Retopology panel
- **WHEN** a user opens the addon sidebar with this release installed
- **THEN** it shows a shared guidance section and a Retopology panel, and no panels for unreleased operations

#### Scenario: Guidance stored once
- **WHEN** a user draws a surface curve and later runs two different operations that both accept curves
- **THEN** both runs use the same stored curve, without the user redrawing it

### Requirement: Supported Blender versions and packaging
The addon SHALL install as a Blender extension package that bundles the engine shared library. It SHALL support Blender 4.2 LTS and every later release up to the latest at time of release, on Windows x64.

#### Scenario: Install from package
- **WHEN** a user installs the addon package in a supported Blender version
- **THEN** the addon enables without errors and its panel appears in the 3D viewport sidebar

#### Scenario: Incompatible library
- **WHEN** the bundled library reports an incompatible API version
- **THEN** the addon shows an error naming both versions and disables its run operators

### Requirement: Run retopology on the active object
The addon SHALL retopologise the active mesh object, using its evaluated geometry (modifiers applied) by default. The source object SHALL never be modified.

#### Scenario: Result object created
- **WHEN** a user runs retopology on a sculpt
- **THEN** a new mesh object is created alongside the source with the source's world transform, and the source mesh is unchanged

### Requirement: Non-blocking execution
Runs SHALL execute without freezing the Blender UI, show stage progress and allow cancellation from the UI.

#### Scenario: Cancel from UI
- **WHEN** a user presses cancel during a run
- **THEN** the run stops, no result object is created and Blender stays responsive throughout

### Requirement: Edge-flow stroke tool
The addon SHALL provide a tool for drawing direction strokes on the source surface in the viewport. Strokes SHALL be stored with the object, and be visible, selectable, editable in weight and deletable.

#### Scenario: Strokes persist
- **WHEN** a user draws strokes, saves the .blend file and reopens it
- **THEN** the strokes are still present and are applied on the next run

### Requirement: Density painting
The addon SHALL let users paint the density map on the source mesh with Blender's attribute painting, using a named attribute that the addon creates and maps to the engine's density layer.

#### Scenario: Painted density applied
- **WHEN** a user paints higher density on the face of a head and runs retopology
- **THEN** the output is visibly denser in the painted area, and the run report shows that density layer as active

### Requirement: Feature, region and boundary authoring
The addon SHALL let users mark or exclude feature edges, assign faces to named regions with overrides (including deformation zones and "keep existing topology"), and draw surface boundary curves. Each SHALL map to the matching engine guidance layer.

#### Scenario: Region override applied
- **WHEN** a user assigns faces to a region with a density multiplier of 2 and runs retopology
- **THEN** the engine receives a region constraint covering exactly those faces with multiplier 2

### Requirement: Objective and parameter controls
The Retopology panel SHALL expose the objective choice, per-weight overrides, target face count or edge length, output mode, strategy choice and preset save/load.

#### Scenario: Objective choice reaches the engine
- **WHEN** a user selects the hard-surface objective and runs
- **THEN** the run report shown in Blender names the hard-surface objective

### Requirement: Symmetry controls and preview
The shared guidance section SHALL let users enable symmetry, set the plane (by axis or by picking a point and normal), adjust the tolerance, force regions to be symmetric or asymmetric, and preview the symmetry map as a colour overlay on the source mesh before running.

#### Scenario: Symmetry map preview
- **WHEN** a user enables symmetry on a character holding a sword and requests the preview
- **THEN** the sword arm is highlighted as asymmetric, the body as symmetric, and the symmetric coverage percentage is shown

### Requirement: Field and quality preview
The addon SHALL be able to draw the computed cross field and singularities as a viewport overlay before meshing, and show per-face quality as an overlay on the result.

#### Scenario: Field preview
- **WHEN** a user requests a field preview
- **THEN** the field directions and singularity markers are drawn over the source mesh, and no result object is created

### Requirement: Run report display
After each run, the addon SHALL show the key metrics (face count, quad ratio, irregular vertices, maximum deviation) and any warnings or conflicts. The full report SHALL be viewable as text.

#### Scenario: Conflict warning surfaced
- **WHEN** a run reports a guidance conflict
- **THEN** the addon shows a warning in its panel that describes the conflict
