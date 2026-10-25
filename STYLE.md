# C Coding Style — Mirva

> Work in progress (actively changing)

## Principle

Everything that is not forbidden is allowed.

## Architecture and Modules

- Scale the architecture modularly.
- Emphasize composition and interfaces rather than hierarchical structures.
- Avoid hierarchy: deeply nested structures and long dependency chains between modules.
- Everything is separated into directories.
- A module is a directory; each module gets its own nested directory. A module encapsulates a specific aspect of functionality.
- One file — one entity.
- The consumer sees a module through its header (see "Headers").

### Encapsulation

- Through opaque types.
- Public and private headers; the private one gets the `_internal` suffix in the file name.

## Naming

- Types — PascalCase.
- Procedures — snake_case.
- Struct fields — snake_case.
- `enum` members — PascalCase.
- Constants — `SCREAMING_SNAKE_CASE`.
- Macros — `SCREAMING_SNAKE_CASE`; function-like macros that must look consistent with functions — snake_case.
- Avoid abbreviations and acronyms in names.
- Exception — generally accepted abbreviations, e.g. `AABB` (bounding box).
- Abbreviations are allowed for local code fragments, strictly controlled and only when truly necessary: when it improves readability and simplifies writing computations.
- Lifecycle operations — as a suffix to the type name (see "Object Lifecycle").

### Prefixes

**By structure or by file.** The prefix must correspond to the module — the API unit that the consumer sees, i.e. the header file. In most cases the module name matches the name of the main exported structure, and the difference between "by structure" and "by file" disappears: `widget.h` exports `struct widget` and functions `widget_*`.

The difference becomes significant only when the module and the type do not match. Rules for these cases:

#### A module exports several types

The prefix stays at the module level, the type name is the second segment:

```c
net_conn_create(), net_conn_destroy()
net_packet_alloc(), net_packet_free()
```

Alternative — make the type name compound (`struct net_conn`), then the function prefix naturally matches the type. Both options are correct; the important thing is not to mix them in one module.

#### A type spans several files

The prefix is tied to the type; files are merely a division of the implementation:

```text
widget.c        → widget_create, widget_init
widget_render.c → widget_render, widget_draw
widget_event.c  → widget_on_event, widget_dispatch
```

The consumer perceives this as a single `widget_*` API, regardless of the number of `.c` files.

#### Utility module without a dominant type

Module prefix, without binding to a structure:

```text
str_dup, str_split, str_join
hash_compute, hash_compare
```

#### Restrictions

Do not use generic names (`list_`, `node_`, `item_`, `context_`) as a prefix without module qualification. In C's flat namespace this is a direct path to collisions at link time. Either `sched_list_*` or a unique type name `sched_list`.

## Formatting

- Indent — 4 spaces.
- The opening `{` is never moved to a new line, including in functions.

### Declarations

- The `*` in a pointer declaration is attached to the variable name: `int *ptr;`.
- Multiple variable declarations separated by commas are forbidden.

## Code Structure

### Nesting

- Maximum nesting depth is 3 levels.
- Level 1 is the function body. Each nested block (`if` / `for` / `while` / `switch`) adds +1.
- Consequence: no more than two nested blocks are allowed inside a function.
- If the limit is exceeded, resolve it via invariants or by extracting logic into separate procedures.
- Invariants — a way to write code so that nesting is lower.

### goto

- `goto` is strictly forbidden unless the goal is optimization.

## Comments

- No comments.
- Documentation of the public API in public headers is allowed.
- Exceptions — only where the code would be misleading without a comment:
  - unexpected logic behavior: non-obvious behavior, workarounds (compiler bug, platform or hardware specifics);
  - places that, on a plain reading, seem unclear, strange, or extraordinary;
  - notes about the need to change code, including future changes. Such notes live in the code as long as there is a need for them; the recommended format is `TODO:` / `FIXME:` / `XXX:`, but it is not mandatory.

## Object Lifecycle

### Definition

**`init` / `deinit`** — functions for initializing and deinitializing an object whose storage is provided by the calling code. Storage for the structure itself is neither allocated nor freed. The object's internal resources (buffers, descriptors, locks, registrations) are acquired in `init` and released in `deinit`.

**`create` / `destroy`** — functions that own the structure's storage. `create` allocates storage, initializes the object, and returns a pointer to it. `destroy` deinitializes the object and frees the storage. They are implemented as `malloc` + `init` and `deinit` + `free`.

### Lifecycle Function Names

The operation (`create`, `destroy`, `init`, `deinit`) is placed as a **suffix** to the type name: `widget_create`, `widget_destroy`, `widget_init`, `widget_deinit`. This provides:

- grouping of all functions of a type under a common prefix (`widget_*`) in C's flat namespace;
- predictable lookup and autocompletion;
- compatibility with the practice of large projects (Git: `commit_list_insert`; wlroots: `wlr_backend_create`).

The `alloc` / `free` suffixes are also allowed as memory lifecycle functions, but only if they do nothing except allocate and free memory. These are more like service functions for internal use rather than the external API.

### Selection Rules

1. Opaque type → `create`/`destroy` are required: the calling code does not know the structure size and cannot provide storage.
2. Non-opaque type → `init`/`deinit` are sufficient. `create`/`destroy` are optional.
3. Both usage scenarios → both pairs; `create`/`destroy` are defined in terms of `init`/`deinit` without duplicating logic.
4. No internal resources requiring release → `deinit` is not defined.
5. No initialization invariants → `init` is not defined; aggregate initialization is used.
6. Memory allocation without initialization (pools, arenas) → a separate function with a name reflecting the operation (e.g. `pool_alloc`).

### Guarantees

- The pairs are independent: having one does not imply the need for the other.
- The function name matches the operation: `create` allocates storage, `init` does not.
- `destroy` and `deinit` handle `NULL` correctly.
- `create` includes the `init` operations, but is not applicable to an existing object.

## Types

- Use `stdint.h` when exact types are important and high fault tolerance is required.

## Headers

- A module's interface is a header declaring its types and functions.
- Always use include guards.
- `#pragma once` is not used: the target is `-pedantic` C.

## Preprocessor

- `const` / `enum` / `static inline` instead of macros — when needed; not forbidden.

## Standard and Build

- Explicitly specify the C version.
- `-pedantic` if the project does not plan to deviate from pure C.
