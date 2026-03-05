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
  requestId?:    string;
  workspaceId?:  string;
  userId?:       string;
  invoiceId?:    string;
  correlationId?: string;
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
    if (process.env.LOG_LEVEL === "DEBUG") this.write("DEBUG", message, data);
  }

  info(message: string, data?: Record<string, unknown>): void  { this.write("INFO",  message, data); }
  warn(message: string, data?: Record<string, unknown>): void  { this.write("WARN",  message, data); }
  error(message: string, data?: Record<string, unknown>): void { this.write("ERROR", message, data); }
}

export const logger = new Logger();
```

Using it in a handler:

```typescript
// src/functions/invoices/list.ts
import { logger } from "../../lib/logger";
import type { ValidatedHandler } from "../../lib/handler";

export const handler: ValidatedHandler = async (event, context) => {
  const log = logger.withContext({
    requestId: context.awsRequestId,
    path:      event.rawPath,
    method:    event.requestContext.http.method,
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
      stack: err instanceof Error ? err.stack  : undefined,
    });
    return { statusCode: 500, body: JSON.stringify({ error: "Internal server error" }) };
  }
};
```

Every log entry for this request now has `requestId`, `workspaceId`, `path`, and `method`. Filter by `workspaceId` in CloudWatch Logs Insights and every relevant log line appears instantly.

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
fields @timestamp, message, sent, skipped, total
| filter message = "Invoice chaser complete"
| sort @timestamp desc
| limit 20

-- Trace ID lookup (find all logs for one request)
fields @timestamp, level, message, workspaceId
| filter @xrayTraceId = "1-abc123-..."
| sort @timestamp asc
```

Save these as named queries in CloudWatch (Logs Insights → Saved queries). When something breaks at 11pm you want to click "run query," not reconstruct the syntax from memory.

### Correlation IDs Across Service Boundaries

When an API request triggers an SQS message that triggers a worker Lambda, the `requestId` from the original API call isn't carried forward automatically. You lose the thread.

Fix: include a `correlationId` in every SQS message and EventBridge event. The worker logs with that ID.

```typescript
// src/types/queue-messages.ts — add correlationId to all message types
export const InvoicePaidEmailSchema = z.object({
  type:          z.literal("invoice.paid.email"),
  correlationId: z.string(),
  invoiceId:     z.string(),
  clientEmail:   z.string(),
  clientName:    z.string(),
});
```

```typescript
// src/functions/invoices/update.ts — include the Lambda request ID when queuing
await sendToEmailQueue({
  type:          "invoice.paid.email",
  correlationId: context.awsRequestId,
  invoiceId:     invoice.invoiceId,
  clientEmail:   invoice.clientEmail,
  clientName:    invoice.clientName,
});
```

```typescript
// src/workers/email.ts — log with the correlation ID from the original request
const scopedLog = logger.withContext({
  correlationId: message.correlationId,
  invoiceId:     message.invoiceId,
  type:          message.type,
});
scopedLog.info("Processing email");
```

Now search CloudWatch for a `correlationId` value and see the full chain: API handler → queue send → worker processing → email sent. One query, the whole story.

---

## 10.3 Log Retention and Cost Management

CloudWatch Logs default retention is *never expire*. Every `console.log` your Lambda emits is kept forever at $0.50/GB stored per month. Left unconfigured, a busy production system accumulates gigabytes of logs that nobody reads, and you notice when the AWS bill arrives.

Configure retention in `sst.config.ts` using Pulumi's `aws.cloudwatch.LogGroup` resource:

```typescript
// sst.config.ts — set retention on all Lambda log groups
import * as aws from "@pulumi/aws";

// Helper: create a log group with retention for any SST function
function withRetention(fn: sst.aws.Function, retentionDays = 30) {
  return new aws.cloudwatch.LogGroup(`${fn.name}Logs`, {
    name:            $interpolate`/aws/lambda/${fn.nodes.function.name}`,
    retentionInDays: retentionDays,
  });
}

// Apply to each function after defining it
withRetention(invoiceHandler, 30);
withRetention(emailWorker,    14);
withRetention(invoiceChaser,  90);  // Keep cron logs longer for audit purposes
```

