# Appendix D: TypeScript Patterns Reference

The TypeScript patterns used throughout this book, explained in one place. These aren't exotic — they show up in production codebases and once you've seen each one, you'll use it everywhere.

---

## `satisfies` — Validate Without Widening

`satisfies` checks that a value matches a type without changing the inferred type of the value. The difference matters when you want type safety on the value *and* you want TypeScript to keep inferring the specific type.

Without `satisfies`, an explicit type annotation widens the inferred type:

```typescript
const config: Record<string, string> = {
  region: "eu-west-1",
  stage: "production",
};
config.region.toUpperCase(); // fine
config.oops;                 // no error — widened to Record<string, string>
```

With `satisfies`, the type is checked but TypeScript still infers the specific object literal type:

```typescript
const config = {
  region: "eu-west-1",
  stage: "production",
} satisfies Record<string, string>;

config.region.toUpperCase(); // fine — TypeScript knows it's a string
config.oops;                 // error — 'oops' does not exist on this type
```

Where this shows up in Runway: route handler registries, SST config objects, error code maps. Each code key keeps its literal type rather than being widened to `string`:

```typescript
const HTTP_ERRORS = {
  NOT_FOUND: { status: 404, message: "Not found" },
  UNAUTHORIZED: { status: 401, message: "Unauthorized" },
  CONFLICT: { status: 409, message: "Conflict" },
} satisfies Record<string, { status: number; message: string }>;

HTTP_ERRORS.NOT_FOUND.status; // number (not Record<string, ...>)
HTTP_ERRORS.OOPS;             // error: property does not exist
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

getWorkspace(workspaceId, userId); // fine
getWorkspace(userId, workspaceId); // error: Argument of type 'UserId' is not assignable to 'WorkspaceId'
```

**Where to use brands:** IDs that cross module boundaries, monetary amounts (avoid mixing cents and pounds), anything where passing the wrong primitive would cause silent data corruption.

**Where not to use brands:** Everywhere. They add friction. Use them for IDs that appear frequently in function signatures where confusion is plausible.

---

## `unknown` vs `any` — Choosing Correctly

`any` disables the type checker. `unknown` keeps it engaged.

`any` disables the type checker entirely — TypeScript won't catch errors at the call site:

```typescript
function processPayload(data: any) {
  data.invoiceId.toUpperCase(); // allowed even if data.invoiceId is undefined at runtime
}
```

`unknown` requires narrowing before use:

```typescript
function processPayload(data: unknown) {
  data.invoiceId; // error: Object is of type 'unknown'

  if (typeof data === "object" && data !== null && "invoiceId" in data) {
    const id = (data as { invoiceId: string }).invoiceId;
  }

  // better: use Zod to parse and narrow in one step
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
interface PageResult<T> {
  items: T[];
  cursor: string | null;
  hasMore: boolean;
}

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
```

Usage — TypeScript infers `result.items` as `Workspace[]` from the decode function:

```typescript
const result = await paginatedQuery(
  { TableName: "RunwayTable", KeyConditionExpression: "PK = :pk", ExpressionAttributeValues: { ":pk": "WORKSPACE#123" } },
  (item) => WorkspaceSchema.parse(item)
);
```

---

## `as const` — Literal Types for Config Objects

`as const` makes all values in an object or array their narrowest possible literal type. Useful for configuration, enums, and lookup tables.

Without `as const`, array literals widen to `string[]`, losing literal types:

```typescript
const PLANS = ["free", "pro", "enterprise"];
// inferred as string[]
```

With `as const`, the array becomes `readonly ["free", "pro", "enterprise"]`:

```typescript
const PLANS = ["free", "pro", "enterprise"] as const;
type Plan = typeof PLANS[number]; // "free" | "pro" | "enterprise"

function getPlanLimit(plan: Plan): number {
  const limits = {
    free: 5,
    pro: 50,
    enterprise: Infinity,
  } as const satisfies Record<Plan, number>;

  return limits[plan];
}

getPlanLimit("pro");    // fine
getPlanLimit("trial");  // error: not a valid Plan
```

