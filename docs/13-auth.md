# 13. Authentication and authorisation

**Authentication** = proving who you are (login). **Authorisation** = what you are allowed to do (roles, permissions).

## 13.1 Storing passwords

**Never store passwords.** Store a **slow, salted hash**. When the user logs in, hash what they typed and compare. Use **Argon2id** (recommended) or **bcrypt**. Never MD5, SHA-1 or plain SHA-256 — they are too fast to brute-force.

```bash
npm install argon2
```

```ts
import argon2 from 'argon2';

const passwordHash = await argon2.hash(plainPassword);               // salt included automatically
const ok = await argon2.verify(user.passwordHash, plainPassword);    // true / false
```

Password rules: minimum 10–12 characters, check against breached-password lists, no forced complexity puzzles, allow password managers.

## 13.2 Sessions vs tokens

|  | Server sessions | JWT (JSON Web Token) |
| --- | --- | --- |
| Where state lives | Server (Redis/DB); client holds an opaque session ID cookie | Inside the signed token itself |
| Revoke instantly | Yes — delete the session | Hard — valid until it expires (hence short lifetimes) |
| Scales across servers | Needs a shared store (Redis) | Yes, any server can verify the signature |
| Best for | Same-site web apps | APIs used by SPAs, mobile apps, other services |

**PharmaCare's approach (common best practice):**

- **Access token** — a JWT valid for **15 minutes**, sent as `Authorization: Bearer <token>`. Contains `sub` (user id) and `role`.
- **Refresh token** — a random, opaque string valid for **7 days**, stored as an **HttpOnly, Secure, SameSite=Strict cookie** scoped to `/v1/auth`. Stored **hashed** in PostgreSQL so it can be revoked. **Rotated** on every use: using an old one means it was stolen, so the whole session family is revoked.

## 13.3 JWTs explained

A JWT has three base64url parts: `header.payload.signature`. The payload is **readable by anyone** (it is not encrypted) — never put secrets or personal data in it. The signature proves the server created it and nobody changed it.

```bash
npm install jose
```

```ts
// src/lib/tokens.ts
import { SignJWT, jwtVerify } from 'jose';
import { randomBytes, createHash } from 'node:crypto';
import { env } from '../config/env.ts';

const accessKey = new TextEncoder().encode(env.JWT_ACCESS_SECRET);

export function signAccessToken(user: { id: number; role: Role }) {
  return new SignJWT({ role: user.role })
    .setProtectedHeader({ alg: 'HS256' })
    .setSubject(String(user.id))
    .setIssuer('pharmacare-api')
    .setAudience('pharmacare')
    .setIssuedAt()
    .setExpirationTime('15m')
    .sign(accessKey);
}

export async function verifyAccessToken(token: string) {
  const { payload } = await jwtVerify(token, accessKey, { issuer: 'pharmacare-api', audience: 'pharmacare' });
  return { id: Number(payload.sub), role: payload.role as Role };
}

export const newRefreshToken = () => randomBytes(32).toString('base64url');
export const hashToken = (t: string) => createHash('sha256').update(t).digest('hex');
```

For tokens other services must verify without sharing your secret, sign with an asymmetric key (`RS256`/`EdDSA`) and publish the public key (JWKS).

## 13.4 Login, refresh and logout

```ts
// schema: refresh_tokens(id, user_id, token_hash, family_id, expires_at, revoked_at, user_agent, ip, created_at)

// src/modules/auth/auth.service.ts
export const authService = {
  async login(email: string, password: string, meta: { ip: string; userAgent?: string }) {
    const user = await db.query.users.findFirst({ where: eq(users.email, email.toLowerCase()) });
    // Same error and similar timing whether the email exists or not (no user enumeration)
    const valid = user ? await argon2.verify(user.passwordHash, password) : await argon2.verify(DUMMY_HASH, password).then(() => false);
    if (!user || !valid || !user.isActive) throw new AppError(401, 'Invalid email or password', 'INVALID_CREDENTIALS');

    return issueTokens(user, crypto.randomUUID(), meta);
  },

  async refresh(oldToken: string, meta: { ip: string; userAgent?: string }) {
    const row = await db.query.refreshTokens.findFirst({ where: eq(refreshTokens.tokenHash, hashToken(oldToken)) });
    if (!row || row.expiresAt < new Date()) throw new AppError(401, 'Session expired', 'SESSION_EXPIRED');

    if (row.revokedAt) {
      // REUSE DETECTED: an old token was used again → likely stolen. Revoke the whole family.
      await db.update(refreshTokens).set({ revokedAt: new Date() }).where(eq(refreshTokens.familyId, row.familyId));
      throw new AppError(401, 'Session revoked', 'TOKEN_REUSE');
    }

    await db.update(refreshTokens).set({ revokedAt: new Date() }).where(eq(refreshTokens.id, row.id));   // rotate
    const user = await db.query.users.findFirst({ where: eq(users.id, row.userId) });
    if (!user?.isActive) throw new AppError(401, 'Account disabled', 'ACCOUNT_DISABLED');
    return issueTokens(user, row.familyId, meta);
  },

  logout: (token: string) =>
    db.update(refreshTokens).set({ revokedAt: new Date() }).where(eq(refreshTokens.tokenHash, hashToken(token))),
};

async function issueTokens(user: User, familyId: string, meta: { ip: string; userAgent?: string }) {
  const refreshToken = newRefreshToken();
  await db.insert(refreshTokens).values({
    userId: user.id, tokenHash: hashToken(refreshToken), familyId,
    expiresAt: new Date(Date.now() + 7 * 24 * 3600 * 1000), ip: meta.ip, userAgent: meta.userAgent,
  });
  return {
    accessToken: await signAccessToken(user),
    refreshToken,
    user: { id: user.id, name: user.name, role: user.role },
  };
}
```

