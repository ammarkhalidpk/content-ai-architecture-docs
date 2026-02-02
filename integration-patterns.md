# Content AI Platform - Integration Patterns

**Last Updated:** February 3, 2026

This document describes the integration patterns, communication protocols, error handling strategies, and best practices for the Content AI Platform.

---

## Table of Contents

1. [Service Communication Patterns](#1-service-communication-patterns)
2. [Event-Driven Architecture](#2-event-driven-architecture)
3. [Error Handling and Retry Logic](#3-error-handling-and-retry-logic)
4. [Authentication and Authorization](#4-authentication-and-authorization)
5. [Data Consistency Patterns](#5-data-consistency-patterns)
6. [Performance Optimization](#6-performance-optimization)
7. [Testing Integration Points](#7-testing-integration-points)
8. [Troubleshooting Guide](#8-troubleshooting-guide)

---

## 1. Service Communication Patterns

### 1.1 Synchronous Communication

#### Pattern: API Gateway → Lambda (Request-Response)

**Use Case:** Frontend-initiated API calls requiring immediate response

```typescript
// Client (React)
const response = await fetch(`${API_URL}/api/v1/jobs`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${jwt}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ jobName: 'My Job' })
});

// API Gateway validates JWT and routes to Lambda

// Lambda (RequestUploadApi)
export const handler = async (event: APIGatewayProxyEvent) => {
  const body = JSON.parse(event.body);
  const jobId = uuidv4();

  // Create job in DynamoDB
  await dynamoDB.put({
    TableName: JOBS_TABLE,
    Item: {
      PK: `JOB#${jobId}`,
      type: 'JOB',
      userId: event.requestContext.authorizer.claims.email,
      jobName: body.jobName,
      status: 'CREATED',
      createdAt: new Date().toISOString()
    }
  }).promise();

  // Generate presigned URL for upload
  const presignedUrl = await s3.getSignedUrlPromise('putObject', {
    Bucket: BUCKET_NAME,
    Key: `jobs/${jobId}/input.zip`,
    Expires: 900 // 15 minutes
  });

  return {
    statusCode: 201,
    body: JSON.stringify({ jobId, uploadUrl: presignedUrl })
  };
};
```

**Characteristics:**
- **Latency:** Low (< 1 second)
- **Coupling:** Tight (client waits for response)
- **Error Handling:** Immediate feedback to client
- **Use Cases:** CRUD operations, status queries, presigned URL generation

---

### 1.2 Asynchronous Communication via SNS/SQS

#### Pattern: Publisher → SNS Topic → SQS Queue → Consumer Lambda

**Use Case:** Service completion notifications (CIP/VPE → Self-Service)

```typescript
// Publisher: CIP Service completes job
await sns.publish({
  TopicArn: CIP_POST_PROCESSING_SNS_ARN,
  Message: JSON.stringify({
    jobName: jobId,
    presignedURL: downloadUrl,
    service: ['SEARCHABLE_PDF', 'PII_REDACTION'],
    contentType: 'application/zip'
  }),
  MessageAttributes: {
    notificationType: {
      DataType: 'String',
      StringValue: 'JOB_COMPLETED'
    }
  }
}).promise();

// SNS Topic has subscription filter
// Subscription Filter Policy:
{
  "notificationType": ["JOB_COMPLETED"]
}

// SQS Queue receives filtered messages
// Queue configuration:
{
  "VisibilityTimeout": 180,  // 3 minutes
  "MessageRetentionPeriod": 345600,  // 4 days
  "ReceiveMessageWaitTimeSeconds": 20,  // Long polling
  "RedrivePolicy": {
    "deadLetterTargetArn": DLQ_ARN,
    "maxReceiveCount": 3
  }
}

// Consumer: ProcessCIPOutputEvent Lambda
export const handler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    const snsMessage = JSON.parse(record.body);
    const cipNotification = JSON.parse(snsMessage.Message);

    // Download results from CIP
    const response = await axios.get(cipNotification.presignedURL);

    // Extract and store results
    const results = await processZip(response.data);

    // Update job status in DynamoDB
    await dynamoDB.update({
      TableName: JOBS_TABLE,
      Key: { PK: `JOB#${cipNotification.jobName}` },
      UpdateExpression: 'SET #status = :status, cipResults = :results',
      ExpressionAttributeNames: { '#status': 'status' },
      ExpressionAttributeValues: {
        ':status': 'CIP_COMPLETED',
        ':results': results
      }
    }).promise();

    // Resume Step Functions
    await stepFunctions.sendTaskSuccess({
      taskToken: taskToken,
      output: JSON.stringify(results)
    }).promise();
  }
};
```

**Characteristics:**
- **Latency:** Medium (seconds to minutes)
- **Coupling:** Loose (publisher doesn't wait for consumer)
- **Error Handling:** Retry with backoff, DLQ for failed messages
- **Use Cases:** Job completion notifications, asynchronous processing

**Benefits:**
- Decoupling: Publisher and consumer are independent
- Scalability: SQS buffers messages, consumers scale independently
- Reliability: Messages persisted, automatic retry
- Filtering: SNS subscription filters reduce unnecessary processing

---

### 1.3 Step Functions Orchestration

#### Pattern: Callback Pattern for Long-Running Tasks

**Use Case:** Orchestration waits for external service completion

```typescript
// Step Functions Definition
{
  "StartAt": "StartCIPTask",
  "States": {
    "StartCIPTask": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Parameters": {
        "FunctionName": "StartCipTask",
        "Payload": {
          "taskToken.$": "$$.Task.Token",
          "input.$": "$"
        }
      },
      "Next": "ProcessCIPOutput",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "HandleError"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 30,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ]
    }
  }
}

