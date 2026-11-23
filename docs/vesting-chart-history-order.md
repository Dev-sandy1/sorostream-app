# VestingChart — History Array Order and the `vestingChartKey` Comparator

**Issue:** [#964](https://github.com/SoroStream/sorostream-app/issues/964)

This document describes how `VestingChart`'s `React.memo` comparator handles
re-renders when the `history` prop is re-created with the same entries in a
different order, what the test added under this issue verifies, and the design
decision the test asks contributors to make explicit.

## How the comparator works

`VestingChart` is exported as `React.memo(VestingChart, comparator)`. The
comparator calls `vestingChartKey` for both old and new props and skips the
re-render if the keys are equal:

```ts
export default memo(VestingChart, (prev, next) => {
  if (prev.stream === next.stream && prev.history === next.history) return true;
  return (
    vestingChartKey(prev.stream, prev.history) ===
    vestingChartKey(next.stream, next.history)
  );
});
```

`vestingChartKey` builds the history portion of the key by mapping each entry
to a `type:timestamp:amount` string and joining them with `|`:

```ts
const historyKey = history
  .map((e) => `${e.type}:${e.timestamp}:${e.amount}`)
  .join("|");
```

The join preserves insertion order. Two arrays with the same entries in a
different order produce different strings — `"a|b"` versus `"b|a"` — and the
comparator therefore treats them as different, triggering a re-render.

## The design question

Whether that is intentional depends on what the chart is meant to do when
history changes:

- **Order matters (current behaviour):** A re-ordered array might represent
  history that was re-fetched and re-sorted, which could indicate stale data.
  Treating order changes as content changes is conservative and always correct,
  at the cost of extra renders when the caller re-creates the array in a
  different sequence without changing any entry.

- **Order does not matter:** If `history` is just a set of events and the chart
  does not care about their sequence (withdrawal reference lines are positioned
  by their `timestamp` field, not by array index), then sorting the entries
  before joining would make the key order-independent and prevent unnecessary
  renders caused by reshuffled fetches.

The current implementation takes the first position: order matters. The test
documents this as a deliberate choice so that future contributors understand
the behaviour before changing it.

## What the test checks

The test file is
`components/__tests__/VestingChart.test.tsx`. The cases added under this issue
sit alongside the existing memo comparator tests.

### Case 1 — reordered history triggers a re-render

```tsx
it('treats a reordered history array as a changed prop (order-sensitive key)', () => {
  const entryA: StreamHistoryEntry = {
    timestamp: '2026-01-05T00:00:00Z',
    type: 'withdrawal',
    amount: '50000000',
    txHash: 'tx-1',
  };
  const entryB: StreamHistoryEntry = {
    timestamp: '2026-01-10T00:00:00Z',
    type: 'withdrawal',
    amount: '25000000',
    txHash: 'tx-3',
  };

  const { rerender } = render(<VestingChart stream={STREAM} history={[entryA, entryB]} />);
  const rendersBefore = chartProps('LineChart').length;

  // Same entries, reversed order — should be treated as DIFFERENT by the
  // current key because the join order changes.
  rerender(<VestingChart stream={STREAM} history={[entryB, entryA]} />);

  expect(chartProps('LineChart').length).toBeGreaterThan(rendersBefore);
});
```

This asserts the current behaviour. If someone adds a sort to
`vestingChartKey` to make it order-independent, this test will fail and
force a conscious update to the assertion.

### Case 2 — identical reordering does not affect what the chart renders

The chart positions withdrawal reference lines by the `timestamp` field of
each entry, not by array index. This test confirms that even though a re-order
causes a re-render, the rendered output is the same:

```tsx
it('renders the same reference lines regardless of history array order', () => {
  const entryA: StreamHistoryEntry = {
    timestamp: '2026-01-05T00:00:00Z',
    type: 'withdrawal',
    amount: '50000000',
    txHash: 'tx-1',
  };
  const entryB: StreamHistoryEntry = {
    timestamp: '2026-01-10T00:00:00Z',
    type: 'withdrawal',
    amount: '25000000',
    txHash: 'tx-3',
  };

  const { rerender } = render(<VestingChart stream={STREAM} history={[entryA, entryB]} />);
  const linesBeforeXs = chartProps('ReferenceLine').map((l) => l.x).sort();

  for (const key of Object.keys(mockProps)) delete mockProps[key];
  rerender(<VestingChart stream={STREAM} history={[entryB, entryA]} />);
  const linesAfterXs = chartProps('ReferenceLine').map((l) => l.x).sort();

  expect(linesAfterXs).toEqual(linesBeforeXs);
});
```

## If you want to make the key order-independent

Replace the join in `vestingChartKey` with a sort before the join:

```ts
const historyKey = history
  .map((e) => `${e.type}:${e.timestamp}:${e.amount}`)
  .sort()          // ← add this
  .join("|");
```

Then update the Case 1 assertion to expect the render count to stay the same
after a reorder. Add a comment explaining the choice so the next reader
understands why the sort is there.

## Running the tests

```bash
npx vitest run components/__tests__/VestingChart.test.tsx
```

## Related

- **`components/VestingChart.tsx`** — the component under test.
- **Issue #964** — the issue this test was added under.
- The existing test suite already covers the content-equality path (same values,
  new object identities do not re-render) and the content-change path (a changed
  `flowRate` does re-render). This issue adds the reorder case that sits between
  those two: same entries, different order.
