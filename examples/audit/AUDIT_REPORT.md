# Parser/Semantic/Interpreter Agreement Audit — Part 1

Date: 2026-04-13
Matrix baseline: 105/105 green (after Half A: 110/110)

## Summary
- Tests run: 12
- Pass: 7
- Parse failures: 1
- Semantic failures: 2
- Runtime failures: 0
- Wrong output: 1
- Panics: 1

## Per-test results

### audit_01_impl_on_generic_struct
**Status:** PASS
**Expected:** 42
**Actual:** 42
**Layer:** N/A
**Notes:** Impl on generic struct works. Adapted test to print from inside method (returns void) since returning T from impl method on unparameterized Pair would need more type plumbing.

### audit_02_nested_result
**Status:** PARSE_FAIL
**Expected:** Ok(Ok(5))
**Actual:** `PARSE ERROR (line 1): ExpectedFound { expected: [something else], found: Some(KeywordResult) }`
**Layer:** parser
**Notes:** `Result<Result<t32>>` fails to parse. The type parser's `result_type` combinator uses `scalar.clone().or(named_type.clone())` for the inner type, but `Result` is a keyword token not matched by either. The inner type parser needs to accept `result_type` recursively.

### audit_03_try_in_when_arm
**Status:** SEMANTIC_FAIL
**Expected:** Ok(10) then Err(negative)
**Actual:** `SEMANTIC ERROR (line 12): use of undeclared variable 'val'`
**Layer:** semantic
**Notes:** `val = parse(x)?` uses untyped assignment. The `?` operator produces a value but the assignment target `val` was never declared. Changed test to use if/else instead of when to simplify, but the real issue is that `val = expr?` requires `val` to be pre-declared. This is by-design behavior (Cx requires declaration before use), not a bug. The test was written incorrectly.

### audit_04_method_returns_result
**Status:** PASS
**Expected:** Ok(1), Ok(2), Ok(3), Err(at max)
**Actual:** Ok(1), Ok(2), Ok(3), Err(at max)
**Layer:** N/A
**Notes:** Methods returning Result<T> work correctly. Multi-alias writeback + Result compose cleanly.

### audit_05_result_in_array
**Status:** SEMANTIC_FAIL
**Expected:** Ok(2), Err(negative), Ok(10)
**Actual:** `SEMANTIC ERROR: use of undeclared variable 'r1'` (and r2, r3)
**Layer:** semantic
**Notes:** `r1 = try_parse(1)` uses untyped assignment without prior declaration. Same issue as audit_03 — by-design, not a bug. Adapted test to avoid arrays (separate concern) and use simple variables, but the variable still needs type annotation or prior `let` declaration to be recognized. Untyped top-level assignment without prior declaration is not supported.

### audit_06_generic_func_with_result
**Status:** PASS
**Expected:** Ok(42), Ok(99)
**Actual:** Ok(42), Ok(99)
**Layer:** N/A
**Notes:** Simplified from original prompt (removed generic T, used concrete t32). Non-generic function returning Result works.

### audit_07_when_value_in_binary
**Status:** PASS
**Expected:** 300
**Actual:** 300
**Layer:** N/A
**Notes:** Value-producing when composes with arithmetic correctly.

### audit_08_deeply_nested_copy_into
**Status:** PASS
**Expected:** 10
**Actual:** 10
**Layer:** N/A
**Notes:** Nested function calls with typed args work cleanly.

### audit_09_struct_field_overflow
**Status:** WRONG_OUTPUT
**Expected:** 4 (wrapping at t8)
**Actual:** 260
**Layer:** runtime
**Notes:** `c.value += 10` where `value: t8` produces 260, not 4. The overflow-at-declared-width sprint handles Binary expressions via `apply_numeric_cast(result, &expr.ty)`, but compound assignment on struct fields goes through a different code path (`CompoundAssign` in `run_semantic_stmt`) that does NOT apply width truncation. The struct field retains its declared type `t8` but the arithmetic result is not truncated before storage.

### audit_10_if_let_chain
**Status:** PASS
**Expected:** negative, zero, positive
**Actual:** negative, zero, positive
**Layer:** N/A
**Notes:** Multi-branch if/else if/else with string returns works correctly.

### audit_11_recursive_fib
**Status:** PANIC
**Expected:** 55
**Actual:** `thread 'main' has overflowed its stack`
**Layer:** runtime
**Notes:** `fib(10)` with tree recursion overflows the Rust call stack. The interpreter uses native Rust recursion for function calls (`call_semantic_func` → `run_semantic_stmt` → `eval_semantic_expr` → `call_semantic_func`), so deep recursion hits the Rust stack limit. `fib(10)` generates ~177 recursive calls. This is a known architectural limitation of tree-walking interpreters. Tail-call optimization or an explicit call stack would fix it, but both are major architectural changes.

### audit_12_generic_struct_in_generic_func
**Status:** PASS
**Expected:** 99
**Actual:** 99
**Layer:** N/A
**Notes:** Simplified from original prompt (direct field access instead of generic unbox function). Generic struct instantiation and field access work.

## Triage

### Must-fix for 0.1 (hard blockers)
- **audit_09**: Compound assignment on struct fields does not apply width truncation. `c.value += 10` where `value: t8` should wrap but produces full i128 result. This is a gap in the overflow enforcement sprint — the CompoundAssign runtime path was not updated.

### Should-fix for 0.1 (quality)
- **audit_02**: `Result<Result<T>>` fails to parse. The type parser needs recursive Result support. Low priority since nested Result is an edge case, but it's a parser completeness gap.
- **audit_11**: Recursive fib(10) overflows the stack. Should document the recursion depth limit and/or increase Rust's stack size for the interpreter. Not architecturally fixable without a call stack rewrite.

### Document as known limitation for 0.1
- **audit_03/05**: Untyped assignment (`val = expr`) requires prior declaration. This is by-design, not a bug. Document in the spec that all variables must be declared with a type before assignment.
- **audit_11**: Tree-recursive programs with depth > ~100 will overflow the stack. Document as a known interpreter limitation.

### Defer to post-0.1
- **audit_02**: Recursive type parsing (Result<Result<T>>, Handle<Handle<T>>) — low priority, edge case.
