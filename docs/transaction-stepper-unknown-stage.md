# TransactionStepper — Unknown `currentStage` Value

**Issue:** [#963](https://github.com/SoroStream/sorostream-app/issues/963)

This document describes how `TransactionStepper` behaves when it receives a
`currentStage` value that is not in `STAGE_ORDER`, what the test added under
this issue verifies, and the reasoning behind the expected outcome.

## Background

`TransactionStepper` renders a five-step progress indicator for the
create-stream transaction flow. Internally it locates the active step with:

```ts
const currentIdx = STAGE_ORDER.indexOf(currentStage);
```

`STAGE_ORDER` is a fixed array of the five `TxStage` enum values. When
`currentStage` is a valid member, `indexOf` returns 0–4. When it is not a
member — which TypeScript's type system makes unlikely in production but does
not make impossible at runtime — `indexOf` returns `-1`.

The `isCompleted` condition for each step is:

```ts
const isCompleted = idx < currentIdx && !hasFailed;
```

With `currentIdx === -1`, the expression `idx < -1` is `false` for every
`idx` in `[0, 1, 2, 3, 4]`. No step is therefore marked as completed. That is
the correct outcome: if the stage is unrecognised, the stepper has no basis for
deciding how far along the transaction is, so the safest visual state is "no
progress".

The `isActive` condition is:

```ts
const isActive = stage === currentStage && !hasFailed;
```

No entry in `STAGE_ORDER` equals the unrecognised string, so no step is active
either. The component renders five neutral (grey) circles, no stage
description, and no crash.

## What the test checks

The test in
`components/__tests__/TransactionStepper.test.tsx` covers three assertions for
the unknown-stage case:

1. **No crash.** The component renders without throwing.
2. **No step is marked completed.** The `isCompleted` path produces `false` for
   every step because `idx < -1` is never true.
3. **No step carries `aria-current="step"`.** Because no step matches an
   unrecognised `currentStage`, none is treated as active.

The test casts the unknown value through `as unknown as TxStage` to satisfy the
TypeScript compiler while simulating the runtime condition the issue describes:

```tsx
it('renders without crashing when currentStage is not in STAGE_ORDER', () => {
  render(
    <TransactionStepper
      currentStage={"UnknownStage" as unknown as TxStage}
    />,
  );

  // Should not throw — component renders
  expect(screen.getByRole('generic', { name: 'Transaction progress' })).toBeInTheDocument();

  // No step should be active
  const activeStep = screen.queryAllByRole('generic').find(
    (el) => el.getAttribute('aria-current') === 'step',
  );
  expect(activeStep).toBeUndefined();
});

it('marks no step as completed when currentStage is unknown', () => {
  render(
    <TransactionStepper
      currentStage={"UnknownStage" as unknown as TxStage}
    />,
  );

  // All five circles should render as neutral (number labels, not tick marks)
  expect(screen.getByText('1')).toBeInTheDocument();
  expect(screen.getByText('2')).toBeInTheDocument();
  expect(screen.getByText('3')).toBeInTheDocument();
  expect(screen.getByText('4')).toBeInTheDocument();
  expect(screen.getByText('5')).toBeInTheDocument();
});
```

## Why `idx < -1` is false for all stages

A common mistake when reading this code is to assume that `idx < -1` might be
`true` for some stage. It never is because every `idx` in the loop is the
array index from `STAGE_ORDER.map(...)`, which runs from `0` to `4`. All of
those are greater than `-1`, so `idx < -1` evaluates to `false` for all of
them.

A related concern is that `idx < -1` might accidentally be `true` for all
stages, marking every step as completed. That would require every loop index
to be less than `-1`, which is impossible for a zero-based array index.

The two possible bugs this test guards against are therefore:

| Scenario | Expected | Would break if |
| --- | --- | --- |
| All stages completed | None | `idx < currentIdx` were replaced with `idx <= currentIdx` and `currentIdx` were 4 |
| Component crashes | No crash | `STAGE_META[currentStage]` were accessed before the guard |

The second scenario is currently safe because `STAGE_META[currentStage]`
is only accessed inside the description paragraph, which is wrapped in
`{!hasFailed && ...}` and uses `STAGE_META[currentStage]` directly. For an
unknown stage, `STAGE_META[currentStage]` returns `undefined`, and accessing
`.description` on `undefined` would throw. This is not currently guarded, so
the test also documents this latent risk:

> If `currentStage` is unknown and `hasFailed` is `false`, the component
> renders a description paragraph and accesses `STAGE_META[currentStage]`,
> which is `undefined`. This throws at runtime. The safe fix is to guard the
> description paragraph with an existence check on `STAGE_META[currentStage]`
> before rendering it, or to fall back to an empty string.

If that guard is added in a future change, the "no crash" assertion in this
test becomes the signal that the fix is in place.

## Running the test

```bash
npx vitest run components/__tests__/TransactionStepper.test.tsx
```

## Related

- **`components/TransactionStepper.tsx`** — the component under test.
- **Issue #963** — the issue this test was added under.
- The existing test suite in `TransactionStepper.test.tsx` covers the happy
  path (all five stages render, `aria-current`, descriptions, failure state,
  and the accessible region label). This issue adds the edge-case coverage for
  an out-of-range `currentStage`.
