# Mobius coding style

## 1. Scope and status

These rules apply to all C++ in `engine/` and `tests/`. The C API and the Python addon have their own short sections (9 and 10). The rules come from [decision 003](decisions/003-coding-style.md) and [decision 004](decisions/004-coding-design-rules.md). Do not change them without a new decision.

C++20 named modules are the chosen way to organise code. They are pending proof by task 1.10 of the OpenSpec change (a proof build with Sharpmake and MSVC). Until that task passes, do not write module-based engine code. If the proof fails, the fallback is ordinary headers (`.h` with `#pragma once`), a superseding decision is recorded, and section 3 lists the header layout. Every other rule stays the same in both cases.

## 2. Naming

| Thing | Style | Example |
|-------|-------|---------|
| Types (class, struct, enum, alias) | snake_case | `working_mesh`, `cross_field` |
| Functions and methods | snake_case | `load_mesh`, `principal_curvatures` |
| Variables and parameters | snake_case | `vertex_count` |
| Private data members | `m_` prefix | `m_positions` |
| Public struct members | no prefix | `k1`, `dir1` |
| Constants | PascalCase | `constexpr int MaxIterations = 500;` |
| Enumerators | PascalCase | `enum class symmetry_kind { Mirror, None };` |
| Macros | `MOBIUS_UPPER_CASE` | `MOBIUS_ASSERT` |
| Template parameters | PascalCase | `T`, `Scalar`, `Tag` |
| Namespaces | snake_case | `mobius::math` |
| Modules | `mobius.<folder>` | `mobius.math` |
| Module partitions | `mobius.<folder>:<file>` | `mobius.math:vec` |

Types and variables share snake_case, so a name must say which one it is by its role. Name a variable for what it does in the scope, and never give it the same identifier as its type.

```cpp
working_mesh mesh;       // role: the mesh being edited
vertex_index v;          // role: one vertex
working_mesh working_mesh;  // wrong: the variable hides its own type
```

Each core folder has one primary interface that re-exports its partitions. For example `mobius.math` re-exports `mobius.math:vec`, `mobius.math:mat` and `mobius.math:frame`.

## 3. Files and modules

File names are snake_case. Extensions have fixed roles:

- `.ixx`: module interface unit (primary interface or partition).
- `.cpp`: module implementation unit, executable entry point, or test file.
- `.h`: the C API header `mobius.h`, and small macro-only headers. Macros such as feature tests and `MOBIUS_ASSERT` cannot be exported from a module, so they live in a `.h` file that is included from a global module fragment.

Rules:

- One module partition per `.ixx` file. The file name matches the partition name: `engine/core/math/vec.ixx` declares `export module mobius.math:vec;`.
- Folder, module and namespace follow each other: `engine/core/math` is module `mobius.math` and namespace `mobius::math`. Internals go in `mobius::math::detail`.
- `engine/core/base` is the lowest folder: module `mobius.base`, namespace `mobius::base`. It depends only on `std` and the vendored value_types, and holds the strong types, `indirect`, `contract_violation` and the generic attribute arrays. Every other module may import it.
- Use `import std;` instead of standard library headers.
- Wrap macro headers and third-party headers (simdjson, doctest) in the global module fragment, before the module declaration.
- Where `#include` is still used (the global module fragment, test files that need it), order the includes as: own header, Mobius, third-party, standard library. Leave a blank line between groups and sort each group alphabetically. clang-format does this.

```cpp
// engine/core/io/mesh_loader.ixx
module;
#include "mobius/config/features.h"
#include <simdjson.h>

export module mobius.io:mesh_loader;

import std;
import mobius.mesh;
```

Header fallback layout (only if task 1.10 fails): `engine/core/math/vec.h` with `#pragma once`, `engine/core/math/vec.cpp`, the same snake_case names, and the same namespaces.

## 4. Formatting

clang-format is the authority. The `.clang-format` file at the repository root holds the settings. Format on save, or press Ctrl+K, Ctrl+D. Never hand-format against it. `.editorconfig` covers the editor basics for all file types.

The result looks like this:

