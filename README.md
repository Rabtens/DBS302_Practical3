# DBS302_Practical3

# DBS302 – Practical 3 Report
## E-Commerce Platform Schema in MongoDB: Aggregation Framework & Query Optimization


## 1. Aim and Objectives

To design and implement an e-commerce platform schema using MongoDB, write advanced aggregation queries, and apply indexing and query analysis techniques to optimize performance for real-world workloads.

### Key Objectives

- Model an e-commerce domain (users, products, orders, categories) using document-oriented design
- Implement collections with appropriate fields and sample data
- Construct aggregation pipelines for analytics: daily sales, top products, customer statistics
- Create and tune indexes (single, compound, text) for common query patterns
- Use `explain()` to identify slow queries and verify the impact of optimizations

---

## 2. Schema Design

The schema follows **query-first design** — data that is read together is stored together. Four main collections are used. Embedding is applied for bounded, tightly coupled data; referencing is used for shared or unbounded entities.

| Collection | Design Choice | Justification |
|---|---|---|
| `users` | Standalone document | Referenced by many orders; updated independently |
| `categories` | Standalone document | Shared across many products; slow-changing |
| `products` | Attribute Pattern on `attributes{}` | Products differ in attributes per category; flexible schema avoids frequent changes |
| `orders` | `items[]` embedded | Items always read with the order; bounded size; preserves historical price at purchase |

> **Note:** `productId` and `userId` are stored as ObjectId references inside orders, enabling `$lookup` joins while keeping documents lean.

---

## 3. Step 1 – Create Database and Collections

Connect to `mongosh` and switch to the `ecommerce` database, then create all four collections.

```js
use ecommerce;

db.createCollection("users");
db.createCollection("categories");
db.createCollection("products");
db.createCollection("orders");
```

### Screenshot 1
> Run `show collections` — capture all four collections listed under the `ecommerce` database.

![alt text](<assets/Screenshot from 2026-04-24 11-05-14.png>)

![alt text](<assets/Screenshot from 2026-04-24 11-06-45.png>)
---

## 4. Step 2 – Insert Sample Data

### 4.1 Users

Two customers inserted with embedded address subdocuments.

```js
db.users.insertMany([
  {
    name: "Tashi Dorji",
    email: "tashi@example.com",
    phone: "+975-17-123-456",
    address: {
      line1: "Building 12",
      city: "Thimphu",
      country: "Bhutan",
      postalCode: "11001"
    },
    createdAt: new Date("2026-04-18T08:00:00Z")
  },
  {
    name: "Sonam Choden",
    email: "sonam@example.com",
    phone: "+975-17-654-321",
    address: {
      line1: "Flat 3B",
      city: "Phuntsholing",
      country: "Bhutan",
      postalCode: "21001"
    },
    createdAt: new Date("2026-04-19T10:30:00Z")
  }
]);
```

### Screenshot 2
> Run `db.users.find().pretty()` — capture both user documents with embedded address subdocuments.

![alt text](<assets/Screenshot from 2026-04-24 11-07-25.png>)

![alt text](<assets/Screenshot from 2026-04-24 11-07-51.png>)

---

### 4.2 Categories

Electronics (parent) and Accessories (child) inserted. The `parentCategoryId` field on Accessories references `Electronics._id`, demonstrating a self-referencing tree structure.

```js
const electronicsId = new ObjectId();
const accessoriesId = new ObjectId();

db.categories.insertMany([
  { _id: electronicsId, name: "Electronics", slug: "electronics", parentCategoryId: null },
  { _id: accessoriesId, name: "Accessories", slug: "accessories", parentCategoryId: electronicsId }
]);
```

### Screenshot 3
> Run `db.categories.find().pretty()` — capture both documents, noting `parentCategoryId` on Accessories.

![alt text](<assets/Screenshot from 2026-04-24 11-08-44.png>)

![alt text](<assets/Screenshot from 2026-04-24 11-09-05.png>)

---

### 4.3 Products – Attribute Pattern

Three products inserted. Each uses an `attributes{}` sub-document with category-specific fields. This is the **Attribute Pattern** — avoiding a rigid schema for heterogeneous product data.

