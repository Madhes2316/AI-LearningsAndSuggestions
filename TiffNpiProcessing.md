

---

### 1. High-Level Architecture

The system follows an event-driven, serverless approach:

TIFF Upload → Amazon S3 → Amazon SQS → Lambda (Controller) → SQS (Batch Queue) → Lambda (Workers) → DynamoDB → Lambda (Aggregator) → S3 (Final Output)

---

### 2. Detailed Flow

Step 1: Upload
- TIFF files are uploaded to Amazon S3.
- No preprocessing or conversion (e.g., TIFF → PDF) is performed at this stage.

Step 2: Event Trigger
- S3 Event Notification pushes a message to SQS (File Queue) for each uploaded file.

Step 3: Controller Lambda
- Triggered by SQS (File Queue).
- Reads S3 file metadata.
- Splits the TIFF into logical batches (e.g., 5–15 pages per batch).
- Creates a metadata record in DynamoDB.
- Pushes batch messages into SQS (Batch Queue).

Step 4: Worker Lambdas (Parallel Processing)
- Triggered by SQS (Batch Queue).
- Each Lambda processes a batch:
  - Downloads TIFF from S3
  - Runs OCR + NPI detection
  - Stores batch-level results in DynamoDB
- Updates batch status and increments completion counter.

Step 5: DynamoDB (State Tracking)
- Tracks:
  - Total batches
  - Completed batches
  - Processing status
  - NPI results

Step 6: Aggregator Lambda
- Triggered via DynamoDB Streams.
- When all batches are completed:
  - Aggregates results
  - Calculates NPI percentage
  - Updates final status
  - Optionally generates final PDF

Step 7: Output
- Final results (JSON/PDF) stored in S3.

---

### 3. DynamoDB Schema Design

Table Name: NPI_Processing

Primary Key:
- PK: FILE#<fileId>
- SK: METADATA or BATCH#<batchId>

---

File Metadata Item Example:

{
  "PK": "FILE#abc123",
  "SK": "METADATA",
  "s3Key": "uploads/file1.tiff",
  "totalBatches": 2,
  "completedBatches": 1,
  "status": "PROCESSING",
  "npiPages": [2,5,7],
  "npiPercentage": 60,
  "isNpiProcessed": false,
  "isPdfCreated": false,
  "createdAt": 1700000000,
  "updatedAt": 1700000100
}

---

Batch Item Example:

{
  "PK": "FILE#abc123",
  "SK": "BATCH#1",
  "batchId": 1,
  "pages": "1-10",
  "status": "COMPLETED",
  "npiPages": [2,5],
  "confidence": 87,
  "processedAt": 1700000050
}

---

### 4. Key Design Principles

- DynamoDB is used as a state tracker, not as file inventory.
- Records are created only when processing starts (not at upload stage).
- S3 remains the source of truth for files.
- SQS ensures buffering, retry handling, and decoupling.
- Lambda provides auto-scaling compute with no idle cost.

---

### 5. Benefits of This Approach

- Eliminates always-running ECS/Fargate cost
- Fully event-driven and auto-scalable
- Handles millions of files efficiently
- Reduces infrastructure complexity (no CSV, no SQL dependency)
- Improves fault tolerance with SQS retries and DynamoDB tracking

---

### 6. Estimated Cost (for ~8M files)

- Lambda: $130 – $250  
- SQS: $5 – $10  
- DynamoDB: $50 – $150  
- S3 (6 TB storage + requests): $150 – $200  

Total Estimated Cost: $350 – $600

---

### 7. Additional Recommendations

- Use batch size of 5–15 pages per Lambda
- Set SQS visibility timeout > Lambda execution time
- Implement idempotency using (fileId + batchId)
- Enable DynamoDB TTL for automatic cleanup
- Add DLQ (Dead Letter Queue) for failed processing
- Limit logging to avoid CloudWatch cost overhead

---

### 8. Conclusion

This architecture replaces the current ECS-based approach with a cost-efficient, scalable, and maintainable serverless design. It is well-suited for high-volume document processing workloads and significantly reduces operational overhead and cost.

---

---