Alternatively, SST's `Function` component exposes a `transform` property to configure the underlying `aws.lambda.Function` Pulumi resource. For log groups, the explicit `withRetention` helper is clearer.

### What to retain and for how long

| Log group | Retention | Reason |
|-----------|-----------|--------|
| API handlers | 30 days | Enough to investigate any reported issue |
| SQS workers | 14 days | Short-lived processing, rarely needed beyond a week |
| Cron jobs | 90 days | Audit trail for payment-adjacent automation |
| Migration runner | 1 year | Schema change history has compliance value |
| Staging | 7 days | Cheap, infrequently referenced |

One month for production API logs covers every realistic incident investigation window while keeping storage costs reasonable.

### Querying Logs from the CLI

The console is fine for one-off queries. For scripts, automation, or during an incident when you don't want to navigate a UI, use the CLI:

```bash
# Live tail — equivalent to sst dev log streaming for a deployed function
aws logs tail /aws/lambda/production-runway-InvoiceHandler \
  --follow \
  --format short

# Filter for errors in the last 30 minutes
aws logs filter-log-events \
  --log-group-name /aws/lambda/production-runway-InvoiceHandler \
  --start-time $(date -d '30 minutes ago' +%s000) \
  --filter-pattern '{ $.level = "ERROR" }'

# Run a Logs Insights query and wait for results
aws logs start-query \
  --log-group-name /aws/lambda/production-runway-InvoiceHandler \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, message, workspaceId | filter level = "ERROR" | limit 20'

# (copy the queryId from the response, then:)
aws logs get-query-results --query-id <queryId>
```

Add the most-used commands to your RUNBOOK.md so the right flags are one paste away during an incident.

---

## 10.4 Custom CloudWatch Metrics

Lambda's built-in metrics (error count, duration, throttles) tell you about the infrastructure. They don't tell you about the business.

"Invoice chaser sent 47 emails" is a business metric. "Payment success rate" is a business metric. "New workspace signups per hour" is a business metric. None of these come for free — you have to emit them.

### Emitting Custom Metrics with EMF

The cheapest way to emit custom CloudWatch metrics from Lambda is the Embedded Metric Format (EMF). You log a specially formatted JSON entry and CloudWatch extracts metrics from it automatically. No separate API call, no extra cost per metric write — it's included in your log ingestion.

```typescript
// src/lib/metrics.ts
interface MetricEntry {
  name:        string;
  value:       number;
  unit?:       "Count" | "Milliseconds" | "Bytes" | "Percent";
  dimensions?: Record<string, string>;
}

export function emitMetric(metric: MetricEntry): void {
  const dimensionKeys = Object.keys(metric.dimensions ?? {});

  const emf = {
    _aws: {
      Timestamp: Date.now(),
      CloudWatchMetrics: [
        {
          Namespace:  "Runway/Business",
          Dimensions: [dimensionKeys],
          Metrics:    [{ Name: metric.name, Unit: metric.unit ?? "Count" }],
        },
      ],
    },
    [metric.name]: metric.value,
    ...metric.dimensions,
  };

  console.log(JSON.stringify(emf));
}
```

Using it in the invoice chaser:

```typescript
// src/jobs/invoice-chaser.ts — emit metrics after the run
logger.info("Invoice chaser complete", { sent, skipped, total: overdueInvoices.length });

emitMetric({
  name:       "InvoiceChaserEmailsSent",
  value:      sent,
  dimensions: { Stage: process.env.SST_STAGE ?? "unknown" },
});

emitMetric({
  name:       "OverdueInvoiceCount",
  value:      overdueInvoices.length,
  dimensions: { Stage: process.env.SST_STAGE ?? "unknown" },
});
```

In the email worker:

```typescript
// src/workers/email.ts — track per-type success/failure
emitMetric({ name: "EmailSent",   value: 1, dimensions: { Type: message.type } });
// or on failure:
emitMetric({ name: "EmailFailed", value: 1, dimensions: { Type: message.type } });
```

These metrics appear in CloudWatch under the `Runway/Business` namespace within a few minutes. Build dashboards and alarms on top of them.

---

## 10.5 CloudWatch Dashboards and Alarms

