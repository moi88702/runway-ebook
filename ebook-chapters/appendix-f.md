# Appendix F: Runway Data Model Reference

This appendix is the complete DynamoDB data model for the Runway application as built across chapters 4, 6, 7, and 8. Use it as a reference when adding new features, debugging access patterns, or designing migrations.

---

## Table Configuration

One table. All entities. SST definition:

```typescript
// sst.config.ts
const table = new sst.aws.Dynamo("RunwayTable", {
  fields: {
    pk:     "string",
    sk:     "string",
    gsi1pk: "string",
    gsi1sk: "string",
    gsi2pk: "string",
  },
  primaryIndex: { hashKey: "pk", rangeKey: "sk" },
  globalIndexes: {
    gsi1: { hashKey: "gsi1pk", rangeKey: "gsi1sk" },
    gsi2: { hashKey: "gsi2pk", rangeKey: "sk" },
  },
});
```

**Primary index:** `pk` (hash) + `sk` (range)
**GSI1:** `gsi1pk` (hash) + `gsi1sk` (range) — workspace-scoped invoice queries
**GSI2:** `gsi2pk` (hash) + `sk` (range) — entity lookups without parent context

DynamoDB attributes are only defined in `fields` if they appear in an index. All other attributes (`name`, `status`, `createdAt`, etc.) are stored but not declared — DynamoDB is schemaless beyond the key structure.

---

## Key Naming Conventions

| Concept | Format | Example |
|---------|--------|---------|
| Workspace scope | `WORKSPACE#<id>` | `WORKSPACE#ws_xK9m` |
| Client entity | `CLIENT#<id>` | `CLIENT#cl_3nPq` |
| Project entity | `PROJECT#<id>` | `PROJECT#pr_7tKj` |
| Invoice entity | `INVOICE#<id>` | `INVOICE#inv_2rBw` |
| Deliverable entity | `DELIVERABLE#<projectId>#<deliverableId>` | `DELIVERABLE#pr_7tKj#dlv_9mXc` |
| Email idempotency | `EMAIL_SENT#<type>` | `EMAIL_SENT#paid` |
| Chase dedup | `CHASED#<date>` | `CHASED#2024-11-15` |
| Workspace root | `WORKSPACE` (literal) | — |

All IDs are strings with a short prefix (`ws_`, `cl_`, `pr_`, `inv_`, `dlv_`) that identifies the entity type without needing to parse the key. The prefix is set by the ID generation function at creation time.

---

## Entity Reference

### Workspace

Represents a freelancer or agency account. Every other entity belongs to a workspace.

| Attribute | Key Role | Value |
|-----------|----------|-------|
| `pk` | Hash key | `WORKSPACE#<workspaceId>` |
| `sk` | Range key | `WORKSPACE` |

**TypeScript type:**

```typescript
type WorkspaceItem = {
  pk:          string;   // WORKSPACE#<workspaceId>
  sk:          string;   // "WORKSPACE"
  type:        "WORKSPACE";
  workspaceId: string;
  name:        string;
  ownerId:     string;   // Cognito sub — USER#<sub>
  plan:        "free" | "pro" | "enterprise";
  createdAt:   string;   // ISO 8601
  updatedAt:   string;
};
```

**Access patterns:**

| Pattern | Operation | Key condition |
|---------|-----------|---------------|
| Get workspace | `GetItem` | `pk = WORKSPACE#<id>`, `sk = WORKSPACE` |
| Get by owner (auth check) | `GetItem` | `pk = WORKSPACE#<id>`, `sk = WORKSPACE` — then compare `ownerId` |

---

### Client

A client belongs to a workspace. Carries a GSI2 key to support ownership checks by client ID alone (without knowing the workspace ID upfront).

| Attribute | Key Role | Value |
|-----------|----------|-------|
| `pk` | Hash key | `WORKSPACE#<workspaceId>` |
| `sk` | Range key | `CLIENT#<clientId>` |
| `gsi2pk` | GSI2 hash | `CLIENT#<clientId>` |

**TypeScript type:**

```typescript
type ClientItem = {
  pk:          string;   // WORKSPACE#<workspaceId>
  sk:          string;   // CLIENT#<clientId>
  gsi2pk:      string;   // CLIENT#<clientId>
  type:        "CLIENT";
  workspaceId: string;
  clientId:    string;
  name:        string;
  email:       string;
  company?:    string;
  createdAt:   string;
  updatedAt:   string;
};
```

**Access patterns:**

