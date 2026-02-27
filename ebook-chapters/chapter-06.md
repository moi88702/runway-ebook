# Chapter 6: S3 and File Uploads

Runway needs two types of file storage.

The first: deliverables. Freelancers upload PDFs, design files, exported videos, and source assets to projects. Clients need to download them. These files are private — a client from workspace A must never be able to reach a file from workspace B.

The second: invoice PDFs. When a freelancer generates an invoice, Runway renders it server-side and stores it in S3. The freelancer can download it. When they send it to a client, the client gets a link that works for a limited time and then expires.

Both patterns use S3, but the data flows are different. Deliverables flow from the user's browser directly to S3 — Lambda generates an upload URL and steps out of the way. Invoice PDFs flow from Lambda to S3 — the function generates the PDF and stores it without involving the browser at all.

This chapter builds both. Along the way you'll see why file uploads in serverless architectures work differently from traditional web servers, how to lock down S3 assets to specific users, and how to wire S3 events into Lambda for post-upload processing.

By the end, Runway has a complete file storage layer. Chapter 7 adds authentication that scopes these assets to the logged-in user — the groundwork you're laying here makes that straightforward.

---

## 6.1 S3 Fundamentals in TypeScript

### The mental model

S3 is a flat key-value store. There are no real directories — only keys that happen to contain `/` characters, which the AWS Console renders as a folder hierarchy for your convenience. Under the hood, `workspaces/ws-01/projects/p-01/deliverables/file.pdf` is a single string key pointing to a single object.

This matters because there's no concept of "list the contents of this directory" at the filesystem level. When you list objects with a prefix, S3 scans keys that start with that string. It's efficient (S3 indexes keys), but understanding the model prevents surprises when you're designing key structures.

**Buckets** are globally-unique namespaced containers. The name you choose must be unique across all AWS accounts and all regions. `runway-deliverables` is almost certainly taken. `runway-deliverables-{accountId}-{region}` is not.

**Objects** are the files. Each object has a key, metadata, and optionally a version ID if versioning is enabled.

**Keys** are the object identifiers — think of them as file paths. Choose key structures that give you useful prefixes for listing and that enforce your security model through naming conventions.

### Key design for Runway

The key structure encodes your access control. Design it first.

For deliverables:
```
workspaces/{workspaceId}/projects/{projectId}/deliverables/{fileId}/{filename}
```

For invoice PDFs:
```
workspaces/{workspaceId}/invoices/{invoiceId}/invoice-{invoiceNumber}.pdf
```

For processing artifacts (thumbnails, extracted metadata):
```
workspaces/{workspaceId}/projects/{projectId}/deliverables/{fileId}/thumbnails/thumb-{size}.jpg
workspaces/{workspaceId}/projects/{projectId}/deliverables/{fileId}/metadata.json
```

Why is this structure important? Because when Lambda generates a presigned download URL, it first checks that the requesting user belongs to `workspaceId`. The key structure guarantees that any object the user can validly access starts with their `workspaceId`. A user who somehow requests a signed URL for a key not in their workspace is caught before the URL is generated.

An alternative pattern — flat keys like `{fileId}.pdf` with metadata in DynamoDB — is technically workable but requires a separate DynamoDB lookup to verify every download. The hierarchical key structure bakes the security model into the keys themselves.

### Bucket configuration in SST

Runway uses two private buckets:

```typescript
// sst.config.ts

const deliverablesBucket = new sst.aws.Bucket("RunwayDeliverables", {
  cors: [
    {
      allowedHeaders: ["*"],
      allowedMethods: ["GET", "PUT", "POST"],
      allowedOrigins: ["https://app.runway.so", "http://localhost:3000"],
      exposedHeaders: ["ETag"],
      maxAge: 3000,
    },
  ],
  versioning: false,   // Deliverables don't need version history
  transform: {
    bucket: {
      forceDestroy: $app.stage !== "production",
    },
  },
});

const invoicesBucket = new sst.aws.Bucket("RunwayInvoices", {
  versioning: false,
  transform: {
    bucket: {
      forceDestroy: $app.stage !== "production",
    },
  },
});
```

Neither bucket is public. There is no `public: true` flag. Every read goes through a Lambda that validates access and issues a time-limited signed URL. This is the correct default for user-owned assets.

The CORS configuration on `deliverablesBucket` allows the browser to PUT directly to S3. Without it, the browser's preflight check will fail. The `exposedHeaders: ["ETag"]` line exposes the ETag header so the client can confirm the upload completed — we'll use this in section 6.3.

`forceDestroy: true` on non-production stages means `sst remove` can tear down the bucket even if it has objects. Without this, you'd have to manually empty the bucket before SST could delete it. In production, `forceDestroy: false` means you cannot accidentally destroy a bucket with client data in it.

### Linking buckets to functions

```typescript
// sst.config.ts — API routes for file operations

api.route("POST /workspaces/{workspaceId}/projects/{projectId}/deliverables/upload-url", {
  handler: "src/functions/deliverables/create-upload-url.handler",
  link: [deliverablesBucket],
});

api.route("POST /workspaces/{workspaceId}/projects/{projectId}/deliverables/{deliverableId}/confirm", {
  handler: "src/functions/deliverables/confirm-upload.handler",
  link: [deliverablesBucket, table],
});

api.route("GET /workspaces/{workspaceId}/projects/{projectId}/deliverables/{deliverableId}/download", {
  handler: "src/functions/deliverables/get-download-url.handler",
  link: [deliverablesBucket, table],
});

api.route("GET /workspaces/{workspaceId}/invoices/{invoiceId}/download", {
  handler: "src/functions/invoices/get-download-url.handler",
  link: [invoicesBucket, table],
});
```

`link: [deliverablesBucket]` injects `Resource.RunwayDeliverables.name` (the bucket name) into the Lambda's environment and grants the function permission to call S3 operations on that bucket. No manual IAM policy required.

### Installing the AWS SDK

```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```

The S3 client lives in `src/lib/s3.ts`:

```typescript
// src/lib/s3.ts
import { S3Client } from "@aws-sdk/client-s3";

// Module-level — reused across warm invocations
const s3Client = new S3Client({});

export { s3Client };
```

One client instance, created at module level. This is the lazy initialisation pattern from Chapter 2: the AWS SDK client establishes a connection pool once and reuses it across warm Lambda invocations.

---

## 6.2 Direct Uploads with Presigned URLs

### Why Lambda should not touch file bytes

The instinct when building an upload endpoint is to POST the file to your API and let the Lambda write it to S3. Don't.

Lambda has a payload limit of 6MB for synchronous invocations via API Gateway. Most design files, exported PDFs, and video assets are larger than that. You could use API Gateway's binary media types and work around the limit, but you're creating complexity to solve a problem that doesn't need to exist.

More importantly: routing file data through Lambda means paying for Lambda execution time while data is in transit. If a user uploads a 50MB file over a slow connection, your Lambda function is running for 30 seconds doing nothing but waiting for bytes. Lambda bills per millisecond.

The right approach is to have the browser talk to S3 directly. Lambda's only role is to generate a short-lived signed URL authorising the upload, then get out of the way. The upload happens browser → S3, with no Lambda in the middle.

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Browser  ──1. POST /upload-url──▶  Lambda                          │
│                                         │                           │
│             ◀──2. {uploadUrl, key}──────┘                           │
│                                                                     │
│  Browser  ──3. PUT {uploadUrl} (file bytes) ──▶  S3                 │
│                                                                     │
│  Browser  ──4. POST /confirm ──────────────▶  Lambda                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Lambda handles step 1 (generate the URL) and step 4 (confirm the upload happened). S3 handles step 3 (receive the actual bytes). Lambda never sees the file.

### Presigned PUT vs presigned POST

AWS S3 supports two styles of presigned upload URL:

**Presigned PUT** — A signed URL for a single PUT request. The client PUTs the file directly to this URL. Simple, easy to implement in the browser with `fetch`.

**Presigned POST** — A signed form POST with fields embedded in the response. More complex client-side, but supports server-side constraints: content type restrictions, size limits, and conditions that are enforced by S3 before accepting the upload.

For Runway, use presigned PUT for its simplicity and use server-side validation for content type and size. The size limit is enforced by creating the metadata record first with an expected size and then validating the actual uploaded object size in the post-upload Lambda trigger.

### Generating presigned PUT URLs

```typescript
// src/functions/deliverables/create-upload-url.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { Resource } from "sst";
import { randomUUID } from "crypto";
import { z } from "zod";
import { s3Client } from "../../lib/s3";
import { ok, badRequest, forbidden } from "../../lib/response";
import { createLogger } from "../../lib/logger";
import { getWorkspaceMembership } from "../../repositories/workspaces";

const ALLOWED_CONTENT_TYPES = new Set([
  "application/pdf",
  "image/jpeg",
  "image/png",
  "image/gif",
  "image/webp",
  "image/svg+xml",
  "video/mp4",
  "video/quicktime",
  "application/zip",
  "application/x-zip-compressed",
  "application/illustrator",
  "application/postscript",
  "application/vnd.adobe.photoshop",
  "application/sketch",
  "application/figma",
  "font/otf",
  "font/ttf",
  "font/woff",
  "font/woff2",
]);

const MAX_FILE_SIZE_BYTES = 500 * 1024 * 1024; // 500MB

const requestSchema = z.object({
  filename: z.string().min(1).max(255),
  contentType: z.string().min(1),
  sizeBytes: z.number().int().positive().max(MAX_FILE_SIZE_BYTES),
});

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const logger = createLogger(context.awsRequestId);

  const workspaceId = event.pathParameters?.workspaceId;
  const projectId = event.pathParameters?.projectId;

  if (!workspaceId || !projectId) {
    return badRequest("Missing path parameters");
  }

  // Parse and validate request body
  let body: z.infer<typeof requestSchema>;
  try {
    body = requestSchema.parse(JSON.parse(event.body ?? "{}"));
  } catch {
    return badRequest("Invalid request body");
  }

  const { filename, contentType, sizeBytes } = body;

  // Validate content type
  if (!ALLOWED_CONTENT_TYPES.has(contentType)) {
    return badRequest(`Content type not allowed: ${contentType}`);
  }

  // In Chapter 7 (Cognito auth), this comes from the JWT.
  // For now, we expect it in a header for development purposes.
  const userId = event.headers["x-user-id"];
  if (!userId) {
    return forbidden("Authentication required");
  }

  // Verify the user belongs to this workspace
  const membership = await getWorkspaceMembership(workspaceId, userId);
  if (!membership) {
    return forbidden("Not a member of this workspace");
  }

  // Generate a stable, unique key for this deliverable
  const fileId = randomUUID();
  const sanitisedFilename = sanitiseFilename(filename);
  const key = `workspaces/${workspaceId}/projects/${projectId}/deliverables/${fileId}/${sanitisedFilename}`;

  // Generate presigned URL valid for 15 minutes
  const command = new PutObjectCommand({
    Bucket: Resource.RunwayDeliverables.name,
    Key: key,
    ContentType: contentType,
    ContentLength: sizeBytes,
    // Metadata stored with the object — available without downloading the file
    Metadata: {
      "workspace-id": workspaceId,
      "project-id": projectId,
      "file-id": fileId,
      "uploaded-by": userId,
      "original-filename": sanitisedFilename,
    },
  });

  const uploadUrl = await getSignedUrl(s3Client, command, {
    expiresIn: 15 * 60, // 15 minutes in seconds
  });

  logger.info("Generated upload URL", {
    workspaceId,
    projectId,
    fileId,
    contentType,
    sizeBytes,
  });

  return ok({
    uploadUrl,
    key,
    fileId,
    expiresIn: 15 * 60,
  });
};

function sanitiseFilename(filename: string): string {
  return filename
    .replace(/[^a-zA-Z0-9._-]/g, "-")  // Replace unsafe chars with hyphen
    .replace(/-+/g, "-")                // Collapse multiple hyphens
    .replace(/^-|-$/g, "")             // Strip leading/trailing hyphens
    .slice(0, 200);                    // Enforce a max length
}
```

