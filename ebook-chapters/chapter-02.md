# Chapter 2: Lambda Functions That Don't Suck

In Chapter 1 we got a health check endpoint deployed. One function, one route, twenty lines of code. That's not a system — that's a proof of concept.

Chapter 2 is where Runway becomes real. We're going to write the Lambda functions that handle workspaces, clients, projects, and invoices. More importantly, we're going to do it in a way that still makes sense six months later, at 3am, when something's broken in production and you need to figure out why in under ten minutes.

That's the actual bar. Not "does it work?" — anything can pass that. The bar is: does it hold up under operational pressure? Can you deploy with confidence? Can you diagnose a failure quickly? Can a second developer pick this up without a 90-minute walkthrough?

The patterns in this chapter are the foundation. Everything else in the book builds on them.

---

## 2.1 Handler Typing Done Right

Let's start with the type system, because everything else follows from it.

Your Lambda handler receives an `event` object whose shape depends entirely on what triggered the function. API Gateway sends a different shape than SQS, which is different from S3, which is different from EventBridge. TypeScript can help you here — but only if you tell it which trigger you're dealing with.

### The main types you need

The `aws-lambda` package (included in the Lambda Node.js runtime) exports typed handler types. Install the type definitions:

```bash
npm install --save-dev @types/aws-lambda
```

Here are the handler types you'll use in Runway:

```typescript
import type {
  APIGatewayProxyHandlerV2,        // HTTP API (API Gateway v2) — use this
  APIGatewayProxyHandler,           // REST API (API Gateway v1) — avoid this
  SQSHandler,                       // SQS queue consumer
  ScheduledHandler,                 // EventBridge scheduled rule (cron)
  S3Handler,                        // S3 event trigger
  APIGatewayProxyEventV2,          // The event type for HTTP API
  APIGatewayProxyResultV2,         // The return type for HTTP API
} from "aws-lambda";
```

The `V2` suffix matters. `APIGatewayProxyHandlerV2` is for the **HTTP API** (API Gateway v2). `APIGatewayProxyHandler` is for the **REST API** (API Gateway v1). They have meaningfully different event shapes, and if you use the wrong type, TypeScript will give you false confidence.

In this book we always use API Gateway v2 (HTTP API). It's cheaper, faster, simpler, and has everything Runway needs. Whenever you see `V2` in a type name, that's what we're using.

### What each type looks like

```typescript
// HTTP API handler — the one you'll write most often
export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  // event.body is string | null | undefined
  // event.pathParameters is Record<string, string | undefined> | undefined
  // event.queryStringParameters is Record<string, string | undefined> | undefined
  // event.headers is Record<string, string | undefined>
  // event.requestContext.http.method is "GET" | "POST" | "PUT" | etc.

  // context.requestId — the unique ID for this invocation
  // context.functionName — the Lambda function name
  // context.getRemainingTimeInMillis() — time until Lambda times out

  return {
    statusCode: 200,
    body: JSON.stringify({ ok: true }),
  };
};
```

```typescript
// SQS consumer — processes messages from a queue
export const handler: SQSHandler = async (event, context) => {
  // event.Records is an array of SQS messages
  for (const record of event.Records) {
    // record.body is the raw message string
    const message = JSON.parse(record.body);
    // process message...
  }
  // Return void — SQS handlers don't return HTTP responses
  // Throw an error to NACK the message (send to DLQ after retries)
};
```

```typescript
// Scheduled handler — triggered by EventBridge cron
export const handler: ScheduledHandler = async (event, context) => {
  // event.source === "aws.events"
  // event.detail-type === "Scheduled Event"
  // event.time is the scheduled time as ISO string

  // Run your scheduled job logic here
};
```

```typescript
// S3 event handler — triggered when an object is created/deleted
export const handler: S3Handler = async (event, context) => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, " "));
    // process the uploaded file...
  }
};
```

### Why `async (event, context)` matters

Always include `context` in your handler signature. You might not use it in every function, but two things make it invaluable:

**`context.requestId`** is the Lambda request ID — a UUID that identifies this specific invocation. When you log it with every log entry, you can filter CloudWatch Logs to a single request and see everything that happened. Without it, debugging concurrent requests is guesswork.

**`context.getRemainingTimeInMillis()`** tells you how much of the Lambda timeout is left. If you're in a loop processing items and you want to stop before Lambda hard-kills you, this is how you do it safely:

```typescript
export const handler: SQSHandler = async (event, context) => {
  for (const record of event.Records) {
    // Stop processing if we have less than 5 seconds left
    if (context.getRemainingTimeInMillis() < 5000) {
      break;
    }
    await processRecord(record);
  }
};
```

The type signature for `async (event, context)` is:
```
(event: APIGatewayProxyEventV2, context: Context) => Promise<APIGatewayProxyResultV2>
```

When you use `APIGatewayProxyHandlerV2`, TypeScript knows all three pieces. If you try to return the wrong shape or access a property that doesn't exist on the event, you get a compile error, not a runtime error in production.

### Runway's first real handlers

Let's look at what the workspace and invoice handlers look like with proper typing. We'll fill in the business logic as the chapter progresses, but the signatures are the foundation:

```typescript
// src/functions/workspaces/list.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  // List all workspaces for the authenticated user
  // (Authentication comes in Chapter 7 — for now we'll read from headers)
  return {
    statusCode: 200,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ workspaces: [] }),
  };
};
```

```typescript
// src/functions/invoices/create.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  // Parse request body, validate it, create invoice
  const body = event.body ? JSON.parse(event.body) : {};

  return {
    statusCode: 201,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ invoice: {} }),
  };
};
```

Right now these don't do anything useful. By the end of this chapter, they'll have proper response helpers, structured logging, error handling, and environment validation. We'll build each layer.

### A note on file structure

Before we go further, let's establish Runway's file structure. This isn't arbitrary — the organisation reflects the architecture:

```
src/
├── functions/                    # Lambda handler entry points
│   ├── workspaces/
│   │   ├── list.ts
│   │   ├── get.ts
│   │   ├── create.ts
│   │   └── update.ts
│   ├── clients/
│   │   ├── list.ts
│   │   ├── get.ts
│   │   └── create.ts
│   ├── projects/
│   │   ├── list.ts
│   │   └── create.ts
│   └── invoices/
│       ├── list.ts
│       ├── get.ts
│       ├── create.ts
│       └── send.ts
├── services/                     # Business logic (no Lambda deps)
│   ├── workspaces.ts
│   ├── clients.ts
│   ├── projects.ts
│   └── invoices.ts
├── lib/                          # Shared utilities
│   ├── response.ts               # Response helpers (section 2.2)
│   ├── logger.ts                 # Structured logging (section 2.5)
│   ├── errors.ts                 # Error classes (section 2.6)
│   ├── config.ts                 # Environment validation (section 2.7)
│   └── db/                       # Database clients (lazy init, section 2.3)
│       └── dynamodb.ts
└── types/                        # Shared TypeScript types
    ├── workspace.ts
    ├── client.ts
    ├── project.ts
    └── invoice.ts
```

The key principle: **handler files are thin**. They wire things together. The actual work happens in services. We'll come back to this in section 2.4.

---

## 2.2 Response Helpers

Every HTTP API returns responses. Every response needs a status code, headers, and a body. If you write that out by hand in every handler, three things will go wrong:

1. You'll forget to set `Content-Type` in exactly the place where it matters
2. Error responses will be inconsistently shaped (some have `error`, some have `message`, some have `errors`, clients have to handle all of them)
3. You'll `JSON.stringify` the wrong thing and ship malformed JSON without realising

The fix is a response library. Write it once, use it everywhere.

### Building the response library

```typescript
// src/lib/response.ts

export type ApiResponse = {
  statusCode: number;
  headers: Record<string, string>;
  body: string;
};

const json = (statusCode: number, body: unknown): ApiResponse => ({
  statusCode,
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify(body),
});

// Success responses
export const ok = <T>(data: T): ApiResponse =>
  json(200, { data });

export const created = <T>(data: T): ApiResponse =>
  json(201, { data });

export const noContent = (): ApiResponse => ({
  statusCode: 204,
  headers: {},
  body: "",
});

// Error responses
export const badRequest = (message: string, details?: unknown): ApiResponse =>
  json(400, {
    error: {
      code: "BAD_REQUEST",
      message,
      ...(details !== undefined && { details }),
    },
  });

export const unauthorized = (message = "Unauthorized"): ApiResponse =>
  json(401, {
    error: {
      code: "UNAUTHORIZED",
      message,
    },
  });

export const forbidden = (message = "Forbidden"): ApiResponse =>
  json(403, {
    error: {
      code: "FORBIDDEN",
      message,
    },
  });

export const notFound = (resource: string): ApiResponse =>
  json(404, {
    error: {
      code: "NOT_FOUND",
      message: `${resource} not found`,
    },
  });

export const conflict = (message: string): ApiResponse =>
  json(409, {
    error: {
      code: "CONFLICT",
      message,
    },
  });

export const unprocessable = (message: string, details?: unknown): ApiResponse =>
  json(422, {
    error: {
      code: "UNPROCESSABLE",
      message,
      ...(details !== undefined && { details }),
    },
  });

export const internalError = (message = "Internal server error"): ApiResponse =>
  json(500, {
    error: {
      code: "INTERNAL_ERROR",
      message,
    },
  });

export const res = {
  ok,
  created,
  noContent,
  badRequest,
  unauthorized,
  forbidden,
  notFound,
  conflict,
  unprocessable,
  internalError,
};
```

