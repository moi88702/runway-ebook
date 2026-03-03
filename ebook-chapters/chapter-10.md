# Chapter 10: Monitoring and Observability

Runway is deployed. Users are creating invoices, marking payments, chasing clients. Things are working.

Until they aren't.

A Lambda function starts returning 500s for one workspace. The invoice chaser ran at 9am and produced no output — was that because there were no overdue invoices, or because it silently crashed? The PDF generation step of the year-end report is taking 8 minutes; last week it took 2. Are you about to hit the Lambda timeout?

You won't know any of this unless you instrument it. Observability isn't something you add after a production incident teaches you the hard way — or rather, everyone does add it after their first production incident, and the lesson is that you should have added it first.

This chapter covers the three pillars of observability in an AWS/Lambda context, what SST sets up for you automatically, and the minimum setup that makes the difference between "something is broken, we have no idea what" and "that Lambda is returning 500s for workspace X since 14:22, here's the trace."

---

## 10.1 The Three Pillars in an AWS Context

**Logs** are the raw output of your system. CloudWatch Logs captures every `console.log`, `console.error`, and structured JSON log your Lambda emits. They're the first thing you look at when something is broken.

**Metrics** are counts, rates, and distributions over time. Lambda error rate, API Gateway p99 latency, DynamoDB consumed capacity, SQS queue depth. Metrics tell you *that* something is wrong. Logs tell you *what*.

**Traces** connect the dots across service boundaries. A single user request might hit API Gateway, invoke a Lambda, query DynamoDB, put a message on SQS, and trigger a worker Lambda. A trace is the end-to-end record of that request, with timing at each hop. Traces tell you *where* the latency or failure lives.

SST sets up some of this automatically:

- CloudWatch Logs: automatic. Every Lambda gets a log group.
- Lambda built-in metrics: automatic. Error count, duration, throttles, concurrency.
- X-Ray tracing: opt-in per function (SST makes this one line).

What you have to build: structured log format, correlation IDs across service calls, custom business metrics, useful dashboards, and alerts that fire on things that matter.

---

## 10.2 Structured Logging

`console.log("it broke")` is useless at 3am. You can't search it, filter it, aggregate it, or correlate it to a specific request.

Structured logging means every log entry is a JSON object with consistent fields. CloudWatch Logs Insights can query JSON logs natively, which means "show me all errors in workspace X in the last hour" is a 10-second query rather than a grep expedition.

### Building a Logger

```typescript
// src/lib/logger.ts
type LogLevel = "DEBUG" | "INFO" | "WARN" | "ERROR";

interface LogContext {
  requestId?: string;
  workspaceId?: string;
  userId?: string;
  invoiceId?: string;
  [key: string]: unknown;
}

class Logger {
  private context: LogContext = {};

  withContext(ctx: LogContext): Logger {
    const child = new Logger();
    child.context = { ...this.context, ...ctx };
    return child;
  }

  private write(level: LogLevel, message: string, data?: Record<string, unknown>): void {
    const entry = {
      level,
      message,
      timestamp: new Date().toISOString(),
      ...this.context,
      ...data,
    };
    if (level === "ERROR") {
      console.error(JSON.stringify(entry));
    } else {
      console.log(JSON.stringify(entry));
    }
  }

  debug(message: string, data?: Record<string, unknown>): void {
    if (process.env.LOG_LEVEL === "DEBUG") {
      this.write("DEBUG", message, data);
    }
  }

  info(message: string, data?: Record<string, unknown>): void {
    this.write("INFO", message, data);
  }

  warn(message: string, data?: Record<string, unknown>): void {
    this.write("WARN", message, data);
  }

  error(message: string, data?: Record<string, unknown>): void {
    this.write("ERROR", message, data);
  }
}

export const logger = new Logger();
```

Using it in a handler:

```typescript
// src/handlers/api/invoices.ts
import { logger } from "../../lib/logger";
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const log = logger.withContext({
    requestId: context.awsRequestId,
    path: event.rawPath,
    method: event.requestContext.http.method,
  });

  const { workspaceId } = event.pathParameters ?? {};

  if (!workspaceId) {
    log.warn("Missing workspaceId in path");
    return { statusCode: 400, body: JSON.stringify({ error: "Missing workspaceId" }) };
  }

  const scopedLog = log.withContext({ workspaceId });

  try {
    const invoices = await listInvoices(workspaceId);
    scopedLog.info("Invoices fetched", { count: invoices.length });
    return { statusCode: 200, body: JSON.stringify({ invoices }) };
  } catch (err) {
    scopedLog.error("Failed to fetch invoices", {
      error: err instanceof Error ? err.message : String(err),
      stack: err instanceof Error ? err.stack : undefined,
    });
    return { statusCode: 500, body: JSON.stringify({ error: "Internal server error" }) };
  }
};
```

