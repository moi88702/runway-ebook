# Chapter 9: CI/CD and Deployment Automation

Runway works. The code runs locally. The SST dev environment tails logs in real time. Everything looks good.

Now deploy it to production, reliably, repeatedly, without manually running `sst deploy` from your laptop. Without wondering whether the Prisma migration ran before or after the Lambda updated. Without a colleague accidentally deploying their in-progress feature branch to the live environment.

That's what this chapter is about. A deployment pipeline that makes shipping Runway's weekly releases a non-event — something that happens in the background while you're already working on the next thing.

---

## 9.1 Repository Structure for an SST Project

Before automating deployments, the repo needs a structure that works for it.

The fundamental question: monorepo or polyrepo?

For Runway, monorepo. Everything lives in one repository: infrastructure (`sst.config.ts`), Lambda functions (`src/`), the frontend (if there is one), shared types, and config. Deployments touch all of these simultaneously, so keeping them together avoids the "what version of the shared types package does production use?" problem.

A clean Runway repo structure:

```
runway/
├── sst.config.ts              # Infrastructure definition
├── package.json               # Root package.json (workspaces if needed)
├── tsconfig.json              # Base TypeScript config
├── .env.example               # Safe template — never commit .env
├── .gitignore
│
├── src/
│   ├── functions/             # Lambda handler entry points (thin)
│   │   ├── api/               # HTTP API handlers
│   │   ├── workers/           # SQS/EventBridge workers
│   │   └── jobs/              # Cron + Step Function Lambdas
│   ├── repositories/          # Data access layer (DynamoDB, Aurora)
│   ├── lib/                   # Shared utilities (queue, events, email)
│   ├── types/                 # Shared TypeScript types and Zod schemas
│   └── test/                  # Test helpers and setup
│
├── prisma/
│   ├── schema.prisma
│   └── migrations/            # Never delete these
│
├── scripts/
│   ├── run-migrations.ts      # CI migration invoker
│   └── seed-preview.ts        # Preview environment seeder
│
└── .github/
    └── workflows/
        ├── deploy.yml         # Main deploy pipeline
        └── pr-checks.yml      # Type check + lint on every PR
```

A few conventions worth adopting early:

**Handlers are thin.** A handler's job is to parse the event, call a repository or service, and return a response. Business logic lives in `src/repositories/` and eventually `src/services/` as the codebase grows. Thin handlers are trivially unit-testable without Lambda event machinery.

**Types are shared.** Zod schemas in `src/types/` are imported by both handlers and workers. The same `InvoicePaidEvent` type that the invoice handler emits is what the analytics worker validates on arrival. One source of truth.

**Never commit `.env`.** Add it to `.gitignore` on day one. Use `.env.example` with placeholder values to document what variables are needed. Actual secrets live in SST secrets (`sst secret set`).

### The `.env.example` File

The `.env.example` file is documentation. It tells every developer (and your CI pipeline) what environment variables exist and what they're for:

```bash
# .env.example
# Copy to .env and fill in values for local development.
# Never commit .env — it is in .gitignore.
# Production values are stored in SST secrets (SSM Parameter Store).

# AWS region for all SDK clients
AWS_REGION=eu-west-1

# SST stage — controls which stack you deploy to
# Values: dev, staging, production, or pr-<number>
SST_STAGE=dev

# Local Aurora/RDS connection (used by Prisma in dev only)
# In production, Prisma connects via the SST Resource binding
DATABASE_URL=postgresql://runway:password@localhost:5432/runway_dev

# Email provider (Mailgun)
# In production, set via: npx sst secret set MailgunApiKey "..."
MAILGUN_API_KEY=

# Stripe webhook secret for local testing via Stripe CLI
STRIPE_WEBHOOK_SECRET=

# Cognito (populated automatically by sst dev in local dev)
COGNITO_USER_POOL_ID=
COGNITO_CLIENT_ID=
```

Check this file into the repo. It is a contract: if a variable exists in `.env.example`, every environment (dev, staging, production, CI) must have a value for it.

---

## 9.2 GitHub Actions Pipeline

