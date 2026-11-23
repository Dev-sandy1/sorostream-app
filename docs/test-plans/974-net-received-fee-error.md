# #974 — NetReceivedDisplay: a failed fee fetch for a new amount shows the error, not the old fee

**Component:** [components/NetReceivedDisplay.tsx](../../components/NetReceivedDisplay.tsx)
**Test file:** [components/__tests__/NetReceivedDisplay.test.tsx](../../components/__tests__/NetReceivedDisplay.test.tsx). It has no render tests today, so this test needs a `vi.mock("@/src/lib/sorostream")` that overrides only `getFeeConfig` and keeps the real `parseStroops` and `formatUSDC`. The same pattern is used in `FeeEstimationPanel.test.tsx`.

## Behaviour under test

The effect runs on `[amount]`. For a positive amount it does the following:

- sets `let active = true`, `feeLoading = true` and `feeError = false`
- calls `getFeeConfig()`
- on success, `setFeeBasisPoints(basisPoints)`
- on failure, `setFeeError(true)` and `setFeeBasisPoints(0)`
- in `finally`, clears `feeLoading`
- each of these state writes happens only `if (active)`, and the cleanup sets `active = false`

When `feeError` is true, the component shows `Fee config unavailable` in place of the net amount, and `Could not load fee information` in place of the fee description.

## Test

1. The first `getFeeConfig` call resolves `{ basisPoints: 250 }`. The second one rejects.
2. Render `amount="100"`. Wait for the net amount (`97.5`) and the 2.5% fee text to show.
3. `rerender` with `amount="200"`.
4. Wait for `Fee config unavailable` and `Could not load fee information`.
5. Expect none of these on screen: `97.5`, `195` (the stale 2.5% applied to the new amount), `2.5`, or `(5 … fee)`.

The fee description goes through `useTranslations("stream_new")`. Either render inside the i18n provider the app uses, or assert on the fee amount text, `(… fee)`, and on the error strings, which are hardcoded.

## Optional: an out-of-order case

Make the first `getFeeConfig` call slow (a deferred) and the second one reject. Change the amount before the first call resolves, then resolve it with `250`. The error state must stay, because the `active` flag drops the stale result. This checks the cancellation directly, not only the reset.

## Done when

- The test passes with `npx vitest run components/__tests__/NetReceivedDisplay.test.tsx`
- `npm run lint` and `npx tsc --noEmit` are clean
- The PR description says `Closes #974`