With this library, your handlers look like:

```typescript
import { res } from "../../lib/response";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const workspace = await workspaceService.findById(workspaceId);

  if (!workspace) {
    return res.notFound("Workspace");
  }

  return res.ok(workspace);
};
```

Clean. Consistent. The response shape is locked in.

### The consistent error shape

The error shape matters. Your frontend clients will parse these responses, and they need predictability. Here's the contract we're establishing:

**Success response:**
```json
{
  "data": { ... }
}
```

**Error response:**
```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Workspace not found",
    "details": { ... }  // optional — validation errors, etc.
  }
}
```

Every error has a `code` (machine-readable, stable across versions) and a `message` (human-readable, can change). Clients can switch on `code` reliably. The `details` field carries extra context — we'll use it in Chapter 3 for Zod validation errors.

### HTTP status codes worth knowing

People misuse status codes constantly. Here's the Runway policy:

| Status | When to use |
|--------|-------------|
| 200 OK | Successful GET, successful PUT (with body) |
| 201 Created | Successful POST that creates a resource |
| 204 No Content | Successful DELETE, successful PUT/PATCH with no response body |
| 400 Bad Request | Malformed request syntax, invalid JSON, missing required body |
| 401 Unauthorized | Authentication required but missing or invalid token |
| 403 Forbidden | Authenticated, but not allowed to access this resource |
| 404 Not Found | Resource doesn't exist — or user isn't allowed to know it exists |
| 409 Conflict | Resource already exists (idempotency conflict) |
| 422 Unprocessable Entity | Request is valid syntax but semantically invalid (failed validation) |
| 500 Internal Server Error | Something went wrong on our side that the client can't fix |

The 401/403 distinction matters: 401 means "I don't know who you are", 403 means "I know who you are, and you can't do this." In practice, for resources that belong to other users, you often want 404 rather than 403 — returning 403 tells an attacker that the resource exists.

One misuse that's rampant in the wild: using 200 with a success field in the body:
```json
// Do NOT do this
{ "success": false, "error": "Something went wrong" }
```

HTTP has status codes. Use them. Clients know how to check them.

### Typed response wrappers

If you want to go one step further, you can make `ok()` and `created()` infer the types from what you pass in — useful when you have strict return type checks on your services:

```typescript
// src/types/workspace.ts
export type Workspace = {
  id: string;
  name: string;
  ownerId: string;
  plan: "free" | "pro" | "enterprise";
  createdAt: string;
};

export type WorkspaceList = {
  workspaces: Workspace[];
  total: number;
  cursor?: string;
};
```

```typescript
// Handler with typed response
export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const result: WorkspaceList = await workspaceService.list(ownerId);
  return res.ok(result); // TypeScript knows the body shape
};
```

The response helper doesn't enforce the shape — `JSON.stringify` will take anything — but the TypeScript inference helps catch bugs where you accidentally pass the wrong thing to `res.ok()`.

---

## 2.3 Cold Starts: The Honest Guide

Cold starts are the most misunderstood performance topic in serverless. There's a lot of cargo-culting and voodoo around them. Let's be precise.

### What a cold start actually is

Lambda functions run in "execution environments" — think of them as very lightweight container instances. When a function hasn't been invoked recently (or has never been invoked), AWS has to:

1. Allocate a container
2. Download your code bundle
3. Start the Node.js runtime
4. Run all top-level code in your module
5. Call your handler

Steps 1-4 are the cold start. Step 5 is the invocation. In a "warm" invocation (execution environment already exists), only step 5 happens.

Cold starts typically add 200-500ms to your response time for a Node.js Lambda with a reasonable bundle size. They're not the end of the world. For a background job or async function, they're completely irrelevant. For a user-facing API, they're noticeable but not usually catastrophic.

The things that actually matter:

**Bundle size** is the biggest lever. A 1MB bundle cold starts faster than a 10MB bundle. Keep your Lambda bundles lean. SST uses esbuild by default, which tree-shakes aggressively — don't fight it by importing entire SDKs.

**Top-level I/O** is the worst offender. If you connect to a database, call an external API, or read a file at the module level (outside the handler function), that I/O happens during the cold start, blocking everything. Don't do it.

**SDK client initialisation** is cheaper than I/O but still has cost. Creating a `DynamoDBClient` involves loading a module, parsing config, creating internal objects. Do it lazily.

### The lazy initialisation pattern

The wrong way:

```typescript
// ❌ This runs at module load time — during the cold start
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";

const dynamodb = new DynamoDBClient({ region: process.env.AWS_REGION });

export const handler: APIGatewayProxyHandlerV2 = async (event) => {
  // use dynamodb...
};
```

The right way:

```typescript
// ✅ Module-level variable, but only initialised on first call
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient } from "@aws-sdk/lib-dynamodb";

let _client: DynamoDBDocumentClient | undefined;

const getClient = (): DynamoDBDocumentClient => {
  if (!_client) {
    const base = new DynamoDBClient({ region: process.env.AWS_REGION });
    _client = DynamoDBDocumentClient.from(base, {
      marshallOptions: {
        removeUndefinedValues: true,
        convertClassInstanceToMap: true,
      },
    });
  }
  return _client;
};

export { getClient };
```

On the first invocation (warm or cold), `getClient()` initialises the client and caches it. On every subsequent invocation **in the same execution environment**, it returns the cached instance. The client is reused across invocations in the same environment, which means connection overhead is paid once per environment lifecycle, not once per request.

Here's the full implementation for Runway's DynamoDB client:

```typescript
// src/lib/db/dynamodb.ts
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  GetCommand,
  PutCommand,
  UpdateCommand,
  DeleteCommand,
  QueryCommand,
  TransactWriteCommand,
} from "@aws-sdk/lib-dynamodb";

// Re-export command types so services don't import from the SDK directly
export {
  GetCommand,
  PutCommand,
  UpdateCommand,
  DeleteCommand,
  QueryCommand,
  TransactWriteCommand,
};

let _client: DynamoDBDocumentClient | undefined;

export const getDb = (): DynamoDBDocumentClient => {
  if (_client) return _client;

  const base = new DynamoDBClient({
    region: process.env.AWS_REGION,
    // Reduce cold start time: limit retries during initialisation
    maxAttempts: 3,
  });

  _client = DynamoDBDocumentClient.from(base, {
    marshallOptions: {
      // Remove undefined values instead of setting to NULL
      removeUndefinedValues: true,
      // Convert Date objects to ISO strings
      convertClassInstanceToMap: false,
    },
    unmarshallOptions: {
      // Numbers come back as native numbers, not strings
      wrapNumbers: false,
    },
  });

  return _client;
};
```

The same pattern applies to every external client in Runway: Stripe, SES (for emails), S3:

```typescript
// src/lib/stripe.ts
import Stripe from "stripe";
import { getConfig } from "./config";

let _stripe: Stripe | undefined;

export const getStripe = (): Stripe => {
  if (_stripe) return _stripe;
  const config = getConfig();
  _stripe = new Stripe(config.STRIPE_SECRET_KEY, {
    apiVersion: "2024-06-20",
    typescript: true,
  });
  return _stripe;
};
```

### Measuring your cold start

Don't optimise what you haven't measured. CloudWatch logs every invocation's `Init Duration` when a cold start occurs. Filter for it:

```
fields @timestamp, @requestId, @initDuration, @duration
| filter @type = "REPORT"
| filter ispresent(@initDuration)
| sort @timestamp desc
| limit 20
```

The `@initDuration` value is the cold start time in milliseconds. If it's consistently under 500ms, you're fine. If it's over 1 second, look at your bundle size and lazy init patterns.

### Bundle size: the esbuild story

SST bundles each Lambda function separately using esbuild. Each function gets only the code it imports — not everything in your project. This is a key difference from some older frameworks that deploy your entire `node_modules` to every function.

To see the size of a bundled function:

```bash
# After sst build (or during sst dev, check .sst/outputs)
ls -la .sst/dist/
```

