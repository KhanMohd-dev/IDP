# Orchestration Layer Design for Intelligent Document Processing (IDP) System

> This design document outlines a comprehensive architecture for the orchestration layer of an Intelligent Document Processing system, focusing on managing asynchronous workflow tasks while ensuring scalability, reliability, and observability.


### Functional Requirements:
1. Workflow coordination
   Managing document splitting -> classification -> extraction sequence
   Handle sub-document relationships after splitting
2. Task Scheduling
   Prioritize VIP documents over regular ones
   Implement retries with exponential backoff (3 attempts for splitting)
3. State Management
   Track real-time status of each sub-document
   Maintain processing history with timestamps
4. Error Handling
   Route failed tasks to dead-letter queue
   Send alerts for manual intervention after 3 retries
5. Logging
   tracing of document workflows

### Non-Functional Requirements:
1. Availability > Consistency (Availability + Partition Tolerance)
   99.95% uptime SLA
   Multi-AZ deployment + circuit breakers
2. Scalability
   Handle 10k+ documents/day
   Auto-scale workers when queue depth > 100
3. Latency
   VIP documents processed in <2sec (P99)
4. Security
   TLS L3 encryption for data in transit
5. Observability
   Track end-to-end processing latency

### Out-of-scope Requirements
1. Document Processing Implementation
   Actual ML models for classification
   OCR/text extraction algorithms
   Document splitting logic implementation
2. Storage & Infrastructure
   Physical document storage solutions
   Database sharding implementation details
3. End-user features
   Document preview UI
4. Third-Party integrations
   ERP/CRM systems integrations
   Payment gateway connections


### High-Level Design


For our high-level design, focusing on getting a simple system up and running that satisfies our core functional requirements by simply following the data flow outlined above.
We shall improve upon this design in coming sections.

The core components of our high-level design are: <br/>
**Client:** Where the user visits the app and upload the document for processing. <br/>

**API Gateway:** A place where all the routing/ rate limiting / authentication takes place. Its also a place where the loas balancer also resides
to take routing for available service layer. We also have priority service client which is places at this stage because it directly interacts with
priority router service helps in adding priority type to document which the user requested to process VIP / Regular. <br/>

**Priority Router Service:** It handles the priority handling of dynamic queue prioritization (VIP/Regular) and aging promotions for stuck documents when failed add it
to DLQ. We can multiple scheduling methods implemented at this level like weighted round robin(based on size of documents) or batch processing. For better
handling of documents we can combine size-aware batching with priority adjusted weighted round robin which will provide better throughput and fairness. <br/>

**Dead Letter Queue(DLQ):** a dead letter end queue is to store all the failed documents when the priority router service fails or orchestartion layer is down or processing
fails due to an error then we can have document stored here. and once when the service is free and ready to process we can push back these failed
documents back to processing. We can use AWS SQS for this. Since the priority type is already known, we can directly inject these fail messages back into
respective priority queues. <br/>

**VIP queue / Regular queue:** Here we can use AWS SQS for this IDP system because of the rich features which can be utilized efficiently like:
- Using FIFO queues for VIP prioritization
- Use standard queues for regular processing <br/>

**Orchestration layer:**
Where all the processing happens in details from Document splitting -> document classification and field extraction and finally this data is stored at multiple places:
1. AWS S3 : Storing the original documents as is here and
2. SQL DB: Storing the document, subdocument here in order to store and as its metadata stores the link url to original document stored in S3
3. state DB: Basically a NoSQL db that stores the information of each and every phase of the orchestration layer in order to keep track <br/>

**Logging:** To monitor the process and provide logs to server


### Data Flow / Sequence Diagrams

Key components and flow details:

1. Priority Router Service
   VIP/ Regular detection occurs immediately after upload using:
   - customer and document metadata checks
   - Document attributes: priority - 'VIP' or 'regular'
   - Business rules (eg: high-value invoices)
2. Queue Management
    - VIP documents bypass regular queue
    - Separate processing pipelines to maintain priority isolation
3. Parallel Processing
    - Both paths use same services but different resource allocation
    - VIP path gets:
      dedicated compute resources
      Higher retry points
      Shorter timeout thresholds
4. Type specific
    - This field extraction phase again takes up priority isolation based on type of fields of files
5. aggregation
    - combines results from both paths
    - final validation applies uniform quality checks
    - write to DB

Error Handling Differences
 - Regular Flow Error Handling:
 - Standard retry with exponential backoff
 - Manual intervention after all retries fail
 - Standard SLA for manual review

