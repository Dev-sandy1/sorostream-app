# #977 — WalletBalanceDisplay: ✓ only on a genuine balance change

**Component:** [`components/WalletBalanceDisplay.tsx`](../../components/WalletBalanceDisplay.tsx)
**Existing tests:** `src/lib/__tests__/walletBalanceDisplay.test.tsx`, `components/__tests__/WalletBalanceDisplay.assets.test.tsx`

## Behaviour under test

`fetchBalances` serialises the parsed balances and compares them with
`prevBalancesRef.current`:

```ts
const balancesStr = JSON.stringify(tokenBalances);
if (balancesStr !== prevBalancesRef.current) {
  setBalances(tokenBalances);
  if (prevBalancesRef.current) {      // "" on first fetch → falsy
    setBalanceUpdated(true);           // shows the ✓ for 2s
    ...
    return ...;
  }
  prevBalancesRef.current = balancesStr;
}
```

The ref starts out as `""`, so the first successful fetch fills it without
setting `balanceUpdated`. Only a later fetch whose result is different turns on
the ✓ (`<span aria-live="polite">✓</span>`), which clears after 2000 ms.

## Test plan

Use fake timers and a mocked `fetch` that returns a Horizon `/accounts` payload.

| # | Scenario | Expectation |
|---|----------|-------------|
| 1 | Mount with an address; first fetch resolves with `XLM = 100` | XLM balance renders; **no ✓** at any point (also check after the fetch settles) |
| 2 | Advance 60 s; second fetch returns the **same** payload | Still **no ✓** |
| 3 | Advance 60 s; fetch returns a **different** payload (`XLM = 150`) | ✓ appears; new balance renders |
| 4 | Advance 2000 ms after (3) | ✓ disappears |
| 5 | `address` → `null` → a new address (the ref resets to `""`) | First fetch for the new address shows **no ✓** |

Suggested assertions: `screen.queryByText("✓")` should be `null` for cases 1, 2 and 5, and
`screen.getByText("✓")` should find the indicator for case 3.

## Notes / possible defect

When balances change, the code `return`s **before** it runs
`prevBalancesRef.current = balancesStr`. The ref keeps the *first* snapshot, so
every later poll that returns the new (unchanged) balances still differs from
the ref and flashes the ✓ again. The `return () => clearTimeout(timer)` also
does nothing, because nothing uses the return value of an async callback.

A test that does **change → same → same** shows this: the third fetch should
not flash the ✓, but today it will. Decide whether #977 should only document
this or also fix it. The fix is to update the ref before the early return.
