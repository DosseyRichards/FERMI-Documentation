# Python client

The reference path: import directly from the WaveLedger codebase.

## Install

```bash
git clone https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software.git
cd Fermi-Mining-ASIC-Software
pip install -r deploy/requirements-seed.txt
```

## Create a wallet

```python
from crypto.kyber_crypto import WaveLedgerCrypto

crypto = WaveLedgerCrypto()
wallet = crypto.create_wallet()

print(wallet['address'])     # '34378b1ba5be9d0999acd60be3a8a1f1'
print(wallet['public_key'])  # ML-DSA-87 public key (object or hex)
# wallet['private_key']      # Do not log
```

## Sign + submit a transfer

```python
import hashlib, json, time, requests

# Build the canonical envelope
tx_data = {
    'sender': wallet['address'],
    'recipient': '39848b500426d8f1...',
    'amount': 1.0,
    'timestamp': time.time(),
    'fee': 0.001,
}
tx_id = hashlib.sha3_512(
    json.dumps(tx_data, sort_keys=True).encode()
).hexdigest()
signature = crypto.sign_transaction(
    tx_data, wallet['private_key'],
    signing_key=wallet.get('signing_key'),
)

# Submit via the dashboard's POST /api/tx/submit
# (requires Bearer API key — see API/auth)
tx_envelope = {
    **tx_data,
    'transaction_id': tx_id,
    'signature': signature,
    'data': {'memo': 'hello chain'},
    'nonce': -1,
}
r = requests.post(
    'http://127.0.0.1:8080/api/tx/submit',
    headers={'Authorization': f'Bearer {api_key}'},
    json=tx_envelope,
)
print(r.json())   # {"status":"submitted","tx_id":"..."}
```

## Compile + deploy a Fourier contract

```python
from fourier import compile_source
from api.fourier_abi import extract_abi
from core.contract_engine import build_deploy_tx_data

source = '''
contract Counter {
    storage value: uint @ 0;
    fn init() { value = 0; }
    pub fn get() -> uint { return value; }
    pub fn inc() { value = value + 1; }
}
'''
bytecode = compile_source(source)
abi = extract_abi(source)

tx_data_dict = build_deploy_tx_data(bytecode)
# Then sign + submit as above, with:
#   recipient = 'contract'
#   amount    = 0.0
#   fee       = 0.001
#   data      = tx_data_dict
```

The contract address derives from `(sender, sender_nonce)` and lands
in the receipt once mined. Poll the receipt:

```python
import time
for _ in range(30):
    r = requests.get(f'http://127.0.0.1:8080/api/tx/{tx_id}').json()
    if r.get('receipt'):
        print(r['receipt']['contract'])
        break
    time.sleep(1)
```

## Call a method

```python
from api.fourier_abi import encode_call, decode_return
from core.contract_engine import build_call_tx_data

# inc() — selector 2, no args
calldata = encode_call(selector=2, args=[], params=[])
tx_data_dict = build_call_tx_data(
    bytes.fromhex(contract_addr_hex),
    calldata,
)
# Build, sign, submit, poll as before.

# get() — selector 1, returns uint
calldata = encode_call(selector=1, args=[], params=[])
# After submit + poll:
return_value = decode_return(receipt['return_data'], 'uint')
print(return_value)   # 1
```

## Read chain state (no auth)

The explorer endpoints are public:

```python
import requests

stats = requests.get('https://chat.waveledger.net/api/explorer/stats').json()
print(f"height={stats['height']} mempool={stats['mempool_size']}")

addr = '34378b1ba5be9d0999acd60be3a8a1f1'
me = requests.get(f'https://chat.waveledger.net/api/explorer/address/{addr}').json()
print(f"balance={me['balance']} WAVE  tx_count={me['tx_count']}")
```

## Subscribe to chat messages (SSE)

```python
import requests, json

with requests.get(
    'https://chat.waveledger.net/api/stream',
    cookies={'session': '<session-cookie>'},
    stream=True,
) as r:
    for line in r.iter_lines():
        if line.startswith(b'data: '):
            ev = json.loads(line[6:])
            print(ev)
```

## When to use this vs the dApp

| Use the dApp UI | Use the Python client |
|---|---|
| Demoing | Bulk scripting (script 100 contracts) |
| User-facing flows | Integration tests |
| Quick exploration | Indexer / explorer / bot |
| Anyone non-technical | Anything programmatic |

Both hit the same node and produce identical on-chain effects. There's
no "test mode" — every send is real.

## A note on async

The chain's internals are async (`asyncio`). Most of the client-facing
operations above run synchronously because they go through `requests`.
If you want to drive the chain in-process (e.g. an indexer that calls
`blockchain.get_balance()` directly), you'll be in an `asyncio` context
and should use `aiohttp` instead.
