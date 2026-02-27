# Ship It With Types — eBook Outline

**Subtitle:** The Complete TypeScript + AWS + SST Handbook
**Target reader:** TypeScript engineers who know basic AWS and want to ship production-grade systems without it becoming a full-time job
**Tone:** Direct, opinionated, practical. No fluff. Real code that compiles.
**Scope decisions:** DynamoDB-first data chapter, full RDS chapter with Lambda event sourcing, full queues chapter, no Next.js (stays on the blog)

---

## Front Matter

- Preface — Why this book exists (the gap between "hello world Lambda" and production-grade SaaS)
- How to use this book — linear is fine, but Part II chapters are independent if you already know the basics
- Prerequisites — TypeScript intermediate, basic AWS familiarity (IAM, regions, console), Node.js 22+
- Setting up your environment — AWS credentials, SST CLI, IDE config

---

## Part I: Foundations

### Chapter 1: Setting Up SST v3

The fastest path from zero to a deployed Lambda. No detours.

1.1 What SST actually is (and what it isn't)
- IaC + local dev + deploy tooling in one
- Built on Pulumi under the hood — you never touch it directly
- How it compares to CDK, SAM, and raw CloudFormation

1.2 Your first project
- `npx create-sst@latest` walkthrough
- Project structure explained file by file
- `sst.config.ts` anatomy — `app()`, `run()`, components

1.3 SST Components: the building blocks
- `sst.aws.Function` — Lambda with sensible defaults
- `sst.aws.ApiGatewayV2` — HTTP routing
- Resource linking: how `link: [table]` replaces 20 lines of IAM policy

1.4 Live development with `sst dev`
- How Live Lambda works (WebSocket proxy, local execution against real AWS)
- The development loop: edit → save → instant
- Using the SST Console (logs, function triggers, table browser)

1.5 Stages and environments
- dev, staging, production as separate AWS stacks
- Stage-specific config and removal policies
- Environment variable strategy across stages

1.6 Your first deploy
- `sst deploy --stage production`
- What gets created in your AWS account
- Cleaning up with `sst remove`

---

### Chapter 2: Lambda Functions That Don't Suck

The patterns that separate functions you're proud of from functions you're scared to touch.

2.1 Handler typing done right
- `APIGatewayProxyHandlerV2` vs `APIGatewayProxyHandler` — which and when
- `SQSHandler`, `ScheduledHandler`, `S3Handler` — a full reference
- Why `async (event, context)` matters (request ID, timeout tracking)

2.2 Response helpers
- Building a typed response library
- Consistent error shapes across your API
- HTTP status codes worth knowing (and the ones people misuse)

2.3 Cold starts: the honest guide
- What a cold start actually is (execution environment lifecycle)
- The main culprits: bundle size, top-level I/O, slow imports
- Lazy initialisation pattern for AWS SDK clients
- Provisioned concurrency: when it's worth it and when it isn't

2.4 Structuring a production Lambda
- Thin handlers, fat services
- Where to put validation, business logic, and data access
- Module organisation for a Lambda package

2.5 Structured logging
- Why `console.log("it broke")` is useless at 3am
- Building a JSON logger with request ID correlation
- CloudWatch Logs Insights queries that actually help

2.6 Error handling
- Custom error classes with typed codes and HTTP status codes
- The handler-level try/catch pattern
- Never losing an unhandled error silently

2.7 Environment variable validation
- Failing fast at startup vs failing mysteriously at runtime
- Building a typed `config.ts` with early validation
- Secrets vs environment variables — the SST way

2.8 Testing Lambda functions
- Unit testing handlers with Vitest
- Integration testing against real AWS (sst dev makes this practical)
- What to mock and what not to mock

---

## Part II: Data

### Chapter 3: Type-Safe APIs with API Gateway

Building APIs that are impossible to misuse.

3.1 HTTP API vs REST API — choose correctly the first time
- Feature comparison, price difference, latency difference
- When REST API is actually the right answer (it's rare)

3.2 Route handlers and path parameters
- Typed path parameters with `event.pathParameters`
- Query strings: parsing and validation
- Request body parsing and validation with Zod

3.3 Request validation middleware
- A validation wrapper that keeps handlers clean
- Zod schemas as the single source of truth for request shape
- Returning structured 400 errors that clients can actually parse

3.4 CORS, rate limiting, and caching
- CORS done right in SST — common mistakes and how they manifest
- API Gateway built-in throttling and per-route limits
- Response caching with CloudFront

3.5 API versioning and evolution
- Path versioning (`/v1/`) with SST routes
- How to evolve an API without breaking clients
- Deprecation patterns

---

### Chapter 4: DynamoDB with Full Type Safety

Single-table design without the mystery.

4.1 Why DynamoDB (and when not to use it)
- The real trade-offs vs relational databases
- Access patterns first: the mental model shift required
- Signs you should be reading Chapter 5 instead

4.2 Single-table design fundamentals
- Primary keys, sort keys, composite key patterns
- GSIs and when you need them
- Entity prefixes and naming conventions that save you
- Access pattern mapping before you write a line of code

4.3 Setting up the typed client
- `DynamoDBDocumentClient` vs raw `DynamoDBClient`
- Building typed wrapper functions for each operation
- The `marshall`/`unmarshall` story

4.4 CRUD patterns with full types
- `PutCommand` with `ConditionExpression` (no more accidental overwrites)
- `GetCommand` and nullable return types
- `UpdateCommand` with safe attribute name aliases
- `DeleteCommand` with condition checks
- `QueryCommand` for paginated results by PK + SK
- `TransactWriteCommand` for atomic multi-item operations

4.5 GSI query patterns
- Defining GSIs in SST
- Query by secondary index
- Sparse GSI indexing (the underused pattern)

4.6 Pagination
- `LastEvaluatedKey` and `ExclusiveStartKey`
- Building a cursor-based pagination utility
- Consistent page sizes under variable DynamoDB behaviour

4.7 Optimistic locking and versioning
- Version attribute pattern
- `ConditionalCheckFailedException` as a control flow mechanism
- Building retry logic for concurrent updates

4.8 Common DynamoDB mistakes
- Hot partitions and how to avoid them
- Scan vs Query (and why you almost never want Scan)
- Over-indexing and its costs
- TTL for automatic data expiry

---

### Chapter 5: RDS and Aurora Serverless

When you actually need relational.

5.1 When to reach for RDS
- Honest comparison: DynamoDB vs Aurora Serverless v2
- Use cases that genuinely need relational (complex reporting, existing schemas, joins)
- Cost profile: Aurora Serverless v2 vs always-on RDS

5.2 Aurora Serverless v2 with SST
- `sst.aws.Aurora` component setup
- VPC configuration — why Lambda + RDS needs a VPC and how to set it up without losing your mind
- Database credentials and secret rotation

5.3 Connecting from Lambda
- RDS Proxy: what it solves (connection pooling), what it costs, when you need it
- Direct connection patterns and their limits (cold start + connection overhead)
- Connection string management with SST secrets

5.4 Prisma in Lambda
- Prisma as the TypeScript ORM of choice for Aurora
- Bundle size considerations and the `prisma generate` step in CI
- Handling Prisma Client in a Lambda execution environment
- Query engine binary and layer strategies

5.5 Schema migrations in production
- Why `prisma migrate deploy` in Lambda is risky
- Migration runner as a separate Lambda (triggered on deploy)
- Rollback strategy for failed migrations
- Zero-downtime migrations: expand/contract pattern

5.6 Lambda event sourcing from RDS
- Aurora native Lambda invocation: calling a Lambda directly from a stored procedure or trigger
- Use cases: audit logs, notifications, cache invalidation on write
- RDS event subscriptions via SNS → Lambda for database lifecycle events (failovers, backups, parameter changes)
- Change data capture (CDC) with DMS → Kinesis → Lambda for streaming row-level changes
- Choosing the right pattern: native invocation vs CDC vs event subscriptions

5.7 Aurora and read replicas
- Read replica setup for reporting workloads
- Routing reads vs writes in Lambda
- Connection string management for multi-endpoint setups

---

### Chapter 6: S3 and File Uploads

Handling files without building a file server.

6.1 S3 fundamentals in TypeScript
- Buckets, objects, keys — the mental model
- Bucket configuration in SST (public vs private, CORS, versioning)

6.2 Direct uploads with presigned URLs
- Why you should never route file data through Lambda
- Generating presigned POST URLs for client-side upload
- Generating presigned GET URLs for download
- URL expiry and security considerations

6.3 Upload flows in practice
- Client-side upload to S3 with presigned POST
- Progress tracking
- Post-upload confirmation back to your API

6.4 S3 event triggers
- Triggering a Lambda on object creation
- Processing uploaded files: image resizing, CSV parsing, document extraction
- Error handling for S3 event Lambdas (and why they're fire-and-forget)

6.5 Access patterns
- Private assets with signed URLs (user-scoped content)
- Public assets via CloudFront
- Bucket policies and IAM — minimum required permissions

6.6 Lifecycle policies and cost control
- Moving old objects to cheaper storage tiers automatically
- Automatic deletion of temporary uploads
- Versioning: when you need it and when it just bloats costs

---

## Part III: Auth and Async

### Chapter 7: Authentication with Cognito

Shipping auth without building auth.

7.1 Cognito concepts
- User Pools vs Identity Pools — which you actually need
- Hosted UI vs custom UI trade-offs
- App clients and token lifetimes

7.2 Setting up Cognito with SST
- `sst.aws.CognitoUserPool` and `sst.aws.CognitoUserPoolClient`
- Linking to API Gateway for JWT authorisation
- Email verification and password policy config

7.3 JWT validation in Lambda
- Verifying Cognito JWTs without a round-trip
- Extracting user claims from the token
- An `auth` middleware that extracts the user and passes it down

7.4 Protected API routes
- Attaching a Cognito authoriser to API Gateway routes in SST
- Mixing public and protected routes in one API
- Handling `401` vs `403` correctly

7.5 User identity in your data model
- Scoping DynamoDB records to a user (`USER#<sub>`)
- Ensuring users can only access their own data
- Multi-tenant patterns (one user pool, multiple tenants)

7.6 Social login
- Adding Google / GitHub OAuth via Cognito federation
- Handling the callback flow
- Linking social accounts to existing Cognito users

7.7 Alternatives worth knowing
- Auth.js (NextAuth) in an SST stack
- Clerk for teams who want to buy vs build
- When to roll your own JWT (almost never)

---

### Chapter 8: Queues, Events, and Background Jobs

Decoupling your system so a slow email provider doesn't fail a user registration.

8.1 Why async matters
- Keeping API response times fast
- Resilience through decoupling
- The cost of tight coupling at scale

8.2 SQS in depth
- `sst.aws.Queue` and subscribing a Lambda
- Typed message bodies — no stringly-typed payloads
- Visibility timeout, retry behaviour, and exponential backoff
- Dead-letter queues: what goes there and why
- Batch processing and partial batch failures (the `batchItemFailures` pattern)
- FIFO queues: when ordering matters and what it costs you

8.3 EventBridge for event-driven architecture
- SQS vs EventBridge: point-to-point vs pub/sub
- Defining events as TypeScript types and keeping a schema registry
- Fan-out patterns: one event, multiple consumers
- Event filtering rules — keeping consumers focused
- Event replay and archiving for debugging and backfill

8.4 Scheduled jobs with cron
- `sst.aws.Cron` for time-based triggers
- Patterns for idempotent scheduled jobs
- Avoiding duplicate executions with DynamoDB conditional writes
- Monitoring and alerting on cron failures

8.5 Long-running jobs
- Lambda's 15-minute limit and how to work around it
- Step Functions for multi-step workflows (with SST)
- Progress reporting back to the client: polling vs WebSocket vs Server-Sent Events
- Async job status tracking in DynamoDB

---

## Part IV: Ship and Maintain

### Chapter 9: CI/CD and Deployment Automation

Deployments that happen in the background, not the foreground of your week.

9.1 Repository structure for an SST project
- Monorepo vs polyrepo trade-offs
- Where infrastructure, functions, and frontend live
- Shared TypeScript config and package patterns

9.2 GitHub Actions pipeline
- SST deploy action setup
- Configuring AWS credentials safely (OIDC vs access keys — always OIDC)
- Branch → stage mapping (main → production, PRs → preview)

9.3 Preview environments
- Deploying per-PR with dynamic stage names
- Seeding preview environments with test data
- Teardown on PR close (cost control)

9.4 Environment promotion
- Staging → production promotion flow
- Manual approval gates for production deploys
- Rollback strategy: SST doesn't have one built-in, here's what you do instead

9.5 Secrets management in CI
- GitHub Actions secrets → SST secrets
- Never committing credentials (and what to do if you do)
- Rotating secrets without downtime

9.6 Database migrations in CI
- Running Prisma migrations as part of a deploy pipeline
- Ordering: migrate before Lambda deploy, not after
- Failed migration rollback and the expand/contract safety net

---

### Chapter 10: Monitoring and Observability

Knowing what's broken before your users do.

10.1 The three pillars in an AWS context
- Logs (CloudWatch), metrics (CloudWatch Metrics), traces (X-Ray)
- What SST sets up for you by default vs what you have to wire yourself

10.2 Structured logging at scale
- Log levels and when to use each
- Correlation IDs across service boundaries
- CloudWatch Logs Insights: the queries worth saving

10.3 Custom CloudWatch metrics
- Emitting business metrics from Lambda (not just technical ones)
- Dashboards that tell you if the business is working, not just if Lambda is running
- Anomaly detection for cost spikes

10.4 Distributed tracing with X-Ray
- Enabling X-Ray in SST
- Tracing across Lambda → DynamoDB → SQS
- Reading a trace map and identifying bottlenecks

10.5 Alerting that doesn't cry wolf
- CloudWatch Alarms: error rate, latency p99, throttles
- Routing alerts to Slack or PagerDuty
- Alert fatigue and how to avoid it — the minimum viable alerting surface

10.6 The minimum viable observability setup
- What to instrument on day one vs what can wait
- A starter CloudWatch dashboard worth shipping with every project

---

## Appendices

**Appendix A: AWS Account Setup for Production**
- Root account lockdown, billing alerts, IAM users and roles, MFA

**Appendix B: Common Errors Reference**
- `ConditionalCheckFailedException`, `ProvisionedThroughputExceededException`, `AccessDeniedException`, `InvalidSignatureException`, RDS connection errors — causes and fixes

**Appendix C: SST Component Reference Cheat Sheet**
- Quick reference for every component used in the book with common config patterns

**Appendix D: TypeScript Patterns Reference**
- `satisfies`, discriminated unions, branded types, and other patterns used throughout

**Appendix E: Cost Estimation Guide**
- Lambda, API Gateway, DynamoDB, Aurora Serverless, S3, Cognito pricing at common scales (1k, 10k, 100k users)

---

## Chapter map

| # | Chapter | Part | Est. pages |
|---|---------|------|-----------|
| 1 | Setting Up SST v3 | Foundations | 35 |
| 2 | Lambda Functions That Don't Suck | Foundations | 45 |
| 3 | Type-Safe APIs with API Gateway | Data | 30 |
| 4 | DynamoDB with Full Type Safety | Data | 50 |
| 5 | RDS and Aurora Serverless | Data | 40 |
| 6 | S3 and File Uploads | Data | 30 |
| 7 | Authentication with Cognito | Auth & Async | 35 |
| 8 | Queues, Events, and Background Jobs | Auth & Async | 40 |
| 9 | CI/CD and Deployment Automation | Ship & Maintain | 35 |
| 10 | Monitoring and Observability | Ship & Maintain | 30 |
| — | Front matter + Appendices | — | 30 |
| | **Total** | | **~370** |
