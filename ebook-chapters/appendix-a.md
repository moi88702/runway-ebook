# Appendix A: AWS Account Setup for Production

Most tutorials start with an IAM user, an access key, and `aws configure`. That's fine for a weekend project. For Runway — something you're running a business on — it's a security liability from day one.

This appendix covers the account setup you should do before anything else. It takes about an hour. It saves you from a billing shock, a credential leak, or a compromised account at the worst possible time.

---

## A.1 Root Account Lockdown

When you create an AWS account, you log in with an email address and password. That's the root account. It can do anything in your account — including things that can't be undone.

Do these four things immediately and never use the root account again:

**1. Enable MFA on root.** Go to IAM → Security recommendations → Add MFA. Use a hardware key (YubiKey) if you have one; an authenticator app (Authy, 1Password) if you don't. Not SMS — SIM-swapping is a real attack.

**2. Don't create root access keys.** If they exist, delete them. You should never need them. The only things that require root credentials are a small set of account-level operations (closing the account, changing root email, some support tier changes).

**3. Set a strong root password and store it in a password manager.** You'll almost never type it.

**4. Set a root account contact email that isn't your day-to-day address.** AWS sends billing and compliance emails to the root address. A dedicated `aws-billing@yourdomain.com` alias keeps noise out of your main inbox and ensures the right person sees the right email.

---

## A.2 IAM: Users, Roles, and the Right Mental Model

The IAM mental model that matters for Runway:

- **IAM Users** have long-lived credentials (username/password or access keys). Minimise their use. Humans should use IAM Identity Center (SSO) instead of IAM users wherever possible.
- **IAM Roles** are assumed temporarily. Lambda functions, GitHub Actions (via OIDC), and services use roles. Roles never have long-lived credentials.
- **IAM Policies** define what a user or role can do. Attach policies to roles, not inline to users.

For a solo developer or small team on Runway:

**Create one IAM user for day-to-day console access.** Attach `PowerUserAccess` (everything except IAM management). Don't use the root account for console work.

**Better: use IAM Identity Center.** If your team has more than one person, IAM Identity Center (formerly AWS SSO) lets you manage access centrally, use your existing identity provider (Google Workspace, Okta), and get short-lived credentials via `aws sso login`. No access keys to rotate. No credentials to leak.

**For CI/CD (GitHub Actions): use OIDC.** Chapter 9 covers this in detail. The principle: GitHub proves its identity to AWS, AWS issues temporary credentials, no long-lived keys anywhere.

**For local development: use SSO or named profiles, never a root access key in `~/.aws/credentials`.**

```ini
# ~/.aws/config — the right way
[profile runway-dev]
sso_start_url = https://your-org.awsapps.com/start
sso_region = eu-west-1
sso_account_id = 123456789012
sso_role_name = DeveloperAccess
region = eu-west-1

[profile runway-prod]
sso_start_url = https://your-org.awsapps.com/start
sso_region = eu-west-1
sso_account_id = 987654321098
sso_role_name = ReadOnlyAccess  # Minimal prod access by default
region = eu-west-1
```

```bash
aws sso login --profile runway-dev
export AWS_PROFILE=runway-dev
npx sst dev
```

---

## A.3 Billing Alerts

AWS has no spending cap by default. A misconfigured resource or a DDoS attack can run up a bill of thousands of dollars before you notice.

Set these up immediately, before deploying anything:

**Free tier alerts:** Go to Billing → Billing preferences → enable "Receive Free Tier Usage Alerts." AWS emails you when you're approaching or exceeding free tier limits.

**Billing alarm via CloudWatch:**

```typescript
// Add to a bootstrap stack or manually via console
import * as cloudwatch from "aws-cdk-lib/aws-cloudwatch";
import * as cloudwatchActions from "aws-cdk-lib/aws-cloudwatch-actions";
import * as sns from "aws-cdk-lib/aws-sns";
import * as snsSubscriptions from "aws-cdk-lib/aws-sns-subscriptions";

// Must be deployed to us-east-1 — billing metrics only exist there
const billingTopic = new sns.Topic(stack, "BillingAlertTopic");
billingTopic.addSubscription(
  new snsSubscriptions.EmailSubscription("you@yourdomain.com")
);

// Alert at $10 — adjust to your expected spend
new cloudwatch.Alarm(stack, "BillingAlarm10", {
  alarmName: "MonthlySpend-$10",
  metric: new cloudwatch.Metric({
    namespace: "AWS/Billing",
    metricName: "EstimatedCharges",
    statistic: "Maximum",
    period: cdk.Duration.hours(6),
    dimensionsMap: { Currency: "USD" },
  }),
  threshold: 10,
  evaluationPeriods: 1,
  comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
});
```

