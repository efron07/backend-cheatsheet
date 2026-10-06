# 2. Node.js basics

## 2.1 What Node.js is

**What it is:** Node.js is a **runtime** that runs JavaScript outside the browser, on servers and your computer. It uses Chrome's V8 engine plus **libuv**, a C library that handles files, network and timers asynchronously.

**Why it is good for APIs:** most API work is waiting — for the database, for M-Pesa, for disk. Node's **non-blocking I/O** lets one process handle thousands of connections while it waits, instead of one thread per request.

**Not good for:** long CPU-heavy work (image processing, big PDF reports) on the main thread — it blocks everyone. Move that to worker threads or background jobs (sections 3, 15).

**Node vs browser JS:** same language, different environment. Node has no `window` or `document`, but has `process`, the file system (`fs`), networking (`http`), and npm's huge package ecosystem. Alternatives: **Bun** and **Deno** (also run TS natively).

## 2.2 Installing and versions

Use a version manager so each project can pin its Node version:

```bash
# nvm (macOS/Linux) — or fnm, or Volta; on Windows use nvm-windows or fnm
nvm install --lts          # install the latest Long-Term Support version
nvm use 22
node -v; npm -v
echo "22" > .nvmrc         # pin the version for the project
```

Use only **LTS** (even-numbered) versions in production. Odd versions are short-lived "current" releases.

## 2.3 Running code

```bash
node                       # REPL: interactive console (.exit to quit)
node app.js                # run a file
node --watch app.js        # restart on file changes (built in)
node --env-file=.env app.js        # load environment variables (built in, no dotenv needed)
node src/server.ts         # recent Node versions strip TypeScript types and run .ts directly
npx tsx watch src/server.ts        # tsx: run TS with full support + watch mode (common in dev)
```

## 2.4 Modules: ESM and CommonJS

**ES Modules (use these):** set `"type": "module"` in `package.json`.

```js
// src/utils/money.js
export function formatTZS(amount) {
  return new Intl.NumberFormat('en-TZ').format(amount) + ' TZS';
}

// src/server.js
import { formatTZS } from './utils/money.js';    // ESM in Node needs the file extension in plain JS
import fs from 'node:fs/promises';               // `node:` prefix for built-in modules
```

**CommonJS (older):**

```js
const express = require('express');
module.exports = { formatTZS };
```

In ESM, `__dirname` is `import.meta.dirname` and `__filename` is `import.meta.filename`. You can `import` CommonJS packages from ESM; the reverse needs `require(esm)` support (available in recent Node versions) or dynamic `import()`.

## 2.5 npm and package.json

```bash
npm init -y                         # create package.json
npm install express zod             # dependencies
npm install -D typescript tsx @types/node @types/express vitest   # dev dependencies
npm run dev                         # run a script
npm ci                              # clean, exact install from the lockfile (CI, servers)
npm outdated; npm update; npm audit
npx <tool>                          # run a package binary without installing globally
```

```json
{
  "name": "pharmacare-api",
  "version": "1.0.0",
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "tsx watch --env-file=.env src/server.ts",
    "build": "tsc -p tsconfig.build.json",
    "start": "node --env-file-if-exists=.env dist/server.js",
    "test": "vitest",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate"
  }
}
```

See the Frontend sheet (section 25.3) for pnpm/Yarn/Bun equivalents and semantic versioning.

## 2.6 Global objects

| Global | What it is |
| --- | --- |
| `process` | The running process: `process.env`, `process.argv`, `process.exit()`, `process.on('SIGTERM')`, `process.memoryUsage()` |
| `console` | Logging |
| `globalThis` | The global object |
| `Buffer` | Binary data (section 3) |
| `setTimeout`, `setInterval`, `setImmediate`, `queueMicrotask` | Timers and scheduling |
| `fetch`, `URL`, `URLSearchParams`, `AbortController`, `structuredClone`, `crypto` | Web-standard APIs built into Node |
| `import.meta.dirname`, `import.meta.filename` | Current folder and file (ESM) |

## 2.7 Environment variables and `process`

```ts
// src/config/env.ts — validate configuration at startup (fail fast)
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(4000),
  DATABASE_URL: z.url(),                         // Neon Postgres
  MONGODB_URI: z.string().startsWith('mongodb'),
  REDIS_URL: z.url(),
  JWT_ACCESS_SECRET: z.string().min(32),
  JWT_REFRESH_SECRET: z.string().min(32),
  CORS_ORIGINS: z.string().transform((s) => s.split(',')),   // "https://pharmacare.co.tz,http://localhost:3000"
  MPESA_CONSUMER_KEY: z.string(),
  MPESA_CONSUMER_SECRET: z.string(),
});

export const env = EnvSchema.parse(process.env);   // crashes on startup if anything is missing or wrong
```

```ini
# .env — never commit; commit a .env.example with dummy values instead
NODE_ENV=development
PORT=4000
DATABASE_URL=postgresql://user:pass@ep-cool-name-123456-pooler.eu-central-1.aws.neon.tech/pharmacare?sslmode=require
MONGODB_URI=mongodb+srv://user:pass@cluster0.abcde.mongodb.net/pharmacare
REDIS_URL=redis://localhost:6379
```

**Command-line arguments:** `process.argv` (or `util.parseArgs`) for scripts like `node scripts/import-prices.js --file prices.csv`.

## 2.8 Built-in modules

| Module | Use | PharmaCare example |
| --- | --- | --- |
| `node:fs/promises` | Read/write files | Read a supplier CSV, write a report |
| `node:path` | Build file paths safely across OSes | `path.join(dir, 'reports', name)` |
| `node:http` | Low-level HTTP server (Express uses it) | — |
| `node:events` | EventEmitter | Internal events (`sale.completed`) |
| `node:stream` | Process data piece by piece | Stream a large CSV export |
| `node:crypto` | Hashing, random values, HMAC | Verify webhook signatures, `randomUUID()` |
| `node:os` | System info | CPU count for clustering |
| `node:child_process` | Run other programs | Call a PDF tool |
| `node:worker_threads` | Run JS on other threads | Heavy report calculation |
| `node:util` | `promisify`, `parseArgs`, `inspect`, `styleText` | Helpers |
| `node:test`, `node:assert` | Built-in test runner | Quick tests without a framework |
| `node:zlib` | gzip/brotli compression | Compress exports |
| `node:url` | URLs and `fileURLToPath` | Paths in ESM |

```ts
import { readFile, writeFile, mkdir } from 'node:fs/promises';
import path from 'node:path';
import { randomUUID, createHash } from 'node:crypto';

const dir = path.join(import.meta.dirname, '..', 'storage', 'reports');
await mkdir(dir, { recursive: true });
await writeFile(path.join(dir, 'daily.json'), JSON.stringify(report, null, 2));
const csv = await readFile('supplier-prices.csv', 'utf8');

randomUUID();                                                  // '7f9c2e7a-...'
createHash('sha256').update(fileBuffer).digest('hex');         // file fingerprint
```

## 2.9 A minimal HTTP server without Express

Worth seeing once, to understand what Express does for you:

```ts
import http from 'node:http';

const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({ status: 'ok' }));
  }
  res.writeHead(404, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ message: 'Not found' }));
});

server.listen(4000, () => console.log('Listening on http://localhost:4000'));
```

You would have to parse URLs, bodies, routes and errors yourself. Express (section 4) does that.
