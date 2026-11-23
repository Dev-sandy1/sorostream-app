# StreamTagEditor — Backspace-with-Text Does Not Remove Tags

**Issue:** [#960](https://github.com/SoroStream/sorostream-app/issues/960)

This document describes the `Backspace` guard in `StreamTagEditor`'s
`handleKeyDown`, what it protects against, and what the test for this issue
must verify.

## The behavior

`StreamTagEditor` lets users delete the last tag chip by pressing `Backspace`
when the text input is empty. The guard in `handleKeyDown` is:

```ts
} else if (e.key === "Backspace" && !input && tags.length > 0) {
  removeTag(tags[tags.length - 1]);
}
```

The condition `!input` is the key check. When the user is in the middle of
typing a tag name, `input` is non-empty, so `!input` is `false` and the branch
is skipped. `Backspace` then falls through to the browser's default behavior:
it deletes the character before the cursor in the input field, as a user
expects.

Without this guard, any `Backspace` keypress — including one that is intended
to edit the tag text — would delete the most recently added chip, potentially
destroying data the user never intended to remove.

## What the test must verify

The test extends the existing suite in
`components/__tests__/StreamTagEditor.test.tsx`.

The core assertion: given an existing tag and text typed into the input, pressing
`Backspace` must not remove the tag. The tag chip must still be present in the
DOM after the keypress.

```tsx
it('Backspace with uncommitted text in the input does not remove an existing tag chip (#960)', () => {
  render(<StreamTagEditor streamId="stream-abc" />);

  const input = screen.getByRole('textbox', { name: /add a tag/i });

  // Add one tag.
  fireEvent.change(input, { target: { value: 'alpha' } });
  fireEvent.keyDown(input, { key: 'Enter' });

  // Confirm the tag chip is visible.
  expect(screen.getByText('alpha')).toBeInTheDocument();

  // Start typing a second tag.
  fireEvent.change(input, { target: { value: 'bet' } });

  // Press Backspace — this should edit the input text, not delete 'alpha'.
  fireEvent.keyDown(input, { key: 'Backspace' });

  // The 'alpha' chip is still there.
  expect(screen.getByText('alpha')).toBeInTheDocument();

  // No error alert should have been raised.
  expect(screen.queryByRole('alert')).not.toBeInTheDocument();
});
```

A complementary case confirms that `Backspace` on an *empty* input still
removes the last chip (the intended behaviour):

```tsx
it('Backspace with an empty input removes the last tag chip', () => {
  render(<StreamTagEditor streamId="stream-abc" />);

  const input = screen.getByRole('textbox', { name: /add a tag/i });

  // Add two tags.
  fireEvent.change(input, { target: { value: 'first' } });
  fireEvent.keyDown(input, { key: 'Enter' });
  fireEvent.change(input, { target: { value: 'second' } });
  fireEvent.keyDown(input, { key: 'Enter' });

  // Both chips are visible.
  expect(screen.getByText('first')).toBeInTheDocument();
  expect(screen.getByText('second')).toBeInTheDocument();

  // Input is now empty; Backspace should remove 'second'.
  fireEvent.keyDown(input, { key: 'Backspace' });

  expect(screen.queryByText('second')).not.toBeInTheDocument();
  expect(screen.getByText('first')).toBeInTheDocument();
});
```

## Setup note

The existing `afterEach` in `StreamTagEditor.test.tsx` calls
`localStorage.clear()` and `cleanup()`. Append the new tests inside or after
the existing `describe` block so they share that teardown; each test must start
with a clean `localStorage` state.

## Running the tests

```bash
npx vitest run components/__tests__/StreamTagEditor.test.tsx
```

## Related

- **`components/StreamTagEditor.tsx`** — the `handleKeyDown` function,
  specifically the `Backspace` branch on the line beginning
  `} else if (e.key === "Backspace" && !input && tags.length > 0)`.
- **Issue #960** — the issue this test was added under.
- The existing tests in `StreamTagEditor.test.tsx` cover tag-length and
  tag-count limits. This issue adds coverage for the keyboard interaction guard.
