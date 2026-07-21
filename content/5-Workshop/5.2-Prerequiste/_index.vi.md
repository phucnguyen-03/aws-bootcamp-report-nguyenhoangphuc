---
title: "Chuẩn bị & Thiết lập hạ tầng"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

## Chuẩn bị & Thiết lập hạ tầng

Trước khi chạy pipeline xử lý tài liệu, cần thiết lập hạ tầng AWS dưới đây. Toàn bộ tài nguyên được **AWS Amplify Gen 2** tự động tạo qua CDK.

---

### 1. Khởi tạo project Amplify

```bash
npm create amplify@latest smart-document-assistant
cd smart-document-assistant
npm install @aws-amplify/backend @aws-amplify/backend-cli
npm install aws-amplify
```

---

### 2. Amazon Cognito — Xác thực người dùng

Định nghĩa xác thực trong `amplify/auth/resource.ts`:

```typescript
export const auth = defineAuth({
  loginWith: {
    email: true,
  },
  email: {
    fromEmail: 'noreply@yourdomain.com',
    fromName: 'Smart Document Assistant',
  },
});
```

Cognito User Pool xử lý:
- Đăng ký và đăng nhập bằng **Email/Password**
- Xác thực **OTP** khi đăng ký (gửi qua Amazon SES)
- Luồng đầy đủ quên mật khẩu và đặt lại mật khẩu

Sau khi đăng nhập, Cognito cấp **JWT Token** để AppSync thực hiện **User Pool Authorization** trên mọi request GraphQL. Frontend dùng **Amplify Auth SDK** — không cần quản lý token thủ công.

> **Bước sau khi deploy:** Vào SES Console → Verified Identities → xác thực địa chỉ `fromEmail`. Request SES production access để gửi được đến mọi email (sandbox chỉ gửi được đến địa chỉ đã verify).

---

### 3. Amazon S3 — Lưu trữ tài liệu

Định nghĩa storage trong `amplify/storage/resource.ts`:

```typescript
export const storage = defineStorage({
  name: 'smart-doc-storage',
  access: (allow) => ({
    'raw/{entity_id}/*': [
      allow.entity('identity').to(['read', 'write', 'delete']),
    ],
    'processed/*': [
      allow.authenticated.to(['read']),
    ],
  }),
});
```

**Cấu trúc bucket:**

| Prefix | Mục đích |
|---|---|
| `raw/` | Đích upload. File lưu tại `raw/{identityId}/{timestamp}-{filename}` — phân tách dữ liệu theo từng người dùng. |
| `processed/` | Lambda B tạo khi text trích xuất vượt **300 KB** để tránh giới hạn 400 KB item của DynamoDB. |

`S3 NotificationConfiguration` (`s3:ObjectCreated:Put` trên `raw/`) được gắn trực tiếp vào `CfnBucket` trong `amplify/backend.ts` — đây là điểm khởi đầu của pipeline.

---

### 4. AWS AppSync — GraphQL API

Định nghĩa data schema trong `amplify/data/resource.ts`:

```typescript
const schema = a.schema({
  Document: a.model({
    owner:          a.string(),
    fileName:       a.string(),
    fileType:       a.string(),
    fileSize:       a.integer(),
    s3Key:          a.string(),
    status:         a.string(),   // uploaded | processing | text_extracted | done | error
    textractJobId:  a.string(),
    summary:        a.string(),
    category:       a.string(),   // Hợp đồng | Hóa đơn | Báo cáo | Khác
    extractedText:  a.string(),
    processedS3Key: a.string(),
    createdAt:      a.string(),
  }).authorization((allow) => [allow.owner()]),

  UserQuota: a.model({
    uploadedCount: a.integer(),
    maxUploads:    a.integer(),   // mặc định 50
  }).identifier(['owner'])
    .authorization((allow) => [allow.owner()]),
});

export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: 'userPool',
  },
});
```

Cả hai model dùng `allow.owner()` — mỗi người dùng chỉ đọc và ghi được dữ liệu của chính mình.

---

### 5. Chạy sandbox cục bộ

```bash
npx ampx sandbox
```

Lệnh này tạo file `amplify_outputs.json` với các endpoint AWS thật. Frontend đọc file này để kết nối với Cognito, AppSync và S3.

---

### 6. Cấu hình AWS SSM Parameter Store

Lưu OpenRouter API key trước khi deploy Lambda:

1. Mở [AWS SSM Parameter Store](https://console.aws.amazon.com/systems-manager/parameters)
2. Tạo parameter mới:
   - **Name:** `/amplify/{APP_ID}/main/OPENROUTER_API_KEY`
   - **Type:** `SecureString`
   - **Value:** `sk-or-v1-...`
3. Bật quyền truy cập model Bedrock:
   - Mở [Bedrock Console](https://console.aws.amazon.com/bedrock/) → **Model access** → bật **Amazon Nova Lite**