| Pattern | Operation | Key condition |
|---------|-----------|---------------|
| List clients in workspace | `Query` | `pk = WORKSPACE#<wsId>`, `sk begins_with CLIENT#` |
| Get client (with workspace) | `GetItem` | `pk = WORKSPACE#<wsId>`, `sk = CLIENT#<clientId>` |
| Get client by ID only (GSI2) | `Query` on GSI2 | `gsi2pk = CLIENT#<clientId>` — used by `getClientWithOwnerCheck` |

The GSI2 query is used in Chapter 7's `getClientWithOwnerCheck`: it fetches the client by ID alone (from the JWT claim), then verifies the returned `workspaceId` matches the authenticated user's workspace. This avoids accepting a `workspaceId` from the request.

---

### Project

A project belongs to a client and, transitively, to a workspace. Carries a GSI2 key for the same reason as `ClientItem` — to look up a project by ID without knowing its parent client.

| Attribute | Key Role | Value |
|-----------|----------|-------|
| `pk` | Hash key | `WORKSPACE#<workspaceId>` |
| `sk` | Range key | `PROJECT#<projectId>` |
| `gsi2pk` | GSI2 hash | `PROJECT#<projectId>` |

**TypeScript type:**

```typescript
type ProjectItem = {
  pk:           string;   // WORKSPACE#<workspaceId>
  sk:           string;   // PROJECT#<projectId>
  gsi2pk:       string;   // PROJECT#<projectId>
  type:         "PROJECT";
  workspaceId:  string;
  clientId:     string;
  projectId:    string;
  name:         string;
  description?: string;
  status:       "active" | "completed" | "archived";
  createdAt:    string;
  updatedAt:    string;
};
```

**Access patterns:**

| Pattern | Operation | Key condition |
|---------|-----------|---------------|
| List projects in workspace | `Query` | `pk = WORKSPACE#<wsId>`, `sk begins_with PROJECT#` |
| List projects for client | `Query` | `pk = WORKSPACE#<wsId>`, `sk begins_with PROJECT#` + filter `clientId` |
| Get project by ID (GSI2) | `Query` on GSI2 | `gsi2pk = PROJECT#<projectId>` |

---

### Invoice

An invoice belongs to a project. Carries GSI1 keys so all invoices across a workspace can be queried without scanning every project. The `clientEmail` and `clientName` fields are denormalized here in Chapter 8 to support the email worker and invoice chaser without extra lookups.

| Attribute | Key Role | Value |
|-----------|----------|-------|
| `pk` | Hash key | `PROJECT#<projectId>` |
| `sk` | Range key | `INVOICE#<invoiceId>` |
| `gsi1pk` | GSI1 hash | `WORKSPACE#<workspaceId>` |
| `gsi1sk` | GSI1 range | `INVOICE#<invoiceId>` |

**TypeScript type:**

```typescript
type InvoiceItem = {
  pk:            string;   // PROJECT#<projectId>
  sk:            string;   // INVOICE#<invoiceId>
  gsi1pk:        string;   // WORKSPACE#<workspaceId>
  gsi1sk:        string;   // INVOICE#<invoiceId>
  type:          "INVOICE";
  workspaceId:   string;
  clientId:      string;
  projectId:     string;
  invoiceId:     string;
  invoiceNumber: string;
  status:        "draft" | "sent" | "paid" | "overdue" | "cancelled";
  amountCents:   number;   // Integer — never float
  currency:      string;   // "GBP" | "USD" | "EUR" | ...
  dueDate:       string;   // ISO 8601 date
  clientEmail?:  string;   // Denormalized for email worker (added Ch. 8)
  clientName?:   string;   // Denormalized for chaser (added Ch. 8)
  paidAt?:       string;   // Set when status transitions to "paid"
  createdAt:     string;
  updatedAt:     string;
};
```

**Access patterns:**

| Pattern | Operation | Key condition |
|---------|-----------|---------------|
| Get invoice | `GetItem` | `pk = PROJECT#<projectId>`, `sk = INVOICE#<invoiceId>` |
| List invoices for project | `Query` | `pk = PROJECT#<projectId>`, `sk begins_with INVOICE#` |
| List invoices for workspace (GSI1) | `Query` on GSI1 | `gsi1pk = WORKSPACE#<wsId>`, `gsi1sk begins_with INVOICE#` |
| List invoices by status (GSI1 + filter) | `Query` on GSI1 | as above + `FilterExpression: status = :status` |
| Scan overdue invoices (cron) | `Scan` | `FilterExpression: #type = INVOICE AND #status = sent AND dueDate < :today` |

