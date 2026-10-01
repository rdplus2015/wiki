# AWS Lambda — Key Concepts & Notes

Conceptual reference: what Lambda is, how it's invoked, how permissions work, how it's deployed/monitored, and the reasoning behind a real handler written for this pattern.

---

## Serverless computing — why it exists

Cloud computing lets you abstract away the infrastructure layer so you can focus on business logic instead of operations. On a spectrum of abstraction:

```
On-premises → Virtual servers in the cloud → Containers → Serverless
```

With traditional servers, you must think about: configuring instances, patching the OS, installing the platform, configuring scaling/load balancing, building and deploying, continuously securing/monitoring. With serverless, you only think about: building/deploying applications and monitoring/maintaining them. Everything else is handled for you, and you pay only for the time your code actually executes — not for idle capacity.

Modern serverless applications favor: **small pieces, loosely joined** (so parts can change/scale/deploy independently), **purpose-built data stores**, **specialized managed services for integration**, and **infrastructure as code for deployment automation**.

### Monolith vs microservices

A monolith does everything in one deployable unit — simpler to start, but as it grows: one shared tech stack, one shared database sized for the worst-case workload, and a shared release pipeline that causes friction (merge conflicts, full-app rebuilds/retests on every change). Microservices let you scale, choose tech, and deploy each piece independently — at the cost of a higher initial learning curve.

---

## What Lambda is

A serverless compute service that runs your code without provisioning or managing servers.