// Lambda: StartCipTask
export const handler = async (event: TaskEvent) => {
  const { taskToken, input } = event;

  // Store task token in DynamoDB for later retrieval
  await dynamoDB.put({
    TableName: TASKS_TABLE,
    Item: {
      PK: `TASK#${taskToken}`,
      jobId: input.jobId,
      taskToken,
      status: 'RUNNING',
      createdAt: new Date().toISOString()
    }
  }).promise();

  // Call CIP service
  await cipApi.post('/upload', {
    jobName: input.jobId,
    service: input.cipServices,
    presignedUrl: await generatePresignedUrl(input.jobId)
  });

  // Don't return - Step Functions waits for callback
  // Lambda will be invoked again by ProcessCIPOutputEvent
  // which calls stepFunctions.sendTaskSuccess(taskToken)
};

// Lambda: ProcessCIPOutputEvent (triggered by SQS)
export const handler = async (event: SQSEvent) => {
  const cipNotification = parseCIPNotification(event);

  // Retrieve task token from DynamoDB
  const task = await dynamoDB.get({
    TableName: TASKS_TABLE,
    Key: { PK: `TASK#${cipNotification.jobName}` }
  }).promise();

  // Download and process results
  const results = await downloadResults(cipNotification.presignedURL);

  // Resume Step Functions
  await stepFunctions.sendTaskSuccess({
    taskToken: task.Item.taskToken,
    output: JSON.stringify(results)
  }).promise();

  // Clean up task record
  await dynamoDB.delete({
    TableName: TASKS_TABLE,
    Key: { PK: `TASK#${task.Item.PK}` }
  }).promise();
};
```

**Characteristics:**
- **Latency:** High (minutes to hours)
- **Coupling:** Medium (orchestrator waits but doesn't block)
- **Error Handling:** Timeout, retry, error states
- **Use Cases:** Multi-step workflows, long-running processes

**Benefits:**
- Visual workflow: Step Functions provides visual representation
- State management: Automatic state persistence
- Error handling: Built-in retry and catch mechanisms
- Audit trail: Full execution history

---

### 1.4 Direct S3 Event Notification

#### Pattern: S3 Event → SQS → Lambda

**Use Case:** Direct file upload triggers processing (CIP/VPE)

```typescript
// S3 Bucket Event Configuration
{
  "LambdaFunctionConfigurations": [],
  "QueueConfigurations": [
    {
      "QueueArn": "arn:aws:sqs:region:account:queue",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            { "Name": "prefix", "Value": "input/" },
            { "Name": "suffix", "Value": ".pdf" }
          ]
        }
      }
    }
  ]
}

// Lambda: StartTextract (triggered by SQS from S3)
export const handler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    const s3Event = JSON.parse(record.body);
    const s3Record = s3Event.Records[0].s3;

    const bucket = s3Record.bucket.name;
    const key = decodeURIComponent(s3Record.object.key);

    // Create job and transaction records
    const jobId = uuidv4();
    const transactionId = uuidv4();

    await dynamoDB.batchWrite({
      RequestItems: {
        [JOBS_TABLE]: [
          {
            PutRequest: {
              Item: {
                ID: jobId,
                TYPE: 'J',
                STATUS: 'UPLOADED',
                CREATED_AT: new Date().toISOString()
              }
            }
          },
          {
            PutRequest: {
              Item: {
                ID: transactionId,
                TYPE: 'T',
                JOB_ID: jobId,
                SOURCE_KEY: `s3://${bucket}/${key}`,
                STATUS: 'UPLOADED',
                CREATED_AT: new Date().toISOString()
              }
            }
          }
        ]
      }
    }).promise();

    // Start Textract
    const textractJobId = await textract.startDocumentAnalysis({
      DocumentLocation: {
        S3Object: { Bucket: bucket, Name: key }
      },
      FeatureTypes: ['FORMS', 'TABLES'],
      NotificationChannel: {
        SNSTopicArn: TEXTRACT_SNS_ARN,
        RoleArn: TEXTRACT_ROLE_ARN
      }
    }).promise();

    // Update transaction with Textract job ID
    await dynamoDB.update({
      TableName: JOBS_TABLE,
      Key: { ID: transactionId },
      UpdateExpression: 'SET TEXTRACT_JOB_ID = :jobId, #status = :status',
      ExpressionAttributeNames: { '#status': 'STATUS' },
      ExpressionAttributeValues: {
        ':jobId': textractJobId.JobId,
        ':status': 'TEXTRACT_STARTED'
      }
    }).promise();
  }
};
```

**Characteristics:**
- **Latency:** Low (near real-time)
- **Coupling:** Loose (S3 doesn't wait for processing)
- **Error Handling:** SQS retry, DLQ
- **Use Cases:** Batch processing, ETL pipelines

**Benefits:**
- Serverless: No polling required
- Scalable: Lambda auto-scales with S3 events
- Decoupled: S3 upload independent of processing
- Reliable: SQS ensures message delivery

---

### 1.5 DynamoDB Streams

#### Pattern: DynamoDB Stream → Lambda

**Use Case:** React to database changes (e.g., transaction created)

```typescript
// DynamoDB Stream Configuration
{
  "StreamSpecification": {
    "StreamEnabled": true,
    "StreamViewType": "NEW_AND_OLD_IMAGES"
  }
}