A few things to note:

The `fileId` is generated here, in the server. Not by the client. The client never gets to choose their own file ID. If a client generated the ID, they could potentially guess or enumerate IDs from other workspaces (though the workspace-scoped key prefix would still protect downloads).

`ContentLength` is set on the presigned URL command. This means S3 will reject PUT requests that don't match the declared size. It's not a hard security control (the client could lie about the size in the initial request), but it prevents accidental misuse.

The metadata stored with the object (`Metadata` field) is available without downloading the file — you can retrieve it with a `HeadObjectCommand`. This is useful in the processing Lambda: you can check the workspace, project, and file ID from object metadata without maintaining a separate mapping.

The 15-minute expiry is a judgement call. Short enough that a leaked URL isn't useful for long. Long enough that a user on a slow connection can complete the upload. You could make this configurable per content type (longer for large video files, shorter for PDFs).

### Generating presigned GET URLs

Download URLs work the same way — sign a `GetObjectCommand` instead:

```typescript
// src/functions/deliverables/get-download-url.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { GetObjectCommand, HeadObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { Resource } from "sst";
import { s3Client } from "../../lib/s3";
import { ok, notFound, forbidden, serverError } from "../../lib/response";
import { createLogger } from "../../lib/logger";
import { getDeliverable } from "../../repositories/deliverables";
import { getWorkspaceMembership } from "../../repositories/workspaces";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const logger = createLogger(context.awsRequestId);

  const workspaceId = event.pathParameters?.workspaceId;
  const projectId = event.pathParameters?.projectId;
  const deliverableId = event.pathParameters?.deliverableId;

  if (!workspaceId || !projectId || !deliverableId) {
    return notFound("Not found");
  }

  const userId = event.headers["x-user-id"];
  if (!userId) {
    return forbidden("Authentication required");
  }

  // Verify workspace membership
  const membership = await getWorkspaceMembership(workspaceId, userId);
  if (!membership) {
    return forbidden("Access denied");
  }

  // Look up the deliverable metadata from DynamoDB
  const deliverable = await getDeliverable(workspaceId, projectId, deliverableId);
  if (!deliverable) {
    return notFound("Deliverable not found");
  }

  // The key is derived from the workspaceId — a user from another workspace
  // cannot construct a valid key that passes the membership check above
  const key = deliverable.s3Key;

  // Verify the key belongs to this workspace (belt-and-suspenders)
  if (!key.startsWith(`workspaces/${workspaceId}/`)) {
    logger.error("Key workspace mismatch", { key, workspaceId, deliverableId });
    return forbidden("Access denied");
  }

  // Check the object still exists (it might have been deleted or not yet uploaded)
  try {
    await s3Client.send(
      new HeadObjectCommand({
        Bucket: Resource.RunwayDeliverables.name,
        Key: key,
      })
    );
  } catch (err: unknown) {
    if ((err as { name?: string }).name === "NotFound") {
      return notFound("File not found");
    }
    throw err;
  }

  // Generate a presigned GET URL valid for 1 hour
  const command = new GetObjectCommand({
    Bucket: Resource.RunwayDeliverables.name,
    Key: key,
    // Force browser to download rather than display inline
    ResponseContentDisposition: `attachment; filename="${deliverable.filename}"`,
  });

  const downloadUrl = await getSignedUrl(s3Client, command, {
    expiresIn: 60 * 60, // 1 hour
  });

  logger.info("Generated download URL", {
    workspaceId,
    deliverableId,
    userId,
  });

  return ok({
    downloadUrl,
    filename: deliverable.filename,
    contentType: deliverable.contentType,
    sizeBytes: deliverable.sizeBytes,
    expiresIn: 60 * 60,
  });
};
```

The security model here is layered:

1. The user must authenticate (Chapter 7 replaces the header with a JWT check)
2. The user must be a member of the workspace
3. The deliverable must exist in DynamoDB and belong to that workspace/project
4. The S3 key is retrieved from DynamoDB, not from the request — the user can't specify an arbitrary key
5. Belt-and-suspenders: we verify the key prefix even though it came from our own database

Step 5 feels paranoid. It protects against a bug in `getDeliverable` that returns the wrong record, or a database injection scenario where an attacker gets a crafted key into DynamoDB. The check is a millisecond of string comparison that prevents an entire class of security bug.

### URL expiry considerations

**Upload URLs (15 minutes):** Short because the upload should happen immediately after the URL is generated. If someone gets this URL, they can only upload a file with the exact content type and size specified. They can't read existing content.

**Download URLs (1 hour):** Longer because the user might share the link or open it in a download manager. The risk of a leaked download URL is that someone else can download the file for up to an hour.

**Invoice URLs (24 hours or more):** Invoice PDFs are often forwarded to accountants or clients by email. A 1-hour URL breaks in those scenarios. Use 24-72 hours for invoice downloads. The file is not highly sensitive — it's a financial document sent to clients deliberately.

If you need indefinite access (a client portal link that never expires), the pattern is different: generate a short-lived URL on every page load rather than a persistent long-lived URL. This keeps URLs short-lived while giving the appearance of permanent access.

---

## 6.3 Upload Flows in Practice

### The complete client-side flow

Here's the complete flow from the client's perspective. This is framework-agnostic TypeScript — adapt it to your frontend framework.

