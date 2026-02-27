# Chapter 7: Authentication with Cognito

Every route in Runway has been unprotected until now. Any request with the right URL gets a response. That's fine for building — you want fast iteration without auth getting in the way — but it's not a product you can ship to real users.

This chapter adds authentication to Runway. By the end, every API route is protected. Freelancers can only access their own workspace. Clients can only see the projects and invoices that belong to them. The file uploads from Chapter 6 are scoped to the requesting user. The `ownerId` placeholder that's been sitting on the workspace model since Chapter 4 finally means something.

The tool: AWS Cognito. Specifically, Cognito User Pools — managed user directories that handle sign-up, sign-in, email verification, password reset, and token issuance. We write almost none of this logic. Cognito handles it.

---

## 7.1 Cognito Concepts

### User Pools vs Identity Pools

These sound similar. They're not.

**User Pools** are directories. They store usernames, passwords, email addresses, and profile attributes. They issue JWTs when users sign in. User Pools are what you want for application authentication — "log in as a user of this app."

**Identity Pools** (also called Federated Identities) are for granting AWS credentials directly to end users. They let your frontend code call AWS services directly — S3, DynamoDB, API Gateway — using temporary IAM credentials. You need Identity Pools if you want users to call AWS from the browser or a mobile app without routing through Lambda.

Runway uses a User Pool only. All AWS access goes through Lambda functions, not directly from clients. The frontend will call Runway's API, which calls AWS. Identity Pools would be over-engineering this.

### Tokens

When a user signs in to a Cognito User Pool, they receive three tokens:

**ID Token** — a JWT containing the user's identity claims: `sub` (a unique user ID), `email`, `custom:*` attributes, and more. This is what you use in Runway — it tells you who the user is.

**Access Token** — a JWT for authorising API requests. It contains the user's `sub` and scopes but fewer identity claims than the ID token. API Gateway's Cognito authoriser validates this by default.

**Refresh Token** — long-lived (default 30 days). Used to get new ID and Access tokens without re-authenticating. Stored securely by the client, never sent to your API.

For Runway's API Gateway authoriser, we'll use the Access Token. For extracting user identity in Lambda handlers, we'll read the claims that API Gateway forwards from the ID Token.

### Hosted UI vs custom UI

Cognito provides a hosted sign-in page at a Cognito domain (e.g., `runway.auth.eu-west-1.amazoncognito.com`). You redirect users there, they log in, they're redirected back with tokens.

For Runway's initial version, the hosted UI is fine. It handles:
- Sign up with email verification
- Sign in
- Forgot password / reset password
- MFA (if you enable it)

Custom UI means building your own sign-in forms and calling Cognito's API directly from your frontend. More control, more work. Runway will add a custom UI eventually — but not in this chapter.

---

## 7.2 Setting Up Cognito with SST

