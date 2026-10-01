# Amazon API Gateway — Concepts & Notes

Theory reference for Amazon API Gateway: what an API is, what API Gateway does, the API types it supports, how it integrates with backends, and how to deploy, secure, monitor, and optimize an API. Based on official AWS Academy course content ("Developing REST APIs").

---

## 1. What is an API?

An API (Application Programming Interface):

- **Abstracts implementation** — the consumer doesn't need to know how the provider does its work internally, only what it exposes.
- **Exposes only needed objects/actions** — a deliberate, controlled surface, not the whole system.
- **Establishes a communication protocol** between a provider and a consumer.

A typical request flow: client → web server (API) → database, with the API translating client requests into backend operations and backend responses back into client-facing responses.

### Protocols

APIs can use different underlying protocols: SOAP, WebSockets, HTTP, HTTPS.

### Status code categories

```
20x  Success
40x  Client error   (bad request, auth failure, not found, rate-limited...)
50x  Server error   (backend failure, timeout...)
```

### REST vs RESTful

- **REST** (Representational State Transfer) is an **architectural style** — a set of constraints/principles for designing networked APIs (statelessness, resource-based URIs, standard HTTP verbs, etc.).
- **RESTful** describes an API that **adheres to REST principles**. "REST API" and "RESTful API" are often used interchangeably in practice.

---

## 2. Introducing API Gateway

### API Gateway as a proxy

An API gateway sits in front of backend services and acts as a protective, controlling layer — conceptually similar to a nightclub doorman:

- Rate limiting (controls how many people/requests get in).
- Developer key validation (checks ID before letting a request through).
- Geo-routing (routes requests based on origin).
- Discards invalid requests before they ever reach the backend.

### What Amazon API Gateway is

A **fully managed, serverless** service that acts as the **"front door"** for backend services. It can front:

- AWS Lambda functions
- Amazon EC2
- Any other AWS service
- VPC endpoints (private resources)
- On-premises systems (via AWS Direct Connect)

**Pricing**: pay per API call received, plus data transferred out — no servers to provision or manage.

### Supported protocols / API types

API Gateway supports two broad protocol families:

- **RESTful** — request/response model (REST APIs and HTTP APIs)
- **WebSockets** — persistent, bidirectional connections

#### REST API vs HTTP API (both are "RESTful" in API Gateway)

| | REST API | HTTP API |
|---|---|---|
| Feature set | Full control, advanced features (mock endpoints, fine-grained request/response transformation, usage plans, API keys, resource policies, client-side SDK generation) | Lightweight, proxy-focused |
| Latency/cost | Higher | Lower latency, **up to ~70% cheaper** |
| Use case | Need advanced features, monetization (usage plans/API keys), or private VPC-only APIs | Simple proxy pass-through to Lambda/HTTP backend |

**Guidance**: default to HTTP API unless you specifically need a REST-API-only feature.

#### WebSocket APIs

- **Bidirectional** — either side can push a message at any time, unlike request/response REST.
- Efficient for **"chatty"**, real-time applications (chat apps, live dashboards, multiplayer features) — avoids the overhead of a full HTTP request/response envelope for every small message.
- **WSS** = the secure (TLS) variant of the WebSocket protocol.
- Requires defining route keys: `$connect`, `$disconnect`, `$default` (plus custom routes based on a route selection expression, e.g. `$request.body.action`).

### Managing an API

- **Develop**: AWS Management Console, AWS CLI, or by importing/exporting an OpenAPI (Swagger) definition.
- **Deploy**: AWS SAM, AWS CloudFormation, or AWS CDK.

---

## 3. Creating a REST API

### Base URI structure

```
https://{restapi_id}.execute-api.{region}.amazonaws.com/{stage_name}/
```

Every deployed API is reachable at this auto-generated URL unless a custom domain is configured.

### Routes and methods

A route combines an HTTP method with a resource path, e.g.:

```
GET  /products          → list all products
GET  /products/{id}     → get one product
PUT  /products/{id}     → update one product
```

Each route is wired to a backend integration (a Lambda function, an HTTP endpoint, an AWS service, etc.).

### Greedy path variable: `{proxy+}`

