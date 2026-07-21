---
title: "Prerequisites & Infrastructure Setup"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

## Prerequisites & Infrastructure Setup

Before running the document processing pipeline, the following AWS infrastructure must be in place. All resources are provisioned automatically by **AWS Amplify Gen 2** via CDK.

---

### 1. Initialize the Amplify Project

```bash
npm create amplify@latest smart-document-assistant
cd smart-document-assistant
npm install @aws-amplify/backend @aws-amplify/backend-cli
npm install aws-amplify
```

---

### 2. Amazon Cognito — User Authentication

Define authentication in `amplify/auth/resource.ts`:

```typescript
export const auth = defineAuth({
  loginWith: {
    email: true,
  },
  email: {
    fromEmail: 'noreply@yourdomain.com',
    fromName: 'Smart Document Assistant',
  },
});
```

The Cognito User Pool handles:
- User registration and login via **Email/Password**
- **OTP verification** on sign-up (delivered via Amazon SES)
- Full forgot-password and reset-password flows

After login, Cognito issues a **JWT Token** used by AppSync for **User Pool Authorization** on all GraphQL requests. The Angular frontend uses **Amplify Auth SDK** — no manual token management needed.

> **Post-deploy step:** Go to SES Console → Verified Identities → verify the `fromEmail` address. Request SES production access to send to any recipient (sandbox only allows verified addresses).

---

### 3. Amazon S3 — Document Storage

Define storage in `amplify/storage/resource.ts`:

```typescript
export const storage = defineStorage({
  name: 'smart-doc-storage',
  access: (allow) => ({
    'raw/{entity_id}/*': [
      allow.entity('identity').to(['read', 'write', 'delete']),
    ],
    'processed/*': [
      allow.authenticated.to(['read']),
    ],
  }),
});
```

**Bucket structure:**

| Prefix | Purpose |
|---|---|
| `raw/` | Upload destination. Files stored at `raw/{identityId}/{timestamp}-{filename}` — data isolated per user. |
| `processed/` | Created by Lambda B when extracted text exceeds **300 KB** to avoid DynamoDB's 400 KB item size limit. |

The S3 `NotificationConfiguration` (`s3:ObjectCreated:Put` on `raw/`) is attached directly to `CfnBucket` in `amplify/backend.ts` — this is the pipeline entry point.

---

### 4. AWS AppSync — GraphQL API

Define the data schema in `amplify/data/resource.ts`:

```typescript
const schema = a.schema({
  Document: a.model({
    owner:          a.string(),
    fileName:       a.string(),
    fileType:       a.string(),
    fileSize:       a.integer(),
    s3Key:          a.string(),
    status:         a.string(),   // uploaded | processing | text_extracted | done | error
    textractJobId:  a.string(),
    summary:        a.string(),
    category:       a.string(),   // Hợp đồng | Hóa đơn | Báo cáo | Khác
    extractedText:  a.string(),
    processedS3Key: a.string(),
    createdAt:      a.string(),
  }).authorization((allow) => [allow.owner()]),

  UserQuota: a.model({
    uploadedCount: a.integer(),
    maxUploads:    a.integer(),   // default 50
  }).identifier(['owner'])
    .authorization((allow) => [allow.owner()]),
});

export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: 'userPool',
  },
});
```

Both models use `allow.owner()` — each user can only read and write their own records.

---

### 5. Run Sandbox Locally

```bash
npx ampx sandbox
```

This creates `amplify_outputs.json` with real AWS endpoints. The frontend reads this file at runtime to connect to Cognito, AppSync, and S3.

---

### 6. Connect to AWS SSM Parameter Store

Store the OpenRouter API key before deploying Lambda functions:

1. Open [AWS SSM Parameter Store](https://console.aws.amazon.com/systems-manager/parameters)
2. Create a new parameter:
   - **Name:** `/amplify/{APP_ID}/main/OPENROUTER_API_KEY`
   - **Type:** `SecureString`
   - **Value:** `sk-or-v1-...`
3. Enable Amazon Bedrock model access:
   - Open [Bedrock Console](https://console.aws.amazon.com/bedrock/) → **Model access** → enable **Amazon Nova Lite**
