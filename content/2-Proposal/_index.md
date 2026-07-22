---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Smart Document Assistant
## A Serverless AWS Solution for Automated Document Processing

### 1. Executive Summary

**Smart Document Assistant** is a web application that allows users to upload documents (PDF, Word, PowerPoint, images), automatically extract text via OCR, and analyze content using AI to generate summaries and classify documents by topic.

- **Live URL:** https://main.d149j5w6r2swxd.amplifyapp.com
- **GitHub:** https://github.com/VuDaiLoc/smart-document-assistant

---

### 2. Problem Statement

In modern work environments, users frequently deal with large volumes of scanned documents and office files. Key pain points include:

- Time-consuming manual reading and summarization of documents
- Difficulty classifying large numbers of documents simultaneously (contracts, invoices, reports, etc.)
- Scanned image documents cannot be searched or have text copied
- Fragmented storage with no centralized management system

---

### 3. Solution Architecture

**Overall architecture:**

![Architecture Diagram](/images/2-Proposal/architecture.png)

**AWS Services used:**

| Service | Role |
|---|---|
| Amplify Hosting | Host Angular SPA, automated CI/CD from GitHub |
| Amazon S3 | Store original files (`raw/`) and processed text (`processed/`) |
| AWS Lambda ×2 | Document processing pipeline (upload trigger + OCR result) |
| Amazon Textract | OCR text extraction from PDF and images (async) |
| Amazon SNS | Receive async callback from Textract when OCR completes |
| Amazon DynamoDB | Store metadata and AI results for each document |
| AWS AppSync | GraphQL API for frontend data queries |
| Amazon Cognito | User authentication (email + OTP) |
| Amazon Bedrock | AI fallback — Amazon Nova Lite (`apac.amazon.nova-lite-v1:0`) |
| Amazon SES | Send OTP emails for registration / password reset |
| AWS SSM Parameter Store | Store secrets (OPENROUTER_API_KEY) |
| AWS CloudFormation | Infrastructure as code via Amplify CDK |

---

### 4. Component Design

#### 4.1. Frontend — Angular 22 SPA

- **Framework:** Angular 22 (standalone components, signals, computed)
- **Auth:** Email + OTP registration/login, forgot password, confirm reset
- **Dashboard:**
  - Drag-and-drop upload with progress bar, validates type + size (max 10 MB)
  - Document list table with search by name, filter by AI classification
  - Quota tracking (counts actual documents from Document list)
  - 4-second polling to update processing status
- **Preview Modal:** inline PDF via `<iframe>`, images via `<img>`, DOCX/PPTX as text viewer; AI summary + category badge; Retry button when `status = error`
- **UX:** Toast notifications, custom confirm dialog, dark/light theme, name prompt popup, responsive layout (mobile breakpoint 900px)

#### 4.2. Lambda A — `smart-doc-upload-trigger`

- **Trigger:** S3 `ObjectCreated:Put` at prefix `raw/`
- **Runtime:** Node.js (TypeScript, bundled with esbuild)
- **PDF/JPG/PNG:** queries DynamoDB via `s3Key-index` GSI → sets `status = processing` → calls `StartDocumentTextDetection` → saves `textractJobId`
- **DOCX/PPTX:** downloads file from S3 → parses with `mammoth` or `officeParser` → calls AI pipeline → sets `status = done`
- **AI Pipeline:** Primary: OpenRouter (`meta-llama/llama-3.3-70b:free`) → Fallback: Bedrock Nova Lite → if both fail: `status = error`

#### 4.3. Lambda B — `smart-doc-textract-result`

- **Trigger:** SNS topic `textract-ocr-completed-topic`
- **Runtime:** Node.js (TypeScript)
- Parses SNS message → queries DynamoDB via `textractJobId-index` GSI → calls `GetDocumentTextDetection` (paginated) → AI pipeline → if text > 300 KB saves to S3 `processed/` → sets `status = done`

#### 4.4. DynamoDB Schema

**Table: Document**

| Field | Description |
|---|---|
| `id` (PK) | UUID |
| `owner` | Cognito `sub::username` |
| `fileName`, `fileType`, `fileSize` | File metadata |
| `s3Key` | S3 object path |
| `status` | `uploaded` → `processing` → `text_extracted` → `done` → `error` |
| `textractJobId` | Async Textract job ID |
| `summary` | AI-generated summary |
| `category` | `Hợp đồng` / `Hóa đơn` / `Báo cáo` / `Khác` |
| `extractedText` | OCR text if < 300 KB |
| `processedS3Key` | S3 path if text > 300 KB |
| `createdAt` | ISO timestamp |