Runway has two types of users: **freelancers** (workspace owners) and **clients** (the freelancer's customers, who access their portal). Both need to authenticate. One User Pool handles both, differentiated by a custom attribute.

Add to `sst.config.ts`:

```typescript
/// <reference path="./.sst/platform/config.d.ts" />

export default $config({
  app(input) {
    return {
      name: "runway",
      removal: input?.stage === "production" ? "retain" : "remove",
      home: "aws",
    };
  },
  async run() {
    // ... existing resources from previous chapters ...

    // Cognito User Pool
    const userPool = new sst.aws.CognitoUserPool("RunwayUserPool", {
      usernames: ["email"],
      triggers: {
        // Auto-confirm users in non-production (skip email verification)
        ...(($app.stage !== "production") && {
          preSignUp: "src/functions/auth/pre-signup.handler",
        }),
        // Trigger workspace creation after first sign-up
        postConfirmation: "src/functions/auth/post-confirmation.handler",
      },
    });

    // App client for the frontend
    const userPoolClient = userPool.addClient("RunwayWebClient", {
      // OAuth flow for hosted UI
      callbackUrls: [
        $app.stage === "production"
          ? "https://app.runway.com/auth/callback"
          : "http://localhost:3000/auth/callback",
      ],
      logoutUrls: [
        $app.stage === "production"
          ? "https://app.runway.com/auth/logout"
          : "http://localhost:3000/auth/logout",
      ],
    });

    // Cognito domain for the hosted UI
    const domain = userPool.addDomain("RunwayDomain", {
      prefix: `runway-${$app.stage}`,
    });

    const api = new sst.aws.ApiGatewayV2("RunwayApi", {
      // Attach Cognito as the default authoriser for all routes
      authorizer: {
        type: "user_pool",
        userPool: {
          id: userPool.id,
          clientIds: [userPoolClient.id],
        },
      },
    });

    // Public routes (no auth required)
    api.route("GET /health", {
      handler: "src/functions/health.handler",
      auth: { iam: false },  // Override default authoriser
    });

    // Protected routes (Cognito JWT required)
    api.route("GET /workspaces/{workspaceId}", {
      handler: "src/functions/workspaces/get.handler",
      link: [table],
    });

    // ... all other routes ...

    return {
      api: api.url,
      userPoolId: userPool.id,
      userPoolClientId: userPoolClient.id,
      authUrl: domain.url,
    };
  },
});
```

A few things happening here:

**`usernames: ["email"]`** — users sign in with their email address, not a username. Cleaner UX, less friction.

**`preSignUp` trigger in non-production** — auto-confirms users without email verification. Invaluable for development and testing. You don't want to check your email every time you test the sign-up flow.

**`postConfirmation` trigger** — fires after a user confirms their account (clicks the verification link in their email). We use this to create the user's workspace automatically — one less step for the freelancer.

**`authorizer` on the API Gateway** — this applies Cognito JWT validation to every route by default. Any request without a valid `Authorization: Bearer <token>` header gets a 401 before your Lambda even runs.

### The postConfirmation trigger: automatic workspace creation

```typescript
// src/functions/auth/post-confirmation.ts
import type { PostConfirmationTriggerHandler } from "aws-lambda";
import { createWorkspace } from "../../repositories/workspaces";
import { createLogger } from "../../lib/logger";
import { nanoid } from "nanoid";

export const handler: PostConfirmationTriggerHandler = async (event) => {
  const logger = createLogger();

  // Only fire on sign-up confirmation, not on other triggers
  if (event.triggerSource !== "PostConfirmation_ConfirmSignUp") {
    return event;
  }

  const userId = event.request.userAttributes.sub;
  const email = event.request.userAttributes.email;
  const name = event.request.userAttributes.name ?? email.split("@")[0];

  logger.info("Creating workspace for new user", { userId, email });

  try {
    await createWorkspace({
      workspaceId: nanoid(),
      name: `${name}'s Workspace`,
      ownerId: userId,  // The Cognito sub — permanent, never changes
    });
  } catch (err) {
    // Log but don't throw — a failed workspace creation shouldn't fail the sign-up
    // The user can create their workspace on first login instead
    logger.error("Failed to create workspace for new user", {
      userId,
      err: String(err),
    });
  }

  // Cognito triggers must return the event
  return event;
};
```

The Cognito `sub` is the user's permanent, immutable identifier. Emails can change. Names can change. The `sub` never does. Always use it as the foreign key when scoping data to a user.

### The preSignUp trigger: skip verification in dev

```typescript
// src/functions/auth/pre-signup.ts
import type { PreSignUpTriggerHandler } from "aws-lambda";

export const handler: PreSignUpTriggerHandler = async (event) => {
  // Auto-confirm and auto-verify in non-production stages
  event.response.autoConfirmUser = true;
  event.response.autoVerifyEmail = true;
  return event;
};
```

Two lines. Saves you from checking your email inbox a hundred times during development.

---

## 7.3 JWT Validation in Lambda

When API Gateway validates a Cognito JWT, it injects the token's claims into the request context. Your Lambda handler receives them without making any additional calls — no `verifyJwt()`, no Cognito API call, no network round-trip.

The claims arrive at:

```typescript
event.requestContext.authorizer?.jwt?.claims
```

The shape:

```typescript
{
  sub: "a1b2c3d4-...",          // Cognito user ID — use this as the user identifier
  email: "mike@example.com",
  email_verified: "true",
  iss: "https://cognito-idp.eu-west-1.amazonaws.com/eu-west-1_xxx",
  aud: "6xxxxxxxxxxxxxxxxxxx",   // App client ID
  iat: "1740000000",
  exp: "1740003600",
  "cognito:username": "a1b2c3d4-...",
}
```

Build an auth utility that extracts the user from the event:

```typescript
// src/lib/auth.ts
import type { APIGatewayProxyEventV2WithJWTAuthorizer } from "aws-lambda";
import { AppError } from "./errors";

