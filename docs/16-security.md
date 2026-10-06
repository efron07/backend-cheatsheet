# 16. API and backend security

## 16.1 OWASP API Security Top 10 (2023) in PharmaCare terms

| Risk | What goes wrong | Defence |
| --- | --- | --- |
| **API1 Broken object-level authorisation** | Cashier changes `/v1/sales/1052` to `/1053` and sees another cashier's sale | Check ownership/branch on every object access (section 13.6) |
| **API2 Broken authentication** | Weak passwords, no rate limit on login, tokens that never expire | Argon2, rate limits, short JWTs, refresh rotation |
| **API3 Broken object-property-level authorisation** | API returns `passwordHash`; client sends `{ "role": "admin" }` in a profile update | Explicit response DTOs; Zod schemas that only accept allowed fields (mass-assignment protection) |
| **API4 Unrestricted resource consumption** | `?limit=1000000`, huge uploads, unlimited SMS sends | Cap limits, body size limits, rate limits, timeouts |
| **API5 Broken function-level authorisation** | Cashier calls `DELETE /v1/medicines/7` directly | `authorize`/`can` middleware on every route |
| **API6 Unrestricted access to sensitive business flows** | A bot reserves all stock of a scarce medicine | Per-user limits, quantity caps, anomaly alerts |
| **API7 Server-side request forgery (SSRF)** | Endpoint fetches a user-supplied URL (`imageUrl`) that points to internal services | Allowlist domains; never fetch arbitrary URLs |
| **API8 Security misconfiguration** | Stack traces in responses, open CORS, debug endpoints live | Helmet, strict CORS, generic 500 errors, hardened config |
| **API9 Improper inventory management** | Old `/v0` endpoints still running without auth | Document every endpoint (OpenAPI), retire old versions |
| **API10 Unsafe consumption of APIs** | Blindly trusting M-Pesa/supplier responses | Validate external data with Zod; verify webhook signatures and amounts |

## 16.2 Security middleware

```bash
npm install helmet cors express-rate-limit rate-limit-redis
```

```ts
// Security headers (CSP, HSTS, no-sniff, frame protections...)
app.use(helmet());

// CORS: only your frontends, with credentials for the refresh cookie
app.use(cors({
  origin: env.CORS_ORIGINS,                 // ['https://pharmacare.co.tz', 'http://localhost:3000'] — never '*' with credentials
  credentials: true,
  methods: ['GET', 'POST', 'PATCH', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'Idempotency-Key'],
  maxAge: 600,
}));

// Body size limits
app.use(express.json({ limit: '1mb' }));
```

**CORS reminder:** CORS is enforced by **browsers** only. It does not protect your API from scripts, Postman or other servers — authentication does.

## 16.3 Rate limiting

```ts
// src/middleware/rate-limit.ts
import { rateLimit } from 'express-rate-limit';
import { RedisStore } from 'rate-limit-redis';
import { redis } from '../lib/redis.ts';

const store = (prefix: string) => new RedisStore({ prefix, sendCommand: (...args: string[]) => redis.call(...(args as [string, ...string[]])) as never });

export const apiLimiter = rateLimit({
  windowMs: 60_000, limit: 120,                         // 120 requests per minute
  standardHeaders: 'draft-8', legacyHeaders: false,
  store: store('rl:api:'),
});

export const loginLimiter = rateLimit({
  windowMs: 15 * 60_000, limit: 10,                     // 10 login attempts per 15 minutes
  keyGenerator: (req) => `${req.ip}:${String(req.body?.email ?? '').toLowerCase()}`,   // per IP + email
  store: store('rl:login:'),
  message: { type: 'too_many_requests', title: 'Too many login attempts. Try again later.', status: 429 },
});

app.use('/v1', apiLimiter);
```

Also rate-limit expensive or abusable actions: SMS sending, password reset, report generation, search.

## 16.4 Input validation and injection

- **Validate everything** with Zod: body, params, query, headers you use, webhook payloads, environment variables.
- **SQL injection:** always use parameterised queries (Drizzle/Prisma do this). Never build SQL by joining strings with user input; whitelist sort columns.
- **NoSQL injection:** never pass raw `req.body` objects into Mongo filters (`{ $ne: null }` tricks); validate types first.
- **Command injection:** never pass user input to `exec`; use `execFile`/`spawn` with argument arrays.
- **Path traversal:** never build file paths from user input (`../../.env`); use generated keys.
- **Prototype pollution:** do not deep-merge untrusted objects into config; Zod strips unknown keys by default.
- **ReDoS:** avoid complex user-controlled regular expressions.

## 16.5 Secrets management

- Secrets live in **environment variables** or a secrets manager (Doppler, Infisical, AWS Secrets Manager, your host's encrypted env settings) — never in code or Git.
- Commit a `.env.example` with placeholder values; add `.env` to `.gitignore`.
- Use different secrets per environment; rotate them when people leave or a leak is suspected.
- Scan the repo for leaked secrets (GitHub secret scanning, `gitleaks`).
- Least privilege everywhere: a database role with only the needed grants (section 9.8), API keys with minimal scopes.

## 16.6 Transport and data protection

- **HTTPS only**; HSTS header; redirect HTTP to HTTPS at the proxy.
- TLS to the databases (`sslmode=require` for Neon; Atlas uses TLS by default).
- **Encrypt sensitive fields** at rest when required (patient/prescription data) with `node:crypto` AES-256-GCM and keys from a KMS.
- **Personal data:** collect only what you need, restrict who can read it, log access to sensitive records, and follow Tanzania's Personal Data Protection Act and health-data rules.
- **Logs:** redact tokens, passwords, cookies and full card or ID numbers (pino `redact`, section 5.4).

## 16.7 Dependency and supply-chain security

```bash
npm audit                     # known vulnerabilities
npm audit fix
npx npm-check-updates         # see available updates
```

- Enable Dependabot or Renovate for automatic update PRs.
- Commit the lockfile; use `npm ci` in CI and production.
- Prefer well-maintained packages; review new dependencies before adding them.
- Run the API as a non-root user in Docker (section 20).

## 16.8 Security checklist

- [ ] Every route has `authenticate` (except public ones listed explicitly) and an authorisation check.
- [ ] Object-level checks in services for anything user- or branch-owned.
- [ ] Zod validation on all inputs; responses never include secrets.
- [ ] Helmet, strict CORS, body size limits, rate limits (global + login + expensive actions).
- [ ] Passwords hashed with Argon2id; short-lived JWTs; rotating refresh tokens in HttpOnly cookies.
- [ ] Webhooks verified, idempotent, amounts checked.
- [ ] Secrets in env/secret manager; `.env` git-ignored; DB users least-privileged.
- [ ] Generic 500 messages; detailed errors only in logs; logs redact sensitive fields.
- [ ] Dependencies audited and updated automatically.
- [ ] HTTPS everywhere; backups encrypted and restore-tested.
