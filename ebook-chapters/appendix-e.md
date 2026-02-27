# Appendix E: Cost Estimation Guide

The numbers that tell you whether Runway's AWS bill makes sense at different scales. All prices are US East 1 / EU West 1 approximate as of early 2026 — always verify with the [AWS Pricing Calculator](https://calculator.aws/pricing/2/home) for current figures.

The goal here isn't precision. It's order-of-magnitude intuition: what costs almost nothing, what scales linearly, and what has step-function pricing that will surprise you.

---

## Assumptions

Three usage tiers:

| Tier | Monthly Active Users | API Requests/Month | Description |
|------|---------------------|-------------------|-------------|
| **Small** | 1,000 | 500,000 | Early product, paying customers, low traffic |
| **Medium** | 10,000 | 5,000,000 | Growing SaaS, B2B |
| **Large** | 100,000 | 50,000,000 | Scaled product |

Runway usage pattern assumed:
- Average 5 API requests per user session, 10 sessions/month per MAU
- 60% read (DynamoDB Query/Get), 40% write (Put/Update)
- Average Lambda duration: 50ms for reads, 150ms for writes
- Average Lambda memory: 512MB
- ~20% of users trigger at least one file upload/month
- Email: ~2 transactional emails per MAU per month

---

## Lambda

**Pricing:** $0.20 per 1M requests + $0.0000166667 per GB-second

**Free tier:** 1M requests/month + 400,000 GB-seconds/month (permanent, not expiring)

```
GB-seconds per request at 512MB:
- Read request (50ms):  0.512 GB × 0.05s = 0.0256 GB-s
- Write request (150ms): 0.512 GB × 0.15s = 0.0768 GB-s
Average: ~0.045 GB-s per request (60/40 mix)
```

| Tier | Requests | GB-seconds | Request cost | Compute cost | **Total** |
|------|----------|-----------|-------------|-------------|-----------|
| Small | 500K | 22,500 | $0.00 (free tier) | $0.00 (free tier) | **~$0** |
| Medium | 5M | 225,000 | $0.80 | $3.75 | **~$4.55** |
| Large | 50M | 2,250,000 | $9.60 | $31.25 | **~$41** |

Lambda is effectively free at small scale. At medium scale it's noise. At large scale it's $40/month — still not the bill driver.

---

## API Gateway V2 (HTTP API)

**Pricing:** $1.00 per 1M requests (first 300M/month)

| Tier | Requests | **Total** |
|------|----------|-----------|
| Small | 500K | **$0.50** |
| Medium | 5M | **$5.00** |
| Large | 50M | **$50.00** |

Linear scaling. At small and medium scale, this is negligible. At large scale, it's material but still not dominant.

---

## DynamoDB (On-Demand)

**Pricing:** $1.25 per million write request units (WRU), $0.25 per million read request units (RRU)

One write = 1 WRU per KB. One read = 0.5 RRU per KB (eventually consistent) or 1 RRU per KB (strongly consistent). Assume average item size of 1KB.

Assume: 40% of requests are writes, 60% are reads.

| Tier | Writes | Reads | Write cost | Read cost | **Total** |
|------|--------|-------|-----------|----------|-----------|
| Small | 200K | 300K | $0.25 | $0.08 | **~$0.33** |
| Medium | 2M | 3M | $2.50 | $0.75 | **~$3.25** |
| Large | 20M | 30M | $25.00 | $7.50 | **~$32.50** |

**Storage:** $0.25 per GB/month. At 100K users with ~100 items each averaging 1KB: ~10GB = $2.50/month.

**GSI costs:** each write that touches a GSI attribute counts as an additional write per GSI. One write to a table with two GSIs = 3 WRUs. Factor this in if you have multiple GSIs.

DynamoDB on-demand is cheap at all scales shown here. Provisioned capacity can be cheaper at very high, predictable throughput — but only if you have the traffic data to provision correctly.

---

## Aurora Serverless v2

**Pricing:** $0.12 per ACU-hour (eu-west-1). Storage: $0.10 per GB/month.

ACU usage depends on active query load. Aurora Serverless v2 scales from 0.5 ACU (minimum, idle) to your configured maximum.

**Rough ACU-hours estimation:**
- Idle (no queries): 0.5 ACU minimum, always running
- Light load (small tier): average 1 ACU
- Medium load: average 2–3 ACU
- Heavy load: 4–8 ACU with peaks higher

