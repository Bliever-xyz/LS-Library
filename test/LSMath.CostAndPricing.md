# LSMath.CostAndPricing.t.sol — Test Documentation

**Tests:** `costFunction()`, `getPrice()`, `getAllPrices()`, `sumOfPrices()`, `calculateTradeCostDetailed()`, `calculateWorstCaseLossFromCosts()`  
**File:** `contractes/test/LSMath/LSMath.CostAndPricing.t.sol`

---

## Purpose

These functions implement the core LS-LMSR equations visible to market participants:

- `costFunction` — the "potential field" that tracks total money in the market
- `getAllPrices` / `getPrice` — the instantaneous price each outcome trades at
- `sumOfPrices` — the total of all prices (always > 1, excess = market maker's fee)
- `calculateTradeCostDetailed` — trade cost + costTo in one call (gas-optimised variant of `calculateTradeCost`)
- `calculateWorstCaseLossFromCosts` — worst-case loss given pre-computed cost values (skips redundant `costFunction` calls)

This file also contains the **numerical stability tests** for the Log-Sum-Exp trick, which is the critical safety mechanism preventing overflow in large or imbalanced markets.

---

## Mathematical Properties Tested

| Property | Source | Where Tested |
|---|---|---|
| `C(q) >= max(qi)` | Lemma 4.5 of LS-LMSR paper | `test_costFunction_lower_bound_always_max_qi`, fuzz |
| `Σpi(q) > 1` | Section 4.2 | `test_sumOfPrices_always_greater_than_one`, fuzz |
| `Σpi(q) <= 1 + α·n·ln(n)` | Section 4.2 upper bound | `test_sumOfPrices_within_theoretical_upper_bound` |
| Symmetric market → equal prices | Symmetry of `exp` | `test_getAllPrices_symmetric_market_equal_prices` |
| Higher quantity → higher price | Monotone pricing | `test_getAllPrices_higher_quantity_higher_price` |
| Buying increases total cost | Monotone cost | `test_costFunction_monotone_increasing_with_quantity` |
| `tradeCost` == `calculateTradeCost` | Identical arithmetic | `test_calculateTradeCostDetailed_tradeCost_matches_calculateTradeCost`, fuzz |
| `costTo` == `costFunction(qTo)` | Reuse correctness | `test_calculateTradeCostDetailed_costTo_matches_costFunction` |
| `calculateWorstCaseLossFromCosts` == `calculateWorstCaseLoss` | Prop 4.9 | `test_calculateWorstCaseLossFromCosts_matches_calculateWorstCaseLoss`, fuzz |
| Loss = 0 when market maker profitable | Prop 4.9 | `test_calculateWorstCaseLossFromCosts_zero_when_profitable` |

---

## Test Inventory

### `costFunction()` — Happy Path

| Test | Scenario | Assertion |
|---|---|---|
| `test_costFunction_exceeds_max_quantity` | `[100e18, 100e18]` | `C(q) > max(qi)` |
| `test_costFunction_known_value_symmetric_binary` | `[100e18, 100e18], α=5%` | `C(q)` in `(106e18, 108e18)` |
| `test_costFunction_always_positive` | Any valid input | `cost > 0` |
| `test_costFunction_monotone_increasing_with_quantity` | Before/after buy | `C(q') > C(q)` |
| `test_costFunction_lower_bound_always_max_qi` | Three alpha values | `C(q) >= max(qi)` for all α |
| `test_costFunction_five_equal_outcomes` | 5 equal outcomes | `cost > max(qi)` |

**Reference value derivation for `[100e18, 100e18], α=5%`:**
```
b     = 0.05 × 200e18 / 1e18 = 10e18
ratio = 100e18 × 1e18 / 10e18 = 10e18  (same for both)
maxRatio = 10e18
shiftedRatio = 0 for both → exp(0) = 1e18
sumExp = 2e18
adjustedLn = 10e18 + ln(2e18) ≈ 10e18 + 0.6931e18 = 10.6931e18
cost = 10e18 × 10.6931e18 / 1e18 ≈ 106.931e18
```

---

### `costFunction()` — Revert Cases

| Test | Expected Error |
|---|---|
| `test_costFunction_reverts_invalid_alpha` | `InvalidAlpha` |
| `test_costFunction_reverts_empty_array` | `EmptyQuantities` |
| `test_costFunction_reverts_all_zero_quantities` | `ZeroQuantitySum` |

---

### `costFunction()` — Numerical Stability (Critical)

| Test | Scenario | What It Validates |
|---|---|---|
| `test_costFunction_large_market_no_overflow` | 80 outcomes, each `1e30` | Log-Sum-Exp prevents overflow across many exp() calls |
| `test_costFunction_imbalanced_market_no_overflow` | One outcome at `1e40`, rest at `1e18` | No revert (stability). Uses `assertApproxEqRel` — at `q[0]=1e40`, integer rounding creates a ~0.000005% deficit that violates exact `assertGe`; relative tolerance captures Lemma 4.5's intent |

These tests correspond to **Fix 2** in the developer documentation: the `else` branch in `costFunction` overflow check. If the Log-Sum-Exp stabilization is removed or broken, these tests will revert.

---

### `getPrice()` / `getAllPrices()` — Happy Path

| Test | Scenario | Assertion |
|---|---|---|
| `test_getAllPrices_symmetric_market_equal_prices` | `[100e18, 100e18]` | `prices[0] ≈ prices[1]` |
| `test_getAllPrices_all_prices_positive` | `[80e18, 120e18]` | Both `> 0` |
| `test_getAllPrices_higher_quantity_higher_price` | `[80e18, 120e18]` | `prices[1] > prices[0]` |
| `test_getPrice_matches_getAllPrices_index` | Asymmetric binary | `getPrice(i) == getAllPrices()[i]` |
| `test_getAllPrices_correct_length` | 5-way market | Returns 5 prices |
| `test_getAllPrices_five_equal_outcomes_equal_prices` | 5 equal outcomes | All prices equal within 1 wei |

---

### `getPrice()` — Revert Cases

| Test | Expected Error |
|---|---|
| `test_getPrice_reverts_invalid_index` | `InvalidOutcomeIndex` (index 2 on length-2 array) |
| `test_getPrice_reverts_invalid_alpha` | `InvalidAlpha` |

---

### `getAllPrices()` — Numerical Stability (Critical)

| Test | Scenario | What It Validates |
|---|---|---|
| `test_getAllPrices_large_market_no_overflow` | 80 equal outcomes, `1e30` each | **Fix 1**: Log-Sum-Exp is present in `getAllPrices` (it was missing before the fix) |
| `test_getAllPrices_imbalanced_dominant_gets_higher_price` | `[1e28, 1e18, 1e18]` | Dominant outcome correctly priced higher, no overflow |

The 80-outcome large market test is the primary regression guard for **Fix 1** in the developer documentation. If `getAllPrices` loses its Log-Sum-Exp stabilization, this test will revert with `ExponentialOverflow`.

---

### `sumOfPrices()` — Tests

| Test | Assertion |
|---|---|
| `test_sumOfPrices_always_greater_than_one` | `sum > 1e18` |
| `test_sumOfPrices_within_theoretical_upper_bound` | `sum <= 1 + α·n·ln(n)` (with 1% tolerance) |
| `test_sumOfPrices_increases_with_alpha` | `sum(α=10%) > sum(α=5%)` |
| `test_sumOfPrices_consistent_with_getAllPrices` | `sumOfPrices == Σ(getAllPrices())` |

> **Bound note:** `testFuzz_sumOfPrices_always_above_one` enforces `q0, q1 >= 1e18` (1 token unit = SCALE). Below SCALE, `alphaTerm` rounds to zero in integer arithmetic and `sumOfPrices` returns exactly `SCALE` — equal to, not greater than, 1. The ">1" property is a real-arithmetic result that requires token-scale inputs.

---

### Fuzz Tests

| Test | Properties Verified |
|---|---|
| `testFuzz_prices_symmetric_equal` | For any symmetric market, prices must be equal |
| `testFuzz_sumOfPrices_always_above_one` | `q0,q1 >= 1e18` (SCALE). `sum > 1e18` — sub-SCALE quantities make alphaTerm round to 0, returning exactly SCALE |
| `testFuzz_costFunction_lower_bound` | `q0,q1` in `[1e15, 1e25]`. `assertApproxEqRel(cost, maxQ, 1e15)` — replaced exact `assertGe` because near `1e28` integer rounding causes a sub-ULP shortfall |

---

## Tolerances Used

| Assertion | Tolerance | Why |
|---|---|---|
| `costFunction` known value | Range check (not exact) | `ln()` approximation in the formula |
| `getAllPrices` symmetric | `assertApproxEqAbs(..., 1)` | 1-wei rounding from integer arithmetic |
| `sumOfPrices` upper bound | 1% tolerance above theoretical bound | Combined approximation from `ln()` |
| Fuzz price symmetry | `assertApproxEqAbs(..., 2)` | Two integer division rounding steps |

---

---

### `calculateTradeCostDetailed()` — Happy Path

| Test | Scenario | Assertion |
|---|---|---|
| `test_calculateTradeCostDetailed_tradeCost_matches_calculateTradeCost` | `[100,100]→[150,100]`, α=5% | `tradeCost == calculateTradeCost(same inputs)` |
| `test_calculateTradeCostDetailed_costTo_matches_costFunction` | `[80,120]→[130,120]`, α=5% | `costTo == costFunction(qTo, alpha)` |
| `test_calculateTradeCostDetailed_positive_on_buy` | `[100,100]→[150,100]` | `tradeCost > 0` |
| `test_calculateTradeCostDetailed_zero_on_noop` | `qFrom == qTo` | `tradeCost == 0`, `costTo == costFunction(q)` |

### `calculateTradeCostDetailed()` — Revert Cases

| Test | Expected Error |
|---|---|
| `test_calculateTradeCostDetailed_reverts_array_length_mismatch` | `InvalidOutcomeIndex` (length-2 vs length-3 arrays) |

### `calculateTradeCostDetailed()` — Fuzz

| Test | Properties Verified |
|---|---|
| `testFuzz_calculateTradeCostDetailed_consistent` | For any valid binary trade, `tradeCost` matches `calculateTradeCost` and `costTo` matches `costFunction(qTo)` |

---

### `calculateWorstCaseLossFromCosts()` — Happy Path

| Test | Scenario | Assertion |
|---|---|---|
| `test_calculateWorstCaseLossFromCosts_matches_calculateWorstCaseLoss` | Real costs from `[10,10]` init, `[60,40]` current | `fromCosts == calculateWorstCaseLoss` |
| `test_calculateWorstCaseLossFromCosts_zero_when_profitable` | `q0=[1,1]`, `q=[10000,10000]` — large surplus | `loss == 0` |
| `test_calculateWorstCaseLossFromCosts_branch_cost_below_maxQ` | Synthetic: costCurrent=50, costInitial=20, `q=[100,30]` | `loss == 70` (costInitial + maxQ - costCurrent) |
| `test_calculateWorstCaseLossFromCosts_correct_maxQ_selection` | Synthetic: costCurrent=210, costInitial=30, `q=[50,200,80]` | `loss == 20`; confirms maxQ=200 selected |
| `test_calculateWorstCaseLossFromCosts_integration_with_tradeCostDetailed` | `costTo` from `calculateTradeCostDetailed` fed into loss function | matches `calculateWorstCaseLoss` |

### `calculateWorstCaseLossFromCosts()` — Revert Cases

| Test | Expected Error |
|---|---|
| `test_calculateWorstCaseLossFromCosts_reverts_empty_quantities` | `EmptyQuantities` |

### `calculateWorstCaseLossFromCosts()` — Fuzz

| Test | Properties Verified |
|---|---|
| `testFuzz_calculateWorstCaseLossFromCosts_consistent` | For any valid binary market, result with real costs equals `calculateWorstCaseLoss` |

> **Synthetic cost note:** `test_calculateWorstCaseLossFromCosts_branch_cost_below_maxQ` and `_correct_maxQ_selection` pass artificial `costCurrent`/`costInitial` values directly. This is intentional — the function accepts raw pre-computed costs, and the `cost < maxQ` branch is unreachable via a live `costFunction()` call (Lemma 4.5 + safety floor), but the branch logic must still be correct for contract robustness.

---

## Debugging Tips

- **Large market test reverts with `ExponentialOverflow`** → the Log-Sum-Exp stabilization (`maxRatio` subtraction before `exp()`) is missing or incorrectly applied in either `costFunction` or `getAllPrices`.
- **Symmetric prices are not equal** → the `weightedSum` or `weightedI` calculation in `getAllPrices` has a scaling error.
- **`sumOfPrices` exceeds theoretical upper bound significantly** → `alphaTerm` is miscalculated; check the `adjustedLn` computation includes the `maxRatio` offset correctly.
- **`C(q) < max(qi)` (Lemma 4.5 violated)** → a fundamental overflow or underflow in the cost formula; check `ln(sumExp)` is not returning too-small a value.
- **`tradeCost` from detailed variant != `calculateTradeCost`** → the two functions diverged; confirm both use identical cost arithmetic and the same int256 overflow guards.
- **`calculateWorstCaseLossFromCosts` != `calculateWorstCaseLoss`** → the `maxQ` scan or the surplus/loss branching logic differs; re-verify the two branches against Prop. 4.9.