The deploy pipeline has one job: when code is pushed to `main`, deploy to production. When a PR is opened, deploy to a preview environment.

First, the credentials problem.

### OIDC over Access Keys

The naive approach: create an IAM user, generate access keys, store them in GitHub Actions secrets as `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. This works. It's also a security liability — long-lived credentials that can be leaked, rotated incorrectly, or forgotten.

The right approach: OIDC. GitHub and AWS have a trust relationship. GitHub can prove "this workflow is running in the `yourorg/runway` repository on the `main` branch," and AWS grants temporary credentials in response. No long-lived keys anywhere.

OIDC setup is a one-time piece of infra. SST v3 uses Pulumi under the hood, and the cleanest way to add AWS IAM resources that don't have a direct SST component is via the `aws.iam` Pulumi provider directly in `sst.config.ts`:

```typescript
// sst.config.ts — add to a dedicated "tools" or "infra" stage

const githubOrg  = "your-org";
const githubRepo = "runway";

const oidcProvider = new aws.iam.OpenIdConnectProvider("GithubOidcProvider", {
  url: "https://token.actions.githubusercontent.com",
  clientIdLists: ["sts.amazonaws.com"],
  thumbprintLists: ["6938fd4d98bab03faadb97b34396831e3780aea1"],
});

const deployRole = new aws.iam.Role("GithubDeployRole", {
  assumeRolePolicy: oidcProvider.arn.apply((arn) =>
    JSON.stringify({
      Version: "2012-10-17",
      Statement: [
        {
          Effect: "Allow",
          Principal: { Federated: arn },
          Action: "sts:AssumeRoleWithWebIdentity",
          Condition: {
            StringLike: {
              "token.actions.githubusercontent.com:sub":
                `repo:${githubOrg}/${githubRepo}:*`,
            },
            StringEquals: {
              "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
            },
          },
        },
      ],
    })
  ),
});

new aws.iam.RolePolicyAttachment("GithubDeployRolePolicy", {
  role: deployRole.name,
  policyArn: "arn:aws:iam::aws:policy/AdministratorAccess",
});

return {
  deployRoleArn: deployRole.arn,
};
```

Run this once with `npx sst deploy --stage tools`. The output prints the `deployRoleArn`. Add it to GitHub → Settings → Secrets and variables → Actions → `AWS_DEPLOY_ROLE_ARN`.

> **Why AdministratorAccess?** SST needs broad permissions to create CloudFormation stacks, IAM roles for Lambda execution, DynamoDB tables, S3 buckets, Cognito pools, and everything else in your infrastructure. `AdministratorAccess` is the practical starting point. Tighten the policy once the project is stable — the exact permissions SST needs are documented in the SST docs under "IAM permissions."

### The Deploy Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - main
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - main

permissions:
  id-token: write      # Required for OIDC
  contents: read
  pull-requests: write # To comment preview URLs on PRs

jobs:
  # ── PR checks (no AWS credentials needed) ──────────────────────────────────
  pr-checks:
    name: Type check + tests
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request' && github.event.action != 'closed'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - run: npx tsc --noEmit
      - run: npx vitest run
        env:
          AWS_REGION: eu-west-1   # Required by AWS SDK in unit tests

  # ── Preview deploy (one per PR) ────────────────────────────────────────────
  preview:
    name: Deploy preview
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request' && github.event.action != 'closed'
    needs: pr-checks
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: eu-west-1
          role-session-name: Preview-PR${{ github.event.number }}-${{ github.run_id }}
      - name: Deploy preview
        id: deploy
        run: |
          npx sst deploy --stage pr-${{ github.event.number }} 2>&1 | tee deploy.log
          API_URL=$(grep -oP 'ApiEndpoint:\s+\K\S+' deploy.log || echo "")
          echo "api_url=$API_URL" >> "$GITHUB_OUTPUT"
      - name: Seed preview data
        run: npx tsx scripts/seed-preview.ts
        env:
          SST_STAGE: pr-${{ github.event.number }}
      - name: Comment preview URL
        uses: actions/github-script@v7
        with:
          script: |
            const url = "${{ steps.deploy.outputs.api_url }}";
            const body = url
              ? `🚀 **Preview deployed:** ${url}\n\nStage: \`pr-${{ github.event.number }}\``
              : `✅ Preview deployed (stage: \`pr-${{ github.event.number }}\`)`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body,
            });

  # ── Tear down preview when PR closes ─────────────────────────────────────
  teardown:
    name: Tear down preview
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: eu-west-1
      - run: npx sst remove --stage pr-${{ github.event.number }} --force

  # ── Production deploy (push to main) ──────────────────────────────────────
  deploy:
    name: Deploy to production
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: production  # Requires manual approval if configured
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - run: npx tsc --noEmit
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: eu-west-1
          role-session-name: Deploy-${{ github.sha }}
      - name: Run database migrations
        run: npx tsx scripts/run-migrations.ts
        env:
          SST_STAGE: production
      - name: Deploy to production
        run: npx sst deploy --stage production
