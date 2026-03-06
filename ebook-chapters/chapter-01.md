# Chapter 1: Setting Up SST v3

## What This Chapter Does

You have the prerequistes. You know what Runway is. This chapter gets you from zero to a running SST project with a real endpoint live on AWS — the foundation that every subsequent chapter builds on.

Here's the full roadmap, so you can see where Chapter 1 sits in the build:

```
Chapter 1:  [API Gateway] → [Lambda: health check]        ← you are here
Chapter 2:  [API Gateway] → [Lambda: workspaces, invoices]
Chapter 3:  type-safe validation on every route
Chapter 4:  [DynamoDB] ← real data behind every handler
Chapter 5:  [Aurora Serverless] ← relational reporting
Chapter 6:  [S3] ← file uploads and invoice PDFs
Chapter 7:  [Cognito] ← authentication and user scoping
Chapter 8:  [SQS + EventBridge] ← async queues and events
Chapter 9:  [GitHub Actions] ← automated deploys
Chapter 10: [CloudWatch] ← monitoring and alerting
```

By the end of this chapter you'll have SST installed, a dev environment running against real AWS infrastructure, and a clear mental model of how SST's components fit together.

---

## 1.1 What SST Actually Is

SST stands for Serverless Stack. The name undersells it.

At its core, SST is three things bundled into a tool that feels like one:

**Infrastructure as code.** You define your AWS resources — Lambda functions, databases, queues, S3 buckets, APIs — in TypeScript. Not YAML, not JSON, not HCL. TypeScript. The same language you're using to write your application code.

**A local development environment.** `sst dev` deploys a lightweight version of your infrastructure to AWS and then runs your Lambda functions locally, intercepting real invocations. You get a live URL that routes to code on your machine. Code changes are instant — no deploy cycle.

**A deployment tool.** `sst deploy` takes your infrastructure definition and makes it real in AWS. Multiple environments (stages) are handled with a single flag.

SST v3 is a complete rewrite of v2. The biggest change: v2 was built on top of AWS CDK, which meant slow deployments and an abstraction that leaked constantly. v3 dropped CDK entirely and uses Pulumi under the hood. You never interact with Pulumi directly — it's the engine under the hood — but the result is meaningfully faster deployments and a cleaner API.

### How SST compares to the alternatives

**CloudFormation** is the foundational AWS IaC tool. YAML or JSON, verbose, slow to deploy, hard to debug. Nobody writes CloudFormation directly if they can avoid it.

**CDK** (Cloud Development Kit) lets you write CloudFormation in TypeScript. Better than raw CloudFormation, but you're still essentially generating YAML. No live dev experience. Slow feedback loop. SST v2 was CDK with a better DX layer; v3 left CDK behind entirely.

**SAM** (Serverless Application Model) is AWS's official framework for serverless. Template-based, Lambda-focused, limited beyond Lambda + API Gateway. Fine for simple functions, quickly limiting for anything more complex.

**Terraform** is infrastructure-agnostic and battle-tested for complex, multi-cloud environments. No TypeScript-native experience, no live dev. Excellent for teams with existing Terraform expertise or genuinely complex multi-provider setups.

**SST** is the right choice when: you're building a TypeScript-first application on AWS, you want fast feedback during development, and you'd rather write TypeScript than learn a new templating language. That describes most of this book's readers, which is why we're using it.

---

## 1.2 Your First Project

Let's create the Runway project.

### Prerequisites

Before we start, make sure you have:

- **Node.js 22+** — `node --version` should return v22 or higher
- **An AWS account** — the free tier covers everything in this chapter
- **AWS credentials configured** — either via `aws configure` or environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`)

To verify your credentials work:

```bash
aws sts get-caller-identity
```

You should see your account ID and user ARN. If you get an error, sort your credentials before continuing — nothing in this chapter works without them.

### Creating the project

```bash
npx create-sst@latest runway
cd runway
npm install
```

`create-sst` scaffolds a minimal SST v3 project. The structure it creates:

```
runway/
├── sst.config.ts       # Infrastructure definition
├── package.json
├── tsconfig.json
└── .gitignore
```

That's it. No `src/` directory yet, no functions. SST starts minimal and you build up from there.

Open `sst.config.ts`. This is the file you'll be editing constantly:

```typescript
/// <reference path="./.sst/platform/config.d.ts" />