SST v3 uses Pulumi under the hood, so infrastructure resources — including CloudWatch dashboards, alarms, and SNS topics — are defined using Pulumi's `aws` provider directly. No CDK imports needed.

### Alarms

```typescript
// sst.config.ts — define alarms after all functions are created
import * as aws from "@pulumi/aws";

// ── API error rate ──────────────────────────────────────────────────────────
const apiErrorAlarm = new aws.cloudwatch.MetricAlarm("ApiErrorAlarm", {
  alarmName:          `${$app.stage}-RunwayApiErrors`,
  comparisonOperator: "GreaterThanOrEqualToThreshold",
  evaluationPeriods:  2,
  datapointsToAlarm:  2,
  metricName:         "Errors",
  namespace:          "AWS/Lambda",
  period:             300,  // 5 minutes
  statistic:          "Sum",
  threshold:          5,
  treatMissingData:   "notBreaching",
  dimensions: {
    FunctionName: invoiceHandler.nodes.function.name,
  },
});

// ── Email DLQ depth ─────────────────────────────────────────────────────────
const emailDlqAlarm = new aws.cloudwatch.MetricAlarm("EmailDlqAlarm", {
  alarmName:          `${$app.stage}-EmailDlqHasMessages`,
  comparisonOperator: "GreaterThanOrEqualToThreshold",
  evaluationPeriods:  1,
  datapointsToAlarm:  1,
  metricName:         "ApproximateNumberOfMessagesVisible",
  namespace:          "AWS/SQS",
  period:             300,
  statistic:          "Sum",
  threshold:          1,
  treatMissingData:   "notBreaching",
  dimensions: {
    QueueName: emailQueue.nodes.deadLetterQueue.name,
  },
});

// ── Invoice chaser duration approaching timeout ─────────────────────────────
const chaserTimeoutAlarm = new aws.cloudwatch.MetricAlarm("InvoiceChaserDurationWarning", {
  alarmName:          `${$app.stage}-InvoiceChaserSlowdown`,
  comparisonOperator: "GreaterThanOrEqualToThreshold",
  evaluationPeriods:  1,
  datapointsToAlarm:  1,
  metricName:         "Duration",
  namespace:          "AWS/Lambda",
  period:             900,  // 15 minutes (matches cron frequency)
  extendedStatistic:  "p99",
  threshold:          240_000,  // 240s = 80% of 300s timeout
  treatMissingData:   "notBreaching",
  dimensions: {
    FunctionName: invoiceChaserJob.nodes.function.name,
  },
});
```

`datapointsToAlarm: 2` with `evaluationPeriods: 2` means two consecutive bad data points are required before the alarm fires. One noisy data point doesn't wake anyone up.

### Routing Alarms to Slack

Create an SNS topic and a Lambda that forwards alarm messages to a Slack webhook:

```typescript
// sst.config.ts
const alertTopic = new aws.sns.Topic("AlertTopic", {
  name: `${$app.stage}-runway-alerts`,
});

const slackAlerter = new sst.aws.Function("SlackAlerter", {
  handler:     "src/workers/slack-alert.handler",
  environment: { SLACK_WEBHOOK_URL: process.env.SLACK_WEBHOOK_URL ?? "" },
});

new aws.sns.TopicSubscription("SlackAlerterSubscription", {
  topic:    alertTopic.arn,
  protocol: "lambda",
  endpoint: slackAlerter.nodes.function.arn,
});

// Allow SNS to invoke the Lambda
new aws.lambda.Permission("SlackAlerterSnsPermission", {
  action:    "lambda:InvokeFunction",
  function:  slackAlerter.nodes.function.name,
  principal: "sns.amazonaws.com",
  sourceArn: alertTopic.arn,
});

// Point alarms at the topic
new aws.cloudwatch.MetricAlarmAlarmAction("ApiErrorAlarmAction", {
  alarmArn:        apiErrorAlarm.arn,
  alarmActionArns: [alertTopic.arn],
});
```

The Slack alert Lambda:

