# AWS SAM — Command Reference

Practical reference for the SAM CLI.
---

## Core commands

```bash
sam init      # scaffolds a new SAM project's base structure
sam build     # compiles/prepares the code and dependencies before deployment
sam deploy    # deploys the stack to AWS (creates/updates resources)
sam local     # simulates Lambda/API Gateway locally, without deploying
sam validate  # checks the template's syntax before deployment
```

```bash
# Typical full cycle for a change
sam build
sam validate
sam deploy
```



## Testing a function — two approaches

### Approach 1 — `sam local invoke` (fast, no deployment needed)

```bash
sam build
sam local invoke <FunctionLogicalName> --event events/test-event.json --profile <profile-name>
```

Simulates the Lambda execution **locally**, but still calls the **real**
AWS services it depends on (e.g. DynamoDB) — this is not a full offline
simulation. Local AWS credentials must be available, hence `--profile`.

A test event file only needs the fields the function's code actually reads
— for a function triggered via API Gateway that only reads `event["body"]`,
a minimal file is enough:

```json
{
  "body": "{\"amount\": 5200, \"merchant\": \"Best Buy\", \"user_id\": \"42\"}"
}
```

### Approach 2 — real deployment + `curl`

```bash
sam deploy
curl -X POST https://<api-id>.execute-api.<region>.amazonaws.com/Prod/<path> \
  -H "Content-Type: application/json" \
  -d '{"amount": 5200, "merchant": "Best Buy", "user_id": "42"}'
```

Tests the function through the real, deployed API Gateway endpoint rather
than a local simulation.

### Verifying the result directly in DynamoDB

```bash
aws dynamodb scan --table-name <table-name> --profile <profile-name>
```



## Running the API locally (full local simulation)

```bash
sam build
sam local start-api
```

Starts a local API Gateway simulation on `http://127.0.0.1:3000`, routing
requests to the corresponding Lambda functions exactly as the real deployed
API would — useful for iterating quickly without redeploying after every
code change.



sam local invoke calls a single Lambda function, once, with the event you give it (test-event.json). No API Gateway, no server: it runs the handler in a Docker container, prints the result, and exits. Useful for testing a function's logic in isolation, or a Lambda triggered by something other than HTTP (SQS, S3, EventBridge…).

sam local start-api starts a local HTTP server (port 3000) that simulates API Gateway. It stays running, and each request (curl, Postman, your Next.js frontend) is routed to the right Lambda based on the Events: Api in your template.yaml. API Gateway builds the event from the HTTP request, so you don't have to write it yourself. Useful for testing endpoints, routing, methods, and paths.

## Other useful commands

```bash
sam logs --name <FunctionLogicalName> --tail       # stream a function's CloudWatch logs
sam delete                                          # deletes the entire stack and its resources
sam list stack-outputs                              # shows the current stack's Outputs values
sam list resources                                  # lists every resource in the deployed stack
```

## References

- [SAM CLI command reference](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference.html)
- [`sam local invoke`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-local-invoke.html)
- [`sam local start-api`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-local-start-api.html)
- [API Gateway Lambda proxy integration — full input format](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-input-format)
