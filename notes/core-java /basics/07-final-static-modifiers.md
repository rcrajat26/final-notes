# `final` and `static` keyword

## `final` keyword
| Applied to             | Meaning                                                                                                                                                          |
|------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Variable (local/field) | Can be assigned once. For objects, it freezes the reference, not the contents — `final List` still lets you mutate the list, just not reassign it to a new list. |
| Method                 | Cannot be overridden by subclasses (can still be overloaded).                                                                                                    |
| Class                  | Cannot be extended at all (e.g. `String`, `Integer` are `final`).                                                                                                |
- Final field can be initiated either at the point of declaration, in the constructor or in an initialization block. It must be assigned exactly once before the constructor finishes. It must be declared in all the constructors.

## `static` keyword
| Applied to        | Meaning                                                                   |
|-------------------|---------------------------------------------------------------------------|
| Field             | lives in Metaspace. Shared across all `Client` instances                  |
| Method            | No `this`, can't touch instance fields directly, resolved at compile time |
| Nested class      | Doesn't hold an implicit reference to an enclosing instance               |
| Initializer block | `static { ... }` — runs once, at class load.                              |


## Access modifier levels
> private < default < protected < public

### Miscellaneous 
**1. Final/effectively-final local variables and lambdas:** A local variable must be final or effectively final (assigned once, never reassigned) to be used inside a lambda because lambdas capture local variables by value, not by reference. The captured value gets baked into the lambda at creation time since the lambda may outlive the method's stack frame (which disappears once the method returns). If reassignment were allowed, it'd be unclear whether the lambda should see the old or new value — a value that might not even exist anymore. This restriction only applies to local variables; instance fields can be freely mutated inside a lambda because they're accessed via `this` on the heap, with no stack-frame lifetime issue.

**2. Instance init block vs. constructor for a final field:** Assigning a `final` field in both the instance init block and the constructor is a compile-time error, not a runtime "last write wins." The compiler treats the instance block and constructor body as one continuous sequence (block code runs first, spliced in right after `super()`), and a `final` field can be assigned exactly once across that whole sequence — so the second assignment fails to compile. This is different from a non-final field, where both assignments are legal and the constructor's value simply overwrites the instance block's value at runtime, since it runs later.

**3. Static imports with colliding names (`java.lang.Math.PI` vs `custom.my.Math.PI`):** Two single static imports of the same name cause an immediate compile error at the import declarations. Two wildcard static imports exposing the same name compile fine, but using the bare name causes an "ambiguous reference" error only at the point of use. Either way, the fix is to avoid static-importing the colliding name and instead qualify it explicitly (`java.lang.Math.PI`, `custom.my.Math.PI`) — generally the safer practice whenever a naming collision is plausible.
