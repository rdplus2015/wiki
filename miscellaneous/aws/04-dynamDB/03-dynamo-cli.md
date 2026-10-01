# DynamoDB — Commands Reference

Practical reference for working with Amazon DynamoDB: AWS CLI, Python SDK (boto3), and the debugging commands used alongside it. Table names, profile names, and resource identifiers below are placeholders — replace them with your own.

---

## Identity check before any operation

```bash
aws sts get-caller-identity --profile my-profile
```

Confirms which account/identity the CLI will act as before creating, reading, or deleting table data — especially important when multiple profiles exist.

---

## Table management (Control operations)

### Create a table (AWS CLI)

```bash
aws dynamodb create-table \
  --table-name my-table \
  --attribute-definitions \
      AttributeName=PK,AttributeType=S \
      AttributeName=SK,AttributeType=S \
  --key-schema \
      AttributeName=PK,KeyType=HASH \
      AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --profile my-profile
```

- `--billing-mode PAY_PER_REQUEST` = on-demand throughput. Use `PROVISIONED` with `--provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5` instead if you need provisioned capacity.
- `HASH` = partition key, `RANGE` = sort key.

### Create a table (boto3 / Python SDK)

```python
import boto3

dynamodb = boto3.resource('dynamodb')

table = dynamodb.create_table(
    TableName='my-table',
    KeySchema=[
        {'AttributeName': 'PK', 'KeyType': 'HASH'},
        {'AttributeName': 'SK', 'KeyType': 'RANGE'}
    ],
    AttributeDefinitions=[
        {'AttributeName': 'PK', 'AttributeType': 'S'},
        {'AttributeName': 'SK', 'AttributeType': 'S'}
    ],
    ProvisionedThroughput={
        'ReadCapacityUnits': 5,
        'WriteCapacityUnits': 5
    }
)
```

`create_table` is asynchronous — DynamoDB immediately returns `TableStatus: CREATING`, then flips to `ACTIVE` once ready. Reads/writes only work on an `ACTIVE` table.

### Create a table with a Global Secondary Index (full CloudFormation / SAM, when `AWS::Serverless::SimpleTable` isn't enough)

```yaml
MyTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: my-table
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: PK
        AttributeType: S
      - AttributeName: SK
        AttributeType: S
      - AttributeName: status
        AttributeType: S
      - AttributeName: created_at
        AttributeType: S
    KeySchema:
      - AttributeName: PK
        KeyType: HASH
      - AttributeName: SK
        KeyType: RANGE
    GlobalSecondaryIndexes:
      - IndexName: status-index
        KeySchema:
          - AttributeName: status
            KeyType: HASH
          - AttributeName: created_at
            KeyType: RANGE
        Projection:
          ProjectionType: ALL
```

> `AWS::Serverless::SimpleTable` (the SAM shortcut) does **not** support GSIs. Use the full `AWS::DynamoDB::Table` CloudFormation resource whenever a GSI/LSI is needed.

### Describe / list tables

```bash
aws dynamodb describe-table --table-name my-table --profile my-profile
aws dynamodb list-tables --profile my-profile
```

### Delete a table

```bash
aws dynamodb delete-table --table-name my-table --profile my-profile
```

---

## Data operations (CRUD)

### Put an item — create or overwrite (AWS CLI)

```bash
aws dynamodb put-item \
  --table-name my-table \
  --item '{
    "PK": {"S": "USER#42"},
    "SK": {"S": "PROFILE"},
    "usual_country": {"S": "CA"}
  }' \
  --profile my-profile
```

### Put an item (boto3)

```python
table = boto3.resource('dynamodb').Table('my-table')

table.put_item(
    Item={
        'PK': 'USER#42',
        'SK': 'TXN#2026-01-15T10:30:00Z#a1b2c3',
        'amount': 5200,
        'merchant': 'Example Store',
        'currency': 'CAD',
        'status': 'LEGIT',
        'entity_type': 'TRANSACTION'
    }
)
```

