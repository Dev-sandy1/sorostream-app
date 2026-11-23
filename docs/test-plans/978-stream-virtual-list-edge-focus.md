# #978 — StreamVirtualList: arrow-key focus stays put at list edges

**Component:** [`components/StreamVirtualList.tsx`](../../components/StreamVirtualList.tsx)
**Existing tests:** `components/__tests__/StreamVirtualList.test.tsx`

## Behaviour under test

The cards are laid out in a 2-column grid. `handleItemKeyDown` maps keys to a target index:

| Key | Target | Bound guard |
|-----|--------|-------------|
| `ArrowDown` | `i + 2` | ternary → stays at `i` if `i + 2 >= length` |
| `ArrowUp` | `i - 2` | ternary → stays at `i` if `i - 2 < 0` |
| `ArrowRight` | `i + 1` | `Math.min(i + 1, length - 1)` |
| `ArrowLeft` | `i - 1` | `Math.max(i - 1, 0)` |

`focusStreamAt` also returns early for an out-of-range index and uses `el?.focus()`, so an
unregistered ref does nothing.

> The issue says all four keys use `Math.min`/`Math.max`. In the code, Up and Down
> use a ternary instead. Both keep focus on the current card.

## Test plan

Render 4–6 streams so every card is inside the virtualised window
(`startRow`/`endRow` with overscan). If jsdom reports `clientHeight = 0`, stub it.

| # | Focus on | Key | Expectation |
|---|----------|-----|-------------|
| 1 | first card | `ArrowUp` | `document.activeElement` is still the first card's link |
| 2 | first card | `ArrowLeft` | still first card |
| 3 | last card | `ArrowDown` | still last card |
| 4 | last card | `ArrowRight` | still last card |
| 5 | last card in a list of odd length (e.g. 5), at index 3 | `ArrowDown` | stays on index 3 (`3 + 2 = 5` is out of range) |
| 6 | single-item list | each of the four arrows | focus never leaves the only card |

For every case:
- `expect(() => fireEvent.keyDown(...)).not.toThrow()`
- `document.activeElement` is not `document.body` and not `null`
- `defaultPrevented` is `true` (the page doesn't scroll)

Find the card links with `screen.getAllByRole("link")` or with `href="/stream/<id>"`.