VIP Flow Error Handling:
 - Quick retry with minimal backoff
 - Priority queue for manual intervention
 - Expedited SLA for manual review
 - Immediate notification to administrators
 - Option for alternative processing paths

### Priority Handling Strategy
1. VIP Identification Methods
- Metadata tagging: Customer-tier flags in document headers (VIP/Regular tags)
- Content analysis: NLP detection of urgency keywords ("URGENT", "PRIORITY")
- Business rules:
    - High-value thresholds (e.g., invoices >$100k)
    - Strategic customer IDs from CRM integrations

2. Using separate queues to provide true isolation i.e. VIP documents are bever bloacked begind regular ones.
    - And can assign dedicated resources or strictier SLAs to the VIP queue.
    - Simple to implement and easy to monitor

3. priority handling strategy interacts with autoscaling and worker pools when using separate queues for VIP and regular documents:
   1. Dedicated Scaling Policies per Queue
   VIP Queue:
   - Scale out aggressively (e.g., add 1 worker per 5 messages)
   - Scale in conservatively (e.g., remove 1 worker per 15 minutes of low usage)

                Regular Queue:
                 - Scale out gradually (e.g., add 1 worker per 50 messages)
                 - Scale in aggressively (e.g., remove 1 worker per 5 minutes of low usage)
        
        2. Worker Pool Isolation
                VIP Workers:
                 - Reserved instances (always-running baseline)
                 - GPU-accelerated instances for ML tasks
                
                Regular Workers:
                - Spot instances for cost optimization
                - Scale to zero during off-peak hours

Example Workflow:
1. Document uploaded → Router assigns to VIP/regular queue
2. CloudWatch metrics trigger scaling policies:
    - VIP: 15 messages → +3 GPU workers
    - Regular: 500 messages → +10 spot workers
3. Workers process documents:
    - VIP: Guaranteed 10s SLA
    - Regular: Best-effort 2h SLA
4. During scale-in:
    - Regular workers terminated first
    - VIP workers retained until queue empty  



### Scalability Plan
1. Component-Specific Scaling Strategies
    1. Document Ingestion Layer
       Scaling Approach:
       - Serverless API Gateway (e.g., AWS API Gateway, Azure API Management) auto-scales to handle upload spikes.
       - Burst Capacity: Use pre signed URLs for direct uploads to object storage (e.g., S3) to bypass gateway bottlenecks.
       Metrics: Requests/sec, upload latency.

    2. Document Splitting Service
       Scaling Approach:
       - GPU-Optimized Kubernetes Pods: Scale GPU nodes based on page count/document size.
       - Dynamic Batch Sizing: Adjust batch sizes using queue depth (e.g., 10 pages/batch at low load → 100 pages/batch at high load).
       Metrics: Pages processed/sec, GPU memory utilization.

    3. Classification Engine
       Scaling Approach:
       - Model-Tier Autoscaling: Separate scaling groups for lightweight (CPU) vs. complex (GPU) models.
       - Warm Pool: Maintain 10-15% pre-warmed instances for VIP documents.
       Metrics: Classification latency (95th percentile), model inference throughput.

    4. Field Extraction Workers
       Scaling Approach:
        - Vertical Pod Autoscaling: Adjust CPU/memory for OCR-heavy tasks.
        - Spot Instances: Use cost-effective spot instances for non-VIP documents
          Metrics: Fields extracted/sec, OCR accuracy rate.

    5. Orchestration Layer
       Scaling Approach:
       - Sharded Workflow Engines: Split workflows by document type/customer using consistent hashing.
       - Stateful vs. Stateless Separation: Redis Cluster (state) + serverless coordinators (stateless).
       Metrics: Workflow execution time, active workflows.

    6. Queues
       Scaling Approach:
       - Kafka Partition Scaling: Add partitions dynamically during spikes.
       - SQS Auto-Scaling: VIP FIFO queue (guaranteed throughput) vs. Standard queue (best-effort).
       Metrics: Queue depth, message age.

    7. Databases
       Scaling Approach:
       - PostgreSQL: Read replicas + connection pooling.
       - Time-Series (TimescaleDB): Hypertable compression + retention policies.
       Metrics: Query latency, connection pool usage.

2. Workload Spike Handling Strategies
    1. Multi-Layer Buffering
        - Backpressure Signaling: Workers publish system_load metrics to throttle ingestion.
    2. Predictive scaling
        - Scheduled scaling: Process documents directly from s3 without local state
    3. Stateless Workers
        - Process documents directly from S3 without local state