Set thresholds at $10, $50, $100 — each escalating the alert severity. The $10 alert is your "something unexpected is happening" signal. The $100 alert is your "stop everything and look at this" signal.

**AWS Budgets** (in the Billing console) is more flexible: set monthly budgets per service, per tag, per linked account. Alert at 50%, 80%, 100% of budget. Recommended for teams.

---

## A.4 Multi-Account Strategy

For Runway at scale, consider separate AWS accounts for production vs development:

```
[Management Account] — billing, IAM Identity Center
  ├── [Development Account] — dev + staging stages
  └── [Production Account] — production only
```

Benefits:
- Production resources are physically isolated from dev mistakes
- Cost visibility per environment is built-in
- IAM boundary between environments: a dev account credential leak can't touch production

AWS Organizations makes this manageable — create child accounts, apply Service Control Policies (SCPs), and centralise billing. AWS Control Tower automates the guardrails.

For solo developers or early-stage products: one account with strict IAM separation is fine. Migrate to multi-account when you have a team, a compliance requirement, or a scary enough production database.

---

## A.5 Account-Level Security Hardening

Run these once when setting up a new account:

**GuardDuty** — threat detection service. Analyses CloudTrail, VPC Flow Logs, and DNS logs for suspicious activity. Enable it in every region you use. ~$3–10/month for typical Runway usage. Worth it.

```bash
# Enable GuardDuty in your primary region
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES --region eu-west-1
```

**CloudTrail** — logs every API call in your account. Who did what, when, from where. Critical for incident investigation and compliance. Enable in all regions, store logs to S3 with CloudWatch Logs integration.

```bash
# Enable an organisation trail (or account trail)
aws cloudtrail create-trail \
  --name runway-audit-trail \
  --s3-bucket-name runway-cloudtrail-logs \
  --is-multi-region-trail \
  --enable-log-file-validation
aws cloudtrail start-logging --name runway-audit-trail
```

**Security Hub** — aggregates findings from GuardDuty, Inspector, and other services into a single dashboard with a security score. Enable after GuardDuty. Not free ($0.001 per finding) but the visibility is worth it early.

**S3 Block Public Access** — at the account level, not just per-bucket:

```bash
aws s3control put-public-access-block \
  --account-id YOUR_ACCOUNT_ID \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

This means no S3 bucket in your account can accidentally become public, regardless of its own ACL or policy settings. Only override this at the bucket level intentionally (e.g., your public Runway assets bucket).

---

## A.6 Region Selection

Pick one primary region and stick with it. For a UK-based product: `eu-west-1` (Ireland) or `eu-west-2` (London). `eu-west-2` has slightly fewer services available and costs marginally more; `eu-west-1` is the most mature EU region.

For GDPR: customer data must stay in the EU if you're serving EU users. Deploying to `eu-west-1` and not replicating to US regions covers this for most use cases. Document your data residency choices.

Deploy supporting infrastructure (billing alarms, CloudTrail, IAM Identity Center) to `us-east-1` — some AWS services only support that region for global resources.

---

## Checklist

Before deploying Runway to production, verify:

- [ ] Root account MFA enabled
- [ ] Root access keys deleted (or never created)
- [ ] IAM user or IAM Identity Center configured for daily use
- [ ] Billing alarm set at $10 and $50
- [ ] Free tier usage alerts enabled
- [ ] GuardDuty enabled in primary region
- [ ] CloudTrail enabled (multi-region)
- [ ] S3 Block Public Access enabled at account level
- [ ] No access keys in `~/.aws/credentials` for root
- [ ] CI/CD using OIDC (no long-lived keys in GitHub secrets)
