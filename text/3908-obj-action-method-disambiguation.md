- Feature Name: `obj-action-method-disambiguation`
- Start Date: 2026-01-20
- RFC PR: [rust-lang/rfcs#3908](https://github.com/rust-lang/rfcs/pull/3908)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

This RFC proposes two extensions to Rust's method call syntax to unify method call syntax and maintain fluent method chaining ("noun-verb" style) in the presence of naming ambiguities:

1.  **Trait Method Call**: `expr.(path::to::Trait::method)(args)` allows invoking a specific trait's method inline without breaking the method chain. (The parentheses around `path::to::Trait::method` are required)
2.  **Inherent Method Call**: `expr.Self::method(args)` is an explicit way to call an inherent method. (Unlike the previous case parentheses are not allowed)

## Motivation
[motivation]: #motivation

### Method chain break
Currently, Rust's "Fully Qualified Syntax" (UFCS), e.g., `Trait::method(&obj)`, is the main mechanism to disambiguate method calls between inherent implementations and traits, or between multiple traits.

While robust, UFCS forces a reversal of the visual data flow, breaking the fluent interface pattern:
*   **Fluent (Ideal)**: `object.process().output()`
*   **Broken (Current)**: `Trait::output(&object.process())`

### Silent bugs and Fragility

Currently, Rust's method resolution follows a fixed priority: it defaults to an inherent method if one exists. If no inherent method is found, the compiler looks for traits in scope that provide the method. If exactly one such trait is implemented for the type, the compiler selects it; otherwise, it returns an error.

This creates a "Primary and Fallback" mechanism where the compiler can silently switch between logic. If a primary (inherent) method is removed or renamed, the compiler may silently fall back to a trait implementation. Conversely, adding an inherent method can unexpectedly shadow an existing trait method call.

In rare cases, modifying one part of the code can unexpectedly alter logic elsewhere, causing a chain reaction of errors that makes it difficult to locate the root cause.

### Summary

This RFC aims to fully solve the problem of fluent method chaining. The second problem (fragility) requires a more complex approach, with this RFC being the first step. More details on potential future solutions are discussed in the [Future Possibilities](#future-possibilities) section.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are three ways to have something callable inside of `obj`:
- as a field containing a pointer to a function
- as an inherent method
- as a trait method 

While the first one is not confusing and has its unique syntax `(value.field)(args)`, the other two may cause some unexpected errors that seem unrelated to the actual mistake.

Imagine you have this piece of code 
```rust
use std::fmt::Display;

struct SomeThing<T> {
    something: fn(T),
}

impl<T: Copy + Display> SomeThing<T> {
    fn something(&self, _arg: T) {
        println!("inherent fn called, got {}", _arg)
    }
}

trait SomeTrait<T: Copy + Display> {
    fn something(&self, _arg: T);
}

impl<T: Copy + Display> SomeTrait<T> for SomeThing<T> {
    fn something(&self, _arg: T) {
        println!("trait fn called, got {}", _arg);

        print!("\t");
        self.something(_arg);
    }
}

fn main() {
    let value = SomeThing { something: |_arg: i32| {println!("fn pointer called, got {}", _arg)} };

    value.something(1);
    (value.something)(2);
    SomeTrait::something(&value, 3);
}
```

it works, it handles all three ways and prints
```plain
inherent fn called, got 1
fn pointer called, got 2
trait fn called, got 3
    inherent fn called, got 3
```

but if you change the line `impl<T: Copy + Display> SomeTrait<T> for SomeThing<T> {` to `impl<T: Copy + Display, U: Copy> SomeTrait<T> for SomeThing<U> {` instead of producing an error for the mismatch, the code compiles successfully and prints 

```plain
inherent fn called, got 1
fn pointer called, got 2
trait fn called, got 3
    trait fn called, got 3
    trait fn called, got 3
    trait fn called, got 3
    trait fn called, got 3
    trait fn called, got 3
    trait fn called, got 3
    trait fn called, got 3
    trait fn called, got 3
    trait fn called, got 3
    trait fn called, got 3
    ...
```

NOTE: If instead of `impl<T: Copy + Display, U: Copy> SomeTrait<T> for SomeThing<U> {` you wrote `impl<T: Copy + Display, U: Copy + Display> SomeTrait<T> for SomeThing<U> {` the compiler would return a comprehensible error but because you've forgotten `+ Display` the compiler does not even try to do what you suppose it to do 

You would also get the same undesirable behavior in another case. You could rename `something` in `SomeThing`'s impl block and forget to rename it in the `SomeTrait`'s impl block
```rust
impl<T: Copy + Display> SomeTrait<T> for SomeThing<T> {
    fn something(&self, _arg: T) {
        println!("trait fn called, got {}", _arg);

        print!("\t");
        self.something(_arg); // here
    }
}
```

To prevent this and ensure the compiler rejects broken code, it would be better to use `self.Self::something(_arg)` instead of `self.something(_arg)`.

```rust
impl<T: Copy + Display> SomeTrait<T> for SomeThing<T> {
    fn something(&self, _arg: T) {
        println!("trait fn called, got {}", _arg);

        print!("\t");
        self.Self::something(_arg);
    }
}
```

`value.Self::method()` allows the compiler to only use an inherent method called `method` and errors if it hasn't been found.

### Method Chain Conflicts

Sometimes the ambiguity arises not within an implementation, but when using a type that implements traits with overlapping method names.

Consider a scenario where you have a `Builder` struct that implements both a `Reset` trait and has an inherent `reset` method.

```rust
struct Builder;
impl Builder {
    fn build(&self) -> String { "done".to_string() }
    fn reset(&self) -> &Self { self }
}

trait Reset {
    fn reset(&self) -> &Self;
}

impl Reset for Builder {
    fn reset(&self) -> &Self { self }
}

fn main() {
    let b = Builder {};
    // Defaults to the inherent method `reset` but silently falls back to the trait implementation if the inherent method is removed or renamed
    b.reset().build(); 
}
```

Using the new syntax, you can explicitly choose which method to use without breaking the chain:

```rust
fn main() {
    let b = Builder;

    // Use the inherent reset method
    b.Self::reset().build();

    // Use the trait's reset method explicitly
    b.(Reset::reset)().build();
}
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Grammar Extensions

The `MethodCallExpr` grammar is extended in two specific ways:

1.  **Parenthesized Path**: `Expr '.' '(' TypePath ')' '(' Args ')'`
    *   This syntax is used for **Explicit Trait Method Calls**.
    *   **Resolution**: The `TypePath` is resolved. If it resolves to a trait method, it is invoked with `Expr` as the receiver (the first argument).
    *   **Desugaring**: `obj.(Path::method)(args)` desugars to `Path::method(obj, args)`, ensuring correct autoref/autoderef behavior for `obj`.
    *   **Restriction**: `Expr.(Self::Ident)(Args)` is not allowed.

2.  **Explicit Inherent Path**: `Expr '.' 'Self' '::' Ident '(' Args ')'`
    *   This syntax is used for **Explicit Inherent Method Calls**.
    *   **Resolution**: The `Ident` is looked up strictly within the inherent implementation of `Expr`'s type.
    *   **Semantics**: `obj.Self::method()` resolves to the inherent method `method`. It effectively bypasses trait method lookup.

### Resolution Logic Summary

*   **Case: `obj.(Trait::method)(...)`**
    *   Compiler verifies `Trait` is in scope or fully qualified.
    *   Resolves to UFCS call with `obj` as first arg.

*   **Case: `obj.Self::method(...)`**
    *   Compiler looks up `method` in `obj`'s inherent implementation.

## Drawbacks
[drawbacks]: #drawbacks

*   **Parser Complexity**: The parser requires lookahead or distinct rules to distinguish `.` followed by `(` (method call) versus `.` followed by `Self` followed by `::`.
*   **Punctuation Noise**: The syntax `.(...)` introduces more "Perl-like" punctuation symbols to the language, which some may find unaesthetic.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

*   **Why Parentheses for Trait Method Calls?**
    *   `value.Trait::method` looks like there is something called `Trait` inside the `value` while `Trait` is coming from the scope of the call.
    *   `value.(Trait::method)` shows that `Trait::method` is evaluated first and then applied to the `value`.
    *   **Reservation**: We specifically reserve the unparenthesized syntax `value.Category::method()` (where `Category` is not `Self`) for possible future language features, such as "Categorical" or "Facet" views of an object. Using parentheses around trait paths avoids closing the door on this design space.

*   **Why No Parentheses for Inherent Method Call?**
    *   Unlike `Trait`, which comes from the outer scope, `Self` conceptually belongs to the instance itself.
    *   `value.Self::method()` aligns with the mental model that `Self` is intrinsic to `value`, acting as a specific "facet" of the object itself, rather than an external function applied to it. This justifies the lack of parentheses, matching the reserved `value.Category::method()` syntax.

## Prior art
[prior-art]: #prior-art

*   **C++**: Allows explicit qualification of method calls using `obj.Base::method()`.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

*   **Syntax Choice**: Should we consider other bracket types to avoid confusion with tuple grouping? (e.g., `obj.<Trait::method>()` or `obj.[Trait::method]()` or)?

## Future possibilities
[future-possibilities]: #future-possibilities

*   **Scoped Prioritization**: We can also introduce syntax like `use Trait for Foo` or `use Self for Foo` within a function scope to change default resolution without changing call sites.
*   **Disabling Inherent Preference**: A specialized macro or attribute could be introduced to opt-out of the default "inherent-first" resolution rule (effectively canceling the implicit `use Self for *`). This aligns with Rust's philosophy of explicit over implicit behavior where ambiguity exists, ensuring that code correctness is verifiable.