export interface AuthUser {
  sub: string;       // Cognito user ID
  email: string;
}

export function getAuthUser(
  event: APIGatewayProxyEventV2WithJWTAuthorizer,
): AuthUser {
  const claims = event.requestContext.authorizer?.jwt?.claims;

  if (!claims?.sub || !claims?.email) {
    // This shouldn't happen if API Gateway authorisation is configured
    // correctly — but be defensive
    throw new AppError("Missing auth claims", "UNAUTHORIZED", 401);
  }

  return {
    sub: String(claims.sub),
    email: String(claims.email),
  };
}
```

Note the event type: `APIGatewayProxyEventV2WithJWTAuthorizer` instead of the plain `APIGatewayProxyHandlerV2` we've been using. This variant has the `authorizer.jwt.claims` field typed. Import the handler type to match:

```typescript
import type {
  APIGatewayProxyHandlerV2WithJWTAuthorizer,
  APIGatewayProxyEventV2WithJWTAuthorizer,
} from "aws-lambda";
```

---

## 7.4 Protected Route Handlers

Update the workspace handler to use authentication:

```typescript
// src/functions/workspaces/get.ts
import type { APIGatewayProxyHandlerV2WithJWTAuthorizer } from "aws-lambda";
import { getAuthUser } from "../../lib/auth";
import { getWorkspaceByOwner } from "../../repositories/workspaces";
import { ok, forbidden, notFound } from "../../lib/response";
import { createLogger } from "../../lib/logger";

export const handler: APIGatewayProxyHandlerV2WithJWTAuthorizer = async (
  event,
  context,
) => {
  const logger = createLogger(context.awsRequestId);
  const user = getAuthUser(event);
  const workspaceId = event.pathParameters?.workspaceId!;

  const workspace = await getWorkspaceByOwner(workspaceId, user.sub);

  if (!workspace) {
    // Could be not found, or could belong to a different user.
    // Return 404 in both cases — don't leak existence of other workspaces.
    return notFound("Workspace");
  }

  logger.info("Workspace fetched", { workspaceId, userId: user.sub });
  return ok(workspace);
};
```

The key line: `getWorkspaceByOwner(workspaceId, user.sub)`. This fetches the workspace only if the `ownerId` matches the authenticated user's `sub`. A freelancer can't access another freelancer's workspace by guessing a workspace ID.

Add the scoped query to the workspace repository:

```typescript
// src/repositories/workspaces.ts (new function)
export async function getWorkspaceByOwner(
  workspaceId: string,
  ownerId: string,
): Promise<WorkspaceItem | null> {
  const workspace = await getWorkspace(workspaceId);

  // Check ownership — don't return the workspace if it belongs to someone else
  if (!workspace || workspace.ownerId !== ownerId) {
    return null;
  }

  return workspace;
}
```

### Handling 401 vs 403

These are different things. Use them correctly.

**401 Unauthorized** — "I don't know who you are." The request has no valid JWT, or the JWT is expired. The client should re-authenticate.

**403 Forbidden** — "I know who you are, and you don't have access to this." The JWT is valid, but the authenticated user doesn't own the resource.

In Runway's `getWorkspaceByOwner` above, we return `notFound` (404) rather than `forbidden` (403) when the workspace belongs to someone else. This is intentional — returning 403 confirms the workspace exists, which is information an attacker could use to enumerate valid workspace IDs. 404 is safer.

Return 403 when the user is authenticated but genuinely lacks permission for a non-sensitive action — like a client trying to access the workspace management section only freelancers can use:

```typescript
// src/lib/response.ts (additions)
export function unauthorized(message = "Unauthorized"): JsonResponse {
  return {
    statusCode: 401,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ error: "UNAUTHORIZED", message }),
  };
}

export function forbidden(message = "Forbidden"): JsonResponse {
  return {
    statusCode: 403,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ error: "FORBIDDEN", message }),
  };
}
```

---

## 7.5 User Identity in the Data Model

Every entity in Runway's DynamoDB table needs to be scoped to a workspace, and every workspace is owned by a Cognito user. This is the chain that enforces data isolation.

The pattern for every protected read:

```
Request with JWT
    ↓
