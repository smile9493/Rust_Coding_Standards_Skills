# Rust FFI & Cross-Language Interoperability

This specification is designed for architectural scenarios that need to break through single-language ecosystems, coexist with C/C++ low-level assets, or serve as high-performance components callable from Python/Go/Node.js.

The core philosophy is the **"Breakwater Principle"**: absolutely block the chaos of the C language world (raw pointers, manual memory management, no lifetime concepts) outside system boundaries. On the business side, it must behave like a pure, safe, idiomatic Rust library.

---

## 1. Architecture Layering and Safe Boundary Design

**Core Philosophy**: Never let business-side code touch the `unsafe` keyword or raw pointers.

### 1.1 `-sys` Crate Separation Principle

**Rule**: Cross-language projects must be physically split into two crates.

- `xxx-sys`: Pure low-level bindings, containing all auto-generated C function signatures and raw structs, **no logic at all**, all `unsafe`.
- `xxx`: High-level safe wrapper. Handles all pointer dereferencing, lifetime binding, and error conversion here. Business side can only depend on this crate.

### 1.2 Lifetime Binding for Raw Pointers

**Rule**: Immediately convert C language's separate `*const u8` and `size_t` into Rust slices with lifetimes `&'a [u8]` at the safe wrapper entry point.

**Code Example**:
```rust
// ❌ Wrong: Let business side guess pointer validity and length
pub fn read_data(ptr: *const u8, len: usize) { ... }

// ✅ Correct: Use slice::from_raw_parts to bind lifetime
pub fn read_data<'a>(data: &'a [u8]) { ... }
```

### 1.3 Opaque Types to Isolate Internal State

**Rule**: If Rust needs to pass a complex struct for C to hold, or vice versa, **absolutely prohibit** defining a struct with identical fields on the other side to forcibly parse it (field alignment is extremely error-prone).

**Must Execute**: Use opaque pointers. In Rust, use zero-sized types (ZST) or private structs to represent C-side handles: `#[repr(C)] pub struct CxHandle { _private: [u8; 0] }`.

---

## 2. Memory Ownership and Cross-Language Handoff

**Core Philosophy**: Whoever allocates memory is responsible for releasing it (Symmetric Allocation). Absolutely prohibit Rust and C from cross-deallocating each other's memory.

### 2.1 Rust Allocation, Held by C (Leak & Reclaim)

**Rule**: When Rust passes a heap object (like `Box`, `Vec`, `String`) to C for long-term holding (e.g., as callback context), must use `into_raw()` to deprive Rust's implicit destruction rights (intentional memory leak).

**Closed-Loop Release**: Must simultaneously expose a dedicated release function to C.

**Code Example**:
```rust
// 1. Rust transfers ownership to C
#[no_mangle]
pub extern "C" fn create_engine() -> *mut Engine {
    Box::into_raw(Box::new(Engine::new()))
}

// 2. After C is done, must call this function; Rust reclaims and drops
#[no_mangle]
pub unsafe extern "C" fn destroy_engine(ptr: *mut Engine) {
    if !ptr.is_null() {
        drop(Box::from_raw(ptr)); // Reclaim ownership and auto-destroy
    }
}
```

### 2.2 C Allocation, Held by Rust

**Rule**: If C returns a pointer to Rust via `malloc`, the Rust-side wrapper struct **absolutely prohibit** using Rust's default behavior in its `Drop` implementation. Must call C's provided `free_xxx` API, or call `libc::free`.

### 2.3 Temporary Borrowed Cross-Language Copy (Copy vs Borrow)

**Trade-off**: If C briefly calls a Rust function to process a string, Rust side borrows via `CStr::from_ptr` (zero-copy). If Rust needs async processing or long-term preservation of that string, **must** immediately deep-copy into Rust's `String`. Never trust that C-provided pointers will remain valid.

---

## 3. Panic Containment and Exception Boundaries

**Core Philosophy**: Panics across FFI boundaries are undefined behavior (UB) and can cause the entire process to crash in extremely bizarre ways.

### 3.1 Catch All Unwind

**Rule**: All `extern "C"` interfaces exposed to C must wrap their internal logic in `std::panic::catch_unwind`.

**Error Code Mapping**: If a Panic is caught, must return a predetermined error code to C (like `-1` or null pointer). Never let exceptions escape Rust boundaries.

**Code Implementation**:
```rust
#[no_mangle]
pub extern "C" fn do_risky_work() -> i32 {
    let result = std::panic::catch_unwind(|| {
        // Core business logic that might panic
        business_logic()
    });

    match result {
        Ok(val) => val,
        Err(_) => -1, // Catch Panic and return C-understandable error code
    }
}
```

---

## 4. Automated Toolchain System

**Core Philosophy**: Manually writing C bindings by hand is the root of all evil. Must hand it off to toolchain and CI automation.

### 4.1 C Calling Rust (`cbindgen`)

**Rule**: Configure `cbindgen.toml` in the root directory, and automatically generate `header.h` from Rust source in CI or `build.rs`.

