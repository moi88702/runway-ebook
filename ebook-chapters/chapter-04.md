# Chapter 4: DynamoDB with Full Type Safety

This is the chapter where Runway gets a real database.

Up to this point, our Lambda handlers have been returning mock data. In this chapter we'll design Runway's entire data model, implement it in DynamoDB, and build the typed repository layer that the rest of the book builds on. By the end, every workspace, client, project, and invoice in Runway will persist in a real DynamoDB table — with TypeScript types that make it impossible to put the wrong data in the wrong place.

DynamoDB is also the chapter where people get lost. The mental model is genuinely different from relational databases, and most tutorials don't help because they either oversimplify (just treat it like a key-value store!) or overcomplicate (you need to understand all seventeen data modelling patterns before writing a single line of code).

We're going to do it the practical way: start with Runway's access patterns, design the table to support them, and build the code that talks to it. The concepts will make sense because you'll see why they're needed, not just what they are.

---

## 4.1 Why DynamoDB (and When Not To)

DynamoDB is a fully managed NoSQL database with predictable single-digit millisecond latency at any scale. That's the marketing pitch. The honest pitch is different.

**What DynamoDB is actually good at:**

- **Consistent performance at any scale.** A DynamoDB table with 10 items performs identically to one with 10 billion items, assuming you've designed your keys correctly. There's no query planner, no slow full-table scans degrading a shared connection pool. Read and write operations go straight to the right partition.

- **No server to manage.** It's genuinely serverless. No instance type to size, no connection pool to configure, no vacuum jobs to schedule. You pay per request and per GB stored. For Runway's early days, that means near-zero database costs.

- **It pairs naturally with Lambda.** Lambda functions are stateless and short-lived. Traditional databases require persistent connections, which Lambda handles poorly (connection pool exhaustion at scale is a real failure mode). DynamoDB uses an HTTP API — each request is independent.

**What DynamoDB is not good at:**

- **Ad-hoc queries.** If you need to run arbitrary SQL like `SELECT * FROM invoices WHERE client = 'Acme' AND amount > 1000 AND due_date < NOW()`, DynamoDB will fight you every step of the way. Every query must use a key, and the data model must be designed around your queries in advance.

- **Complex joins.** There are no joins. Related entities either live in the same item, or you fetch them in separate requests.

- **Reporting and analytics.** Aggregations, window functions, GROUP BY — none of this exists. For Runway's end-of-month revenue reports, we'll use Aurora Serverless (Chapter 5).

The rule of thumb for Runway: DynamoDB for operational data (the stuff that powers the live application — listing clients, fetching projects, creating invoices), Aurora for analytical data (the stuff that powers reports).

### The fundamental mental model shift

In a relational database, you design tables around your entities, then figure out queries later. Indexes are an optimisation you add when something is slow.

In DynamoDB, you design your table around your queries first. Entities are secondary. The table is a substrate for access patterns, not a normalised representation of your domain.

This feels backwards until it clicks. Then it feels obvious: you're building a system that serves specific use cases, and your data model should reflect that directly. There's no query planner to save you from a bad schema. There's no slow query log to alert you that a table scan is running at 3am. You design the access patterns in advance or you pay for it in production.

---

## 4.2 Single-Table Design: Runway's Data Model

Before writing a line of code, we need to answer: what does Runway need to do with its data?

### Access patterns

Runway has four entities — **Workspace**, **Client**, **Project**, and **Invoice** — each with the full set of standard CRUD endpoints (create, get, list, update). Those are all straightforward primary-key operations and don't drive the key design. The patterns that do are the ones that require querying across entity boundaries:

| # | Operation | Why it drives key design |
|---|-----------|--------------------------|
| 1 | List all clients in a workspace | Needs workspace-scoped query without scanning |
| 2 | List all projects in a workspace | Same — workspace is the natural list boundary |
| 3 | List all invoices for a project | Project is the parent; need a range query |
| 4 | List all invoices for a workspace | Cross-project; can't serve from primary key alone — drives GSI1 |
| 5 | Get invoice by ID (no project ID known) | Drives GSI2 in Chapter 7 |
| 6 | List unpaid/overdue invoices | Cross-workspace; Chapter 8 cron — drives sparse index or scan decision |
| 7 | Create project (atomic with workspace check) | Drives TransactWriteCommand in §4.7 |
| 8 | Update invoice status with version guard | Drives optimistic locking in §4.8 |

In a relational database, all eight are trivial `SELECT` queries. In DynamoDB, each must be served by a primary key lookup, a GSI query, or an explicitly designed access pattern. The table structure we design next serves all eight.

### Entities

Runway has four core entities:

The four core entities. `ownerId` is a Cognito user ID; Chapter 7 wires up the JWT extraction. All dates are ISO 8601 strings. `amountCents` is always an integer — never a float.