// Lambda: ProcessTransaction (triggered by DynamoDB Stream)
export const handler = async (event: DynamoDBStreamEvent) => {
  for (const record of event.Records) {
    // Only process INSERT events for transactions with STATUS=UPLOADED
    if (
      record.eventName === 'INSERT' &&
      record.dynamodb.NewImage.TYPE.S === 'T' &&
      record.dynamodb.NewImage.STATUS.S === 'UPLOADED'
    ) {
      const transaction = unmarshall(record.dynamodb.NewImage);

      // Start transcription
      const transcriptionJob = await transcribe.startTranscriptionJob({
        TranscriptionJobName: transaction.ID,
        Media: {
          MediaFileUri: transaction.WORKING_KEY
        },
        MediaFormat: 'mp4',
        LanguageCode: 'en-US',
        Settings: {
          ShowSpeakerLabels: transaction.IDENTIFY_SPEAKERS,
          MaxSpeakerLabels: transaction.MAX_SPEAKER_LABELS
        }
      }).promise();

      // Update transaction status
      await dynamoDB.update({
        TableName: JOBS_TABLE,
        Key: { ID: transaction.ID },
        UpdateExpression: 'SET #status = :status, TRANSCRIPTION_JOB_NAME = :jobName',
        ExpressionAttributeNames: { '#status': 'STATUS' },
        ExpressionAttributeValues: {
          ':status': 'TRANSCRIPTION_STARTED',
          ':jobName': transcriptionJob.TranscriptionJob.TranscriptionJobName
        }
      }).promise();
    }
  }
};

// Lambda Configuration
{
  "EventSourceMapping": {
    "EventSourceArn": DYNAMODB_STREAM_ARN,
    "StartingPosition": "LATEST",
    "BatchSize": 10,
    "MaximumBatchingWindowInSeconds": 10,
    "FilterCriteria": {
      "Filters": [
        {
          "Pattern": "{\"eventName\":[\"INSERT\"],\"dynamodb\":{\"NewImage\":{\"TYPE\":{\"S\":[\"T\"]},\"STATUS\":{\"S\":[\"UPLOADED\"]}}}}"
        }
      ]
    }
  }
}
```

**Characteristics:**
- **Latency:** Near real-time (< 1 second)
- **Coupling:** Loose (database changes trigger processing)
- **Error Handling:** Lambda retry, DLQ
- **Use Cases:** Event-driven processing, data synchronization

**Benefits:**
- No polling: Stream pushes changes to Lambda
- Ordered processing: Stream guarantees order per partition key
- Exactly-once processing: Lambda handles deduplication
- Efficient: Only process relevant changes with filters

---

## 2. Event-Driven Architecture

### 2.1 EventBridge Integration

#### Pattern: AWS Service → EventBridge → Lambda

**Use Case:** AI service completion events (Textract, Transcribe, Rekognition)

```typescript
// EventBridge Rule for Textract Completion
{
  "EventPattern": {
    "source": ["aws.textract"],
    "detail-type": ["Textract Job State Change"],
    "detail": {
      "Status": ["SUCCEEDED", "FAILED"]
    }
  },
  "Targets": [
    {
      "Arn": LAMBDA_ARN,
      "Id": "TextractCompletionHandler"
    }
  ]
}

// Lambda: ResumeStepFunctions (triggered by EventBridge)
export const handler = async (event: EventBridgeEvent) => {
  const { JobId, Status, DocumentLocation } = event.detail;

  // Find transaction by Textract Job ID
  const result = await dynamoDB.query({
    TableName: JOBS_TABLE,
    IndexName: 'TextractJobIdIndex',
    KeyConditionExpression: 'TEXTRACT_JOB_ID = :jobId',
    ExpressionAttributeValues: { ':jobId': JobId }
  }).promise();

  if (result.Items.length === 0) {
    console.warn(`No transaction found for Textract job ${JobId}`);
    return;
  }

  const transaction = result.Items[0];

  if (Status === 'SUCCEEDED') {
    // Get Textract results
    const textractResults = await textract.getDocumentAnalysis({
      JobId
    }).promise();

    // Update transaction status
    await dynamoDB.update({
      TableName: JOBS_TABLE,
      Key: { ID: transaction.ID },
      UpdateExpression: 'SET #status = :status',
      ExpressionAttributeNames: { '#status': 'STATUS' },
      ExpressionAttributeValues: { ':status': 'TEXTRACT_COMPLETED' }
    }).promise();

    // Start Step Functions for post-processing
    await stepFunctions.startExecution({
      stateMachineArn: CIP_STATE_MACHINE_ARN,
      input: JSON.stringify({
        transactionId: transaction.ID,
        jobId: transaction.JOB_ID,
        textractResults
      })
    }).promise();

  } else if (Status === 'FAILED') {
    // Handle failure
    await dynamoDB.update({
      TableName: JOBS_TABLE,
      Key: { ID: transaction.ID },
      UpdateExpression: 'SET #status = :status, errorMessage = :error',
      ExpressionAttributeNames: { '#status': 'STATUS' },
      ExpressionAttributeValues: {
        ':status': 'FAILED',
        ':error': event.detail.StatusMessage
      }
    }).promise();

    // Send error notification
    await sns.publish({
      TopicArn: ERROR_SNS_ARN,
      Subject: 'Textract Job Failed',
      Message: JSON.stringify(event.detail)
    }).promise();
  }
};
```

**Event Types:**

| AWS Service | Event Pattern |
|-------------|---------------|
| **Textract** | `aws.textract` → Textract Job State Change |
| **Transcribe** | `aws.transcribe` → Transcription Job State Change |
| **Rekognition** | `aws.rekognition` → Video Analysis State Change |

**Benefits:**
- Serverless: No polling for job completion
- Scalable: EventBridge handles any volume of events
- Flexible: Multiple targets per event (Lambda, SQS, SNS, etc.)
- Observability: Event archive and replay for debugging

---

### 2.2 SNS Fanout Pattern

#### Pattern: One Publisher, Multiple Subscribers

**Use Case:** Broadcast events to multiple consumers

```typescript
// Single SNS Topic with Multiple Subscriptions

