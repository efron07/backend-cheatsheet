# 11. MongoDB 1: documents, CRUD and data modelling

## 11.1 What MongoDB is

**What it is:** MongoDB is a **document database** (NoSQL). Instead of tables and rows, it stores **documents** — JSON-like objects — inside **collections**. Documents in the same collection can have different fields, and related data can be **embedded** inside one document.

| SQL (PostgreSQL) | MongoDB |
| --- | --- |
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |
| Primary key | `_id` (automatically an `ObjectId`) |
| JOIN | Embedding, or `$lookup` in aggregation |
| Schema enforced by the DB | Flexible; optional JSON Schema validation, or enforced in code (Mongoose) |

**BSON:** MongoDB stores documents as BSON (binary JSON), which adds types JSON lacks: `ObjectId`, `Date`, `Int32`/`Int64`, `Decimal128`, binary data. Max document size is 16 MB.

**NoSQL families:** document (MongoDB), key-value (Redis), wide-column (Cassandra), graph (Neo4j), search (Elasticsearch).

**When MongoDB fits (in PharmaCare):**

- **Audit logs** — every change with a `before`/`after` snapshot of different shapes.
- **Activity events** — high-volume, append-only.
- **Customer reviews and notes** — nested replies, varying fields.
- **Supplier catalogue imports** — every supplier's price list has different columns.

**When it does not:** money, stock and anything needing multi-table constraints and joins — keep those in PostgreSQL.

## 11.2 Setup: Atlas, local and tools

1. **MongoDB Atlas** (cloud): create a free cluster, a database user, and allow your server's IP under Network Access. Copy the `mongodb+srv://` connection string into `MONGODB_URI`.
2. **Local:** `docker run -d --name mongo -p 27017:27017 mongo:8`.
3. **Tools:** **mongosh** (shell), **MongoDB Compass** (GUI), the VS Code MongoDB extension, Atlas Data Explorer.

```bash
mongosh "$MONGODB_URI"
```

```js
show dbs
use pharmacare                      // switch (created on first write)
show collections
db.auditLogs.countDocuments()
db.stats()
```

## 11.3 CRUD in mongosh

```js
// CREATE
db.reviews.insertOne({
  medicineId: 7,                                  // id from PostgreSQL
  customer: { name: 'Asha', phone: '0754000000' },
  rating: 5,
  comment: 'Genuine and fairly priced',
  tags: ['genuine', 'price'],
  createdAt: new Date(),
});

db.reviews.insertMany([{ medicineId: 7, rating: 4 }, { medicineId: 3, rating: 2, comment: 'Out of stock twice' }]);

// READ
db.reviews.find({ medicineId: 7 });                                  // all matches
db.reviews.findOne({ _id: ObjectId('6702...') });
db.reviews.find({ medicineId: 7 }, { rating: 1, comment: 1, _id: 0 });   // projection: only these fields
db.reviews.find().sort({ createdAt: -1 }).skip(0).limit(20);         // newest first, paginated

// UPDATE
db.reviews.updateOne({ _id: ObjectId('6702...') }, { $set: { comment: 'Updated text' }, $currentDate: { updatedAt: true } });
db.reviews.updateMany({ rating: { $lt: 3 } }, { $set: { flagged: true } });
db.reviews.replaceOne({ _id: id }, { medicineId: 7, rating: 5 });    // replaces the whole document

// UPSERT: update, or insert if not found
db.supplierPrices.updateOne(
  { supplierId: 4, code: 'AMX500' },
  { $set: { price: 350, updatedAt: new Date() } },
  { upsert: true },
);

// DELETE
db.reviews.deleteOne({ _id: id });
db.reviews.deleteMany({ flagged: true, createdAt: { $lt: ISODate('2025-01-01') } });
```

## 11.4 Query operators