- K&R braces: the opening brace is on the same line.
- 4 spaces of indent, no tabs.
- 120 columns at most.
- West const: `const vec3d& p`, `const T* ptr`.
- The `*` and `&` attach to the type: `T* ptr`, `const T& ref`.

```cpp
[[nodiscard]] auto cotangent_weight(const vec3d& a, const vec3d& b, const vec3d& c) -> double {
    const auto u = a - c;
    const auto v = b - c;
    if (length(cross(u, v)) < Epsilon) {
        return 0.0;
    }
    return dot(u, v) / length(cross(u, v));
}
```

## 5. Language usage

### auto

Use `auto` almost always. Where a type must be named, use the initialiser style `auto x = type{...};`.

```cpp
auto mesh = working_mesh{};
auto curvature = principal_curvatures(mesh, v);
auto weights = std::vector<double>(mesh.vertex_count(), 0.0);
```

Write the type out only when `auto` would hide something the reader needs and no initialiser can name it. This is rare.

### Function return types

Every C++ function and member function uses a trailing return type, including functions that return nothing. Constructors, destructors and conversion operators have no return type, so the rule does not apply to them. Lambdas leave the return type out unless the compiler cannot deduce it.

```cpp
[[nodiscard]] auto principal_curvatures(const working_mesh& mesh, vertex_index v) -> curvature_result;
auto normalize_in_place(vec3d& v) -> void;

auto weight = [](double angle) { return 1.0 / std::tan(angle); };
```

The C API is C99 and keeps the leading form.

### Strong index types

Vertex, face, edge and half-edge indices are distinct types. A plain integer lets a face index be passed where a vertex index is expected, which compiles and gives wrong geometry.

They are built with the in-house `base::strong_type<T, Tag, Modifiers...>` (modelled on rollbear/strong_type). Each modifier adds one capability. Read the underlying value with `value_of(x)`.

```cpp
template <typename Tag>
using index = base::strong_type<std::uint32_t, Tag, base::regular, base::ordered, base::hashable, base::formattable,
                                base::default_value<0xFFFFFFFFu>>;
using vertex_index = index<struct vertex_tag>;
using face_index = index<struct face_tag>;
using edge_index = index<struct edge_tag>;
using halfedge_index = index<struct halfedge_tag>;

[[nodiscard]] auto vertex_position(const working_mesh& mesh, vertex_index v) -> vec3d;

auto f = first_face(mesh);
auto p = vertex_position(mesh, f);  // compile error: face_index is not vertex_index
auto raw = value_of(f);             // the std::uint32_t
```

Every strong type has `static_assert(sizeof(S) == sizeof(T))`, so the structure-of-arrays storage keeps its layout. Add other strong types (for example angles in radians) only where a mix-up is likely. The modifiers are `regular`, `ordered`, `hashable`, `formattable`, `arithmetic`, `difference`, `affine_point<D>` and `default_value<V>`.

### Scalar precision

Use `double` inside the engine. Use `float` only at I/O boundaries, where a file format, the C API or the Blender buffers require it. Convert once at the boundary.

### Errors

Errors follow design D10:

- Anticipated failures (a missing file, a bad mesh, a failed solve) return `std::expected<T, E>`, where `E` is the module's own error enum (`io_error`, `mesh_error`, and so on).
- Exceptions are only for programmer errors and allocation failure.
- A module that calls another module converts that module's error into its own enum at the call.
- The C API catches everything and returns status codes. Each module enum is mapped to a status code in one place, at the C API boundary.

```cpp
[[nodiscard]] auto load_mesh(const std::filesystem::path& path) -> std::expected<source_mesh, io_error>;

[[nodiscard]] auto load_working_mesh(const std::filesystem::path& path) -> std::expected<working_mesh, mesh_error> {
    auto source = io::load_mesh(path);
    if (!source) {
        return std::unexpected{mesh_error::SourceUnreadable};  // io_error converted at the module boundary
    }
    return build_working_mesh(*source);
}
```

See "Errors and warnings" in section 6 for warnings.

### [[nodiscard]]

Mark every function that returns `std::expected` or a computed value with `[[nodiscard]]`. An ignored return value is an ignored failure.