Every log entry for this request now has `requestId`, `workspaceId`, `path`, and `method`. Filter by `workspaceId` in CloudWatch Logs Insights and every relevant log line appears.

### CloudWatch Logs Insights Queries Worth Saving

```sql
-- All errors in the last hour
fields @timestamp, level, message, workspaceId, requestId
| filter level = "ERROR"
| sort @timestamp desc
| limit 50

-- Error rate by workspace (last 24h)
fields workspaceId
| filter level = "ERROR"
| stats count() as errorCount by workspaceId
| sort errorCount desc

-- Slow requests (> 1000ms)
fields @timestamp, path, @duration, workspaceId
| filter @type = "REPORT" and @duration > 1000
| sort @duration desc
| limit 20

-- Invoice chaser job history
fields @timestamp, sent, skipped, total
| filter message = "Invoice chaser complete"
| sort @timestamp desc
| limit 20
```

Save these as named queries in CloudWatch (Logs Insights → Saved queries). When something breaks at 11pm you want to click "run query" not reconstruct the syntax from memory.

### Correlation IDs Across Service Boundaries

When an API request triggers an SQS message that triggers a worker Lambda, the `requestId` from the original API call isn't carried forward. You lose the thread.

Fix: include a `correlationId` in every SQS message and EventBridge event. The worker logs with that ID.

```typescript
// src/types/queue-messages.ts — add correlationId to all messages
export const InvoicePaidEmailMessage = z.object({
  type: z.literal("invoice.paid.email"),
  correlationId: z.string(),
  invoiceId: z.string(),
  // ...
});
```

```typescript
// src/handlers/api/invoices.ts — when queueing the email
await sendToEmailQueue({
  type: "invoice.paid.email",
  correlationId: context.awsRequestId,
  invoiceId: invoice.invoiceId,
  // ...
});
```

```typescript
// src/workers/email.ts — log with the correlation ID
const scopedLog = logger.withContext({
  correlationId: message.correlationId,
  invoiceId: message.invoiceId,
});
scopedLog.info("Processing email");
```

Now you can search CloudWatch for a `correlationId` and see the full chain: API handler → queue send → worker processing → email sent. One query, the whole story.

---

## 10.3 Custom CloudWatch Metrics

Lambda's built-in metrics (error count, duration, throttles) tell you about the infrastructure. They don't tell you about the business.

"Invoice chaser sent 47 emails" is a business metric. "Payment success rate by plan tier" is a business metric. "New workspace signups per hour" is a business metric. None of these come free — you have to emit them.

### Emitting Custom Metrics