```typescript
// client-side upload logic (browser/frontend)

interface UploadRequest {
  file: File;
  workspaceId: string;
  projectId: string;
  apiBaseUrl: string;
  authToken: string; // Chapter 7 JWT token
  onProgress?: (percent: number) => void;
}

interface UploadResult {
  fileId: string;
  key: string;
  filename: string;
  contentType: string;
  sizeBytes: number;
}

export async function uploadDeliverable(req: UploadRequest): Promise<UploadResult> {
  const { file, workspaceId, projectId, apiBaseUrl, authToken, onProgress } = req;

  // Step 1: Request a presigned upload URL from the API
  const urlResponse = await fetch(
    `${apiBaseUrl}/workspaces/${workspaceId}/projects/${projectId}/deliverables/upload-url`,
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${authToken}`,
      },
      body: JSON.stringify({
        filename: file.name,
        contentType: file.type,
        sizeBytes: file.size,
      }),
    }
  );

  if (!urlResponse.ok) {
    const error = await urlResponse.json();
    throw new Error(`Failed to get upload URL: ${error.message}`);
  }

  const { uploadUrl, key, fileId } = await urlResponse.json();

  // Step 2: Upload the file directly to S3 using the presigned URL
  const etag = await uploadToS3({
    uploadUrl,
    file,
    onProgress,
  });

  // Step 3: Confirm the upload to the API (creates the DynamoDB record)
  const confirmResponse = await fetch(
    `${apiBaseUrl}/workspaces/${workspaceId}/projects/${projectId}/deliverables/${fileId}/confirm`,
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${authToken}`,
      },
      body: JSON.stringify({
        key,
        etag,
        filename: file.name,
        contentType: file.type,
        sizeBytes: file.size,
      }),
    }
  );

  if (!confirmResponse.ok) {
    const error = await confirmResponse.json();
    throw new Error(`Failed to confirm upload: ${error.message}`);
  }

  const deliverable = await confirmResponse.json();

  return {
    fileId,
    key,
    filename: file.name,
    contentType: file.type,
    sizeBytes: file.size,
    ...deliverable,
  };
}

async function uploadToS3(params: {
  uploadUrl: string;
  file: File;
  onProgress?: (percent: number) => void;
}): Promise<string> {
  const { uploadUrl, file, onProgress } = params;

  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();

    // XHR is used instead of fetch because fetch doesn't support upload
    // progress events natively in all environments
    if (onProgress) {
      xhr.upload.addEventListener("progress", (event) => {
        if (event.lengthComputable) {
          const percent = Math.round((event.loaded / event.total) * 100);
          onProgress(percent);
        }
      });
    }

    xhr.addEventListener("load", () => {
      if (xhr.status === 200) {
        // ETag is returned by S3 on successful PUT — it's the MD5 of the uploaded content
        const etag = xhr.getResponseHeader("ETag")?.replace(/"/g, "") ?? "";
        resolve(etag);
      } else {
        reject(new Error(`S3 upload failed with status ${xhr.status}: ${xhr.responseText}`));
      }
    });

    xhr.addEventListener("error", () => {
      reject(new Error("Network error during upload"));
    });

    xhr.addEventListener("abort", () => {
      reject(new Error("Upload aborted"));
    });

    xhr.open("PUT", uploadUrl);
    xhr.setRequestHeader("Content-Type", file.type);
    xhr.send(file);
  });
}
```

The `XMLHttpRequest` is used instead of `fetch` for the S3 PUT because `fetch` doesn't expose upload progress events in a cross-browser way. If progress tracking isn't required, `fetch` works fine:

```typescript
// Simpler version without progress tracking
async function uploadToS3Simple(uploadUrl: string, file: File): Promise<string> {
  const response = await fetch(uploadUrl, {
    method: "PUT",
    headers: { "Content-Type": file.type },
    body: file,
  });

  if (!response.ok) {
    throw new Error(`S3 upload failed: ${response.statusText}`);
  }

  return response.headers.get("ETag")?.replace(/"/g, "") ?? "";
}
```

### The confirm endpoint

The confirm step is not optional. Here's why:

Without a confirm step, you have an S3 bucket full of objects and no DynamoDB record saying "this file is uploaded and belongs to project P." The upload happened but your application doesn't know about it. You need a database record to:
- Show the file in the project's deliverables list
- Associate it with the correct project and workspace
- Track metadata (upload time, who uploaded it)
- Gate access through the download URL endpoint

The confirm Lambda reads the object metadata from S3 (verifying the upload actually happened), creates the DynamoDB record, and returns the deliverable data to the client:

```typescript
// src/functions/deliverables/confirm-upload.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { HeadObjectCommand } from "@aws-sdk/client-s3";
import { Resource } from "sst";
import { z } from "zod";
import { s3Client } from "../../lib/s3";
import { ok, badRequest, forbidden, notFound, conflict } from "../../lib/response";
import { createLogger } from "../../lib/logger";
import { createDeliverable, getDeliverable } from "../../repositories/deliverables";
import { getWorkspaceMembership } from "../../repositories/workspaces";
import { randomUUID } from "crypto";

const requestSchema = z.object({
  key: z.string().min(1),
  etag: z.string().optional(),
  filename: z.string().min(1).max(255),
  contentType: z.string().min(1),
  sizeBytes: z.number().int().positive(),
});

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const logger = createLogger(context.awsRequestId);

  const workspaceId = event.pathParameters?.workspaceId;
  const projectId = event.pathParameters?.projectId;
  const deliverableId = event.pathParameters?.deliverableId;

  if (!workspaceId || !projectId || !deliverableId) {
    return badRequest("Missing path parameters");
  }

  const userId = event.headers["x-user-id"];
  if (!userId) {
    return forbidden("Authentication required");
  }

  const membership = await getWorkspaceMembership(workspaceId, userId);
  if (!membership) {
    return forbidden("Access denied");
  }

  let body: z.infer<typeof requestSchema>;
  try {
    body = requestSchema.parse(JSON.parse(event.body ?? "{}"));
  } catch {
    return badRequest("Invalid request body");
  }

  // Verify the key belongs to this workspace and project
  const expectedKeyPrefix = `workspaces/${workspaceId}/projects/${projectId}/deliverables/${deliverableId}/`;
  if (!body.key.startsWith(expectedKeyPrefix)) {
    return forbidden("Invalid key");
  }

  // Verify the object actually exists in S3
  let s3Metadata: Record<string, string>;
  try {
    const head = await s3Client.send(
      new HeadObjectCommand({
        Bucket: Resource.RunwayDeliverables.name,
        Key: body.key,
      })
    );
    s3Metadata = head.Metadata ?? {};
  } catch (err: unknown) {
    if ((err as { name?: string }).name === "NotFound") {
      return notFound("File not found in S3 — upload may have failed or not yet completed");
    }
    throw err;
  }

  // Guard against duplicate confirms (idempotency)
  const existing = await getDeliverable(workspaceId, projectId, deliverableId);
  if (existing) {
    // Already confirmed — return the existing record (idempotent)
    return ok(existing);
  }

  // Create the DynamoDB record
  const deliverable = await createDeliverable({
    workspaceId,
    projectId,
    deliverableId,
    s3Key: body.key,
    filename: body.filename,
    contentType: body.contentType,
    sizeBytes: body.sizeBytes,
    uploadedBy: userId,
    etag: body.etag ?? s3Metadata["x-amz-meta-etag"],
    status: "active",
    createdAt: new Date().toISOString(),
  });

  logger.info("Deliverable confirmed", {
    workspaceId,
    projectId,
    deliverableId,
    filename: body.filename,
    sizeBytes: body.sizeBytes,
  });

  return ok(deliverable);
};
```

The `HeadObjectCommand` call verifies the object exists without downloading it. It returns object metadata (the `Metadata` map we set on the presigned URL command) and size information. If S3 doesn't have the object, the upload failed and we return a 404 with a clear message.

The idempotency guard matters: if the client's confirm request times out and they retry, they shouldn't get an error — they should get the existing record back. Idempotent operations are a recurring theme in distributed systems. Always ask: "What happens if this is called twice?"

### The DynamoDB deliverables repository

```typescript
// src/repositories/deliverables.ts
import {
  PutCommand,
  GetCommand,
  QueryCommand,
  DeleteCommand,
  UpdateCommand,
} from "@aws-sdk/lib-dynamodb";
import { Resource } from "sst";
import { docClient } from "../lib/dynamo";

interface Deliverable {
  workspaceId: string;
  projectId: string;
  deliverableId: string;
  s3Key: string;
  filename: string;
  contentType: string;
  sizeBytes: number;
  uploadedBy: string;
  etag?: string;
  status: "active" | "processing" | "deleted";
  thumbnailKey?: string;    // Set by the processing Lambda
  pageCount?: number;       // For PDFs
  dimensions?: { width: number; height: number }; // For images
  createdAt: string;
  updatedAt?: string;
}

// DynamoDB key structure:
// pk: WORKSPACE#{workspaceId}
// sk: DELIVERABLE#{projectId}#{deliverableId}

export async function createDeliverable(deliverable: Deliverable): Promise<Deliverable> {
  await docClient.send(
    new PutCommand({
      TableName: Resource.RunwayTable.name,
      Item: {
        pk: `WORKSPACE#${deliverable.workspaceId}`,
        sk: `DELIVERABLE#${deliverable.projectId}#${deliverable.deliverableId}`,
        entityType: "DELIVERABLE",
        ...deliverable,
      },
      ConditionExpression: "attribute_not_exists(pk)", // Don't overwrite existing
    })
  );

  return deliverable;
}

export async function getDeliverable(
  workspaceId: string,
  projectId: string,
  deliverableId: string
): Promise<Deliverable | null> {
  const result = await docClient.send(
    new GetCommand({
      TableName: Resource.RunwayTable.name,
      Key: {
        pk: `WORKSPACE#${workspaceId}`,
        sk: `DELIVERABLE#${projectId}#${deliverableId}`,
      },
    })
  );

  return (result.Item as Deliverable) ?? null;
}

export async function listProjectDeliverables(
  workspaceId: string,
  projectId: string
): Promise<Deliverable[]> {
  const result = await docClient.send(
    new QueryCommand({
      TableName: Resource.RunwayTable.name,
      KeyConditionExpression: "pk = :pk AND begins_with(sk, :skPrefix)",
      ExpressionAttributeValues: {
        ":pk": `WORKSPACE#${workspaceId}`,
        ":skPrefix": `DELIVERABLE#${projectId}#`,
      },
    })
  );

  return (result.Items ?? []) as Deliverable[];
}

export async function updateDeliverableMetadata(
  workspaceId: string,
  projectId: string,
  deliverableId: string,
  updates: Partial<Pick<Deliverable, "thumbnailKey" | "pageCount" | "dimensions" | "status" | "updatedAt">>
): Promise<void> {
  const setExpressions: string[] = [];
  const expressionAttributeNames: Record<string, string> = {};
  const expressionAttributeValues: Record<string, unknown> = {};

  for (const [key, value] of Object.entries(updates)) {
    if (value !== undefined) {
      const nameToken = `#${key}`;
      const valueToken = `:${key}`;
      setExpressions.push(`${nameToken} = ${valueToken}`);
      expressionAttributeNames[nameToken] = key;
      expressionAttributeValues[valueToken] = value;
    }
  }

  if (setExpressions.length === 0) return;

  await docClient.send(
    new UpdateCommand({
      TableName: Resource.RunwayTable.name,
      Key: {
        pk: `WORKSPACE#${workspaceId}`,
        sk: `DELIVERABLE#${projectId}#${deliverableId}`,
      },
      UpdateExpression: `SET ${setExpressions.join(", ")}`,
      ExpressionAttributeNames: expressionAttributeNames,
      ExpressionAttributeValues: expressionAttributeValues,
    })
  );
}
```

---

## 6.4 S3 Event Triggers

When a file lands in S3, you often want to do something with it. For Runway:

- Uploaded images → generate a thumbnail for the project deliverables gallery
- Uploaded PDFs → extract page count for display
- Uploaded design files → store content type metadata

This is exactly what S3 event notifications are for: a Lambda function subscribed to the bucket fires on every object creation.

### Wiring up the S3 trigger in SST

```typescript
// sst.config.ts
const deliverablesBucket = new sst.aws.Bucket("RunwayDeliverables", {
  // ... (same as before)
});

// Processor Lambda — triggered by S3 object creation
deliverablesBucket.subscribe(
  {
    handler: "src/functions/deliverables/process-upload.handler",
    link: [deliverablesBucket, table],
    timeout: "2 minutes",  // Processing can take time — give it room
    memory: "1024 MB",     // Sharp (image resizing) needs memory
  },
  {
    // Only fire on object creation events
    events: ["s3:ObjectCreated:*"],
    // Only process files in the deliverables path (not thumbnails or metadata)
    filterPrefix: "workspaces/",
    filterSuffix: undefined, // Process all file types
  }
);
```

SST's `bucket.subscribe()` creates the Lambda, the S3 event notification, and the necessary IAM permissions in one call. Without SST, this requires an SNS topic or direct Lambda notification, IAM permission grants, and careful ordering of resource creation to avoid circular dependencies.

The `timeout: "2 minutes"` is intentional — image resizing and PDF inspection can take tens of seconds for large files. The default Lambda timeout of 3 seconds is not enough.

The `filterPrefix: "workspaces/"` ensures the trigger only fires on actual deliverable uploads, not on the thumbnail files the processor itself creates. Without this, every thumbnail write would trigger the processor again: an infinite loop.

### The processing Lambda

```typescript
// src/functions/deliverables/process-upload.ts
import type { S3Handler, S3Event } from "aws-lambda";
import {
  GetObjectCommand,
  PutObjectCommand,
  HeadObjectCommand,
} from "@aws-sdk/client-s3";
import { Resource } from "sst";
import { s3Client } from "../../lib/s3";
import { createLogger } from "../../lib/logger";
import { updateDeliverableMetadata } from "../../repositories/deliverables";
import { getObjectStream, streamToBuffer } from "../../lib/s3-utils";

export const handler: S3Handler = async (event: S3Event) => {
  const logger = createLogger("s3-processor");

  // S3 can batch multiple events in one invocation
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, " "));
    const sizeBytes = record.s3.object.size;

    logger.info("Processing upload", { bucket, key, sizeBytes });

    try {
      await processObject(bucket, key, sizeBytes, logger);
    } catch (err) {
      // Log the error but don't re-throw — see note below
      logger.error("Failed to process upload", {
        bucket,
        key,
        err: String(err),
      });
      // In production, push to a DLQ or update the deliverable status to "failed"
    }
  }
};

