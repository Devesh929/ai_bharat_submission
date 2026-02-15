# Requirements Document: DisputePack AI

## Introduction

DisputePack AI is a GenAI-powered evidence kit assistant that helps online merchants win returns and chargeback disputes by transforming scattered information into submission-ready evidence packs. The system automates evidence collection, fact extraction, timeline generation, completeness validation, response drafting, and document packaging while ensuring privacy compliance and submission quality.

## Glossary

- **Merchant**: An online seller who receives dispute claims from customers or payment processors
- **Dispute**: A formal claim filed by a customer challenging a transaction (returns, chargebacks, refunds)
- **Evidence_Pack**: A complete submission package containing response letter, timeline, exhibit index, and labeled attachments
- **Intake_System**: The component that receives documents via WhatsApp, email, or direct upload
- **Orchestrator**: The AWS Lambda-based component that coordinates processing workflows
- **Document_Processor**: The component using Amazon Textract for OCR and text extraction
- **Fact_Extractor**: The component using Amazon Comprehend and Bedrock to identify key information
- **Timeline_Builder**: The component that constructs chronological event sequences
- **Completeness_Checker**: The component that validates evidence against dispute-type playbooks
- **Response_Drafter**: The component that generates policy-grounded dispute responses
- **Privacy_Redactor**: The component using Amazon Comprehend for PII detection and redaction
- **Playbook**: A dispute-type-specific checklist of required evidence and response patterns
- **Case**: A single dispute instance tracked in DynamoDB with associated documents and metadata
- **Exhibit**: A labeled piece of evidence (invoice, delivery proof, screenshot, etc.)

## Requirements

### Requirement 1: Multi-Channel Document Intake

**User Story:** As a merchant, I want to submit dispute documents through multiple channels, so that I can use my preferred communication method without switching tools.

#### Acceptance Criteria

1. WHEN a merchant sends a WhatsApp message with attachments to the system number, THE Intake_System SHALL receive the message via Twilio API and store attachments in S3
2. WHEN a merchant forwards an email to the system address, THE Intake_System SHALL receive the email via Amazon SES and extract all attachments to S3
3. WHEN a merchant uploads files through a web interface, THE Intake_System SHALL accept the files and store them in S3
4. WHEN documents are received, THE Intake_System SHALL create a Case record in DynamoDB with unique case identifier and timestamp
5. WHEN documents are stored in S3, THE Intake_System SHALL enqueue a processing message to Amazon SQS with the case identifier

### Requirement 2: Document Processing and Text Extraction

**User Story:** As a merchant, I want the system to read all my documents regardless of format, so that I don't need to manually transcribe information from images or PDFs.

#### Acceptance Criteria

1. WHEN a document is an image file (PNG, JPG, JPEG), THE Document_Processor SHALL apply Amazon Textract OCR to extract text content
2. WHEN a document is a PDF file, THE Document_Processor SHALL apply Amazon Textract to extract text and preserve structure
3. WHEN a document is a text file or email body, THE Document_Processor SHALL extract the text content directly
4. WHEN text extraction completes, THE Document_Processor SHALL store the extracted text in DynamoDB linked to the source document
5. WHEN text extraction fails, THE Document_Processor SHALL log the error and mark the document as requiring manual review

### Requirement 3: Fact Extraction from Documents

**User Story:** As a merchant, I want the system to automatically identify key facts from my documents, so that I don't need to manually highlight important information.

#### Acceptance Criteria

1. WHEN extracted text is available, THE Fact_Extractor SHALL identify order dates, amounts, tracking numbers, and delivery dates using Amazon Comprehend entity recognition
2. WHEN customer communications are processed, THE Fact_Extractor SHALL extract customer statements and complaint reasons using Bedrock LLM
3. WHEN multiple documents contain conflicting information, THE Fact_Extractor SHALL flag the conflict for merchant review
4. WHEN facts are extracted, THE Fact_Extractor SHALL store structured fact records in DynamoDB with source document references
5. WHEN a required fact type is not found in any document, THE Fact_Extractor SHALL mark it as missing in the case metadata

### Requirement 4: Timeline Generation

**User Story:** As a merchant, I want an automatic chronological timeline of events, so that I can clearly demonstrate the sequence of what happened.

#### Acceptance Criteria

1. WHEN all facts are extracted, THE Timeline_Builder SHALL construct a chronological sequence of events from order creation to dispute filing
2. WHEN events have timestamps, THE Timeline_Builder SHALL sort them in ascending chronological order
3. WHEN events lack precise timestamps, THE Timeline_Builder SHALL infer relative ordering from document context
4. WHEN the timeline is complete, THE Timeline_Builder SHALL generate a one-page formatted timeline document with event descriptions and source references
5. WHEN timeline generation completes, THE Timeline_Builder SHALL store the timeline in S3 and update the case status in DynamoDB

