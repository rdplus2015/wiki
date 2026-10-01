# DynamoDB — Real Schema Example (Fraud Detection System)

A concrete, worked single-table design, taken from an actual project (a fraud-detection portfolio app). Shows the full reasoning path — access patterns first, then key design — applied to real entities instead of abstract placeholders.

---

## 1. Start from access patterns, not from entities

The SQL reflex is "what columns does a transaction have?" — the wrong starting question in DynamoDB. The right one: **what will the application need to read, and how?**

DynamoDB gives two fast, native read paths:

```
get_item(PK, SK)        → one exact item
query(PK, SK condition) → every item sharing the same PK, sorted/filtered by SK
```

No `WHERE amount > 100` without a dedicated GSI. A full `scan()` exists but is slow and expensive — not a real access pattern.

Access patterns actually needed by this project:

```
A) Write a new transaction for a given user
B) Read a user's full transaction history, most recent first
C) Read a user's profile (usual country) for fraud scoring
D) Read a user's transactions from the last 60 seconds (velocity rule)
E) Read every transaction currently flagged FRAUD, across all users
```

Everything below is designed to answer these five patterns directly.

---

## 2. Table

```
Table: FraudDetectionTable
```

Single-table design: one physical table holding several different item types.

---

## 3. Partition Key (PK) — groups what should be read together

```
PK = USER#<user_id>
```

Example: `USER#42`

The PK must be **shared by everything you want to retrieve together in one `query()`**. Here, a user's profile and all of their transactions live under the same PK, in the same physical partition.

The `USER#` prefix isn't decoration — it prevents ID collisions between entity types. If a `merchant_id` ever happened to equal a `user_id` numerically, an unprefixed key would silently collide. `USER#42` vs `MERCHANT#42` can never be confused.

---

## 4. Sort Key (SK) — orders and distinguishes within the group

Two different SK shapes coexist under the same PK, one per entity type:

```
SK (profile):      PROFILE
SK (transaction):  TXN#<timestamp ISO8601>#<transaction_id>
```

Example: `TXN#2026-08-19T14:32:07Z#a1b2c3d4`

- **`PROFILE`** — a fixed value. Only one profile item exists per user, so no variable component is needed.
- **`TXN#...`** — a prefix (`TXN#`, consistent with `USER#`), followed by:
  - **ISO 8601 timestamp first** — DynamoDB sorts sort keys **lexicographically** (as text), not as dates. ISO 8601 (`YYYY-MM-DDTHH:MM:SSZ`) has the property that alphabetical order = chronological order (year first, then month, then day...). A format like `19/08/2026` would break sorting completely.
  - **transaction ID last** — a tie-breaker. If two transactions land at the exact same millisecond, the SK stays unique; without it, the second write would silently overwrite the first (same PK+SK = same item).

Sorted read, most recent first:

```python
table.query(
    KeyConditionExpression=Key('PK').eq('USER#42'),
    ScanIndexForward=False  # descending → most recent first
)
```

---

## 5. Why single-table design here

- One `query(PK="USER#42")` returns a user's profile **and** recent transactions in a single network call — no joins (DynamoDB doesn't support them), no second round trip.
- Fewer tables to declare and manage in `template.yaml`.
- The pattern AWS recommends for serverless DynamoDB usage.

The real trade-off: more upfront design discipline. You can't improvise a new table casually later — every new entity type has to fit cleanly into the existing PK/SK scheme.

Optional but useful convention: an explicit `entity_type` attribute on every item, redundant with the SK prefix but more readable when eyeballing the console:

```json
{
  "PK": "USER#42",
  "SK": "TXN#2026-08-19T14:32:07Z#a1b2c3d4",
  "entity_type": "TRANSACTION",
  "amount": 5200
}
```

---

## 6. Shared PK vs dedicated PK

**Dedicated PK** (e.g. `MERCHANT#456`) — the entity lives alone in its own partition; `query()` on it returns only that entity. *Not used in this schema* — see §7, merchant was deliberately left out as a standalone entity.

**Shared PK** (e.g. `USER#42` for both profile and transactions) — the core single-table technique. Multiple item types deliberately cohabit under the same PK, distinguished by SK:

```
PK="USER#42", SK="PROFILE"                              → profile (usual country)
PK="USER#42", SK="TXN#2026-08-19T14:32:07Z#a1b2c3d4"    → transaction 1
PK="USER#42", SK="TXN#2026-08-19T15:01:00Z#f9e8d7c6"    → transaction 2
```

Why it matters here: the fraud-scoring Lambda needs **both** the profile (usual country) **and** recent transactions (velocity rule) to do its job. With this structure, a single `query(PK="USER#42")` returns everything — not two separate queries against two separate tables. That's the actual performance win the pattern promises.

To tell returned items apart in application code: check the SK prefix (`SK.startswith("TXN#")` vs `SK == "PROFILE"`), or the `entity_type` attribute if present.

---