```

The checks job runs first and independently — it doesn't need AWS credentials, so it can run in parallel with any other credential-free work. The preview deploy only starts if checks pass (`needs: pr-checks`). The production deploy runs only on `push` to `main` — not on PR events.

### The PR Checks Workflow

A lighter, standalone workflow for PRs that don't need deployment:

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    branches:
      - main

jobs:
  check:
    name: Type check + lint + test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - run: npx tsc --noEmit
      - run: npx eslint src --max-warnings 0
      - run: npx vitest run
        env:
          AWS_REGION: eu-west-1
```

This workflow exists separately from `deploy.yml` so you can require it as a branch protection check (covered in section 9.8) without coupling it to the deploy machinery.

> **Why `AWS_REGION` in unit tests?** The AWS SDK clients in `src/lib/` are instantiated with `new DynamoDBClient({})`. The SDK reads `AWS_REGION` at construction time. Without it, you'll see "Region is missing" errors in tests even if no real AWS calls happen, because the client constructor validates config immediately. Setting `AWS_REGION: eu-west-1` in the test environment is the fix — no credentials needed, just a valid region string.

---

## 9.3 Preview Environments

SST stages are isolated AWS stacks. `sst deploy --stage pr-42` creates a completely separate Runway environment: its own Lambda functions, its own DynamoDB table, its own API Gateway endpoint. Every PR gets its own playground.

### Cost Management for Preview Environments

Aurora Serverless v2 has a minimum ACU of 0.5, which runs at approximately $0.06/hour even when completely idle. For ten open PRs each with an Aurora cluster, that's $14/day for databases that are barely being touched.

Two practical options:

**Skip Aurora in previews.** Use only DynamoDB in preview environments. Aurora deploys only in `staging` and `production`. Most PR testing is about Lambda behaviour and API responses — full Aurora fidelity isn't needed for that.

**Shared preview Aurora cluster.** Create one Aurora cluster in a long-lived `preview` stage. Each PR environment connects to its own database name (`runway_pr_42`) within that cluster. Less isolation than fully separate clusters, but substantially cheaper.

The stage-conditional approach in `sst.config.ts`:

```typescript
// sst.config.ts
const stage     = $app.stage;
const isProd    = stage === "production";
const isStaging = stage === "staging";
const isPreview = stage.startsWith("pr-");

// Aurora only in staging and production
const needsAurora = isProd || isStaging;

let db: sst.aws.Aurora | undefined;

if (needsAurora) {
  const vpc = new sst.aws.Vpc("RunwayVpc");

  db = new sst.aws.Aurora("RunwayDb", {
    engine:  "postgresql15.4",
    scaling: {
      min: "0.5 ACU",
      max: isProd ? "8 ACU" : "2 ACU",
    },
    vpc,
  });
}

// DynamoDB is always available — serverless, free when idle
const table = new sst.aws.Dynamo("RunwayTable", {
  fields:   { PK: "string", SK: "string", gsi1pk: "string", /* ... */ },
  primaryIndex: { hashKey: "PK", rangeKey: "SK" },
  // ... other indices
});
```