The cheapest way to emit custom CloudWatch metrics from Lambda is the Embedded Metric Format (EMF). You log a specially formatted JSON entry and CloudWatch extracts metrics from it automatically. No separate API call, no cost per metric write (it's included in your log ingestion).

```typescript
// src/lib/metrics.ts
interface MetricEntry {
  name: string;
  value: number;
  unit?: "Count" | "Milliseconds" | "Bytes" | "Percent";
  dimensions?: Record<string, string>;
}

export function emitMetric(metric: MetricEntry): void {
  const emf = {
    _aws: {
      Timestamp: Date.now(),
      CloudWatchMetrics: [
        {
          Namespace: "Runway/Business",
          Dimensions: [Object.keys(metric.dimensions ?? {})],
          Metrics: [
            {
              Name: metric.name,
              Unit: metric.unit ?? "Count",
            },
          ],
        },
      ],
    },
    [metric.name]: metric.value,
    ...metric.dimensions,
  };

  console.log(JSON.stringify(emf));
}
```

```typescript
// src/jobs/invoice-chaser.ts — emit metrics after the run
emitMetric({
  name: "InvoiceChaserEmailsSent",
  value: sent,
  dimensions: { Stage: process.env.SST_STAGE ?? "unknown" },
});

emitMetric({
  name: "OverdueInvoiceCount",
  value: overdueInvoices.length,
  dimensions: { Stage: process.env.SST_STAGE ?? "unknown" },
});
```

```typescript
// src/workers/email.ts — track email success/failure
emitMetric({ name: "EmailSent", value: 1, dimensions: { Type: message.type } });
// or on failure:
emitMetric({ name: "EmailFailed", value: 1, dimensions: { Type: message.type } });
```

These metrics appear in CloudWatch under the `Runway/Business` namespace within a few minutes. Build dashboards and alarms on top of them.

### Dashboards That Actually Help

A CloudWatch dashboard that shows Lambda error rate and duration is table stakes. More useful: a dashboard that shows whether the *business* is working.

```typescript
import * as cloudwatch from "aws-cdk-lib/aws-cloudwatch";
import * as cdk from "aws-cdk-lib";
import { Stack } from "aws-cdk-lib";

const stack = Stack.of(invoiceHandler.nodes.function);

const dashboard = new cloudwatch.Dashboard(stack, "RunwayDashboard", {
  dashboardName: `Runway-${$app.stage}`,
  widgets: [
    [
      new cloudwatch.GraphWidget({
        title: "Invoices Marked Paid (24h)",
        left: [new cloudwatch.Metric({
          namespace: "Runway/Business",
          metricName: "InvoicePaid",
          statistic: "Sum",
          period: cdk.Duration.hours(1),
          dimensionsMap: { Stage: $app.stage },
        })],
      }),
      new cloudwatch.GraphWidget({
        title: "Email Send Rate",
        left: [
          new cloudwatch.Metric({
            namespace: "Runway/Business",
            metricName: "EmailSent",
            statistic: "Sum",
            period: cdk.Duration.minutes(15),
          }),
          new cloudwatch.Metric({
            namespace: "Runway/Business",
            metricName: "EmailFailed",
            statistic: "Sum",
            period: cdk.Duration.minutes(15),
          }),
        ],
      }),
    ],
    [
      new cloudwatch.GraphWidget({
        title: "API Error Rate",
        left: [api.nodes.httpApi.metricClientError(), api.nodes.httpApi.metric5XXError()],
      }),
      new cloudwatch.GraphWidget({
        title: "Lambda Duration p99",
        left: [invoiceHandler.nodes.function.metricDuration({ statistic: "p99" })],
      }),
    ],
  ],
});
```

The first row answers "is the product working?" The second row answers "is the infrastructure healthy?" Keep them separate — a spike in Lambda duration might not cause errors, but a spike in business errors usually will.

---

## 10.4 Distributed Tracing with X-Ray

X-Ray gives you end-to-end traces across Lambda, API Gateway, DynamoDB, and SQS. Enable it in SST:

```typescript
// sst.config.ts — enable X-Ray on all functions
const api = new sst.aws.ApiGatewayV2("Api", {
  // ...
});

const invoiceHandler = new sst.aws.Function("InvoiceHandler", {
  handler: "src/handlers/api/invoices.handler",
  tracing: "active",
  link: [table],
});
```

Or enable it globally via the `transform` option:

```typescript
// sst.config.ts
app.transform({
  function: {
    tracing: "active",
  },
});
```

With tracing enabled, every Lambda invocation creates a trace segment. DynamoDB calls within the Lambda create sub-segments automatically (the AWS SDK instruments itself when X-Ray is active). You get timing for the full request without any code changes.

### Annotating Traces

Automatic instrumentation captures timings. Manual annotations add context — the kind that makes "find the slow request" a query rather than a guess.

```typescript
// src/handlers/api/invoices.ts
import * as AWSXRay from "aws-xray-sdk-core";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const segment = AWSXRay.getSegment();

  segment?.addAnnotation("workspaceId", workspaceId);
  segment?.addAnnotation("userId", userId);

  const subsegment = segment?.addNewSubsegment("listInvoices");
  try {
    const invoices = await listInvoices(workspaceId);
    subsegment?.addMetadata("count", invoices.length);
    return { statusCode: 200, body: JSON.stringify({ invoices }) };
  } finally {
    subsegment?.close();
  }
};
```

With `workspaceId` as an annotation, you can filter the X-Ray trace map to show only requests for a specific workspace. With custom subsegments, you can see that `listInvoices` is taking 340ms while the rest of the handler is under 10ms.

### Reading a Trace Map

The X-Ray service map in the AWS console shows nodes (services) and edges (calls between them). A healthy Runway service map looks like:

```
[API Gateway] → [Invoice Lambda] → [DynamoDB]
                                 → [SQS]
                                       ↓
                               [Email Worker Lambda]
```

Thick edges mean high call volume. Red edges mean errors. High latency edges are highlighted. Click any node to see its error rate, latency distribution, and sample traces.

When investigating an incident: look for red edges first, then check the slowest traces on the highlighted edges, then read the logs for the request IDs in those traces.

---

## 10.5 Alerting That Doesn't Cry Wolf

An alert should mean: "you need to look at this now." If alerts fire for things that sort themselves out, or fire multiple times for the same issue, or fire at 3am for something that can wait until morning, they stop meaning anything. Alert fatigue is real and it kills incident response.

The minimum alerting surface for Runway:

### Lambda Error Rate

```typescript
// sst.config.ts
// stack is already defined above via Stack.of(invoiceHandler.nodes.function)
const errorAlarm = new cloudwatch.Alarm(stack, "ApiErrorAlarm", {
  alarmName: `${$app.stage}-RunwayApiErrors`,
  metric: invoiceHandler.nodes.function.metricErrors({
    period: cdk.Duration.minutes(5),
    statistic: "Sum",
  }),
  threshold: 5,
  evaluationPeriods: 2,
  datapointsToAlarm: 2,
  comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
  treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,
});
```

Set `threshold` based on your traffic. 5 errors in 5 minutes might be fine if you have 10,000 requests/minute (that's a 0.01% error rate). Set it as an absolute count for low-traffic services, as a rate metric for high-traffic ones.

`datapointsToAlarm: 2` with `evaluationPeriods: 2` means the alarm only fires if the threshold is breached in two consecutive evaluation periods. One bad data point doesn't page you.

### SQS DLQ Depth

Any message in a DLQ is a symptom. You want to know about it.

```typescript
const emailDlq = emailQueue.nodes.deadLetterQueue;

const dlqAlarm = emailDlq
  ? new cloudwatch.Alarm(stack, "EmailDlqAlarm", {
      alarmName: `${$app.stage}-EmailDlqHasMessages`,
      metric: emailDlq.metricApproximateNumberOfMessagesVisible({
        period: cdk.Duration.minutes(5),
        statistic: "Sum",
      }),
      threshold: 1,
      evaluationPeriods: 1,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
      treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,
    })
  : undefined;
```

Zero tolerance on DLQs. One message in the DLQ should alert.

### Lambda Duration Approaching Timeout

A Lambda function approaching its timeout is a ticking clock. Set a duration alarm at 80% of the configured timeout:

```typescript
const invoiceChaserFn = invoiceChaserJob.nodes.function;

const timeoutWarning = new cloudwatch.Alarm(stack, "InvoiceChaserDurationWarning", {
  alarmName: `${$app.stage}-InvoiceChaserSlowdown`,
  metric: invoiceChaserFn.metricDuration({
    period: cdk.Duration.minutes(15),
    statistic: "p99",
  }),
  threshold: 240_000, // 240 seconds = 80% of 300s timeout
  evaluationPeriods: 1,
  comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
  treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,
});
```

### Routing Alerts to Slack

Create an SNS topic, subscribe Slack (via a Lambda that forwards to the Slack Incoming Webhook), and point your alarms at the topic:

```typescript
// sst.config.ts
import * as sns from "aws-cdk-lib/aws-sns";
import * as snsSubscriptions from "aws-cdk-lib/aws-sns-subscriptions";
import * as cloudwatchActions from "aws-cdk-lib/aws-cloudwatch-actions";
// stack is already defined via Stack.of(invoiceHandler.nodes.function) above

const alertTopic = new sns.Topic(stack, "AlertTopic");

const slackAlerter = new sst.aws.Function("SlackAlerter", {
  handler: "src/workers/slack-alert.handler",
  environment: {
    SLACK_WEBHOOK_URL: process.env.SLACK_WEBHOOK_URL ?? "",
  },
});

alertTopic.addSubscription(
  new snsSubscriptions.LambdaSubscription(slackAlerter.nodes.function)
);

errorAlarm.addAlarmAction(new cloudwatchActions.SnsAction(alertTopic));
if (dlqAlarm) dlqAlarm.addAlarmAction(new cloudwatchActions.SnsAction(alertTopic));
```

```typescript
// src/workers/slack-alert.ts
import type { SNSHandler } from "aws-lambda";

export const handler: SNSHandler = async (event) => {
  for (const record of event.Records) {
    const alarm = JSON.parse(record.Sns.Message);
    const emoji = alarm.NewStateValue === "ALARM" ? "🚨" : "✅";

    await fetch(process.env.SLACK_WEBHOOK_URL!, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        text: `${emoji} *${alarm.AlarmName}*\n${alarm.NewStateReason}`,
      }),
    });
  }
};
```

Keep the Slack alert short. Name, state, reason. Don't dump a CloudWatch wall of text into the channel — include a deep link to the alarm instead.

### What Not to Alert On

- Lambda cold starts (expected, usually sub-second, not actionable)
- Individual 4xx errors (clients send bad requests, this is normal)
- Lambda throttles under 10/minute (burstable, recovers quickly)
- DynamoDB consumed capacity approaching provisioned (irrelevant with on-demand billing)
- Anything you'd snooze

Every alert you add that you'd snooze teaches you to ignore all alerts. Start with five alerts max. Add more when a production incident reveals a gap.

---

## 10.6 The Minimum Viable Observability Setup

If you're starting from scratch, here's what to ship on day one:

**1. Structured JSON logging.** The `Logger` class from section 10.2. Add `requestId` from Lambda context to every log. Takes an hour.

**2. Lambda X-Ray tracing.** One line in `sst.config.ts`. Gives you trace maps and latency breakdown with no code changes.

**3. Four CloudWatch alarms:**
   - API Lambda error rate (> 5 in 5 minutes)
   - Email DLQ depth (≥ 1)
   - Invoice chaser cron failures (any error)
   - Lambda duration p99 approaching timeout (for any function with a job > 60s)

**4. Two CloudWatch Logs Insights saved queries:**
   - "Recent errors" — filter level = ERROR, last hour
   - "Cron job history" — filter message = "complete", last 7 days

**5. A single dashboard with two rows:** business metrics (invoices, emails) and infrastructure metrics (error rate, p99 latency).

That's it. Two hours of work. It's the difference between "the invoice chaser silently stopped running three days ago and nobody noticed" and "the invoice chaser alarm fired at 9:01am this morning."

Add traces with annotations, custom business metrics, anomaly detection, and per-workspace alerting as the product and team grow. The foundation carries everything else.

---

## Where We Are

Runway is complete. The full architecture, from first request to last observable metric:

```
[GitHub Actions CI/CD]
  └─ Type check → Migrate → Deploy (preview per PR, staging on merge, production on approval)

[Cognito User Pool]
  └─ JWT tokens → verified at API Gateway

[API Gateway V2]
  └─ Routes → Lambda handlers (thin, typed, structured logging)

[Lambda handlers]
  ├─ [DynamoDB] — single-table, typed repositories (Ch 4)
  ├─ [Aurora Serverless v2 + Prisma] — relational data, migrations (Ch 5)
  ├─ [S3] — file uploads via presigned URLs (Ch 6)
  ├─ [SQS] — reliable async work (Ch 8)
  └─ [EventBridge] — domain event fan-out (Ch 8)

[Background layer]
  ├─ Cron (invoice chaser, weekly digest)
  └─ Step Functions (year-end report generation)

[Observability]
  ├─ CloudWatch Logs — structured JSON, Logs Insights queries
  ├─ X-Ray — distributed traces across all services
  ├─ CloudWatch Metrics — business + infrastructure dashboards
  └─ CloudWatch Alarms → SNS → Slack
```

Runway is a real SaaS. Type-safe end to end, from the API contract to the database schema to the queue message body. Deployed automatically on every push to `main`. Monitored in production with alerts that mean something.

This is what shipping TypeScript on AWS looks like when it's done properly. Not the tutorial version. The version you'd actually run a business on.

---

## What's Next

The appendices cover the things that didn't fit cleanly into the chapter flow:

- **Appendix A** — AWS account setup done right (root account lockdown, billing alerts, MFA)
- **Appendix B** — Common error reference (ConditionalCheckFailedException, connection pool exhaustion, IAM permission errors — causes and fixes)
- **Appendix C** — SST component cheat sheet
- **Appendix D** — TypeScript patterns reference (`satisfies`, discriminated unions, branded types)
- **Appendix E** — Cost estimation at 1k, 10k, and 100k users

If you've read this far and built Runway alongside it: you now know more about running TypeScript on AWS than most teams who've been doing it for years. The gap isn't the knowledge — it's taking the time to get it right.

Ship it.

---

> **The code for this chapter** is available at `10-monitoring-observability/` in the companion repository.
