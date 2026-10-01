# DynamoDB — Key Concepts & Notes

Conceptual reference: what DynamoDB is, how it's structured internally, and the schema-design reasoning behind a real single-table design. Commands live in a separate file — this one is theory and decisions only.

---

## Relational vs nonrelational databases

| | Relational (SQL) | Nonrelational (NoSQL) |
|---|---|---|
| Schema | Fixed, defined upfront | Flexible / schemaless |
| Scaling | Mostly vertical | Horizontal (built-in) |
| Relationships | Joins across tables | Denormalized, data duplicated by design |
| Best for | Complex queries, reporting, transactions across many entities | High-throughput simple-key access, scale, flexible item shape |

AWS offers both: RDS (relational engines) for the relational side, and DynamoDB, ElastiCache, Neptune, etc. for nonrelational needs. Choosing between them is a deliberate trade-off, not a default — a transactional, high-frequency, simple-key workload fits DynamoDB; an analytical, join-heavy, ad-hoc-query workload fits a relational engine instead.

---

## What DynamoDB is

A fully managed, serverless NoSQL key-value and document database. No servers to provision or patch, scales horizontally and automatically, and bills either on-demand (per request) or on provisioned, pre-allocated throughput.

---

## Basic components

- **Table** — a collection of items (roughly: a "table" in SQL terms, but schemaless).
- **Item** — a single record (roughly: a "row"). Each item is a set of attributes.
- **Attribute** — a single data field on an item (roughly: a "column"), but attributes are not fixed across items — one item can have attributes another item in the same table doesn't have.

---

## Primary keys

Two forms:

- **Simple primary key** — a single **partition key** (hash key) uniquely identifies each item.
- **Composite primary key** — a **partition key** + a **sort key**. The partition key alone can repeat across items; the combination of partition key + sort key must be unique. This allows multiple related items to share the same partition key and be ordered/filtered by the sort key.

### Partitioning

DynamoDB distributes data across physical partitions based on the **partition key's hash value**. Each distinct partition key value lives in its own partition. This is what allows horizontal scaling — DynamoDB spreads partitions (and therefore traffic) across many physical nodes, each handling a slice of the key space.

**Hot partition**: a single partition key receiving disproportionately more traffic than others can become a throughput bottleneck, even if the table overall is well within its capacity. Partition key design should aim for an even spread of request volume across key values. (Common DVA-C02 exam topic.)

---

## Item & attribute types, size limits

- Scalar types: String, Number, Binary, Boolean, Null.
- Set types: String Set, Number Set, Binary Set.
- Document types: List, Map (nested JSON-like structures).
- **Maximum item size: 400 KB** (including attribute names and values).

---

## Secondary indexes

Secondary indexes let you query the table by attributes other than the primary key.

### Global Secondary Index (GSI)

- Has its own partition key and (optionally) sort key, **different from the table's**.
- Spans **all partitions** of the base table — it regroups items that may live in completely different partitions of the main table into new partitions based on the GSI's key.
- Can be added or removed after the table is created.
- Up to **20 GSIs per table**.
- Good GSI partition key candidates have **few distinct, frequently repeated values** (e.g. a `status` field with values like `LEGIT`/`FRAUD`/`PENDING`) — the opposite of what makes a good *table* partition key, which benefits from high cardinality.

### Local Secondary Index (LSI)

- Shares the **same partition key** as the table, but a different sort key.
- Scope stays within a single partition (a single partition key value) — cannot span across partition key values.
- **Must be defined at table creation** — cannot be added later.
- Up to **5 LSIs per table**, each capped at **10 GB per partition key value**.

---

## Read consistency

- **Eventually consistent read** (default) — may not reflect the very latest write immediately after it happens, but costs less (lower RCU). Fine for most read patterns.
- **Strongly consistent read** — always reflects the latest successful write, at a higher RCU cost. Required when the application cannot tolerate reading stale data (e.g. immediately re-reading data just written in the same flow).

---

## Transactions

DynamoDB supports ACID transactions across one or more items, within or across tables:

- `TransactWriteItems` — bundles Put/Update/Delete operations as a single all-or-nothing unit, optionally gated by condition checks.
- `TransactGetItems` — bundles multiple GetItem reads as a single consistent read set.

If any single operation or condition inside the transaction fails, the **entire transaction is rejected** — nothing is partially applied.

---

## Throughput modes

### Provisioned throughput (RCU / WCU)

You pre-allocate Read Capacity Units (RCU) and Write Capacity Units (WCU):

- **1 RCU** = one strongly consistent read of up to 4 KB per second (or two eventually consistent reads of up to 4 KB per second).
- **1 WCU** = one write of up to 1 KB per second.

