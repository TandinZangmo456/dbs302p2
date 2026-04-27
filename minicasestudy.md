# DBS302 – Database Systems
## Mini Case Study Report: Product Attribute Pattern in MongoDB — Hands-On Laboratory Exercises

## Table of Contents

1. [Introduction](#1-introduction)
2. [Task 1 — Reviews Collection with Average Rating Aggregation](#2-task-1--reviews-collection-with-average-rating-aggregation)
3. [Task 2 — Inventory-Focused Low Stock Query](#3-task-2--inventory-focused-low-stock-query)
4. [Task 3 — Customer Segmentation by Spending Tier](#4-task-3--customer-segmentation-by-spending-tier)
5. [Task 4 — Custom Index Experiments with explain()](#5-task-4--custom-index-experiments-with-explain)
6. [Task 5 — Text Search Enhancement with Description Field](#6-task-5--text-search-enhancement-with-description-field)
7. [Discussion and Observations](#7-discussion-and-observations)
8. [Conclusion](#8-conclusion)

## 1. Introduction

### 1.1 Background

Modern e-commerce platforms manage catalogs of heterogeneous products - clothing, electronics, furniture, and accessories — each carrying a distinct set of attributes. A relational database would require either a wide table with many nullable columns or an entity-attribute-value (EAV) structure, both of which introduce complexity and performance penalties. MongoDB's flexible document model addresses this naturally through the **Attribute Pattern**, in which product-specific key-value pairs are stored within an embedded `attributes` sub-document. This approach allows diverse product types to coexist within a single collection without requiring schema migrations.

### 1.2 Scope of This Case Study

Building upon the e-commerce schema established in Practical 3, this case study extends the database with five hands-on laboratory exercises that deepen practical understanding of:

- Document design and aggregation for a new `reviews` collection.
- Inventory-focused analytical queries using the aggregation framework.
- Customer segmentation using `$switch`, `$merge`, and update pipelines.
- Empirical comparison of query performance across no-index, single-field, and compound index configurations using `explain("executionStats")`.
- Text index extension to include a product `description` field with weighted relevance scoring.

### 1.3 Database Context

All tasks were performed on the `ecommerce` database containing the following collections established in Practical 3:

| Collection   | Documents | Purpose                          |
|--------------|-----------|----------------------------------|
| `users`      | 2         | Customer accounts                |
| `categories` | 2         | Product taxonomy (hierarchical)  |
| `products`   | 6         | Product catalog (3 original + 3 low-stock added in Task 2) |
| `orders`     | 2         | Customer purchase records        |
| `reviews`    | 4         | Added in Task 1                  |

---

## 2. Task 1 — Reviews Collection with Average Rating Aggregation

### 2.1 Objective

To design and populate a `reviews` collection containing embedded ratings and comments, and to construct an aggregation pipeline that computes the average rating and total review count per product.

### 2.2 Schema Design for Reviews

The `reviews` collection was designed as a separate collection rather than embedding reviews inside product documents. This decision was made because the number of reviews per product is **unbounded** — a product may accumulate thousands of reviews over time. Embedding an unbounded array inside a document risks exceeding MongoDB's 16MB BSON document size limit and degrades write performance. Referencing via `productId` is therefore the appropriate pattern here.

Each review document stores the following fields:

```json
{
  "_id": ObjectId("..."),
  "productId": ObjectId("..."),
  "userId": ObjectId("..."),
  "rating": 5,
  "comment": "Excellent sound quality and very comfortable!",
  "createdAt": ISODate("2026-04-20T09:00:00Z")
}
```

### 2.3 Data Insertion

Four review documents were inserted across three products:

```javascript
db.reviews.insertMany([
  {
    productId: db.products.findOne({ name: "Wireless Bluetooth Headphones" })._id,
    userId: db.users.findOne({ name: "Tashi Dorji" })._id,
    rating: 5,
    comment: "Excellent sound quality and very comfortable!",
    createdAt: new Date("2026-04-20T09:00:00Z")
  },
  {
    productId: db.products.findOne({ name: "Wireless Bluetooth Headphones" })._id,
    userId: db.users.findOne({ name: "Sonam Choden" })._id,
    rating: 4,
    comment: "Great headphones, battery life is impressive.",
    createdAt: new Date("2026-04-21T11:00:00Z")
  },
  {
    productId: db.products.findOne({ name: "Mechanical Keyboard" })._id,
    userId: db.users.findOne({ name: "Tashi Dorji" })._id,
    rating: 3,
    comment: "Good keyboard but the keys are a bit loud.",
    createdAt: new Date("2026-04-21T14:00:00Z")
  },
  {
    productId: db.products.findOne({ name: "USB-C Cable 1m" })._id,
    userId: db.users.findOne({ name: "Sonam Choden" })._id,
    rating: 5,
    comment: "Perfect cable, fast charging and durable.",
    createdAt: new Date("2026-04-22T08:00:00Z")
  }
])
```

### 2.4 Aggregation Pipeline — Average Rating per Product

```javascript
db.reviews.aggregate([
  {
    $group: {
      _id: "$productId",
      averageRating: { $avg: "$rating" },
      totalReviews: { $sum: 1 }
    }
  },
  {
    $lookup: {
      from: "products",
      localField: "_id",
      foreignField: "_id",
      as: "product"
    }
  },
  { $unwind: "$product" },
  {
    $project: {
      _id: 0,
      productName: "$product.name",
      averageRating: { $round: ["$averageRating", 2] },
      totalReviews: 1
    }
  },
  { $sort: { averageRating: -1 } }
])
```

**Pipeline Stage Explanation:**

| Stage      | Purpose                                                              |
|------------|----------------------------------------------------------------------|
| `$group`   | Groups reviews by `productId`, computing average rating and count   |
| `$lookup`  | Joins the `products` collection to retrieve product names           |
| `$unwind`  | Deconstructs the joined array into a single document per product    |
| `$project` | Shapes the output, rounding `averageRating` to 2 decimal places     |
| `$sort`    | Orders results by highest average rating descending                  |

### 2.5 Output and Screenshot

**Screenshot — Task 1: Reviews Aggregation Output:**

![Task 1 Reviews Aggregation](screenshots/t1_reviews_aggregation.png)

### 2.6 Results

| Product Name                   | Average Rating | Total Reviews |
|--------------------------------|----------------|---------------|
| USB-C Cable 1m                 | 5.0            | 1             |
| Wireless Bluetooth Headphones  | 4.5            | 2             |
| Mechanical Keyboard            | 3.0            | 1             |

### 2.7 Analysis

The aggregation pipeline correctly computed average ratings per product using `$avg` within the `$group` stage. The `$lookup` join against the `products` collection enriched the output with human-readable product names in place of raw ObjectIds. USB-C Cable 1m achieved the highest average rating of 5.0, while Wireless Bluetooth Headphones received 4.5 — consistent with the two individual ratings of 5 and 4 inserted. The `$round` operator in `$project` ensured clean numeric output.

---

## 3. Task 2 — Inventory-Focused Low Stock Query

### 3.1 Objective

To identify products with critically low stock levels (stock < 10) and present them sorted by stock quantity in ascending order, enriched with their category names.

### 3.2 Low Stock Product Insertion

Three new products with low stock levels were added to the catalog to provide meaningful test data:

```javascript
db.products.insertMany([
  {
    name: "Portable Charger",
    categoryId: db.categories.findOne({ name: "Electronics" })._id,
    price: 24.99, currency: "USD", stock: 3,
    attributes: { brand: "Acme Power", capacity: "10000mAh", color: "white" },
    tags: ["charger", "portable", "power-bank"],
    createdAt: new Date("2026-04-20T10:00:00Z")
  },
  {
    name: "Screen Cleaning Kit",
    categoryId: db.categories.findOne({ name: "Accessories" })._id,
    price: 4.99, currency: "USD", stock: 7,
    attributes: { brand: "Acme Clean", pieces: 2, color: "blue" },
    tags: ["cleaning", "screen", "accessories"],
    createdAt: new Date("2026-04-20T11:00:00Z")
  },
  {
    name: "Laptop Stand",
    categoryId: db.categories.findOne({ name: "Accessories" })._id,
    price: 34.99, currency: "USD", stock: 1,
    attributes: { brand: "Acme Desk", material: "aluminium", color: "silver" },
    tags: ["stand", "laptop", "desk"],
    createdAt: new Date("2026-04-20T12:00:00Z")
  }
])
```

**Screenshot — Task 2: Low Stock Products Inserted:**

![Task 2 Low Stock Insert](screenshots/t2_lowstock_insert.png)

### 3.3 Aggregation Pipeline — Low Stock Query

```javascript
db.products.aggregate([
  { $match: { stock: { $lt: 10 } } },
  {
    $lookup: {
      from: "categories",
      localField: "categoryId",
      foreignField: "_id",
      as: "category"
    }
  },
  { $unwind: "$category" },
  {
    $project: {
      _id: 0,
      productName: "$name",
      stock: 1,
      price: 1,
      categoryName: "$category.name"
    }
  },
  { $sort: { stock: 1 } }
])
```

### 3.4 Output and Screenshot

**Screenshot — Task 2: Low Stock Query Output:**

![Task 2 Low Stock Query](screenshots/t2_lowstock_query.png)

### 3.5 Results

| Product Name       | Stock | Price   | Category    |
|--------------------|-------|---------|-------------|
| Laptop Stand       | 1     | $34.99  | Accessories |
| Portable Charger   | 3     | $24.99  | Electronics |
| Screen Cleaning Kit| 7     | $4.99   | Accessories |

### 3.6 Analysis

The `$match` stage with `{ stock: { $lt: 10 } }` correctly filtered out the three original products (Headphones: 200, Cable: 500, Keyboard: 150) and returned only the three newly inserted low-stock items. The `$lookup` enriched each result with its category name, and the final `$sort` on `stock: 1` presented the results from lowest to highest stock level — Laptop Stand at 1 unit being the most urgent restocking priority.

---

## 4. Task 3 — Customer Segmentation by Spending Tier

### 4.1 Objective

To classify customers into spending tiers (Bronze, Silver, Gold) based on their cumulative order spend, and to persist the tier classification back into the `users` collection using `$merge`.

### 4.2 Tier Classification Rules

| Tier   | Condition              |
|--------|------------------------|
| Gold   | Total spent ≥ USD 200  |
| Silver | Total spent ≥ USD 50   |
| Bronze | Total spent < USD 50   |

### 4.3 Aggregation and Merge Pipeline

```javascript
db.orders.aggregate([
  { $match: { status: "PAID" } },
  {
    $group: {
      _id: "$userId",
      totalSpent: { $sum: "$grandTotal" }
    }
  },
  {
    $addFields: {
      tier: {
        $switch: {
          branches: [
            { case: { $gte: ["$totalSpent", 200] }, then: "Gold"   },
            { case: { $gte: ["$totalSpent", 50]  }, then: "Silver" }
          ],
          default: "Bronze"
        }
      }
    }
  },
  {
    $merge: {
      into: "users",
      on: "_id",
      whenMatched: [
        {
          $addFields: {
            totalSpent: "$$new.totalSpent",
            tier: "$$new.tier"
          }
        }
      ],
      whenNotMatched: "discard"
    }
  }
])
```

**Pipeline Stage Explanation:**

| Stage        | Purpose                                                               |
|--------------|-----------------------------------------------------------------------|
| `$match`     | Restricts the pipeline to PAID orders only                           |
| `$group`     | Computes cumulative spend per customer (`userId`)                    |
| `$addFields` | Applies `$switch` logic to assign a tier based on `totalSpent`       |
| `$merge`     | Writes `totalSpent` and `tier` back into the matching `users` document |

### 4.4 Verification Query

```javascript
db.users.find({}, { name: 1, totalSpent: 1, tier: 1, _id: 0 })
```

### 4.5 Output and Screenshots

**Screenshot — Task 3: Segmentation Pipeline Execution:**

![Task 3 Segmentation Pipeline](screenshots/t3_segmentation_pipeline.png)

**Screenshot — Task 3: Tier Fields Written to Users Collection:**

![Task 3 Segmentation Verified](screenshots/t3_segmentation_verify.png)

### 4.6 Results

| Customer Name  | Total Spent | Tier   |
|----------------|-------------|--------|
| Tashi Dorji    | USD 269.97  | Gold   |
| Sonam Choden   | USD 79.99   | Silver |

### 4.7 Analysis

The `$switch` expression correctly evaluated each customer's `totalSpent` against the tier thresholds. Tashi Dorji's spend of USD 269.97 exceeded the Gold threshold of USD 200, while Sonam Choden's spend of USD 79.99 fell within the Silver range (≥ USD 50). The `$merge` stage successfully wrote the computed `tier` and `totalSpent` fields back into the respective user documents in the `users` collection, demonstrating MongoDB's capability to use aggregation pipelines as update mechanisms — avoiding a separate application-layer read-compute-write cycle.

---

## 5. Task 4 — Custom Index Experiments with explain()

### 5.1 Objective

To empirically compare query performance across three index configurations — no index, single-field index, and compound index — using `explain("executionStats")` on the same query, and to interpret the impact on `totalDocsExamined`, `totalKeysExamined`, and `executionTimeMillis`.

### 5.2 Test Query

The following query was used consistently across all three experiments:

```javascript
db.products.find({
  "attributes.brand": "Acme Audio",
  "attributes.color": "black"
}).explain("executionStats")
```

---

### 5.3 Experiment A — No Index (COLLSCAN)

All non-default indexes were dropped before this experiment:

```javascript
db.products.dropIndexes()
```

**Screenshot — Task 4A: No Index (COLLSCAN):**

![Task 4 No Index](screenshots/t4_no_index.png)

**Screenshot — Task 4A: No Index Execution Stats:**

![Task 4 No Index Stats](screenshots/t4_no_index_stats.png)

**Results:**

| Metric                | Value      |
|-----------------------|------------|
| `winningPlan.stage`   | COLLSCAN   |
| `totalDocsExamined`   | 6          |
| `totalKeysExamined`   | 0          |
| `executionTimeMillis` | 0          |
| `nReturned`           | 1          |

---

### 5.4 Experiment B — Single-Field Index on `attributes.brand`

```javascript
db.products.createIndex(
  { "attributes.brand": 1 },
  { name: "idx_single_brand" }
)
```

**Screenshot — Task 4B: Single-Field Index (IXSCAN):**

![Task 4 Single Index](screenshots/t4_single_index.png)

**Screenshot — Task 4B: Single-Field Index Execution Stats:**

![Task 4 Single Index Stats](screenshots/t4_single_index_stats.png)

**Results:**

| Metric                | Value      |
|-----------------------|------------|
| `winningPlan.stage`   | IXSCAN     |
| `indexName`           | idx_single_brand |
| `totalDocsExamined`   | 1          |
| `totalKeysExamined`   | 1          |
| `executionTimeMillis` | 1          |
| `nReturned`           | 1          |

**Observation:** The single-field index on `attributes.brand` reduced `totalDocsExamined` from 6 to 1. However, the `color` filter was applied as a post-index FETCH stage filter — the index could only narrow the candidate set by brand, and color was checked in-document.

---

### 5.5 Experiment C — Compound Index on `attributes.brand` + `attributes.color`

```javascript
db.products.dropIndex("idx_single_brand")

db.products.createIndex(
  { "attributes.brand": 1, "attributes.color": 1 },
  { name: "idx_compound_brand_color" }
)
```

**Screenshot — Task 4C: Compound Index (IXSCAN with tight bounds):**

![Task 4 Compound Index](screenshots/t4_compound_index.png)

**Screenshot — Task 4C: Compound Index Execution Stats:**

![Task 4 Compound Stats](screenshots/t4_compound_stats.png)

**Screenshot — Task 4C: Compound Index Bounds Detail:**

![Task 4 Compound Stats 2](screenshots/t4_compound_stats2.png)

**Results:**

| Metric                | Value                    |
|-----------------------|--------------------------|
| `winningPlan.stage`   | IXSCAN                   |
| `indexName`           | idx_compound_brand_color |
| `totalDocsExamined`   | 1                        |
| `totalKeysExamined`   | 1                        |
| `executionTimeMillis` | 1                        |
| `nReturned`           | 1                        |
| `indexBounds.brand`   | `["Acme Audio","Acme Audio"]` |
| `indexBounds.color`   | `["black","black"]`      |

---

### 5.6 Comparative Summary

| Metric                | No Index   | Single-Field Index | Compound Index |
|-----------------------|------------|--------------------|----------------|
| Plan Stage            | COLLSCAN   | IXSCAN             | IXSCAN         |
| `totalDocsExamined`   | 6          | 1                  | 1              |
| `totalKeysExamined`   | 0          | 1                  | 1              |
| `executionTimeMillis` | 0          | 1                  | 1              |
| Color filter applied  | In-memory  | Post-fetch         | Within index ✅|

### 5.7 Analysis

The COLLSCAN with no index examined all 6 documents in the collection regardless of their attribute values — a pattern that degrades linearly with collection growth. Both indexed experiments reduced `totalDocsExamined` to 1, representing a 6× improvement in document examination efficiency for this dataset.

The key distinction between the single-field and compound indexes lies in how the `color` filter was handled. With the single-field index, MongoDB used the index to narrow candidates by `brand`, then applied the `color` filter during the FETCH stage as an in-document check. With the compound index, both `brand` and `color` constraints were encoded directly in the index bounds — `["Acme Audio","Acme Audio"]` and `["black","black"]` respectively — meaning the result was resolved entirely within the index without requiring a separate document-level filter. This is the principal advantage of a compound index over a single-field index when multiple equality conditions are present.

---

## 6. Task 5 — Text Search Enhancement with Description Field

### 6.1 Objective

To extend the products text index to include a `description` field, configure field weights to reflect relative importance, and demonstrate relevance-ranked full-text search across multiple search terms.

### 6.2 Adding Description Fields to Products

A `description` field was added to all six products using individual `updateOne` operations:

```javascript
db.products.updateOne(
  { name: "Wireless Bluetooth Headphones" },
  { $set: { description: "Premium wireless headphones with noise cancellation and long battery life." } }
)
db.products.updateOne(
  { name: "USB-C Cable 1m" },
  { $set: { description: "Durable braided USB-C cable supporting fast charging and data transfer." } }
)
db.products.updateOne(
  { name: "Mechanical Keyboard" },
  { $set: { description: "Tactile mechanical keyboard with RGB backlight and blue switches for typists." } }
)
db.products.updateOne(
  { name: "Portable Charger" },
  { $set: { description: "Compact 10000mAh power bank with dual USB ports for travel." } }
)
db.products.updateOne(
  { name: "Screen Cleaning Kit" },
  { $set: { description: "Microfiber cleaning kit safe for all screens including monitors and phones." } }
)
db.products.updateOne(
  { name: "Laptop Stand" },
  { $set: { description: "Adjustable aluminium laptop stand for ergonomic desk setup." } }
)
```

**Screenshot — Task 5: Description Fields Added:**

![Task 5 Descriptions](screenshots/t5_descriptions.png)

### 6.3 Extended Text Index Creation

The original text index had been dropped during Task 4's `dropIndexes()` call. The extended index was created directly with three fields and configured weights:

```javascript
db.products.createIndex(
  { name: "text", tags: "text", description: "text" },
  {
    name: "idx_products_text_extended",
    weights: { name: 10, tags: 5, description: 3 }
  }
)
```

**Weight Configuration Rationale:**

| Field         | Weight | Reasoning                                                    |
|---------------|--------|--------------------------------------------------------------|
| `name`        | 10     | An exact or partial name match is the strongest signal       |
| `tags`        | 5      | Tag matches indicate category-level relevance                |
| `description` | 3      | Description matches indicate contextual relevance only       |

### 6.4 Text Search Queries and Results

Three search queries were executed to test the extended index:

```javascript
// Search 1: "wireless"
db.products.find(
  { $text: { $search: "wireless" } },
  { score: { $meta: "textScore" }, name: 1, description: 1, _id: 0 }
).sort({ score: { $meta: "textScore" } })

// Search 2: "charging"
db.products.find(
  { $text: { $search: "charging" } },
  { score: { $meta: "textScore" }, name: 1, description: 1, _id: 0 }
).sort({ score: { $meta: "textScore" } })

// Search 3: "ergonomic aluminium"
db.products.find(
  { $text: { $search: "ergonomic aluminium" } },
  { score: { $meta: "textScore" }, name: 1, description: 1, _id: 0 }
).sort({ score: { $meta: "textScore" } })
```

**Screenshot — Task 5: All Three Text Search Results:**

![Task 5 Text Searches](screenshots/t5_text_searches.png)

### 6.5 Results

| Search Term           | Top Result                     | Score  | Match Location          |
|-----------------------|--------------------------------|--------|-------------------------|
| "wireless"            | Wireless Bluetooth Headphones  | 13.85  | `name` + `description`  |
| "charging"            | USB-C Cable 1m                 | 1.65   | `description` only      |
| "ergonomic aluminium" | Laptop Stand                   | 3.43   | `description` only (2 terms) |

### 6.6 Analysis

The relevance scoring behaviour confirmed the weight configuration was functioning correctly. The search term "wireless" matched both the `name` field ("Wireless Bluetooth Headphones") and the `description` field, yielding a high composite score of 13.85 — reflecting the combined weight contribution of 10 (name) and 3 (description). The term "charging" appeared only in the USB-C Cable description, producing a lower score of 1.65 consistent with the description weight of 3. The multi-term search "ergonomic aluminium" matched both words within the Laptop Stand description, producing a moderate score of 3.43.

This demonstrates that the weighted text index enables relevance-ranked search results that prioritise products whose names match search terms over those that match only in descriptions — a behaviour desirable in production e-commerce search experiences.

---

## 7. Discussion and Observations

The following cross-cutting observations were drawn from the five tasks completed in this case study:

**On Collection Design for Reviews:** The decision to store reviews in a separate collection rather than embedding them within product documents is justified by the unbounded growth pattern of user-generated content. A product may receive thousands of reviews over its lifetime, making embedding impractical and a potential source of document size violations. Referencing via `productId` maintains a clean separation of concerns while enabling efficient per-product aggregation via `$group` and `$lookup`.

**On `$merge` as an Update Mechanism:** Task 3 demonstrated that `$merge` can function as a powerful alternative to application-layer read-compute-write cycles. By computing tier classifications within the aggregation pipeline and writing results directly back to the `users` collection, the operation required a single database round-trip rather than iterating over documents in application code. This pattern is particularly valuable for batch enrichment workflows such as nightly tier recalculations.

**On Index Selectivity and Compound Indexes:** Task 4 confirmed that while a single-field index can reduce document examination significantly, a compound index provides superior query coverage when multiple equality conditions are present. The compound index encoded both `brand` and `color` constraints directly in its index bounds, eliminating the post-fetch filtering step that the single-field index required. In production workloads where the `attributes` fields are frequently queried together, the compound index is the correct choice.

**On Text Index Weight Design:** Task 5 illustrated that text index weights directly control the relative influence of each indexed field on the final relevance score. A match in the `name` field (weight 10) produced a score approximately 8× higher than a match in the `description` field alone (weight 3), creating a natural ranking hierarchy that mirrors how users perceive search result relevance. The multi-term search for "ergonomic aluminium" demonstrated that MongoDB's text search tokenises and scores each term independently, accumulating scores across matching fields.

**On the Attribute Pattern:** Throughout all five tasks, the `attributes` sub-document proved effective at accommodating heterogeneous product data — headphones carrying `batteryLifeHours`, cables carrying `lengthMeters`, and stands carrying `material` — all within a single `products` collection. The ability to create indexes directly on nested attribute fields (e.g., `"attributes.brand"`) without any schema restructuring demonstrates the practical value of the Attribute Pattern in a MongoDB e-commerce context.

---

## 8. Conclusion

This mini case study extended the e-commerce database schema established in Practical 3 through five targeted laboratory exercises, each addressing a distinct aspect of MongoDB application development.

**Task 1** demonstrated the design of a reviews collection that correctly separates unbounded user-generated content from the product document, and constructed an aggregation pipeline that computed average ratings per product using `$group`, `$lookup`, and `$round`.

**Task 2** implemented an inventory monitoring query that identified low-stock products using `$match` with a range condition, enriched results with category names via `$lookup`, and presented results ordered by ascending stock level.

**Task 3** applied customer segmentation logic using `$switch` within an aggregation pipeline and demonstrated the `$merge` stage as an efficient mechanism for writing computed classification fields back into the `users` collection in a single database operation.

**Task 4** provided empirical evidence of the performance impact of indexing. The COLLSCAN with no index examined all 6 documents; both indexed configurations reduced examination to 1 document. The compound index further improved query precision by encoding both filter conditions within the index bounds, eliminating post-fetch filtering.

**Task 5** demonstrated that MongoDB's text index can be extended to cover additional fields with configurable weights, enabling relevance-ranked product search that prioritises name matches over description matches — a behaviour that reflects real-world search expectations.

Collectively, these exercises reinforce the principle that MongoDB performance and correctness in analytical workloads are achieved through the deliberate combination of query-driven schema design, targeted indexing, and pipeline-based data transformation.
