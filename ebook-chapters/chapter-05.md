# Chapter 5: RDS and Aurora Serverless

Runway's end-of-month revenue report looks like this: total invoiced, total paid, breakdown by client, breakdown by month, average project value. A freelancer wants to know if they're growing.

Try to build that in DynamoDB. You'll end up with six queries, a lot of application-side arithmetic, and a fragile mess that breaks the moment someone asks for year-over-year comparison.

Some queries need SQL. This chapter adds Aurora Serverless v2 to Runway's stack — a PostgreSQL-compatible relational database that scales to zero when idle and to 128 ACUs under load, with no servers to manage. DynamoDB handles the operational data (the live application). Aurora handles the analytical data (the reports).

By the end of this chapter, every paid invoice in Runway will be recorded in Aurora, and the reporting endpoints will serve real revenue data with proper SQL aggregations.

---

## 5.1 When to Reach for RDS

The question isn't "which database is better." It's "which database is better for this access pattern."

DynamoDB is better when:
- You know your access patterns in advance
- You need consistent sub-millisecond reads at any scale
- You're querying by primary key or a known GSI
- Your data model fits key-value or document patterns

Aurora (or any relational database) is better when:
- You need ad-hoc queries — flexible filters you can't predict
- You need aggregations: `SUM`, `COUNT`, `AVG`, `GROUP BY`
- You need joins across entities
- You're building reports where the query shape varies
- You have an existing schema that's inherently relational

For Runway, the split is clean:
- **DynamoDB**: workspaces, clients, projects, invoices (live application data)
- **Aurora**: invoice payments, revenue aggregations (report data)

### Aurora Serverless v2 vs provisioned RDS

Aurora Serverless v2 is the right choice for Runway because:

- **It scales to zero.** When no queries are running, it scales down and you pay nothing (roughly). For a staging environment that's idle most of the time, this is the difference between $30/month and $200/month.
- **It scales up automatically.** Under load, it scales from 0.5 ACUs (roughly 1GB RAM) to 128 ACUs in seconds. No instance resizing.
- **It's Aurora.** PostgreSQL-compatible, multi-AZ, automated backups, point-in-time recovery. Production-grade without any operational work.

The trade-off: Aurora Serverless v2 has a cold start (~3 seconds after being idle). For reporting endpoints, this is acceptable. For latency-sensitive application queries, it isn't — which is why we're not using it as Runway's primary database.

---

## 5.2 Aurora Serverless v2 with SST

### VPC: the unavoidable conversation

Aurora lives inside a VPC. Lambda functions that connect to Aurora also need to be inside the VPC. This is the bit that trips people up.

Lambda functions not in a VPC can reach the internet (and AWS services) directly. Lambda functions inside a VPC go through the VPC's network stack. To reach AWS services (DynamoDB, S3, SQS) from a VPC Lambda, you need VPC endpoints — otherwise traffic has to route out through a NAT gateway, which costs money and adds latency.

For Runway, the approach is:
- Aurora lives in the VPC (mandatory)
- Report Lambda functions go in the VPC (to reach Aurora)
- All other Lambda functions stay outside the VPC (to reach DynamoDB directly, cheaply)

SST makes VPC setup manageable. Add it to `sst.config.ts`:

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
      // ... (same as Chapter 4)
    });

    const vpc = new sst.aws.Vpc("RunwayVpc", {
      az: 2,
    });

    const db = new sst.aws.Aurora("RunwayDb", {
      engine: "postgres",
      vpc,
      scaling: {
        min: "0 ACU",
        max: "4 ACU",
      },
    });

    const api = new sst.aws.ApiGatewayV2("RunwayApi");

    api.route("GET /workspaces/{workspaceId}/reports/revenue", {
      handler: "src/functions/reports/revenue.handler",
      link: [db, table],
      vpc,  // ← puts this Lambda inside the VPC
    });

    api.route("GET /workspaces/{workspaceId}/reports/clients", {
      handler: "src/functions/reports/clients.handler",
      link: [db],
      vpc,
    });

    return {
      api: api.url,
      dbEndpoint: db.host,
    };
  },
});
```

`sst.aws.Vpc` creates a VPC with public and private subnets across two availability zones — the minimum Aurora requires for multi-AZ. SST handles the subnet creation, route tables, and internet gateway automatically.

`sst.aws.Aurora` creates the Aurora Serverless v2 cluster, the subnet group, and the security group allowing Lambda → Aurora traffic. The `link: [db]` on the Lambda routes injects `DATABASE_URL` as an environment variable and grants the necessary VPC permissions.

### Credentials and secret rotation

Aurora generates a master username and password. SST stores these in AWS Secrets Manager and injects them as `DATABASE_URL` when you link `db` to a function. You never see the password in your code or environment files.

The linked `db` resource exposes these fields on `Resource.RunwayDb`: `host` (Aurora endpoint), `port` (5432 for Postgres), `database`, `username`, and `password` (injected at runtime from Secrets Manager — never in your code).

For the Prisma connection URL:

```typescript
const databaseUrl =
  `postgresql://${Resource.RunwayDb.username}:${Resource.RunwayDb.password}` +
  `@${Resource.RunwayDb.host}:${Resource.RunwayDb.port}/${Resource.RunwayDb.database}`;
