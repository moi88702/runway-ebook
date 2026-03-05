# Chapter 3: Type-Safe APIs with API Gateway

In Chapter 2 we built handlers that don't suck. Proper typing, structured logging, consistent error shapes, lazy-initialised clients, a service layer completely decoupled from Lambda. The foundation is solid.

But there's a gap. Look at the invoice creation handler from the end of Chapter 2:

```typescript
let body: unknown;
try {
  body = event.body ? JSON.parse(event.body) : {};
} catch {
  return res.badRequest("Request body must be valid JSON");
}

const invoice = await invoiceService.create(workspaceId, body as any, log);
```

`body as any`. That cast is a lie. We've told TypeScript to stop checking. Whatever the client sends, however wrong, however malformed, gets passed directly to the service. Right now the service doesn't validate either — it trusts the input it receives.

This is how you end up creating an invoice with a negative total, or a due date before the issue date, or a line item with no description. Not because a developer made a mistake, but because a client sent garbage and we had no defence.

Chapter 3 closes that gap. By the end, every route in Runway has a Zod schema. That schema is the single source of truth: it validates incoming requests, generates TypeScript types, and produces machine-parseable validation errors that clients can display field by field. The `as any` disappears. The `body: unknown` becomes `body: CreateInvoiceInput`.

This isn't ceremony. It's the difference between an API that's resilient and one that's fragile.

---

## 3.1 The Event Shape You're Working With

`sst.aws.ApiGatewayV2` creates an **HTTP API** (API Gateway v2). This is the right choice for Runway: 3.5× cheaper than the older REST API, ~50ms lower p99 latency, and native JWT authoriser support for Cognito. The only time you'd reach for a REST API is if you need built-in API key usage plans or VTL request transformation — neither applies here.

One thing to know before reading the type definitions: the HTTP API event shape differs from the REST API shape in a few places that catch people out.

```typescript
// HTTP API v2 — APIGatewayProxyEventV2
event.requestContext.http.method  // "POST" — nested, not event.httpMethod
event.body                         // string | undefined — not null
event.rawQueryString               // always a string, never null
```

Always use `APIGatewayProxyHandlerV2` (and the corresponding `V2` event/result types). Using the v1 types with an HTTP API means TypeScript won't catch property access bugs at compile time.

---

## 3.2 Route Handlers and Path Parameters

Before we get to validation middleware, let's nail the mechanics of extracting data from an HTTP API event. Every handler does this, and it's worth doing it right.

### Path parameters

Route parameters are defined in SST with curly braces:

```typescript
api.route("GET /workspaces/{workspaceId}/invoices/{invoiceId}", {
  handler: "src/functions/invoices/get.handler",
});
```

In the handler, they arrive in `event.pathParameters`:

The type of `pathParameters` is `Record<string, string | undefined> | undefined`, so every access needs optional chaining. Both values can be `undefined` in TypeScript, though in practice API Gateway wouldn't route here without them. The guard narrows both to `string`.

```typescript
export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const workspaceId = event.pathParameters?.workspaceId;
  const invoiceId = event.pathParameters?.invoiceId;

  if (!workspaceId || !invoiceId) {
    return res.badRequest("Missing path parameters");
  }
};
```

Path parameters from a matched route are always present if the route matched — API Gateway wouldn't route to this handler without both `workspaceId` and `invoiceId` in the URL. But the TypeScript type is `string | undefined` to account for routes where the parameter might not exist. The guard is still good practice — it tells TypeScript the type has been narrowed.

One important thing: path parameters from API Gateway HTTP API are **not URL-decoded** in the raw event. If a parameter contains encoded characters (like `%20` for space), you need to decode it:

```typescript
const workspaceId = event.pathParameters?.workspaceId
  ? decodeURIComponent(event.pathParameters.workspaceId)
  : undefined;
```

In practice, IDs in Runway (`ws_abc123`, `inv_xyz789`) never contain encoded characters. But if you use slugs or user-provided identifiers as path parameters, decode them.

### Query strings

Query string parameters arrive in `event.queryStringParameters`. For HTTP API v2, this is always an object (never null), but it's `Record<string, string | undefined>`:

For a route like `GET /workspaces/{workspaceId}/invoices?status=draft&limit=20&cursor=abc`, every query parameter arrives as `string | undefined`. Numeric parameters need explicit coercion — `parseInt` with `isNaN` validation is the pattern.

```typescript
export const handler: APIGatewayProxyHandlerV2 = async (event) => {
  const status = event.queryStringParameters?.status;
  const limitStr = event.queryStringParameters?.limit;
  const cursor = event.queryStringParameters?.cursor;

  const limit = limitStr ? parseInt(limitStr, 10) : 20;
  if (isNaN(limit) || limit < 1 || limit > 100) {
    return res.badRequest("limit must be a number between 1 and 100");
  }
};
```

For multi-value query parameters (same key multiple times: `?status=draft&status=sent`), HTTP API combines them into a comma-separated string: `"draft,sent"`. You split them yourself:

```typescript
const statusParam = event.queryStringParameters?.status;
const statuses = statusParam ? statusParam.split(",") : [];
```

This manual parsing is exactly why we need a validation layer. Writing these coercions and guards by hand in every handler is tedious, error-prone, and inconsistent. Zod handles all of it.

### Request bodies

HTTP API passes the body as a string (or undefined for requests without a body). For JSON APIs:

Always wrap `JSON.parse` in a try/catch — a malformed body throws a `SyntaxError`. After parsing, `body` is `unknown`; TypeScript won't let you access properties without narrowing it first, which is exactly why we need schema validation next.

```typescript
export const handler: APIGatewayProxyHandlerV2 = async (event) => {
  let body: unknown;
  try {
    body = event.body ? JSON.parse(event.body) : {};
  } catch {
    return res.badRequest("Request body must be valid JSON");
  }
};
```

Some clients send base64-encoded bodies (API Gateway does this for binary content types). The `isBase64Encoded` flag tells you:

```typescript
let rawBody: string | undefined;
if (event.isBase64Encoded && event.body) {
  rawBody = Buffer.from(event.body, "base64").toString("utf-8");
} else {
  rawBody = event.body;
}
```

For Runway's JSON API, we can ignore base64 — clients send `Content-Type: application/json` and API Gateway passes the string through as-is.

### Headers

Headers in HTTP API v2 are always lowercase keys (API Gateway normalises them):

```typescript
const contentType = event.headers["content-type"];  // not "Content-Type"
const authHeader = event.headers["authorization"];   // not "Authorization"
const userId = event.headers["x-user-id"];           // custom headers lowercase too
```

The TypeScript type is `Record<string, string | undefined>`, so any access can return undefined.

### Building an input extraction utility

The pattern of extracting and coercing inputs is repetitive. A small utility makes handlers cleaner. Create `src/lib/event.ts` and build it up function by function.

**Parsing the request body**

API Gateway can base64-encode the body (for binary payloads or certain content types). `parseBody` handles both cases, returning `undefined` for an empty body and letting `JSON.parse` throw naturally on malformed JSON — you'll catch that at the handler level and return a 400.

```typescript
// src/lib/event.ts
import type { APIGatewayProxyEventV2 } from "aws-lambda";

export const parseBody = (event: APIGatewayProxyEventV2): unknown => {
  if (!event.body) return undefined;

  const raw = event.isBase64Encoded
    ? Buffer.from(event.body, "base64").toString("utf-8")
    : event.body;

  return JSON.parse(raw); // throws SyntaxError on invalid JSON
};
```

**Requiring a path parameter**

When a route matches `/invoices/:invoiceId`, API Gateway guarantees the parameter exists — but the TypeScript types say it could be `undefined`. `requirePathParam` narrows that to a `string` and throws with a clear message if the impossible happens. It also decodes URI encoding so handlers always get clean values.

```typescript
export const requirePathParam = (
  event: APIGatewayProxyEventV2,
  name: string
): string => {
  const value = event.pathParameters?.[name];
  if (!value) {
    throw new Error(`Missing required path parameter: ${name}`);
  }
  return decodeURIComponent(value);
};
```

**Getting an optional path parameter**

Some routes use optional segments. `getPathParam` returns `string | undefined` without throwing, so you can branch on presence in the handler.

```typescript
export const getPathParam = (
  event: APIGatewayProxyEventV2,
  name: string
): string | undefined => {
  const value = event.pathParameters?.[name];
  return value ? decodeURIComponent(value) : undefined;
};
```

**Reading a query string parameter**

Query parameters are always strings (or absent). `getQuery` is a thin wrapper that makes the intent explicit and avoids optional chaining noise in every handler.

```typescript
export const getQuery = (
  event: APIGatewayProxyEventV2,
  name: string
): string | undefined => {
  return event.queryStringParameters?.[name];
};
```

**Reading a multi-value query parameter**

For list filters like `?status=draft,sent`, `getQueryArray` splits on commas and filters empty strings, returning a clean `string[]`. If the parameter is absent it returns an empty array, so handlers can always safely iterate.

```typescript
export const getQueryArray = (
  event: APIGatewayProxyEventV2,
  name: string
): string[] => {
  const value = event.queryStringParameters?.[name];
  return value ? value.split(",").filter(Boolean) : [];
};
```

---

## 3.3 Request Validation with Zod

Install Zod:

```bash
npm install zod
```

Zod is a TypeScript-first schema validation library. You define a schema, Zod validates data against it, and if validation fails, you get a structured error describing exactly what was wrong. The schema also serves as the TypeScript type — you write the shape once and get runtime validation and compile-time types for free.

**Defining the schema**

Each field gets a validator that describes exactly what's acceptable. Required strings need `.min(1)` or they'd pass on empty string. Line items are an array with `.min(1)` so you can't create an invoice with no items. Currency defaults to `"GBP"` if omitted. Dates are validated against a regex so garbage strings can't sneak through.

