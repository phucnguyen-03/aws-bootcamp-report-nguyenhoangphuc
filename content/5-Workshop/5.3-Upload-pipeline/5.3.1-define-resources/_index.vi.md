---
title: "Khai báo Lambda Functions, Storage & Data Schema"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.3.1. </b> "
---

## Khai báo Lambda Functions, Storage & Data Schema

Trước khi cấu hình pipeline trong `backend.ts`, ta cần khai báo từng resource riêng lẻ theo cấu trúc của Amplify Gen 2. Mỗi resource (Lambda function, Storage, Data model) được define trong thư mục riêng dưới `amplify/`, sau đó import vào `backend.ts` để kết nối với nhau.

Phần này sẽ tạo:
- **Lambda A** (`lambda-a-trigger`): function nhận S3 event và trích xuất text
- **Lambda B** (`lambda-b-textract-result`): function nhận Textract callback (SNS) và chạy AI analysis (DDB Stream)
- **Storage** (`documentAssistantStorage`): S3 bucket với access paths phân tách theo user
- **Data schema**: Document model và UserQuota model trong DynamoDB

---

### Tạo Lambda A — Upload Trigger

**Bước 1.** Tạo thư mục `amplify/functions/lambda-a-trigger/` và file `resource.ts`:

```typescript
import { defineFunction, secret } from '@aws-amplify/backend';

export const lambdaATrigger = defineFunction({
  name: 'lambda-a-trigger',
  entry: './handler.ts',
  timeoutSeconds: 60,
  environment: {
    OPENROUTER_API_KEY: secret('OPENROUTER_API_KEY'),
  },
});
```



{{% notice note %}}
- `timeoutSeconds: 60` — Lambda A cần đủ thời gian để download file từ S3 và gọi Textract API.
- `secret('OPENROUTER_API_KEY')` — secret được lưu trong Amplify secret store (SSM Parameter Store), tự động inject vào environment variable khi deploy.
{{% /notice %}}

---

### Tạo Lambda B — Textract Result & AI Analysis

**Bước 2.** Tạo thư mục `amplify/functions/lambda-b-textract-result/` và file `resource.ts`:

```typescript
import { defineFunction, secret } from '@aws-amplify/backend';

export const lambdaBTextractResult = defineFunction({
  name: 'lambda-b-textract-result',
  entry: './handler.ts',
  timeoutSeconds: 300,
  environment: {
    OPENROUTER_API_KEY: secret('OPENROUTER_API_KEY'),
  },
});
```



{{% notice note %}}
`timeoutSeconds: 300` (5 phút) — Lambda B cần thời gian lâu hơn vì phải gọi AI model (OpenRouter / Bedrock) để phân tích văn bản, quá trình này có thể mất vài chục giây.
{{% /notice %}}

---

### Tạo Storage — S3 Bucket

**Bước 3.** Tạo file `amplify/storage/resource.ts`:

```typescript
import { defineStorage } from '@aws-amplify/backend';

export const storage = defineStorage({
  name: 'documentAssistantStorage',
  access: (allow) => ({
    'raw/{entity_id}/*': [
      allow.entity('identity').to(['read', 'write', 'delete']),
    ],
    'processed/{entity_id}/*': [
      allow.entity('identity').to(['read', 'write', 'delete']),
    ],
  }),
});
```



**Giải thích access paths:**

| Path | Ý nghĩa |
|---|---|
| `raw/{entity_id}/*` | Thư mục chứa file gốc user upload. `{entity_id}` = Cognito Identity ID, phân tách dữ liệu giữa các user. |
| `processed/{entity_id}/*` | Thư mục chứa text trích xuất lớn (>300K chars). Chỉ user sở hữu mới đọc/ghi/xóa được. |

---

### Tạo Data Schema — Document & UserQuota

**Bước 4.** Mở file `amplify/data/resource.ts` và thêm hai model `Document` và `UserQuota`:

```typescript
const schema = a.schema({
  Document: a
    .model({
      owner: a.string(),
      fileName: a.string().required(),
      fileType: a.string().required(),
      fileSize: a.integer().required(),
      s3Key: a.string().required(),
      status: a.enum(['uploaded', 'processing', 'text_extracted', 'done', 'error']),
      analysisMode: a.string(),
      summary: a.string(),
      category: a.string(),
      textractJobId: a.string(),
      extractedText: a.string(),
      processedS3Key: a.string(),
      createdAt: a.datetime().required(),
    })
    .secondaryIndexes((index) => [
      index('owner')
        .sortKeys(['createdAt'])
        .queryField('listDocumentsByOwnerAndCreatedAt')
        .name('owner-createdAt-index'),
      index('textractJobId')
        .queryField('listDocumentsByTextractJobId')
        .name('textractJobId-index'),
      index('s3Key')
        .queryField('listDocumentsByS3Key')
        .name('s3Key-index'),
    ])
    .authorization((allow) => [
      allow.owner(),
    ]),

  UserQuota: a
    .model({
      owner: a.string().required(),
      uploadedCount: a.integer().default(0),
      maxUploads: a.integer().default(50),
    })
    .identifier(['owner'])
    .authorization((allow) => [
      allow.owner(),
    ]),
});
```



