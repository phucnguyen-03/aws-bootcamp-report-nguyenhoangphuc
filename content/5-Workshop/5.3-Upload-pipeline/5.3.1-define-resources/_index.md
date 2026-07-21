---
title: "Define Lambda Functions, Storage & Data Schema"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.3.1. </b> "
---

## Define Lambda Functions, Storage & Data Schema

Before configuring the pipeline inside `backend.ts`, we need to declare individual resources following Amplify Gen 2's structure. Each resource (Lambda function, Storage, Data model) is defined in its own subdirectory under `amplify/` and then imported into the central `backend.ts` file to link them together.

In this section, we will create:
- **Lambda A** (`lambda-a-trigger`): A function triggered by S3 upload events to parse and extract text.
- **Lambda B** (`lambda-b-textract-result`): A function triggered by Textract completions (SNS) and DynamoDB Stream events to analyze text.
- **Storage** (`documentAssistantStorage`): An S3 bucket with user-isolated access paths.
- **Data Schema**: `Document` and `UserQuota` models in DynamoDB.

---

### Create Lambda A — Upload Trigger

**Step 1.** Create directory `amplify/functions/lambda-a-trigger/` and add the file `resource.ts`:

```typescript
import { defineFunction, secret } from '@aws-amplify/backend';

export const lambdaATrigger = defineFunction({
  name: 'lambda-a-trigger',
  entry: './handler.ts',
  timeoutSeconds: 60,
  environment: {
    OPENROUTER_API_KEY: secret('OPENROUTER_API_KEY'),
  },
});
```

{{% notice note %}}
- `timeoutSeconds: 60` — Lambda A needs enough time to download files from S3 and make requests to the Textract API.
- `secret('OPENROUTER_API_KEY')` — The API key is securely stored in the SSM Parameter Store and injected into environment variables during deployment.
{{% /notice %}}

---

### Create Lambda B — Textract Result & AI Analysis

**Step 2.** Create directory `amplify/functions/lambda-b-textract-result/` and add the file `resource.ts`:

```typescript
import { defineFunction, secret } from '@aws-amplify/backend';

export const lambdaBTextractResult = defineFunction({
  name: 'lambda-b-textract-result',
  entry: './handler.ts',
  timeoutSeconds: 300,
  environment: {
    OPENROUTER_API_KEY: secret('OPENROUTER_API_KEY'),
  },
});
```

{{% notice note %}}
`timeoutSeconds: 300` (5 minutes) — Lambda B needs a longer timeout as invoking generative AI models (OpenRouter or Bedrock) can take up to several dozen seconds.
{{% /notice %}}

---

### Create Storage — S3 Bucket

**Step 3.** Create the file `amplify/storage/resource.ts`:

```typescript
import { defineStorage } from '@aws-amplify/backend';

export const storage = defineStorage({
  name: 'documentAssistantStorage',
  access: (allow) => ({
    'raw/{entity_id}/*': [
      allow.entity('identity').to(['read', 'write', 'delete']),
    ],
    'processed/{entity_id}/*': [
      allow.entity('identity').to(['read', 'write', 'delete']),
    ],
  }),
});
```

**Access Paths Explanation:**

| Path | Meaning |
|---|---|
| `raw/{entity_id}/*` | Directory for raw files uploaded by users. `{entity_id}` maps to the Cognito Identity ID to isolate data between users. |
| `processed/{entity_id}/*` | Directory for large extracted text files (>300K characters). Restricted to the owner. |

---

### Create Data Schema — Document & UserQuota

**Step 4.** Open `amplify/data/resource.ts` and add the `Document` and `UserQuota` models:

```typescript
const schema = a.schema({
  Document: a
    .model({
      owner: a.string(),
      fileName: a.string().required(),
      fileType: a.string().required(),
      fileSize: a.integer().required(),
      s3Key: a.string().required(),
      status: a.enum(['uploaded', 'processing', 'text_extracted', 'done', 'error']),
      analysisMode: a.string(),
      summary: a.string(),
      category: a.string(),
      textractJobId: a.string(),
      extractedText: a.string(),
      processedS3Key: a.string(),
      createdAt: a.datetime().required(),
    })
    .secondaryIndexes((index) => [
      index('owner')
        .sortKeys(['createdAt'])
        .queryField('listDocumentsByOwnerAndCreatedAt')
        .name('owner-createdAt-index'),
      index('textractJobId')
        .queryField('listDocumentsByTextractJobId')
        .name('textractJobId-index'),
      index('s3Key')
        .queryField('listDocumentsByS3Key')
        .name('s3Key-index'),
    ])
    .authorization((allow) => [
      allow.owner(),
    ]),

  UserQuota: a
    .model({
      owner: a.string().required(),
      uploadedCount: a.integer().default(0),
      maxUploads: a.integer().default(50),
    })
    .identifier(['owner'])
    .authorization((allow) => [
      allow.owner(),
    ]),
});
```