```typescript
type Workspace = {
  workspaceId: string;
  name: string;
  ownerId: string;
  createdAt: string;
  updatedAt: string;
};

type Client = {
  clientId: string;
  workspaceId: string;
  name: string;
  email: string;
  company?: string;
  createdAt: string;
  updatedAt: string;
};

type Project = {
  projectId: string;
  workspaceId: string;
  clientId: string;
  name: string;
  description?: string;
  status: "active" | "completed" | "archived";
  createdAt: string;
  updatedAt: string;
};

type Invoice = {
  invoiceId: string;
  workspaceId: string;
  clientId: string;
  projectId: string;
  invoiceNumber: string;
  status: "draft" | "sent" | "paid" | "overdue";
  amountCents: number;
  currency: string;
  dueDate: string;
  createdAt: string;
  updatedAt: string;
};
```

A note on `amountCents`: never store money as a float. `0.1 + 0.2 === 0.30000000000000004` in JavaScript. Store amounts as integer pence/cents, display them divided by 100. This avoids rounding errors in invoice totals that are extremely unpleasant to explain to paying clients.

### Key design

We're using a single DynamoDB table for all of Runway's operational data. One table, all entities.

The key structure:

| Entity | PK | SK |
|--------|----|----|
| Workspace | `WORKSPACE#<workspaceId>` | `METADATA` |
| Client | `WORKSPACE#<workspaceId>` | `CLIENT#<clientId>` |
| Project | `CLIENT#<clientId>` | `PROJECT#<projectId>` |
| Invoice | `PROJECT#<projectId>` | `INVOICE#<invoiceId>` |

This directly serves access patterns 1–3 and 7–11:

- **Get workspace** → `pk = WORKSPACE#<id>`, `sk = METADATA`
- **List clients** → `pk = WORKSPACE#<id>`, `sk begins_with CLIENT#`
- **Get client** → `pk = WORKSPACE#<id>`, `sk = CLIENT#<clientId>`
- **List projects** → `pk = CLIENT#<clientId>`, `sk begins_with PROJECT#`
- **Get project** → `pk = CLIENT#<clientId>`, `sk = PROJECT#<projectId>`
- **List invoices for project** → `pk = PROJECT#<projectId>`, `sk begins_with INVOICE#`
- **Get invoice** → `pk = PROJECT#<projectId>`, `sk = INVOICE#<invoiceId>`

Access patterns 12 and 16 — list all invoices for a workspace, and list unpaid invoices — need a GSI because the primary key is `PROJECT#`, not `WORKSPACE#`. We'll use a GSI with:

- `gsi1pk = WORKSPACE#<workspaceId>`
- `gsi1sk = INVOICE#<invoiceId>`

Every invoice item carries these attributes, giving us a workspace-scoped index of all invoices. For pattern 16 (unpaid invoices), we'll use a filter expression on `status` — acceptable here because filtering a workspace's invoices is never a full-table scan, just filtering within a bounded partition.

### Putting it together

Here's the complete item shape for each entity, showing all attributes including the key fields and GSI fields:

Every item in the table extends `DynamoItem`. Items that appear in GSI1 additionally carry `gsi1pk` and `gsi1sk`.

```typescript
type DynamoItem = {
  pk: string;
  sk: string;
};

type Gsi1Item = {
  gsi1pk: string;
  gsi1sk: string;
};

type WorkspaceItem = DynamoItem & {
  type: "WORKSPACE";
  workspaceId: string;
  name: string;
  ownerId: string;
  createdAt: string;
  updatedAt: string;
};

type ClientItem = DynamoItem & {
  type: "CLIENT";
  workspaceId: string;
  clientId: string;
  name: string;
  email: string;
  company?: string;
  createdAt: string;
  updatedAt: string;
};

type ProjectItem = DynamoItem & {
  type: "PROJECT";
  workspaceId: string;
  clientId: string;
  projectId: string;
  name: string;
  description?: string;
  status: "active" | "completed" | "archived";
  createdAt: string;
  updatedAt: string;
};

type InvoiceItem = DynamoItem & Gsi1Item & {
  type: "INVOICE";
  workspaceId: string;
  clientId: string;
  projectId: string;
  invoiceId: string;
  invoiceNumber: string;
  status: "draft" | "sent" | "paid" | "overdue";
  amountCents: number;
  currency: string;
  dueDate: string;
  createdAt: string;
  updatedAt: string;
};
```

The `type` field on every item is a discriminator — it tells you what kind of entity you're looking at when you pull items out of DynamoDB. Useful for `switch` statements in repository functions, and essential when you're querying a partition that might contain multiple entity types.

---

## 4.3 Setting Up the Table in SST

