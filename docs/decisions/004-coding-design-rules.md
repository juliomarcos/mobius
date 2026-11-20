# 004. Coding design rules

- **Date:** 2026-11-20
- **Module:** Coding style part 2 (gate 2.0)
- **Status:** accepted

## Question
Decision 003 fixed how Mobius code looks: names, files, formatting and the basic use of C++ features. It did not fix how Mobius code is designed. Part 2 covered 22 design questions in nine groups: values and types, polymorphism and generic code, algorithms and functions, ownership and state, contracts and errors, data layout, concurrency, names of relationships, and testing. The owner answered all of them on 2026-11-20 in [docs/learning/coding-style-decisions-2.html](../learning/coding-style-decisions-2.html) (published at https://claude.ai/artifact/GGweDr3FRyRRL3zDX37J21). The page drew on Sean Parent (value semantics, no raw loops, no raw pointers, contracts), Stepanov (regular types), Lakos (value semantics, narrow and wide contracts), Mike Acton and Richard Fabian (data-oriented design), the C++ Core Guidelines, and others. Several answers were then refined in chat. Those follow-ups are recorded below.

## Options
Each decision offered two or three options. The table lists them with the recommendation and the owner's choice. Where the owner's choice was refined in chat, the follow-up list below is the final rule.

| # | Decision | Options offered | Recommended | Chosen |
|---|----------|-----------------|-------------|--------|
| 01 | Value semantics as the default | value semantics by default; `shared_ptr` handles for big objects; decide per type | value semantics by default | value semantics by default (note on `indirect`) |
| 02 | Regular types and what `==` means | every value type regular, `==` exact, `approx_equal` separate; regular except equality; `==` with an epsilon | every value type regular, `==` exact | every value type regular, `==` exact, `approx_equal` separate |
| 03 | Heavy objects: may they be copied silently? | fully regular; move-only with `clone()`; copy-on-write | move-only with `clone()` | move-only heavy types with `clone()` |
| 04 | Immutability by default | `const` on everything that does not change; `const` on parameters and members only; no rule | `const` on everything that does not change | `const` on everything that does not change |
| 05 | Runtime polymorphism | variant for closed sets and type-erased values for open sets; virtual interfaces for open sets; virtual interfaces everywhere | variant for closed sets, type-erased values for open sets | same, refined by follow-up 1 |
| 06 | Templates and concepts | concrete by default, templates with two real users; generic from the start; no templates outside maths | concrete by default | concrete by default, templates only with two real users |
| 07 | Raw loops | Parent's rule; ranges pipelines everywhere; plain loops anywhere | Parent's rule | Parent's rule: named algorithms or named loop functions |
| 08 | Member functions or free functions | minimal classes with free algorithms; rich classes; free functions on open structs | minimal classes | minimal classes, algorithms as free functions (note) |
| 09 | Parameters and results | values in and out, settings structs, `*_into` for hot buffers; output parameters everywhere; builders | values in and out | values in and out, settings structs, `*_into` only for hot buffers |
| 10 | Ownership | owners are values, `unique_ptr`, no mutable `shared_ptr`; `shared_ptr` freely; raw owners | owners are values | owners are values, `unique_ptr` when needed, no mutable `shared_ptr` |
| 11 | Views and lifetimes | views are parameters only, `*_view` types for exceptions; views stored anywhere; no views | views are parameters only | views are parameters only, `*_view` types for exceptions |
| 12 | Global and hidden state | no mutable globals with three listed exceptions; singletons for services; no rule | no mutable globals, three exceptions | singletons allowed for services (refined by follow-up 4) |
| 13 | Contracts: who checks preconditions | validate at boundaries, narrow contracts inside; wide contracts everywhere; no checks | validate at boundaries | validate at boundaries, narrow contracts inside |
| 14 | What a failed `MOBIUS_ASSERT` does, and how much is checked | two levels, failure throws; debug-only, failure aborts; always on, failure aborts | two levels, failure throws | two assert levels, failure throws `contract_violation` |
| 15 | Who guards the half-edge mesh invariants | `working_mesh` class with invariant-keeping operators; immutable mesh; open struct | `working_mesh` class | `working_mesh` class, operators keep invariants, `check_invariants` |
| 16 | The error type and warnings | one engine-wide error struct, warnings in stage outputs; per-module error enums, warnings to a logger; message string | one engine-wide error struct | per-module error enums, warnings to a logger (refined by follow-up 4) |
| 17 | Layout of per-element data | attribute arrays indexed by strong index; array of structures; structure of arrays all the way down | attribute arrays | attribute arrays indexed by strong index |
| 18 | Allocation in hot code | no allocation in hot loops, reuse scratch buffers; plain containers; custom allocators | no allocation in hot loops | no allocation in hot loops, reuse scratch buffers |
| 19 | Who may use threads and synchronization | algorithms never touch threads, only `parallel_for` does; locks allowed with review; single-threaded first | algorithms never touch threads | algorithms never touch threads, only `parallel_for` does |
| 20 | How half-edge navigation is spelled | mesh queries taking index values; cursor handles; direct array access | mesh queries taking index values | mesh queries taking index values |
| 21 | Naming functions by what they do to data | nouns for queries, verbs for changes; `get_`/`set_`; verbs everywhere | nouns for queries, verbs for changes | nouns for queries, verbs for changes, expensive work named as verbs |
| 22 | What kinds of tests Mobius has | layered tests with an in-house generator; examples and corpus only; layered with an external library | layered, in-house generator | layered: examples, properties, determinism, golden, in-house generator |

## Recommendation
Claude recommended the options in the "Recommended" column. The owner agreed with 20 of 22 and differed on these two:
- **12:** the owner allowed singletons for services. Claude had recommended no mutable globals, with three listed exceptions, because every input of a function should be visible in its signature and the determinism promise (design D9) depends on that. Singletons for services stay allowed. Follow-up 4 adds that the logger serves developer diagnostics only and that no service feeds results or the run report.
- **16:** the owner chose per-module error enums with warnings to a logger. Claude had recommended one engine-wide error struct with warnings returned in stage outputs, so that there is one mapping to the C API and warnings are part of a stage's result. The follow-up decisions keep the per-module enums but return artist-visible warnings in stage outputs, so the report stays deterministic.

Answer 05 was agreed with the recommendation, then refined in chat because the owner pointed out that Mobius has no open sets and that a variant is as large as its biggest alternative. Follow-up 1 records the result. Claude had also recommended a small in-house `indirect<T>` instead of a vendored library (follow-up 3). The owner chose to vendor the reference implementation.

## Decision
The owner's answers, verbatim:

```
Coding style decisions, part 2
01 Value semantics as the default: Value semantics by default (note: You should also consider things like https://www.sandordargo.com/blog/2026/08/12/cpp26-indirect and some other value type polymorhpism)
02 Regular types and what == means: Every value type regular, == exact, approx_equal separate
03 Heavy objects: may they be copied silently?: Move-only heavy types with clone()
04 Immutability by default: const on everything that does not change
05 Runtime polymorphism: Variant for closed sets, type-erased values for open sets (note: Why would I have open sets? I control the entire code stack. But at the same time, variants are not always appropriate as they are sized as the biggest of the options. Isn't there an alternative?)
06 Templates and concepts: Concrete by default, templates only with two real users
07 Raw loops: Parent's rule: named algorithms or named loop functions
08 Member functions or free functions: Minimal classes, algorithms as free functions (note: classes should only be used when structs can't maintain their invariants. but at the same time, I don't want to forbidden member functions when sensible or when perf is not critical.)
09 Parameters and results: Values in and out, settings structs, *_into only for hot buffers
10 Ownership: Owners are values, unique_ptr when needed, no mutable shared_ptr
11 Views and lifetimes: Views are parameters only, *_view types for exceptions
12 Global and hidden state: Singletons allowed for services
13 Contracts: who checks preconditions: Validate at boundaries, narrow contracts inside
14 What a failed MOBIUS_ASSERT does, and how much is checked: Two assert levels, failure throws contract_violation
15 Who guards the half-edge mesh invariants: working_mesh class, operators keep invariants, check_invariants
16 The error type and warnings: Per-module error enums, warnings to a logger
17 Layout of per-element data: Attribute arrays indexed by strong index
18 Allocation in hot code: No allocation in hot loops, reuse scratch buffers
19 Who may use threads and synchronization: Algorithms never touch threads, only parallel_for does
20 How half-edge navigation is spelled: Mesh queries taking index values
21 Naming functions by what they do to data: Nouns for queries, verbs for changes, expensive work named as verbs
22 What kinds of tests Mobius has: Layered: examples, properties, determinism, golden, in-house generator
```

Notes recorded under the answers:
- **08:** classes are used only when a struct cannot keep its invariants. Member functions are not forbidden where they read naturally or where the code is not performance-critical. Algorithms stay free functions by default.
- **16:** the per-module error enums (`io_error`, `mesh_error`, and so on) are each mapped to C API status codes in one place, at the C API boundary. A module that calls another converts that module's error into its own enum at the call.

Follow-up decisions made in chat (2026-11-20):
1. **Polymorphism (refines 05).** There are no open sets inside an operation, because Mobius controls the whole stack. The form is picked by how the data is stored. One chosen value (for example a strategy with its settings): `std::variant`, since size does not matter for one object. Many mixed values (for example guidance layers): one container per kind, so nothing is padded to the largest alternative (data-oriented, existence-based storage). A variant whose alternatives differ a lot in size: put the large alternative behind `indirect<T>` so the variant stays small. No inheritance hierarchies and no `polymorphic<T>`. Dispatch happens once per stage or batch, never per element in a hot loop.
2. **Operations are a closed list too.** The owner was asked whether operations (retopology now, UV and normals later) should register as descriptor values, so that adding one never edits shared code, as `openspec/config.yaml` and the D9 risk note required. The owner answered: "Why not? It'll be a new release". Claude read this as: adding an operation may edit one shared list, because each new operation ships in a new release. After Claude explained both designs (a registry of descriptor values versus a closed list, including that self-registration in the static `MobiusCore` library would need an explicit list anyway), the owner confirmed: "yes, closed list is what I meant". So operations, layer kinds, strategies and compute backends are closed sets (`std::variant` or `enum class`), and names map to them through a fixed name table, so the C API and the addon still select by name.
3. **`indirect<T>` (from the note on 01).** C++26 `std::indirect` and `std::polymorphic` come from P3019 (https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3019r14.pdf). Microsoft's STL has not implemented them yet (https://github.com/microsoft/STL/issues/5302 is open). The owner chose to vendor jbcoe/value_types (https://github.com/jbcoe/value_types, MIT licence, the paper authors' reference implementation, with allocator support), pinned, in `third_party/`. Mobius uses only its `indirect`. It is exposed as `mobius::base::indirect<T>`, an alias that switches to `std::indirect` behind a feature-test guard (`__cpp_lib_indirect`) in `features.h` once MSVC ships it. Claude had recommended a small in-house `indirect<T>` instead.
4. **Logger (reconciles 12 and 16 with determinism).** A singleton logger is allowed for developer diagnostics only (trace, timings, debug output). It never feeds results or the run report. Warnings that the artist must see (solver hit its iteration cap, weak symmetry, an ignored stroke, conflicts, repairs) still travel inside each stage's output struct into the run report, in stage order, so reports stay byte-identical (design D9, D12; spec processing-pipeline "run report"). This partly reverses the 16 answer, and the owner chose it.
5. **Strong types library (the owner's request in chat).** Mobius gets a lightweight in-house strong-types library modelled on rollbear/strong_type (https://github.com/rollbear/strong_type, BSL-1.0; used as a design reference, not a dependency). Shape: `base::strong_type<T, Tag, Modifiers...>`, where each modifier is a struct with a nested CRTP `template <typename T> class modifier`, as in rollbear. It starts with only the modifiers Mobius needs: `regular` (copy, assign, `==`), `ordered` (`<=>`), `hashable`, `formattable` (`std::format`), `arithmetic`, `difference` and `affine_point<D>` (for quantities and offsets), and `default_value<V>` (default construction gives V). The underlying value is read with `value_of(x)`. Every strong type has `static_assert(sizeof(S) == sizeof(T))`. The index types become `template <typename Tag> using index = base::strong_type<std::uint32_t, Tag, base::regular, base::ordered, base::hashable, base::formattable, base::default_value<0xFFFFFFFFu>>;` with `vertex_index`, `face_index`, `edge_index` and `halfedge_index` as before (decision 003 already fixed the 0xFFFFFFFF invalid default). Other strong types (for example angles in radians) are added only where a mix-up is likely.
6. **New `mobius.base` module.** A new lowest folder `engine/core/base` (module `mobius.base`, namespace `mobius::base`, depending only on `std` and the vendored value_types) holds the strong types, `indirect`, `contract_violation` and the generic attribute arrays (`base::attribute<Index, T>`, with `vertex_attribute<T>` and similar as aliases in `mobius.mesh`). This changes the D1 folder list by one entry.

## Consequences
- [docs/decisions/004-coding-design-rules.md](004-coding-design-rules.md): this record.
- [docs/coding-style.md](../coding-style.md): new section 6, "Design rules", with one subsection per rule group. Later sections renumbered (Comments 7, Tests 8, C API 9, Python 10). The scope paragraph now cites decisions 003 and 004. "Strong index types" uses `base::strong_type` and `value_of(v)`. Section 3 mentions `engine/core/base` and `mobius.base`. The Errors subsection describes per-module enums mapped at the C API. The Tests section lists the test kinds from decision 22.
- [openspec/changes/establish-guided-retopology-engine/design.md](../../openspec/changes/establish-guided-retopology-engine/design.md):
  - D1 adds `base/` to the shared core folders.
  - D2 adds jbcoe/value_types and notes that strong types are in-house.
  - D9 makes operations, strategies, layer kinds and compute backends closed sets selected through name tables, drops stage interfaces, and limits the logger to developer diagnostics.
  - D10 adds per-module error enums and `contract_violation`.
  - D15 adds `__cpp_lib_indirect` to the `features.h` guards.
  - The Risks list changes the dummy-operation item.
- [openspec/config.yaml](../../openspec/config.yaml): the project context says new operations are added to the shared operation list (one entry, shipped in a new release) instead of registering.
- [openspec/changes/establish-guided-retopology-engine/tasks.md](../../openspec/changes/establish-guided-retopology-engine/tasks.md): tasks 2.1, 2.6, 6.1, 11.1 and 11.5 are reworded. Two tasks are added at the end of group 2: 2.10 (vendor value_types and expose `base::indirect`) and 2.11 (implement `mobius.base`).
- Claude's review then reworded tasks 2.5 (two assertion levels, `contract_violation`) and 11.4 (dummy operation added by one list entry), the design's Goals bullet and the "interface" wording in D6, D16 and the Risks list, and the guidance-constraints spec sentence on operation layer kinds.
- Tasks that now depend on this record: 2.1 (error types and diagnostics), 2.5 (assertion levels and `contract_violation`), 2.6 (backend name table), 2.8 (maths types use `base::strong_type` where a mix-up is likely), 2.10 and 2.11 (new), 3.1 and 3.4 (attribute arrays, `working_mesh` and its invariant-keeping operators), 6.1 (layer kinds as a closed set), 11.1, 11.4 and 11.5 (operation list and strategy name tables), and 12.3 (names map to the closed sets).
- Decisions 12 and 16 are partly reversed by follow-up 4. The rulebook states the final rule.