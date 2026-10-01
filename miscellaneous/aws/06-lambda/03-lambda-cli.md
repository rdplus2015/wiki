# AWS Lambda — Commands Reference

Practical reference: AWS CLI commands for managing Lambda functions and permissions, plus the boto3 (Python SDK) patterns used when writing a Lambda itself. Identifiers are placeholders — replace with your own.

---

## Invoking a function

### Synchronously (caller waits for the response)

```bash
aws lambda invoke \
  --function-name my-function \
  --payload '{ "name": "Bob" }' \
  response.json
```

Response (written to `response.json`, summary also printed):
```json
{"ExecutedVersion": "$LATEST", "StatusCode": 200}
```

### Asynchronously (fire-and-forget, Lambda queues the event)

```bash
aws lambda invoke \
  --function-name my-function \
  --invocation-type Event \
  --payload '{ "name": "Bob" }' \
  response.json
```

Acknowledgement:
```json
{"StatusCode": 202}
```

`202` only confirms the event was accepted into the queue — not that the function succeeded.

---

## Permissions

### Give an AWS service permission to invoke a function (resource policy)

```bash
aws lambda add-permission \
  --function-name my-function \
  --action lambda:InvokeFunction \
  --statement-id sns-invoke \
  --principal sns.amazonaws.com
```

This creates/updates the function's **resource policy** — it governs who/what may call `lambda:InvokeFunction` on this function. Different from the execution role (below), which governs what the function itself is allowed to do.

Example resulting resource policy (S3 invoking a function, scoped to one specific bucket via a condition):

```json
{
  "Version": "2012-10-17",
  "Id": "default",
  "Statement": [
    {
      "Sid": "lambda-fd269e28-988b-4d2b-96ae-eabcd7dc399c",
      "Effect": "Allow",
      "Principal": { "Service": "s3.amazonaws.com" },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:function:myFirstFunction",
      "Condition": {
        "ArnLike": { "AWS:SourceARN": "arn:aws:s3:::myBucket1" }
      }
    }
  ]
}
```

### Execution role (what the function itself can do)

Not a CLI one-liner — defined as an IAM role with two policies attached when the function is created:

- **IAM policy** — the actions the function may take on other AWS resources (e.g. write to a DynamoDB table, write CloudWatch logs).
- **Trust policy** — allows the Lambda *service* to assume this role in the first place.

Example trust policy (always the same shape for Lambda):
```json
{
  "Effect": "Allow",
  "Principal": { "Service": "lambda.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

Example IAM policy (minimum needed just to write CloudWatch logs):
```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Resource": "arn:aws:logs:*:*:*"
}
```

---

## Creating and updating functions (.zip deployment)

### Create a function

```bash
aws lambda create-function \
  --function-name my-function \
  --runtime python3.12 \
  --zip-file fileb://my-function.zip \
  --handler app.lambda_handler \
  --role arn:aws:iam::123456789012:role/service-role/my-function-role
```

### Update a function's code only

```bash
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://my-function.zip
```

### Update a function's configuration (memory, timeout, env vars, etc.)

```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --memory-size 256 \
  --timeout 10 \
  --environment "Variables={TABLE_NAME=my-table,LOG_LEVEL=INFO}"
```

---

## Versions and aliases

### Publish a new version (immutable snapshot of $LATEST)

```bash
aws lambda publish-version --function-name my-function
```

Each version gets its own ARN: `arn:aws:lambda:region:account-id:function:my-function:1`

### Create / update an alias (mutable pointer to a version)

```bash
aws lambda create-alias \
  --function-name my-function \
  --name PROD \
  --function-version 1

aws lambda update-alias \
  --function-name my-function \
  --name PROD \
  --function-version 2
```

Invoking (or configuring an event source to invoke) the alias ARN means promoting/rolling back a function never requires touching the event source's configuration:

```
arn:aws:lambda:region:account-id:function:my-function:PROD
```

---

## Layers

```bash
# Publish a layer from a local zip
aws lambda publish-layer-version \
  --layer-name my-shared-deps \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.12

