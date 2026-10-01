# API Gateway — Commands Reference

Practical reference: AWS CLI commands for creating, deploying, integrating, securing, and monitoring APIs with Amazon API Gateway (mostly HTTP APIs via `apigatewayv2`, with REST API-specific commands via `apigateway` where noted). Identifiers (API IDs, ARNs, domains) are placeholders — replace with your own.

---

## Creating an API

### Create an HTTP API (lightweight, proxy-only, cheapest)

```bash
aws apigatewayv2 create-api \
  --name my-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:region:account-id:function:my-function
```

Using `--target` with a Lambda ARN is a quick-create shortcut: it auto-creates a `$default` route and stage pointing at that Lambda.

### Create a WebSocket API

```bash
aws apigatewayv2 create-api \
  --name my-websocket-api \
  --protocol-type WEBSOCKET \
  --route-selection-expression '$request.body.action'
```

A WebSocket API requires three special route keys to be defined: `$connect`, `$disconnect`, `$default`.

### Create a REST API (full control, advanced features)

```bash
aws apigateway create-rest-api --name my-rest-api
```

---

## Routes and methods

### Create a route (HTTP/WebSocket APIs)

```bash
aws apigatewayv2 create-route \
  --api-id api-id \
  --route-key 'GET /products/{product_id}' \
  --target integrations/integration-id
```

### Greedy path variable (catch-all routing)

```
ANY /{proxy+}
```

`{proxy+}` forwards any path/method combination to a single backend integration (typically used with Lambda proxy integration) — lets one Lambda handle an entire API surface without defining every route explicitly.

### Importing / exporting API definitions (OpenAPI/Swagger)

```bash
# Import a new API from an OpenAPI/Swagger definition
aws apigatewayv2 import-api --body file://api-definition.json

# Export an existing REST API's definition
aws apigateway get-export \
  --rest-api-id my-api-id \
  --stage-name prod \
  --export-type swagger \
  swagger-export.json
```

---

## Integrations

### Lambda proxy integration (first-class, simplest)

```bash
aws apigatewayv2 create-integration \
  --api-id api-id \
  --integration-type AWS_PROXY \
  --integration-uri arn:aws:lambda:region:account-id:function:my-function \
  --payload-format-version 2.0
```

With `AWS_PROXY`, API Gateway passes the full request through untouched, and the Lambda function must return the exact expected JSON shape (`statusCode`, `headers`, `body`, `isBase64Encoded`).

### First-class service integration (e.g. SQS, no Lambda needed)

```bash
aws apigatewayv2 create-integration \
  --api-id api-id \
  --integration-type AWS_PROXY \
  --integration-subtype SQS-SendMessage \
  --request-parameters '{"QueueUrl":"$request.header.queueUrl","MessageBody":"$request.body.message"}'
```

Lets API Gateway call certain AWS services (SQS, Step Functions, EventBridge, etc.) directly — no Lambda function in between.

### HTTP proxy integration (forward to another HTTP backend, e.g. legacy EC2/monolith)

```bash
aws apigatewayv2 create-integration \
  --api-id api-id \
  --integration-type HTTP_PROXY \
  --integration-uri https://my-backend.example.com/path \
  --integration-method ANY
```

Common in monolith-to-microservices migrations: route specific paths to a new Lambda while everything else still goes to the existing EC2/backend.

### Private integration via VPC Link (reach a resource inside a VPC, e.g. internal ALB)

```bash
aws apigatewayv2 create-integration \
  --api-id api-id \
  --integration-type HTTP_PROXY \
  --connection-type VPC_LINK \
  --connection-id vpc-link-id \
  --integration-uri arn:aws:elasticloadbalancing:region:account-id:listener/app/my-alb/...
```

### Parameter mapping (HTTP APIs) — modify requests on the way through

```bash
aws apigatewayv2 update-integration \
  --api-id api-id \
  --integration-id integration-id \
  --request-parameters '{"append:header.header1": "$context.requestId"}'
```

Supports `append:`, `overwrite:`, and `remove:` operations on headers, query strings, and path parameters. Does not require a VTL template — a lighter-weight alternative to REST API mapping templates.

---

## Permissions (letting API Gateway invoke a Lambda)

```bash
aws lambda add-permission \
  --function-name my-function \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:region:account-id:api-id/*/*/products"
```

---

## Stages and deployment

### REST API: create a deployment to a stage

```bash
aws apigateway create-deployment \
  --rest-api-id my-api-id \
  --stage-name prod
```

REST and WebSocket APIs require an explicit deployment to a named stage before the API is reachable. HTTP APIs deploy to a default stage automatically (auto-deploy).

