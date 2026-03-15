# LSMath — Developer Reference

**Contract**: `LSMath.sol`  
**Solidity**: `0.8.31` (strict, non-floating)  
**Type**: `library` (stateless, `internal pure` functions only)  
**Audience**: Developers analyzing, extending, debugging, or writing tests for the LS-LMSR math library.

---

## Table of Contents

1. [Library Structure](#1-library-structure)
2. [Constants](#2-constants)
3. [Error Catalogue](#3-error-catalogue)
4. [Fixed-Point Arithmetic Conventions](#4-fixed-point-arithmetic-conventions)
5. [Core Function Implementations](#5-core-function-implementations)
   - [5.1 liquidityParameter()](#51-liquidityparameter)
   - [5.2 costFunction()](#52-costfunction)
   - [5.3 getAllPrices()](#53-getallprices)
   - [5.4 getPrice()](#54-getprice)
   - [5.5 calculateTradeCost()](#55-calculatetradecost)
   - [5.6 sumOfPrices()](#56-sumofprices)
6. [Mathematical Primitive Implementations](#6-mathematical-primitive-implementations)
   - [6.1 exp()](#61-exp)
   - [6.2 ln()](#62-ln)
   - [6.3 mulScale()](#63-mulscale)
   - [6.4 divScale()](#64-divscale)
7. [Utility Function Implementations](#7-utility-function-implementations)
   - [7.1 calculateWorstCaseLoss()](#71-calculateworstcaseloss)
   - [7.2 hasOutcomeIndependentProfit()](#72-hasoutcomeindependentprofit)
8. [Validation Helpers](#8-validation-helpers)
9. [Numerical Stability: Log-Sum-Exp Trick](#9-numerical-stability-log-sum-exp-trick)
10. [Precision and Rounding Policy](#10-precision-and-rounding-policy)
11. [Gas Considerations](#11-gas-considerations)
12. [Known Limitations](#12-known-limitations)
13. [Testing Guidance](#13-testing-guidance)

---

## 1. Library Structure

`LSMath` is declared as a Solidity `library`. All functions are `internal`, meaning they are inlined (via `JUMP`) into any contract that imports the library. There is no deployment address, no proxy, and no external call overhead.

**File layout (logical sections):**

```
ERRORS              — 12 custom error types
CONSTANTS           — 7 named constants
CORE LS-LMSR        — liquidityParameter, costFunction, getPrice, getAllPrices,
                       calculateTradeCost, sumOfPrices
MATHEMATICAL HELPERS — exp, ln
FIXED-POINT         — mulScale, divScale
VALIDATION          — validateQuantities, validateAlpha
UTILITY             — calculateWorstCaseLoss, hasOutcomeIndependentProfit
```

**Compiler choice rationale (`pragma solidity 0.8.31;`):**
- Strict (non-floating) pragma ensures deterministic bytecode across all builds and audit snapshots
- 0.8.x enables built-in checked arithmetic (overflow/underflow reverts by default), except inside `unchecked` blocks
- Explicit `unchecked { ++i; }` blocks in loops are safe because the loop counter is bounded by `length`, which is itself bounded by `MAX_OUTCOMES = 100`

---

## 2. Constants

All constants are `uint256 internal constant`. They exist only in the compiled artifact — no storage slots consumed.

| Constant | Raw Value | Decoded Value | Purpose |
|---|---|---|---|
| `SCALE` | `1000000000000000000` | `1e18` | Fixed-point base. All quantities, prices, alpha, and results are expressed in multiples of this. |
| `MIN_ALPHA` | `1000000000000` | `1e12` (= 0.000001) | Lower bound on α. Below this, the `b` numerics collapse and prices become undefined. |
| `MAX_ALPHA` | `200000000000000000` | `2e17` (= 0.2) | Upper bound on α. Corresponds to a 20% maximum commission. |
| `MAX_OUTCOMES` | `100` | 100 | Hard cap on array length in loops. Prevents unbounded gas use. |
| `LN_2` | `693147180559945309` | ln(2) · 1e18 | Used in `exp()` range reduction and `ln()` binary search. Precision: 18 decimal digits. |
| `MAX_EXP_INPUT` | `135305999368893231589` | ≈ 135.306 · 1e18 | Maximum safe argument to `exp()`. Derived from: `ln(2^255 / 1e18)` ≈ 135.3. |
| `MIN_TERM` | `100` | 100 (raw units) | Taylor series early-exit threshold. When the current term drops below this raw value, further terms contribute less than ~1e-16 relative error. |

**On `MAX_EXP_INPUT`:** The Taylor series result after range reduction lives in approximately `[1e18, 2e18)`. The range reduction applies `shift` left-bit-shifts to this result. The maximum shift before overflow is governed by: `sum << shift ≤ type(uint256).max`, i.e., `sum · 2^shift ≤ 2^256 - 1`. With `sum ≈ 2e18 ≈ 2^61`, the maximum safe shift is `255 - 61 = 194`. The number of range-reduction steps for input `x` is `floor(x / LN_2)`. Setting `MAX_EXP_INPUT ≈ 135.3e18` limits steps to approximately 195, which is the safe boundary.

---

## 3. Error Catalogue

All errors are custom (`error` keyword, Solidity 0.8.4+). Each encodes as a 4-byte selector in the revert data, enabling programmatic handling by callers.

| Error | Thrown By | Condition |
|---|---|---|
| `InvalidAlpha()` | `liquidityParameter` | `alpha < MIN_ALPHA` or `alpha > MAX_ALPHA` |
| `EmptyQuantities()` | `liquidityParameter`, `calculateWorstCaseLoss`, `hasOutcomeIndependentProfit` | `quantities.length == 0` |
| `NegativeQuantity()` | (reserved) | Declared but not currently thrown; `uint256` type enforces non-negativity by construction |
| `ZeroQuantitySum()` | `liquidityParameter` | `Σqᵢ == 0` (all quantities are zero) |
| `ArithmeticOverflow()` | `liquidityParameter`, `costFunction`, `getAllPrices`, `sumOfPrices`, `calculateTradeCost` | Integer overflow detected in accumulation |
| `MultiplicationOverflow()` | `mulScale`, `divScale` | `a * b` overflows `uint256` (detected via division check) |
| `DivisionByZero()` | `divScale` | `b == 0` passed as denominator |
| `InvalidMarketState()` | `costFunction`, `getAllPrices` | `sumExp == 0` after Log-Sum-Exp accumulation, or `denominator == 0` in price computation |
| `InsufficientOutcomes()` | (reserved) | Declared; outcome count < 2 is handled by `validateQuantities` returning `false` rather than reverting |
| `ArrayLengthMismatch()` | `calculateWorstCaseLoss`, `hasOutcomeIndependentProfit` | Input arrays have different lengths |
| `InvalidLogInput()` | `ln` | `x == 0` or `x < SCALE` (logarithm of values less than 1 would be negative; not supported) |
| `ExponentialOverflow()` | `exp` | `x > MAX_EXP_INPUT`, or final bit-shift would overflow `uint256` |

**Note on `NegativeQuantity` and `InsufficientOutcomes`:** These errors are declared in the interface but not currently thrown by any execution path. `NegativeQuantity` is structurally impossible with `uint256` inputs. `InsufficientOutcomes` is reserved for future use; the current code handles this by returning `false` from `validateQuantities()`.

**Catching errors in consuming contracts:**

```solidity
try LSMath.costFunction(quantities, alpha) returns (uint256 cost) {
    // success
} catch (bytes memory reason) {
    bytes4 selector = bytes4(reason);
    if (selector == LSMath.InvalidAlpha.selector) { ... }
    if (selector == LSMath.ExponentialOverflow.selector) { ... }
    // etc.
}
```

---

## 4. Fixed-Point Arithmetic Conventions

All values in this library use **18-decimal fixed-point** representation:

- `1.0` is represented as `1e18 = 1000000000000000000`
- `0.5` is represented as `5e17`
- `2.5` is represented as `25e17`

**Addition/subtraction:** No scaling needed. `(a + b)` in fixed-point = `a + b`.

**Multiplication:** `a * b` in 18-decimal produces a value with 36 decimal places. Divide by `SCALE` to return to 18-decimal: `(a * b) / SCALE`. Use `mulScale()` for this.

**Division:** To divide `a` by `b` in fixed-point: `(a * SCALE) / b`. Use `divScale()` for this.

**Rounding policy:** Both `mulScale` and `divScale` use **round-half-up** (also called round-to-nearest) rather than truncation:

```
mulScale: (a * b + SCALE/2) / SCALE
divScale: (a * SCALE + b/2) / b
```

This halves the expected rounding error per operation and eliminates the systematic downward bias that pure truncation introduces. In long chains of fixed-point operations (as occur in `getAllPrices` and `costFunction`), unbiased rounding is important for accuracy.

**Overflow detection in multiplication:** Both `mulScale` and `divScale` detect overflow using the standard post-multiplication division check:

```solidity
uint256 product = a * b;
if (product / a != b) revert MultiplicationOverflow();
```

This works because in Solidity 0.8.x, `a * b` silently wraps on overflow (since we are in checked context, actually it *reverts* — but in `mulScale` the multiplication itself is the product of user-supplied values, not scaled, and we want a specific error). Wait — actually in `mulScale`, the multiplication `a * b` will revert with a panic if it overflows in checked context. The `product / a != b` pattern is used in `divScale` where `scaled = a * SCALE` — but since `SCALE` is a constant and `a` is potentially large, this multiplication could overflow. The check verifies it did not.

---

## 5. Core Function Implementations

### 5.1 `liquidityParameter`

**Signature:** `function liquidityParameter(uint256[] memory quantities, uint256 alpha) internal pure returns (uint256 b, uint256 sumQ)`

**Step-by-step:**

1. **Validate inputs:**
   ```solidity
   if (alpha < MIN_ALPHA || alpha > MAX_ALPHA) revert InvalidAlpha();
   if (quantities.length == 0) revert EmptyQuantities();
   if (quantities.length > MAX_OUTCOMES) revert InvalidOutcomeIndex();
   ```

2. **Accumulate `Σqᵢ` with overflow detection:**
   ```solidity
   for (uint256 i = 0; i < length; ) {
       uint256 qi = quantities[i];
       sumQ += qi;
       if (sumQ < qi) revert ArithmeticOverflow(); // post-addition overflow check
       unchecked { ++i; }
   }
   if (sumQ == 0) revert ZeroQuantitySum();
   ```
   The post-addition check `sumQ < qi` catches uint256 wraparound that would otherwise silently corrupt the sum. The `unchecked { ++i; }` is safe because `i < length ≤ MAX_OUTCOMES = 100`.

3. **Compute `b = α · Σqᵢ / SCALE`:**
   ```solidity
   b = (alpha * sumQ) / SCALE;
   ```
   Both `alpha` and `sumQ` are in 18-decimal fixed-point. Multiplying them produces a 36-decimal value; dividing by `SCALE` returns to 18-decimal.

4. **Clamp to minimum 1:**
   ```solidity
   if (b == 0) b = 1;
   ```
   This is triggered when `alpha * sumQ < SCALE`, i.e., when quantities are very small relative to `α`. For example, with `alpha = MIN_ALPHA = 1e12` and `sumQ = 1e5`, the product `1e12 * 1e5 = 1e17 < 1e18`, so `b = 0` before clamping. Clamping to 1 ensures `b ≥ 1` at all times, preventing division-by-zero in downstream functions. Mathematically, `b = 1 wei` represents maximum price sensitivity, which is the correct behavior for an extremely thinly-funded market.

**Why the tuple return:** `sumQ` is needed by `getAllPrices()` to compute the price denominator `(sumQ * sumExp) / SCALE`. Previously `sumQ` was recomputed internally in those callers. Returning the tuple eliminates this redundant loop (saves ~300–500 gas per price query for typical outcome counts).

---

### 5.2 `costFunction`

**Signature:** `function costFunction(uint256[] memory quantities, uint256 alpha) internal pure returns (uint256 cost)`

The cost function implements `C(q) = b(q) · ln(Σ exp(qᵢ/b(q)))` using the Log-Sum-Exp trick for numerical stability (see Section 9 for detailed treatment of the trick).

**Step-by-step:**

1. **Obtain b:** `(uint256 b, ) = liquidityParameter(quantities, alpha);`

2. **Find `maxRatio` and `maxQ` in a single pass:**
   ```solidity
   uint256 maxRatio = 0;
   uint256 maxQ = 0;
   for (uint256 i = 0; i < length; ) {
       uint256 qi = quantities[i];
       if (qi > maxQ) maxQ = qi;
       uint256 ratio = (qi * SCALE) / b;  // qᵢ/b in 18-decimal
       if (ratio > maxRatio) maxRatio = ratio;
       unchecked { ++i; }
   }
   ```
   `maxRatio` is `max(qᵢ/b)`, used to shift the Log-Sum-Exp computation. `maxQ` is `max(qᵢ)`, used for the invariant clamp in step 5.

3. **Accumulate `Σ exp(qᵢ/b - maxRatio)` (stabilized):**
   ```solidity
   uint256 sumExp = 0;
   for (uint256 i = 0; i < length; ) {
       uint256 ratio = (quantities[i] * SCALE) / b;
       uint256 term;
       if (ratio >= maxRatio) {
           term = exp(ratio - maxRatio);   // exp(0) to exp(small positive)
       } else {
           uint256 diff = maxRatio - ratio;
           uint256 expDiff = exp(diff);
           term = (SCALE * SCALE) / expDiff;  // exp(-diff) = 1/exp(diff)
       }
       sumExp += term;
       if (sumExp < term) revert ArithmeticOverflow();
       unchecked { ++i; }
   }
   if (sumExp == 0) revert InvalidMarketState();
   ```
   The branch for `ratio < maxRatio` computes `exp(ratio - maxRatio) = exp(-(maxRatio - ratio)) = 1 / exp(maxRatio - ratio)` in fixed-point as `SCALE² / exp(diff)`. The `SCALE²` numerator is necessary because both SCALE factors in `SCALE * SCALE` would otherwise disappear in integer division.

4. **Compute the adjusted log-sum:**
   ```solidity
   uint256 lnSum = ln(sumExp);
   uint256 adjustedLn = maxRatio + lnSum;
   // This equals: maxRatio + ln(Σ exp(qᵢ/b - maxRatio))
   //            = ln(exp(maxRatio)) + ln(Σ exp(qᵢ/b - maxRatio))
   //            = ln(Σ exp(qᵢ/b))      [by Log-Sum-Exp identity]
   cost = (b * adjustedLn) / SCALE;
   ```

5. **Enforce `C(q) ≥ max(qᵢ)` invariant:**
   ```solidity
   if (cost < maxQ) cost = maxQ;
   ```
   This is a defensive clamp. In exact arithmetic, Lemma 4.5 guarantees `C(q) ≥ max(qᵢ)`. In fixed-point, truncation in the Log-Sum-Exp computation can occasionally produce a result slightly below `max(qᵢ)` for highly imbalanced markets. Clamping enforces the invariant and prevents downstream solvency calculation errors.

---

### 5.3 `getAllPrices`

**Signature:** `function getAllPrices(uint256[] memory quantities, uint256 alpha) internal pure returns (uint256[] memory prices)`

This is the most complex function in the library. It computes the full LS-LMSR price vector per the formula from Section 4.1 of the paper.

**Complete execution flow:**

**Phase 1 — Setup:**
```solidity
(uint256 b, uint256 sumQ) = liquidityParameter(quantities, alpha);
```

**Phase 2 — Find `maxRatio` for Log-Sum-Exp stabilization:**
```solidity
uint256 maxRatio = 0;
for (uint256 i = 0; i < length; ) {
    uint256 ratio = (quantities[i] * SCALE) / b;
    if (ratio > maxRatio) maxRatio = ratio;
    unchecked { ++i; }
}
```

**Phase 3 — Compute shifted `expValues[]` and `sumExp`:**
```solidity
uint256[] memory expValues = new uint256[](length);
uint256 sumExp = 0;
for (uint256 i = 0; i < length; ) {
    uint256 ratio = (quantities[i] * SCALE) / b;
    uint256 expValue;
    if (ratio >= maxRatio) {
        expValue = exp(ratio - maxRatio);
    } else {
        uint256 diff = maxRatio - ratio;
        expValue = (SCALE * SCALE) / exp(diff);
    }
    expValues[i] = expValue;
    sumExp += expValue;
    if (sumExp < expValue) revert ArithmeticOverflow();
    unchecked { ++i; }
}
if (sumExp == 0) revert InvalidMarketState();
```
The `expValues` array stores the stabilized `exp(qᵢ/b - maxRatio)` for each outcome. These are reused in both the weighted-sum pass and the per-outcome price pass.

**Phase 4 — Compute the shared `alphaTerm`:**
```solidity
uint256 lnSum = ln(sumExp);
uint256 adjustedLn = maxRatio + lnSum;  // = ln(Σ exp(qᵢ/b))
uint256 alphaTerm = (alpha * adjustedLn) / SCALE;  // = α · ln(Σ exp(qᵢ/b))
```
`alphaTerm` is the same for all outcomes and represents the first term of the price formula.

**Phase 5 — Compute `weightedSum` (shared across all outcomes):**
```solidity
uint256 weightedSum = 0;
for (uint256 i = 0; i < length; ) {
    weightedSum += (quantities[i] * expValues[i]) / SCALE;
    unchecked { ++i; }
}
```
`weightedSum` = `Σⱼ (qⱼ · exp(qⱼ/b - maxRatio)) / SCALE`. Note: because all `expValues` are shifted by `exp(-maxRatio)`, this is a stabilized version of `Σⱼ qⱼ · exp(qⱼ/b)`.

**Phase 6 — Compute `denominator`:**
```solidity
uint256 denominator = (sumQ * sumExp) / SCALE;
if (denominator == 0) revert InvalidMarketState();
```
`denominator` = `Σqⱼ · Σ exp(qⱼ/b - maxRatio) / SCALE`.

**Phase 7 — Per-outcome price computation:**
```solidity
for (uint256 i = 0; i < length; ) {
    uint256 weightedI = (sumQ * expValues[i]) / SCALE;
    // weightedI = Σqⱼ · exp(qᵢ/b - maxRatio) / SCALE

    int256 numerator = int256(weightedI) - int256(weightedSum);
    int256 term2 = (numerator * int256(SCALE)) / int256(denominator);
    // term2 = (weightedI - weightedSum) / denominator

    int256 priceInt = int256(alphaTerm) + term2;
    if (priceInt <= 0) revert ArithmeticOverflow();
    prices[i] = uint256(priceInt);
    unchecked { ++i; }
}
```

`term2` is the outcome-specific adjustment. It is positive for the highest-quantity outcome and negative for others (since `weightedI > weightedSum` iff `exp(qᵢ/b)` exceeds the weighted average). The signed arithmetic is necessary here because `term2` can be negative.

**Important numeric note on stabilization consistency:** Because all `expValues` are shifted by the same factor `exp(-maxRatio)`, the ratio `weightedI / denominator` is unchanged (the `exp(-maxRatio)` factor cancels in numerator and denominator). The `alphaTerm`, however, uses `adjustedLn = maxRatio + lnSum`, which correctly accounts for the shift. The final prices are therefore mathematically equivalent to the unstabilized formula.

---

### 5.4 `getPrice`

**Signature:** `function getPrice(uint256[] memory quantities, uint256 outcomeIndex, uint256 alpha) internal pure returns (uint256 price)`

A thin wrapper:
```solidity
if (outcomeIndex >= quantities.length) revert InvalidOutcomeIndex();
uint256[] memory allPrices = getAllPrices(quantities, alpha);
price = allPrices[outcomeIndex];
```

This delegates to `getAllPrices()` to avoid code duplication and ensure both functions use identical logic. The gas overhead of computing unused prices is acceptable for single-outcome queries; callers that need all prices should call `getAllPrices()` directly.

---

### 5.5 `calculateTradeCost`

**Signature:** `function calculateTradeCost(uint256[] memory quantitiesFrom, uint256[] memory quantitiesTo, uint256 alpha) internal pure returns (int256 tradeCost)`

```solidity
if (quantitiesFrom.length != quantitiesTo.length) revert InvalidOutcomeIndex();
uint256 costFrom = costFunction(quantitiesFrom, alpha);
uint256 costTo = costFunction(quantitiesTo, alpha);

// Bounds check before int256 cast
if (costTo > uint256(type(int256).max) || costFrom > uint256(type(int256).max))
    revert ArithmeticOverflow();

tradeCost = int256(costTo) - int256(costFrom);
```

The `int256` cast bound check is necessary because `costFunction` returns `uint256`. For markets with enormous outstanding quantities (close to `type(uint256).max / 2`), casting without checking would silently produce a wrong signed value. In practice, quantities that large would trigger `ArithmeticOverflow` in `liquidityParameter` first, but the check is included for defense-in-depth.

---

### 5.6 `sumOfPrices`

```solidity
uint256[] memory prices = getAllPrices(quantities, alpha);
for (uint256 i = 0; i < prices.length; ) {
    sum += prices[i];
    if (sum < prices[i]) revert ArithmeticOverflow();
    unchecked { ++i; }
}
```

Straightforward sum with overflow check. Primarily useful for diagnostics. The result should theoretically satisfy `sum ≥ 1e18` (prices sum to at least 1.0 in fixed-point).

---

## 6. Mathematical Primitive Implementations

### 6.1 `exp`

**Signature:** `function exp(uint256 x) internal pure returns (uint256 result)`

Computes `eˣ` for a fixed-point input `x` (18-decimal).

**Algorithm: Range Reduction + Taylor Series**

The key problem with computing `eˣ` for large `x` in fixed-point is that `eˣ` grows faster than `uint256` can represent for `x > ~135`. The approach is to express `x = k · ln2 + r` where `r ∈ [0, ln2)` and then use `eˣ = 2^k · e^r`. Since `e^r ∈ [1, 2)` for `r ∈ [0, ln2)`, the Taylor series is well-conditioned for the reduced argument.

**Step 1 — Input bounds:**
```solidity
if (x == 0) return SCALE;  // e⁰ = 1
if (x > MAX_EXP_INPUT) revert ExponentialOverflow();
```

**Step 2 — Range reduction:**
```solidity
uint256 shift = 0;
while (x > LN_2) {
    x -= LN_2;
    shift++;
}
```
After this loop, `x ∈ [0, LN_2)` and `shift = floor(x_original / LN_2)`. `shift` counts how many times we need to double the result.

**Step 3 — Taylor series for `e^r` where `r ∈ [0, LN_2]`:**
```solidity
uint256 sum = SCALE;    // term 0: e⁰ = 1
uint256 term = SCALE;   // current term, starts at x⁰/0! = 1

for (uint256 i = 1; i <= 18; ) {
    term = (term * x) / (i * SCALE);
    sum += term;
    if (term < MIN_TERM) break;  // early exit: term < 100 raw units
    unchecked { ++i; }
}
```

Each iteration computes `xⁱ / i!` by multiplying the previous term by `x / (i · SCALE)`. Division by `i * SCALE` is the combined scaling: we divide by `i` for the factorial and by `SCALE` to keep the result in 18-decimal.

For `x ≤ LN_2 ≈ 0.693`, 18 terms give approximately `(0.693)^18 / 18! ≈ 10^{-18}` relative error, well within fixed-point precision. The `MIN_TERM = 100` early exit triggers well before 18 terms for small inputs.

**Step 4 — Apply the doubling via bitshift:**
```solidity
if (shift > 0) {
    if (sum > type(uint256).max >> shift) revert ExponentialOverflow();
    result = sum << shift;
} else {
    result = sum;
}
```

`sum << shift` is equivalent to `sum * 2^shift`. This is the critical operation: rather than computing `sum * multiplier` where `multiplier = SCALE * 2^shift` (which overflows for large `shift`), we left-shift directly. The pre-shift check `sum > type(uint256).max >> shift` detects whether the shift would overflow.

**Why bitshift instead of multiply:** If we tracked `multiplier = SCALE * 2^shift` and computed `(sum * multiplier) / SCALE`, the intermediate product `sum * multiplier` would equal `sum * SCALE * 2^shift`. With `sum ≈ 2e18 ≈ 2^{61}` and `shift` up to ~195, this intermediate is `2^{61} · 2^{195} = 2^{256}`, which overflows. The bitshift `sum << shift` avoids the `SCALE` factor entirely since we count raw doublings.

---

### 6.2 `ln`

**Signature:** `function ln(uint256 x) internal pure returns (uint256 result)`

Computes `ln(x)` for a fixed-point input `x ≥ SCALE` (18-decimal). Values below `SCALE` (i.e., `x < 1.0`) would produce negative results, which are not supported.

**Algorithm: Integer Part via MSB + Fractional Part via Binary Search**

The identity used is:

```
ln(x) = ln(2^n · m) = n · ln(2) + ln(m)
```

where `n = floor(log₂(x/SCALE))` (the integer part) and `m = x / (SCALE · 2^n) ∈ [1, 2)` (the fractional mantissa).

**Step 1 — Guard and trivial cases:**
```solidity
if (x == 0 || x < SCALE) revert InvalidLogInput();
if (x == SCALE) return 0;  // ln(1) = 0
```

**Step 2 — Find integer part (MSB position of `x/SCALE`):**
```solidity
uint256 intPart = 0;
uint256 y = x / SCALE;
if (y >= 2**128) { y >>= 128; intPart += 128; }
if (y >= 2**64)  { y >>= 64;  intPart += 64;  }
if (y >= 2**32)  { y >>= 32;  intPart += 32;  }
if (y >= 2**16)  { y >>= 16;  intPart += 16;  }
if (y >= 2**8)   { y >>= 8;   intPart += 8;   }
if (y >= 2**4)   { y >>= 4;   intPart += 4;   }
if (y >= 2**2)   { y >>= 2;   intPart += 2;   }
if (y >= 2**1)   { intPart += 1; }
```
This is a standard binary-search for the most significant bit position (de Bruijn / conditional shift pattern). It finds `n` such that `2^n ≤ x/SCALE < 2^{n+1}`.

```solidity
uint256 intPartContribution = intPart * LN_2;
```

**Step 3 — Normalize to Q59 fixed-point for fractional binary search:**
```solidity
uint256 xNorm = (x << 59) / (SCALE << intPart);
```

`xNorm` represents `m = x / (SCALE · 2^intPart)` in Q59 format (i.e., multiplied by `2^59`). So `xNorm = m · 2^59 ∈ [2^59, 2^60)` since `m ∈ [1, 2)`.

The computation `(x << 59) / (SCALE << intPart)` = `x · 2^59 / (SCALE · 2^intPart)` = `(x / SCALE) · 2^(59-intPart)` = `m · 2^59`. For `intPart < 59`, the left shift `x << 59` keeps values in range since `x ≤ type(uint256).max >> 59`. For `intPart ≥ 59`, the right shift `SCALE << intPart` grows to absorb the left shift, keeping things bounded.

**Step 4 — Fractional part via iterative squaring (binary logarithm):**
```solidity
uint256 fracPart = 0;
for (uint256 i = 0; i < 59; ) {
    xNorm = (xNorm * xNorm) >> 59;
    if (xNorm >= 2**60) {      // checks if m² ≥ 2.0 in Q59
        xNorm >>= 1;           // keep m² in [1, 2)
        fracPart += LN_2 >> (i + 1);   // add ln(2) / 2^(i+1)
    }
    unchecked { ++i; }
}
```

This implements the **bit-by-bit logarithm algorithm**. The key insight: to find `ln(m)` for `m ∈ [1, 2)`, we determine its binary expansion bit by bit.

**How it works:**
- Start: `m ∈ [1, 2)`, target is `ln(m) ∈ [0, ln2)`
- At each step `i`, square `m` → if `m² ≥ 2`, then bit `i` of the binary expansion of `ln(m)/ln(2)` is 1
  - In that case, add `ln(2) / 2^(i+1)` to `fracPart` and halve `m²` to keep it in `[1, 2)`
  - If `m² < 2`, bit `i` is 0, continue with `m²`
- After 59 iterations: precision is `ln(2) / 2^59 ≈ 1.2e-18`, i.e., 1 ULP at 18-decimal

**Critical threshold — `>= 2**60`:** In Q59, `2.0` is represented as `2^60`. The check `xNorm >= 2**60` tests whether the squared value is ≥ 2.0. Checking `>= 2**59` (which represents 1.0) would be trivially true after any squaring and would produce garbage results — the threshold must be `2**60`.

**Step 5 — Combine:**
```solidity
result = intPartContribution + fracPart;
```

---

### 6.3 `mulScale`

```solidity
function mulScale(uint256 a, uint256 b) internal pure returns (uint256 result) {
    if (a == 0 || b == 0) return 0;
    uint256 product = a * b;
    if (product / a != b) revert MultiplicationOverflow();
    result = (product + (SCALE / 2)) / SCALE;
}
```

**Zero short-circuit:** Returns 0 immediately for either zero operand. Without this, `a * b = 0` would pass the overflow check (since `0 / a = 0` which happens to equal `b` only if `b = 0` — but actually `0 / a == 0 != b` for any non-zero `b`). The zero check is correct behavior and avoids the edge case.

**Overflow detection:** `product / a != b` detects if `a * b` wrapped. overflow protection in mulScale comes from Solidity 0.8.x's built-in checked arithmetic (the multiplication itself panics). The post-multiplication division check is redundant but harmless.

**Rounding:** `(product + SCALE/2) / SCALE` rounds to nearest. The `SCALE/2 = 5e17` addend shifts the truncation boundary by half an ULP.

---

### 6.4 `divScale`

```solidity
function divScale(uint256 a, uint256 b) internal pure returns (uint256 result) {
    if (b == 0) revert DivisionByZero();
    if (a == 0) return 0;
    uint256 scaled = a * SCALE;
    if (scaled / a != SCALE) revert MultiplicationOverflow();
    result = (scaled + (b / 2)) / b;
}
```

**Zero guard on `a`:** Without this, `scaled = 0 * SCALE = 0`, and then `0 / 0` in the overflow check triggers Panic 0x12 (division by zero) in Solidity 0.8.x. The explicit `if (a == 0) return 0;` prevents this.

**Overflow detection on `a * SCALE`:** The check `scaled / a != SCALE` verifies the multiplication didn't wrap. Since `SCALE = 1e18` is a known constant, `a * SCALE` overflows when `a > type(uint256).max / SCALE ≈ 1.15e59`. For quantities in the protocol (which are bounded by pool liquidity), this is not reached in practice, but the check provides safety.

**Rounding:** `(scaled + b/2) / b` rounds to nearest. Adding `b/2` before dividing by `b` shifts the threshold for rounding up vs. down by half the divisor.

---

## 7. Utility Function Implementations

### 7.1 `calculateWorstCaseLoss`

Implements Definition 4.7 from the paper: `Loss = C(q₀) - C(q) + max(qᵢ)`.

```solidity
if (quantities.length == 0 || initialQuantities.length == 0) revert EmptyQuantities();
if (quantities.length != initialQuantities.length) revert ArrayLengthMismatch();

uint256 costCurrent = costFunction(quantities, alpha);
uint256 costInitial = costFunction(initialQuantities, alpha);

uint256 maxQ = quantities[0];
for (uint256 i = 1; i < quantities.length; ) {
    if (quantities[i] > maxQ) maxQ = quantities[i];
    unchecked { ++i; }
}

if (costCurrent < maxQ) {
    worstCaseLoss = costInitial + maxQ - costCurrent;
} else {
    if (costInitial > costCurrent - maxQ) {
        worstCaseLoss = costInitial - (costCurrent - maxQ);
    } else {
        worstCaseLoss = 0; // market maker is in profit
    }
}
```

The branching on `costCurrent < maxQ` handles the (theoretically impossible, practically guarded) case where the invariant was approached. In normal operation, `costCurrent ≥ maxQ` always, so the else branch executes. Returns 0 when `C(q) - max(qᵢ) ≥ C(q₀)` (market maker has outcome-independent profit).

---

### 7.2 `hasOutcomeIndependentProfit`

Computes `R(q) = C(q) - max(qᵢ) - C(q₀)` and tests its sign.

```solidity
int256 revenue = int256(costCurrent) - int256(maxQ) - int256(costInitial);
if (revenue > 0) {
    hasProfit = true;
    profit = uint256(revenue);
} else {
    hasProfit = false;
    profit = 0;
}
```

The use of `int256` arithmetic is necessary because `revenue` can be negative. The casts `int256(costCurrent)` etc. are safe as long as values stay below `2^255 ≈ 5.8e76`; values this large would have triggered overflow in `costFunction` already.

---

## 8. Validation Helpers

### `validateQuantities`

Returns `false` (does not revert) for:
- `quantities.length < 2` (single-outcome and empty markets are invalid)
- `quantities.length > MAX_OUTCOMES`
- `Σqᵢ == 0` (all zeros)

Returns `true` otherwise. The consuming contract is responsible for deciding whether to revert or handle an invalid return.

### `validateAlpha`

Returns `true` iff `alpha ∈ [MIN_ALPHA, MAX_ALPHA]`. Used for external pre-validation before calling core functions that revert on invalid alpha.

---

## 9. Numerical Stability: Log-Sum-Exp Trick

Both `costFunction` and `getAllPrices` must compute `Σ exp(qᵢ/b)`. For large markets with high outstanding quantities, the individual `exp(qᵢ/b)` values can be enormous, causing overflow when accumulated.

**The identity:**

```
ln(Σ exp(xᵢ)) = m + ln(Σ exp(xᵢ - m))
```

where `m = max(xᵢ)`. Because `xᵢ - m ≤ 0` for all `i`, each `exp(xᵢ - m) ∈ (0, 1]`. The shifted exponentials are bounded and cannot overflow on their own; only their sum might, but with `MAX_OUTCOMES = 100` and each term ≤ `SCALE = 1e18`, the maximum sum is `100 · 1e18 = 1e20`, safely within uint256.

**Implementation detail for the negative-shift branch:** When `ratio < maxRatio`, we need `exp(ratio - maxRatio) = exp(-(maxRatio - ratio))`. In fixed-point unsigned arithmetic, we compute this as:

```
exp(-diff) = 1 / exp(diff) = SCALE² / exp(diff)
```

The numerator is `SCALE² = 1e36` because `exp(diff)` is already in 18-decimal (i.e., it represents `e^diff · SCALE`). Dividing by it gives `(SCALE²) / (e^diff · SCALE) = SCALE / e^diff`, which is `e^{-diff}` in 18-decimal. This is correct.

**Why this matters:** Without the Log-Sum-Exp trick, `exp(qᵢ/b)` for `qᵢ/b ≈ 100` would be `e^100 ≈ 2.7e43`, and summing even a few such terms overflows uint256. With the trick, each term in the shifted sum is at most 1.0 (in fixed-point: at most `SCALE`), and the overflow risk is eliminated.

---

## 10. Precision and Rounding Policy

**Fixed-point precision:** 18 decimal places (`SCALE = 1e18`). The minimum representable non-zero value is `1 wei = 1e-18`.

**Rounding:** `mulScale` and `divScale` use round-half-up (biased toward positive). This provides:
- Expected rounding error ≈ 0 (unbiased) for random inputs
- Worst-case per-operation error: ±0.5 ULP (0.5e-18)

**Error accumulation in `getAllPrices`:** The function performs approximately `3 + 5·n` fixed-point multiplications/divisions (where `n` is the number of outcomes). For `n = 100`, this is ~503 operations. Expected accumulated error: `503 · 0.25e-18 ≈ 1.3e-16`. This is within the precision budget for any practical use case.

**Taylor series precision (`exp`):** For `x ∈ [0, LN_2)`, 18 terms provide error `< (LN_2)^18 / 18! ≈ 10^{-18}`. The early-exit at `MIN_TERM = 100` raw units (= `1e-16` in 18-decimal) triggers before reaching 18 terms for most practical inputs and introduces negligible additional error.

**Binary search precision (`ln`):** 59 iterations provide resolution of `LN_2 / 2^59 ≈ 1.2e-18`, i.e., approximately 1 ULP. This matches the fixed-point precision floor.

**Cost function invariant clamp:** The clamp `if (cost < maxQ) cost = maxQ` introduces at most `maxQ - cost_exact` error, bounded by the fixed-point truncation in the Log-Sum-Exp sum. In practice this difference is less than 1e-12 relative to `maxQ` for realistic market states.

---

## 11. Gas Considerations

**Function call overhead:** As a Solidity `library` with `internal` functions, calls are compiled to `JUMP` instructions, not `CALL`. No external call overhead. No returndata copy.

**Memory allocation:** `getAllPrices` and `getAllPrices` (via `getPrice`) allocate:
- `prices[]` — `n` slots = `32n + 96` bytes
- `expValues[]` — `n` slots = `32n + 96` bytes
- Cost: approximately `3n + 6` memory operations per call

**Loop cost (dominant term):** `getAllPrices` runs three explicit loops over `n` outcomes plus one inside `liquidityParameter`. Total iterations: approximately `4n`. For `n = 100`: ~400 iterations. Each calls `exp()` once (dominant cost ≈ 2,000–5,000 gas per call for typical inputs). Total: 200,000–500,000 gas for full price computation on 100-outcome markets.

**`exp()` gas:** Dominated by the Taylor series loop. For small `x` (common in balanced markets where `qᵢ/b ≈ 1`), early-exit triggers at ~10–12 iterations ≈ 2,000–3,000 gas. For `x` close to `MAX_EXP_INPUT`, all 18 terms run plus the range reduction loop (~150 iterations for x ≈ 135e18) ≈ 8,000–12,000 gas.

**`ln()` gas:** The MSB search is 8 conditional branches (constant). The 59-iteration binary search dominates ≈ 3,000–5,000 gas.

**Optimization opportunities:**
- Callers needing prices for all outcomes should always call `getAllPrices()` directly rather than `getPrice()` in a loop (avoids redundant `exp()` calls)
- `liquidityParameter()` tuple return allows reuse of `sumQ` without a second loop — callers should destructure the return rather than calling it twice
- If only `b` is needed (not `sumQ`), `(uint256 b, ) = liquidityParameter(...)` is idiomatic and the compiler will not compute the unused return

---

## 12. Known Limitations

**1. Unsigned arithmetic only:** All quantities must be non-negative (`uint256`). Negative exposures or short positions require the calling contract to translate trades into positive-orthant form (e.g., selling outcome `i` is expressed as buying all other outcomes equivalently).

**2. No division underflow protection:** For very small quantities relative to `b`, `(qᵢ * SCALE) / b` truncates to zero. This means `exp(0) = SCALE`, contributing a term of `1.0` to `sumExp` regardless of the actual very-small ratio. This is acceptable precision loss for micro-quantity outcomes.

**3. `ln()` input domain:** Only inputs `≥ SCALE` (i.e., `≥ 1.0` in fixed-point) are accepted. This is correct for the LS-LMSR use case, where `ln` is called on `sumExp ≥ SCALE` (since at least one term in the sum is `exp(0) = SCALE`). If `ln` is called with `x < SCALE`, it reverts with `InvalidLogInput`.

**4. Fixed `MAX_OUTCOMES = 100`:** Markets with more than 100 outcomes cannot be served by this library. This is a deliberate gas limit, not a mathematical constraint.

**5. Precision loss for highly imbalanced markets:** When one outcome has quantities orders of magnitude larger than others, the ratios `qⱼ/b` for small outcomes are very close to zero and `exp()` of these tiny values approaches 1.0 from a fixed-point perspective. This is mathematically correct behavior but can cause visible rounding in the second-term price adjustment for minority outcomes.

**6. `MAX_EXP_INPUT` is approximate:** The constant `135305999368893231589` is derived from `ln(2^{255} / 1e18)`. Due to the approximate nature of `LN_2` in fixed-point, inputs very close to this boundary may behave differently from the exact mathematical limit. The `ExponentialOverflow` guard on the final bit-shift provides a second safety net.

---

## 13. Testing Guidance

### Invariant-Based Tests

The following mathematical invariants should hold for all valid inputs and form the basis of property-based (fuzz) tests:

| Invariant | Test expression |
|---|---|
| `C(q) ≥ max(qᵢ)` | `costFunction(q, α) >= max(q)` |
| `Σpᵢ ≥ 1` | `sumOfPrices(q, α) >= 1e18` |
| `Σpᵢ ≤ 1 + α·n·ln(n)` | Compute bound and assert `<=` |
| `exp` result `≥ SCALE` | `exp(x) >= SCALE` for all `x ≥ 0` |
| `ln(exp(x)) ≈ x` | Round-trip within `1e12` error for `x ∈ [0, MAX_EXP_INPUT]` |
| `exp(ln(x)) ≈ x` | Round-trip within `1e12` error for `x ≥ SCALE` |
| `C(q) > 0` for `sumQ > 0` | `costFunction(q, α) > 0` |

### Critical Edge Cases

**`exp()` boundary:**
- `exp(0)` → exactly `1e18`
- `exp(MAX_EXP_INPUT)` → should not revert, result ≈ `type(uint256).max / 2`
- `exp(MAX_EXP_INPUT + 1)` → must revert with `ExponentialOverflow`
- `exp(135e18)` → should not revert (large but valid)

**`ln()` boundary:**
- `ln(SCALE)` → exactly `0` (i.e., `ln(1) = 0`)
- `ln(SCALE - 1)` → must revert with `InvalidLogInput`
- `ln(0)` → must revert with `InvalidLogInput`
- `ln(2e18)` → must be within `1e12` of `LN_2 = 693147180559945309`
- `ln(e_approx)` where `e_approx ≈ 2718281828459045235` → must be within `1e12` of `1e18`

**`divScale()` edge cases:**
- `divScale(0, 1e18)` → must return `0` (not panic)
- `divScale(1e18, 0)` → must revert with `DivisionByZero`
- `divScale(type(uint256).max / SCALE + 1, 1)` → must revert with `MultiplicationOverflow`

**`liquidityParameter()` edge cases:**
- Very small quantities: `quantities = [1, 1]`, `alpha = MIN_ALPHA` → `b` should be clamped to `1`
- `quantities.length == 1` → must return `false` from `validateQuantities`, and revert from `liquidityParameter` (only 1 element, but `EmptyQuantities` isn't thrown; this passes the length check but fails at `getAllPrices` due to `InsufficientOutcomes` path)
- All zeros: `quantities = [0, 0, 0]` → must revert with `ZeroQuantitySum`

**Highly imbalanced market:**
- `quantities = [1e24, 1, 1]` with `alpha = 5e16` → `costFunction` result should satisfy invariant `cost >= 1e24`
- Prices should all be positive (no revert on `priceInt <= 0`)

**`calculateTradeCost()` signs:**
- Buying: `quantitiesTo[i] > quantitiesFrom[i]` for some `i` → `tradeCost > 0`
- Selling: `quantitiesTo[i] < quantitiesFrom[i]` for some `i` → `tradeCost < 0` is possible

### Recommended Fuzz Strategy (Foundry)

```solidity
function testFuzz_costFunctionInvariant(
    uint256[3] memory rawQ,
    uint256 rawAlpha
) public {
    // Bound inputs
    uint256 alpha = bound(rawAlpha, LSMath.MIN_ALPHA, LSMath.MAX_ALPHA);
    uint256[] memory quantities = new uint256[](3);
    for (uint256 i = 0; i < 3; i++) {
        quantities[i] = bound(rawQ[i], 1, 1e30);
    }

    uint256 cost = LSMath.costFunction(quantities, alpha);

    // Invariant: C(q) >= max(qi)
    uint256 maxQ = quantities[0];
    for (uint256 i = 1; i < 3; i++) {
        if (quantities[i] > maxQ) maxQ = quantities[i];
    }
    assertGe(cost, maxQ, "Invariant C(q) >= max(qi) violated");
}

function testFuzz_exp_gte_scale(uint256 x) public {
    x = bound(x, 0, LSMath.MAX_EXP_INPUT);
    uint256 result = LSMath.exp(x);
    assertGe(result, LSMath.SCALE, "exp(x) must be >= 1");
}

function testFuzz_sumOfPrices_bounds(
    uint256[5] memory rawQ,
    uint256 rawAlpha
) public {
    uint256 alpha = bound(rawAlpha, LSMath.MIN_ALPHA, LSMath.MAX_ALPHA);
    uint256[] memory quantities = new uint256[](5);
    for (uint256 i = 0; i < 5; i++) {
        quantities[i] = bound(rawQ[i], 1e6, 1e24);
    }
    uint256 s = LSMath.sumOfPrices(quantities, alpha);
    assertGe(s, 1e18, "Sum of prices must be >= 1.0");
    // Upper bound: 1 + alpha * n * ln(n) where n=5, ln(5) ≈ 1.609
    uint256 upperBound = 1e18 + (alpha * 5 * 1609437912434100000) / 1e18;
    assertLe(s, upperBound + 1e12, "Sum of prices exceeds upper bound");
}
```

### Gas Benchmarking Targets

| Function | Expected Gas Range | Notes |
|---|---|---|
| `exp(1e18)` | 2,000–3,500 | Typical input, early exit |
| `exp(100e18)` | 7,000–12,000 | Many range reduction steps |
| `ln(2e18)` | 3,000–5,000 | Standard case |
| `liquidityParameter(n=10)` | 4,000–6,000 | Loop over 10 outcomes |
| `costFunction(n=10)` | 30,000–60,000 | Includes 10× exp + ln |
| `getAllPrices(n=10)` | 60,000–120,000 | 10× exp, 3 loops |
| `costFunction(n=100)` | 300,000–600,000 | 100× exp + ln |
| `getAllPrices(n=100)` | 500,000–1,000,000 | 100× exp, 3 loops, 100× price computation |
