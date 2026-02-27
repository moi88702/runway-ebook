# Appendix D: TypeScript Patterns Reference

The TypeScript patterns used throughout this book, explained in one place. These aren't exotic — they show up in production codebases and once you've seen each one, you'll use it everywhere.

---

## `satisfies` — Validate Without Widening

`satisfies` checks that a value matches a type without changing the inferred type of the value. The difference matters when you want type safety on the value *and* you want TypeScript to keep inferring the specific type.

```typescript
// Without satisfies — type is widened to Record<string, string>
const config: Record<string, string> = {
  region: "eu-west-1",
  stage: "production",
};
config.region.toUpperCase(); // Fine
config.oops;                 // No error — widened to Record<string, string>

// With satisfies — type is still the object literal type
const config = {
  region: "eu-west-1",
  stage: "production",
} satisfies Record<string, string>;

config.region.toUpperCase(); // Fine — TypeScript knows it's a string
config.oops;                 // Error — 'oops' does not exist on this type
```

Where this shows up in Runway: route handler registries, SST config objects, error code maps.

```typescript
// Error codes with satisfies — each code keeps its literal type
const HTTP_ERRORS = {
  NOT_FOUND: { status: 404, message: "Not found" },
  UNAUTHORIZED: { status: 401, message: "Unauthorized" },
  CONFLICT: { status: 409, message: "Conflict" },
} satisfies Record<string, { status: number; message: string }>;

// TypeScript infers the specific shape, not just Record<string, ...>
HTTP_ERRORS.NOT_FOUND.status; // number
// HTTP_ERRORS.OOPS             // Error at compile time
```

---

## Discriminated Unions — Type-Safe State Machines

A discriminated union is a union of types that each have a common literal field (the discriminant). TypeScript narrows the type based on that field.

```typescript
type InvoiceStatus =
  | { status: "draft"; createdAt: string }
  | { status: "sent"; sentAt: string; clientEmail: string }
  | { status: "paid"; paidAt: string; amount: number }
  | { status: "overdue"; dueDate: string; daysOverdue: number };

function describeInvoice(invoice: InvoiceStatus): string {
  switch (invoice.status) {
    case "draft":
      return `Draft, created ${invoice.createdAt}`;
    case "sent":
      return `Sent to ${invoice.clientEmail} on ${invoice.sentAt}`;
    case "paid":
      return `Paid — £${invoice.amount / 100}`;
    case "overdue":
      return `Overdue by ${invoice.daysOverdue} days`;
  }
}
```

Inside each `case`, TypeScript knows exactly which properties are available. `invoice.clientEmail` only exists when `status === "sent"`. Accessing it on a `"draft"` invoice is a compile error.

**Exhaustiveness checking** — add a `never` check to catch missing cases:

```typescript
function describeInvoice(invoice: InvoiceStatus): string {
  switch (invoice.status) {
    case "draft": return "...";
    case "sent": return "...";
    case "paid": return "...";
    case "overdue": return "...";
    default: {
      const _exhaustive: never = invoice;
      throw new Error(`Unhandled status: ${JSON.stringify(invoice)}`);
    }
  }
}
```

If you add a new status to `InvoiceStatus` and forget to handle it here, TypeScript errors at `const _exhaustive: never = invoice`. The compiler finds the gap before you do.

This pattern appears in every SQS worker in the book where message types are dispatched via `switch`.

---

## Branded Types — Preventing Primitive Confusion

TypeScript's structural type system means `string` is `string` everywhere. A `userId: string` and a `workspaceId: string` are interchangeable — which means you can accidentally pass one where the other is expected, silently.

Branded types add a phantom type marker that makes primitives nominally distinct:

```typescript
declare const __brand: unique symbol;
type Brand<T, B> = T & { readonly [__brand]: B };

type UserId = Brand<string, "UserId">;
type WorkspaceId = Brand<string, "WorkspaceId">;
type InvoiceId = Brand<string, "InvoiceId">;

// Constructor functions that create branded values
function toUserId(id: string): UserId {
  return id as UserId;
}
function toWorkspaceId(id: string): WorkspaceId {
  return id as WorkspaceId;
}
```

Now the types are distinct at compile time:

```typescript
function getWorkspace(workspaceId: WorkspaceId, ownerId: UserId): Promise<Workspace> {
  // ...
}

const userId = toUserId("USER#abc123");
const workspaceId = toWorkspaceId("WORKSPACE#def456");

getWorkspace(workspaceId, userId); // Fine
getWorkspace(userId, workspaceId); // Error: Argument of type 'UserId' is not assignable to parameter of type 'WorkspaceId'
```

