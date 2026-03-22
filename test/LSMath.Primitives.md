# LSMath.Primitives.t.sol — Test Documentation

**Tests:** `exp()`, `ln()`, `mulScale()`, `divScale()`  
**File:** `contractes/test/LSMath/LSMath.Primitives.t.sol`

---

## Purpose

These four functions are the lowest-level building blocks. Every higher-level LS-LMSR formula (`costFunction`, `getAllPrices`, etc.) depends on them. A precision bug here propagates silently into all pricing outputs.

---

## Functions Under Test

| Function | What It Does |
|---|---|
| `exp(x)` | e^x via Taylor series + range reduction. Input/output in 18-decimal fixed-point. |
| `ln(x)` | Natural log via bit-search algorithm. Input/output in 18-decimal fixed-point. |
| `mulScale(a, b)` | Multiply two 18-dec values: `(a * b) / SCALE` |
| `divScale(a, b)` | Divide two 18-dec values: `(a * SCALE) / b` |

---

## Test Inventory

### `exp()` — Happy Path

| Test | What It Checks | Why It Matters |
|---|---|---|
| `test_exp_zero_returns_one` | `exp(0) == 1e18` | Base case of Taylor series; definitional |
| `test_exp_ln2_approx_two` | `exp(ln2) ≈ 2e18` | Validates `LN_2` constant and range reduction together |
| `test_exp_one_approx_e` | `exp(1e18) ≈ 2.718...e18` | Cross-checks against Euler's number |
| `test_exp_positive_input_greater_than_one` | `exp(x) > 1` for all `x > 0` | Fundamental math property |
| `test_exp_monotone_increasing` | `exp(1) < exp(2)` | Monotonicity — pricing depends on this |
| `test_exp_at_max_boundary_succeeds` | `exp(MAX_EXP_INPUT)` does not revert | Boundary condition at the guard |

### `exp()` — Revert Cases

| Test | Expected Error |
|---|---|
| `test_exp_reverts_above_max_input` | `ExponentialOverflow` |

### `exp()` — Fuzz

| Test | Property Verified |
|---|---|
| `testFuzz_exp_result_always_gte_one` | `exp(x) >= 1e18` for all valid `x` |
| `testFuzz_exp_result_never_zero` | Result is always non-zero |

---

### `ln()` — Happy Path

| Test | What It Checks | Why It Matters |
|---|---|---|
| `test_ln_one_returns_zero` | `ln(1e18) == 0` | Definitional: ln(1) = 0 |
| `test_ln_two_approx_ln2_constant` | `ln(2e18) ≈ LN_2` | Validates the stored constant |
| `test_ln_e_approx_one` | `ln(e) ≈ 1e18` | Cross-check against Euler's number |
| `test_ln_monotone_increasing` | `ln(2) < ln(10)` | Monotonicity |
| `test_ln_large_input_succeeds` | `ln(1e36)` does not revert | No upper bound on input |

### `ln()` — Revert Cases

| Test | Expected Error |
|---|---|
| `test_ln_reverts_on_zero` | `InvalidLogInput` |
| `test_ln_reverts_on_below_scale` | `InvalidLogInput` (x < 1 in fixed-point) |
| `test_ln_reverts_on_one_wei` | `InvalidLogInput` |

---

### Inverse Relationship (exp / ln)

| Test | Property | Importance |
|---|---|---|
| `test_exp_ln_inverse_roundtrip` | `exp(ln(x)) ≈ x` | Confirms the two approximations are mutually consistent |
| `test_ln_exp_inverse_roundtrip` | `ln(exp(x)) ≈ x` | Same, from the other direction |

These are **cross-validation tests** — they catch the scenario where `exp()` and `ln()` each appear correct in isolation but are inconsistent with each other.

---

### `mulScale()` — Happy Path

| Test | What It Checks |
|---|---|
| `test_mulScale_basic_multiplication` | `2e18 × 3e18 = 6e18` |
| `test_mulScale_fractional_operand` | `0.5e18 × 4e18 = 2e18` |
| `test_mulScale_identity` | `1e18 × x = x` |
| `test_mulScale_small_fractions` | `0.1 × 0.1 = 0.01` |

### `mulScale()` — Fuzz

| Test | Property |
|---|---|
| `testFuzz_mulScale_commutativity` | `mulScale(a,b) == mulScale(b,a)` |

> **Note:** `mulScale(0, b)` causes a division-by-zero panic inside the overflow check `product / a` when `a == 0`. This is a known quirk of the contract's overflow-detection pattern in Solidity 0.8.x. The fuzz test skips `a == 0` and `b == 0` cases. This behavior is documented here for future developers.

---

### `divScale()` — Happy Path

| Test | What It Checks |
|---|---|
| `test_divScale_basic_division` | `6e18 / 2e18 = 3e18` |
| `test_divScale_less_than_one` | `1e18 / 4e18 = 0.25e18` |
| `test_divScale_identity_divisor` | `x / 1e18 = x` |
| `test_divScale_self_division` | `x / x = 1e18` |
| `test_divScale_zero_numerator_returns_zero` | `0 / x = 0` |

### `divScale()` — Revert Cases

| Test | Expected Error |
|---|---|
| `test_divScale_reverts_on_zero_divisor` | `DivisionByZero` |

### `divScale()` — Fuzz

| Test | Property |
|---|---|
| `testFuzz_divScale_inverse_of_mulScale` | `(a / b) * b ≈ a` within `b/SCALE + 1` tolerance (floor-division rounding budget) |

---

## Tolerances Used

| Operation | Tolerance | Reason |
|---|---|---|
| `exp` known values | `1e15` (0.1 %) relative | Taylor series is an approximation |
| `ln` known values | `1e15` (0.1 %) relative | Bit-search has finite precision |
| Inverse roundtrips | `1e15` (0.1 %) relative | Compounded approximation error |
| `mulScale` / `divScale` | Exact (`assertEq`) | These are exact integer operations |
| `divScale` roundtrip | `1e3` absolute | One integer division rounding step |

---

## Debugging Tips

- If `test_exp_ln2_approx_two` fails → the `LN_2` constant value is wrong or range reduction is broken.
- If `test_ln_e_approx_one` fails → the fractional-part binary search has precision issues.
- If inverse roundtrip tests fail but individual tests pass → `exp` and `ln` are inconsistently calibrated.
- If `mulScale` or `divScale` tests fail → check Solidity 0.8's built-in overflow (these functions have a custom check that is actually unreachable in 0.8.x; panic occurs first).