Handlers that use Aurora check the `db` binding — if it's undefined (preview stage), they return early or fall back to DynamoDB-only logic. In practice this usually means financial reporting endpoints aren't available in previews, which is acceptable.

### Seeding Preview Data

A preview environment with an empty database isn't useful for manual testing. The seed script creates a workspace, a client, and a project so there's something to work with:

```typescript
// scripts/seed-preview.ts
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";
import { Resource } from "sst";

const client    = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

async function seed() {
  const workspaceId = "WORKSPACE#preview-seed";
  const clientId    = "CLIENT#preview-client";
  const projectId   = "PROJECT#preview-project";
  const now         = new Date().toISOString();

  // Workspace
  await docClient.send(new PutCommand({
    TableName: Resource.RunwayTable.name,
    Item: {
      PK:        workspaceId,
      SK:        "WORKSPACE",
      ownerId:   "USER#preview-user",
      name:      "Preview Workspace",
      plan:      "pro",
      createdAt: now,
    },
  }));

  // Client
  await docClient.send(new PutCommand({
    TableName: Resource.RunwayTable.name,
    Item: {
      PK:          workspaceId,
      SK:          clientId,
      gsi1pk:      clientId,
      name:        "Acme Corp",
      email:       "acme@example.com",
      workspaceId: "preview-seed",
      createdAt:   now,
    },
  }));

  // Project
  await docClient.send(new PutCommand({
    TableName: Resource.RunwayTable.name,
    Item: {
      PK:          workspaceId,
      SK:          projectId,
      gsi1pk:      projectId,
      gsi2pk:      projectId,
      name:        "Website Redesign",
      clientId:    "preview-client",
      status:      "active",
      workspaceId: "preview-seed",
      createdAt:   now,
    },
  }));

  console.log("Preview seed complete");
}

seed().catch((err) => {
  console.error("Seed failed:", err);
  process.exit(1);
});
```

The script uses `Resource.RunwayTable.name` to resolve the correct DynamoDB table name for the current stage — so `pr-42` seeds into `pr-42`'s table, not the production one. This requires `SST_STAGE` to be set when the script runs, which the deploy workflow handles.

---

## 9.4 Environment Promotion

Runway uses three stages: `dev` (local SST dev), `staging` (continuous deployment from `main`), `production` (manual promotion from `staging`).

The flow:

```
PR branch → preview environment (auto-deployed on PR open)
    ↓ merged to main
staging (auto-deployed on push to main)
    ↓ manual approval
production (deployed after manual sign-off)
```

To add a manual approval gate for production, use GitHub Environments:

1. In your GitHub repo: Settings → Environments → New environment → `production`
2. Add "Required reviewers" — the people who must approve production deploys
3. The workflow references it with `environment: production` — GitHub pauses before that job and sends a notification

```yaml
  deploy-staging:
    name: Deploy to staging
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      # ... same steps as production deploy, SST_STAGE: staging

  deploy-production:
    name: Deploy to production
    runs-on: ubuntu-latest
    needs: deploy-staging           # Staging must succeed first
    environment: production         # Pauses for manual approval
    steps:
      # ... same steps, SST_STAGE: production
```

### Rollback Strategy

SST doesn't have a built-in rollback command. When production breaks:

**The fast path is a revert commit.** `git revert HEAD && git push`. The pipeline redeploys the previous version automatically. This takes as long as a normal deploy — typically 2–4 minutes.

**For Lambda-only rollbacks**, Lambda maintains a version history. If the infrastructure didn't change but a code change caused failures, you can point the function alias to the previous version in the AWS console — this takes 30 seconds while the revert deploy runs.

**For database rollbacks**, the story is harder and this is exactly why the expand/contract pattern from Chapter 5 matters. If you deploy a migration that removes a column the previous Lambda version still reads, you cannot cleanly roll back the Lambda without rolling back the database too. The principle: never deploy a destructive migration in the same push as the code that stops using the old column. Always deploy in two pushes: add new column + new code, then remove old column once confirmed stable.

### RUNBOOK.md

