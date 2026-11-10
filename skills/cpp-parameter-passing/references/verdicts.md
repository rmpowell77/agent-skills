# Verdicts for every parameter form

Read this when reviewing an unusual signature, or when someone argues for a form the main skill prunes. Sections: Foo, pointers, optional, span, unique_ptr and shared_ptr, why the pruning works.

## Foo (a user-defined type)

Starting grid: `Foo`, `Foo&`, `Foo&&`, `Foo*`, `Foo*&`, `Foo*&&`, plus the const-qualified versions. `const` can't attach to a reference, so `Foo& const` is ill-formed. A `const` after `*` is legal and adds `Foo* const`, `Foo* const&`, `Foo* const&&`, `Foo const* const`, `Foo const* const&`, `Foo const* const&&`. The full set is 18.

| Form | Verdict | Reason |
|---|---|---|
| `Foo` | Keep | Snapshot; callee owns a copy |
| `Foo const` | Collapse into `Foo` | Same function; const only restricts the callee's body |
| `Foo&` | Keep | Modify the caller's object |
| `Foo const&` | Keep | Observe the caller's object |
| `Foo&&` | Spell as `Foo` | Same role as by value, except for conditional consumption and expensive moves |
| `Foo const&&` | Prune | Matches nothing useful; legitimate only as a deleted overload |
| `Foo*`, `Foo const*` | Keep | Optional modify / optional observe |
| `Foo* const`, `Foo const* const` | Collapse | Same as above, top-level const |
| `Foo* const&`, `Foo const* const&` | Prune | A pointer is a scalar; reference adds indirection and aliasing |
| `Foo*&&`, `Foo const*&&`, `Foo* const&&`, `Foo const* const&&` | Prune | Moving a pointer is a copy |
| `Foo*&`, `Foo const*&` | Prune | Reseats the caller's pointer; return it instead |

Survivors: `Foo`, `Foo const&`, `Foo&`, `Foo const*`, `Foo*`, and `Foo&&` as a conditional-consume spelling.

These pruned types do appear as *deduced* types in generic code (a `vector<Foo*>` `push_back` parameter is `Foo* const&`). That is fine; this guidance is about what people write by hand.

## Why "const" on a by-value parameter doesn't help

Top-level const is stripped from the function's type. `void f(Foo)` and `void f(Foo const)` are the same function. In the definition, the const parameter can't be moved from (`std::move` on a const object yields `const&&`, which selects the copy constructor) and `return p;` copies instead of moving. A const reference to the *referent* (`Foo const&`, `Foo const*`) is part of the type and matters.

## Optional

| Form | Verdict | Reason |
|---|---|---|
| `optional<Foo>` | Keep, for cheap-to-copy Foo | Optional snapshot |
| `optional<Foo> const&` | Prune | A plain `Foo` argument builds a temporary and copies |
| `optional<Foo>&` | Prune | Return the new value instead |
| `optional<Foo>&&` | Spell as by value | Same as `Foo&&` |
| `optional<Foo>*` (and `*&`, `*&&`, `const*`) | Prune | Three states; take `Foo*` |
| `optional<Foo const>` and every reference form | Prune | Inner const constrains the callee and fragments callers; `optional<Foo const>&` can't assign |

`optional` is a vocabulary type: a value with an extra "absent" state, passed like the value it wraps. Pointer is an optional *reference*; `optional<Foo>` is an optional *value*.

## Span

A `span<Foo>` behaves like `Foo*` plus a size: two words, trivially copyable, shallow const.

| Form | Verdict | Reason |
|---|---|---|
| `span<Foo>` | Keep | A run of Foos, modified in place; the callee may narrow its own copy of the view |
| `span<Foo const>` | Keep | A run of Foos, read only; accepts const containers and temporaries |
| `span<Foo> const`, `span<Foo const> const` | Collapse | Same function; shallow const, does not make elements const |
| `span<Foo>&`, `span<Foo const>&` | Prune | Reseating; a cursor is the only real use |
| `span<Foo> const&`, `span<Foo const> const&` | Prune | Indirection on a cheap handle |
| `span<Foo>&&`, `span<Foo const>&&` | Prune | Moving a trivially-copyable handle is a copy |
| `span<Foo>*` | Prune | An empty span already means "none" |
| `span<Foo&>` | Ill-formed | A reference isn't an object type |

Which arguments bind: `span<Foo>` accepts non-const lvalue containers, arrays, and spans of Foo. It rejects const containers (would drop const) and temporaries (a non-const view of a temporary is almost certainly a bug). `span<Foo const>` accepts all of these. Storing either past the call dangles in the same way a pointer would.

`string_view` is `span<char const>` with string operations. It isn't guaranteed null-terminated, which matters when passing to C APIs. Pass by value.

## unique_ptr and shared_ptr

| Form | Verdict | Reason |
|---|---|---|
| `unique_ptr<Foo>` | Keep | Transfer ownership |
| `unique_ptr<Foo const>` | Keep | Transfer ownership of an object the receiver may only read |
| `unique_ptr<Foo> const&` | Prune; take `Foo const*` or `Foo const&` | Burdens callers with a `unique_ptr` for no benefit |
| `unique_ptr<Foo>&` | Rare | Reseat; prefer returning |
| `unique_ptr<Foo>&&` | Spell as by value | Same as `Foo&&` |
| `shared_ptr<Foo>` | Keep | Share ownership |
| `shared_ptr<Foo const>` | Keep | Share a read-only view |
| `shared_ptr<Foo> const&` | Keep | Optionally share ownership |
| `shared_ptr<Foo>&` | Rare | Reseat the caller's pointer |
| `shared_ptr<Foo>&&` | Prune | A callee that shares lifetime but requires the caller to give it up doesn't make sense; take by value |

A `const&` to a smart pointer lets the callee copy it and reseat or reset *the copy*. The caller's pointer is unaffected. Only the non-const `&` form lets the callee change the caller's pointer.

## Why the pruning works

Each form is judged by what it tells the caller. A form is pruned when it is the same function as another (collapse), when it adds indirection to a scalar or cheap handle, when it matches no useful argument, when another spelling says the same thing more plainly, or when it reseats the caller's variable (return the new value instead). Think of each parameter type as a filter on what a call site can pass: `Foo&` accepts only non-const lvalues, `Foo const&` accepts everything, `Foo&&` accepts rvalues.