```

---

## 5.3 Connecting from Lambda

### The connection pooling problem

Traditional Node.js servers maintain a pool of database connections that persist across requests. A pool of 10 connections handles hundreds of concurrent requests efficiently.

Lambda doesn't work like this. Each Lambda execution environment maintains its own connections. With 100 concurrent Lambda invocations, you have 100 separate connections to Aurora. With 1,000 — 1,000 connections. PostgreSQL starts struggling around 300-500 connections. Aurora has a connection limit per ACU.

There are two solutions:

**RDS Proxy** — an AWS-managed connection pooler that sits between Lambda and Aurora. Lambda connects to RDS Proxy, which maintains a smaller pool of actual database connections. RDS Proxy handles the multiplexing.

Cost: ~$0.015 per vCPU hour of your Aurora instance, billed per hour. For Runway, that's roughly $15-30/month on top of Aurora costs.

**PgBouncer** — a lightweight open-source connection pooler you run yourself (on EC2 or ECS). Free, but requires operational work.

For Runway, use RDS Proxy. The cost is reasonable and the operational overhead is zero. Add it to `sst.config.ts`:

```typescript
const db = new sst.aws.Aurora("RunwayDb", {
  engine: "postgres",
  vpc,
  scaling: { min: "0 ACU", max: "4 ACU" },
  proxy: true,
});
```

With `proxy: true`, SST creates the RDS Proxy and updates `Resource.RunwayDb.host` to point at the proxy endpoint instead of the Aurora endpoint directly. Your connection string code doesn't change.

### Direct connection (without proxy)

For low-concurrency scenarios — a small side project, a staging environment, report jobs that run once a night — you can skip RDS Proxy and connect directly:

```typescript
const db = new sst.aws.Aurora("RunwayDb", {
  engine: "postgres",
  vpc,
  scaling: { min: "0 ACU", max: "4 ACU" },
});
```

The risk: if you get a traffic spike and spin up 200 Lambda instances simultaneously, each trying to open a database connection, you'll hit Aurora's connection limit and start seeing `too many connections` errors.

For Runway's reporting endpoints, which are low-frequency (users check reports a few times a day, not hundreds of times per second), direct connection is fine in staging. Use RDS Proxy in production.

---

## 5.4 Prisma in Lambda

Prisma is the ORM of choice for TypeScript + Aurora. It generates a fully-typed client from your schema, handles connection management, and produces readable SQL that you can inspect.

### Install and configure

```bash
npm install prisma @prisma/client
npx prisma init
```

Prisma creates a `prisma/` directory with `schema.prisma`. Update it for Runway's reporting model:

The `driverAdapters` preview feature switches Prisma to the WASM query engine (~6MB vs the default ~50MB native binary). This is essential for Lambda's package size limits.

```prisma
// prisma/schema.prisma

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["driverAdapters"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Workspace {
  id        String   @id
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  payments InvoicePayment[]
}

model InvoicePayment {
  id            String    @id @default(cuid())
  workspaceId   String
  workspace     Workspace @relation(fields: [workspaceId], references: [id])
  clientId      String
  clientName    String    // Denormalized — Aurora is for reporting, not lookups
  projectId     String
  projectName   String    // Denormalized
  invoiceId     String    @unique  // One payment record per invoice
  invoiceNumber String
  amountCents   Int       // Always integer cents
  currency      String    @default("GBP")
  paidAt        DateTime
  createdAt     DateTime  @default(now())

  @@index([workspaceId, paidAt])     // Revenue by workspace over time
  @@index([workspaceId, clientId])   // Revenue by client
}
```

Notice the denormalised `clientName` and `projectName`. In a reporting database, you want the data self-contained for the query you're running — you don't want a JOIN back to DynamoDB to look up names. When a client updates their name in DynamoDB, that doesn't retroactively change historical reports. This is correct behaviour.

### The Lambda bundle size problem

Prisma's default query engine is a native binary (~50MB). Lambda has a package size limit of 250MB (unzipped). More importantly, large packages mean slower cold starts.

The solution: the Prisma WASM engine, which is ~6MB and works in any environment.

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["driverAdapters"]
}
```

Then use `@prisma/adapter-pg` to connect:

```bash
npm install @prisma/adapter-pg pg
npm install -D @types/pg
```

```typescript
// src/lib/prisma.ts
import { PrismaClient } from "@prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";
import { Pool } from "pg";
import { Resource } from "sst";

let prisma: PrismaClient | undefined;

export function getPrismaClient(): PrismaClient {
  if (prisma) return prisma;

  const connectionString =
    `postgresql://${Resource.RunwayDb.username}:${Resource.RunwayDb.password}` +
    `@${Resource.RunwayDb.host}:${Resource.RunwayDb.port}/${Resource.RunwayDb.database}`;

  const pool = new Pool({
    connectionString,
    max: 1,
    idleTimeoutMillis: 0,
    connectionTimeoutMillis: 5000,
  });

  const adapter = new PrismaPg(pool);
  prisma = new PrismaClient({ adapter });

  return prisma;
}
```

`max: 1` is intentional. Each Lambda execution environment is single-threaded and handles one request at a time. A pool of one connection is correct. Larger pool sizes just mean you're holding connections open unnecessarily.

The module-level `prisma` variable means the client is created once per execution environment and reused across warm invocations — the same lazy initialisation pattern from Chapter 2.

---

## 5.5 Schema Migrations in Production

The question every Aurora-on-Lambda deployment has to answer: who runs `prisma migrate deploy`, and when?

**Why not from Lambda directly:**

- Lambda functions are stateless and concurrent. If two Lambda instances start simultaneously and both try to run migrations, you get a race condition.
- Lambda has a 15-minute timeout. Long migrations could be interrupted.
- You don't want application traffic hitting an in-progress migration.

**The right approach: migration Lambda triggered by deploy**

Create a dedicated migration function that runs outside the request path:

```typescript
// src/functions/migrate.ts
import { getPrismaClient } from "../../lib/prisma";
import { execSync } from "child_process";
import path from "path";