```typescript
import { z } from "zod";

const CreateInvoiceSchema = z.object({
  clientId: z.string().min(1),
  projectId: z.string().optional(),
  lineItems: z.array(z.object({
    description: z.string().min(1),
    quantity: z.number().positive(),
    unitPriceCents: z.number().int().nonnegative(),
    amountCents: z.number().int().nonnegative(),
  })).min(1),
  currency: z.enum(["GBP", "USD", "EUR"]).default("GBP"),
  issueDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, "Must be YYYY-MM-DD"),
  dueDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, "Must be YYYY-MM-DD"),
  notes: z.string().max(2000).optional(),
});
```

**Deriving the TypeScript type**

`z.infer<>` extracts a TypeScript type directly from the schema. You never write the type separately — the schema is the single source of truth, and TypeScript figures out the rest. Optional fields become `string | undefined`, the currency enum becomes `"GBP" | "USD" | "EUR"`, and so on.

```typescript
type CreateInvoiceInput = z.infer<typeof CreateInvoiceSchema>;
```

**Validating input**

`safeParse` never throws. It returns a discriminated union: either `{ success: true, data: CreateInvoiceInput }` or `{ success: false, error: ZodError }`. The error's `issues` array is structured — each entry has a `path` pointing to the offending field and a `message` describing the problem. That structure maps directly to what a frontend needs to show per-field errors.

```typescript
const result = CreateInvoiceSchema.safeParse(unknownBody);

if (!result.success) {
  console.log(result.error.issues);
  // [
  //   { path: ["clientId"], message: "Required", code: "invalid_type" },
  //   { path: ["lineItems", 0, "description"], message: "String must contain at least 1 character(s)", code: "too_small" }
  // ]
} else {
  const input = result.data; // typed as CreateInvoiceInput
}
```

One source of truth (the schema), two outputs (runtime validation + TypeScript types), structured errors that a client can display field by field.

### Zod patterns used in Runway

The patterns that actually appear in this book's schemas:

```typescript
// Strings — the three modifiers you'll reach for most
z.string().min(1)                          // non-empty
z.string().trim().max(100)                 // trim then limit
z.string().regex(/^[a-z0-9-]+$/, "msg")   // format validation

// Numbers — query params arrive as strings, coerce them
z.number().int().nonnegative()             // e.g. amountCents
z.coerce.number().int().positive().max(100) // query string limit param

// Enums — produces a union type automatically
z.enum(["draft", "sent", "paid", "overdue", "cancelled"])

// Optional fields and defaults
z.string().optional()          // string | undefined
z.string().default("GBP")      // undefined → "GBP"

// Arrays with minimum length
z.array(LineItemSchema).min(1) // at least one item required

// Cross-field validation with superRefine
z.object({ issueDate: z.string(), dueDate: z.string() })
  .superRefine((data, ctx) => {
    if (data.dueDate <= data.issueDate) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        path: ["dueDate"],
        message: "Due date must be after issue date",
      });
    }
  });
```

