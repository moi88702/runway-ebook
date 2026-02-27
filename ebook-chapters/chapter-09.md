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
│   ├── handlers/              # Lambda handler entry points (thin)
│   │   ├── api/               # HTTP API handlers
│   │   ├── workers/           # SQS/EventBridge workers
│   │   └── jobs/              # Cron + Step Function Lambdas
│   ├── repositories/          # Data access layer (DynamoDB, Aurora)
│   ├── lib/                   # Shared utilities (queue, events, email)
│   ├── types/                 # Shared TypeScript types and Zod schemas
│   └── services/              # Business logic (no AWS SDK in here)
│
├── prisma/
│   ├── schema.prisma
│   └── migrations/            # Never delete these
│
├── scripts/
│   └── migrate.ts             # Migration runner Lambda entry point
│
└── .github/
    └── workflows/
        ├── deploy.yml         # Main deploy pipeline
        └── pr-checks.yml      # Type check + lint on every PR
```

A few conventions worth adopting early:

**Handlers are thin.** A handler's job is to parse the event, call a service, and return a response. Business logic lives in `src/services/`. This makes unit testing services trivial without Lambda event machinery.

**Types are shared.** Zod schemas in `src/types/` are imported by both handlers and workers. The same `InvoicePaidEvent` type that the invoice handler emits is what the analytics worker validates on arrival. One source of truth.

**Never commit `.env`.** Add it to `.gitignore` on day one. Use `.env.example` with placeholder values to document what's needed. Actual secrets live in SST secrets (`sst secret set`).

---

## 9.2 GitHub Actions Pipeline

The deploy pipeline has one job: when code is pushed to `main`, deploy to production. When a PR is opened, deploy to a preview environment.

First, the credentials problem.

### OIDC over Access Keys

The naive approach: create an IAM user, generate access keys, store them in GitHub Actions secrets as `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. This works. It's also a security liability — long-lived credentials that can be leaked, rotated incorrectly, or forgotten.

The right approach: OIDC. GitHub and AWS have a trust relationship. GitHub can prove "this workflow is running in the `yourorg/runway` repository on the `main` branch," and AWS grants temporary credentials in response. No long-lived keys anywhere.

Setting it up in AWS:

```typescript
// sst.config.ts — add this to your infrastructure
import * as iam from "aws-cdk-lib/aws-iam";
import * as cdk from "aws-cdk-lib";
import { Stack } from "aws-cdk-lib";

const githubOrg = "your-org";
const githubRepo = "runway";

// Stack.of() retrieves the CDK stack that SST builds internally.
// Pass any .nodes construct — here we use the API's underlying CDK construct.
const stack = Stack.of(api.nodes.httpApi);

const oidcProvider = new iam.OpenIdConnectProvider(stack, "GithubOidcProvider", {
  url: "https://token.actions.githubusercontent.com",
  clientIds: ["sts.amazonaws.com"],
  thumbprints: ["6938fd4d98bab03faadb97b34396831e3780aea1"],
});

const deployRole = new iam.Role(stack, "GithubDeployRole", {
  assumedBy: new iam.WebIdentityPrincipal(oidcProvider.openIdConnectProviderArn, {
    StringLike: {
      "token.actions.githubusercontent.com:sub": `repo:${githubOrg}/${githubRepo}:*`,
    },
    StringEquals: {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    },
  }),
  managedPolicies: [
    // Be more restrictive in practice — this is broad for simplicity
    iam.ManagedPolicy.fromAwsManagedPolicyName("AdministratorAccess"),
  ],
});

new cdk.CfnOutput(stack, "GithubDeployRoleArn", {
  value: deployRole.roleArn,
});
```

Run this once with `sst deploy --stage tools` (or a dedicated infra stage). Copy the output role ARN and add it as a GitHub Actions secret: `AWS_DEPLOY_ROLE_ARN`.

In practice, scope the IAM role to only the permissions SST needs — CloudFormation, Lambda, DynamoDB, S3, IAM for role creation. `AdministratorAccess` is convenient but gives the deploy role more access than it needs. Start broader and tighten as you go.

