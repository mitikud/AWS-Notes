# OrderFlow — Full-Platform Reference Architecture

*Utilizing the full AWS service catalog from the engineer's reference, with every service given a justified role.*

## The honest premise

Real production systems don't use 50 AWS services — many are alternatives (Redshift *or* Athena, ECS *or* EKS, DynamoDB *or* DocumentDB, Kinesis *or* MSK). Using all of them in one app "for the sake of it" is bad architecture and interviewers know it.

So OrderFlow is deliberately expanded into a **realistic multi-domain platform** where nearly every service earns a genuine place. Where two services overlap, I either (a) split the domain so each does the job it's actually best at, or (b) flag it clearly with **[ALT]** — "this is the alternative you'd choose instead of X; included so you learn it, not because a sane system runs both."

**Legend:** [CORE] used everywhere · [ROLE] owns a real sub-domain · [ALT] alternative-to-another, learning inclusion · [OPS] platform/governance concern.

---

## The expanded OrderFlow platform

Seven domains, each a bounded context:

| Domain | Responsibility |
|---|---|
| **Storefront** | Browse catalog, search, reviews, recommendations, cart |
| **Orders** | Checkout, payments, order lifecycle (relational, ACID) |
| **Fulfilment** | Async processing, invoices, shipping, notifications |
| **Analytics** | Clickstream, data lake, warehouse, BI dashboards |
| **AI** | Search embeddings, support chatbot, description/review generation, fraud & reco models |
| **Admin** | Internal back-office, catalog management, reporting |
| **Platform** | Networking, identity, security, observability, CI/CD, cost — cross-cutting |

---

## Master service map

Every service, its role in OrderFlow, and an honesty flag. Grouped by layer.

### Foundation · Identity · Networking

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **IAM** | Every service/task assumes a scoped role; zero long-lived keys anywhere | [CORE] |
| **Organizations** | Separate accounts: `prod`, `staging`, `dev`, `security`, `analytics`; SCP guardrails | [OPS] |
| **Regions/AZs** | Primary `us-east-1` across 3 AZs; DR in `us-west-2` | [CORE] |
| **VPC** | Public subnets (ALB/NAT), private subnets (ECS, RDS, cache), endpoints for S3/DynamoDB | [CORE] |
| **Route 53** | `orderflow.com` DNS; latency routing + failover to DR region | [ROLE] |
| **CloudFront** | CDN for the SPA + product images from S3; edge TLS; WAF attach point | [ROLE] |
| **API Gateway** | Public HTTP API front door for SPA/mobile → routes to backend services | [ROLE] |
| **ALB / NLB** | ALB fronts each ECS service (L7 routing, health checks); NLB for internal gRPC | [CORE] |
| **ACM** | Auto-renewing TLS certs on CloudFront, ALB, API Gateway | [CORE] |

### Compute

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **ECS + Fargate** | The always-on microservices: catalog, orders, fulfilment, notification, admin | [CORE] |
| **Lambda** | Event glue: S3-triggered thumbnail/receipt generation, DynamoDB-stream processors, Cognito triggers, scheduled tasks | [CORE] |
| **Auto Scaling** | ECS service auto-scaling on CPU/queue-depth; EC2 ASG for the Batch fleet | [CORE] |
| **AWS Batch** | Nightly bulk jobs: catalog image re-processing, data-lake compaction, reco training-data prep | [ROLE] |
| **EC2** | The compute environment *under* AWS Batch (Spot fleet) — where the heavy batch containers actually run | [ROLE] |
| **EKS** | The shared **data-science/ML platform** cluster (SageMaker pipelines + custom training) — chosen for k8s ecosystem, separate from the app tier | [ALT] |
| **Elastic Beanstalk** | The internal marketing microsite — a small standalone app where PaaS convenience is fine | [ALT] |

---

### Storage

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **S3** | Invoices, product images, static SPA hosting, the **data lake**, log archive, backups | [CORE] |
| **EBS** | Volumes for the EC2/Batch fleet workers | [ROLE] |
| **EFS** | Shared POSIX filesystem: invoice/PDF templates + shared ML model artifacts mounted by several containers | [ROLE] |
| **Glacier** | S3 lifecycle target — invoices & compliance records aged past 90 days | [ROLE] |
| **Storage Gateway** | One-time migration bridge for legacy on-prem order history into S3 | [ALT] |
| **FSx** | FSx for Windows backing a legacy Windows accounting integration (SMB) | [ALT] |