`z.coerce.number()` is the key one for query string parameters — they always arrive as strings, and coerce handles the conversion before validation runs. For everything else Zod supports, the [Zod docs](https://zod.dev) are comprehensive.

### Runway's schemas

Now let's write the real schemas for Runway. These live in a new directory:

```typescript
// src/schemas/workspace.ts
import { z } from "zod";

export const WorkspaceIdSchema = z.string()
  .min(1)
  .regex(/^ws_[a-zA-Z0-9]+$/, "Invalid workspace ID format");

export const CreateWorkspaceSchema = z.object({
  name: z.string().trim().min(1, "Name is required").max(100, "Name must be 100 characters or less"),
  slug: z.string()
    .trim()
    .min(2, "Slug must be at least 2 characters")
    .max(50, "Slug must be 50 characters or less")
    .regex(/^[a-z0-9-]+$/, "Slug can only contain lowercase letters, numbers, and hyphens")
    .refine(s => !s.startsWith("-") && !s.endsWith("-"), {
      message: "Slug cannot start or end with a hyphen",
    }),
});

export const UpdateWorkspaceSchema = z.object({
  name: z.string().trim().min(1).max(100).optional(),
  logoUrl: z.string().url("Must be a valid URL").optional().nullable(),
});

export const WorkspaceQuerySchema = z.object({
  limit: z.coerce.number().int().positive().max(100).default(20),
  cursor: z.string().optional(),
});

export type CreateWorkspaceInput = z.infer<typeof CreateWorkspaceSchema>;
export type UpdateWorkspaceInput = z.infer<typeof UpdateWorkspaceSchema>;
export type WorkspaceQuery = z.infer<typeof WorkspaceQuerySchema>;
```

```typescript
// src/schemas/client.ts
import { z } from "zod";

const AddressSchema = z.object({
  line1: z.string().trim().min(1),
  line2: z.string().trim().optional(),
  city: z.string().trim().min(1),
  postcode: z.string().trim().min(1),
  country: z.string().length(2, "Must be a 2-letter country code").toUpperCase(),
});

export const CreateClientSchema = z.object({
  name: z.string().trim().min(1, "Client name is required").max(200),
  email: z.string().trim().email("Must be a valid email address").toLowerCase(),
  company: z.string().trim().max(200).optional(),
  vatNumber: z.string()
    .trim()
    .regex(/^[A-Z]{2}[0-9A-Z]+$/, "Must be a valid VAT number (e.g. GB123456789)")
    .optional(),
  address: AddressSchema.optional(),
});

export const UpdateClientSchema = CreateClientSchema.partial();

export const ClientQuerySchema = z.object({
  limit: z.coerce.number().int().positive().max(100).default(20),
  cursor: z.string().optional(),
  search: z.string().trim().max(100).optional(),
});

export type CreateClientInput = z.infer<typeof CreateClientSchema>;
export type UpdateClientInput = z.infer<typeof UpdateClientSchema>;
export type ClientQuery = z.infer<typeof ClientQuerySchema>;
```

```typescript
// src/schemas/project.ts
import { z } from "zod";

const ISO_DATE = /^\d{4}-\d{2}-\d{2}$/;
const isoDate = z.string().regex(ISO_DATE, "Must be a date in YYYY-MM-DD format");

export const CreateProjectSchema = z.object({
  clientId: z.string().min(1, "Client ID is required"),
  name: z.string().trim().min(1, "Project name is required").max(200),
  description: z.string().trim().max(2000).optional(),
  hourlyRate: z.number().nonnegative("Hourly rate cannot be negative").optional(),
  budget: z.number().positive("Budget must be positive").optional(),
  currency: z.enum(["GBP", "USD", "EUR", "AUD", "CAD", "SGD"]).default("GBP"),
  startDate: isoDate.optional(),
  endDate: isoDate.optional(),
}).superRefine((data, ctx) => {
  if (data.startDate && data.endDate && data.endDate < data.startDate) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["endDate"],
      message: "End date must be on or after start date",
    });
  }
});

export const UpdateProjectSchema = z.object({
  name: z.string().trim().min(1).max(200).optional(),
  description: z.string().trim().max(2000).optional().nullable(),
  status: z.enum(["active", "paused", "completed", "archived"]).optional(),
  hourlyRate: z.number().nonnegative().optional().nullable(),
  budget: z.number().positive().optional().nullable(),
  endDate: isoDate.optional().nullable(),
});

export const ProjectQuerySchema = z.object({
  limit: z.coerce.number().int().positive().max(100).default(20),
  cursor: z.string().optional(),
  clientId: z.string().optional(),
  status: z.enum(["active", "paused", "completed", "archived"]).optional(),
});

export type CreateProjectInput = z.infer<typeof CreateProjectSchema>;
export type UpdateProjectInput = z.infer<typeof UpdateProjectSchema>;
export type ProjectQuery = z.infer<typeof ProjectQuerySchema>;
```

```typescript
// src/schemas/invoice.ts
import { z } from "zod";

const ISO_DATE = /^\d{4}-\d{2}-\d{2}$/;
const isoDate = z.string().regex(ISO_DATE, "Must be a date in YYYY-MM-DD format");

export const LineItemSchema = z.object({
  description: z.string().trim().min(1, "Description is required").max(500),
  quantity: z.number().positive("Quantity must be positive"),
  unitPriceCents: z.number().int().nonnegative("Unit price cannot be negative"),
  amountCents: z.number().int().nonnegative(),
}).superRefine((item, ctx) => {
  const expected = Math.round(item.quantity * item.unitPriceCents);
  const actual = item.amountCents;
  if (Math.abs(expected - actual) > 1) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["amountCents"],
      message: `amountCents (${item.amountCents}) does not match quantity × unitPriceCents (${expected})`,
    });
  }
});

export const CreateInvoiceSchema = z.object({
  clientId: z.string().min(1, "Client ID is required"),
  projectId: z.string().optional(),
  lineItems: z
    .array(LineItemSchema)
    .min(1, "At least one line item is required")
    .max(50, "Maximum 50 line items"),
  currency: z.enum(["GBP", "USD", "EUR", "AUD", "CAD", "SGD"]).default("GBP"),
  issueDate: isoDate,
  dueDate: isoDate,
  notes: z.string().trim().max(2000).optional(),
  invoiceNumber: z.string().regex(/^[A-Z0-9-]+$/, "Invoice number can only contain uppercase letters, numbers, and hyphens").max(50).optional(),
}).superRefine((data, ctx) => {
  if (data.dueDate < data.issueDate) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["dueDate"],
      message: "Due date must be on or after issue date",
    });
  }
});

export const UpdateInvoiceSchema = z.object({
  lineItems: z.array(LineItemSchema).min(1).max(50).optional(),
  notes: z.string().trim().max(2000).optional().nullable(),
  issueDate: isoDate.optional(),
  dueDate: isoDate.optional(),
}).superRefine((data, ctx) => {
  if (data.issueDate && data.dueDate && data.dueDate < data.issueDate) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["dueDate"],
      message: "Due date must be on or after issue date",
    });
  }
});

export const InvoiceQuerySchema = z.object({
  limit: z.coerce.number().int().positive().max(100).default(20),
  cursor: z.string().optional(),
  status: z.enum(["draft", "sent", "paid", "overdue", "cancelled"]).optional(),
  clientId: z.string().optional(),
  projectId: z.string().optional(),
});

export type CreateInvoiceInput = z.infer<typeof CreateInvoiceSchema>;
export type UpdateInvoiceInput = z.infer<typeof UpdateInvoiceSchema>;
export type InvoiceQuery = z.infer<typeof InvoiceQuerySchema>;
export type LineItem = z.infer<typeof LineItemSchema>;
```

Notice what we've done: the `LineItem` type in `src/types/invoice.ts` from Chapter 2 is now gone. `LineItem` is derived from `LineItemSchema`. The schema is the source of truth. Any time you change the shape, the TypeScript type updates automatically. You can't forget to update one without updating the other because they're the same thing.

### Replacing the hand-written types

Go back to `src/types/invoice.ts`. Most of it goes away:

After Chapter 3, the type file slims down significantly. Output types — what the database returns, what the API serialises — stay here. Input types are derived from Zod schemas and re-exported from `src/schemas/`.

```typescript
// src/types/invoice.ts — updated for Chapter 3
export type InvoiceStatus = "draft" | "sent" | "paid" | "overdue" | "cancelled";

export type Invoice = {
  id: string;
  workspaceId: string;
  invoiceNumber: string;
  clientId: string;
  projectId?: string;
  status: InvoiceStatus;
  lineItems: import("../schemas/invoice").LineItem[];
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

export type { CreateInvoiceInput, UpdateInvoiceInput, InvoiceQuery } from "../schemas/invoice";
```

Do the same for the other entity type files. Output types (what the database returns, what the API serves) live in `src/types/`. Input types are derived from Zod schemas and re-exported from there.

---

## 3.4 The Validation Middleware

Right now, every handler that needs to validate input would have to call `safeParse`, check the result, format the errors, and return a 422. That's twenty lines of boilerplate per handler.

We need a validation wrapper. Write it once, use it everywhere.

### Understanding what we need

The wrapper should:
1. Accept a Zod schema for the request body (optional)
2. Accept a Zod schema for query parameters (optional)
3. Parse and validate the relevant inputs
4. If validation fails: return a structured 422 with field-level errors
5. If validation succeeds: call the handler with typed, validated inputs

The key design decision: **don't try to create a generic "all validation in one middleware" system**. Lambda doesn't have a middleware framework the way Express does — there's no `next()` to call. Instead, create a thin wrapper function that handles the common case cleanly.

### Building the validation wrapper

`formatZodError` converts Zod's nested error structure into a flat `Record<string, string[]>` keyed by dot-notation field paths — `"lineItems.0.description"` rather than `["lineItems", 0, "description"]`. Top-level issues (no path) land under `"_root"`.

```typescript
// src/lib/validate.ts

import { z, type ZodSchema, type ZodError } from "zod";
import type { APIGatewayProxyHandlerV2, APIGatewayProxyEventV2, Context } from "aws-lambda";
import type { ApiResponse } from "./response";
import { res } from "./response";
import { parseBody } from "./event";
import { logger } from "./logger";

export const formatZodError = (error: ZodError): Record<string, string[]> => {
  const fieldErrors: Record<string, string[]> = {};

  for (const issue of error.issues) {
    const field = issue.path.length > 0
      ? issue.path.join(".")
      : "_root";

    if (!fieldErrors[field]) {
      fieldErrors[field] = [];
    }
    fieldErrors[field].push(issue.message);
  }

  return fieldErrors;
};

type ValidatedHandler<TBody, TQuery> = (
  event: APIGatewayProxyEventV2,
  context: Context,
  validated: {
    body: TBody;
    query: TQuery;
  }
) => Promise<ApiResponse>;

type ValidateOptions<TBody, TQuery> = {
  body?: ZodSchema<TBody>;
  query?: ZodSchema<TQuery>;
};

/**
 * Wraps a handler with Zod validation for body and/or query parameters.
 *
 * Usage:
 *   export const handler = validate(
 *     { body: CreateInvoiceSchema, query: InvoiceQuerySchema },
 *     async (event, context, { body, query }) => {
 *       // body is typed as CreateInvoiceInput
 *       // query is typed as InvoiceQuery
 *     }
 *   );
 */
export const validate = <
  TBody = undefined,
  TQuery = undefined,
>(
  options: ValidateOptions<TBody, TQuery>,
  fn: ValidatedHandler<TBody, TQuery>
): APIGatewayProxyHandlerV2 => {
  return async (event, context) => {
    const log = logger.child({ requestId: context.awsRequestId });

    let validatedBody: TBody = undefined as TBody;
    if (options.body) {
      let rawBody: unknown;
      try {
        rawBody = parseBody(event);
      } catch {
        log.warn("Invalid JSON in request body");
        return res.badRequest("Request body must be valid JSON");
      }

      const bodyResult = options.body.safeParse(rawBody ?? {});
      if (!bodyResult.success) {
        log.warn("Request body validation failed", {
          issues: bodyResult.error.issues,
        });
        return res.unprocessable("Validation failed", formatZodError(bodyResult.error));
      }
      validatedBody = bodyResult.data;
    }

    let validatedQuery: TQuery = undefined as TQuery;
    if (options.query) {
      const queryResult = options.query.safeParse(
        event.queryStringParameters ?? {}
      );
      if (!queryResult.success) {
        log.warn("Query parameter validation failed", {
          issues: queryResult.error.issues,
        });
        return res.unprocessable("Invalid query parameters", formatZodError(queryResult.error));
      }
      validatedQuery = queryResult.data;
    }

    return fn(event, context, {
      body: validatedBody,
      query: validatedQuery,
    });
  };
};
```

### Extending the response library for validation errors

The `unprocessable` helper in `res` from Chapter 2 needs to accept structured field errors properly:

```typescript
// src/lib/response.ts — update the unprocessable helper

export const unprocessable = (
  message: string,
  fieldErrors?: Record<string, string[]>
): ApiResponse =>
  json(422, {
    error: {
      code: "UNPROCESSABLE",
      message,
      ...(fieldErrors && { fields: fieldErrors }),
    },
  });
```

The validation error response shape becomes:
```json
{
  "error": {
    "code": "UNPROCESSABLE",
    "message": "Validation failed",
    "fields": {
      "clientId": ["Required"],
      "lineItems": ["Array must contain at least 1 element(s)"],
      "dueDate": ["Due date must be on or after issue date"]
    }
  }
}
```

Every field gets an array of error messages. Clients display them next to the corresponding input. This is what usable validation errors look like.

### Using the validation wrapper in handlers

Here's the invoice creation handler rewritten with the validation wrapper:

```typescript
// src/functions/invoices/create.ts

import { validate } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { AppError } from "../../lib/errors";
import { invoiceService } from "../../services/invoices";
import { CreateInvoiceSchema } from "../../schemas/invoice";

export const handler = validate(
  { body: CreateInvoiceSchema },
  async (event, context, { body }) => {
    const log = logger.child({
      requestId: context.awsRequestId,
      function: context.functionName,
      path: event.rawPath,
    });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) {
      return res.badRequest("workspaceId path parameter is required");
    }

    try {
      // body is typed as CreateInvoiceInput — no cast, no unknown
      const invoice = await invoiceService.create(workspaceId, body, log);

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
          log.error("AppError (server)", { code: err.code, err: err.message });
        } else {
          log.warn("AppError (client)", { code: err.code, message: err.message });
        }
        return res.fromAppError(err);
      }

      log.error("Unhandled error", {
        err: err instanceof Error
          ? { name: err.name, message: err.message, stack: err.stack }
          : String(err),
      });
      return res.internalError();
    }
  }
);
```

Compare this to the version from Chapter 2. The body parsing, JSON error handling, and input validation are completely gone from the handler. The `body` parameter arrives typed as `CreateInvoiceInput`. The service receives a validated, typed value.

The try/catch still handles `AppError` and unexpected errors — that's always the handler's responsibility. But input validation is no longer in that category.

### Handlers with query validation

```typescript
// src/functions/invoices/list.ts

import { validate } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { AppError } from "../../lib/errors";
import { invoiceService } from "../../services/invoices";
import { InvoiceQuerySchema } from "../../schemas/invoice";

export const handler = validate(
  { query: InvoiceQuerySchema },
  async (event, context, { query }) => {
    const log = logger.child({
      requestId: context.awsRequestId,
      function: context.functionName,
    });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) {
      return res.badRequest("workspaceId path parameter is required");
    }

    try {
      // query.limit is number (coerced from string), query.status is the enum type
      const result = await invoiceService.list(workspaceId, query, log);

      return res.ok(result);

    } catch (err) {
      if (err instanceof AppError) {
        log.warn("AppError", { code: err.code, message: err.message });
        return res.fromAppError(err);
      }
      log.error("Unhandled error", { err });
      return res.internalError();
    }
  }
);
```

The query schema uses `z.coerce.number()` for `limit`, so even though API Gateway passes query parameters as strings, `query.limit` arrives as a proper `number`. No manual `parseInt`. No NaN check. If the client sends `?limit=abc`, Zod catches it and returns a 422 before the handler function body runs.

### Handlers with both body and query validation

To validate both body and query in one call, pass both schemas to `validate`. The `expand` param arrives as a comma-separated string (`?expand=client,project`) — the schema coerces it to `Array<"client" | "project">` before it reaches your handler.

```typescript
import { z } from "zod";
import { validate } from "../../lib/validate";
import { UpdateInvoiceSchema } from "../../schemas/invoice";

const UpdateInvoiceQuerySchema = z.object({
  expand: z.array(z.enum(["client", "project"])).optional(),
});

export const handler = validate(
  { body: UpdateInvoiceSchema, query: UpdateInvoiceQuerySchema },
  async (event, context, { body, query }) => {
    // body is UpdateInvoiceInput, query.expand is Array<"client"|"project"> | undefined
  }
);
```

### The handler try/catch pattern — extracted

Every handler ends up with the same try/catch boilerplate. Extract it:

```typescript
// src/lib/handler.ts

import type { APIGatewayProxyHandlerV2, APIGatewayProxyEventV2, Context } from "aws-lambda";
import { AppError } from "./errors";
import { res } from "./response";
import { logger } from "./logger";
import type { ApiResponse } from "./response";

type HandlerFn = (
  event: APIGatewayProxyEventV2,
  context: Context
) => Promise<ApiResponse>;

/**
 * Wraps a handler function with standard error handling and logging.
 * Catches AppError, SyntaxError, and unexpected errors.
 */
export const withErrorHandling = (fn: HandlerFn): APIGatewayProxyHandlerV2 => {
  return async (event, context) => {
    const log = logger.child({
      requestId: context.awsRequestId,
      function: context.functionName,
      path: event.rawPath,
      method: event.requestContext.http.method,
    });

    try {
      return await fn(event, context);
    } catch (err) {
      if (err instanceof SyntaxError) {
        log.warn("JSON parse error", { message: err.message });
        return res.badRequest("Request body must be valid JSON");
      }

      if (err instanceof AppError) {
        const level = err.statusCode >= 500 ? "error" : "warn";
        log[level]("AppError", {
          code: err.code,
          message: err.message,
          statusCode: err.statusCode,
        });
        return res.fromAppError(err);
      }

      log.error("Unhandled error", {
        err: err instanceof Error
          ? { name: err.name, message: err.message, stack: err.stack }
          : String(err),
      });
      return res.internalError();
    }
  };
};
```

Now combine `withErrorHandling` and `validate`:

```typescript
// src/lib/validate.ts — updated to compose with error handling

import { withErrorHandling } from "./handler";

// ... (existing formatZodError, validate code)

/**
 * Combines validation and error handling into a single wrapper.
 * Use this in preference to validate() alone.
 */
export const validateHandler = <TBody = undefined, TQuery = undefined>(
  options: ValidateOptions<TBody, TQuery>,
  fn: ValidatedHandler<TBody, TQuery>
): APIGatewayProxyHandlerV2 => {
  return withErrorHandling(
    validate(options, fn) as unknown as HandlerFn
  );
};
```

The final form of a handler using everything:

```typescript
// src/functions/invoices/create.ts — final form

import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { invoiceService } from "../../services/invoices";
import { CreateInvoiceSchema } from "../../schemas/invoice";

export const handler = validateHandler(
  { body: CreateInvoiceSchema },
  async (event, context, { body }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId path parameter is required");

    const invoice = await invoiceService.create(workspaceId, body, log);

    log.info("Invoice created", { invoiceId: invoice.id, workspaceId });
    return res.created(invoice);
  }
);
```

Error handling, JSON parsing, schema validation, request logging — all invisible. The handler is four lines of business logic.

---

## 3.5 All Runway Routes, Fully Validated

Let's write all the handlers now. This is the complete API surface for Runway — every route with proper validation.

### Workspaces

```typescript
// src/functions/workspaces/create.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { workspaceService } from "../../services/workspaces";
import { CreateWorkspaceSchema } from "../../schemas/workspace";

export const handler = validateHandler(
  { body: CreateWorkspaceSchema },
  async (event, context, { body }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const ownerId = event.headers["x-user-id"]; // Chapter 7 replaces this with JWT extraction
    if (!ownerId) return res.unauthorized();

    const workspace = await workspaceService.create(body, ownerId, log);

    log.info("Workspace created", { workspaceId: workspace.id, ownerId });
    return res.created(workspace);
  }
);
```

```typescript
// src/functions/workspaces/get.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { workspaceService } from "../../services/workspaces";

export const handler = validateHandler(
  {},
  async (event, context) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId is required");

    const workspace = await workspaceService.findById(workspaceId, log);
    if (!workspace) return res.notFound("Workspace");

    return res.ok(workspace);
  }
);
```

```typescript
// src/functions/workspaces/update.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { workspaceService } from "../../services/workspaces";
import { UpdateWorkspaceSchema } from "../../schemas/workspace";

export const handler = validateHandler(
  { body: UpdateWorkspaceSchema },
  async (event, context, { body }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId is required");

    const workspace = await workspaceService.update(workspaceId, body, log);
    if (!workspace) return res.notFound("Workspace");

    log.info("Workspace updated", { workspaceId });
    return res.ok(workspace);
  }
);
```

### Clients

```typescript
// src/functions/clients/create.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { clientService } from "../../services/clients";
import { CreateClientSchema } from "../../schemas/client";

export const handler = validateHandler(
  { body: CreateClientSchema },
  async (event, context, { body }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId is required");

    const client = await clientService.create(workspaceId, body, log);

    log.info("Client created", { clientId: client.id, workspaceId });
    return res.created(client);
  }
);
```

```typescript
// src/functions/clients/list.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { clientService } from "../../services/clients";
import { ClientQuerySchema } from "../../schemas/client";

export const handler = validateHandler(
  { query: ClientQuerySchema },
  async (event, context, { query }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId is required");

    const result = await clientService.list(workspaceId, query, log);
    return res.ok(result);
  }
);
```

```typescript
// src/functions/clients/get.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { clientService } from "../../services/clients";

export const handler = validateHandler(
  {},
  async (event, context) => {
    const workspaceId = event.pathParameters?.workspaceId;
    const clientId = event.pathParameters?.clientId;

    if (!workspaceId || !clientId) return res.badRequest("Missing path parameters");

    const client = await clientService.findById(workspaceId, clientId);
    if (!client) return res.notFound("Client");

    return res.ok(client);
  }
);
```

```typescript
// src/functions/clients/update.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { clientService } from "../../services/clients";
import { UpdateClientSchema } from "../../schemas/client";

export const handler = validateHandler(
  { body: UpdateClientSchema },
  async (event, context, { body }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    const clientId = event.pathParameters?.clientId;

    if (!workspaceId || !clientId) return res.badRequest("Missing path parameters");

    const client = await clientService.update(workspaceId, clientId, body, log);
    if (!client) return res.notFound("Client");

    return res.ok(client);
  }
);
```

### Projects

```typescript
// src/functions/projects/create.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { projectService } from "../../services/projects";
import { CreateProjectSchema } from "../../schemas/project";

export const handler = validateHandler(
  { body: CreateProjectSchema },
  async (event, context, { body }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId is required");

    const project = await projectService.create(workspaceId, body, log);

    log.info("Project created", { projectId: project.id, workspaceId });
    return res.created(project);
  }
);
```

```typescript
// src/functions/projects/list.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { projectService } from "../../services/projects";
import { ProjectQuerySchema } from "../../schemas/project";

export const handler = validateHandler(
  { query: ProjectQuerySchema },
  async (event, context, { query }) => {
    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId is required");

    const result = await projectService.list(workspaceId, query);
    return res.ok(result);
  }
);
```

### Invoices

```typescript
// src/functions/invoices/list.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { invoiceService } from "../../services/invoices";
import { InvoiceQuerySchema } from "../../schemas/invoice";

export const handler = validateHandler(
  { query: InvoiceQuerySchema },
  async (event, context, { query }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId is required");

    const result = await invoiceService.list(workspaceId, query, log);
    return res.ok(result);
  }
);
```

```typescript
// src/functions/invoices/get.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { invoiceService } from "../../services/invoices";

export const handler = validateHandler(
  {},
  async (event) => {
    const workspaceId = event.pathParameters?.workspaceId;
    const invoiceId = event.pathParameters?.invoiceId;

    if (!workspaceId || !invoiceId) return res.badRequest("Missing path parameters");

    const invoice = await invoiceService.findById(workspaceId, invoiceId);
    if (!invoice) return res.notFound("Invoice");

    return res.ok(invoice);
  }
);
```

```typescript
// src/functions/invoices/create.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { invoiceService } from "../../services/invoices";
import { CreateInvoiceSchema } from "../../schemas/invoice";

export const handler = validateHandler(
  { body: CreateInvoiceSchema },
  async (event, context, { body }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId is required");

    const invoice = await invoiceService.create(workspaceId, body, log);

    log.info("Invoice created", {
      invoiceId: invoice.id,
      workspaceId,
      total: invoice.total,
      currency: invoice.currency,
    });
    return res.created(invoice);
  }
);
```

```typescript
// src/functions/invoices/update.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { invoiceService } from "../../services/invoices";
import { UpdateInvoiceSchema } from "../../schemas/invoice";

export const handler = validateHandler(
  { body: UpdateInvoiceSchema },
  async (event, context, { body }) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    const invoiceId = event.pathParameters?.invoiceId;

    if (!workspaceId || !invoiceId) return res.badRequest("Missing path parameters");

    const invoice = await invoiceService.update(workspaceId, invoiceId, body, log);
    if (!invoice) return res.notFound("Invoice");

    log.info("Invoice updated", { invoiceId, workspaceId });
    return res.ok(invoice);
  }
);
```

The send handler takes no body — the `POST` to the URL is itself the action.

```typescript
// src/functions/invoices/send.ts
import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { invoiceService } from "../../services/invoices";

export const handler = validateHandler(
  {},
  async (event, context) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    const invoiceId = event.pathParameters?.invoiceId;

    if (!workspaceId || !invoiceId) return res.badRequest("Missing path parameters");

    const invoice = await invoiceService.send(workspaceId, invoiceId, log);

    log.info("Invoice sent", { invoiceId, workspaceId, clientId: invoice.clientId });
    return res.ok(invoice);
  }
);
```

### Updating the services to use schema types

Now that input types come from schemas, update the service signatures:

```typescript
// src/services/invoices.ts — updated signatures
import type { CreateInvoiceInput, UpdateInvoiceInput, InvoiceQuery } from "../schemas/invoice";
import type { Invoice } from "../types/invoice";
import type { Logger } from "../lib/logger";

export const invoiceService = {
  async create(
    workspaceId: string,
    input: CreateInvoiceInput,  // from schema — validated, typed
    log: Logger
  ): Promise<Invoice> {
    // input.lineItems is LineItem[] — validated
    // input.currency is "GBP" | "USD" | "EUR" | "AUD" | "CAD" | "SGD" — not any string
    // input.dueDate is after input.issueDate — guaranteed by superRefine
    // ...
  },

  async list(
    workspaceId: string,
    query: InvoiceQuery,  // limit is number, not string
    log: Logger
  ): Promise<{ invoices: Invoice[]; total: number; cursor?: string }> {
    // query.limit is a number — coerced by Zod
    // query.status is the enum type or undefined — not any string
    // ...
  },

  async update(
    workspaceId: string,
    invoiceId: string,
    input: UpdateInvoiceInput,  // all optional, validated
    log: Logger
  ): Promise<Invoice | null> {
    // ...
  },

  async send(
    workspaceId: string,
    invoiceId: string,
    log: Logger
  ): Promise<Invoice> {
    // ...
  },
};
```

The service receives typed input. It doesn't re-validate — that happened in the middleware. It trusts the types, because the types are backed by runtime validation.

---

## 3.6 Testing the Validation Layer

The validation layer deserves its own tests. Not the handlers — the schemas and the middleware.

### Testing schemas

```typescript
// src/schemas/__tests__/invoice.test.ts
import { describe, it, expect } from "vitest";
import { CreateInvoiceSchema, LineItemSchema } from "../invoice";

describe("CreateInvoiceSchema", () => {
  const validInput = {
    clientId: "cli_abc123",
    lineItems: [
      { description: "Website redesign", quantity: 1, unitPriceCents: 5000, amountCents: 5000 },
    ],
    issueDate: "2025-03-01",
    dueDate: "2025-04-01",
  };

  it("accepts valid input", () => {
    const result = CreateInvoiceSchema.safeParse(validInput);
    expect(result.success).toBe(true);
    if (result.success) {
      expect(result.data.currency).toBe("GBP"); // default applied
    }
  });

  it("rejects missing clientId", () => {
    const result = CreateInvoiceSchema.safeParse({ ...validInput, clientId: "" });
    expect(result.success).toBe(false);
    if (!result.success) {
      const fields = result.error.issues.map(i => i.path.join("."));
      expect(fields).toContain("clientId");
    }
  });

  it("rejects empty lineItems array", () => {
    const result = CreateInvoiceSchema.safeParse({ ...validInput, lineItems: [] });
    expect(result.success).toBe(false);
    if (!result.success) {
      expect(result.error.issues[0].path).toContain("lineItems");
    }
  });

  it("rejects dueDate before issueDate", () => {
    const result = CreateInvoiceSchema.safeParse({
      ...validInput,
      issueDate: "2025-04-01",
      dueDate: "2025-03-01",  // before issueDate
    });
    expect(result.success).toBe(false);
    if (!result.success) {
      const dueDateIssue = result.error.issues.find(
        i => i.path.includes("dueDate")
      );
      expect(dueDateIssue?.message).toBe("Due date must be on or after issue date");
    }
  });

  it("rejects invalid currency", () => {
    const result = CreateInvoiceSchema.safeParse({
      ...validInput,
      currency: "JPY", // not in our enum
    });
    expect(result.success).toBe(false);
  });

  it("accepts valid currency", () => {
    const result = CreateInvoiceSchema.safeParse({ ...validInput, currency: "USD" });
    expect(result.success).toBe(true);
  });

  it("applies default currency when omitted", () => {
    const result = CreateInvoiceSchema.safeParse(validInput);
    expect(result.success).toBe(true);
    if (result.success) {
      expect(result.data.currency).toBe("GBP");
    }
  });
});

describe("LineItemSchema", () => {
  it("validates amountCents matches quantity × unitPriceCents", () => {
    const valid = { description: "Work", quantity: 2, unitPriceCents: 100, amountCents: 200 };
    expect(LineItemSchema.safeParse(valid).success).toBe(true);
  });

  it("rejects amount that doesn't match", () => {
    const invalid = { description: "Work", quantity: 2, unitPriceCents: 100, amountCents: 150 };
    const result = LineItemSchema.safeParse(invalid);
    expect(result.success).toBe(false);
    if (!result.success) {
      const amountIssue = result.error.issues.find(i => i.path.includes("amount"));
      expect(amountIssue).toBeDefined();
    }
  });
});

describe("formatZodError", () => {
  it("formats nested path errors", () => {
    const { formatZodError } = require("../../lib/validate");
    const result = CreateInvoiceSchema.safeParse({
      clientId: "cli_1",
      lineItems: [
        { description: "", quantity: 1, unitPriceCents: 100, amountCents: 100 },
      ],
      issueDate: "2025-03-01",
      dueDate: "2025-04-01",
    });

    expect(result.success).toBe(false);
    if (!result.success) {
      const formatted = formatZodError(result.error);
      expect(formatted["lineItems.0.description"]).toBeDefined();
      expect(formatted["lineItems.0.description"][0]).toContain("at least 1 character");
    }
  });
});
```

### Testing the validate middleware

```typescript
// src/lib/__tests__/validate.test.ts
import { describe, it, expect, vi } from "vitest";
import { z } from "zod";
import { validate, validateHandler } from "../validate";
import { mockApiEvent, mockContext } from "../../test/helpers";

const TestBodySchema = z.object({
  name: z.string().min(1),
  count: z.number().positive(),
});

const TestQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  search: z.string().optional(),
});

describe("validate middleware", () => {
  it("calls handler with validated body", async () => {
    const innerFn = vi.fn().mockResolvedValue({
      statusCode: 200,
      headers: {},
      body: "{}",
    });

    const handler = validate({ body: TestBodySchema }, innerFn);

    const event = mockApiEvent({
      method: "POST",
      body: { name: "Test", count: 5 },
    });

    await handler(event, mockContext(), () => {});

    expect(innerFn).toHaveBeenCalledWith(
      event,
      expect.any(Object),
      expect.objectContaining({
        body: { name: "Test", count: 5 },
      })
    );
  });

  it("returns 422 for invalid body", async () => {
    const innerFn = vi.fn();
    const handler = validate({ body: TestBodySchema }, innerFn);

    const event = mockApiEvent({
      method: "POST",
      body: { name: "", count: -1 },
    });

    const result = await handler(event, mockContext(), () => {});

    expect(result?.statusCode).toBe(422);
    expect(innerFn).not.toHaveBeenCalled();

    const body = JSON.parse(result?.body ?? "{}");
    expect(body.error.code).toBe("UNPROCESSABLE");
    expect(body.error.fields).toHaveProperty("name");
    expect(body.error.fields).toHaveProperty("count");
  });

  it("returns 400 for invalid JSON", async () => {
    const innerFn = vi.fn();
    const handler = validate({ body: TestBodySchema }, innerFn);

    const event = mockApiEvent({ method: "POST" });
    event.body = "{ not valid json }";

    const result = await handler(event, mockContext(), () => {});

    expect(result?.statusCode).toBe(400);
    expect(innerFn).not.toHaveBeenCalled();
  });

  it("coerces string query params", async () => {
    const innerFn = vi.fn().mockResolvedValue({
      statusCode: 200,
      headers: {},
      body: "{}",
    });

    const handler = validate({ query: TestQuerySchema }, innerFn);

    const event = mockApiEvent({
      queryStringParameters: { page: "3", search: "test" },
    });

    await handler(event, mockContext(), () => {});

    expect(innerFn).toHaveBeenCalledWith(
      event,
      expect.any(Object),
      expect.objectContaining({
        query: { page: 3, search: "test" },
      })
    );
  });

  it("applies default values from schema", async () => {
    const innerFn = vi.fn().mockResolvedValue({
      statusCode: 200,
      headers: {},
      body: "{}",
    });

    const handler = validate({ query: TestQuerySchema }, innerFn);

    const event = mockApiEvent({});

    await handler(event, mockContext(), () => {});

    expect(innerFn).toHaveBeenCalledWith(
      event,
      expect.any(Object),
      expect.objectContaining({
        query: { page: 1 },
      })
    );
  });

  it("returns 422 for invalid query params", async () => {
    const innerFn = vi.fn();
    const handler = validate({ query: TestQuerySchema }, innerFn);

    const event = mockApiEvent({
      queryStringParameters: { page: "-5" },
    });

    const result = await handler(event, mockContext(), () => {});

    expect(result?.statusCode).toBe(422);
    expect(innerFn).not.toHaveBeenCalled();
    const body = JSON.parse(result?.body ?? "{}");
    expect(body.error.fields).toHaveProperty("page");
  });
});
```

---

## 3.7 CORS Done Right

CORS (Cross-Origin Resource Sharing) is the browser mechanism that prevents one website from making API requests to a different domain without permission. When Runway's frontend (served from `app.runway.so`) calls the API (at `api.runway.so`), the browser enforces CORS. If you don't configure it correctly, the API silently rejects browser requests while still responding to curl.

### The problem CORS solves (and doesn't solve)

CORS protects users, not your API. A browser enforces CORS. `curl`, Postman, backend-to-backend requests — none of them check CORS. So CORS doesn't prevent abuse from non-browser clients; it prevents malicious websites from making authenticated requests on behalf of your users.

For a mobile app or backend client, CORS doesn't matter. For a web frontend, it absolutely does.

### How HTTP API CORS works in SST

SST's `sst.aws.ApiGatewayV2` accepts a `cors` property:

```typescript
const api = new sst.aws.ApiGatewayV2("RunwayApi", {
  cors: {
    allowOrigins: ["https://app.runway.so"],
    allowMethods: ["GET", "POST", "PATCH", "DELETE", "OPTIONS"],
    allowHeaders: [
      "Content-Type",
      "Authorization",
      "X-User-Id",      // our placeholder auth header from earlier
      "X-Request-Id",
    ],
    exposeHeaders: ["X-Request-Id"],  // headers the client JS can access
    allowCredentials: true,           // allow cookies/auth headers
    maxAge: "1 day",                  // preflight cache duration
  },
});
```

This tells API Gateway to handle preflight requests (`OPTIONS`) automatically and add the correct `Access-Control-*` headers to every response. You don't write CORS logic in your Lambda functions.

### The common CORS mistakes

**Mistake 1: `allowOrigins: ["*"]` with `allowCredentials: true`**

Browsers reject this combination. You cannot use wildcard origin with credentials. If your frontend sends cookies or `Authorization` headers (which Cognito does), you must specify exact origins.

```typescript
// ❌ This doesn't work in browsers with credentials
cors: {
  allowOrigins: ["*"],
  allowCredentials: true,
}

// ✅ Exact origins only when credentials are involved
cors: {
  allowOrigins: ["https://app.runway.so"],
  allowCredentials: true,
}
```

**Mistake 2: Not including your custom headers in `allowHeaders`**

If your frontend sends `Authorization: Bearer ...` and `Authorization` isn't in `allowHeaders`, the preflight request fails and your API call never reaches Lambda.

The browser-safe minimum for a JWT-authenticated API:
```typescript
allowHeaders: [
  "Content-Type",
  "Authorization",
  "X-Amz-Date",            // needed if using AWS Signature V4 anywhere
  "X-Api-Key",             // needed if using API Gateway API keys
],
```

Add your custom headers (`X-User-Id`, `X-Correlation-Id`, etc.) explicitly.

**Mistake 3: CORS errors from Lambda errors**

When Lambda returns a 5xx error, API Gateway may not add CORS headers to the error response. The browser sees a network error instead of the 500 — which is confusing. To fix this, ensure your Lambda always returns a valid HTTP response (never throws uncaught), and configure CORS at the stage level (not per-Lambda) so API Gateway adds headers even to error responses.

SST's CORS configuration is stage-level, so this is handled correctly out of the box.

**Mistake 4: Forgetting to update CORS for new development origins**

During development, you often run the frontend on `http://localhost:3000` or a Vite dev server. Production CORS config won't allow that.

Handle it per stage:

```typescript
// sst.config.ts
const isProd = $app.stage === "production";

const allowedOrigins = isProd
  ? ["https://app.runway.so"]
  : [
      "http://localhost:3000",
      "http://localhost:5173",  // Vite
      `https://${$app.stage}.runway.so`,
    ];