**Giải thích Document model:**

| Field | Kiểu | Mục đích |
|---|---|---|
| `owner` | `string` | Cognito user ID, dùng cho owner-based authorization |
| `fileName` | `string` (required) | Tên file gốc user upload |
| `fileType` | `string` (required) | Loại file: `pdf`, `docx`, `pptx`, `jpg`, `png` |
| `s3Key` | `string` (required) | Đường dẫn S3 của file gốc: `raw/{identityId}/{timestamp}-{filename}` |
| `status` | `enum` | Trạng thái xử lý: `uploaded` → `processing` → `text_extracted` → `done` hoặc `error` |
| `analysisMode` | `string` | Chế độ phân tích AI: `summary_detailed`, `summary_short`, `key_points`, `classify_only`, `extract_text` |
| `textractJobId` | `string` | Job ID của Textract (chỉ khi file là PDF scan/ảnh) |
| `extractedText` | `string` | Toàn bộ text trích xuất (nếu ≤300K chars) |
| `processedS3Key` | `string` | Đường dẫn S3 của text file (nếu text >300K chars) |

**Secondary Indexes:**

| Index | Partition Key | Sort Key | Mục đích |
|---|---|---|---|
| `owner-createdAt-index` | `owner` | `createdAt` | Lấy danh sách tài liệu của user, sắp theo thời gian |
| `textractJobId-index` | `textractJobId` | — | Lambda B tìm document theo Textract Job ID (callback SNS) |
| `s3Key-index` | `s3Key` | — | Lambda A tìm document theo S3 key |

**UserQuota model:**
- `identifier(['owner'])` — dùng `owner` làm primary key thay vì auto-generated `id`.
- Mỗi user có tối đa `maxUploads = 50` lần upload. `uploadedCount` tăng 1 sau mỗi lần Lambda A xử lý thành công.

---

### Import vào Backend

**Bước 5.** Mở file `amplify/backend.ts` và import tất cả resource:

```typescript
import { defineBackend } from '@aws-amplify/backend';
import { auth } from './auth/resource';
import { data } from './data/resource';
import { storage } from './storage/resource';
import { lambdaATrigger } from './functions/lambda-a-trigger/resource';
import { lambdaBTextractResult } from './functions/lambda-b-textract-result/resource';

const backend = defineBackend({
  auth,
  data,
  storage,
  lambdaATrigger,
  lambdaBTextractResult,
});
```



---

### Dependencies cho Lambda Functions

**Bước 6.** Đảm bảo `package.json` trong thư mục của mỗi Lambda function có đầy đủ dependencies:

| Package | Mục đích | Dùng bởi |
|---|---|---|
| `@aws-sdk/client-dynamodb` | DynamoDB operations | Lambda A & B |
| `@aws-sdk/lib-dynamodb` | DynamoDB Document Client | Lambda A & B |
| `@aws-sdk/client-s3` | S3 operations | Lambda A & B |
| `@aws-sdk/client-textract` | Textract API | Lambda A (start) & B (get result) |
| `@aws-sdk/client-bedrock-runtime` | Bedrock AI | Lambda A & B |
| `mammoth` | DOCX → HTML → text | Lambda A |
| `officeparser` | PPTX/XLSX → text | Lambda A |
| `pdf-parse` | PDF → text (giữ Unicode) | Lambda A |

{{% notice warning %}}
`pdf-parse` là CommonJS module. Khi dùng trong ESM context (Amplify Gen 2 mặc định), bạn cần tạo file type declaration `amplify/functions/lambda-a-trigger/pdf-parse.d.ts` với nội dung:
```typescript
declare module 'pdf-parse';
```
Xem thêm phần Troubleshooting ở mục [5.3.4](../5.3.4-verify-pipeline/) nếu gặp lỗi import.
{{% /notice %}}

---

### Kết quả mong đợi

Sau khi hoàn thành bước này, bạn sẽ có:

- Thư mục `amplify/functions/lambda-a-trigger/` với file `resource.ts` và `handler.ts`
- Thư mục `amplify/functions/lambda-b-textract-result/` với file `resource.ts` và `handler.ts`
- File `amplify/storage/resource.ts` khai báo S3 bucket với 2 access paths
- File `amplify/data/resource.ts` chứa Document model (5 GSI indexes) và UserQuota model
- File `amplify/backend.ts` import đầy đủ 5 resource: `auth`, `data`, `storage`, `lambdaATrigger`, `lambdaBTextractResult`

![PLACEHOLDER-5.3-01](/images/5-Workshop/5.3-Upload-pipeline/5.3-01-amplify-folder-structure.png)

{{% notice tip %}}
Chưa deploy ở bước này. Các file `handler.ts` chứa logic xử lý sẽ được giải thích chi tiết trong các mục sau. Bước tiếp theo sẽ cấu hình `backend.ts` để kết nối các resource với nhau.
{{% /notice %}}
