# LSMath — LS-LMSR Mathematics Library

**Contract**: `LSMath.sol`  
**Solidity**: `0.8.31` (strict, non-floating)  
**Type**: `library` (stateless, internal-only)  
**Chain**: Base L2 (EVM-compatible)  
**Reference Paper**: Othman, Pennock, Reeves, Sandholm — *"A Practical Liquidity-Sensitive Automated Market Maker"* (ACM TEAC, 2013)

---

## Table of Contents

1. [What This Library Is](#1-what-this-library-is)
2. [Background: LMSR and Its Limitations](#2-background-lmsr-and-its-limitations)
3. [The LS-LMSR Model](#3-the-ls-lmsr-model)
4. [Mathematical Foundations](#4-mathematical-foundations)
   - [4.1 Liquidity Parameter b(q)](#41-liquidity-parameter-bq)
   - [4.2 Cost Function C(q)](#42-cost-function-cq)
   - [4.3 Price Function pᵢ(q)](#43-price-function-piq)
   - [4.4 Trade Cost](#44-trade-cost)
5. [Theoretical Properties](#5-theoretical-properties)
   - [5.1 Path Independence](#51-path-independence)
   - [5.2 Liquidity Sensitivity](#52-liquidity-sensitivity)
   - [5.3 Positive Homogeneity](#53-positive-homogeneity)
   - [5.4 Bounded Loss (Lemma 4.5 / Proposition 4.9)](#54-bounded-loss-lemma-45--proposition-49)
   - [5.5 Outcome-Independent Profit](#55-outcome-independent-profit)
   - [5.6 Sum-of-Prices Bounds](#56-sum-of-prices-bounds)
   - [5.7 Non-Translation Invariance](#57-non-translation-invariance)
6. [The Alpha Parameter](#6-the-alpha-parameter)
7. [Valid Market Region](#7-valid-market-region)
8. [Library Architecture](#8-library-architecture)
9. [Function Catalog](#9-function-catalog)
10. [Parameter Reference](#10-parameter-reference)
11. [Invariants Summary](#11-invariants-summary)

---

## 1. What This Library Is

`LSMath` is a **stateless Solidity library** implementing the core mathematics of the Liquidity-Sensitive Logarithmic Market Scoring Rule (LS-LMSR). It provides the complete mathematical engine for an automated market maker (AMM) designed specifically for binary-payout prediction markets.

The library is purely computational — it holds no state, emits no events, and performs no token transfers. Every function is `internal pure`, meaning it is embedded at compile time into any contract that imports it and executes without any storage reads or writes.

Any contract implementing a prediction market pool, LP vault, or settlement system imports `LSMath` to perform pricing, cost computation, liquidity measurement, and market solvency analysis.

---

## 2. Background: LMSR and Its Limitations

The **Logarithmic Market Scoring Rule (LMSR)**, introduced by Robin Hanson (2003, 2007), is the canonical automated market maker for prediction markets. Its cost function is:

```
C_LMSR(q) = b · ln(Σ exp(qᵢ / b))
```

where `b` is a constant liquidity parameter set before the market opens.

The LMSR has two practical problems:

**Problem 1 — Fixed Liquidity**: The parameter `b` determines how much prices move per trade. Set too low, every trade causes wild price swings. Set too high, prices barely move even after large trades. The "right" value of `b` depends on total future trading volume, which is unknown at market creation. This is a permanent design flaw — `b` is constant regardless of how much activity the market actually sees.

**Problem 2 — Guaranteed Loss**: The LMSR operator always loses money. The worst-case loss is `b · ln(n)` where `n` is the number of outcomes. Increasing liquidity (higher `b`) directly increases this potential loss. In practice, almost every LMSR deployment uses play money rather than real money because of this.

Theorem 2.9 from the paper establishes that no market maker can simultaneously satisfy all three of: (1) **path independence**, (2) **translation invariance** (prices sum to exactly 1), and (3) **liquidity sensitivity**. Something must give.

---

## 3. The LS-LMSR Model

The LS-LMSR makes one targeted modification to the LMSR: it replaces the constant liquidity parameter `b` with a **variable** one that grows as market volume grows.

```
b(q) = α · Σqᵢ
```

The cost function becomes:

```
C(q) = b(q) · ln(Σ exp(qᵢ / b(q)))
```

The tradeoff accepted to enable this is the **relaxation of translation invariance**. Prices no longer sum to exactly 1 — they sum to slightly above 1. This excess above 1 is the market maker's commission, analogous to the bid-ask spread in a traditional market. The paper proves this relaxation is not just a design choice but a mathematical *necessity*: liquidity sensitivity and translation invariance are mutually exclusive for any path-independent market maker (Theorem 2.9).

What is gained:

- **Self-adjusting liquidity**: `b(q)` grows automatically as total quantity `Σqᵢ` grows, so the market naturally deepens with activity without any manual parameter changes.
- **Profit potential**: Because prices sum to more than 1, the market maker earns a spread on every round trip, enabling outcome-independent profit across a wide range of terminal market states.
- **Arbitrarily small loss**: By setting the initial quantity vector close to zero, the worst-case loss `C(q₀)` can be made arbitrarily small. Unlike the LMSR, this near-zero initial loss does not lock in permanently thin markets — sensitivity is high only during the initial stage and diminishes as volume accumulates.
- **Positive homogeneity**: `C(γq) = γ · C(q)`, meaning the market scales proportionally. A market operating in millions of yen behaves identically (in relative terms) to one operating in fractions of a dollar.

---

## 4. Mathematical Foundations

### 4.1 Liquidity Parameter b(q)

**Definition:**

```
b(q) = α · Σᵢ qᵢ
```

**Inputs:**
- `q = [q₁, q₂, ..., qₙ]` — the quantity vector, where `qᵢ` is the total outstanding quantity for outcome `i`
- `α` — a fixed commission parameter set at market creation

**Interpretation:**

`b(q)` represents how elastic the market is to a given trade size. When `b` is small (early in market life), each unit of quantity purchased moves prices by a large amount. When `b` is large (as trading volume accumulates), the same unit moves prices very little.

The formula `b(q) = α · Σqᵢ` ties liquidity depth directly to total market volume. This is the mechanism by which LS-LMSR achieves automatic liquidity adjustment — no oracle, no external parameter updates, no governance.

**Properties:**
- `b(q) > 0` for any non-zero quantity vector and any valid `α`
- `b(γq) = γ · b(q)` — linear scaling with volume
- `b → 0` as volume → 0 (maximum price sensitivity at market open)
- `b → ∞` as volume → ∞ (prices become increasingly stable)

---

### 4.2 Cost Function C(q)

**Definition:**

```
C(q) = b(q) · ln(Σᵢ exp(qᵢ / b(q)))
```

**Interpretation:**

`C(q)` represents the cumulative amount of money paid into the market to reach state `q` from the initial state. It is the scalar potential field of which the price function is the gradient. Because the market is path-independent, `C(q)` uniquely determines the total cost for any sequence of trades that leads to quantity vector `q`.

The inner term `Σ exp(qᵢ / b(q))` is a softmax-like aggregation. The logarithm combined with the leading `b(q)` factor produces units in the same domain as the quantities themselves.

**Fundamental Invariant (Lemma 4.5):**

```
C(q) ≥ max(qᵢ)   for all valid q
```

This states that the cost function is always at least as large as the largest outstanding quantity. Intuitively: if one outcome has `qᵢ` tokens outstanding, at least that much must have been paid in to create them. This invariant is the backbone of the bounded-loss guarantee.

---

### 4.3 Price Function pᵢ(q)

**Definition** (from paper Section 4.1):

```
pᵢ(q) = α · ln(Σⱼ exp(qⱼ/b)) + [Σⱼ qⱼ · exp(qᵢ/b) - Σⱼ qⱼ · exp(qⱼ/b)] / [Σⱼ qⱼ · Σⱼ exp(qⱼ/b)]
```

This is the partial derivative of `C(q)` with respect to `qᵢ`. Because the cost function is the potential field, prices are its gradient — this is what makes the market path-independent.

**Decomposition:**

The price formula has two terms:

```
pᵢ(q) = α · ln(Σ exp(qⱼ/b))     ← alphaTerm (shared across all outcomes)
       + (weightedᵢ - weightedSum) / denominator   ← outcome-specific adjustment
```

Where:
- `alphaTerm` = `α · ln(Σ exp(qⱼ/b))` — a market-wide term reflecting total volume
- `weightedᵢ` = `Σqⱼ · exp(qᵢ/b)` — the weight for outcome `i`
- `weightedSum` = `Σⱼ (qⱼ · exp(qⱼ/b))` — total weighted sum across all outcomes
- `denominator` = `Σqⱼ · Σ exp(qⱼ/b)`

The second term is always zero in a balanced market (all `qᵢ` equal), and is positive for outcomes with above-average quantities. The first term is always positive. Together they produce prices that are individually positive and collectively sum to slightly above 1.

**Relationship to Standard LMSR Price:**

The standard LMSR price is simply `exp(qᵢ/b) / Σ exp(qⱼ/b)`. The LS-LMSR price is substantially more complex because the derivative must account for the fact that `b(q)` itself depends on `q` — moving `qᵢ` changes both the numerator and denominator of every ratio in the sum.

---

### 4.4 Trade Cost

**Definition:**

```
ΔC = C(q₁) - C(q₀)
```

To move the market from state `q₀` to state `q₁`, a trader pays `C(q₁) - C(q₀)`. If this value is positive, the trader pays the vault. If negative, the vault pays the trader (e.g., when selling shares back).

Path independence guarantees that the cost is the same regardless of whether the trader moves from `q₀` to `q₁` in a single transaction or a series of smaller ones.

---

## 5. Theoretical Properties

### 5.1 Path Independence

**Property:** The cost to move from any market state `q₀` to any market state `q₁` depends only on the endpoints, not the path taken between them.

**Why it matters:** Path independence prevents money pumps (no trader can profit by cycling through a sequence of trades that returns to the original state). It also means traders have no incentive to split orders into many smaller trades.

Path independence follows directly from the existence of a cost function `C(q)` (Lemma 2.5): if a pricing rule is path-independent and has a convex pre-image, prices are the gradient of a scalar potential field, which is exactly the cost function.

---

### 5.2 Liquidity Sensitivity

**Property:** For any valid quantity vector `q` and any scalar `α > 0`:

```
pᵢ(q + α·1) ≠ pᵢ(q)     (in general)
```

where `1 = [1, 1, ..., 1]`. Adding volume to all outcomes equally does change prices in LS-LMSR, unlike the standard LMSR where such an addition leaves prices unchanged.

**Why it matters:** In real markets, prices in highly liquid markets are more stable than in thin ones. LS-LMSR captures this: as total volume (`Σqᵢ`) grows, `b(q)` grows, and the ratio `qᵢ/b(q)` for any incremental trade becomes smaller. Smaller exponent differences mean smaller price differences — the market is deeper.

---

### 5.3 Positive Homogeneity

**Property** (Proposition 4.10):

```
C(γq) = γ · C(q)   for all γ > 0
```

**Proof sketch:** Substituting `γq` into the cost function, `b(γq) = α · Σ(γqᵢ) = γ · b(q)`, so the ratio `γqᵢ / b(γq) = qᵢ / b(q)` is invariant. The `b(γq)` prefactor provides the factor of `γ`.

**Consequence** (Proposition 4.12): Prices are homogeneous of degree zero:

```
pᵢ(γq) = pᵢ(q)   for all γ > 0
```

This means prices depend only on the *relative* distribution of quantities, not their absolute magnitudes. A market with quantities `[100, 200]` displays identical prices to one with `[1, 2]`. The market is currency-independent — it functions equivalently regardless of whether quantities are expressed in wei or whole tokens.

---

### 5.4 Bounded Loss (Lemma 4.5 / Proposition 4.9)

**Lemma 4.5:**

```
C(q) ≥ max(qᵢ)
```

**Proposition 4.9:** The market maker's loss is bounded by the initial cost `C(q₀)`:

```
Loss = C(q₀) - C(q) + max(qᵢ) ≤ C(q₀)
```

**Why it matters:** As the initial quantity vector `q₀ → 0`, `C(q₀) → 0`. The operator can choose initial conditions that make worst-case loss arbitrarily small. Unlike the LMSR (where reducing `b` for low loss permanently makes the market too sensitive), in LS-LMSR the high sensitivity applies only during the startup phase. Once volume accumulates, `b(q)` grows and sensitivity naturally decreases.

---

### 5.5 Outcome-Independent Profit

**Definition:**

```
R(q) = C(q) - max(qᵢ) - C(q₀)
```

When `R(q) > 0`, the market maker earns a profit regardless of which outcome is realized. This is because `C(q)` represents total money collected, `max(qᵢ)` is the maximum payout in any scenario, and `C(q₀)` is the initial subsidy cost.

**Condition:** `R(q) > 0` holds over a broad region of market states, particularly when prices are not too extreme (no outcome has converged to near-certainty). As shown in the paper's figures, the profitable region grows with `α` and shrinks as the initial subsidy `C(q₀)` increases.

---

### 5.6 Sum-of-Prices Bounds

**Theorem** (Propositions 4.1–4.4):

```
1 + O(α²)  ≤  Σᵢ pᵢ(q)  ≤  1 + α·n·ln(n)
```

- **Upper bound** `1 + α·n·ln(n)`: achieved when all outcomes have equal quantity (`q = k·1`)
- **Lower bound** → 1 as α → 0: achieved when one outcome has all the quantity

The excess above 1 represents the market maker's commission spread. For small `α`, this spread is tiny (first-order term is O(α²)), so prices closely approximate probabilities even though translation invariance is technically broken.

**Practical implication:** To cap the maximum commission at `v` percent, set:

```
α = v / (n · ln(n))
```

For example, a 5% cap on a 10-outcome market: `α = 0.05 / (10 · ln(10)) ≈ 0.00217`.

---

### 5.7 Non-Translation Invariance

**Property:** Prices do **not** sum to exactly 1:

```
Σᵢ pᵢ(q) ≥ 1   (strictly greater in general)
```

**Why this is acceptable:** The deviation from 1 is bounded by `α·n·ln(n)`, which is small for the `α` values natural to the protocol. Probabilities can be recovered approximately by normalizing: `P(outcome i) ≈ pᵢ(q) / Σⱼ pⱼ(q)`.

**Why this is necessary:** Theorem 2.9 proves that any path-independent market maker that is liquidity-sensitive cannot be translation invariant. The choice to relax translation invariance (rather than path independence or liquidity sensitivity) is mathematically forced for the class of cost-function market makers LS-LMSR belongs to.

---

## 6. The Alpha Parameter

`α` (alpha) is the single governance-level parameter of the LS-LMSR mechanism. It has a direct economic interpretation as the market maker's **commission rate**.

**Bounds enforced by the library:**

| Constant | Value | Meaning |
|---|---|---|
| `MIN_ALPHA` | `1e12` (= 0.000001) | Prevents near-zero pricing with zero elasticity |
| `MAX_ALPHA` | `2e17` (= 0.2) | Caps commission at 20%; above this, prices become excessively super-unitary |

**Effect of increasing α:**
- `b(q)` grows faster for the same quantity vector → prices become less elastic sooner
- Sum of prices increases → larger market maker spread
- Profitable region expands (Proposition 4.6: cost function is non-decreasing in α)
- At `α → 0`: approaches standard LMSR behavior (nearly translation invariant, nearly liquidity-insensitive)
- At `α = 0.2` (max): 20% maximum commission, deep markets, wide spread

**Setting α in practice:** The paper suggests `α = v / (n · ln(n))` where `v` is the desired maximum commission rate and `n` is the number of outcomes. The protocol's outer contract is responsible for enforcing a sensible `α` for the specific market; `LSMath` only enforces the hard bounds.

---

## 7. Valid Market Region

LS-LMSR is defined over the **positive orthant**: all `qᵢ ≥ 0`. Negative quantities are not permitted.

This constraint comes from the design choice to "always move forward in obligation space" (Section 3.3 of the paper). A trader wishing to sell outcome `i` does not place a negative bet; instead, the market is translated so the trade is expressed as a positive-quantity purchase of the complementary outcomes.

The library enforces this via `uint256` types — unsigned integers cannot represent negative values, so the type system itself enforces the positive orthant constraint.

**Minimum quantity vector:** The market may not have all quantities equal to zero. `ZeroQuantitySum` is thrown if `Σqᵢ = 0`, since `b(q) = 0` is undefined.

**Minimum outcomes:** At least 2 outcomes are required (`validateQuantities` returns false for `n < 2`). A single-outcome market has no information content and is mathematically degenerate.

**Maximum outcomes:** The library hard-caps at 100 outcomes (`MAX_OUTCOMES`). Beyond this, gas costs for loops over the quantity vector become prohibitive.

---

## 8. Library Architecture

```
LSMath (library)
│
├── Core LS-LMSR Functions
│   ├── liquidityParameter()   — b(q) = α · Σqᵢ
│   ├── costFunction()         — C(q) = b · ln(Σ exp(qᵢ/b))
│   ├── getPrice()             — pᵢ(q) for a single outcome
│   ├── getAllPrices()          — pᵢ(q) for all outcomes (efficient)
│   ├── calculateTradeCost()   — C(q₁) - C(q₀)
│   └── sumOfPrices()          — Σᵢ pᵢ(q)
│
├── Utility Functions
│   ├── calculateWorstCaseLoss()        — C(q₀) - C(q) + max(qᵢ)
│   └── hasOutcomeIndependentProfit()   — R(q) = C(q) - max(qᵢ) - C(q₀)
│
├── Mathematical Helpers (internal primitives)
│   ├── exp()        — Fixed-point eˣ via Taylor series + range reduction
│   └── ln()         — Fixed-point ln(x) via integer part + binary search
│
├── Fixed-Point Arithmetic
│   ├── mulScale()   — (a · b) / SCALE with overflow guard and rounding
│   └── divScale()   — (a · SCALE) / b with overflow guard and rounding
│
└── Validation Helpers
    ├── validateQuantities()   — structural validity of a quantity vector
    └── validateAlpha()        — α within [MIN_ALPHA, MAX_ALPHA]
```

All functions are `internal pure`. The library has no constructor, no storage, no events, and no payable functions. It is purely a mathematical computation module.

---

## 9. Function Catalog

### Core LS-LMSR Functions

#### `liquidityParameter(quantities, alpha) → (b, sumQ)`

Computes `b(q) = α · Σqᵢ`. Returns both `b` and `sumQ` as a tuple so callers can reuse `sumQ` without recomputing it. If the computed `b` rounds to zero due to fixed-point truncation (possible for very small quantities), it is clamped to `1` to preserve the liquidity invariant and prevent downstream division by zero.

#### `costFunction(quantities, alpha) → cost`

Computes `C(q) = b(q) · ln(Σ exp(qᵢ/b(q)))`. Uses the Log-Sum-Exp numerical stabilization technique internally to handle large quantities safely. Enforces the `C(q) ≥ max(qᵢ)` invariant: if arithmetic truncation causes the raw result to fall below `max(qᵢ)`, the result is clamped up to `max(qᵢ)`.

#### `getPrice(quantities, outcomeIndex, alpha) → price`

Returns `pᵢ(q)` for a single outcome. Internally delegates to `getAllPrices()` to avoid duplicating the shared intermediate computation. The price is in 18-decimal fixed-point and may exceed `1e18` (since prices can sum to above 1).

#### `getAllPrices(quantities, alpha) → prices[]`

Computes the complete price vector for all outcomes in a single pass. More gas-efficient than calling `getPrice()` repeatedly, as intermediate values (`expValues`, `sumExp`, `weightedSum`, `denominator`) are computed once and reused across all outcomes. The price for each outcome is computed per the full LS-LMSR price formula from Section 4.1 of the paper.

#### `calculateTradeCost(quantitiesFrom, quantitiesTo, alpha) → tradeCost`

Returns `C(q₁) - C(q₀)` as a signed integer. Positive values indicate the trader pays; negative values indicate the market maker pays (e.g., when a user sells shares back). Both cost values are bounds-checked against `int256` max before the subtraction to prevent silent overflow in casting.

#### `sumOfPrices(quantities, alpha) → sum`

Returns `Σᵢ pᵢ(q)`. Useful for diagnostics and commission monitoring. The result is bounded by the theoretical range `[1 + O(α²), 1 + α·n·ln(n)]`.

---

### Utility Functions

#### `calculateWorstCaseLoss(quantities, initialQuantities, alpha) → worstCaseLoss`

Computes `Loss = C(q₀) - C(q) + max(qᵢ)` per Definition 4.7 of the paper. Returns zero if the market is in a profitable state (`R(q) > 0`). Array lengths must match.

#### `hasOutcomeIndependentProfit(quantities, initialQuantities, alpha) → (hasProfit, profit)`

Returns `true` and the profit amount if `R(q) = C(q) - max(qᵢ) - C(q₀) > 0`. Returns `(false, 0)` otherwise. Array lengths must match.

---

### Validation Helpers

#### `validateQuantities(quantities) → valid`

Returns `true` if: the array has at least 2 and at most `MAX_OUTCOMES` elements, and `Σqᵢ > 0`. Does not revert — returns `false` for invalid inputs.

#### `validateAlpha(alpha) → valid`

Returns `true` if `α ∈ [MIN_ALPHA, MAX_ALPHA]`. Does not revert.

---

## 10. Parameter Reference

| Constant | Value | Description |
|---|---|---|
| `SCALE` | `1e18` | Fixed-point scaling factor (18 decimal places) |
| `MIN_ALPHA` | `1e12` | Minimum α (0.000001 = 0.0001%) |
| `MAX_ALPHA` | `2e17` | Maximum α (0.2 = 20%) |
| `MAX_OUTCOMES` | `100` | Maximum number of outcomes per market |
| `LN_2` | `693147180559945309` | `ln(2)` in 18-decimal fixed-point |
| `MAX_EXP_INPUT` | `135305999368893231589` | Maximum input to `exp()` without overflow |
| `MIN_TERM` | `100` | Taylor series early-exit threshold (in raw units) |

**Quantity representation:** All `qᵢ` values are in 18-decimal fixed-point (i.e., `1e18` represents one unit of quantity). Alpha and all prices are also 18-decimal.

---

## 11. Invariants Summary

The following invariants hold for all valid inputs (quantities in positive orthant, `α` in valid range):

| Invariant | Statement | Source |
|---|---|---|
| Positive liquidity | `b(q) ≥ 1` | Implementation (clamped) |
| Bounded cost | `C(q) ≥ max(qᵢ)` | Lemma 4.5 (enforced) |
| Bounded loss | `Loss ≤ C(q₀)` | Proposition 4.9 |
| Price sum lower bound | `Σpᵢ ≥ 1` | Proposition 4.3 |
| Price sum upper bound | `Σpᵢ ≤ 1 + α·n·ln(n)` | Proposition 4.1 |
| Homogeneity | `C(γq) = γ·C(q)` | Proposition 4.10 |
| Price scale invariance | `pᵢ(γq) = pᵢ(q)` | Proposition 4.12 |
| Cost non-decreasing in α | `∂C/∂α ≥ 0` | Proposition 4.6 |
| Path independence | `ΔC = C(q₁) - C(q₀)` (no path dependence) | Cost function existence |