async function processObject(
  bucket: string,
  key: string,
  sizeBytes: number,
  logger: ReturnType<typeof createLogger>
): Promise<void> {
  // Parse workspace/project/deliverable from the key structure
  const parsed = parseDeliverableKey(key);
  if (!parsed) {
    logger.warn("Could not parse deliverable key — skipping", { key });
    return;
  }

  const { workspaceId, projectId, deliverableId } = parsed;

  // Get the object metadata (without downloading the body)
  const head = await s3Client.send(
    new HeadObjectCommand({ Bucket: bucket, Key: key })
  );

  const contentType = head.ContentType ?? "application/octet-stream";
  const metadata: Record<string, unknown> = {};

  // Process based on content type
  if (isImage(contentType)) {
    logger.info("Processing image", { key, contentType });

    try {
      const thumbnailKey = await generateThumbnail(bucket, key, workspaceId, projectId, deliverableId);
      metadata.thumbnailKey = thumbnailKey;

      // Get image dimensions using sharp
      const { width, height } = await getImageDimensions(bucket, key);
      metadata.dimensions = { width, height };
    } catch (err) {
      logger.warn("Thumbnail generation failed", { key, err: String(err) });
      // Non-fatal — continue to update other metadata
    }
  } else if (contentType === "application/pdf") {
    logger.info("Processing PDF", { key });

    try {
      const pageCount = await getPdfPageCount(bucket, key);
      metadata.pageCount = pageCount;
    } catch (err) {
      logger.warn("PDF page count extraction failed", { key, err: String(err) });
    }
  }

  // Update the DynamoDB record with extracted metadata
  await updateDeliverableMetadata(workspaceId, projectId, deliverableId, {
    ...metadata,
    status: "active",
    updatedAt: new Date().toISOString(),
  });

  logger.info("Processing complete", {
    workspaceId,
    projectId,
    deliverableId,
    metadata,
  });
}

function parseDeliverableKey(key: string): {
  workspaceId: string;
  projectId: string;
  deliverableId: string;
  filename: string;
} | null {
  // Expected format: workspaces/{workspaceId}/projects/{projectId}/deliverables/{deliverableId}/{filename}
  const match = key.match(
    /^workspaces\/([^/]+)\/projects\/([^/]+)\/deliverables\/([^/]+)\/(.+)$/
  );

  if (!match) return null;

  return {
    workspaceId: match[1],
    projectId: match[2],
    deliverableId: match[3],
    filename: match[4],
  };
}

function isImage(contentType: string): boolean {
  return contentType.startsWith("image/") && contentType !== "image/svg+xml";
}
```

### Thumbnail generation with Sharp

```typescript
// src/functions/deliverables/process-upload.ts (continued)
import Sharp from "sharp";

const THUMBNAIL_SIZES = [
  { name: "sm", width: 120, height: 90 },
  { name: "md", width: 400, height: 300 },
  { name: "lg", width: 800, height: 600 },
] as const;

async function generateThumbnail(
  bucket: string,
  key: string,
  workspaceId: string,
  projectId: string,
  deliverableId: string
): Promise<string> {
  // Download the original image
  const originalObject = await s3Client.send(
    new GetObjectCommand({ Bucket: bucket, Key: key })
  );

  const imageBuffer = await streamToBuffer(originalObject.Body as NodeJS.ReadableStream);

  // Generate the medium thumbnail
  const { name: sizeName, width, height } = THUMBNAIL_SIZES[1]; // "md"

  const thumbnailBuffer = await Sharp(imageBuffer)
    .resize(width, height, {
      fit: "inside",         // Maintain aspect ratio, fit within dimensions
      withoutEnlargement: true, // Don't upscale small images
    })
    .jpeg({ quality: 85, progressive: true })
    .toBuffer();

  const thumbnailKey = `workspaces/${workspaceId}/projects/${projectId}/deliverables/${deliverableId}/thumbnails/thumb-${sizeName}.jpg`;

  await s3Client.send(
    new PutObjectCommand({
      Bucket: bucket,
      Key: thumbnailKey,
      Body: thumbnailBuffer,
      ContentType: "image/jpeg",
      // Thumbnails can be cached aggressively — they don't change
      CacheControl: "public, max-age=31536000, immutable",
    })
  );

  return thumbnailKey;
}

async function getImageDimensions(
  bucket: string,
  key: string
): Promise<{ width: number; height: number }> {
  const object = await s3Client.send(
    new GetObjectCommand({ Bucket: bucket, Key: key })
  );

  const buffer = await streamToBuffer(object.Body as NodeJS.ReadableStream);
  const metadata = await Sharp(buffer).metadata();

  return {
    width: metadata.width ?? 0,
    height: metadata.height ?? 0,
  };
}
```

Install Sharp:
```bash
npm install sharp
npm install -D @types/sharp
```

Sharp is a native Node.js module that compiles to platform-specific binaries. When SST bundles your Lambda, it needs the correct binary for Linux x64 (the Lambda runtime). SST handles this automatically via esbuild's external bundling for native modules, but you need to tell it:

```typescript
// sst.config.ts — on the subscriber function
deliverablesBucket.subscribe(
  {
    handler: "src/functions/deliverables/process-upload.handler",
    link: [deliverablesBucket, table],
    timeout: "2 minutes",
    memory: "1024 MB",
    nodejs: {
      install: ["sharp"],  // Bundle sharp with its native binary
    },
  },
  // ...
);
```

The `install: ["sharp"]` tells SST to install Sharp with `npm install` after bundling, ensuring the correct Linux binary is used. Without this, you'll get "sharp binary not found" errors at runtime because SST bundled the macOS binary that was in your local `node_modules`.

### PDF processing

For PDFs, extract the page count. This is metadata displayed in the Runway UI so clients know what they're downloading before they do.

```typescript
// Install: npm install pdf-parse
// npm install -D @types/pdf-parse
import pdfParse from "pdf-parse";

async function getPdfPageCount(bucket: string, key: string): Promise<number> {
  const object = await s3Client.send(
    new GetObjectCommand({ Bucket: bucket, Key: key })
  );

  const buffer = await streamToBuffer(object.Body as NodeJS.ReadableStream);

  try {
    const data = await pdfParse(buffer, {
      // Only parse page count — don't extract text content
      max: 0,
    });
    return data.numpages;
  } catch {
    // Malformed or encrypted PDF — return 0 as a safe default
    return 0;
  }
}
```

For more advanced PDF processing (text extraction, thumbnail generation), consider AWS Textract for OCR workloads. That's beyond Runway's scope, but the pattern is the same: download the object, send it to Textract, store the results.

### Stream utilities

```typescript
// src/lib/s3-utils.ts
import { Readable } from "stream";

export async function streamToBuffer(stream: NodeJS.ReadableStream | Readable): Promise<Buffer> {
  return new Promise((resolve, reject) => {
    const chunks: Buffer[] = [];

    stream.on("data", (chunk: Buffer) => chunks.push(chunk));
    stream.on("end", () => resolve(Buffer.concat(chunks)));
    stream.on("error", reject);
  });
}

export async function getObjectStream(
  s3Client: import("@aws-sdk/client-s3").S3Client,
  bucket: string,
  key: string
): Promise<NodeJS.ReadableStream> {
  const { GetObjectCommand } = await import("@aws-sdk/client-s3");
  const response = await s3Client.send(
    new GetObjectCommand({ Bucket: bucket, Key: key })
  );

  if (!response.Body) {
    throw new Error(`Empty response body for s3://${bucket}/${key}`);
  }

  return response.Body as NodeJS.ReadableStream;
}
```

### Error handling for S3 event Lambdas

This is where S3 Lambdas differ from API Gateway Lambdas.

With API Gateway, if your Lambda throws, the request fails with a 500 and the user can retry. Clean.

With S3 event triggers, there's no user waiting for a response. S3 fires the notification and moves on. If your Lambda throws, S3 will retry it (up to 2 more times, with a delay), but there's no mechanism to surface that failure to the user.

The options:

**Option 1: Dead-letter queue.** Configure the Lambda with an SQS DLQ. Failed events after retries land in the queue. You monitor the queue and alert on non-empty DLQ. Chapter 8 covers SQS in detail — this is the right long-term approach.

**Option 2: Update the DynamoDB record.** When processing fails, set the deliverable's `status` to `"processing_failed"`. The Runway UI can check this and show a "processing failed" state. Users can see that something went wrong even though no exception bubbled up to them.

**Option 3: Catch everything and log.** The simplest approach, and what the code above does. Processing failure is logged but doesn't affect the core functionality — the file is still uploaded and downloadable, it just won't have a thumbnail or page count. For Runway's MVP, this is acceptable.

Production recommendation: combine options 2 and 1. Update the status to `"processing_failed"` on error (user-visible), and configure a DLQ (ops-visible). Both signals, no gaps.

```typescript
// In process-upload.ts handler — enhanced error handling
export const handler: S3Handler = async (event: S3Event) => {
  const logger = createLogger("s3-processor");

  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, " "));

    try {
      await processObject(bucket, key, record.s3.object.size, logger);
    } catch (err) {
      logger.error("Processing failed", { key, err: String(err) });

      const parsed = parseDeliverableKey(key);
      if (parsed) {
        // Mark the deliverable as failed in DynamoDB
        // The UI can surface this to the user
        try {
          await updateDeliverableMetadata(
            parsed.workspaceId,
            parsed.projectId,
            parsed.deliverableId,
            {
              status: "processing_failed",
              updatedAt: new Date().toISOString(),
            }
          );
        } catch (updateErr) {
          logger.error("Failed to update deliverable status", { key, updateErr: String(updateErr) });
        }
      }

      // Re-throw to trigger Lambda retry and (if configured) DLQ delivery
      throw err;
    }
  }
};
```

Rethrowing after handling makes S3 invoke the Lambda again (up to 3 attempts total). If all retries fail, the event goes to the DLQ. If you don't rethrow, S3 considers the event handled successfully even though processing failed — the DLQ never receives it.

---

## 6.5 Generating Invoice PDFs

The second file storage pattern: Lambda generates a PDF and writes it directly to S3 without a browser upload involved.

This flow triggers when a freelancer requests to generate their invoice. The Lambda:
1. Fetches the invoice data from DynamoDB
2. Renders it as a PDF
3. Writes the PDF to S3
4. Returns the download URL

```typescript
// sst.config.ts — add the invoice generation route
api.route("POST /workspaces/{workspaceId}/invoices/{invoiceId}/generate-pdf", {
  handler: "src/functions/invoices/generate-pdf.handler",
  link: [invoicesBucket, table],
  timeout: "30 seconds",  // PDF generation can take a moment
  memory: "512 MB",
});
```

### Generating PDFs with Puppeteer

The most reliable way to generate a PDF that looks exactly like your web UI is to render it with a headless browser. Puppeteer in Lambda is achievable with `@sparticuz/chromium` — a package that bundles a Lambda-compatible Chromium binary.

```bash
npm install puppeteer-core @sparticuz/chromium
```

```typescript
// src/functions/invoices/generate-pdf.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import { PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { Resource } from "sst";
import chromium from "@sparticuz/chromium";
import puppeteer from "puppeteer-core";
import { s3Client } from "../../lib/s3";
import { ok, notFound, forbidden, serverError } from "../../lib/response";
import { createLogger } from "../../lib/logger";
import { getInvoice } from "../../repositories/invoices";
import { getWorkspaceMembership } from "../../repositories/workspaces";
import { renderInvoiceHtml } from "../../lib/invoice-renderer";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const logger = createLogger(context.awsRequestId);

  const workspaceId = event.pathParameters?.workspaceId;
  const invoiceId = event.pathParameters?.invoiceId;

  if (!workspaceId || !invoiceId) {
    return notFound("Not found");
  }

  const userId = event.headers["x-user-id"];
  if (!userId) {
    return forbidden("Authentication required");
  }

  const membership = await getWorkspaceMembership(workspaceId, userId);
  if (!membership) {
    return forbidden("Access denied");
  }

  const invoice = await getInvoice(workspaceId, invoiceId);
  if (!invoice) {
    return notFound("Invoice not found");
  }

  logger.info("Generating invoice PDF", { workspaceId, invoiceId });

  const pdfBuffer = await generateInvoicePdf(invoice);

  const key = `workspaces/${workspaceId}/invoices/${invoiceId}/invoice-${invoice.invoiceNumber}.pdf`;

  // Store the PDF in S3
  await s3Client.send(
    new PutObjectCommand({
      Bucket: Resource.RunwayInvoices.name,
      Key: key,
      Body: pdfBuffer,
      ContentType: "application/pdf",
      Metadata: {
        "workspace-id": workspaceId,
        "invoice-id": invoiceId,
        "invoice-number": invoice.invoiceNumber,
      },
    })
  );

  // Generate a presigned URL valid for 72 hours
  // Invoice PDFs are often forwarded to accountants or clients by email —
  // they need to remain accessible long enough for that workflow
  const downloadUrl = await getSignedUrl(
    s3Client,
    new (await import("@aws-sdk/client-s3")).GetObjectCommand({
      Bucket: Resource.RunwayInvoices.name,
      Key: key,
      ResponseContentDisposition: `attachment; filename="invoice-${invoice.invoiceNumber}.pdf"`,
    }),
    { expiresIn: 72 * 60 * 60 }
  );

  logger.info("Invoice PDF generated and stored", {
    workspaceId,
    invoiceId,
    key,
  });

  return ok({
    downloadUrl,
    key,
    expiresIn: 72 * 60 * 60,
  });
};

