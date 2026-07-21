---
title: "SNS, Textract Role & DynamoDB Stream"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3.3. </b> "
---

## SNS, Textract Role & DynamoDB Stream

Phần này cấu hình các thành phần còn lại trong pipeline — tập trung vào **luồng async** của Textract và **trigger AI analysis** từ DynamoDB Stream:

1. **SNS Topic** — nhận callback từ Textract khi OCR hoàn thành
2. **IAM Role** — cho phép Textract service publish message vào SNS topic
3. **SNS Subscription** — kết nối SNS topic → Lambda B
4. **DynamoDB Stream** — trigger Lambda B khi user chọn analysis mode

Sau khi hoàn thành, toàn bộ pipeline event-driven sẽ được kết nối đầy đủ.

---

### Tạo SNS Topic

**Bước 1.** Trong file `amplify/backend.ts`, thêm đoạn tạo SNS Topic:

```typescript
const topicName = 'textract-ocr-completed-topic';
const topic = new sns.Topic(backend.stack, 'TextractOcrCompletedTopic', {
  topicName,
});
const topicArn = `arn:aws:sns:${region}:${account}:${topicName}`;
```

**Giải thích:**
- `topicName` dùng string cố định — nhất quán với cách override tên resource ở mục 5.3.2.
- `topicArn` được construct bằng string — KHÔNG dùng `topic.topicArn` (CDK token) ở những chỗ cross-stack.


---

### Tạo Textract IAM Role

**Bước 2.** Tạo IAM Role cho phép Textract service publish kết quả OCR vào SNS topic:

```typescript
const textractSnsRoleName = 'TextractSnsPublishRole';
const textractSnsRole = new iam.Role(backend.stack, 'TextractSnsRole', {
  roleName: textractSnsRoleName,
  assumedBy: new iam.ServicePrincipal('textract.amazonaws.com'),
});
textractSnsRole.addToPolicy(new iam.PolicyStatement({
  actions: ['sns:Publish'],
  resources: [topic.topicArn],
}));

const roleArn = `arn:aws:iam::${account}:role/${textractSnsRoleName}`;
```


**Giải thích:**
- `assumedBy: textract.amazonaws.com` — chỉ Textract service mới được assume role này
- Policy cho phép `sns:Publish` trên đúng SNS topic vừa tạo
- `roleArn` construct bằng string — Lambda A sẽ dùng ARN này khi gọi `StartDocumentTextDetection` (tham số `NotificationChannel.RoleArn`)

{{% notice tip %}}
Đến đây, bạn có thể uncomment các dòng `TEXTRACT_SNS_TOPIC_ARN` và `TEXTRACT_SNS_ROLE_ARN` trong phần environment variables đã tạo ở mục 5.3.2 (nếu trước đó đã comment tạm).
{{% /notice %}}

---

### Cấu hình SNS → Lambda B Subscription

Tương tự như S3 → Lambda A ở mục 5.3.2, ta KHÔNG dùng `LambdaSubscription` (sẽ gây circular dependency). Thay vào đó, tạo thủ công permission + CfnSubscription.

**Bước 3.** Thêm permission cho Lambda B nhận invoke từ SNS:

```typescript
backend.lambdaBTextractResult.resources.lambda.addPermission('AllowSNSInvoke', {
  principal: new iam.ServicePrincipal('sns.amazonaws.com'),
  action: 'lambda:InvokeFunction',
  sourceArn: topicArn,
});
```

**Bước 4.** Lấy CfnPermission resource và tạo SNS subscription:

```typescript
const snsInvokePermission = backend.lambdaBTextractResult.resources.lambda.node
  .findChild('AllowSNSInvoke') as cdk.CfnResource;

const lambdaBSubscription = new sns.CfnSubscription(backend.stack, 'LambdaBSnsSubscription', {
  topicArn: topic.topicArn,
  protocol: 'lambda',
  endpoint: lambdaBArn,
});
lambdaBSubscription.addDependency(snsInvokePermission);
```


{{% notice warning %}}
`lambdaBSubscription.addDependency(snsInvokePermission)` — bắt buộc phải có. Lý do tương tự mục 5.3.2: CloudFormation cần tạo Lambda permission TRƯỚC rồi mới tạo SNS subscription, nếu không SNS không validate được endpoint.
{{% /notice %}}

---

### Cấu hình DynamoDB Stream → Lambda B

DynamoDB Stream là cơ chế trigger Lambda B khi user chọn analysis mode trên frontend. Frontend update DynamoDB (đặt `status='processing'` và `analysisMode='...'`), stream event sẽ tự động invoke Lambda B.

**Bước 5.** Thêm DynamoDB Stream permissions cho Lambda B:

```typescript
backend.lambdaBTextractResult.resources.lambda.addToRolePolicy(new iam.PolicyStatement({
  actions: [
    'dynamodb:GetRecords',
    'dynamodb:GetShardIterator',
    'dynamodb:DescribeStream',
    'dynamodb:ListStreams',
  ],
  resources: [
    `${documentTable.tableArn}/stream/*`,
  ],
}));
```

**Bước 6.** Thêm DynamoDB Event Source cho Lambda B với filter criteria:

```typescript
backend.lambdaBTextractResult.resources.lambda.addEventSource(
  new lambdaEventSources.DynamoEventSource(documentTable as any, {
    startingPosition: lambda.StartingPosition.LATEST,
    batchSize: 1,
    bisectBatchOnError: true,
    retryAttempts: 2,
    filters: [
      lambda.FilterCriteria.filter({
        eventName: lambda.FilterRule.isEqual('MODIFY'),
        dynamodb: {
          NewImage: {
            status: { S: lambda.FilterRule.isEqual('processing') },
            analysisMode: { S: lambda.FilterRule.exists() },
          },
        },
      }),
    ],
  })
);
```


**Giải thích các tham số:**

| Tham số | Giá trị | Ý nghĩa |
|---|---|---|
| `startingPosition` | `LATEST` | Chỉ xử lý event mới, bỏ qua event cũ |
| `batchSize` | `1` | Xử lý từng record một — đảm bảo mỗi document được phân tích riêng biệt |
| `bisectBatchOnError` | `true` | Nếu batch fail, chia đôi để retry — giảm ảnh hưởng |
| `retryAttempts` | `2` | Retry tối đa 2 lần trước khi bỏ qua |

**Giải thích filter criteria:**

| Filter | Điều kiện | Tại sao cần |
|---|---|---|
| `eventName = MODIFY` | Chỉ trigger khi record bị UPDATE | Bỏ qua INSERT (upload lần đầu) và DELETE |
| `status.S = 'processing'` | Chỉ khi status chuyển thành `processing` | Tránh trigger khi status chuyển thành `text_extracted`, `done`, `error` |
| `analysisMode.S exists` | Chỉ khi field `analysisMode` tồn tại | Đảm bảo user đã chọn mode phân tích — tránh trigger khi Lambda A update status |

{{% notice note %}}
Ba điều kiện filter kết hợp đảm bảo Lambda B (DDB Stream path) CHỈ được trigger khi user chọn analysis mode trên frontend — đúng thời điểm cần chạy AI. Các update khác (từ Lambda A cập nhật status, Lambda B cập nhật kết quả) sẽ KHÔNG trigger lại.
{{% /notice %}}

---

### Tổng hợp file backend.ts

Sau khi hoàn thành cả mục 5.3.2 và 5.3.3, file `backend.ts` sẽ bao gồm các phần chính sau (theo thứ tự):

```
1. Import modules (cdk, iam, sns, lambda, lambdaEventSources)
2. defineBackend({ auth, data, storage, lambdaATrigger, lambdaBTextractResult })
3. Override tên Lambda A, Lambda B → construct ARNs
4. Override tên S3 bucket
5. S3 → Lambda A: permission + notification
6. SNS Topic + Textract IAM Role
7. SNS → Lambda B: permission + subscription
8. Environment variables (Lambda A & B)
9. IAM policies (Textract, Bedrock, DynamoDB, S3)
10. DynamoDB Stream → Lambda B (permissions + event source + filter)
```

![PLACEHOLDER-5.3-02](/images/5-Workshop/5.3-Upload-pipeline/5.3-02-backend-ts-full-overview.png)
*[ẢNH CẦN CHỤP 5.3-02]: Toàn bộ file `backend.ts` trong IDE (collapsed view hoặc minimap), cho thấy tổng quan cấu trúc file ~250 dòng*

---

### Kết quả mong đợi

Sau khi hoàn thành bước này:

- ✅ SNS Topic `textract-ocr-completed-topic` được tạo
- ✅ IAM Role `TextractSnsPublishRole` cho Textract → SNS
- ✅ SNS subscription kết nối topic → Lambda B
- ✅ DynamoDB Stream trigger Lambda B với filter chính xác (MODIFY + processing + analysisMode exists)
- ✅ File `backend.ts` hoàn chỉnh — tất cả resource được kết nối

{{% notice tip %}}
Giờ bạn có thể chạy `npx ampx sandbox` (nếu đang develop) hoặc push code lên GitHub để Amplify CI/CD tự động deploy. Xem mục 5.3.4 để kiểm tra pipeline hoạt động đúng.
{{% /notice %}}