```js
const headphonesId = new ObjectId();
const cableId      = new ObjectId();
const keyboardId   = new ObjectId();

db.products.insertMany([
  {
    _id: headphonesId,
    name: "Wireless Bluetooth Headphones",
    slug: "wireless-bluetooth-headphones",
    categoryId: electronicsId,
    price: 129.99,
    currency: "USD",
    stock: 200,
    attributes: { brand: "Acme Audio", color: "black", wireless: true, batteryLifeHours: 24 },
    tags: ["audio", "wireless", "headphones"],
    createdAt: new Date("2026-04-18T10:00:00Z")
  },
  {
    _id: cableId,
    name: "USB-C Cable 1m",
    slug: "usb-c-cable-1m",
    categoryId: accessoriesId,
    price: 9.99,
    currency: "USD",
    stock: 500,
    attributes: { brand: "Acme Tech", lengthMeters: 1, color: "white" },
    tags: ["cable", "usb-c"],
    createdAt: new Date("2026-04-18T11:00:00Z")
  },
  {
    _id: keyboardId,
    name: "Mechanical Keyboard",
    slug: "mechanical-keyboard",
    categoryId: electronicsId,
    price: 79.99,
    currency: "USD",
    stock: 150,
    attributes: { brand: "Acme Input", layout: "US", switchType: "blue", backlight: true },
    tags: ["keyboard", "mechanical", "backlit"],
    createdAt: new Date("2026-04-19T09:00:00Z")
  }
]);
```

### Screenshot 4
> Run `db.products.find().pretty()` — capture all three documents. Note each has a **different** `attributes{}` structure (Attribute Pattern).

![alt text](<assets/Screenshot from 2026-04-24 11-09-52.png>)

![alt text](<assets/Screenshot from 2026-04-24 11-10-02.png>)

![alt text](<assets/Screenshot from 2026-04-24 11-10-38.png>)

---

### 4.4 Orders – Embedded Items

Two orders inserted. Each embeds an `items[]` array directly. Key product info (`productName`, `unitPrice`) is duplicated inside the item to preserve historical accuracy independent of future product changes.

```js
const tashi = db.users.findOne({ email: "tashi@example.com" });
const sonam = db.users.findOne({ email: "sonam@example.com" });

db.orders.insertMany([
  {
    userId: tashi._id,
    status: "PAID",
    items: [
      { productId: headphonesId, productName: "Wireless Bluetooth Headphones",
        unitPrice: 129.99, quantity: 2, lineTotal: 259.98 },
      { productId: cableId, productName: "USB-C Cable 1m",
        unitPrice: 9.99, quantity: 1, lineTotal: 9.99 }
    ],
    grandTotal: 269.97,
    currency: "USD",
    createdAt: new Date("2026-04-19T15:30:00Z"),
    paymentMethod: "CARD"
  },
  {
    userId: sonam._id,
    status: "PAID",
    items: [
      { productId: keyboardId, productName: "Mechanical Keyboard",
        unitPrice: 79.99, quantity: 1, lineTotal: 79.99 }
    ],
    grandTotal: 79.99,
    currency: "USD",
    createdAt: new Date("2026-04-20T09:15:00Z"),
    paymentMethod: "COD"
  }
]);
```

### Screenshot 5
> Run `db.orders.find().pretty()` — capture both order documents showing embedded `items[]` arrays.

![alt text](<assets/Screenshot from 2026-04-24 11-11-46.png>)
![alt text](<assets/Screenshot from 2026-04-24 11-11-56.png>)
![alt text](<assets/Screenshot from 2026-04-24 11-12-33.png>)
---

## 5. Step 3 – Aggregation Framework Queries

The aggregation pipeline processes documents through sequential stages. Each stage transforms the stream: `$match` filters, `$group` computes, `$project` reshapes, `$lookup` joins, `$sort` orders, `$limit` reduces.

---

### Query 1 – Daily Sales Totals

**Stages:** `$match` (PAID only) → `$group` by date parts → `$project` date field → `$sort` ascending