---

## Conditional Types — Type Transformations

Conditional types express "if T extends U, then X, else Y." Used internally by TypeScript's utility types and occasionally useful in application code.

```typescript
type ExtractSuccess<T> = T extends { success: true; data: infer D } ? D : never;

type CreateInvoiceResult =
  | { success: true; data: Invoice }
  | { success: false; error: string };

type InvoiceData = ExtractSuccess<CreateInvoiceResult>; // resolves to Invoice
```

**`infer`** captures a type from within a conditional. It's how you extract the resolved type of a Promise or the element type of an array:

```typescript
type Awaited<T> = T extends Promise<infer U> ? U : T;
// built into TypeScript 4.5+ as the Awaited<T> utility type

type ArrayElement<T> = T extends (infer E)[] ? E : never;

type InvoiceList = Invoice[];
type SingleInvoice = ArrayElement<InvoiceList>; // resolves to Invoice
```

In Runway, conditional types appear in the event catalog typing (Chapter 8) and the generic repository interfaces.

---

## Template Literal Types — Typed String Patterns

TypeScript can express string patterns as types.

```typescript
type Stage = "dev" | "staging" | "production";
type EnvVar = `${Uppercase<Stage>}_DATABASE_URL`;
// resolves to "DEV_DATABASE_URL" | "STAGING_DATABASE_URL" | "PRODUCTION_DATABASE_URL"
```

The Runway DynamoDB access layer uses template literal types to make key shapes explicit:

```typescript
type WorkspaceKey = `WORKSPACE#${string}`;
type ClientKey = `CLIENT#${string}`;
type InvoiceKey = `INVOICE#${string}`;

function getInvoiceKey(invoiceId: string): InvoiceKey {
  return `INVOICE#${invoiceId}`;
}

const key: InvoiceKey = "INVOICE#abc123";    // fine
const bad: InvoiceKey = "WORKSPACE#abc123";  // error: type '"WORKSPACE#..."' is not assignable
```

Template literal types don't validate at runtime — they're compile-time only. For runtime validation, use Zod with `.startsWith("INVOICE#")` or a regex.

---

## Mapped Types — Transforming Object Shapes

Mapped types create new types by iterating over the keys of an existing type.

```typescript
type Partial<T> = { [K in keyof T]?: T[K] };   // all fields optional (built-in)

type RequireFields<T, K extends keyof T> = T & Required<Pick<T, K>>;  // specific fields required

type DataOnly<T> = {                            // strip all methods, keep data fields only
  [K in keyof T as T[K] extends Function ? never : K]: T[K];
};
```

In Runway, `InvoiceUpdateInput` is derived from the full entity type — optional fields, immutable keys excluded:

```typescript
type InvoiceUpdateInput = Partial<Omit<Invoice, "invoiceId" | "workspaceId" | "createdAt" | "version">>;
```

The built-in TypeScript utility types (`Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record`, `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`) are all implemented as mapped or conditional types. Knowing they exist saves you from writing them.

---

## Null Handling Patterns

With `strictNullChecks: true` (part of `strict`), TypeScript tracks `null` and `undefined` separately. Handle them explicitly:

```typescript
const name = invoice.clientName ?? "Unknown Client";  // nullish coalescing: default on null/undefined

const email = user?.profile?.email;  // optional chaining: undefined if any step is null

const id = event.pathParameters!.invoiceId;  // non-null assertion: avoid if you can narrow instead

const id = event.pathParameters?.invoiceId;  // better: narrow explicitly
if (!id) return { statusCode: 400, body: "..." };
// id is string here, not string | undefined

options.timeout ??= 5000;  // nullish assignment: assign only if currently null/undefined
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

    // strict mode — all of these matter
    "strict": true,
    "noUncheckedIndexedAccess": true,  // array[n] returns T | undefined
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": false, // often too strict for API types

    // source maps for readable CloudWatch stack traces
    "sourceMap": true,
    "inlineSources": true,

    // module resolution
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,  // skip .d.ts checks in node_modules

    // path aliases (optional)
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