**Document Model Fields:**

| Field | Type | Purpose |
|---|---|---|
| `owner` | `string` | Cognito User ID, used for owner-based authorization. |
| `fileName` | `string` (required) | Original name of the uploaded file. |
| `fileType` | `string` (required) | File extension/type: `pdf`, `docx`, `pptx`, `jpg`, `png`. |
| `s3Key` | `string` (required) | S3 key location of the raw file: `raw/{identityId}/{timestamp}-{filename}`. |
| `status` | `enum` | Pipeline state: `uploaded` → `processing` → `text_extracted` → `done` or `error`. |
| `analysisMode` | `string` | AI analysis options: `summary_detailed`, `summary_short`, `key_points`, `classify_only`, `extract_text`. |
| `textractJobId` | `string` | AWS Textract Job ID (used only for PDF scans and image files). |
| `extractedText` | `string` | Extracted text content (saved here if ≤300K characters). |
| `processedS3Key` | `string` | S3 path of the saved text file (saved here if >300K characters). |

**Secondary Indexes:**

| Index Name | Partition Key | Sort Key | Query Purpose |
|---|---|---|---|
| `owner-createdAt-index` | `owner` | `createdAt` | Retrieve user documents sorted chronologically. |
| `textractJobId-index` | `textractJobId` | — | Help Lambda B lookup the document from a Textract Job ID. |
| `s3Key-index` | `s3Key` | — | Help Lambda A find the document matching the S3 upload key. |

**UserQuota Model:**
- `identifier(['owner'])` — Uses the owner ID as the primary key instead of an auto-generated ID.
- Restricts each user to `maxUploads = 50`. `uploadedCount` increases by 1 each time Lambda A processes a file.

---

### Import into Backend

**Step 5.** Open `amplify/backend.ts` and import the declared resources:

```typescript
import { defineBackend } from '@aws-amplify/backend';
import { auth } from './auth/resource';
import { data } from './data/resource';
import { storage } from './storage/resource';
import { lambdaATrigger } from './functions/lambda-a-trigger/resource';
import { lambdaBTextractResult } from './functions/lambda-b-textract-result/resource';

const backend = defineBackend({
  auth,
  data,
  storage,
  lambdaATrigger,
  lambdaBTextractResult,
});
```

---

### Lambda Dependencies

**Step 6.** Ensure each Lambda subdirectory's `package.json` includes these dependencies:

| Package | Purpose | Used By |
|---|---|---|
| `@aws-sdk/client-dynamodb` | DynamoDB AWS SDK client | Lambda A & B |
| `@aws-sdk/lib-dynamodb` | DynamoDB Document Client helper | Lambda A & B |
| `@aws-sdk/client-s3` | S3 AWS SDK client | Lambda A & B |
| `@aws-sdk/client-textract` | Textract AWS SDK client | Lambda A & B |
| `@aws-sdk/client-bedrock-runtime` | Bedrock AI AWS SDK client | Lambda A & B |
| `mammoth` | DOCX parser | Lambda A |
| `officeparser` | PPTX / XLSX parser | Lambda A |
| `pdf-parse` | PDF text parser (retains Vietnamese accents) | Lambda A |

{{% notice warning %}}
`pdf-parse` is a CommonJS module. Since Amplify Gen 2 compiles TypeScript to ESM by default, create a type declaration file `amplify/functions/lambda-a-trigger/pdf-parse.d.ts` containing:
```typescript
declare module 'pdf-parse';
```
If you encounter import compilation errors, consult the Troubleshooting guide in [5.3.4](../5.3.4-verify-pipeline/).
{{% /notice %}}

---

### Expected Results

By the end of this step, you will have established:
- Directory `amplify/functions/lambda-a-trigger/` with `resource.ts` and `handler.ts`.
- Directory `amplify/functions/lambda-b-textract-result/` with `resource.ts` and `handler.ts`.
- S3 Storage resources defined under `amplify/storage/resource.ts`.
- DynamoDB data schema containing `Document` and `UserQuota` models in `amplify/data/resource.ts`.
- Central imports in `amplify/backend.ts`.

![PLACEHOLDER-5.3-01](/images/5-Workshop/5.3-Upload-pipeline/5.3-01-amplify-folder-structure.png)

{{% notice tip %}}
Do not deploy yet. We will configure the pipeline linkages inside `backend.ts` in the next section before running deployment.
{{% /notice %}}
