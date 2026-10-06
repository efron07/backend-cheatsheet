# 15. Background jobs, real-time and integrations

## 15.1 Why background jobs

**What it is:** Some work should not happen inside the HTTP request: sending SMS receipts, emailing reports, generating PDFs, calling slow external APIs, retrying failed payments. Put it on a **queue**; a separate **worker** process does it, with retries if it fails.

**Why it matters:** the cashier gets a response in 100 ms instead of waiting 5 seconds for an SMS gateway; a crashed SMS provider does not fail the sale; failed jobs retry automatically.

## 15.2 BullMQ (Redis-based queues)

```bash
npm install bullmq
```

```ts
// src/lib/queues.ts
import { Queue } from 'bullmq';
import { redis } from './redis.ts';

export const notificationsQueue = new Queue('notifications', {
  connection: redis,
  defaultJobOptions: {
    attempts: 5,                                         // retry up to 5 times
    backoff: { type: 'exponential', delay: 10_000 },     // 10s, 20s, 40s...
    removeOnComplete: 1000,                              // keep the last 1000 for inspection
    removeOnFail: 5000,
  },
});

export const reportsQueue = new Queue('reports', { connection: redis });
```

**Producer** — add a job (from a service or event listener):

```ts
bus.on('sale.completed', async (sale) => {
  if (sale.customerPhone) {
    await notificationsQueue.add('receipt-sms', { saleId: sale.id, phone: sale.customerPhone }, { jobId: `receipt-${sale.id}` });   // jobId = no duplicates
  }
});
```

**Worker** — a separate process (`node dist/worker.js`):

```ts
// src/jobs/worker.ts
import { Worker } from 'bullmq';
import { redis } from '../lib/redis.ts';
import { sms } from '../lib/sms.ts';

const worker = new Worker('notifications', async (job) => {
  switch (job.name) {
    case 'receipt-sms': {
      const sale = await saleRepository.findWithItems(job.data.saleId);
      await sms.send(job.data.phone, `PharmaCare receipt ${sale.receiptNo}: ${sale.total.toLocaleString()} TZS. Asante!`);
      return { sent: true };
    }
    case 'low-stock-alert':
      return notifyPharmacists(job.data);
    default:
      throw new Error(`Unknown job ${job.name}`);
  }
}, { connection: redis, concurrency: 10 });

worker.on('failed', (job, err) => logger.error({ jobId: job?.id, err }, 'Job failed'));
```

**Scheduled (repeatable) jobs:**

```ts
await reportsQueue.upsertJobScheduler('daily-sales-report', { pattern: '0 21 * * *', tz: 'Africa/Dar_es_Salaam' }, { name: 'daily-report' });
await reportsQueue.upsertJobScheduler('expiry-check', { pattern: '0 7 * * *', tz: 'Africa/Dar_es_Salaam' }, { name: 'expiring-batches' });
```

**Job design rules:** jobs must be **idempotent** (safe to run twice); pass IDs, not whole objects (data may change); keep each job small; monitor the queue with **Bull Board** (`@bull-board/express`) behind admin auth.

**Alternatives:** `node-cron` (simple in-process schedules — runs on every instance unless you guard it), **pg-boss** (queues inside PostgreSQL, no Redis), cloud queues (SQS, Google Pub/Sub), **Inngest / Trigger.dev** (hosted durable workflows), **RabbitMQ / Kafka** (message brokers, section 19).

## 15.3 Email and SMS

- **Email:** Nodemailer (any SMTP), or APIs like Resend, Postmark, Amazon SES. Build HTML emails with React Email or MJML. Always send from a queue.
- **SMS (Tanzania):** providers such as Africa's Talking, Beem, or mobile-network bulk-SMS APIs. Wrap the provider in your own `sms.send(phone, text)` interface so you can switch providers.
- Normalise phone numbers to international format (`+2557...`) before sending.

## 15.4 Mobile money integration (M-Pesa flow)

The exact endpoints differ by provider and country (Vodacom M-Pesa Tanzania, Safaricom Daraja, Airtel Money, aggregators like Selcom, AzamPay or ClickPesa), but the flow is the same:

1. **Get an access token** from the provider (cache it in Redis until shortly before it expires).
2. **Initiate a payment** ("push" to the customer's phone) with amount, phone, your reference (the receipt number) and your **callback URL**. Save a `payments` row with status `pending` and the provider's request ID.
3. Respond to the POS **immediately** (`202 Accepted`); the POS shows "Waiting for customer to confirm…".
4. The provider calls your **callback (webhook)** with the result. Verify it, update `payments` and `sales` in one transaction, idempotently.
5. Notify the POS in real time (SSE/WebSocket, section 15.5).
6. A scheduled job **queries the status** of payments still `pending` after a few minutes (callbacks can be lost).

```ts
// src/modules/payments/mpesa.callback.ts — webhook handler
paymentRouter.post('/mpesa/callback', express.json(), async (req, res) => {
  res.json({ ResultCode: 0, ResultDesc: 'Accepted' });     // acknowledge fast

  const parsed = MpesaCallbackSchema.safeParse(req.body);
  if (!parsed.success) return req.log.warn({ body: req.body }, 'Bad M-Pesa callback');

  await db.transaction(async (tx) => {
    const [payment] = await tx.select().from(payments).where(eq(payments.providerRequestId, parsed.data.requestId)).for('update');
    if (!payment || payment.status !== 'pending') return;  // already processed → idempotent

    const paid = parsed.data.resultCode === 0 && parsed.data.amount === payment.amount;   // verify the amount too
    await tx.update(payments).set({ status: paid ? 'paid' : 'failed', providerReceipt: parsed.data.receipt, raw: req.body }).where(eq(payments.id, payment.id));
    await tx.update(sales).set({ status: paid ? 'paid' : 'pending' }).where(eq(sales.id, payment.saleId));
  });

  realtime.publish(`sale:${parsed.data.reference}`, { type: 'payment.updated' });
});
```

Store the raw callback payload (or log it to MongoDB) for reconciliation and disputes. Reconcile daily against the provider's statement.

## 15.5 Real-time updates: SSE and WebSockets

**Server-Sent Events (SSE)** — simplest one-way push over HTTP. Perfect for "payment confirmed" and live stock levels.

```ts
// src/modules/realtime/sse.ts
const clients = new Map<string, Set<Response>>();

realtimeRouter.get('/sales/:receiptNo/events', authenticate, (req, res) => {
  res.set({ 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', Connection: 'keep-alive' });
  res.flushHeaders();
  const channel = `sale:${req.params.receiptNo}`;
  if (!clients.has(channel)) clients.set(channel, new Set());
  clients.get(channel)!.add(res);

  const ping = setInterval(() => res.write(': ping\n\n'), 25_000);   // keep proxies from closing it
  req.on('close', () => { clearInterval(ping); clients.get(channel)?.delete(res); });
});

export const realtime = {
  publish(channel: string, data: unknown) {
    for (const res of clients.get(channel) ?? []) res.write(`data: ${JSON.stringify(data)}\n\n`);
  },
};
```

```ts
// Browser
const source = new EventSource(`${API}/v1/realtime/sales/${receiptNo}/events`, { withCredentials: true });
source.onmessage = (e) => { if (JSON.parse(e.data).type === 'payment.updated') refetchSale(); };
```

With several API instances, a client may be connected to a different instance than the one that received the callback: broadcast through **Redis pub/sub** so every instance forwards the message to its own clients.

**WebSockets** (two-way) — use **Socket.IO** (rooms, reconnection, Redis adapter for multiple instances) or the lightweight `ws` library. Example uses: a live POS dashboard for managers, staff chat. Hosted alternatives: Ably, Pusher, Supabase Realtime.

```ts
import { Server } from 'socket.io';
const io = new Server(httpServer, { cors: { origin: env.CORS_ORIGINS, credentials: true } });
io.use(async (socket, next) => { try { socket.data.user = await verifyAccessToken(socket.handshake.auth.token); next(); } catch { next(new Error('unauthorized')); } });
io.on('connection', (socket) => socket.join(`branch:${socket.data.user.branchId}`));
io.to('branch:1').emit('stock:low', { medicineId: 7, stock: 8 });
```

**Long polling** (client asks every few seconds) is the fallback when push is not possible.

## 15.6 File storage

Use **S3-compatible object storage** (Cloudflare R2, AWS S3, Backblaze B2, MinIO self-hosted). Best practice is **pre-signed URLs**: the API authorises the upload and returns a short-lived URL; the browser uploads directly to storage, so large files never pass through your Node process.

```ts
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({ region: 'auto', endpoint: env.S3_ENDPOINT, credentials: { accessKeyId: env.S3_KEY, secretAccessKey: env.S3_SECRET } });

// 1. Client asks to upload a prescription scan
saleRouter.post('/:id/prescription/upload-url', authenticate, async (req, res) => {
  const key = `prescriptions/${req.params.id}/${crypto.randomUUID()}.pdf`;
  const url = await getSignedUrl(s3, new PutObjectCommand({ Bucket: 'pharmacare-private', Key: key, ContentType: 'application/pdf' }), { expiresIn: 300 });
  res.json({ data: { url, key } });          // 2. Browser PUTs the file to `url`, then 3. tells the API the key
});

// Download: a short-lived signed GET URL; the bucket itself stays private
const downloadUrl = await getSignedUrl(s3, new GetObjectCommand({ Bucket: 'pharmacare-private', Key: key }), { expiresIn: 120 });
```

## 15.7 Calling external APIs reliably

- Always set **timeouts**: `fetch(url, { signal: AbortSignal.timeout(10_000) })`.
- **Retry** only safe/idempotent requests, with exponential backoff and jitter.
- Use a **circuit breaker** (e.g. `opossum`) for providers that go down.
- Log request IDs from providers; store raw responses for money-related calls.
- Put each provider behind your own interface (`PaymentGateway`, `SmsProvider`) so you can swap or mock them in tests.