async function generateInvoicePdf(invoice: Awaited<ReturnType<typeof getInvoice>>): Promise<Buffer> {
  if (!invoice) throw new Error("Invoice is null");

  // Render the invoice as HTML
  const html = renderInvoiceHtml(invoice);

  let browser: Awaited<ReturnType<typeof puppeteer.launch>> | null = null;

  try {
    browser = await puppeteer.launch({
      args: chromium.args,
      defaultViewport: chromium.defaultViewport,
      executablePath: await chromium.executablePath(),
      headless: chromium.headless,
    });

    const page = await browser.newPage();
    await page.setContent(html, { waitUntil: "networkidle0" });

    const pdfBuffer = await page.pdf({
      format: "A4",
      printBackground: true,
      margin: { top: "20mm", bottom: "20mm", left: "20mm", right: "20mm" },
    });

    return Buffer.from(pdfBuffer);
  } finally {
    if (browser) await browser.close();
  }
}
```

### Invoice HTML renderer

The HTML template for the invoice is regular TypeScript that returns a string. No framework needed here — it's just an HTML document.

```typescript
// src/lib/invoice-renderer.ts

interface Invoice {
  invoiceNumber: string;
  workspaceName: string;
  clientName: string;
  clientEmail: string;
  billingAddress?: string;
  lineItems: Array<{
    description: string;
    quantity: number;
    unitPrice: number;
    total: number;
  }>;
  subtotalCents: number;
  taxRatePct?: number;
  taxCents?: number;
  totalCents: number;
  currency: string;
  issuedAt: string;
  dueAt: string;
  notes?: string;
  status: string;
}

export function renderInvoiceHtml(invoice: Invoice): string {
  const currency = invoice.currency;
  const formatCents = (cents: number) =>
    new Intl.NumberFormat("en-GB", {
      style: "currency",
      currency,
      minimumFractionDigits: 2,
    }).format(cents / 100);

  const lineItemsHtml = invoice.lineItems
    .map(
      (item) => `
      <tr>
        <td class="description">${escapeHtml(item.description)}</td>
        <td class="number">${item.quantity}</td>
        <td class="number">${formatCents(item.unitPrice)}</td>
        <td class="number">${formatCents(item.total)}</td>
      </tr>`
    )
    .join("");

  return `
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      font-size: 14px;
      color: #111;
      background: white;
    }
    .invoice-header {
      display: flex;
      justify-content: space-between;
      margin-bottom: 40px;
      padding-bottom: 20px;
      border-bottom: 2px solid #e5e7eb;
    }
    .company-name { font-size: 24px; font-weight: 700; }
    .invoice-meta { text-align: right; }
    .invoice-number { font-size: 20px; font-weight: 600; color: #6366f1; }
    .parties { display: flex; justify-content: space-between; margin-bottom: 40px; }
    .party-label { font-size: 11px; font-weight: 600; text-transform: uppercase; color: #6b7280; margin-bottom: 8px; }
    table { width: 100%; border-collapse: collapse; margin: 30px 0; }
    th { text-align: left; padding: 10px 12px; background: #f9fafb; border-bottom: 2px solid #e5e7eb; font-size: 11px; text-transform: uppercase; color: #6b7280; }
    td { padding: 12px; border-bottom: 1px solid #e5e7eb; }
    .number { text-align: right; }
    .totals { margin-left: auto; width: 280px; }
    .totals-row { display: flex; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #e5e7eb; }
    .totals-row.grand-total { font-size: 18px; font-weight: 700; border-bottom: none; padding-top: 12px; }
    .notes { margin-top: 40px; padding: 20px; background: #f9fafb; border-radius: 4px; }
    .notes-label { font-weight: 600; margin-bottom: 8px; }
    .status-badge {
      display: inline-block;
      padding: 4px 10px;
      border-radius: 4px;
      font-size: 12px;
      font-weight: 600;
      text-transform: uppercase;
      background: ${invoice.status === "paid" ? "#d1fae5" : "#fee2e2"};
      color: ${invoice.status === "paid" ? "#065f46" : "#991b1b"};
    }
  </style>
</head>
<body>
  <div class="invoice-header">
    <div>
      <div class="company-name">${escapeHtml(invoice.workspaceName)}</div>
    </div>
    <div class="invoice-meta">
      <div class="invoice-number">Invoice #${escapeHtml(invoice.invoiceNumber)}</div>
      <div style="margin-top:8px">
        <span class="status-badge">${escapeHtml(invoice.status)}</span>
      </div>
      <div style="margin-top:8px; color:#6b7280">
        Issued: ${formatDate(invoice.issuedAt)}<br>
        Due: ${formatDate(invoice.dueAt)}
      </div>
    </div>
  </div>

  <div class="parties">
    <div>
      <div class="party-label">From</div>
      <div>${escapeHtml(invoice.workspaceName)}</div>
    </div>
    <div style="text-align:right">
      <div class="party-label">To</div>
      <div>${escapeHtml(invoice.clientName)}</div>
      <div style="color:#6b7280">${escapeHtml(invoice.clientEmail)}</div>
      ${invoice.billingAddress ? `<div style="color:#6b7280;white-space:pre-line">${escapeHtml(invoice.billingAddress)}</div>` : ""}
    </div>
  </div>

  <table>
    <thead>
      <tr>
        <th>Description</th>
        <th class="number">Qty</th>
        <th class="number">Unit Price</th>
        <th class="number">Total</th>
      </tr>
    </thead>
    <tbody>
      ${lineItemsHtml}
    </tbody>
  </table>

  <div class="totals">
    <div class="totals-row">
      <span>Subtotal</span>
      <span>${formatCents(invoice.subtotalCents)}</span>
    </div>
    ${invoice.taxRatePct ? `
    <div class="totals-row">
      <span>VAT (${invoice.taxRatePct}%)</span>
      <span>${formatCents(invoice.taxCents ?? 0)}</span>
    </div>` : ""}
    <div class="totals-row grand-total">
      <span>Total Due</span>
      <span>${formatCents(invoice.totalCents)}</span>
    </div>
  </div>

  ${invoice.notes ? `
  <div class="notes">
    <div class="notes-label">Notes</div>
    <div>${escapeHtml(invoice.notes)}</div>
  </div>` : ""}
</body>
</html>`;
}

function escapeHtml(str: string): string {
  return str
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#039;");
}

function formatDate(iso: string): string {
  return new Date(iso).toLocaleDateString("en-GB", {
    day: "2-digit",
    month: "long",
    year: "numeric",
  });
}
```

The `escapeHtml` function is not optional. Invoice data comes from user input — client names, project descriptions, notes. Without HTML escaping, a client name like `<script>alert(1)</script>` would execute JavaScript in the generated PDF. Always escape untrusted data before inserting it into HTML.

### Puppeteer Lambda considerations

Puppeteer + Chromium is resource-heavy. A few things to know:

**Bundle size:** `@sparticuz/chromium` is ~50MB compressed. Your Lambda package will be larger. This is fine within Lambda's 250MB unzipped limit, but worth knowing.

**Memory:** Set the Lambda to at least 512MB. Chromium needs it. 1GB for complex invoice layouts.

**Concurrency:** Each invocation starts a new Chromium process. PDF generation takes 2-5 seconds. Under load, you could have 100 concurrent Chromium instances. That's fine from a Lambda perspective (Lambda scales horizontally), but monitor memory usage.

**Cold start:** Chromium adds ~3 seconds to cold start. For a PDF generation endpoint that users click deliberately, this is acceptable. It's not acceptable for sub-100ms API responses.

An alternative for simpler invoices: `pdf-lib` — a pure JavaScript PDF library with no native dependencies. Much faster, smaller bundle, but you write PDF primitives instead of HTML. If your invoice is simple (tabular data, no custom fonts, basic layout), it's worth considering.

```bash
npm install pdf-lib
```

The `pdf-lib` approach requires more code to position elements but produces a smaller, faster Lambda. The Puppeteer approach lets you prototype in HTML and use CSS for layout, which is faster to iterate on. Choose based on your constraints.

---

## 6.6 Access Patterns: Private Assets with Signed URLs

Runway's entire file storage model is private-by-default. No object in either bucket is publicly accessible.

### Blocking all public access

This should be your default. Both buckets have public access blocked:

```typescript
// In SST, buckets are private by default
// The sst.aws.Bucket component sets BlockPublicAcls,
// IgnorePublicAcls, BlockPublicPolicy, and RestrictPublicBuckets automatically
// when no public configuration is specified