// Subscription 1: SQS Queue → ProcessCIPOutput Lambda
{
  "Protocol": "sqs",
  "Endpoint": CIP_OUTPUT_QUEUE_ARN,
  "FilterPolicy": {
    "notificationType": ["JOB_COMPLETED"]
  }
}

// Subscription 2: SQS Queue → HumanReview Lambda
{
  "Protocol": "sqs",
  "Endpoint": HUMAN_REVIEW_QUEUE_ARN,
  "FilterPolicy": {
    "notificationType": ["HUMAN_REVIEW_REQUIRED"]
  }
}

// Subscription 3: Email for Critical Errors
{
  "Protocol": "email",
  "Endpoint": "ops-team@example.com",
  "FilterPolicy": {
    "notificationType": ["JOB_FAILED"],
    "severity": ["CRITICAL"]
  }
}

// Subscription 4: Lambda for Audit Logging
{
  "Protocol": "lambda",
  "Endpoint": AUDIT_LAMBDA_ARN
  // No filter - receives all events
}

// Publisher (CIP Service)
await sns.publish({
  TopicArn: CIP_SNS_ARN,
  Message: JSON.stringify(notification),
  MessageAttributes: {
    notificationType: {
      DataType: 'String',
      StringValue: 'JOB_COMPLETED'
    },
    severity: {
      DataType: 'String',
      StringValue: 'INFO'
    }
  }
}).promise();
```

**Benefits:**
- Loose coupling: Publisher doesn't know subscribers
- Scalability: Add/remove subscribers without changing publisher
- Filtering: Subscribers only receive relevant messages
- Reliability: SNS retries delivery, DLQs for failed deliveries

---

## 3. Error Handling and Retry Logic

### 3.1 SQS Retry with Exponential Backoff

**Configuration:**

```typescript
// SQS Queue with DLQ
const queue = new Queue(this, 'ProcessingQueue', {
  visibilityTimeout: Duration.seconds(180),
  retentionPeriod: Duration.days(4),
  receiveMessageWaitTime: Duration.seconds(20), // Long polling
  deadLetterQueue: {
    queue: dlq,
    maxReceiveCount: 3  // Retry 3 times before DLQ
  }
});

// Lambda processes message
export const handler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    try {
      await processMessage(record);
      // Success - message auto-deleted
    } catch (error) {
      // Error - message returns to queue
      // ApproximateReceiveCount increments
      // After 3 retries, moves to DLQ
      throw error; // Propagate to trigger retry
    }
  }
};
```

**Retry Timeline:**

```
Attempt 1: Immediate (visibility timeout 0s)
  ↓ (failure)
Attempt 2: After 180s visibility timeout
  ↓ (failure)
Attempt 3: After 180s visibility timeout
  ↓ (failure)
Move to DLQ
```

**DLQ Monitoring:**

```typescript
// CloudWatch Alarm on DLQ depth
const alarm = new Alarm(this, 'DLQAlarm', {
  metric: dlq.metricApproximateNumberOfMessagesVisible(),
  threshold: 1,
  evaluationPeriods: 1,
  datapointsToAlarm: 1,
  alarmDescription: 'Messages in DLQ require investigation'
});

alarm.addAlarmAction(new SnsAction(errorNotificationTopic));
```

---

### 3.2 Lambda Retry Configuration

**Lambda Event Source Mappings:**

```typescript
// SQS Event Source
const eventSource = new SqsEventSource(queue, {
  batchSize: 10,
  maxBatchingWindow: Duration.seconds(10),
  reportBatchItemFailures: true,  // Partial batch failures
  maxConcurrency: 5  // Limit concurrent executions
});

lambda.addEventSource(eventSource);

