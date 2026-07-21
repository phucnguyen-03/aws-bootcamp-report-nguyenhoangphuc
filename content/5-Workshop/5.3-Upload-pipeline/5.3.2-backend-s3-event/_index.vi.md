---
title: "Cấu hình Backend — Override Names & S3 Event"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.3.2. </b> "
---

## Cấu hình Backend — Override Names & S3 Event

Trong phần này, ta sẽ cấu hình file `amplify/backend.ts` để kết nối các resource đã khai báo ở mục 5.3.1. Công việc chính gồm:

1. **Override tên Lambda & S3 bucket** thành chuỗi cố định — tránh circular dependency giữa các nested stack của CDK
2. **Cấu hình S3 Event Notification** — tự động trigger Lambda A khi có file mới upload vào prefix `raw/`
3. **Truyền environment variables** cho Lambda A & B

{{% notice warning %}}
**Vấn đề Circular Dependency trong Amplify Gen 2:** Amplify Gen 2 tạo nested CloudFormation stack riêng cho mỗi resource (function, storage, data). Nếu bạn dùng CDK token (ví dụ `lambda.functionArn`, `bucket.bucketArn`) để tham chiếu giữa các stack, CDK sẽ tạo cross-stack import/export → dẫn đến **circular dependency** và deploy sẽ fail. Giải pháp là override tên resource thành string cố định, rồi tự construct ARN bằng pseudo-params (`region`, `account`).
{{% /notice %}}

---

### Override tên Lambda Functions

**Bước 1.** Trong file `amplify/backend.ts`, thêm đoạn code sau ngay sau `defineBackend(...)`:

```typescript
// Override function names
const lambdaAFunctionName = 'smart-doc-upload-trigger';
const lambdaBFunctionName = 'smart-doc-textract-result';

const cfnLambdaA = backend.lambdaATrigger.resources.lambda.node.defaultChild as cdk.CfnResource;
cfnLambdaA.addPropertyOverride('FunctionName', lambdaAFunctionName);

const cfnLambdaB = backend.lambdaBTextractResult.resources.lambda.node.defaultChild as cdk.CfnResource;
cfnLambdaB.addPropertyOverride('FunctionName', lambdaBFunctionName);

// Construct ARNs from pseudo-params
const region = backend.stack.region;
const account = backend.stack.account;
const lambdaAArn = `arn:aws:lambda:${region}:${account}:function:${lambdaAFunctionName}`;
const lambdaBArn = `arn:aws:lambda:${region}:${account}:function:${lambdaBFunctionName}`;
```

**Giải thích:**
- `node.defaultChild as cdk.CfnResource` — truy cập L1 CloudFormation resource bên dưới L2 construct của Amplify
- `addPropertyOverride('FunctionName', ...)` — ghi đè property `FunctionName` trong CloudFormation template
- `backend.stack.region` và `backend.stack.account` — pseudo-params của CDK, resolve lúc deploy → KHÔNG tạo cross-stack dependency

---

### Override tên S3 Bucket

**Bước 2.** Tiếp tục trong `backend.ts`, override tên bucket:

```typescript
// Stable bucket name
const storageBucketName = `smart-doc-storage-${account}-${region}`;
const cfnBucketName = bucket.node.defaultChild as cdk.CfnResource;
cfnBucketName.addPropertyOverride('BucketName', storageBucketName);
```

{{% notice note %}}
Tên bucket bao gồm `account` và `region` để đảm bảo tính duy nhất toàn cầu (S3 bucket names phải unique trên toàn AWS). Ví dụ thực tế: `smart-doc-storage-094337892396-ap-southeast-1`.
{{% /notice %}}

---

### Cấu hình S3 → Lambda A Event Notification

Đây là bước quan trọng nhất — kết nối S3 bucket với Lambda A để mỗi khi có file upload vào prefix `raw/`, Lambda A tự động được invoke.

**Bước 3.** Thêm permission cho Lambda A nhận invoke từ S3:

```typescript
// S3 → Lambda A permission + notification
backend.lambdaATrigger.resources.lambda.addPermission('AllowS3Invoke', {
  principal: new iam.ServicePrincipal('s3.amazonaws.com'),
  action: 'lambda:InvokeFunction',
  sourceAccount: account,
});
```

**Bước 4.** Lấy CfnPermission resource và thiết lập dependency order:

```typescript
const s3InvokePermission = backend.lambdaATrigger.resources.lambda.node
  .findChild('AllowS3Invoke') as cdk.CfnResource;

const cfnBucket = bucket.node.defaultChild as cdk.CfnResource;
cfnBucket.addDependency(s3InvokePermission);
```