### The Deploy Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - main

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    name: Deploy to production
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval if configured in GitHub

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: eu-west-1
          role-session-name: GitHubDeploy-${{ github.run_id }}

      - name: Run database migrations
        run: npx tsx scripts/run-migrations.ts
        env:
          SST_STAGE: production

      - name: Deploy
        run: npx sst deploy --stage production
        env:
          # SST picks this up to know which stage to deploy
          SST_STAGE: production
```

The type check runs before deploy. Migrations run before the Lambda update. If migrations fail, the deploy stops. If the type check fails, nothing reaches AWS.

### PR Checks

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    branches:
      - main

jobs:
  check:
    name: Type check + lint
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Lint
        run: npx eslint src --max-warnings 0

      - name: Run unit tests
        run: npx vitest run
```

This runs on every PR. Type errors and lint failures block the merge. No AWS credentials needed here — unit tests don't touch AWS.

---

## 9.3 Preview Environments

SST stages are isolated AWS stacks. `sst deploy --stage pr-42` creates a completely separate Runway environment: its own Lambda functions, its own DynamoDB table, its own Aurora cluster (or a smaller substitute). Every PR gets its own playground.

```yaml
# .github/workflows/deploy.yml — add PR preview job
name: Deploy

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  id-token: write
  contents: read
  pull-requests: write  # To comment the preview URL on the PR

jobs:
  preview:
    name: Deploy preview
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: eu-west-1

      - name: Deploy preview
        id: deploy
        run: |
          OUTPUT=$(npx sst deploy --stage pr-${{ github.event.number }} 2>&1)
          echo "$OUTPUT"
          # Extract the API URL from SST output
          API_URL=$(echo "$OUTPUT" | grep -oP 'ApiEndpoint: \K[^\s]+' || echo "")
          echo "api_url=$API_URL" >> $GITHUB_OUTPUT

      - name: Comment preview URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `🚀 Preview deployed: ${{ steps.deploy.outputs.api_url }}\n\nStage: \`pr-${{ github.event.number }}\``
            })

  teardown:
    name: Tear down preview
    runs-on: ubuntu-latest
    if: github.event.action == 'closed'

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: eu-west-1

      - name: Remove preview stack
        run: npx sst remove --stage pr-${{ github.event.number }} --force
```

When the PR closes (merged or abandoned), the preview stack is torn down. No orphaned resources, no surprise costs.

### Cost Management for Preview Environments

Aurora Serverless v2 has a minimum ACU of 0.5 (~$0.06/hour) even when idle. For 10 open PRs each with an Aurora cluster, that's $14/day for idle databases.

Two options:

**Shared preview database.** Create one Aurora cluster in a `preview` stage that all PR environments share. Each PR gets a separate database name (`runway_pr_42`) rather than a separate cluster. Less isolation, much cheaper.

**Skip RDS in previews.** Use only DynamoDB for preview environments — it's serverless and genuinely free when idle. Aurora only gets deployed in `staging` and `production`.

```typescript
// sst.config.ts
const isProd = $app.stage === "production";
const isStaging = $app.stage === "staging";
const needsAurora = isProd || isStaging;

if (needsAurora) {
  const db = new sst.aws.Aurora("RunwayDb", {
    engine: "postgresql15.4",
    scaling: { min: "0.5 ACU", max: isProd ? "8 ACU" : "2 ACU" },
    vpc,
  });
}
```

For most teams, the shared preview database approach is the right call. Preview environments exist for testing HTTP endpoints and Lambda behaviour — they don't need production-identical data stores.

### Seeding Preview Data

A preview environment with an empty database isn't useful. Add a seed step:

```typescript
// scripts/seed-preview.ts
import { docClient } from "../src/lib/dynamo";
import { PutCommand } from "@aws-sdk/lib-dynamodb";
import { Resource } from "sst";

async function seed() {
  const workspaceId = "WORKSPACE#preview-seed";

  await docClient.send(new PutCommand({
    TableName: Resource.RunwayTable.name,
    Item: {
      PK: workspaceId,
      SK: "WORKSPACE",
      ownerId: "USER#preview-user",
      name: "Preview Workspace",
      plan: "pro",
      createdAt: new Date().toISOString(),
    },
  }));

  // Seed clients, projects, invoices...
  console.log("Preview data seeded");
}

