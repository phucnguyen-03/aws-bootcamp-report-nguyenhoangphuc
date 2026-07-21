---
title: "Verification & Troubleshooting"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> <b> 5.3.4. </b> </b> "
---

## Verification & Troubleshooting

After deploying your resources, follow these steps to verify that every component in the document processing pipeline is working correctly. We will test the data path step-by-step: Upload → Lambda A → Textract OCR → SNS → Lambda B → AI Analysis.

---

### Verify S3 Event Notification → Lambda A

**Step 1.** Upload a test file (PDF, DOCX, or image) through the frontend dashboard.

**Step 2.** Go to **AWS Console → CloudWatch → Log groups** and select the log group for Lambda A (`/aws/lambda/smart-doc-upload-trigger`).

**Step 3.** Open the latest log stream and confirm that you see the following log entries:

```
Received S3 Event: { Records: [{ s3: { bucket: { name: 'smart-doc-storage-...' }, 
 object: { key: 'raw/ap-southeast-1%3A.../1234567890-test.pdf' } } }] }
Processing file: raw/ap-southeast-1:.../1234567890-test.pdf from bucket: ...
```

![PLACEHOLDER-5.3-03](/images/5-Workshop/5.3-Upload-pipeline/5.3-03-cloudwatch-lambda-a-s3-trigger.png)

---

### Verify PDF Native Text Extraction

**Step 4.** Upload a native PDF (containing real selectable text, not scanned images). Review Lambda A's logs:

```
Trying direct PDF text extraction for: raw/...
PDF has native text (1234 chars), skipping Textract.
```

If you see "skipping Textract," Lambda A has successfully bypassed the Textract OCR process and extracted the text locally using the `pdf-parse` library, preserving formatting and Vietnamese characters.

![PLACEHOLDER-5.3-04](/images/5-Workshop/5.3-Upload-pipeline/5.3-04-cloudwatch-pdf-native-text.png)


---

### Verify Office Document Local Parsing (DOCX/PPTX)

**Step 5.** Upload a `.docx` or `.pptx` file. Review Lambda A's logs:

```
Parsing office file locally: raw/...
Extracted 5678 characters.
```

![PLACEHOLDER-5.3-05](/images/5-Workshop/5.3-Upload-pipeline/5.3-05-cloudwatch-office-file-extract.png)


---

### Verify Textract Async OCR (Scans/Images)

**Step 6.** Upload a scanned PDF or a raw image file (`.jpg`/`.png`). Review Lambda A's logs:

```
Starting Textract async job for: raw/...
Textract Job started: <JobId>
```

**Step 7.** Wait a few seconds, then open the log group for Lambda B (`/aws/lambda/smart-doc-textract-result`). Look for the following entries:

```
[SNS] Textract callback. JobId: <JobId>, Status: SUCCEEDED
[SNS] OCR complete. <N> characters extracted.
[SNS] Document <id> ready for analysis (text_extracted).
```

![PLACEHOLDER-5.3-06](/images/5-Workshop/5.3-Upload-pipeline/5.3-06-cloudwatch-textract-lambda-a-start.png)


![PLACEHOLDER-5.3-07](/images/5-Workshop/5.3-Upload-pipeline/5.3-07-cloudwatch-lambda-b-sns-callback.png)

---

### Verify DynamoDB Stream → AI Analysis

**Step 8.** On the frontend, choose a document that has reached the `text_extracted` status. Select an analysis mode (e.g., `summary_detailed`).

**Step 9.** Open Lambda B's logs and verify that the DynamoDB Stream event has triggered the AI model:

```
[DDB Stream] Analyzing document <id> with mode: summary_detailed
[DDB Stream] Calling AI (mode=summary_detailed, textLen=1234)...
[DDB Stream] Document <id> analysis complete.
```

![PLACEHOLDER-5.3-08](/images/5-Workshop/5.3-Upload-pipeline/5.3-08-cloudwatch-lambda-b-ddb-stream-ai.png)

---

### Verify in the AWS Console

**Step 10.** **S3 Console:**
- Go to the **S3 Console → Buckets → `smart-doc-storage-...`**.
- Confirm that the uploaded file is in the `raw/` folder.
- If the extracted text was >300K characters, verify that a corresponding text file exists in the `processed/` folder.

![PLACEHOLDER-5.3-09](/images/5-Workshop/5.3-Upload-pipeline/5.3-09-s3-console-raw-folder.png)


**Step 11.** **DynamoDB Console:**
- Go to the **DynamoDB Console → Tables → Document table**.
- Select **Explore table items**.
- Verify that the document's status progressed: `uploaded` → `processing` → `text_extracted` → `processing` → `done`.

![PLACEHOLDER-5.3-10](/images/5-Workshop/5.3-Upload-pipeline/5.3-10-dynamodb-document-status-done.png)


**Step 12.** **SNS Console:**
- Go to the **SNS Console → Topics → `textract-ocr-completed-topic`**.
- Confirm that Lambda B has an active, confirmed subscription.

![PLACEHOLDER-5.3-11](/images/5-Workshop/5.3-Upload-pipeline/5.3-11-sns-console-subscription-active.png)