Calculation examples:
- Reading an item up to **4 KB**, strongly consistent, **30 reads/second** → **30 RCU**.
- Writing an item up to **1 KB**, **14 writes/second** → **14 WCU**.
- Larger items round up: an item bigger than the per-unit size consumes multiple units proportionally (e.g. an 8 KB strongly-consistent read needs 2 RCU per read).

### On-demand throughput

No capacity planning — DynamoDB scales to the workload automatically and bills per request. Simplest option for unpredictable or spiky traffic, or when you don't want to manage capacity at all; provisioned mode can be cheaper at steady, predictable, high volume.

---

## DynamoDB Streams

An optional, time-ordered log of item-level changes (create/update/delete) in a table.

- Changes are grouped into **shards**.
- Each change produces a **stream record**, which can include the item's "before" image, "after" image, or both depending on the stream's view type.
- Records are retained for **24 hours**.
- Common uses: triggering a Lambda on every write (event-driven processing), replication (Global Tables are built on Streams), audit logging.

---

## Global Tables

Multi-Region replication for DynamoDB:

- One **replica table** per participating Region, kept in sync automatically.
- Replication is implemented using **DynamoDB Streams** under the hood.
- Conflict resolution: **last writer wins** (based on timestamp) if the same item is written concurrently in two Regions.
- Strongly consistent reads are only available **within the same Region** as the write — cross-Region reads are always eventually consistent.
- Transactions are **disabled by default** on global tables (multi-Region ACID transactions are a different guarantee than a single-Region transaction).

---

## Backup and restore

- **On-demand backups** — full table backup, created asynchronously, available for restore within minutes, no limit on how many you can take, doesn't consume provisioned throughput. Good for long-term retention/compliance needs.
- **Point-in-time recovery (PITR)** — continuous incremental backups; lets you restore the table to **any point within the last 35 days** without having to schedule anything manually. Typical use case: an accidental write/delete against a table (e.g. a test script hitting production by mistake).
- Both backup and restore can be triggered from the AWS Console, the CLI, or the DynamoDB API.
- A restore (either kind) always produces a **new table** — it does not overwrite the source table.

---

## DynamoDB API operation categories (recap)

- **Control operations** — create/manage tables and their indexes (e.g. `CreateTable`).
- **Data operations** — create/read/update/delete on items (`PutItem`, `GetItem`, `UpdateItem`, `DeleteItem`, `Query`, `Scan`).
- **Batch operations** — read or write multiple items in one call (`BatchGetItem`, `BatchWriteItem`).
- **Transaction operations** — coordinated, all-or-nothing multi-item changes (`TransactWriteItems`, `TransactGetItems`).

---

---

## Applied schema design — single-table design walkthrough

The following documents a real schema built with an **access-pattern-first** approach: start from "what queries does the application actually need?", then design the key structure to serve those queries efficiently — rather than modeling entities first and hoping the keys happen to support the access patterns later.

### Single-table design