The AWS SDK v3 (`@aws-sdk/...`) is split into small packages specifically to enable tree-shaking. Importing `@aws-sdk/client-dynamodb` doesn't bring in S3, SES, or anything else — only the DynamoDB client. This is why we moved from SDK v2 to v3.

### Provisioned concurrency: when it's worth it and when it isn't

Provisioned concurrency (PC) keeps a set of Lambda execution environments permanently warm. You pay for it continuously, whether your function receives requests or not.

**When it's worth it:**
- User-facing endpoints where you've measured cold starts causing p50 latency to spike noticeably
- Functions that have genuinely expensive cold starts (>1 second) you can't reduce
- High-traffic APIs where every request is warm anyway and PC just makes it guaranteed

**When it isn't worth it:**
- Everything in development
- Background jobs, SQS consumers, cron functions — these are async, cold starts don't matter
- Sporadically-invoked functions (you'd be paying for idle time constantly)
- When you haven't measured whether cold starts are actually causing problems

For Runway in its early stages: don't add provisioned concurrency. Get the bundle small, use lazy initialisation, and measure. Add PC when you have evidence it's needed, not before.

If you do add it, here's how in SST:

```typescript
// sst.config.ts
api.route("GET /invoices", {
  handler: "src/functions/invoices/list.handler",
  // Only provision concurrency in production
  ...(isProd && {
    warmup: { minConcurrency: 2 },
  }),
});
```

---

## 2.4 Structuring a Production Lambda

The health check handler from Chapter 1 is a bad template for real handlers. It does everything in one function: parsing, validation, logic, response formatting. For one endpoint that returns a static response, that's fine. For a function that creates an invoice, it's not.

### Thin handlers, fat services

The handler is the boundary between the Lambda runtime and your code. Its job is:

1. Extract the inputs from the event (path params, query string, body, headers)
2. Call the service with those inputs
3. Map the result to an HTTP response

The service is where the business logic lives. It has no knowledge of Lambda, API Gateway, HTTP status codes, or event shapes. It takes typed parameters and returns typed results.

Here's what this looks like in practice:

```typescript
// src/functions/invoices/create.ts  — the handler
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { AppError } from "../../lib/errors";
import { invoiceService } from "../../services/invoices";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const log = logger.child({ requestId: context.awsRequestId });

  try {
    // 1. Extract inputs
    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("Missing workspaceId");

    const body = event.body ? JSON.parse(event.body) : {};

    // 2. Call service (validation is here for now, moves to middleware in Ch. 3)
    const invoice = await invoiceService.create(workspaceId, body);

    log.info("Invoice created", { invoiceId: invoice.id, workspaceId });

    // 3. Return response
    return res.created(invoice);
  } catch (err) {
    if (err instanceof AppError) {
      return res.fromAppError(err);
    }
    log.error("Unhandled error in createInvoice", { err });
    return res.internalError();
  }
};
```

```typescript
// src/services/invoices.ts  — the service
import { nanoid } from "nanoid";
import { getDb, PutCommand, QueryCommand } from "../lib/db/dynamodb";
import { AppError } from "../lib/errors";
import { getConfig } from "../lib/config";
import type { Invoice, CreateInvoiceInput } from "../types/invoice";

const config = getConfig();

export const invoiceService = {
  async create(workspaceId: string, input: CreateInvoiceInput): Promise<Invoice> {
    // Business logic here — no Lambda, no HTTP, no event shapes
    const db = getDb();

    // Generate a human-readable invoice number
    const invoiceNumber = await generateInvoiceNumber(workspaceId);

    const invoice: Invoice = {
      id: `inv_${nanoid()}`,
      workspaceId,
      invoiceNumber,
      clientId: input.clientId,
      projectId: input.projectId,
      lineItems: input.lineItems,
      status: "draft",
      subtotal: input.lineItems.reduce((sum, item) => sum + item.amount, 0),
      tax: 0,
      total: 0,
      currency: input.currency ?? "GBP",
      issueDate: input.issueDate,
      dueDate: input.dueDate,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };

    invoice.tax = Math.round(invoice.subtotal * 0.2); // 20% VAT for UK
    invoice.total = invoice.subtotal + invoice.tax;

    await db.send(
      new PutCommand({
        TableName: config.TABLE_NAME,
        Item: {
          pk: `WORKSPACE#${workspaceId}`,
          sk: `INVOICE#${invoice.id}`,
          ...invoice,
          type: "INVOICE",
        },
        // Prevent overwriting an existing invoice
        ConditionExpression: "attribute_not_exists(pk)",
      })
    );

    return invoice;
  },

  async findById(workspaceId: string, invoiceId: string): Promise<Invoice | null> {
    const db = getDb();
    // ... implementation
    return null;
  },

  async list(workspaceId: string): Promise<Invoice[]> {
    const db = getDb();
    // ... implementation
    return [];
  },
};

async function generateInvoiceNumber(workspaceId: string): Promise<string> {
  // Implementation — queries DynamoDB for the last invoice number
  // and increments it. Returns "INV-0001" format.
  return "INV-0001";
}
```

The separation is clean:
- The handler parses Lambda event types and returns Lambda-shaped responses
- The service deals in domain types (`Invoice`, `CreateInvoiceInput`)
- Neither bleeds into the other

### What goes in the service layer

The service handles:
- Business logic and validation rules
- Database reads and writes
- Calls to external APIs (Stripe, email, etc.)
- Domain-level errors (throw `AppError`, not `{ statusCode: 404 }`)

The service does NOT handle:
- Parsing HTTP event shapes
- Setting HTTP status codes
- JSON serialisation
- Logging request IDs (it can log business events, but not request-level metadata)

### Module organisation for a Lambda package

Lambda cares about what's imported at the module level because of bundle size and cold start time. The rule: only import what you use.

```typescript
// ✅ Good — specific imports
import { PutCommand } from "@aws-sdk/lib-dynamodb";

// ❌ Bad — pulls in the entire dynamodb module
import * as dynamodb from "@aws-sdk/lib-dynamodb";
```

For shared code (`lib/`, `types/`), be deliberate about barrel files (`index.ts` that re-exports everything). They're convenient but can cause esbuild to include more than you need. Prefer direct imports from the file that has what you need:

```typescript
// ✅ Direct import
import { AppError } from "../lib/errors";

// ⚠️ Barrel import — fine if the barrel is small and well-managed
import { AppError } from "../lib";
```

The SST build output will tell you the bundle size for each function. Keep an eye on it as you add dependencies.

### Types for Runway's domain

Before we go further, let's define the types that will carry through the rest of the book:

```typescript
// src/types/workspace.ts
export type WorkspacePlan = "free" | "pro" | "enterprise";

export type Workspace = {
  id: string;
  name: string;
  slug: string;
  ownerId: string;
  plan: WorkspacePlan;
  logoUrl?: string;
  createdAt: string;
  updatedAt: string;
};

export type CreateWorkspaceInput = {
  name: string;
  slug: string;
};
```

```typescript
// src/types/client.ts
export type Client = {
  id: string;
  workspaceId: string;
  name: string;
  email: string;
  company?: string;
  vatNumber?: string;
  address?: {
    line1: string;
    line2?: string;
    city: string;
    postcode: string;
    country: string;
  };
  createdAt: string;
  updatedAt: string;
};

export type CreateClientInput = Pick<Client, "name" | "email" | "company" | "vatNumber" | "address">;
```

```typescript
// src/types/project.ts
export type ProjectStatus = "active" | "paused" | "completed" | "archived";

export type Project = {
  id: string;
  workspaceId: string;
  clientId: string;
  name: string;
  description?: string;
  status: ProjectStatus;
  hourlyRate?: number;
  budget?: number;
  currency: string;
  startDate?: string;
  endDate?: string;
  createdAt: string;
  updatedAt: string;
};
```

```typescript
// src/types/invoice.ts
export type LineItem = {
  description: string;
  quantity: number;
  unitPrice: number;
  amount: number;
};

export type InvoiceStatus = "draft" | "sent" | "paid" | "overdue" | "cancelled";

export type Invoice = {
  id: string;
  workspaceId: string;
  invoiceNumber: string;
  clientId: string;
  projectId?: string;
  status: InvoiceStatus;
  lineItems: LineItem[];
  subtotal: number;
  tax: number;
  total: number;
  currency: string;
  issueDate: string;
  dueDate: string;
  paidAt?: string;
  notes?: string;
  createdAt: string;
  updatedAt: string;
};

