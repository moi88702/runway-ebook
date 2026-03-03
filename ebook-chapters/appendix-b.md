# Appendix B: Common Errors Reference

The errors you'll hit in production, what they actually mean, and how to fix them. Ordered by how often you'll encounter them.

---

## DynamoDB Errors

### `ConditionalCheckFailedException`

**What it means:** A `PutCommand`, `UpdateCommand`, or `DeleteCommand` had a `ConditionExpression` that evaluated to false. The operation was rejected.

**When you'll see it:**
- You tried to create an item that already exists (using `attribute_not_exists(PK)`)
- You tried to update an item with an optimistic lock version that doesn't match (`version = :expected`)
- You tried to delete an item that doesn't exist (using `attribute_exists(PK)`)

**This is not always an error.** In Chapter 4, `ConditionalCheckFailedException` is used deliberately for idempotency checks and optimistic locking. Catch it and handle it:

```typescript
try {
  await docClient.send(new PutCommand({
    TableName: tableName,
    Item: { PK: `WORKSPACE#${id}`, SK: "WORKSPACE", ... },
    ConditionExpression: "attribute_not_exists(PK)",
  }));
} catch (err: any) {
  if (err.name === "ConditionalCheckFailedException") {
    throw new ConflictError("Workspace already exists");
  }
  throw err;
}
```

**In a `TransactWriteCommand`:** the error is wrapped as a `TransactionCanceledException` with a `CancellationReasons` array. Check the reason at the index of the failing item:

```typescript
} catch (err: any) {
  if (err.name === "TransactionCanceledException") {
    const reasons = err.CancellationReasons ?? [];
    const versionConflict = reasons.find(r => r.Code === "ConditionalCheckFailed");
    if (versionConflict) throw new OptimisticLockError("Version mismatch");
  }
  throw err;
}
```

---

### `ProvisionedThroughputExceededException`

**What it means:** Your table or GSI has provisioned read or write capacity units, and you've exceeded the allocated throughput.

**When you'll see it:** Almost exclusively when using provisioned billing mode. If you're on on-demand billing (the SST default), DynamoDB scales automatically and this error is rare.

**Fix:**
1. Switch to on-demand billing: `billingMode: "PAY_PER_REQUEST"` in SST. Done. On-demand costs more per operation but removes capacity management entirely.
2. If you need provisioned: increase the RCU/WCU allocation, or enable auto-scaling with a target utilisation percentage.
3. Check for hot partitions: if all writes are going to the same partition key, no amount of provisioned capacity fixes it. Redesign the access pattern.

**Exponential backoff** handles transient throttling gracefully. The AWS SDK v3 retries throttling errors automatically with jitter. Don't disable retries.

---

### `ResourceNotFoundException`

**What it means:** The table or index you're querying doesn't exist.

**When you'll see it:**
- You're connected to the wrong AWS account or region
- The table hasn't been deployed yet (sst deploy hasn't run)
- You hard-coded a table name instead of using `Resource.RunwayTable.name`

**Fix:** Always use `Resource.RunwayTable.name` from the SST `Resource` helper. Never hard-code table names. This ensures you're always referencing the correct table for the current stage.

---

### `ValidationException`

**What it means:** The DynamoDB request itself is malformed — wrong data types, missing required fields, attribute name conflicts with reserved words, etc.

**Common causes:**
- Attribute name is a DynamoDB reserved word (e.g., `name`, `status`, `type`, `data`). Use `ExpressionAttributeNames` aliases:
  ```typescript
  UpdateCommand({
    UpdateExpression: "SET #status = :status",
    ExpressionAttributeNames: { "#status": "status" },
    ExpressionAttributeValues: { ":status": "active" },
  })
  ```
- Mixing `PK` types — you wrote a string to `PK` in one item and a number in another. DynamoDB is schema-less but type-consistent per key.
- `FilterExpression` referencing a key attribute (use `KeyConditionExpression` for key attributes, `FilterExpression` only for non-key).

---

### `ItemCollectionSizeLimitExceededException`

**What it means:** A single partition key has exceeded 10GB of data across the table and all its local secondary indexes.

**When you'll see it:** Rare in most applications. If you're storing large binary data in DynamoDB (don't), or your single-table design has one partition key that accumulates unbounded data, you'll eventually hit this.

**Fix:** Ensure unbounded data uses time-based partition keys (e.g., `LOGS#2026-02#<workspaceId>`) rather than accumulating under a single PK. Use TTL to expire old items. Store blobs in S3, not DynamoDB.

---

## Aurora / RDS Errors

### `too many connections`

**What it means:** Aurora's maximum connection count has been reached. Lambda functions each open their own connection on cold start. Under load, hundreds of concurrent Lambdas can exhaust the database connection pool.

**Connection limits by instance size:**
- Aurora Serverless v2 at 0.5 ACU: ~90 connections
- At 2 ACU: ~405 connections
- At 8 ACU: ~1,620 connections

