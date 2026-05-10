# claimsprocessing

The Enterprise Claims Processing application employs a robust, highly scalable, and decoupled architecture designed to handle up to 1 billion claims per day in real time. 
 
  Event-driven Java Microservices backend combined with a Python Agentic AI processing stack
  Robust AWS data pipelines
  A rigorous 5-layer PII security model.

### End-to-End System Architecture Layers
The platform is organized into seven decoupled layers:
1. **Source Systems Layer:** Origin points for claims (web portals, mobile apps, legacy mainframes) that send JSON payloads secured via HTTPS/mTLS.
2. **API Gateway & Ingestion Layer (Java):** Handles routing, rate limiting, and OAuth2/JWT validation using Spring Cloud Gateway. It performs critical idempotency checks, schema validation, and PII masking before passing data forward.
3. **Agentic AI Processing Layer (Python):** Coordinates AI evaluation using LangGraph. It employs specialized agents (Fraud, Policy, Investigation, Settlement) powered by a multi-vendor LLM setup (SageMaker, OpenAI, Gemini) and a LangChain RAG pipeline querying policy data. 
4. **Response-Back Delivery Layer (Java):** Ensures reliable delivery of the final claim decision back to the source systems through Kafka, webhooks, or REST polling.
5. **Data Pipeline Layer (AWS Glue ELT):** Serverless Glue workflows orchestrated via EventBridge that extract, mask, transform, and load (ELT) claims data into the data lake.
6. **Data Lake Layer (Apache Iceberg + S3):** Provides ACID-compliant, schema-evolving storage. Data is partitioned across raw, standardized, and enriched zones.
7. **Analytics & MLOps Layer:** Uses Amazon Athena for serverless SQL querying, Amazon QuickSight for dashboards, and SageMaker Model Monitor for continuous ML model tracking and drift detection.

### Step-by-Step Claims Processing Flow

**1. Ingestion and Pre-Processing**
When a source system submits a claim, it passes through the API Gateway to the `ClaimIngestionService`. The service computes an idempotency key (stored in Redis) to block duplicate submissions. After schema validation, the `PiiMaskingService` synchronously intercepts the payload, replacing structured and free-text PII (like SSNs, emails, and phone numbers) with vault tokens. 

**2. Event Streaming**
The validated and PII-masked claim is then published to the `claims-submitted` topic in Amazon MSK (Kafka) using exactly-once semantics.

**3. AI Orchestration and Decisioning**
The Java `ClaimProcessingOrchestrator` consumes the event and triggers the AI layer. A LangGraph Coordinator evaluates the claim's complexity:
* **Complex/High-Value claims** are routed to an `INVESTIGATION_AGENT` utilizing models with high reasoning capabilities, like `gpt-4.1-mini`.
* **Standard claims** are processed by the `FRAUD_AGENT` and `POLICY_AGENT`.
Before interacting with any LLM, a Python `GuardrailsEngine` ensures no raw PII leaks into the prompts. Based on the agents' evaluations, the platform calculates a settlement (using the `SETTLEMENT_AGENT`) and assigns a terminal status: `APPROVED` (Straight-Through Processing), `DENIED`, `PENDING_HUMAN_REVIEW`, or `ESCALATED_SIU`.

**4. Response-Back Delivery**
Once a terminal status is reached, the `SourceSystemCallbackService` guarantees delivery of the response to the originating system using a three-channel approach:
* **Kafka:** Asynchronously publishing to the `claims-responses` topic.
* **Webhook:** Sending an HTTP POST secured by an HMAC-SHA256 signature.
* **REST Polling:** Persisting the result in Amazon DynamoDB for source systems to query via an API endpoint.

**5. ELT Data Processing and Analytics**
Concurrently, the raw data landing in S3 triggers an EventBridge rule that initiates an AWS Glue Workflow. 
* A data ingestion job reads the data, applies a final layer of PII masking (using PySpark UDFs for hashing/pattern masking), deduplicates the records, and upserts them into an **Apache Iceberg** table using a `MERGE INTO` operation.
* A subsequent enrichment job joins these processed claims with the AI decision outputs (fraud scores, net payouts) to create an analytics-ready table. 
* Finally, analysts can run partition-pruned queries via Amazon Athena to review volume, STP rates, and fraud metrics without accessing any raw PII.