**Where to use brands:** IDs that cross module boundaries, monetary amounts (avoid mixing cents and pounds), anything where passing the wrong primitive would cause silent data corruption.

**Where not to use brands:** Everywhere. They add friction. Use them for IDs that appear frequently in function signatures where confusion is plausible.

---

## `unknown` vs `any` — Choosing Correctly

`any` disables the type checker. `unknown` keeps it engaged.

```typescript
// any — no type safety at all
function processPayload(data: any) {
  data.invoiceId.toUpperCase(); // TypeScript allows this, even if data.invoiceId is undefined
}

// unknown — must narrow before use
function processPayload(data: unknown) {
  data.invoiceId; // Error: Object is of type 'unknown'

  // Must verify shape first
  if (typeof data === "object" && data !== null && "invoiceId" in data) {
    const id = (data as { invoiceId: string }).invoiceId;
  }

  // Better: use Zod to parse and narrow
  const payload = PayloadSchema.parse(data);
  payload.invoiceId; // string — safe
}
```

**Rule:** never use `any` in new code. Use `unknown` for genuinely unknown values (JSON.parse output, API responses before validation, error catch clauses). Use Zod to turn `unknown` into a typed value.

**Error handling:** In TypeScript 4.0+, `catch` clause variables are `unknown` by default (with `useUnknownInCatchVariables: true`, which is part of `strict`). Handle it:

```typescript
try {
  await riskyOperation();
} catch (err: unknown) {
  if (err instanceof Error) {
    logger.error("Operation failed", { message: err.message, stack: err.stack });
  } else {
    logger.error("Operation failed with unknown error", { err: String(err) });
  }
}
```

---

## Generic Utility Functions — Typing the Pattern Once

Generic functions let you write typed utilities that work across multiple types. The pattern appears throughout the book in query builders, pagination, and result wrappers.

```typescript
// Typed paginated query result
interface PageResult<T> {
  items: T[];
  cursor: string | null;
  hasMore: boolean;
}

// Generic paginator — works for any item type
async function paginatedQuery<T>(
  params: QueryCommandInput,
  decode: (item: Record<string, unknown>) => T
): Promise<PageResult<T>> {
  const result = await docClient.send(new QueryCommand(params));
  return {
    items: (result.Items ?? []).map(decode),
    cursor: result.LastEvaluatedKey
      ? Buffer.from(JSON.stringify(result.LastEvaluatedKey)).toString("base64url")
      : null,
    hasMore: !!result.LastEvaluatedKey,
  };
}

// Usage — TypeScript infers the return type
const result = await paginatedQuery(
  { TableName: "RunwayTable", KeyConditionExpression: "PK = :pk", ExpressionAttributeValues: { ":pk": "WORKSPACE#123" } },
  (item) => WorkspaceSchema.parse(item)  // decode function
);
// result.items is Workspace[]
```

---

## `as const` — Literal Types for Config Objects

`as const` makes all values in an object or array their narrowest possible literal type. Useful for configuration, enums, and lookup tables.

```typescript
// Without as const
const PLANS = ["free", "pro", "enterprise"];
// PLANS is string[] — loses the literal types

// With as const
const PLANS = ["free", "pro", "enterprise"] as const;
// PLANS is readonly ["free", "pro", "enterprise"]

type Plan = typeof PLANS[number]; // "free" | "pro" | "enterprise"

function getPlanLimit(plan: Plan): number {
  const limits = {
    free: 5,
    pro: 50,
    enterprise: Infinity,
  } as const satisfies Record<Plan, number>;

  return limits[plan];
}

getPlanLimit("pro");    // Fine
getPlanLimit("trial");  // Error: not a valid Plan
```

---

## Conditional Types — Type Transformations

Conditional types express "if T extends U, then X, else Y." Used internally by TypeScript's utility types and occasionally useful in application code.

```typescript
// Extract the success type from a Result union
type ExtractSuccess<T> = T extends { success: true; data: infer D } ? D : never;

type CreateInvoiceResult =
  | { success: true; data: Invoice }
  | { success: false; error: string };

type InvoiceData = ExtractSuccess<CreateInvoiceResult>; // Invoice
```

**`infer`** captures a type from within a conditional. It's how you extract the return type of a function, the element type of an array, etc.:

```typescript
// The type of the resolved value of a Promise
type Awaited<T> = T extends Promise<infer U> ? U : T;
// This is actually built into TypeScript as Awaited<T>

// The element type of an array
type ArrayElement<T> = T extends (infer E)[] ? E : never;

type InvoiceList = Invoice[];
type SingleInvoice = ArrayElement<InvoiceList>; // Invoice
```

