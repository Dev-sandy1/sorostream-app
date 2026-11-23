# #1157 — Metadata version comparison tests

**Repo:** Stellar-split/split-contracts

> This plan was written without access to the contract source. Map the placeholder names (`set_metadata`, `get_metadata`, `get_metadata_version`, `get_metadata_at`, `compare_versions`) to the real API. If the contract has no history or compare functions yet, first confirm with the maintainers whether this issue covers implementing them too.

## Goal

Check that metadata updates increment a version counter, that comparing two versions gives the correct result, that both the current and historical versions can be queried, and that no earlier version is lost.

## Behaviour to pin down first

Answer these before writing tests, because they change the expected values:
- Does the version start at 0 or 1? Does creating the split count as version 1?
- Does writing **identical** metadata bump the version?
- How much history is kept: all of it, the last N versions, or only the current one?
- Comparison semantics: a numeric `u32` comparison, or semver (`major.minor.patch`)? What does it return: `Ordering`, a bool, or a diff?

## Test matrix

### Version tracking
| Case | Expected |
|---|---|
| Fresh split | Version is at its initial value |
| One update | Version is the initial value + 1 |
| N updates | Version is the initial value + N, strictly increasing |
| Update by someone other than the owner | Rejected, version unchanged |
| Event emitted on update | Includes the old and new versions |

### Comparison logic
| Case | Expected |
|---|---|
| `compare(v, v)` | Equal |
| `compare(v1, v2)` with v1 < v2 | Less |
| `compare(v2, v1)` | Greater (antisymmetric) |
| Transitivity across three versions | Holds |
| Semver only: `1.10.0` vs `1.9.0` | Greater (not a string comparison) |
| Comparing against a version that doesn't exist | A defined error, not a panic |

### Version queries
- `get_metadata()` returns the latest version.
- `get_metadata_at(v)` returns exactly what was written at version `v`.
- Querying version 0, a future version, or `u32::MAX` returns a defined error.
- Each query costs about the same regardless of how much history exists. Check `env.budget()` CPU and memory with 1 entry and with 100 entries, and assert the difference stays under a bound. This covers "Queries efficient".

### History
- After N updates, every version from 1 to N can be fetched and matches its input (this covers "No data loss").
- If history is capped, the oldest entries are evicted in order and the newest are kept.
- History of one split is unaffected by updates to another split.
- History survives TTL extension or archival handling, if the contract uses persistent storage with `extend_ttl`.

### Edge cases
- Empty metadata or the maximum allowed size
- Metadata just over the size limit (rejected, version unchanged)
- Version counter near `u32::MAX`. It should error on overflow, not wrap. Set storage directly to reach this state.
- Rapid updates in the same ledger (`env.ledger().set(...)` unchanged)

## Structure

```
tests/metadata_versioning.rs
  mod tracking / comparison / queries / history / edge_cases
```

Use a helper `write_n_versions(&client, split_id, n) -> Vec<Metadata>` that returns the inputs so assertions can compare history against them.

## Verify

```bash
cargo test --test metadata_versioning
cargo clippy --all-targets -- -D warnings
```

## Acceptance criteria
- [ ] Versions tracked and strictly increasing
- [ ] Comparison is correct, including the ordering properties
- [ ] Queries for current and historical versions are correct, with bounded cost
- [ ] History is complete, with no data loss
- [ ] All tests pass
