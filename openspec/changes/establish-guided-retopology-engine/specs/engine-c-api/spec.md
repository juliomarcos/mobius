# Spec Delta

## Purpose

Gives host applications (Blender first, later others) a stable, language-neutral way to embed the engine and run any of its operations: they pass meshes and guidance in, and get results, intermediates, progress and reports out, without depending on C++ ABI details.

## ADDED Requirements

### Requirement: C linkage and opaque handles
The public API SHALL use C linkage only, with plain C types and opaque handles. No C++ types, exceptions or STL containers SHALL cross the API boundary.

#### Scenario: Consumed from a C compiler
- **WHEN** the public header is compiled as C99
- **THEN** it compiles without errors or warnings

#### Scenario: Exception does not escape
- **WHEN** an internal error throws inside the engine during an API call
- **THEN** the call returns an error status and the exception does not propagate into the host

### Requirement: API and ABI versioning
The library SHALL expose its semantic version and an API version number. A host SHALL be able to check compatibility before calling anything else. Incompatible changes SHALL increment the major version.

#### Scenario: Version check
- **WHEN** a host built against API major version 1 loads a library reporting major version 2
- **THEN** the compatibility check reports the versions as incompatible

### Requirement: Error reporting
Every fallible API call SHALL return a status code. A thread-safe call SHALL return a human-readable message for the last error on the calling thread.

#### Scenario: Invalid argument
- **WHEN** a host passes a null mesh handle to a run call
- **THEN** the call returns an invalid-argument status and the message names the parameter

### Requirement: Data exchange without hidden ownership
Hosts SHALL pass mesh data and guidance as caller-owned buffers. The engine SHALL copy what it needs before returning. Results SHALL be returned in engine-owned objects that the host frees with matching release calls.

#### Scenario: Host buffers reusable after call
- **WHEN** a host frees its input buffers immediately after submitting a run
- **THEN** the run completes correctly

#### Scenario: Result release
- **WHEN** a host releases a result handle
- **THEN** all memory associated with it is freed, and later use of the handle returns an invalid-handle status rather than crashing

### Requirement: Operation-generic API
Runs SHALL be started by operation name. Operation-specific settings SHALL be passed as named parameters, and results SHALL be retrieved as named outputs (for example "mesh", "field" or "singularities"). Adding a new operation SHALL NOT require new API functions or break existing hosts.

#### Scenario: Discovering an operation's interface
- **WHEN** a host asks the API to describe the "retopology" operation
- **THEN** it receives the operation's parameters with types and defaults, its accepted guidance layer kinds, its objectives and its named outputs

#### Scenario: Unknown output name
- **WHEN** a host requests an output name that the finished run's operation does not produce
- **THEN** the call returns an invalid-argument status and the message lists the available outputs

### Requirement: Asynchronous runs
The API SHALL support starting a run asynchronously, polling its status and progress, cancelling it, and retrieving its report and named outputs (final results and requested intermediates such as analysis, symmetry map or field) once it finishes.

#### Scenario: Non-blocking host
- **WHEN** a host starts an asynchronous run
- **THEN** the start call returns immediately, and the host can poll progress until the run completes

### Requirement: Full feature access
Everything available through the engine's pipeline SHALL be reachable through the C API: operation listing and description, validation, repair options, all guidance kinds, symmetry analysis, objectives and overrides, strategy selection, compute-backend listing and selection, stop-after-stage previews, evaluation and preset load/save.

#### Scenario: Command-line tool uses only the C API
- **WHEN** the command-line tool is built
- **THEN** it links only against the shared library's public C API and still exposes every option
