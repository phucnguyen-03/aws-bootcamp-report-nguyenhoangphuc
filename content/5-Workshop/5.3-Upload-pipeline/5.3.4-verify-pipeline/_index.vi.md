---
title: "Kiểm tra & Xác minh Pipeline"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.3.4. </b> "
---

## Kiểm tra & Xác minh Pipeline

Sau khi deploy thành công, phần này hướng dẫn bạn kiểm tra từng bước trong pipeline để đảm bảo mọi thành phần hoạt động đúng. Chúng ta sẽ verify theo đúng luồng dữ liệu: Upload → Lambda A → Textract → SNS → Lambda B → AI Analysis.

---

### Verify S3 Event Notification → Lambda A

**Bước 1.** Upload một file bất kỳ (PDF, DOCX, hoặc ảnh JPG/PNG) qua frontend dashboard.

**Bước 2.** Mở **AWS Console → CloudWatch → Log groups**, tìm log group của Lambda A (`/aws/lambda/smart-doc-upload-trigger`).

**Bước 3.** Mở log stream mới nhất, xác nhận thấy các log sau:

```
Received S3 Event: { Records: [{ s3: { bucket: { name: 'smart-doc-storage-...' }, 
 object: { key: 'raw/ap-southeast-1%3A.../1234567890-test.pdf' } } }] }
Processing file: raw/ap-southeast-1:.../1234567890-test.pdf from bucket: ...
```

![PLACEHOLDER-5.3-03](/images/5-Workshop/5.3-Upload-pipeline/5.3-03-cloudwatch-lambda-a-s3-trigger.png)


---

### Verify PDF Native Text Extraction

**Bước 4.** Upload một file PDF có text thật (không phải ảnh scan). Kiểm tra log Lambda A:

```
Trying direct PDF text extraction for: raw/...
PDF has native text (1234 chars), skipping Textract.
```

Khi thấy dòng "skipping Textract", nghĩa là Lambda A đã dùng `pdf-parse` để trích xuất text trực tiếp, không cần gọi Textract OCR — giữ nguyên dấu tiếng Việt.

![PLACEHOLDER-5.3-04](/images/5-Workshop/5.3-Upload-pipeline/5.3-04-cloudwatch-pdf-native-text.png)


---

### Verify Office File Extraction (DOCX/PPTX)

**Bước 5.** Upload một file DOCX hoặc PPTX. Kiểm tra log Lambda A:

```
Parsing office file locally: raw/...
Extracted 5678 characters.
```

![PLACEHOLDER-5.3-05](/images/5-Workshop/5.3-Upload-pipeline/5.3-05-cloudwatch-office-file-extract.png)


---

### Verify Textract Async OCR (PDF scan / Ảnh)

**Bước 6.** Upload một file PDF dạng scan (ảnh text) hoặc file JPG/PNG. Kiểm tra log Lambda A:

```
Starting Textract async job for: raw/...
Textract Job started: <JobId>
```

**Bước 7.** Đợi vài giây, mở log group của Lambda B (`/aws/lambda/smart-doc-textract-result`). Xác nhận thấy:

```
[SNS] Textract callback. JobId: <JobId>, Status: SUCCEEDED
[SNS] OCR complete. <N> characters extracted.
[SNS] Document <id> ready for analysis (text_extracted).
```

![PLACEHOLDER-5.3-06](/images/5-Workshop/5.3-Upload-pipeline/5.3-06-cloudwatch-textract-lambda-a-start.png)


![PLACEHOLDER-5.3-07](/images/5-Workshop/5.3-Upload-pipeline/5.3-07-cloudwatch-lambda-b-sns-callback.png)


---

### Verify DynamoDB Stream → AI Analysis

**Bước 8.** Quay lại frontend, chọn một document đã ở trạng thái `text_extracted`. Chọn analysis mode (ví dụ: `summary_detailed`).

**Bước 9.** Kiểm tra log Lambda B (DDB Stream path):