const api = new sst.aws.ApiGatewayV2("RunwayApi", {
  cors: {
    allowOrigins: allowedOrigins,
    allowMethods: ["GET", "POST", "PATCH", "DELETE", "OPTIONS"],
    allowHeaders: ["Content-Type", "Authorization", "X-User-Id"],
    allowCredentials: true,
    maxAge: isProd ? "1 day" : "1 minute",
  },
});
```

**Mistake 5: Not testing CORS in the browser**

CORS errors only appear in browsers. Always test your API with an actual browser request during setup. A quick test:

Paste this in the browser console on your frontend app origin:

```javascript
fetch("https://your-api-url.execute-api.eu-west-1.amazonaws.com/health", {
  method: "GET",
  headers: { "Authorization": "Bearer test" },
  credentials: "include",
})
.then(r => r.json())
.then(console.log)
.catch(console.error);
```

If you see `Access to fetch at '...' from origin '...' has been blocked by CORS policy`, check `allowOrigins` and `allowHeaders`.

### Preflight requests

`OPTIONS` requests are automatically handled by API Gateway when CORS is configured at the gateway level. You don't need to add `OPTIONS` routes to your SST config, and you don't need Lambda functions for preflight — API Gateway responds directly.

If you see Lambda invocations for `OPTIONS` requests, your CORS is configured at the wrong level (per-route instead of gateway-level). With SST's `cors` property on `ApiGatewayV2`, this shouldn't happen.

---

## 3.8 Rate Limiting and Throttling

API Gateway HTTP API has built-in throttling. It's not a replacement for a full rate-limiting solution, but it protects your Lambda functions from runaway clients and reduces costs during abuse.

### Configuring throttling in SST

```typescript
const api = new sst.aws.ApiGatewayV2("RunwayApi", {
  transform: {
    stage: {
      defaultRouteSettings: {
        throttlingBurstLimit: 100,
        throttlingRateLimit: 50,
      },
    },
  },
});
```

The throttling settings:
- **`throttlingRateLimit`**: The steady-state requests per second. API Gateway uses a token bucket algorithm — tokens fill at this rate.
- **`throttlingBurstLimit`**: The maximum token bucket size. You can exceed the rate limit in short bursts up to this limit, then requests are throttled.

When throttled, API Gateway returns a 429 Too Many Requests response directly — your Lambda function is never invoked. This protects you from both accidental DDoS and cost spikes from misbehaving clients.

### Per-route throttling

Some routes need different limits. The invoice send endpoint is more expensive than a simple GET:

Per-route throttling uses the `transform.route.routeSettings` property. The `transform` pattern is how SST exposes CloudFormation properties that don't have a first-class SST API yet.

```typescript
api.route("POST /workspaces/{workspaceId}/invoices/{invoiceId}/send", {
  handler: "src/functions/invoices/send.handler",
  ...sharedFunctionConfig,
  transform: {
    route: {
      routeSettings: {
        throttlingBurstLimit: 10,
        throttlingRateLimit: 5,
      },
    },
  },
});
```

This limits invoice sends to 5 per second per stage (not per user — API Gateway throttling is not per-user). For per-user rate limiting, you need a DynamoDB-based solution (discussed below).

### Implementing per-user rate limiting

API Gateway's built-in throttling is global. For Runway, you likely want per-user limits — a freelancer shouldn't be able to send 100 invoices per second just because they've scripted it.

The cleanest approach: a DynamoDB-based sliding window counter, checked in a middleware.

```typescript
// src/lib/rateLimit.ts