- Invokes your code in response to events.
- Scales automatically.
- Provides built-in monitoring/logging via CloudWatch.
- You bring your own code (Node.js, Python, Java, C#, Go, Ruby, or a custom runtime) — Lambda doesn't force a new language on you.
- Pay only for the compute time actually used — no charge while idle.

### How it works, step by step

1. Upload your code to Lambda, or write it directly in the Lambda console editor.
2. Configure your code to run when events occur — in other AWS services, at HTTP endpoints (via API Gateway), or as part of in-app activity.
3. Lambda runs your code only when an event activates it, using only the compute resources needed.
4. You pay only for the compute time used.

**Functions and event sources are the two core components.** An *event source* is whatever publishes events to Lambda. A *function* is your custom code. Lambda runs the function on your behalf whenever the event source fires.

### Example use cases

- **Web app**: S3 hosts a static front end → user action calls API Gateway → API Gateway invokes Lambda → Lambda reads/writes DynamoDB → data returned to the user.
- **Real-time stream processing**: events land in Kinesis → Lambda is invoked on new records → processes them → writes results to DynamoDB.
- **Backends** for web/mobile/IoT/third-party API requests (commonly paired with API Gateway).
- **Data processing** triggered by changes in state (S3 upload, DynamoDB write, a CloudWatch alarm, a schedule).
- **Chatbots** (Amazon Lex) and **Alexa skills**.
- **IT automation** — scheduled/event-driven housekeeping tasks (firmware updates, starting/stopping EC2 instances, security group updates).

---

## Lambda functions — what makes one up

A Lambda function bundles:

- **Access permissions** — who/what can invoke it, and what it's allowed to do (see Permissions below).
- **Initiating events** — the trigger(s) configured.
- **Runtime and deployment package** — your application code plus its dependencies/libraries.
- **Configuration** — memory, timeout, environment variables, etc.

Lambda supports Node.js, Python, Ruby, Java, Go, .NET natively, plus **custom runtimes** for any other language.

### Ephemeral execution environments & concurrency

Each invocation runs in a temporary ("ephemeral") environment: created on demand, lives briefly, then torn down. You don't control these environments directly — Lambda manages their lifecycle.

**Concurrency** = the number of function invocations running at the same time. Lambda creates new environments on demand to keep up with invocation rate, which is how it scales horizontally so quickly. Factors that affect concurrency:

- **Reserved concurrency** — an optional per-function cap, carved out of (and limiting) the account's regional pool. Prevents one function from starving others, or from overwhelming a downstream system.
- **Regional quota** — a soft limit on total concurrent invocations across all functions in an account, per Region.
- **Burst quota** — a hard limit (not configurable) on how fast concurrency can ramp up during a sudden spike.
- **Request rate and function duration** — how fast requests arrive, combined with how long each one takes to run.
- **The event source** — different services manage concurrency of their own invocations differently.

---

## Invoking a Lambda function — push vs pull

Every event source invokes Lambda through one of two models:

### Push model
The event source directly invokes the function when the event occurs.

- **Synchronous push** — the caller waits for a response (e.g. API Gateway). No retry is built into Lambda for this path — your application code owns the retry strategy. Examples: Elastic Load Balancing, Cognito, Lex, Alexa, API Gateway, CloudFormation, CloudFront, Kinesis Data Firehose.
- **Asynchronous push** — Lambda queues the event, the event source gets an immediate success response once queued, and moves on ("send it and forget it"). If the function errors, Lambda automatically retries **up to twice**. Failed events can be routed to a dead-letter queue or an `OnFailure` destination. Examples: S3, SNS, SES, CloudFormation, CloudWatch Logs, CloudWatch Events, AWS Config.

### Pull (polling) model
The event source puts information into a stream or queue; **Lambda itself polls** that stream/queue and invokes the function with batches of records when it finds some.

- **Stream-based** (DynamoDB Streams, Kinesis Data Streams) — records organized into **shards**. Lambda polls a shard and invokes the function in batches. On a processing failure, Lambda **blocks further reads of that shard** until the failing batch is resolved (processed successfully or expires — streams retain records 24h). You can configure an `OnFailure` destination to bypass a permanently failing record for offline handling.
- **Queue-based** (SQS) — on failure, the message becomes visible again in the queue and is retried, up to a configurable retry limit or the message's retention period. Persistently failing messages can be routed to a **dead-letter queue (DLQ)**.
- Polling itself is free — you're only charged for actions that result from a poll.

The mapping between an event source and a function (for the pull model) is a **CreateEventSourceMapping** operation.

### Invoking manually, with an explicit invocation type

```bash
# Synchronous
aws lambda invoke --function-name my-function --payload '{...}' response.json
# → {"StatusCode": 200}

# Asynchronous
aws lambda invoke --function-name my-function --invocation-type Event --payload '{...}' response.json
# → {"StatusCode": 202}  (accepted, not "succeeded")
```

When an AWS service is the trigger, its invocation type (sync/async/poll) is predetermined — you don't get to choose it.

---

## Permissions — the two separate questions

Lambda permissions always answer two different questions, both handled through IAM:

```
Event source → [needs permission to invoke] → Lambda function → [needs permission to act] → AWS service/resource
```

### 1. Invocation permissions — who can call this function (resource policy)

An **IAM resource policy** attached to the Lambda function grants an event source the `lambda:InvokeFunction` action. Despite living "on" the function, conceptually it answers "who is allowed to invoke me" — and for a non-Lambda event source like S3, the equivalent permission can also live on the *other* resource (the bucket), not the function. This also enables cross-account invocation (account A's bucket invoking a function in account B).

### 2. Execution role — what this function can do (identity)

The **Lambda execution role** specifies what the function itself is permitted to do once running. It's an IAM role with two attached policies:

- **IAM policy** — defines the actions allowed on other AWS resources (e.g. write to a DynamoDB table, read from SQS, write CloudWatch logs). Answers "who can do what."
- **Trust policy** — allows the *Lambda service* to assume this role in the first place, regardless of who created it. Answers "who can assume this role." Lambda itself needs the `iam:PassRole` permission to attach an execution role to a function.

A polling event source (a stream or queue) also relies on the **execution role**, not a resource policy, to give the function permission to read from it.

---

## The function handler

```
handler(event, context)
```

The **handler** is the entry point Lambda calls to start your function. It always receives two objects:

- **Event object** — information about what triggered the function. Shape depends entirely on the event source: an API Gateway event includes path, query string, request body; an S3 event includes bucket and object details. Can also be a custom, user-defined object for testing.
- **Context object** — runtime information generated by AWS, varies slightly by language runtime, but at minimum includes:
  - `awsRequestId` — tracks a specific invocation (important for error reporting / AWS Support).
  - `logStreamName` — the CloudWatch log stream this invocation writes to.
  - `getRemainingTimeInMillis()` — milliseconds left before the function times out.

Each language has its own convention for defining and referencing the handler inside the deployment package (e.g. Python: `module_name.function_name`).

---

## Configuration options

### Performance-related

| Setting | What it controls |
|---|---|
| Memory | 128 MB – 10,240 MB. CPU is allocated proportionally to memory — more memory also means more CPU. |
| Timeout | 1 second – 15 minutes. Caps max run time; prevents runaway cost from long-running/stuck functions. |
| Concurrency | Default: 1,000 concurrent invocations per account per Region (soft limit). Can reserve a per-function subset. |
| Provisioned concurrency | Keeps a number of environments pre-warmed to avoid **cold starts** (the startup latency of initializing a brand-new environment). Priced separately. |
| Monitoring/operations | Enable X-Ray active tracing and/or CloudWatch Lambda Insights. |

Pricing is based on number of requests **and** duration — and the per-millisecond cost increases as memory increases. But more memory → more CPU → shorter duration, which can make a *higher* memory setting cheaper overall for CPU-bound work. Use a power-tuning utility to find the memory setting that minimizes total cost for a given function.

### Resource-related

| Setting | What it controls |
|---|---|
| Triggers | Event sources configured to invoke the function |
| Permissions | Who can invoke it / what it can access (see Permissions above) |
| Destinations | Where async invocation results go on success/failure (SNS topic, SQS queue, another Lambda, EventBridge bus) |
| Asynchronous invocation | Retry count (0–2) and max event age (up to 6h) for async invocations; DLQ configuration |
| VPC | Access to resources inside a private VPC (databases, caches, internal services) |
| State machines | Step Functions state machines that can invoke the function directly |
| Database proxies | RDS Proxy settings for connection pooling |
| File systems | EFS mount settings |

### Code-related

| Setting | What it controls |
|---|---|
| Runtime | Language/runtime version, or a custom runtime |
| Environment variables | Key-value pairs available to the function code without changing code — good for config and (non-secret or KMS-encrypted) sensitive values |
| Tags | Labels for cost tracking/filtering |
| Code signing | Ensures code came from an approved, unaltered source |

---

## Design & coding best practices

### Designing the function itself

- **Treat functions as stateless** — no in-memory state should be assumed to persist between invocations (the environment can be torn down any time).
- **Include only what you need** — minimize deployment package size and dependency count; don't bundle an entire SDK when you only need specific modules; smaller/simpler dependency trees reduce startup time.
- **Reuse the temporary runtime environment** — initialize expensive resources (DB connections, SDK clients) once, outside the handler, so warm invocations reuse them instead of re-initializing every call.

### Writing the code

- Separate core business logic from the handler method — more portable, easier to unit test without worrying about Lambda-specific configuration.
- Write modular, single-purpose functions.
- Include logging statements (written to CloudWatch).
- Return results information from the function so Lambda/the caller knows the outcome.
- Use environment variables for config instead of hardcoding (e.g. a bucket/table name).
- Avoid recursive code — a function invoking itself can spiral out of control via concurrency.
- Don't call one Lambda directly from another — use destinations or orchestrate with Step Functions instead.

---

## Deployment

Two deployment package types:

### .zip archive
- Choose a runtime when creating the function.
- Code + dependencies compressed and uploaded.
- Three ways to get code in: edit directly in the Lambda console editor (simple scenarios, no custom libraries beyond the AWS SDK), upload a package built in your IDE, or upload to an S3 bucket and point Lambda at it.
- Size limits: 50 MB compressed (direct upload), 250 MB uncompressed including layers, 3 MB via the console editor, 512 KB per individual file.

### Container image
- Choose a runtime and Linux distribution when building the image.
- Package code + dependencies as a container image, push to **Amazon ECR**.
- On create/update, Lambda pulls the image from ECR, optimizes it, and deploys — the function then moves from `PENDING` to `ACTIVE` status, ready to invoke.

### CLI examples

```bash
aws lambda create-function \
  --function-name my-function --runtime nodejs10.x \
  --zip-file fileb://my-function.zip --handler my-function.handler \
  --role arn:aws:iam::123456789012:role/service-role/MyTestFunction-role

aws lambda update-function-code \
  --function-name my-function --zip-file fileb://my-function.zip
```

### Versioning and aliases

- A **version** is an immutable snapshot of a function's code + configuration. Only `$LATEST` exists until you explicitly **publish** a version (creates version `1`, `2`, ... each with its own ARN).
- An **alias** is a mutable, named pointer to a specific version (e.g. `DEV`, `TEST`, `PROD`), with its own ARN. Promoting or rolling back just means repointing the alias — no code change, and critically, **no need to update the event source's configuration** (an S3 notification, an API Gateway integration, etc.) every time a new version ships, since it can target the alias ARN instead of a version-specific ARN.

### Layers

A **layer** is a .zip archive of libraries, a custom runtime, or other shared dependencies, attachable to a function without bundling them into its own deployment package.

- Up to **5 layers** per function.
- Combined extracted size of function + all layers capped at **250 MB**.
- Keeps deployment packages small (easier console editing — under 3 MB qualifies for Node.js/Python/Ruby) and lets multiple functions share one dependency package instead of duplicating it in each.
- AWS publishes a public layer with NumPy/SciPy for Python workloads.

### Custom runtimes

Let you use a language Lambda doesn't support natively. Lambda exposes a **runtime interface** for getting events and sending responses; the **bootstrap file** is the runtime's entry point. A custom runtime can be deployed alongside the function code or shipped as its own layer.

---

## Monitoring and debugging

Modern (especially serverless) applications challenge traditional monitoring because of: short-lived resources, many interconnected services/devices, faster release velocity, and the user-experience dimension (latency matters, not just uptime).

### CloudWatch

Collects monitoring/operational data as logs, metrics, and events:

- A unified view across AWS resources, applications, and services.
- Logs reviewable as a time-ordered flow of events.
- **CloudWatch Logs Insights** for interactively querying log data.
- Default operational metrics from AWS services out of the box.
- Alarms on metric thresholds, able to trigger automated actions.

**Lambda logging structure**: every function gets its own log group, `/aws/lambda/<function-name>` (created on first invocation, or when provisioned concurrency is configured). Inside, **log streams** are organized by date, function version, and a unique ID. Each invocation produces a unique `RequestId` and three key log lines:

- `START` — invocation begins.
- `END` — invocation ends.
- `REPORT` — summary: billed duration, configured memory size, actual memory used, and (when present) `Init Duration` — which flags that this invocation paid the cold-start cost of a brand-new environment.

### AWS X-Ray

Traces requests as they travel across your application's components and renders a service map.

- One-click enablement for Lambda and integration with other AWS services.
- The X-Ray SDK captures metadata for AWS SDK API calls your code makes.
- Visual detection of latency distribution; quickly isolates outliers/trends.
- Filtering and grouping of traces by error type.

**Trace structure**:
- **Segment** — data about the work done by one compute resource: resource name, request details, subsegments.
- **Subsegment** — finer-grained breakdown within a segment: more granular timing, details of downstream calls (a DB query, an external API, an AWS service), can be defined around specific functions or code blocks.
- **Annotations** — indexed key-value pairs, usable in filter expressions to group/search traces in the console.
- **Metadata** — key-value pairs of any type, **not** indexed, for extra trace context you won't search on.

---

## Applied: a real event-driven Lambda (Fraud Detection System)

Walkthrough of a handler built from first principles — API Gateway event in, validation, server-generated identifiers, a DynamoDB write, and a well-formed HTTP response out. This progression matches a set of practice exercises worked through before writing the real function.

### 1. Parsing the event

API Gateway's proxy integration always hands the HTTP body to the function as a **JSON string** inside `event["body"]` — never already parsed:

```python
body = json.loads(event["body"])
amount = body.get("amount")
merchant = body.get("merchant")
```

### 2. Defensive extraction with `.get()`

Using `body["user_id"]` would raise `KeyError` if the client omits the field. `.get(key, default)` degrades gracefully instead:

```python
user_id = body.get("user_id", "unknown")  # "unknown" when absent, instead of crashing
```

In this project, `user_id` was allowed to default to `"unknown"` only temporarily, before authentication (Cognito) existed — a deliberate, time-boxed shortcut, not a long-term pattern.

### 3. Never trust the client for identity or time

`transaction_id` and the write timestamp are generated **server-side**, never read from the request body — the server is the source of truth for identity and time, and trusting client-supplied values here would let a client fabricate or collide IDs, or backdate/forge timestamps:

```python
transaction_id = str(uuid.uuid4())
timestamp = datetime.now(timezone.utc).isoformat(timespec="seconds").replace("+00:00", "Z")
```

### 4. Building the DynamoDB key structure inline

```python
pk = f"USER#{user_id}"                       # groups everything for this user into one partition
sk = f"TXN#{timestamp}#{transaction_id}"      # ISO8601 first = chronological sort; ID last = uniqueness tie-breaker
```

(Full rationale for this PK/SK shape lives in the DynamoDB schema notes — the point here is simply that it's assembled with plain Python f-strings, no special library needed.)

### 5. Validating before writing

```python
if amount is None or merchant is None:
    return build_response(400, {"error": "amount and merchant are required"})
```

Fail fast with a clear 4xx before touching DynamoDB — don't let a malformed request reach `put_item`.

### 6. The actual write

```python
table.put_item(Item=item)
```

This is the one line that distinguishes the "real" Lambda from the pure-logic practice version: everything before it (parsing, validation, ID/timestamp generation, item construction, response formatting) can and should be fully exercised and tested *without* touching DynamoDB at all — boto3 is the last piece added once the surrounding logic is already correct.

### 7. A valid API Gateway response, every time

```python
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
```

API Gateway requires at minimum `statusCode` and `body`, and `body` **must be a JSON string**, not a raw dict — returning a dict directly for `body` is a common first mistake. CORS headers are included explicitly here rather than relying solely on API Gateway's CORS configuration (see the DynamoDB debugging notes for why that distinction matters with Lambda proxy integration).

### Practice progression this handler was built from

The exercises worked through, in order, before writing this for real: (1) parse an API Gateway event body, (2) handle a missing key gracefully with `.get()`, (3) build the PK/SK strings, (4) generate a real ISO 8601 UTC timestamp, (5) generate a UUID, (6) assemble the full DynamoDB item dict, (7) write `build_response()`, (8) assemble a complete `lambda_handler` structure — validation and response wiring only, deliberately **without** boto3 yet. The real function then added exactly one new piece on top of exercise 8: the actual `table.put_item(Item=item)` call.

---

## References

- Lambda Runtimes: https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html
- Configuring reserved concurrency: https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html
- Lambda quotas: https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html
- Lambda function scaling (burst limits): https://docs.aws.amazon.com/lambda/latest/dg/invocation-scaling.html
- CreateEventSourceMapping: https://docs.aws.amazon.com/lambda/latest/dg/API_CreateEventSourceMapping.html
- Using Lambda with Kinesis: https://docs.aws.amazon.com/lambda/latest/dg/with-kinesis.html
- Using Lambda with SQS: https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html
- Lambda layers: https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html
- AWS Lambda Power Tuning: https://github.com/alexcasalboni/aws-lambda-power-tuning
- AWS X-Ray concepts: https://docs.aws.amazon.com/xray/latest/devguide/xray-concepts.html
- X-Ray segment documents: https://docs.aws.amazon.com/xray/latest/devguide/xray-api-segmentdocuments.html
- X-Ray console filter expressions: https://docs.aws.amazon.com/xray/latest/devguide/xray-console-filters.html
- CloudWatch Logs for Lambda: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs.html
