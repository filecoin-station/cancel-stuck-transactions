# cancel-stuck-transactions

Cancel stuck transactions on the Filecoin network.

## Usage

```js
import { StuckTransactionsCanceller } from 'cancel-stuck-transactions'
import timers from 'node:timers/promises'
import fs from 'node:fs/promises'

const stuckTransactionsCanceller = new StuckTransactionsCanceller({
  // Pass a storage adapter, so that pending cancellations are persisted across
  // process restarts
  store: {
    async set ({ hash, timestamp, from, maxPriorityFeePerGas, gasLimit, nonce }) {
      await fs.writeFile(
        `transactions/${hash}`,
        JSON.stringify({
          hash,
          timestamp,
          from,
          maxPriorityFeePerGas,
          gasLimit,
          nonce
        })
      )
    },
    async list () {
      const cids = await fs.readdir('transactions')
      return Promise.all(cids.map(async cid => {
        return JSON.parse(await fs.readFile(`transactions/${cid}`))
      }))
    },
    async remove (hash) {
      await fs.unlink(`transactions/${hash}`)
    },
  },
  log (str) {
    console.log(str)
  },
  // Pass to an ethers signer for sending replacement transactions
  sendTransaction (tx) {
    return signer.sendTransaction(tx)
  }
})

// Start the cancel transactions loop
;(async () => {
  while (true) {
    await stuckTransactionsCanceller.cancelOlderThan(TEN_MINUTES)
    await timers.setTimeout(ONE_MINUTE)
  }
})()

// Create a transaction somehow
const tx = await ethers.createTransaction(/* ... */)

// After you create a transactions, add it as pending
stuckTransactionsCanceller.addPending(tx)

// Start waiting for confirmations
await tx.wait()

// Once confirmed, remove it
await stuckTransactionsCanceller.removeConfirmed(tx)
```

## Installation

```console
npm install cancel-stuck-transactions
```

## API

### `StuckTransactionsCanceller({ store, log, sendTransaction })`

```js
import { StuckTransactionsCanceller } from 'cancel-stuck-transactions'
```

Options:

- `store`:
  - `store.set`: `({ hash: string, timestamp: string, from: string, maxPriorityFeePerGas: bigint, gasLimit: bigint, nonce: number }) -> Promise`
  - `store.list`: `() -> Promise<{ hash: string, timestamp: string, from: string, maxPriorityFeePerGas: bigint, nonce: number }[]>`
  - `store.remove`: `(hash: string) -> Promise`
- `log`: `(string) -> null`
- `sendTransaction`: `(Transaction) -> Promise<Transaction>`

### `#addPending(tx) -> Promise`

Add `tx` as pending.

`tx` should be a
[Transaction](https://docs.ethers.org/v6/api/transaction/#Transaction) object
from ethers.js.

### `#removeConfirmed(tx) -> Promise`

Remove `tx` because it is successful.

### `#cancelOlderThan(ms, { concurrency = 50 }) -> Promise<{status,value,reason}[]?>`

Cancel transactions older than `ms`. Returns the return value of
[`Promise.allSettled()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)
for all performed cancellations.

Throws:
- _See `cancelTx()`_
- _See `getRecentSendMessage()`_
- _potentially more_

### `cancelTx({ tx, recentGasLimit, recentGasFeeCap, log, sendTransaction }) -> Promise<tx>`

```js
import { cancelTx } from 'cancel-stuck-transactions'
```

Helper method that manually cancels transaction `tx`.

Options:

- `tx`: `ethers.Transaction`
- `recentGasLimit`: `number`
- `recentGasFeeCap`: `bigint`
- `log`: `(str: string) -> null`
- `sendTransaction`: `(Transaction) -> Promise<Transaction>`

Throws:
- `err.code === 'NONCE_EXPIRED'`: The transaction can't be replaced because
it has already been confirmed
- _potentially more_

### `getRecentSendMessage() -> Promise<SendMessage>`

```js
import { getRecentSendMessage } from 'cancel-stuck-transactions'
```

Helper method that fetches a recent `SendMessage`.

`SendMessage` has keys (and more):
- `cid`: `string`
- `timestamp`: `number`
- `receipt`: `object`
  -  `gasUsed`: `number`
- `gasFeeCap`: `string`
- `gasLimit`: `number`

Throws:
- `err.code === 'FILFOX_REQUEST_FAILED'`: This method relies on `filfox.info`.
Requests may fail at any point
- _potentially more_
