# #967 — WithdrawConfirmModal: Confirm re-enables after a failed confirm

**Repo:** SoroStream/sorostream-app
**Files:** [components/WithdrawConfirmModal.tsx](../../components/WithdrawConfirmModal.tsx), [components/__tests__/WithdrawConfirmModal.test.tsx](../../components/__tests__/WithdrawConfirmModal.test.tsx), [src/app/stream/[id]/page.tsx](../../src/app/stream/[id]/page.tsx)

## Goal

If `onConfirm` rejects, the Confirm button must become enabled again on its own. The user should not have to retype the amount, because `typed` is not cleared when a confirm fails.

## Current state (read this first)

The issue was written against a version of the modal that doesn't exist yet. Today:

| Issue assumes | Actual code |
|---|---|
| `confirmDisabled = !matches \|\| isSubmitting` | `disabled={!matches}`, with no submitting state |
| `onConfirm` may return a rejecting promise | `onConfirm: () => void`, called synchronously |
| Modal stays open during submission | The page closes the modal *before* the withdraw runs: `onConfirm={() => { setWithdrawConfirmAmount(null); void executeWithdraw(); }}` |

So the work comes in two parts. First add the submitting behaviour, then test it.

## Implementation

1. **Widen the prop type**
   ```ts
   onConfirm: () => void | Promise<unknown>;
   ```
   Existing synchronous callers keep working.

2. **Add submitting state to the modal**
   ```ts
   const [isSubmitting, setIsSubmitting] = useState(false);
   const matches = typed === amount;
   const confirmDisabled = !matches || isSubmitting;

   async function handleConfirm() {
     if (confirmDisabled) return;
     setIsSubmitting(true);
     try {
       await onConfirm();
     } catch {
       // Parent owns error reporting (toast etc.). Keep `typed` so the user can retry.
     } finally {
       setIsSubmitting(false);
     }
   }
   ```
   - Use `disabled={confirmDisabled}` and `onClick={handleConfirm}`.
   - **Do not** call `setTyped("")` anywhere on the failure path.
   - Optional: set `aria-busy={isSubmitting}` on the button and change the label (e.g. "Withdrawing…"). If you change the label, update the button-name queries in the existing tests.
   - If the parent unmounts the modal on success, `setIsSubmitting(false)` in `finally` runs after unmount. React 18 doesn't warn about this. Use a mounted ref only if lint asks for one.

3. **Page integration (optional, but makes the feature real)**
   In `src/app/stream/[id]/page.tsx`, keep the modal open until the withdraw finishes:
   ```tsx
   onConfirm={async () => {
     await executeWithdraw();          // must reject on failure
     setWithdrawConfirmAmount(null);   // close only on success
   }}
   ```
   Check that `executeWithdraw` actually rethrows or rejects. If it swallows errors and only shows a toast, the modal will close on failure as well, so either rethrow or return a success flag.

## Tests to add

Add these to `components/__tests__/WithdrawConfirmModal.test.tsx`. That file already has `AMOUNT` and `renderModal(overrides)`.

```tsx
import { act, waitFor } from '@testing-library/react';

it('re-enables Confirm after onConfirm rejects, without retyping', async () => {
  let reject!: (e: unknown) => void;
  const onConfirm = vi.fn(() => new Promise<void>((_, r) => { reject = r; }));
  renderModal({ onConfirm });

  const input = screen.getByRole('textbox');
  const button = screen.getByRole('button', { name: /confirm withdrawal/i });

  fireEvent.change(input, { target: { value: AMOUNT } });
  fireEvent.click(button);

  // While in flight: disabled
  expect(button).toBeDisabled();

  await act(async () => { reject(new Error('tx failed')); });

  // After failure: enabled again, text preserved
  await waitFor(() => expect(button).not.toBeDisabled());
  expect(input).toHaveValue(AMOUNT);
  expect(onConfirm).toHaveBeenCalledTimes(1);
});

it('allows a retry after a failed confirm', async () => {
  const onConfirm = vi.fn()
    .mockRejectedValueOnce(new Error('tx failed'))
    .mockResolvedValueOnce(undefined);
  renderModal({ onConfirm });

  fireEvent.change(screen.getByRole('textbox'), { target: { value: AMOUNT } });
  const button = screen.getByRole('button', { name: /confirm withdrawal/i });

  fireEvent.click(button);
  await waitFor(() => expect(button).not.toBeDisabled());

  fireEvent.click(button);
  await waitFor(() => expect(onConfirm).toHaveBeenCalledTimes(2));
});

it('ignores double clicks while submitting', async () => {
  const onConfirm = vi.fn(() => new Promise<void>(() => {})); // never settles
  renderModal({ onConfirm });

  fireEvent.change(screen.getByRole('textbox'), { target: { value: AMOUNT } });
  const button = screen.getByRole('button', { name: /confirm withdrawal/i });
  fireEvent.click(button);
  fireEvent.click(button);

  expect(onConfirm).toHaveBeenCalledTimes(1);
});
```

The existing test `calls onConfirm when Confirm is clicked after correct input` uses a sync `vi.fn()` and should still pass unchanged.

## Verify

```bash
npx vitest run components/__tests__/WithdrawConfirmModal.test.tsx
npm run lint
npx tsc --noEmit
```

## Acceptance criteria

- [ ] `isSubmitting` and `confirmDisabled` implemented in the modal
- [ ] Rejected `onConfirm` → button re-enabled, input value preserved
- [ ] New tests pass, and existing tests still pass
- [ ] Lint and type checks pass
- [ ] PR description includes `Closes #967`
