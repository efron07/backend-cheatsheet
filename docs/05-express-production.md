# 5. Express in production: errors, validation and layered architecture

## 5.1 Error handling

**What it is:** One central **error-handling middleware** (four arguments) turns every error into a consistent JSON response. In Express 5, errors thrown or rejected inside `async` handlers are forwarded to it automatically (in Express 4 you had to call `next(err)` or wrap handlers).

```ts
// src/middleware/errors.ts
import type { ErrorRequestHandler, RequestHandler } from 'express';
import { ZodError, z } from 'zod';
import { AppError } from '../lib/errors.ts';

export const notFound: RequestHandler = (req, _res, next) => {
  next(new AppError(404, `Route ${req.method} ${req.path} not found`, 'NOT_FOUND'));
};

export const errorHandler: ErrorRequestHandler = (err, req, res, _next) => {
  if (err instanceof ZodError) {
    return res.status(422).json({
      type: 'validation_error',
      title: 'Validation failed',
      status: 422,
      errors: z.flattenError(err).fieldErrors,      // { quantity: ['Must be at least 1'] }
    });
  }

  if (err instanceof AppError) {
    return res.status(err.status).json({ type: err.code.toLowerCase(), title: err.message, status: err.status, details: err.details });
  }

  if (err?.type === 'entity.parse.failed') {        // invalid JSON body
    return res.status(400).json({ type: 'bad_request', title: 'Malformed JSON', status: 400 });
  }

  req.log.error({ err }, 'Unhandled error');        // full details in the logs only
  res.status(500).json({ type: 'internal_error', title: 'Something went wrong', status: 500, requestId: res.locals.requestId });
};
```

**Rules:** never send stack traces or SQL errors to the client; always log them with the request ID; use specific status codes (section 6).

## 5.2 Validation with Zod middleware

**What it is:** Every input (body, params, query) is checked against a Zod schema **before** the controller runs. Invalid input never reaches your business logic.

```ts
// src/modules/medicines/medicine.schemas.ts
import { z } from 'zod';

export const CreateMedicineSchema = z.object({
  name: z.string().trim().min(2).max(150),
  categoryId: z.number().int().positive(),
  supplierId: z.number().int().positive().optional(),
  price: z.number().int().min(0),                 // whole TZS
  requiresPrescription: z.boolean().default(false),
  reorderLevel: z.number().int().min(0).default(20),
});
export type CreateMedicineInput = z.infer<typeof CreateMedicineSchema>;

export const UpdateMedicineSchema = CreateMedicineSchema.partial();

export const ListMedicinesQuery = z.object({
  search: z.string().trim().max(100).optional(),
  categoryId: z.coerce.number().int().optional(),
  inStock: z.stringbool().optional(),             // "true"/"false" → boolean
  sort: z.enum(['name', '-name', 'price', '-price']).default('name'),
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});
export type ListMedicinesQuery = z.infer<typeof ListMedicinesQuery>;

export const IdParams = z.object({ id: z.coerce.number().int().positive() });
```

```ts
// src/middleware/validate.ts
import type { RequestHandler } from 'express';
import type { ZodType } from 'zod';

export function validate(schemas: { body?: ZodType; params?: ZodType; query?: ZodType }): RequestHandler {
  return (req, res, next) => {
    if (schemas.body) req.body = schemas.body.parse(req.body);              // throws ZodError → 422
    if (schemas.params) res.locals.params = schemas.params.parse(req.params);
    if (schemas.query) res.locals.query = schemas.query.parse(req.query);    // req.query is read-only in Express 5
    next();
  };
}
```

## 5.3 Layered architecture

**What it is:** Split each feature into layers with one job each. This keeps code testable and lets you swap pieces (e.g. a different database) without touching business rules.

| Layer | Job | Knows about |
| --- | --- | --- |
| **Routes** | URL + method → middleware + controller | Express |
| **Controller** | Read validated input, call the service, shape the HTTP response | Express `req`/`res` |
| **Service** | Business rules (stock checks, FEFO, permissions logic), orchestrates repositories | Nothing about HTTP |
| **Repository** | Database queries | Drizzle / Mongoose |

```ts
// src/modules/medicines/medicine.routes.ts
import { Router } from 'express';
import { authenticate, authorize } from '../../middleware/auth.ts';
import { validate } from '../../middleware/validate.ts';
import * as controller from './medicine.controller.ts';
import { CreateMedicineSchema, UpdateMedicineSchema, ListMedicinesQuery, IdParams } from './medicine.schemas.ts';

export const medicineRouter = Router();

medicineRouter.get('/', validate({ query: ListMedicinesQuery }), controller.list);
medicineRouter.get('/:slug', controller.getBySlug);
medicineRouter.post('/', authenticate, authorize('pharmacist', 'admin'), validate({ body: CreateMedicineSchema }), controller.create);
medicineRouter.patch('/:id', authenticate, authorize('pharmacist', 'admin'), validate({ params: IdParams, body: UpdateMedicineSchema }), controller.update);
```