export const handler = async () => {
  console.log("Running database migrations...");

  try {
    execSync("npx prisma migrate deploy", {
      env: {
        ...process.env,
        DATABASE_URL: buildConnectionString(),
      },
      cwd: path.resolve(process.cwd()),
      stdio: "inherit",
    });

    console.log("Migrations complete");
    return { success: true };
  } catch (err) {
    console.error("Migration failed:", err);
    throw err;
  }
};

function buildConnectionString() {
  const { Resource } = require("sst");
  return (
    `postgresql://${Resource.RunwayDb.username}:${Resource.RunwayDb.password}` +
    `@${Resource.RunwayDb.host}:${Resource.RunwayDb.port}/${Resource.RunwayDb.database}`
  );
}
```

Wire it in `sst.config.ts` as a one-off function (not a route):

```typescript
const migrationRunner = new sst.aws.Function("MigrationRunner", {
  handler: "src/functions/migrate.handler",
  link: [db],
  vpc,
  timeout: "5 minutes",
  nodejs: {
    install: ["prisma", "@prisma/client"],
  },
});
```

In your CI pipeline (Chapter 9), invoke the migration function before deploying the application Lambdas:

```bash
# Run migrations before deploying
aws lambda invoke \
  --function-name runway-production-MigrationRunner \
  --payload '{}' \
  /tmp/migration-output.json

cat /tmp/migration-output.json  # Check for errors