```
[DDB Stream] Analyzing document <id> with mode: summary_detailed
[DDB Stream] Calling AI (mode=summary_detailed, textLen=1234)...
[DDB Stream] Document <id> analysis complete.
```

![PLACEHOLDER-5.3-08](/images/5-Workshop/5.3-Upload-pipeline/5.3-08-cloudwatch-lambda-b-ddb-stream-ai.png)


---

### Verify trên AWS Console

**Bước 10.** Kiểm tra S3 Console:
- Mở **S3 → Buckets → `smart-doc-storage-...`**
- Xác nhận thư mục `raw/` chứa file vừa upload
- Nếu text >300K chars, xác nhận thư mục `processed/` chứa file `-text.txt`

![PLACEHOLDER-5.3-09](/images/5-Workshop/5.3-Upload-pipeline/5.3-09-s3-console-raw-folder.png)


**Bước 11.** Kiểm tra DynamoDB Console:
- Mở **DynamoDB → Tables → Document table → Explore table items**
- Xác nhận document có progression trạng thái đúng: `uploaded` → `processing` → `text_extracted` → (user chọn mode) → `processing` → `done`

![PLACEHOLDER-5.3-10](/images/5-Workshop/5.3-Upload-pipeline/5.3-10-dynamodb-document-status-done.png)


**Bước 12.** Kiểm tra SNS Console:
- Mở **SNS → Topics → `textract-ocr-completed-topic`**
- Xác nhận có subscription active cho Lambda B

![PLACEHOLDER-5.3-11](/images/5-Workshop/5.3-Upload-pipeline/5.3-11-sns-console-subscription-active.png)


**Bước 13.** Kiểm tra Lambda Console:
- Mở **Lambda → Functions → `smart-doc-upload-trigger`** và **`smart-doc-textract-result`**
- Xác nhận invocation count tăng, không có errors

![PLACEHOLDER-5.3-12](/images/5-Workshop/5.3-Upload-pipeline/5.3-12-lambda-console-monitoring.png)


---

### Verify Quota

**Bước 14.** Sau khi upload file, kiểm tra **DynamoDB → Tables → UserQuota table → Explore table items**. Xác nhận `uploadedCount` tăng 1 sau mỗi lần upload thành công.

[THIẾU: không có log mẫu cụ thể cho quota check, cần test thực tế]

---

### Troubleshooting

#### Lỗi 1: CDK Circular Dependency (Cross-stack)

**Triệu chứng:** `cdk synth` hoặc `ampx deploy` fail với lỗi circular dependency giữa nested stacks (function ↔ storage, function ↔ parent).

**Nguyên nhân:** Dùng CDK token (`bucket.bucketArn`, `topic.topicArn`, `lambda.functionArn`) trong cross-stack references → CDK tạo import/export giữa stacks → circular.

**Cách fix:** Override tên resource thành string cố định → construct ARN bằng pseudo-params (`region`, `account`) → KHÔNG tạo CDK dependency giữa stacks. Xem lại mục 5.3.2 Bước 1-2.

---

#### Lỗi 2: S3 Notification Validation Error

**Triệu chứng:**
```
Unable to validate the following destination configurations
```
Hoặc `400 InvalidRequest` → stack rollback.

**Nguyên nhân:** CloudFormation tạo/update bucket notification TRƯỚC khi Lambda permission tồn tại → S3 không validate được destination.

**Cách fix:** Thêm `cfnBucket.addDependency(s3InvokePermission)` — force CloudFormation tạo permission TRƯỚC notification config. Xem lại mục 5.3.2 Bước 4.

---

#### Lỗi 3: CDK Assembly Error — "Cannot read properties of undefined"

**Triệu chứng:**
```
[BackendBuildError] Unable to deploy due to CDK Assembly Error
Caused by: [TypeError] Cannot read properties of undefined (reading 'addPropertyOverride')
```

**Nguyên nhân:** Cách lookup CfnResource (`node.defaultChild`) không đúng — object trả về `undefined`.