### struct and class

Use `struct` for plain data whose members can vary independently. Use `class` when an invariant must be protected. Declare members in this order: public, protected, private. Private data members take the `m_` prefix.

```cpp
struct curvature_result {
    double k1;
    double k2;
    vec3d dir1;
};

class working_mesh {
public:
    [[nodiscard]] auto vertex_count() const -> std::size_t { return m_positions.size(); }

private:
    std::vector<vec3d> m_positions;
};
```

## 6. Design rules

These rules come from [decision 004](decisions/004-coding-design-rules.md), which gives the reasons. Naming rules from section 2 apply to every example.

### Values and types

Every type is a value: a copy is independent, and `==` is exact and defaulted. Approximate comparison is a separate function, `approx_equal(a, b, tolerance)`. A short list of heavy types (`working_mesh`, sparse matrices, fields, run results) is move-only and has a `clone()` for a deliberate copy. Put `const` on every local that does not change, and `const&` on read-only parameters. A local that is returned is not `const`.

```cpp
struct vec3d {
    double x = 0.0;
    double y = 0.0;
    double z = 0.0;

    [[nodiscard]] auto operator==(const vec3d&) const -> bool = default;
};

[[nodiscard]] auto approx_equal(const vec3d& a, const vec3d& b, double tolerance) -> bool;

class working_mesh {
public:
    working_mesh(working_mesh&&) noexcept = default;
    auto operator=(working_mesh&&) noexcept -> working_mesh& = default;
    working_mesh(const working_mesh&) = delete;
    auto operator=(const working_mesh&) -> working_mesh& = delete;

    /// Returns an independent copy of the mesh.
    [[nodiscard]] auto clone() const -> working_mesh;
};
```

### Polymorphism

Do not write inheritance hierarchies and do not use `polymorphic<T>`. Mobius controls the whole stack, so every set of alternatives is closed. Pick the form by how the data is stored:

- One chosen value (a strategy and its settings): `std::variant`.
- Many mixed values (guidance layers): one container per kind, so nothing is padded to the largest alternative.
- A variant with one large alternative: put that alternative behind `base::indirect<T>` so the variant stays small.

Dispatch once per stage or batch, never per element in a hot loop.

```cpp
using meshing_choice = std::variant<field_aligned_extract_settings, base::indirect<global_param_settings>>;

struct guidance_layers {
    std::vector<direction_stroke> strokes;
    std::vector<density_map> densities;
};
```

### Templates

Write concrete code by default. Add a template only when two real types use it now (`vec<T, N>`, `base::strong_type`, `base::attribute`). Constrain every template parameter with a named concept.

```cpp
template <std::floating_point T, std::size_t N>
struct vec {
    std::array<T, N> data;
};
```

### Loops and algorithms

Use a named standard algorithm, or move the loop into a small function whose name says what it does. A loop that is the whole body of a named function is fine. Range-`for` is fine for "do this for each element". Avoid long `views` pipelines in hot code. A numerical sweep (a Gauss-Seidel pass) stays a loop in its own named function.

```cpp
[[nodiscard]] auto face_areas(const working_mesh& mesh) -> face_attribute<double> {
    auto areas = face_attribute<double>(mesh.face_count());
    std::ranges::transform(mesh.faces(), areas.begin(), [&](face_index f) { return face_area(mesh, f); });
    return areas;
}
```

### Functions and parameters

Algorithms are free functions in their module namespace. A class is for a type whose invariants a struct cannot keep. A member function is fine where it reads naturally or where speed does not matter, but it does not replace a free algorithm. Pass cheap types (indices, `vec3d`, `double`) by value, other types by `const&`, and read-only arrays as `std::span<const T>`. Return results by value, or as a struct for several results. A function with more than about four settings takes a `*_settings` struct with defaults, passed with designated initialisers. Write to a caller's buffer only in a function named `*_into`, and only in hot code.

```cpp
struct smooth_settings {
    int iterations = 10;
    double weight = 0.5;
};

[[nodiscard]] auto smooth_field(const working_mesh& mesh, const cross_field& field, const smooth_settings& settings)
    -> cross_field;

auto field = smooth_field(mesh, initial, {.iterations = 20, .weight = 0.5});
```

