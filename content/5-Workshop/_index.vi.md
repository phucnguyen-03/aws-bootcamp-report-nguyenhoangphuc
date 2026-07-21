---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Smart Document Assistant

#### Tổng quan

**Smart Document Assistant** là hệ thống xử lý tài liệu serverless xây dựng trên AWS. Người dùng tải file lên (PDF, Word, PowerPoint, ảnh) qua giao diện Angular, hệ thống tự động trích xuất văn bản bằng OCR và phân tích nội dung bằng AI để tạo tóm tắt và phân loại tài liệu theo chủ đề.

Kiến trúc gồm hai Lambda function kết nối qua SNS topic, DynamoDB lưu metadata, AppSync cung cấp GraphQL API, và Cognito xác thực người dùng.

#### Sơ đồ kiến trúc

```
Người dùng (Browser)
    │
    ▼ Tải file
Angular SPA (Amplify Hosting)
    │
    ▼ S3 PUT
Amazon S3 (raw/)
    │
    ▼ S3 Event Trigger (ObjectCreated:Put)
Lambda A — smart-doc-upload-trigger
    │                           │
    ▼ PDF / JPG / PNG           ▼ DOCX / PPTX
Amazon Textract             mammoth / officeParser
(OCR bất đồng bộ)           (Xử lý cục bộ)
    │                           │
    ▼ SNS Callback              │
Lambda B — smart-doc-textract-result
                │
                ▼ Phân tích AI
        OpenRouter API (Primary)
                │
                ▼ Fallback nếu lỗi
        Amazon Bedrock Nova Lite
                │
                ▼ { summary, category }
        Amazon DynamoDB
                │
                ▼ AppSync GraphQL
        Angular SPA — Dashboard
```

#### Nội dung

1. [Tổng quan Workshop](5.1-Workshop-overview/)
2. [Chuẩn bị & Thiết lập hạ tầng](5.2-Prerequiste/)
3. [Upload Pipeline — Lambda A & Textract](5.3-Upload-pipeline/)
