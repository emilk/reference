# Functions

> **<sup>Syntax</sup>**\
> _Function_ :\
> &nbsp;&nbsp; _FunctionQualifiers_ `fn` [IDENTIFIER]&nbsp;[_Generics_]<sup>?</sup>\
> &nbsp;&nbsp; &nbsp;&nbsp; `(` _FunctionParameters_<sup>?</sup> `)`\
> &nbsp;&nbsp; &nbsp;&nbsp; _FunctionReturnType_<sup>?</sup> [_WhereClause_]<sup>?</sup>\
> &nbsp;&nbsp; &nbsp;&nbsp; [_BlockExpression_]
>
> _FunctionQualifiers_ :\
> &nbsp;&nbsp; _AsyncConstQualifiers_<sup>?</sup> `unsafe`<sup>?</sup> (`extern` _Abi_<sup>?</sup>)<sup>?</sup>
>
> _AsyncConstQualifiers_ :\
> &nbsp;&nbsp; `async` | `const`
>
> _Abi_ :\
> &nbsp;&nbsp; [STRING_LITERAL] | [RAW_STRING_LITERAL]
>
> _FunctionParameters_ :\
> &nbsp;&nbsp; _FunctionParam_ (`,` _FunctionParam_)<sup>\*</sup> `,`<sup>?</sup>
>
> _FunctionParam_ :\
> &nbsp;&nbsp; [_Pattern_] `:` [_Type_]
>
> _FunctionReturnType_ :\
> &nbsp;&nbsp; `->` [_Type_]

A _function_ consists of a [block], along with a name and a set of parameters.
Other than a name, all these are optional. Functions are declared with the
keyword `fn`. Functions may declare a set of *input* [*variables*][variables]
as parameters, through which the caller passes arguments into the function, and
the *output* [*type*][type] of the value the function will return to its caller
on completion.

When referred to, a _function_ yields a first-class *value* of the
corresponding zero-sized [*function item type*], which
when called evaluates to a direct call to the function.

For example, this is a simple function:
```rust
fn answer_to_life_the_universe_and_everything() -> i32 {
    return 42;
}
```

As with `let` bindings, function arguments are irrefutable [patterns], so any
pattern that is valid in a let binding is also valid as an argument:

```rust
fn first((value, _): (i32, i32)) -> i32 { value }
```

The block of a function is conceptually wrapped in a block that binds the
argument patterns and then `return`s the value of the function's block. This
means that the tail expression of the block, if evaluated, ends up being
returned to the caller. As usual, an explicit return expression within
the body of the function will short-cut that implicit return, if reached.

For example, the function above behaves as if it was written as:

```rust,ignore
// argument_0 is the actual first argument passed from the caller
let (value, _) = argument_0;
return {
    value
};
```

## Generic functions

A _generic function_ allows one or more _parameterized types_ to appear in its
signature. Each type parameter must be explicitly declared in an
angle-bracket-enclosed and comma-separated list, following the function name.

```rust
// foo is generic over A and B

fn foo<A, B>(x: A, y: B) {
# }
```

Inside the function signature and body, the name of the type parameter can be
used as a type name. [Trait] bounds can be specified for type
parameters to allow methods with that trait to be called on values of that
type. This is specified using the `where` syntax:

```rust
# use std::fmt::Debug;
fn foo<T>(x: T) where T: Debug {
# }
```

When a generic function is referenced, its type is instantiated based on the
context of the reference. For example, calling the `foo` function here:

```rust
use std::fmt::Debug;

fn foo<T>(x: &[T]) where T: Debug {
    // details elided
}

foo(&[1, 2]);
```

will instantiate type parameter `T` with `i32`.

The type parameters can also be explicitly supplied in a trailing [path]
component after the function name. This might be necessary if there is not
sufficient context to determine the type parameters. For example,
`mem::size_of::<u32>() == 4`.

## Extern function qualifier

The `extern` function qualifier allows providing function _definitions_ that can
be called with a particular ABI:

```rust,ignore
extern "ABI" fn foo() { ... }
```

These are often used in combination with [external block] items which provide
function _declarations_ that can be used to call functions without providing
their _definition_:

```rust,ignore
extern "ABI" {
  fn foo(); /* no body */
}
unsafe { foo() }
```

