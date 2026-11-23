# #1156 — Fee tier calculation edge case tests

**Repo:** Stellar-split/split-contracts

> This plan was written without access to the contract source. Fill in the real tier table, the fee function name (`calculate_fee` below), and the rounding mode, then turn the tables into concrete values.

## Goal

Test the fee calculation at tier boundaries, with discounts applied, across tier transitions, and at extreme amounts. Fees must never be over- or under-charged because of off-by-one or rounding errors.

## Facts to record before writing tests

Write these down at the top of the test file as constants:
- The tier table: thresholds and fee bps per tier. Is each threshold **inclusive** (`>=`) or exclusive?
- Is the fee tiered by **bracket**, like income tax (each slice charged at its own rate), or **flat** per tier (the whole amount charged at the tier's rate)?
- Discount model: a percentage off the fee or off the bps? Can discounts stack? Is there a cap or minimum fee?
- Rounding: floor, ceil, or half-up? In whose favour?
- Amount type (`i128`) and whether negative or zero amounts are rejected.

## Test matrix

### Boundary values
For **each** threshold `T`, test `T - 1`, `T`, and `T + 1`.
| Amount | Expected tier |
|---|---|
| `T - 1` | Lower tier |
| `T` | Depends on inclusivity (assert the documented rule) |
| `T + 1` | Upper tier |

Write this as a table-driven test that loops over the tier table, so adding a tier adds coverage automatically.

### Discount application
| Case | Expected |
|---|---|
| No discount | Base fee |
| 0% discount | Equal to the base fee |
| 100% discount | Fee is 0 (or the minimum fee, if one exists) |
| Discount > 100% or negative | Rejected |
| Stacked discounts (if supported) | Combined as specified (additive or multiplicative) and capped |
| Discount at a tier boundary | Applied after the tier is chosen |

### Tier transitions
- The fee must not drop when the amount goes up (monotonic): for an amount sequence `a < b`, assert `fee(a) <= fee(b)`. Run this across an amount range that spans every tier.
- For a flat-per-tier model, write down the jump at each threshold so it's expected rather than a surprise. For a bracket model, check that the fee is continuous with no jump.
- Moving from tier N to tier N+1 changes the fee by exactly the expected delta.

### Rounding
| Case | Expected |
|---|---|
| Amount where `amount * bps` divides evenly by 10_000 | Exact |
| Remainder just below half | Rounds per spec |
| Remainder exactly half | Rounds per spec (half-up or banker's) |
| Amount so small the fee rounds to 0 | 0, or the minimum fee |
| `fee + net == amount` for many amounts | Always true: no stroop created or lost |

### Edge amounts
- `0`, `1`, `-1` (rejected?)
- `i128::MAX` and large values near it. Check that `amount * bps` doesn't overflow: the code must use `checked_mul` or reorder the operations, and should error cleanly rather than panic.
- An amount exactly at the highest tier threshold

## Structure

```
tests/fee_tiers.rs
  const TIERS: &[(i128 /*threshold*/, u32 /*bps*/)] = &[ ... ];
  mod boundaries / discounts / transitions / rounding / edge_amounts
```

If the fee calculation is a pure function, unit-test it directly. Add a few integration tests through the contract client to confirm it's wired up the same way.

## Verify

```bash
cargo test --test fee_tiers
cargo clippy --all-targets -- -D warnings
```

## Acceptance criteria
- [ ] Every threshold tested at −1, 0, and +1
- [ ] Discounts applied correctly, with invalid discounts rejected
- [ ] Transitions are monotonic and any jumps are documented
- [ ] Rounding is correct, and `fee + net == amount`
- [ ] Overflow is handled without panics
- [ ] All tests pass
