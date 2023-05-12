---
title: Types
---


Icicle's has a principled type system, meaning that every expression in
the language is both able to be inferred, and has a single most general
type.

In practice, this means that type annotations are _never_ required.

Icicle support basic types, such as `Int`, `Double`, `String`, and `Bool`,
as well as more exotic types, such as `Unit`, `Time`, `Option`, and
`Either`.

It is also possible to create your own tuple and record types, as well
as custom types.

The Icicle type system also includes modalities, which are covered in
the next section of this guide.
