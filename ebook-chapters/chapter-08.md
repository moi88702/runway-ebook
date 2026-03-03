# Chapter 8: Queues, Events, and Background Jobs

Runway's API is fast, authenticated, and typed end-to-end. But there's a class of work it shouldn't do in the API request itself.

When a freelancer marks an invoice as paid, three things need to happen: the invoice status updates in the database, a receipt email goes to the client, and Runway's internal analytics get updated. You could do all three synchronously in the handler. You probably shouldn't. The email provider might be slow. The analytics write might fail. And now a simple invoice update takes 600ms and fails with a 500 when Mailgun has a bad moment.

The fix is async. Hand off work that doesn't need to happen before you return 200. Let queues absorb traffic spikes. Let events decouple parts of your system so they can fail independently. Let cron handle the work nobody requested but everyone expects to happen.

This chapter covers the async layer of Runway: SQS for reliable point-to-point messaging, EventBridge for pub/sub event routing, cron for scheduled jobs, and patterns for the long-running work that doesn't fit in 15 minutes.

---

## 8.1 Why Async Matters

The API contract says: receive a request, do the work, return a response. It doesn't say the work has to complete before you respond.

Most of what happens after a user action doesn't need to be in the response path:

- Sending an email when an invoice is paid
- Generating a PDF when a report is requested
- Updating search indexes when data changes
- Sending a Slack notification when a client comment arrives
- Chasing unpaid invoices every morning at 9am

All of these can happen after you return 200. And because they happen later, a few things become true:

**Your API stays fast.** You're not blocked on Mailgun, Slack, or a PDF generator. The user gets their response in under 100ms.

**Your system is more resilient.** If the email provider is down, the invoice still got saved. The email will retry. The user never knew anything was wrong.

**You can absorb traffic spikes.** If 1000 invoices get paid simultaneously, the payment handlers return instantly and queue up 1000 emails. The email worker processes them at its own pace. No timeouts, no cascading failures.

**You get decoupling.** The invoice service doesn't need to know about the email service. It just emits an event. What handles it is someone else's problem.

The trade-offs are real. Async systems are harder to debug. Failures are silent by default. Work can arrive out of order. You need dead-letter queues and retries and idempotency. This chapter covers all of that.

---

## 8.2 SQS in Depth

SQS (Simple Queue Service) is point-to-point messaging. One producer puts a message on a queue. One consumer reads it. Simple, reliable, and the right tool for most async work in a Lambda stack.

### Setting Up a Queue in SST

```typescript
// sst.config.ts
const emailQueue = new sst.aws.Queue("EmailQueue", {
  visibilityTimeout: "120 seconds",
  dlq: {
    queue: "EmailDlq",
    retry: 3,
  },
});

const api = new sst.aws.ApiGatewayV2("Api");

const emailWorker = new sst.aws.Function("EmailWorker", {
  handler: "src/workers/email.handler",
  link: [emailQueue, /* ses, secrets, etc. */],
  timeout: "120 seconds",
});

emailWorker.subscribe(emailQueue);
```

A few things to note:

- `visibilityTimeout` is how long SQS hides a message from other consumers while your Lambda is processing it. Set it to match your Lambda timeout — if Lambda takes 90 seconds and visibility is 30, SQS will re-queue the message while you're still working on it, and you'll process it twice.
- `dlq` wires up a dead-letter queue automatically. After 3 failed retries, messages go to `EmailDlq` instead of disappearing.
- `link: [emailQueue]` on the API function means it can send to the queue without hand-crafting IAM policies.

### Typed Message Bodies

The number one mistake with SQS: treating message bodies as strings. You lose every bit of TypeScript safety the moment you do `JSON.parse(record.body)` without a schema.

Define your message types explicitly and validate them on arrival:

