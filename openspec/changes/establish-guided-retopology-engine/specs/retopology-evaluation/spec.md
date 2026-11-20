# Spec Delta

## Purpose

Measures output quality objectively, so that strategies, objectives and parameter changes can be compared and regressions caught, both by developers and by artists inspecting a result.

## ADDED Requirements

### Requirement: Topology metrics
The evaluation SHALL report: face count by arity, quad ratio, vertex valence histogram, count of irregular vertices (valence other than 4 for interior vertices), and singularity count compared with the field's singularity count.

#### Scenario: Regular grid
- **WHEN** a regular quad grid on a plane is evaluated
- **THEN** quad ratio is 1.0 and no interior irregular vertices are reported

### Requirement: Element quality metrics
The evaluation SHALL report per-face and summary statistics (min, mean, max, 5th percentile) for quad angle deviation from 90 degrees, aspect ratio, and edge-length deviation from the target density.

#### Scenario: Skewed quads detected
- **WHEN** a mesh containing a quad with a 30 degree corner is evaluated
- **THEN** the maximum angle deviation reported is at least 60 degrees and that face is identified

### Requirement: Surface deviation metrics
The evaluation SHALL report the one-sided and symmetric Hausdorff distance and the mean distance between output and source, both absolute and relative to the bounding-box diagonal.

#### Scenario: Identical meshes
- **WHEN** a mesh is evaluated against itself
- **THEN** all deviation metrics are zero within floating-point tolerance

### Requirement: Feature and silhouette preservation metrics
The evaluation SHALL report, for each preserved feature curve, the fraction of its length represented by output edges within tolerance. It SHALL also report a silhouette deviation score computed from a fixed set of view directions.

#### Scenario: Missing feature detected
- **WHEN** an output mesh that has lost one of a cube's twelve feature curves is evaluated
- **THEN** that curve's preservation fraction is below the tolerance threshold and the curve is listed

### Requirement: Guidance adherence metrics
The evaluation SHALL report the mean angular deviation from each direction stroke, and the ratio of achieved to requested density for each density layer and region.

#### Scenario: Stroke adherence reported
- **WHEN** a run with two direction strokes is evaluated
- **THEN** the report lists both strokes with their mean angular deviation

### Requirement: Symmetry metrics
When a symmetry plane is declared, the evaluation SHALL report the fraction of output faces that have a mirror counterpart, and the maximum positional deviation between mirrored vertex pairs after reflection.

#### Scenario: Fully symmetric result
- **WHEN** a retopologised exactly symmetric mesh is evaluated with its plane
- **THEN** the mirrored-face fraction is 1.0

### Requirement: Regression corpus
The repository SHALL include a reference corpus of meshes (closed organic, open-boundary, hard-surface, high-genus, noisy scan-like, partially symmetric and large-sculpt cases) with guidance and expected metric bounds. The regression suite SHALL fail when any metric falls outside its bounds.

#### Scenario: Quality regression caught
- **WHEN** a code change raises the irregular vertex count on a corpus mesh above its recorded bound
- **THEN** the regression suite fails and names the mesh and metric

### Requirement: Per-element quality export
The evaluation SHALL expose per-face quality values to hosts, so they can be shown as an overlay on the output mesh.

#### Scenario: Quality overlay data
- **WHEN** a host requests per-face quality for an output mesh
- **THEN** it receives one quality value per output face, in face order
