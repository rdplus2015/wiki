# AWS SAM — Template Structure Deep-Dive

Detailed look at the two sections of a SAM template that carry the most
weight in practice: `Globals` and `Resources`, plus IAM permissions via
policy templates. Builds on the general anatomy covered separately — this
file goes one level deeper into each piece.


## `Globals` — avoiding repetition across functions

`Globals` defines default values **automatically inherited** by every
`AWS::Serverless::*` resource (Function, Api, HttpApi, SimpleTable) — not
native CloudFormation resource types like `AWS::DynamoDB::Table`.

**The problem without `Globals`**: with several Lambda functions in a
project, repeating `Runtime`, `Timeout`, `MemorySize` on each one is a
source of mistakes and inconsistencies if one of those values ever needs to
change later — easy to update one function and forget another.

```yaml
Globals:
  Function:
    Runtime: python3.13
    Timeout: 10
    MemorySize: 128
    LoggingConfig:
      LogFormat: JSON

Resources:
  IngestorFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ingestor/
      Handler: app.lambda_handler
      # Runtime, Timeout, MemorySize, LoggingConfig all inherited from Globals
```

A function can still define its own value explicitly in its own
`Properties` — in that case, the local value **overrides** the global one
(standard cascading-override behavior, same idea as CSS specificity or a
function default argument).

**Important scope limit**: a resource declared with a native CloudFormation
type — for example `AWS::DynamoDB::Table` rather than the SAM shortcut
`AWS::Serverless::SimpleTable` — does **not** inherit anything from
`Globals`. All of its properties must be written out explicitly, every
time, on that resource.

`Globals.Api` can also carry shared API-level settings such as CORS. That
block must sit at the same level as `Function:` inside `Globals` — not
nested inside `Function:` — since `Api` and `Function` are two separate
sibling keys under `Globals`, each scoping defaults for its own resource
category:

```yaml
Globals:
  Function:
    Timeout: 3
  Api:
    Cors:
      AllowMethods: "'GET,POST,OPTIONS'"
      AllowHeaders: "'Content-Type'"
      AllowOrigin: "'*'"
```

Also worth noting: `Globals.Api.Cors` only configures the OPTIONS preflight
mock response that API Gateway returns automatically. It does **not** make
every Lambda's actual response include the CORS header — each function
still has to explicitly return `Access-Control-Allow-Origin` in its own
response body/headers for the browser to accept the real (non-preflight)
response.



## `Resources` — the DynamoDB table case in detail

DynamoDB is a good illustration of why `Resources` sometimes needs the full
native CloudFormation syntax instead of a SAM shortcut.

SAM's simplest table shortcut, `AWS::Serverless::SimpleTable`, does **not**
support Global Secondary Indexes (GSIs). Any table that needs a GSI must
use the full native type, `AWS::DynamoDB::Table`, instead — with every
property spelled out explicitly (no `Globals` inheritance, as noted above).

```yaml
FraudDetectionTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: FraudDetectionTable
    BillingMode: PAY_PER_REQUEST

    # STEP 1 — Declare the existence and type of every attribute
    # that will act as a key somewhere (main table OR a GSI).
    # Does NOT include "free" attributes like amount, merchant, etc.
    AttributeDefinitions:
      - AttributeName: PK
        AttributeType: S
      - AttributeName: SK
        AttributeType: S
      - AttributeName: status
        AttributeType: S
      - AttributeName: created_at
        AttributeType: S

    # STEP 2 — Assign the role of each attribute for THE MAIN TABLE
    KeySchema:
      - AttributeName: PK
        KeyType: HASH
      - AttributeName: SK
        KeyType: RANGE

    # STEP 2bis — Same principle, but for a separate GSI:
    # define its own KeySchema, reusing attributes declared above
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

**The mental model that matters here**: `AttributeDefinitions` only
declares an attribute's *existence and data type* — it's the registry of
every attribute that will play the role of a key *somewhere* (main table or
any GSI). It does not list every attribute a table will ever store — only
the ones acting as a key. `KeySchema` is where an attribute actually gets
*assigned* a role (`HASH` = partition key, `RANGE` = sort key) — and this
assignment happens separately for the main table and for each GSI, each
with its own `KeySchema` block, reusing attributes already declared once in
`AttributeDefinitions`.



## IAM permissions for a function — `Policies`

A Lambda function has, by default, no permission to access other AWS
services (e.g. DynamoDB) beyond basic execution logging. SAM provides
shorthand **policy templates** to avoid writing raw IAM JSON by hand:

```yaml
Policies:
  - DynamoDBCrudPolicy:
      TableName: !Ref FraudDetectionTable
```

`DynamoDBCrudPolicy` grants standard CRUD permissions (get/put/update/
delete/query/scan) scoped specifically to the referenced table —
`!Ref FraudDetectionTable` resolves dynamically to the table's actual name
rather than hardcoding it, so the permission stays correct even if the
table's logical name or physical name changes.

This one line replaces what would otherwise be a hand-written IAM policy
document (trust policy + permissions policy + attaching both to a role) —
SAM generates and attaches the role automatically based on the policy
template declared.



## References

- [SAM Globals section](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification-template-anatomy-globals.html)
- [AWS::DynamoDB::Table reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dynamodb-table.html)
- [SAM policy templates](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-policy-templates.html)
- [SAM Function LoggingConfig](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-function.html#sam-function-loggingconfig)