### Stage variables (point a stage at a different backend, e.g. a Lambda alias)

```bash
aws apigateway update-stage \
  --rest-api-id my-api-id \
  --stage-name prod \
  --patch-operations op=replace,path=/variables/lambdaAlias,value=PROD
```

The stage variable (e.g. `lambdaAlias`) is then referenced inside the integration URI as `${stageVariables.lambdaAlias}`, letting `dev`/`beta`/`prod` stages each invoke a different Lambda alias without changing the API definition.

### Canary deployment (shift a percentage of traffic to a new version)

```bash
aws apigateway create-deployment \
  --rest-api-id my-api-id \
  --stage-name prod \
  --canary-settings '{
    "percentTraffic": 10.5,
    "useStageCache": false,
    "stageVariableOverrides": {"lambdaAlias": "PROD_NEW"}
  }'
```

### Update (increase) canary traffic percentage

```bash
aws apigateway update-stage \
  --rest-api-id my-api-id \
  --stage-name prod \
  --patch-operations op=replace,path=/canarySettings/percentTraffic,value=25.0
```

Promoting a canary (making it the new 100% production version) is done by updating the base stage to match the canary's settings, then removing the canary.

---

## Access control

### Custom domain

See "Setting Up Custom Domain Names for REST/HTTP/WebSocket APIs" in the Developer Guide — SSL certificate managed via AWS Certificate Manager (ACM), supports wildcard domains and multiple domains via base path mapping. Required before client certificates (mTLS) can be used.

### Creating an access log group + enabling access logging on a stage

```bash
aws logs create-log-group --log-group-name my-api-log-group

aws apigatewayv2 update-stage \
  --api-id api-id \
  --stage-name '$default' \
  --access-log-settings '{
    "DestinationArn": "arn:aws:logs:region:account-id:log-group:my-api-log-group",
    "Format": "$context.identity.sourceIp - [$context.requestTime] \"$context.httpMethod $context.routeKey $context.protocol\" $context.status $context.responseLength $context.requestId"
  }'
```

### JWT authorizer (e.g. Amazon Cognito user pool)

```bash
# 1. Create the authorizer
aws apigatewayv2 create-authorizer \
  --name my-cognito-authorizer \
  --api-id api-id \
  --authorizer-type JWT \
  --identity-source '$request.header.Authorization' \
  --jwt-configuration Audience=my-app-client-id,Issuer=https://cognito-idp.region.amazonaws.com/my-user-pool-id

# 2. Attach it to a route
aws apigatewayv2 update-route \
  --api-id api-id \
  --route-id route-id \
  --authorization-type JWT \
  --authorizer-id authorizer-id \
  --authorization-scopes user.id user.email
```

### Throttling — account/stage/method rate limits

```bash
aws apigateway update-stage \
  --rest-api-id my-api-id \
  --stage-name prod \
  --patch-operations op=replace,path=/*/*/throttling/rateLimit,value=500
```

Account-level default is a soft limit of 10,000 requests/second, shared across all APIs in the account/Region. Per-stage/per-method limits can be set lower than the account limit, never higher.

### Payload compression

```bash
aws apigateway update-rest-api \
  --rest-api-id my-api-id \
  --patch-operations op=replace,path=/minimumCompressionSize,value=0
```

`minimumCompressionSize=0` compresses all payloads; a higher byte threshold compresses only payloads above that size. Supported codings: `deflate`, `gzip`, `identity`.

---

## Monitoring

```bash
# Tail API Gateway execution logs (if logging enabled, typically via CloudWatch Logs)
aws logs tail /aws/apigateway/my-api-log-group --follow

# Enable X-Ray tracing on a REST API stage
aws apigateway update-stage \
  --rest-api-id my-api-id \
  --stage-name prod \
  --patch-operations op=replace,path=/tracingEnabled,value=true
```

Default CloudWatch metrics (sent automatically, no setup needed): `Count`, `4XXError`, `5XXError`, `Latency` (client round trip), `IntegrationLatency` (backend response time only).

---

## Quick reference — status code categories

```
20x  Success
40x  Client error  (e.g. 403 Forbidden, 429 Too Many Requests = throttled)
50x  Server error
```

---

## References

- API Gateway Developer Guide: https://docs.aws.amazon.com/apigateway/latest/developerguide/
- Setting Up CloudWatch Logging for a REST API: https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-logging.html
- Configuring Logging for an HTTP API: https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-logging.html
- Enable Payload Compression for an API: https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-enable-compression.html
- Setting Up Custom Domain Names for REST APIs: https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-custom-domains.html
