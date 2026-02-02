# Content AI Platform - Component Overview

**Last Updated:** February 3, 2026

This document provides detailed documentation for each component of the Content AI Platform, including responsibilities, AWS services, entry points, Lambda functions, and data models.

---

## Table of Contents

1. [Frontend: fbdms-adhoc-imaging-ui](#1-frontend-fbdms-adhoc-imaging-ui)
2. [Orchestration: self-service-imaging-app](#2-orchestration-self-service-imaging-app)
3. [Document AI: textract-imaging-service (CIP)](#3-document-ai-textract-imaging-service-cip)
4. [Video AI: video-processing-enrichment (VPE)](#4-video-ai-video-processing-enrichment-vpe)
5. [Cross-Cutting Concerns](#5-cross-cutting-concerns)

---

## 1. Frontend: fbdms-adhoc-imaging-ui

### Purpose and Responsibilities

React-based single-page application providing self-service interface for document and video processing.

**Key Responsibilities:**
- User authentication and session management
- Job creation wizard with service selection
- File upload interface with validation
- Real-time job status monitoring
- Results download and visualization
- Classification policy management
- Human review dashboard
- Dashboard analytics and metrics

### Technology Stack

| Layer | Technology |
|-------|------------|
| Framework | React 18.3.1 |
| UI Library | Material-UI (MUI) 7.1.0 |
| Routing | React Router DOM 7.5.3 |
| Styling | Tailwind CSS 4.1.5 + Emotion CSS-in-JS |
| Authentication | AWS Cognito Identity SDK |
| Charts | Recharts 3.3.0 |
| Notifications | Notistack 3.0.2 |
| File Handling | JSZip 3.10.1 |
| Build Tool | Create React App (Webpack + Babel) |

### Key Features

#### 1. Job Creation Wizard

**Multi-step form for creating processing jobs:**

- **Step 1: Service Selection**
  - Document services: Searchable PDF, PII Redaction/Masking, Classification, Data Extraction
  - Video services: Transcription, Subtitles, Face Detection, Content Moderation

- **Step 2: File Upload**
  - Drag-and-drop file upload
  - ZIP file support for batch processing
  - File validation (type, size)
  - Preview uploaded files

- **Step 3: Configuration**
  - Service-specific options
  - Redaction/masking labels
  - Classification policies
  - Output preferences

- **Step 4: Review & Submit**
  - Summary of selections
  - Cost estimation
  - Submit job for processing

#### 2. Dashboard

**Real-time analytics and monitoring:**

- Job statistics (total, completed, failed, in-progress)
- Processing time charts
- Success/failure rate graphs
- Recent jobs list
- Quick actions (create job, view results)

#### 3. Jobs Management

**Job listing and monitoring:**

- Paginated job list with filters
- Status indicators (pending, processing, completed, failed)
- Search by job ID or name
- Sort by creation date, status, completion time
- Bulk actions (delete, retry)

#### 4. Job Details View

**Detailed job information:**

- Job metadata (ID, name, created date, user)
- Service configuration
- File list with individual status
- Processing logs
- Error details (if failed)
- Download processed results
- Retry failed jobs

#### 5. Human Review Dashboard

**Review interface for classification:**

- List of pending reviews
- Document preview with annotations
- AI classification results with confidence scores
- Approve/reject/escalate actions
- Review history and audit trail

#### 6. Policy Management

**Classification policy CRUD:**

- Create new policies with training samples
- Edit existing policies
- Test policies with sample documents
- View policy metrics (accuracy, usage)
- Enable/disable policies

### Authentication Flow

```typescript
// Cognito authentication
1. User enters credentials
2. CognitoUser.authenticateUser()
3. Cognito returns JWT tokens (IdToken, AccessToken, RefreshToken)
4. Store tokens in cookies (secure, httpOnly)
5. Attach Authorization header to all API requests
6. Refresh tokens automatically on expiry
```

### API Integration

**Base API Configuration:**

```typescript
const API_BASE_URL = process.env.REACT_APP_API_URL;
const API_TIMEOUT = 30000; // 30 seconds

// Request interceptor
axios.interceptors.request.use((config) => {
  const token = getAuthToken(); // From cookie
  config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor
axios.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      await refreshToken();
      return axios.request(error.config);
    }
    return Promise.reject(error);
  }
);
```

### State Management

**Component-level state with React hooks:**

- `useState` for local component state
- `useEffect` for side effects (API calls, polling)
- `useContext` for theme and configuration
- LocalStorage for user preferences
- No global state management (Redux/MobX)

### File Upload Flow

```typescript
1. User selects files
2. Validate file types and sizes
3. POST /api/v1/jobs → Get presigned URL
4. PUT file to S3 presigned URL
5. POST /api/v1/jobs/{id}/process → Start processing
6. Poll GET /api/v1/jobs/{id} for status
7. Download results when completed
```

### Build and Deployment

**Development:**
```bash
npm start  # Runs on http://localhost:3000
```

**Production Build:**
```bash
npm run build  # Creates optimized build in /build
```

**Deployment:**
- Static files deployed to S3
- CloudFront distribution for CDN
- Custom domain via Route 53
- Continuous deployment via CI/CD pipeline

---

## 2. Orchestration: self-service-imaging-app

### Purpose and Responsibilities

AWS CDK application providing orchestration layer between frontend and processing services.

**Key Responsibilities:**
- API Gateway management (public and private endpoints)
- Step Functions workflow orchestration
- Job and file metadata management
- Service coordination (CIP and VPE)
- Authentication and authorization
- SNS/SQS messaging setup
- S3 bucket management
- Error handling and notifications

### AWS Services Used

| Service | Purpose |
|---------|---------|
| API Gateway | REST API endpoints (public + private) |
| Lambda | API handlers, event processors, state machine tasks |
| Step Functions | Job processing workflow orchestration |
| DynamoDB | Jobs and files metadata storage |
| S3 | File storage (input, output, temporary) |
| SNS | Publish job events and notifications |
| SQS | Message queues for async processing |
| Cognito | User authentication and authorization |
| CloudFront | Content delivery and WAF protection |
| Route 53 | Custom domain DNS |
| CloudWatch | Logging and monitoring |
| Secrets Manager | API keys and credentials |
| IAM | Role and policy management |

### API Endpoints

#### Public API (User-facing)

**Authentication:** Cognito JWT Bearer token

```
POST   /api/v1/jobs                    # Create new job
GET    /api/v1/jobs                    # List jobs
GET    /api/v1/jobs/{id}               # Get job status
POST   /api/v1/jobs/{id}/process       # Start processing
GET    /api/v1/jobs/{id}/output        # Get output files
GET    /api/v1/jobs/{id}/audit         # Get audit file
DELETE /api/v1/jobs/{id}               # Delete job

GET    /api/v1/transactions            # List transactions
GET    /api/v1/transactions/{id}       # Get transaction details

POST   /api/v1/human-review            # Submit human review
GET    /api/v1/human-review            # List pending reviews
GET    /api/v1/human-review/{id}       # Get review details

GET    /api/v1/classification/policies # List classification policies
POST   /api/v1/classification/policies # Create policy
PUT    /api/v1/classification/policies/{id} # Update policy
DELETE /api/v1/classification/policies/{id} # Delete policy

GET    /api/v1/classification/metrics  # Get classification metrics

POST   /api/v1/login                   # User login (returns tokens)
```

#### Private API (Internal)

**Authentication:** API Key or VPC endpoint

```
POST   /internal/jobs                  # Internal job creation
GET    /internal/jobs/{id}             # Internal job status
POST   /internal/notifications         # Receive service notifications
```

### Lambda Functions

#### API Lambda Functions

| Function | Trigger | Purpose |
|----------|---------|---------|
| **RequestUploadApi** | POST /api/v1/jobs | Create job record, generate presigned URL |
| **GetJobStatusApi** | GET /api/v1/jobs/{id} | Query job and file status |
| **StartProcessingApi** | POST /api/v1/jobs/{id}/process | Trigger Step Functions workflow |
| **GetJobOutputApi** | GET /api/v1/jobs/{id}/output | Generate presigned URLs for outputs |
| **GetAuditFileApi** | GET /api/v1/jobs/{id}/audit | Generate audit trail document |
| **ListTransactions** | GET /api/v1/transactions | List transactions with filters |
| **HumanReview** | POST/GET /api/v1/human-review | Manage human review workflow |
| **ClassificationPolicyManagement** | CRUD /api/v1/classification/policies | Manage classification policies |
| **ClassificationMetrics** | GET /api/v1/classification/metrics | Get classification analytics |
| **LoginApi** | POST /api/v1/login | Cognito authentication |
| **GetSwaggerDocsApi** | GET /api/v1/swagger | API documentation |

#### Event Lambda Functions

| Function | Trigger | Purpose |
|----------|---------|---------|
| **RawInboundProcessingS3Event** | S3 PUT to /raw-inbound/ | Validate and process raw uploads |
| **ProcessCIPOutputEvent** | SQS (from CIP SNS) | Process CIP job completion |
| **ProcessVPEOutputEvent** | SQS (from VPE SNS) | Process VPE job completion |
| **UpdateJobForHumanReviewEvent** | SQS (from CIP SNS) | Update job for human review |
| **JobDeletionCleanupEvent** | DynamoDB Stream | Clean up S3 files when job deleted |
| **PollRedactionService** | EventBridge Schedule | Poll external redaction service |

#### State Machine Task Functions

| Function | Purpose |
|----------|---------|
| **CopyInputFilesTask** | Copy files to processing location |
| **StartCipTask** | Invoke CIP service with job config |
| **StartVpeTask** | Invoke VPE service with job config |
| **RenameFilesTask** | Rename output files per manifest |
| **UpdateManifestTask** | Update manifest with processing results |
| **MergePdfOutputTask** | Merge multiple PDFs into one |
| **XmlFileCreationTask** | Generate XML from XSD schema |
| **FrsTransferTask** | Transfer files to FRS system |
| **IescArchiveDocumentTask** | Archive documents in IESC |
| **CompleteJobTask** | Mark job complete, send notifications |
| **UpdateJobStatusOnErrorTask** | Handle workflow errors |

### Step Functions Workflow

**Job Processing State Machine:**

```
Start
  ↓
Copy Input Files
  ↓
Parallel Branch:
  ├─> Start CIP Task (if document services)
  │     ↓
  │   Wait for CIP Completion (callback pattern)
  │     ↓
  │   Process CIP Output
  │
  └─> Start VPE Task (if video services)
        ↓
      Wait for VPE Completion (callback pattern)
        ↓
      Process VPE Output
  ↓
Merge Outputs (if needed)
  ↓
Update Manifest (if applicable)
  ↓
Rename Files (if applicable)
  ↓
FRS Transfer (if configured)
  ↓
IESC Archive (if configured)
  ↓
Complete Job
  ↓
End
```

**Error Handling:**
- Retry with exponential backoff (3 attempts)
- Catch errors and invoke `UpdateJobStatusOnErrorTask`
- Send SNS notification on failure
- Update job status to FAILED in DynamoDB

### DynamoDB Schema

#### Jobs Table

**Table Name:** `{app-name}-jobs-table`
**Partition Key:** `PK` (string) - Format: `JOB#{jobId}` or `FILE#{fileId}`

**Job Record:**
```typescript
{
  PK: "JOB#uuid",
  type: "JOB",
  userId: "user-email",
  jobName: "My Processing Job",
  status: "PROCESSING", // CREATED, PENDING_UPLOAD, UPLOADED, PROCESSING, COMPLETED, FAILED
  createdAt: "2026-02-03T10:00:00Z",
  updatedAt: "2026-02-03T10:05:00Z",
  cipServices: ["SEARCHABLE_PDF", "PII_REDACTION"],
  vpeServices: ["TRANSCRIPTION", "FACE_DETECTION"],
  documentLabels: ["SSN", "CREDIT_CARD"],
  redactionTargets: ["SSN"],
  maskingTargets: ["CREDIT_CARD"],
  options: {
    outputSearchablePdf: true,
    vpeGenerateSubtitles: true,
    vpeIncludeTranscript: true,
    // ... other options
  },
  fileCount: 5,
  completedFiles: 3,
  failedFiles: 0,
  ttl: 1738569600 // Expiry timestamp
}
```

**File Record:**
```typescript
{
  PK: "FILE#uuid",
  type: "FILE",
  parentId: "JOB#parent-uuid",
  fileName: "document.pdf",
  fileSize: 1024000,
  mimeType: "application/pdf",
  s3Key: "jobs/job-id/input/document.pdf",
  status: "PROCESSING",
  createdAt: "2026-02-03T10:00:00Z",
  updatedAt: "2026-02-03T10:05:00Z",
  cipTransactionId: "cip-transaction-id", // If processed by CIP
  vpeTransactionId: "vpe-transaction-id", // If processed by VPE
  outputKeys: {
    searchablePdf: "s3://bucket/output/searchable.pdf",
    redacted: "s3://bucket/output/redacted.pdf"
  }
}
```

**Global Secondary Indexes:**
- `parentIdIndex`: Query files by job ID
- `statusIndex`: Query jobs/files by status
- `userIdIndex`: Query jobs by user

### S3 Bucket Structure

```
{bucket-name}/
  jobs/
    {job-id}/
      input/              # Original uploaded files
        document1.pdf
        document2.pdf
      working/            # Temporary processing files
        extracted/
      output/             # Final processed files
        searchable/
        redacted/
        transcripts/
      audit/              # Audit files
        audit.json

  raw-inbound/            # Direct S3 uploads (legacy)
    {file}.pdf

  temp/                   # Temporary files (auto-delete)
```

### SNS Topics

| Topic | Purpose | Subscribers |
|-------|---------|-------------|
| **Error Notification Topic** | Critical errors and failures | Operations team email/Slack |
| **CIP Post-Processing Output** | Receive CIP job completion | SQS → ProcessCIPOutputEvent Lambda |
| **VPE Post-Processing Output** | Receive VPE job completion | SQS → ProcessVPEOutputEvent Lambda |

**Message Attributes for Filtering:**
```typescript
{
  notificationType: "JOB_COMPLETED" | "HUMAN_REVIEW_REQUIRED" | "JOB_FAILED"
}
```

### SQS Queues

| Queue | Source | Consumer | Visibility Timeout |
|-------|--------|----------|-------------------|
| **CIP Output Queue** | CIP SNS (filter: JOB_COMPLETED) | ProcessCIPOutputEvent | 180s |
| **VPE Output Queue** | VPE SNS | ProcessVPEOutputEvent | 180s |
| **Human Review Queue** | CIP SNS (filter: HUMAN_REVIEW_REQUIRED) | UpdateJobForHumanReview | 180s |
| **Raw Inbound Queue** | S3 events (/raw-inbound/) | RawInboundProcessingS3Event | 180s |

**Dead Letter Queues (DLQ):**
- Each queue has associated DLQ with max retry count of 3
- DLQ messages trigger SNS alarm for manual investigation

### Configuration Management

**Environment-specific configs:**

```yaml
# config/dev/dev1.yaml
app: self-service-imaging-dev1
bucketName: self-service-imaging-dev1-bucket
region: us-east-1
environment: dev
cipApiUrl: https://cip-dev.example.com
vpeApiUrl: https://vpe-dev.example.com
cognitoUserPoolId: us-east-1_xxxxx
cognitoClientId: xxxxxxxxx
domainName: self-service-dev.example.com
certificateArn: arn:aws:acm:us-east-1:...
hostedZoneId: Z1234567890ABC
s3ObjectExpirationDays: 30
logLevel: debug
```

**Secrets (Secrets Manager):**
```json
{
  "cipApiKey": "xxx",
  "vpeApiKey": "xxx",
  "cipBucketName": "cip-bucket",
  "vpeBucketName": "vpe-bucket",
  "frsEndpoint": "https://frs.example.com",
  "frsApiKey": "xxx",
  "iescEndpoint": "https://iesc.example.com",
  "iescApiKey": "xxx"
}
```

---

## 3. Document AI: textract-imaging-service (CIP)

### Purpose and Responsibilities

Core Content Imaging Platform for intelligent document processing using AWS AI services.

**Key Responsibilities:**
- OCR and document analysis with Textract
- PII detection and redaction with Comprehend
- Document classification with Bedrock
- Searchable PDF generation
- Data extraction (forms, tables)
- Expense analysis
- Custom queries
- SageMaker model training for classification

### AWS Services Used

| Service | Purpose |
|---------|---------|
| **Textract** | OCR, form extraction, table extraction, layout analysis |
| **Comprehend** | PII entity detection, sentiment analysis |
| **Bedrock** | Claude for document classification and reasoning |
| **SageMaker** | Custom model training for classification |
| **Lambda** | Event handlers, API handlers, processing tasks |
| **Step Functions** | Document processing workflow orchestration |
| **DynamoDB** | Job/transaction status, client config, policies, knowledge base |
| **S3** | Document storage (input, intermediate, output, training data) |
| **SNS** | Job notifications, post-processing events |
| **SQS** | Queue for processing tasks |
| **EventBridge** | Textract job completion events |
| **KMS** | Encryption for sensitive data |
| **API Gateway** | Internal and external APIs |

### Entry Points

#### 1. S3 Upload (Direct)

```
S3 PUT → SQS Queue → StartTextract Lambda → Textract → EventBridge → Resume Lambda → Step Functions
```

#### 2. API Upload

```
POST /upload → Create Job → Return Presigned URL → Client uploads to S3 → S3 event triggers processing
```

#### 3. SNS Notification (from Self-Service)

```
Self-Service SNS → CIP Lambda → Create Job → Start Processing
```

### Lambda Functions

#### Event-Driven Lambdas

| Function | Trigger | Purpose |
|----------|---------|---------|
| **startTextract** | S3 PUT event | Start Textract analysis job |
| **resumeStepFunctions** | EventBridge (Textract complete) | Resume Step Functions with Textract results |
| **deleteDownloadedObject** | Step Functions task | Clean up temporary S3 objects |
| **redriveToLambda** | Manual/Scheduled | Redrive failed messages from DLQ |

#### API Lambdas

| Function | Endpoint | Purpose |
|----------|----------|---------|
| **upload** | POST /upload | Create job and return presigned URL |
| **jobStatus** | GET /jobs/{jobId} | Get job and transaction status |
| **classificationPolicyManagement** | CRUD /policies | Manage classification policies |
| **classificationReview** | POST /classification/review | Submit human review decision |
| **classificationPolicyGeneratorUpload** | POST /policies/generate | Generate policy from training samples |
| **classificationMetrics** | GET /classification/metrics | Get classification analytics |

### Step Functions Workflows

#### Main Processing State Machine

```
Start
  ↓
Process Textract Results
  ↓
Extract Text, Forms, Tables
  ↓
Parallel Branch:
  ├─> Comprehend PII Detection (if redaction/masking)
  ├─> Bedrock Classification (if classification enabled)
  └─> Custom Queries (if custom query specified)
  ↓
Generate Outputs:
  ├─> Searchable PDF
  ├─> Redacted PDF
  ├─> Masked PDF
  ├─> JSON metadata
  └─> CSV data extract
  ↓
Upload to S3
  ↓
Update DynamoDB Status
  ↓
Publish SNS Notification
  ↓
End
```

#### SageMaker Training State Machine

```
Start
  ↓
Prepare Training Data
  ↓
Upload to S3
  ↓
Start SageMaker Training Job
  ↓
Wait for Completion
  ↓
Deploy Model to Endpoint
  ↓
Update Model Registry
  ↓
End
```

### DynamoDB Tables

#### Job Status Table

**Table Name:** `{app}-job-status-table`
**Partition Key:** `ID` (Job ID or Transaction ID)

**Job Record:**
```typescript
{
  ID: "job-uuid",
  TYPE: "J",  // Job
  NAME: "job-name",
  STATUS: "COMPLETED",
  FILE_EXTENSION: ".zip",
  CONTENT_TYPE: "application/zip",
  CREATED_AT: "2026-02-03T10:00:00Z",
  UPDATED_AT: "2026-02-03T10:05:00Z",
  TRANSACTIONS: ["trans-1", "trans-2"],
  FILE_COUNT: 2,
  COMPLETED_COUNT: 2,
  ExpireAt: 1738569600  // TTL
}
```

**Transaction Record:**
```typescript
{
  ID: "transaction-uuid",
  TYPE: "T",  // Transaction
  JOB_ID: "job-uuid",
  NAME: "document",
  STATUS: "COMPLETED",
  FILE_EXTENSION: ".pdf",
  CONTENT_TYPE: "application/pdf",
  SOURCE_KEY: "s3://bucket/input/doc.pdf",
  WORKING_KEY: "s3://bucket/working/doc.pdf",
  OUTPUT_KEY: "s3://bucket/output/doc_searchable.pdf",
  TEXTRACT_JOB_ID: "textract-job-id",
  CREATED_AT: "2026-02-03T10:00:00Z",
  UPDATED_AT: "2026-02-03T10:05:00Z",
  METADATA: {
    pageCount: 10,
    wordCount: 5000,
    confidence: 0.95
  }
}
```

**GSIs:**
- `TypeGSI`: PK=TYPE, SK=CREATED_AT (query all jobs or transactions)
- `StatusGSI`: PK=STATUS, SK=UPDATED_AT (query by status)
- `JobIdGSI`: PK=JOB_ID, SK=CREATED_AT (query transactions for a job)

#### Client Table

Stores client-specific configuration.

```typescript
{
  CLIENT_BUCKET: "client-bucket-name",
  SERVICES: ["SEARCHABLE_PDF", "PII_REDACTION"],
  WEBHOOK_URL: "https://client.example.com/webhook",
  API_KEY: "encrypted-api-key",
  SETTINGS: {
    autoClassify: true,
    defaultPolicy: "policy-id",
    piiEntityTypes: ["SSN", "CREDIT_CARD"]
  }
}
```

#### Policy Table

Stores classification policies and training data.

```typescript
{
  ID: "policy-uuid",
  NAME: "Invoice Classification",
  DESCRIPTION: "Classifies invoices by type",
  CATEGORIES: ["Purchase Order", "Receipt", "Invoice"],
  TRAINING_SAMPLES: [
    {
      category: "Invoice",
      documentId: "doc-id",
      s3Key: "s3://bucket/training/invoice1.pdf",
      features: {...}
    }
  ],
  MODEL_VERSION: "v1.2",
  SAGEMAKER_ENDPOINT: "invoice-classifier-endpoint",
  ACCURACY: 0.95,
  CREATED_AT: "2026-01-01T00:00:00Z",
  UPDATED_AT: "2026-02-03T10:00:00Z",
  ACTIVE: true
}
```

#### Knowledge Base Table

Stores document knowledge base for semantic search.

```typescript
{
  KnowledgeId: "kb-uuid",
  TransactionId: "transaction-uuid",
  DocumentText: "Full document text",
  EmbeddingVector: [0.1, 0.2, ...],  // Titan embeddings
  Metadata: {
    documentType: "Invoice",
    extractedDate: "2026-02-03",
    amount: 1000.00
  }
}
```

### CIP Processing Types

| Service | Description | AWS Service |
|---------|-------------|-------------|
| **SEARCHABLE_PDF** | Generate searchable PDF with OCR | Textract + PDF library |
| **DATA_EXTRACT** | Extract forms and tables | Textract Forms/Tables |
| **EXPENSE_ANALYSIS** | Extract expense line items | Textract Expense |
| **DOCUMENT_CLASSIFICATION** | Classify documents | Bedrock Claude + SageMaker |
| **PII_REDACTION** | Redact PII entities | Textract + Comprehend |
| **PII_MASKING** | Mask PII entities | Textract + Comprehend |
| **PAGE_REDACTION** | Redact entire pages | Custom logic |
| **AUTO_ROTATION** | Auto-rotate pages | Textract orientation |
| **SPLIT_DOCUMENT** | Split by regex or pages | Custom logic |
| **CONTENT_EXTRACTION** | Extract specific content | Textract Queries |
| **CUSTOM_QUERY** | Answer questions about document | Textract Queries API |
| **COMPREHEND_PII_REDACTION** | Redact using Comprehend only | Comprehend |
| **COMPREHEND_PII_MASKING** | Mask using Comprehend only | Comprehend |
| **REGEX_REDACTION** | Redact by regex patterns | Custom regex engine |
| **DOCUMENT_QUALITY_CHECK** | Check document quality | Textract confidence scores |

### Classification Workflow

#### Training Phase

```
1. Upload training samples (PDF/images)
2. Extract text with Textract
3. Generate embeddings with Titan
4. Store in Knowledge Base Table
5. Train SageMaker model (batch processing)
6. Deploy model to endpoint
7. Update Policy Table with endpoint
```

#### Inference Phase

```
1. Document arrives for classification
2. Extract text with Textract
3. Generate embedding with Titan
4. Query Knowledge Base for similar documents
5. Call Bedrock Claude with:
   - Document text
   - Similar documents from KB
   - Policy categories
   - Classification instructions
6. Get classification + confidence + reasoning
7. If confidence < threshold → Human Review
8. Store classification in transaction metadata
```

### S3 Bucket Structure

```
{cip-bucket}/
  input/                    # Input documents
    {job-id}/
      {transaction-id}.pdf

  textract/                 # Textract output
    {job-id}/
      {transaction-id}/
        analysis.json
        forms.json
        tables.json

  working/                  # Temporary processing
    {job-id}/
      {transaction-id}/

  output/                   # Final output
    {job-id}/
      {transaction-id}/
        searchable.pdf
        redacted.pdf
        masked.pdf
        metadata.json
        extract.csv

  training/                 # SageMaker training data
    policies/
      {policy-id}/
        train/
          invoices/
          receipts/
        validation/

  models/                   # Trained model artifacts
    {model-id}/
      model.tar.gz

  policies/                 # Policy documents
    active/
      {policy-id}.json
    blueprints/
      template.json

  classification-output/    # Classification results
    {job-id}/
      {transaction-id}/
        classification.json
```

### SNS Topic Structure

**CIP Job Notifications:**

```typescript
// Job completed
{
  MessageAttributes: {
    notificationType: "JOB_COMPLETED"
  },
  Message: {
    jobName: "job-id",
    presignedURL: "https://...",
    service: ["SEARCHABLE_PDF", "PII_REDACTION"],
    contentType: "application/zip"
  }
}

// Human review required
{
  MessageAttributes: {
    notificationType: "HUMAN_REVIEW_REQUIRED"
  },
  Message: {
    jobId: "job-id",
    transactionId: "transaction-id",
    reviewId: "review-id",
    documentPreviewUrl: "https://...",
    classification: {
      policy_id: "policy-id",
      confidence: 0.65,
      reasoning: "Document structure suggests invoice but lacks key fields"
    }
  }
}
```

---

## 4. Video AI: video-processing-enrichment (VPE)

### Purpose and Responsibilities

Video and audio processing service using AWS Rekognition, Transcribe, Translate, and Bedrock.

**Key Responsibilities:**
- Audio/video transcription with speaker identification
- Subtitle generation (SRT, VTT)
- Face detection and recognition
- Celebrity recognition
- Content moderation
- Label detection (objects, scenes, activities)
- Text detection in video
- Person tracking
- Translation of transcripts
- AI-generated summaries with Bedrock

### AWS Services Used

| Service | Purpose |
|---------|---------|
| **Transcribe** | Audio transcription with speaker diarization |
| **Rekognition** | Video analysis (faces, labels, text, content moderation) |
| **Translate** | Translate transcripts to multiple languages |
| **Bedrock** | Generate summaries and insights from transcripts |
| **Lambda** | API handlers, event processors, processing tasks |
| **Step Functions** | Video processing workflow orchestration |
| **DynamoDB** | Job/transaction status, face metadata |
| **S3** | Media storage (input, output, transcripts, subtitles) |
| **SNS** | Job completion notifications |
| **EventBridge** | Transcribe/Rekognition job completion events |
| **API Gateway** | Job management APIs |

### Entry Points

#### 1. API Upload (Job/Transaction Model)

```
POST /jobs → Create Job → Return Presigned URL → POST /jobs/receive → Create Transaction → Process
```

#### 2. Direct S3 Upload

```
S3 PUT → S3 Event → Lambda → Create Job/Transaction → Start Processing
```

#### 3. SNS Notification (from Self-Service)

```
Self-Service SNS → VPE Lambda → Create Job → Start Processing
```

### Lambda Functions

| Function | Trigger | Purpose |
|----------|---------|---------|
| **uploadJob** | POST /jobs | Create job record, return presigned URL |
| **receiveUpload** | POST /jobs/receive | Process uploaded file (single or ZIP) |
| **processTransaction** | DynamoDB Stream | Start transcription when transaction created |
| **transcribeCompletionHandler** | EventBridge (Transcribe complete) | Update status, trigger state machine |
| **subtitleGeneration** | Step Functions task | Generate SRT/VTT subtitles |
| **getJobStatus** | GET /jobs/{jobId} | Get job and transactions status |
| **rekognitionCompletionHandler** | EventBridge (Rekognition complete) | Process Rekognition results |
| **faceClusteringHandler** | Step Functions task | Cluster detected faces into persons |

### Step Functions Workflow

```
Start
  ↓
Generate Subtitles (SRT + VTT)
  ↓
Parallel Branch (if video analysis enabled):
  ├─> Rekognition Label Detection
  ├─> Rekognition Face Detection
  ├─> Rekognition Celebrity Recognition
  ├─> Rekognition Content Moderation
  ├─> Rekognition Text Detection
  └─> Rekognition Person Tracking
  ↓
Wait for Rekognition Jobs
  ↓
Process Rekognition Results
  ↓
Face Clustering (group faces into persons)
  ↓
Translate Transcript (if requested)
  ↓
Generate Summary with Bedrock (if requested)
  ↓
Package Outputs
  ↓
Upload to S3
  ↓
Update DynamoDB Status
  ↓
Publish SNS Notification
  ↓
End
```

### DynamoDB Tables

#### Job Table (Jobs + Transactions)

**Table Name:** `{app}-job-table`
**Partition Key:** `ID`

**Job Record:**
```typescript
{
  ID: "job-uuid",
  TYPE: "J",  // Job
  NAME: "My Video",
  STATUS: "COMPLETED",
  FILE_EXTENSION: ".mp4",
  CONTENT_TYPE: "video/mp4",
  CREATED_AT: "2026-02-03T10:00:00Z",
  UPDATED_AT: "2026-02-03T10:05:00Z",
  TRANSACTIONS: ["trans-1", "trans-2"],
  FILE_COUNT: 2,
  IDENTIFY_SPEAKERS: true,
  MAX_SPEAKER_LABELS: 3,
  ttl: 1738569600
}
```

**Transaction Record:**
```typescript
{
  ID: "transaction-uuid",
  TYPE: "T",  // Transaction
  JOB_ID: "job-uuid",
  NAME: "video",
  STATUS: "COMPLETED",
  FILE_EXTENSION: ".mp4",
  CONTENT_TYPE: "video/mp4",
  SOURCE_KEY: "s3://bucket/media/in/video.mp4",
  WORKING_KEY: "s3://bucket/media/ready/job-id/trans-id.mp4",
  TRANSCRIPTION_JOB_NAME: "trans-job-name",
  TRANSCRIPTION_STATUS: "COMPLETED",
  SRT_KEY: "s3://bucket/jobs/job-id/subtitles/subtitles.srt",
  VTT_KEY: "s3://bucket/jobs/job-id/subtitles/subtitles.vtt",
  IDENTIFY_SPEAKERS: true,
  MAX_SPEAKER_LABELS: 3,
  REKOGNITION_JOBS: {
    labelDetection: "job-id-1",
    faceDetection: "job-id-2",
    celebrityRecognition: "job-id-3",
    contentModeration: "job-id-4",
    textDetection: "job-id-5",
    personTracking: "job-id-6"
  },
  CREATED_AT: "2026-02-03T10:00:00Z",
  UPDATED_AT: "2026-02-03T10:05:00Z"
}
```

**GSIs:**
- `TypeIndex`: PK=TYPE, SK=CREATED_AT
- `StatusIndex`: PK=STATUS, SK=UPDATED_AT
- `JobIdIndex`: PK=JOB_ID, SK=CREATED_AT

#### Face Metadata Table

**Table Name:** `{app}-face-metadata`
**Partition Key:** `FACE_ID`
**Sort Key:** `TIMESTAMP`

```typescript
{
  FACE_ID: "face-uuid",
  TIMESTAMP: 123456,  // Milliseconds in video
  TRANSACTION_ID: "transaction-uuid",
  COLLECTION_ID: "collection-uuid",
  PERSON_ID: "person-uuid",  // Clustered person identifier
  BOUNDING_BOX: {
    Width: 0.1,
    Height: 0.15,
    Left: 0.4,
    Top: 0.3
  },
  CONFIDENCE: 0.99,
  FACE_ATTRIBUTES: {
    AgeRange: { Low: 25, High: 35 },
    Gender: { Value: "Male", Confidence: 0.98 },
    Emotions: [
      { Type: "HAPPY", Confidence: 0.95 }
    ]
  },
  CREATED_AT: "2026-02-03T10:00:00Z",
  ttl: 1738569600
}
```

**GSIs:**
- `PersonIndex`: PK=PERSON_ID, SK=CREATED_AT
- `TransactionIndex`: PK=TRANSACTION_ID, SK=TIMESTAMP
- `CollectionIndex`: PK=COLLECTION_ID, SK=CREATED_AT

### VPE Processing Options

```typescript
{
  generateSubtitles: true,      // Generate SRT/VTT
  includeTranscript: true,       // Include raw transcript JSON
  generateSummary: true,         // Bedrock summary
  includeSourceVideo: true,      // Include original video in output
  analyzeVideo: true,            // Enable Rekognition analysis
  rekognitionOptions: {
    detectLabels: true,          // Objects, scenes, activities
    detectFaces: true,           // Face detection and attributes
    trackPersons: true,          // Person tracking across frames
    recognizeCelebrities: true,  // Celebrity recognition
    moderateContent: true,       // Content moderation
    detectText: true             // Text in video (OCR)
  },
  identifySpeakers: true,        // Speaker diarization
  maxSpeakerLabels: 3,          // Max number of speakers
  translateLanguages: ["es", "fr"], // Translate to Spanish and French
  faceCollection: "my-collection"  // Use existing face collection
}
```

### S3 Bucket Structure

```
{vpe-bucket}/
  media/
    in/                         # Initial uploads
      {job-id}.mp4
      {job-id}.zip

    ready/                      # Working copies
      {job-id}/
        {transaction-id}.mp4
        {transaction-id}.wav

  jobs/
    {job-id}/
      {transaction-id}/
        transcription.json      # Raw transcription

      subtitles/
        subtitles.srt           # Subtitle file (SRT format)
        subtitles.vtt           # Subtitle file (VTT format)
        subtitles_es.srt        # Spanish subtitles
        subtitles_fr.srt        # French subtitles

      rekognition/
        labels.json             # Label detection results
        faces.json              # Face detection results
        celebrities.json        # Celebrity recognition
        moderation.json         # Content moderation
        text.json               # Text detection
        persons.json            # Person tracking

      summary/
        summary.txt             # AI-generated summary
        summary.json            # Structured summary

      output/
        video_with_subtitles.mp4  # Video with burned-in subtitles (optional)
```

### Subtitle Generation

**SRT Format:**
```
1
00:00:00,000 --> 00:00:05,500
spk_0: Hello, welcome to this video.

2
00:00:05,500 --> 00:00:10,000
spk_1: Thank you for having me.
```

**VTT Format:**
```
WEBVTT

00:00:00.000 --> 00:00:05.500
spk_0: Hello, welcome to this video.

00:00:05.500 --> 00:00:10.000
spk_1: Thank you for having me.
```

### Transcription with Speaker Diarization

**Transcribe Output:**
```json
{
  "jobName": "trans-job-name",
  "results": {
    "transcripts": [
      {
        "transcript": "Hello, welcome to this video. Thank you for having me."
      }
    ],
    "speaker_labels": {
      "speakers": 2,
      "segments": [
        {
          "speaker_label": "spk_0",
          "start_time": "0.0",
          "end_time": "5.5",
          "items": [
            { "start_time": "0.0", "end_time": "0.5", "speaker_label": "spk_0" }
          ]
        }
      ]
    },
    "items": [
      {
        "start_time": "0.0",
        "end_time": "0.5",
        "alternatives": [
          { "confidence": "0.99", "content": "Hello" }
        ],
        "type": "pronunciation"
      }
    ]
  }
}
```

### Face Clustering Algorithm

**Purpose:** Group detected faces across video frames into unique persons.

**Algorithm:**
1. Collect all face detections from Rekognition
2. Extract face features/embeddings
3. Use DBSCAN clustering:
   - Distance metric: Face similarity (Rekognition SearchFacesByImage)
   - Epsilon: 0.3 (similarity threshold)
   - MinPoints: 3 (minimum faces to form cluster)
4. Assign PERSON_ID to each cluster
5. Store in Face Metadata Table with PERSON_ID

**Use Case:** Track individuals across video (e.g., count unique people, track speaker faces)

---

## 5. Cross-Cutting Concerns

### Authentication and Authorization

#### Cognito Configuration

**User Pool:**
- Email as username
- Password requirements: Min 8 chars, uppercase, lowercase, number, special char
- MFA: Optional (TOTP or SMS)
- Password recovery via email
- Email verification required

**User Pool Client:**
- App client ID for React UI
- OAuth 2.0 flows: Authorization code grant
- Allowed scopes: openid, email, profile
- Token expiration: Access token 1 hour, Refresh token 30 days

**JWT Token Structure:**
```json
{
  "sub": "user-uuid",
  "email": "user@example.com",
  "cognito:username": "user@example.com",
  "cognito:groups": ["Admins", "Reviewers"],
  "exp": 1738569600,
  "iat": 1738566000
}
```

#### API Gateway Authorization

**Public API:**
- Cognito User Pool Authorizer
- Validates JWT tokens
- Extracts user identity from token
- Passes user info to Lambda via context

**Private API:**
- API Key authorization
- VPC endpoint restriction
- IAM role authorization (for internal services)

### Error Handling and Monitoring

#### CloudWatch Logs

**Log Groups:**
```
/aws/lambda/{app-name}           # All Lambda logs
/aws/apigateway/{api-name}       # API Gateway access logs
/aws/states/{state-machine-name} # Step Functions execution logs
```

**Structured Logging:**
```typescript
logger.info("Processing job", {
  jobId: "job-uuid",
  userId: "user-email",
  service: "CIP",
  duration: 1234
});
```

#### CloudWatch Metrics

**Custom Metrics:**
- `JobsCreated`: Count of jobs created
- `JobsCompleted`: Count of jobs completed
- `JobsFailed`: Count of jobs failed
- `ProcessingDuration`: Average processing time
- `APILatency`: API response time
- `TextractPages`: Pages processed by Textract
- `TranscribeMinutes`: Minutes transcribed
- `RekognitionFrames`: Frames analyzed

**Alarms:**
- High error rate (> 5%)
- DLQ message count (> 0)
- API latency (> 3000ms p99)
- Lambda throttling
- DynamoDB throttling

#### Error Notification Flow

```
Lambda/Step Function Error
  ↓
CloudWatch Alarm
  ↓
SNS Error Notification Topic
  ↓
Email/Slack/PagerDuty
```

### Security Best Practices

#### Encryption

- **At Rest:**
  - S3: SSE-S3 or SSE-KMS
  - DynamoDB: Encryption enabled
  - Secrets Manager: Encrypted with KMS

- **In Transit:**
  - TLS 1.2+ for all API calls
  - HTTPS only (enforced by CloudFront and API Gateway)

#### IAM Least Privilege

- Lambda execution roles have minimal permissions
- Service-specific roles (separate roles for CIP, VPE, orchestration)
- Resource-based policies for cross-account access
- No hardcoded credentials (use IAM roles and Secrets Manager)

#### API Security

- WAF rules on CloudFront:
  - Rate limiting (100 requests/5 min per IP)
  - SQL injection protection
  - XSS protection
  - Geographic restrictions (if needed)

- API Gateway throttling:
  - Burst limit: 500 requests
  - Rate limit: 1000 requests/second

#### Data Privacy

- PII redaction before logging
- Presigned URLs expire after 15 minutes
- Temporary files deleted after processing
- S3 lifecycle policies for automatic deletion
- DynamoDB TTL for automatic cleanup

### Cost Optimization

#### S3 Lifecycle Policies

```
Input files:  Delete after 30 days
Working files: Delete after 7 days
Output files: Move to Glacier after 90 days
Logs: Delete after 30 days
```

#### DynamoDB TTL

- Jobs expire after 90 days
- Transactions expire with parent job
- Face metadata expires after 30 days

#### Lambda Optimization

- Right-sized memory allocation (1024-10240 MB based on workload)
- Timeout configuration (30s for API, 15min for processing)
- Reserved concurrency for critical functions
- Provisioned concurrency for latency-sensitive APIs (optional)

#### AI Service Cost Management

- Textract: Batch processing to reduce per-page costs
- Transcribe: Use standard mode (not real-time) for batch jobs
- Rekognition: Only enable required analysis types
- Bedrock: Use smaller models where sufficient (Haiku vs Sonnet vs Opus)

### Disaster Recovery

#### Backup Strategy

- **DynamoDB:** Point-in-time recovery enabled (35 days)
- **S3:** Versioning enabled on critical buckets
- **Secrets Manager:** Automatic rotation disabled (manual rotation with testing)

#### Regional Failover

- All resources in single region (us-east-1)
- Multi-region deployment planned for future (Route 53 failover)

#### Business Continuity

- RTO (Recovery Time Objective): 4 hours
- RPO (Recovery Point Objective): 1 hour
- Runbook for incident response documented

---

## Next Steps

- Review [Data Flow Diagrams](./data-flows.md) for integration patterns
- Examine [Integration Patterns](./integration-patterns.md) for implementation details
- See [Architecture Diagram](./architecture-diagram.drawio) for visual reference
- Consult individual service READMEs for deployment instructions
