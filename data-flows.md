# Content AI Platform - Data Flow Diagrams

**Last Updated:** February 3, 2026

This document illustrates the data flow patterns across the Content AI Platform using Mermaid diagrams. Each flow represents a different entry point and processing path through the system.

---

## Table of Contents

1. [Frontend-Initiated Flow](#1-frontend-initiated-flow)
2. [API-Initiated Flow](#2-api-initiated-flow)
3. [Direct S3 Upload to CIP](#3-direct-s3-upload-to-cip)
4. [Direct S3 Upload to VPE](#4-direct-s3-upload-to-vpe)
5. [Direct API to CIP/VPE](#5-direct-api-to-cipvpe)
6. [Human Review Flow](#6-human-review-flow)
7. [External Integration Flows](#7-external-integration-flows)

---

## 1. Frontend-Initiated Flow

### User uploads documents/videos via React UI

```mermaid
sequenceDiagram
    participant User
    participant ReactUI as React UI<br/>(fbdms-adhoc-imaging-ui)
    participant Cognito as AWS Cognito
    participant APIGW as API Gateway<br/>(Public)
    participant Lambda as API Lambda<br/>(Request Upload)
    participant DDB as DynamoDB<br/>(Jobs Table)
    participant S3 as S3 Bucket
    participant SFN as Step Functions<br/>(Job Processing)
    participant CIP as CIP Service
    participant VPE as VPE Service
    participant SNS as SNS Topic
    participant SQS as SQS Queue
    participant EventLambda as Event Lambda<br/>(Process Output)

    User->>ReactUI: 1. Login
    ReactUI->>Cognito: 2. Authenticate
    Cognito-->>ReactUI: 3. JWT Token

    User->>ReactUI: 4. Create Job (Upload Files)
    ReactUI->>APIGW: 5. POST /api/v1/jobs<br/>(Authorization: Bearer JWT)
    APIGW->>Cognito: 6. Validate JWT
    Cognito-->>APIGW: 7. Authorized
    APIGW->>Lambda: 8. Invoke RequestUploadApi
    Lambda->>DDB: 9. Create Job Record
    Lambda->>S3: 10. Generate Presigned URL
    Lambda-->>APIGW: 11. Return presigned URL
    APIGW-->>ReactUI: 12. Job ID + Upload URL

    ReactUI->>S3: 13. PUT files to presigned URL
    S3-->>ReactUI: 14. Upload Complete

    ReactUI->>APIGW: 15. POST /api/v1/jobs/{id}/process
    APIGW->>Lambda: 16. Invoke StartProcessingApi
    Lambda->>DDB: 17. Update Job Status
    Lambda->>SFN: 18. StartExecution (Job Processing)
    Lambda-->>APIGW: 19. Processing Started
    APIGW-->>ReactUI: 20. 202 Accepted

    Note over SFN: Step Functions Workflow
    SFN->>S3: 21. Copy Input Files
    SFN->>CIP: 22. Start CIP Task (if document)
    SFN->>VPE: 23. Start VPE Task (if video)

    Note over CIP,VPE: Services Process Files
    CIP->>SNS: 24. Publish Job Completed
    VPE->>SNS: 25. Publish Job Completed
    SNS->>SQS: 26. Message to Queue
    SQS->>EventLambda: 27. Trigger ProcessOutput
    EventLambda->>S3: 28. Download Results
    EventLambda->>DDB: 29. Update Job Status
    EventLambda->>SFN: 30. SendTaskSuccess (Resume)

    SFN->>DDB: 31. Complete Job

    User->>ReactUI: 32. Poll Job Status
    ReactUI->>APIGW: 33. GET /api/v1/jobs/{id}
    APIGW->>Lambda: 34. Invoke GetJobStatusApi
    Lambda->>DDB: 35. Query Job
    Lambda-->>APIGW: 36. Job Status + Results
    APIGW-->>ReactUI: 37. Display Status

    User->>ReactUI: 38. Download Results
    ReactUI->>APIGW: 39. GET /api/v1/jobs/{id}/output
    APIGW->>Lambda: 40. Invoke GetJobOutputApi
    Lambda->>S3: 41. Generate Presigned URL
    Lambda-->>APIGW: 42. Download URL
    APIGW-->>ReactUI: 43. Presigned URL
    ReactUI->>S3: 44. Download Files
    S3-->>ReactUI: 45. Files Downloaded
```

### Flow Summary

| Step | Component | Action |
|------|-----------|--------|
| 1-3 | Authentication | User logs in via Cognito |
| 4-12 | Job Creation | Create job record and get presigned upload URL |
| 13-14 | File Upload | Upload files directly to S3 |
| 15-20 | Start Processing | Trigger Step Functions workflow |
| 21-30 | Processing | Step Functions orchestrates CIP/VPE processing |
| 31 | Completion | Job marked complete in DynamoDB |
| 32-37 | Status Polling | User polls for job completion |
| 38-45 | Download | User downloads processed results |

---

## 2. API-Initiated Flow

### External system calls self-service-imaging-app API directly

```mermaid
sequenceDiagram
    participant External as External System
    participant APIGW as API Gateway<br/>(Private/Public)
    participant Lambda as API Lambda
    participant DDB as DynamoDB
    participant S3 as S3 Bucket
    participant SFN as Step Functions
    participant CIP as CIP/VPE Service
    participant SNS as SNS Topic
    participant SQS as SQS Queue
    participant EventLambda as Event Lambda

    External->>APIGW: 1. POST /api/v1/jobs<br/>(x-api-key or JWT)
    APIGW->>Lambda: 2. Invoke RequestUploadApi
    Lambda->>DDB: 3. Create Job Record
    Lambda->>S3: 4. Generate Presigned URL
    Lambda-->>APIGW: 5. Return Job ID + Upload URL
    APIGW-->>External: 6. 201 Created (Job ID, Upload URL)

    External->>S3: 7. PUT document/video ZIP
    S3-->>External: 8. Upload Complete

    External->>APIGW: 9. POST /api/v1/jobs/{id}/process<br/>{"options": {...}}
    APIGW->>Lambda: 10. Invoke StartProcessingApi
    Lambda->>DDB: 11. Update Job Status
    Lambda->>SFN: 12. StartExecution
    Lambda-->>APIGW: 13. 202 Accepted
    APIGW-->>External: 14. Processing Started

    Note over SFN,CIP: Orchestration & Processing
    SFN->>CIP: 15. Invoke CIP/VPE
    CIP->>SNS: 16. Publish Completion
    SNS->>SQS: 17. Queue Message
    SQS->>EventLambda: 18. Trigger Lambda
    EventLambda->>SFN: 19. SendTaskSuccess
    SFN->>DDB: 20. Complete Job

    External->>APIGW: 21. GET /api/v1/jobs/{id}
    APIGW->>Lambda: 22. Invoke GetJobStatusApi
    Lambda->>DDB: 23. Query Job
    Lambda-->>APIGW: 24. Job Status
    APIGW-->>External: 25. 200 OK (Status: COMPLETED)

    External->>APIGW: 26. GET /api/v1/jobs/{id}/output
    APIGW->>Lambda: 27. Invoke GetJobOutputApi
    Lambda->>S3: 28. Generate Download URL
    Lambda-->>APIGW: 29. Presigned URL
    APIGW-->>External: 30. 200 OK (Download URL)

    External->>S3: 31. GET processed files
    S3-->>External: 32. Files Downloaded
```

### Key Differences from Frontend Flow

- Uses API key or JWT authentication (no Cognito login UI)
- Typically batch/automated operations
- Supports both public and private API Gateway endpoints
- Can specify processing options in request body

---

## 3. Direct S3 Upload to CIP

### Files uploaded directly to CIP S3 bucket trigger processing

```mermaid
sequenceDiagram
    participant User as User/System
    participant S3 as S3 Bucket<br/>(CIP Input)
    participant SQS as SQS Queue
    participant Lambda as Lambda<br/>(Start Textract)
    participant EventBridge as EventBridge
    participant DDB as DynamoDB<br/>(Job Status Table)
    participant Textract as AWS Textract
    participant SFN as Step Functions<br/>(CIP State Machine)
    participant Comprehend as AWS Comprehend
    participant Bedrock as AWS Bedrock
    participant S3Out as S3 Bucket<br/>(CIP Output)
    participant SNS as SNS Topic<br/>(Post-Processing)

    User->>S3: 1. PUT document.pdf<br/>s3://bucket/input/doc.pdf
    S3->>SQS: 2. S3 Event Notification
    SQS->>Lambda: 3. Trigger StartTextract Lambda
    Lambda->>DDB: 4. Create Job Record<br/>(TYPE='J')
    Lambda->>DDB: 5. Create Transaction Record<br/>(TYPE='T')
    Lambda->>Textract: 6. StartDocumentAnalysis
    Lambda->>DDB: 7. Update Status: TEXTRACT_STARTED

    Note over Textract: Textract processes document

    Textract->>EventBridge: 8. Textract Job Complete Event
    EventBridge->>Lambda: 9. Trigger ResumeStepFunctions
    Lambda->>DDB: 10. Update Status: TEXTRACT_COMPLETED
    Lambda->>SFN: 11. Resume State Machine

    Note over SFN: Step Functions orchestrates

    SFN->>Lambda: 12. Process Textract Results
    Lambda->>S3Out: 13. Download Textract JSON
    Lambda->>Comprehend: 14. DetectPiiEntities (if PII redaction)
    Comprehend-->>Lambda: 15. PII entities
    Lambda->>Bedrock: 16. ClassifyDocument (if classification)
    Bedrock-->>Lambda: 17. Classification result
    Lambda->>S3Out: 18. Generate searchable PDF
    Lambda->>S3Out: 19. Upload processed document
    Lambda->>DDB: 20. Update Status: COMPLETED

    SFN->>SNS: 21. Publish Job Completed
    SNS->>SQS: 22. Notify subscribers (optional)

    Note over S3Out: Results available in output bucket
```

### Entry Point Characteristics

- **Trigger**: S3 PUT event
- **Authentication**: IAM role-based (no user authentication)
- **Use Case**: Batch processing, automated pipelines, ETL jobs
- **Isolation**: Processing isolated to CIP service (no orchestration layer)

---

## 4. Direct S3 Upload to VPE

### Video/audio files uploaded directly to VPE S3 bucket

```mermaid
sequenceDiagram
    participant User as User/System
    participant S3 as S3 Bucket<br/>(VPE Media Input)
    participant DDBStream as DynamoDB Stream
    participant Lambda as Lambda<br/>(Process Transaction)
    participant DDB as DynamoDB<br/>(Job Table)
    participant Transcribe as AWS Transcribe
    participant EventBridge as EventBridge
    participant SFN as Step Functions<br/>(VPE State Machine)
    participant Rekognition as AWS Rekognition
    participant Translate as AWS Translate
    participant Bedrock as AWS Bedrock
    participant S3Out as S3 Bucket<br/>(VPE Output)
    participant SNS as SNS Topic

    User->>S3: 1. PUT video.mp4<br/>s3://bucket/media/in/video.mp4

    Note over User,DDB: Job/Transaction Creation Flow
    User->>Lambda: 2. POST /jobs (Create Upload Job)
    Lambda->>DDB: 3. Create Job (TYPE='J', STATUS='PENDING_UPLOAD')
    Lambda-->>User: 4. Job ID + Presigned URL
    User->>S3: 5. Upload to presigned URL
    User->>Lambda: 6. POST /jobs/receive (Notify upload complete)
    Lambda->>DDB: 7. Create Transaction (TYPE='T', STATUS='UPLOADED')
    Lambda->>DDB: 8. Update Job (STATUS='UPLOADED')

    Note over DDBStream: DynamoDB Stream triggers processing
    DDB->>DDBStream: 9. Stream: New Transaction (STATUS='UPLOADED')
    DDBStream->>Lambda: 10. Trigger ProcessTransaction Lambda
    Lambda->>S3: 11. Verify file exists
    Lambda->>Transcribe: 12. StartTranscriptionJob
    Lambda->>DDB: 13. Update Transaction: TRANSCRIPTION_STARTED

    Note over Transcribe: Transcribe processes audio

    Transcribe->>EventBridge: 14. Transcription Complete Event
    EventBridge->>Lambda: 15. Trigger Completion Handler
    Lambda->>DDB: 16. Update Transaction: TRANSCRIPTION_COMPLETED
    Lambda->>SFN: 17. Start State Machine

    Note over SFN: Step Functions orchestrates enrichment

    SFN->>Lambda: 18. Generate Subtitles (SRT/VTT)
    Lambda->>S3Out: 19. Download transcription JSON
    Lambda->>S3Out: 20. Upload subtitles
    Lambda->>DDB: 21. Update: SUBTITLES_GENERATED

    SFN->>Rekognition: 22. StartLabelDetection (if video analysis)
    SFN->>Rekognition: 23. StartFaceDetection
    SFN->>Rekognition: 24. StartContentModeration

    Note over Rekognition: Rekognition analyzes video

    Rekognition->>EventBridge: 25. Analysis Complete Events
    EventBridge->>Lambda: 26. Trigger Completion Handlers
    Lambda->>S3Out: 27. Store Rekognition results
    Lambda->>DDB: 28. Update: VIDEO_ANALYSIS_COMPLETE

    SFN->>Translate: 29. Translate subtitles (if requested)
    Translate-->>SFN: 30. Translated text
    SFN->>Bedrock: 31. Generate summary (if requested)
    Bedrock-->>SFN: 32. Summary

    SFN->>S3Out: 33. Upload final results
    SFN->>DDB: 34. Update Transaction: COMPLETED
    SFN->>DDB: 35. Update Job: COMPLETED
    SFN->>SNS: 36. Publish Job Completed

    Note over S3Out: Results available in output bucket
```

### VPE Job/Transaction Model

- **Job**: Represents upload request (can contain multiple files if ZIP)
- **Transaction**: Individual media file being processed
- **Statuses**: `PENDING_UPLOAD → UPLOADED → TRANSCRIPTION_STARTED → TRANSCRIPTION_COMPLETED → SUBTITLES_GENERATED → COMPLETED`

---

## 5. Direct API to CIP/VPE

### External systems call CIP or VPE APIs directly (bypassing orchestration)

```mermaid
sequenceDiagram
    participant External as External System
    participant APIGW as API Gateway<br/>(CIP or VPE)
    participant Lambda as API Lambda
    participant DDB as DynamoDB
    participant S3 as S3 Bucket
    participant AI as AWS AI Service<br/>(Textract/Transcribe/etc)
    participant SFN as Step Functions
    participant SNS as SNS Topic

    External->>APIGW: 1. POST /upload<br/>{"service": ["SEARCHABLE_PDF"], "jobName": "..."}
    APIGW->>Lambda: 2. Invoke Upload Lambda
    Lambda->>DDB: 3. Create Job/Transaction
    Lambda->>S3: 4. Generate Presigned URL
    Lambda-->>APIGW: 5. Job ID + Upload URL
    APIGW-->>External: 6. 201 Created

    External->>S3: 7. PUT file to presigned URL
    S3-->>External: 8. Upload Complete

    Note over S3,AI: S3 event triggers processing
    S3->>Lambda: 9. S3 Event Notification
    Lambda->>DDB: 10. Update Status
    Lambda->>AI: 11. Start AI Job (Textract/Transcribe)

    Note over AI: AI service processes

    AI->>Lambda: 12. Job Complete (via EventBridge)
    Lambda->>SFN: 13. Start/Resume State Machine
    SFN->>Lambda: 14. Process Results
    Lambda->>S3: 15. Generate Output
    Lambda->>DDB: 16. Update Status: COMPLETED
    Lambda->>SNS: 17. Publish Job Completed

    Note over SNS: External system can subscribe to SNS
    SNS->>External: 18. Job Complete Notification (optional)

    External->>APIGW: 19. GET /jobs/{jobId}
    APIGW->>Lambda: 20. Get Job Status
    Lambda->>DDB: 21. Query Job
    Lambda-->>APIGW: 22. Job Details
    APIGW-->>External: 23. Status + Results URL
```

### Use Cases for Direct API Access

- **CIP Direct API**: Document processing systems that don't need VPE
- **VPE Direct API**: Video platforms that don't need document processing
- **Isolation**: Each service operates independently
- **Integration**: Services can still publish to SNS for downstream consumers

---

## 6. Human Review Flow

### Document classification requires human review

```mermaid
sequenceDiagram
    participant CIP as CIP Service
    participant SNS as CIP SNS<br/>(Post-Processing)
    participant SQS as SQS Queue<br/>(Human Review)
    participant Lambda as Lambda<br/>(Update Job for Review)
    participant DDB as DynamoDB<br/>(Self-Service)
    participant User as Reviewer
    participant ReactUI as React UI
    participant APIGW as API Gateway
    participant ReviewLambda as Lambda<br/>(Human Review API)
    participant CIPApi as CIP API<br/>(Submit Review)

    Note over CIP: Classification confidence < threshold

    CIP->>DDB: 1. Store classification result
    CIP->>SNS: 2. Publish HUMAN_REVIEW_REQUIRED<br/>{"notificationType": "HUMAN_REVIEW_REQUIRED"}
    SNS->>SQS: 3. Filter: HUMAN_REVIEW_REQUIRED → Queue
    SQS->>Lambda: 4. Trigger UpdateJobForHumanReview
    Lambda->>DDB: 5. Update Job Status: PENDING_HUMAN_REVIEW
    Lambda->>DDB: 6. Store review metadata

    Note over User: Reviewer opens dashboard

    User->>ReactUI: 7. Open Human Review Dashboard
    ReactUI->>APIGW: 8. GET /api/v1/human-review
    APIGW->>ReviewLambda: 9. List pending reviews
    ReviewLambda->>DDB: 10. Query PENDING_HUMAN_REVIEW
    ReviewLambda-->>APIGW: 11. Review list
    APIGW-->>ReactUI: 12. Display reviews

    User->>ReactUI: 13. Review document
    ReactUI->>APIGW: 14. GET /api/v1/human-review/{reviewId}
    APIGW->>ReviewLambda: 15. Get review details
    ReviewLambda->>DDB: 16. Query review + job
    ReviewLambda->>S3: 17. Generate document preview URL
    ReviewLambda-->>APIGW: 18. Review details + preview URL
    APIGW-->>ReactUI: 19. Display document + classification

    User->>ReactUI: 20. Approve/Reject classification
    ReactUI->>APIGW: 21. POST /api/v1/human-review/{reviewId}/submit<br/>{"decision": "APPROVED", "classification": "..."}
    APIGW->>ReviewLambda: 22. Submit review decision
    ReviewLambda->>DDB: 23. Update review status
    ReviewLambda->>CIPApi: 24. POST /classification/review/submit
    CIPApi->>DDB: 25. Update CIP job status
    CIPApi->>SFN: 26. Resume processing (if needed)
    ReviewLambda-->>APIGW: 27. Review submitted
    APIGW-->>ReactUI: 28. Success

    Note over CIPApi: CIP continues processing with human decision
```

### Human Review Trigger Conditions

- Classification confidence below threshold (e.g., < 70%)
- Multiple possible classifications with similar scores
- Document type not in training set
- Manual review flag set in policy

### Review Outcomes

- **APPROVED**: Use AI classification, continue processing
- **REJECTED**: Use human classification, retrain model
- **ESCALATED**: Forward to senior reviewer

---

## 7. External Integration Flows

### 7.1 FRS (File Repository Service) Transfer

```mermaid
sequenceDiagram
    participant SFN as Step Functions<br/>(Self-Service)
    participant Lambda as Lambda<br/>(FRS Transfer Task)
    participant S3 as S3 Bucket<br/>(Processed Output)
    participant FRS as FRS Service<br/>(External)
    participant DDB as DynamoDB

    Note over SFN: Job processing completed

    SFN->>Lambda: 1. Invoke FrsTransferTask
    Lambda->>DDB: 2. Get job details
    Lambda->>S3: 3. List output files
    S3-->>Lambda: 4. File list

    loop For each file
        Lambda->>S3: 5. Download file
        S3-->>Lambda: 6. File data
        Lambda->>FRS: 7. POST /files<br/>(Transfer file)
        FRS-->>Lambda: 8. Transfer confirmation
        Lambda->>DDB: 9. Update file transfer status
    end

    Lambda->>DDB: 10. Update job: FRS_TRANSFER_COMPLETE
    Lambda-->>SFN: 11. Task Success

    Note over SFN: Continue to next task
```

### 7.2 IESC (Integrated Enterprise System) Archive

```mermaid
sequenceDiagram
    participant SFN as Step Functions<br/>(Self-Service)
    participant Lambda as Lambda<br/>(IESC Archive Task)
    participant S3 as S3 Bucket
    participant DDB as DynamoDB
    participant IESC as IESC Service<br/>(External)
    participant SM as Secrets Manager

    Note over SFN: Job processing completed

    SFN->>Lambda: 1. Invoke IescArchiveDocumentTask
    Lambda->>DDB: 2. Get job metadata
    Lambda->>SM: 3. Get IESC credentials
    SM-->>Lambda: 4. API key
    Lambda->>S3: 5. Get audit file
    S3-->>Lambda: 6. Audit data

    Lambda->>IESC: 7. POST /archive<br/>{"jobId": "...", "files": [...]}
    IESC->>IESC: 8. Store in enterprise archive
    IESC-->>Lambda: 9. Archive ID

    Lambda->>DDB: 10. Update job: ARCHIVED<br/>Store archive ID
    Lambda-->>SFN: 11. Task Success

    Note over SFN: Complete job workflow
```

---

## Flow Comparison Matrix

| Flow Type | Entry Point | Authentication | Orchestration | Use Case |
|-----------|-------------|----------------|---------------|----------|
| **Frontend** | React UI → API Gateway | Cognito JWT | Step Functions | End users, self-service |
| **Direct API** | External → API Gateway | API Key/JWT | Step Functions | External systems, automation |
| **S3 to CIP** | S3 PUT → CIP | IAM Role | CIP State Machine | Batch document processing |
| **S3 to VPE** | S3 PUT → VPE | IAM Role | VPE State Machine | Batch video processing |
| **Direct CIP/VPE API** | External → CIP/VPE API | API Key | Service State Machine | Direct service integration |
| **Human Review** | SNS → SQS → Lambda | N/A (Event-driven) | Review workflow | Classification review |
| **FRS Transfer** | Step Function Task | IAM Role | Part of main workflow | File repository integration |
| **IESC Archive** | Step Function Task | IAM Role + API Key | Part of main workflow | Enterprise archiving |

---

## Processing Status Progression

### Self-Service Job Statuses

```
CREATED → PENDING_UPLOAD → UPLOADED → PROCESSING →
  ├─> CIP_PROCESSING → CIP_COMPLETED
  ├─> VPE_PROCESSING → VPE_COMPLETED
  ├─> PENDING_HUMAN_REVIEW → REVIEW_COMPLETED (if applicable)
  └─> COMPLETED / FAILED
```

### CIP Transaction Statuses

```
CREATED → TEXTRACT_STARTED → TEXTRACT_COMPLETED →
  ├─> PII_DETECTION (if applicable)
  ├─> CLASSIFICATION (if applicable)
  └─> COMPLETED
```

### VPE Transaction Statuses

```
UPLOADED → TRANSCRIPTION_STARTED → TRANSCRIPTION_COMPLETED →
  SUBTITLES_GENERATED →
  ├─> VIDEO_ANALYSIS (if applicable)
  └─> COMPLETED
```

---

## Error Handling Patterns

### Retry Flow (SQS Dead Letter Queue)

```mermaid
flowchart TD
    A[Message arrives in SQS] --> B{Lambda processes}
    B -->|Success| C[Delete message]
    B -->|Failure| D{Retry count < 3?}
    D -->|Yes| E[Message returns to queue]
    E --> F[Wait with backoff]
    F --> B
    D -->|No| G[Move to DLQ]
    G --> H[SNS notification]
    H --> I[Manual intervention]
```

### State Machine Error Handling

```mermaid
flowchart TD
    A[Task Execution] --> B{Task succeeds?}
    B -->|Yes| C[Next Task]
    B -->|No| D{Retryable error?}
    D -->|Yes| E[Retry with backoff]
    E --> F{Max retries?}
    F -->|No| A
    F -->|Yes| G[Catch block]
    D -->|No| G
    G --> H[Update Job Status: FAILED]
    H --> I[SNS notification]
    I --> J[End workflow]
```

---

## Performance Characteristics

| Flow Type | Latency | Throughput | Cost Optimization |
|-----------|---------|------------|-------------------|
| Frontend | 2-10 min | 10-100 jobs/min | On-demand scaling |
| Direct API | 2-10 min | 100-1000 jobs/min | Batch processing |
| S3 to CIP | 1-5 min | 1000+ docs/min | Event-driven, no polling |
| S3 to VPE | 5-30 min | 100-500 videos/min | Asynchronous processing |
| Human Review | Variable | 1-10 reviews/hr | Manual process |

---

## Monitoring & Observability

### Key Metrics per Flow

- **API Latency**: API Gateway → Lambda → Response time
- **Processing Duration**: Job start → Completion time
- **Success Rate**: Completed jobs / Total jobs
- **Error Rate**: Failed jobs / Total jobs
- **Queue Depth**: Messages in SQS queues
- **DLQ Messages**: Messages in dead letter queues
- **Step Function Executions**: Running, succeeded, failed
- **AI Service Usage**: Textract pages, Transcribe minutes, Bedrock requests

### CloudWatch Log Insights Queries

Available in each component's documentation for troubleshooting specific flows.

---

## Next Steps

- Review [Component Overview](./component-overview.md) for detailed service documentation
- Examine [Integration Patterns](./integration-patterns.md) for implementation details
- See [Architecture Diagram](./architecture-diagram.drawio) for visual reference