| Tier | Avg ACU | Hours/month | ACU-hours | Compute cost | Storage (10GB) | **Total** |
|------|---------|-------------|-----------|-------------|----------------|-----------|
| Small | 1 | 730 | 730 | $87.60 | $1.00 | **~$89** |
| Medium | 2.5 | 730 | 1,825 | $219.00 | $2.50 | **~$222** |
| Large | 6 | 730 | 4,380 | $525.60 | $10.00 | **~$536** |

**Aurora Serverless v2 is the most expensive line item by far.** This is the trade-off from Chapter 5: you pay for the clock, not just the queries. At small scale, that's $90/month whether you have 100 users or 1,000.

**RDS Proxy:** adds ~$0.015/hour per endpoint (~$11/month) per stage. Worth it when connections are a problem; unnecessary overhead when they aren't.

**When Aurora pays off vs DynamoDB:**
- If you have complex reporting queries, joins, or transactions that DynamoDB can't model cleanly, Aurora is worth the premium.
- If your data model fits DynamoDB, the $87/month minimum cost for Aurora is hard to justify at small scale.
- At large scale, the gap narrows — Aurora's per-query cost is low and DynamoDB write costs add up.

---

## S3

**Pricing:** $0.023 per GB/month (Standard storage, eu-west-1). PUT/POST: $0.005 per 1,000 requests. GET: $0.0004 per 1,000 requests. Data transfer out: $0.09/GB first 10TB.

Runway assumption: 20% of MAU upload one file/month, average 2MB per file.

| Tier | Files uploaded | Storage added | PUT cost | GET cost | Storage | Data transfer (20% downloaded) | **Total** |
|------|---------------|--------------|----------|----------|---------|-------------------------------|-----------|
| Small | 200 | 0.4 GB | $0.001 | ~$0 | $0.01 | ~$0 | **~$0.01** |
| Medium | 2,000 | 4 GB/month cumulative | $0.01 | ~$0.001 | $2.30 (100GB total) | $1.80 | **~$4.10** |
| Large | 20,000 | 40 GB/month | $0.10 | ~$0.01 | $23.00 (1TB total) | $18.00 | **~$41** |

S3 is cheap. At small and medium scale, negligible. At large scale, data transfer out is the main cost driver — CloudFront significantly reduces this for public assets (CDN caches at the edge, reducing origin transfer).

**S3 Intelligent-Tiering** automatically moves objects to cheaper storage tiers after 30–90 days of no access. Worth enabling for document storage where users rarely re-download old invoices.

---

## Cognito

**Pricing:** Free up to 10,000 MAUs. $0.0055 per MAU above 10,000.

| Tier | MAUs | Free | Paid MAUs | **Total** |
|------|------|------|-----------|-----------|
| Small | 1,000 | 1,000 | 0 | **$0** |
| Medium | 10,000 | 10,000 | 0 | **$0** |
| Large | 100,000 | 10,000 | 90,000 | **$495** |

Cognito is free until you're successful. At 100K MAUs it's $495/month — meaningful but proportionate for a product at that scale. Cognito's pricing model strongly favours early-stage products.

**Social login (federation):** Cognito federated identities have a separate pricing tier — $0.0025 per DAU (daily active user) for federated identities above 50,000 DAUs. Check if you're using Identity Pools vs User Pools federation.

---

## SQS

**Pricing:** $0.40 per million requests. First 1 million/month free.

Assume 2 messages per API request (one to queue, one delivery):

| Tier | Messages/month | **Total** |
|------|---------------|-----------|
| Small | 1M | **$0** (free tier) |
| Medium | 10M | **$4.00** |
| Large | 100M | **$40.00** |

Negligible at all scales shown. SQS is not a cost concern for Runway.

---

## EventBridge

**Pricing:** $1.00 per million custom events. First 1 million/month free.

At 1 event per API write (40% of requests):

| Tier | Events/month | **Total** |
|------|-------------|-----------|
| Small | 200K | **$0** (free tier) |
| Medium | 2M | **$1.00** |
| Large | 20M | **$19.00** |

Negligible.

---

## CloudWatch

**Pricing:** Logs ingestion $0.50 per GB. Storage $0.03 per GB/month. X-Ray traces $5.00 per million. Custom metrics $0.30 per metric/month (first 10 metrics free).