const deliverablesBucket = new sst.aws.Bucket("RunwayDeliverables", {
  // No public:true — private by default
});
```

To verify in the AWS Console: S3 → your bucket → Permissions → Block public access. All four settings should be enabled.

### The workspace isolation guarantee

The access pattern for deliverables has an important property: the workspace ID is in the key path, and Lambda checks workspace membership before generating any URL.

```
Browser → GET /workspaces/{workspaceId}/projects/{projectId}/deliverables/{id}/download
            ↓
         Lambda: "Does this user belong to workspaceId?"
            ↓ Yes
         DynamoDB: "Get the deliverable record (which contains the S3 key)"
            ↓
         Lambda: "Does the key start with workspaces/{workspaceId}/?"
            ↓ Yes — belt-and-suspenders check
         S3: Generate signed URL for the key
            ↓
         Return signed URL to browser
            ↓
Browser → S3: GET {signedUrl}  (goes directly to S3, Lambda not involved)
```

A freelancer from workspace A making a request for a deliverable in workspace B would fail at the membership check. Even if they somehow bypassed that (they can't, but hypothetically), the belt-and-suspenders key prefix check would catch it.

The signed URL itself is time-limited and cryptographically bound to the exact key. There's no mechanism for the browser to modify the URL to point at a different key — the signature would break.

### How presigned URLs work under the hood

When you call `getSignedUrl`, the SDK:

1. Takes the S3 command (bucket, key, any response headers)
2. Signs it using AWS Signature Version 4 with your Lambda's IAM credentials
3. Returns a URL with `X-Amz-Algorithm`, `X-Amz-Credential`, `X-Amz-Date`, `X-Amz-Expires`, `X-Amz-SignedHeaders`, and `X-Amz-Signature` query parameters

When the browser makes the request, S3:
1. Verifies the signature against the request parameters
2. Checks the expiry timestamp
3. Verifies the signing credentials had permission to perform this action on this key

If any parameter has been modified — different key, different response header, expired timestamp — S3 returns 403 Forbidden.

The Lambda's IAM role must have `s3:GetObject` permission on the bucket (which SST grants via `link: [deliverablesBucket]`). The signed URL inherits those permissions, bound to the specific object, for the specified time window.

### Public assets via CloudFront

Runway's deliverables are private — they belong to specific workspaces and should not be accessible to the public internet.

But the pattern for public assets (profile pictures, publicly shared project portfolios, published invoices) is different: instead of signed URLs, you put CloudFront in front of S3 and let CloudFront cache and serve the objects.

```typescript
// sst.config.ts — for a hypothetical public portfolio bucket
const portfolioBucket = new sst.aws.Bucket("RunwayPortfolio");

// CloudFront distribution in front of the bucket
const cdn = new sst.aws.Router("RunwayCdn", {
  routes: {
    "/*": portfolioBucket,  // Route all traffic to the bucket
  },
  domain: {
    name: "assets.runway.so",
    dns: sst.aws.dns(),
  },
});
```

With this pattern, `https://assets.runway.so/portfolios/my-portfolio/work.jpg` is publicly accessible without any signed URL. CloudFront caches the response at edge locations globally.

**Don't use CloudFront for Runway's deliverables.** The security model breaks: CloudFront caches the object at the edge and serves it to anyone who knows the URL. A shared link that "expires" actually doesn't expire from CloudFront's cache. For private assets, presigned URLs directly to S3 are correct.

CloudFront is the right choice when:
- The content is public by design (published portfolios, logos, public assets)
- You need low-latency global delivery (CloudFront has 400+ edge locations)
- You want to serve content via a custom domain with SSL

Presigned URLs are the right choice when:
- Content is private to specific users or workspaces
- You need the access to expire
- You don't want a CDN caching and distributing your content

### Minimum required IAM permissions

SST handles IAM automatically via `link`, but understanding what permissions each operation requires prevents debugging headaches.

| Operation | Permission |
|-----------|-----------|
| Generate presigned PUT URL | `s3:PutObject` |
| Generate presigned GET URL | `s3:GetObject` |
| HeadObject (check existence) | `s3:GetObject` |
| ListObjects | `s3:ListBucket` |
| Delete an object | `s3:DeleteObject` |

Lambda functions that generate presigned PUT URLs need `s3:PutObject`. Lambda functions that generate presigned GET URLs need `s3:GetObject`. The Lambda executing the presigned URL is not the same Lambda that originally signed it — by the time the browser uploads the file, the signing Lambda's execution has ended. S3 validates the signature independently.

SST's `link: [bucket]` grants `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, and `s3:ListBucket` on the linked bucket. This is the right default — each Lambda function linked to the bucket gets the full set of permissions, which is safe because the bucket itself is private and only reachable via signed URLs or your own Lambda code.

For more restrictive deployments, you can customise the IAM policy. SST exposes this via the `transform` option on the function, but for Runway, the defaults are appropriate.

---

## 6.7 Lifecycle Policies and Cost Control

S3 is cheap. But S3 is not free, and without lifecycle policies, your storage grows unboundedly. For Runway, there are three cleanup concerns:

1. **Incomplete uploads:** Presigned PUT URLs that were generated but never used. S3 has no knowledge of whether a PUT was completed — the object either exists or it doesn't. But multipart uploads (for large files) leave partial data in S3 until either completed or aborted.

2. **Temporary files:** Processing artifacts that are only needed briefly (e.g., an intermediate resizing step).

3. **Old deleted deliverables:** When a freelancer deletes a deliverable from Runway, the DynamoDB record is deleted but the S3 object remains. Without cleanup, deleted files accumulate.

### Lifecycle rules in SST

```typescript
const deliverablesBucket = new sst.aws.Bucket("RunwayDeliverables", {
  cors: [/* ... */],
  transform: {
    bucket: (args) => {
      // Lifecycle configuration applied directly to the bucket
      args.lifecycleRules = [
        {
          // Rule 1: Abort incomplete multipart uploads after 24 hours
          id: "abort-incomplete-multipart",
          status: "Enabled",
          abortIncompleteMultipartUpload: {
            daysAfterInitiation: 1,
          },
        },
        {
          // Rule 2: Move old deliverables to Infrequent Access after 90 days
          // Infrequent Access is ~40% cheaper than Standard
          // But has a 30-day minimum and retrieval cost — worth it for old files
          id: "ia-transition-deliverables",
          status: "Enabled",
          prefix: "workspaces/",
          transitions: [
            {
              days: 90,
              storageClass: "STANDARD_IA",
            },
            {
              days: 365,
              storageClass: "GLACIER_INSTANT_RETRIEVAL",
            },
          ],
        },
        {
          // Rule 3: Clean up thumbnails if the deliverable is deleted
          // (Handled by application logic — see below)
          // Thumbnails older than 30 days with no parent object can be deleted
          id: "cleanup-orphaned-thumbnails",
          status: "Enabled",
          prefix: "workspaces/",
          filter: {
            // Only apply to thumbnail keys
            prefix: "", // We'll rely on application logic instead
          },
        },
      ];
    },
  },
});
```

The `transform` API gives you access to the underlying Pulumi resource arguments. Use it for anything SST doesn't expose directly.

### Storage classes explained

| Class | Use Case | Cost | Retrieval |
|-------|----------|------|-----------|
| `STANDARD` | Active files, frequently accessed | ~$0.023/GB/month | Immediate, free |
| `STANDARD_IA` | Files accessed < once/month | ~$0.0125/GB/month | Immediate, per-GB charge |
| `GLACIER_INSTANT_RETRIEVAL` | Archives, accessed < once/quarter | ~$0.004/GB/month | Immediate, higher per-GB charge |
| `GLACIER_FLEXIBLE_RETRIEVAL` | Long-term archive, rare access | ~$0.0036/GB/month | 1-5 hours |
| `DEEP_ARCHIVE` | Compliance, almost never accessed | ~$0.00099/GB/month | 12+ hours |

For Runway's deliverables:
- New uploads → Standard (active projects need instant access)
- 90 days old → Standard-IA (old project files are rarely re-downloaded)
- 1 year old → Glacier Instant Retrieval (archived project files, accessible on demand but cheap to store)

The transition thresholds depend on your users' access patterns. If freelancers routinely re-download project files months after project completion, keep them in Standard longer.

### Handling deleted deliverables

When a freelancer deletes a deliverable from Runway, you need to delete the S3 object (and its thumbnails). Don't just delete the DynamoDB record and leave the S3 object:

```typescript
// src/functions/deliverables/delete.ts
import type { APIGatewayProxyHandlerV2 } from "aws-lambda";
import {
  DeleteObjectCommand,
  ListObjectsV2Command,
  DeleteObjectsCommand,
} from "@aws-sdk/client-s3";
import { Resource } from "sst";
import { s3Client } from "../../lib/s3";
import { ok, notFound, forbidden } from "../../lib/response";
import { createLogger } from "../../lib/logger";
import { getDeliverable, deleteDeliverable } from "../../repositories/deliverables";
import { getWorkspaceMembership } from "../../repositories/workspaces";

export const handler: APIGatewayProxyHandlerV2 = async (event, context) => {
  const logger = createLogger(context.awsRequestId);

  const workspaceId = event.pathParameters?.workspaceId;
  const projectId = event.pathParameters?.projectId;
  const deliverableId = event.pathParameters?.deliverableId;

  if (!workspaceId || !projectId || !deliverableId) {
    return notFound("Not found");
  }

  const userId = event.headers["x-user-id"];
  if (!userId) {
    return forbidden("Authentication required");
  }

  const membership = await getWorkspaceMembership(workspaceId, userId);
  if (!membership) {
    return forbidden("Access denied");
  }

  const deliverable = await getDeliverable(workspaceId, projectId, deliverableId);
  if (!deliverable) {
    return notFound("Deliverable not found");
  }

  // Delete the DynamoDB record first
  // If S3 deletion fails, the record is gone and the file is effectively orphaned —
  // the lifecycle policy will clean it up eventually
  await deleteDeliverable(workspaceId, projectId, deliverableId);

  // Delete all objects with this deliverable's prefix (original + thumbnails + metadata)
  const prefix = `workspaces/${workspaceId}/projects/${projectId}/deliverables/${deliverableId}/`;

  // List all objects under the prefix
  const listResult = await s3Client.send(
    new ListObjectsV2Command({
      Bucket: Resource.RunwayDeliverables.name,
      Prefix: prefix,
    })
  );

  if (listResult.Contents && listResult.Contents.length > 0) {
    // Delete all objects in a single batch request (up to 1000 per request)
    await s3Client.send(
      new DeleteObjectsCommand({
        Bucket: Resource.RunwayDeliverables.name,
        Delete: {
          Objects: listResult.Contents.map((obj) => ({ Key: obj.Key! })),
          Quiet: true, // Don't return individual deletion results
        },
      })
    );

    logger.info("Deleted deliverable and S3 objects", {
      workspaceId,
      deliverableId,
      deletedObjectCount: listResult.Contents.length,
    });
  }

  return ok({ deleted: true });
};
```