## 7. Deciding what actually becomes an item type

Decided from real project needs, not in the abstract:

| Candidate entity | Decision |
|---|---|
| `TRANSACTION` | Needed — core data, written on every request |
| `USER_PROFILE` | Needed, but minimal — only `usual_country`, required by the "IP country ≠ usual country" scoring rule. **No** name, email, or other identity data: identity is Cognito's responsibility, this table is not the identity source of truth |
| `MERCHANT` | **Not** modeled as a separate entity — the merchant is just a free-text field on the transaction; no business logic needs it as its own item |

What does *not* live in DynamoDB at all: analysts, review decisions, consolidated alerts — that data lives in a relational store (Aurora) instead, because it needs joins and ad-hoc analytical queries that DynamoDB isn't built for.

---

## 8. Attributes

### Transaction item

```json
{
  "PK": "USER#42",
  "SK": "TXN#2026-08-19T14:32:07Z#a1b2c3d4",
  "entity_type": "TRANSACTION",
  "amount": 5200,
  "merchant": "Best Buy",
  "currency": "CAD",
  "status": "FRAUD",
  "score": 2,
  "country_detected": "FR",
  "created_at": "2026-08-19T14:32:07Z",
  "ttl": 1755610327
}
```

Schemaless in practice: not every `TRANSACTION` item has every attribute at the same time. `score` and `country_detected` only start existing once the scoring feature ships — added via `update_item` on items that were created earlier without them. No schema migration needed. `entity_type` is redundant with the SK prefix but easier to read/filter on in the console. `ttl` is a Unix timestamp enabling automatic expiry (here, 90 days after creation).

### Profile item

```json
{
  "PK": "USER#42",
  "SK": "PROFILE",
  "entity_type": "USER_PROFILE",
  "usual_country": "CA"
}
```

Deliberately minimal — just enough for the scoring Lambda to compare "detected country" vs "usual country". No PII beyond what's strictly needed.

---

## 9. GSI — `status-index`

```
GSI: status-index
  GSI Partition Key: status
  GSI Sort Key:       created_at
```

A GSI builds a second physical organization of the same items, independent of the base table's PK partitions.

- **GSI PK = `status`** — a good candidate specifically because it has few distinct values (`LEGIT` / `FRAUD` / `PENDING`) repeated heavily across many different users' items. A GSI partition key needs repeated values to be useful — this is the *opposite* of what makes a good table partition key. It lets you query "every FRAUD transaction, across every user" without a full table scan — used by the alerting and reporting logic.
- **GSI SK = `created_at`** — sorts items inside each status "drawer" chronologically (e.g. all FRAUD transactions, oldest to newest). A sort key is expected to vary per item, unlike a partition key — a precise timestamp in a GSI SK is completely normal.
- Created from day one, even though it's barely used until the scoring feature exists, to avoid a schema migration later (a GSI *can* be added after table creation — unlike an LSI, see below).

---

## 10. Consolidated final schema

```
Table: FraudDetectionTable

PK: USER#<user_id>
SK (profile):      PROFILE
SK (transaction):  TXN#<timestamp ISO8601>#<transaction_id>

Transaction attributes: amount, merchant, currency, status (LEGIT/FRAUD/PENDING),
                         score, country_detected, created_at, entity_type, ttl (90d)
Profile attributes:     usual_country, entity_type

GSI: status-index
  GSI PK: status
  GSI SK: created_at
```

---

## 11. Conceptual reminders this schema relies on

- **Partitioning** — each distinct PK value lives in its own physical partition. `USER#42` and `USER#43` are never in the same partition. This is what enables horizontal scaling: DynamoDB spreads partitions across many servers, each handling a slice of the key space.
- **GSI** — doesn't just query already-grouped data, it **regroups items coming from different base-table partitions** (e.g. FRAUD transactions belonging to many different users) into new partitions based on a different key (`status` instead of `user_id`).
- **LSI** — same PK as the table, an alternate SK; stays confined to a single partition (one user) at a time. Must be defined at table creation, cannot be added afterward. Not used in this schema — no identified need to sort differently *within* a single user's items.
- **Many-to-many relationships** — handled via a relation item whose SK doubles as a GSI PK (and vice versa). Out of scope here (the analyst↔alert relationship lives in Aurora instead), but relevant knowledge for the DVA-C02 exam.
- **Hot partition** — a PK receiving disproportionate traffic relative to others can bottleneck even if the rest of the table scales fine. One reason `user_id` (naturally high-cardinality, evenly distributed) was chosen as the base table's partition key instead of something coarser like `status`.

---

## 12. One technical point before writing IaC

`AWS::Serverless::SimpleTable` (the simplest SAM shortcut for a DynamoDB table) **does not support GSIs**. A table like `FraudDetectionTable` with a `status-index` GSI requires the full CloudFormation `AWS::DynamoDB::Table` resource inside the SAM `template.yaml` — SAM and native CloudFormation resources coexist in the same template.

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
