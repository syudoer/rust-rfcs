- Feature Name: `obj-action-method-disambiguation`
- Start Date: 2026-01-20
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

This RFC proposes two extensions to Rust's method call syntax to unify method resolution and maintain fluent method chaining ("noun-verb" style) in the presence of naming ambiguities:

1.  **Ad-hoc Disambiguation**: `expr.(Trait::method)(args)` allows invoking a specific trait method inline without breaking the method chain.
2.  **Definition-site Aliases**: `pub use Trait as Alias;` within `impl` blocks enables `expr.Alias::method(args)`, allowing type authors to expose traits as named "facets" of their API.

## Motivation
[motivation]: #motivation

Currently, Rust's "Fully Qualified Syntax" (UFCS), e.g., `Trait::method(&obj)`, is the main mechanism to resolve method name conflicts between inherent implementations and traits, or between multiple traits.

While robust, UFCS forces a reversal of the visual data flow, breaking the fluent interface pattern:
*   **Fluent (Ideal)**: `object.process().output()`
*   **Broken (Current)**: `Trait::output(&object.process())`

This creates significant friction:
1.  **Cognitive Load**: The user must stop writing logic, look up the full trait path, import it, and restructure the code to wrap the object in a function call.
2.  **API Opacity**: Consumers often do not know which specific module a trait comes from, nor should they need to manage those imports just to call a method.

We propose a solution that restores the `object.action()` left-to-right flow in all cases and provides tools for both API consumers (ad-hoc fixes) and API producers (aliases).

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### The Problem: Ambiguous Methods

Imagine you are using a type `Image` that has an optimized inherent `rotate` method. Later, you import a graphics library with a `Transform` trait that also defines `rotate`.

```rust
struct Image;
impl Image {
    fn rotate(&self) { println!("Optimized internal rotation"); }
}

use graphic_lib::Transform;
impl Transform for Image {
    fn rotate(&self) { println!("Generic geometric rotation"); }
    fn crop(&self) { ... }
}
```

Calling `img.rotate()` is now ambiguous or defaults to the inherent method when you might intend to use the trait implementation.

### Solution 1: Ad-hoc Disambiguation (The "Quick Fix")

If you are a consumer of this API and need to resolve this ambiguity immediately without breaking your method chain, you can use parentheses to specify the trait:

```rust
// Calls the Trait implementation while maintaining the chain
img.crop().(Transform::rotate)().save();
```

This tells the compiler: "Use the `rotate` method from the `Transform` trait on this object."

**Note on `Self`**: While `img.(Self::rotate)()` is grammatically possible in some cases, it is discouraged. The compiler will warn you to remove the parentheses and use the explicit alias syntax described below.

### Solution 2: Definition-site Aliases (The "API Design" Fix)

As the author of `Image`, you can prevent this friction for your users. Instead of forcing them to import `graphic_lib::Transform` to access specific functionality, you can expose that trait as a named part of your `Image` API.

```rust
impl Image {
    // Inherent method
    fn rotate(&self) { ... }

    // Expose the Transform trait as 'OtherOps' (Operations)
    pub use graphic_lib::Transform as OtherOps;
}
```

Now, users can access these methods via the alias, which is conceptually part of the `Image` type:

```rust
let img = Image::new();

img.Self::rotate();     // Explicitly calls inherent (Optimized)
img.OtherOps::rotate(); // Calls via Alias -> Transform (Generic)
```

The `Self` keyword is implicitly treated as an alias for the inherent implementation, ensuring symmetry.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Grammar Extensions

The `MethodCallExpr` grammar is extended in two specific ways:

1.  **Parenthesized Path**: `Expr '.' '(' TypePath ')' '(' Args ')'`
    *   This syntax is used for **Ad-hoc Disambiguation**.
    *   **Resolution**: The `TypePath` is resolved. If it resolves to a trait method, it is invoked with `Expr` as the receiver (the first argument).
    *   **Desugaring**: `obj.(Path::method)(args)` desugars to `Path::method(obj, args)`, ensuring correct autoref/autoderef behavior for `obj`.
    *   **Restriction**: Using `(Self::method)` inside an `impl` block where `Self` is a type alias triggers a compiler warning, suggesting the removal of parentheses and usage of `Expr.Self::method()`.

2.  **Aliased Path**: `Expr '.' Ident '::' Ident '(' Args ')'`
    *   This syntax is used for **Definition-site Aliases**.
    *   **Resolution**: The first `Ident` is looked up in the `impl` block of the `Expr`'s type.
    *   **Alias Matching**:
        *   If `Ident` matches a `pub use Trait as Alias;` statement, the call resolves to `<Type as Trait>::method`.
        *   The keyword `Self` is implicitly treated as an alias for the inherent implementation. `obj.Self::method()` resolves to the inherent method.

3.  **Inherent Impl Items**:
    *   A `use Trait as Alias;` item is now valid within an inherent `impl` block.
    *   `use Trait;` is also supported as a shorthand for `use Trait as Trait;`.
    *   Visibility modifiers (e.g., `pub`) are supported and determine the visibility of the alias for method resolution.

### Resolution Logic Summary

*   **Case: `obj.(Trait::method)(...)`**
    *   Compiler verifies `Trait` is in scope or fully qualified.
    *   Resolves to UFCS call with `obj` as first arg.

*   **Case: `obj.Alias::method(...)`**
    *   Compiler looks up `Alias` in `obj`'s type definition.
    *   If found, maps to corresponding Trait implementation.

## Drawbacks
[drawbacks]: #drawbacks

*   **Cognitive Load** Increasing the cognitive load of users and Rust developers by adding more features. The definition-site aliases may eventually become a common way to group methods forcing users to deal with this feature not only in the case of method name disambiguation 
*   **Parser Complexity**: The parser requires lookahead or distinct rules to distinguish `.` followed by `(` (method call) versus `.` followed by `Ident` followed by `::` (aliased call).
*   **Punctuation Noise**: The syntax `.(...)` introduces more "Perl-like" punctuation symbols to the language, which some may find unaesthetic.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

*   **Why Aliases?**
    *   The primary benefit is for the **consumer**. They should not need to know the origin module of a trait to use it. Aliasing bundles the dependency with the type, treating the trait as a named interface/facet of the object.
    *   It mirrors C++ explicit qualification (e.g., `obj.Base::method()`).
*   **Why Parentheses for Ad-hoc?**
    *   `obj.Trait::method` is syntactically ambiguous with field access.
    *   `obj.(Trait::method)` is unambiguous and visually distinct.

## Prior art
[prior-art]: #prior-art

*   **C++**: Allows explicit qualification of method calls using `obj.Base::method()`, which served as inspiration for the Aliased Path syntax.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

*   **Syntax Choice**: Should we consider other bracket types to avoid confusion with tuple grouping? (e.g., `obj.{Trait::method}()` or `obj.[Trait::method]()`).

## Future possibilities
[future-possibilities]: #future-possibilities

*   **Scoped Prioritization**: We can also introduce syntax like `use Trait for Foo` or `use Self for Foo` within a function scope to change default resolution without changing call sites.