Add the table to `sst.config.ts`:

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
    const table = new sst.aws.Dynamo("RunwayTable", {
      fields: {
        pk: "string",
        sk: "string",
        gsi1pk: "string",
        gsi1sk: "string",
      },
      primaryIndex: {
        hashKey: "pk",
        rangeKey: "sk",
      },
      globalIndexes: {
        gsi1: {
          hashKey: "gsi1pk",
          rangeKey: "gsi1sk",
        },
      },
    });

    const api = new sst.aws.ApiGatewayV2("RunwayApi");

    const routeDefaults = {
      link: [table],
    };

    api.route("GET /health", {
      handler: "src/functions/health.handler",
    });

    api.route("GET /v1/workspaces/{workspaceId}", {
      handler: "src/functions/workspaces/get.handler",
      ...routeDefaults,
    });

    api.route("POST /v1/workspaces", {
      handler: "src/functions/workspaces/create.handler",
      ...routeDefaults,
    });

    api.route("GET /v1/workspaces/{workspaceId}/clients", {
      handler: "src/functions/clients/list.handler",
      ...routeDefaults,
    });

    api.route("GET /v1/workspaces/{workspaceId}/clients/{clientId}", {
      handler: "src/functions/clients/get.handler",
      ...routeDefaults,
    });

    api.route("POST /v1/workspaces/{workspaceId}/clients", {
      handler: "src/functions/clients/create.handler",
      ...routeDefaults,
    });

    api.route("GET /v1/clients/{clientId}/projects", {
      handler: "src/functions/projects/list.handler",
      ...routeDefaults,
    });

    api.route("POST /v1/clients/{clientId}/projects", {
      handler: "src/functions/projects/create.handler",
      ...routeDefaults,
    });

    api.route("GET /v1/projects/{projectId}/invoices", {
      handler: "src/functions/invoices/list.handler",
      ...routeDefaults,
    });

    api.route("POST /v1/projects/{projectId}/invoices", {
      handler: "src/functions/invoices/create.handler",
      ...routeDefaults,
    });

    return {
      api: api.url,
      table: table.name,
    };
  },
});
```

SST's `globalIndexes` creates the GSI alongside the table. The `fields` object tells SST which attributes are key attributes — these must be declared here even if they're only used in a GSI. Attributes that aren't keys (like `name`, `status`, `amountCents`) don't need to be declared.

---

## 4.4 The Typed DynamoDB Client

Install the AWS SDK v3 DynamoDB packages:

```bash
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

Create the client module:

```typescript
// src/lib/dynamo.ts
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});

export const docClient = DynamoDBDocumentClient.from(client, {
  marshallOptions: {
    removeUndefinedValues: true,
    convertEmptyValues: false,
  },
});
```

Two things to note here:

**`removeUndefinedValues: true`** means if you put an item with `company: undefined`, DynamoDB won't create a `company` attribute on the item at all. Without this, you'd get a marshalling error. This is almost always what you want.

**`convertEmptyValues: false`** means if you accidentally try to store an empty string, DynamoDB throws an error rather than silently converting it to `null`. Silent type coercion in a database is how you get data that looks correct but isn't.

Now build the key helpers. These are the functions that construct DynamoDB key objects — centralising this means you only write the key pattern once:

```typescript
// src/lib/keys.ts

export const keys = {
  workspace: {
    pk: (workspaceId: string) => `WORKSPACE#${workspaceId}`,
    sk: () => "METADATA",
    primary: (workspaceId: string) => ({
      pk: keys.workspace.pk(workspaceId),
      sk: keys.workspace.sk(),
    }),
  },
  client: {
    pk: (workspaceId: string) => `WORKSPACE#${workspaceId}`,
    sk: (clientId: string) => `CLIENT#${clientId}`,
    primary: (workspaceId: string, clientId: string) => ({
      pk: keys.client.pk(workspaceId),
      sk: keys.client.sk(clientId),
    }),
  },
  project: {
    pk: (clientId: string) => `CLIENT#${clientId}`,
    sk: (projectId: string) => `PROJECT#${projectId}`,
    primary: (clientId: string, projectId: string) => ({
      pk: keys.project.pk(clientId),
      sk: keys.project.sk(projectId),
    }),
  },
  invoice: {
    pk: (projectId: string) => `PROJECT#${projectId}`,
    sk: (invoiceId: string) => `INVOICE#${invoiceId}`,
    primary: (projectId: string, invoiceId: string) => ({
      pk: keys.invoice.pk(projectId),
      sk: keys.invoice.sk(invoiceId),
    }),
    gsi1: (workspaceId: string, invoiceId: string) => ({
      gsi1pk: `WORKSPACE#${workspaceId}`,
      gsi1sk: `INVOICE#${invoiceId}`,
    }),
  },
} as const;
```

This is worth the ceremony. When you're debugging a production issue at midnight, you want to search your codebase for `keys.invoice.pk` and find every place that constructs an invoice primary key — not grep for string literals scattered across 30 files.

---

## 4.5 CRUD Patterns with Full Types

Now the actual data operations. We'll build these as a repository layer — plain functions, no classes, no ORMs.

### Workspaces

```typescript
// src/repositories/workspaces.ts
import { Resource } from "sst";
import {
  GetCommand,
  PutCommand,
  UpdateCommand,
} from "@aws-sdk/lib-dynamodb";
import { ConditionalCheckFailedException } from "@aws-sdk/client-dynamodb";
import { docClient } from "../lib/dynamo";
import { keys } from "../lib/keys";
import type { WorkspaceItem } from "../types/dynamo";
import { AppError } from "../lib/errors";