API Gateway validates JWT → extracts claims
    ↓
Lambda: getAuthUser(event) → { sub, email }
    ↓
Fetch workspace by (workspaceId, ownerId: sub) — 404 if mismatch
    ↓
Fetch resource by (workspaceId, resourceId) — safe, bounded to this workspace
```

The workspace ownership check is the security boundary. Everything below it is safe because you've already verified the user owns the workspace.

A complete protected handler for listing clients:

```typescript
// src/functions/clients/list.ts
import type { APIGatewayProxyHandlerV2WithJWTAuthorizer } from "aws-lambda";
import { getAuthUser } from "../../lib/auth";
import { getWorkspaceByOwner } from "../../repositories/workspaces";
import { listClients } from "../../repositories/clients";
import { ok, notFound } from "../../lib/response";
import { createLogger } from "../../lib/logger";

export const handler: APIGatewayProxyHandlerV2WithJWTAuthorizer = async (
  event,
  context,
) => {
  const logger = createLogger(context.awsRequestId);
  const user = getAuthUser(event);
  const workspaceId = event.pathParameters?.workspaceId!;

  // Verify the workspace exists and belongs to this user
  const workspace = await getWorkspaceByOwner(workspaceId, user.sub);
  if (!workspace) return notFound("Workspace");

  // Now safe to query clients — they're scoped to this workspace
  const limit = Number(event.queryStringParameters?.limit ?? 50);
  const cursor = event.queryStringParameters?.cursor;

  const result = await listClients(workspaceId, {
    limit: Math.min(limit, 100),
    cursor,
  });

  logger.info("Clients listed", {
    workspaceId,
    userId: user.sub,
    count: result.items.length,
  });

  return ok(result);
};
```

### Multi-tenant: Runway's model

Runway is multi-tenant — multiple freelancers (tenants) share the same infrastructure but each sees only their own data.

The tenancy boundary in DynamoDB is the workspace. Every item has a `workspaceId`. Every query is scoped to a `workspaceId`. Workspace access is gated on `ownerId === user.sub`.

This is **workspace-level isolation** — the most common and practical model for SaaS. It doesn't require row-level security or separate databases per tenant. It requires discipline: every query must include the `workspaceId` in the key, and every workspace fetch must verify `ownerId`.

The alternative — separate DynamoDB tables per tenant, or separate AWS accounts — provides stronger isolation but is operationally brutal. For Runway, workspace-level isolation with strict ownership checks is correct.

---

## 7.6 The Client Portal

Runway's clients (the freelancer's customers) also need to log in. They see a restricted view: only their projects, only their invoices.

Clients use the same User Pool. They're differentiated by a custom attribute:

```typescript
// When the freelancer invites a client:
// src/functions/clients/invite.ts

import type { APIGatewayProxyHandlerV2WithJWTAuthorizer } from "aws-lambda";
import { CognitoIdentityProviderClient, AdminCreateUserCommand } from "@aws-sdk/client-cognito-identity-provider";
import { Resource } from "sst";
import { getAuthUser } from "../../lib/auth";
import { getWorkspaceByOwner } from "../../repositories/workspaces";
import { getClient } from "../../repositories/clients";
import { ok, notFound, badRequest } from "../../lib/response";

const cognito = new CognitoIdentityProviderClient({});

export const handler: APIGatewayProxyHandlerV2WithJWTAuthorizer = async (event) => {
  const user = getAuthUser(event);
  const workspaceId = event.pathParameters?.workspaceId!;
  const clientId = event.pathParameters?.clientId!;

  const workspace = await getWorkspaceByOwner(workspaceId, user.sub);
  if (!workspace) return notFound("Workspace");

  const client = await getClient(workspaceId, clientId);
  if (!client) return notFound("Client");

  // Create the client's Cognito user
  // AdminCreateUser sends them a temporary password via email automatically
  await cognito.send(
    new AdminCreateUserCommand({
      UserPoolId: Resource.RunwayUserPool.id,
      Username: client.email,
      UserAttributes: [
        { Name: "email", Value: client.email },
        { Name: "email_verified", Value: "true" },
        // Custom attributes to scope their access
        { Name: "custom:role", Value: "client" },
        { Name: "custom:workspaceId", Value: workspaceId },
        { Name: "custom:clientId", Value: clientId },
      ],
      DesiredDeliveryMediums: ["EMAIL"],
    }),
  );

  return ok({ message: `Invitation sent to ${client.email}` });
};
```

`AdminCreateUser` creates the user and sends them an email with a temporary password. Cognito forces them to change it on first login.

### Scoping client access in Lambda

When a client (not a freelancer) makes an API request, their JWT contains:

```json
{
  "sub": "b2c3d4e5-...",
  "email": "client@acmecorp.com",
  "custom:role": "client",
  "custom:workspaceId": "workspace_abc123",
  "custom:clientId": "client_xyz789"
}
```

Extend `getAuthUser` to extract these:

```typescript
// src/lib/auth.ts