export default $config({
  app(input) {
    return {
      name: "runway",
      removal: input?.stage === "production" ? "retain" : "remove",
      home: "aws",
    };
  },
  async run() {
    // ...
  },
});
```

A few things to note immediately:

The `/// <reference path="./.sst/platform/config.d.ts" />` at the top gives TypeScript access to SST's global types — `$config`, `$app`, and the component types like `sst.aws.Function`. This file is generated the first time you run `sst dev` or `sst deploy`.

The `removal` property tells SST what to do when you run `sst remove` on a stage. For production, `"retain"` means your database and S3 bucket won't be deleted even if you accidentally run `sst remove --stage production`. For every other stage, `"remove"` means full teardown — which is what you want for dev environments you spin up and down freely.

`home: "aws"` tells SST to store your infrastructure state in an S3 bucket in your AWS account. The first time you deploy, SST creates this bucket automatically.

### Adding your first function

Create the first piece of Runway: a simple health check endpoint.

```bash
mkdir -p src/functions
```

```typescript
// src/functions/health.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";

export const handler: APIGatewayProxyHandlerV2 = async () => {
  return {
    statusCode: 200,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      status: "ok",
      service: "runway-api",
      timestamp: new Date().toISOString(),
    }),
  };
};
```

Now wire it to an API in `sst.config.ts`:

```typescript
/// <reference path="./.sst/platform/config.d.ts" />

export default $config({
  app(input) {
    return {
      name: "runway",
      removal: input?.stage === "production" ? "retain" : "remove",
      home: "aws",
    };
  },
  async run() {
    const api = new sst.aws.ApiGatewayV2("RunwayApi");

    api.route("GET /health", {
      handler: "src/functions/health.handler",
    });

    return {
      api: api.url,
    };
  },
});
```

The `return` from `run()` is what SST prints as outputs after a deploy. Anything you put here shows up in the terminal — useful for URLs, table names, queue URLs, anything you need to copy after deploying.

---

## 1.3 SST Components: The Building Blocks

Before we go further, let's understand what SST components actually are.

A component is a TypeScript class that represents one or more AWS resources. `sst.aws.ApiGatewayV2` doesn't just create an API Gateway — it creates the gateway, the default stage, the CloudWatch log group, and the right permissions for Lambda to be invoked by it. SST calls this "batteries included."

The components you'll use most in this book:

| Component | What it creates |
|-----------|----------------|
| `sst.aws.ApiGatewayV2` | HTTP API (API Gateway v2) + stage + logs |
| `sst.aws.Function` | Lambda function + IAM role + log group |
| `sst.aws.Dynamo` | DynamoDB table + auto-scaling config |
| `sst.aws.Bucket` | S3 bucket + CORS + public/private config |
| `sst.aws.Queue` | SQS queue + optional DLQ |
| `sst.aws.Cron` | EventBridge scheduled rule + Lambda target |
| `sst.aws.CognitoUserPool` | Cognito User Pool + app client |
| `sst.aws.Aurora` | Aurora Serverless v2 cluster + subnet group |

You can pass these components to each other, and SST handles the wiring. This is the feature that changes how you think about infrastructure.

### Resource linking

The most powerful SST feature is `link`. When you link a resource to a function, SST does three things automatically:

1. Injects the resource's key properties as environment variables
2. Grants the Lambda's IAM role the appropriate permissions
3. Makes the resource available via the typed `Resource` import in your function code

Here's a preview of what this looks like. We'll build the full table in Chapter 4, but the pattern is the same:

```typescript
// sst.config.ts
const table = new sst.aws.Dynamo("RunwayTable", {
  fields: { pk: "string", sk: "string" },
  primaryIndex: { hashKey: "pk", rangeKey: "sk" },
});

api.route("GET /workspaces", {
  handler: "src/functions/workspaces.handler",
  link: [table],  // ← this line
});
```

In the function itself, `Resource` gives you typed access to everything linked to that function. `Resource.RunwayTable.name` is a fully typed string — TypeScript knows exactly what it is. The Lambda also gets DynamoDB read/write permissions wired up automatically; no IAM policy documents, no `process.env` wrangling.

```typescript
// src/functions/workspaces.ts
import { Resource } from "sst";

const tableName = Resource.RunwayTable.name;
```

No manual environment variable configuration. No IAM policy documents. One line.

This doesn't feel significant until you have 15 resources and 30 Lambda functions and you realise you haven't written a single IAM policy by hand. Then it feels very significant.