```typescript
// src/types/queue-messages.ts
import { z } from "zod";

export const InvoicePaidEmailMessage = z.object({
  type: z.literal("invoice.paid.email"),
  invoiceId: z.string(),
  workspaceId: z.string(),
  clientEmail: z.string().email(),
  clientName: z.string(),
  invoiceNumber: z.string(),
  amountCents: z.number().int().positive(),
  currency: z.string().default("GBP"),
  paidAt: z.string().datetime(),
});

export const InvoiceReminderEmailMessage = z.object({
  type: z.literal("invoice.reminder.email"),
  invoiceId: z.string(),
  workspaceId: z.string(),
  clientEmail: z.string().email(),
  clientName: z.string(),
  invoiceNumber: z.string(),
  amountCents: z.number().int().positive(),
  dueDateIso: z.string(),
  daysOverdue: z.number().int(),
});

export const QueueMessage = z.discriminatedUnion("type", [
  InvoicePaidEmailMessage,
  InvoiceReminderEmailMessage,
]);

export type QueueMessage = z.infer<typeof QueueMessage>;
export type InvoicePaidEmailMessage = z.infer<typeof InvoicePaidEmailMessage>;
export type InvoiceReminderEmailMessage = z.infer<typeof InvoiceReminderEmailMessage>;
```

Sending a message from the API:

```typescript
// src/lib/queue.ts
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";
import type { QueueMessage } from "../types/queue-messages";
import { Resource } from "sst";

const client = new SQSClient({});

export async function sendToEmailQueue(message: QueueMessage): Promise<void> {
  await client.send(
    new SendMessageCommand({
      QueueUrl: Resource.EmailQueue.url,
      MessageBody: JSON.stringify(message),
    })
  );
}
```

```typescript
await sendToEmailQueue({
  type: "invoice.paid.email",
  invoiceId: invoice.invoiceId,
  workspaceId: invoice.workspaceId,
  clientEmail: client.email,
  clientName: client.name,
  invoiceNumber: invoice.invoiceNumber,
  amountCents: invoice.amountCents,
  currency: invoice.currency,
  paidAt: new Date().toISOString(),
});
```

The handler returns 200. The email sends in the background. If the email fails, SQS retries. If it fails 3 times, it lands in the DLQ where you can inspect it.

### The Worker

```typescript
// src/workers/email.ts
import type { SQSHandler, SQSRecord } from "aws-lambda";
import { QueueMessage } from "../types/queue-messages";
import { sendInvoicePaidEmail, sendInvoiceReminderEmail } from "../lib/email";

export const handler: SQSHandler = async (event) => {
  const results = await Promise.allSettled(
    event.Records.map(processRecord)
  );

  const batchItemFailures = results
    .map((result, index) => ({ result, record: event.Records[index] }))
    .filter(({ result }) => result.status === "rejected")
    .map(({ record }) => ({ itemIdentifier: record.messageId }));

  return { batchItemFailures };
};

async function processRecord(record: SQSRecord): Promise<void> {
  let message: QueueMessage;

  try {
    message = QueueMessage.parse(JSON.parse(record.body));
  } catch (err) {
    console.error("Invalid message body", { body: record.body, err });
    return;
  }

  switch (message.type) {
    case "invoice.paid.email":
      await sendInvoicePaidEmail(message);
      break;
    case "invoice.reminder.email":
      await sendInvoiceReminderEmail(message);
      break;
    default:
      const _exhaustive: never = message;
  }
}
```

Two things worth unpacking here:

**`batchItemFailures`** is the partial batch failure pattern. Without it, if a batch of 10 messages arrives and record 7 fails, SQS retries all 10. Record 7 might be genuinely broken while records 1–6 and 8–10 processed fine. Returning `batchItemFailures` tells SQS exactly which records to retry. Everything else is acknowledged and removed from the queue.

To enable partial batch failure in SST:

```typescript
emailWorker.subscribe(emailQueue, {
  filters: [],
  batchSize: 10,
  reportBatchItemFailures: true,
});
```

**The malformed message case** deserves thought. If a message can't be parsed, throwing will cause SQS to retry it forever (until the DLQ catches it). But if the message is structurally broken, retrying won't fix it. Return without throwing to let it pass through; or throw intentionally to park it in the DLQ for human review. Choose based on whether you want it visible in the DLQ.

### Visibility Timeout and Retry Behaviour

When a Lambda picks up an SQS message, SQS sets a visibility timer. During that window, no other consumer sees the message. If the Lambda finishes successfully, SQS deletes the message. If the Lambda crashes or times out, SQS makes the message visible again after the visibility window expires, and another Lambda picks it up.

This means your SQS workers must be **idempotent** — processing the same message twice must produce the same result as processing it once. For email, that means deduplication: track which `invoiceId` has already had a paid email sent, and skip if it has.