3. Worker Scaling Policies
   De-duplication cache: For potential duplicate documents which can be avoided using Redis

| Queue Type | Scaling Trigger       | Instance Type          | Cooldown |
|------------|-----------------------|------------------------|----------|
| VIP        | >5 messages for 1m    | GPU Reserved Instances | 60s      |
| Regular    | >100 messages for 5m  | CPU Spot Instances     | 300s     |

4. Load Balancing Considerations
    1. Global vs. Service-Level LB
        - Global: Geo-DNS + Anycast (Cloudflare, AWS Global Accelerator).
        - Service-Level: Envoy for internal service routing.

    2. Adaptive Algorithms
        - VIP Documents: Weighted Least Connections (prioritize low-latency paths).
        - Regular Documents: Round-Robin with Health Checks.

    3. Circuit Breaking
        - Failure Thresholds: Bypass unhealthy workers after 3 consecutive timeouts.
        - Graceful Degradation: Shed non-VIP tasks during overload.


### Reliability & Fault Tolerance
1. Retry Policies
   1. Implementation:
   - Base Delay: Start with 1s delay, doubling after each failure (1s → 2s → 4s → ...).
   - Jitter: Add ±30% randomness to avoid synchronized retry storms.
   - Max Retries: 5 attempts for non-VIP docs, 8 for VIP.

   2. Context Awareness:
   - Retry only on transient errors (HTTP 5xx, network timeouts).
   - Skip retries for permanent errors (invalid document formats, auth failures).

2. Dead-Letter Queue (DLQ) Mechanism
   1. Implementation:
   DLQ Setup:
   - Retention period = 2x source queue retention (e.g., 14 days vs. 7 days).
   - Attach metadata: original_queue, retry_count, failure_reason.

        Reprocessing:
                - Auto-Retry: Lambda function replays messages after root cause fixes (e.g., schema updates).
                - Manual Review: Dashboard for inspecting/reprocessing VIP documents.
        
        Monitoring:
        - Alert when DLQ depth >100 messages.
        - Track ApproximateAgeOfOldestMessage for SLA compliance.

   2. Circuit Breaking & Backpressure
       I think the best solution here could be Adaptive circuit breaker and pull-based backpressure

   3. Idempotency Handling
       Idempotency Keys:
       - Generate UUIDs per document/operation.
       - Store in Redis with 24h TTL (deduplication cache as shown in design)

       Callback Handling
       - Versioned Endpoints: callback/v1/{idempotency_key}.

       Compensation:
       - Rollback API for non-idempotent operations (e.g., invoice duplicate payments).
      
- Possible Flow:


### Observability & Monitoring
1. Logging Strategy
   - AWS CloudWatch Logs: Centralized log aggregation with embedded metric filters
   - AWS X-Ray Distributed tracing for latenccy analysis across services

2. Metric to Track

| Metric              | Description                                                | Source/Tool                     |
|---------------------|------------------------------------------------------------|----------------------------------|
| ProcessingLatency   | Time per stage (splitting \→ classification \→ extraction) | AWS Step Functions Metrics      |
| ErrorRate           | Failures per stage                                         | AWS Lambda/ECS Logs             |
| QueueDepth          | Messages in VIP/Regular queues                             | AWS SQS CloudWatch Metrics      |
| SLACompliance       | % VIP docs processed within 10s                            | Custom CloudWatch Metrics       |

VIP Specific metrics:
- Age of oldest unprocessed VIP document
- Retries for VIP docs(alert if > 3)

3. Alerting strategy

| Alert Condition                     | Severity | Action                          |
|-------------------------------------|----------|---------------------------------|
| VIP\_QueueDepth \> 20 for 5m        | P1       | Auto-scale GPU workers + SMS    |
| Regular\_QueueDepth \> 100 for 15m  | P2       | Add spot instances + Slack      |
| VIP\_ProcessingLatency \> 10s       | P1       | Bypass to reserved capacity     |
| ErrorRate \> 5% (any stage)         | P2       | Trigger DLQ inspection + email  |


### Security Considerations


1. Secure Handling of Document Metadata and Payloads

The core principle here is to minimize the risk of sensitive data leakage and ensure confidentiality and integrity at every stage of document handling.

- Metadata Sanitization:  
  Before documents enter the processing pipeline, strip unnecessary metadata (such as author, software version, or embedded comments) that could reveal sensitive information or internal process details.
  This is especially important for externally sourced documents or those that may be shared with third parties.

