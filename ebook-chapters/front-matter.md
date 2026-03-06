# Ship It With Types

## Preface

This book started as a frustration.

The AWS documentation is thorough. The SST documentation is good. The TypeScript documentation is excellent. And yet, every time I tried to find a resource that combined all three into something you could actually use to build a real product, I came up empty. What existed was either too shallow — "here's a hello world Lambda" — or too abstract — sprawling architecture diagrams with no code in sight.

The gap isn't information. It's synthesis.

What this book does is show you how the pieces fit together, through the process of building something real. Runway is a client portal and invoicing SaaS for freelancers. It is not a toy. By chapter 10, it has authentication, a type-safe API, two databases serving different purposes, file storage, async queues, background jobs, a CI/CD pipeline, and production monitoring. You would not be embarrassed to show this codebase to another engineer.

That is the bar. Not "it works on my machine." Shipped. Observable. Maintainable.

A note on opinions: this book has them. AWS has a hundred ways to solve any given problem, and I have picked one. The choices here — SST over CDK, DynamoDB for operational data, Aurora for reporting, SQS over SNS for guaranteed delivery — are defensible, not arbitrary. I explain the reasoning. You are welcome to disagree, but "it depends" is not useful when you are trying to get something into production before your runway runs out.

---

## How to Use This Book

### Reading order

The book is designed to be read front to back. Each chapter builds on the previous one. The Runway codebase grows incrementally — by chapter 4 you have a working API with a real database; by chapter 7 you have authentication; by chapter 10 you have a production-grade system with monitoring and alerting.

If you are already comfortable with SST setup and basic Lambda patterns, you can skim chapters 1 and 2. Everything from chapter 3 onward assumes you understand the chapter before it.

### Code examples

Every code example in this book is taken from the Runway codebase. The full source is available at:

```
https://github.com/moi88702/runway-example-project
```

The repository has one branch per chapter (`chapter-1` through `chapter-10`). Each branch represents the complete state of the codebase at the end of that chapter — not just the files introduced in that chapter, but everything. If you get stuck or something doesn't compile, check the branch for that chapter.

The examples are TypeScript-strict. They pass `tsc --noEmit` and have a test suite that runs on every push. If the code in this book doesn't work, that is a bug — file an issue.

### Conventions

Code blocks show complete, runnable files unless marked with `// ...`:

```typescript
// src/lib/queue.ts
import { SQSClient } from "@aws-sdk/client-sqs";

export const sqs = new SQSClient({ region: process.env.AWS_REGION });

// ... rest of file
```

When a file path appears as a comment at the top of a block, that is where the file lives in the project. Create it there.

Shell commands are shown with a `$` prefix. Do not type the `$`:

```bash
$ npx sst dev
```

When I introduce a new concept that warrants extra attention, it appears in a block like this:

> **Why not X?** These callouts explain why I chose one approach over an obvious alternative. Skip them on a first read if you want to stay in flow.

### Following along

The most effective way to use this book is to create your own Runway project and build it chapter by chapter. Copy the code, run it, break it, fix it. The second most effective way is to read it cover to cover and then refer back to specific sections when you encounter the same problems on your own projects.

Reading without typing is the least effective approach, but the book is written to be useful that way too — each chapter stands on its own as a reference for its topic.

---

## Prerequisites

This book assumes you know TypeScript and have used AWS before. It does not assume you know SST, DynamoDB, Aurora, Cognito, or any of the other specific services covered.

**TypeScript:** You should be comfortable with generics, utility types, discriminated unions, and async/await. If `Promise<T>`, `Partial<User>`, and `type Result = { ok: true; data: T } | { ok: false; error: string }` are readable to you, you are in good shape.

**AWS:** You should have an AWS account and understand the conceptual model — regions, IAM, the idea of serverless vs. provisioned resources. You do not need to be an IAM expert. Appendix A covers account setup if you are starting from scratch.

**Node.js:** Node 18 or later. The book uses ES modules throughout.

**Not required:** Prior experience with SST, DynamoDB, Prisma, SQS, EventBridge, Cognito, or GitHub Actions. All of these are introduced and explained as we build.

### What you will need

- An AWS account with administrator access (or an IAM user with broad permissions — Appendix A has the policy)
- Node.js 22+ and npm
- Git
- A terminal you are comfortable in
- Approximately 10–15 hours to work through the full book at a reasonable pace

SST deploys real AWS infrastructure. The services used in this book fall comfortably within the AWS free tier for development workloads, but once you start running preview environments and Aurora clusters you will see small charges. Appendix E has a cost breakdown.

---