The overdue invoice scan in the `InvoiceChaser` cron job (Chapter 8) uses `Scan` rather than `Query`. This is acceptable for a daily background job — it only runs once a day and the table is bounded by workspace size, not global scale.

---

### Deliverable

A file uploaded by a workspace owner and associated with a project. Added in Chapter 6.

| Attribute | Key Role | Value |
|-----------|----------|-------|
| `pk` | Hash key | `WORKSPACE#<workspaceId>` |
| `sk` | Range key | `DELIVERABLE#<projectId>#<deliverableId>` |

**TypeScript type:**

```typescript
type DeliverableItem = {
  pk:             string;    // WORKSPACE#<workspaceId>
  sk:             string;    // DELIVERABLE#<projectId>#<deliverableId>
  type:           "DELIVERABLE";
  workspaceId:    string;
  projectId:      string;
  deliverableId:  string;
  filename:       string;
  s3Key:          string;    // Full S3 object key
  mimeType:       string;
  sizeBytes:      number;
  pageCount?:     number;    // PDF only — set by S3 event processor
  thumbnailKey?:  string;    // Set by S3 event processor (images)
  status:         "pending" | "ready";
  createdAt:      string;
  updatedAt:      string;
};
```

**Access patterns:**

| Pattern | Operation | Key condition |
|---------|-----------|---------------|
| Get deliverable | `GetItem` | `pk = WORKSPACE#<wsId>`, `sk = DELIVERABLE#<projectId>#<deliverableId>` |
| List deliverables for project | `Query` | `pk = WORKSPACE#<wsId>`, `sk begins_with DELIVERABLE#<projectId>#` |

The compound `sk` (`DELIVERABLE#<projectId>#<deliverableId>`) allows listing all deliverables for a specific project without a GSI — a `begins_with` filter on the sort key is sufficient because the project ID is embedded in the prefix.

---

### EmailSent (idempotency record)

A guard record written before sending each email. Prevents duplicate emails when SQS delivers a message more than once. Added in Chapter 8.

| Attribute | Key Role | Value |
|-----------|----------|-------|
| `pk` | Hash key | `INVOICE#<invoiceId>` |
| `sk` | Range key | `EMAIL_SENT#<type>` |

**TypeScript type:**

```typescript
type EmailSentItem = {
  pk:     string;   // INVOICE#<invoiceId>
  sk:     string;   // EMAIL_SENT#<type>  (e.g. EMAIL_SENT#paid)
  type:   "EMAIL_SENT";
  sentAt: string;
  // No TTL — these records are permanent dedup guards
};
```

**Access patterns:**

| Pattern | Operation | Key condition |
|---------|-----------|---------------|
| Check if email sent | `PutItem` with `ConditionExpression: attribute_not_exists(pk)` | `pk = INVOICE#<id>`, `sk = EMAIL_SENT#<type>` |

The `PutItem` with condition doubles as both the check and the write — it atomically confirms "not sent" and records "now sent" in a single operation. If the condition fails (`ConditionalCheckFailedException`), the email was already sent.

---

### InvoiceChase (cron dedup record)

A record written after each chase email is sent by the `InvoiceChaser` cron job. Prevents the same invoice being chased twice in one day. Expires automatically after 90 days via DynamoDB TTL. Added in Chapter 8.

| Attribute | Key Role | Value |
|-----------|----------|-------|
| `pk` | Hash key | `INVOICE#<invoiceId>` |
| `sk` | Range key | `CHASED#<date>` |

**TypeScript type:**

```typescript
type InvoiceChaseItem = {
  pk:       string;   // INVOICE#<invoiceId>
  sk:       string;   // CHASED#<YYYY-MM-DD>
  type:     "INVOICE_CHASE";
  chasedAt: string;
  ttl:      number;   // Unix timestamp — auto-expires after 90 days
};
```

**Access patterns:**

| Pattern | Operation | Key condition |
|---------|-----------|---------------|
| Check if chased today | `GetItem` | `pk = INVOICE#<id>`, `sk = CHASED#<today>` |
| Mark as chased | `PutItem` with condition | `pk = INVOICE#<id>`, `sk = CHASED#<today>` + `attribute_not_exists` |

---

## GSI Reference

### GSI1 — Workspace Invoice Index

**Hash key:** `gsi1pk` | **Range key:** `gsi1sk`

Only `InvoiceItem` populates this index.