// Partial Batch Failure Handling
export const handler = async (event: SQSEvent): Promise<SQSBatchResponse> => {
  const batchItemFailures: SQSBatchItemFailure[] = [];

  for (const record of event.Records) {
    try {
      await processMessage(record);
    } catch (error) {
      console.error(`Failed to process message ${record.messageId}`, error);
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  // Return failed message IDs for retry
  return { batchItemFailures };
};
```

**Benefits:**
- Granular retry: Only failed messages retry
- Efficiency: Successful messages don't reprocess
- Visibility: Failed message IDs logged

---

### 3.3 Step Functions Error Handling

**Retry and Catch:**

```json
{
  "StartAt": "ProcessDocument",
  "States": {
    "ProcessDocument": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 6,
          "BackoffRate": 2.0
        },
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 30,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["DocumentTooLargeError"],
          "ResultPath": "$.error",
          "Next": "HandleLargeDocument"
        },
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "NotifyError"
        }
      ],
      "Next": "Success"
    },
    "HandleLargeDocument": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "Next": "Success"
    },
    "NotifyError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...",
      "End": true
    },
    "Success": {
      "Type": "Succeed"
    }
  }
}
```

**Retry Schedule Example:**

```
Attempt 1: Immediate
Attempt 2: After 2s (2 × 2^0)
Attempt 3: After 4s (2 × 2^1)
Attempt 4: After 8s (2 × 2^2)
Attempt 5: After 16s (2 × 2^3)
Attempt 6: After 32s (2 × 2^4)
Total: ~62 seconds over 6 attempts
```

---

### 3.4 Idempotency

**Problem:** Duplicate messages can cause duplicate processing

**Solution:** Idempotency keys and conditional writes

```typescript
// Lambda: Idempotent message processing
export const handler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    const message = JSON.parse(record.body);
    const idempotencyKey = message.jobId;

    try {
      // Conditional write - only create if doesn't exist
      await dynamoDB.put({
        TableName: JOBS_TABLE,
        Item: {
          PK: `JOB#${idempotencyKey}`,
          status: 'PROCESSING',
          createdAt: new Date().toISOString()
        },
        ConditionExpression: 'attribute_not_exists(PK)'
      }).promise();

      // Process job
      await processJob(message);

    } catch (error) {
      if (error.code === 'ConditionalCheckFailedException') {
        // Job already exists - skip (idempotent)
        console.log(`Job ${idempotencyKey} already processed`);
        return;
      }
      throw error;
    }
  }
};
```

**Benefits:**
- Prevents duplicate processing
- Safe to retry
- Consistent state

---

## 4. Authentication and Authorization

### 4.1 Cognito JWT Validation

**API Gateway Cognito Authorizer:**

```typescript
// CDK Configuration
const authorizer = new CognitoUserPoolsAuthorizer(this, 'Authorizer', {
  cognitoUserPools: [userPool],
  identitySource: 'method.request.header.Authorization'
});

const api = new RestApi(this, 'API', {
  defaultCorsPreflightOptions: {
    allowOrigins: ['https://app.example.com'],
    allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
    allowHeaders: ['Content-Type', 'Authorization']
  }
});

api.root.addMethod('GET', new LambdaIntegration(lambda), {
  authorizer,
  authorizationType: AuthorizationType.COGNITO
});
```

**Lambda receives user context:**

```typescript
export const handler = async (event: APIGatewayProxyEventWithCognitoAuthorizer) => {
  const userId = event.requestContext.authorizer.claims.sub;
  const email = event.requestContext.authorizer.claims.email;
  const groups = event.requestContext.authorizer.claims['cognito:groups'] || [];

  // Authorize based on groups
  if (!groups.includes('Admins') && !groups.includes('Users')) {
    return {
      statusCode: 403,
      body: JSON.stringify({ error: 'Forbidden' })
    };
  }

  // Process request for authorized user
  // ...
};
```

---

### 4.2 Cross-Account IAM Roles

**Pattern:** Self-Service assumes CIP/VPE roles to publish to SNS

```typescript
// CIP Account: Create role for Self-Service to assume
const cipSnsPublishRole = new Role(this, 'CipSnsPublishRole', {
  assumedBy: new AccountPrincipal(SELF_SERVICE_ACCOUNT_ID),
  roleName: 'cip-sns-publish-role',
  description: 'Role for Self-Service to publish to CIP SNS'
});

cipPostProcessingSns.grantPublish(cipSnsPublishRole);

// Self-Service Account: Assume role and publish
const sts = new STS();
const credentials = await sts.assumeRole({
  RoleArn: CIP_SNS_PUBLISH_ROLE_ARN,
  RoleSessionName: 'SelfServiceSession'
}).promise();

const sns = new SNS({
  credentials: {
    accessKeyId: credentials.Credentials.AccessKeyId,
    secretAccessKey: credentials.Credentials.SecretAccessKey,
    sessionToken: credentials.Credentials.SessionToken
  }
});

await sns.publish({
  TopicArn: CIP_SNS_ARN,
  Message: JSON.stringify(notification)
}).promise();
```

**Benefits:**
- Secure cross-account communication
- Least privilege (role only has SNS publish permission)
- Auditable (CloudTrail logs AssumeRole calls)

---

### 4.3 API Key Management

**For internal/private APIs:**

```typescript
// API Gateway API Key
const apiKey = new ApiKey(this, 'ApiKey', {
  apiKeyName: 'InternalApiKey',
  description: 'API key for internal service integration'
});

