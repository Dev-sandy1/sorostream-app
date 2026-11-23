# #972 — DelegatesSection: a fast double submit must not add the same delegate twice

**Component:** [components/DelegatesSection.tsx](../../components/DelegatesSection.tsx)
**Test file:** [components/__tests__/DelegatesSection.test.tsx](../../components/__tests__/DelegatesSection.test.tsx) (it already has `ADDR_A`, a `localStorage` mock and `storedDelegates()`)

## Behaviour under test

`handleAdd` does the following:

1. Calls `validate(input)`, including the duplicate check against `delegates`.
2. Calls `setAdding(true)`, which disables the button and changes its label to `Adding…`.
3. Waits `await new Promise(r => setTimeout(r, 500))`, the simulated SDK call.
4. Builds `next = [...delegates, newDelegate]`, saves it, and resets `input` and `adding`.

During the 500 ms wait, `delegates` does not contain the new address yet, so `validate` cannot catch a second submit. Only `disabled={adding}` stops one.

## Tests

Use `vi.useFakeTimers()` so the test controls the 500 ms wait, and `vi.useRealTimers()` in `afterEach`.

1. **Double click**
   - Type `ADDR_A`, then click `Add Delegate` twice in a row with no timer advance between the clicks.
   - After the first click the button is `toBeDisabled()` and its text is `Adding…`.
   - Run `await act(() => vi.advanceTimersByTimeAsync(500))`.
   - Expect one `<li>` in the list, `storedDelegates()` to have length 1, and the button back to `Add Delegate` and enabled.
2. **Click, then wait out the delay, then click again**
   - Type `ADDR_A` and add it, advance 500 ms, type `ADDR_A` again and click.
   - Expect `This address is already a delegate.` and still one delegate.
   - This shows `validate` takes over once the wait is over.

## Possible gap to check: Enter key

`onKeyDown` calls `handleAdd()` on Enter whether or not `adding` is true, so the input field is not disabled. Add a third test that presses Enter twice on the input during the 500 ms window.

- Both calls close over the same stale `delegates`, so each one writes `old + ADDR_A`. That should still give one entry, but each call starts its own timer and writes to storage twice.
- If the test finds a duplicate, or you want to guard it, add a ref guard such as `if (addingRef.current) return;` at the top of `handleAdd`. A ref is needed because `adding` state is stale inside the same tick. Mention the fix in the PR.

## Done when

- The tests pass with `npx vitest run components/__tests__/DelegatesSection.test.tsx`
- `npm run lint` and `npx tsc --noEmit` are clean
- The PR description says `Closes #972`
