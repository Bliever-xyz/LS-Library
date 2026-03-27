# LSMath.Trading.t.sol — Test Documentation

**Tests:** `calculateTradeCost()`, `calculateWorstCaseLoss()`, `hasOutcomeIndependentProfit()`  
**File:** `contractes/test/LSMath/LSMath.Trading.t.sol`

---

## Purpose

These three functions are the "business logic layer" that market contracts and LP vault contracts will call directly:

- `calculateTradeCost` — how much a trader pays (or receives) for a specific trade
- `calculateWorstCaseLoss` — risk management: maximum the market maker can lose
- `hasOutcomeIndependentProfit` — whether the market maker has guaranteed profit regardless of which outcome resolves

All three build on `costFunction()` results and add the economically meaningful interpretation.

---

## Functions Under Test

### `calculateTradeCost(qFrom, qTo, alpha) → int256`

Formula: `C(q1) - C(q0)`

- Positive → trader pays the market maker (buying shares)
- Negative → market maker pays the trader (selling shares / exit)

### `calculateWorstCaseLoss(q, q0, alpha) → uint256`

Formula: `Loss = C(q0) - C(q) + max(qi)`

The maximum amount the market maker would lose if forced to pay out in the worst-case outcome scenario.

### `hasOutcomeIndependentProfit(q, q0, alpha) → (bool, uint256)`

Formula: `Revenue = C(q) - max(qi) - C(q0)`

Returns `(true, revenue)` if revenue > 0, meaning the market maker profits regardless of which outcome resolves.

---

## Test Inventory

### `calculateTradeCost()` — Happy Path

| Test | Scenario | Assertion |
|---|---|---|
| `test_calculateTradeCost_buying_is_positive` | Buy 50 of outcome 0 from `[100, 100]` state | `tradeCost > 0` |
| `test_calculateTradeCost_zero_trade_zero_cost` | `qFrom == qTo` | `tradeCost == 0` |
| `test_calculateTradeCost_larger_buy_costs_more` | Buy 10 vs buy 50 | `cost(50) > cost(10)` |
| `test_calculateTradeCost_selling_is_negative` | Move from larger to smaller state | `tradeCost < 0` |
| `test_calculateTradeCost_buy_sell_sign_flip` | Buy then simulate sell of same trade | Signs must be opposite |
| `test_calculateTradeCost_multi_outcome_buying` | 5-outcome market, buy one | `tradeCost > 0` |

**Note on antisymmetry:** Buy cost + sell cost for the same trade is NOT exactly zero because LS-LMSR has a variable liquidity parameter `b(q)` — the market is path-dependent. The test only verifies sign flip, not exact cancellation.

---

### `calculateTradeCost()` — Revert Cases

| Test | Input | Expected Error |
|---|---|---|
| `test_calculateTradeCost_reverts_length_mismatch` | `qFrom.length=2`, `qTo.length=3` | `InvalidOutcomeIndex` |
| `test_calculateTradeCost_reverts_invalid_alpha` | `alpha=0` | `InvalidAlpha` |

---

### `calculateTradeCost()` — Fuzz

| Test | Property |
|---|---|
| `testFuzz_calculateTradeCost_buying_always_positive` | For any valid buy (increasing one outcome), `tradeCost > 0` |

---

### `calculateWorstCaseLoss()` — Happy Path

| Test | Scenario | Assertion |
|---|---|---|
| `test_calculateWorstCaseLoss_non_negative` | After a buy trade | Does not revert; result logged |
| `test_calculateWorstCaseLoss_at_initial_state_equals_max_q` | `q == q0` | `loss == max(qi)` |
| `test_calculateWorstCaseLoss_no_revert_after_trade` | After typical trade | Must not revert |

**Why `loss == max(qi)` at initial state?**

At `q == q0`, the formula simplifies:
```
Loss = C(q0) - C(q0) + max(qi) = max(qi)
```
Since `C(q) >= max(qi)` (Lemma 4.5), the conditional `if (costCurrent < maxQ)` branch is taken:
```
worstCaseLoss = costInitial + maxQ - costCurrent = C(q0) + max(qi) - C(q0) = max(qi)
```

---

### `calculateWorstCaseLoss()` — Revert Cases

| Test | Input | Expected Error |
|---|---|---|
| `test_calculateWorstCaseLoss_reverts_empty_current` | `quantities = []` | `EmptyQuantities` |
| `test_calculateWorstCaseLoss_reverts_empty_initial` | `initialQuantities = []` | `EmptyQuantities` |
| `test_calculateWorstCaseLoss_reverts_length_mismatch` | Length 2 vs length 3 | `ArrayLengthMismatch` |

---

### `hasOutcomeIndependentProfit()` — Happy Path

| Test | Scenario | Assertion |
|---|---|---|
| `test_hasOutcomeIndependentProfit_no_profit_at_initial_state` | `q == q0` | `hasProfit == false`, `profit == 0` |
| `test_hasOutcomeIndependentProfit_after_large_volume` | 10× volume increase | Logs result; validates internal consistency |
| `test_hasOutcomeIndependentProfit_consistency` | After one buy | `hasProfit=true → profit > 0`, `hasProfit=false → profit == 0` |

**Why no profit at initial state?**

At `q == q0`, revenue = `C(q0) - max(q0) - C(q0) = -max(q0) < 0`. So the market maker is in a loss position initially, which is expected — they collect fees as trading volume accumulates.

---

### `hasOutcomeIndependentProfit()` — Revert Cases

| Test | Expected Error |
|---|---|
| `test_hasOutcomeIndependentProfit_reverts_empty_current` | `EmptyQuantities` |
| `test_hasOutcomeIndependentProfit_reverts_length_mismatch` | `ArrayLengthMismatch` |

---

### `hasOutcomeIndependentProfit()` — Fuzz

| Test | Property |
|---|---|
| `testFuzz_hasOutcomeIndependentProfit_consistent_return` | If `hasProfit == true` then `profit > 0`; if `false` then `profit == 0` |

---

## Key Economic Invariants Verified

| Invariant | Verified By |
|---|---|
| Buying always costs money (positive cost) | `test_calculateTradeCost_buying_is_positive` + fuzz |
| Selling returns money (negative cost) | `test_calculateTradeCost_selling_is_negative` |
| No free lunch: zero trade = zero cost | `test_calculateTradeCost_zero_trade_zero_cost` |
| MM has no profit at market open | `test_hasOutcomeIndependentProfit_no_profit_at_initial_state` |
| Return type consistency: profit=0 when hasProfit=false | All consistency tests + fuzz |

---