const usagePlan = new UsagePlan(this, 'UsagePlan', {
  name: 'InternalUsagePlan',
  throttle: {
    rateLimit: 1000,
    burstLimit: 5000
  },
  quota: {
    limit: 1000000,
    period: Period.MONTH
  }
});

usagePlan.addApiKey(apiKey);
usagePlan.addApiStage({ stage: api.deploymentStage });

// Lambda validates API key (handled by API Gateway)
// Key must be in x-api-key header
```

---

## 5. Data Consistency Patterns

### 5.1 Eventually Consistent Updates

**Pattern:** Update multiple data stores asynchronously

```typescript
// Job completion updates multiple locations
async function completeJob(jobId: string) {
  // 1. Update DynamoDB (primary source of truth)
  await dynamoDB.update({
    TableName: JOBS_TABLE,
    Key: { PK: `JOB#${jobId}` },
    UpdateExpression: 'SET #status = :status, completedAt = :timestamp',
    ExpressionAttributeNames: { '#status': 'status' },
    ExpressionAttributeValues: {
      ':status': 'COMPLETED',
      ':timestamp': new Date().toISOString()
    }
  }).promise();

  // 2. Publish event (other services update themselves)
  await sns.publish({
    TopicArn: JOB_EVENTS_SNS_ARN,
    Message: JSON.stringify({
      eventType: 'JOB_COMPLETED',
      jobId,
      timestamp: new Date().toISOString()
    })
  }).promise();

  // Subscribers handle their own updates:
  // - Audit service logs completion
  // - Metrics service updates dashboard
  // - Notification service sends email
}
```

**Benefits:**
- Decoupled: Services update independently
- Scalable: No synchronous dependencies
- Resilient: Failure in one service doesn't block others

---

### 5.2 Transactional Writes (DynamoDB)

**Pattern:** Atomic updates to multiple items

```typescript
// Update job and all transactions atomically
await dynamoDB.transactWrite({
  TransactItems: [
    // Update job status
    {
      Update: {
        TableName: JOBS_TABLE,
        Key: { ID: jobId },
        UpdateExpression: 'SET #status = :status',
        ExpressionAttributeNames: { '#status': 'STATUS' },
        ExpressionAttributeValues: { ':status': 'COMPLETED' }
      }
    },
    // Update transaction 1
    {
      Update: {
        TableName: JOBS_TABLE,
        Key: { ID: transactionId1 },
        UpdateExpression: 'SET #status = :status',
        ExpressionAttributeNames: { '#status': 'STATUS' },
        ExpressionAttributeValues: { ':status': 'COMPLETED' }
      }
    },
    // Update transaction 2
    {
      Update: {
        TableName: JOBS_TABLE,
        Key: { ID: transactionId2 },
        UpdateExpression: 'SET #status = :status',
        ExpressionAttributeNames: { '#status': 'STATUS' },
        ExpressionAttributeValues: { ':status': 'COMPLETED' }
      }
    }
  ]
}).promise();
// All updates succeed or all fail
```

**Limitations:**
- Max 100 items per transaction
- All items must be in same region
- Higher latency than individual writes

---

## 6. Performance Optimization

### 6.1 Lambda Optimization

**Memory Tuning:**

```typescript
// CDK Configuration
const lambda = new NodejsFunction(this, 'ProcessingLambda', {
  memorySize: 3008,  // CPU scales with memory
  timeout: Duration.minutes(15),
  reservedConcurrentExecutions: 10,  // Limit concurrency
  environment: {
    NODE_OPTIONS: '--max-old-space-size=2900'  // Leave buffer
  }
});
```

**Performance Testing Results:**

| Memory | Duration | Cost per Invocation |
|--------|----------|---------------------|
| 1024 MB | 45s | $0.00075 |
| 2048 MB | 23s | $0.00077 |
| 3008 MB | 16s | $0.00080 |
| 10240 MB | 12s | $0.00100 |

**Optimal:** 3008 MB (best cost/performance ratio for document processing)

---

### 6.2 DynamoDB Optimization

**Batch Operations:**

```typescript
// Bad: Individual writes (25 API calls)
for (const item of items) {
  await dynamoDB.put({ TableName, Item: item }).promise();
}

// Good: Batch write (1 API call)
await dynamoDB.batchWrite({
  RequestItems: {
    [TableName]: items.map(item => ({
      PutRequest: { Item: item }
    }))
  }
}).promise();
```

**On-Demand Billing:**
- No capacity planning required
- Auto-scales to workload
- Cost-effective for variable traffic

**Query Optimization:**
- Use GSIs for non-primary key queries
- Limit result set size
- Use pagination for large datasets

---

### 6.3 S3 Transfer Acceleration

**For large file uploads:**

```typescript
// Enable Transfer Acceleration on bucket
const bucket = new Bucket(this, 'Bucket', {
  bucketName: 'my-bucket',
  transferAcceleration: true
});