**Compromise**: Ensure structs exposed to C are tagged with `#[repr(C)]`. For complex Rust enums (carrying data), C cannot directly understand them. Must manually flatten to C-style structs and unions on the Rust side.

### 4.2 Rust Calling C (`bindgen`)

**Rule**: Use `bindgen` in `build.rs` to auto-generate Rust raw bindings from C headers.

**Engineering Compromise**: `bindgen` depends on underlying `libclang`, which makes build environments extremely demanding (all developer machines must have LLVM installed).

**Best Practice**: Don't run `bindgen` on every compilation. The correct approach: Run `bindgen` on a machine with specific environment (by architect or CI), commit the generated `bindings.rs` **into version control (Git)**. Regular business developers pull code and directly compile the generated `.rs` file, achieving build environment decoupling.

---

## Key Principles Summary

| Principle | Rule | Compromise |
|-----------|------|------------|
| **Crate Separation** | `-sys` + safe wrapper crates | No business logic in `-sys` |
| **Pointer Safety** | Convert to `&'a [u8]` at wrapper boundary | Never expose raw pointers to business |
| **Opaque Types** | Use ZST/private structs for C handles | Never match C struct fields manually |
| **Memory Ownership** | Symmetric allocation/deallocation | Never cross-deallocate |
| **Rust-to-C Handoff** | Use `into_raw()` + `destroy_xxx` | Deliberate leak for ownership transfer |
| **C-to-Rust Handoff** | Call `free_xxx` or `libc::free` in Drop | Never use Rust's default drop |
| **Temporary Borrow** | Use `CStr::from_ptr` for brief borrow | Deep-copy for async/long-term use |
| **Panic Containment** | Wrap all `extern "C"` in `catch_unwind` | Return error codes, never unwind |
| **C Header Generation** | Use `cbindgen` in CI/build.rs | Flatten Rust enums to C unions |
| **Rust Bindings** | Use `bindgen` in build.rs | Commit `bindings.rs` to version control |

## FFI Safety Checklist

Before any FFI code goes live:

- [ ] Two-crate separation: `-sys` (unsafe only) + wrapper (safe only)
- [ ] All raw pointers converted to lifetime-bound slices at boundary
- [ ] Opaque types used for cross-language struct handles
- [ ] Ownership transfer functions (`create_xxx`/`destroy_xxx`) paired
- [ ] C-allocated memory freed with C's `free_xxx` or `libc::free`
- [ ] Brief borrows via `CStr::from_ptr`, deep-copy for long-term use
- [ ] All `extern "C"` wrapped in `std::panic::catch_unwind`
- [ ] Error codes returned instead of panics crossing FFI boundary
- [ ] `cbindgen.toml` configured for C header generation
- [ ] `bindings.rs` committed to version control (not generated at build time)

## 5. Safe C++ Interop with `cxx`

When interfacing with C++ (not C), prefer `cxx` over raw FFI for type safety:

```rust
// Rust side: bridge definition
#[cxx::bridge]
mod ffi {
    extern "C++" {
        include "mylib/engine.h";

        type Engine;  // Opaque C++ type

        fn create_engine() -> UniquePtr<Engine>;
        fn process(engine: &Engine, data: &[u8]) -> Result<u64>;
    }
}

// Usage — fully type-safe, no raw pointers
let engine = ffi::create_engine();
let result = ffi::process(&engine, &data)?;
```

### `cxx` vs Raw FFI

| Feature | Raw FFI | `cxx` |
|---------|---------|-------|
| Type safety | Manual (unsafe) | Automatic (safe) |
| C++ types | Manual opaque pointers | `UniquePtr<T>`, `CxxString` |
| Error handling | Error codes | `Result<T>` across boundary |
| Boilerplate | High | Low (bridge macro) |
| Build integration | Manual | `cc` crate integration |

**Rule**: Use `cxx` for C++ interop. Use raw FFI only for C interop or when `cxx` doesn't support the required C++ feature.

## 6. Callback Functions Across FFI

### Rule: Never Pass Rust Closures as C Function Pointers

```rust
// ❌ Undefined behavior: Rust closure as C callback
extern "C" fn register_callback(cb: fn(*const u8)) {
    // Cannot pass Rust closure — it captures environment
}

// ✅ Pass data pointer + trampoline function
extern "C" fn register_callback(cb: extern "C" fn(*const u8, *mut c_void), user_data: *mut c_void);

// Rust trampoline
extern "C" fn trampoline(data: *const u8, user_data: *mut c_void) {
    let closure = unsafe { &*(user_data as *mut Box<dyn Fn(&[u8])>) };
    let slice = unsafe { slice_from_raw_parts(data, len) };
    closure(slice);
}
```

**Rule**: Always use the "trampoline + user_data" pattern for callbacks. Never cast Rust closures to C function pointers.

## Related References

- [toolchain.md](toolchain.md) — CI configuration and build automation
- [data-architecture.md](data-architecture.md) — Memory ownership principles
- [error-handling.md](error-handling.md) — Error handling strategies
- [concurrency.md](concurrency.md) — Thread safety in FFI callbacks
- [api-design.md](api-design.md) — API boundary design
