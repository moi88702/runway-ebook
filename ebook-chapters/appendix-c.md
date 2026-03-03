# Appendix C: SST Component Reference Cheat Sheet

Quick reference for every SST component used in this book. All examples assume SST v3 with the `sst` package and TypeScript.

Full docs: [sst.dev/docs](https://sst.dev/docs)

---

## sst.aws.Function

Lambda function with sensible defaults (Node.js runtime, bundled with esbuild, X-Ray optional).

```typescript
const fn = new sst.aws.Function("MyFunction", {
  handler: "src/handlers/my-function.handler",
  runtime: "nodejs22.x",              // default
  timeout: "30 seconds",              // default 10s, max "900 seconds"
  memory: "1024 MB",                  // default 1024 MB
  tracing: "active",                  // "active" | "passthrough" | "disabled"
  vpc,
  link: [table, bucket, queue, secret],
  permissions: [
    { actions: ["ses:SendEmail"], resources: ["*"] },
  ],
  environment: {
    LOG_LEVEL: "INFO",
    SST_STAGE: $app.stage,
  },
  nodejs: {
    install: ["pg"],                  // force-include packages esbuild might tree-shake
    esbuild: {
      external: ["@prisma/client"],   // exclude from bundle (e.g. Prisma with binaries)
    },
  },
  reservedConcurrency: 100,           // hard limit on concurrent executions
});
```

Access in linked resources: `Resource.MyFunction.name` (function name), `Resource.MyFunction.arn`.

---

## sst.aws.ApiGatewayV2

HTTP API backed by API Gateway v2. Fast, cheap (~$1/million requests), WebSocket support optional.

```typescript
const api = new sst.aws.ApiGatewayV2("Api", {
  domain: {                     // requires Route53 hosted zone
    name: "api.runwayapp.com",
    dns: sst.aws.dns(),
  },
  cors: {
    allowOrigins: ["https://app.runwayapp.com"],
    allowMethods: ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allowHeaders: ["Content-Type", "Authorization"],
    maxAge: "1 day",
  },
  accessLog: {
    retention: "1 month",
  },
});

api.route("GET /invoices", { handler: "src/handlers/api/invoices.list", link: [table] });
api.route("POST /invoices", { handler: "src/handlers/api/invoices.create", link: [table, emailQueue] });
api.route("GET /invoices/{invoiceId}", { handler: "src/handlers/api/invoices.get", link: [table] });

api.route("DELETE /invoices/{invoiceId}", {  // with Cognito JWT auth
  handler: "src/handlers/api/invoices.remove",
  link: [table],
  auth: { jwt: { authorizer: myAuthorizer } },
});

api.route("$default", { handler: "src/handlers/api/not-found.handler" }); // catch-all 404
```

Outputs:
```typescript
api.url  // e.g. "https://abc123.execute-api.eu-west-1.amazonaws.com"
```

---

## sst.aws.ApiGatewayV2Authorizer

JWT authoriser for Cognito (or any JWKS endpoint).

```typescript
const authorizer = api.addAuthorizer({
  name: "CognitoAuthoriser",
  jwt: {
    issuer: `https://cognito-idp.eu-west-1.amazonaws.com/${userPool.id}`,
    audiences: [userPoolClient.id],
  },
});
```

Use in routes:
```typescript
api.route("GET /me", {
  handler: "src/handlers/api/me.handler",
  auth: { jwt: { authorizer } },
});
```

---

## sst.aws.Dynamo (DynamoDB)

Single DynamoDB table with optional GSIs and streams.

```typescript
const table = new sst.aws.Dynamo("RunwayTable", {
  fields: {
    PK: "string",
    SK: "string",
    GSI1PK: "string",   // must be listed here alongside primary key fields
    GSI1SK: "string",
  },
  primaryIndex: { hashKey: "PK", rangeKey: "SK" },
  globalIndexes: {
    GSI1: { hashKey: "GSI1PK", rangeKey: "GSI1SK" },
  },
  stream: "new-and-old-images",   // triggers a Lambda on every write
  transform: {
    table: {
      removalPolicy: $app.stage === "production"
        ? cdk.RemovalPolicy.RETAIN     // table survives sst remove in prod
        : cdk.RemovalPolicy.DESTROY,
    },
  },
});
```

In Lambda: `Resource.RunwayTable.name` (table name string).

---

## sst.aws.Aurora

Aurora Serverless v2 cluster (PostgreSQL or MySQL).

```typescript
const vpc = new sst.aws.Vpc("RunwayVpc");