```js
db.orders.aggregate([
  { $match: { status: "PAID" } },
  {
    $group: {
      _id: {
        year:  { $year:        "$createdAt" },
        month: { $month:       "$createdAt" },
        day:   { $dayOfMonth:  "$createdAt" }
      },
      totalRevenue: { $sum: "$grandTotal" },
      orderCount:   { $sum: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      date: { $dateFromParts: { year: "$_id.year", month: "$_id.month", day: "$_id.day" } },
      totalRevenue: 1,
      orderCount: 1
    }
  },
  { $sort: { date: 1 } }
]);
```

**Expected output:**

| date | totalRevenue | orderCount |
|---|---|---|
| 2026-04-19 | 269.97 | 1 |
| 2026-04-20 | 79.99 | 1 |

### Screenshot 6
> Capture the full aggregation output — two date groups with `totalRevenue` and `orderCount`.

![alt text](<assets/Screenshot from 2026-04-24 11-14-39.png>)

![alt text](<assets/Screenshot from 2026-04-24 11-14-48.png>)
---

### Query 2 – Top 5 Products by Revenue

**Stages:** `$match` → `$unwind` items[] → `$group` by productId → `$sort` descending → `$limit` 5

```js
db.orders.aggregate([
  { $match: { status: "PAID" } },
  { $unwind: "$items" },
  {
    $group: {
      _id:           "$items.productId",
      productName:   { $first: "$items.productName" },
      totalRevenue:  { $sum:   "$items.lineTotal" },
      totalQuantity: { $sum:   "$items.quantity" }
    }
  },
  { $sort:  { totalRevenue: -1 } },
  { $limit: 5 }
]);
```

**Expected output:** Headphones (259.98) → Keyboard (79.99) → Cable (9.99)

###  Screenshot 7
> Capture three products sorted by `totalRevenue` descending.

![alt text](<assets/Screenshot from 2026-04-24 11-15-19.png>)

---

### Query 3 – Average Order Value per User

Groups orders by `userId` to compute spending statistics, then uses `$lookup` to join the `users` collection and enrich results with the customer name.

```js
db.orders.aggregate([
  { $match: { status: "PAID" } },
  {
    $group: {
      _id:           "$userId",
      totalOrders:   { $sum: 1 },
      totalSpent:    { $sum: "$grandTotal" },
      minOrderValue: { $min: "$grandTotal" },
      maxOrderValue: { $max: "$grandTotal" },
      avgOrderValue: { $avg: "$grandTotal" }
    }
  },
  {
    $lookup: {
      from:         "users",
      localField:   "_id",
      foreignField: "_id",
      as:           "user"
    }
  },
  { $unwind: "$user" },
  {
    $project: {
      _id: 0,
      userId:        "$_id",
      userName:      "$user.name",
      totalOrders:   1,
      totalSpent:    1,
      minOrderValue: 1,
      maxOrderValue: 1,
      avgOrderValue: 1
    }
  },
  { $sort: { totalSpent: -1 } }
]);
```

**Expected output:** Tashi (269.97) then Sonam (79.99), both with `userName` populated via `$lookup`.

### Screenshot 8
> Capture both users with `userName` joined from `users` collection, sorted by `totalSpent`.

![alt text](<assets/Screenshot from 2026-04-24 11-15-56.png>)

![alt text](<assets/Screenshot from 2026-04-24 11-16-04.png>)


---

### Query 4 – Product Catalog with Category Name

`$lookup` joins `products` to `categories` to display `categoryName` alongside product fields.

```js
db.products.aggregate([
  {
    $lookup: {
      from:         "categories",
      localField:   "categoryId",
      foreignField: "_id",
      as:           "category"
    }
  },
  { $unwind: "$category" },
  {
    $project: {
      _id: 0,
      name: 1,
      price: 1,
      "attributes.brand": 1,
      "attributes.color": 1,
      categoryName: "$category.name"
    }
  },
  { $sort: { categoryName: 1, name: 1 } }
]);
```

### Screenshot 9
> Capture all three products with `categoryName` joined, sorted alphabetically by category then name.