```typescript
// src/workers/email.ts — inside sendInvoicePaidEmail
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

async function markEmailSent(invoiceId: string, type: string): Promise<boolean> {
  try {
    await docClient.send(
      new PutCommand({
        TableName: Resource.RunwayTable.name,
        Item: {
          PK: `INVOICE#${invoiceId}`,
          SK: `EMAIL_SENT#${type}`,
          sentAt: new Date().toISOString(),
        },
        ConditionExpression: "attribute_not_exists(PK)",
      })
    );
    return true;
  } catch (err: any) {
    if (err.name === "ConditionalCheckFailedException") {
      return false;
    }
    throw err;
  }
}

export async function sendInvoicePaidEmail(message: InvoicePaidEmailMessage): Promise<void> {
  const shouldSend = await markEmailSent(message.invoiceId, "paid");
  if (!shouldSend) {
    console.log("Skipping duplicate email", { invoiceId: message.invoiceId });
    return;
  }
  // ... actual email send
}
```

This is the DynamoDB `ConditionExpression` from Chapter 4 doing real work. The first call succeeds, writes the record, and sends the email. Any duplicate call hits the condition, returns false, and skips cleanly.

### Dead-Letter Queues

Messages land in the DLQ when they've failed `maxReceiveCount` times (3 in the config above). At that point, something is genuinely wrong: the message is malformed, the downstream service is broken, or there's a bug in your worker.

Set up a CloudWatch alarm on the DLQ so you know when it gets messages:

```typescript
// sst.config.ts
import * as cloudwatch from "aws-cdk-lib/aws-cloudwatch";
import * as cloudwatchActions from "aws-cdk-lib/aws-cloudwatch-actions";
import * as sns from "aws-cdk-lib/aws-sns";
import { Stack } from "aws-cdk-lib";

const emailDlq = emailQueue.nodes.deadLetterQueue;
if (emailDlq) {
  const stack = Stack.of(emailDlq);

  const alarm = new cloudwatch.Alarm(stack, "EmailDlqAlarm", {
    metric: emailDlq.metricApproximateNumberOfMessagesVisible(),
    threshold: 1,
    evaluationPeriods: 1,
    comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
    alarmName: "EmailDlqHasMessages",
  });
}
```

In practice for Runway: DLQ messages are a support ticket in waiting. A failed email might mean a client never got their receipt. Check the DLQ regularly and have a reprocessing runbook.

### FIFO Queues: When Ordering Matters

Standard SQS queues are "at-least-once, best-effort ordering" — they're fast and scalable, but messages can arrive out of order or occasionally twice. For most async work in Runway, this is fine.

FIFO queues (First-In, First-Out) guarantee ordering within a message group and exactly-once processing. They're capped at 3,000 messages/second (300 without batching) and cost slightly more. Use them when:

- Processing events for a specific entity in order matters (e.g., invoice status transitions: draft → sent → paid — you don't want "paid" processed before "sent")
- You genuinely cannot tolerate duplicate processing even with idempotency guards

For Runway's email queue, standard is fine — an email arriving twice is annoying, but the idempotency check handles it. For a payment event queue tied to Stripe webhooks, FIFO is worth considering.

FIFO queues require a `MessageGroupId` (messages within a group are delivered in order) and a `MessageDeduplicationId` (prevents exactly-once violations within a 5-minute window).

```typescript
const paymentEventQueue = new sst.aws.Queue("PaymentEventQueue", {
  fifo: true,
  visibilityTimeout: "60 seconds",
});

await client.send(
  new SendMessageCommand({
    QueueUrl: Resource.PaymentEventQueue.url,
    MessageBody: JSON.stringify(event),
    MessageGroupId: invoice.invoiceId,
    MessageDeduplicationId: `${invoice.invoiceId}-${event.type}-${event.timestamp}`,
  })
);
```

---

## 8.3 EventBridge for Event-Driven Architecture

SQS is point-to-point: one sender, one receiver. EventBridge is pub/sub: one sender, potentially many receivers, with routing in the middle.

When an invoice is marked paid in Runway, the SQS approach means the invoice handler explicitly knows about the email queue and sends to it. What if you also want to:

- Update your analytics data warehouse
- Trigger a congratulations notification to the freelancer
- Sync the payment to Xero

With SQS you'd add three more `sendToQueue()` calls to the handler. The invoice service now knows about analytics, notifications, and Xero. Tight coupling.

With EventBridge, the invoice handler emits one event: `invoice.paid`. Three consumers subscribe to it and do their thing. The invoice handler has no idea any of them exist.

### Setting Up EventBridge in SST

```typescript
// sst.config.ts
const eventBus = new sst.aws.Bus("RunwayBus");