const TABLE = Resource.RunwayTable.name;

export async function getWorkspace(
  workspaceId: string,
): Promise<WorkspaceItem | null> {
  const result = await docClient.send(
    new GetCommand({
      TableName: TABLE,
      Key: keys.workspace.primary(workspaceId),
    }),
  );

  return (result.Item as WorkspaceItem) ?? null;
}

export async function createWorkspace(data: {
  workspaceId: string;
  name: string;
  ownerId: string;
}): Promise<WorkspaceItem> {
  const now = new Date().toISOString();

  const item: WorkspaceItem = {
    ...keys.workspace.primary(data.workspaceId),
    type: "WORKSPACE",
    workspaceId: data.workspaceId,
    name: data.name,
    ownerId: data.ownerId,
    createdAt: now,
    updatedAt: now,
  };

  try {
    await docClient.send(
      new PutCommand({
        TableName: TABLE,
        Item: item,
        ConditionExpression: "attribute_not_exists(pk)",
      }),
    );
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new AppError("Workspace already exists", "WORKSPACE_EXISTS", 409);
    }
    throw err;
  }

  return item;
}

export async function updateWorkspaceName(
  workspaceId: string,
  name: string,
): Promise<void> {
  try {
    await docClient.send(
      new UpdateCommand({
        TableName: TABLE,
        Key: keys.workspace.primary(workspaceId),
        UpdateExpression: "SET #name = :name, updatedAt = :updatedAt",
        ExpressionAttributeNames: {
          "#name": "name",  // 'name' is a reserved word in DynamoDB expressions
        },
        ExpressionAttributeValues: {
          ":name": name,
          ":updatedAt": new Date().toISOString(),
        },
        ConditionExpression: "attribute_exists(pk)",
      }),
    );
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new AppError("Workspace not found", "NOT_FOUND", 404);
    }
    throw err;
  }
}
```

Three patterns that appear everywhere in this codebase:

**`ConditionExpression: "attribute_not_exists(pk)"` on Put** — prevents overwriting an existing item. Without it, `PutCommand` is an upsert. If two requests race to create the same workspace, the second one wins silently. With the condition, the second one throws `ConditionalCheckFailedException` and you can return a 409.

**`ExpressionAttributeNames` with `#name`** — DynamoDB has a long list of reserved words (`name`, `status`, `type`, `date`, `size`, `value`, and dozens more). If your attribute name is in the list, you must alias it with a `#` prefix in your expressions. The `ExpressionAttributeNames` object maps the alias to the real attribute name. You'll be doing this constantly.

**`ConditionExpression: "attribute_exists(pk)"` on Update** — ensures the item exists before updating. Without it, UpdateCommand creates the item if it doesn't exist. Usually not what you want — if the workspace doesn't exist, you want a 404, not a ghost record.

### Clients

