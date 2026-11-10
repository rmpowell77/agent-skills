---
name: cpp-parameter-passing
description: Guidance for choosing C++ function parameter types (by value, const&, &, &&, raw pointers, std::optional, std::span, std::string_view, unique_ptr, shared_ptr) so the signature says what the function does. Use this whenever writing, reviewing, refactoring, or generating C++ functions, methods, or constructors, even if the user never mentions parameter passing, and whenever someone asks "should this take a reference, a pointer, or a smart pointer?" Based on the talk "How To Argue(ment)" by Richard Powell.
---

# C++ parameter passing

A function signature is its thesis statement. The **callee** declares the parameter types, and the **caller** supplies arguments that satisfy them. Each parameter type makes a promise about what the callee will do with the object and what the caller can rely on afterward. Pick the type that says what you mean, and a reader (human or agent) can understand the contract without reading the body.

Use "parameter" for what the signature declares and "argument" for what the call site passes.

## Workflow for each parameter

1. **Decide the role first**, then pick the spelling. Ask: does the callee want a copy of its own, a look at the caller's object, a change to the caller's object, ownership, or shared ownership? Is the object optional? Is it a run of several?
2. **Look up the role** in the table below.
3. **Check the project's language standard** (CMake, Bazel, compile flags) before using `std::optional` and `std::string_view` (C++17), `std::span` (C++20), or `std::optional<T&>` (C++26). Match the conventions already in the codebase when they conflict with this guidance, and mention the conflict instead of rewriting existing APIs unprompted.
4. **Before finishing**, run the self-check at the bottom.

## Roles and spellings

| Role | Spelling | Meaning to the caller |
|---|---|---|
| Snapshot (callee owns a copy) | `Foo` | I'll copy (or move) one in for you |
| Observe | `Foo const&` | Your object is only read during the call |
| Modify | `Foo&` | Your object will be changed by the call |
| Optional observe | `Foo const*` | Pass `nullptr` if there's nothing |
| Optional modify | `Foo*` | Pass `nullptr` if there's nothing |
| Optional snapshot | `std::optional<Foo>` | Copy in a value, or nothing |
| Observe a contiguous run | `std::span<Foo const>` (strings: `std::string_view`) | Zero or more Foos, read-only |
| Modify a contiguous run | `std::span<Foo>` | Zero or more Foos, changed in place |
| Take ownership | `std::unique_ptr<Foo>` by value | Caller gives up the object |
| Share ownership | `std::shared_ptr<Foo>` by value | The callee will keep it alive past the call |
| Optionally share ownership | `std::shared_ptr<Foo> const&` | The callee may copy it to extend lifetime |

Everything not in this table is rare, redundant, or a mistake. `references/verdicts.md` explains each pruned form; read it when you meet an unusual signature in review or when someone argues for one.

## Defaults

**Pass by value when you can, by `const&` when you must.** Small, cheap-to-copy types (roughly two or three machine words, trivially copyable) go by value: the callee gets a stable snapshot, and `const&` would force an indirection and make the compiler re-read memory after any opaque call. Use `const&` for types that are expensive to copy or can't be copied. This applies to scalars too: `int const&` and `Foo* const&` are strictly worse than by value.

**Return results; don't use output parameters.** Return values (with guaranteed copy elision, structured bindings, `std::optional`, and `std::expected` where available) cover what out-parameters used to do. Never use a pointer as an output parameter.

**Spell `Foo&&` as `Foo`.** A by-value parameter already says "I take this, and you won't use it afterward," and the caller chooses between copying and `std::move`. Use `Foo&&` only for:
- conditional consumption, where the function reports whether it consumed the argument (`bool try_push(Foo&&)`). Nothing moves until the callee decides, so the caller's object is intact on failure. The result is the only reliable signal, because a moved-from object is valid but unspecified.
- an expensive move, or a `const&` / `&&` overload pair in library-style code.

In a template, `T&&` with deduced `T` is a forwarding reference, a different thing. This skill doesn't cover forwarding; leave existing ones alone.

## Things not to write, and why