export interface AuthUser {
  sub: string;
  email: string;
  role: "owner" | "client";
  workspaceId?: string;   // Set for client role
  clientId?: string;      // Set for client role
}

export function getAuthUser(
  event: APIGatewayProxyEventV2WithJWTAuthorizer,
): AuthUser {
  const claims = event.requestContext.authorizer?.jwt?.claims;

  if (!claims?.sub) {
    throw new AppError("Missing auth claims", "UNAUTHORIZED", 401);
  }

  const role = (claims["custom:role"] as string) ?? "owner";

  return {
    sub: String(claims.sub),
    email: String(claims.email ?? ""),
    role: role === "client" ? "client" : "owner",
    workspaceId: claims["custom:workspaceId"]
      ? String(claims["custom:workspaceId"])
      : undefined,
    clientId: claims["custom:clientId"]
      ? String(claims["custom:clientId"])
      : undefined,
  };
}
```

A client-aware handler for fetching projects:

```typescript
// src/functions/projects/list.ts
export const handler: APIGatewayProxyHandlerV2WithJWTAuthorizer = async (event) => {
  const user = getAuthUser(event);
  const clientId = event.pathParameters?.clientId!;

  if (user.role === "owner") {
    // Freelancer: verify they own the workspace containing this client
    const client = await getClientWithOwnerCheck(clientId, user.sub);
    if (!client) return notFound("Client");

    const projects = await listProjects(clientId);
    return ok(projects);
  }

  if (user.role === "client") {
    // Client user: can only see their own projects
    if (user.clientId !== clientId) {
      return notFound("Client"); // 404, not 403 — don't leak existence
    }

    const projects = await listProjects(clientId);
    return ok(projects);
  }

  return forbidden();
};
```

The role check is explicit. An owner can see any client in their workspace. A client can only see themselves. The fallthrough `forbidden()` handles any future roles that aren't accounted for.

---

## 7.7 Social Login

Adding Google or GitHub sign-in to Cognito is a federation question: "Let Cognito act as a broker between your user pool and an external identity provider."

### Google OAuth

In the AWS Console (or via SST):

1. Create OAuth credentials in [Google Cloud Console](https://console.cloud.google.com) — client ID and secret
2. Add Google as an identity provider in Cognito
3. Map Google attributes to Cognito attributes (email → email, sub → sub)

In SST:

```typescript
const google = new aws.cognito.IdentityProvider("GoogleProvider", {
  userPoolId: userPool.id,
  providerName: "Google",
  providerType: "Google",
  providerDetails: {
    client_id: Resource.GoogleClientId.value,
    client_secret: Resource.GoogleClientSecret.value,
    authorize_scopes: "email profile openid",
  },
  attributeMapping: {
    email: "email",
    name: "name",
    picture: "picture",
    username: "sub",
  },
});
```

The hosted UI automatically shows a "Sign in with Google" button when the identity provider is configured.

### Handling the callback

When a user signs in via Google, Cognito handles the OAuth flow and issues its own tokens. From your Lambda's perspective, the JWT looks the same as a password-based sign-in — same `sub`, same claims format.

There's one nuance: the `sub` for a federated user is `Google_<googleUserId>`, not a bare UUID. Your code doesn't need to care about this — `sub` is still the unique, stable identifier to use as your foreign key. But if you're logging it, don't be surprised by the format.

### Linking accounts

Problem: a user signs up with email/password, then later tries "Sign in with Google" with the same email. Cognito creates two separate users.

The `preSignUp` Lambda trigger handles this:

```typescript
// src/functions/auth/pre-signup.ts
import type { PreSignUpTriggerHandler } from "aws-lambda";
import {
  CognitoIdentityProviderClient,
  ListUsersCommand,
  AdminLinkProviderForUserCommand,
} from "@aws-sdk/client-cognito-identity-provider";