### Databases

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **Aurora (RDS)** | Orders, payments, ledger — ACID relational core (Aurora PostgreSQL, Multi-AZ + read replicas) | [CORE] |
| **DynamoDB** | Product catalog, carts, sessions, **idempotency keys**; Streams feed Lambda | [CORE] |
| **ElastiCache (Redis)** | Catalog cache, sessions, rate limiting, distributed locks | [CORE] |
| **OpenSearch** | Product full-text search, log analytics, and **vector search** for the AI layer | [ROLE] |
| **Redshift** | The analytics **warehouse** for structured BI (sales, cohorts) — Spectrum reads the S3 lake | [ROLE] |
| **DocumentDB** | Product **reviews** and rich CMS content (document-shaped, MongoDB API) | [ALT] |
| **Neptune** | Recommendation **graph** ("bought-together") and fraud-ring detection | [ROLE] |
| **Timestream** | Operational **time-series**: order-rate, latency, and IoT warehouse-sensor metrics | [ROLE] |
| **Keyspaces** | High-write raw event landing (Cassandra API) — [ALT] to a Kinesis→S3 landing; learn the wide-column model | [ALT] |

### Messaging · Streaming · Integration

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **SQS** | Async fulfilment queues + DLQs; idempotent consumers | [CORE] |
| **SNS** | Fan-out `OrderPlaced` to fulfilment/analytics/notification queues; ops alerts | [CORE] |
| **EventBridge** | Domain event bus + scheduled (cron) jobs + SaaS (shipping carrier) integration | [CORE] |
| **Kinesis** | Real-time **clickstream** ingestion for analytics/AI | [ROLE] |
| **MSK (Kafka)** | **CDC pipeline**: Debezium streams Aurora changes → MSK → data lake (distinct from clickstream) | [ROLE] |
| **Step Functions** | Fulfilment **saga** and refund workflow orchestration (retries, compensation) | [ROLE] |
| **SES** | Transactional email: confirmations, receipts, password resets | [CORE] |
| **Amazon MQ** | JMS/AMQP bridge to a legacy partner system that can't speak SQS | [ALT] |

### Security

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **KMS** | Envelope-encrypt PII/card tokens; keys behind S3/RDS/EBS encryption | [CORE] |
| **Secrets Manager** | DB creds + payment API keys, auto-rotated | [CORE] |
| **Parameter Store** | Feature flags, business tunables, non-rotating config | [CORE] |
| **Cognito** | Customer authentication (User Pools → JWT); resource-server validation in each API | [CORE] |
| **IAM Identity Center** | Workforce SSO for engineers/admins across all accounts | [OPS] |
| **WAF** | Rules on CloudFront + API Gateway (rate-limit, SQLi/XSS) | [ROLE] |
| **Shield** | DDoS protection (Advanced) on the edge | [OPS] |
| **GuardDuty** | Continuous threat detection across accounts | [OPS] |
| **Security Hub** | Aggregated security posture / findings | [OPS] |
| **Inspector** | Vulnerability scanning of ECR images and EC2 | [OPS] |
| **Macie** | Scans the S3 data lake for exposed PII | [OPS] |

### Observability · Governance

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **CloudWatch** | Metrics, logs, alarms, dashboards for every service | [CORE] |
| **CloudTrail** | Account-wide API audit → locked central S3 bucket | [CORE] |
| **X-Ray** | Distributed tracing across the microservices | [ROLE] |
| **Config** | Compliance rules + drift detection (e.g. "no unencrypted volumes") | [OPS] |
| **Systems Manager** | Session Manager (keyless shell), Patch Manager, Parameter Store, Automation runbooks | [ROLE] |

### IaC · DevOps

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **CloudFormation / CDK** | Every resource defined as code (CDK in TypeScript) | [CORE] |
| **CodePipeline / Build / Deploy** | CI/CD: build → test (LocalStack) → push ECR → blue/green to ECS | [ROLE] |
| **ECR** | Private image registry with scan-on-push + lifecycle pruning | [CORE] |