A catch-all route (e.g. `ANY /{proxy+}`) that matches any path and method combination, forwarding everything to a single integration — commonly a Lambda function doing its own internal routing. Avoids having to define every single route explicitly in API Gateway.

### Mock endpoints

A mock integration returns a canned response directly from API Gateway, without ever calling a real backend. Useful for:

- Prototyping a frontend before the backend exists.
- Handling CORS preflight (`OPTIONS`) requests automatically.

### Importing/exporting API definitions

REST and HTTP APIs can be defined via an OpenAPI/Swagger document and imported wholesale, or exported from an existing API for version control / portability.

---

## 4. Integrating with API Gateway

### Endpoint types

| Type | Scope | Notes |
|---|---|---|
| **Edge-optimized** | REST APIs only | Routed through a CloudFront edge location automatically; built-in DDoS protection. Best for geographically distributed clients. |
| **Regional** | REST, HTTP, WebSocket | Served from a single Region. Recommended best practice: put your own CloudFront distribution in front for CDN benefits + more control. |
| **Private** | REST APIs only | Reachable only from within a VPC, via a VPC endpoint. |

### Client-side SDK generation

API Gateway can generate client SDKs (JavaScript, Android, iOS, Ruby, Java) for REST APIs. **Caution**: a client-side SDK typically needs client-side AWS credentials, which is a security concern — plain HTTPS calls are generally preferred unless the SDK is genuinely needed.

### Backend integration types

1. **Lambda proxy integration** (`AWS_PROXY`) — API Gateway passes the entire raw request through to Lambda, and Lambda's response is returned as-is to the client. The Lambda function **must** return the exact expected shape:

```json
{
  "statusCode": 200,
  "headers": { "Content-Type": "application/json" },
  "body": "{\"message\": \"ok\"}",
  "isBase64Encoded": false
}
```

   Example 403 error response from Lambda proxy integration:
```json
{
  "statusCode": 403,
  "headers": { "Set-Cookie": "session=abc123" },
  "body": "{\"error\": \"Forbidden\"}"
}
```

2. **Lambda non-proxy (custom) integration** — API Gateway and Lambda exchange data through **mapping templates** (VTL — Velocity Template Language), letting API Gateway transform the request/response shape between client and backend. More setup, but decouples the API's public contract from the Lambda's internal data shape.

   Example VTL mapping template use: renaming/restructuring incoming JSON fields before they reach Lambda, and again on the way back out.

3. **First-class AWS service integrations** — API Gateway can call certain AWS services directly (e.g. SQS `SendMessage`, Step Functions `StartExecution`) without an intermediate Lambda function — removes a layer of compute entirely for simple pass-through use cases.

4. **HTTP proxy integration** — forwards requests to another HTTP(S) endpoint (e.g. an existing EC2-based backend or external API). Common pattern when **migrating a monolith to microservices**: route specific paths to new Lambda-backed routes while the rest of the traffic still goes to the legacy backend, incrementally.

5. **Private integrations via VPC Link** — lets API Gateway reach resources inside a VPC that have no public endpoint (e.g. an internal Application Load Balancer), without exposing them to the internet directly.

### Example integration request/response flow

```
Client → Method Request → Integration Request → Backend
                                                     ↓
Client ← Method Response ← Integration Response ← (backend response)
```

Each stage (method request/response, integration request/response) is a point where API Gateway can validate, transform, or reject the data.

### Parameter mapping (HTTP APIs)

A lighter-weight alternative to VTL mapping templates (REST APIs), used on HTTP APIs to append, overwrite, or remove headers/query strings/path parameters as a request passes through — e.g. injecting `$context.requestId` as a header before forwarding to the backend.

---

## 5. Deploying an API

### Stages are deployment snapshots

A **stage** is a **named, immutable snapshot of a specific API version/configuration** — e.g. `dev`, `beta`, `prod`. Stages are:

- **Required** for REST and WebSocket APIs — you must explicitly create a deployment to a stage before the API becomes reachable at that stage's URL.
- **Automatic** for HTTP APIs — deploys to a default stage, and can be configured to auto-deploy on every change.

The stage name appears in the invoke URL: `.../execute-api.../{stage_name}/...`.

### Using stages as environments

Different stages can represent different environments (dev vs prod), each potentially pointing at a different backend version — e.g. a `dev` stage invoking a `DEV` Lambda alias while `prod` invokes the `PROD` alias.