import { getDb, GetCommand, UpdateCommand } from "./db/dynamodb";
import { getConfig } from "./config";
import type { ApiResponse } from "./response";
import { res } from "./response";

type RateLimitConfig = {
  key: string;          // e.g. "send-invoice:user_123"
  limit: number;        // max requests in window
  windowSeconds: number; // rolling window size
};

type RateLimitResult =
  | { allowed: true }
  | { allowed: false; response: ApiResponse };

/**
 * Sliding window rate limiter backed by DynamoDB.
 * Uses atomic UpdateExpression to increment counter.
 *
 * Adds ~5ms of DynamoDB latency — only apply to expensive operations.
 * Fails open on DynamoDB errors: if the rate limiter breaks, requests are allowed through
 * rather than blocking users. A broken rate limiter is bad; a broken API is worse.
 */
export const checkRateLimit = async (
  config: RateLimitConfig
): Promise<RateLimitResult> => {
  const db = getDb();
  const cfg = getConfig();
  const now = Math.floor(Date.now() / 1000);
  const windowStart = now - config.windowSeconds;

  const windowKey = Math.floor(now / config.windowSeconds);
  const itemKey = `RATELIMIT#${config.key}#${windowKey}`;

  try {
    const result = await db.send(
      new UpdateCommand({
        TableName: cfg.TABLE_NAME,
        Key: {
          pk: itemKey,
          sk: itemKey,
        },
        UpdateExpression:
          "SET #count = if_not_exists(#count, :zero) + :one, #ttl = :ttl",
        ExpressionAttributeNames: {
          "#count": "count",
          "#ttl": "ttl",
        },
        ExpressionAttributeValues: {
          ":zero": 0,
          ":one": 1,
          ":ttl": now + config.windowSeconds * 2, // keep for 2 windows
        },
        ReturnValues: "ALL_NEW",
      })
    );

    const count = (result.Attributes?.count as number) ?? 1;

    if (count > config.limit) {
      return {
        allowed: false,
        response: {
          statusCode: 429,
          headers: {
            "Content-Type": "application/json",
            "X-RateLimit-Limit": String(config.limit),
            "X-RateLimit-Remaining": "0",
            "Retry-After": String(config.windowSeconds),
          },
          body: JSON.stringify({
            error: {
              code: "RATE_LIMITED",
              message: `Too many requests. Limit: ${config.limit} per ${config.windowSeconds} seconds.`,
            },
          }),
        },
      };
    }

    return { allowed: true };
  } catch {
    return { allowed: true };
  }
};
```

Apply it in the invoice send handler:

```typescript
// src/functions/invoices/send.ts — with rate limiting