const analyticsWorker = new sst.aws.Function("AnalyticsWorker", {
  handler: "src/workers/analytics.handler",
  link: [/* dynamodb, etc. */],
});

const notificationWorker = new sst.aws.Function("NotificationWorker", {
  handler: "src/workers/notification.handler",
  link: [/* push notification service, etc. */],
});

eventBus.subscribe("invoice.paid", analyticsWorker, {
  pattern: {
    source: ["runway.invoices"],
    detailType: ["invoice.paid"],
  },
});

eventBus.subscribe("invoice.paid.notify", notificationWorker, {
  pattern: {
    source: ["runway.invoices"],
    detailType: ["invoice.paid"],
  },
});
```

Both workers subscribe to the same event. EventBridge fans it out to both.

### Defining Events as TypeScript Types

EventBridge events have a fixed envelope:

```json
{
  "source": "runway.invoices",
  "detail-type": "invoice.paid",
  "detail": { ... your payload ... }
}
```

Define your event catalog in TypeScript:

```typescript
// src/types/events.ts
import { z } from "zod";

export const RunwayEventEnvelope = z.object({
  source: z.string(),
  "detail-type": z.string(),
  detail: z.record(z.unknown()),
});

export const InvoicePaidEvent = z.object({
  invoiceId: z.string(),
  workspaceId: z.string(),
  ownerId: z.string(),
  clientId: z.string(),
  invoiceNumber: z.string(),
  amountCents: z.number().int().positive(),
  currency: z.string(),
  paidAt: z.string().datetime(),
});

export type InvoicePaidEvent = z.infer<typeof InvoicePaidEvent>;

export const RunwayEvents = {
  "invoice.paid": InvoicePaidEvent,
  "invoice.sent": z.object({
    invoiceId: z.string(),
    workspaceId: z.string(),
    clientEmail: z.string(),
    sentAt: z.string().datetime(),
  }),
  "client.created": z.object({
    clientId: z.string(),
    workspaceId: z.string(),
    email: z.string(),
    createdAt: z.string().datetime(),
  }),
} as const;

export type RunwayEventType = keyof typeof RunwayEvents;
```

Emitting an event from the API:

```typescript
// src/lib/events.ts
import { EventBridgeClient, PutEventsCommand } from "@aws-sdk/client-eventbridge";
import { Resource } from "sst";
import { RunwayEventType, RunwayEvents } from "../types/events";
import { z } from "zod";

const client = new EventBridgeClient({});

export async function emitEvent<T extends RunwayEventType>(
  type: T,
  detail: z.infer<typeof RunwayEvents[T]>
): Promise<void> {
  await client.send(
    new PutEventsCommand({
      Entries: [
        {
          EventBusName: Resource.RunwayBus.name,
          Source: "runway.invoices",
          DetailType: type,
          Detail: JSON.stringify(detail),
        },
      ],
    })
  );
}
```

```typescript
await emitEvent("invoice.paid", {
  invoiceId: invoice.invoiceId,
  workspaceId: invoice.workspaceId,
  ownerId: invoice.ownerId,
  clientId: invoice.clientId,
  invoiceNumber: invoice.invoiceNumber,
  amountCents: invoice.amountCents,
  currency: invoice.currency,
  paidAt: new Date().toISOString(),
});
```

The function is typed: pass the wrong shape for `invoice.paid` and TypeScript tells you before it gets anywhere near AWS.

### Receiving an Event

```typescript
// src/workers/analytics.ts
import type { EventBridgeHandler } from "aws-lambda";
import { InvoicePaidEvent } from "../types/events";