seed().catch(console.error);
```

```yaml
# In the preview deploy step, after deploy:
- name: Seed preview data
  run: npx tsx scripts/seed-preview.ts
  env:
    SST_STAGE: pr-${{ github.event.number }}
```

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
2. Add "Required reviewers" — whoever must approve the production deploy
3. Reference it in the workflow: `environment: production`

When the deploy job reaches the production step, GitHub pauses and sends a notification to the required reviewers. They approve, the deploy proceeds.

```yaml
  deploy-production:
    name: Deploy to production
    runs-on: ubuntu-latest
    needs: deploy-staging  # Must pass staging first
    environment: production  # Pauses for manual approval

    steps:
      - uses: actions/checkout@v4
      # ... same steps as staging deploy, with SST_STAGE: production
```

### Rollback Strategy

SST doesn't have a built-in rollback command. When production breaks:

1. The fastest path is to revert the commit and push: `git revert HEAD && git push`. The pipeline redeploys the previous version.

2. For Lambda-specific rollbacks (code change caused the issue, not infrastructure), Lambda maintains a version history. You can manually point the function alias to the previous version in the AWS console — this is a 30-second fix while the revert deploy is running.

3. For DynamoDB and Prisma schema changes, rollback is harder. This is why Chapter 5 covers the expand/contract migration pattern — never deploy a migration that removes a column that the previous version of the code still reads. Always: add the new column, deploy new code, remove the old column.

Keep a `RUNBOOK.md` in the repo with the rollback steps. You will need it at 11pm on a Friday and you will not be in a state to reason clearly.

---

## 9.5 Secrets Management in CI

Runway has secrets: database credentials, Mailgun API keys, Stripe webhook secrets, Cognito client IDs. They need to be available in production Lambdas without appearing in the codebase or the CI logs.

### SST Secrets

SST has a first-class secrets system:

```bash
# Set a secret for the production stage
npx sst secret set MailgunApiKey "key-xxxxx" --stage production

# List secrets (shows names, not values)
npx sst secret list --stage production
```

SST stores secrets in AWS SSM Parameter Store (encrypted with KMS). Lambda functions linked to a secret can read it at runtime via the `Resource` helper — no environment variable hacks, no `process.env.MAILGUN_API_KEY` that can leak in logs.

```typescript
// sst.config.ts
const mailgunKey = new sst.Secret("MailgunApiKey");

const emailWorker = new sst.aws.Function("EmailWorker", {
  handler: "src/workers/email.handler",
  link: [mailgunKey, emailQueue, table],
});
```

```typescript
// src/workers/email.ts
import { Resource } from "sst";

const mailgunClient = new Mailgun({
  apiKey: Resource.MailgunApiKey.value,
  domain: "mail.runwayapp.com",
});
```

`Resource.MailgunApiKey.value` resolves at runtime from SSM. It never appears in CloudFormation templates or Lambda environment variables in plaintext.

### Setting Secrets in CI

The deploy pipeline needs to set secrets in CI, or at least the secrets need to exist before deploy. Two approaches:

**Pre-populated (recommended):** Set secrets manually once via the CLI. They persist in SSM across deploys. CI just runs `sst deploy` and the secrets are already there.

**CI-injected:** Store secret values in GitHub Actions secrets and inject them during deploy:

```yaml
- name: Set SST secrets
  run: |
    npx sst secret set MailgunApiKey "${{ secrets.MAILGUN_API_KEY }}" --stage production
    npx sst secret set StripeWebhookSecret "${{ secrets.STRIPE_WEBHOOK_SECRET }}" --stage production
