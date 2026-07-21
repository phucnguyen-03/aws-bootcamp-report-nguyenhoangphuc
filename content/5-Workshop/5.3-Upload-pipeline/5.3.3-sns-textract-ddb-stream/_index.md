---
title: "SNS, Textract Role & DynamoDB Stream"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3.3. </b> "
---

## SNS, Textract Role & DynamoDB Stream

This section configures the asynchronous components of the document pipeline:

1. **SNS Topic** — Receives job completion callbacks from AWS Textract.
2. **IAM Role** — Permits AWS Textract to publish messages to the SNS topic.
3. **SNS Subscription** — Routes topic updates to Lambda B.
4. **DynamoDB Stream** — Invokes Lambda B to run AI analysis when users choose their preferred processing mode.

Connecting these components completes the serverless event-driven flow.

---

### Create SNS Topic

**Step 1.** Add the SNS topic construction to `amplify/backend.ts`:

```typescript
const topicName = 'textract-ocr-completed-topic';
const topic = new sns.Topic(backend.stack, 'TextractOcrCompletedTopic', {
  topicName,
});
const topicArn = `arn:aws:sns:${region}:${account}:${topicName}`;
```

**Key Points:**
- `topicName` matches our static naming rule for cross-stack decoupling.
- `topicArn` is built using string parameters, bypassing cross-stack references.

---

### Create Textract IAM Role

**Step 2.** Create the IAM Role that allows AWS Textract to write messages to the SNS topic:

```typescript
const textractSnsRoleName = 'TextractSnsPublishRole';
const textractSnsRole = new iam.Role(backend.stack, 'TextractSnsRole', {
  roleName: textractSnsRoleName,
  assumedBy: new iam.ServicePrincipal('textract.amazonaws.com'),
});
textractSnsRole.addToRolePolicy(new iam.PolicyStatement({
  actions: ['sns:Publish'],
  resources: [topic.topicArn],
}));

const roleArn = `arn:aws:iam::${account}:role/${textractSnsRoleName}`;
```

**Key Points:**
- `assumedBy` limits roles to the `textract.amazonaws.com` service principal.
- `roleArn` is passed as a string configuration to Lambda A, which uses it when starting text detection commands.

---

### Route SNS to Lambda B (Subscription)

We avoid dynamic subscription bindings (like `LambdaSubscription`) to prevent circular dependencies. Instead, configure it manually.

**Step 3.** Allow SNS to invoke Lambda B:

```typescript
backend.lambdaBTextractResult.resources.lambda.addPermission('AllowSNSInvoke', {
  principal: new iam.ServicePrincipal('sns.amazonaws.com'),
  action: 'lambda:InvokeFunction',
  sourceArn: topicArn,
});
```

**Step 4.** Bind the subscription to the topic:

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
`lambdaBSubscription.addDependency(snsInvokePermission)` ensures CloudFormation configures the Lambda permissions *before* activating the subscription. Without this, AWS will fail to validate the endpoint.
{{% /notice %}}

---

### Configure DynamoDB Stream → Lambda B

A DynamoDB Stream triggers Lambda B for AI analysis when the user selects a processing mode on the frontend. The frontend sets `status='processing'` and `analysisMode='...'` on the document record, sending an event through the stream.

**Step 5.** Grant Stream reading rights to Lambda B:

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

**Step 6.** Map the DynamoDB Stream event source with filters:

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

**Stream Configuration Parameters:**

| Parameter | Value | Purpose |
|---|---|---|
| `startingPosition` | `LATEST` | Evaluates new stream items, skipping older entries. |
| `batchSize` | `1` | Processes one document at a time for simpler failure isolation. |
| `bisectBatchOnError` | `true` | Splits failing batches in half during retries. |
| `retryAttempts` | `2` | Discards failing records after 2 retry attempts. |

**Stream Event Filters:**

| Target Pattern | Constraint | Reason |
|---|---|---|
| `eventName` | `MODIFY` | Filters out document creation (`INSERT`) and deletion (`REMOVE`) events. |
| `NewImage.status.S` | `processing` | Ignores database updates that do not change the status to `processing`. |
| `NewImage.analysisMode.S` | `exists()` | Ensures the user has chosen an AI analysis mode before starting the job. |

{{% notice note %}}
Combining these filters ensures Lambda B is only triggered for AI analysis when the user changes the mode. It prevents loops when Lambda A updates statuses or Lambda B saves the final output.
{{% /notice %}}

---

### Backend file structure overview

Once sections 5.3.2 and 5.3.3 are merged, your `backend.ts` file will follow this structure:

```
1. Imports (cdk, iam, sns, lambda, lambdaEventSources)
2. defineBackend({ auth, data, storage, lambdaATrigger, lambdaBTextractResult })
3. Override Lambda A & B names and build ARNs
4. Override S3 Bucket Name
5. Map S3-to-Lambda A Permissions & Notifications
6. Create SNS Topic & Textract IAM Role
7. Subscribe Lambda B to the SNS Topic
8. Map Environment Variables (Lambda A & B)
9. Map IAM Policy Statements (Textract, Bedrock, DynamoDB, S3)
10. Set up DynamoDB Stream triggers for Lambda B
```

![PLACEHOLDER-5.3-02](/images/5-Workshop/5.3-Upload-pipeline/5.3-02-backend-ts-full-overview.png)
*[IMAGE TO CAPTURE 5.3-02]: High-level overview of the backend.ts configuration inside the IDE editor panel (~250 lines)*

---

### Expected Results

By the end of this sub-section:
- ✅ SNS Topic `textract-ocr-completed-topic` is initialized.
- ✅ Textract IAM role `TextractSnsPublishRole` is configured.
- ✅ Lambda B is subscribed to SNS callbacks.
- ✅ DynamoDB Stream triggers Lambda B using strict modification filters.

{{% notice tip %}}
Run `npx ampx sandbox` or commit code to trigger Amplify Hosting's automatic deployment. Next, check [5.3.4](../5.3.4-verify-pipeline/) to verify the pipeline.
{{% /notice %}}