### Stage variables

A **stage variable** is a key/value pair scoped to one stage, referenced inside an integration's configuration (e.g. `${stageVariables.lambdaAlias}`). This lets the *same* API definition serve different backends per stage — e.g. mapping a stage variable `lambdaAlias` to different Lambda aliases (`dev`, `beta`, `prod`) without duplicating the API itself.

### Canary deployments

A canary deployment sends a small percentage of **production** traffic to a new version while the rest continues to hit the stable version — lets you catch problems with real traffic before a full rollout.

- Configure `percentTraffic` (e.g. 10.5%) and optional `stageVariableOverrides` for the canary slice.
- Monitor results via CloudWatch.
- Gradually increase the percentage as confidence grows.
- **Promoting** a canary makes it the new 100%-traffic production version; the canary configuration is then removed.

---

## 6. Controlling access to a REST API

API Gateway supports multiple, layered ways to protect an API from unwanted traffic, and multiple ways to authorize legitimate requests.

### Protecting from unwanted traffic

| Mechanism | What it does |
|---|---|
| **IAM resource policies** | Restrict who/what can call the API's endpoint — e.g. allow only a specific AWS account, or only requests originating from a specific office IP range. REST APIs only; HTTP APIs cannot use resource policies this way. |
| **Custom domain + client certificates (mTLS)** | A custom domain (SSL via ACM) lets you then issue client-side certificates (expire after 365 days) so the backend can cryptographically verify that a request really came through API Gateway (or from a specific trusted IoT device) — mutual TLS, only possible with a custom domain. |
| **AWS WAF** | A web application firewall deployable in front of API Gateway (or CloudFront). Protects against common exploits (SQL injection, XSS), matches string/regex patterns in headers/body/URIs, and can block traffic by IP range, country/region, or known bad user agents/bots. Supports AWS-managed, pre-configured, regularly updated rule sets. |
| **CORS (Cross-Origin Resource Sharing)** | A **browser-enforced** security feature restricting cross-origin AJAX requests (different domain, subdomain, port, or protocol all count as "cross-origin"). "Simple" requests like plain GETs are exempt. Non-simple cross-origin requests trigger a **preflight** `OPTIONS` request first, which the server must explicitly approve (allowed headers/methods/origins) before the real request is sent. API Gateway can handle the preflight automatically (a mock endpoint is set up behind the scenes for REST APIs; simpler/built-in for HTTP APIs), removing this burden from your own backend code. |
| **Throttling** | Protects against rate-based attacks/overload. All APIs in an account share a Region-wide soft quota (default **10,000 requests/second**, raisable on request). Throttling can be layered more granularly: per stage, per method/route, and per client via a **usage plan** — settings apply from most to least granular (client+method → client → method/stage → account). A throttled request returns **HTTP 429 "Too Many Requests"**. |

### Usage plans (REST APIs)

A usage plan specifies **which clients** (identified by API key) can access which deployed stages/methods, and sets **rate limits** and **quotas** (e.g. max requests per day/week/month) per client. Supports different plan tiers (e.g. "User", "Partner", "Service" usage plans) with different limits.

### Authorizing API access (who is allowed to call the API at all)

Four options:

1. **None** — fully open/public access.
2. **IAM** — the caller signs requests with AWS credentials (typically via a client-side SDK using temporary credentials). An unsigned request is rejected with a permissions error.
3. **JWT-based (JSON Web Token)** — used behind OpenID Connect (OIDC) or OAuth 2.0 flows.
   - REST APIs: typically use **Amazon Cognito** user pools as the JWT authorizer.
   - HTTP APIs: can use Cognito or other third-party JWT identity providers.
   - API Gateway validates the token, checks scope, and determines whether the caller may access the requested resource. Authorizers can be configured per-route or shared across routes.
4. **Lambda authorizers** — a Lambda function that runs *before* the real integration, validating a bearer token (e.g. custom OAuth, SAML) or other request parameters against custom logic.
   - Without authorization info, the end-target function just receives the original event.
   - With a Lambda authorizer in front, the end-target function additionally receives an `isAuthorized: true/false` boolean (and any custom `context` data) — letting your own code handle a rejection gracefully instead of API Gateway returning an abrupt 40x.
   - Writing custom Lambda authorizer code is implementation-specific; AWS provides examples via the Developer Guide, GitHub, and the Lambda console.