Assume ~200 bytes per log entry, 1 entry per Lambda request:

| Tier | Log GB/month | Ingestion | Storage (30 day) | X-Ray (sampled 5%) | **Total** |
|------|-------------|-----------|-----------------|-------------------|-----------|
| Small | 0.1 GB | $0.05 | $0.003 | ~$0 | **~$0.05** |
| Medium | 1 GB | $0.50 | $0.03 | $1.25 | **~$1.78** |
| Large | 10 GB | $5.00 | $0.30 | $12.50 | **~$17.80** |

At large scale, structured logging verbosity matters. If every request logs 10 lines instead of 2, costs multiply by 5. Use `DEBUG` level only in dev; `INFO` in production; log once per request, not per DynamoDB call.

**X-Ray sampling:** SST defaults to 5% sampling. For debugging a specific issue, you can temporarily set 100% sampling. Reset after — at large scale, 100% X-Ray tracing gets expensive.

---

## Total Cost Summary

| Service | Small (1K MAU) | Medium (10K MAU) | Large (100K MAU) |
|---------|---------------|-----------------|-----------------|
| Lambda | ~$0 | ~$4.55 | ~$41 |
| API Gateway | $0.50 | $5.00 | $50 |
| DynamoDB | ~$0.33 | ~$3.25 | ~$32.50 |
| Aurora Serverless v2 | ~$89 | ~$222 | ~$536 |
| S3 | ~$0 | ~$4.10 | ~$41 |
| Cognito | $0 | $0 | $495 |
| SQS | $0 | $4.00 | $40 |
| EventBridge | $0 | $1.00 | $19 |
| CloudWatch | ~$0.05 | ~$1.78 | ~$17.80 |
| **Total** | **~$90** | **~$246** | **~$1,272** |

Three observations:

**Aurora dominates at every scale.** It's 99% of the small-tier bill. If Runway can use DynamoDB for everything, the small-tier total drops to under $2/month.

**Cognito becomes material at scale.** At 100K MAUs, it's the second-largest line item after Aurora.

**Everything else is noise at small/medium scale.** Lambda, API Gateway, DynamoDB, SQS, EventBridge — all under $10/month at 10K MAUs.

---

## Cost Optimisation by Scale

### At 1K MAUs

- Drop Aurora if you can. Use DynamoDB only — bill drops from ~$90 to ~$2.
- If you need Aurora, reduce ACU minimum to 0.5 and set a low maximum for dev/staging stages.
- Use Cognito's free tier — it's free until 10K.

### At 10K MAUs

- Aurora is still the main cost. Consider whether the relational model is genuinely needed or whether a DynamoDB schema redesign could replace it.
- Enable S3 Intelligent-Tiering on document storage.
- Review CloudWatch log verbosity — cut unnecessary log lines.

### At 100K MAUs

- Evaluate Aurora Serverless v2 vs provisioned Aurora with autoscaling — provisioned can be cheaper at predictable high load.
- Cognito at $495/month: consider whether Clerk or Auth.js + DynamoDB would be cheaper (depends on auth complexity).
- CloudFront in front of S3 for public assets significantly reduces data transfer costs.
- Consider Reserved Instances / Savings Plans for predictable Aurora capacity — up to 40% discount vs on-demand.
- Review DynamoDB access patterns — at 20M writes/month, $25/month in write costs. Batching writes (BatchWriteItem) and caching hot reads (DynamoDB DAX or ElastiCache) can reduce this further.

---

## The "Are We Profitable?" Calculation

For Runway as a freelancer invoicing tool at £20/month per user:

| Tier | Revenue | AWS Cost | AWS as % of revenue |
|------|---------|----------|-------------------|
| Small (1K MAU) | £20,000/month | ~£73 | 0.37% |
| Medium (10K MAU) | £200,000/month | ~£199 | 0.10% |
| Large (100K MAU) | £2,000,000/month | ~£1,032 | 0.05% |

At £20/user/month, AWS infrastructure is under 0.5% of revenue at every scale. The serverless model earns its premium: you're paying for what you use, it scales without ops work, and it stays a rounding error on the P&L until you're very large indeed.

The cost to worry about isn't AWS. It's the team, the support, the marketing. The infrastructure bill for a well-built serverless SaaS is not the constraint.