**Step 13.** **Lambda Console:**
- Go to **Lambda → Functions** and review invocation charts for both functions. Verify that the invocation counts increased without errors.

![PLACEHOLDER-5.3-12](/images/5-Workshop/5.3-Upload-pipeline/5.3-12-lambda-console-monitoring.png)


---

### Verify Quota Calculations

**Step 14.** Open **DynamoDB Console → Tables → UserQuota table** and verify that `uploadedCount` increases by 1 for each new document upload.

*[MISSING: Log formats for quota logic require confirmation]*

---

### Troubleshooting

#### Issue 1: CDK Circular Dependency (Cross-stack)

- **Symptom:** Running `cdk synth` or `ampx deploy` fails with cross-stack import dependency errors (e.g., function ↔ storage).
- **Cause:** Linking raw construct references (like `bucket.bucketArn` or `lambda.functionArn`) across nested stacks compiles into circular logical imports.
- **Resolution:** Override logical names with static strings and construct ARNs manually using region/account parameters. Review [5.3.2 Step 1-2](../5.3.2-backend-s3-event/).

---

#### Issue 2: S3 Notification Validation Failures

- **Symptom:** Deployment rolls back with the error:
  ```
  Unable to validate the following destination configurations
  ```
- **Cause:** CloudFormation attempts to apply the notification config to the S3 bucket before S3 has permission to invoke the Lambda function.
- **Resolution:** Explicitly require the S3 invocation permissions block to be created first: `cfnBucket.addDependency(s3InvokePermission)`. Review [5.3.2 Step 4](../5.3.2-backend-s3-event/).

---

#### Issue 3: CDK Assembly Error — "Cannot read properties of undefined"

- **Symptom:** Deploy fails during validation checks with:
  ```
  [BackendBuildError] Unable to deploy due to CDK Assembly Error
  Caused by: [TypeError] Cannot read properties of undefined (reading 'addPropertyOverride')
  ```
- **Cause:** The default child lookup target (`node.defaultChild`) returned `undefined` because of an invalid construct ID reference.
- **Resolution:** Double-check your node traversal syntax:
  ```typescript
  // Correct method:
  const cfnLambdaA = backend.lambdaATrigger.resources.lambda.node.defaultChild as cdk.CfnResource;
  ```

*[MISSING: Exact diff mapping commit d55ced2 is required to confirm the lookup syntax]*

---

#### Issue 4: IAM Role Name Collision (TextractSnsPublishRole)

- **Symptom:** Deployment fails because the role `TextractSnsPublishRole` already exists.
- **Resolution:** Delete orphaned roles from the IAM console or update the code to append environment suffixes to the role name.

*[MISSING: Final fix strategy details require confirmation]*

---

#### Issue 5: Vietnamese Accents Missing from Textract OCR Output

- **Symptom:** Non-native PDF text output contains corrupted Unicode or missing Vietnamese accents.
- **Cause:** Scanned PDFs are routed through OCR, whereas digital PDFs can be parsed directly.
- **Resolution:** Try parsing the file locally with `pdf-parse` first. Only fall back to AWS Textract OCR if the locally extracted text is less than 100 characters:

  ```typescript
  const buffer = await downloadS3Object(bucketName, key);
  const pdfData = await pdfParse(buffer);
  const extractedText = pdfData.text?.trim() ?? '';
  if (extractedText.length > 100) {
    // Save locally parsed text and skip AWS Textract
    continue;
  }
  // Fall back to Textract for scanned documents
  ```

---

#### Issue 6: pdf-parse Import Failures (CommonJS inside ESM)

- **Symptom:** The build tool throws import resolution errors for `pdf-parse` inside the handler.
- **Cause:** `pdf-parse` is packaged as a CommonJS module, which is incompatible with Gen 2's default ESM compilation output.
- **Resolution:** Create a type declaration file `pdf-parse.d.ts` inside the Lambda function's source directory:

  ```typescript
  declare module 'pdf-parse';
  ```

---

### Pipeline Verification Checklist

Once everything is working correctly, you should observe these behaviors:

| User Action | Pipeline Response |
|---|---|
| Upload digital PDF | Local parsing triggers → DynamoDB status changes to `text_extracted`. |
| Upload scanned document | AWS Textract runs OCR → Lambda B receives SNS callback → status changes to `text_extracted`. |
| Upload DOCX/PPTX | Local Mammoth parser runs → status changes to `text_extracted`. |
| Choose analysis mode | DynamoDB Stream triggers Lambda B → status changes to `done` with AI `summary` and `category` set. |
| Check DynamoDB | The `Document` item contains metadata, category classification, and summaries. |
| Check SNS | The subscription is active and mapped to the Lambda B endpoint. |
| Check Quota | The user's `uploadedCount` increases after each upload. |

![PLACEHOLDER-5.3-13](/images/5-Workshop/5.3-Upload-pipeline/5.3-13-frontend-document-done-result.png)


{{% notice tip %}}
If your tests fail, check:
1. **CloudWatch Logs** for detailed stack traces.
2. **CloudFormation Event logs** for deployment errors.
3. **IAM Role policies** to confirm permissions for Textract, S3, DynamoDB, Bedrock, and SNS.
{{% /notice %}}
