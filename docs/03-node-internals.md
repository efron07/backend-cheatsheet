# 3. Node.js internals: event loop, streams, errors and threads

## 3.1 The Node.js event loop

**What it is:** Your JavaScript runs on **one main thread**. Slow I/O (database queries, HTTP calls, files) is handed to the operating system or libuv's thread pool, and when it finishes, its callback is queued. The **event loop** keeps picking queued callbacks and running them.

**The phases of each loop turn:**

1. **Timers** — `setTimeout` / `setInterval` callbacks that are due.
2. **Pending callbacks** — some system-level I/O callbacks.
3. **Poll** — wait for and run new I/O callbacks (most of your request handling happens here).
4. **Check** — `setImmediate` callbacks.
5. **Close callbacks** — e.g. socket `close` events.

Between every step, Node empties the **microtask queues**: first `process.nextTick` callbacks, then promise callbacks (`.then`, code after `await`).

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');
// sync → nextTick → promise → timeout/immediate (order of these two can vary outside I/O callbacks)
```

**The golden rule: never block the event loop.** While your code runs a long synchronous task, **no other request is served**.

```ts
// BAD — blocks every customer for seconds
app.get('/v1/reports/yearly', (req, res) => {
  const result = heavySyncCalculation(allSalesOfTheYear);   // 4 seconds of CPU
  res.json(result);
});
```

**Fixes:** do the work in the database (SQL aggregates), stream it, move it to a **worker thread**, or run it as a **background job** and notify when ready. Also avoid sync APIs in request paths (`fs.readFileSync`, `crypto.pbkdf2Sync`, huge `JSON.parse`).

## 3.2 Async patterns in Node

```ts
// 1. Error-first callbacks (old Node style)
fs.readFile('prices.csv', 'utf8', (err, data) => {
  if (err) return console.error(err);
  console.log(data);
});

// 2. Promises / async-await (modern — use these)
import { readFile } from 'node:fs/promises';
const data = await readFile('prices.csv', 'utf8');

// 3. Convert an old callback API into a promise
import { promisify } from 'node:util';
const sleep = promisify(setTimeout);

// 4. Run independent tasks in parallel
const [medicines, categories] = await Promise.all([repo.medicines.list(), repo.categories.list()]);

// 5. Limit concurrency: send 500 SMS, 10 at a time (p-limit package)
import pLimit from 'p-limit';
const limit = pLimit(10);
await Promise.allSettled(customers.map((c) => limit(() => sms.send(c.phone, message))));
```

## 3.3 Events: EventEmitter

**What it is:** The publish/subscribe pattern built into Node. One part of the code **emits** a named event; any number of **listeners** react. Streams, HTTP servers and sockets are all EventEmitters.

```ts
// src/events/bus.ts
import { EventEmitter } from 'node:events';

type Events = {
  'sale.completed': [sale: { id: number; total: number; items: { medicineId: number; qty: number }[] }];
  'stock.low': [medicine: { id: number; name: string; stock: number }];
};

export const bus = new EventEmitter<Events>();

// listeners
bus.on('sale.completed', (sale) => auditLog.write('sale.completed', sale));
bus.on('stock.low', (m) => notifyPharmacists(`${m.name} has ${m.stock} left`));

// emit after the sale is committed
bus.emit('sale.completed', sale);
```

In-process events are lost if the process crashes. For important work (sending receipts, syncing), use a durable **queue** (section 15).

## 3.4 Buffers

**What it is:** A `Buffer` holds raw binary data (bytes): uploaded files, images, encrypted data, network packets.

```ts
const buf = Buffer.from('PharmaCare', 'utf8');
buf.length;                         // 10 bytes
buf.toString('base64');             // "UGhhcm1hQ2FyZQ==" — M-Pesa passwords use base64
Buffer.from(auth, 'base64').toString('utf8');
const bytes = await readFile('logo.png');   // a Buffer
```

## 3.5 Streams

**What it is:** Process data **piece by piece** instead of loading everything into memory. A 2 GB sales export streams in small chunks with constant memory.

| Type | Example |
| --- | --- |
| Readable | File read stream, HTTP request body, database cursor |
| Writable | File write stream, HTTP response |
| Duplex | TCP socket |
| Transform | gzip compression, CSV conversion |

```ts
// Stream a year of sales as CSV directly to the client
import { pipeline } from 'node:stream/promises';
import { Readable, Transform } from 'node:stream';

