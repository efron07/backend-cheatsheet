# 21. Connecting the Next.js frontend

## 21.1 The recommended setup: Next.js as BFF

```
Browser ──(cookies on pharmacare.co.tz)──▶ Next.js (Server Components, Server Actions)
                                                   │  Authorization: Bearer <access token>
                                                   ▼
                                     Express API (api.pharmacare.co.tz) ──▶ Neon / Atlas / Redis
```

- The browser talks only to Next.js; **tokens never reach browser JavaScript**.
- Next.js stores the access and refresh tokens in **HttpOnly cookies on its own domain**.
- Server Components fetch data from the API on the server; Server Actions perform writes.
- Public catalogue data can be cached in Next.js (`'use cache'`) and revalidated by the API.
- Interactive client-only features (live payment status) can call the API directly with CORS.

## 21.2 A server-side API client with automatic refresh

```ts
// pharmacare-web/src/lib/api.ts
import 'server-only';
import { cookies } from 'next/headers';

const API = process.env.API_URL!;           // https://api.pharmacare.co.tz/v1

export class ApiError extends Error {
  constructor(public status: number, public body: { type?: string; title?: string; errors?: Record<string, string[]> }) {
    super(body.title ?? `API error ${status}`);
  }
}

async function rawFetch(path: string, init: RequestInit & { token?: string } = {}) {
  return fetch(`${API}${path}`, {
    ...init,
    headers: {
      Accept: 'application/json',
      ...(init.body ? { 'Content-Type': 'application/json' } : {}),
      ...(init.token ? { Authorization: `Bearer ${init.token}` } : {}),
      ...init.headers,
    },
    signal: AbortSignal.timeout(10_000),
  });
}

export async function api<T>(path: string, init: RequestInit = {}): Promise<T> {
  const jar = await cookies();
  let res = await rawFetch(path, { ...init, token: jar.get('access_token')?.value });

  // Access token expired → refresh once, then retry
  if (res.status === 401 && jar.get('refresh_token')) {
    const refreshed = await rawFetch('/auth/refresh', { method: 'POST', headers: { Cookie: `refresh_token=${jar.get('refresh_token')!.value}` } });
    if (refreshed.ok) {
      const { data } = await refreshed.json();
      const newRefresh = refreshed.headers.getSetCookie().find((c) => c.startsWith('refresh_token='))?.split(';')[0]?.split('=')[1];
      try {                                              // cookies can only be SET in Server Actions / Route Handlers
        jar.set('access_token', data.accessToken, { httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge: 15 * 60 });
        if (newRefresh) jar.set('refresh_token', newRefresh, { httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge: 7 * 86400 });
      } catch { /* in a Server Component render: the refresh still works for this request */ }
      res = await rawFetch(path, { ...init, token: data.accessToken });
    }
  }

  if (res.status === 204) return undefined as T;
  const body = await res.json().catch(() => ({}));
  if (!res.ok) throw new ApiError(res.status, body);
  return body as T;
}
```

For an even simpler setup, put token refresh in Next.js `proxy.ts`, which can read and set cookies on every request before pages render.

## 21.3 Login from a Server Action

```ts
// pharmacare-web/src/app/(auth)/login/actions.ts
'use server';
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export async function login(_prev: unknown, formData: FormData) {
  const res = await fetch(`${process.env.API_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email: formData.get('email'), password: formData.get('password') }),
  });
  if (!res.ok) return { error: res.status === 429 ? 'Too many attempts. Try later.' : 'Invalid email or password.' };

  const { data } = await res.json();
  const refresh = res.headers.getSetCookie().find((c) => c.startsWith('refresh_token='))!.split(';')[0]!.split('=')[1]!;
  const jar = await cookies();
  jar.set('access_token', data.accessToken, { httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge: 15 * 60 });
  jar.set('refresh_token', refresh, { httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge: 7 * 86400 });
  redirect('/dashboard');
}
```

## 21.4 Reading data in Server Components

```tsx
// pharmacare-web/src/app/(public)/medicines/page.tsx
import { api } from '@/lib/api';

export default async function MedicinesPage({ searchParams }: PageProps<'/medicines'>) {
  const { search = '', page = '1' } = await searchParams;
  const qs = new URLSearchParams({ search: String(search), page: String(page), limit: '24' });
  const { data, meta } = await api<Paginated<Medicine>>(`/medicines?${qs}`);
  return <MedicineGrid medicines={data} meta={meta} />;
}
```

## 21.5 Writing data from Server Actions and showing API errors in forms

```ts
// pharmacare-web/src/app/(staff)/dashboard/stock/actions.ts
'use server';
import { updateTag } from 'next/cache';
import { api, ApiError } from '@/lib/api';
import { z } from 'zod';
import { batchSchema } from '@pharmacare/schemas';     // shared with the API (21.7)

