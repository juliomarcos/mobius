# Mobius coding style

## 1. Scope and status

These rules apply to all C++ in `engine/` and `tests/`. The C API and the Python addon have their own short sections (8 and 9). The rules come from [decision 003](decisions/003-coding-style.md). Do not change them without a new decision.

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

Vertex, face and edge indices are distinct types. A plain integer lets a face index be passed where a vertex index is expected, which compiles and gives wrong geometry.

```cpp
template <typename Tag>
struct index {
    std::uint32_t value = 0xFFFFFFFF;
};
using vertex_index = index<struct vertex_tag>;
using face_index = index<struct face_tag>;
using edge_index = index<struct edge_tag>;

[[nodiscard]] auto vertex_position(const working_mesh& mesh, vertex_index v) -> vec3d;

auto f = first_face(mesh);
auto p = vertex_position(mesh, f);  // compile error: face_index is not vertex_index
```

The wrapper has the size of `uint32_t`, so the structure-of-arrays storage keeps its layout.

### Scalar precision

Use `double` inside the engine. Use `float` only at I/O boundaries, where a file format, the C API or the Blender buffers require it. Convert once at the boundary.

### Errors

Errors follow design D10:

- Anticipated failures (a missing file, a bad mesh, a failed solve) return `std::expected<T, error>`.
- Exceptions are only for programmer errors and allocation failure.
- The C API catches everything and returns status codes.

```cpp
[[nodiscard]] auto load_mesh(const std::filesystem::path& path) -> std::expected<working_mesh, error>;

[[nodiscard]] auto solve_from_file(const std::filesystem::path& path) -> std::expected<cross_field, error> {
    auto mesh = load_mesh(path);
    if (!mesh) {
        return std::unexpected{mesh.error()};
    }
    return solve_cross_field(*mesh);
}
```

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

## 6. Comments and documentation

- Put a `///` brief on every exported or public declaration. One short sentence is enough.
- Use `//` inside implementations to explain why the code does something, not what it does.
- Do not leave commented-out code. Version control keeps the history.

```cpp
/// Returns the cotangent weight of the angle at c.
[[nodiscard]] auto cotangent_weight(const vec3d& a, const vec3d& b, const vec3d& c) -> double;

// Skip degenerate triangles: their cross product is zero and the weight would be infinite.
```

## 7. Tests

`tests/unit` mirrors `engine/core`. One test file per source file, with the same relative path and a `_test.cpp` suffix.

- `engine/core/math/vec.ixx` is tested in `tests/unit/math/vec_test.cpp`.
- Test names follow the pattern `<folder>/<file>: <behaviour>`, matching the source path under `engine/core`.
- Use `SUBCASE` for variants of the same behaviour.
- Regression tests live in `tests/regression`.

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

## 8. C API

The C API is fixed by the spec.

- C99, with the header `engine/capi/include/mobius/mobius.h`.
- Every public name has the `mobius_` prefix and is snake_case.
- Handles are opaque pointers such as `mobius_mesh*`.
- Functions return a status code and write results through out parameters.
- No C++ types cross the boundary. Exceptions never cross it either.

```c
mobius_status mobius_mesh_load(const char* path, mobius_mesh** out);
```

## 9. Python (Blender addon)

Follow PEP 8 with 4 spaces of indent. Use 120 columns to match the C++ code. Names are snake_case, and classes are PascalCase as PEP 8 asks. The addon calls the engine only through the C API.