| Query | Key condition |
|-------|---------------|
| All invoices for workspace | `gsi1pk = WORKSPACE#<wsId>` |
| All invoices for workspace, by ID range | `gsi1pk = WORKSPACE#<wsId>` AND `gsi1sk between INVOICE#<a> and INVOICE#<z>` |
| Invoices by status | `gsi1pk = WORKSPACE#<wsId>` + `FilterExpression: #status = :status` |

The primary key for invoices is `PROJECT#<projectId>`, which means there's no way to query all of a workspace's invoices without scanning every project partition. GSI1 solves this by projecting all invoice items into a workspace-scoped partition.

### GSI2 — Entity Self-Index

**Hash key:** `gsi2pk` | **Range key:** `sk`

`ClientItem` and `ProjectItem` populate this index. The range key is the same `sk` as the primary table, which means you can use `GetItem`-style lookups on GSI2 (query by `gsi2pk`, optionally filter by `sk`).

| Query | Key condition |
|-------|---------------|
| Get client by client ID | `gsi2pk = CLIENT#<clientId>` |
| Get project by project ID | `gsi2pk = PROJECT#<projectId>` |

Both queries return a single item in normal operation. The calling code verifies `item.workspaceId` against the authenticated user's workspace ID to prevent cross-workspace access.

---

## Complete Access Pattern Map

This table maps every original access pattern from Chapter 4's design exercise to the query it uses in the final implementation.

| # | Pattern | Index | Operation |
|---|---------|-------|-----------|
| 1 | Get workspace by ID | Primary | `GetItem` pk=`WORKSPACE#<id>` sk=`WORKSPACE` |
| 2 | Create workspace | Primary | `PutItem` |
| 3 | List clients for workspace | Primary | `Query` pk=`WORKSPACE#<id>` sk `begins_with CLIENT#` |
| 4 | Get client by ID | Primary | `GetItem` pk=`WORKSPACE#<id>` sk=`CLIENT#<id>` |
| 5 | Get client by ID (auth check, no workspace) | GSI2 | `Query` gsi2pk=`CLIENT#<id>` |
| 6 | Create client | Primary | `PutItem` |
| 7 | List projects for workspace | Primary | `Query` pk=`WORKSPACE#<id>` sk `begins_with PROJECT#` |
| 8 | List projects for client | Primary | `Query` pk=`WORKSPACE#<id>` sk `begins_with PROJECT#` + filter clientId |
| 9 | Get project by ID (auth check) | GSI2 | `Query` gsi2pk=`PROJECT#<id>` |
| 10 | Create project | Primary | `PutItem` |
| 11 | List invoices for project | Primary | `Query` pk=`PROJECT#<id>` sk `begins_with INVOICE#` |
| 12 | List invoices for workspace | GSI1 | `Query` gsi1pk=`WORKSPACE#<id>` |
| 13 | List invoices by status (workspace) | GSI1 | `Query` + `FilterExpression` |
| 14 | Get invoice | Primary | `GetItem` pk=`PROJECT#<id>` sk=`INVOICE#<id>` |
| 15 | Create invoice | Primary | `PutItem` |
| 16 | Update invoice status | Primary | `UpdateItem` with `ConditionExpression` |
| 17 | List overdue invoices (cron) | Primary | `Scan` + `FilterExpression` (status=sent, dueDate < today) |
| 18 | List deliverables for project | Primary | `Query` pk=`WORKSPACE#<id>` sk `begins_with DELIVERABLE#<projectId>#` |
| 19 | Get deliverable | Primary | `GetItem` pk=`WORKSPACE#<id>` sk=`DELIVERABLE#<projectId>#<id>` |
| 20 | Create deliverable | Primary | `PutItem` |
| 21 | Update deliverable metadata | Primary | `UpdateItem` (thumbnailKey, pageCount, status) |
| 22 | Check email sent (idempotency) | Primary | `PutItem` with `attribute_not_exists` condition |
| 23 | Check invoice chased today | Primary | `GetItem` pk=`INVOICE#<id>` sk=`CHASED#<date>` |
| 24 | Mark invoice chased | Primary | `PutItem` with condition + TTL |

---

## Item Type Discriminator

Every item in the table has a `type` field that identifies its entity type. This makes it safe to store multiple entity types in the same partition without ambiguity.

| `type` value | Entity | Chapters |
|---|---|---|
| `WORKSPACE` | Workspace root record | Ch. 4 |
| `CLIENT` | Client | Ch. 4 |
| `PROJECT` | Project | Ch. 4 |
| `INVOICE` | Invoice | Ch. 4 |
| `DELIVERABLE` | File deliverable | Ch. 6 |
| `EMAIL_SENT` | Email dedup guard | Ch. 8 |
| `INVOICE_CHASE` | Chase dedup record | Ch. 8 |