```ts
// src/modules/auth/auth.routes.ts
const REFRESH_COOKIE = { httpOnly: true, secure: true, sameSite: 'strict' as const, path: '/v1/auth', maxAge: 7 * 24 * 3600 * 1000 };

authRouter.post('/login', loginLimiter, validate({ body: LoginSchema }), async (req, res) => {
  const { accessToken, refreshToken, user } = await authService.login(req.body.email, req.body.password, { ip: req.ip!, userAgent: req.get('user-agent') });
  res.cookie('refresh_token', refreshToken, REFRESH_COOKIE);
  res.json({ data: { accessToken, user } });
});

authRouter.post('/refresh', async (req, res) => {
  const token = req.cookies.refresh_token;
  if (!token) throw new AppError(401, 'Not logged in', 'NO_SESSION');
  const { accessToken, refreshToken, user } = await authService.refresh(token, { ip: req.ip!, userAgent: req.get('user-agent') });
  res.cookie('refresh_token', refreshToken, REFRESH_COOKIE);
  res.json({ data: { accessToken, user } });
});

authRouter.post('/logout', async (req, res) => {
  if (req.cookies.refresh_token) await authService.logout(req.cookies.refresh_token);
  res.clearCookie('refresh_token', { path: '/v1/auth' });
  res.sendStatus(204);
});
```

## 13.5 The `authenticate` middleware

```ts
// src/middleware/auth.ts
import type { RequestHandler } from 'express';
import { verifyAccessToken } from '../lib/tokens.ts';
import { AppError } from '../lib/errors.ts';

export const authenticate: RequestHandler = async (req, res, next) => {
  const header = req.get('authorization');
  if (!header?.startsWith('Bearer ')) throw new AppError(401, 'Missing access token', 'UNAUTHENTICATED');
  try {
    res.locals.user = await verifyAccessToken(header.slice(7));   // { id, role }
  } catch {
    throw new AppError(401, 'Invalid or expired access token', 'TOKEN_INVALID');   // client should call /auth/refresh
  }
  next();
};
```

## 13.6 Authorisation: roles and ownership

**Role-based access control (RBAC):**

```ts
export function authorize(...roles: Role[]): RequestHandler {
  return (_req, res, next) => {
    if (!roles.includes(res.locals.user.role)) throw new AppError(403, 'You are not allowed to do this', 'FORBIDDEN');
    next();
  };
}

reportRouter.get('/daily', authenticate, authorize('pharmacist', 'admin'), controller.daily);
```

**Permission-based** (finer than roles, easier to change):

```ts
const PERMISSIONS = {
  admin: ['medicine:write', 'sale:create', 'sale:void', 'report:read', 'user:manage', 'price:change'],
  pharmacist: ['medicine:write', 'sale:create', 'sale:void', 'report:read', 'price:change'],
  cashier: ['sale:create'],
} as const satisfies Record<Role, readonly string[]>;

export const can = (permission: string): RequestHandler => (_req, res, next) => {
  if (!(PERMISSIONS[res.locals.user.role as Role] as readonly string[]).includes(permission)) throw new AppError(403, 'Forbidden', 'FORBIDDEN');
  next();
};

saleRouter.post('/:id/void', authenticate, can('sale:void'), controller.void);
```

**Resource-level checks** (in the service): a cashier may view only their own sales; a branch manager only their branch's data. Checking only the role is the #1 API vulnerability (broken object-level authorisation, section 16).

```ts
if (user.role === 'cashier' && sale.cashierId !== user.id) throw new AppError(404, 'Sale not found', 'NOT_FOUND');   // 404 hides existence
```

## 13.7 Other authentication topics

- **Login protection:** rate limit `/auth/login` per IP **and** per email (section 16); lock or slow down after repeated failures; alert on unusual logins.
- **Password reset:** generate a random token, store its **hash** with a 30-minute expiry, email a link, accept it once, then revoke all refresh tokens of the user.
- **Two-factor authentication (2FA):** TOTP codes from an authenticator app (`otplib`), or passkeys/WebAuthn (`@simplewebauthn/server`) for phishing-resistant login. Require it for admins.
- **OAuth 2.0 / OpenID Connect:** "Login with Google" — libraries **Arctic** or **Passport.js**, or a hosted provider (Auth0, Clerk, Keycloak). Always validate the `state` parameter and use PKCE.
- **API keys for partners:** generate random keys, show them once, store only a hash, scope them, allow rotation.
- **Service-to-service:** API keys, signed JWTs with short lifetimes, or mTLS.

## 13.8 How the Next.js frontend uses this

Two safe patterns (details in section 21):

1. **BFF pattern (recommended for Next.js):** the browser talks only to Next.js; Next.js Server Actions call the Express API and keep tokens in **HttpOnly cookies on the Next.js domain**. The browser never sees a token.
2. **SPA pattern:** the browser keeps the access token **in memory** (never `localStorage`) and calls `/v1/auth/refresh` (cookie sent automatically) when it gets a `401`, then retries the request.