```typescript
// src/repositories/clients.ts
import { Resource } from "sst";
import {
  GetCommand,
  PutCommand,
  UpdateCommand,
  DeleteCommand,
  QueryCommand,
} from "@aws-sdk/lib-dynamodb";
import { ConditionalCheckFailedException } from "@aws-sdk/client-dynamodb";
import { docClient } from "../lib/dynamo";
import { keys } from "../lib/keys";
import type { ClientItem } from "../types/dynamo";
import { AppError } from "../lib/errors";

const TABLE = Resource.RunwayTable.name;

export async function listClients(workspaceId: string): Promise<ClientItem[]> {
  const result = await docClient.send(
    new QueryCommand({
      TableName: TABLE,
      KeyConditionExpression:
        "pk = :pk AND begins_with(sk, :skPrefix)",
      ExpressionAttributeValues: {
        ":pk": keys.client.pk(workspaceId),
        ":skPrefix": "CLIENT#",
      },
    }),
  );

  return (result.Items as ClientItem[]) ?? [];
}

export async function getClient(
  workspaceId: string,
  clientId: string,
): Promise<ClientItem | null> {
  const result = await docClient.send(
    new GetCommand({
      TableName: TABLE,
      Key: keys.client.primary(workspaceId, clientId),
    }),
  );

  return (result.Item as ClientItem) ?? null;
}

export async function createClient(data: {
  workspaceId: string;
  clientId: string;
  name: string;
  email: string;
  company?: string;
}): Promise<ClientItem> {
  const now = new Date().toISOString();

  const item: ClientItem = {
    ...keys.client.primary(data.workspaceId, data.clientId),
    type: "CLIENT",
    workspaceId: data.workspaceId,
    clientId: data.clientId,
    name: data.name,
    email: data.email,
    company: data.company,
    createdAt: now,
    updatedAt: now,
  };

  await docClient.send(
    new PutCommand({
      TableName: TABLE,
      Item: item,
      ConditionExpression: "attribute_not_exists(pk)",
    }),
  );

  return item;
}

export async function updateClient(
  workspaceId: string,
  clientId: string,
  updates: { name?: string; email?: string; company?: string },
): Promise<void> {
  const expressions: string[] = [];
  const names: Record<string, string> = {};
  const values: Record<string, unknown> = {
    ":updatedAt": new Date().toISOString(),
  };

  if (updates.name !== undefined) {
    expressions.push("#name = :name");
    names["#name"] = "name";
    values[":name"] = updates.name;
  }
  if (updates.email !== undefined) {
    expressions.push("email = :email");
    values[":email"] = updates.email;
  }
  if (updates.company !== undefined) {
    expressions.push("company = :company");
    values[":company"] = updates.company;
  }

  if (expressions.length === 0) return;

  try {
    await docClient.send(
      new UpdateCommand({
        TableName: TABLE,
        Key: keys.client.primary(workspaceId, clientId),
        UpdateExpression: `SET ${expressions.join(", ")}, updatedAt = :updatedAt`,
        ExpressionAttributeNames: Object.keys(names).length ? names : undefined,
        ExpressionAttributeValues: values,
        ConditionExpression: "attribute_exists(pk)",
      }),
    );
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new AppError("Client not found", "NOT_FOUND", 404);
    }
    throw err;
  }
}

export async function deleteClient(
  workspaceId: string,
  clientId: string,
): Promise<void> {
  try {
    await docClient.send(
      new DeleteCommand({
        TableName: TABLE,
        Key: keys.client.primary(workspaceId, clientId),
        ConditionExpression: "attribute_exists(pk)",
      }),
    );
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new AppError("Client not found", "NOT_FOUND", 404);
    }
    throw err;
  }
}
```

The `updateClient` function builds the `UpdateExpression` dynamically based on which fields are present in `updates`. This is the right approach for partial updates — you don't want to require callers to send the full object just to change an email address, and you don't want to accidentally overwrite fields that weren't included.

### Invoices

Invoices are the most interesting entity because they span multiple key patterns:

```typescript
// src/repositories/invoices.ts
import { Resource } from "sst";
import {
  GetCommand,
  PutCommand,
  UpdateCommand,
  QueryCommand,
  TransactWriteCommand,
} from "@aws-sdk/lib-dynamodb";
import { ConditionalCheckFailedException } from "@aws-sdk/client-dynamodb";
import { docClient } from "../lib/dynamo";
import { keys } from "../lib/keys";
import type { InvoiceItem } from "../types/dynamo";
import { AppError } from "../lib/errors";

const TABLE = Resource.RunwayTable.name;

export async function listInvoicesByProject(
  projectId: string,
): Promise<InvoiceItem[]> {
  const result = await docClient.send(
    new QueryCommand({
      TableName: TABLE,
      KeyConditionExpression:
        "pk = :pk AND begins_with(sk, :skPrefix)",
      ExpressionAttributeValues: {
        ":pk": keys.invoice.pk(projectId),
        ":skPrefix": "INVOICE#",
      },
      ScanIndexForward: false,
    }),
  );

  return (result.Items as InvoiceItem[]) ?? [];
}

export async function listInvoicesByWorkspace(
  workspaceId: string,
): Promise<InvoiceItem[]> {
  const result = await docClient.send(
    new QueryCommand({
      TableName: TABLE,
      IndexName: "gsi1",
      KeyConditionExpression:
        "gsi1pk = :gsi1pk AND begins_with(gsi1sk, :gsi1skPrefix)",
      ExpressionAttributeValues: {
        ":gsi1pk": `WORKSPACE#${workspaceId}`,
        ":gsi1skPrefix": "INVOICE#",
      },
      ScanIndexForward: false,
    }),
  );

  return (result.Items as InvoiceItem[]) ?? [];
}