In Runway, conditional types appear in the event catalog typing (Chapter 8) and the generic repository interfaces.

---

## Template Literal Types — Typed String Patterns

TypeScript can express string patterns as types.

```typescript
type Stage = "dev" | "staging" | "production";
type EnvVar = `${Uppercase<Stage>}_DATABASE_URL`;
// "DEV_DATABASE_URL" | "STAGING_DATABASE_URL" | "PRODUCTION_DATABASE_URL"

// DynamoDB key patterns
type WorkspaceKey = `WORKSPACE#${string}`;
type ClientKey = `CLIENT#${string}`;
type InvoiceKey = `INVOICE#${string}`;

function getInvoiceKey(invoiceId: string): InvoiceKey {
  return `INVOICE#${invoiceId}`;
}

const key: InvoiceKey = "INVOICE#abc123";    // Fine
const bad: InvoiceKey = "WORKSPACE#abc123";  // Error: type '"WORKSPACE#..."' is not assignable
```

Template literal types don't validate at runtime — they're compile-time only. For runtime validation, use Zod with `.startsWith("INVOICE#")` or a regex.

---

## Mapped Types — Transforming Object Shapes

Mapped types create new types by iterating over the keys of an existing type.

```typescript
// Make all fields optional (the built-in Partial<T> does this)
type Partial<T> = { [K in keyof T]?: T[K] };

// Make specific fields required
type RequireFields<T, K extends keyof T> = T & Required<Pick<T, K>>;

// Omit all methods, keep only data fields
type DataOnly<T> = {
  [K in keyof T as T[K] extends Function ? never : K]: T[K];
};

// Runway example: update request type from full entity type
type InvoiceUpdateInput = Partial<Omit<Invoice, "invoiceId" | "workspaceId" | "createdAt" | "version">>;
// Fields that can be updated, all optional, without the immutable fields
```

The built-in TypeScript utility types (`Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record`, `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`) are all implemented as mapped or conditional types. Knowing they exist saves you from writing them.

---

## Null Handling Patterns

With `strictNullChecks: true` (part of `strict`), TypeScript tracks `null` and `undefined` separately. Handle them explicitly:

```typescript
// Nullish coalescing — use default when null/undefined
const name = invoice.clientName ?? "Unknown Client";

// Optional chaining — short-circuit on null/undefined
const email = user?.profile?.email;  // undefined if any step is null/undefined

// Non-null assertion — use when you know it's not null
// (but prefer narrowing when possible)
const id = event.pathParameters!.invoiceId;

// Better — narrow explicitly
const id = event.pathParameters?.invoiceId;
if (!id) return { statusCode: 400, body: "..." };
// id is string here, not string | undefined

// Nullish assignment — assign only if currently null/undefined
options.timeout ??= 5000;
```

**Avoid `!` (non-null assertion)** where possible. It tells the compiler to trust you, not itself. When you're wrong, the crash happens at runtime with no type error to warn you.

---

## tsconfig.json for Lambda

The TypeScript config that makes Runway's Lambda functions safe and fast to build:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "dist",
    "rootDir": "src",

    // Strict mode — all of these matter
    "strict": true,
    "noUncheckedIndexedAccess": true,  // array[n] returns T | undefined
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": false, // Often too strict for API types

    // Source maps for readable CloudWatch stack traces
    "sourceMap": true,
    "inlineSources": true,

    // Module resolution
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,  // Skip .d.ts checks in node_modules

    // Path aliases (optional)
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

**`noUncheckedIndexedAccess`** is not part of `strict` but should be. It makes `array[0]` return `T | undefined` instead of `T`, which forces you to handle the case where the array is empty. Worth enabling.

**`NODE_OPTIONS="--enable-source-maps"`** in the Lambda environment translates stack traces from compiled line numbers back to source lines. SST sets this automatically when you use the `nodejs` runtime.

---

## Quick Reference: When to Use Which Pattern

| Situation | Pattern |
|---|---|
| Config object that must match a type but keep literal values | `satisfies` |
| Union of states where each state has different fields | Discriminated union |
| Making IDs of different entity types incompatible | Branded types |
| Unknown external data (JSON, API response) | `unknown` + Zod parse |
| Typed list of allowed string values | `as const` array + `typeof arr[number]` |
| Function that works for multiple types | Generic `<T>` |
| Extracting part of a type | Conditional type + `infer` |
| Creating a variant of an existing type (all optional, omit some fields) | Mapped utility types (`Partial`, `Omit`, `Pick`) |
| Typed string patterns (e.g., DynamoDB keys) | Template literal types |
| Possibly null/undefined values | Optional chaining `?.`, nullish coalescing `??` |
