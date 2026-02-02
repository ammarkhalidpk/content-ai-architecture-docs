# Content AI Platform - Architecture Documentation

**Last Updated:** February 3, 2026
**Platform Version:** 3.0
**Document Status:** Current

---

## Executive Summary

The Content AI Platform is a comprehensive, serverless AWS solution for automated document and video processing. The platform provides intelligent content analysis, extraction, redaction, classification, and enrichment capabilities using AWS AI/ML services including Textract, Comprehend, Rekognition, Transcribe, Translate, and Bedrock.

### Platform Capabilities

- **Document Processing**: OCR, PII redaction/masking, document classification, searchable PDF generation
- **Video Processing**: Transcription, subtitle generation, face detection, content moderation, translation
- **Automated Workflows**: Step Functions orchestration for complex multi-service processing
- **Human-in-the-Loop**: Review workflows for classification and sensitive content
- **Enterprise Integration**: FRS file transfer, IESC document archiving

---

## Platform Components

The platform consists of four primary components:

### 1. **fbdms-adhoc-imaging-ui** (Frontend)
React-based web application providing user interface for job creation, monitoring, and management.

- **Technology**: React 18, Material-UI, AWS Cognito
- **Purpose**: Self-service document/video processing interface
- **Key Features**: Job wizard, dashboard, policy management, human review

### 2. **self-service-imaging-app** (Orchestration Layer)
AWS CDK application providing API Gateway, Step Functions orchestration, and service coordination.

- **Technology**: AWS CDK, TypeScript, Node.js 20
- **Purpose**: Orchestrate workflows between frontend, CIP, and VPE services
- **Key AWS Services**: API Gateway, Step Functions, Lambda, DynamoDB, S3, SNS, SQS, Cognito

### 3. **textract-imaging-service (CIP)** (Document AI Service)
Core Content Imaging Platform for document processing using AWS Textract, Comprehend, and Bedrock.

- **Technology**: AWS CDK, TypeScript, Node.js 20
- **Purpose**: Intelligent document processing and analysis
- **Key AWS Services**: Textract, Comprehend, Bedrock, SageMaker, Lambda, Step Functions, DynamoDB

### 4. **video-processing-enrichment (VPE)** (Video AI Service)
Video analysis and enrichment service using AWS Rekognition, Transcribe, Translate, and Bedrock.

- **Technology**: AWS CDK, TypeScript, Node.js 20
- **Purpose**: Video/audio transcription, analysis, and enrichment
- **Key AWS Services**: Rekognition, Transcribe, Translate, Bedrock, Lambda, DynamoDB

---

## Documentation Index

### Core Documentation

1. **[Architecture Diagram](./architecture-diagram.drawio)** - Visual system architecture (Draw.io format)
2. **[Data Flow Diagrams](./data-flows.md)** - Mermaid diagrams showing all integration patterns
3. **[Component Overview](./component-overview.md)** - Detailed component documentation
4. **[Integration Patterns](./integration-patterns.md)** - Service communication and error handling
5. **[Cost Analysis](./cost-analysis.md)** - Comprehensive AWS service pricing and volume scenarios

### Quick Links