### Data · Analytics · AI

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **Glue** | Serverless ETL + the data **catalog** over the S3 lake | [ROLE] |
| **Athena** | Ad-hoc SQL over the S3 lake (pay-per-scan) — [ROLE] alongside Redshift, for exploratory queries | [ROLE] |
| **EMR** | Heavy Spark jobs: reco feature engineering on large history | [ROLE] |
| **Lake Formation** | Data-lake permissions/governance over the S3 lake | [OPS] |
| **QuickSight** | BI dashboards on Redshift/Athena for ops & business | [ROLE] |
| **Bedrock** | GenAI: RAG **support chatbot**, product-description generation, review summarization, search embeddings | [ROLE] |
| **SageMaker** | Custom **fraud** and **recommendation** model training + hosted inference | [ROLE] |

### Cost Management

| Service | Role in OrderFlow | Flag |
|---|---|---|
| **Cost Explorer / Budgets / Savings Plans** | Spend visibility, budget alarms, committed-use discounts on Fargate/EC2 | [OPS] |

---

## The architecture in layers

Think of the platform as horizontal bands, with security and observability cutting vertically through all of them.

**1. Edge layer** — A user hits `orderflow.com`. **Route 53** resolves to **CloudFront**, which serves the SPA and cached product images from **S3**, with **ACM** TLS and **WAF** + **Shield** filtering malicious/DDoS traffic at the edge. API calls go to **API Gateway**.

**2. Identity layer** — **Cognito** authenticates customers and issues JWTs; every backend service validates them as an OIDC resource server. Engineers reach AWS through **IAM Identity Center** SSO. Every workload's own permissions come from an **IAM** role — never keys.

**3. Application layer** — Inside the **VPC** (private subnets), **ECS/Fargate** runs the microservices behind **ALB**s, auto-scaled by **Auto Scaling**. **Lambda** handles event-driven glue. Config comes from **Parameter Store**; secrets from **Secrets Manager**; sensitive fields are encrypted with **KMS**.

**4. Data layer** — **Aurora** owns orders/payments (ACID). **DynamoDB** owns catalog/carts/sessions. **ElastiCache** caches hot reads. **OpenSearch** powers search. **DocumentDB** holds reviews, **Neptune** the reco graph, **Timestream** operational metrics. **VPC endpoints** keep S3/DynamoDB traffic off the internet.

**5. Event & streaming layer** — Orders publish to **SNS**, fanning out to **SQS** queues consumed by fulfilment/analytics/notification. **EventBridge** routes domain events and runs scheduled jobs. **Kinesis** ingests clickstream; **MSK** carries database CDC. **Step Functions** orchestrates the multi-step fulfilment saga.

**6. Analytics & AI layer** — Streams and CDC land in the **S3** data lake, cataloged by **Glue**, governed by **Lake Formation**. **Athena** queries it ad-hoc; **Redshift** serves the BI warehouse; **EMR**/**Batch** run heavy Spark jobs; **QuickSight** visualizes. **Bedrock** powers the RAG chatbot and content generation; **SageMaker** trains fraud/reco models served back to the app.

**7. Platform layer (cross-cutting)** — **CloudWatch** (metrics/logs/alarms), **CloudTrail** (audit), **X-Ray** (tracing), **Config** (compliance), **GuardDuty/Security Hub/Inspector/Macie** (security posture), **Systems Manager** (ops). Everything is provisioned by **CDK/CloudFormation** through **CodePipeline** into **ECR**, with **Cost Explorer/Budgets** watching spend.

---

## End-to-end walkthroughs

Following a real request through the services makes the architecture concrete — and these are exactly the "walk me through what happens when…" interview questions.

### A) Customer places an order
1. SPA (served by **CloudFront**/**S3**) calls **API Gateway** with a **Cognito** JWT; **WAF** vets the request.
2. API Gateway → **ALB** → the **orders** service on **ECS/Fargate**.
3. Orders validates the JWT, reads the cart from **DynamoDB**, checks a hot product price from **ElastiCache**, and writes the order to **Aurora** in a transaction. Card data is tokenized and the token encrypted with **KMS**; the payment key comes from **Secrets Manager**.
4. Orders publishes `OrderPlaced` to **SNS**.
5. Fan-out: a fulfilment **SQS** queue, an analytics queue, and a notification queue each receive it.
6. **Step Functions** runs the fulfilment saga: a **Lambda** generates the invoice PDF → stored in **S3** → an S3 event triggers a receipt **Lambda** → **SES** emails the customer a presigned link.
7. Meanwhile the clickstream/event flows into **Kinesis**, and Aurora's change is captured via **MSK** (CDC) — both landing in the **S3** lake.
8. **CloudWatch** records metrics/logs throughout; **X-Ray** stitches the trace; **CloudTrail** logs every AWS API call.