`DeleteObjectsCommand` can delete up to 1,000 objects in a single request. For deliverables with multiple thumbnails and a metadata JSON, this is more than sufficient. For workspaces with thousands of files (project deletions that cascade to many deliverables), paginate the list results.

### Versioning: when to enable it

Versioning keeps every version of every object when enabled. Overwrites don't delete the previous version — they create a new one. Deletes add a "delete marker" but preserve all prior versions.

Enable versioning when:
- **Compliance requires it** — you need to prove what a file contained at a specific point in time
- **Audit trail matters** — invoice revisions should be reversible
- **User mistakes are costly** — accidental deletions can be recovered

Don't enable versioning when:
- Your objects are generated (thumbnails, processed files) — you can always regenerate them
- Storage cost is a concern — versioning can dramatically increase storage costs for frequently updated objects
- Your objects are immutable by design (each upload is a new object with a new key)

For Runway, versioning is off on both buckets. Deliverables are immutable — each upload gets a unique `fileId`, so there's no overwriting. Invoice PDFs are regenerated on demand — if the invoice changes, a new PDF is generated with the same key, overwriting the old one. Versioning would bloat storage without benefit.

If you add a "version history" feature to Runway (view past versions of an invoice), that's the time to enable versioning on the invoices bucket.

### Monitoring storage costs

S3 doesn't have built-in cost alerting per bucket. Set up AWS Cost Explorer with resource-level tagging:

```typescript
const deliverablesBucket = new sst.aws.Bucket("RunwayDeliverables", {
  transform: {
    bucket: (args) => {
      args.tags = {
        Project: "runway",
        Component: "deliverables",
        Stage: $app.stage,
      };
    },
  },
});
```

In AWS Cost Explorer, filter by tag to see monthly S3 costs per bucket. Set a budget alert at a threshold that would indicate unexpected growth (a sign of a leak: a loop uploading objects repeatedly, undeleted temp files, etc.).

---

## 6.8 Putting It All Together

Here's the complete updated `sst.config.ts` for Chapter 6, with both buckets, their event triggers, and the API routes wired together:

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
    // ─── DynamoDB ──────────────────────────────────────────────────────────
    const table = new sst.aws.Dynamo("RunwayTable", {
      fields: {
        pk: "string",
        sk: "string",
        gsi1pk: "string",
        gsi1sk: "string",
      },
      primaryIndex: { hashKey: "pk", rangeKey: "sk" },
      globalIndexes: {
        gsi1: { hashKey: "gsi1pk", rangeKey: "gsi1sk" },
      },
    });

    // ─── VPC (for Aurora — from Chapter 5) ─────────────────────────────────
    const vpc = new sst.aws.Vpc("RunwayVpc", { az: 2 });

    // ─── Aurora (from Chapter 5) ────────────────────────────────────────────
    const db = new sst.aws.Aurora("RunwayDb", {
      engine: "postgres",
      vpc,
      scaling: { min: "0 ACU", max: "4 ACU" },
      proxy: $app.stage === "production",
    });

    // ─── S3 Buckets ─────────────────────────────────────────────────────────
    const deliverablesBucket = new sst.aws.Bucket("RunwayDeliverables", {
      cors: [
        {
          allowedHeaders: ["*"],
          allowedMethods: ["GET", "PUT", "POST"],
          allowedOrigins: [
            $app.stage === "production"
              ? "https://app.runway.so"
              : "http://localhost:3000",
          ],
          exposedHeaders: ["ETag"],
          maxAge: 3000,
        },
      ],
      transform: {
        bucket: (args) => {
          args.forceDestroy = $app.stage !== "production";
          args.lifecycleRules = [
            {
              id: "abort-incomplete-multipart",
              status: "Enabled",
              abortIncompleteMultipartUpload: { daysAfterInitiation: 1 },
            },
            {
              id: "ia-transition",
              status: "Enabled",
              transitions: [
                { days: 90, storageClass: "STANDARD_IA" },
                { days: 365, storageClass: "GLACIER_INSTANT_RETRIEVAL" },
              ],
            },
          ];
          args.tags = {
            Project: "runway",
            Component: "deliverables",
            Stage: $app.stage,
          };
        },
      },
    });

    const invoicesBucket = new sst.aws.Bucket("RunwayInvoices", {
      transform: {
        bucket: (args) => {
          args.forceDestroy = $app.stage !== "production";
          args.lifecycleRules = [
            {
              id: "abort-incomplete-multipart",
              status: "Enabled",
              abortIncompleteMultipartUpload: { daysAfterInitiation: 1 },
            },
            {
              // Invoice PDFs don't need Glacier — they're small and occasionally re-downloaded
              id: "ia-transition",
              status: "Enabled",
              transitions: [
                { days: 180, storageClass: "STANDARD_IA" },
              ],
            },
          ];
          args.tags = {
            Project: "runway",
            Component: "invoices",
            Stage: $app.stage,
          };
        },
      },
    });

    // ─── S3 Event Triggers ───────────────────────────────────────────────────
    deliverablesBucket.subscribe(
      {
        handler: "src/functions/deliverables/process-upload.handler",
        link: [deliverablesBucket, table],
        timeout: "2 minutes",
        memory: "1024 MB",
        nodejs: { install: ["sharp"] },
      },
      {
        events: ["s3:ObjectCreated:*"],
        filterPrefix: "workspaces/",
      }
    );

    // ─── API Gateway ─────────────────────────────────────────────────────────
    const api = new sst.aws.ApiGatewayV2("RunwayApi");

    // Existing routes (Chapters 1-5)
    // ... workspaces, clients, projects, invoices, reports ...

    // ─── Deliverable Routes ──────────────────────────────────────────────────
    api.route(
      "POST /workspaces/{workspaceId}/projects/{projectId}/deliverables/upload-url",
      {
        handler: "src/functions/deliverables/create-upload-url.handler",
        link: [deliverablesBucket, table],
      }
    );

    api.route(
      "POST /workspaces/{workspaceId}/projects/{projectId}/deliverables/{deliverableId}/confirm",
      {
        handler: "src/functions/deliverables/confirm-upload.handler",
        link: [deliverablesBucket, table],
      }
    );

    api.route(
      "GET /workspaces/{workspaceId}/projects/{projectId}/deliverables",
      {
        handler: "src/functions/deliverables/list.handler",
        link: [table],
      }
    );

    api.route(
      "GET /workspaces/{workspaceId}/projects/{projectId}/deliverables/{deliverableId}/download",
      {
        handler: "src/functions/deliverables/get-download-url.handler",
        link: [deliverablesBucket, table],
      }
    );

    api.route(
      "DELETE /workspaces/{workspaceId}/projects/{projectId}/deliverables/{deliverableId}",
      {
        handler: "src/functions/deliverables/delete.handler",
        link: [deliverablesBucket, table],
      }
    );

    // ─── Invoice Routes ──────────────────────────────────────────────────────
    api.route(
      "POST /workspaces/{workspaceId}/invoices/{invoiceId}/generate-pdf",
      {
        handler: "src/functions/invoices/generate-pdf.handler",
        link: [invoicesBucket, table],
        timeout: "30 seconds",
        memory: "512 MB",
      }
    );

    api.route(
      "GET /workspaces/{workspaceId}/invoices/{invoiceId}/download",
      {
        handler: "src/functions/invoices/get-download-url.handler",
        link: [invoicesBucket, table],
      }
    );

    return {
      api: api.url,
      deliverablesBucket: deliverablesBucket.name,
      invoicesBucket: invoicesBucket.name,
    };
  },
});
```

### The complete directory structure

By the end of Chapter 6, Runway's `src/` directory looks like:

```
src/
├── functions/
│   ├── health.ts                          # Chapter 1
│   ├── workspaces/                        # Chapters 2-4
│   │   ├── create.ts
│   │   ├── get.ts
│   │   └── list.ts
│   ├── clients/                           # Chapter 4
│   ├── projects/                          # Chapter 4
│   ├── invoices/                          # Chapter 4
│   │   ├── create.ts
│   │   ├── update-status.ts
│   │   ├── generate-pdf.ts                ← Chapter 6
│   │   └── get-download-url.ts            ← Chapter 6
│   ├── reports/                           # Chapter 5
│   │   ├── revenue.ts
│   │   └── clients.ts
│   └── deliverables/                      ← Chapter 6
│       ├── create-upload-url.ts
│       ├── confirm-upload.ts
│       ├── list.ts
│       ├── get-download-url.ts
│       ├── delete.ts
│       └── process-upload.ts
├── repositories/
│   ├── workspaces.ts
│   ├── clients.ts
│   ├── projects.ts
│   ├── invoices.ts
│   ├── payments.ts                        # Chapter 5
│   └── deliverables.ts                    ← Chapter 6
└── lib/
    ├── dynamo.ts
    ├── prisma.ts                          # Chapter 5
    ├── s3.ts                              ← Chapter 6
    ├── s3-utils.ts                        ← Chapter 6
    ├── invoice-renderer.ts               ← Chapter 6
    ├── response.ts
    └── logger.ts
```

---

## 6.9 Testing the File Storage Layer

Testing file upload flows is harder than testing CRUD endpoints because S3 is involved. The right approach depends on what you're testing.

### Unit testing the URL generation logic

The URL generation logic is the part worth unit testing. Mock the S3 client and verify that the correct bucket, key pattern, content type restrictions, and expiry are used:

```typescript
// src/functions/deliverables/create-upload-url.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { handler } from "./create-upload-url";

// Mock the AWS SDK
vi.mock("@aws-sdk/client-s3", () => ({
  S3Client: vi.fn().mockImplementation(() => ({})),
  PutObjectCommand: vi.fn(),
}));

vi.mock("@aws-sdk/s3-request-presigner", () => ({
  getSignedUrl: vi.fn().mockResolvedValue("https://s3.amazonaws.com/presigned-url"),
}));

// Mock SST Resource (injected env vars in production)
vi.mock("sst", () => ({
  Resource: {
    RunwayDeliverables: { name: "test-bucket" },
  },
}));

vi.mock("../../repositories/workspaces", () => ({
  getWorkspaceMembership: vi.fn().mockResolvedValue({ role: "owner" }),
}));