const cognito = new CognitoIdentityProviderClient({});

export const handler: PreSignUpTriggerHandler = async (event) => {
  // Auto-confirm in non-production (from the dev shortcut)
  if (process.env.SST_STAGE !== "production") {
    event.response.autoConfirmUser = true;
    event.response.autoVerifyEmail = true;
  }

  // Link federated sign-in to existing account with the same email
  if (event.triggerSource === "PreSignUp_ExternalProvider") {
    const email = event.request.userAttributes.email;

    const existing = await cognito.send(
      new ListUsersCommand({
        UserPoolId: event.userPoolId,
        Filter: `email = "${email}"`,
        Limit: 1,
      }),
    );

    if (existing.Users?.length) {
      const existingUser = existing.Users[0];

      // Link the Google account to the existing Cognito user
      await cognito.send(
        new AdminLinkProviderForUserCommand({
          UserPoolId: event.userPoolId,
          DestinationUser: {
            ProviderName: "Cognito",
            ProviderAttributeValue: existingUser.Username!,
          },
          SourceUser: {
            ProviderName: event.userName.split("_")[0], // "Google"
            ProviderAttributeName: "Cognito_Subject",
            ProviderAttributeValue: event.userName.split("_")[1],
          },
        }),
      );
    }
  }

  return event;
};
```

This merges the Google account into the existing Cognito user. After the link, both sign-in methods work and both result in the same `sub`.

---

## 7.8 Alternatives Worth Knowing

This chapter uses Cognito because it's native to AWS, serverless, and has no per-monthly-active-user cost up to 50,000 MAUs (after that it's $0.0055/MAU). For Runway, Cognito is the right call.

For completeness, the alternatives:

### Auth.js (NextAuth)

If Runway's frontend was Next.js, Auth.js would be a strong contender — it integrates natively with Next.js API routes and supports dozens of providers out of the box. For a decoupled TypeScript API on Lambda, it's more friction than it's worth.

### Clerk

Clerk is a hosted auth service with polished pre-built UI components, organisations, roles, and permissions. It charges per monthly active user (~$0.02 at most tiers). For teams that want to ship auth in an hour and don't want to think about it again, Clerk is genuinely excellent.

The trade-off: you're dependent on an external service. Clerk going down means your auth goes down. For most teams, this is an acceptable trade. For fintech or healthcare where you need data sovereignty, it isn't.

### Rolling your own JWT

You could issue your own JWTs, store user records in DynamoDB, and validate tokens in a Lambda middleware. It's not that much code.

Don't. You'll miss edge cases that cause security vulnerabilities. You'll implement password reset incorrectly. You'll forget to rotate signing keys. Authentication is the wrong thing to treat as a learning exercise. Use a managed service.

---

## Where We Are

Runway is now a properly authenticated multi-tenant SaaS. The architecture:

```
[Cognito User Pool]
  ├── Freelancer accounts (role: owner)
  └── Client accounts (role: client, scoped to workspaceId + clientId)
        ↓
[API Gateway — JWT Authoriser]
  ↓
[Lambda handlers]
  ├── getAuthUser() — extracts sub, email, role from JWT claims
  ├── getWorkspaceByOwner() — ownership check on every workspace access
  └── Role-based access — owners see everything in workspace, clients see their slice
        ↓
[DynamoDB + Aurora]
  └── All data scoped to workspaceId, workspaces gated on ownerId
```

The data model from Chapter 4 now has real users behind it. The `ownerId` on workspace items maps to a Cognito `sub`. File uploads from Chapter 6 will be prefixed with the `workspaceId` — which you can now verify against the authenticated user.

Chapter 8 adds the async layer: queues and background jobs. When an invoice is marked as paid, Runway should send a notification email. When a reminder job runs at 9am, it should chase outstanding invoices. None of that should happen synchronously in the API request — that's what Chapter 8 is for.

---

> **The code for this chapter** is available at `07-authentication-cognito/` in the companion repository.