const db = new sst.aws.Aurora("RunwayDb", {
  engine: "postgresql15.4",   // or "mysql8.0"
  vpc,

  scaling: {
    min: "0.5 ACU",   // minimum for Serverless v2
    max: "8 ACU",     // scales automatically up to this
  },
  // credentials auto-generated and stored in Secrets Manager
});
```

In Lambda (link it):
```typescript
const fn = new sst.aws.Function("Handler", {
  handler: "...",
  vpc,
  link: [db],
});
```

Linked Resource fields: `host` (cluster endpoint), `port` (5432), `database`, `username`, `password` (from Secrets Manager).

---

## sst.aws.Bucket (S3)

S3 bucket with optional public access, CORS, and versioning.

```typescript
const bucket = new sst.aws.Bucket("RunwayDeliverables", {
  public: false,  // set true only for intentionally public assets

  cors: [
    {
      allowedMethods: ["GET", "PUT", "POST", "DELETE", "HEAD"],
      allowedOrigins: ["https://app.runwayapp.com"],
      allowedHeaders: ["*"],
      maxAge: 3600,
    },
  ],
  versioning: false,
  transform: {
    bucket: (args) => {
      args.lifecycleRules = [{
        expiration: { days: 7 },
        prefix: "tmp/",   // deletes objects in tmp/ after 7 days
        status: "Enabled",
      }];
    },
  },
});
```

In Lambda: `Resource.RunwayDeliverables.name` (bucket name).

---

## sst.aws.Queue (SQS)

SQS queue with optional DLQ, FIFO, and Lambda subscriber.

```typescript
const queue = new sst.aws.Queue("EmailQueue", {
  visibilityTimeout: "120 seconds",

  dlq: {
    queue: "EmailDlq",
    retry: 3,       // retry count before DLQ
  },
  fifo: false,
});

const worker = new sst.aws.Function("EmailWorker", {
  handler: "src/workers/email.handler",
  timeout: "120 seconds",
  link: [queue],
});

worker.subscribe(queue, {
  batchSize: 10,
  reportBatchItemFailures: true,
  filters: [
    { body: { type: ["invoice.paid.email"] } },
  ],
});
```

In Lambda: `Resource.EmailQueue.url` (queue URL for `SendMessageCommand`).

---

## sst.aws.Bus (EventBridge)

EventBridge custom event bus.

```typescript
const bus = new sst.aws.Bus("RunwayBus");

bus.subscribe("invoice-paid-handler", analyticsWorker, {
  pattern: {
    source: ["runway.invoices"],
    detailType: ["invoice.paid"],
  },
});

bus.subscribe("invoice-paid-notifications", notificationWorker, {
  pattern: {
    source: ["runway.invoices"],
    detailType: ["invoice.paid"],
  },
});
```

In Lambda: `Resource.RunwayBus.name` (event bus name for `PutEventsCommand`).

---

## sst.aws.Cron

EventBridge-scheduled Lambda invocation.

```typescript
const chaser = new sst.aws.Cron("InvoiceChaser", {
  schedule: "cron(0 9 * * ? *)",  // daily at 9am UTC; use "rate(1 hour)" for intervals

  job: {
    handler: "src/jobs/invoice-chaser.handler",
    timeout: "300 seconds",
    link: [table, emailQueue],
    environment: {
      SST_STAGE: $app.stage,
    },
  },
});
```

AWS cron field order: `cron(Minutes Hours Day-of-month Month Day-of-week Year)`
- `?` means "no specific value" — use in day-of-month OR day-of-week (not both)
- `*` in day-of-week with `?` in day-of-month: triggers every day

Common schedules:
```
cron(0 9 * * ? *)        Every day at 9:00 UTC
cron(0 9 ? * MON *)      Every Monday at 9:00 UTC
cron(0 */6 * * ? *)      Every 6 hours
cron(0 3 1 * ? *)        1st of every month at 3:00 UTC
rate(5 minutes)          Every 5 minutes
rate(1 hour)             Every hour
```

---

## sst.aws.CognitoUserPool

Cognito User Pool for authentication.

```typescript
const userPool = new sst.aws.CognitoUserPool("RunwayUserPool", {
  usernames: ["email"],
  triggers: {
    preSignUp: preSignUpFn,          // validate + auto-confirm
    postConfirmation: postConfirmFn, // provision workspace on signup
    preTokenGeneration: preTokenFn,  // add custom claims to JWT
  },
});