export type CreateInvoiceInput = {
  clientId: string;
  projectId?: string;
  lineItems: LineItem[];
  currency?: string;
  issueDate: string;
  dueDate: string;
  notes?: string;
};
```

These types will evolve as we add DynamoDB in Chapter 4, but the domain shape stays consistent throughout.

---

## 2.5 Structured Logging

At 3am when Runway is down, you open CloudWatch and find:

```
Error: Cannot read properties of undefined (reading 'id')
it worked before
processing invoice
done
```

This is useless. You don't know which request caused the error. You don't know which invoice. You don't know which user. You don't know if it's happening to one user or all of them.

Structured logging makes 3am survivable.

### Why `console.log` isn't enough

`console.log` is fine for local development. In production, it has two problems:

**You can't query it.** CloudWatch Logs Insights is a powerful query tool — but only for structured data. Plain text logs are just a wall of text. Structured JSON logs are queryable, filterable, and aggregatable.

**Context is missing.** Every log entry needs to carry context: which request, which user, which function. With `console.log`, you'd have to remember to include all that every time. You won't. A structured logger bakes it in automatically.

### Building Runway's logger

```typescript
// src/lib/logger.ts

type LogLevel = "debug" | "info" | "warn" | "error";

type LogContext = Record<string, unknown>;

type Logger = {
  debug(message: string, context?: LogContext): void;
  info(message: string, context?: LogContext): void;
  warn(message: string, context?: LogContext): void;
  error(message: string, context?: LogContext): void;
  child(context: LogContext): Logger;
};

const LOG_LEVELS: Record<LogLevel, number> = {
  debug: 0,
  info: 1,
  warn: 2,
  error: 3,
};

const currentLevel = (): LogLevel => {
  const env = process.env.LOG_LEVEL?.toLowerCase();
  if (env === "debug" || env === "info" || env === "warn" || env === "error") {
    return env;
  }
  return process.env.NODE_ENV === "production" ? "info" : "debug";
};

const createLogger = (baseContext: LogContext = {}): Logger => {
  const shouldLog = (level: LogLevel): boolean =>
    LOG_LEVELS[level] >= LOG_LEVELS[currentLevel()];

  const write = (level: LogLevel, message: string, context?: LogContext): void => {
    if (!shouldLog(level)) return;

    const entry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      ...baseContext,
      ...context,
    };

    // CloudWatch picks these up correctly when written to stdout
    const output = JSON.stringify(entry);
    if (level === "error" || level === "warn") {
      process.stderr.write(output + "\n");
    } else {
      process.stdout.write(output + "\n");
    }
  };

  return {
    debug: (message, context) => write("debug", message, context),
    info: (message, context) => write("info", message, context),
    warn: (message, context) => write("warn", message, context),
    error: (message, context) => write("error", message, context),
    child: (context) => createLogger({ ...baseContext, ...context }),
  };
};

// Base logger — used at module level
export const logger = createLogger({
  service: "runway-api",
  stage: process.env.SST_STAGE ?? "unknown",
});
```

### Using the logger in handlers

The key pattern: create a child logger at the start of every handler with the request ID attached. Every log from that handler and the services it calls should use this child logger.

```typescript
// src/functions/invoices/create.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { AppError } from "../../lib/errors";
import { invoiceService } from "../../services/invoices";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const log = logger.child({
    requestId: context.awsRequestId,
    function: context.functionName,
    path: event.rawPath,
    method: event.requestContext.http.method,
  });

  log.info("Request received");

  try {
    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) {
      log.warn("Missing workspaceId in path");
      return res.badRequest("Missing workspaceId");
    }

    const body = event.body ? JSON.parse(event.body) : {};

    log.debug("Creating invoice", { workspaceId, clientId: body.clientId });

    const invoice = await invoiceService.create(workspaceId, body, log);

    log.info("Invoice created", {
      invoiceId: invoice.id,
      workspaceId,
      total: invoice.total,
      currency: invoice.currency,
    });

    return res.created(invoice);
  } catch (err) {
    if (err instanceof SyntaxError) {
      log.warn("Invalid JSON in request body", { err: err.message });
      return res.badRequest("Invalid JSON");
    }
    if (err instanceof AppError) {
      log.warn("Application error", { code: err.code, message: err.message });
      return res.fromAppError(err);
    }
    log.error("Unhandled error", { err });
    return res.internalError();
  }
};
```

Notice that `invoiceService.create` receives `log` as a parameter. This is the cleanest way to propagate request context into services without coupling them to the Lambda runtime:

```typescript
// src/services/invoices.ts
import type { Logger } from "../lib/logger";

export const invoiceService = {
  async create(
    workspaceId: string,
    input: CreateInvoiceInput,
    log: Logger  // receives the request-scoped child logger
  ): Promise<Invoice> {
    log.debug("Looking up client", { clientId: input.clientId });
    // ... implementation
  },
};
```

Every log line from both the handler and the service carries `requestId`, `service`, and `stage` — plus whatever the service adds. When you search CloudWatch for a specific `requestId`, you see the complete story of that request.

### CloudWatch Logs Insights queries

With structured JSON logs, CloudWatch Logs Insights becomes genuinely useful. Here are the queries worth saving:

**Find all errors in the last hour:**
```
fields @timestamp, @logStream, requestId, message, err.message
| filter level = "error"
| sort @timestamp desc
| limit 100
```

**Find slow requests (> 2 seconds):**
```
fields @timestamp, requestId, path, @duration
| filter @type = "REPORT"
| filter @duration > 2000
| sort @duration desc
| limit 50
```

**Find all requests for a specific user:**
```
fields @timestamp, requestId, path, message
| filter userId = "usr_abc123"
| sort @timestamp desc
| limit 100
```

**Invoice creation failures:**
```
fields @timestamp, requestId, message, code
| filter message = "Application error"
| filter code like /INVOICE/
| sort @timestamp desc
| limit 50
```

**P99 latency for a function:**
```
fields @duration
| filter @type = "REPORT"
| stats pct(@duration, 99) as p99, pct(@duration, 95) as p95, avg(@duration) as avg
```

These queries only work when your logs are structured JSON. Text logs give you grep, nothing more.

### Log levels and when to use each

**`debug`**: Fine-grained details useful during development and troubleshooting. Database queries, intermediate values, step-by-step logic. Not logged in production unless `LOG_LEVEL=debug` is set.

**`info`**: Normal operational events. Request received, resource created, background job started. These are the events that tell you the system is working correctly.

**`warn`**: Something unexpected but handled. A validation failure, an optional external service being unavailable (with fallback), a deprecated code path being used.

**`error`**: Something failed. Unhandled exceptions, unexpected states, external service failures without fallback. Every error log should be actionable.

The anti-pattern: logging everything as `info` because you're not sure. That's how you end up drowning in noise when CloudWatch costs start climbing.

---

## 2.6 Error Handling

Two things go wrong with error handling in Lambda functions:

1. All errors look the same from the outside (500, opaque message)
2. Errors are silently swallowed because someone forgot a try/catch

The fix is a small error class hierarchy and a single wrapper pattern.

### Custom error classes

```typescript
// src/lib/errors.ts

export type ErrorCode =
  | "BAD_REQUEST"
  | "UNAUTHORIZED"
  | "FORBIDDEN"
  | "NOT_FOUND"
  | "CONFLICT"
  | "UNPROCESSABLE"
  | "INTERNAL_ERROR"
  // Domain-specific error codes
  | "WORKSPACE_NOT_FOUND"
  | "CLIENT_NOT_FOUND"
  | "PROJECT_NOT_FOUND"
  | "INVOICE_NOT_FOUND"
  | "INVOICE_ALREADY_SENT"
  | "INVOICE_ALREADY_PAID"
  | "WORKSPACE_LIMIT_REACHED"
  | "DUPLICATE_INVOICE_NUMBER";

export class AppError extends Error {
  readonly code: ErrorCode;
  readonly statusCode: number;
  readonly details?: unknown;