When `"extern" Abi?*` is omitted from `FunctionQualifiers` in function items,
the ABI `"Rust"` is assigned. For example:

```rust
fn foo() {}
```

is equivalent to:

```rust
extern "Rust" fn foo() {}
```

Functions in Rust can be called by foreign code, and using an ABI that
differs from Rust allows, for example, to provide functions that can be
called from other programming languages like C:

```rust
// Declares a function with the "C" ABI
extern "C" fn new_i32() -> i32 { 0 }

// Declares a function with the "stdcall" ABI
# #[cfg(target_arch = "x86_64")]
extern "stdcall" fn new_i32_stdcall() -> i32 { 0 }
```

Just as with [external block], when the `extern` keyword is used and the `"ABI`
is omitted, the ABI used defaults to `"C"`. That is, this:

```rust
extern fn new_i32() -> i32 { 0 }
let fptr: extern fn() -> i32 = new_i32;
```

is equivalent to:

```rust
extern "C" fn new_i32() -> i32 { 0 }
let fptr: extern "C" fn() -> i32 = new_i32;
```

Functions with an ABI that differs from `"Rust"` do not support unwinding in the
exact same way that Rust does. Therefore, unwinding past the end of functions
with such ABIs causes the process to abort.

> **Note**: The LLVM backend of the `rustc` implementation
aborts the process by executing an illegal instruction.

## Const functions

Functions qualified with the `const` keyword are const functions. _Const
functions_  can be called from within [const context]s. When called from a const
context, the function is interpreted by the compiler at compile time. The
interpretation happens in the environment of the compilation target and not the
host. So `usize` is `32` bits if you are compiling against a `32` bit system,
irrelevant of whether you are building on a `64` bit or a `32` bit system.

If a const function is called outside a [const context], it is indistinguishable
from any other function. You can freely do anything with a const function that
you can do with a regular function.

Const functions have various restrictions to make sure that they can be
evaluated at compile-time. It is, for example, not possible to write a random
number generator as a const function. Calling a const function at compile-time
will always yield the same result as calling it at runtime, even when called
multiple times. There's one exception to this rule: if you are doing complex
floating point operations in extreme situations, then you might get (very
slightly) different results. It is advisable to not make array lengths and enum
discriminants depend on floating point computations.

Exhaustive list of permitted structures in const functions:

> **Note**: this list is more restrictive than what you can write in
> regular constants

* Type parameters where the parameters only have any [trait bounds]
  of the following kind:
    * lifetimes
    * `Sized` or [`?Sized`]

    This means that `<T: 'a + ?Sized>`, `<T: 'b + Sized>`, and `<T>`
    are all permitted.

    This rule also applies to type parameters of impl blocks that
    contain const methods

* Arithmetic and comparison operators on integers
* All boolean operators except for `&&` and `||` which are banned since
  they are short-circuiting.
* Any kind of aggregate constructor (array, `struct`, `enum`, tuple, ...)
* Calls to other *safe* const functions (whether by function call or method call)
* Index expressions on arrays and slices
* Field accesses on structs and tuples
* Reading from constants (but not statics, not even taking a reference to a static)
* `&` and `*` (only dereferencing of references, not raw pointers)
* Casts except for raw pointer to integer casts
* `unsafe` blocks and `const unsafe fn` are allowed, but the body/block may only do
  the following unsafe operations:
    * calls to const unsafe functions

## Async functions

Functions may be qualified as async, and this can also be combined with the
`unsafe` qualifier:

```rust,edition2018
async fn regular_example() { }
async unsafe fn unsafe_example() { }
```

Async functions do no work when called: instead, they
capture their arguments into a future. When polled, that future will
execute the function's body.

An async function is roughly equivalent to a function
that returns [`impl Future`] and with an [`async move` block][async-blocks] as
its body:

```rust,edition2018
// Source
async fn example(x: &str) -> usize {
    x.len()
}
```

is roughly equivalent to:

```rust,edition2018
# use std::future::Future;
// Desugared
fn example<'a>(x: &'a str) -> impl Future<Output = usize> + 'a {
    async move { x.len() }
}
```

The actual desugaring is more complex:

- The return type in the desugaring is assumed to capture all lifetime
  parameters from the `async fn` declaration. This can be seen in the
  desugared example above, which explicitly outlives, and hence
  captures, `'a`.
