# #976 — OnboardingWizard: `skip()` always lands on the "done" step

**Component:** [`components/OnboardingWizard.tsx`](../../components/OnboardingWizard.tsx)
**Existing tests:** `components/__tests__/OnboardingWizard.test.tsx` (it covers skip from the first step only)

## Behaviour under test

```ts
const skip = useCallback(() => {
  trackEvent({ type: "onboarding_skip", step, stepId: current.id });
  setStep(STEPS.length - 1);
}, [step, current.id]);
```

Steps: `welcome` → `connect-wallet` → `choose-stream` → `confirm` → `done`.
`skip()` sets the step to the last index (4) every time, whatever the current
step is. `address` comes from `useWallet()` and skip never changes it.

## Test plan

Mock `useWallet`, `useRouter` and `trackEvent`, and clear `localStorage` so the wizard opens.

| # | Start step | How to reach it | Expectation after clicking **Skip** |
|---|-----------|-----------------|-------------------------------------|
| 1 | `welcome` | initial render | "You're all set 🎉", "Step 5 of 5", button "Go to dashboard" |
| 2 | `connect-wallet` | click "Get started" | same as (1) |
| 3 | `choose-stream` | mock `address` set, advance twice | same as (1) |
| 4 | `confirm` | advance three times | same as (1) |

Write this as `it.each` over the four start steps, and render the "done" content
from one shared assertion helper so all four cases are checked the same way.

For each case, also check:
- `trackEvent` was called with `{ type: "onboarding_skip", step: <index>, stepId: <id> }`.
- `markComplete` has **not** run yet (`localStorage` is not `"true"`). Skip only
  moves to the last step, and **Go to dashboard** is what finishes the wizard.

### connect-wallet consistency cases

| # | Scenario | Expectation |
|---|----------|-------------|
| 5 | `address = null`, skip from `connect-wallet` | Done step renders; no "Wallet connected" banner; `connect` **never called** |
| 6 | `connect()` pending (`isConnecting = true`), then skip, then `connect()` resolves `true` | Still on the done step (`Math.min(s + 1, 4)` keeps it at 4); no crash |
| 7 | `connect()` resolves `false` / rejects **after** skip | See note below |
| 8 | Connection fails (error shown), **then** skip | See note below |

## Notes / possible defect

`skip()` does **not** call `setError(null)`. In cases 7 and 8, the message
"Wallet connection was not completed…" or "Could not connect to Freighter…"
ends up on the "You're all set" step. Either assert this behaviour as it is
today, or clear the error in `skip()` and assert that no `role="alert"` is
present on the done step. The second option fits the issue's "no inconsistent
state" wording better.