import { validateHandler } from "../../lib/validate";
import { res } from "../../lib/response";
import { logger } from "../../lib/logger";
import { invoiceService } from "../../services/invoices";
import { checkRateLimit } from "../../lib/rateLimit";

export const handler = validateHandler(
  {},
  async (event, context) => {
    const log = logger.child({ requestId: context.awsRequestId });

    const workspaceId = event.pathParameters?.workspaceId;
    const invoiceId = event.pathParameters?.invoiceId;
    if (!workspaceId || !invoiceId) return res.badRequest("Missing path parameters");

    const rateLimitResult = await checkRateLimit({
      key: `send-invoice:${workspaceId}`,
      limit: 10,
      windowSeconds: 60,
    });

    if (!rateLimitResult.allowed) {
      log.warn("Rate limit exceeded", { workspaceId });
      return rateLimitResult.response;
    }

    const invoice = await invoiceService.send(workspaceId, invoiceId, log);

    log.info("Invoice sent", { invoiceId, workspaceId, clientId: invoice.clientId });
    return res.ok(invoice);
  }
);
```

This adds a DynamoDB round-trip before the main operation. Only apply it to endpoints where the cost of abuse is high — sending emails, creating resources, triggering payments.

### What the rate limit headers tell clients

Include `X-RateLimit-*` headers on successful responses too, so clients know how close they are to the limit:

In your response helper or middleware, add these to successful responses:

- `X-RateLimit-Limit: 10`
- `X-RateLimit-Remaining: 7`
- `X-RateLimit-Reset: 1740820860` (Unix timestamp when the current window resets)

Well-behaved API clients read these headers and back off before they get 429s. Document them in your API docs.

---

## 3.9 Response Caching

For some Runway endpoints, the same data is requested frequently and doesn't change often. Caching those responses reduces Lambda invocations, DynamoDB reads, and latency.

### Where to cache

HTTP API does not have built-in response caching (that's a REST API feature). For HTTP API, caching happens at CloudFront — in front of API Gateway.

The right architecture:
```
Client → CloudFront → API Gateway → Lambda → DynamoDB
              ↑
          Cache lives here