| Kind | Operators | Example |
| --- | --- | --- |
| Comparison | `$eq $ne $gt $gte $lt $lte $in $nin` | `{ rating: { $gte: 4 } }`, `{ action: { $in: ['sale.voided', 'price.changed'] } }` |
| Logical | `$and $or $not $nor` | `{ $or: [{ rating: { $lte: 2 } }, { flagged: true }] }` |
| Element | `$exists $type` | `{ comment: { $exists: true } }` |
| Array | `$all $size $elemMatch` | `{ tags: { $all: ['genuine', 'price'] } }` |
| Text/pattern | `$regex`, `$text` (needs a text index) | `{ comment: { $regex: 'stock', $options: 'i' } }` |
| Nested fields | dot notation | `{ 'customer.phone': '0754000000' }` |

## 11.5 Update operators

| Operator | Does | Example |
| --- | --- | --- |
| `$set` / `$unset` | Set / remove fields | `{ $set: { status: 'resolved' } }` |
| `$inc` | Increment a number | `{ $inc: { helpfulVotes: 1 } }` |
| `$push` / `$addToSet` | Add to an array (`$addToSet` = no duplicates) | `{ $push: { replies: { by: 'pharmacist', text: '...' } } }` |
| `$pull` | Remove matching array items | `{ $pull: { tags: 'price' } }` |
| `$currentDate` | Set to now | `{ $currentDate: { updatedAt: true } }` |
| `$rename`, `$min`, `$max`, `$mul` | Rename / keep min / keep max / multiply |  |
| `$` / `$[]` / `$[elem]` | Update matching array elements | `{ $set: { 'replies.$[r].hidden': true } }` with `arrayFilters` |

## 11.6 Data modelling: embed or reference?

The main design question in MongoDB. **Design around how the data is read.**

| Embed (nest inside) | Reference (store an id) |
| --- | --- |
| Data is read together | Data is read separately |
| One-to-few (a review's replies) | One-to-many or many-to-many with large/unbounded sides |
| Child has no meaning alone | Child is shared or queried on its own |
| Doesn't grow without limit | Would grow forever (an "all events" array) |

```js
// EMBED: a review with its replies (always shown together, few replies)
{
  _id: ObjectId('...'),
  medicineId: 7,
  rating: 2,
  comment: 'Out of stock twice this month',
  replies: [
    { by: { userId: 3, name: 'Pharmacist Neema' }, text: 'Restocked today, sorry!', at: ISODate('2026-10-05') },
  ],
}

// REFERENCE: audit logs point to users and entities by id (unbounded, queried on their own)
{
  _id: ObjectId('...'),
  actorId: 3,                           // users.id in PostgreSQL
  action: 'medicine.updated',
  entity: 'medicine',
  entityId: 7,
  before: { price: 500 },
  after: { price: 550 },
  ip: '102.68.1.20',
  at: ISODate('2026-10-06T09:15:00Z'),
}
```

**Common schema patterns:** **subset** (embed only the latest 5 reviews in a summary document), **extended reference** (copy a few fields such as the customer's name to avoid lookups), **computed** (store running totals like `ratingAvg`), **bucket** (group time-series readings per hour), **outlier** (handle rare huge documents separately), **schema versioning** (`schemaVersion: 2` field while migrating).

**Anti-patterns:** unbounded arrays, massive documents, too many collections that need joining, and copying data that changes often.

## 11.7 Schema validation in the database

Even with a flexible schema, protect important collections:

```js
db.createCollection('auditLogs', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['actorId', 'action', 'entity', 'entityId', 'at'],
      properties: {
        actorId: { bsonType: ['int', 'long'] },
        action: { bsonType: 'string', pattern: '^[a-z]+\\.[a-z_]+$' },
        entity: { enum: ['medicine', 'batch', 'sale', 'user', 'supplier'] },
        at: { bsonType: 'date' },
      },
    },
  },
  validationLevel: 'strict',
  validationAction: 'error',
});
```

In Node.js, Mongoose schemas (section 12) add validation in code as well.

## 11.8 Capped and time series collections

- **Capped collections** have a fixed size and keep only the newest documents (rolling logs).
- **Time series collections** store measurements efficiently: `db.createCollection('posEvents', { timeseries: { timeField: 'at', metaField: 'branch', granularity: 'minutes' } })` — e.g. POS heartbeat/activity per branch.
