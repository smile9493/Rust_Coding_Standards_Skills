# Metaprogramming & Macro Magic: Intercepting Boilerplate

> **Core Philosophy — Intercepting Boilerplate**: Macros are not for creating "magic," but for intercepting repetitive, mechanical, and error-prone boilerplate code at compile time. The highest-level metaprogramming should be **transparent**: the code it generates should be as efficient and debuggable as hand-written code.

---

## 1. Declarative Macros: Economy of Force

> **Economy of Motion — Eliminate structural repetition.**

### 1.1 Hygiene & Explicit Contracts

**Rule**: `macro_rules!` must maintain hygiene. **Prohibit** implicitly capturing external scope variables. All dependent identifiers must be explicitly passed as parameters.

**Purpose**: Ensure logical consistency when macros are called across different modules, avoiding naming pollution.

```rust
// ❌ Implicit capture — fragile
macro_rules! process {
    () => { $db_conn.execute(...) };
}

// ✅ Explicit contract — robust
macro_rules! process {
    ($conn:ident) => { $conn.execute(...) };
}
```

### 1.2 Intercepting Complex Recursion (Complexity Control)

**Rule**: Prohibit writing recursive macros (TT Munchers) with more than 3 levels of nested parsing logic.

**Judgment**: If a macro's logic starts involving complex Token transformations, it **must** be upgraded to a procedural macro.

### 1.3 Locality Principle

**Rule**: Unless providing global scaffolding (e.g., `vec!`-level), prohibit overusing `#[macro_export]`. Prefer using `macro_rules!` within module scope to minimize pollution radius.

---

## 2. Procedural Macros: Compiler-Level Interception

> **Intercepting Way — Write macros like writing a compiler.**

### 2.1 Precise Span-Level Error Reporting

**Rule**: **Absolutely prohibit** using `panic!` inside procedural macros.

**Execution**: Must use `syn::Error` and `Span` to precisely locate errors to the user's source code line.

```rust
// ✅ Correct: indicates which field violates the requirement
return Err(syn::Error::new_spanned(field_ident, "This field must be marked with #[napi]"));
```

### 2.2 Derive Macro Priority Principle

**Rule**: When building plugin systems or serialization frameworks, prioritize `#[derive(Trait)]`, then attribute macros `#[attr]`, and lastly function-like macros `macro!()`.

**Reason**: Derive macros only append code without modifying the original structure, making them most IDE-friendly for type inference — conforming to the principle of "aligning with the physical substrate."

### 2.3 Compile-Cost Economic Isolation

**Rule**: Procedural macros must be isolated in separate crates with `proc-macro = true`. Since `syn` compiles extremely slowly, prohibit introducing complex procedural macros in lightweight crates.

---

## 3. Compile-Time Computing: Zero-Overhead Limit

> **Physical Alignment — Squeeze the compiler's computational power in exchange for runtime silence.**

### 3.1 Universal `const fn`

**Rule**: All helper functions that do not involve I/O and have deterministic inputs and outputs (e.g., hash computations, configuration parsing) **must** be declared as `const fn`.

**Purpose**: Achieve true "inch-power" — absolute zero runtime overhead.

```rust
const fn hash_tag(tag: &str) -> u64 {
    let mut hash: u64 = 0;
    let bytes = tag.as_bytes();
    let mut i = 0;
    while i < bytes.len() {
        hash = hash.wrapping_mul(31).wrapping_add(bytes[i] as u64);
        i += 1;
    }
    hash
}

static TAG_HASH: u64 = hash_tag("my_tag"); // Computed at compile time
```

### 3.2 Const Generics Replacing Heap Allocation

**Rule**: For data structures with known dimensions (e.g., cryptographic buffers, linear algebra matrices), **mandate** using const generics `[T; N]` instead of `Vec<T>`.

**Architectural Benefit**: Return memory allocation from heap to stack, aligning with CPU cache locality.

```rust
// ❌ Runtime heap allocation
struct Matrix { data: Vec<f64>, rows: usize, cols: usize }

// ✅ Zero heap allocation, stack-resident, cache-friendly
struct Matrix<const R: usize, const C: usize> { data: [[f64; C]; R] }
```

---

## 4. Maintainability & Black-Box Transparency

> **Unity of Opposites — Magic must be auditable.**

### 4.1 Macro Expansion Review (Expand)

**Rule**: Before committing any code containing complex macros, must review the generated raw Rust code via `cargo expand`.

**Self-check points**: Does the expanded code contain redundant clones? Unnecessary allocations?

```bash
cargo expand --lib  # Review all macro expansions
```

### 4.2 Compile-Fail Tests (UI Tests)

**Rule**: Macro projects must include `trybuild` or `compiletest` test suites, verifying that when users incorrectly use macros, the compiler produces expected "human-readable" error messages.

```rust
#[test]
fn ui_tests() {
    let t = trybuild::TestCases::new();
    t.compile_fail("tests/ui/*.rs"); // Verify error messages are clear
}
```

---

## 5. Agent Metaprogramming Checklist

1. **Can this macro be replaced with `impl Trait` generics?** (If yes, abandon the macro)
2. **Does error handling use `syn::Error` with correct `Span` association?**
3. **Is `const fn` used to pre-compute results on hot paths?**
4. **Does the procedural macro cause unreasonable doubling of total project compile time?**
5. **Does the generated code pass `clippy` checks?** (Macro-generated code must also be elegant)

---

**Reduction Directive**: When generating macro magic, the Agent must maintain "cold rationality." Behind every piece of macro-generated code, there must be a clear "boilerplate elimination" benefit or "performance-frontloading" motivation. Prohibit introducing hard-to-maintain symbol nesting for the sake of showing off.