```typescript
// src/workers/slack-alert.ts
import type { SNSHandler } from "aws-lambda";

export const handler: SNSHandler = async (event) => {
  const webhookUrl = process.env.SLACK_WEBHOOK_URL;
  if (!webhookUrl) return;

  for (const record of event.Records) {
    const alarm = JSON.parse(record.Sns.Message) as {
      AlarmName:      string;
      NewStateValue:  "ALARM" | "OK" | "INSUFFICIENT_DATA";
      NewStateReason: string;
      AWSAccountId:   string;
    };

    const emoji   = alarm.NewStateValue === "ALARM" ? "🚨" : "✅";
    const color   = alarm.NewStateValue === "ALARM" ? "danger" : "good";
    const region  = record.Sns.TopicArn.split(":")[3];
    const alarmUrl =
      `https://${region}.console.aws.amazon.com/cloudwatch/home` +
      `#alarmsV2:alarm/${encodeURIComponent(alarm.AlarmName)}`;

    await fetch(webhookUrl, {
      method:  "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        attachments: [
          {
            color,
            title:      `${emoji} ${alarm.AlarmName}`,
            text:       alarm.NewStateReason,
            title_link: alarmUrl,
            footer:     `AWS ${alarm.AWSAccountId} · ${region}`,
          },
        ],
      }),
    });
  }
};
```

The alarm message links directly to the CloudWatch alarm in the console. One click from Slack to the relevant alarm view.

### Dashboard

```typescript
// sst.config.ts — CloudWatch dashboard
const stage = $app.stage;

const dashboard = new aws.cloudwatch.Dashboard("RunwayDashboard", {
  dashboardName: `Runway-${stage}`,
  dashboardBody: $jsonStringify({
    widgets: [
      // Row 1: Business metrics
      {
        type: "metric",
        x: 0, y: 0, width: 12, height: 6,
        properties: {
          title:  "Invoices Paid (24h)",
          view:   "timeSeries",
          stacked: false,
          metrics: [[
            "Runway/Business", "InvoicePaid",
            "Stage", stage,
            { stat: "Sum", period: 3600 },
          ]],
        },
      },
      {
        type: "metric",
        x: 12, y: 0, width: 12, height: 6,
        properties: {
          title:  "Email Send Rate",
          view:   "timeSeries",
          metrics: [
            ["Runway/Business", "EmailSent",   "Type", "invoice.paid.email",     { stat: "Sum", period: 900 }],
            ["Runway/Business", "EmailFailed",  "Type", "invoice.paid.email",    { stat: "Sum", period: 900 }],
            ["Runway/Business", "EmailSent",   "Type", "invoice.reminder.email", { stat: "Sum", period: 900 }],
          ],
        },
      },
      // Row 2: Infrastructure health
      {
        type: "metric",
        x: 0, y: 6, width: 12, height: 6,
        properties: {
          title:  "API Error Rate",
          view:   "timeSeries",
          metrics: [
            ["AWS/Lambda", "Errors",       "FunctionName", invoiceHandler.nodes.function.name, { stat: "Sum", period: 300 }],
            ["AWS/Lambda", "Throttles",    "FunctionName", invoiceHandler.nodes.function.name, { stat: "Sum", period: 300 }],
          ],
        },
      },
      {
        type: "metric",
        x: 12, y: 6, width: 12, height: 6,
        properties: {
          title:  "Invoice Handler Duration",
          view:   "timeSeries",
          metrics: [
            ["AWS/Lambda", "Duration", "FunctionName", invoiceHandler.nodes.function.name, { stat: "p99", period: 300 }],
            ["AWS/Lambda", "Duration", "FunctionName", invoiceHandler.nodes.function.name, { stat: "p50", period: 300 }],
          ],
        },
      },
    ],
  }),
});
```

The first row answers "is the product working?" The second row answers "is the infrastructure healthy?" Keep them visually separate — Lambda duration anomalies don't always cause business errors, but business metric anomalies almost always point to broken infrastructure.

### What Not to Alert On

An alert should mean: "you need to look at this right now." If alerts fire for things that sort themselves out, or fire multiple times for the same issue, or fire at 3am for something that can wait until morning, they stop meaning anything. Alert fatigue is real and it kills incident response.

Do not alert on:
- Individual 4xx errors (clients send bad requests — this is normal)
- Lambda cold starts (expected, usually sub-second, not actionable)
- Lambda throttles under 10 per minute (burstable, recovers without intervention)
- DynamoDB consumed capacity approaching provisioned (irrelevant with on-demand billing)
- Any metric you'd habitually snooze

Start with the four alarms in this chapter and add more only when a production incident reveals a specific gap that an alarm would have closed.

---

## 10.6 Distributed Tracing with X-Ray

X-Ray gives you end-to-end traces across Lambda, API Gateway, DynamoDB, and SQS. Enable it in `sst.config.ts`:

```typescript
// sst.config.ts — enable X-Ray on specific functions
const invoiceHandler = new sst.aws.Function("InvoiceHandler", {
  handler: "src/functions/invoices/list.handler",
  tracing: "active",
  link:    [table, emailQueue],
});
```

Or enable it globally for all functions using the transform option:

```typescript
// sst.config.ts — global X-Ray tracing
app.transform({
  function: (args) => {
    args.tracing = "active";
  },
});
```

With `tracing: "active"`, every Lambda invocation creates a trace segment. DynamoDB and SQS calls within the Lambda create sub-segments automatically — the AWS SDK instruments itself when X-Ray is enabled. You get timing breakdowns for the full request without any code changes.

### X-Ray Sampling Rules

By default, X-Ray samples 5% of requests. For a low-traffic service like a new Runway deployment, 5% means you'll see traces for about 1 in 20 invocations — enough to get latency data, but you'll miss most individual failures.

Configure a sampling rule that captures more of the interesting traffic:

```typescript
// sst.config.ts
const samplingRule = new aws.xray.SamplingRule("RunwaySamplingRule", {
  ruleName:     "RunwayDefaultRule",
  priority:     1000,
  reservoirSize: 5,      // Always sample this many req/second (reservoir)
  fixedRate:    0.10,    // Then sample 10% of remaining requests
  host:         "*",
  httpMethod:   "*",
  urlPath:      "*",
  serviceName:  "*",
  serviceType:  "*",
  resourceArn:  "*",
  version:      1,
});
```

`reservoirSize: 5` means the first 5 requests per second are always sampled regardless of `fixedRate`. This ensures you never completely miss a burst of errors even at high traffic. `fixedRate: 0.10` samples 10% of everything else.

For error investigation, you don't need to change sampling rules — X-Ray always records 100% of errored requests regardless of the sampling rate.

### Annotating Traces

Automatic instrumentation captures timings. Manual annotations add business context that makes traces searchable.

```typescript
// src/functions/invoices/list.ts
import * as AWSXRay from "aws-xray-sdk-core";