### Ownership and views

An owner is a value: a member or a container. Use `std::unique_ptr` when heap storage is needed. Use `std::shared_ptr<const T>` only for large immutable data that must be shared. Never use `shared_ptr` to mutable data. `T*` and `T&` never own. Do not write `new` or `delete` outside low-level containers. A view (`std::span`, `std::string_view`, a reference) is a parameter only: never store or return one, except in a type named `*_view` whose `///` brief says what it must not outlive. Objects refer to mesh elements by strong index.

```cpp
/// A read-only window onto backend-owned storage. Must not outlive the backend buffer it was taken from.
struct buffer_view {
    std::span<const double> values;
};
```

### Global state and logging

Singletons are allowed for services such as the logger, a profiler or an allocator. A service never feeds results or the run report, so results stay deterministic. Other mutable globals are not allowed; the C API's thread-local last-error message is the one exception. Name tables (operations, strategies, backends) are read-only constants. Random generators and settings reach code through parameters or the `run_context`. The logger is for developer diagnostics only (trace, timings, debug output). Warnings the artist must see travel in the stage's output struct.

```cpp
log_debug("field solve took {} iterations", iterations);  // developer diagnostics only
```

### Contracts and assertions

Validate at the boundaries (file loading, C API, guidance parsing, the validate/repair stage) and return an error there. Inside the engine, functions have narrow contracts: the caller must satisfy them. State them in the `///` brief as "Requires: ..." and check them with `MOBIUS_ASSERT`. `MOBIUS_ASSERT(cond)` is cheap and on in every build. `MOBIUS_ASSERT_AUDIT(cond)` is expensive and on in Debug and tests only. A failure throws `mobius::base::contract_violation` with the file, line and expression text.

```cpp
/// Returns the position of v. Requires: v is a vertex of mesh.
[[nodiscard]] auto vertex_position(const working_mesh& mesh, vertex_index v) -> vec3d {
    MOBIUS_ASSERT(value_of(v) < mesh.vertex_count());
    return mesh.position(v);
}
```

### Mesh invariants

`working_mesh` is a class with private structure-of-arrays storage. Read it through `const` accessors. Change topology only through the invariant-keeping operators (`split_edge`, `split_face`, `flip_edge`, `collapse_edge`). `check_invariants(mesh)` lists every violation. Run it under `MOBIUS_ASSERT_AUDIT` after every operator, and at every pipeline stage boundary.

```cpp
// Topology operators are members: only they may write the private arrays.
auto working_mesh::flip_edge(edge_index e) -> void {
    // ... keeps twin, next and origin consistent ...
    MOBIUS_ASSERT_AUDIT(check_invariants(*this).empty());
}
```

### Errors and warnings

Each module has its own error enum. The C API maps each enum to a status code in one place. Warnings the artist must see (iteration cap hit, weak symmetry, an ignored stroke, a conflict, a repair) are `diagnostic` values returned in the stage's output struct. The pipeline collects them into the run report in stage order, so reports are byte-identical.

```cpp
struct field_result {
    cross_field field;
    std::vector<diagnostic> diagnostics;
};

[[nodiscard]] auto solve_cross_field(const working_mesh& mesh) -> std::expected<field_result, field_error>;
```

### Data layout and allocation

Bulk per-element data is one attribute array per attribute, indexed by strong index: `base::attribute<Index, T>`, with `vertex_attribute<T>`, `face_attribute<T>` and similar aliases in `mobius.mesh`. A small struct is fine for one element's result (`curvature_result`) and for values always used together (`vec3d`). Do not allocate in per-element or per-iteration loops. Size buffers before the loop, and reuse scratch buffers across iterations.

```cpp
auto curvature = vertex_attribute<double>(mesh.vertex_count());
auto scratch = std::vector<double>(mesh.vertex_count());  // sized once, reused each iteration
for (auto i = 0; i < iterations; ++i) {
    smooth_into(mesh, curvature, scratch);
}
```

### Concurrency