export async function receiveStock(_prev: unknown, formData: FormData) {
  const parsed = batchSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { ok: false, errors: z.flattenError(parsed.error).fieldErrors };   // fast feedback

  try {
    await api('/batches', { method: 'POST', body: JSON.stringify(parsed.data) });
  } catch (e) {
    if (e instanceof ApiError && e.status === 422) return { ok: false, errors: e.body.errors };     // API validation → fields
    if (e instanceof ApiError && e.status === 409) return { ok: false, message: e.body.title };     // duplicate batch
    if (e instanceof ApiError && e.status === 403) return { ok: false, message: 'You are not allowed to receive stock.' };
    throw e;                                                                                         // → error.tsx
  }
  updateTag('medicines');
  return { ok: true, message: 'Stock received.' };
}
```

The API remains the source of truth for validation and permissions; validating in Next.js too gives instant feedback.

## 21.6 Keeping Next.js caches fresh: API → Next.js revalidation webhook

When stock or prices change in the API (from any client), tell Next.js to refresh cached pages:

```ts
// Express: after a price update or sale commit (from a queue job)
await fetch(`${env.WEB_URL}/api/revalidate`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'x-revalidate-secret': env.REVALIDATE_SECRET },
  body: JSON.stringify({ tags: ['medicines', `medicine-${medicine.slug}`] }),
});
```

```ts
// Next.js: app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache';
export async function POST(req: Request) {
  if (req.headers.get('x-revalidate-secret') !== process.env.REVALIDATE_SECRET) return new Response('Forbidden', { status: 403 });
  const { tags } = (await req.json()) as { tags: string[] };
  tags.forEach((t) => revalidateTag(t, 'max'));
  return Response.json({ revalidated: tags });
}
```

## 21.7 Sharing types and schemas (monorepo)

One repository with the web app, the API and shared packages — both sides import the same Zod schemas and types, so a change breaks the build instead of production.

```
pharmacare/
├── apps/
│   ├── web/          # Next.js
│   └── api/          # Express
├── packages/
│   ├── schemas/      # Zod schemas + inferred types (@pharmacare/schemas)
│   └── config/       # shared tsconfig, eslint
├── package.json      # workspaces
└── turbo.json        # Turborepo: build/test tasks with caching
```

```json5
// root package.json
{ "private": true, "workspaces": ["apps/*", "packages/*"], "scripts": { "dev": "turbo dev", "build": "turbo build", "test": "turbo test" } }
```

**Without a monorepo:** generate types from the API's OpenAPI document:

```bash
npx openapi-typescript https://api.pharmacare.co.tz/openapi.json -o src/types/api.d.ts
```

```ts
import createClient from 'openapi-fetch';
import type { paths } from '@/types/api';
const client = createClient<paths>({ baseUrl: process.env.API_URL });
const { data, error } = await client.GET('/v1/medicines/{slug}', { params: { path: { slug } } });   // fully typed
```

**End-to-end typed alternatives:** tRPC or oRPC (TypeScript-only RPC), or Hono's typed client.

## 21.8 Calling the API directly from the browser (when needed)

For live features (payment status via SSE, POS search-as-you-type):

- Configure **CORS** on the API for `https://pharmacare.co.tz` with `credentials: true` (section 16.2).
- Hosting the API on a subdomain of the same site (`api.pharmacare.co.tz`) keeps cookies **same-site**, so `SameSite=Lax/Strict` cookies work.
- In Client Components, keep the access token **in memory only** (from a Server Action or a short-lived endpoint), never in `localStorage`.
- Use TanStack Query for caching and retries on the client (Frontend sheet, section 15.3).

## 21.9 Environment variables across both apps

| Variable | Lives in | Example |
| --- | --- | --- |
| `API_URL` | Next.js (server only) | `https://api.pharmacare.co.tz/v1` |
| `NEXT_PUBLIC_API_URL` | Next.js (browser, only if calling directly) | Same URL |
| `REVALIDATE_SECRET` | Both | Shared secret for the revalidation webhook |
| `WEB_URL` | API | `https://pharmacare.co.tz` |
| `CORS_ORIGINS` | API | `https://pharmacare.co.tz,http://localhost:3000` |

## 21.10 Local development of the full stack

```bash
# terminal 1 — databases + API + worker
cd pharmacare-api && docker compose up -d && npm run dev        # http://localhost:4000, docs at /docs

# terminal 2 — frontend
cd pharmacare-web && npm run dev                                # http://localhost:3000 (API_URL=http://localhost:4000/v1)
```