`put_item` creates a new item, or **completely replaces** an existing item sharing the same primary key. Add a `ConditionExpression` (e.g. `attribute_not_exists(PK)`) to avoid accidental overwrites.

### Get a single item (AWS CLI)

```bash
aws dynamodb get-item \
  --table-name my-table \
  --key '{"PK": {"S": "USER#42"}, "SK": {"S": "PROFILE"}}' \
  --profile my-profile
```

### Get a single item (boto3)

```python
response = table.get_item(
    Key={'PK': 'USER#42', 'SK': 'PROFILE'}
)
item = response.get('Item')
```

`get_item` requires the **full primary key** (PK + SK if composite). Returns `Decimal` types for numbers — see the JSON serialization gotcha below. Defaults to eventually consistent reads; pass `ConsistentRead=True` for a strongly consistent read.

### Update an item

```bash
aws dynamodb update-item \
  --table-name my-table \
  --key '{"PK": {"S": "USER#42"}, "SK": {"S": "TXN#2026-01-15T10:30:00Z#a1b2c3"}}' \
  --update-expression "SET #s = :val" \
  --expression-attribute-names '{"#s": "status"}' \
  --expression-attribute-values '{":val": {"S": "FRAUD"}}' \
  --profile my-profile
```

```python
table.update_item(
    Key={'PK': 'USER#42', 'SK': 'TXN#2026-01-15T10:30:00Z#a1b2c3'},
    UpdateExpression='SET #s = :val',
    ExpressionAttributeNames={'#s': 'status'},
    ExpressionAttributeValues={':val': 'FRAUD'}
)
```

`update_item` edits specific attributes without rewriting the whole item, and can also create the item if it doesn't exist. This is also how you add new attributes to items created earlier without a schema migration (e.g. adding `score` to transactions that didn't have it at write time).

### Delete an item

```bash
aws dynamodb delete-item \
  --table-name my-table \
  --key '{"PK": {"S": "USER#42"}, "SK": {"S": "PROFILE"}}' \
  --profile my-profile
```

```python
table.delete_item(
    Key={'PK': 'USER#42', 'SK': 'PROFILE'}
)
```

### Condition expressions (works with put-item, update-item, delete-item)

Only run the write if a condition evaluates to true — e.g. only unlock an account if the last failed login was more than 24 hours ago:

```python
table.update_item(
    Key={'userId': '1'},
    UpdateExpression='SET accountLocked = :n',
    ConditionExpression='currentLoginTime > lastFailedLoginTime + :window',
    ExpressionAttributeValues={':n': 'N', ':window': 86400}
)
```

---

## Query vs Scan

### Query — read items matching a key condition (efficient)

```bash
aws dynamodb query \
  --table-name my-table \
  --key-condition-expression "PK = :pk" \
  --expression-attribute-values '{":pk": {"S": "USER#42"}}' \
  --profile my-profile
```

```python
from boto3.dynamodb.conditions import Key

response = table.query(
    KeyConditionExpression=Key('PK').eq('USER#42')
)
items = response['Items']
```

Query requires a key condition (on PK, optionally narrowed by SK) and can be further refined with an optional `FilterExpression`. Always prefer Query over Scan when the access pattern allows it.

### Query a GSI

```python
response = table.query(
    IndexName='status-index',
    KeyConditionExpression=Key('status').eq('FRAUD')
)
```

### Scan — read every item in the table (expensive, avoid at scale)

```bash
aws dynamodb scan --table-name my-table --profile my-profile
```

```python
from boto3.dynamodb.conditions import Attr

response = table.scan(
    FilterExpression=Attr('amount').gt(5000)
)
```

Scan reads the **entire table** (or index) before filtering — it does not use the primary key to narrow the read. It's the most common diagnostic/debugging command (quickly dumping a whole table's content), but should not be used as an application access pattern once volume grows. Use a parallel scan if a full scan is unavoidable.

### Limiting returned data

```python
response = table.query(
    KeyConditionExpression=Key('PK').eq('USER#42'),
    Limit=10
)
```

