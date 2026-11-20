# Spec Delta

## Purpose

Extracts the geometric properties of the source surface (curvature, principal directions, features and local scale) that drive field alignment, density and feature preservation in later stages.

## ADDED Requirements

### Requirement: Curvature estimation
The engine SHALL estimate mean curvature, Gaussian curvature, the two principal curvatures and the two principal directions at every vertex. A configurable neighbourhood radius SHALL control robustness to noise.

#### Scenario: Sphere curvature
- **WHEN** curvature is estimated on a finely tessellated sphere of radius r
- **THEN** both principal curvatures at every vertex are within 5% of 1/r

#### Scenario: Cylinder principal directions
- **WHEN** curvature is estimated on a finely tessellated cylinder away from its caps
- **THEN** the minimum-curvature direction at each vertex is within 5 degrees of the cylinder axis

### Requirement: Principal direction confidence
Each principal-direction estimate SHALL carry a confidence value that falls toward zero where the two principal curvatures are nearly equal (umbilic or flat regions). Later stages SHALL use the confidence to weight how strongly a direction is followed.

#### Scenario: Flat region has low confidence
- **WHEN** principal directions are estimated on a planar region
- **THEN** their confidence is below the configured umbilic threshold

### Requirement: Feature-edge detection
The engine SHALL detect sharp feature edges from the dihedral angle using a configurable threshold, and SHALL group detected edges into feature curves. It SHALL also detect feature corners where three or more curves meet, or where one curve turns sharply.

#### Scenario: Cube features
- **WHEN** feature detection runs on a cube with a 30 degree threshold
- **THEN** 12 feature curves and 8 feature corners are reported

#### Scenario: Smooth surface has no features
- **WHEN** feature detection runs on a smooth sphere with a 30 degree threshold
- **THEN** no feature edges are reported

### Requirement: Noise-robust feature filtering
Feature curves shorter than a configurable minimum length SHALL be discarded, so that scan noise and sculpt texture do not produce spurious features.

#### Scenario: Noisy scan
- **WHEN** feature detection runs on a smooth surface with added high-frequency noise and a minimum curve length above the noise wavelength
- **THEN** no feature curves are reported for the noise

### Requirement: Local scale estimation
The engine SHALL compute a per-vertex local length scale from curvature, used as the default basis for adaptive density. It SHALL be clamped between configurable minimum and maximum edge lengths.

#### Scenario: Adaptive scale follows curvature
- **WHEN** local scale is computed on a mesh with a tightly curved region and a flat region
- **THEN** the local scale in the curved region is smaller than in the flat region and stays within the configured clamps
