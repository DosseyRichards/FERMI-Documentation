# JavaScript / browser

The dApp itself is plain JavaScript — no framework, no build step.
That makes it a good reference for browser-side integration.

## Read chain state

The explorer endpoints work directly from the browser:

```javascript
const r = await fetch('/api/explorer/stats');
const stats = await r.json();
console.log(stats.height, stats.mempool_size);
```

CORS is `*` so any origin can hit them.

## Authenticated requests

Anything under `/api/wallet/*`, `/api/playground/*`, `/api/send`, or
`/api/stream` requires a session cookie. The browser includes it
automatically when the request is same-origin:

```javascript
const r = await fetch('/api/wallet');         // cookie included
const w = await r.json();
```

For cross-origin requests, add `credentials: 'include'`:

```javascript
const r = await fetch('https://chat.waveledger.net/api/wallet', {
  credentials: 'include',
});
```

## SSE — live message stream

```javascript
const es = new EventSource('/api/stream');
es.onmessage = (e) => {
  const ev = JSON.parse(e.data);
  if (ev.type === 'message') {
    console.log(ev.msg.sender, ev.msg.text, ev.msg.status);
  }
};
es.onerror = () => {
  // EventSource auto-reconnects with exponential backoff
};
```

## Sending a tx (currently server-mediated)

To submit a transfer or contract call from the browser today, you call
the messenger endpoints — the server signs the tx with the user's
server-side wallet:

```javascript
const r = await fetch('/api/wallet/send', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ to: '39848b50...', amount: 1.0, memo: 'hi' }),
});
console.log(await r.json());
```

This is the testnet trust model: keys live on the server, the browser
controls them via the session.

## Browser-side signing (roadmap)

The honest answer for today: there is no browser ML-DSA-87 signer
available. The reference Python impl (`dilithium-py`) hasn't been
ported to WASM yet.

Once that ships, the dApp will become non-custodial:

```javascript
// Future API
import { Wallet, signTransaction } from '@waveledger/sdk';

const w = await Wallet.fromPrivateKey(privKeyHex);
const signed = await signTransaction(w, {
  recipient: '39848b50...',
  amount: 1.0,
  fee: 0.001,
});
await fetch('/api/tx/submit', {
  method: 'POST',
  body: JSON.stringify(signed),
});
```

Track the WASM port in the [TODO](https://github.com/DosseyRichards/FERMI-Documentation/blob/main/TODO.md)
in the docs repo.

## Compile-only path (no signing)

Even without browser-side signing, you can already do quite a lot from
the browser today:

- Compile Fourier source (`/api/playground/compile`)
- Decode any chain object (every explorer endpoint)
- Render real-time state (SSE)
- Trigger server-signed deploys + calls

The only operation that requires server-side custody is
*producing a transaction signature*.

## Decoding contract return values

The server returns `return_hex` and (when the result is a 32-byte word)
`return_uint`. For richer ABI decoding in browser, port the rules from
[`api/fourier_abi.py`](https://fourier.fermi.world/abi/encoding/):

```javascript
function decodeUint(hex)    { return BigInt('0x' + hex.slice(0, 64)); }
function decodeAddress(hex) { return '0x' + hex.slice(24, 64); }
function decodeBool(hex)    { return BigInt('0x' + hex.slice(0, 64)) === 1n; }
```

For tuples, split into 64-char chunks and decode each by type.