export async function listOutstandingInvoices(
  workspaceId: string,
): Promise<InvoiceItem[]> {
  const result = await docClient.send(
    new QueryCommand({
      TableName: TABLE,
      IndexName: "gsi1",
      KeyConditionExpression: "gsi1pk = :gsi1pk AND begins_with(gsi1sk, :gsi1skPrefix)",
      FilterExpression: "#status IN (:sent, :overdue)",
      ExpressionAttributeNames: {
        "#status": "status",
      },
      ExpressionAttributeValues: {
        ":gsi1pk": `WORKSPACE#${workspaceId}`,
        ":gsi1skPrefix": "INVOICE#",
        ":sent": "sent",
        ":overdue": "overdue",
      },
    }),
  );

  return (result.Items as InvoiceItem[]) ?? [];
}

export async function createInvoice(data: {
  workspaceId: string;
  clientId: string;
  projectId: string;
  invoiceId: string;
  invoiceNumber: string;
  amountCents: number;
  currency: string;
  dueDate: string;
}): Promise<InvoiceItem> {
  const now = new Date().toISOString();

  const item: InvoiceItem = {
    ...keys.invoice.primary(data.projectId, data.invoiceId),
    ...keys.invoice.gsi1(data.workspaceId, data.invoiceId),
    type: "INVOICE",
    workspaceId: data.workspaceId,
    clientId: data.clientId,
    projectId: data.projectId,
    invoiceId: data.invoiceId,
    invoiceNumber: data.invoiceNumber,
    status: "draft",
    amountCents: data.amountCents,
    currency: data.currency,
    dueDate: data.dueDate,
    createdAt: now,
    updatedAt: now,
  };

  await docClient.send(
    new PutCommand({
      TableName: TABLE,
      Item: item,
      ConditionExpression: "attribute_not_exists(pk)",
    }),
  );

  return item;
}

export async function updateInvoiceStatus(
  projectId: string,
  invoiceId: string,
  status: InvoiceItem["status"],
): Promise<void> {
  try {
    await docClient.send(
      new UpdateCommand({
        TableName: TABLE,
        Key: keys.invoice.primary(projectId, invoiceId),
        UpdateExpression: "SET #status = :status, updatedAt = :updatedAt",
        ExpressionAttributeNames: { "#status": "status" },
        ExpressionAttributeValues: {
          ":status": status,
          ":updatedAt": new Date().toISOString(),
        },
        ConditionExpression: "attribute_exists(pk)",
      }),
    );
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new AppError("Invoice not found", "NOT_FOUND", 404);
    }
    throw err;
  }
}
```

The `FilterExpression` in `listOutstandingInvoices` deserves attention. Filter expressions are applied *after* DynamoDB fetches items — they reduce what's returned to you, but DynamoDB still reads (and charges you for) every item in the partition before filtering. For a workspace with 1,000 invoices, this reads and discards up to 1,000 items just to find the 50 that are unpaid.

This is acceptable for Runway because:
1. Invoices are bounded per workspace — a freelancer with 1,000 invoices is a power user, not a typical case
2. The query is bounded to a single workspace's partition, not a full table scan
3. The alternative — a separate GSI keyed on status — adds write cost to every invoice operation

The rule: filter expressions are fine when the result set is bounded and the filtering ratio isn't terrible. They're not fine when you're filtering 99% of a large dataset. We're not doing that here.

---

## 4.6 Pagination

`QueryCommand` returns up to 1MB of data per call. When a result set exceeds 1MB, DynamoDB sets `LastEvaluatedKey` on the response to tell you where to resume. If you ignore this, you silently return partial results.

Build a pagination utility:

```typescript
// src/lib/paginate.ts
import { QueryCommand, type QueryCommandInput } from "@aws-sdk/lib-dynamodb";
import { docClient } from "./dynamo";

interface PaginatedResult<T> {
  items: T[];
  nextCursor: string | null;
}

