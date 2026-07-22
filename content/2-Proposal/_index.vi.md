---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Smart Document Assistant
## Giải pháp AWS Serverless cho xử lý tài liệu tự động

### 1. Tóm tắt

**Smart Document Assistant** là ứng dụng web cho phép người dùng tải lên tài liệu (PDF, Word, PowerPoint, ảnh), hệ thống tự động trích xuất văn bản bằng OCR và phân tích nội dung bằng AI để tạo ra bản tóm tắt và phân loại tài liệu theo chủ đề.

- **URL deploy:** https://main.d149j5w6r2swxd.amplifyapp.com
- **GitHub:** https://github.com/VuDaiLoc/smart-document-assistant

---

### 2. Vấn đề thực tế

Trong môi trường làm việc hiện đại, người dùng thường xuyên phải xử lý khối lượng lớn tài liệu dạng scan hoặc file văn phòng. Các vấn đề cụ thể:

- Tốn thời gian đọc và tóm tắt tài liệu thủ công
- Khó phân loại khi có nhiều tài liệu cùng lúc (hợp đồng, hóa đơn, báo cáo...)
- Tài liệu scan dạng ảnh không thể tìm kiếm hay copy text
- Lưu trữ phân tán, thiếu hệ thống quản lý tập trung

---

### 3. Giải pháp đề xuất

**Kiến trúc tổng thể:**

![Architecture Diagram](/images/2-Proposal/architecture.png)

**Các dịch vụ AWS sử dụng:**

| Dịch vụ | Vai trò |
|---|---|
| Amplify Hosting | Host Angular SPA, CI/CD tự động từ GitHub |
| Amazon S3 | Lưu file gốc (`raw/`) và text đã xử lý (`processed/`) |
| AWS Lambda ×2 | Xử lý pipeline tài liệu (upload trigger + OCR result) |
| Amazon Textract | OCR trích xuất text từ PDF và ảnh (bất đồng bộ) |
| Amazon SNS | Nhận callback async từ Textract khi OCR hoàn thành |
| Amazon DynamoDB | Lưu metadata và kết quả AI của từng tài liệu |
| AWS AppSync | GraphQL API cho frontend truy vấn dữ liệu |
| Amazon Cognito | Xác thực người dùng (email + OTP) |
| Amazon Bedrock | AI fallback — Amazon Nova Lite (`apac.amazon.nova-lite-v1:0`) |
| Amazon SES | Gửi email OTP khi đăng ký / reset mật khẩu |
| AWS SSM Parameter Store | Lưu trữ secrets (OPENROUTER_API_KEY) |
| AWS CloudFormation | Quản lý infrastructure as code qua Amplify CDK |

---

### 4. Thiết kế từng thành phần

#### 4.1. Frontend — Angular 22 SPA

- **Framework:** Angular 22 (standalone components, signals, computed)
- **Auth:** Đăng ký/đăng nhập email + OTP, forgot password, confirm reset
- **Dashboard:**
  - Upload drag-drop với progress bar, validate type + size (tối đa 10 MB)
  - Danh sách tài liệu dạng bảng với search theo tên, filter theo phân loại
  - Quota tracking (đếm số tài liệu thực tế từ Document list)
  - Polling 4 giây để cập nhật trạng thái xử lý
- **Preview Modal:** PDF inline qua `<iframe>`, ảnh qua `<img>`, DOCX/PPTX dạng text viewer; AI summary + category badge; nút Retry khi `status = error`
- **UX:** Toast notification, custom confirm dialog, dark/light theme, name prompt popup, responsive (mobile breakpoint 900px)

#### 4.2. Lambda A — `smart-doc-upload-trigger`

- **Trigger:** S3 `ObjectCreated:Put` tại prefix `raw/`
- **Runtime:** Node.js (TypeScript, bundled bằng esbuild)
- **PDF/JPG/PNG:** query DynamoDB theo `s3Key-index` GSI → cập nhật `status = processing` → gọi `StartDocumentTextDetection` → lưu `textractJobId`
- **DOCX/PPTX:** download file từ S3 → parse bằng `mammoth` hoặc `officeParser` → gọi AI pipeline → cập nhật `status = done`
- **AI Pipeline:** Primary: OpenRouter (`meta-llama/llama-3.3-70b:free`) → Fallback: Bedrock Nova Lite → nếu cả 2 fail: `status = error`

#### 4.3. Lambda B — `smart-doc-textract-result`

- **Trigger:** SNS topic `textract-ocr-completed-topic`
- **Runtime:** Node.js (TypeScript)
- Parse SNS message → query DynamoDB theo `textractJobId-index` GSI → gọi `GetDocumentTextDetection` (có pagination) → AI pipeline → nếu text > 300 KB lưu lên S3 `processed/` → cập nhật `status = done`

#### 4.4. DynamoDB Schema

**Bảng: Document**

| Trường | Mô tả |
|---|---|
| `id` (PK) | UUID |
| `owner` | Cognito `sub::username` |
| `fileName`, `fileType`, `fileSize` | Metadata file |
| `s3Key` | Đường dẫn S3 object |
| `status` | `uploaded` → `processing` → `text_extracted` → `done` → `error` |
| `textractJobId` | ID job Textract bất đồng bộ |
| `summary` | Tóm tắt do AI tạo |
| `category` | `Hợp đồng` / `Hóa đơn` / `Báo cáo` / `Khác` |
| `extractedText` | Văn bản OCR nếu < 300 KB |
| `processedS3Key` | Đường dẫn S3 nếu text > 300 KB |
| `createdAt` | Timestamp ISO |

