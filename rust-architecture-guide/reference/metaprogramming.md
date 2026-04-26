# Rust Metaprogramming & Macro Magic

This specification is designed for architects who need to build low-level frameworks, eliminate boilerplate code, or pursue zero-cost abstractions. The core philosophy is **"Restraint and Transparency"** — macros are the sharpest weapon in Rust. Mastering them gives you a god's-eye view, but abuse will quickly push your codebase into an unreadable, undebuggable abyss.

**Absolute Prerequisite**: Before deciding to write a macro, you must ask yourself three times: "Can this problem really not be solved with Generics, Traits, or simple inline functions?" Only when the conventional type system is powerless should you invoke macro magic.

---

## 1. Declarative Macros (`macro_rules!`)

**Core Philosophy**: Treat `macro_rules!` as "smart copy-paste with pattern matching," only used for eliminating structural boilerplate repetition.

### 1.1 Scope and Hygiene

**Rule**: Declarative macros must remain "hygienic." **Absolutely prohibit** implicitly assuming or capturing variable names from the external environment inside the macro. All variables and identifiers needed inside the macro must be explicitly passed through macro parameters.

- **Anti-pattern**: Directly using outer scope variables like `db_conn` inside the macro.
- **Elegant**: Pass identifiers as `$conn:ident` into the macro.

### 1.2 Beware the "Abyss Recursion" (TT Munchers)

**Rule**: **Prohibit** writing overly complex recursive macros based on Token Tree Munching in business code. This turns compilation error messages into hieroglyphics.

**Compromise**: If a declarative macro's nested parsing logic exceeds 3 levels, or if you need to implement a DSL (Domain-Specific Language) with macros, **immediately** abandon `macro_rules!` and write procedural macros instead.

### 1.3 Macro Visibility and Export

**Rule**: Unless providing global scaffolding for the entire project (like `vec!`), leverage local scope (define macros inside modules or functions), avoid using `#[macro_export]` to pollute the global namespace.

---

## 2. Procedural Macros (AST Manipulation)

**Core Philosophy**: Write procedural macros like writing a mini compiler. The highest standard for codegen is "precision of error localization."

Procedural macros are essentially Rust programs that run during the compilation phase. They consume code and produce code.

### 2.1 Isolation and Dependency Control

**Rule**: Procedural macros must be isolated in separate crates with `proc-macro = true`. Since procedural macros depend on the extremely large `syn` and `quote` libraries, **strictly prohibit** pure business logic crates from depending on procedural macro crate internals, to prevent dragging down the entire project's compilation speed.

### 2.2 Master-Level Error Throwing (Span-level Errors)

**Rule**: The dividing line for writing procedural macros lies in failure handling. **Absolutely prohibit** using `panic!` to report errors inside procedural macros, as this causes the compiler to crash directly and point to the macro invocation's location.

**Must Execute**: When AST parsing fails, you must capture the error and use `syn::Error::new_spanned` to precisely bind the error to the **specific variable name or attribute** that caused the problem.

**Code Example**:
```rust
// ❌ Amateur approach: Direct panic, user has no idea which part of their code is wrong
if !field.is_public() { panic!("Field must be public!"); }

// ✅ Master approach: Compiler precisely highlights that private field in red in IDE
return Err(syn::Error::new_spanned(&field.ident, "This field must be declared as pub"));
```

### 2.3 Prefer Derive Macros, Use Attribute Macros with Caution

**Rule**: In business architecture, prioritize `#[derive(MyTrait)]` macros because they only append code without destroying the original structure, making them very IDE-friendly (rust-analyzer can easily infer types).

**Compromise**: Use custom attribute macros (like `#[my_route("/api")] fn...`) extremely cautiously. Since attribute macros **completely devour** and rewrite the original function, they can easily cause large-scale IDE features like go-to-definition, auto-completion, and type inference to fail.

---

## 3. Compile-time Computing (`const`)

**Core Philosophy**: Squeeze every ounce of the compiler's computational power in exchange for ultimate runtime lightness.

### 3.1 Pervasive Const Functions (`const fn`)

**Rule**: For functions that do not involve heap allocation (no `Vec`, `Box`, `String`), do not involve I/O, and are completely deterministic (like hash computations, state machine initialization, matrix transformations), **add the `const` keyword whenever possible**.

**Benefit**: This allows these functions' computation to complete at compile time and can be directly used to initialize global static variables (`static`), completely eliminating runtime lock overhead from `lazy_static` or `OnceCell`.

### 3.2 Const Generics to Replace Heap Allocation

**Rule**: When handling low-level logic like cryptography, network protocol frames, or matrix operations, if data dimensions/lengths are known at compile time, **absolutely prohibit** using `Vec<T>`. Must use arrays with const generics `[T; N]`.

**Code Example**:
```rust
// ❌ Runtime heap allocation, extremely inefficient
struct Matrix { data: Vec<f64>, rows: usize, cols: usize }

// ✅ Zero heap allocation, all data on stack and contiguous, ultimate cache affinity
struct Matrix<const R: usize, const C: usize> { data: [[f64; C]; R] }
```

### 3.3 Beware the "Viral Spread" of Const Generics

**Trade-off**: Const generics are very contagious. If you add `<const N: usize>` to a core struct, then all functions and upper-layer structs using it will be forced to carry this generic parameter.

**Compromise**: If generic parameters exceed 3, or if it causes massive template repetition in the codebase, immediately draw the line and retreat to using `&[T]` slices (trading a bit of runtime size-checking overhead for extreme code simplicity).

---

## 4. Ultimate Debugging Toolchain for Macro Development

As an architect, you must mandate that your team uses the following tools when developing macros:

1. **`cargo expand`**: The "mirror that reveals true nature" for macro development. It completely expands all `macro_rules!` and procedural macros into raw Rust code. Before committing any macro, you must first review with expand whether the generated code has redundancy or logical flaws.

2. **`macrotest` or `trybuild`**: **Must** write compile-time tests for your procedural macros. Not only test that macros generate correct code, but also test that when users write incorrect code, macros can throw expected "compile errors" (Compile-fail tests).

---

## Key Principles Summary

| Principle | Rule | Compromise |
|-----------|------|------------|
| **Macro Decision** | Ask 3 times if generics/traits/functions suffice | Only use macros when type system is powerless |
| **Declarative Macro Hygiene** | Explicitly pass all identifiers via parameters | Avoid capturing outer scope variables |
| **TT Munching** | Prohibit >3 levels of nested macros | Switch to procedural macros for complex DSLs |
| **Macro Visibility** | Use local scope over `#[macro_export]` | Global only for project-wide scaffolding |
| **Procedural Macro Crate** | Isolate in separate `proc-macro = true` crate | Never depend on macro internals from business code |
| **Error Handling** | Use `syn::Error::new_spanned` | Never `panic!` in procedural macros |
| **Derive vs Attribute** | Prefer `#[derive(...)]` | Use `#[...]` attribute macros sparingly |
| **Const Functions** | Add `const` to pure, deterministic functions | No heap allocation, no I/O |
| **Const Generics** | Use `[T; N]` over `Vec<T>` when dimensions known | Retreat to `&[T]` if generics exceed 3 |
| **Macro Testing** | Use `cargo expand` and `trybuild` | Mandatory compile-fail tests |

## Related References

- [traits.md](traits.md) — Trait design patterns
- [data-architecture.md](data-architecture.md) — Zero-cost abstraction boundaries
- [toolchain.md](toolchain.md) — Development toolchain configuration
- [api-design.md](api-design.md) — API engineering and boundary abstraction