### Requirement 5: Dispute Type Classification and Playbook Selection

**User Story:** As a merchant, I want the system to identify what type of dispute I'm facing, so that it can check for the right evidence.

#### Acceptance Criteria

1. WHEN a case is processed, THE Completeness_Checker SHALL classify the dispute into one of four types: Item Not Received, Not as Described, Unauthorized/Fraud, or Refund Not Processed
2. WHEN the dispute type is identified, THE Completeness_Checker SHALL load the corresponding playbook from the system configuration
3. WHEN the dispute type cannot be determined with high confidence, THE Completeness_Checker SHALL flag the case for merchant clarification
4. WHEN multiple dispute types apply, THE Completeness_Checker SHALL select the primary type and note secondary concerns
5. WHEN the playbook is loaded, THE Completeness_Checker SHALL store the dispute type and playbook reference in DynamoDB

### Requirement 6: Evidence Completeness Validation

**User Story:** As a merchant, I want to know if I'm missing critical evidence before submission, so that I can gather it and avoid automatic rejection.

#### Acceptance Criteria

1. WHEN a playbook is selected, THE Completeness_Checker SHALL compare extracted facts and documents against the playbook's required evidence list
2. WHEN required evidence is present, THE Completeness_Checker SHALL mark it as satisfied with document references
3. WHEN required evidence is missing, THE Completeness_Checker SHALL create a missing evidence alert with description and importance level
4. WHEN all required evidence is present, THE Completeness_Checker SHALL mark the case as submission-ready
5. WHEN critical evidence is missing, THE Completeness_Checker SHALL mark the case as incomplete and notify the merchant with specific gaps

### Requirement 7: Policy-Grounded Response Drafting

**User Story:** As a merchant, I want an automatically written response that references my evidence and follows dispute resolution policies, so that I don't need to write it from scratch or worry about missing key points.

#### Acceptance Criteria

1. WHEN a case is ready for drafting, THE Response_Drafter SHALL retrieve relevant policy sections from Amazon OpenSearch vector store using the dispute type and facts as query
2. WHEN policy sections are retrieved, THE Response_Drafter SHALL use Amazon Bedrock LLM to generate a response that addresses each dispute point with evidence references
3. WHEN generating the response, THE Response_Drafter SHALL ensure every factual statement links to a specific exhibit or document
4. WHEN the response is complete, THE Response_Drafter SHALL format it as a professional business letter with proper structure
5. WHEN drafting completes, THE Response_Drafter SHALL store the response document in S3 and update the case status in DynamoDB

### Requirement 8: Evidence Pack Assembly

**User Story:** As a merchant, I want a complete, organized package ready to submit, so that I can send it directly to the payment processor without additional formatting work.

#### Acceptance Criteria

1. WHEN all components are ready, THE Evidence_Pack generator SHALL create an exhibit index listing all attachments with labels and descriptions
2. WHEN attachments are processed, THE Evidence_Pack generator SHALL rename files with clear exhibit labels (Exhibit_A_Invoice, Exhibit_B_Tracking, etc.)
3. WHEN the response letter references exhibits, THE Evidence_Pack generator SHALL ensure exhibit labels match the index
4. WHEN all documents are prepared, THE Evidence_Pack generator SHALL bundle the response letter, timeline, exhibit index, and labeled attachments into a single ZIP file
5. WHEN the Evidence_Pack is complete, THE Evidence_Pack generator SHALL store it in S3 and provide a download link to the merchant

### Requirement 9: Privacy and PII Redaction

**User Story:** As a merchant, I want sensitive customer information automatically redacted where appropriate, so that I comply with privacy regulations while still proving my case.

#### Acceptance Criteria

1. WHEN documents are processed, THE Privacy_Redactor SHALL use Amazon Comprehend to detect PII including full credit card numbers, social security numbers, and passport numbers
2. WHEN PII is detected in evidence documents, THE Privacy_Redactor SHALL redact it by replacing with masked values (e.g., **** **** **** 1234)
3. WHEN PII is essential to the case (e.g., last 4 digits of card, customer name), THE Privacy_Redactor SHALL preserve it based on redaction rules
4. WHEN redaction is applied, THE Privacy_Redactor SHALL create both redacted and unredacted versions of documents
5. WHEN the Evidence_Pack is generated, THE Privacy_Redactor SHALL include only appropriately redacted documents by default

### Requirement 10: Confidence Scoring and Human Review Flagging