- **`Foo const` as a by-value parameter in a declaration.** The compiler treats it as identical to `Foo`, and it tells the caller nothing. In a definition it only restricts the body, and it quietly stops `return x;` and `std::move(x)` from moving. The same applies to `Foo* const` and `std::span<Foo> const`, where const is shallow and doesn't make the elements const.
- **`Foo const&&`.** It matches nothing useful. The one legitimate use is `= delete` on an overload to reject temporaries.
- **`Foo* const&`, `Foo*&&`.** A pointer is a scalar, so a reference to it adds indirection and nothing else.
- **`Foo*&` and `Foo const*&`.** They reseat the caller's pointer and interfere with lifetime management. Return the new pointer instead.
- **`std::span<Foo> const&`, `std::span<Foo>&&`.** A span is a two-word trivially-copyable handle. Pass it by value. A `&` form is only justified for a parser-style cursor, and returning the remaining span is usually clearer.
- **`std::optional<Foo> const&`.** If the caller holds a plain `Foo`, binding builds a temporary optional and copies the Foo. `Foo const*` costs nothing.
- **`std::optional<Foo const>`** in any form. The inner const constrains the callee and splits callers into two non-interoperating types. Inner `const` matters only when the wrapper refers to an object that isn't yours (pointers, spans, `shared_ptr`).
- **`std::optional<Foo>&`.** To change whether there's a value, return one. If the callee only edits the contents, take `Foo*` or `Foo&`.
- **`std::optional<Foo>*`.** It has three states (null, empty, engaged). Take `Foo const*` or `Foo*`.
- **`std::unique_ptr<Foo> const&`.** It forces every caller to hold a `unique_ptr`. Take `Foo const*` or `Foo const&` to observe. Calling `.get()` is fine for observing.
- **`std::unique_ptr<Foo>&&`.** Same as `Foo&&`: take it by value.
- **`std::shared_ptr<Foo>&&`.** Take it by value.
- **`std::unique_ptr<Foo>&`, `std::shared_ptr<Foo>&`.** These reseat the caller's smart pointer. They're rare; prefer returning the new one.
- **`std::span<Foo&>`, `std::optional<Foo&>` below C++26.** Neither compiles before the language and library support them (`span<Foo&>` never does). Use `span<Foo>` or a pointer.
- **`(Foo*, size_t)` or `Foo items[N]` for a range.** The latter decays to `Foo*`, and the size is ignored. Use `std::span`.

## Smart pointers

- Raw pointers and references are fine for **observing**; smart pointers express **lifetime**. Use `unique_ptr` to transfer ownership and `shared_ptr` only when the callee will actually share it (storing it in a member, handing it to another thread).
- The common constructor shape is `explicit Runner(std::shared_ptr<Foo> r) : r_{std::move(r)} {}`.
- Don't take a `shared_ptr<Foo>` by value to merely use the object; that's a refcount bump for nothing. Take `Foo&` or `Foo const&`.
- `shared_ptr<Foo> const&` can still be reseated and reset by *other* code during the call. If the callee calls into unknown code and then keeps using the object, take a local copy first.

## Optional, honestly

For optional **parameters**, `Foo const*` / `Foo*` are still the default. `std::optional<Foo&>` (C++26) is a pointer-equivalent with a nicer interface, and it doesn't add a capability. Use it only when the project targets C++26 and the toolchain supports it, and mention that the call `f(x)` hides that `x` may change. Moving from an optional doesn't empty it.

**Choosing between `std::optional<Foo>` and a pointer.** Ask what the callee needs from the caller's object:
- A *value* (a snapshot it can keep or modify privately)? If `Foo` is cheap to copy, take `std::optional<Foo>` by value (`std::optional<int>`, a small struct). A small struct that arrives as `const std::optional<Foo>&` should usually become this, not a pointer.
- To *observe or modify the caller's actual object* (identity matters, or `Foo` is expensive or non-copyable)? Take `Foo const*` or `Foo*`, because `optional<Foo>` would copy the object in and the callee couldn't see changes or modify the original.

When both would work, prefer the one the call sites already use, and say which you chose and why.

## Lifetime: avoid surprising sharing

A reference or pointer parameter is valid for the call. If the callee stores it anywhere that outlives the call (a member, a global, a callback, a thread, a coroutine frame), the signature is lying. Change the type: take by value, or `shared_ptr` and store a `weak_ptr` if observation is enough. `std::span` and `std::string_view` have the same hazard, and C++20 coroutines make it worse, because reference parameters dangle once the coroutine suspends. Take coroutine parameters by value.

## Out of scope: callables and templates

This skill doesn't cover callable parameters (`std::function`, `std::move_only_function`, `std::function_ref`, function pointers, lambdas as templates) or forwarding references. For a callable that the function **stores**, take it by value and `std::move` it into the member, as with any other stored parameter. For a callable that's only **called during the call**, follow the project's existing convention rather than inventing one. Say in a sentence that the choice is outside this skill's guidance.

## Reviewing existing code

For each signature, report: what the parameter type promises, whether the body honors it (does a `const&` parameter get stored? does a `Foo&` get only read?), and a concrete suggested change when there's a mismatch. Flag; don't rewrite public APIs unless asked. Keep the output short: one line per issue, with the parameter, the problem, and the replacement.

**When you can only see declarations** (a header, a diff without bodies), you can't tell whether a parameter is stored or only used. State the assumption in the suggestion, for example "assuming `log` doesn't keep the pointer after returning, `Widget const&` is enough; if it does, keep `shared_ptr<Widget>` by value." Don't present a lifetime-dependent change as certain.

## Self-check before finishing a function

- Could any parameter be by value instead of `const&`?
- Is every `T*` genuinely optional? Is every array a `span`?
- Does any parameter get stored past the call? If so, is the type honest about that?
- Is there an output parameter that should be a return value?
- Did I write `const` on a by-value parameter in a declaration, or `&&` where by value would do?
- Does the signature follow what the codebase already does?