export const handler: EventBridgeHandler<"invoice.paid", unknown, void> = async (event) => {
  const detail = InvoicePaidEvent.parse(event.detail);

  await recordPaymentForAnalytics({
    workspaceId: detail.workspaceId,
    amountCents: detail.amountCents,
    currency: detail.currency,
    paidAt: detail.paidAt,
  });
};
```

Validate the `event.detail` against your schema on arrival. EventBridge doesn't enforce schemas at runtime (Schema Registry exists but adds complexity) — Zod parse is your safety net.

### Event Filtering

EventBridge routes events using content-based filtering rules. Rather than routing everything to every consumer and letting Lambda decide, push the filtering into EventBridge:

```typescript
eventBus.subscribe("analytics.paid.gbp", analyticsWorker, {
  pattern: {
    source: ["runway.invoices"],
    detailType: ["invoice.paid"],
    detail: {
      currency: ["GBP"],
    },
  },
});
```

You can filter on string values, prefixes, numeric ranges, or even presence/absence of fields. This keeps Lambda invocations down — you're not spinning up a function for events you don't care about.

### SQS vs EventBridge: The Decision

| | SQS | EventBridge |
|---|---|---|
| Delivery | Point-to-point | Fan-out (many consumers) |
| Retry | Built-in (visibility + DLQ) | Configurable retry policy |
| Throughput | Effectively unlimited | 10,000 events/sec (default, raiseable) |
| Cost | ~$0.40/million messages | ~$1.00/million events |
| Ordering | Best-effort (FIFO option) | Best-effort |
| Filtering | None (filter in Lambda) | Content-based filtering rules |
| Use when | One consumer, reliable delivery | Multiple consumers, routing needed |

For Runway's email sending: SQS. One producer (API), one consumer (email worker), reliable retry matters more than fan-out.

For Runway's domain events (invoice paid, client created, project updated): EventBridge. Multiple consumers will eventually want these events. Emitting once and routing is cleaner than adding queue sends to every handler.

In practice, you often use both: EventBridge for fan-out with an SQS queue as the target, so each consumer gets reliable delivery with its own retry and DLQ behaviour.

### Event Replay and Archiving

EventBridge has a built-in archive and replay feature. Turn it on and every event is stored for up to 90 days (or indefinitely at a small cost). When you:

- Deploy a new consumer that needs historical events
- Have a bug in a consumer and need to reprocess the last 24 hours
- Want to backfill analytics after adding a new metric

...you can replay archived events to a specific time window. This is genuinely useful in production and costs almost nothing to set up:

```typescript
// sst.config.ts — enable archiving on the event bus
const eventBus = new sst.aws.Bus("RunwayBus");

import * as events from "aws-cdk-lib/aws-events";
import { Stack } from "aws-cdk-lib";

const cfnBus = eventBus.nodes.bus;
const stack = Stack.of(cfnBus);

new events.CfnArchive(stack, "RunwayBusArchive", {
  sourceArn: cfnBus.attrArn,
  retentionDays: 90,
  description: "All Runway domain events",
});
```

To replay: go to the EventBridge console, select your archive, choose a time window, and replay to the original bus (or a different one for testing). Consumers process replayed events exactly as they would real ones — make sure they're idempotent.

---

## 8.4 Scheduled Jobs with Cron

Some work isn't triggered by a user action. It happens on a schedule:

- Chase unpaid invoices every morning at 9am
- Send Runway users a weekly summary every Monday
- Expire draft invoices that haven't been sent after 60 days
- Sync currency exchange rates every hour

SST makes this simple:

```typescript
// sst.config.ts
const invoiceChaserJob = new sst.aws.Cron("InvoiceChaser", {
  schedule: "cron(0 9 * * ? *)",
  job: {
    handler: "src/jobs/invoice-chaser.handler",
    timeout: "300 seconds",
    link: [table, emailQueue],
  },
});
```

The `schedule` supports both `rate()` expressions (e.g., `rate(1 hour)`) and `cron()` expressions (standard 6-field AWS cron format, but note the `?` for day-of-week when day-of-month is specified).

### Idempotent Scheduled Jobs

Scheduled Lambdas are invoked exactly once per trigger (unlike SQS which can retry). But EventBridge occasionally delivers duplicate events, and you might also manually re-trigger a job during debugging. Design for it.

The invoice chaser job: find all invoices that are past due and haven't been chased today, send a reminder email, record that they were chased.

```typescript
// src/jobs/invoice-chaser.ts
import type { ScheduledHandler } from "aws-lambda";
import { queryOverdueInvoices } from "../repositories/invoices";
import { sendToEmailQueue } from "../lib/queue";
import { markInvoiceChased, wasInvoiceChasedToday } from "../repositories/invoice-chases";

