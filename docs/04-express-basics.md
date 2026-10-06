# 4. Express basics: setup, routing and middleware

## 4.1 What Express is

**What it is:** Express is a minimal web framework for Node.js. It adds **routing** (which code runs for which URL and method), **middleware** (a chain of functions every request passes through), and helpers for reading requests and sending responses. It is the most widely used Node framework and the base of many others.

**Express 5** (the current major version) handles errors thrown in `async` handlers automatically, updates route path syntax and drops old APIs.

**Alternatives:** **Fastify** (faster, schema-based), **Hono** (tiny, runs everywhere including edge), **NestJS** (structured, Angular-style, uses Express or Fastify underneath), Koa.

## 4.2 Project setup with TypeScript

```bash
mkdir pharmacare-api && cd pharmacare-api
npm init -y
npm pkg set type=module
npm install express cors helmet compression cookie-parser zod pino pino-http
npm install -D typescript tsx @types/node @types/express @types/cors @types/compression @types/cookie-parser vitest supertest @types/supertest
npx tsc --init
```

```json5
// tsconfig.json (key options)
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",              // Node's real ESM rules
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "sourceMap": true,
    "verbatimModuleSyntax": true,
    "rewriteRelativeImportExtensions": true   // write `./app.ts` imports, emit `./app.js`
  },
  "include": ["src"]
}
```

## 4.3 Project structure

```
pharmacare-api/
├── src/
│   ├── server.ts              # starts the HTTP server, graceful shutdown
│   ├── app.ts                 # creates the Express app (imported by tests too)
│   ├── config/env.ts          # validated environment variables
│   ├── db/
│   │   ├── postgres.ts        # Drizzle + Neon client
│   │   ├── schema.ts          # tables
│   │   └── mongo.ts           # Mongoose connection
│   ├── modules/               # one folder per feature
│   │   ├── medicines/
│   │   │   ├── medicine.routes.ts
│   │   │   ├── medicine.controller.ts
│   │   │   ├── medicine.service.ts
│   │   │   ├── medicine.repository.ts
│   │   │   ├── medicine.schemas.ts      # Zod
│   │   │   └── medicine.test.ts
│   │   ├── sales/  batches/  auth/  payments/  reports/  audit/
│   ├── middleware/            # auth, validate, errorHandler, notFound, rateLimit
│   ├── lib/                   # logger, errors, redis, queue, mpesa client
│   └── jobs/                  # BullMQ workers
├── drizzle/                   # generated SQL migrations
├── tests/                     # integration tests
├── .env  .env.example  Dockerfile  package.json  tsconfig.json
```

Organise **by feature** (medicines, sales) rather than by type (all controllers together): everything about one feature lives in one folder.

## 4.4 The app and the server

```ts
// src/app.ts
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import compression from 'compression';
import cookieParser from 'cookie-parser';
import { pinoHttp } from 'pino-http';
import { env } from './config/env.ts';
import { logger } from './lib/logger.ts';
import { apiRouter } from './routes.ts';
import { notFound, errorHandler } from './middleware/errors.ts';

export function createApp() {
  const app = express();

  app.disable('x-powered-by');
  app.set('trust proxy', 1);                         // behind Nginx/a load balancer: real client IP

  app.use(pinoHttp({ logger }));                     // request logging
  app.use(helmet());                                 // security headers
  app.use(cors({ origin: env.CORS_ORIGINS, credentials: true }));
  app.use(compression());                            // gzip responses
  app.use(express.json({ limit: '1mb' }));           // parse JSON bodies
  app.use(express.urlencoded({ extended: true }));   // parse HTML form bodies
  app.use(cookieParser());

  app.get('/health', (_req, res) => res.json({ status: 'ok' }));
  app.use('/v1', apiRouter);

  app.use(notFound);                                 // 404 for unknown routes
  app.use(errorHandler);                             // MUST be last
  return app;
}
```

```ts
// src/server.ts
import { createApp } from './app.ts';
import { env } from './config/env.ts';
import { logger } from './lib/logger.ts';

const server = createApp().listen(env.PORT, () => logger.info(`PharmaCare API on :${env.PORT}`));
// graceful shutdown: section 18
```

Splitting `app.ts` from `server.ts` lets tests import the app without opening a port.

## 4.5 Routing

**What it is:** A route = HTTP method + path + handler(s).