# Then deploy everything else
npx sst deploy --stage production
```

Migrations run first, in isolation, before any application code deploys. If a migration fails, the deploy stops before broken code reaches production.

### Zero-downtime migrations: the expand/contract pattern

Adding a non-nullable column is the classic migration danger. If you add `invoiceNumber NOT NULL` to `InvoicePayment` and your application code tries to insert a row without `invoiceNumber` before the deploy completes, you get a constraint violation.

The expand/contract pattern prevents this:

**Phase 1 (expand): Add the column as nullable**
```sql
ALTER TABLE invoice_payments ADD COLUMN invoice_number VARCHAR(50);
```
Old code continues working (it doesn't know about `invoice_number`). New code can start writing to it.

**Phase 2: Deploy new code that writes the column**
Both old and new code work.

**Phase 3 (contract): Make it non-nullable**
```sql
UPDATE invoice_payments SET invoice_number = 'LEGACY-' || id WHERE invoice_number IS NULL;
ALTER TABLE invoice_payments ALTER COLUMN invoice_number SET NOT NULL;
```
Now safe — all rows have the value.

Three deploys instead of one. Worth it for columns where existing rows need backfilling. For new tables or nullable columns, a single migration is fine.

---

## 5.6 Lambda Event Sourcing from RDS

This is where Runway's two databases talk to each other.

When an invoice is marked as paid in DynamoDB (operational), we need to record that payment in Aurora (analytical). There are three patterns for this. They're solving different problems.

### Pattern 1: Aurora Native Lambda Invocation

Aurora MySQL and PostgreSQL can invoke Lambda functions directly from a stored procedure or trigger. A database write can call a Lambda.

```sql
-- PostgreSQL trigger that calls Lambda when a row is inserted
CREATE OR REPLACE FUNCTION notify_payment_lambda()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM aws_lambda.invoke(
    aws_commons.create_lambda_function_arn(
      'runway-production-PaymentProcessor',
      'eu-west-1'
    ),
    json_build_object(
      'invoiceId', NEW.invoice_id,
      'workspaceId', NEW.workspace_id,
      'amountCents', NEW.amount_cents,
      'paidAt', NEW.paid_at
    )::json,
    'Event'  -- Async invocation
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER after_payment_insert
  AFTER INSERT ON invoice_payments
  FOR EACH ROW EXECUTE FUNCTION notify_payment_lambda();
```

**Use it when:** You want Aurora to be the event source — something happens in the database and you want to trigger downstream processing. Cache invalidation, audit logging, webhook dispatch.

**Watch out for:** The Lambda invocation is fire-and-forget (`'Event'` async mode). If the Lambda fails, you don't know from the database trigger. For Runway, we're not using this pattern (we're writing *to* Aurora, not firing from it), but it's useful in the other direction — if you had Aurora as a primary database and wanted to notify Lambda when data changes.

### Pattern 2: RDS Event Subscriptions (Database Lifecycle Events)

RDS event subscriptions publish database lifecycle events to SNS — things like failovers, parameter group changes, backup completions, and storage alarms.

SST doesn't have a first-class component for RDS event subscriptions yet, so use a raw `aws.rds.EventSubscription` Pulumi resource alongside the SST resources.

```typescript
// sst.config.ts
const dbEventTopic = new sst.aws.SnsTopic("DbEvents");

const dbEventHandler = new sst.aws.Function("DbEventHandler", {
  handler: "src/functions/db-events.handler",
});
dbEventTopic.subscribe(dbEventHandler.arn);

const eventSubscription = new aws.rds.EventSubscription("DbEventSubscription", {
  snsTopic: dbEventTopic.arn,
  sourceType: "db-cluster",
  sourceIds: [db.clusterIdentifier],
  eventCategories: ["failover", "maintenance", "notification"],
});
```

```typescript
// src/functions/db-events.ts
import type { SNSHandler } from "aws-lambda";

export const handler: SNSHandler = async (event) => {
  for (const record of event.Records) {
    const message = JSON.parse(record.Sns.Message);

    if (message.EventID === "RDS-EVENT-0049") {
      await sendSlackAlert(`Aurora failover in progress: ${message.SourceIdentifier}`);
    }

    if (message.EventID === "RDS-EVENT-0051") {
      await sendSlackAlert(`Aurora failover complete — connections restored`);
    }
  }
};
```

**Use it when:** You need to react to database infrastructure events — failovers, maintenance windows, storage threshold alerts. This is operational monitoring, not application logic.

**Not for:** Row-level change notifications. RDS event subscriptions don't fire when your application writes data — only when the database engine itself has something to report.

### Pattern 3: Change Data Capture with DMS

AWS Database Migration Service (DMS) can stream row-level changes from a database as they happen. Changes go to Kinesis Data Streams, and Lambda consumes them.

```
Aurora (source) → DMS Replication Task → Kinesis Data Stream → Lambda
```

This is the pattern for streaming every INSERT, UPDATE, DELETE from Aurora to downstream consumers.

```typescript
const cdcStream = new sst.aws.KinesisStream("AuroraCdcStream");

const cdcProcessor = new sst.aws.Function("CdcProcessor", {
  handler: "src/functions/cdc.handler",
});

cdcStream.subscribe(cdcProcessor.arn);
```

```typescript
// src/functions/cdc.ts
import type { KinesisStreamHandler } from "aws-lambda";

interface CdcEvent {
  metadata: { operation: "INSERT" | "UPDATE" | "DELETE"; "table-name": string };
  data: Record<string, unknown>;
}

export const handler: KinesisStreamHandler = async (event) => {
  for (const record of event.Records) {
    const payload = Buffer.from(record.kinesis.data, "base64").toString();
    const cdcEvent: CdcEvent = JSON.parse(payload);

    if (
      cdcEvent.metadata.operation === "INSERT" &&
      cdcEvent.metadata["table-name"] === "invoice_payments"
    ) {
      await handleNewPayment(cdcEvent.data);
    }
  }
};
```

**Use it when:** You need a full audit log of every database change, you're building event-driven architecture where Aurora is a source of truth, or you're replicating data to another system.

**Cost note:** DMS has an hourly cost (~$0.18/hour for a small replication instance). For Runway's scale, this is overkill. You'd choose CDC for large systems where the database is a hub.

### What Runway actually uses

For Runway, the flow is simple and deliberate:

1. Invoice is marked as paid via the DynamoDB API (from Chapter 4)
2. The `updateInvoiceStatus` function calls a `recordPayment` function directly
3. `recordPayment` writes to Aurora

No event bus needed at this stage. Direct function call, easy to reason about, easy to test. The event-driven patterns above are worth knowing for when you have multiple systems that need to react to the same event — Runway grows into them as it scales.

```typescript
// src/repositories/payments.ts
import { getPrismaClient } from "../lib/prisma";

export async function recordPayment(data: {
  workspaceId: string;
  workspaceName: string;
  clientId: string;
  clientName: string;
  projectId: string;
  projectName: string;
  invoiceId: string;
  invoiceNumber: string;
  amountCents: number;
  currency: string;
  paidAt: Date;
}): Promise<void> {
  const prisma = getPrismaClient();

  await prisma.invoicePayment.upsert({
    where: { invoiceId: data.invoiceId },
    create: {
      workspaceId: data.workspaceId,
      clientId: data.clientId,
      clientName: data.clientName,
      projectId: data.projectId,
      projectName: data.projectName,
      invoiceId: data.invoiceId,
      invoiceNumber: data.invoiceNumber,
      amountCents: data.amountCents,
      currency: data.currency,
      paidAt: data.paidAt,
    },
    update: {},
  });

  await prisma.workspace.upsert({
    where: { id: data.workspaceId },
    create: { id: data.workspaceId, name: data.workspaceName },
    update: { name: data.workspaceName },
  });
}
```

The `upsert` makes this idempotent — if `recordPayment` is called twice for the same invoice (retry logic, duplicate events), the second call is a no-op. Always make data writes idempotent in distributed systems.

---

## 5.7 The Reporting Endpoints

With payments in Aurora, the reports are simple SQL:

```typescript
// src/functions/reports/revenue.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { getPrismaClient } from "../../lib/prisma";
import { ok, notFound } from "../../lib/response";
import { createLogger } from "../../lib/logger";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const logger = createLogger(context.awsRequestId);
  const workspaceId = event.pathParameters?.workspaceId!;
  const prisma = getPrismaClient();

  try {
    const monthlyRevenue = await prisma.$queryRaw<
      Array<{ month: Date; total_cents: bigint; invoice_count: bigint }>
    >`
      SELECT
        DATE_TRUNC('month', paid_at) AS month,
        SUM(amount_cents)            AS total_cents,
        COUNT(*)                     AS invoice_count
      FROM invoice_payments
      WHERE workspace_id = ${workspaceId}
        AND paid_at >= NOW() - INTERVAL '12 months'
      GROUP BY DATE_TRUNC('month', paid_at)
      ORDER BY month DESC
    `;

    const revenueByClient = await prisma.invoicePayment.groupBy({
      by: ["clientId", "clientName"],
      where: { workspaceId },
      _sum: { amountCents: true },
      _count: { invoiceId: true },
      orderBy: { _sum: { amountCents: "desc" } },
      take: 10,
    });

    const totals = await prisma.invoicePayment.aggregate({
      where: { workspaceId },
      _sum: { amountCents: true },
      _count: { invoiceId: true },
    });

    return ok({
      monthly: monthlyRevenue.map((row) => ({
        month: row.month.toISOString().slice(0, 7),
        totalCents: Number(row.total_cents),
        invoiceCount: Number(row.invoice_count),
      })),
      byClient: revenueByClient.map((row) => ({
        clientId: row.clientId,
        clientName: row.clientName,
        totalCents: row._sum.amountCents ?? 0,
        invoiceCount: row._count.invoiceId,
      })),
      totals: {
        allTimeCents: totals._sum.amountCents ?? 0,
        allTimeInvoices: totals._count.invoiceId,
      },
    });
  } catch (err) {
    logger.error("Revenue report failed", { err: String(err) });
    throw err;
  }
};
```

One thing to note: `$queryRaw` returns `bigint` for `SUM` and `COUNT` results because PostgreSQL returns `BIGINT` for those. JavaScript's `number` type can't safely represent integers above 2^53. `Number(bigint)` is fine for Runway's invoice counts and totals — you'd need proper BigInt handling if you were summing billions of cents.

---

## 5.8 Read Replicas for Reporting

Aurora Serverless v2 supports read replicas — additional endpoints that handle read traffic, allowing the primary writer to focus on writes.

For Runway's reporting workload, where all queries are reads, a read replica makes sense. SST doesn't have a first-class read replica component yet, so the replica is created as a raw `aws.rds.ClusterInstance` Pulumi resource alongside the SST Aurora cluster.

```typescript
const db = new sst.aws.Aurora("RunwayDb", {
  engine: "postgres",
  vpc,
  scaling: { min: "0 ACU", max: "4 ACU" },
  proxy: true,
});