export const handler: ValidatedHandler = async (event, context) => {
  const segment = AWSXRay.getSegment();

  // Annotations are indexed — you can filter traces by these in the X-Ray console
  segment?.addAnnotation("workspaceId", workspaceId);
  segment?.addAnnotation("userId",      userId);

  const subsegment = segment?.addNewSubsegment("listInvoices");
  try {
    const invoices = await listInvoices(workspaceId);
    // Metadata is not indexed but shows in the trace detail view
    subsegment?.addMetadata("count", invoices.length);
    return { statusCode: 200, body: JSON.stringify({ invoices }) };
  } finally {
    subsegment?.close();
  }
};
```

With `workspaceId` as an annotation, you can filter the X-Ray trace list to show only requests from a specific workspace — essential for investigating a customer complaint.

### Reading a Trace Map

The X-Ray service map in the AWS console shows nodes (services) and edges (calls between them). A healthy Runway service map looks like:

```
[API Gateway] → [Invoice Lambda] → [DynamoDB]
                                 ↘ [SQS: EmailQueue]
                                       ↓
                               [Email Worker Lambda]
```

Thick edges mean high call volume. Red edges mean errors. Slow edges are highlighted in amber. Click any edge to see the call breakdown, then click any node to see its error rate, latency distribution, and sample traces.

When investigating an incident: red edges first, then slowest traces on the highlighted edges, then read the full logs for the request IDs in those traces.

---

## 10.7 Lambda Insights

Lambda Insights is an enhanced monitoring feature from AWS that captures performance metrics not available in the standard Lambda metrics: CPU usage, memory utilisation, initialisation duration (cold start time), and network I/O.

Enable it in `sst.config.ts`:

```typescript
// sst.config.ts
const invoiceChaser = new sst.aws.Function("InvoiceChaser", {
  handler:  "src/jobs/invoice-chaser.handler",
  timeout:  "5 minutes",
  tracing:  "active",
  layers: [
    // Lambda Insights extension — check the AWS docs for the latest ARN in your region
    "arn:aws:lambda:eu-west-1:580247275435:layer:LambdaInsightsExtension:38",
  ],
  // Lambda Insights requires CloudWatch permissions
  transform: {
    function: (args) => {
      args.policies = [
        ...(args.policies ?? []),
        "arn:aws:iam::aws:policy/CloudWatchLambdaInsightsExecutionRolePolicy",
      ];
    },
  },
});
```

Once enabled, Lambda Insights adds a `/aws/lambda-insights` log group. These logs feed a pre-built dashboard in CloudWatch (Lambda → Functions → select function → Monitoring → Lambda Insights).

The metrics worth watching:

**`init_duration`** is cold start time. If this is consistently above 500ms, your Lambda bundle is too large, or you're importing modules you don't need on the hot path. Check for accidental `import *` patterns.

**`memory_utilisation`** as a percentage of configured memory. If consistently below 30%, you've over-provisioned. If regularly above 90%, increase the memory allocation (which also increases CPU, since Lambda allocates CPU proportionally to memory).

**`rx_bytes` and `tx_bytes`** are network I/O. An unexpectedly high `tx_bytes` on an invoice handler could indicate you're returning unnecessarily large payloads.

Lambda Insights adds approximately $0.20/million invocations to your bill — negligible for production workloads, worth monitoring in staging.

---

## 10.8 Debugging a Production Incident

All of the observability tooling in this chapter is only useful if you know how to use it under pressure. Here's the playbook for the most common Runway incident: an elevated error rate.

### Scenario: API errors spiking for one workspace

**Step 1 — Confirm the scope from the alarm.**

The CloudWatch alarm fires: `production-RunwayApiErrors` in ALARM state. Open the alarm in the console. Check the time range and the metric graph. Is it one function or multiple? Is it sustained or a brief spike?

**Step 2 — Find the errors in Logs Insights.**

Open CloudWatch Logs Insights. Select the log group for the relevant function. Run the saved "Recent errors" query, then add a filter:

```sql
fields @timestamp, level, message, workspaceId, requestId, error
| filter level = "ERROR"
| stats count() as errorCount by workspaceId
| sort errorCount desc
| limit 20
```

If the errors are concentrated in one workspace, that's a data problem. If they're spread across workspaces, it's a code or infrastructure problem.

**Step 3 — Get a full log trace for a failing request.**

Take one of the `requestId` values from the query results. Run:

```sql
fields @timestamp, level, message, workspaceId
| filter requestId = "abc123-..."
| sort @timestamp asc
```

This shows every log line emitted during that specific request in chronological order. You'll usually see where the failure occurs.

**Step 4 — Find the X-Ray trace.**

Copy the `requestId` to X-Ray → Search traces → Filter by annotation or trace ID. The trace shows exactly which downstream call failed (DynamoDB? SQS? Aurora?) and how long each step took.

**Step 5 — Reproduce locally if needed.**

If the trace shows a DynamoDB `ConditionalCheckFailedException`, or a Prisma query timing out, or a Zod parse failure — you now have enough context to reproduce it in `sst dev`. The combination of structured logs and X-Ray traces should give you the workspace ID, the request payload shape, and the exact line the error originated.

**Step 6 — Fix and deploy.**

With the cause identified: fix, push to main, watch the error rate drop in the dashboard. Confirm in Logs Insights that the errors have stopped. Close the alarm manually once you're satisfied (`Set alarm state → OK`).

The full investigation — alarm fires to root cause identified — should take under 10 minutes for anything with good observability in place. Without it, the same investigation takes hours of guesswork.

---

## 10.9 The Minimum Viable Observability Setup

If you're starting from scratch, here's what to ship on day one. Each item is an hour or less.

**Structured JSON logging.** The `Logger` class from section 10.2. Add `requestId` from Lambda context to every handler. This alone cuts incident investigation time by 80%.

**Log retention.** Set 30-day retention on all production log groups. Without this, the bill grows indefinitely. Fifteen minutes to add the `withRetention` calls in `sst.config.ts`.

**X-Ray tracing.** One line in `sst.config.ts`: `app.transform({ function: (args) => { args.tracing = "active"; } })`. Immediate distributed trace visibility, no code changes required.

**Four CloudWatch alarms:**
- API Lambda error rate (≥ 5 in 5 minutes, two consecutive periods)
- Email DLQ depth (≥ 1 message)
- Invoice chaser any error
- Any Lambda duration p99 approaching 80% of timeout

**Slack routing for alarms.** The `SlackAlerter` Lambda from section 10.5. Alarms that fire into email get missed. Alarms that fire into the team Slack channel get fixed.

**Two saved Logs Insights queries:**
- "Recent errors" — `filter level = "ERROR"`, last hour, sorted by timestamp
- "Cron job history" — `filter message = "complete"`, last 7 days

**A single dashboard.** Row 1: business metrics (invoices paid, email send rate). Row 2: infrastructure metrics (error rate, p99 latency). Ten widgets total. This is enough to see whether the system is healthy at a glance.

That's eight items. Two to four hours of work. It's the difference between "the invoice chaser silently stopped running three days ago" and "the invoice chaser alarm fired at 9:01am and was resolved by 9:18am."

Add Lambda Insights, custom sampling rules, per-workspace dashboards, and anomaly detection as the product grows. The foundation carries everything else.

---

## Where We Are

Runway is complete. The full architecture, from first request to last observable metric:

```
[GitHub Actions CI/CD]
  └─ Type check → Migrate → Deploy
     (preview per PR, staging on merge, production on manual approval)

