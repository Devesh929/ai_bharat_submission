# Design Document: DisputePack AI

## Overview

DisputePack AI is an event-driven, serverless system built on AWS that automates the creation of dispute evidence packages for online merchants. The architecture follows a pipeline pattern where documents flow through intake, processing, analysis, drafting, and packaging stages. Each stage is loosely coupled through Amazon SQS queues and DynamoDB state management, enabling independent scaling and fault tolerance.

The system leverages AWS AI/ML services (Textract, Comprehend, Bedrock, OpenSearch) to extract facts, understand context, and generate policy-grounded responses. The design prioritizes reliability (retry mechanisms, dead-letter queues), auditability (CloudWatch/CloudTrail logging), and privacy compliance (PII detection and redaction).

## Architecture

### High-Level Architecture

```mermaid
graph TB
    subgraph Intake
        WA[WhatsApp/Twilio] --> API[API Gateway]
        EM[Email/SES] --> API
        WEB[Web Upload] --> API
        API --> Lambda1[Intake Lambda]
        Lambda1 --> S3[S3 Document Store]
        Lambda1 --> DDB[(DynamoDB Cases)]
        Lambda1 --> SQS1[Processing Queue]
    end
    
    subgraph Processing
        SQS1 --> Lambda2[Document Processor]
        Lambda2 --> TX[Textract]
        TX --> Lambda2
        Lambda2 --> DDB
        Lambda2 --> SQS2[Extraction Queue]
    end
    
    subgraph Analysis
        SQS2 --> Lambda3[Fact Extractor]
        Lambda3 --> COMP[Comprehend]
        Lambda3 --> BR1[Bedrock LLM]
        Lambda3 --> DDB
        Lambda3 --> SQS3[Timeline Queue]
        
        SQS3 --> Lambda4[Timeline Builder]
        Lambda4 --> DDB
        Lambda4 --> SQS4[Validation Queue]
        
        SQS4 --> Lambda5[Completeness Checker]
        Lambda5 --> DDB
        Lambda5 --> SQS5[Drafting Queue]
    end
    
    subgraph Generation
        SQS5 --> Lambda6[Response Drafter]
        Lambda6 --> OS[OpenSearch Vector Store]
        Lambda6 --> BR2[Bedrock LLM]
        Lambda6 --> S3
        Lambda6 --> SQS6[Packaging Queue]
        
        SQS6 --> Lambda7[Pack Assembler]
        Lambda7 --> RED[Privacy Redactor]
        Lambda7 --> S3
        Lambda7 --> DDB
        Lambda7 --> SNS[SNS Notifications]
    end
    
    subgraph Monitoring
        Lambda1 --> CW[CloudWatch Logs]
        Lambda2 --> CW
        Lambda3 --> CW
        Lambda4 --> CW
        Lambda5 --> CW
        Lambda6 --> CW
        Lambda7 --> CW
        API --> CT[CloudTrail]
    end