**GSIs:**
- `owner-createdAt-index` — query documents by user, sorted by time
- `textractJobId-index` — Lambda B lookup from Textract callback
- `s3Key-index` — Lambda A lookup from S3 event

**Table: UserQuota** — `owner` (PK), `uploadedCount`, `maxUploads` (default 50)

#### 4.5. Infrastructure Notes

- Fixed S3 bucket name: `smart-doc-storage-{account}-{region}`
- Fixed Lambda names to avoid CDK circular dependency
- ARNs constructed as strings rather than CDK tokens
- S3 `NotificationConfiguration` attached directly to `CfnBucket`
- `SNS CfnSubscription` used instead of `LambdaSubscription` to avoid cross-stack tokens
- `addDependency()` ensures Lambda permission is created before bucket notification

---

### 5. Implementation Plan

| Phase | Timeline | Activities |
|---|---|---|
| Phase 1 — Init & Backend | Day 1 | Angular 22 + Amplify Gen 2 setup, DynamoDB + AppSync schema, Cognito auth, S3 storage rules |
| Phase 2 — Lambda Pipeline | Day 1–2 | Lambda A (S3 trigger, Textract, DOCX/PPTX parser), Lambda B (SNS trigger, OCR retrieval, AI), SNS + IAM roles |
| Phase 3 — Frontend | Day 2 | Auth components, dashboard (upload, list, preview, quota), search/filter, retry, toast, dark/light theme, responsive |
| Phase 4 — Deploy & Production | Day 3 | GitHub → Amplify Hosting, `amplify.yml` config, SES identity, SSM Parameter Store, end-to-end test |

---

### 6. Budget Estimation

**Region:** ap-southeast-1 (Singapore) | **Usage:** ~50 uploads/month, ~10 users

| Service | Parameters | Cost/month |
|---|---|---|
| Amplify Hosting | 30 build min, 1 GB served | ~$0.05 |
| Lambda ×2 | 200 invocations, 128 MB, 30s avg | $0.00 (Free Tier) |
| Amazon S3 | 2 GB storage, 200 requests | ~$0.05 |
| DynamoDB | On-demand, ~2,000 R/W | $0.00 (Free Tier) |
| AppSync | 50,000 requests | $0.00 (Free Tier) |
| Cognito | < 50,000 MAU | $0.00 (Free Tier) |
| Amazon Textract | 100 pages PDF/image | ~$0.15 |
| Amazon SNS | 100 messages | $0.00 (Free Tier) |
| Amazon SES | 100 OTP emails | ~$0.01 |
| Amazon Bedrock | Fallback only | ~$0.00 |
| SSM Parameter Store | Standard tier | $0.00 |
| CloudFormation | — | $0.00 |
| **Total** | | **~$0.26–$0.59/month** |
| **Total 12 months** | | **~$3.12–$7.08/year** |

> **Note:** OpenRouter (free tier) is the primary AI provider — Bedrock cost is effectively $0. Textract is the largest cost item, billed per page. DOCX/PPTX files use local libraries (mammoth/officeParser) and do not incur Textract charges.

---

### 7. Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| OpenRouter API instability | Medium | Automatic fallback to Bedrock |
| Textract timeout on large files | Low | Async job + SNS callback |
| DynamoDB 400 KB item size limit | Medium | Text > 300 KB stored separately on S3 |
| Cognito emails caught by spam filters | Low | Use SES verified identity |
| CDK circular dependency | High | Stable names + construct ARN as string |
| S3 event notification race condition | Medium | `addDependency()` in CDK |
| Bedrock throttling (token/day limit) | Medium | Encountered in practice → switched to OpenRouter |
| SES sandbox recipient restrictions | Medium | Request SES production access |
| Node version mismatch on Amplify | Medium | `nvm install 22.22.3` in build spec |
| Angular budget exceeded | Low | Increase `anyComponentStyle` budget |
| SSM Parameter Store key conflict | Medium | Use correct path `/amplify/{appId}/main/` |

---

### 8. Expected Outcomes

**Functional deliverables:**
- Account registration with email OTP, login, password reset
- Upload PDF/Word/PowerPoint/images (max 10 MB)
- Automatic OCR and AI analysis within 30–60 seconds
- Dashboard with search by filename, filter by AI classification
- Inline PDF/image preview, text viewer for DOCX/PPTX in browser
- AI summary + classification: Hợp đồng / Hóa đơn / Báo cáo / Khác
- Retry failed processing without re-uploading the file
- 50-document quota per user
- Personalized display name (stored in Cognito user attribute)

**Technical metrics:**
- Average PDF processing time: < 60 seconds
- DOCX/PPTX processing time: < 15 seconds
- Availability: 99.9% (serverless)
- Operating cost: < $1/month (light usage)
- Server maintenance: none — 100% serverless