export const handler: ScheduledHandler = async (event) => {
  const today = new Date().toISOString().slice(0, 10); // "2026-02-27"

  console.log("Invoice chaser running", { date: today });

  const overdueInvoices = await queryOverdueInvoices();

  let sent = 0;
  let skipped = 0;

  for (const invoice of overdueInvoices) {
    const alreadyChased = await wasInvoiceChasedToday(invoice.invoiceId, today);
    if (alreadyChased) {
      skipped++;
      continue;
    }

    await sendToEmailQueue({
      type: "invoice.reminder.email",
      invoiceId: invoice.invoiceId,
      workspaceId: invoice.workspaceId,
      clientEmail: invoice.clientEmail,
      clientName: invoice.clientName,
      invoiceNumber: invoice.invoiceNumber,
      amountCents: invoice.amountCents,
      dueDateIso: invoice.dueDate,
      daysOverdue: invoice.daysOverdue,
    });

    await markInvoiceChased(invoice.invoiceId, today);
    sent++;
  }

  console.log("Invoice chaser complete", { sent, skipped, total: overdueInvoices.length });
};
```

```typescript
// src/repositories/invoice-chases.ts
import { DynamoDBDocumentClient, PutCommand, GetCommand } from "@aws-sdk/lib-dynamodb";
import { Resource } from "sst";

export async function markInvoiceChased(invoiceId: string, date: string): Promise<void> {
  await docClient.send(
    new PutCommand({
      TableName: Resource.RunwayTable.name,
      Item: {
        PK: `INVOICE#${invoiceId}`,
        SK: `CHASED#${date}`,
        chasedAt: new Date().toISOString(),
        TTL: Math.floor(Date.now() / 1000) + 60 * 60 * 24 * 90, // 90 days
      },
      ConditionExpression: "attribute_not_exists(PK) AND attribute_not_exists(SK)",
    })
  );
}

export async function wasInvoiceChasedToday(invoiceId: string, date: string): Promise<boolean> {
  const result = await docClient.send(
    new GetCommand({
      TableName: Resource.RunwayTable.name,
      Key: {
        PK: `INVOICE#${invoiceId}`,
        SK: `CHASED#${date}`,
      },
    })
  );
  return !!result.Item;
}
```

The TTL on chase records means they auto-expire after 90 days — you're not accumulating dead data forever.

### Monitoring Cron Jobs

Scheduled jobs are silent by default. They don't surface in your API metrics. They don't return anything you can check. When they fail, nothing happens.

Set up a CloudWatch alarm on Lambda errors for each cron job:

```typescript
import * as cloudwatch from "aws-cdk-lib/aws-cloudwatch";
import * as cdk from "aws-cdk-lib";
import { Stack } from "aws-cdk-lib";

const chaserFunction = invoiceChaserJob.nodes.function;
const stack = Stack.of(chaserFunction);

