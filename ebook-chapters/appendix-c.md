# Appendix C: SST Component Reference Cheat Sheet

Quick reference for every SST component used in this book. All examples assume SST v3 with the `sst` package and TypeScript.

Full docs: [sst.dev/docs](https://sst.dev/docs)

---

## sst.aws.Function

Lambda function with sensible defaults (Node.js runtime, bundled with esbuild, X-Ray optional).

```typescript
const fn = new sst.aws.Function("MyFunction", {
  handler: "src/handlers/my-function.handler",

  // Runtime and execution
  runtime: "nodejs22.x",              // default
  timeout: "30 seconds",              // default 10s, max "900 seconds"
  memory: "1024 MB",                  // default 1024 MB

  // Tracing
  tracing: "active",                  // X-Ray: "active" | "passthrough" | "disabled"

  // VPC (required for RDS access)
  vpc,

  // Link other SST resources (auto-generates IAM + injects Resource helper)
  link: [table, bucket, queue, secret],

  // Additional IAM permissions for non-SST resources
  permissions: [
    { actions: ["ses:SendEmail"], resources: ["*"] },
  ],

  // Environment variables (use SST secrets for sensitive values)
  environment: {
    LOG_LEVEL: "INFO",
    SST_STAGE: $app.stage,
  },

  // Bundle config
  nodejs: {
    install: ["pg"],                  // Force-include packages esbuild might tree-shake
    esbuild: {
      external: ["@prisma/client"],   // Exclude from bundle (e.g. Prisma with binaries)
    },
  },

  // Concurrency
  reservedConcurrency: 100,           // Hard limit on concurrent executions
  // provisioned: 5,                  // Warm instances (eliminates cold starts, costs $)
});
```

Access in other functions:
```typescript
import { Resource } from "sst";
// Resource.MyFunction.name — function name
// Resource.MyFunction.arn  — function ARN
```

---

## sst.aws.ApiGatewayV2

HTTP API backed by API Gateway v2. Fast, cheap (~$1/million requests), WebSocket support optional.

```typescript
const api = new sst.aws.ApiGatewayV2("Api", {
  // Custom domain (requires Route53 hosted zone)
  domain: {
    name: "api.runwayapp.com",
    dns: sst.aws.dns(),
  },

  // CORS (for browser clients)
  cors: {
    allowOrigins: ["https://app.runwayapp.com"],
    allowMethods: ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allowHeaders: ["Content-Type", "Authorization"],
    maxAge: "1 day",
  },

  // Access logging
  accessLog: {
    retention: "1 month",
  },
});

// Add routes
api.route("GET /invoices", {
  handler: "src/handlers/api/invoices.list",
  link: [table],
});

api.route("POST /invoices", {
  handler: "src/handlers/api/invoices.create",
  link: [table, emailQueue],
});

api.route("GET /invoices/{invoiceId}", {
  handler: "src/handlers/api/invoices.get",
  link: [table],
});

// Route with Cognito JWT authoriser
api.route("DELETE /invoices/{invoiceId}", {
  handler: "src/handlers/api/invoices.remove",
  link: [table],
  auth: { jwt: { authorizer: myAuthorizer } },
});

// Catch-all 404
api.route("$default", {
  handler: "src/handlers/api/not-found.handler",
});
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
  // Primary key
  fields: {
    PK: "string",
    SK: "string",
    // GSI key attributes must be listed here
    GSI1PK: "string",
    GSI1SK: "string",
  },
  primaryIndex: { hashKey: "PK", rangeKey: "SK" },

  // Global Secondary Indexes
  globalIndexes: {
    GSI1: { hashKey: "GSI1PK", rangeKey: "GSI1SK" },
  },

  // DynamoDB Streams (triggers a Lambda on every write)
  stream: "new-and-old-images",

  // Removal policy (default: "retain" in prod — table survives sst remove)
  transform: {
    table: {
      removalPolicy: $app.stage === "production"
        ? cdk.RemovalPolicy.RETAIN
        : cdk.RemovalPolicy.DESTROY,
    },
  },
});
```

In Lambda:
```typescript
import { Resource } from "sst";
Resource.RunwayTable.name  // Table name string
```

---

## sst.aws.Aurora

Aurora Serverless v2 cluster (PostgreSQL or MySQL).

```typescript
const vpc = new sst.aws.Vpc("RunwayVpc");

const db = new sst.aws.Aurora("RunwayDb", {
  engine: "postgresql15.4",   // or "mysql8.0"
  vpc,

  scaling: {
    min: "0.5 ACU",           // Minimum capacity (0.5 = minimum for Serverless v2)
    max: "8 ACU",             // Maximum capacity (scales automatically)
  },

  // Credentials auto-generated and stored in Secrets Manager
  // Access via Resource.RunwayDb.secretArn or Resource.RunwayDb.clusterArn
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

```typescript
import { Resource } from "sst";
Resource.RunwayDb.host        // Aurora cluster endpoint
Resource.RunwayDb.port        // 5432 (PostgreSQL)
Resource.RunwayDb.database    // Database name
Resource.RunwayDb.username    // Admin username
Resource.RunwayDb.password    // Admin password (from Secrets Manager)
```

---

## sst.aws.Bucket (S3)

S3 bucket with optional public access, CORS, and versioning.

```typescript
const bucket = new sst.aws.Bucket("RunwayAssets", {
  // Public bucket (for public assets — be intentional)
  public: false,  // default false

  // CORS (for direct browser uploads)
  cors: [
    {
      allowedMethods: ["GET", "PUT", "POST", "DELETE", "HEAD"],
      allowedOrigins: ["https://app.runwayapp.com"],
      allowedHeaders: ["*"],
      maxAge: 3600,
    },
  ],

  // Versioning (for audit trail or rollback)
  versioning: false,  // default false

  // Lifecycle rules
  transform: {
    bucket: (args) => {
      args.lifecycleRules = [{
        expiration: { days: 7 },
        prefix: "tmp/",   // Auto-delete objects in tmp/ after 7 days
        status: "Enabled",
      }];
    },
  },
});
```

In Lambda:
```typescript
Resource.RunwayAssets.name  // Bucket name
```

---

## sst.aws.Queue (SQS)

SQS queue with optional DLQ, FIFO, and Lambda subscriber.

```typescript
const queue = new sst.aws.Queue("EmailQueue", {
  visibilityTimeout: "120 seconds",

  // Dead-letter queue
  dlq: {
    queue: "EmailDlq",
    retry: 3,       // Retry count before moving to DLQ
  },

  // FIFO queue
  fifo: false,      // default false
});

// Subscribe a Lambda
const worker = new sst.aws.Function("EmailWorker", {
  handler: "src/workers/email.handler",
  timeout: "120 seconds",
  link: [queue],
});

worker.subscribe(queue, {
  batchSize: 10,
  reportBatchItemFailures: true,

  // Filter: only process messages with a specific attribute
  filters: [
    { body: { type: ["invoice.paid.email"] } },
  ],
});
```

In Lambda:
```typescript
Resource.EmailQueue.url  // SQS queue URL for SendMessageCommand
```

---

## sst.aws.Bus (EventBridge)

EventBridge custom event bus.

```typescript
const bus = new sst.aws.Bus("RunwayBus");

// Subscribe a Lambda to specific event patterns
bus.subscribe("invoice-paid-handler", analyticsWorker, {
  pattern: {
    source: ["runway.invoices"],
    detailType: ["invoice.paid"],
  },
});

// Fan-out: multiple subscribers to the same event
bus.subscribe("invoice-paid-notifications", notificationWorker, {
  pattern: {
    source: ["runway.invoices"],
    detailType: ["invoice.paid"],
  },
});
```

In Lambda:
```typescript
Resource.RunwayBus.name  // Event bus name for PutEventsCommand
```

---

## sst.aws.Cron

EventBridge-scheduled Lambda invocation.

```typescript
const chaser = new sst.aws.Cron("InvoiceChaser", {
  // Cron expression (6-field AWS format)
  schedule: "cron(0 9 * * ? *)",  // Daily at 9am UTC
  // or rate expression:
  // schedule: "rate(1 hour)",

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
  // Email as username
  usernames: ["email"],

  // Allow Cognito to send emails (SES in production)
  // transform for custom SES sender in prod

  // Triggers
  triggers: {
    preSignUp: preSignUpFn,       // Validate + auto-confirm
    postConfirmation: postConfirmFn, // Provision workspace on signup
    preTokenGeneration: preTokenFn,  // Add custom claims to JWT
  },
});

const client = userPool.addClient("RunwayWebClient", {
  // Token expiry
  accessTokenValidity: cdk.Duration.hours(1),
  idTokenValidity: cdk.Duration.hours(1),
  refreshTokenValidity: cdk.Duration.days(30),

  // OAuth flows
  oAuth: {
    flows: { authorizationCodeGrant: true },
    scopes: [cognito.OAuthScope.EMAIL, cognito.OAuthScope.OPENID, cognito.OAuthScope.PROFILE],
    callbackUrls: ["https://app.runwayapp.com/auth/callback"],
    logoutUrls: ["https://app.runwayapp.com/auth/logout"],
  },
});
```

Outputs:
```typescript
userPool.id       // User Pool ID
client.id         // App Client ID (audience for JWT validation)
```

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

```typescript
import { Resource } from "sst";
Resource.MailgunApiKey.value  // Resolved at runtime from SSM
```

---

## sst.aws.Vpc

Managed VPC for resources that require private networking (Aurora, ElastiCache, etc.).

```typescript
const vpc = new sst.aws.Vpc("RunwayVpc", {
  // SST creates public + private subnets by default
  // NAT gateway enabled by default (~$32/month per AZ — consider turning off for dev)
  nat: $app.stage === "production" ? "managed" : "ec2",
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
    // Apply defaults to all functions
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

```typescript
import { Resource } from "sst";

// Only available if the resource is in link: []
Resource.RunwayTable.name    // DynamoDB table name
Resource.RunwayBus.name      // EventBridge bus name
Resource.EmailQueue.url      // SQS queue URL
Resource.RunwayAssets.name   // S3 bucket name
Resource.RunwayDb.host       // Aurora host
Resource.MailgunApiKey.value // Secret value (SSM Parameter Store)
```

If a resource isn't linked, TypeScript will tell you at build time. You won't discover it at 2am when the Lambda explodes.