| Topic | Description |
|-------|-------------|
| [Frontend Flow](#frontend-flow) | User-initiated processing via React UI |
| [API Flow](#api-flow) | Direct API integration for external systems |
| [S3 Trigger Flow](#s3-flow) | Direct S3 uploads to CIP/VPE services |
| [Authentication](#authentication) | Cognito-based authentication and authorization |
| [Error Handling](#error-handling) | SNS/SQS dead letter queues and retry patterns |
| [Cost Analysis](./cost-analysis.md) | AWS pricing for 10K-1M documents/videos per month |

---

## Architecture Highlights

### Multi-Entry Point Design

The platform supports **four distinct entry points** for flexibility:

1. **Frontend UI** → Self-Service API → Orchestration → CIP/VPE
2. **Direct API** → Self-Service API → Orchestration → CIP/VPE
3. **S3 Upload** → CIP/VPE Services (Direct)
4. **External Integration** → FRS/IESC Systems

### Event-Driven Architecture

- **SNS/SQS Messaging**: Asynchronous communication between services
- **DynamoDB Streams**: Trigger processing on job/transaction state changes
- **EventBridge**: Transcribe/Rekognition job completion events
- **Step Functions**: Long-running workflow orchestration

### Security & Compliance

- **Authentication**: AWS Cognito with JWT tokens
- **Authorization**: IAM roles and policies, API keys
- **Encryption**: S3 encryption at rest, TLS in transit, KMS for sensitive data
- **PII Protection**: Automated detection and redaction/masking capabilities
- **Audit Trail**: Comprehensive logging to CloudWatch

### Scalability

- **Serverless**: Auto-scaling Lambda, DynamoDB on-demand, Step Functions
- **Batch Processing**: SQS batch processing for high throughput
- **Parallel Processing**: Step Functions parallel branches for multiple services
- **Resource Optimization**: Lambda memory tuning, SQS visibility timeouts

---

## Getting Started

### For Developers

1. Review [Component Overview](./component-overview.md) to understand each service
2. Study [Data Flow Diagrams](./data-flows.md) for integration patterns
3. Examine [Integration Patterns](./integration-patterns.md) for implementation details

### For Architects

1. Open [Architecture Diagram](./architecture-diagram.drawio) in Draw.io
2. Review [Integration Patterns](./integration-patterns.md) for cross-service communication
3. Analyze [Component Overview](./component-overview.md) for AWS service usage

### For Operations

1. Understand [Error Handling](#error-handling) patterns in Integration Patterns doc
2. Review [Monitoring](#monitoring) section in Component Overview
3. Familiarize with [Data Flow Diagrams](./data-flows.md) for troubleshooting

---

## Key Architectural Decisions

### 1. Separation of Orchestration and Processing
**Decision**: Self-service-imaging-app orchestrates, CIP/VPE process
**Rationale**: Enables independent scaling, deployment, and reuse of CIP/VPE by other systems

### 2. Job/Transaction Model
**Decision**: Jobs contain multiple transactions (files)
**Rationale**: Supports ZIP uploads with multiple files, batch processing, granular status tracking

### 3. SNS/SQS Messaging Pattern
**Decision**: Services publish to SNS topics, consumers subscribe via SQS
**Rationale**: Decoupling, retry logic, dead letter queues, multiple consumers per event

### 4. Step Functions for Orchestration
**Decision**: Use Step Functions for multi-step workflows
**Rationale**: Visual workflow, error handling, retry logic, long-running processes, state management

### 5. DynamoDB Streams for Event Processing
**Decision**: DynamoDB streams trigger Lambda for job state changes
**Rationale**: Real-time processing, automatic retry, no polling overhead

---

## Technology Stack Summary

| Layer | Technologies |
|-------|-------------|
| **Frontend** | React 18, Material-UI, AWS Cognito, Recharts |
| **API Gateway** | REST API, Custom Domain, WAF Protection |
| **Compute** | Lambda (Node.js 20), Step Functions |
| **Storage** | S3, DynamoDB (on-demand) |
| **Messaging** | SNS, SQS, EventBridge |
| **AI/ML Services** | Textract, Comprehend, Rekognition, Transcribe, Translate, Bedrock, SageMaker |
| **Security** | Cognito, IAM, KMS, Secrets Manager |
| **Networking** | CloudFront, Route 53, VPC Endpoints |
| **Monitoring** | CloudWatch Logs, CloudWatch Metrics, X-Ray |
| **IaC** | AWS CDK (TypeScript) |

---

## Support and Contributions

### Documentation Updates
To update this documentation:
1. Edit markdown files in `/architecture-docs/`
2. Update Draw.io diagram using Draw.io desktop/web
3. Regenerate Mermaid diagrams as needed
4. Update version and date in README

### Contact Information
For questions or clarifications about the architecture, contact the platform team.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 3.0 | Feb 2026 | Comprehensive architecture documentation created |
| 2.0 | Jan 2026 | VPE service integration added |
| 1.0 | 2025 | Initial CIP integration with self-service UI |