  constructor(code: ErrorCode, message: string, options?: {
    statusCode?: number;
    details?: unknown;
    cause?: Error;
  }) {
    super(message, { cause: options?.cause });
    this.name = "AppError";
    this.code = code;
    this.statusCode = options?.statusCode ?? statusCodeForErrorCode(code);
    this.details = options?.details;

    // Maintains proper prototype chain for instanceof checks
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

// Map error codes to sensible default HTTP status codes
function statusCodeForErrorCode(code: ErrorCode): number {
  const map: Record<ErrorCode, number> = {
    BAD_REQUEST: 400,
    UNAUTHORIZED: 401,
    FORBIDDEN: 403,
    NOT_FOUND: 404,
    WORKSPACE_NOT_FOUND: 404,
    CLIENT_NOT_FOUND: 404,
    PROJECT_NOT_FOUND: 404,
    INVOICE_NOT_FOUND: 404,
    CONFLICT: 409,
    DUPLICATE_INVOICE_NUMBER: 409,
    INVOICE_ALREADY_SENT: 409,
    INVOICE_ALREADY_PAID: 409,
    UNPROCESSABLE: 422,
    WORKSPACE_LIMIT_REACHED: 422,
    INTERNAL_ERROR: 500,
  };
  return map[code] ?? 500;
}

// Convenience factory functions — keeps service code readable
export const errors = {
  badRequest: (message: string, details?: unknown) =>
    new AppError("BAD_REQUEST", message, { details }),

  notFound: (resource: string) =>
    new AppError(`${resource.toUpperCase()}_NOT_FOUND` as ErrorCode,
      `${resource} not found`),

  workspaceNotFound: (workspaceId: string) =>
    new AppError("WORKSPACE_NOT_FOUND", `Workspace ${workspaceId} not found`),

  clientNotFound: (clientId: string) =>
    new AppError("CLIENT_NOT_FOUND", `Client ${clientId} not found`),

  invoiceNotFound: (invoiceId: string) =>
    new AppError("INVOICE_NOT_FOUND", `Invoice ${invoiceId} not found`),

  invoiceAlreadySent: (invoiceId: string) =>
    new AppError("INVOICE_ALREADY_SENT",
      `Invoice ${invoiceId} has already been sent and cannot be modified`),

  invoiceAlreadyPaid: (invoiceId: string) =>
    new AppError("INVOICE_ALREADY_PAID",
      `Invoice ${invoiceId} has already been paid`),

  workspaceLimitReached: () =>
    new AppError("WORKSPACE_LIMIT_REACHED",
      "You have reached the workspace limit for your plan. Upgrade to add more workspaces."),

  forbidden: (message = "You do not have permission to perform this action") =>
    new AppError("FORBIDDEN", message),

  unauthorized: () =>
    new AppError("UNAUTHORIZED", "Authentication required"),

  conflict: (message: string) =>
    new AppError("CONFLICT", message),

  internal: (message = "An unexpected error occurred") =>
    new AppError("INTERNAL_ERROR", message, { statusCode: 500 }),
};
```

Now extend the response library to handle `AppError`:

```typescript
// src/lib/response.ts  — add this to the existing file

import { AppError } from "./errors";

// Add to the response library
export const fromAppError = (err: AppError): ApiResponse =>
  json(err.statusCode, {
    error: {
      code: err.code,
      message: err.message,
      ...(err.details !== undefined && { details: err.details }),
    },
  });

// Update res object to include fromAppError
export const res = {
  ok,
  created,
  noContent,
  badRequest,
  unauthorized,
  forbidden,
  notFound,
  conflict,
  unprocessable,
  internalError,
  fromAppError,
};
```

### Service code with proper errors

Here's what the service layer looks like with custom errors:

```typescript
// src/services/invoices.ts
import { errors } from "../lib/errors";
import type { Logger } from "../lib/logger";

export const invoiceService = {
  async send(
    workspaceId: string,
    invoiceId: string,
    log: Logger
  ): Promise<Invoice> {
    const invoice = await invoiceService.findById(workspaceId, invoiceId);

    if (!invoice) {
      throw errors.invoiceNotFound(invoiceId);
    }

    if (invoice.status === "sent") {
      throw errors.invoiceAlreadySent(invoiceId);
    }

    if (invoice.status === "paid") {
      throw errors.invoiceAlreadyPaid(invoiceId);
    }

    if (invoice.status === "cancelled") {
      throw errors.badRequest("Cannot send a cancelled invoice");
    }

    // Send the invoice email...
    log.info("Sending invoice email", { invoiceId, clientId: invoice.clientId });

    const updated = await invoiceService.update(workspaceId, invoiceId, {
      status: "sent",
    });

    return updated;
  },
};
```

The service throws typed errors. The handler catches them and maps to HTTP responses. The error code and status code are defined once, in one place.

### The handler-level try/catch pattern

Every handler needs a try/catch. Every single one. Here's the canonical pattern:

```typescript
export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const log = logger.child({ requestId: context.awsRequestId });

  try {
    // ... handler logic

  } catch (err) {
    // 1. Handle JSON parse errors
    if (err instanceof SyntaxError && err.message.includes("JSON")) {
      log.warn("Invalid JSON in request body");
      return res.badRequest("Request body is not valid JSON");
    }

    // 2. Handle our own errors
    if (err instanceof AppError) {
      // Only log warn for expected errors (not found, forbidden, etc.)
      // Errors that indicate a bug get logged as error
      if (err.statusCode >= 500) {
        log.error("Application error", { code: err.code, err: err.message });
      } else {
        log.warn("Application error", { code: err.code, message: err.message });
      }
      return res.fromAppError(err);
    }

    // 3. Everything else is unexpected — log it fully
    log.error("Unhandled error", {
      err: err instanceof Error ? {
        name: err.name,
        message: err.message,
        stack: err.stack,
      } : String(err),
    });

    return res.internalError();
  }
};
```

Tier 3 (unhandled) is where bugs live. Every time something hits tier 3, it's a signal: either add handling for this error type, or fix the underlying bug.

### Never silently swallow errors

The worst error handling in Lambda:

```typescript
// ❌ The error disappears. You have no idea it happened.
try {
  await doSomething();
} catch {
  return res.internalError();
}
```

At minimum:

```typescript
// ✅ The error is logged before returning
try {
  await doSomething();
} catch (err) {
  log.error("doSomething failed", { err });
  return res.internalError();
}
```

If you're unsure whether to re-throw or handle, re-throw. Let the handler-level catch deal with it. The outer catch has the full error context and knows how to log properly.

### DynamoDB-specific errors

DynamoDB throws its own error types that you'll encounter frequently. Add handling for the common ones:

```typescript
// src/lib/errors.ts — add helpers for AWS errors

import { ConditionalCheckFailedException } from "@aws-sdk/client-dynamodb";

export const isConditionalCheckFailed = (err: unknown): boolean =>
  err instanceof ConditionalCheckFailedException ||
  (err instanceof Error && err.name === "ConditionalCheckFailedException");

export const isTransactionConflict = (err: unknown): boolean =>
  err instanceof Error && err.name === "TransactionCanceledException";
```

Use them in services:

```typescript
// src/services/workspaces.ts
import { isConditionalCheckFailed } from "../lib/errors";

export const workspaceService = {
  async create(input: CreateWorkspaceInput, ownerId: string): Promise<Workspace> {
    const db = getDb();
    const workspace: Workspace = {
      id: `ws_${nanoid()}`,
      ...input,
      ownerId,
      plan: "free",
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };

    try {
      await db.send(new PutCommand({
        TableName: config.TABLE_NAME,
        Item: {
          pk: `WORKSPACE#${workspace.id}`,
          sk: `WORKSPACE#${workspace.id}`,
          ...workspace,
        },
        // Fail if a workspace with this slug already exists
        ConditionExpression: "attribute_not_exists(pk)",
      }));
    } catch (err) {
      if (isConditionalCheckFailed(err)) {
        throw errors.conflict(`A workspace with slug "${input.slug}" already exists`);
      }
      throw err; // Re-throw unexpected DynamoDB errors
    }

    return workspace;
  },
};
```

The service converts the AWS error into an `AppError`, which the handler converts into a 409 response. The error identity is never lost.

---

## 2.7 Environment Variable Validation

Lambda functions use environment variables for configuration: table names, queue URLs, API keys, feature flags. Environment variables have no type. They're all strings. They can be missing. They can be misconfigured.

The default behaviour when an environment variable is missing is that your code throws a confusing error somewhere deep in a library, at runtime, on a real request. The fix is to fail early — at startup, before any request is served.

### Building a typed config module

```typescript
// src/lib/config.ts

type Config = {
  TABLE_NAME: string;
  STRIPE_SECRET_KEY: string;
  STRIPE_WEBHOOK_SECRET: string;
  SES_FROM_ADDRESS: string;
  INVOICE_BUCKET_NAME: string;
  APP_URL: string;
  NODE_ENV: "development" | "production" | "test";
  LOG_LEVEL: "debug" | "info" | "warn" | "error";
  SST_STAGE: string;
};

type RequiredKeys = Exclude<keyof Config, "LOG_LEVEL">;

const DEFAULTS: Partial<Config> = {
  NODE_ENV: "production",
  LOG_LEVEL: "info",
};

let _config: Config | undefined;

export const getConfig = (): Config => {
  if (_config) return _config;

  const errors: string[] = [];

  const get = <K extends keyof Config>(key: K): Config[K] => {
    const value = process.env[key] ?? DEFAULTS[key];
    if (value === undefined || value === "") {
      errors.push(`Missing required environment variable: ${key}`);
      return undefined as unknown as Config[K];
    }
    return value as Config[K];
  };

  const config: Config = {
    TABLE_NAME: get("TABLE_NAME"),
    STRIPE_SECRET_KEY: get("STRIPE_SECRET_KEY"),
    STRIPE_WEBHOOK_SECRET: get("STRIPE_WEBHOOK_SECRET"),
    SES_FROM_ADDRESS: get("SES_FROM_ADDRESS"),
    INVOICE_BUCKET_NAME: get("INVOICE_BUCKET_NAME"),
    APP_URL: get("APP_URL"),
    NODE_ENV: get("NODE_ENV"),
    LOG_LEVEL: process.env.LOG_LEVEL as Config["LOG_LEVEL"] ?? "info",
    SST_STAGE: get("SST_STAGE"),
  };

  if (errors.length > 0) {
    // This fails at startup — before any invocation completes
    throw new Error(
      `Lambda configuration error:\n${errors.map(e => `  - ${e}`).join("\n")}`
    );
  }

  _config = config;
  return _config;
};
```

Call `getConfig()` at the module level in services (not inside the handler function):

```typescript
// src/services/invoices.ts
import { getConfig } from "../lib/config";

// Called once at module load time — fails fast if config is missing
const config = getConfig();

export const invoiceService = {
  async create(...) {
    // config.TABLE_NAME is guaranteed to exist here
    await db.send(new PutCommand({ TableName: config.TABLE_NAME, ... }));
  },
};
```

If `TABLE_NAME` is missing, the Lambda fails during initialisation — before the handler function is ever called. The error is logged clearly and appears in CloudWatch as a startup failure, not a mysterious null reference error 30% of the way through request handling.

### Wiring environment variables with SST

SST's resource linking handles most environment variables automatically. But some things — like your app URL and SES sender address — need to be set explicitly:

```typescript
// sst.config.ts
const api = new sst.aws.ApiGatewayV2("RunwayApi");

// DynamoDB table — linked via resource linking (TABLE_NAME injected automatically)
const table = new sst.aws.Dynamo("RunwayTable", {
  fields: { pk: "string", sk: "string" },
  primaryIndex: { hashKey: "pk", rangeKey: "sk" },
});

// S3 bucket for invoice PDFs
const invoiceBucket = new sst.aws.Bucket("InvoiceBucket");

// Secrets — set via `npx sst secret set`
const stripeSecretKey = new sst.Secret("StripeSecretKey");
const stripeWebhookSecret = new sst.Secret("StripeWebhookSecret");

// Register routes with the right environment
const functionConfig = {
  link: [table, invoiceBucket, stripeSecretKey, stripeWebhookSecret],
  environment: {
    SST_STAGE: $app.stage,
    APP_URL: $app.stage === "production"
      ? "https://app.runway.so"
      : `https://${$app.stage}.runway.so`,
    SES_FROM_ADDRESS: "invoices@runway.so",
    LOG_LEVEL: $app.stage === "production" ? "info" : "debug",
  },
};

api.route("POST /workspaces", {
  handler: "src/functions/workspaces/create.handler",
  ...functionConfig,
});

api.route("GET /workspaces/{workspaceId}/invoices", {
  handler: "src/functions/invoices/list.handler",
  ...functionConfig,
});

api.route("POST /workspaces/{workspaceId}/invoices", {
  handler: "src/functions/invoices/create.handler",
  ...functionConfig,
});
```

When SST links `table` to a function, it injects `TABLE_NAME` automatically (using `Resource.RunwayTable.name` internally). The `environment` block adds the non-linked variables. Together, `getConfig()` gets everything it needs.

### Secrets vs environment variables

**Environment variables** are fine for:
- Feature flags
- Configuration that varies by stage (log level, app URL)
- DynamoDB table names, queue URLs (injected by SST linking — these aren't secret)
- Non-sensitive settings

**Secrets** (SST's `sst.Secret`, backed by AWS SSM Parameter Store) are required for:
- API keys (Stripe, external services)
- Webhook secrets
- JWT signing keys
- Database passwords (Chapter 5)
- Anything that would be damaging if leaked in logs or config

Never put secrets in environment variables that are committed to source control. The `.env` file in your repo is for local development only, not for staging or production secrets.

For local development with `sst dev`, SST pulls secrets from SSM automatically — you set them once with `npx sst secret set` and they're available in all your functions without touching a `.env` file.

---

## 2.8 Testing Lambda Functions

Testing Lambda functions has a reputation for being painful. It doesn't have to be.

The secret: **unit test the service layer, integration test the handlers**. Don't unit test handlers — they're thin wiring code and the unit tests end up being brittle mocks of the event structure. Test the business logic directly, and test the full request/response flow end-to-end.

### Setting up Vitest

Vitest is the testing framework of choice for SST projects. It's fast, ESM-native, and has first-class TypeScript support without any transpilation configuration.

```bash
npm install --save-dev vitest @vitest/coverage-v8
```

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
    coverage: {
      reporter: ["text", "lcov"],
      include: ["src/**/*.ts"],
      exclude: ["src/functions/**", "src/test/**"],
    },
  },
});
```

Note the coverage exclusion: we exclude `src/functions/` from coverage because handler files are thin wiring code — there's almost nothing to unit test there. Coverage should measure the service layer.

```typescript
// src/test/setup.ts
// Set required environment variables for tests
process.env.TABLE_NAME = "runway-test-table";
process.env.STRIPE_SECRET_KEY = "sk_test_fake";
process.env.STRIPE_WEBHOOK_SECRET = "whsec_fake";
process.env.SES_FROM_ADDRESS = "test@runway.so";
process.env.INVOICE_BUCKET_NAME = "runway-test-bucket";
process.env.APP_URL = "http://localhost:3000";
process.env.SST_STAGE = "test";
process.env.NODE_ENV = "test";
process.env.LOG_LEVEL = "error"; // Suppress logs during tests
```

### Unit testing services

Service layer tests are pure TypeScript unit tests. You mock the database and external services, and test the business logic:

```typescript
// src/services/__tests__/invoices.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { invoiceService } from "../invoices";
import { errors } from "../../lib/errors";
import { getDb } from "../../lib/db/dynamodb";

// Mock the database client
vi.mock("../../lib/db/dynamodb", () => ({
  getDb: vi.fn(),
  PutCommand: vi.fn((args) => args),
  GetCommand: vi.fn((args) => args),
  QueryCommand: vi.fn((args) => args),
  UpdateCommand: vi.fn((args) => args),
}));

const mockDb = {
  send: vi.fn(),
};

// Silent logger for tests
const mockLog = {
  debug: vi.fn(),
  info: vi.fn(),
  warn: vi.fn(),
  error: vi.fn(),
  child: vi.fn().mockReturnThis(),
};

beforeEach(() => {
  vi.clearAllMocks();
  vi.mocked(getDb).mockReturnValue(mockDb as any);
});

describe("invoiceService.create", () => {
  const workspaceId = "ws_test123";

  const validInput = {
    clientId: "cli_test456",
    lineItems: [
      {
        description: "Website redesign",
        quantity: 1,
        unitPrice: 5000_00, // pence
        amount: 5000_00,
      },
    ],
    currency: "GBP",
    issueDate: "2025-03-01",
    dueDate: "2025-03-31",
  };

  it("creates an invoice with correct totals", async () => {
    mockDb.send.mockResolvedValue({});

    const invoice = await invoiceService.create(workspaceId, validInput, mockLog as any);

    expect(invoice.subtotal).toBe(500_000);
    expect(invoice.tax).toBe(100_000); // 20% VAT
    expect(invoice.total).toBe(600_000);
    expect(invoice.status).toBe("draft");
    expect(invoice.workspaceId).toBe(workspaceId);
    expect(invoice.clientId).toBe(validInput.clientId);
  });

  it("generates an invoice number", async () => {
    mockDb.send.mockResolvedValue({});

    const invoice = await invoiceService.create(workspaceId, validInput, mockLog as any);

    expect(invoice.invoiceNumber).toMatch(/^INV-\d{4}$/);
  });

  it("sets correct timestamps", async () => {
    mockDb.send.mockResolvedValue({});

    const before = new Date();
    const invoice = await invoiceService.create(workspaceId, validInput, mockLog as any);
    const after = new Date();

    const createdAt = new Date(invoice.createdAt);
    expect(createdAt.getTime()).toBeGreaterThanOrEqual(before.getTime());
    expect(createdAt.getTime()).toBeLessThanOrEqual(after.getTime());
  });

  it("throws conflict when invoice already exists", async () => {
    const { ConditionalCheckFailedException } = await import("@aws-sdk/client-dynamodb");
    mockDb.send.mockRejectedValue(new ConditionalCheckFailedException({ message: "", $metadata: {} }));

    await expect(
      invoiceService.create(workspaceId, validInput, mockLog as any)
    ).rejects.toMatchObject({
      code: "CONFLICT",
    });
  });
});

describe("invoiceService.send", () => {
  const workspaceId = "ws_test123";
  const invoiceId = "inv_test789";

  const draftInvoice = {
    id: invoiceId,
    workspaceId,
    status: "draft",
    clientId: "cli_test456",
    // ... other fields
  };

  it("throws NOT_FOUND when invoice does not exist", async () => {
    mockDb.send.mockResolvedValue({ Item: undefined });

    await expect(
      invoiceService.send(workspaceId, invoiceId, mockLog as any)
    ).rejects.toMatchObject({
      code: "INVOICE_NOT_FOUND",
    });
  });

  it("throws INVOICE_ALREADY_SENT when invoice is already sent", async () => {
    mockDb.send.mockResolvedValue({
      Item: { ...draftInvoice, status: "sent" },
    });

    await expect(
      invoiceService.send(workspaceId, invoiceId, mockLog as any)
    ).rejects.toMatchObject({
      code: "INVOICE_ALREADY_SENT",
    });
  });

  it("throws INVOICE_ALREADY_PAID when invoice is paid", async () => {
    mockDb.send.mockResolvedValue({
      Item: { ...draftInvoice, status: "paid" },
    });

    await expect(
      invoiceService.send(workspaceId, invoiceId, mockLog as any)
    ).rejects.toMatchObject({
      code: "INVOICE_ALREADY_PAID",
    });
  });
});
```

### Testing the error classes

Error classes deserve their own tests — particularly the mapping from error code to HTTP status:

```typescript
// src/lib/__tests__/errors.test.ts
import { describe, it, expect } from "vitest";
import { AppError, errors } from "../errors";

describe("AppError", () => {
  it("has correct default status codes", () => {
    expect(errors.invoiceNotFound("inv_1").statusCode).toBe(404);
    expect(errors.invoiceAlreadySent("inv_1").statusCode).toBe(409);
    expect(errors.forbidden().statusCode).toBe(403);
    expect(errors.unauthorized().statusCode).toBe(401);
    expect(errors.workspaceLimitReached().statusCode).toBe(422);
  });

  it("is instanceof AppError", () => {
    const err = errors.invoiceNotFound("inv_1");
    expect(err).toBeInstanceOf(AppError);
    expect(err).toBeInstanceOf(Error);
  });

  it("includes error code", () => {
    const err = errors.invoiceNotFound("inv_1");
    expect(err.code).toBe("INVOICE_NOT_FOUND");
  });
});
```

### Testing the response library

```typescript
// src/lib/__tests__/response.test.ts
import { describe, it, expect } from "vitest";
import { res } from "../response";
import { AppError, errors } from "../errors";

describe("res.ok", () => {
  it("returns 200 with data wrapper", () => {
    const response = res.ok({ id: "123" });
    expect(response.statusCode).toBe(200);
    expect(JSON.parse(response.body)).toEqual({ data: { id: "123" } });
  });
});

describe("res.notFound", () => {
  it("returns 404 with NOT_FOUND code", () => {
    const response = res.notFound("Invoice");
    expect(response.statusCode).toBe(404);
    const body = JSON.parse(response.body);
    expect(body.error.code).toBe("NOT_FOUND");
    expect(body.error.message).toBe("Invoice not found");
  });
});

describe("res.fromAppError", () => {
  it("maps AppError to correct status code", () => {
    const err = errors.invoiceAlreadySent("inv_1");
    const response = res.fromAppError(err);
    expect(response.statusCode).toBe(409);
    const body = JSON.parse(response.body);
    expect(body.error.code).toBe("INVOICE_ALREADY_SENT");
  });

  it("includes details when present", () => {
    const err = errors.badRequest("Validation failed", { field: "clientId" });
    const response = res.fromAppError(err);
    const body = JSON.parse(response.body);
    expect(body.error.details).toEqual({ field: "clientId" });
  });
});
```

### Testing handlers with a mock event

When you do need to test a handler (for integration between the handler and service, or testing the error handling flow), create a factory function for mock events:

```typescript
// src/test/helpers.ts
import type { APIGatewayProxyEventV2, Context } from "aws-lambda";

export const mockContext = (overrides: Partial<Context> = {}): Context => ({
  callbackWaitsForEmptyEventLoop: false,
  functionName: "test-function",
  functionVersion: "$LATEST",
  invokedFunctionArn: "arn:aws:lambda:eu-west-1:123456789:function:test",
  memoryLimitInMB: "512",
  awsRequestId: "test-request-id-1234",
  logGroupName: "/aws/lambda/test",
  logStreamName: "2025/03/01/[$LATEST]abc123",
  getRemainingTimeInMillis: () => 30000,
  done: () => {},
  fail: () => {},
  succeed: () => {},
  ...overrides,
});

type MockEventOptions = {
  method?: string;
  path?: string;
  pathParameters?: Record<string, string>;
  queryStringParameters?: Record<string, string>;
  body?: unknown;
  headers?: Record<string, string>;
};

export const mockApiEvent = (options: MockEventOptions = {}): APIGatewayProxyEventV2 => {
  const {
    method = "GET",
    path = "/",
    pathParameters,
    queryStringParameters,
    body,
    headers = {},
  } = options;

  return {
    version: "2.0",
    routeKey: `${method} ${path}`,
    rawPath: path,
    rawQueryString: queryStringParameters
      ? new URLSearchParams(queryStringParameters).toString()
      : "",
    headers: {
      "content-type": "application/json",
      ...headers,
    },
    requestContext: {
      accountId: "123456789",
      apiId: "test-api",
      domainName: "test.execute-api.eu-west-1.amazonaws.com",
      domainPrefix: "test",
      http: {
        method,
        path,
        protocol: "HTTP/1.1",
        sourceIp: "127.0.0.1",
        userAgent: "test",
      },
      requestId: "test-request-id-1234",
      routeKey: `${method} ${path}`,
      stage: "$default",
      time: "01/Mar/2025:10:00:00 +0000",
      timeEpoch: 1740820800000,
    },
    pathParameters,
    queryStringParameters,
    body: body !== undefined ? JSON.stringify(body) : undefined,
    isBase64Encoded: false,
  };
};
```

```typescript
// src/functions/__tests__/invoices.create.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { handler } from "../invoices/create";
import { mockApiEvent, mockContext } from "../../test/helpers";
import * as invoiceServiceModule from "../../services/invoices";

vi.mock("../../services/invoices");

describe("POST /workspaces/{workspaceId}/invoices", () => {
  const workspaceId = "ws_test123";

  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("returns 400 when workspaceId is missing", async () => {
    const event = mockApiEvent({
      method: "POST",
      path: "/workspaces//invoices",
      // No pathParameters
    });

    const result = await handler(event, mockContext(), () => {});
    expect(result?.statusCode).toBe(400);
  });

  it("returns 201 when invoice is created successfully", async () => {
    const createdInvoice = {
      id: "inv_new123",
      workspaceId,
      status: "draft",
      total: 600_000,
    };

    vi.mocked(invoiceServiceModule.invoiceService.create).mockResolvedValue(
      createdInvoice as any
    );

    const event = mockApiEvent({
      method: "POST",
      path: `/workspaces/${workspaceId}/invoices`,
      pathParameters: { workspaceId },
      body: {
        clientId: "cli_test456",
        lineItems: [{ description: "Work", quantity: 1, unitPrice: 500000, amount: 500000 }],
        issueDate: "2025-03-01",
        dueDate: "2025-03-31",
      },
    });

    const result = await handler(event, mockContext(), () => {});
    expect(result?.statusCode).toBe(201);
    const body = JSON.parse(result?.body ?? "{}");
    expect(body.data.id).toBe("inv_new123");
  });

  it("returns 404 when invoice service throws NOT_FOUND", async () => {
    const { errors } = await import("../../lib/errors");
    vi.mocked(invoiceServiceModule.invoiceService.create).mockRejectedValue(
      errors.clientNotFound("cli_missing")
    );

    const event = mockApiEvent({
      method: "POST",
      path: `/workspaces/${workspaceId}/invoices`,
      pathParameters: { workspaceId },
      body: {
        clientId: "cli_missing",
        lineItems: [],
        issueDate: "2025-03-01",
        dueDate: "2025-03-31",
      },
    });

    const result = await handler(event, mockContext(), () => {});
    expect(result?.statusCode).toBe(404);
  });

  it("returns 400 for invalid JSON body", async () => {
    const event = mockApiEvent({
      method: "POST",
      path: `/workspaces/${workspaceId}/invoices`,
      pathParameters: { workspaceId },
    });
    // Manually override body with invalid JSON
    event.body = "{ this is not json }";

    const result = await handler(event, mockContext(), () => {});
    expect(result?.statusCode).toBe(400);
  });
});
```

### What to mock and what not to mock

**Mock:**
- Database clients (`getDb()`) — you're testing business logic, not DynamoDB
- External API clients (Stripe, SES) — tests should be deterministic and not cost money
- Time (`Date.now()`, `new Date()`) — when business logic depends on the current time

**Don't mock:**
- Your own services (in service-layer unit tests, test the real service)
- Error classes — test the real ones
- Response helpers — test the real ones
- Pure utility functions

**Test against real AWS** for integration tests using `sst dev`. SST's live development environment means your integration tests can call the real API Gateway URL, which routes to your local handler, which talks to the real DynamoDB table. No LocalStack, no complex mocking of AWS responses. This is the right place to verify the full stack:

```typescript
// src/test/integration/invoices.integration.test.ts
// Run with: npx sst dev (in another terminal), then vitest --run integration

import { describe, it, expect } from "vitest";

// The API URL is set by sst dev output
const API_URL = process.env.API_URL ?? "http://localhost:3000";

describe("Invoice API (integration)", () => {
  it("creates and retrieves an invoice", async () => {
    const workspaceId = "ws_integtest";

    const createRes = await fetch(`${API_URL}/workspaces/${workspaceId}/invoices`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        clientId: "cli_integtest",
        lineItems: [{ description: "Test work", quantity: 1, unitPrice: 100, amount: 100 }],
        issueDate: "2025-03-01",
        dueDate: "2025-03-31",
      }),
    });

    expect(createRes.status).toBe(201);
    const { data: created } = await createRes.json();

    const getRes = await fetch(
      `${API_URL}/workspaces/${workspaceId}/invoices/${created.id}`
    );
    expect(getRes.status).toBe(200);
    const { data: fetched } = await getRes.json();
    expect(fetched.id).toBe(created.id);
  });
});
```

Integration tests against real AWS surface a class of bugs that unit tests can't catch: IAM permission errors, DynamoDB schema mismatches, API Gateway routing issues, SST resource linking problems. Run them before every production deploy.

---

## Putting It All Together

Let's assemble a complete, production-ready handler using every pattern from this chapter:

```typescript
// src/functions/invoices/create.ts — the complete version

import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { AppError } from "../../lib/errors";
import { invoiceService } from "../../services/invoices";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  // Child logger with request context baked in
  const log = logger.child({
    requestId: context.awsRequestId,
    function: context.functionName,
    path: event.rawPath,
    method: event.requestContext.http.method,
  });

  log.info("Request received");

  try {
    // Extract path parameters
    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) {
      log.warn("Missing workspaceId");
      return res.badRequest("workspaceId path parameter is required");
    }

    // Parse request body
    let body: unknown;
    try {
      body = event.body ? JSON.parse(event.body) : {};
    } catch {
      log.warn("Invalid JSON in request body");
      return res.badRequest("Request body must be valid JSON");
    }

    // Call service (full validation moves to middleware in Chapter 3)
    const invoice = await invoiceService.create(workspaceId, body as any, log);

    log.info("Invoice created", {
      invoiceId: invoice.id,
      workspaceId,
      total: invoice.total,
      currency: invoice.currency,
    });

    return res.created(invoice);

  } catch (err) {
    if (err instanceof AppError) {
      if (err.statusCode >= 500) {
        log.error("AppError (server)", {
          code: err.code,
          message: err.message,
          stack: err.stack,
        });
      } else {
        log.warn("AppError (client)", {
          code: err.code,
          message: err.message,
        });
      }
      return res.fromAppError(err);
    }

    log.error("Unhandled exception", {
      err: err instanceof Error ? {
        name: err.name,
        message: err.message,
        stack: err.stack,
      } : String(err),
    });
    return res.internalError();
  }
};
```

And the SST configuration that deploys it:

```typescript
// sst.config.ts — final version for Chapter 2
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
    const isProd = $app.stage === "production";

    const api = new sst.aws.ApiGatewayV2("RunwayApi", {
      accessLog: isProd ? { retention: "1 month" } : false,
    });

    const table = new sst.aws.Dynamo("RunwayTable", {
      fields: { pk: "string", sk: "string" },
      primaryIndex: { hashKey: "pk", rangeKey: "sk" },
    });

    const invoiceBucket = new sst.aws.Bucket("InvoiceBucket");

    const stripeSecretKey = new sst.Secret("StripeSecretKey");
    const stripeWebhookSecret = new sst.Secret("StripeWebhookSecret");

    const sharedFunctionConfig = {
      link: [table, invoiceBucket, stripeSecretKey, stripeWebhookSecret],
      environment: {
        SST_STAGE: $app.stage,
        APP_URL: isProd
          ? "https://app.runway.so"
          : `https://${$app.stage}.runway.so`,
        SES_FROM_ADDRESS: "invoices@runway.so",
        LOG_LEVEL: isProd ? "info" : "debug",
      },
      // Tune Lambda settings
      memory: "512 MB",
      timeout: "30 seconds",
    } as const;

    // Health check (from Chapter 1)
    api.route("GET /health", {
      handler: "src/functions/health.handler",
    });

    // Workspaces
    api.route("POST /workspaces", {
      handler: "src/functions/workspaces/create.handler",
      ...sharedFunctionConfig,
    });

    api.route("GET /workspaces/{workspaceId}", {
      handler: "src/functions/workspaces/get.handler",
      ...sharedFunctionConfig,
    });

    api.route("PATCH /workspaces/{workspaceId}", {
      handler: "src/functions/workspaces/update.handler",
      ...sharedFunctionConfig,
    });

    // Clients
    api.route("GET /workspaces/{workspaceId}/clients", {
      handler: "src/functions/clients/list.handler",
      ...sharedFunctionConfig,
    });

    api.route("POST /workspaces/{workspaceId}/clients", {
      handler: "src/functions/clients/create.handler",
      ...sharedFunctionConfig,
    });

    api.route("GET /workspaces/{workspaceId}/clients/{clientId}", {
      handler: "src/functions/clients/get.handler",
      ...sharedFunctionConfig,
    });

    // Projects
    api.route("GET /workspaces/{workspaceId}/projects", {
      handler: "src/functions/projects/list.handler",
      ...sharedFunctionConfig,
    });

    api.route("POST /workspaces/{workspaceId}/projects", {
      handler: "src/functions/projects/create.handler",
      ...sharedFunctionConfig,
    });

    // Invoices
    api.route("GET /workspaces/{workspaceId}/invoices", {
      handler: "src/functions/invoices/list.handler",
      ...sharedFunctionConfig,
    });

    api.route("POST /workspaces/{workspaceId}/invoices", {
      handler: "src/functions/invoices/create.handler",
      ...sharedFunctionConfig,
    });

    api.route("GET /workspaces/{workspaceId}/invoices/{invoiceId}", {
      handler: "src/functions/invoices/get.handler",
      ...sharedFunctionConfig,
    });

    api.route("PATCH /workspaces/{workspaceId}/invoices/{invoiceId}", {
      handler: "src/functions/invoices/update.handler",
      ...sharedFunctionConfig,
    });

    api.route("POST /workspaces/{workspaceId}/invoices/{invoiceId}/send", {
      handler: "src/functions/invoices/send.handler",
      ...sharedFunctionConfig,
    });

    return {
      api: api.url,
    };
  },
});
```

---

## The Layer We've Built

Take a step back and look at what we have now.

Every handler in Runway follows the same pattern. Same error handling shape. Same logging structure. Same response format. When something breaks, you open CloudWatch, filter by `requestId`, and you see the complete story of that request from start to finish — the method, the path, the function name, every service call, the final outcome.

When a client's API breaks, you know immediately if it's a 4xx (they're doing something wrong) or a 5xx (we're doing something wrong). You know exactly which error code and status code they received. The `AppError` class means the error is never ambiguous — it has a code, a status, and a message, and they're all set in the same place.

The service layer has no Lambda contamination. You can run the service functions in a test, a cron job, a CLI script, or a different Lambda handler without changing a line. The handler is the adapter that translates between Lambda and your business logic.

Here's the state of the system:

```
Chapter 1:  [API Gateway] ─────────────────────→ [Lambda: GET /health]
Chapter 2:  [API Gateway] → [Handlers (typed)] → [Services (pure)]
                                   ↑                      ↓
                            [Response helpers]      [Error classes]
                            [Structured logging]    [Config validation]
                            [Try/catch pattern]     [DB clients (lazy)]
```

The patterns are in place. Chapter 3 adds the final layer of the handler tier: request validation with Zod. Right now, handlers trust whatever arrives in `event.body`. That's about to change — you'll define a schema, validate the input, and get automatic 422 responses with field-level error details, without touching your service code.

---

> **The code for this chapter** is available at `02-lambda-functions/` in the companion repository. The complete file structure matches what we've built here. Run `npm install && npx sst dev` to start the local environment, then try the integration tests with `vitest --run src/test/integration`.