{{% notice warning %}}
`cfnBucket.addDependency(s3InvokePermission)` là **bắt buộc**. Nếu không có dòng này, CloudFormation có thể tạo/update bucket notification TRƯỚC khi Lambda permission tồn tại → S3 validate fail với lỗi: **"Unable to validate the following destination configurations"** → stack rollback.
{{% /notice %}}

**Bước 5.** Override NotificationConfiguration trên bucket:

```typescript
cfnBucket.addPropertyOverride('NotificationConfiguration', {
  LambdaConfigurations: [
    {
      Event: 's3:ObjectCreated:Put',
      Filter: {
        S3Key: { Rules: [{ Name: 'prefix', Value: 'raw/' }] },
      },
      Function: lambdaAArn,
    },
  ],
});
```


**Giải thích cấu hình notification:**

| Property | Giá trị | Ý nghĩa |
|---|---|---|
| `Event` | `s3:ObjectCreated:Put` | Chỉ trigger khi có object mới được PUT (upload) |
| `Filter.S3Key.Rules` | `prefix: 'raw/'` | Chỉ trigger cho file trong thư mục `raw/` — tránh trigger lại khi Lambda ghi file vào `processed/` |
| `Function` | `lambdaAArn` (constructed string) | ARN của Lambda A, dùng string tĩnh thay vì CDK token |

{{% notice tip %}}
**Tại sao KHÔNG dùng `LambdaDestination`?** CDK cung cấp API `bucket.addEventNotification(LambdaDestination(...))` tiện hơn, nhưng nó gọi `lambda.addPermission` bên trong với `bucket.bucketArn` (CDK token) → tạo cross-stack dependency → circular dependency. Ta phải tự tay tạo permission + notification bằng CfnResource override.
{{% /notice %}}

---

### Truyền Environment Variables

**Bước 6.** Thêm environment variables cho cả hai Lambda:

```typescript
// Lambda A environment variables
backend.lambdaATrigger.resources.lambda.addEnvironment('TEXTRACT_SNS_TOPIC_ARN', topicArn);
backend.lambdaATrigger.resources.lambda.addEnvironment('TEXTRACT_SNS_ROLE_ARN', roleArn);
backend.lambdaATrigger.resources.lambda.addEnvironment('DOCUMENT_TABLE_NAME', documentTable.tableName);
backend.lambdaATrigger.resources.lambda.addEnvironment('USER_QUOTA_TABLE_NAME', userQuotaTable.tableName);
backend.lambdaATrigger.resources.lambda.addEnvironment('STORAGE_BUCKET_NAME', storageBucketName);

// Lambda B environment variables
backend.lambdaBTextractResult.resources.lambda.addEnvironment('DOCUMENT_TABLE_NAME', documentTable.tableName);
backend.lambdaBTextractResult.resources.lambda.addEnvironment('USER_QUOTA_TABLE_NAME', userQuotaTable.tableName);
backend.lambdaBTextractResult.resources.lambda.addEnvironment('STORAGE_BUCKET_NAME', storageBucketName);
```

**Giải thích:**

| Variable | Lambda | Giá trị | Ghi chú |
|---|---|---|---|
| `TEXTRACT_SNS_TOPIC_ARN` | A | Constructed string | ARN của SNS topic cho Textract callback |
| `TEXTRACT_SNS_ROLE_ARN` | A | Constructed string | ARN của IAM role cho Textract publish SNS |
| `DOCUMENT_TABLE_NAME` | A & B | Từ data stack | Sibling dependency OK (data ↔ function) |
| `USER_QUOTA_TABLE_NAME` | A & B | Từ data stack | Sibling dependency OK |
| `STORAGE_BUCKET_NAME` | A & B | Plain string override | Dùng string tĩnh, KHÔNG dùng `bucket.bucketName` |
| `OPENROUTER_API_KEY` | A & B | Từ Amplify secret store | Đã khai báo trong `resource.ts` ở bước 5.3.1 |

{{% notice note %}}
`DOCUMENT_TABLE_NAME` và `USER_QUOTA_TABLE_NAME` lấy từ data stack bằng `documentTable.tableName`. Đây là sibling dependency (cùng level, không cross-stack) nên CDK xử lý OK, không gây circular dependency.