Keep a `RUNBOOK.md` in the repository with the exact commands for common incidents. You will need it at 11pm on a Friday and you will not be in a state to reason clearly.

```markdown
# Runway Production Runbook

## Emergency Contacts

- AWS account root: (stored in 1Password under "AWS Runway Production")
- On-call: [your name / your team]

---

## Rollback a Bad Deploy

**Option 1 — Revert commit (recommended)**

```bash
git revert HEAD
git push origin main
```

Wait for the pipeline. Takes ~4 minutes.

**Option 2 — Lambda alias rollback (Lambda-only, instant)**

1. Go to AWS Console → Lambda → [function name] → Aliases
2. Edit the `live` alias → point to the previous version number
3. Takes effect immediately for new invocations

**Option 3 — Manual SST deploy from specific commit**

```bash
git checkout <commit-sha>
npx sst deploy --stage production
git checkout main
```

Use this if the revert commit approach isn't working and you need to pin a specific version.

---

## Run Database Migrations Manually

```bash
AWS_PROFILE=runway-production npx tsx scripts/run-migrations.ts
```

Or invoke the migration Lambda directly:

```bash
aws lambda invoke \
  --function-name runway-production-MigrationRunner \
  --log-type Tail \
  --query 'LogResult' \
  --output text \
  response.json | base64 -d
```

---

## Check What's Deployed

```bash
# Show all SST stacks and their status
npx sst status --stage production

# List Lambda function versions
aws lambda list-versions-by-function \
  --function-name runway-production-ApiHandler \
  --query 'Versions[-3:].[Version,LastModified]' \
  --output table
```

---

## Tear Down a Preview Environment

```bash
npx sst remove --stage pr-<number> --force
```

If the remove hangs (usually because of non-empty S3 buckets):

```bash
# Empty the bucket first
aws s3 rm s3://runway-pr-<number>-deliverables --recursive

# Then retry
npx sst remove --stage pr-<number> --force
```

---

## Secret Leaked — Immediate Response

1. Rotate the credential at the provider (Mailgun, Stripe, etc.) — do this before anything else
2. Update in SST: `npx sst secret set MailgunApiKey "new-key" --stage production`
3. Redeploy to pick up the new value: `npx sst deploy --stage production`
4. Scrub from git history if it was committed: `git filter-repo --path .env --invert-paths`
5. Force-push all branches: `git push --force --all`

---

## DynamoDB Table Getting Large

Check item count and table size:

```bash
aws dynamodb describe-table \
  --table-name runway-production-RunwayTable \
  --query 'Table.[TableSizeBytes,ItemCount]' \
  --output text
```

Items with a `ttl` attribute are automatically deleted by DynamoDB's TTL process within 48 hours of expiry. No action needed unless the table is growing unexpectedly.
```

The RUNBOOK.md is a living document. Every time you fix a production incident, add the steps you took. Future-you will thank past-you.

---

## 9.5 Secrets Management in CI

Runway has secrets: database credentials, Mailgun API keys, Stripe webhook secrets. They need to be available in production Lambdas without appearing in the codebase or the CI logs.

### SST Secrets

SST's secrets system stores values in AWS SSM Parameter Store, encrypted with KMS. Lambda functions linked to a secret read the value at runtime via the `Resource` helper — no plaintext environment variables that show up in logs or CloudFormation templates.

```bash
# Set a secret for the production stage
npx sst secret set MailgunApiKey "key-xxxxx" --stage production

# List secrets (shows names, not values)
npx sst secret list --stage production

# Remove a secret
npx sst secret remove MailgunApiKey --stage production
```

Wire it in `sst.config.ts`:

```typescript
// sst.config.ts
const mailgunKey        = new sst.Secret("MailgunApiKey");
const stripeWebhookKey  = new sst.Secret("StripeWebhookSecret");

const sharedFunctionConfig = {
  link: [table, emailQueue, eventBus, mailgunKey, stripeWebhookKey],
  environment: { SST_STAGE: $app.stage },
};
```

Read in a Lambda:

```typescript
// src/workers/email.ts
import { Resource } from "sst";

// Resource.MailgunApiKey.value resolves at runtime from SSM
const mailgunClient = new Mailgun({
  apiKey:  Resource.MailgunApiKey.value,
  domain:  "mail.runwayapp.com",
});
```

The value is fetched once on Lambda cold start. If you rotate the secret, you need to redeploy for the new value to take effect — SST doesn't hot-reload secrets.

### Setting Secrets Before First Deploy

Secrets must exist in SSM before the first deploy that references them. If you deploy before setting a secret, SST will fail with a "secret not found" error during the stack build.

The workflow:

```bash
# Set all secrets before first production deploy
npx sst secret set MailgunApiKey   "..."  --stage production
npx sst secret set StripeWebhookSecret "..." --stage production

# Then deploy
npx sst deploy --stage production
```

For a staging environment, set the same secrets with `--stage staging`. For preview environments, set secrets once in a shared `preview` stage and reference them, or accept that preview environments won't have email/payment secrets (which is usually fine — those integrations don't need to work in PRs).

### CI-Injected Secrets

If secrets change frequently or you want CI to manage them, inject from GitHub Actions secrets:

```yaml
- name: Set SST secrets
  run: |
    npx sst secret set MailgunApiKey "${{ secrets.MAILGUN_API_KEY }}" --stage production
    npx sst secret set StripeWebhookSecret "${{ secrets.STRIPE_WEBHOOK_SECRET }}" --stage production
```

Run this step before `sst deploy`. The values come from GitHub Actions secrets (Settings → Secrets and variables → Actions), which are encrypted at rest and redacted from logs.

### If a Secret Leaks

If credentials are committed to the repo or appear in logs:

1. Rotate immediately at the provider — do this first, before anything else
2. Update in SST: `npx sst secret set <SecretName> "new-value" --stage production`
3. Redeploy to pick up the new value
4. Scrub from git history if committed: `git filter-repo --path .env --invert-paths && git push --force --all`
5. Check whether the exposed credential was used (Mailgun, Stripe, and Cognito all have access logs)

GitHub has secret scanning enabled by default on public repos and will email you if it detects known secret patterns in commits. This is a useful safety net — but act on it immediately, don't treat it as a delayed notification.

---

## 9.6 Database Migrations in CI

Prisma migrations in production deserve their own section because the wrong approach causes downtime.

The wrong approach: running `prisma migrate deploy` inside the Lambda handler on cold start. Problems:

- Multiple Lambda instances cold-starting simultaneously all try to run migrations at once
- If the migration takes more than a few seconds, the Lambda times out
- Failed migrations leave the database in a state inconsistent with the code that just deployed

The right approach: run migrations as a separate CI step, before the Lambda deploy. If the migration fails, the deploy stops. The old Lambda stays live, the old schema stays in the database. Everything is consistent.

### The Migration Runner Script

```typescript
// scripts/run-migrations.ts
import { LambdaClient, InvokeCommand, InvokeCommandInput } from "@aws-sdk/client-lambda";

const client = new LambdaClient({ region: process.env.AWS_REGION ?? "eu-west-1" });

// The function name follows SST's naming convention: <stage>-<stack>-<resourceName>
const stage        = process.env.SST_STAGE ?? "production";
const functionName = `${stage}-runway-MigrationRunner`;

async function runMigrations() {
  console.log(`Running migrations via ${functionName}...`);

  const params: InvokeCommandInput = {
    FunctionName:   functionName,
    InvocationType: "RequestResponse",
    LogType:        "Tail",
  };

  const response = await client.send(new InvokeCommand(params));

  // Tail logs are base64-encoded
  if (response.LogResult) {
    const logs = Buffer.from(response.LogResult, "base64").toString("utf-8");
    console.log("=== Migration Lambda logs ===");
    console.log(logs);
    console.log("=============================");
  }

  if (response.FunctionError) {
    console.error(`Migration Lambda error: ${response.FunctionError}`);
    process.exit(1);
  }

  const payload = response.Payload
    ? JSON.parse(Buffer.from(response.Payload).toString("utf-8"))
    : null;

  if (!payload?.success) {
    console.error("Migration reported failure:", payload);
    process.exit(1);
  }

  console.log("Migrations complete:", payload.message);
}

runMigrations().catch((err) => {
  console.error("Migration runner failed:", err);
  process.exit(1);
});
```