One DynamoDB table holding multiple types of entities (e.g. a user profile and that user's transactions), rather than one table per entity type as in a relational model. Trade-off: more upfront key-design effort, but it allows retrieving related data (a user's profile *and* their recent transactions) in a single `Query` call instead of multiple round trips or joins (which DynamoDB doesn't support).

### Partition key convention: prefixed, shared partition key

```
PK: USER#<user_id>
```

Everything belonging to one user lives in the same physical partition, because they share the same partition key value. The `USER#` prefix exists specifically to avoid collisions with a different entity type that might reuse the same raw ID value by coincidence (e.g. a numeric ID that could belong to both a user and some other unrelated entity).

### Sort key convention: entity-type discrimination within one partition

```
SK (profile item):      PROFILE
SK (transaction item):  TXN#<timestamp ISO8601>#<id>
```

Two different SK *shapes* under the same PK let two different entity types coexist in one partition:

- A fixed value (`PROFILE`) for a singleton item — there's only ever one per user, so no need for a variable component.
- A prefixed, variable value (`TXN#...`) for a repeating item type — lets a user accumulate many items under the same partition key.

**Why ISO 8601 first in the sort key**: ISO 8601 timestamps (`YYYY-MM-DDTHH:MM:SSZ`) have the property that lexicographic (alphabetical) string ordering matches chronological ordering. Putting the timestamp first in the SK means a plain sorted `Query` on that partition returns items in chronological order automatically, with no extra client-side sorting.

**Why an ID at the end of the sort key**: appending a unique ID after the timestamp guarantees uniqueness even if two items are created within the same millisecond — it acts as a tie-breaker.

### Schemaless attributes, added incrementally

Not every item of a given type needs every attribute at the same time. A field can be added later to existing items via `UpdateItem` (see Commands file), with no schema migration required — this is a core practical benefit of DynamoDB's schemaless item model over a relational `ALTER TABLE`.

Minimal-data principle: don't store identity data (name, email) in a profile item if an identity provider already owns that responsibility elsewhere — store only what the application logic actually needs to read.

### GSI design rationale

```
GSI: status-index
  GSI partition key: status
  GSI sort key:       created_at
```

- `status` is a good **GSI** partition key specifically *because* it has very few distinct values that repeat heavily across items belonging to many different users. A GSI's job is to regroup items from different base-table partitions into a new grouping — querying "every item with `status = X`, across all users" is impossible on the base table's PK/SK alone, but trivial once a GSI reorganizes the same items by `status`.
- `created_at` as the GSI sort key orders results within each status group chronologically. A sort key is expected to vary per item — unlike a partition key, having a unique-ish value per item is completely normal for a sort key.
- Creating a secondary index upfront (even before it's needed) avoids a schema migration later — but note a **GSI can be added after table creation**, while an **LSI cannot** (must be defined at creation time).

### Related concepts worth knowing for the exam

- **Many-to-many relationships** in a single-table design are typically modeled with a dedicated relation item whose sort key doubles as the GSI partition key (and vice versa) — letting the same relation item be queried from either direction. Relevant for DVA-C02; not needed when the relationship in question is better served by a relational database instead.
- **Hot partition** risk applies here too: a partition key design that concentrates write/read volume on one value (e.g. one extremely active user) can bottleneck that partition even if the table as a whole is healthy.

---

## Debugging DynamoDB-backed Lambdas

### Why CloudWatch Logs matters here

A Lambda function has no visible terminal — anything it prints, or any exception it raises, is only visible after the fact in **CloudWatch Logs**. Each function gets its own log group (`/aws/lambda/<function-name>`), and every invocation writes a log stream into that group.

Typical debugging flow: a client-facing error (API Gateway deliberately returns a generic `"Internal server error"`, hiding internals) → find the function's log group → tail recent logs → read the actual stack trace and error type → fix the root cause → redeploy → retest.

### Gotcha #1 — `Decimal` is not JSON-serializable

`boto3.resource('dynamodb')` converts DynamoDB numeric values into Python's `Decimal` type on read (`Query`, `Scan`, `GetItem`) — to preserve exact numeric precision. Python's standard `json.dumps()` doesn't know how to serialize `Decimal` natively, which raises:

```
TypeError: Object of type Decimal is not JSON serializable
```

**Fix** — pass a custom `default` converter to `json.dumps`:

```python
from decimal import Decimal

def decimal_default(obj):
    if isinstance(obj, Decimal):
        return int(obj) if obj % 1 == 0 else float(obj)
    raise TypeError(f"Object of type {type(obj)} is not JSON serializable")

json.dumps(data, default=decimal_default)
```

**Rule of thumb**: this converter is only needed in functions that **read** from DynamoDB and return the result as JSON. A function that only **writes** (`put_item`) and echoes back the plain Python values it already had in memory never encounters a `Decimal` and doesn't need this at all — adding it unnecessarily (referencing a function that isn't defined in that file) just introduces a different error (`NameError`).

### Gotcha #2 — importing what you reference

After adding the `decimal_default` function above, forgetting the corresponding import:

```python
from decimal import Decimal
```

...produces a *different*, misleading error (`NameError: name 'Decimal' is not defined`) that looks unrelated to the original bug. Lesson: whenever a code snippet references a type/class, double check its import exists — this class of mistake is easy to make and easy to misdiagnose.

### Gotcha #3 — CORS with Lambda proxy integration

A request can succeed server-side (`200 OK`) and still be blocked client-side by the browser with a CORS error. Root cause: with **Lambda proxy integration**, the API Gateway-level CORS configuration (e.g. a `Cors` block in a SAM template) only auto-generates the CORS headers for the **OPTIONS preflight** mock response — it does **not** inject headers into the actual Lambda response. The Lambda's returned dict fully controls the real response (status, body, *and* headers), so CORS headers must be returned explicitly by the Lambda itself, on every response path including error responses:

```python
return {
    "statusCode": status_code,
    "headers": {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Headers": "Content-Type",
        "Access-Control-Allow-Methods": "GET,POST,OPTIONS"
    },
    "body": json.dumps(data)
}
```

A successful OPTIONS preflight is **not** proof that the real request will pass CORS — they're two independent response paths.

---

## References

- Core Components: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html
- Query: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.html
- Single-Table Design: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-general-nosql-design.html
- Best Practices for Modeling Relational Data (adjacency list, many-to-many): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-adjacency-graphs.html
- Global Secondary Indexes: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html
- Local Secondary Indexes: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/LSI.html
- Partition key design / hot partitions: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html
- `AWS::DynamoDB::Table` (CloudFormation): https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dynamodb-table.html
- `AWS::Serverless::SimpleTable` (SAM, no GSI support): https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-simpletable.html
- CloudWatch Logs for Lambda: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs.html