app.get('/v1/reports/sales.csv', async (req, res) => {
  res.setHeader('Content-Type', 'text/csv');
  res.setHeader('Content-Disposition', 'attachment; filename="sales-2026.csv"');

  const rows = Readable.from(salesRepo.streamYear(2026));           // async generator of rows
  const toCsv = new Transform({
    objectMode: true,
    transform(row, _enc, cb) { cb(null, `${row.receiptNo},${row.total},${row.createdAt.toISOString()}\n`); },
  });

  res.write('receipt_no,total_tzs,created_at\n');
  await pipeline(rows, toCsv, res);       // handles backpressure and errors, closes everything
});
```

**Backpressure:** when the writer (a slow client) cannot keep up, `pipeline` pauses the reader automatically so memory does not explode.

## 3.6 Error handling in Node

**Kinds of errors:**

- **Operational errors** — expected failures: database down, invalid input, M-Pesa timeout. Handle them: retry, return 4xx/5xx, log.
- **Programmer errors** — bugs: reading a property of `undefined`. Fix the code; let the process restart.

```ts
// Custom error class with an HTTP status (used by the Express error handler, section 5)
export class AppError extends Error {
  constructor(public status: number, message: string, public code = 'APP_ERROR', public details?: unknown) {
    super(message);
    this.name = 'AppError';
  }
}

export class InsufficientStockError extends AppError {
  constructor(medicineName: string, available: number) {
    super(409, `Only ${available} of ${medicineName} in stock`, 'INSUFFICIENT_STOCK', { available });
  }
}
```

**Last-resort process handlers** — log and exit; a process manager (Docker, PM2, systemd) restarts it in a clean state:

```ts
process.on('unhandledRejection', (reason) => {
  logger.fatal({ reason }, 'Unhandled promise rejection');
  process.exit(1);
});

process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught exception');
  process.exit(1);           // state may be corrupted: do not keep serving
});
```

Always `await` promises (or `.catch` them). A forgotten promise that rejects crashes the process. The lint rule `@typescript-eslint/no-floating-promises` catches this.

## 3.7 Worker threads, child processes and cluster

| Tool | What it does | Use for |
| --- | --- | --- |
| `worker_threads` | Runs JS on another thread in the same process | CPU-heavy work: large report calculations, image resizing |
| `child_process` | Runs another program (`spawn`, `exec`, `fork`) | Call `pg_dump`, a Python script, a PDF tool |
| `cluster` | Runs several copies of your server sharing one port | Use all CPU cores (PM2 cluster mode or containers do this for you) |

```ts
// src/workers/report.worker.ts
import { parentPort, workerData } from 'node:worker_threads';
parentPort!.postMessage(computeYearlyReport(workerData.sales));

// caller
import { Worker } from 'node:worker_threads';
function runReport(sales: Sale[]) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL('./workers/report.worker.js', import.meta.url), { workerData: { sales } });
    worker.once('message', resolve);
    worker.once('error', reject);
  });
}
```

Pools like **Piscina** manage a fixed set of worker threads for you.

## 3.8 Debugging and profiling

```bash
node --inspect dist/server.js          # then open chrome://inspect, or use VS Code's debugger
node --inspect-brk dist/server.js      # pause on the first line
node --cpu-prof dist/server.js         # CPU profile file for Chrome DevTools
node --heap-prof dist/server.js        # memory allocation profile
```

- VS Code: create a launch configuration or use the **JavaScript Debug Terminal**, then set breakpoints.
- **Memory leaks:** watch `process.memoryUsage().heapUsed` over time; compare heap snapshots in Chrome DevTools; common causes are unbounded in-memory caches, listeners never removed, and global arrays.
- **Clinic.js** and `0x` produce flame graphs to find slow functions; `autocannon` load-tests an endpoint (`npx autocannon -c 50 -d 20 http://localhost:4000/v1/medicines`).

## 3.9 Node's built-in test runner

For small utilities you do not need a framework:

```ts
// src/utils/money.test.ts
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { lineTotal } from './money.ts';

test('line total multiplies price by quantity', () => {
  assert.equal(lineTotal(500, 3), 1500);
});
```

```bash
node --test
```

This sheet uses **Vitest** for the full suite (section 17) because of its richer mocking and watch mode.