**Fixes, in order of preference:**
1. **RDS Proxy:** A connection pooler managed by AWS, deployed in front of Aurora. Lambda functions connect to the Proxy, which maintains a stable pool to Aurora. Adds ~1ms latency. Costs ~$0.015/hour per endpoint. Required at scale.
2. **Connection pooling in Lambda:** Use `pg-pool` (PostgreSQL) or Prisma's connection pool with a small `connection_limit` in the connection string: `postgresql://...?connection_limit=2`. Each Lambda instance holds at most 2 connections.
3. **Keep connections alive:** Reuse the Prisma client across warm invocations (instantiate it outside the handler, not inside).

❌ Don't instantiate the client inside the handler — it opens a new connection on every cold start and every warm invocation:

```typescript
export const handler = async () => {
  const prisma = new PrismaClient();
  const invoices = await prisma.invoice.findMany();
  await prisma.$disconnect();
};
```

✅ Instantiate at module level so the client is reused across warm invocations. Don't call `$disconnect()` — keep the connection alive:

```typescript
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient({
  datasources: {
    db: {
      url: `${process.env.DATABASE_URL}?connection_limit=2&pool_timeout=10`,
    },
  },
});

export const handler = async () => {
  const invoices = await prisma.invoice.findMany();
};
```

---

### `Can't reach database server at ...`

**What it means:** The Lambda can't establish a TCP connection to Aurora. This is almost always a VPC/security group issue.

**Checklist:**
- [ ] Lambda is deployed into the same VPC as Aurora
- [ ] Lambda's security group is allowed in Aurora's security group inbound rules (port 5432 for PostgreSQL, port 3306 for MySQL)
- [ ] Aurora security group inbound rule references Lambda's security group ID, not a CIDR range
- [ ] Lambda subnet has a route to the database subnet (they're in the same VPC — this is usually fine)
- [ ] If using a private subnet with no NAT: Lambda can still reach Aurora (same VPC) but cannot reach the internet. Add a VPC endpoint for SSM if you're reading secrets from Parameter Store.

```typescript
const vpc = new sst.aws.Vpc("RunwayVpc");

const db = new sst.aws.Aurora("RunwayDb", {
  engine: "postgresql15.4",
  vpc,
});

const invoiceHandler = new sst.aws.Function("InvoiceHandler", {
  handler: "src/handlers/api/invoices.handler",
  vpc,   // same VPC as Aurora
  link: [db],
});
```

---

### `Prisma Client is not configured to run in this environment`

**What it means:** Prisma generated a query engine binary for the wrong architecture or operating system. Lambda runs on `linux/amd64` (or `linux/arm64` for Graviton). If you ran `prisma generate` on macOS, the binary may be wrong.

**Fix:** Configure Prisma to generate the correct binary for Lambda:

```prisma
// schema.prisma
generator client {
  provider      = "prisma-client-js"
  binaryTargets = ["native", "rhel-openssl-3.0.x"]
}
```

`native` generates the binary for your local machine (for dev). `rhel-openssl-3.0.x` is the correct target for Lambda (Amazon Linux 2023).

Run `prisma generate` after updating `schema.prisma`. The generated client in `node_modules/.prisma/client/` will include both binaries. SST bundles the correct one for Lambda.

---

### `P2002: Unique constraint failed`

**What it means:** A Prisma write violated a unique constraint in the database schema.

**Handle it explicitly — don't surface it as a 500:**

```typescript
import { Prisma } from "@prisma/client";

try {
  await prisma.invoice.create({ data: { ... } });
} catch (err) {
  if (err instanceof Prisma.PrismaClientKnownRequestError && err.code === "P2002") {
    const field = (err.meta?.target as string[])?.join(", ");
    throw new ConflictError(`Duplicate value for: ${field}`);
  }
  throw err;
}
```

Common Prisma error codes:
- `P2002` — unique constraint violation
- `P2025` — record not found (on update/delete)
- `P2003` — foreign key constraint violation
- `P2034` — transaction conflict (serialisation failure under concurrent writes — retry)

---

## IAM / Auth Errors

### `AccessDeniedException`

**What it means:** The IAM role or user making the call doesn't have permission for the operation.

**The message tells you exactly what's missing:**

```
User: arn:aws:sts::123456789012:assumed-role/InvoiceHandler/... is not authorized
to perform: dynamodb:Query on resource: arn:aws:dynamodb:eu-west-1:...:table/RunwayTable
```

**Fixes:**
1. If using SST `link:` — verify the resource is linked: `link: [table]`. SST automatically generates the correct IAM policy. If it's not linked, the role has no permissions.
2. If using GSIs — you need permission on the index ARN, not just the table ARN: `arn:...:table/RunwayTable/index/GSI1`. SST handles this when you use `link`.
3. If calling a service not managed by SST (e.g., SES, external API) — add the policy manually via the `permissions` option:
   ```typescript
   const handler = new sst.aws.Function("Handler", {
     handler: "...",
     permissions: [
       { actions: ["ses:SendEmail"], resources: ["*"] },
     ],
   });
   ```

---

### `InvalidSignatureException`

**What it means:** The AWS SDK request signature is invalid. Almost always a clock skew issue — the request timestamp is too far from AWS's current time (>15 minutes).

**When you'll see it:** On Lambda this is rare (Lambda gets NTP sync). More common when testing locally with Docker containers or VMs that have drifted clocks.

**Fix:** Sync the clock. On Linux: `sudo ntpdate pool.ntp.org` or `timedatectl set-ntp true`. On Docker Desktop: restart Docker (it resets the VM clock).

---

### `ExpiredTokenException`

**What it means:** Temporary credentials (from STS, assumed roles, SSO sessions) have expired.

**In Lambda:** The Lambda execution role issues fresh credentials automatically on each invocation. You should never see this in Lambda unless you're caching credentials across invocations (don't).