export async function paginatedQuery<T>(
  input: QueryCommandInput,
  limit = 50,
  cursor?: string,
): Promise<PaginatedResult<T>> {
  const exclusiveStartKey = cursor
    ? JSON.parse(Buffer.from(cursor, "base64url").toString("utf-8"))
    : undefined;

  const result = await docClient.send(
    new QueryCommand({
      ...input,
      Limit: limit,
      ExclusiveStartKey: exclusiveStartKey,
    }),
  );

  const nextCursor = result.LastEvaluatedKey
    ? Buffer.from(JSON.stringify(result.LastEvaluatedKey)).toString("base64url")
    : null;

  return {
    items: (result.Items as T[]) ?? [],
    nextCursor,
  };
}
```

The cursor is base64url-encoded JSON — the exact `LastEvaluatedKey` object from DynamoDB, round-tripped through the API. Clients get an opaque string they can pass back on the next request. The implementation detail (DynamoDB's LastEvaluatedKey structure) never leaks to API consumers.

Update `listClients` to support pagination:

```typescript
export async function listClients(
  workspaceId: string,
  options: { limit?: number; cursor?: string } = {},
): Promise<PaginatedResult<ClientItem>> {
  return paginatedQuery<ClientItem>(
    {
      TableName: Resource.RunwayTable.name,
      KeyConditionExpression: "pk = :pk AND begins_with(sk, :skPrefix)",
      ExpressionAttributeValues: {
        ":pk": keys.client.pk(workspaceId),
        ":skPrefix": "CLIENT#",
      },
    },
    options.limit ?? 50,
    options.cursor,
  );
}
```

And the handler that exposes it:

```typescript
// src/functions/clients/list.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { listClients } from "../../repositories/clients";
import { res } from "../../lib/response";
import { getWorkspace } from "../../repositories/workspaces";

export const handler: APIGatewayProxyHandlerV2 = async (event) => {
  const workspaceId = event.pathParameters?.workspaceId!;
  const limit = event.queryStringParameters?.limit
    ? parseInt(event.queryStringParameters.limit, 10)
    : 50;
  const cursor = event.queryStringParameters?.cursor;

  const workspace = await getWorkspace(workspaceId);
  if (!workspace) return res.notFound("Workspace");

  const result = await listClients(workspaceId, {
    limit: Math.min(limit, 100),
    cursor,
  });

  return res.ok(result);
};
```

The response shape:

```json
{
  "items": [...],
  "nextCursor": "eyJwayI6Ildv..."
}
```

When `nextCursor` is `null`, the caller knows they've fetched everything. When it's a string, they pass it as `?cursor=<string>` on the next request.

---

## 4.7 Atomic Writes with TransactWriteCommand

Some operations in Runway need to write multiple items atomically — either all succeed or none do.

When a freelancer creates a new project, we want to:
1. Create the project item
2. Increment a project counter on the client item

If the project creation succeeds but the counter update fails, the workspace's project count is wrong. `TransactWriteCommand` handles this:

```typescript
// src/repositories/projects.ts (excerpt)
export async function createProject(data: {
  workspaceId: string;
  clientId: string;
  projectId: string;
  name: string;
  description?: string;
}): Promise<ProjectItem> {
  const now = new Date().toISOString();

  const project: ProjectItem = {
    ...keys.project.primary(data.clientId, data.projectId),
    type: "PROJECT",
    workspaceId: data.workspaceId,
    clientId: data.clientId,
    projectId: data.projectId,
    name: data.name,
    description: data.description,
    status: "active",
    createdAt: now,
    updatedAt: now,
  };

  await docClient.send(
    new TransactWriteCommand({
      TransactItems: [
        {
          Put: {
            TableName: Resource.RunwayTable.name,
            Item: project,
            ConditionExpression: "attribute_not_exists(pk)",
          },
        },
        {
          Update: {
            TableName: Resource.RunwayTable.name,
            Key: keys.client.primary(data.workspaceId, data.clientId),
            UpdateExpression:
              "SET projectCount = if_not_exists(projectCount, :zero) + :one, updatedAt = :updatedAt",
            ExpressionAttributeValues: {
              ":zero": 0,
              ":one": 1,
              ":updatedAt": now,
            },
            ConditionExpression: "attribute_exists(pk)",
          },
        },
      ],
    }),
  );

  return project;
}
```

`TransactWriteCommand` supports up to 25 operations in a single transaction. The entire transaction either succeeds or fails — no partial writes.

The `if_not_exists(projectCount, :zero)` pattern initialises `projectCount` to 0 if it doesn't exist on the item yet, then adds 1. This means you don't have to pre-populate `projectCount` on every client item — it starts counting from the first project creation.

---

## 4.8 Optimistic Locking

Optimistic locking prevents lost updates when two processes try to modify the same item concurrently.

Imagine two simultaneous requests both updating an invoice: one marking it as `sent`, another updating the `dueDate`. Without locking, the second write wins silently and the first update is lost.

Add a `version` attribute to any entity that needs it:

```typescript
type InvoiceItem = DynamoItem & Gsi1Item & {
  // ... other fields ...
  version: number;
};
```

Update `updateInvoiceStatus` to use optimistic locking:

```typescript
export async function updateInvoiceStatus(
  projectId: string,
  invoiceId: string,
  status: InvoiceItem["status"],
  currentVersion: number,
): Promise<void> {
  try {
    await docClient.send(
      new UpdateCommand({
        TableName: Resource.RunwayTable.name,
        Key: keys.invoice.primary(projectId, invoiceId),
        UpdateExpression:
          "SET #status = :status, updatedAt = :updatedAt, version = version + :inc",
        ConditionExpression:
          "attribute_exists(pk) AND version = :currentVersion",
        ExpressionAttributeNames: { "#status": "status" },
        ExpressionAttributeValues: {
          ":status": status,
          ":updatedAt": new Date().toISOString(),
          ":currentVersion": currentVersion,
          ":inc": 1,
        },
      }),
    );
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new AppError(
        "Invoice was modified by another request. Fetch the latest version and try again.",
        "CONFLICT",
        409,
      );
    }
    throw err;
  }
}
```

The condition `version = :currentVersion` means: "only apply this update if the version hasn't changed since I read it." If another request updated the invoice first, the version will be different and `ConditionalCheckFailedException` fires. The caller gets a 409 Conflict, fetches the latest invoice, and can decide whether to retry.

For Runway's invoice status updates, which are infrequent user-initiated actions, a 409 is fine. For automated processes that might race, you'd add a retry loop.

---

## 4.9 Common DynamoDB Mistakes

### Scanning when you should be querying

`ScanCommand` reads every item in the table. In a table with 10 million items, that's 10 million reads. At DynamoDB on-demand pricing, that's a bill.

Never use `ScanCommand` in application code. If you find yourself reaching for it, you either need a GSI or your access pattern requires a different tool (Aurora, OpenSearch).

The only acceptable uses of Scan:
- One-off data migration scripts (run once, not in Lambda)
- Admin tooling (explicitly rate-limited, never user-facing)
- Local development inspection

### Hot partitions

DynamoDB distributes data across physical partitions by primary key hash. If all your writes go to the same PK, they all hit the same partition — you get throttled.

The classic mistake: using a date as the partition key. All writes on a given day go to the same partition.

For Runway, our keys are `WORKSPACE#<uuid>` and `CLIENT#<uuid>` — UUIDs are random enough to distribute evenly across partitions. We're safe.

