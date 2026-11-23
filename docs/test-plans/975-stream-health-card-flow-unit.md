# #975 — StreamHealthCard: flow-rate unit survives a stream prop update

**Component:** [`components/StreamHealthCard.tsx`](../../components/StreamHealthCard.tsx)
**Existing tests:** `components/__tests__/StreamHealthCard.test.tsx`

## Behaviour under test

```ts
const [flowUnit, setFlowUnit] = useState<FlowRateUnit>("day");
```

`flowUnit` is local state that only the `/d`, `/w`, `/m` buttons change
(aria-label `Show flow rate per day|week|month`). `flowRateDisplay` is
memoised on `[flowPerSec, flowUnit, stream.token]`. When the `stream` prop changes,
the display is recalculated, but the selected unit should not reset.

## Test plan

Use `render` and then `rerender` on the **same** instance, with no `key` change.

| # | Steps | Expectation |
|---|-------|-------------|
| 1 | Render active stream (1 XLM/s); click `/w` | `health-flow-rate` shows `604,800.00 XLM/week`; `/w` has the active class |
| 2 | `rerender` with the same stream id but a larger `deposit` / later `endTime` (top-up) | Still `/week`; `/w` still active |
| 3 | `rerender` with the status changed (e.g. `active` → `paused` or `cancelled`) | Unit still `week` |
| 4 | `rerender` with a different `flowRate` (2 XLM/s) | Shows `1,209,600.00 XLM/week`: the value is recalculated and the unit is kept |
| 5 | Repeat 1–2 starting from `/m` | Shows `/month` (30-day multiplier), kept after rerender |
| 6 | Control: unmount and mount a fresh instance | Starts at `/day` again |

Use fake timers. The component ticks `now` every second, and the test shouldn't
depend on real time.

## Notes

- The unit is only kept for the **same mounted instance**. A parent that renders
  `<StreamHealthCard key={stream.id} />`, or remounts it on refetch, will reset it
  to `day`. Case 6 records that this is expected.
- Match the number with a regex or with `toLocaleString`, so the test doesn't
  depend on the locale's thousands separator.
