# #1158 — Dynamic recipient management tests

**Repo:** Stellar-split/split-contracts

> This plan was written without access to the contract source. Replace the placeholder names (`add_recipient`, `remove_recipient`, `get_recipients`, `Error::*`) with the contract's real entry points and error variants.

## Goal

Test adding and removing recipients on an existing split. The split's state must stay valid after every operation: no orphaned shares, no duplicates, and no share totals that don't add up.

## Invariants to assert after every operation

These are the "consistency" and "no corruption" criteria. Put them in a single helper, e.g. `assert_split_invariants(&env, &client, split_id)`:

1. Recipient shares sum to exactly the total basis points (e.g. `10_000`), or whatever the contract treats as 100%.
2. No recipient address appears twice.
3. `recipient_count` (if it's stored) equals `get_recipients().len()`.
4. Balances that haven't been distributed are unchanged by add/remove. Removing a recipient must not lose any amount already owed to them.
5. Stored indexes or maps agree with the list. If there's a lookup by address, every listed recipient can be found and a removed one can't.

## Test matrix

### Add operations
| Case | Expected |
|---|---|
| Add a recipient to a split that has room, then rebalance shares | Recipient present, invariants hold |
| Add several recipients in sequence | Order is stable (if the contract defines one) and invariants hold |
| Add, then distribute | New recipient receives its correct share |
| Event emitted on add | Topic and data match |

### Remove operations
| Case | Expected |
|---|---|
| Remove a middle recipient | Remaining order and indexes stay correct |
| Remove the first or last recipient | No off-by-one errors |
| Remove, then distribute | Removed address receives 0, the others get the rebalanced amounts |
| Remove a recipient with an unclaimed balance | Balance is paid out, kept claimable, or the call errors. Pin down which one the spec requires |
| Event emitted on remove | Topic and data match |

### Validation
| Case | Expected error |
|---|---|
| Called by someone other than the owner/admin (`env.mock_auths` without the owner) | Auth failure |
| Add a duplicate address | `Error::DuplicateRecipient` |
| Remove an address that isn't a recipient | `Error::RecipientNotFound` |
| Add with share = 0 | Rejected (if the contract disallows zero shares) |
| Shares that would no longer sum to 100% | `Error::InvalidShares` |
| Add beyond the max recipient count | `Error::TooManyRecipients` |
| Operating on a split that doesn't exist | `Error::SplitNotFound` |

Every failing call must leave state **unchanged**. Snapshot `get_recipients()` before the call and compare it afterwards.

### Edge cases
- Remove down to a single recipient, then try to remove that last one (should it be allowed?)
- Add then remove the same address, then add it again. There should be no stale data.
- Fill the split to the maximum, remove one, and add one.
- Operations on a split that is paused, finalized, or locked (if those states exist)
- An address that is a contract rather than an account (if that matters)
- A large number of add/remove cycles, followed by the invariant check. This is a cheap fuzz.

## Structure

```
tests/
  recipient_management.rs
    mod add { ... }
    mod remove { ... }
    mod validation { ... }
    mod edge_cases { ... }
  common/mod.rs   // setup(), create_split(), assert_split_invariants()
```

Use `#[should_panic(expected = "...")]` or `try_*` client methods with `assert_eq!(res, Err(Ok(Error::X)))`, whichever the repo already uses.

## Verify

```bash
cargo test --test recipient_management
cargo clippy --all-targets -- -D warnings
```

## Acceptance criteria
- [ ] Add/remove work and are covered
- [ ] Every validation error path is tested
- [ ] Invariant helper runs after every mutation
- [ ] Edge cases above are covered
- [ ] All tests pass, with no state corruption on failed calls