Watch out for: sequential IDs, timestamps, enum-based PKs with low cardinality (`STATUS#sent` for all sent invoices is a hot partition waiting to happen).

### Not handling pagination

DynamoDB returns at most 1MB per query. Applications that ignore `LastEvaluatedKey` silently return partial results. This is one of those bugs that doesn't manifest in development (data volumes are small) and causes mysterious production incidents.

Always use the `paginatedQuery` utility from section 4.6, or at minimum check `result.LastEvaluatedKey` and throw an error rather than silently truncating.

### Storing calculated totals in the table

It's tempting to store `invoiceTotal` on the workspace item and update it every time an invoice is created or paid. Don't. This creates a consistency problem — what if the update fails? What if two invoices are created simultaneously?

For Runway, the workspace overview shows invoice counts and totals. Compute these by querying invoices and summing in application code, or use Aurora for the aggregated reports. Keep DynamoDB items as the source of truth for individual entities, not pre-calculated aggregations.

### TTL for automatic data expiry

One DynamoDB feature worth knowing: Time to Live. Set a `ttl` attribute on an item (Unix timestamp in seconds) and DynamoDB will automatically delete it after that time. No Lambda, no cron job, no cost.

Runway uses this for temporary tokens — password reset links, email verification tokens:

```typescript
await docClient.send(
  new PutCommand({
    TableName: TABLE,
    Item: {
      pk: `TOKEN#${token}`,
      sk: "METADATA",
      type: "RESET_TOKEN",
      userId: userId,
      createdAt: now,
      ttl: Math.floor(Date.now() / 1000) + 3600, // expires in 1 hour
    },
  }),
);
```

TTL deletion isn't instant — DynamoDB typically processes expired items within 48 hours. Don't use it as a security mechanism. Do use it for cleaning up data you don't want to pay to store long-term.

---

## Where We Are

Runway now has a real database. Every workspace, client, project, and invoice persists in a single DynamoDB table designed around the application's access patterns.

The repository layer gives us:
- Typed reads and writes for all four entities
- Consistent error handling (`ConditionalCheckFailedException` → `AppError`)
- Pagination for list endpoints
- Atomic multi-item writes with `TransactWriteCommand`
- Optimistic locking for conflict-prone updates

The system architecture now looks like:

```
[API Gateway]
    ↓
[Lambda handlers]
    ↓
[Repository layer]  ← new
    ↓
[DynamoDB table]    ← new
    ├── Workspaces
    ├── Clients
    ├── Projects
    └── Invoices (+ GSI for workspace-scoped queries)
```

Chapter 5 adds the second data layer: Aurora Serverless for Runway's financial reporting. The operational data stays in DynamoDB; the analytical queries move to SQL where they belong.

---

> **The code for this chapter** is on the `chapter-4` branch of the companion repository.