![alt text](<assets/Screenshot from 2026-04-24 11-16-24.png>)
![alt text](<assets/Screenshot from 2026-04-24 11-16-28.png>)

---

## 6. Step 4 – Index Creation

Indexes avoid full collection scans. Compound indexes follow the **ESR rule**: Equality fields first, then Sort fields, then Range fields.

| Index Name | Fields | Supports |
|---|---|---|
| `idx_orders_user_createdAt` | `userId:1, createdAt:-1` | Fetch recent orders for a user |
| `idx_orders_status_createdAt_grandTotal` | `status:1, createdAt:-1, grandTotal:1` | Filter by status + date range (ESR) |
| `idx_products_category_price` | `categoryId:1, price:1` | List products in a category by price |
| `idx_products_text` | `name:text (w=10), tags:text (w=5)` | Full-text search with relevance scoring |

```js
// Index 1 – Orders by user and date
db.orders.createIndex(
  { userId: 1, createdAt: -1 },
  { name: "idx_orders_user_createdAt" }
);

// Index 2 – Orders by status and date (ESR pattern)
db.orders.createIndex(
  { status: 1, createdAt: -1, grandTotal: 1 },
  { name: "idx_orders_status_createdAt_grandTotal" }
);

// Index 3 – Products by category and price
db.products.createIndex(
  { categoryId: 1, price: 1 },
  { name: "idx_products_category_price" }
);

// Index 4 – Text index for product search
db.products.createIndex(
  { name: "text", tags: "text" },
  { name: "idx_products_text", weights: { name: 10, tags: 5 } }
);
```

Verify indexes after creation:

```js
db.orders.getIndexes();
db.products.getIndexes();
```

### Screenshot 10
> Capture the output of both `getIndexes()` calls — all four custom indexes listed with their names and key patterns.

![alt text](<assets/Screenshot from 2026-04-24 13-17-47.png>)
![alt text](<assets/Screenshot from 2026-04-24 13-18-14.png>)
![alt text](<assets/Screenshot from 2026-04-24 13-18-29.png>)
![alt text](<assets/Screenshot from 2026-04-24 13-18-46.png>)
![alt text](<assets/Screenshot from 2026-04-24 13-19-10.png>)

### Text Index Search Test

```js
db.products
  .find(
    { $text: { $search: "wireless keyboard" } },
    { score: { $meta: "textScore" }, name: 1, price: 1 }
  )
  .sort({ score: { $meta: "textScore" } });
```

### Screenshot 11
> Capture products ranked by text relevance score — higher score = better match.

![alt text](<assets/Screenshot from 2026-04-24 13-19-54.png>)

---

## 7. Step 5 – Query Optimization with explain()

`explain("executionStats")` reveals whether MongoDB uses a **COLLSCAN** (slow — scans all documents) or **IXSCAN** (fast — uses an index). Key metrics to compare:

- `winningPlan.stage` — COLLSCAN vs IXSCAN
- `executionStats.totalDocsExamined` — documents scanned
- `executionStats.totalKeysExamined` — index keys read
- `executionStats.executionTimeMillis` — query duration

---

### 7.1 Before Index – COLLSCAN

Drop the status+date index to simulate no suitable index existing.

```js
db.orders.dropIndex("idx_orders_status_createdAt_grandTotal");

db.orders.find(
  { status: "PAID", createdAt: { $gte: new Date("2026-04-19") } }
).sort({ createdAt: -1 }).explain("executionStats");
```

With no suitable index, MongoDB performs a **COLLSCAN** — every document in the collection is scanned regardless of whether it matches the filter.

### Screenshot 12
> Capture `explain()` output **BEFORE** index — look for `winningPlan.stage: "COLLSCAN"` and note `totalDocsExamined`.
![alt text](<assets/Screenshot from 2026-04-24 13-20-10.png>)

---

### 7.2 After Index – IXSCAN

```js
db.orders.createIndex(
  { status: 1, createdAt: -1 },
  { name: "idx_orders_status_createdAt" }
);

db.orders.find(
  { status: "PAID", createdAt: { $gte: new Date("2026-04-19") } }
).sort({ createdAt: -1 }).explain("executionStats");
```

