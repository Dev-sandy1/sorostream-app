# StreamQrModal — Federation Address Fallback in buildRecipientQrUri

**Issue:** [#959](https://github.com/SoroStream/sorostream-app/issues/959)

This document describes the federation-address fallback path in
`buildRecipientQrUri`, why it exists, what it produces, and what the test for
this issue must verify.

## The function

`buildRecipientQrUri` in `components/StreamQrModal.tsx` decides what to encode
in the QR code depending on the format of the recipient string:

```ts
export function buildRecipientQrUri(recipient: string, network: Network): string {
  if (!recipient.startsWith("G")) return recipient;
  const params = new URLSearchParams({
    destination: recipient,
    network_passphrase: NETWORK_CONFIG[network].networkPassphrase,
  });
  return `web+stellar:pay?${params.toString().replace(/\+/g, "%20")}`;
}
```

A string beginning with `"G"` is a Stellar account ID (StrKey-encoded public
key). It is wrapped in a SEP-7 `web+stellar:pay` URI so that wallets can parse
the network context from `network_passphrase`.

Anything that does not begin with `"G"` — most commonly a federation address
like `alice*domain.com` — is returned unchanged. The federation address is
itself a valid QR payload: a user scanning it can resolve it via the federation
server independently. Wrapping it in a SEP-7 URI would be incorrect because
SEP-7 `pay` expects a resolved Stellar account ID, not a federation address, as
the `destination` parameter.

## Why this matters

The existing test file `StreamQrModal.network.test.tsx` already includes a
one-line check:

```ts
it('leaves non-account recipients unchanged', () => {
  expect(buildRecipientQrUri('alice*example.com', 'mainnet')).toBe('alice*example.com');
});
```

That test verifies the return value but does not confirm that the downstream
`QRCode.toCanvas` call encodes the bare federation address, nor that the
rendered QR code is not empty or broken. This issue asks for a test that covers
the full rendering path for a federation-address recipient.

## What the test must verify

The test should live in a new file such as
`components/__tests__/StreamQrModal.federation.test.tsx` to keep it separate
from the network-switching tests.

The critical assertions:

1. `buildRecipientQrUri` returns the federation address string as-is (unit
   check on the function directly).
2. When `StreamQrModal` is rendered with a federation-address `recipient`,
   `QRCode.toCanvas` is called with that bare string — not a `web+stellar:` URI
   wrapping it.
3. The modal renders without error and shows the federation address in its
   address label.

```tsx
import { render, screen, fireEvent } from '@testing-library/react';
import QRCode from 'qrcode';
import StreamQrModal, { buildRecipientQrUri } from '../StreamQrModal';
import { NetworkProvider } from '@/src/lib/network';

vi.mock('qrcode', () => ({ default: { toCanvas: vi.fn() } }));

const FEDERATION = 'alice*sorostream.io';

describe('buildRecipientQrUri — federation address (#959)', () => {
  it('returns a federation address unchanged, with no SEP-7 wrapping', () => {
    expect(buildRecipientQrUri(FEDERATION, 'mainnet')).toBe(FEDERATION);
    expect(buildRecipientQrUri(FEDERATION, 'testnet')).toBe(FEDERATION);
  });

  it('encodes the bare federation string in the QR canvas', () => {
    vi.mocked(QRCode.toCanvas).mockClear();

    render(
      <NetworkProvider>
        <StreamQrModal open recipient={FEDERATION} onClose={vi.fn()} />
      </NetworkProvider>,
    );

    // QRCode.toCanvas must have been called exactly once.
    expect(QRCode.toCanvas).toHaveBeenCalledTimes(1);

    // The second argument is the content string; it must be the plain address.
    const encoded = vi.mocked(QRCode.toCanvas).mock.calls[0][1] as string;
    expect(encoded).toBe(FEDERATION);

    // It must not be a web+stellar URI.
    expect(encoded).not.toMatch(/^web\+stellar:/);
  });

  it('shows the federation address in the modal label', () => {
    render(
      <NetworkProvider>
        <StreamQrModal open recipient={FEDERATION} onClose={vi.fn()} />
      </NetworkProvider>,
    );

    // The address paragraph below the canvas displays the raw string.
    expect(screen.getByText(FEDERATION)).toBeInTheDocument();
  });
});
```

## Mocking requirements

Mock `qrcode` as `{ default: { toCanvas: vi.fn() } }` — the same approach used
in `StreamQrModal.network.test.tsx`. Wrap the component in `<NetworkProvider>`
so the `useNetwork()` hook resolves.

## Running the tests

```bash
npx vitest run components/__tests__/StreamQrModal.federation.test.tsx
```

## Related

- **`components/StreamQrModal.tsx`** — `buildRecipientQrUri`, the exported pure
  function that this test targets, and the `useEffect` that calls
  `QRCode.toCanvas`.
- **`components/__tests__/StreamQrModal.network.test.tsx`** — the existing
  suite covering SEP-7 network-passphrase encoding and network-switching
  behaviour.
- **Issue #959** — the issue this test was added under.
- **SEP-7** — the Stellar Ecosystem Proposal that defines the
  `web+stellar:pay` URI scheme. Only resolved Stellar account IDs are valid
  `destination` values.