// Generate accelerated presigned URL
const presignedUrl = await s3.getSignedUrlPromise('putObject', {
  Bucket: bucket.bucketName,
  Key: 'file.zip',
  Expires: 900,
  // Use accelerated endpoint
  Endpoint: `https://${bucket.bucketName}.s3-accelerate.amazonaws.com`
});
```

**Benefits:**
- 50-500% faster uploads (depending on location)
- Automatic routing to nearest edge location
- Cost: $0.04/GB (in addition to S3 PUT costs)

---

## 7. Testing Integration Points

### 7.1 Unit Testing Lambda Functions

```typescript
// Mock AWS SDK
import { mockClient } from 'aws-sdk-client-mock';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';

const ddbMock = mockClient(DynamoDBDocumentClient);

describe('RequestUploadApi', () => {
  beforeEach(() => {
    ddbMock.reset();
  });

  it('should create job and return presigned URL', async () => {
    ddbMock.on(PutCommand).resolves({});

    const event = {
      body: JSON.stringify({ jobName: 'Test Job' }),
      requestContext: {
        authorizer: {
          claims: { email: 'user@example.com' }
        }
      }
    };

    const result = await handler(event);

    expect(result.statusCode).toBe(201);
    const body = JSON.parse(result.body);
    expect(body.jobId).toBeDefined();
    expect(body.uploadUrl).toContain('X-Amz-Signature');
  });
});
```

---

### 7.2 Integration Testing

```typescript
// Test SNS → SQS → Lambda flow
describe('CIP Output Processing', () => {
  it('should process CIP completion notification', async () => {
    // 1. Publish to SNS
    await sns.publish({
      TopicArn: CIP_SNS_ARN,
      Message: JSON.stringify({
        jobName: 'test-job',
        presignedURL: 'https://...',
        service: ['SEARCHABLE_PDF']
      }),
      MessageAttributes: {
        notificationType: {
          DataType: 'String',
          StringValue: 'JOB_COMPLETED'
        }
      }
    }).promise();

    // 2. Wait for SQS to receive message
    await sleep(2000);

    const messages = await sqs.receiveMessage({
      QueueUrl: CIP_OUTPUT_QUEUE_URL,
      MaxNumberOfMessages: 1
    }).promise();

    expect(messages.Messages).toHaveLength(1);

    // 3. Trigger Lambda with SQS event
    const event = {
      Records: [{
        body: messages.Messages[0].Body
      }]
    };

    await processCIPOutputHandler(event);

    // 4. Verify DynamoDB updated
    const job = await dynamoDB.get({
      TableName: JOBS_TABLE,
      Key: { PK: 'JOB#test-job' }
    }).promise();

    expect(job.Item.status).toBe('CIP_COMPLETED');
  });
});
```

---

### 7.3 Load Testing

```typescript
// Artillery load test configuration
{
  "config": {
    "target": "https://api.example.com",
    "phases": [
      { "duration": 60, "arrivalRate": 10 },  // Ramp up
      { "duration": 300, "arrivalRate": 50 }, // Sustained load
      { "duration": 60, "arrivalRate": 10 }   // Ramp down
    ],
    "processor": "./auth.js"  // Custom auth logic
  },
  "scenarios": [
    {
      "name": "Create and process job",
      "flow": [
        {
          "post": {
            "url": "/api/v1/jobs",
            "headers": {
              "Authorization": "Bearer {{ token }}"
            },
            "json": {
              "jobName": "Load Test Job"
            },
            "capture": {
              "json": "$.jobId",
              "as": "jobId"
            }
          }
        },
        {
          "post": {
            "url": "/api/v1/jobs/{{ jobId }}/process"
          }
        }
      ]
    }
  ]
}
```

**Metrics to Monitor:**
- API latency (p50, p95, p99)
- Lambda throttling
- DynamoDB throttling
- SQS queue depth
- Error rate

---

## 8. Troubleshooting Guide

### 8.1 Common Issues and Solutions

#### Issue: Job stuck in PROCESSING status

**Symptoms:**
- Job status never updates to COMPLETED
- Step Functions execution shows running

**Diagnosis:**
```bash
# Check Step Functions execution
aws stepfunctions describe-execution \
  --execution-arn <execution-arn>

# Check for failed tasks
aws stepfunctions get-execution-history \
  --execution-arn <execution-arn> \
  --query 'events[?type==`TaskFailed`]'
```

**Causes:**
1. **CIP/VPE didn't send completion notification**
   - Check CIP/VPE CloudWatch logs
   - Verify SNS publish succeeded

2. **SQS message not delivered**
   - Check SQS queue metrics
   - Check DLQ for failed messages

3. **Lambda failed to process SQS message**
   - Check Lambda CloudWatch logs
   - Check Lambda errors/throttling metrics

4. **Step Functions callback not called**
   - Check if task token stored correctly
   - Verify `SendTaskSuccess` called

**Solution:**
```typescript
// Manual recovery: Send task success
await stepFunctions.sendTaskSuccess({
  taskToken: '<task-token>',
  output: JSON.stringify({ status: 'COMPLETED' })
}).promise();
```

---

#### Issue: High API latency

**Symptoms:**
- API Gateway p99 > 3000ms
- User complaints of slow response

**Diagnosis:**
```bash
# CloudWatch Insights query
fields @timestamp, @message, @duration
| filter @message like /RequestId/
| sort @duration desc
| limit 20
```

**Causes:**
1. **Cold start**
   - Lambda not warmed up
   - Large bundle size

2. **DynamoDB throttling**
   - Read/write capacity exceeded
   - Hot partition

3. **S3 latency**
   - Large presigned URL generation
   - S3 Transfer Acceleration not enabled

**Solutions:**
```typescript
// 1. Reduce cold starts
const lambda = new NodejsFunction(this, 'Lambda', {
  provisionedConcurrentExecutions: 5,  // Pre-warm
  bundling: {
    minify: true,
    sourceMap: false,
    target: 'node20',
    externalModules: ['aws-sdk']  // Exclude heavy dependencies
  }
});

