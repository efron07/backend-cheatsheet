# 12. MongoDB 2: indexes, aggregation and Mongoose

## 12.1 Indexes

Without an index MongoDB scans every document (**COLLSCAN**). Index the fields you filter and sort on.

```js
db.auditLogs.createIndex({ entity: 1, entityId: 1, at: -1 });     // compound: history of one record, newest first
db.auditLogs.createIndex({ actorId: 1, at: -1 });                  // what did this user do?
db.reviews.createIndex({ medicineId: 1, createdAt: -1 });
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });   // TTL: auto-delete when expired
db.auditLogs.createIndex({ at: 1 }, { expireAfterSeconds: 60 * 60 * 24 * 365 * 2 });   // keep logs 2 years
db.reviews.createIndex({ comment: 'text' });                       // text search
db.supplierPrices.createIndex({ supplierId: 1, code: 1 }, { unique: true });
db.reviews.createIndex({ flagged: 1 }, { partialFilterExpression: { flagged: true } });   // partial

db.auditLogs.getIndexes();
db.auditLogs.find({ entity: 'medicine', entityId: 7 }).sort({ at: -1 }).explain('executionStats');   // IXSCAN good, COLLSCAN bad
```

**ESR rule** for compound index order: **E**quality fields first, then **S**ort fields, then **R**ange fields.

## 12.2 The aggregation pipeline

**What it is:** A pipeline of stages that transform documents step by step, like an assembly line. It is MongoDB's tool for reports, grouping and joins.

| Stage | Does |
| --- | --- |
| `$match` | Filter (put it first to use indexes) |
| `$project` / `$addFields` / `$set` | Choose, compute or rename fields |
| `$group` | Group and aggregate (`$sum`, `$avg`, `$min`, `$max`, `$count`, `$push`) |
| `$sort`, `$limit`, `$skip` | Order and page |
| `$lookup` | Join another collection |
| `$unwind` | One document per array element |
| `$facet` | Several sub-pipelines in one pass (results + total count) |
| `$bucket` | Group into ranges |
| `$out` / `$merge` | Write results to a collection (materialised reports) |

```js
// Average rating and review count per medicine, best rated first
db.reviews.aggregate([
  { $match: { createdAt: { $gte: ISODate('2026-01-01') } } },
  { $group: { _id: '$medicineId', avgRating: { $avg: '$rating' }, reviews: { $sum: 1 } } },
  { $match: { reviews: { $gte: 5 } } },
  { $sort: { avgRating: -1 } },
  { $limit: 10 },
  { $project: { _id: 0, medicineId: '$_id', avgRating: { $round: ['$avgRating', 1] }, reviews: 1 } },
]);

// Who changed prices most this month
db.auditLogs.aggregate([
  { $match: { action: 'medicine.updated', 'after.price': { $exists: true }, at: { $gte: ISODate('2026-10-01') } } },
  { $group: { _id: '$actorId', changes: { $sum: 1 }, lastChange: { $max: '$at' } } },
  { $sort: { changes: -1 } },
]);

// Paginated list + total in one query
db.reviews.aggregate([
  { $match: { medicineId: 7 } },
  { $sort: { createdAt: -1 } },
  { $facet: { data: [{ $skip: 0 }, { $limit: 20 }], total: [{ $count: 'count' }] } },
]);
```

## 12.3 Transactions

Single-document writes are always atomic in MongoDB, which is why embedding often removes the need for transactions. Multi-document transactions exist (on replica sets, including Atlas) but are slower; prefer good modelling. Example in Mongoose below.

## 12.4 Replication and sharding

- **Replica set** — one primary + secondaries holding copies; automatic failover if the primary dies. Atlas clusters are replica sets. **Read preference** can send reports to secondaries; **write concern** `majority` waits until most members have the write (safer).
- **Sharding** — splits a huge collection across servers by a **shard key** (e.g. `{ branchId: 1, at: 1 }`). Only needed at very large scale; choosing the shard key is the critical decision.

## 12.5 Mongoose: MongoDB in Node.js

**What it is:** Mongoose is an ODM (Object Document Mapper): it adds schemas, validation, defaults, middleware and typed models on top of the official MongoDB driver. Use the official `mongodb` driver directly when you want less abstraction.

```bash
npm install mongoose
```

```ts
// src/db/mongo.ts
import mongoose from 'mongoose';
import { env } from '../config/env.ts';
import { logger } from '../lib/logger.ts';

export async function connectMongo() {
  mongoose.set('strictQuery', true);
  await mongoose.connect(env.MONGODB_URI, { maxPoolSize: 10, serverSelectionTimeoutMS: 5000 });
  logger.info('MongoDB connected');
}
export const disconnectMongo = () => mongoose.disconnect();
```

