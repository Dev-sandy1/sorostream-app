# StreamTimeline — Zero-Duration Stream Progress

**Issue:** [#961](https://github.com/SoroStream/sorostream-app/issues/961)

This document describes the divide-by-zero edge case in `StreamTimeline`'s
progress calculation when `startTime` equals `endTime`, what a correct result
looks like, and what the test for this issue must verify.

## The problem

`StreamTimeline` computes the progress bar width as:

```ts
const progress = Math.min(100, Math.max(0, ((now - start) / (end - start)) * 100));
```

When `end === start`, the denominator is `0`. JavaScript division by zero
produces `Infinity` or `NaN` depending on the numerator:

| `now` vs `start` | `(now - start)` | result         | after `Math.min/max` |
|------------------|-----------------|----------------|----------------------|
| `now > start`    | positive        | `Infinity`     | `100`                |
| `now === start`  | `0`             | `NaN`          | `NaN` — clamp fails  |
| `now < start`    | negative        | `-Infinity`    | `0`                  |

The `NaN` case is the dangerous one: `Math.min(100, Math.max(0, NaN))` returns
`NaN`, which renders as `width: NaN%` in the DOM — an invalid CSS value. The
browser ignores it, leaving the progress bar at whatever width it inherited from
a previous render, or at `0%` if none exists.

The correct semantics for a zero-duration stream is that it is instantaneously
complete, so progress should be `100%`.

## The fix

Guard against the zero-duration case before dividing:

```ts
const duration = end - start;
const raw = duration === 0 ? 100 : ((now - start) / duration) * 100;
const progress = Math.min(100, Math.max(0, raw));
```

The existing clamp still applies so that normal streams with `now` outside the
`[start, end]` range continue to produce `0` or `100` rather than a value
beyond those bounds.

## What the test must verify

The test lives (or should live) in
`components/__tests__/StreamTimeline.test.tsx`.

The key case: render `<StreamTimeline>` with `startTime` and `endTime` set to
the same value. The progress bar (`role="progressbar"`) must carry
`aria-valuenow` of `100` and have an inline `width` of exactly `"100%"` —
neither `NaN%` nor any value that would leave the bar visually empty.

```tsx
it('renders a fully-complete progress bar when startTime equals endTime (#961)', () => {
  const instant = new Date('2024-01-01T00:00:00Z');

  render(
    <SettingsProvider>
      <StreamTimeline startTime={instant} endTime={instant} />
    </SettingsProvider>,
  );

  const bar = screen.getByRole('progressbar');

  // aria-valuenow must be a number, not NaN.
  const valuenow = Number(bar.getAttribute('aria-valuenow'));
  expect(Number.isNaN(valuenow)).toBe(false);

  // A zero-duration stream is 100 % complete by definition.
  expect(valuenow).toBe(100);

  // The inner width element must carry a valid CSS percentage.
  const fill = bar.querySelector('div') as HTMLElement;
  expect(fill.style.width).toBe('100%');
});
```

A second case confirms that nothing regressed for a normal stream whose
`endTime` is strictly after `startTime`:

```tsx
it('renders a partial progress bar for a normal in-progress stream', () => {
  const start = new Date(Date.now() - 30_000); // 30 s ago
  const end   = new Date(Date.now() + 30_000); // 30 s from now

  render(
    <SettingsProvider>
      <StreamTimeline startTime={start} endTime={end} />
    </SettingsProvider>,
  );

  const valuenow = Number(
    screen.getByRole('progressbar').getAttribute('aria-valuenow'),
  );
  expect(Number.isNaN(valuenow)).toBe(false);
  // Should be close to 50 % — allow ±10 for timing jitter.
  expect(valuenow).toBeGreaterThan(40);
  expect(valuenow).toBeLessThan(60);
});
```

## Mocking requirements

`StreamTimeline` imports `useSettings` and `useTranslations`. Wrap the
component in `<SettingsProvider>` as other component tests do. The
`formatDateShortWithTimezone` and `truncateAddress` utilities are pure
functions that do not need mocking.

## Running the tests

```bash
npx vitest run components/__tests__/StreamTimeline.test.tsx
```

## Related

- **`components/StreamTimeline.tsx`** — the progress formula on the line
  beginning `const progress = Math.min(...)`.
- **Issue #961** — the issue this test was added under.