// 2. Enable DynamoDB auto-scaling (or use on-demand)
const table = new TableV2(this, 'Table', {
  billing: Billing.onDemand()
});

// 3. Enable S3 Transfer Acceleration
const bucket = new Bucket(this, 'Bucket', {
  transferAcceleration: true
});
```

---

#### Issue: DLQ messages accumulating

**Symptoms:**
- DLQ depth alarm triggered
- Messages not processing after retries

**Diagnosis:**
```bash
# Receive message from DLQ
aws sqs receive-message \
  --queue-url <dlq-url> \
  --max-number-of-messages 1

# Check message attributes
aws sqs get-queue-attributes \
  --queue-url <dlq-url> \
  --attribute-names ApproximateNumberOfMessages
```

**Causes:**
1. **Persistent error in Lambda**
   - Bug in code
   - Invalid message format

2. **Timeout**
   - Lambda timeout too short
   - External API slow

3. **Permissions error**
   - Lambda can't access DynamoDB/S3
   - IAM policy missing

**Solutions:**
```typescript
// 1. Redrive messages to source queue (after fixing issue)
// Use redriveToLambda function

// 2. Increase timeout
const lambda = new NodejsFunction(this, 'Lambda', {
  timeout: Duration.minutes(15)  // Increase from 3 min
});

// 3. Add error handling
export const handler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    try {
      await processMessage(record);
    } catch (error) {
      console.error('Error processing message', {
        messageId: record.messageId,
        error: error.message,
        stack: error.stack
      });

      // Log to external system for investigation
      await logError(error);

      throw error;  // Move to DLQ after retries
    }
  }
};
```

---

#### Issue: Textract/Transcribe job not completing

**Symptoms:**
- Transaction stuck in TEXTRACT_STARTED/TRANSCRIPTION_STARTED
- No EventBridge event received

**Diagnosis:**
```bash
# Check Textract job status
aws textract get-document-analysis \
  --job-id <textract-job-id>

# Check Transcribe job status
aws transcribe get-transcription-job \
  --transcription-job-name <job-name>

# Check EventBridge rules
aws events list-rules \
  --name-prefix TextractCompletion
```

**Causes:**
1. **EventBridge rule disabled**
2. **SNS topic misconfigured** (for Textract)
3. **Job actually failed** (check AWS service status)
4. **Permissions** (Textract can't publish to SNS)

**Solutions:**
```bash
# Enable EventBridge rule
aws events enable-rule --name TextractCompletionRule

# Check Textract job (if failed, check StatusMessage)
aws textract get-document-analysis --job-id <job-id>

# Manually trigger completion handler
aws lambda invoke \
  --function-name ResumeStepFunctions \
  --payload '{...}' \
  response.json
```

---

### 8.2 Monitoring Dashboard

**Key Metrics to Monitor:**

```typescript
// CloudWatch Dashboard
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Invocations", { "stat": "Sum" }],
          [".", "Errors", { "stat": "Sum" }],
          [".", "Throttles", { "stat": "Sum" }],
          [".", "Duration", { "stat": "Average" }]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Lambda Metrics"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/SQS", "ApproximateNumberOfMessagesVisible", { "QueueName": "CipOutputQueue" }],
          [".", "ApproximateNumberOfMessagesVisible", { "QueueName": "CipOutputQueue-DLQ" }]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "SQS Queue Depth"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/ApiGateway", "Count", { "stat": "Sum" }],
          [".", "4XXError", { "stat": "Sum" }],
          [".", "5XXError", { "stat": "Sum" }],
          [".", "Latency", { "stat": "p99" }]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "API Gateway Metrics"
      }
    }
  ]
}
```

---

### 8.3 CloudWatch Log Insights Queries

**Find errors in Lambda logs:**
```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100
```

**Track job processing duration:**
```
fields @timestamp, jobId, @duration
| filter @message like /Job completed/
| stats avg(@duration), max(@duration), min(@duration) by bin(5m)
```

**Find slow API calls:**
```
fields @timestamp, @message, @duration
| filter @type = "REPORT"
| filter @duration > 3000
| sort @duration desc
| limit 20
```

**Count jobs by status:**
```
fields jobId, status
| filter @message like /Job status updated/
| stats count() by status
```

---

## Conclusion

This integration patterns document provides comprehensive guidance for implementing, debugging, and optimizing the Content AI Platform. For specific implementation details, refer to:

- [Component Overview](./component-overview.md) for service-specific information
- [Data Flow Diagrams](./data-flows.md) for visual flow representations
- [Architecture Diagram](./architecture-diagram.drawio) for system architecture
- Individual service READMEs for deployment and configuration

For questions or clarifications, contact the platform team.