**GSI:**
- `owner-createdAt-index` — query tài liệu theo user, sort theo thời gian
- `textractJobId-index` — Lambda B lookup từ Textract callback
- `s3Key-index` — Lambda A lookup từ S3 event

**Bảng: UserQuota** — `owner` (PK), `uploadedCount`, `maxUploads` (mặc định 50)

#### 4.5. Ghi chú hạ tầng

- Tên S3 bucket cố định: `smart-doc-storage-{account}-{region}`
- Tên Lambda cố định để tránh CDK circular dependency
- ARN được construct dạng string thay vì CDK token
- `S3 NotificationConfiguration` gắn trực tiếp vào `CfnBucket`
- Dùng `SNS CfnSubscription` thay `LambdaSubscription` để tránh cross-stack token
- `addDependency()` đảm bảo Lambda permission được tạo trước bucket notification

---

### 5. Kế hoạch triển khai

| Giai đoạn | Thời gian | Hoạt động |
|---|---|---|
| Giai đoạn 1 — Khởi tạo & Backend | Ngày 1 | Setup Angular 22 + Amplify Gen 2, schema DynamoDB + AppSync, Cognito auth, S3 storage rules |
| Giai đoạn 2 — Lambda Pipeline | Ngày 1–2 | Lambda A (S3 trigger, Textract, DOCX/PPTX parser), Lambda B (SNS trigger, OCR retrieval, AI), SNS + IAM roles |
| Giai đoạn 3 — Frontend | Ngày 2 | Auth components, dashboard (upload, list, preview, quota), search/filter, retry, toast, dark/light theme, responsive |
| Giai đoạn 4 — Deploy & Production | Ngày 3 | GitHub → Amplify Hosting, cấu hình `amplify.yml`, SES identity, SSM Parameter Store, test end-to-end |

---

### 6. Ước tính chi phí

**Region:** ap-southeast-1 (Singapore) | **Mức sử dụng:** ~50 lần upload/tháng, ~10 user

| Dịch vụ | Thông số | Chi phí/tháng |
|---|---|---|
| Amplify Hosting | 30 build phút, 1 GB served | ~$0.05 |
| Lambda ×2 | 200 invocations, 128 MB, 30s avg | $0.00 (Free Tier) |
| Amazon S3 | 2 GB storage, 200 requests | ~$0.05 |
| DynamoDB | On-demand, ~2.000 R/W | $0.00 (Free Tier) |
| AppSync | 50.000 requests | $0.00 (Free Tier) |
| Cognito | < 50.000 MAU | $0.00 (Free Tier) |
| Amazon Textract | 100 trang PDF/ảnh | ~$0.15 |
| Amazon SNS | 100 messages | $0.00 (Free Tier) |
| Amazon SES | 100 email OTP | ~$0.01 |
| Amazon Bedrock | Fallback only | ~$0.00 |
| SSM Parameter Store | Standard tier | $0.00 |
| CloudFormation | — | $0.00 |
| **Tổng** | | **~$0.26–$0.59/tháng** |
| **Tổng 12 tháng** | | **~$3.12–$7.08/năm** |

> **Ghi chú:** OpenRouter (free tier) là AI provider chính — chi phí Bedrock thực tế là $0. Textract là khoản chi lớn nhất, tính theo trang. DOCX/PPTX dùng thư viện cục bộ (mammoth/officeParser) không tốn Textract.

---

### 7. Đánh giá rủi ro

| Rủi ro | Mức độ | Cách xử lý |
|---|---|---|
| OpenRouter API không ổn định | Trung bình | Fallback tự động sang Bedrock |
| Textract timeout với file lớn | Thấp | Async job + SNS callback |
| DynamoDB giới hạn item 400 KB | Trung bình | Text > 300 KB lưu riêng trên S3 |
| Cognito email bị spam filter | Thấp | Dùng SES verified identity |
| CDK circular dependency | Cao | Stable names + construct ARN dạng string |
| S3 event notification race condition | Trung bình | `addDependency()` trong CDK |
| Bedrock throttling (token/day limit) | Trung bình | Đã gặp thực tế → chuyển sang OpenRouter |
| SES sandbox giới hạn recipient | Trung bình | Request SES production access |
| Node version mismatch trên Amplify | Trung bình | `nvm install 22.22.3` trong build spec |
| Angular budget exceeded | Thấp | Tăng `anyComponentStyle` budget |
| SSM Parameter Store key conflict | Trung bình | Dùng đúng path `/amplify/{appId}/main/` |

---

### 8. Kết quả kỳ vọng

**Chức năng hoàn chỉnh:**
- Đăng ký tài khoản, xác thực email OTP, đăng nhập, reset mật khẩu
- Upload PDF/Word/PowerPoint/ảnh (tối đa 10 MB)
- Tự động OCR và phân tích AI trong vòng 30–60 giây
- Dashboard với search theo tên file, filter theo phân loại AI
- Preview inline PDF/ảnh, đọc text DOCX/PPTX trong browser
- Tóm tắt AI + phân loại: Hợp đồng / Hóa đơn / Báo cáo / Khác
- Retry xử lý thất bại mà không cần tải lại file
- Quản lý quota 50 tài liệu/user
- Tên hiển thị cá nhân hóa (lưu Cognito user attribute)

**Chỉ số kỹ thuật:**
- Thời gian xử lý PDF trung bình: < 60 giây
- Thời gian xử lý DOCX/PPTX: < 15 giây
- Availability: 99.9% (serverless)
- Chi phí vận hành: < $1/tháng (mức sử dụng nhẹ)
- Không có server cần maintain: 100% serverless