Algorithm code never touches threads or synchronisation primitives. Write algorithms as pure per-chunk functions that read shared `const` data and write only their own chunk of the output. Only `parallel_for`, the cancellation flag and the progress counters use threads, atomics or mutexes. Do not use `std::thread`, `std::async` or the `std::execution` parallel policies in algorithm code. For scattered writes, count first and fill second instead of taking a lock.

```cpp
parallel_for(mesh.vertex_count(), [&](chunk range) {
    for (const auto v : range) {
        weights[v] = cotangent_sum(mesh, v);  // writes only its own element
    }
});
```

### Naming of relationships and side effects

Half-edge navigation is a set of mesh queries that take index values: `mesh.next(h)`, `mesh.twin(h)`, `mesh.origin(h)`, `mesh.face(h)`. Neighbourhoods are ranges: `mesh.outgoing(v)`. Names are nouns without `get_`. Provide helpers for common chains, such as `mesh.target(h)`. Name a query as a noun (`vertex_count`, `area`). Name a change as a verb (`flip_edge`, `normalize`). A non-mutating twin of a verb uses the past participle (`normalized`). A boolean query reads as a statement (`is_boundary`, `has_feature`). Name expensive work as a verb that says how (`solve_cross_field`, `estimate_curvature`, `build_bvh`), so it never looks like a cheap query.

```cpp
for (const auto h : mesh.outgoing(v)) {
    const auto neighbour = mesh.target(h);
    total += edge_length(mesh, h) * weights[neighbour];
}
```

## 7. Comments and documentation

- Put a `///` brief on every exported or public declaration. One short sentence is enough.
- Use `//` inside implementations to explain why the code does something, not what it does.
- Do not leave commented-out code. Version control keeps the history.

```cpp
/// Returns the cotangent weight of the angle at c.
[[nodiscard]] auto cotangent_weight(const vec3d& a, const vec3d& b, const vec3d& c) -> double;

// Skip degenerate triangles: their cross product is zero and the weight would be infinite.
```

## 8. Tests

`tests/unit` mirrors `engine/core`. One test file per source file, with the same relative path and a `_test.cpp` suffix.

- `engine/core/math/vec.ixx` is tested in `tests/unit/math/vec_test.cpp`.
- Test names follow the pattern `<folder>/<file>: <behaviour>`, matching the source path under `engine/core`.
- Use `SUBCASE` for variants of the same behaviour.
- Regression tests live in `tests/regression`.

Mobius has five kinds of test ([decision 004](decisions/004-coding-design-rules.md)):

- **Examples:** every function has example tests (one input, one expected output).
- **Properties:** mesh operators and solvers have property tests. A property is a rule that holds for all inputs, checked on many generated inputs. Use a fixed seed and the in-house generator in the tests. Do not add a property-testing library.
- **Determinism:** every parallel kernel has a test that runs it on 1 and 8 threads and requires identical bytes.
- **Golden:** the regression corpus compares run reports with stored files byte for byte.
- **Regression first:** every bug fix starts with a failing test that reproduces the bug.

```cpp
TEST_CASE("math/vec: cross product is anti-commutative") {
    auto a = vec3d{1.0, 0.0, 0.0};
    auto b = vec3d{0.0, 1.0, 0.0};

    SUBCASE("unit axes") {
        CHECK(cross(a, b) == -cross(b, a));
    }
    SUBCASE("parallel vectors give zero") {
        CHECK(cross(a, a) == vec3d{});
    }
}
```

## 9. C API

The C API is fixed by the spec.

- C99, with the header `engine/capi/include/mobius/mobius.h`.
- Every public name has the `mobius_` prefix and is snake_case.
- Handles are opaque pointers such as `mobius_mesh*`.
- Functions return a status code and write results through out parameters.
- No C++ types cross the boundary. Exceptions never cross it either.

```c
mobius_status mobius_mesh_load(const char* path, mobius_mesh** out);
```

## 10. Python (Blender addon)

Follow PEP 8 with 4 spaces of indent. Use 120 columns to match the C++ code. Names are snake_case, and classes are PascalCase as PEP 8 asks. The addon calls the engine only through the C API.