```

When CloudFront has a cached response, Lambda is never invoked.

### What to cache in Runway

Not everything should be cached. The decision:

**Good candidates:**
- `GET /workspaces/{workspaceId}` — workspace settings change rarely
- `GET /workspaces/{workspaceId}/clients/{clientId}` — client details change rarely
- `GET /workspaces/{workspaceId}/projects/{projectId}` — project details change rarely

**Bad candidates:**
- `GET /workspaces/{workspaceId}/invoices` — list endpoints with filters don't cache well
- Any authenticated endpoint where the cache must be user-scoped (CloudFront cache key must include the auth token — complex)
- Anything that can be updated frequently

### Adding CloudFront in front of API Gateway

The `defaultCacheBehavior` uses `CachingDisabled` for all routes (pass-through), with `orderedCacheBehaviors` overriding specific path patterns where caching is safe.

```typescript
// sst.config.ts
const cdn = new sst.aws.CloudFront("RunwayApiCdn", {
  origins: [
    {
      domain: api.url.apply(url => new URL(url).hostname),
      path: "/",
    },
  ],
  defaultCacheBehavior: {
    viewerProtocolPolicy: "redirect-to-https",
    cachePolicy: "CachingDisabled",
    originRequestPolicy: "AllViewerExceptHostHeader",
    allowedMethods: ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"],
    cachedMethods: ["GET", "HEAD"],
  },
  orderedCacheBehaviors: [
    {
      pathPattern: "/workspaces/*/",
      viewerProtocolPolicy: "redirect-to-https",
      compress: true,
      cachePolicyId: "...",
      allowedMethods: ["GET", "HEAD"],
      cachedMethods: ["GET", "HEAD"],
    },
  ],
});
```

Full CloudFront setup is beyond the scope of this chapter — Chapter 6 covers CloudFront for S3 assets in detail. The key principle: for HTTP API, caching is a CloudFront concern, not an API Gateway concern.

### Cache-Control headers from Lambda

The simpler approach: set `Cache-Control` headers in Lambda responses and let CloudFront honour them. This works with any CloudFront configuration that's set up to respect origin cache headers.

```typescript
// src/functions/workspaces/get.ts
export const handler = validateHandler(
  {},
  async (event) => {
    const workspaceId = event.pathParameters?.workspaceId;
    if (!workspaceId) return res.badRequest("workspaceId is required");

    const workspace = await workspaceService.findById(workspaceId);
    if (!workspace) return res.notFound("Workspace");

    return {
      ...res.ok(workspace),
      headers: {
        ...res.ok(workspace).headers,
        "Cache-Control": "public, max-age=300, stale-while-revalidate=60",
      },
    };
  }
);
```

For now, the better approach is to not fight for caching at the HTTP layer. For read-heavy workloads, add DynamoDB Accelerator (DAX) or an in-memory cache in the service layer. The HTTP cache is the hardest to invalidate correctly — Lambda can't tell CloudFront to evict a specific response when a workspace is updated without a CloudFront invalidation call (which has its own latency and complexity).

Cache at the data layer in Chapter 4. Let HTTP caching come later when you have evidence you need it.

---

## 3.10 API Versioning and Evolution

Building APIs is easy. Evolving them without breaking existing clients is hard. The time to think about versioning is before you have clients depending on your API — not after.

### The three approaches and when to use each

**URL versioning** (`/v1/invoices`, `/v2/invoices`) is the most explicit and the most common. Clients opt into a new version deliberately. Old clients keep working on v1 indefinitely. You maintain two code paths until you sunset v1.

**Header versioning** (`API-Version: 2025-01-01`) treats the API version as a request header. Stripe does this, using dates. Cleaner URLs, but requires clients to set a header they might forget, and proxies/CDNs often don't vary on headers by default.

**Field addition** (no versioning) is the most pragmatic approach for early-stage APIs: add new optional fields to responses without removing old ones. Never rename or remove existing fields. This works well until you need a genuinely breaking change (removing a field, changing a field's type, changing resource relationships).

**For Runway**, start with field addition (the most pragmatic option) and add URL versioning when you have a breaking change that can't be avoided.

### Setting up URL versioning with SST

SST's routing lets you prefix all routes with a version path:

The routes are defined as a typed array and registered in a loop. This scales well: adding a new v1 route is one line in the array. When you need to add v2, you create a `v2Routes` array alongside it.

```typescript
// sst.config.ts
const v1Routes = [
  ["POST",  "/v1/workspaces",                                     "workspaces/create"],
  ["GET",   "/v1/workspaces/{workspaceId}",                       "workspaces/get"],
  ["PATCH", "/v1/workspaces/{workspaceId}",                       "workspaces/update"],
  ["GET",   "/v1/workspaces/{workspaceId}/clients",               "clients/list"],
  ["POST",  "/v1/workspaces/{workspaceId}/clients",               "clients/create"],
  ["GET",   "/v1/workspaces/{workspaceId}/clients/{clientId}",    "clients/get"],
  ["PATCH", "/v1/workspaces/{workspaceId}/clients/{clientId}",    "clients/update"],
  ["GET",   "/v1/workspaces/{workspaceId}/projects",              "projects/list"],
  ["POST",  "/v1/workspaces/{workspaceId}/projects",              "projects/create"],
  ["GET",   "/v1/workspaces/{workspaceId}/invoices",              "invoices/list"],
  ["POST",  "/v1/workspaces/{workspaceId}/invoices",              "invoices/create"],
  ["GET",   "/v1/workspaces/{workspaceId}/invoices/{invoiceId}",  "invoices/get"],
  ["PATCH", "/v1/workspaces/{workspaceId}/invoices/{invoiceId}",  "invoices/update"],
  ["POST",  "/v1/workspaces/{workspaceId}/invoices/{invoiceId}/send", "invoices/send"],
] as const;

for (const [method, path, handlerPath] of v1Routes) {
  api.route(`${method} ${path}`, {
    handler: `src/functions/${handlerPath}.handler`,
    ...sharedFunctionConfig,
  });
}

