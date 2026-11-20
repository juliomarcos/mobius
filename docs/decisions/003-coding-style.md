# 003. Coding style

- **Date:** 2026-11-18
- **Module:** Coding style (gate 2.0)
- **Status:** accepted

## Question
Before any engine code is written, Mobius needs one fixed coding style: how things are named, how files are named and organised, how code is formatted, how C++ features are used, how errors are reported, how code is documented and how tests are laid out. The compiler (MSVC only, `/std:c++latest` with C++23 as the guaranteed baseline), the tools (Sharpmake, doctest, simdjson), the folder layout and the C API conventions were already fixed. Twenty decisions remained. The owner answered all of them on 2026-11-18 in `docs/learning/coding-style-decisions.html`.

## Options
Each decision offered two or three options. The table lists them with the recommendation and the owner's choice.

| # | Decision | Options offered | Recommended | Chosen |
|---|----------|-----------------|-------------|--------|
| 01 | Type names | PascalCase, snake_case | PascalCase | snake_case |
| 02 | Function and variable names | snake_case, camelCase | snake_case | snake_case |
| 03 | Private data members | trailing underscore, `m_` prefix, no marker | `count_` | `m_count` |
| 04 | Constants and enumerators | PascalCase, UPPER_CASE, kPascalCase | PascalCase | PascalCase |
| 05 | File names and extensions | snake_case `.hpp`, PascalCase `.hpp`, snake_case `.h` | snake_case `.hpp` | snake_case `.h` and `.cpp` |
| 06 | Automatic formatting | clang-format with format on save, manual by convention | clang-format | clang-format |
| 07 | Brace style | K&R, Allman | K&R | K&R |
| 08 | Maximum line length | 100, 120, 80 | 100 | 120 |
| 09 | const and pointer placement | west const, east const | west const | west const |
| 10 | Include order | own, Mobius, third-party, standard; or standard first | own header first | own header first |
| 11 | Strong index types | distinct types, plain `uint32_t` | distinct types | distinct types |
| 12 | Scalar precision | double inside with float at I/O; float storage with double solves; templated scalar type | double inside | double inside |
| 13 | Use of auto | when the type is obvious or unspeakable; almost always; avoid | when obvious | almost always |
| 14 | Error handling | as designed in D10; exceptions everywhere; error codes everywhere | as designed | as designed |
| 15 | `[[nodiscard]]` | every function returning `std::expected` or a computed value; only `std::expected` returns; not used | every such function | every such function |
| 16 | Classes and structs | struct for plain data and class for invariants; always class with accessors | struct/class split | struct/class split |
| 17 | Headers or modules | headers with `#pragma once`, C++20 named modules | headers | C++20 named modules |
| 18 | Namespaces | `mobius::` plus one per core folder with `detail`; flat `mobius::` | per folder | per folder |
| 19 | Comments | `///` brief plus `//` for why; comments only where needed | `///` plus `//` | `///` plus `//` |
| 20 | Tests | one test file per source file, mirrored path, SUBCASE; grouped by feature | mirrored | mirrored |

## Recommendation
Claude recommended the options in the "Recommended" column. The owner agreed with 14 of 20 and differed on these six:
- **01:** snake_case type names instead of PascalCase.
- **03:** `m_` prefix instead of a trailing underscore.
- **05:** `.h` instead of `.hpp`.
- **08:** 120 columns instead of 100.
- **13:** almost always `auto` instead of `auto` when the type is obvious.
- **17:** C++20 named modules instead of headers. Claude had recommended headers because module support on this toolchain was unverified.

## Decision
The owner's answers, verbatim:

1. Type names: snake_case (std style)
2. Function and variable names: snake_case
3. Private data members: Prefix: m_count
4. Constants and enumerators: PascalCase
5. File names and extensions: snake_case with .h and .cpp
6. Automatic formatting: clang-format with a committed .clang-format and format-on-save
7. Brace style: K&R: brace on the same line
8. Maximum line length: 120 columns
9. const and pointer placement: West const: const vec3d& p, T* ptr
10. Include order: Own header, Mobius, third-party, standard library
11. Strong index types: Distinct types (named in snake_case per 01: vertex_index, face_index, edge_index)
12. Scalar precision: double inside, float only at I/O boundaries
13. Use of auto: Almost always auto
14. Error handling (design D10): Confirm as designed (std::expected<T, error> for anticipated failures; exceptions only for programmer errors and allocation failure; C API catches everything and returns status codes)
15. [[nodiscard]]: every function returning std::expected or a computed value
16. Classes and structs: struct for plain data, class when invariants must be protected
17. Headers or modules: C++20 named modules
18. Namespaces: mobius:: plus one sub-namespace per core folder, detail for internals
19. Comments: /// brief on every public declaration, // for why inside implementations
20. Tests: one test file per source file, mirrored path, descriptive names with SUBCASE

Follow-up decisions made in chat after the answers:
- Modules must first be proven by task 1.10 of the OpenSpec change: a proof build with Sharpmake and MSVC with a `.ixx` interface, `import std;`, simdjson and doctest wrapped through a global module fragment, and an IntelliSense check. If the proof fails, the fallback is ordinary headers (`.h` with `#pragma once`) and this record gets a superseding decision. IntelliSense module support is labelled experimental in Visual Studio 2026, and Sharpmake has no built-in MSVC module option, so the MSBuild settings must be injected.
- Module interface units use `.ixx` (the MSVC default). Implementation units use `.cpp`. `.h` is used for the C API header (`mobius.h`) and for small macro-only headers. Macros such as feature tests and `MOBIUS_ASSERT` cannot be exported from modules, so these headers are included from a global module fragment. Third-party headers are wrapped the same way.
- The owner explicitly approved clang-format.
- Function return types (raised during review of the rulebook): trailing return types, `auto f() -> T`, for every C++ function including `-> void`. Constructors, destructors and conversion operators are exempt. Lambdas omit the return type unless it cannot be deduced. The C API keeps the leading C form.

## Consequences
- The rulebook is [docs/coding-style.md](../coding-style.md). The editor settings are in [.editorconfig](../../.editorconfig) and the formatter settings are in [.clang-format](../../.clang-format).
- Design D15 (compiler and language standard) in the OpenSpec change `establish-guided-retopology-engine` now covers modules, `.ixx` files and `import std;`, and its risks list the IntelliSense and Sharpmake caveats. Task 1.10 is the module proof build, and no module-based code is written before it passes.
- Names in the design were updated to snake_case: `compute_backend`, `operation`, `field_strategy`, `meshing_strategy`, `run_context`, `vec<T,N>`, `mat<T,R,C>`, `frame` and `error`.
- Design D10 (errors) is confirmed unchanged. D1 (layout), D3 (mesh internals), D9 (threading), D11 (C API) and D16 (maths) keep their content and only take the new names.
- If task 1.10 fails, a new decision supersedes the module part of this record and section 3 of the rulebook switches to headers.
