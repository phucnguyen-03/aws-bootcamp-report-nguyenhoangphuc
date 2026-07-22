---
title: "Upload Pipeline — Lambda A & Textract"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

## Upload Pipeline — Lambda A & Textract

Phần này hướng dẫn bạn xây dựng **core processing pipeline** của ứng dụng Smart Document Assistant — luồng xử lý chính từ lúc người dùng upload file cho đến khi có kết quả phân tích AI. Đây là phần phức tạp nhất trong toàn bộ workshop vì liên quan đến nhiều dịch vụ AWS tương tác với nhau theo kiến trúc **event-driven, fully async, serverless**.

Sau khi hoàn thành phần này, bạn sẽ có một pipeline hoạt động hoàn chỉnh:

1. User upload file → S3 trigger Lambda A
2. Lambda A trích xuất text (PDF native / Textract OCR / Office parse)
3. Textract hoàn thành → SNS notify → Lambda B nhận kết quả OCR
4. User chọn chế độ phân tích → DynamoDB Stream trigger Lambda B
5. Lambda B gọi AI (OpenRouter / Bedrock) → trả kết quả cuối cùng

---

### Kiến trúc tổng quan

![Architecture Diagram](/images/2-Proposal/architecture.png)

---

### Các resource AWS sẽ tạo trong phần này

| Resource | Tên / Identifier | Vai trò |
|---|---|---|
| **Lambda A** | `smart-doc-upload-trigger` | Nhận S3 event, trích xuất text từ file |
| **Lambda B** | `smart-doc-textract-result` | Nhận Textract callback (SNS) + chạy AI analysis (DDB Stream) |
| **S3 Bucket** | `smart-doc-storage-{account}-{region}` | Lưu file upload (`raw/`) và text trích xuất lớn (`processed/`) |
| **SNS Topic** | `textract-ocr-completed-topic` | Nhận callback từ Textract khi OCR hoàn thành |
| **IAM Role** | `TextractSnsPublishRole` | Cho phép Textract publish message vào SNS topic |
| **DynamoDB Tables** | `Document`, `UserQuota` | Lưu metadata tài liệu và quota upload của user |

---

### Luồng dữ liệu chi tiết

**Bước A — Frontend Upload:**
- Input: File từ user (PDF/DOCX/PPTX/JPG/PNG, max 10MB)
- Output: File trên S3 tại `raw/{identityId}/{timestamp}-{filename}` + Document record trong DynamoDB (`status='uploaded'`)
- Trigger: S3 `ObjectCreated:Put` event trên prefix `raw/`

**Bước B — Lambda A — Text Extraction (3 nhánh):**
- *PDF native text (>100 chars)*: Dùng `pdf-parse` → `status = 'text_extracted'` → chờ user chọn mode
- *PDF scan / ảnh*: Gọi Textract async → `status = 'processing'` → SNS callback → Lambda B
- *DOCX/PPTX/XLSX*: Parse local bằng `mammoth`/`officeParser` → `status = 'text_extracted'` → chờ user chọn mode

**Bước C — Lambda B — SNS Path (Textract Callback):**
- Input: SNS event chứa Textract job completion (JobId, Status)
- Output: Lấy OCR text → lưu vào DynamoDB hoặc S3 → `status = 'text_extracted'`

**Bước D — User chọn Analysis Mode (Frontend):**
- Frontend update DynamoDB: `status='processing'`, `analysisMode='summary_detailed|...'`
- Trigger: DynamoDB Stream event → Lambda B

**Bước E — Lambda B — DDB Stream Path (AI Analysis):**
- Gọi AI (OpenRouter → Bedrock fallback)
- Output: `status = 'done'`, `summary`, `category` (Hợp đồng|Hóa đơn|Báo cáo|Khác)

{{% notice info %}}
Nếu text trích xuất vượt quá 300.000 ký tự, hệ thống sẽ lưu text lên S3 tại `processed/{identityId}/{docId}-text.txt` thay vì lưu trực tiếp trong DynamoDB. DynamoDB chỉ lưu con trỏ `processedS3Key`.
{{% /notice %}}

---

### Các sub-section

| Sub-section | Nội dung |
|---|---|
| [5.3.1. Khai báo Lambda Functions & Storage](5.3.1-define-resources/) | Tạo resource definitions cho Lambda A, Lambda B, Storage và Data schema |
| [5.3.2. Cấu hình Backend — Override Names & S3 Event](5.3.2-backend-s3-event/) | Override tên resource, cấu hình S3 → Lambda A event notification |
| [5.3.3. SNS, Textract Role & DynamoDB Stream](5.3.3-sns-textract-ddb-stream/) | Tạo SNS topic, IAM role cho Textract, DynamoDB Stream trigger |
| [5.3.4. Kiểm tra & Xác minh Pipeline](5.3.4-verify-pipeline/) | Test từng bước trong pipeline, troubleshooting lỗi thường gặp |