---

## 1.4 Live Development with `sst dev`

This is the feature that makes SST worth using over CDK.

```bash
npx sst dev
```

The first time you run this, SST bootstraps your AWS account. It creates:
- An S3 bucket to store infrastructure state
- A few IAM roles SST uses internally
- Your actual infrastructure (the API Gateway and Lambda from `sst.config.ts`)

This takes 2-3 minutes on first run. After that, `sst dev` takes about 10-15 seconds.

When it's ready, you'll see output like:

```
✓  Complete
   RunwayApi: https://abc123.execute-api.eu-west-1.amazonaws.com

   Press Ctrl+C to stop
```

That URL is a real, live API Gateway endpoint. Make a request:

```bash
curl https://abc123.execute-api.eu-west-1.amazonaws.com/health
# {"status":"ok","service":"runway-api","timestamp":"2025-03-01T10:00:00.000Z"}
```

Now open `src/functions/health.ts` and change `"ok"` to `"healthy"`. Save the file. Make the same request again:

```bash
curl https://abc123.execute-api.eu-west-1.amazonaws.com/health
# {"status":"healthy","service":"runway-api","timestamp":"2025-03-01T10:00:01.000Z"}
```

The change is live. No redeploy. No container rebuild. No waiting.

### How Live Lambda works

When `sst dev` is running, API Gateway routes to a thin proxy Lambda deployed in your AWS account. When a request comes in, the proxy opens a WebSocket connection back to your local machine and sends the event. Your local code processes it and returns the response. From API Gateway's perspective, the Lambda responded normally.

This matters for a few reasons:

**You're testing against real AWS.** When you add a DynamoDB table and link it to your function, your local function talks to the real DynamoDB table. There's no mocking, no LocalStack, no "it works locally but breaks in production" class of bugs. What you test is what you ship.

**Error feedback is immediate.** If you've made a mistake that would cause a Lambda error in production, you see it immediately in your terminal — not after a 3-minute deploy cycle.

**You can use the debugger.** Because the code runs locally, you can attach VS Code's debugger to `sst dev` and set breakpoints in your Lambda functions.

### The SST Console

While `sst dev` is running, there's also a web console available at `https://console.sst.dev`. Connect it to your app and you can:

- View real-time Lambda logs across all functions
- Browse your DynamoDB tables and run queries
- Trigger functions manually
- View and replay SQS messages

It's optional but useful, especially when debugging async functions where you don't have an HTTP response to inspect.

---

## 1.5 Stages and Environments

SST's stage system is how you manage multiple environments without duplicating infrastructure code.

Every SST command accepts a `--stage` flag. If you don't provide one, it defaults to your local username (so `sst dev` on Michael's machine uses stage `michael`, on Sarah's machine it uses stage `sarah`). Each developer gets their own isolated AWS environment automatically.

```bash
npx sst dev                           # stage: your username
npx sst deploy --stage staging        # stage: staging
npx sst deploy --stage production     # stage: production
```

Each stage is completely isolated in AWS. Different CloudFormation stacks, different DynamoDB tables, different API URLs. Nothing from dev can reach production data.

### Stage-specific configuration

You already saw the `removal` property vary by stage. Here's how to do that for anything. Derive a boolean at the top of `run()` and branch on it across your resource config — one comparison, used everywhere.

`$app.stage` is SST's typed global for the current stage name. Use it anywhere in `sst.config.ts`.

```typescript
async run() {
  const isProd = $app.stage === "production";
```

**Access logs**

CloudWatch access logs on API Gateway cost money. There's no reason to pay for them on dev or staging stages where you're already watching your terminal. Enable them only in production, and set a retention window so old logs don't accumulate indefinitely.

```typescript
  const api = new sst.aws.ApiGatewayV2("RunwayApi", {
    accessLog: isProd
      ? { retention: "1 month" }
      : false,
  });
```

**DynamoDB provisioned capacity**

DynamoDB provisioned capacity has a minimum cost even when idle. Dev stages that sit unused overnight don't need it. The `...(isProd && { ... })` spread is the idiomatic way to conditionally include properties in an SST config object.

```typescript
  const table = new sst.aws.Dynamo("RunwayTable", {
    fields: { pk: "string", sk: "string" },
    primaryIndex: { hashKey: "pk", rangeKey: "sk" },
    ...(isProd && {
      scaling: { min: 5, max: 100 },
    }),
  });
}
```

