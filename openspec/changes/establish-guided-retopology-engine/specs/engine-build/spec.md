# Spec Delta

## Purpose

Defines how developers generate, build and test the Mobius engine from source, and which artifacts a build produces, so the standalone core can be built reproducibly without any host application.

## ADDED Requirements

### Requirement: Project generation from Sharpmake
The repository SHALL generate its IDE solution and projects from Sharpmake scripts committed to the repository. Generated solution and project files SHALL NOT be committed.

#### Scenario: Fresh checkout generates a solution
- **WHEN** a developer on Windows with Visual Studio 2026 and the .NET SDK runs the documented generate command on a fresh checkout
- **THEN** a Visual Studio 2026 solution is produced that contains the core library, shared library, command-line tool and test projects

#### Scenario: Generated files are not tracked
- **WHEN** project generation completes
- **THEN** every generated solution and project file is excluded from version control

### Requirement: Compiler toolchain
MSVC from Visual Studio 2026, toolset 14.52 or later, SHALL be the only supported compiler. Building with an older MSVC toolset SHALL fail at the start with a message naming the required and found versions.

#### Scenario: Old toolset rejected
- **WHEN** a developer builds with MSVC toolset 14.44
- **THEN** the build stops before compiling engine sources, with a message stating that 14.52 or later is required and 14.44 was found

### Requirement: Language standard and C++23 baseline
All engine code SHALL compile with `/std:c++latest`, and SHALL also compile and pass its tests with `/std:c++23`. Any language or library feature newer than C++23 SHALL be guarded by its feature-test macro, with a C++23 fallback.

#### Scenario: Baseline build passes
- **WHEN** the Baseline configuration (Release with `/std:c++23`) is built and tested
- **THEN** it compiles without errors, and every unit and regression test passes

#### Scenario: C++26 feature without guard
- **WHEN** engine code uses a post-C++23 feature without its feature-test macro guard and a fallback
- **THEN** the Baseline build fails

### Requirement: Build configurations
The build SHALL provide Debug, Release and Baseline configurations for the x64 platform. Release SHALL enable compiler optimisation, Debug SHALL enable assertions and debug symbols, and Baseline SHALL equal Release but compile with `/std:c++23`.

#### Scenario: Release build is optimised
- **WHEN** the Release configuration is built
- **THEN** all engine targets compile with optimisation enabled and without debug assertions

#### Scenario: Debug build checks invariants
- **WHEN** the Debug configuration is built and an internal invariant is violated at runtime
- **THEN** the violation is reported by an assertion rather than continuing silently

### Requirement: Produced artifacts
A full build SHALL produce: a static core library with no host-application dependency, a shared library that exports the public C API, a command-line retopology tool, and a test executable.

#### Scenario: Core library is host-independent
- **WHEN** the core library's link dependencies are inspected
- **THEN** it depends on no Blender, Python or UI library

#### Scenario: Shared library exports only the C API
- **WHEN** the exported symbols of the shared library are listed
- **THEN** only symbols carrying the public C API prefix are exported

### Requirement: Single-command build and test
The repository SHALL provide one documented command that generates the projects, builds a chosen configuration and runs the test suite, returning a non-zero exit code on any failure.

#### Scenario: Failing test fails the command
- **WHEN** any unit or regression test fails during the build-and-test command
- **THEN** the command exits with a non-zero status and names the failing test

### Requirement: Third-party dependency policy
All third-party code SHALL be either vendored in the repository or fetched at a pinned version, and SHALL use a license that permits closed-source commercial distribution.

#### Scenario: Dependency versions are reproducible
- **WHEN** two developers build the same commit
- **THEN** both builds use identical versions of every third-party dependency