- The [`async move` block][async-blocks] in the body captures all function
  parameters, including those that are unused or bound to a `_`
  pattern. This ensures that function parameters are dropped in the
  same order as they would be if the function were not async, except
  that the drop occurs when the returned future has been fully
  awaited.

For more information on the effect of async, see [`async` blocks][async-blocks].

[async-blocks]: ../expressions/block-expr.md#async-blocks
[`impl Future`]: ../types/impl-trait.md

> **Edition differences**: Async functions are only available beginning with
> Rust 2018.

### Combining `async` and `unsafe`

It is legal to declare a function that is both async and unsafe. The
resulting function is unsafe to call and (like any async function)
returns a future. This future is just an ordinary future and thus an
`unsafe` context is not required to "await" it:

```rust,edition2018
// Returns a future that, when awaited, dereferences `x`.
//
// Soundness condition: `x` must be safe to dereference until
// the resulting future is complete.
async unsafe fn unsafe_example(x: *const i32) -> i32 {
  *x
}

async fn safe_example() {
    // An `unsafe` block is required to invoke the function initially:
    let p = 22;
    let future = unsafe { unsafe_example(&p) };

    // But no `unsafe` block required here. This will
    // read the value of `p`:
    let q = future.await;
}
```

Note that this behavior is a consequence of the desugaring to a
function that returns an `impl Future` -- in this case, the function
we desugar to is an `unsafe` function, but the return value remains
the same.

Unsafe is used on an async function in precisely the same way that it
is used on other functions: it indicates that the function imposes
some additional obligations on its caller to ensure soundness. As in any
other unsafe function, these conditions may extend beyond the initial
call itself -- in the snippet above, for example, the `unsafe_example`
function took a pointer `x` as argument, and then (when awaited)
dereferenced that pointer. This implies that `x` would have to be
valid until the future is finished executing, and it is the callers
responsibility to ensure that.

## Attributes on functions

[Outer attributes][attributes] are allowed on functions. [Inner
attributes][attributes] are allowed directly after the `{` inside its [block].

This example shows an inner attribute on a function. The function will only be
available while running tests.

```
fn test_only() {
    #![test]
}
```

> Note: Except for lints, it is idiomatic to only use outer attributes on
> function items.

The attributes that have meaning on a function are [`cfg`], [`deprecated`],
[`doc`], [`export_name`], [`link_section`], [`no_mangle`], [the lint check
attributes], [`must_use`], [the procedural macro attributes], [the testing
attributes], and [the optimization hint attributes]. Functions also accept
attributes macros.

[IDENTIFIER]: ../identifiers.md
[RAW_STRING_LITERAL]: ../tokens.md#raw-string-literals
[STRING_LITERAL]: ../tokens.md#string-literals
[_BlockExpression_]: ../expressions/block-expr.md
[_Generics_]: generics.md
[_Pattern_]: ../patterns.md
[_Type_]: ../types.md#type-expressions
[_WhereClause_]: generics.md#where-clauses
[const context]: ../const_eval.md#const-context
[external block]: external-blocks.md
[path]: ../paths.md
[block]: ../expressions/block-expr.md
[variables]: ../variables.md
[type]: ../types.md#type-expressions
[*function item type*]: ../types/function-item.md
[Trait]: traits.md
[attributes]: ../attributes.md
[`cfg`]: ../conditional-compilation.md
[the lint check attributes]: ../attributes/diagnostics.md#lint-check-attributes
[the procedural macro attributes]: ../procedural-macros.md
[the testing attributes]: ../attributes/testing.md
[the optimization hint attributes]: ../attributes/codegen.md#optimization-hints
[`deprecated`]: ../attributes/diagnostics.md#the-deprecated-attribute
[`doc`]: ../../rustdoc/the-doc-attribute.html
[`must_use`]: ../attributes/diagnostics.md#the-must_use-attribute
[patterns]: ../patterns.md
[`?Sized`]: ../trait-bounds.md#sized
[trait bounds]: ../trait-bounds.md
[`export_name`]: ../abi.md#the-export_name-attribute
[`link_section`]: ../abi.md#the-link_section-attribute
[`no_mangle`]: ../abi.md#the-no_mangle-attribute
[external_block_abi]: external-blocks.md#abi