---

## 7. Monitoring a REST API

### Default CloudWatch metrics (sent automatically every minute, no setup required)

| Metric | Meaning |
|---|---|
| `Count` | Total API calls in a given period |
| `IntegrationLatency` | Backend request/response time (how responsive the backend itself is) |
| `Latency` | Full round-trip time: from API Gateway receiving the request to it returning a response to the client |
| `4XXError` / `5XXError` | Client-side / server-side error counts |

Metrics are retained for **15 months**, enabling historical/trend analysis.

### Access logs (optional, configured per stage)

Unlike metrics, access/error logs are **opt-in** and configured per stage:

```
1. Create a CloudWatch log group.
2. Point the stage at that log group via its access-log settings, with a chosen format
   (Common Log Format or JSON), including fields like source IP, request time,
   HTTP method, route, status, response length, and request ID.
```

API Gateway automatically creates log groups/streams for **error** logs; access logs require this explicit stage configuration.

### Other monitoring tools

| Tool | Answers |
|---|---|
| **AWS X-Ray** | "What's the breakdown of this individual transaction as it passed through my application?" / "Where are problems occurring across my distributed app?" Combines trace data from every service a request touched into one unit called a **trace**. Gives visual latency-distribution breakdowns, isolates outliers/trends, and lets you filter/group requests by error type — can be enabled directly on API Gateway. |
| **AWS Config** | "Does this resource's configuration comply with our rules?" / "How do resources relate to each other?" Provides a normalized configuration snapshot and lets you define compliance rules; non-compliant resources get flagged and can trigger an SNS notification. |
| **AWS CloudTrail** | "Which resources were modified, by whom, and when?" Enabled by default on every account — records every API Gateway-related API call (console, SDK, CLI) as a searchable, downloadable CloudTrail event, including requester identity, call time, parameters, and response. |

---

## 8. Optimizing API Gateway

### Caching (REST APIs)

API Gateway can cache responses to GET requests so repeat requests are served without hitting the backend at all:

- Reduces calls to the backend and improves response latency.
- Cache keys can incorporate path, headers, and query strings.
- Configurable **per stage or per method**.
- Cached values have a configurable **TTL**; cache data encryption can be enabled.
- **Caching is billed hourly whether it's actually used or not** — a fixed cost decision, not pay-per-use.
- Metrics: `CacheHitCount` (served from cache) and `CacheMissCount` (served from backend, cache enabled but missed).

### Payload compression

Reduces the size of data transferred between client and API Gateway, lowering both cost and latency:

- Supported content codings: `deflate`, `gzip`, `identity`.
- A `minimumCompressionSize` setting controls when compression kicks in — setting it to `0` compresses **all** payloads.
- API Gateway supports **decompression of request payloads by default**; response payload compression must be explicitly configured.
- Compressing very small payloads can actually *increase* their size, and compression/decompression adds compute time on both ends — test to find the right threshold rather than assuming "always compress" is optimal.

---

## 9. Worked example scenario (course lab)

A café web application (hosted on S3) needs its frontend to read/write data from a DynamoDB table (`FoodProducts`) through an API, rather than talking to DynamoDB directly from client-side JavaScript. Development approach:

1. Build the API with **mock endpoints** first (prototype without a real backend).
2. Get stakeholder approval on the mocked behavior.
3. Later, swap the mock integrations for real integrations (Lambda reading/writing DynamoDB) without changing the API's public shape — demonstrates the value of API Gateway as a stable "front door" that can be re-pointed at different backends over time.

---

## References

- API Gateway Developer Guide: https://docs.aws.amazon.com/apigateway/latest/developerguide/
- Setting Up CloudWatch Logging for a REST API: https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-logging.html
- Monitoring REST API Execution with Amazon CloudWatch Metrics: https://docs.aws.amazon.com/apigateway/latest/developerguide/monitoring-cloudwatch.html
- Configuring Logging for an HTTP API: https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-logging.html
- Enable Payload Compression for an API: https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-enable-compression.html
- Setting Up Custom Domain Names (REST/HTTP/WebSocket): https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-custom-domains.html