### Environment variables across stages

For secrets and environment-specific values, SST has a built-in secrets manager that stores values in AWS SSM Parameter Store:

```bash
# Set a secret for a specific stage
npx sst secret set STRIPE_SECRET_KEY "sk_test_..." --stage staging
npx sst secret set STRIPE_SECRET_KEY "sk_live_..." --stage production
```

```typescript
// sst.config.ts
const stripeKey = new sst.Secret("StripeSecretKey");

api.route("POST /invoices", {
  handler: "src/functions/invoices.handler",
  link: [stripeKey],
});
```

```typescript
// src/functions/invoices.ts
import { Resource } from "sst";

const stripe = new Stripe(Resource.StripeSecretKey.value);
```

The secret is injected at runtime, stage-specific, and never touches your source code or environment files.

---

## 1.6 Your First Deploy

When you're ready to deploy Runway to a permanent environment:

```bash
npx sst deploy --stage production
```

SST:
1. Reads `sst.config.ts`
2. Computes the difference between your current infrastructure and what's deployed
3. Bundles your Lambda functions with esbuild
4. Deploys the changes to AWS
5. Prints outputs

```
✓  Complete
   RunwayApi: https://xyz789.execute-api.eu-west-1.amazonaws.com
```

That URL is permanent — it doesn't change on subsequent deploys unless you change the API configuration. Bookmark it.

### What gets created in your AWS account

After your first deploy, you'll find in the AWS console:

- **Lambda**: A function named `runway-production-RunwayApi-health` (or similar)
- **API Gateway**: An HTTP API named `runway-production-RunwayApi`
- **IAM**: A role for the Lambda function with minimum necessary permissions
- **CloudWatch Logs**: A log group for the Lambda function
- **S3**: A bucket SST uses to store state (in the region you deployed to)

SST names everything consistently — `{app}-{stage}-{resource}`. This makes it easy to find things in the AWS console and to know at a glance what belongs to what environment.

### Subsequent deploys

Once the infrastructure exists, deploying a code change is fast:

```bash
npx sst deploy --stage production
```

SST detects that the infrastructure hasn't changed (no new resources, no config changes) and only updates the Lambda function code. This typically takes 15-30 seconds.

If you change the infrastructure — adding a new table, a new route, a new queue — SST provisions the new resources before updating the Lambda code, so there's no moment where your code references a resource that doesn't exist yet.

### Cleaning up

To tear down a non-production environment:

```bash
npx sst remove --stage staging
```

Because `removal` is set to `"remove"` for non-production stages, everything gets deleted — API Gateway, Lambda, DynamoDB table, CloudWatch logs. Clean slate.

For production, the `"retain"` policy means `sst remove --stage production` will delete the SST stack metadata but leave your actual AWS resources in place. This protects you from accidental data loss.

---

## Where We Are

You've got a running SST project with a health check endpoint, deployed to AWS and accessible via a real URL. The full Runway system is mapped out: the chapters ahead will add each layer systematically.

Here's the roadmap as we've built it so far, and what we're adding chapter by chapter:

```
Chapter 1:  [API Gateway] → [Lambda: health]
Chapter 2:  [API Gateway] → [Lambda: workspaces, projects, invoices]
Chapter 3:  [API Gateway] → [Lambda: ...] with Zod validation + typed responses
Chapter 4:  [DynamoDB]    ← [Lambda: workspaces, projects, invoices]
Chapter 5:  [Aurora Serverless] ← [Lambda: reporting queries, payment records]
Chapter 6:  [S3]          ← [Lambda: deliverables, invoice PDFs]
Chapter 7:  [Cognito]     ← [API Gateway authoriser] + user-scoped data
Chapter 8:  [SQS]         ← [Lambda: notifications, reminders]
Chapter 9:  [GitHub CI]   → [sst deploy --stage production]
Chapter 10: [CloudWatch]  ← all of the above
```

Each chapter builds on the previous one. By Chapter 10, the health check you just wrote is one small corner of a complete, production-grade system.

In the next chapter, we'll write the Lambda functions that handle Runway's core entities — workspaces, clients, projects, and invoices — and establish the patterns that will carry us through the rest of the book.

---

> **The code for this chapter** is on the `chapter-1` branch of the companion repository. Run `npm install && npx sst dev` to start it.