**In local dev / CI:** Your SSO session or assumed role has expired. Run `aws sso login` or refresh the credentials.

---

## API Gateway Errors

### `502 Bad Gateway`

**What it means:** API Gateway received a response from Lambda that it couldn't parse, or Lambda threw an unhandled exception.

**Common causes:**
- Lambda function timed out (API Gateway timeout is 29 seconds max)
- Lambda returned a response that doesn't match the expected format (missing `statusCode`, wrong body type)
- Lambda had an initialisation error (import failure, environment variable missing at startup)

**Check:** CloudWatch Logs for the Lambda function — the error will be there even if it didn't make it back to the caller.

**Correct Lambda response shape:**

```typescript
return {
  statusCode: 200,
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ data: "..." }),
};
```

---

### `403 Forbidden` from API Gateway (not your Lambda)

**What it means:** API Gateway rejected the request before it reached Lambda. Most commonly:
- JWT authoriser rejected the token (expired, invalid signature, wrong audience)
- Resource policy blocked the request
- CORS preflight failed

**Distinguish from your own 403s** by checking whether the response has your error shape. If it doesn't (`{"message":"Forbidden"}` from API Gateway vs `{"error":"..."}` from your handler), API Gateway itself rejected it.

**JWT authoriser failures:** The token issuer (`iss`) must match your Cognito User Pool endpoint exactly. The audience (`aud`) must match your App Client ID. Common mistake: using the wrong stage's User Pool ID.

---

## SQS Errors

### Messages Stuck in the Queue

**Symptom:** Messages accumulate in the queue, the Lambda never processes them, no DLQ messages.

**Causes:**
- Lambda isn't subscribed to the queue (check SST config)
- Lambda has no permission to receive from the queue (check `link`)
- Lambda is timing out repeatedly — messages return to the queue after visibility timeout expires, but the Lambda keeps failing before the DLQ threshold

**Check:** The `NumberOfMessagesNotVisible` CloudWatch metric — messages that are in-flight (being processed). If this is high and not decreasing, Lambda is picking up messages and failing within the visibility timeout.

---

### `KMSDisabledException` or `KMSAccessDeniedException`

**What it means:** SQS is using server-side encryption with a KMS key, and Lambda's execution role can't use that key.

**Fix:** Add `kms:Decrypt` and `kms:GenerateDataKey` permissions for the key ARN to Lambda's execution role. Or use SSE-SQS (SST default for managed encryption) instead of a customer-managed key.

---

## Lambda Errors

### `Task timed out after X.XX seconds`

**What it means:** The Lambda hit its configured timeout.

**Diagnose:** Check CloudWatch Logs — the last log line before the timeout message reveals where execution was when the clock ran out. Usually an awaited promise that never resolved (hung HTTP request, DB connection timeout, etc.).

**Fixes:**
- Add timeouts to all outbound calls: `{ signal: AbortSignal.timeout(5000) }` for fetch, connection timeout params for DB clients
- Increase the Lambda timeout if the work genuinely takes longer
- Break long operations into smaller Lambdas connected by SQS (Chapter 8)
- Use Step Functions for multi-step jobs with state

---

### `INIT_REPORT` errors / Lambda initialisation failures

**What it means:** Lambda failed during the initialisation phase (before the handler runs). The module failed to import or top-level code threw.

**Common causes:**
- Missing environment variable read at module level: `const key = process.env.API_KEY!` where `API_KEY` isn't set
- Bad import path or missing dependency in the bundle
- Prisma client unable to find the query engine binary

**Check:** The log line before `INIT_REPORT` shows the actual error. Fix the root cause — increasing memory or timeout doesn't help here.

---

## General Debugging Checklist

When something is broken and you don't know what:

1. **Find the right log group.** CloudWatch → Log groups → `/aws/lambda/<FunctionName>`. Sort by "Last event time."

2. **Find the failing request.** Filter logs for `"ERROR"` or the request ID from the failed response.

3. **Read the full error, including the stack trace.** The first line of the stack trace usually tells you exactly where it broke.

4. **Check IAM.** `AccessDeniedException` is the cause of about 30% of mysterious Lambda failures. Check the error message for the exact permission that's missing.

5. **Check environment variables.** Lambda's configuration → Environment variables. Make sure the right secrets and config are there for the right stage.

6. **Check the region.** If you're looking at the wrong region in the console, everything appears to not exist.

7. **Enable X-Ray tracing.** If you can reproduce the issue, a trace will show you exactly which service call failed and how long each step took.
