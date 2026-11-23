# #973 — FeeEstimationPanel: a stale estimate from an earlier activation is never applied

**Component:** [components/FeeEstimationPanel.tsx](../../components/FeeEstimationPanel.tsx)
**Test file:** [components/__tests__/FeeEstimationPanel.test.tsx](../../components/__tests__/FeeEstimationPanel.test.tsx) (it already has `mockSimulate` and `makeFeeEstimate()`)

## Behaviour under test

The effect runs on `[active]`:

- When `active` is false it returns early, and `if (!active) return null;` renders nothing.
- When `active` is true it sets `let cancelled = false`, sets `loading`, and calls `simulateTransactionFee()`. Both `.then` and `.catch` write state only `if (!cancelled)`.
- The cleanup sets `cancelled = true`. It runs when `active` flips to false and again on unmount.

So a request started by an earlier activation must never overwrite state, even if it resolves last.

## Test: toggle false → true → false → true with staggered results

1. Make a promise you can resolve by hand, called a "deferred". Give `mockSimulate` two of them with `mockReturnValueOnce`:
   - `first`, which will resolve **late** with `makeFeeEstimate({ totalFeeLumens: "1.0000000" })`
   - `second`, which will resolve **early** with `makeFeeEstimate({ totalFeeLumens: "2.0000000" })`
2. Render `active={false}`, then `rerender` with `true`, then `false`, then `true`.
3. Expect `mockSimulate` to have been called twice.
4. Resolve `second` inside `act`. Expect `2.0000000 XLM` on screen.
5. Resolve `first` inside `act`. Expect `2.0000000 XLM` still on screen and `1.0000000 XLM` absent.

Do the opposite order as a second case: resolve `first` before `second`. The loading skeleton must stay up (`aria-busy="true"`) until `second` resolves, and `1.0000000` must never appear.

A third, optional case: a late **rejection** from `first` must not replace a successful `second` with `Fee estimate unavailable`.

## Notes

- Deferreds are easier to follow than staggered `setTimeout` mocks plus fake timers. If you do use timers, `mockImplementationOnce(() => new Promise(r => setTimeout(() => r(x), ms)))` works.
- Resolve each deferred and flush it inside `await act(async () => { ... })` to avoid act warnings.

## Done when

- The tests pass with `npx vitest run components/__tests__/FeeEstimationPanel.test.tsx`
- `npm run lint` and `npx tsc --noEmit` are clean
- The PR description says `Closes #973`
