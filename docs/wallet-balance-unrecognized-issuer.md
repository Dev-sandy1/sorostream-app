# WalletBalanceDisplay — Unrecognized Assets and Issuer Matching

**Issue:** [#966](https://github.com/SoroStream/sorostream-app/issues/966)

This document describes how `WalletBalanceDisplay` handles assets that share a
symbol with a supported token but come from a different issuer, what the test
added under this issue verifies, and the invariants that any change to
`fetchBalances` must preserve.

## The matching rule

`fetchBalances` matches a Horizon balance against `SUPPORTED_TOKENS` by
checking **both** `asset_code` and `asset_issuer`:

```ts
const matching = data.balances?.find(
  (b) =>
    isCreditAsset(b) &&
    b.asset_code === token.symbol &&
    b.asset_issuer === token.issuer
);
```

A balance that matches on `asset_code` alone but not on `asset_issuer` does not
satisfy this predicate. It falls through to the second loop, which collects any
credit asset that is not an exact match for a supported token and tags it
`unrecognized: true`:

```ts
for (const b of data.balances ?? []) {
  if (!isCreditAsset(b) || !b.asset_code || !b.asset_issuer) continue;
  const supported = SUPPORTED_TOKENS.some(
    (t) => t.symbol === b.asset_code && t.issuer === b.asset_issuer
  );
  if (supported) continue;
  tokenBalances.push({
    symbol: b.asset_code,
    balance: b.balance,
    issuer: b.asset_issuer,
    unrecognized: true,
  });
}
```

The unrecognized asset is therefore always shown as its own row — with its own
issuer in the display — and never merged into the balance of the supported token
that happens to share its code.

## What the test checks

The test is in
`components/__tests__/WalletBalanceDisplay.assets.test.tsx`, which already
covers the general unrecognized-asset case. The test added under this issue
specifically targets the collision scenario: a Horizon response that contains a
`USDC` balance from an issuer that is not the USDC issuer in `SUPPORTED_TOKENS`.

```tsx
it('shows a fake-USDC as an unrecognized row and never adds it to the real USDC balance', async () => {
  const REAL_USDC_ISSUER = 'CAQCFVLOBK5GIULPNZRGATJJMIZL5BSP7X5YJVMGCPTUEPFM4AVSRCJU';
  const FAKE_USDC_ISSUER = 'GDUI7JVKWZV4JFBS2JJOBBWJZNMFZUQBFKR4YLIZOIU3LFN26ZGXLWL';

  horizonReturns([
    { asset_type: 'native', balance: '50.0000000' },
    // The real USDC — the wallet holds a small amount
    {
      asset_type: 'credit_alphanum4',
      asset_code: 'USDC',
      asset_issuer: REAL_USDC_ISSUER,
      balance: '10.0000000',
    },
    // A USDC clone from a different issuer
    {
      asset_type: 'credit_alphanum4',
      asset_code: 'USDC',
      asset_issuer: FAKE_USDC_ISSUER,
      balance: '500.0000000',
    },
  ]);

  render(<WalletBalanceDisplay address={ADDRESS} />);
  await openDropdown();

  // The real USDC row shows 10.00 — the fake balance is NOT added to it.
  const rows = screen.getAllByText('USDC');
  // Two USDC rows: the supported one and the unrecognized clone.
  expect(rows.length).toBeGreaterThanOrEqual(2);

  // The unrecognized row carries data-testid="unrecognized-balance".
  const unrecognizedRow = screen.getByTestId('unrecognized-balance');
  expect(unrecognizedRow).toHaveTextContent('USDC');
  // Shows a short form of the fake issuer, not the real one.
  expect(unrecognizedRow).toHaveTextContent('(GDUI…LWWL)');

  // The supported USDC row shows only 10.00, not 510.00.
  expect(screen.getByText('10.00')).toBeInTheDocument();
  expect(screen.queryByText('510.00')).not.toBeInTheDocument();
  expect(screen.queryByText('500.00')).not.toBeInTheDocument(); // raw balance not merged
});
```

A second, simpler case confirms that when the wallet holds the USDC clone but
not the real USDC at all, only one unrecognized row appears and no supported
USDC row is fabricated:

```tsx
it('does not fabricate a supported USDC row when only the clone is held', async () => {
  horizonReturns([
    { asset_type: 'native', balance: '5.0000000' },
    {
      asset_type: 'credit_alphanum4',
      asset_code: 'USDC',
      asset_issuer: 'GDUI7JVKWZV4JFBS2JJOBBWJZNMFZUQBFKR4YLIZOIU3LFN26ZGXLWL',
      balance: '100.0000000',
    },
  ]);

  render(<WalletBalanceDisplay address={ADDRESS} />);
  await openDropdown();

  // No supported USDC row — the wallet has no trustline for the real one.
  // The clone appears as an unrecognized row only.
  expect(screen.getByTestId('unrecognized-balance')).toBeInTheDocument();

  // The "+N" badge shows 1 (for the unrecognized row), not 0 or 2.
  const button = screen.getByRole('button', { name: 'wallet_balance' });
  expect(button).toHaveTextContent('+1');
});
```

## Invariants

Any change to `fetchBalances` must preserve these:

1. **Issuer match is mandatory.** A credit asset is only considered a match for
   a supported token when both `asset_code` and `asset_issuer` are equal.
   Matching on code alone allows attackers to phish users by flooding their
   accounts with a fake version of a supported token that appears as a
   legitimate balance.

2. **No balance merging across issuers.** The balance of an unrecognized asset
   is never added to the balance of the supported token that shares its code.
   These are economically distinct assets; their balances cannot be summed.

3. **Every credit asset the account holds is shown.** An unrecognized asset is
   displayed as its own row, never silently dropped. The user must be able to
   see all assets in their account.

4. **The short-issuer label is derived from the unrecognized asset's own
   issuer**, not from any supported token's issuer. The first four and last
   four characters of the issuer public key are shown so the user can manually
   verify the source.

## Running the tests

```bash
npx vitest run components/__tests__/WalletBalanceDisplay.assets.test.tsx
```

## Related

- **`components/WalletBalanceDisplay.tsx`** — the component under test,
  specifically `fetchBalances` and the `SUPPORTED_TOKENS` array.
- **Issue #966** — the issue this test was added under.
- The existing tests in `WalletBalanceDisplay.assets.test.tsx` cover the
  general unsupported-asset case (a code not in `SUPPORTED_TOKENS` at all) and
  the same-code / different-issuer case using `AQUA`. This issue adds explicit
  coverage for the `USDC` collision, which is the most likely phishing vector
  given USDC's prevalence on Stellar.