### B) Analytics & BI
Clickstream (**Kinesis**) + CDC (**MSK**) → **S3** lake → **Glue** catalogs it (**Lake Formation** governs access) → **Athena** for exploration, **EMR**/**Batch** for heavy transforms, **Redshift** for the curated warehouse → **QuickSight** dashboards for the business.

### C) AI support query
Customer asks the support chatbot a question → **Bedrock** embeds it → semantic search over **OpenSearch** vectors (indexed from order/FAQ data) → **Bedrock** generates a grounded RAG answer with guardrails. Separately, **SageMaker** scores each order for fraud and serves recommendations (built from the **Neptune** graph) back into the storefront.

---

## Where services overlap — what you'd *really* pick

Be ready to defend these. Using both in OrderFlow is a *learning* choice; in a lean real system you'd pick one:

- **Kinesis vs MSK** — I split them (clickstream vs CDC), but many teams run one. Pick Kinesis for AWS-native simplicity, MSK if you need real Kafka.
- **Redshift vs Athena** — split (warehouse vs ad-hoc). A lean start is often just Athena on S3; add Redshift when you need consistent warehouse performance.
- **DynamoDB vs DocumentDB** — both are NoSQL; DocumentDB here only because reviews are document-shaped. Most teams would keep reviews in DynamoDB or Aurora and skip DocumentDB.
- **ECS vs EKS** — the app runs on ECS/Fargate; EKS is only the separate ML platform. Don't run both unless you genuinely have two org units with different needs.
- **Elastic Beanstalk, Amazon MQ, FSx, Storage Gateway, Keyspaces** — all **[ALT]**/legacy inclusions. In a greenfield build you'd likely use none of them.
- **Batch vs EMR vs Glue** — three ways to run big jobs. Glue for serverless ETL, EMR for heavy custom Spark, Batch for arbitrary containerized compute. Pick per job; you rarely need all three.

Saying *"I included these to demonstrate breadth, but here's the leaner real design"* is exactly the judgment interviewers want to hear.

---

## A realistic build plan

Don't build 45 services at once. Grow OrderFlow in slices, each independently demoable:

**Phase 1 — Core commerce (the spine).** VPC, IAM, ECS/Fargate, ALB, Aurora, DynamoDB, S3, Cognito, ElastiCache, ACM, CloudWatch, CDK. *You can place and view orders.*

**Phase 2 — Event-driven fulfilment.** SNS, SQS (+DLQ), EventBridge, Step Functions, Lambda, SES, Secrets Manager, Parameter Store, KMS. *Orders now process asynchronously and email receipts.*

**Phase 3 — Search, reviews, reco.** OpenSearch, DocumentDB, Neptune, Bedrock (embeddings + chatbot). *Storefront gets real search and recommendations.*

**Phase 4 — Analytics & AI platform.** Kinesis, MSK (CDC), Glue, Lake Formation, Athena, Redshift, EMR/Batch, EC2/EBS, QuickSight, Timestream, SageMaker, EKS. *You have a data lake, warehouse, and models.*

**Phase 5 — Production hardening.** WAF, Shield, GuardDuty, Security Hub, Inspector, Macie, Config, CloudTrail, X-Ray, Systems Manager, IAM Identity Center, Organizations, CodePipeline/ECR, Cost Explorer/Budgets, Route 53 DR, Glacier/EFS/FSx/Storage Gateway/MQ/Keyspaces/Beanstalk. *Secured, observable, multi-account, cost-governed.*

Each phase is a complete story you can put in a portfolio and talk through in an interview — which is worth far more than a half-finished monolith touching everything at once.

---

## Cost reality check

A platform this size would cost real money if left running. For learning: build phase by phase on **LocalStack** where possible, use **Free Tier** and **Spot**, tear resources down nightly (Terraform/CDK `destroy`), set a **Budget alarm** at a low threshold on day one, and never leave a NAT Gateway, Redshift cluster, OpenSearch domain, or MSK cluster running overnight — those are the silent bill-killers.
