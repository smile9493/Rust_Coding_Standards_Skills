# Agent Self-Check List & Reduction Directive

> **Philosophy — Jeet Kune Do (截拳道, Compile-time Defense First)**: When receiving a Rust coding task, the Agent must fold the logic with interception intuition before output — ensuring every line of code carries maximum energy density.

---

## 1. Agent Self-Check List

Before outputting any Rust code, the Agent **MUST** verify the following:

1. **Are code paths flat?** Can `let else` or `?` eliminate `if let` nesting?
2. **Are collection operations idiomatic?** Prefer iterator adapters where they improve clarity; a simple `for` loop is acceptable when it reads more clearly or when early-exit logic is needed.
3. **Are variable scopes minimized?** Can shadowing remove no-longer-needed `mut`?
4. **Are there implicit copies?** Is `.to_string()` or `.clone()` misused on hot paths?
5. **Is naming stuttering?** e.g., `user::UserConfig` should be `user::Config`.

---

## Reduction Directive

When receiving a Rust coding task, aim for high signal-to-noise ratio: eliminate unnecessary ceremony, use idiomatic patterns, and prefer clarity over cleverness. The philosophy labels (Jeet Kune Do, Economy of Motion) are branding metaphors — operational rules are defined by the priority pyramid and execution mode system.
