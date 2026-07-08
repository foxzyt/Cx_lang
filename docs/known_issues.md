# Cx Known Issues

This is a working log of known gaps and reference/JIT divergences found during
ongoing work, not a roadmap — see `docment/ROADMAP.md` for planned feature
sequencing. Entries are added as findings surface and updated (not deleted)
when fixed, so the history of what broke and why stays visible.

---

## 1. `f64` comparison (`<`, `>`, `<=`, `>=`) rejected by the interpreter

**Status: FIXED — commit `c5e8e22`.**

`src/runtime/ops.rs:173-208`'s comparison dispatch for `Lt`/`Gt`/`LtEq`/`GtEq`
had exactly one match arm per operator — `(Value::Num(a), Value::Num(b))`,
integers only — with no arm for `(Value::Float(a), Value::Float(b))`, falling
through to `RuntimeError::BadOperands`. Semantic analysis already accepted
float operands (`is_numeric` includes `F64`), so this was purely a
runtime-dispatch gap in the interpreter, not a semantic-analysis rejection.

**This was the reverse of the usual pattern.** Every other interpreter/JIT
divergence found this session had the JIT lagging the interpreter (JIT
missing something the interpreter already did correctly). Here the JIT was
already correct and the interpreter — normally treated as the reference
implementation — was the one that was wrong. Keep that context in mind if
anyone looks back at this entry later confused by the direction.

Fix: added the missing `(Value::Float, Value::Float)` arms, mirroring the
existing integer arms exactly. A mixed `Num`/`Float` comparison (e.g.
`5 > 3.14`) is not a separate case — semantic analysis's
`common_numeric_type`/`insert_cast_if_needed` already promotes the int
literal to `F64` via an inserted `Cast` node before it reaches the runtime,
so it flows through the same `(Float, Float)` arm.

---

## 2. Bare builtin call in trailing function/method-body position doesn't JIT-lower

**Status: OPEN.**

A bare `print(...)` call as a function or method body's *sole or last*
statement, with no trailing semicolon, fails to lower on the JIT:

```
unresolved semantic artifact reached lowering: function 'print'
```
Exit code 127. The interpreter handles this correctly.

**Not method-specific** — confirmed via a plain free-function reproducer
(`fnc: show(x: t32) { print(x) }`) with an identical failure and identical
error text/exit code. The original framing ("print inside a method body")
mis-attributed the cause to methods; methods just happen to be a natural
place to write this shape (a one-line `fnc: show_health() { print(p.health) }`).

**Mechanism:** the parser's `func_body` combinator
(`src/frontend/parser.rs:1207-1259`) purely-syntactically promotes the body's
last statement into `ret_expr` whenever it's a bare `ExprStmt` with no
trailing semicolon — with zero understanding of what the expression is. The
resulting `ret_expr` is lowered via the general expression path,
`lower_expr` (`src/ir/lower.rs:686-687`), not the statement path,
`lower_stmt`, which is where `print`/`println`/`printn`/`assert`/`assert_eq`
get their special-case builtin interception (documented at
`src/ir/lower.rs:94-95`). `lower_expr`'s `Call` handling
(`src/ir/lower.rs:1969-1984`) carries an explicit comment acknowledging this
exact scenario was assumed not to happen: *"assert/assert_eq, print,
println, and printn are handled at statement level and should not reach
`lower_expr` in well-formed programs."* `is_cx_builtin`
(`src/ir/lower.rs:96-104`) only recognizes builtins whose JIT status is
`GatedUnsupported` (genuinely-unimplemented ones like `read`/`input`) —
`print`'s status is `Lowered`, so it isn't caught there either, and falls
through to a raw `signature_table` miss, producing the observed
`UnresolvedSemanticArtifact`.

Only `print` was empirically tested; `println`/`printn`/`assert`/`assert_eq`
likely share the same failure by the same code path (same
`lower_stmt`-interception list) but this has not been individually
confirmed for each.

**Risk framing:** low risk of a silent false-parity-fail. This fails with
the JIT's own SKIP exit code (127), so if someone later writes a fixture
with this exact shape, `jit_parity_by_feature` will correctly bucket it as
SKIP, not a misleading PARITY_FAIL — unlike finding #1 above (which, before
the fix, would have shown as a genuine PARITY_FAIL had a fixture existed,
since the JIT succeeded while the interpreter errored).