```ts
// src/modules/medicines/medicine.controller.ts
import type { Request, Response } from 'express';
import { medicineService } from './medicine.service.ts';

export async function list(_req: Request, res: Response) {
  const result = await medicineService.list(res.locals.query);
  res.json(result);                              // { data, meta } — see section 6
}

export async function getBySlug(req: Request<{ slug: string }>, res: Response) {
  res.json({ data: await medicineService.getBySlug(req.params.slug) });
}

export async function create(req: Request, res: Response) {
  const medicine = await medicineService.create(req.body, res.locals.user);
  res.status(201).location(`/v1/medicines/${medicine.slug}`).json({ data: medicine });
}

export async function update(req: Request, res: Response) {
  res.json({ data: await medicineService.update(res.locals.params.id, req.body, res.locals.user) });
}
```

```ts
// src/modules/medicines/medicine.service.ts
import { AppError } from '../../lib/errors.ts';
import { medicineRepository } from './medicine.repository.ts';
import { auditLog } from '../audit/audit.service.ts';
import { cache } from '../../lib/cache.ts';
import { slugify } from '../../lib/slugify.ts';

export const medicineService = {
  list: (query: ListMedicinesQuery) => medicineRepository.list(query),

  async getBySlug(slug: string) {
    const medicine = await medicineRepository.findBySlug(slug);
    if (!medicine) throw new AppError(404, 'Medicine not found', 'NOT_FOUND');
    return medicine;
  },

  async create(input: CreateMedicineInput, user: AuthUser) {
    const slug = slugify(input.name);
    if (await medicineRepository.findBySlug(slug)) throw new AppError(409, 'A medicine with this name already exists', 'DUPLICATE');
    const medicine = await medicineRepository.create({ ...input, slug });
    await auditLog.record({ actorId: user.id, action: 'medicine.created', entity: 'medicine', entityId: medicine.id, after: medicine });
    return medicine;
  },

  async update(id: number, input: UpdateMedicineInput, user: AuthUser) {
    const before = await medicineRepository.findById(id);
    if (!before) throw new AppError(404, 'Medicine not found', 'NOT_FOUND');
    const after = await medicineRepository.update(id, input);
    await auditLog.record({ actorId: user.id, action: 'medicine.updated', entity: 'medicine', entityId: id, before, after });
    await cache.del(`medicine:${before.slug}`);
    return after;
  },
};
```

The repository (Drizzle queries) is in section 10; the audit log (MongoDB) in section 12.

**Dependency injection (optional):** pass repositories into services as arguments (`createMedicineService({ repo, audit })`) instead of importing them, so tests can hand in fakes. Libraries like Awilix or tsyringe automate this; plain factory functions are usually enough.

## 5.4 Configuration and logging helpers

```ts
// src/lib/logger.ts — structured JSON logs (section 18)
import pino from 'pino';
import { env } from '../config/env.ts';

export const logger = pino({
  level: env.NODE_ENV === 'production' ? 'info' : 'debug',
  redact: ['req.headers.authorization', 'req.headers.cookie', '*.password'],   // never log secrets
  transport: env.NODE_ENV === 'development' ? { target: 'pino-pretty' } : undefined,
});
```

## 5.5 File uploads with Multer

**PharmaCare example:** upload a prescription scan.

```bash
npm install multer && npm install -D @types/multer
```

```ts
import multer from 'multer';
import { AppError } from '../../lib/errors.ts';

const upload = multer({
  storage: multer.memoryStorage(),                 // keep in memory, then send to object storage
  limits: { fileSize: 4 * 1024 * 1024 },           // 4 MB
  fileFilter: (_req, file, cb) => {
    const ok = ['application/pdf', 'image/jpeg', 'image/png'].includes(file.mimetype);
    cb(ok ? null : new AppError(422, 'Only PDF, JPG or PNG allowed', 'INVALID_FILE'), ok);
  },
});

saleRouter.post('/:id/prescription', authenticate, upload.single('scan'), async (req, res) => {
  const key = await storage.put(`prescriptions/${req.params.id}-${Date.now()}`, req.file!.buffer, req.file!.mimetype);
  res.status(201).json({ data: { key } });
});
```

Store files in object storage (Cloudflare R2, AWS S3), not on the server's disk: servers are replaceable and may run as several copies (section 15).