# Attach a layer to a function (up to 5 layers per function)
aws lambda update-function-configuration \
  --function-name my-function \
  --layers arn:aws:lambda:region:account-id:layer:my-shared-deps:1
```

---

## Describe / list / delete

```bash
aws lambda get-function --function-name my-function
aws lambda list-functions
aws lambda delete-function --function-name my-function
```

---

## boto3 (Python SDK) patterns actually used inside a Lambda

### Reusing a client/resource across warm invocations

```python
import boto3

# Created ONCE at import time — reused across warm invocations.
# Creating it inside lambda_handler would re-connect on every call.
resource = boto3.resource('dynamodb')
table = resource.Table('my-table')
```

### Building a valid API Gateway response

```python
import json

def build_response(status_code, data):
    return {
        "statusCode": status_code,
        "headers": {
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Headers": "Content-Type",
            "Access-Control-Allow-Methods": "GET,POST,OPTIONS"
        },
        "body": json.dumps(data)  # body MUST be a JSON string, never a raw dict
    }
```

### Parsing the incoming event (API Gateway proxy integration)

```python
def lambda_handler(event, context):
    body = json.loads(event["body"])          # event["body"] is always a JSON string
    amount = body.get("amount")                # .get() avoids a KeyError on missing fields
    merchant = body.get("merchant")
    user_id = body.get("user_id", "unknown")   # default value when the key is absent
```

### Generating server-side values (never trust the client for these)

```python
import uuid
from datetime import datetime, timezone

transaction_id = str(uuid.uuid4())
timestamp = datetime.now(timezone.utc).isoformat(timespec="seconds").replace("+00:00", "Z")
# .isoformat(timespec="seconds") drops microseconds
# .replace("+00:00", "Z") converts to the Z-suffixed UTC form DynamoDB sort keys rely on
```

### Full end-to-end handler (HTTP in → validate → write to DynamoDB → HTTP out)

```python
import json
import uuid
from datetime import datetime, timezone

import boto3

resource = boto3.resource('dynamodb')
table = resource.Table('my-table')


def build_response(status_code, data):
    return {
        "statusCode": status_code,
        "headers": {
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Headers": "Content-Type",
            "Access-Control-Allow-Methods": "GET,POST,OPTIONS"
        },
        "body": json.dumps(data)
    }


def lambda_handler(event, context):
    body = json.loads(event["body"])

    amount = body.get("amount")
    merchant = body.get("merchant")
    user_id = body.get("user_id", "unknown")

    if amount is None or merchant is None:
        return build_response(400, {"error": "amount and merchant are required"})

    transaction_id = str(uuid.uuid4())
    timestamp = datetime.now(timezone.utc).isoformat(timespec="seconds").replace("+00:00", "Z")

    pk = f"USER#{user_id}"
    sk = f"TXN#{timestamp}#{transaction_id}"

    item = {
        "PK": pk,
        "SK": sk,
        "amount": amount,
        "merchant": merchant,
        "currency": "CAD",
        "status": "LEGIT",
        "created_at": timestamp,
        "entity_type": "TRANSACTION"
    }

    table.put_item(Item=item)

    return build_response(200, {"message": "Transaction recorded", "item": item})
```

---

## CloudWatch Logs — reading a function's logs

```bash
aws logs describe-log-groups --profile my-profile \
  --query "logGroups[?contains(logGroupName, 'KEYWORD')].logGroupName" --output text

aws logs tail /aws/lambda/my-function --profile my-profile --since 10m
aws logs tail /aws/lambda/my-function --profile my-profile --follow
```

Every function has its own log group: `/aws/lambda/<function-name>`. Each invocation has a unique `RequestId`, plus `START`/`END`/`REPORT` log lines — the `REPORT` line shows billed duration, configured/used memory, and (if present) `Init Duration`, which flags a cold start.