In practice, most DynamoDB queries target a single entity type — `pk = WORKSPACE#<id>` with `sk begins_with PROJECT#` will only ever return `PROJECT` items. The `type` discriminator is most useful for scan operations and for `switch` statements in generic repository utilities.

---

## Complete TypeScript Types File

All entity types together, as they appear in `src/types/dynamo.ts` at the end of Chapter 8:

```typescript
// src/types/dynamo.ts

export type DynamoItem = {
  pk: string;
  sk: string;
};

export type Gsi1Item = {
  gsi1pk: string;
  gsi1sk: string;
};

export type Gsi2Item = {
  gsi2pk: string;
};

export type WorkspaceItem = DynamoItem & {
  type:        "WORKSPACE";
  workspaceId: string;
  name:        string;
  ownerId:     string;
  plan:        "free" | "pro" | "enterprise";
  createdAt:   string;
  updatedAt:   string;
};

export type ClientItem = DynamoItem & Gsi2Item & {
  type:        "CLIENT";
  workspaceId: string;
  clientId:    string;
  name:        string;
  email:       string;
  company?:    string;
  createdAt:   string;
  updatedAt:   string;
};

export type ProjectItem = DynamoItem & Gsi2Item & {
  type:         "PROJECT";
  workspaceId:  string;
  clientId:     string;
  projectId:    string;
  name:         string;
  description?: string;
  status:       "active" | "completed" | "archived";
  createdAt:    string;
  updatedAt:    string;
};

export type InvoiceItem = DynamoItem & Gsi1Item & {
  type:          "INVOICE";
  workspaceId:   string;
  clientId:      string;
  projectId:     string;
  invoiceId:     string;
  invoiceNumber: string;
  status:        "draft" | "sent" | "paid" | "overdue" | "cancelled";
  amountCents:   number;
  currency:      string;
  dueDate:       string;
  clientEmail?:  string;
  clientName?:   string;
  paidAt?:       string;
  createdAt:     string;
  updatedAt:     string;
};

export type DeliverableItem = DynamoItem & {
  type:           "DELIVERABLE";
  workspaceId:    string;
  projectId:      string;
  deliverableId:  string;
  filename:       string;
  s3Key:          string;
  mimeType:       string;
  sizeBytes:      number;
  pageCount?:     number;
  thumbnailKey?:  string;
  status:         "pending" | "ready";
  createdAt:      string;
  updatedAt:      string;
};

export type EmailSentItem = DynamoItem & {
  type:   "EMAIL_SENT";
  sentAt: string;
};

export type InvoiceChaseItem = DynamoItem & {
  type:     "INVOICE_CHASE";
  chasedAt: string;
  ttl:      number;
};

export type RunwayItem =
  | WorkspaceItem
  | ClientItem
  | ProjectItem
  | InvoiceItem
  | DeliverableItem
  | EmailSentItem
  | InvoiceChaseItem;
```

The `RunwayItem` union type is the discriminated union of all item types. Repository functions that perform generic operations (scan, batch operations) can type their results as `RunwayItem` and use `switch (item.type)` to narrow to the specific type.

---

## Design Decisions

**Why single-table?** DynamoDB billing is per-request, not per-table. A single table with careful key design lets multiple entity types share a write unit without multiple round trips. The tradeoff: the key structure is less obvious than a relational schema. This appendix exists because of that tradeoff.

**Why not normalise ClientItem into the invoice?** Denormalizing `clientEmail` and `clientName` onto `InvoiceItem` (Chapter 8) avoids an extra `GetItem` call inside the email worker and the invoice chaser. Both run at high frequency on the hot path. The duplication is intentional and bounded — if the client's email changes, invoices keep the email address that was valid when they were sent, which is correct behaviour for an audit trail.

**Why `Scan` for overdue invoices?** The `InvoiceChaser` cron scans the full table once per day to find invoices with `status = "sent"` and `dueDate < today`. A full table scan on a busy production table would be expensive. For Runway's scale, a single daily scan is acceptable. If the table grows beyond a few million items, the right fix is to add a GSI with a TTL-based sparse index or move overdue detection to an EventBridge Scheduler per invoice.

**Why no `updatedAt` sort key?** Sort keys enable range queries. An `updatedAt` sort key would let you query "all invoices updated in the last 7 days" without a scan. Runway doesn't have that access pattern in its current form — if it did, a time-based GSI would be the right addition.