> **How does `Resource.MigrationRunner.name` resolve in CI?** It doesn't — the script uses the SST naming convention directly (`${stage}-runway-MigrationRunner`) rather than `Resource`. The `Resource` helper reads from the `.sst/` state directory, which exists locally but isn't present in a fresh CI checkout. For CI scripts that need to reference deployed resources by name, use the naming convention directly or read SST outputs from a file you store between deploy steps.

### Zero-Downtime Migrations

The expand/contract pattern applies in full here. A CI-safe migration sequence:

**Step 1 — Expand (add, never remove):** Add the new column as nullable. The existing Lambda ignores it. Deploy the migration; the Lambda deploy is a no-op.

**Step 2 — Dual-write:** Deploy a Lambda that writes both the old column and the new column. Reads from the old column. No breaking change.

**Step 3 — Backfill:** Run a one-off script (or a separate Lambda) to populate the new column for existing rows. This can be a separate PR.

**Step 4 — Switch reads:** Deploy a Lambda that reads from the new column, falls back to old if null. Writes to both.

**Step 5 — Contract (remove old column):** Once all rows have the new column populated and the Lambda no longer reads the old one, drop it. This is a separate deploy, not bundled with anything else.

The rule: never include a destructive schema change in the same deploy as a code change that depends on the new schema. Always leave a deploy buffer between them.

---

## 9.7 Branch Protection and Required Checks

A CI pipeline is only useful if it's enforced. Without branch protection, a developer can push directly to `main` and bypass everything you've set up.

Configure branch protection in GitHub: Settings → Branches → Add rule → Branch name pattern: `main`.

Required settings:

**Require a pull request before merging.** No direct pushes to `main`. All changes go through a PR.

**Require status checks to pass before merging.** Select the jobs from your workflows that must pass:
- `Type check + lint + test` (from `pr-checks.yml`)
- Any other checks you want enforced

**Require branches to be up to date before merging.** Prevents a PR from merging if `main` has moved on since the checks ran.

**Do not allow bypassing the above settings.** Even repository administrators must follow the rules.

The result: the only way code reaches `main` is through a PR with a passing type check, passing tests, and at least one approval. The only way it reaches production is through that pipeline.

### CODEOWNERS

Add a `CODEOWNERS` file to require specific reviewers for specific paths:

```
# .github/CODEOWNERS

# Infrastructure changes require explicit review
sst.config.ts               @your-org/infrastructure
.github/workflows/          @your-org/infrastructure

# Database migrations require explicit review
prisma/migrations/          @your-org/backend
```

GitHub automatically requests review from the listed owners when a PR touches those files. A migration that hasn't been reviewed by someone who understands the schema doesn't merge.

---

## 9.8 Making the Pipeline Fast

A slow pipeline is a pipeline that gets bypassed. If checks take 8 minutes, developers find workarounds. Target under 3 minutes for PR checks, under 5 minutes for deploys.

### Cache Dependencies

The `setup-node` action with `cache: "npm"` caches the npm cache directory between runs. This saves 30–60 seconds on `npm ci` for a project with a moderate number of dependencies. It's one line and always worth it:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: "22"
    cache: "npm"   # Caches ~/.npm between runs
```

For monorepos with workspaces, add the cache dependency path:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: "22"
    cache: "npm"
    cache-dependency-path: "**/package-lock.json"
```

### Parallelize Independent Jobs

Type checking and tests don't depend on each other. Run them in parallel:

```yaml
jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22", cache: "npm" }
      - run: npm ci
      - run: npx tsc --noEmit

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22", cache: "npm" }
      - run: npm ci
      - run: npx vitest run
        env: { AWS_REGION: eu-west-1 }

  deploy:
    needs: [typecheck, test]  # Both must pass
    # ...
```