**Cách fix:** Kiểm tra lại cách truy cập CfnResource:
```typescript
// Đúng:
const cfnLambdaA = backend.lambdaATrigger.resources.lambda.node.defaultChild as cdk.CfnResource;
// Sai (nếu dùng construct ID khác):
const cfnLambdaA = backend.lambdaATrigger.resources.lambda.node.findChild('...') as cdk.CfnResource;
```

[THIẾU: diff chính xác của fix, cần check commit d55ced2 hoặc cf7d4ef]

---

#### Lỗi 4: IAM Role Name Conflict (TextractSnsPublishRole)

**Triệu chứng:** Deploy fail do role name `TextractSnsPublishRole` đã tồn tại (conflict giữa các deployment hoặc các environment khác nhau).

**Cách fix:**
- Xóa role cũ trong IAM Console nếu thuộc deployment cũ đã xóa
- Hoặc đổi tên role (ví dụ: thêm suffix environment)

[THIẾU: chi tiết cụ thể về conflict và cách resolve cuối cùng]

---

#### Lỗi 5: Vietnamese Diacritics bị mất (Textract OCR)

**Triệu chứng:** Textract OCR trả về text tiếng Việt mất dấu hoặc sai dấu.

**Nguyên nhân:** Textract OCR không giữ dấu tiếng Việt tốt cho PDF có text (không phải scan).

**Cách fix:** Thêm logic try `pdf-parse` TRƯỚC → nếu PDF có native text (>100 chars) thì dùng kết quả từ `pdf-parse` (giữ nguyên Unicode) → chỉ fallback Textract OCR khi PDF là scan (text quá ít). Xem logic trong Lambda A handler:

```typescript
const buffer = await downloadS3Object(bucketName, key);
const pdfData = await pdfParse(buffer);
const extractedText = pdfData.text?.trim() ?? '';
if (extractedText.length > 100) {
  // Có đủ text → lưu thẳng, skip Textract
  // ...
  continue; // bỏ qua Textract
}
// Nếu < 100 chars → PDF scan → gọi Textract
```

---

#### Lỗi 6: pdf-parse Import Issue (CommonJS trong ESM)

**Triệu chứng:** Import `pdf-parse` trong Lambda A handler gây lỗi — `pdf-parse` là CommonJS module, không tương thích ESM context mặc định của Amplify Gen 2.

**Cách fix:** Tạo file type declaration `amplify/functions/lambda-a-trigger/pdf-parse.d.ts`:

```typescript
declare module 'pdf-parse';
```

---

### Kết quả mong đợi

Khi toàn bộ pipeline hoạt động đúng, bạn sẽ thấy:

| Hành động | Kết quả mong đợi |
|---|---|
| Upload PDF có text | Lambda A log "PDF has native text, skipping Textract" → DynamoDB status = `text_extracted` |
| Upload PDF scan / ảnh | Lambda A log "Textract Job started" → vài giây sau Lambda B log "Textract callback SUCCEEDED" → status = `text_extracted` |
| Upload DOCX/PPTX | Lambda A log "Parsing office file locally" → status = `text_extracted` |
| Chọn analysis mode | Lambda B log "Analyzing document with mode: ..." → status = `done`, có `summary` và `category` |
| Kiểm tra DynamoDB | Document item có đầy đủ: `summary`, `category`, `extractedText` (hoặc `processedS3Key`), `status = done` |
| Kiểm tra SNS | Topic có subscription active cho Lambda B |
| Kiểm tra UserQuota | `uploadedCount` tăng sau mỗi lần upload |

![PLACEHOLDER-5.3-13](/images/5-Workshop/5.3-Upload-pipeline/5.3-13-frontend-document-done-result.png)

{{% notice tip %}}
Nếu bạn gặp lỗi không nằm trong danh sách trên, kiểm tra:
1. **CloudWatch Logs** — xem error message chi tiết trong log stream của Lambda A hoặc B
2. **CloudFormation Events** — xem event nào fail trong stack update
3. **IAM Policy** — đảm bảo Lambda có đủ permissions cho tất cả services (Textract, S3, DynamoDB, Bedrock, SNS)
{{% /notice %}}