const chaserErrorAlarm = new cloudwatch.Alarm(stack, "InvoiceChaserErrors", {
  metric: chaserFunction.metricErrors({
    period: cdk.Duration.minutes(15),
  }),
  threshold: 1,
  evaluationPeriods: 1,
  alarmName: "InvoiceChaserFailed",
  treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,
});
```

Also worth logging a structured "job completed" message at the end of every cron handler (as in the example above). A CloudWatch Insights query to see when the chaser last ran successfully is a 30-second job if your logs are structured.

```
fields @timestamp, sent, skipped, total
| filter @message = "Invoice chaser complete"
| sort @timestamp desc
| limit 10
```

### Avoiding Duplicate Executions Across Stages

One footgun with SST cron: if you have a `dev` stage and a `production` stage, both are running real cron jobs. The dev stage invoice chaser is emailing your actual clients' dev records at 9am.

Use stage-specific schedules — or just disable the cron in dev:

```typescript
const invoiceChaserJob = new sst.aws.Cron("InvoiceChaser", {
  schedule: $app.stage === "production"
    ? "cron(0 9 * * ? *)"
    : "rate(7 days)",
  job: {
    handler: "src/jobs/invoice-chaser.handler",
    link: [table, emailQueue],
  },
});
```

Or gate the actual sending behaviour inside the handler:

```typescript
if (process.env.SST_STAGE !== "production") {
  console.log("Non-production stage — skipping email sends", { stage: process.env.SST_STAGE });
  return;
}
```

Both approaches work. The handler-level gate is safer because it also catches manual test invocations.

---

## 8.5 Long-Running Jobs

Lambda's maximum execution time is 15 minutes. For most background work in Runway — sending emails, updating analytics, chasing invoices — that's more than enough.

But some jobs genuinely don't fit:

- Generating a full year-end financial report as a PDF (potentially 20+ minutes of data fetching and rendering)
- Migrating large datasets between storage backends
- Running nightly ML-based invoice fraud scoring across all invoices

### Extending Lambda with Chunked Processing

Before reaching for Step Functions, ask: can this be broken into chunks that each fit in 15 minutes?

The invoice report job: instead of generating one massive report, break it by month:

```typescript
// src/jobs/report-generator.ts
export const handler: ScheduledHandler = async () => {
  const months = getLast12Months(); // ["2025-02", "2025-03", ... "2026-01"]

  for (const month of months) {
    await sendToReportQueue({
      type: "generate.monthly.report",
      workspaceId: workspace.workspaceId,
      month,
    });
  }
};

export const reportWorker: SQSHandler = async (event) => {
  for (const record of event.Records) {
    const { workspaceId, month } = GenerateMonthlyReportMessage.parse(JSON.parse(record.body));
    await generateMonthlyReport(workspaceId, month);
  }
};
```

Fan-out via SQS: the orchestrator Lambda runs in seconds, the workers handle the real work in parallel. This handles 99% of "long-running" jobs.

### Step Functions for Multi-Step Workflows

When the work genuinely has multiple sequential steps with state between them, Step Functions are the right tool. SST wraps them with the `StateMachine` component.

A Runway example: year-end report generation with multiple sequential phases.

```typescript
// sst.config.ts
import * as sfn from "aws-cdk-lib/aws-stepfunctions";
import * as tasks from "aws-cdk-lib/aws-stepfunctions-tasks";
import * as cdk from "aws-cdk-lib";
import { Stack } from "aws-cdk-lib";

const fetchDataFn = new sst.aws.Function("FetchReportData", {
  handler: "src/jobs/report/fetch-data.handler",
  timeout: "900 seconds",
  link: [table],
});

const generatePdfFn = new sst.aws.Function("GenerateReportPdf", {
  handler: "src/jobs/report/generate-pdf.handler",
  timeout: "900 seconds",
  link: [bucket],
});

const notifyUserFn = new sst.aws.Function("NotifyReportReady", {
  handler: "src/jobs/report/notify.handler",
  link: [emailQueue],
});

const stack = Stack.of(fetchDataFn.nodes.function);

const fetchDataTask = new tasks.LambdaInvoke(stack, "FetchData", {
  lambdaFunction: fetchDataFn.nodes.function,
  outputPath: "$.Payload",
});

const generatePdfTask = new tasks.LambdaInvoke(stack, "GeneratePdf", {
  lambdaFunction: generatePdfFn.nodes.function,
  outputPath: "$.Payload",
});

const notifyTask = new tasks.LambdaInvoke(stack, "NotifyUser", {
  lambdaFunction: notifyUserFn.nodes.function,
  outputPath: "$.Payload",
});

const definition = fetchDataTask
  .next(generatePdfTask)
  .next(notifyTask);

const stateMachine = new sfn.StateMachine(stack, "ReportStateMachine", {
  definition,
  timeout: cdk.Duration.hours(2),
});
```

Each Lambda step receives the output of the previous one as its input. State flows through the execution. Step Functions handles retries at the step level — if PDF generation fails, it retries that step, not the whole workflow.

Step Functions are billed per state transition (~$0.025 per 1,000). For Runway's occasional year-end reports, negligible. For thousands of executions per hour, model the cost first.

### Reporting Job Progress to the Client

When a user triggers a long-running report generation, they need feedback. Three options:

**Polling** — simplest. Return a `jobId` immediately, expose a `GET /jobs/:jobId/status` endpoint, let the client poll every few seconds.

```typescript
const jobId = ulid();