[Cognito User Pool]
  └─ JWT tokens, verified at API Gateway

[API Gateway V2]
  └─ Routes → Lambda handlers (thin, typed, structured logging, X-Ray traced)

[Lambda handlers]
  ├─ [DynamoDB] — single-table, typed repositories            (Ch. 4)
  ├─ [Aurora Serverless v2 + Prisma] — relational reporting   (Ch. 5)
  ├─ [S3] — file uploads via presigned URLs                   (Ch. 6)
  ├─ [SQS] — reliable async work, DLQ monitoring              (Ch. 8)
  └─ [EventBridge] — domain event fan-out                     (Ch. 8)

[Background layer]
  ├─ Cron (invoice chaser, daily 9am)
  └─ S3 event processor (thumbnails, page count)

[Observability]
  ├─ CloudWatch Logs — structured JSON, saved Insights queries
  ├─ X-Ray — distributed traces, workspace-level annotations
  ├─ CloudWatch Metrics — Runway/Business namespace via EMF
  ├─ CloudWatch Dashboards — business + infrastructure rows
  └─ CloudWatch Alarms → SNS → Slack
```

Runway is a real SaaS. Type-safe end to end, from the API contract to the database schema to the queue message body. Deployed automatically on every push to `main`. Monitored in production with alerts that mean something.

This is what shipping TypeScript on AWS looks like when it's done properly. Not the tutorial version. The version you'd actually run a business on.

---

## What's Next

The appendices cover the things that didn't fit cleanly into the chapter flow:

- **Appendix A** — AWS account setup done right (root account lockdown, billing alerts, MFA)
- **Appendix B** — Common error reference (`ConditionalCheckFailedException`, connection pool exhaustion, IAM permission errors — causes and fixes)
- **Appendix C** — SST component cheat sheet
- **Appendix D** — TypeScript patterns reference (`satisfies`, discriminated unions, branded types)
- **Appendix E** — Cost estimation at 1k, 10k, and 100k users

If you've read this far and built Runway alongside it: you know more about running TypeScript on AWS than most teams who've been doing it for years. The gap isn't knowledge — it's taking the time to get it right.

Ship it.

---

> **The code for this chapter** is available on the `chapter-10` branch of the companion repository.