- Payload Encryption:  
  All document payloads and metadata should be encrypted both at rest and in transit. Use a centralized key management service (like AWS KMS or Azure Key Vault) to manage encryption keys,
  ensuring that only authorized services can decrypt and access raw document data. This reduces the attack surface and supports compliance with regulations such as GDPR.

- Tamper Detection: <br/>
  To ensure documents or messages have not been altered in transit, apply cryptographic signatures (e.g., HMACs) to all payloads as they move between services.
  Each service verifies the signature before accepting the message, providing a strong guarantee against tampering.

2. Authentication and Authorization of API Endpoints

The orchestration layer must strictly control who and what can access its APIs and internal services.

- OAuth 2.0 and JWT:
  For user or external system access, rely on industry-standard protocols such as OAuth 2.0. Issue JWT tokens that encode user identity and permissions.
  Every API call must present a valid, unexpired token, which is verified at the gateway or load balancer layer. This enables fine-grained, role-based access control and auditability.

- Attribute-Based Access Control (ABAC):
  Go beyond simple role-based permissions by enforcing policies based on document attributes (e.g., type, sensitivity, department ownership).
  This allows dynamic, context-aware access decisions and supports complex enterprise security requirements.

3. Protection Against Message Tampering, Replay, and Unauthorized Task Execution

The orchestration layer must ensure that every task it executes is legitimate, unique, and unaltered.

- Immutable Logging:
  All workflow actions and message handoffs should be logged in an immutable, append-only ledger (such as Amazon QLDB or a write-once S3 bucket). This provides a cryptographically verifiable audit trail for compliance and forensics.

- Replay Prevention:
  Every message or API request should include a unique nonce and timestamp. The orchestration layer maintains a short-term cache of recently seen nonces.
  If a message arrives with a duplicate nonce or an old timestamp, it is rejected. This prevents attackers from resubmitting old (potentially harmful) requests.

- Idempotency:
  All task executions and callbacks must be idempotent. That is, if the same request is received multiple times (due to retries, network glitches, or malicious attempts), only the first execution is honored, and subsequent ones are ignored.
  This is typically enforced by tracking unique operation keys in a fast, persistent store.

### Database / Storage Design
Document Metadata

    PostgreSQL (AWS RDS)
       - Why: Relational structure ensures ACID compliance for complex queries (e.g., joins on document attributes). JSONB supports semi-structured metadata.

Task Logs & Errors

    AWS Timestream
        - Why: Time-series optimization for high-volume logs. Built-in analytics for error rate trends.

Processing Status

    Amazon DynamoDB
        - Why: Serverless scaling handles high-throughput status updates. Single-digit millisecond latency for real-time tracking.

Sub-Document Workflow Progress

    MongoDB (Atlas)
       - Why: Document model natively stores nested workflows. Flexible schema adapts to evolving sub-document structures

#### Core Entities
Document : Represents the uploaded file to be processed
- document_id
- customer_id
- original_name
- upload_time
- status
- priority
- metadata

subDocument : Represents a logical sub-document created from splitting
- subdoc_id
- document_id
- page_range
- status
- type
- extracted_fields

workflow Task : Tracks each atomic task (split, classify, extract) for a (sub-)document
- task_id
- entity_type (document or subdocument)
- entity_id (document_id or subdoc_id)
- task_type (split / classify / extract)
- attempts
- last_updated
- error_message

Priority Override : Supports dynamic workload prioritization (on-the-fly upgrades)
- override_id
- document_id
- new_priority (VIP / Regular)
- reason
- applied_at
- expires_at

Relationships & Indexing
- Document 1 - N subdocument
- Document / subdocument 1 - N workflow task
- Document 0 - N priority Override


### Bonus
1. Dynamic VIP Upgrade Implementation
    - Priority Metadata Store: Use Redis/DynamoDB to track document priorities.
    - Priority Escalation Service:
        - Listens for priority change events (API/webhook).
        - Updates document metadata + moves from regular to VIP queue.

    - In-Progress Handling:
        - For active processing: Cancel via workflow engine API, re-queue to VIP.
        - For queued documents: Reprocess via SQS message attributes.

2. Workflow Extension Strategy
   Modular Task Registration:
   [New Task] → Register in Workflow Engine → Add to Processing Graph

   Conditional Routing:
   - Post-classification: Route to translation/watermarking services via Step Functions choice states.

   Plugin Architecture:
   - Dockerized microservices for new tasks.
   - SNS/SQS topics for task-specific queues (e.g., translation-tasks).