```ts
import { Router } from 'express';

const router = Router();

router.get('/medicines', listMedicines);              // GET /v1/medicines
router.get('/medicines/:slug', getMedicine);          // route parameter
router.post('/medicines', createMedicine);
router.patch('/medicines/:id', updateMedicine);
router.delete('/medicines/:id', deleteMedicine);

// Chain methods on one path
router.route('/batches/:id').get(getBatch).patch(updateBatch);

// Express 5 path syntax
router.get('/files/*filepath', serveFile);             // wildcard must be named
router.get('/reports{/:year}', report);                // optional segment in braces
```

**Mounting routers by feature:**

```ts
// src/routes.ts
import { Router } from 'express';
import { medicineRouter } from './modules/medicines/medicine.routes.ts';
import { saleRouter } from './modules/sales/sale.routes.ts';
import { authRouter } from './modules/auth/auth.routes.ts';

export const apiRouter = Router();
apiRouter.use('/auth', authRouter);           // /v1/auth/...
apiRouter.use('/medicines', medicineRouter);  // /v1/medicines/...
apiRouter.use('/sales', saleRouter);
```

## 4.6 The request and response objects

```ts
import type { Request, Response } from 'express';

async function listMedicines(req: Request, res: Response) {
  req.params;          // { slug: 'amoxicillin-500mg' } from the path
  req.query;           // { search: 'amox', page: '2' } — always strings, validate them
  req.body;            // parsed JSON (needs express.json())
  req.headers;         // lowercased header names
  req.get('authorization');
  req.cookies;         // needs cookie-parser
  req.ip;              // client IP (with trust proxy)
  req.method; req.path; req.originalUrl;

  res.status(200).json({ data: [] });    // JSON response
  res.status(201).location('/v1/sales/1052').json({ data: sale });
  res.sendStatus(204);                   // empty response
  res.set('Cache-Control', 'public, max-age=60');
  res.cookie('refresh_token', token, { httpOnly: true, secure: true, sameSite: 'strict', path: '/v1/auth' });
  res.clearCookie('refresh_token', { path: '/v1/auth' });
  res.redirect(301, '/v1/medicines');
  res.download('/tmp/report.pdf', 'daily-report.pdf');
  res.sendFile(absolutePath);
}
```

**Type your handlers** for params, body and query:

```ts
router.get('/:slug', async (req: Request<{ slug: string }>, res: Response<{ data: Medicine }>) => {
  const medicine = await medicineService.getBySlug(req.params.slug);
  res.json({ data: medicine });
});
```

## 4.7 Middleware

**What it is:** A function `(req, res, next)` that runs between receiving the request and sending the response. It can read or change `req`/`res`, end the response, or call `next()` to pass control to the next function. Middleware runs **in the order you add it**.

```
request → logger → helmet → cors → json parser → authenticate → authorize → validate → controller → response
                                                                   ↓ (any error)
                                                             error handler → response
```

| Kind | Example |
| --- | --- |
| Application-level | `app.use(express.json())` — every request |
| Router-level | `router.use(authenticate)` — every route in that router |
| Route-level | `router.post('/', authorize('admin'), handler)` — one route |
| Error-handling | `(err, req, res, next)` — four arguments |
| Built-in | `express.json`, `express.urlencoded`, `express.static`, `express.raw`, `express.text` |
| Third-party | `helmet`, `cors`, `compression`, `cookie-parser`, `express-rate-limit`, `multer` |

**Writing your own — a request ID for tracing logs:**

```ts
// src/middleware/request-id.ts
import { randomUUID } from 'node:crypto';
import type { RequestHandler } from 'express';

export const requestId: RequestHandler = (req, res, next) => {
  const id = req.get('x-request-id') ?? randomUUID();
  res.set('x-request-id', id);
  res.locals.requestId = id;      // res.locals: per-request storage
  next();                         // forget this and the request hangs forever
};
```

**Route-level middleware — only pharmacists and admins can create medicines:**

```ts
medicineRouter.post('/', authenticate, authorize('pharmacist', 'admin'), validate({ body: CreateMedicineSchema }), controller.create);
```

(`authenticate` and `authorize` are written in section 13, `validate` in section 5.)

## 4.8 Serving static files and other body types

```ts
app.use('/public', express.static(path.join(import.meta.dirname, '..', 'public'), { maxAge: '7d' }));

// Webhooks that need the raw body to verify a signature
app.post('/v1/payments/webhook', express.raw({ type: 'application/json' }), handleWebhook);
```

Register raw-body routes **before** `express.json()`, or the body is already parsed.