**User Story:** As a merchant, I want to know when the system is uncertain about my case, so that I can review it manually before submission for high-stakes disputes.

#### Acceptance Criteria

1. WHEN fact extraction completes, THE Orchestrator SHALL calculate a confidence score based on evidence completeness, fact consistency, and dispute type match
2. WHEN the confidence score is below a threshold, THE Orchestrator SHALL flag the case for human review
3. WHEN critical evidence is missing, THE Orchestrator SHALL automatically flag the case regardless of other confidence factors
4. WHEN a case is flagged for review, THE Orchestrator SHALL update the case status in DynamoDB and notify the merchant
5. WHEN a merchant reviews and approves a flagged case, THE Orchestrator SHALL proceed with Evidence_Pack generation

### Requirement 11: Case Tracking and Status Management

**User Story:** As a merchant, I want to track the status of my dispute cases, so that I know what stage each case is in and what actions are needed.

#### Acceptance Criteria

1. WHEN a case is created, THE Orchestrator SHALL initialize the case status as "Intake_Complete" in DynamoDB
2. WHEN processing stages complete, THE Orchestrator SHALL update the case status to reflect current stage (Processing, Facts_Extracted, Timeline_Generated, Response_Drafted, Pack_Ready)
3. WHEN errors occur, THE Orchestrator SHALL update the case status to "Error" with error details
4. WHEN a case requires merchant action, THE Orchestrator SHALL update the status to "Awaiting_Merchant_Input" with specific action items
5. WHEN a merchant queries case status, THE Orchestrator SHALL return the current status, completion percentage, and next steps

### Requirement 12: Notification and Delivery

**User Story:** As a merchant, I want to be notified when my Evidence Pack is ready, so that I can download and submit it promptly.

#### Acceptance Criteria

1. WHEN an Evidence_Pack is complete, THE Orchestrator SHALL send a notification to the merchant via their preferred channel (WhatsApp, email, or SMS)
2. WHEN sending notifications, THE Orchestrator SHALL include the case identifier, dispute type, and download link
3. WHEN a case requires merchant input, THE Orchestrator SHALL send a notification with specific missing items or actions needed
4. WHEN errors occur during processing, THE Orchestrator SHALL notify the merchant with error details and support contact information
5. WHEN a notification is sent, THE Orchestrator SHALL log the notification event in CloudWatch for audit purposes

### Requirement 13: Monitoring and Audit Trail

**User Story:** As a system administrator, I want comprehensive logging and monitoring, so that I can troubleshoot issues and maintain compliance audit trails.

#### Acceptance Criteria

1. WHEN any component processes a case, THE component SHALL log processing events to CloudWatch with case identifier, timestamp, and action details
2. WHEN documents are accessed or modified, THE Orchestrator SHALL log the access event to CloudTrail with user identity and action type
3. WHEN errors occur, THE component SHALL log error details to CloudWatch with stack traces and context information
4. WHEN PII redaction is applied, THE Privacy_Redactor SHALL log redaction events without logging the actual PII values
5. WHEN cases are created or completed, THE Orchestrator SHALL emit CloudWatch metrics for monitoring processing times and success rates

### Requirement 14: Playbook Configuration and Management

**User Story:** As a system administrator, I want to configure and update dispute playbooks, so that the system adapts to changing payment processor requirements.

#### Acceptance Criteria

1. WHEN a playbook is created, THE system SHALL store it in DynamoDB with dispute type, required evidence list, and response templates
2. WHEN a playbook is updated, THE system SHALL version the playbook and preserve previous versions for audit purposes
3. WHEN processing a case, THE Completeness_Checker SHALL use the latest active version of the relevant playbook
4. WHEN policy documents are updated, THE system SHALL re-index them in Amazon OpenSearch for vector search
5. WHEN a playbook references policy sections, THE Response_Drafter SHALL retrieve the current policy text from OpenSearch

### Requirement 15: Error Handling and Recovery

**User Story:** As a merchant, I want the system to handle errors gracefully, so that temporary failures don't lose my case data or require me to restart from scratch.

#### Acceptance Criteria

1. WHEN a processing step fails, THE Orchestrator SHALL retry the operation up to three times with exponential backoff
2. WHEN retries are exhausted, THE Orchestrator SHALL move the case to error status and preserve all successfully processed data
3. WHEN SQS message processing fails, THE message SHALL return to the queue for reprocessing after visibility timeout
4. WHEN a case is in error status, THE merchant SHALL be able to trigger manual reprocessing from the last successful step
5. WHEN critical AWS services are unavailable, THE Orchestrator SHALL queue operations for retry when services recover