This cuts the serial check time roughly in half if typecheck and tests take similar durations.

### Skip Unnecessary Deploys

Not every file change needs a full deploy. Use path filters to skip the deploy job when only documentation changes:

```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - "**.md"
      - ".github/CODEOWNERS"
      - "docs/**"
```

Be conservative with this — it's easy to accidentally exclude a path that does affect the deployed output. When in doubt, deploy.

### Vitest in CI

Vitest is fast by default, but a few configuration choices keep it fast in CI:

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    poolOptions: {
      threads: {
        // CI machines typically have 2 CPUs — match this
        maxThreads: 2,
        minThreads: 1,
      },
    },
    // Don't watch in CI
    watch: false,
  },
});
```

Add `--reporter=verbose` in CI to get more output when tests fail:

```yaml
- run: npx vitest run --reporter=verbose
  env:
    AWS_REGION: eu-west-1
```

---

## 9.9 Failure Notifications

A failed deploy needs to reach a human quickly. GitHub sends email notifications by default, but email is easy to miss. Wire up Slack or another channel for deploy failures.

GitHub Actions has a built-in notification option, but for more control, use a conditional step that runs only on failure:

```yaml
  deploy:
    runs-on: ubuntu-latest
    steps:
      - # ... all deploy steps

      - name: Notify on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            // Post a comment on the commit that triggered the failure
            github.rest.repos.createCommitComment({
              owner: context.repo.owner,
              repo:  context.repo.repo,
              commit_sha: context.sha,
              body: `❌ Production deploy failed.\n\nRun: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
            });
```

For Slack, use the official action:

```yaml
      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ Runway production deploy failed",
              "attachments": [{
                "color": "danger",
                "fields": [
                  { "title": "Branch", "value": "${{ github.ref_name }}", "short": true },
                  { "title": "Commit", "value": "${{ github.sha }}", "short": true },
                  { "title": "Run", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false }
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_DEPLOY_WEBHOOK }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

Add `SLACK_DEPLOY_WEBHOOK` as a GitHub Actions secret. Get the webhook URL from Slack: App Directory → Incoming Webhooks → Add to Slack.

Notify on success too, but less urgently. A brief "✅ Deployed to production — commit abc123f" in a `#deployments` channel creates a useful audit trail.

---

## 9.10 The Minimum Viable Pipeline

If this chapter feels like a lot, here's the minimum pipeline that's significantly better than "deploy from laptop":

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - run: npx tsc --noEmit
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: eu-west-1
      - run: npx sst deploy --stage production
```

Five real steps. Type check, OIDC credentials, deploy. Add migrations, preview environments, approval gates, and notifications as the project grows and the pain of not having them becomes real.

The goal is that pushing to `main` is boring. Not exciting, not stressful, not a ritual that requires three people in a call. Just something that happens.

---

## Where We Are

Runway deploys automatically, reliably, and without secrets in source control. The full architecture from code to production:

```
[GitHub: PR branch]
    │
    ├─ PR opened
    │   ├─ Type check + tests run (no AWS credentials)
    │   └─ Preview environment deployed on pass (pr-<n> stage)
    │         └─ PR closed → preview torn down
    │
    └─ Merged to main
          ↓
       [GitHub Actions: deploy.yml]
         1. npm ci + tsc --noEmit
         2. OIDC: assume GithubDeployRole (temporary credentials)
         3. Run migrations (migration Lambda; CI exits on failure)
         4. sst deploy --stage staging
         ↓
       [Manual approval gate — GitHub Environments]
         ↓
       [sst deploy --stage production]
         ↓
       ✅ Notify on success / ❌ Notify on failure
```

No long-lived AWS access keys. No "works on my machine" deploys. No schema out-of-sync with Lambda code. No wondering who deployed what and when.

Chapter 10 closes the loop: once Runway is deployed and running, how do you know it's working? Logs, metrics, traces, and the minimum viable alerting setup that tells you something's wrong before your users do.

---

> **The code for this chapter** is available on the `chapter-9` branch of the companion repository.