function makeEvent(overrides: Record<string, unknown> = {}) {
  return {
    pathParameters: { workspaceId: "ws-test", projectId: "p-test" },
    headers: { "x-user-id": "user-test" },
    body: JSON.stringify({
      filename: "design.pdf",
      contentType: "application/pdf",
      sizeBytes: 1024 * 1024,
    }),
    ...overrides,
  };
}

describe("create-upload-url", () => {
  it("returns an upload URL for allowed content types", async () => {
    const event = makeEvent();
    const result = await handler(event as any, {} as any);

    expect(result.statusCode).toBe(200);
    const body = JSON.parse(result.body as string);
    expect(body.uploadUrl).toBe("https://s3.amazonaws.com/presigned-url");
    expect(body.fileId).toBeDefined();
    expect(body.key).toMatch(/^workspaces\/ws-test\/projects\/p-test\/deliverables\//);
  });

  it("rejects disallowed content types", async () => {
    const event = makeEvent({
      body: JSON.stringify({
        filename: "exploit.exe",
        contentType: "application/x-msdownload",
        sizeBytes: 1024,
      }),
    });

    const result = await handler(event as any, {} as any);
    expect(result.statusCode).toBe(400);
    const body = JSON.parse(result.body as string);
    expect(body.message).toContain("not allowed");
  });

  it("rejects files above the size limit", async () => {
    const event = makeEvent({
      body: JSON.stringify({
        filename: "huge.pdf",
        contentType: "application/pdf",
        sizeBytes: 600 * 1024 * 1024, // 600MB — over the 500MB limit
      }),
    });

    const result = await handler(event as any, {} as any);
    expect(result.statusCode).toBe(400);
  });

  it("rejects requests from users not in the workspace", async () => {
    const { getWorkspaceMembership } = await import("../../repositories/workspaces");
    vi.mocked(getWorkspaceMembership).mockResolvedValueOnce(null);

    const event = makeEvent();
    const result = await handler(event as any, {} as any);

    expect(result.statusCode).toBe(403);
  });

  it("sanitises filenames with unsafe characters", async () => {
    const event = makeEvent({
      body: JSON.stringify({
        filename: "my file (v2) [final].pdf",
        contentType: "application/pdf",
        sizeBytes: 1024,
      }),
    });

    const result = await handler(event as any, {} as any);
    expect(result.statusCode).toBe(200);
    const body = JSON.parse(result.body as string);
    // Spaces and brackets should be replaced with hyphens
    expect(body.key).not.toContain(" ");
    expect(body.key).not.toContain("(");
    expect(body.key).not.toContain("[");
  });
});
```

### Integration testing with real S3

For a fuller test that exercises the actual upload flow, use `sst dev` with a test stage and real AWS resources:

```typescript
// tests/integration/deliverables.test.ts
// Run with: NODE_ENV=test npx vitest run --stage test

import { describe, it, expect } from "vitest";
import { Resource } from "sst";

// With sst dev running against the "test" stage, Resource is populated
// with real bucket names and table names

describe("deliverable upload flow (integration)", () => {
  it("completes full upload flow", async () => {
    const baseUrl = process.env.API_URL!;
    const workspaceId = "test-ws";
    const projectId = "test-p";

    // Step 1: Get upload URL
    const uploadUrlRes = await fetch(
      `${baseUrl}/workspaces/${workspaceId}/projects/${projectId}/deliverables/upload-url`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "x-user-id": "test-user",
        },
        body: JSON.stringify({
          filename: "test.pdf",
          contentType: "application/pdf",
          sizeBytes: 1024,
        }),
      }
    );

    expect(uploadUrlRes.status).toBe(200);
    const { uploadUrl, key, fileId } = await uploadUrlRes.json();

    // Step 2: Upload a fake PDF directly to S3
    const fakePdf = new Uint8Array(1024).fill(0x25); // 0x25 = '%' char in PDF header
    const uploadRes = await fetch(uploadUrl, {
      method: "PUT",
      headers: { "Content-Type": "application/pdf" },
      body: fakePdf,
    });

    expect(uploadRes.status).toBe(200);

    // Step 3: Confirm the upload
    const confirmRes = await fetch(
      `${baseUrl}/workspaces/${workspaceId}/projects/${projectId}/deliverables/${fileId}/confirm`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "x-user-id": "test-user",
        },
        body: JSON.stringify({
          key,
          filename: "test.pdf",
          contentType: "application/pdf",
          sizeBytes: 1024,
        }),
      }
    );

    expect(confirmRes.status).toBe(200);
    const deliverable = await confirmRes.json();
    expect(deliverable.deliverableId).toBe(fileId);
    expect(deliverable.status).toBe("active");

    // Step 4: Get a download URL
    const downloadUrlRes = await fetch(
      `${baseUrl}/workspaces/${workspaceId}/projects/${projectId}/deliverables/${fileId}/download`,
      {
        headers: { "x-user-id": "test-user" },
      }
    );

    expect(downloadUrlRes.status).toBe(200);
    const { downloadUrl } = await downloadUrlRes.json();
    expect(downloadUrl).toContain("s3.amazonaws.com");

    // Step 5: Download the file using the signed URL
    const fileRes = await fetch(downloadUrl);
    expect(fileRes.status).toBe(200);
    expect(fileRes.headers.get("Content-Type")).toBe("application/pdf");
  });
});
```

Integration tests against real AWS are slower but catch the things that matter: CORS configuration, bucket permissions, key format assumptions. Run them in CI against a dedicated `test` stage, not against staging or production.

---

## 6.10 The Security Summary

File storage has more security surface area than CRUD APIs. Here's the complete threat model for Runway's file storage layer and how each threat is addressed.

### Threat 1: Cross-workspace file access

**Attack:** A user in workspace A guesses the S3 key of a file in workspace B and requests a download URL for it.

**Defence:** Download URLs are generated by Lambda. Lambda requires a valid workspace membership check against DynamoDB. The key returned to the client comes from DynamoDB, not from the request. A user cannot request a signed URL for a key they specify.

### Threat 2: Upload URL abuse

**Attack:** A user generates an upload URL with `contentType: "application/pdf"` but uploads an `.exe` file.

**Defence:** The presigned URL is signed with `ContentType` in the signature. If the actual PUT request uses a different Content-Type, S3 returns 403. Additionally, the file processor Lambda checks the actual content type from S3 metadata on confirmation.

**Attack:** A user generates an upload URL and uploads a file larger than declared.

**Defence:** `ContentLength` is included in the presigned URL signature. PUT requests with a different body length are rejected by S3. (Note: this is a partial defence — clients control their PUT headers. A belt-and-suspenders check in the confirm Lambda that reads actual size from S3 HeadObject is the complete defence.)

### Threat 3: Direct bucket access

**Attack:** An attacker knows the bucket name and tries to access `https://{bucket}.s3.amazonaws.com/{key}`.

**Defence:** All four Block Public Access settings are enabled. No public bucket policy exists. S3 returns 403 for all unsigned requests. Signed requests are only valid for 15 minutes (upload) or 1 hour (download), are bound to a specific key, and require the Lambda's IAM credentials to sign.

### Threat 4: Presigned URL reuse

**Attack:** An attacker intercepts a download URL (from logs, from a shared screen, from browser history) and uses it after the user revokes access.

**Defence:** URLs expire (15 minutes for uploads, 1 hour for downloads, 72 hours for invoices). For immediate revocation (e.g., an employee is fired and should lose access), the correct approach is to use short-lived URLs and rely on the next request to the Lambda generating a new URL that fails the access check. If you need immediate revocation for downloaded URLs, you need a URL signing key rotation or a CloudFront distribution with field-level encryption — complex, and not required for Runway's threat model.

### Threat 5: Processing Lambda injection

**Attack:** A malicious file is uploaded that exploits a vulnerability in Sharp, `pdf-parse`, or Puppeteer.

**Defence:** The processing Lambda runs with minimum necessary permissions — only access to the deliverables bucket and the DynamoDB table. It has no network access beyond AWS services. Even if a parsing exploit executes code inside the Lambda, the blast radius is limited: it can read and write to the deliverables bucket and the DynamoDB table. It cannot reach the internet, other AWS accounts, or other services.

For higher-security deployments, run the processing Lambda in a VPC with no NAT gateway (no outbound internet), and use SQS as a buffer between the S3 event and the processing Lambda, allowing for rate limiting and controlled re-processing.

---

## Where We Are

Runway now has a complete file storage layer:

```
[S3: RunwayDeliverables]                 [S3: RunwayInvoices]
  ├── workspaces/{wsId}/projects/          ├── workspaces/{wsId}/invoices/
  │   └── {pId}/deliverables/              │   └── {invoiceId}/invoice-*.pdf
  │       └── {fileId}/
  │           ├── {original-file}
  │           ├── thumbnails/
  │           │   └── thumb-md.jpg
  │           └── metadata.json

[API Gateway + Lambda]
  ├── POST .../deliverables/upload-url    → generate presigned PUT URL
  ├── POST .../deliverables/{id}/confirm  → create DynamoDB record
  ├── GET  .../deliverables/{id}/download → generate presigned GET URL
  ├── DELETE .../deliverables/{id}        → delete from S3 + DynamoDB
  ├── POST .../invoices/{id}/generate-pdf → render PDF, store to S3
  └── GET  .../invoices/{id}/download     → generate presigned GET URL

[S3 Event] → [Lambda: process-upload]
  ├── Images: generate thumbnail, extract dimensions
  └── PDFs: extract page count
```

The architecture map for Runway so far:

```
                        ┌─────────────────────────────────┐
                        │         API Gateway              │
                        └──────────────┬──────────────────┘
                                       │
                 ┌─────────────────────┼──────────────────────┐
                 │                     │                       │
          ┌──────▼──────┐     ┌────────▼────────┐     ┌──────▼──────┐
          │  DynamoDB   │     │  Aurora (VPC)   │     │     S3      │
          │  (live app) │     │  (reporting)    │     │   (files)   │
          └─────────────┘     └─────────────────┘     └──────┬──────┘
                                                             │
                                                      ┌──────▼──────┐
                                                      │  Processing │
                                                      │   Lambda    │
                                                      │ (thumbnails,│
                                                      │  PDF meta)  │
                                                      └─────────────┘
```

The storage foundation is solid. The gap that remains — and you'll notice it throughout this chapter's code — is that authentication is placeholder: `event.headers["x-user-id"]` is a stand-in for a real JWT check. Chapter 7 replaces that with Cognito, issues real JWTs, and gates every route — including every file operation — to the authenticated user's workspace.

---

> **The code for this chapter** is available at `06-s3-file-uploads/` in the companion repository. Run `npm install && npx sst dev` to start it. To test the upload flow end-to-end, use the test script at `scripts/test-upload.ts`.
