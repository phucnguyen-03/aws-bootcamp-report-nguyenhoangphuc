---
title: "Backend Configuration — Override Names & S3 Event"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.3.2. </b> "
---

## Backend Configuration — Override Names & S3 Event

In this section, we will customize the `amplify/backend.ts` file to stitch our declared resources together. Key configurations include:

1. **Overriding Lambda and S3 bucket names** using static strings — this prevents CDK cross-stack dependencies from causing circular errors.
2. **Configuring S3 Event Notifications** — to invoke Lambda A automatically whenever a file is uploaded inside the `raw/` directory.
3. **Injecting environment variables and IAM policies** into Lambda A & B.

{{% notice warning %}}
**CDK Circular Dependency Issue in Amplify Gen 2:** Amplify Gen 2 packages resources into nested CloudFormation stacks. Referencing dynamic CDK tokens (like `lambda.functionArn` or `bucket.bucketArn`) across these stacks generates cross-stack imports/exports that frequently cause circular dependency build failures. Our solution is to override the logical names with static strings and construct ARNs using CDK pseudo-parameters (`region` and `account`).
{{% /notice %}}

---

### Override Lambda Function Names

**Step 1.** Open `amplify/backend.ts` and add name overrides immediately below the `defineBackend(...)` construct:

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

**What is happening:**
- `node.defaultChild as cdk.CfnResource` — Retrieves the underlying L1 CloudFormation resource from Amplify's L2 wrappers.
- `addPropertyOverride('FunctionName', ...)` — Hardcodes the `FunctionName` within the synthesized template.
- `backend.stack.region` & `account` — Resolves to standard CDK environment tokens during deployment without creating stack-level imports.

---

### Override S3 Bucket Name

**Step 2.** Append the bucket name override to ensure a clean name pattern:

```typescript
// Stable bucket name
const storageBucketName = `smart-doc-storage-${account}-${region}`;
const cfnBucketName = bucket.node.defaultChild as cdk.CfnResource;
cfnBucketName.addPropertyOverride('BucketName', storageBucketName);
```

{{% notice note %}}
We append AWS account and region identifiers to make the S3 bucket globally unique. For example: `smart-doc-storage-094337892396-ap-southeast-1`.
{{% /notice %}}

---

### Configure S3 → Lambda A Event Notifications

Next, we route object uploads from our S3 bucket to trigger Lambda A.

**Step 3.** Grant S3 invocation permissions to Lambda A:

```typescript
// S3 → Lambda A permission + notification
backend.lambdaATrigger.resources.lambda.addPermission('AllowS3Invoke', {
  principal: new iam.ServicePrincipal('s3.amazonaws.com'),
  action: 'lambda:InvokeFunction',
  sourceAccount: account,
});
```

**Step 4.** Fetch the permission child resource and configure build order dependencies:

```typescript
const s3InvokePermission = backend.lambdaATrigger.resources.lambda.node
  .findChild('AllowS3Invoke') as cdk.CfnResource;

const cfnBucket = bucket.node.defaultChild as cdk.CfnResource;
cfnBucket.addDependency(s3InvokePermission);
```

{{% notice warning %}}
`cfnBucket.addDependency(s3InvokePermission)` is **mandatory**. Otherwise, CloudFormation might create the S3 bucket notification config before the Lambda permission exists, causing the S3 service validation check to fail with **"Unable to validate the following destination configurations"** and rolling back the deployment.
{{% /notice %}}

**Step 5.** Override the S3 bucket's `NotificationConfiguration`:

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

**Notification Configuration Parameters:**

| Property | Value | Purpose |
|---|---|---|
| `Event` | `s3:ObjectCreated:Put` | Triggers exclusively when objects are newly created or replaced. |
| `Filter.S3Key.Rules` | `prefix: 'raw/'` | Fires only for files uploaded to `raw/`, avoiding recursive loops when Lambda writes output back to `processed/`. |
| `Function` | `lambdaAArn` (constructed string) | Hardcoded static Lambda ARN bypassing cross-stack references. |

{{% notice tip %}}
**Why avoid CDK's `LambdaDestination` helper?** High-level methods like `bucket.addEventNotification(LambdaDestination(...))` automatically inject the bucket's dynamic ARN into the Lambda, triggering cross-stack dependency errors during build compilation. Manual property override overrides this issue.
{{% /notice %}}

---

### Inject Environment Variables

**Step 6.** Inject configuration variables to both Lambda functions:

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

{{% notice note %}}
`documentTable.tableName` references tables within the sibling Data stack. Because these belong to the same level, it does not trigger circular dependency errors. 

If you are typing this sequentially, temporarily comment out references to `topicArn` and `roleArn` and uncomment them after setting them up in sub-section 5.3.3.
{{% /notice %}}

---

### Apply IAM Policies

**Step 7.** Define and append IAM policy statements to give functions their required access rights:

**Textract Permissions:**

```typescript
// Lambda A starts text detection
backend.lambdaATrigger.resources.lambda.addToRolePolicy(new iam.PolicyStatement({
  actions: ['textract:StartDocumentTextDetection'],
  resources: ['*'],
}));

// Lambda B retrieves text results
backend.lambdaBTextractResult.resources.lambda.addToRolePolicy(new iam.PolicyStatement({
  actions: ['textract:GetDocumentTextDetection'],
  resources: ['*'],
}));
```

**Bedrock AI Permissions (Both Lambda functions):**

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
**Cross-Region Inference Profile Role:** Amazon Nova Lite in `ap-southeast-1` is accessed using a cross-region inference profile. Therefore, the IAM policies must authorize both the raw foundation model resource ID and the local region's inference-profile ARN.
{{% /notice %}}

**DynamoDB Permissions (Both Lambda functions):**

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

**S3 Permissions (Both Lambda functions):**

```typescript
const s3Policy = new iam.PolicyStatement({
  actions: ['s3:GetObject', 's3:PutObject', 's3:DeleteObject', 's3:ListBucket'],
  resources: ['arn:aws:s3:::*', 'arn:aws:s3:::*/*'],
});
backend.lambdaATrigger.resources.lambda.addToRolePolicy(s3Policy);
backend.lambdaBTextractResult.resources.lambda.addToRolePolicy(s3Policy);
```

---

### Expected Results

You have now updated `backend.ts` to include:
- ✅ Logical overrides for Lambda names and bucket configurations.
- ✅ Custom S3-to-Lambda permission bindings with correct ordering dependencies.
- ✅ Lambda environment variables mapping data table names.
- ✅ Fine-grained IAM policy statements for S3, DynamoDB, Bedrock, and Textract.

{{% notice tip %}}
Leave your sandbox running. We will declare SNS resources and streams in 5.3.3 before executing the deploy build.
{{% /notice %}}