await docClient.send(new PutCommand({
  TableName: Resource.RunwayTable.name,
  Item: {
    PK: `JOB#${jobId}`,
    SK: "STATUS",
    status: "pending",
    workspaceId,
    createdAt: new Date().toISOString(),
    TTL: Math.floor(Date.now() / 1000) + 60 * 60 * 24 * 7, // 7 days
  },
}));

await sfnClient.send(new StartExecutionCommand({
  stateMachineArn: Resource.ReportStateMachine.arn,
  input: JSON.stringify({ jobId, workspaceId, year }),
}));

return { jobId };
```

```typescript
// GET /jobs/:jobId
const item = await docClient.send(new GetCommand({
  TableName: Resource.RunwayTable.name,
  Key: { PK: `JOB#${jobId}`, SK: "STATUS" },
}));

return {
  jobId,
  status: item.Item?.status ?? "not_found",
  resultUrl: item.Item?.resultUrl ?? null,
};
```

Each Lambda step in the state machine updates the DynamoDB item with its progress. The client polls until `status === "complete"`.

**WebSockets** — lower latency, more complex. API Gateway WebSocket APIs allow the server to push to the client. SST has `sst.aws.ApiGatewayWebSocket`. Worth it if users are watching progress in real time (e.g., a progress bar during report generation).

**Server-Sent Events** — one-way streaming from server to client, simpler than WebSockets. Lambda isn't ideal for SSE (streaming HTTP responses require Lambda's response streaming mode, which SST supports but needs explicit configuration). For most cases, polling is simpler and polling every 2 seconds is imperceptible to users.

Start with polling. Add WebSockets when users complain about it.

---

## 8.6 Putting It Together: Runway's Async Architecture

Here's how the async layer fits into Runway:

```
Invoice marked as paid
  │
  ├─ DB update (synchronous)
  │
  ├─ EventBridge: emit "invoice.paid"
  │     ├─ Analytics worker: record payment metrics
  │     ├─ Notification worker: push notification to freelancer
  │     └─ (future: Xero sync, Slack notification, etc.)
  │
  └─ SQS: queue receipt email → Email worker → Mailgun
              (with dedup guard, retry, DLQ)

Invoice chaser (cron, 9am daily)
  │
  ├─ Query DynamoDB for overdue invoices
  ├─ Skip any already chased today
  ├─ Queue reminder emails via SQS
  └─ Record chase date in DynamoDB (TTL: 90 days)

Year-end report requested
  │
  ├─ Return jobId immediately
  ├─ Start Step Functions execution
  │     ├─ Step 1: FetchData Lambda (up to 15 min)
  │     ├─ Step 2: GeneratePdf Lambda (up to 15 min)
  │     └─ Step 3: NotifyUser Lambda → email queue
  └─ Client polls GET /jobs/:jobId until complete
```

The invoice handler returns 200 in under 100ms. The emails send. The analytics update. The client gets their notification. None of it blocks the API.

---

## Where We Are

Runway now has a proper async layer. The architecture looks like this:

```
[Cognito] → [API Gateway + JWT Authoriser]
  ↓
[Lambda handlers — thin, fast, return 200]
  ↓
  ├─ [DynamoDB] — structured data, Ch 4
  ├─ [Aurora Serverless] — relational data, Ch 5
  ├─ [S3] — files and uploads, Ch 6
  ├─ [SQS] — reliable async (email worker, etc.), Ch 8
  └─ [EventBridge] — domain events (fan-out), Ch 8
        ├─ Analytics worker
        ├─ Notification worker
        └─ (future consumers)

[Cron] → Invoice chaser, weekly summary, etc.
[Step Functions] → Year-end reports, multi-step jobs
```

The system is now decoupled. Adding a Slack integration means subscribing a new worker to EventBridge events — the invoice handler never changes. Adding a new scheduled job means adding a cron entry — it doesn't touch anything else.

Chapter 9 covers deploying all of this reliably: CI/CD, preview environments, GitHub Actions with OIDC, and database migrations that don't take the system down.

---

> **The code for this chapter** is available at `08-queues-events-background-jobs/` in the companion repository.
