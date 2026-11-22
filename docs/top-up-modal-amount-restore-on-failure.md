# TopUpModal — Amount Restoration After a Failed Confirmation

**Issue:** [#962](https://github.com/SoroStream/sorostream-app/issues/962)

This document explains the optimistic-clear / restore pattern in
`TopUpModal.handleConfirm`, the edge case it guards against, and what the test
for this issue must verify.

## The pattern

`handleConfirm` in `components/TopUpModal.tsx` follows a three-step sequence:

```ts
const handleConfirm = async () => {
  // 1. Validate.
  const parsedAmount = parseFloat(amount);
  if (!amount || !(parsedAmount > 0)) {
    setError("Please enter a valid amount greater than 0");
    return;
  }

  // 2. Snapshot and clear before the async call, so a stale value is never
  //    left behind after a successful submission (#546).
  const submitted = amount;
  setError("");
  setAmount("");

  try {
    await onConfirm(submitted);
    onClose();         // success → close
  } catch (err) {
    // 3. Restore on failure so the user can correct and retry.
    setAmount(submitted);
    setError(err instanceof Error ? err.message : "Failed to top-up stream");
  }
};
```

The field is cleared *before* `onConfirm` is awaited. If the promise resolves,
the modal closes and the cleared field is never seen. If the promise rejects,
the catch block puts the original amount back and surfaces the error message.

Without the restore in the catch block, a user whose transaction is rejected
(for example, due to insufficient balance or a network error) would see a blank
amount field. They would have to remember and retype the value they just
entered — a poor experience, particularly for precise decimal amounts.

## Relationship to issue #546

Issue #546 introduced the optimistic-clear to prevent a stale amount from
showing in the input when the modal is re-opened after a successful top-up.
The catch-block restore was added in the same change as the other half of that
fix: the field is empty on success and re-populated on failure.

The existing test `keeps the amount and shows the error when the top-up fails`
in `TopUpModal.test.tsx` already covers this path:

```tsx
it('keeps the amount and shows the error when the top-up fails', async () => {
  const onConfirm = vi.fn().mockRejectedValue(new Error('insufficient balance'));
  renderModal(onConfirm);

  fireEvent.change(input(), { target: { value: '42' } });
  fireEvent.click(screen.getByRole('button', { name: 'Confirm Top-up' }));

  await waitFor(() => {
    expect(screen.getByRole('alert')).toHaveTextContent('insufficient balance');
  });
  expect(input().value).toBe('42');
});
```

This issue asks for explicit documentation of that test, its intent, and the
invariants it protects, so that future contributors do not remove the restore
under the impression that it is redundant with the optimistic-clear.

## What additional tests should cover

The existing test above is necessary but not exhaustive. Issue #962 also asks
for confirmation that:

1. The amount is restored even when the error message is a generic string
   (i.e. when the rejection value is not an `Error` instance).
2. The field is *not* pre-populated when the modal is re-opened after a
   failure — opening should always start from blank (issue #546's invariant).

```tsx
it('restores the amount even when the rejection is a non-Error value (#962)', async () => {
  // Some callers reject with a plain string or object rather than an Error.
  const onConfirm = vi.fn().mockRejectedValue('server unavailable');
  renderModal(onConfirm);

  fireEvent.change(input(), { target: { value: '7.5' } });
  fireEvent.click(screen.getByRole('button', { name: 'Confirm Top-up' }));

  await waitFor(() => {
    expect(screen.getByRole('alert')).toHaveTextContent('Failed to top-up stream');
  });
  // The amount must still be in the field.
  expect(input().value).toBe('7.5');
});

it('opens with a blank field even after a previous failure (#962)', async () => {
  const onConfirm = vi.fn().mockRejectedValue(new Error('rejected'));
  const { rerender } = renderModal(onConfirm);

  fireEvent.change(input(), { target: { value: '99' } });
  fireEvent.click(screen.getByRole('button', { name: 'Confirm Top-up' }));

  // Wait for the restore.
  await waitFor(() => expect(input().value).toBe('99'));

  // Simulate closing and re-opening the modal.
  rerender(
    <TopUpModal open={false} onClose={vi.fn()} onConfirm={onConfirm} token="USDC" />,
  );
  rerender(
    <TopUpModal open onClose={vi.fn()} onConfirm={onConfirm} token="USDC" />,
  );

  // A fresh open must always start blank, regardless of prior failures.
  expect(input().value).toBe('');
});
```

## Invariants

Any change to `handleConfirm` must preserve these:

1. **Clear before the async call.** `setAmount("")` must run before
   `await onConfirm(...)` so that a successful confirmation never leaves the
   old value behind.
2. **Restore in the catch block.** `setAmount(submitted)` must run when
   `onConfirm` rejects, using the value snapshotted before the clear — not
   the current state, which is `""`.
3. **Re-open always starts blank.** The `useEffect` on `open` resets `amount`
   and `error` whenever the modal is opened, so a restored value from a failed
   attempt is cleared when the modal is next opened.

## Running the tests

```bash
npx vitest run components/__tests__/TopUpModal.test.tsx
```

## Related

- **`components/TopUpModal.tsx`** — `handleConfirm`, specifically the
  `setAmount("")` before `await onConfirm(submitted)` and the
  `setAmount(submitted)` inside the catch block.
- **`components/__tests__/TopUpModal.test.tsx`** — the existing test suite;
  the `keeps the amount and shows the error when the top-up fails` test covers
  the core path for this issue.
- **Issue #962** — the issue this documentation was added under.
- **Issue #546** — introduced the optimistic-clear pattern that made the
  catch-block restore necessary.