**Relation to #3 below:** sits in the same code region (both concern a
function's trailing-expression / `ret_expr` handling) but is **not a
duplicate** — confirmed by direct mechanism/error-text comparison, not
name-similarity. This bug raises `LoweringError::UnresolvedSemanticArtifact`
at `lower.rs:686-687` (the `lower_expr` call itself failing); #3 raises
`LoweringError::InternalInvariantViolation` one step further down, at
`lower.rs:677-684`. In the specific reproducer tested here (a void function,
no `return` statement, `print(...)` as the only body statement), this bug
fires first and **preempts** #3's check from ever being reached — if this
bug were fixed, the exact same reproducer would then likely hit the
*separate* implicit-return-type gap (see the `t16`-cluster audit) rather
than #3 specifically, since #3 requires an explicit `return` statement to
also be present.

---

## 3. `t03`/`t160`/`t24` — explicit return + trailing expression

**Status: OPEN, deferred** (found during the `t16`-cluster scoping audit;
deferred there as open-ended, not yet sized).

A function with **both** an explicit `return` statement and a separate,
now-dead trailing expression statement after it triggers:

```
LoweringError::InternalInvariantViolation {
    detail: "function '{name}' has both explicit return terminator and trailing return expression"
}
```
at `src/ir/lower.rs:677-684`.

Affects `t03_explicit_return.cx`, `t160_direct_call_explicit_return_exit.cx`,
and (as a compound case, masking a second, separate bug — the `t16`
implicit-return-type gap) `t24_full_system_regression.cx`.

**Distinct from #2 above** — different error variant
(`InternalInvariantViolation` vs. `UnresolvedSemanticArtifact`), different
trigger (requires an explicit `return` statement present in the same
function; #2's reproducer has none) — confirmed by direct comparison of
error text and mechanism, not by name-similarity.

Not yet sized. The `t16`-cluster audit found the related implicit-return-type
gap touches broad, shared function-signature infrastructure with unclear
full extent, and recommended deferring the whole cluster rather than
attempting a quick cleanup. This entry (`t03`/`t160`/`t24` specifically) was
not separately re-scoped beyond that.

---

## 4. `print(enum)` diverges between interpreter and JIT

**Status: OPEN.** Not sized.

Printing a bare enum-typed value produces different output on each backend:

- Interpreter: the variant name, e.g. `Color::Green`
  (`src/runtime/runtime.rs:152`: `Value::EnumVariant(e, v) => format!("{}::{}", e, v)`).
- JIT: the raw tag value, e.g. `1` — because `SemanticType::Enum(_)` erases
  to the tag's IR type, `IrType::I8`, at lowering
  (`src/ir/lower.rs:4563`), and printing an `I8` just prints the integer
  (`route_print_arg`, see #5 below for its full dispatch).

Found incidentally while designing the pattern-matching `as v`-binding
discriminating-canary fixture: `print(v)` on an enum-typed binding was
briefly considered as the test's payload, then dropped in favor of a
nested-`when`-based canary specifically to avoid this exact divergence
contaminating an unrelated feature's regression test.

No existing fixture in the verification matrix exercises `print` on a bare
enum value in a way that's checked for interp/JIT parity, so this gap has
not yet surfaced as a `PARITY_FAIL` in `jit_parity_by_feature` — but would,
if one were added.

---

## 5. Bare `I128` printing not lowered on JIT

**Status: OPEN, fully scoped, ready to build whenever picked up** — no
further audit needed for this one specifically.

`route_print_arg` (`src/ir/lower.rs:4646-4667`) dispatches on the print
argument's `IrType`: `I64` direct, `I8`/`I16`/`I32` via a widening `Cast`,
`Bool`/`TBool` via `cx_print_bool` — and a catch-all `_ => Ok(None)` that
rejects everything else uniformly (`F64`, `I128`, `Ptr`, `Str`, composites).
`I128` is a **plain omission**, not a deliberate architectural exclusion —
nothing else in the surrounding code treats `I128` specially or defers it on
purpose.

Affects any bare `i128`-typed print, including reading a `Handle<T>`'s value
when `T` is `t128`.

Scope for the fix (not built): one new match arm in `route_print_arg` plus
one new host-side callback (mirroring the shape of any single existing
Handle-value print callback already built this session, e.g. the scalar
`Handle` printing path from D2.5).