Two factors cap what a Query/Scan returns: the automatic 1 MB pagination limit per request (paginate with `LastEvaluatedKey` / `ExclusiveStartKey`), and an explicit `Limit` parameter capping the number of items.

---

## Batch operations

### Batch get (read up to 100 items / 16 MB from multiple tables)

```python
dynamodb = boto3.resource('dynamodb')

response = dynamodb.batch_get_item(
    RequestItems={
        'my-table': {
            'Keys': [
                {'PK': 'USER#42', 'SK': 'PROFILE'},
                {'PK': 'USER#43', 'SK': 'PROFILE'}
            ]
        }
    }
)
```

### Batch write (put/delete up to 25 items / 16 MB)

```python
with table.batch_writer() as batch:
    batch.put_item(Item={'PK': 'USER#42', 'SK': 'TXN#...', 'amount': 100})
    batch.put_item(Item={'PK': 'USER#43', 'SK': 'TXN#...', 'amount': 250})
    batch.delete_item(Key={'PK': 'USER#44', 'SK': 'PROFILE'})
```

If one request inside a batch fails, the rest still succeed — failed keys/items are returned so they can be retried. `batch_writer()` handles this retry logic automatically.

---

## Transactions (all-or-nothing across items/tables)

```python
client = boto3.client('dynamodb')

client.transact_write_items(
    TransactItems=[
        {'Put': {'TableName': 'my-table', 'Item': {...}}},
        {'Update': {'TableName': 'my-table', 'Key': {...}, 'UpdateExpression': 'SET ...'}}
    ]
)

client.transact_get_items(
    TransactItems=[
        {'Get': {'TableName': 'my-table', 'Key': {...}}}
    ]
)
```

`TransactWriteItems` bundles PutItem/UpdateItem/DeleteItem across one or more tables — if any operation or condition fails, the entire transaction is rejected. `TransactGetItems` bundles reads the same way.

---

## Backup and restore

### On-demand backup

```bash
aws dynamodb create-backup \
  --table-name my-table \
  --backup-name my-table-backup-2026-01-15 \
  --profile my-profile
```

```bash
aws dynamodb list-backups --table-name my-table --profile my-profile
```

### Restore from an on-demand backup

```bash
aws dynamodb restore-table-from-backup \
  --target-table-name my-table-restored \
  --backup-arn arn:aws:dynamodb:region:account-id:table/my-table/backup/backup-id \
  --profile my-profile
```

### Point-in-time recovery (PITR) — restore to any point within the last 35 days

```bash
# Enable PITR on a table
aws dynamodb update-continuous-backups \
  --table-name my-table \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true \
  --profile my-profile

# Restore to a specific timestamp
aws dynamodb restore-table-to-point-in-time \
  --source-table-name my-table \
  --target-table-name my-table-restored \
  --restore-date-time 2026-01-15T10:00:00Z \
  --profile my-profile
```

Restores (backup or PITR) always create a **new table** — they never overwrite the source table in place.

---

## CloudWatch Logs — debugging commands (used alongside DynamoDB-backed Lambdas)

```bash
# Find a function's log group name (suffix is auto-generated, not predictable)
aws logs describe-log-groups --profile my-profile \
  --query "logGroups[?contains(logGroupName, 'KEYWORD')].logGroupName" --output text

# Read recent logs
aws logs tail /aws/lambda/my-function-name --profile my-profile --since 10m

# Follow logs live (like tail -f)
aws logs tail /aws/lambda/my-function-name --profile my-profile --follow
```

Used to trace errors raised inside a Lambda that reads/writes DynamoDB (e.g. a `TypeError` on `json.dumps()`, a `NameError` from a missing import) — see the Notes file for the full debugging walkthrough.

---

## Quick diagnostic commands (most frequently used)

```bash
# Confirm identity/profile before touching data
aws sts get-caller-identity --profile my-profile

# Dump the whole table quickly (debugging only — not an access pattern)
aws dynamodb scan --table-name my-table --profile my-profile

# Check table status/schema
aws dynamodb describe-table --table-name my-table --profile my-profile
```