Biến `topicArn` và `roleArn` sẽ được tạo ở phần 5.3.3 (SNS Topic & Textract Role). Nếu bạn đang code theo thứ tự, hãy tạm comment các dòng liên quan đến `topicArn` và `roleArn`, rồi uncomment sau khi hoàn thành mục 5.3.3.
{{% /notice %}}

---

### Cấu hình IAM Policies cho Lambda

**Bước 7.** Thêm IAM policies cho cả hai Lambda function:

**Textract permissions:**

```typescript
// Lambda A: start Textract job
backend.lambdaATrigger.resources.lambda.addToRolePolicy(new iam.PolicyStatement({
  actions: ['textract:StartDocumentTextDetection'],
  resources: ['*'],
}));

// Lambda B: get Textract result
backend.lambdaBTextractResult.resources.lambda.addToRolePolicy(new iam.PolicyStatement({
  actions: ['textract:GetDocumentTextDetection'],
  resources: ['*'],
}));
```

**Bedrock permissions (cả hai Lambda):**

```typescript
const bedrockPolicy = new iam.PolicyStatement({
  actions: ['bedrock:InvokeModel'],
  resources: [
    'arn:aws:bedrock:*::foundation-model/amazon.nova-lite-v1:0',
    'arn:aws:bedrock:*::foundation-model/anthropic.claude-3-haiku-20240307-v1:0',
    `arn:aws:bedrock:${region}:${account}:inference-profile/apac.amazon.nova-lite-v1:0`,
  ],
});
backend.lambdaATrigger.resources.lambda.addToRolePolicy(bedrockPolicy);
backend.lambdaBTextractResult.resources.lambda.addToRolePolicy(bedrockPolicy);
```

{{% notice note %}}
**Tại sao cần cả `foundation-model` lẫn `inference-profile`?** Amazon Nova Lite ở region `ap-southeast-1` chỉ invoke được qua cross-region inference profile (`apac.amazon.nova-lite-v1:0`). IAM policy cần cấp quyền trên CẢ foundation-model ARN (wildcard region) LẪN inference-profile ARN (cụ thể region + account).
{{% /notice %}}

**DynamoDB permissions (cả hai Lambda):**

```typescript
const dynamoDbPolicy = new iam.PolicyStatement({
  actions: [
    'dynamodb:GetItem',
    'dynamodb:PutItem',
    'dynamodb:UpdateItem',
    'dynamodb:DeleteItem',
    'dynamodb:Query',
    'dynamodb:Scan',
    'dynamodb:BatchGetItem',
    'dynamodb:BatchWriteItem',
    'dynamodb:DescribeTable',
  ],
  resources: [
    documentTable.tableArn,
    `${documentTable.tableArn}/index/*`,
    userQuotaTable.tableArn,
    `${userQuotaTable.tableArn}/index/*`,
  ],
});
backend.lambdaATrigger.resources.lambda.addToRolePolicy(dynamoDbPolicy);
backend.lambdaBTextractResult.resources.lambda.addToRolePolicy(dynamoDbPolicy);
```

**S3 permissions (cả hai Lambda):**

```typescript
const s3Policy = new iam.PolicyStatement({
  actions: ['s3:GetObject', 's3:PutObject', 's3:DeleteObject', 's3:ListBucket'],
  resources: ['arn:aws:s3:::*', 'arn:aws:s3:::*/*'],
});
backend.lambdaATrigger.resources.lambda.addToRolePolicy(s3Policy);
backend.lambdaBTextractResult.resources.lambda.addToRolePolicy(s3Policy);
```

{{% notice note %}}
S3 policy dùng wildcard `arn:aws:s3:::*` thay vì bucket ARN cụ thể — cũng để tránh circular dependency khi tham chiếu giữa function stack và storage stack.
{{% /notice %}}


---

### Kết quả mong đợi

Sau khi hoàn thành bước này, file `backend.ts` đã có:

- ✅ Override tên Lambda A, Lambda B, S3 bucket thành string cố định
- ✅ Constructed ARN cho cả 3 resource bằng pseudo-params
- ✅ S3 Event Notification trigger Lambda A khi upload file vào `raw/`
- ✅ Lambda permission cho S3 invoke + dependency order đúng
- ✅ Environment variables cho cả hai Lambda
- ✅ IAM policies: Textract, Bedrock, DynamoDB, S3

{{% notice tip %}}
Chưa deploy ở bước này — còn thiếu SNS Topic, Textract IAM Role và DynamoDB Stream config. Tiếp tục sang mục 5.3.3 để hoàn thành.
{{% /notice %}}
