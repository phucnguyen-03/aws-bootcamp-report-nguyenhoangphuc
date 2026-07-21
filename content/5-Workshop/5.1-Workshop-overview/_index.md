---
title: "Workshop Overview"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

## System Architecture Overview

**Smart Document Assistant** is an end-to-end serverless document processing system. Users upload files via an Angular frontend hosted on AWS Amplify, and the system automatically extracts text, generates AI summaries, and classifies each document — all without manual intervention.

---

### AWS Services Used

| Service | Role |
|---|---|
| **Amplify Hosting** | Hosts Angular SPA, automated CI/CD from GitHub |
| **Amazon S3** | Stores uploaded files (`raw/`) and large extracted text (`processed/`) |
| **AWS Lambda ×2** | Two functions drive the document processing pipeline |
| **Amazon Textract** | Async OCR engine — extracts text from PDF and image files |
| **Amazon SNS** | Message broker for Textract async callback to Lambda B |
| **Amazon DynamoDB** | Stores document metadata, processing status, and AI results |
| **AWS AppSync** | GraphQL API consumed by the Angular frontend |
| **Amazon Cognito** | User authentication — email + OTP, forgot/reset password |
| **Amazon Bedrock** | AI fallback — Amazon Nova Lite (`apac.amazon.nova-lite-v1:0`) |
| **OpenRouter API** | Primary AI provider (meta-llama/llama-3.3-70b:free) for summarization and classification |
| **Amazon SES** | Sends OTP emails for registration and password reset |
| **AWS SSM Parameter Store** | Stores `OPENROUTER_API_KEY` as a SecureString |
| **AWS CloudFormation** | Manages all infrastructure as code via Amplify CDK |

---

### End-to-End Processing Flow

**Step 1 — Upload**
The user logs in via Cognito and uploads a document through the Angular dashboard. Amplify Storage SDK places the file at `raw/{identityId}/{timestamp}-{filename}` in S3, scoping it to the authenticated user.

**Step 2 — Trigger**
An S3 Event Notification (`s3:ObjectCreated:Put` on prefix `raw/`) automatically invokes Lambda A (`smart-doc-upload-trigger`).

**Step 3 — Text Extraction**
Lambda A branches based on file type:
- **PDF / JPG / PNG** → calls `StartDocumentTextDetection` (Textract async job), saves `textractJobId` to DynamoDB, sets `status = processing`
- **DOCX / PPTX** → downloads from S3, parses locally with `mammoth` or `officeParser`, proceeds directly to AI analysis

**Step 4 — Async Callback (PDF/image only)**
When Textract finishes, it publishes a message to the SNS topic `textract-ocr-completed-topic`. The SNS subscription triggers Lambda B (`smart-doc-textract-result`).

**Step 5 — AI Analysis**
Lambda B (or Lambda A for DOCX/PPTX) calls the AI pipeline:
- Primary: **OpenRouter API** (`meta-llama/llama-3.3-70b:free`)
- Fallback: **Amazon Bedrock Nova Lite**
- If both fail: `status = error`

The AI returns `{ summary, category }` where category is one of: `Hợp đồng`, `Hóa đơn`, `Báo cáo`, `Khác`.

**Step 6 — Storage & Display**
Results are written to DynamoDB (`status = done`, `summary`, `category`, `extractedText`). If extracted text exceeds 300 KB, it is stored in S3 `processed/` and a `processedS3Key` pointer is saved instead. The Angular dashboard polls AppSync GraphQL every 4 seconds to display updated status and results.

---

### Data Isolation & Security

- Files are stored at `raw/{identityId}/` — scoped to each user's Cognito Identity ID.
- AppSync enforces `allow.owner()` authorization — users can only read and write their own documents.
- The `owner` field in DynamoDB is validated against the Cognito JWT on every AppSync request.
- `OPENROUTER_API_KEY` is stored in SSM Parameter Store as a `SecureString` and injected as a Lambda environment variable at deploy time.
