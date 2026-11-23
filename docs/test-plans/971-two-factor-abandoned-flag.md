# #971 — TwoFactorSetup: abandoned flag across a completed and an abandoned attempt

**Component:** [components/TwoFactorSetup.tsx](../../components/TwoFactorSetup.tsx)
**Test file:** [components/__tests__/TwoFactorSetup.restart.test.tsx](../../components/__tests__/TwoFactorSetup.restart.test.tsx) (it already mocks `qrcode` and `@/src/lib/totp`, and has a `renderSetup()` helper)

## Behaviour under test

The flag is stored in `localStorage` under `ABANDONED_SETUP_KEY` (`sorostream-2fa-abandoned-setup`). `"1"` means abandoned. A missing key means not abandoned.

| Moment | What sets it | Result |
| --- | --- | --- |
| Scan step first shown | the secret effect sets `secretShownRef.current = true` | nothing is written yet |
| Step becomes `complete` | the complete-step effect clears the ref and calls `writeAbandonedFlag(false)` | key removed |
| Unmount | the cleanup writes `true` only `if (secretShownRef.current)` | key is `"1"` only if a secret was shown and setup did not complete |

The existing test `completing setup clears the stale-entry warning` starts with the flag already set and checks it only after unmount. The issue asks for two more tests.

## Test 1: the happy path never sets the flag

1. Start with `localStorage` clear. Use `vi.useFakeTimers()` because verify and save each wait on a 1000 ms `setTimeout`.
2. Render, then check `localStorage.getItem(ABANDONED_SETUP_KEY)` is `null` after **each** step:
   - intro, then click `Get Started`
   - scan, then click `/Scanned the QR Code/`
   - verify: enter `123456`, click `Verify Code`, `await vi.advanceTimersByTimeAsync(1000)` inside `act`
   - recovery: click `/Saved My Codes/` and advance 1000 ms
   - complete: `2FA Enabled` is shown
3. Unmount and check the key is still `null`. This shows the unmount cleanup does not fire after completion.
4. Optional: render again, click `Get Started`, and check the stale warning is not shown.

## Test 2: unmounting mid-flow sets the flag

1. Start with `localStorage` clear, render, click `Get Started` so the secret is shown.
2. Move on to verify (or recovery), then `unmount()` without completing.
3. Check `localStorage.getItem(ABANDONED_SETUP_KEY)` is `"1"`.

The existing `navigating away mid-setup counts as abandoning` checks this only through the warning on a second mount. This test checks the stored value directly. A recovery-step variant is worth adding, because that is the last step before the ref is cleared.

## Done when

- Both tests pass with `npx vitest run components/__tests__/TwoFactorSetup.restart.test.tsx`
- `npm run lint` and `npx tsc --noEmit` are clean
- The PR description says `Closes #971`