```

The pre-populated approach is simpler and means fewer places where the secret value needs to live. Do it once, rotate via CLI when needed.

### If a Secret Leaks

If credentials are committed to the repo or exposed in logs:

1. Rotate the credential immediately (before anything else)
2. Revoke the old credential if possible
3. Check the git history: `git log --all -p -- .env` or `git log --all --full-history -- "**/.env"`
4. Use BFG Repo Cleaner or `git filter-repo` to scrub it from history, then force-push
5. Notify whoever needs to know (Stripe, Mailgun, etc.) that the key was exposed

GitHub has secret scanning enabled by default on public repos and will email you if it detects known secret patterns. This is a warning, not a guarantee — act immediately, don't wait for GitHub.

---

## 9.6 Database Migrations in CI

Prisma migrations in production deserve their own section because the wrong approach causes downtime.

The wrong approach: running `prisma migrate deploy` inside the Lambda handler on cold start. This means:

- Only one Lambda instance runs the migration (race condition if multiple cold starts hit simultaneously)
- If the migration is slow, the cold start times out
- Failed migrations leave the database in an inconsistent state with the code already deployed

The right approach: run migrations as a separate step, before the Lambda deploy.

### Migration Runner

In Chapter 5, you set up a migration Lambda that gets triggered on deploy. In CI, trigger it explicitly before `sst deploy`:

```typescript
// scripts/run-migrations.ts
import { LambdaClient, InvokeCommand } from "@aws-sdk/client-lambda";
import { Resource } from "sst";

const client = new LambdaClient({});

async function runMigrations() {
  console.log("Invoking migration runner...");

  const response = await client.send(
    new InvokeCommand({
      FunctionName: Resource.MigrationRunner.name,
      InvocationType: "RequestResponse", // Wait for completion
      LogType: "Tail",
    })
  );

  const logs = Buffer.from(response.LogResult ?? "", "base64").toString();
  console.log("Migration logs:", logs);

  if (response.FunctionError) {
    console.error("Migration failed:", response.FunctionError);
    process.exit(1); // Fail the CI job
  }

  const result = JSON.parse(
    Buffer.from(response.Payload ?? "").toString()
  );

  console.log("Migration result:", result);

  if (!result.success) {
    console.error("Migration reported failure");
    process.exit(1);
  }
}

runMigrations().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

```yaml
# In deploy.yml — migrations before deploy
- name: Run database migrations
  run: npx tsx scripts/run-migrations.ts
  env:
    SST_STAGE: production

- name: Deploy
  run: npx sst deploy --stage production
```

If `run-migrations.ts` exits with non-zero, the deploy step never runs. The old Lambda stays live, the old schema stays in the database. Everything is consistent.

### Zero-Downtime Migrations

The expand/contract pattern from Chapter 5 applies here in full. A CI-safe migration approach:

**Deploy N (current):** Column `invoiceName` exists and is written + read by the Lambda.

**Migration M (expand):** Add new column `title`. Make it nullable. Old Lambda ignores it, new Lambda writes both.

**Deploy N+1:** Lambda reads `title`, falls back to `invoiceName` if null. Writes both.

**Migration M+1 (backfill):** `UPDATE invoices SET title = invoiceName WHERE title IS NULL`. Can run as a separate job, not in the migration runner.

**Migration M+2 (contract):** Once all rows have `title`, drop `invoiceName`. Only safe after all Lambda instances have been replaced by N+1.

Never do: add a NOT NULL column without a default in the same deploy as the code change that starts writing it. The Lambda deploys after the migration, so there's a window where new rows are being written without the column.

---

## 9.7 The Minimum Viable Pipeline

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

Five real steps. Type check, OIDC credentials, deploy. Add migrations, preview environments, and approval gates as the project grows.

The goal is that pushing to `main` should be boring. Not exciting, not stressful, not a ritual that requires three people in a call. Just something that happens.

---

## Where We Are

Runway deploys automatically, reliably, and without secrets in source control. The architecture map now includes the pipeline:

```
[GitHub: PR branch]
    │
    ├─ PR opened → preview environment deployed (pr-42 stage)
    │   └─ PR closed → preview torn down
    │
    └─ Merged to main
          ↓
       [GitHub Actions]
         1. npm ci + tsc --noEmit
         2. Run migrations (migration Lambda, exits CI on failure)
         3. sst deploy --stage staging
         ↓
       [Manual approval gate]
         ↓
       [sst deploy --stage production]
```

No long-lived AWS access keys. No "works on my machine" deploys. No schema out-of-sync with Lambda code.

Chapter 10 closes the loop: once Runway is deployed and running, how do you know it's working? Logs, metrics, traces, and the minimum viable alerting setup that tells you something's wrong before your users do.

---

> **The code for this chapter** is available at `09-cicd-deployment/` in the companion repository.