With the index in place, MongoDB uses **IXSCAN** — only matching index entries are scanned. `totalDocsExamined` drops and `executionTimeMillis` improves.

### Screenshot 13
> Capture `explain()` output **AFTER** index — look for `IXSCAN` in `winningPlan` and reduced `totalDocsExamined`.

![alt text](<assets/Screenshot from 2026-04-24 13-20-56.png>)
![alt text](<assets/Screenshot from 2026-04-24 13-21-02.png>)
![alt text](<assets/Screenshot from 2026-04-24 13-21-11.png>)
![alt text](<assets/Screenshot from 2026-04-24 13-21-51.png>)
![alt text](<assets/Screenshot from 2026-04-24 13-21-56.png>)
---

### 7.3 Before vs After Comparison

| Metric | BEFORE (COLLSCAN) | AFTER (IXSCAN) |
|---|---|---|
| `winningPlan.stage` | `COLLSCAN` | `IXSCAN` |
| `totalDocsExamined` | All documents | Only matching docs |
| `totalKeysExamined` | 0 | Equal to matches |
| `executionTimeMillis` | Higher | Lower / near 0 |
| Index used | None | `idx_orders_status_createdAt` |

---

### 7.4 Index-Friendly Aggregation Pipeline

When `$match` and `$sort` appear at the start of a pipeline and their fields match an existing index, MongoDB pushes them down to the query engine for an index-assisted scan.

```js
db.orders.aggregate([
  {
    $match: {
      status: "PAID",
      createdAt: { $gte: new Date("2026-04-19") }
    }
  },
  { $sort: { createdAt: -1 } },
  {
    $project: {
      _id: 0,
      userId: 1,
      createdAt: 1,
      grandTotal: 1,
      itemCount: { $size: "$items" }
    }
  },
  { $limit: 20 }
]);
```

### Screenshot 14
> Capture aggregation output — filtered orders with the computed `itemCount` field.

![alt text](<assets/Screenshot from 2026-04-24 13-22-10.png>)

---

### 7.5 Attribute Pattern Query

```js
db.products.find({
  "attributes.brand": "Acme Audio",
  "attributes.color": "black"
});
```

### Screenshot 15
> Capture the result — only the Wireless Bluetooth Headphones document returned.

![alt text](<assets/Screenshot from 2026-04-24 13-22-26.png>)

---

## 8. Discussion

### Embedding vs Referencing Decisions

- **Order items are embedded** because they are always read with the order, have bounded size, and must capture price at time of purchase (historical accuracy). If the product price changes later, the order remains accurate.
- **Users and products are referenced** from orders because they are shared entities updated independently — duplicating them in every order would cause consistency problems at scale.
- **Category hierarchy uses self-referencing** (`parentCategoryId`) to represent parent-child relationships without deeply nested documents.

### ESR Index Rule

The compound index `{ status:1, createdAt:-1, grandTotal:1 }` follows ESR:

- **Equality** (`status`) first — narrows the dataset immediately
- **Sort** (`createdAt`) second — avoids an expensive in-memory sort
- **Range** (`grandTotal`) last — optional further filtering

Incorrect field ordering reduces or eliminates index benefit even if all fields are present.

### Common Anti-Patterns Avoided

- **Unbounded arrays** — `items[]` is bounded per order; new orders create new documents rather than pushing into a single document
- **Over-indexing** — only four indexes created, targeting high-frequency query patterns only
- **COLLSCAN on large collections** — verified with `explain()` and resolved with compound indexes
- **Relational modeling** — `$lookup` used sparingly; embedding handles tight coupling without excessive joins

---

## 9. Conclusion

This practical demonstrated MongoDB schema design for an e-commerce platform using query-first principles. Four collections were implemented with deliberate choices between embedding and referencing. Four aggregation pipelines covered daily sales, product revenue, customer statistics, and catalog enrichment. Four indexes were designed and validated using `explain()`, confirming the transition from COLLSCAN to IXSCAN and reduced document examination. The practical reinforces that schema design, indexing strategy, and aggregation pipeline construction must be treated as an integrated system rather than independent concerns.

---