const client = userPool.addClient("RunwayWebClient", {
  accessTokenValidity: cdk.Duration.hours(1),
  idTokenValidity: cdk.Duration.hours(1),
  refreshTokenValidity: cdk.Duration.days(30),

  oAuth: {
    flows: { authorizationCodeGrant: true },
    scopes: [cognito.OAuthScope.EMAIL, cognito.OAuthScope.OPENID, cognito.OAuthScope.PROFILE],
    callbackUrls: ["https://app.runwayapp.com/auth/callback"],
    logoutUrls: ["https://app.runwayapp.com/auth/logout"],
  },
});
```

Outputs: `userPool.id` (User Pool ID), `client.id` (App Client ID — audience for JWT validation).

---

## sst.Secret

Encrypted secret stored in SSM Parameter Store. Never in CloudFormation or Lambda env vars in plaintext.

```typescript
const mailgunKey = new sst.Secret("MailgunApiKey");
const stripeSecret = new sst.Secret("StripeWebhookSecret");
```

Set values via CLI:
```bash
npx sst secret set MailgunApiKey "key-xxx" --stage production
npx sst secret set MailgunApiKey "key-test-xxx" --stage dev
npx sst secret list --stage production   # Shows names, not values
```

In Lambda (link it):
```typescript
const fn = new sst.aws.Function("Handler", {
  link: [mailgunKey],
});
```

In Lambda: `Resource.MailgunApiKey.value` (resolved at runtime from SSM Parameter Store).

---

## sst.aws.Vpc

Managed VPC for resources that require private networking (Aurora, ElastiCache, etc.).

```typescript
const vpc = new sst.aws.Vpc("RunwayVpc", {
  nat: $app.stage === "production" ? "managed" : "ec2",
  // "managed" = AWS NAT Gateway (~$32/month/AZ); "ec2" = t4g.nano (~$3/month, less HA)
});
```

Pass to any resource that needs VPC placement:
```typescript
new sst.aws.Aurora("Db", { vpc });
new sst.aws.Function("Handler", { vpc });
```

**NAT costs:** `"managed"` = AWS NAT Gateway (~$32/month per AZ + data transfer). `"ec2"` = t4g.nano NAT instance (~$3/month, less HA). For dev stages, `"ec2"` is fine.

---

## Global Transforms

Apply config to all functions in a stack:

```typescript
// sst.config.ts
export default $config({
  app(input) {
    return {
      name: "runway",
      removal: input.stage === "production" ? "retain" : "remove",
      home: "aws",
    };
  },
  async run() {
    $transform(sst.aws.Function, (args) => {
      args.runtime ??= "nodejs22.x";
      args.tracing ??= "active";
      args.environment = {
        ...args.environment,
        SST_STAGE: $app.stage,
        NODE_OPTIONS: "--enable-source-maps",
      };
    });

    // ... resource definitions
  },
});
```

---

## Accessing SST Resource Values

`Resource` is type-safe and auto-generated based on what you `link`:

All fields are only available if the resource is in `link: []`. TypeScript will catch missing links at build time.

| Expression | Value |
|---|---|
| `Resource.RunwayTable.name` | DynamoDB table name |
| `Resource.RunwayBus.name` | EventBridge bus name |
| `Resource.EmailQueue.url` | SQS queue URL |
| `Resource.RunwayDeliverables.name` | S3 bucket name |
| `Resource.RunwayDb.host` | Aurora cluster endpoint |
| `Resource.MailgunApiKey.value` | Secret value (from SSM) |

If a resource isn't linked, TypeScript will tell you at build time. You won't discover it at 2am when the Lambda explodes.
