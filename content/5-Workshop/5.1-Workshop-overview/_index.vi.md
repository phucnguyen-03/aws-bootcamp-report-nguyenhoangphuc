---
title: "Tổng quan Workshop"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

## Tổng quan kiến trúc hệ thống

**Smart Document Assistant** là hệ thống xử lý tài liệu serverless hoàn chỉnh từ đầu đến cuối. Người dùng tải file lên qua giao diện Angular được host trên AWS Amplify, hệ thống tự động trích xuất văn bản, tạo tóm tắt AI và phân loại tài liệu — không cần bất kỳ thao tác thủ công nào.

---

### Các dịch vụ AWS sử dụng

| Dịch vụ | Vai trò |
|---|---|
| **Amplify Hosting** | Host Angular SPA, CI/CD tự động từ GitHub |
| **Amazon S3** | Lưu file tải lên (`raw/`) và text trích xuất lớn (`processed/`) |
| **AWS Lambda ×2** | Hai function điều khiển pipeline xử lý tài liệu |
| **Amazon Textract** | OCR bất đồng bộ — trích xuất text từ PDF và ảnh |
| **Amazon SNS** | Message broker cho callback bất đồng bộ từ Textract đến Lambda B |
| **Amazon DynamoDB** | Lưu metadata, trạng thái xử lý và kết quả AI |
| **AWS AppSync** | GraphQL API cho Angular frontend |
| **Amazon Cognito** | Xác thực người dùng — email + OTP, forgot/reset password |
| **Amazon Bedrock** | AI fallback — Amazon Nova Lite (`apac.amazon.nova-lite-v1:0`) |
| **OpenRouter API** | AI provider chính (meta-llama/llama-3.3-70b:free) để tóm tắt và phân loại |
| **Amazon SES** | Gửi email OTP khi đăng ký và reset mật khẩu |
| **AWS SSM Parameter Store** | Lưu `OPENROUTER_API_KEY` dạng SecureString |
| **AWS CloudFormation** | Quản lý toàn bộ hạ tầng dưới dạng code qua Amplify CDK |

---

### Luồng xử lý từ đầu đến cuối

**Bước 1 — Tải lên**
Người dùng đăng nhập qua Cognito và tải tài liệu lên qua dashboard Angular. Amplify Storage SDK đặt file tại `raw/{identityId}/{timestamp}-{filename}` trong S3, phân tách dữ liệu theo từng người dùng.

**Bước 2 — Kích hoạt**
S3 Event Notification (`s3:ObjectCreated:Put` trên prefix `raw/`) tự động invoke Lambda A (`smart-doc-upload-trigger`).

**Bước 3 — Trích xuất văn bản**
Lambda A phân nhánh theo loại file:
- **PDF / JPG / PNG** → gọi `StartDocumentTextDetection` (job Textract bất đồng bộ), lưu `textractJobId` vào DynamoDB, cập nhật `status = processing`
- **DOCX / PPTX** → tải file từ S3, parse cục bộ bằng `mammoth` hoặc `officeParser`, chuyển thẳng sang phân tích AI

**Bước 4 — Callback bất đồng bộ (chỉ PDF/ảnh)**
Khi Textract hoàn thành, dịch vụ publish message vào SNS topic `textract-ocr-completed-topic`. SNS subscription kích hoạt Lambda B (`smart-doc-textract-result`).

**Bước 5 — Phân tích AI**
Lambda B (hoặc Lambda A với DOCX/PPTX) gọi AI pipeline:
- Primary: **OpenRouter API** (`meta-llama/llama-3.3-70b:free`)
- Fallback: **Amazon Bedrock Nova Lite**
- Nếu cả hai fail: `status = error`

AI trả về `{ summary, category }` với category là một trong: `Hợp đồng`, `Hóa đơn`, `Báo cáo`, `Khác`.

**Bước 6 — Lưu trữ & Hiển thị**
Kết quả ghi vào DynamoDB (`status = done`, `summary`, `category`, `extractedText`). Nếu text trích xuất vượt 300 KB, nội dung lưu lên S3 `processed/` và chỉ lưu con trỏ `processedS3Key` vào DynamoDB. Dashboard Angular polling AppSync GraphQL mỗi 4 giây để cập nhật trạng thái và kết quả.

---

### Phân tách dữ liệu & Bảo mật

- File được lưu tại `raw/{identityId}/` — phân tách theo Cognito Identity ID của từng người dùng.
- AppSync thực thi `allow.owner()` — người dùng chỉ đọc và ghi được dữ liệu của chính mình.
- Trường `owner` trong DynamoDB được xác thực với Cognito JWT trên mọi request AppSync.
- `OPENROUTER_API_KEY` lưu trong SSM Parameter Store dạng `SecureString`, được inject vào Lambda qua environment variable lúc deploy.