```ts
// src/modules/audit/audit.model.ts
import { Schema, model, type InferSchemaType } from 'mongoose';

const auditLogSchema = new Schema({
  actorId: { type: Number, required: true, index: true },
  action: { type: String, required: true, match: /^[a-z]+\.[a-z_]+$/ },
  entity: { type: String, required: true, enum: ['medicine', 'batch', 'sale', 'user', 'supplier'] },
  entityId: { type: Number, required: true },
  before: Schema.Types.Mixed,
  after: Schema.Types.Mixed,
  ip: String,
  requestId: String,
  at: { type: Date, default: Date.now },
}, { versionKey: false });

auditLogSchema.index({ entity: 1, entityId: 1, at: -1 });
auditLogSchema.index({ at: 1 }, { expireAfterSeconds: 60 * 60 * 24 * 365 * 2 });

export type AuditLog = InferSchemaType<typeof auditLogSchema>;
export const AuditLogModel = model('AuditLog', auditLogSchema, 'auditLogs');
```

```ts
// src/modules/audit/audit.service.ts
import { AuditLogModel } from './audit.model.ts';

export const auditLog = {
  async record(entry: Omit<AuditLog, 'at'>) {
    try {
      await AuditLogModel.create(entry);
    } catch (err) {
      logger.error({ err, entry }, 'Audit log write failed');   // never fail the business action because logging failed
    }
  },

  history(entity: AuditLog['entity'], entityId: number, limit = 50) {
    return AuditLogModel.find({ entity, entityId })
      .sort({ at: -1 })
      .limit(limit)
      .select({ _id: 0, actorId: 1, action: 1, before: 1, after: 1, at: 1 })
      .lean();                          // plain objects: faster, less memory
  },

  list({ actorId, action, cursor, limit = 20 }: { actorId?: number; action?: string; cursor?: string; limit?: number }) {
    return AuditLogModel.find({
      ...(actorId && { actorId }),
      ...(action && { action }),
      ...(cursor && { _id: { $lt: cursor } }),            // cursor pagination by ObjectId
    }).sort({ _id: -1 }).limit(limit).lean();
  },
};
```

**A model with embedded documents and middleware — reviews:**

```ts
const replySchema = new Schema({ userId: Number, name: String, text: { type: String, maxlength: 1000 }, at: { type: Date, default: Date.now } }, { _id: false });

const reviewSchema = new Schema({
  medicineId: { type: Number, required: true, index: true },
  customer: { name: String, phone: String },
  rating: { type: Number, min: 1, max: 5, required: true },
  comment: { type: String, trim: true, maxlength: 2000 },
  replies: [replySchema],
  flagged: { type: Boolean, default: false },
}, { timestamps: true });                 // adds createdAt / updatedAt

reviewSchema.pre('save', function () {
  if (this.comment && /\b(fake|scam)\b/i.test(this.comment)) this.flagged = true;   // simple moderation hook
});

reviewSchema.virtual('isPositive').get(function () { return this.rating >= 4; });

export const ReviewModel = model('Review', reviewSchema);

// Add a reply atomically
await ReviewModel.updateOne({ _id: reviewId }, { $push: { replies: { userId: user.id, name: user.name, text } } });
```

**Other Mongoose features:** `populate()` (resolve references between collections), `aggregate()`, `bulkWrite()` (fast batch imports of supplier catalogues), discriminators (inheritance), and sessions for transactions:

```ts
const session = await mongoose.startSession();
await session.withTransaction(async () => {
  await ReviewModel.updateOne({ _id: id }, { $set: { flagged: false } }, { session });
  await AuditLogModel.create([{ actorId: user.id, action: 'review.unflagged', entity: 'medicine', entityId: medId }], { session });
});
await session.endSession();
```

## 12.6 Change streams

**What it is:** Subscribe to real-time changes in a collection (requires a replica set, e.g. Atlas). Useful to push events to dashboards or sync data elsewhere.

```ts
const stream = ReviewModel.watch([{ $match: { operationType: 'insert', 'fullDocument.rating': { $lte: 2 } } }]);
stream.on('change', (change) => notifyManager(`New low rating for medicine ${change.fullDocument.medicineId}`));
```

## 12.7 Operations: backups, security and performance

- **Backups:** Atlas continuous backups with point-in-time restore; self-hosted: `mongodump` / `mongorestore`.
- **Security:** database users with least privilege (`readWrite` on `pharmacare` only), IP access lists, TLS, never expose port 27017 to the internet.
- **Performance:** index for your queries (check `explain`), use projections and `.lean()`, avoid huge documents and unbounded arrays, use the Atlas Performance Advisor and profiler.
- **Injection:** never pass raw request objects into queries — `{ email: req.body.email }` where `email` is `{ "$ne": null }` matches everyone. Validate input with Zod first (Mongoose's `sanitizeFilter` option also helps).

```bash
mongodump --uri="$MONGODB_URI" --out=backup-$(date +%F)
mongorestore --uri="$TARGET_URI" backup-2026-10-06
```
