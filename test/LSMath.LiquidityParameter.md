# LSMath.LiquidityParameter.t.sol — Test Documentation

**Tests:** `liquidityParameter()`, `validateAlpha()`, `validateQuantities()`  
**File:** `contractes/test/LSMath/LSMath.LiquidityParameter.t.sol`

---

## Purpose

`liquidityParameter()` computes `b(q) = α × Σqi` — the single most important intermediate value in all LS-LMSR calculations. Every downstream function (`costFunction`, `getAllPrices`, etc.) calls it first. If `b` is wrong, every price and cost is wrong.

The two validation helpers (`validateAlpha`, `validateQuantities`) are tested here because they share the same input-domain knowledge: what alpha values are safe and what quantity arrays are structurally valid.

---

## Formula Under Test

```
b(q) = (alpha × sumQ) / SCALE
sumQ = q[0] + q[1] + ... + q[n-1]
```

Where:
- `SCALE = 1e18` (18-decimal fixed-point)
- `MIN_ALPHA = 1e12` (0.000001)
- `MAX_ALPHA = 2e17` (0.2 = 20%)
- `MAX_OUTCOMES = 100`

---

## Test Inventory

### `liquidityParameter()` — Happy Path

| Test | Scenario | Assertion |
|---|---|---|
| `test_liquidityParameter_symmetric_binary` | `[100e18, 100e18], α=5%` | `sumQ=200e18`, `b=10e18` |
| `test_liquidityParameter_asymmetric_binary` | `[200e18, 100e18], α=5%` | `sumQ=300e18`, `b=15e18` |
| `test_liquidityParameter_formula_invariant` | Arbitrary inputs | `b == (alpha * sumQ) / SCALE` always |
| `test_liquidityParameter_min_alpha` | `alpha = MIN_ALPHA` | `b > 0` (no zero result) |
| `test_liquidityParameter_max_alpha` | `alpha = MAX_ALPHA = 20%` | `b = 40e18` |
| `test_liquidityParameter_five_outcomes` | 5 equal outcomes | Correct sum and b |
| `test_liquidityParameter_scales_with_alpha` | Double alpha → double b | Linear scaling verified |
| `test_liquidityParameter_scales_with_quantity` | Double all quantities → double b | Linear scaling verified |
| `test_liquidityParameter_one_nonzero_quantity` | `[0, 100e18]` | Zero is a valid element; b computed from non-zero |

**Why the scaling tests?** LS-LMSR is designed to be homogeneous of degree 1. If `b` doesn't scale linearly with α or with Σqi, the price formula breaks.

---

### `liquidityParameter()` — Revert Cases

| Test | Input | Expected Error |
|---|---|---|
| `test_liquidityParameter_reverts_alpha_below_min` | `alpha = MIN_ALPHA - 1` | `InvalidAlpha` |
| `test_liquidityParameter_reverts_alpha_zero` | `alpha = 0` | `InvalidAlpha` |
| `test_liquidityParameter_reverts_alpha_above_max` | `alpha = MAX_ALPHA + 1` | `InvalidAlpha` |
| `test_liquidityParameter_reverts_empty_array` | `quantities.length == 0` | `EmptyQuantities` |
| `test_liquidityParameter_reverts_array_too_large` | `quantities.length == 101` | `InvalidOutcomeIndex` |
| `test_liquidityParameter_reverts_all_zeros` | `[0, 0]` | `ZeroQuantitySum` |

---

### `liquidityParameter()` — Fuzz

| Test | Inputs Fuzzed | Property Verified |
|---|---|---|
| `testFuzz_liquidityParameter_formula_holds` | `q0`, `q1`, `alpha` | `b == (alpha * sumQ) / SCALE` and `sumQ == q0 + q1` |

The fuzz test skips edge cases where `b` would round to zero due to tiny quantities, to avoid false failures from precision loss rather than logic bugs.

---

### `validateAlpha()` — Tests

| Test | Input | Expected |
|---|---|---|
| `test_validateAlpha_returns_true_for_valid_alpha` | `5e16`, `MIN_ALPHA`, `MAX_ALPHA`, `1e17` | `true` |
| `test_validateAlpha_returns_false_for_zero` | `0` | `false` |
| `test_validateAlpha_returns_false_below_min` | `MIN_ALPHA - 1` | `false` |
| `test_validateAlpha_returns_false_above_max` | `MAX_ALPHA + 1` | `false` |
| `testFuzz_validateAlpha_consistent_with_liquidityParameter` | random `alpha` | If `validateAlpha(α)==true`, `liquidityParameter` must not revert with `InvalidAlpha`; if false, it must |

The consistency fuzz test is the most important: it proves the two functions agree on what "valid alpha" means.

---

### `validateQuantities()` — Tests

| Test | Input | Expected |
|---|---|---|
| `test_validateQuantities_valid_binary_market` | `[100e18, 100e18]` | `true` |
| `test_validateQuantities_valid_one_nonzero` | `[0, 100e18]` | `true` (sum > 0, length >= 2) |
| `test_validateQuantities_invalid_single_element` | `[100e18]` | `false` (need at least 2 outcomes) |
| `test_validateQuantities_invalid_empty_array` | `[]` | `false` |
| `test_validateQuantities_invalid_all_zeros` | `[0, 0]` | `false` |
| `test_validateQuantities_invalid_exceeds_max_outcomes` | 101 elements | `false` |
| `test_validateQuantities_valid_exactly_max_outcomes` | exactly 100 elements | `true` |
| `test_validateQuantities_valid_multi_outcome` | 10 elements | `true` |

---