const readReplica = new aws.rds.ClusterInstance("RunwayDbReader", {
  clusterIdentifier: db.clusterIdentifier,
  instanceClass: "db.serverless",
  engine: "aurora-postgresql",
  engineVersion: db.engineVersion,
  promotionTier: 1,
});
```

Use separate connection strings for reads and writes:

The write client targets the primary endpoint (`RunwayDb.host`); the read client targets the reader endpoint (`RunwayDb.readerEndpoint`).

```typescript
// src/lib/prisma.ts
export function getWriteClient(): PrismaClient {
  return createClient(Resource.RunwayDb.host);
}

export function getReadClient(): PrismaClient {
  return createClient(Resource.RunwayDb.readerEndpoint);
}
```

Report functions use the read client. The `recordPayment` write function uses the write client. The reader can handle bursts of dashboard load without affecting invoice creation.

---

## Where We Are

Runway now has a two-database architecture that's right for the job:

```
[DynamoDB]                    [Aurora Serverless]
  ├── Workspaces               ├── invoice_payments
  ├── Clients                  └── workspaces
  ├── Projects
  └── Invoices

[API Gateway + Lambda]
  ├── /workspaces/**         → DynamoDB
  ├── /clients/**            → DynamoDB
  ├── /projects/**           → DynamoDB
  ├── /invoices/**           → DynamoDB (+ records to Aurora on payment)
  └── /reports/**            → Aurora (read-only)
```

Operational reads are sub-millisecond from DynamoDB. Analytical queries use SQL in Aurora where they belong. The two systems stay in sync via the `recordPayment` function, which writes to both atomically.

Chapter 6 adds S3 — file storage for Runway's deliverables and generated invoice PDFs.

---

> **The code for this chapter** is available at `05-rds-aurora-serverless/` in the companion repository.
