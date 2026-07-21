---
title: "Upload Pipeline — Lambda A & Textract"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

## Upload Pipeline — Lambda A & Textract

This section guides you through building the **core processing pipeline** of the Smart Document Assistant application — the main execution flow from the moment a user uploads a file until the final AI analysis results are generated. This is the most complex part of the workshop, involving multiple AWS services interacting in an **event-driven, fully async, and serverless** manner.

Once completed, you will have a fully functional pipeline:

1. User uploads a file → S3 triggers Lambda A
2. Lambda A extracts text (via PDF native parser, Textract OCR, or Office document parsers)
3. Textract finishes processing → publishes to SNS → Lambda B receives the OCR results
4. User selects an analysis mode → DynamoDB Stream triggers Lambda B
5. Lambda B invokes the AI (OpenRouter or Bedrock) → saves and displays the final analysis

---

### Architecture Overview

```
┌──────────┐    ┌──────┐   S3 Event    ┌──────────┐   Textract Async   ┌──────────┐
│ Frontend │───▶│  S3  │─────────────▶│ Lambda A │──────────────────▶│ Textract │
│ (Angular)│    │bucket│              │ (trigger)│                   │   OCR    │
└──────────┘    └──────┘              └──────────┘                   └────┬─────┘
                                           │                             │
                                           │ (DOCX/PPTX:                 │ SNS callback
                                           │  extract local)             ▼
                                           │                        ┌──────────┐
                                           ▼                        │   SNS    │
                                     ┌──────────┐                   │  Topic   │
                                     │ DynamoDB  │                   └────┬─────┘
                                     │ Document  │                        │
                                     │  Table    │◀───────────────────────┤
                                     └─────┬────┘                   ┌──────────┐
                                           │ DDB Stream             │ Lambda B │
                                           │ (MODIFY +              │ (textract│
                                           │  status=processing)    │  result) │
                                           ▼                        └──────────┘
                                     ┌──────────┐                        │
                                     │ Lambda B │◀───────────────────────┘
                                     │ (AI      │
                                     │ analysis)│
                                     └──────────┘
```

---

### AWS Resources to Create

| Resource | Name / Identifier | Role |
|---|---|---|
| **Lambda A** | `smart-doc-upload-trigger` | Receives S3 events and extracts text from uploaded files |
| **Lambda B** | `smart-doc-textract-result` | Receives Textract callbacks (SNS) + runs AI analysis (DDB Stream) |
| **S3 Bucket** | `smart-doc-storage-{account}-{region}` | Stores original files (`raw/`) and large extracted text (`processed/`) |
| **SNS Topic** | `textract-ocr-completed-topic` | Receives a notification callback from Textract when OCR is complete |
| **IAM Role** | `TextractSnsPublishRole` | Allows the Textract service to publish messages to the SNS topic |
| **DynamoDB Tables** | `Document`, `UserQuota` | Stores document metadata and tracks user upload quotas |

---

### Detailed Data Flow

**Step A — Frontend Upload:**
- Input: User file (PDF/DOCX/PPTX/JPG/PNG, max 10MB)
- Output: File uploaded to S3 at `raw/{identityId}/{timestamp}-{filename}` + Document record created in DynamoDB (`status='uploaded'`)
- Trigger: S3 `ObjectCreated:Put` event on the `raw/` prefix

**Step B — Lambda A — Text Extraction (3 paths):**
- *PDF native text (>100 chars)*: Parsed locally using `pdf-parse` → status becomes `text_extracted` → waits for user to choose mode
- *PDF scan / Image*: Invokes Textract async OCR → status becomes `processing` with `textractJobId` → SNS callback triggers Lambda B
- *DOCX/PPTX/XLSX*: Parsed locally using `mammoth`/`officeParser` → status becomes `text_extracted` → waits for user to choose mode

**Step C — Lambda B — SNS Path (Textract Callback):**
- Input: SNS event containing Textract job completion status and JobId
- Output: Fetches paginated OCR text → saves text to DynamoDB or S3 → status becomes `text_extracted`

**Step D — User selects Analysis Mode (Frontend):**
- Frontend updates the document record in DynamoDB: `status='processing'`, `analysisMode='summary_detailed|...'`
- Trigger: DynamoDB Stream event → Lambda B

**Step E — Lambda B — DDB Stream Path (AI Analysis):**
- Invokes AI model (OpenRouter with Bedrock fallback)
- Output: status becomes `done`, saves `summary` and `category` (Contract | Invoice | Report | Other)

{{% notice info %}}
If the extracted text exceeds 300,000 characters, it is saved as a text file in S3 under `processed/{identityId}/{docId}-text.txt` to prevent exceeding DynamoDB item size limits. The table only stores the S3 key reference under `processedS3Key`.
{{% /notice %}}

---

### Sub-sections

| Sub-section | Description |
|---|---|
| [5.3.1. Define Lambda Functions, Storage & Data Schema](5.3.1-define-resources/) | Create resource definitions for Lambda A, Lambda B, Storage, and DynamoDB data schema |
| [5.3.2. Backend Configuration — Override Names & S3 Event](5.3.2-backend-s3-event/) | Override resource names to bypass circular dependencies and configure S3 events |
| [5.3.3. SNS, Textract Role & DynamoDB Stream Configuration](5.3.3-sns-textract-ddb-stream/) | Set up the SNS topic, Textract IAM Role, SNS subscription, and DynamoDB Stream triggers |
| [5.3.4. Verification & Troubleshooting](5.3.4-verify-pipeline/) | Verify each stage of the pipeline and troubleshoot common setup issues |
