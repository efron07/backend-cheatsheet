# 18. Observability and reliability

**Observability** = being able to answer "what is happening inside the system, and why?" from the outside. It has three pillars: **logs** (what happened), **metrics** (how much / how fast), **traces** (the path of one request through every service).

## 18.1 Structured logging

**What it is:** Logs as JSON objects with consistent fields, not free text. Log tools can then filter ("all errors for request X", "all sales by cashier 3 that failed").

```ts
import { logger } from './lib/logger.ts';

logger.info({ saleId: 1052, total: 1500, cashierId: 3 }, 'Sale completed');
logger.warn({ medicineId: 7, stock: 8 }, 'Stock below reorder level');
logger.error({ err, saleId: 1052, provider: 'mpesa' }, 'Payment initiation failed');

// Inside a request: pino-http gives req.log with the request ID attached automatically
req.log.info({ saleId }, 'Voiding sale');
```

```json
{"level":"info","time":"2026-10-06T09:15:02.113Z","reqId":"2b8f1e0a","saleId":1052,"total":1500,"cashierId":3,"msg":"Sale completed"}
```

**Log levels:** `fatal` (process dying) → `error` (failed operation) → `warn` (unexpected but handled) → `info` (business events) → `debug` (developer detail, off in production).

**Rules:** include a **request ID** in every line (correlation); never log secrets or full personal data (redact); log business events (sale, void, price change) at `info`; ship logs to a central store — Grafana Loki, Better Stack, Datadog, Axiom, or ELK/OpenSearch.

## 18.2 Metrics

**What it is:** Numbers measured over time. The **RED** method for every API: **R**ate (requests/sec), **E**rrors (error rate), **D**uration (latency, especially p95/p99). Plus resource metrics (CPU, memory, event-loop lag, DB pool usage) and **business metrics** (sales per hour, payment failures).

```bash
npm install prom-client
```

```ts
import client from 'prom-client';

client.collectDefaultMetrics();                         // CPU, memory, event loop lag, GC

export const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.05, 0.1, 0.25, 0.5, 1, 2, 5],
});
export const salesTotal = new client.Counter({ name: 'pharmacare_sales_total', help: 'Completed sales', labelNames: ['payment_method'] });

app.use((req, res, next) => {
  const end = httpDuration.startTimer();
  res.on('finish', () => end({ method: req.method, route: req.route?.path ?? 'unknown', status: res.statusCode }));
  next();
});

app.get('/metrics', async (_req, res) => {                 // protect this (internal network or auth)
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

Prometheus scrapes `/metrics`; **Grafana** draws dashboards and fires alerts.

## 18.3 Distributed tracing with OpenTelemetry

**What it is:** A **trace** follows one request across Next.js → Express → PostgreSQL → Redis → M-Pesa, showing how long each step took. **OpenTelemetry** (OTel) is the vendor-neutral standard; it auto-instruments Express, `pg`, `ioredis`, MongoDB and `fetch`.

```ts
// src/instrumentation.ts — load before everything else: node --import ./dist/instrumentation.js dist/server.js
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

new NodeSDK({
  serviceName: 'pharmacare-api',
  traceExporter: new OTLPTraceExporter({ url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT }),
  instrumentations: [getNodeAutoInstrumentations()],
}).start();
```

Send traces to Grafana Tempo, Jaeger, Honeycomb, Datadog or SigNoz. Trace context propagates through HTTP headers (`traceparent`), so the Next.js frontend's traces connect to the API's.

## 18.4 Error tracking

**Sentry** (or Rollbar, Bugsnag) groups exceptions, shows stack traces with source maps, the request, the user and how often it happens, and alerts you.

```ts
// at the very top of instrumentation
import * as Sentry from '@sentry/node';
Sentry.init({ dsn: env.SENTRY_DSN, environment: env.NODE_ENV, tracesSampleRate: 0.1 });

// after routes, before your own error handler
Sentry.setupExpressErrorHandler(app);
```

## 18.5 Health checks

```ts
// Liveness: "is the process alive?" — cheap, no dependencies (restart the container if this fails)
app.get('/health/live', (_req, res) => res.json({ status: 'ok' }));

// Readiness: "can I serve traffic?" — checks dependencies (remove from the load balancer if this fails)
app.get('/health/ready', async (_req, res) => {
  const checks = await Promise.allSettled([
    pool.query('SELECT 1'),
    redis.ping(),
    mongoose.connection.db?.admin().ping(),
  ]);
  const [postgres, redisOk, mongo] = checks.map((c) => c.status === 'fulfilled');
  const ready = postgres && redisOk;                  // Mongo (audit) is non-critical: degraded, not down
  res.status(ready ? 200 : 503).json({ status: ready ? (mongo ? 'ok' : 'degraded') : 'unavailable', postgres, redis: redisOk, mongo });
});
```

Point an uptime monitor (Better Stack, UptimeRobot, Checkly) at the readiness endpoint and alert by SMS/WhatsApp.

## 18.6 Graceful shutdown

**What it is:** When the platform stops your container (deploys, scaling), it sends `SIGTERM`. Stop accepting new requests, finish in-flight ones, close connections, then exit. Without this, deploys cut off sales mid-transaction.

```ts
// src/server.ts
const server = createApp().listen(env.PORT, () => logger.info(`API on :${env.PORT}`));
await connectMongo();

let shuttingDown = false;
async function shutdown(signal: string) {
  if (shuttingDown) return;
  shuttingDown = true;
  logger.info({ signal }, 'Shutting down gracefully');

  const force = setTimeout(() => { logger.error('Forced exit'); process.exit(1); }, 25_000);   // under the platform's grace period
  force.unref();

  server.close(async () => {            // stop accepting; wait for in-flight requests
    await Promise.allSettled([pool.end(), redis.quit(), disconnectMongo(), worker?.close()]);
    logger.info('Shutdown complete');
    process.exit(0);
  });
  server.closeIdleConnections();
}

process.on('SIGTERM', () => void shutdown('SIGTERM'));
process.on('SIGINT', () => void shutdown('SIGINT'));
```

## 18.7 Reliability practices

- **Timeouts everywhere:** HTTP server (`server.requestTimeout`), DB (`statement_timeout`), outbound calls (`AbortSignal.timeout`).
- **Retries with backoff + jitter** for transient failures, only on idempotent operations.
- **Idempotency** for payments and jobs (sections 7.3, 15.2).
- **Circuit breakers** and **graceful degradation** (section 14.6).
- **SLOs:** define targets, e.g. "99.5% of `POST /v1/sales` succeed in under 800 ms"; alert when the error budget burns too fast.
- **Runbooks:** short documents for "M-Pesa callbacks stopped", "database is slow", "disk full".
- **Post-mortems:** after incidents, write what happened and what will prevent it — without blame.
- **The Twelve-Factor App** principles: config in env, stateless processes, logs as streams, dev/prod parity, disposability (fast start, graceful shutdown).