api.route("GET /health", {
  handler: "src/functions/health.handler",
});
```

This approach means:
- All routes are at `/v1/` — a clean prefix clients can rely on
- When you need to break something, you add `/v2/` routes pointing to new handlers
- The old handlers at `/v1/` keep working

### Rules for non-breaking changes

These are safe to do on `/v1/` without a version bump:

**Add new optional fields to responses.** A field that wasn't there before is fine to add — clients that don't know about it ignore it.

Before:

```typescript
type Invoice = { id: string; total: number; status: InvoiceStatus };
```

After — safe to add; existing clients ignore new fields:

```typescript
type Invoice = { id: string; total: number; status: InvoiceStatus; subtotal: number; tax: number };
```

**Add new optional fields to request bodies.** If a client doesn't send a field that's now optional, the existing default applies.

**Add new endpoints.** New routes don't affect existing routes.

**Add new enum values to responses.** If you add `"voided"` to `InvoiceStatus`, clients that only know `"draft" | "sent" | "paid"` will receive `"voided"` and should handle it gracefully. This is a grey area — if clients switch on the status enum with no default case, they'll break. Document it and give clients notice.

### Rules for breaking changes (require a version bump)

These require a new version:

- **Removing a field from a response** — clients that read the field get `undefined`
- **Renaming a field** — same effect as removing the old name
- **Changing a field's type** — `string` to `number`, `array` to `object`, etc.
- **Making an optional field required** — existing clients that don't send it get 422s
- **Changing status code semantics** — e.g. a 200 that should have been 201
- **Changing error code values** — clients that switch on error codes break

When you need a breaking change:

1. Create a `/v2/` handler with the new behaviour
2. Keep the `/v1/` handler unchanged — don't break existing integrations
3. Update documentation with a deprecation notice for `/v1/`
4. Set a sunset date (at least 6-12 months for external APIs)
5. Add a `Deprecation` header to `/v1/` responses:
   ```typescript
   headers: {
     ...existingHeaders,
     "Deprecation": "true",
     "Sunset": "Sat, 01 Jan 2027 00:00:00 GMT",
     "Link": '<https://docs.runway.so/api/v2>; rel="successor-version"',
   }
   ```

### Avoiding premature versioning

Don't add `/v1/` on day one if you control all the clients. Internal APIs where you can update the frontend and API simultaneously don't need URL versioning — just deploy both at once. URL versioning is for public APIs and mobile apps where you can't force clients to update.

For Runway:
- If you're building the frontend and API together: start without versioning, add it when you need it
- If you're providing a public API to external developers: start with `/v1/` from day one

---

## 3.11 The Complete SST Configuration

Let's pull everything together into the final `sst.config.ts` for Chapter 3:

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
    const isProd = $app.stage === "production";

    // ─── Secrets ───────────────────────────────────────────────────────────
    const stripeSecretKey = new sst.Secret("StripeSecretKey");
    const stripeWebhookSecret = new sst.Secret("StripeWebhookSecret");

    // ─── Storage (placeholder until Chapter 4 adds DynamoDB properly) ──────
    const table = new sst.aws.Dynamo("RunwayTable", {
      fields: { pk: "string", sk: "string" },
      primaryIndex: { hashKey: "pk", rangeKey: "sk" },
      globalIndexes: {
        gsi1: { hashKey: "gsi1pk", rangeKey: "gsi1sk" },
      },
      ttl: "ttl",
    });

    const deliverablesBucket = new sst.aws.Bucket("RunwayDeliverables");
    const invoicesBucket = new sst.aws.Bucket("RunwayInvoices");

    // ─── API Gateway ───────────────────────────────────────────────────────
    const allowedOrigins = isProd
      ? ["https://app.runway.so"]
      : [
          "http://localhost:3000",
          "http://localhost:5173",
          `https://${$app.stage}.runway.so`,
        ];

    const api = new sst.aws.ApiGatewayV2("RunwayApi", {
      cors: {
        allowOrigins: allowedOrigins,
        allowMethods: ["GET", "POST", "PATCH", "DELETE", "OPTIONS"],
        allowHeaders: [
          "Content-Type",
          "Authorization",
          "X-User-Id",
          "X-Request-Id",
        ],
        exposeHeaders: ["X-Request-Id"],
        allowCredentials: true,
        maxAge: isProd ? "1 day" : "1 minute",
      },
      accessLog: isProd
        ? { retention: "1 month" }
        : { retention: "1 week" },
      transform: {
        stage: {
          defaultRouteSettings: {
            throttlingBurstLimit: isProd ? 500 : 50,
            throttlingRateLimit: isProd ? 200 : 20,
          },
        },
      },
    });

    // ─── Shared function config ─────────────────────────────────────────────
    const sharedFunctionConfig = {
      link: [table, deliverablesBucket, invoicesBucket, stripeSecretKey, stripeWebhookSecret],
      environment: {
        SST_STAGE: $app.stage,
        APP_URL: isProd
          ? "https://app.runway.so"
          : `https://${$app.stage}.runway.so`,
        SES_FROM_ADDRESS: "invoices@runway.so",
        LOG_LEVEL: isProd ? "info" : "debug",
      },
      memory: "512 MB" as const,
      timeout: "30 seconds" as const,
    };

    // ─── Health check ──────────────────────────────────────────────────────
    api.route("GET /health", {
      handler: "src/functions/health.handler",
    });

    // ─── Workspaces ────────────────────────────────────────────────────────
    api.route("POST /v1/workspaces", {
      handler: "src/functions/workspaces/create.handler",
      ...sharedFunctionConfig,
    });

    api.route("GET /v1/workspaces/{workspaceId}", {
      handler: "src/functions/workspaces/get.handler",
      ...sharedFunctionConfig,
    });

    api.route("PATCH /v1/workspaces/{workspaceId}", {
      handler: "src/functions/workspaces/update.handler",
      ...sharedFunctionConfig,
    });

    // ─── Clients ───────────────────────────────────────────────────────────
    api.route("GET /v1/workspaces/{workspaceId}/clients", {
      handler: "src/functions/clients/list.handler",
      ...sharedFunctionConfig,
    });

    api.route("POST /v1/workspaces/{workspaceId}/clients", {
      handler: "src/functions/clients/create.handler",
      ...sharedFunctionConfig,
    });

    api.route("GET /v1/workspaces/{workspaceId}/clients/{clientId}", {
      handler: "src/functions/clients/get.handler",
      ...sharedFunctionConfig,
    });

    api.route("PATCH /v1/workspaces/{workspaceId}/clients/{clientId}", {
      handler: "src/functions/clients/update.handler",
      ...sharedFunctionConfig,
    });

    // ─── Projects ──────────────────────────────────────────────────────────
    api.route("GET /v1/workspaces/{workspaceId}/projects", {
      handler: "src/functions/projects/list.handler",
      ...sharedFunctionConfig,
    });

    api.route("POST /v1/workspaces/{workspaceId}/projects", {
      handler: "src/functions/projects/create.handler",
      ...sharedFunctionConfig,
    });

    api.route("GET /v1/workspaces/{workspaceId}/projects/{projectId}", {
      handler: "src/functions/projects/get.handler",
      ...sharedFunctionConfig,
    });

    api.route("PATCH /v1/workspaces/{workspaceId}/projects/{projectId}", {
      handler: "src/functions/projects/update.handler",
      ...sharedFunctionConfig,
    });

    // ─── Invoices ──────────────────────────────────────────────────────────
    api.route("GET /v1/workspaces/{workspaceId}/invoices", {
      handler: "src/functions/invoices/list.handler",
      ...sharedFunctionConfig,
    });

    api.route("POST /v1/workspaces/{workspaceId}/invoices", {
      handler: "src/functions/invoices/create.handler",
      ...sharedFunctionConfig,
    });

    api.route("GET /v1/workspaces/{workspaceId}/invoices/{invoiceId}", {
      handler: "src/functions/invoices/get.handler",
      ...sharedFunctionConfig,
    });

    api.route("PATCH /v1/workspaces/{workspaceId}/invoices/{invoiceId}", {
      handler: "src/functions/invoices/update.handler",
      ...sharedFunctionConfig,
    });

    api.route("POST /v1/workspaces/{workspaceId}/invoices/{invoiceId}/send", {
      handler: "src/functions/invoices/send.handler",
      ...sharedFunctionConfig,
    });

    // ─── Outputs ───────────────────────────────────────────────────────────
    return {
      api: api.url,
    };
  },
});
```

---

## 3.12 What the Contract Looks Like in Practice

Here's the invoice creation endpoint as a complete example — this is the response shape every client consumes:

```
POST /v1/workspaces/{workspaceId}/invoices

Request body:
  { "clientId": "cli_abc123", "lineItems": [...], "currency": "GBP",
    "issueDate": "2025-03-01", "dueDate": "2025-03-31" }

201 Created:
  { "data": { "id": "inv_mN8kP2xR", "status": "draft", "total": 630000, ... } }

422 Unprocessable (bad dates):
  { "error": { "code": "UNPROCESSABLE", "message": "Validation failed",
    "fields": { "dueDate": ["Due date must be after issue date"] } } }

422 Unprocessable (invalid field):
  { "error": { "code": "UNPROCESSABLE", "message": "Validation failed",
    "fields": { "lineItems": ["Array must contain at least 1 element(s)"] } } }
```

The `fields` map in a 422 response keys directly to the field names in the request body. A frontend can display these errors next to the right inputs without any custom parsing.

---

## What We've Built

The system diagram after Chapter 3:

```
Client
  │
  ▼
API Gateway (HTTP API)
  │  ├── CORS (stage-level, per-environment origins)
  │  ├── Throttling (burst + rate limits)
  │  └── Access logging
  │
  ▼
Lambda Handler
  │  ├── validateHandler()   ← NEW: Zod schema validation
  │  │     ├── Body parsing (safe JSON parse)
  │  │     ├── Schema validation (422 + field errors on failure)
  │  │     └── Query coercion (strings → typed values)
  │  ├── withErrorHandling() ← NEW: centralised error handling
  │  │     ├── AppError → res.fromAppError()
  │  │     ├── SyntaxError → res.badRequest()
  │  │     └── Unknown → res.internalError() + log.error()
  │  └── Handler body (4 lines, all business logic)
  │
  ▼
Service Layer (typed inputs, domain errors)
  │
  ▼
DynamoDB / Stripe / SES (Chapter 4+)
```

The `as any` is gone. Every route has a schema. The schemas generate the TypeScript types. The validation middleware turns schema failures into 422 responses with field-level errors before the handler runs. The handler receives typed, validated inputs and works with domain types.

Two months from now, when a new endpoint gets added, the developer writes a schema first. The schema documents what the endpoint accepts. TypeScript enforces it at compile time. Zod enforces it at runtime. The error messages are consistent with every other endpoint in the API.

This is not ceremony. It's what makes an API possible to maintain.

### The investment was small

Look at the ratio of infrastructure to payoff:

- `src/schemas/*.ts` — four files, each 50-100 lines
- `src/lib/validate.ts` — 70 lines
- `src/lib/handler.ts` — 40 lines
- `src/lib/event.ts` — 40 lines

Under 400 lines of code. Every Runway handler is now three times shorter. Every input is validated. Every error is consistent. Every client can parse validation failures field by field.

That's a good trade.

### What's next

Chapter 4 wires up DynamoDB behind the API. The service layer currently has stub implementations — `return []`, `return null`. In Chapter 4, those stubs get replaced with real single-table DynamoDB operations: PutItem for workspace creation, Query by workspace ID for invoice listing, UpdateItem with condition expressions for status transitions. The schemas you wrote in this chapter feed directly into the DynamoDB items — `CreateInvoiceInput` is what the Lambda receives, and the `Invoice` type is what gets written to the database.

By the end of Chapter 4, every endpoint returns real data from a real database.

---

> **The code for this chapter** is available at `03-type-safe-apis/` in the companion repository. The schemas, validation middleware, and all handlers are complete and tested. Run `npm install && npx sst dev` to start it, then try `curl -X POST https://your-api-url/v1/workspaces -d '{"name": "Acme", "slug": ""}' -H "Content-Type: application/json"` to see the validation error response in action.
