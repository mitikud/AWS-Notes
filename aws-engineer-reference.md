# The AWS Engineer's Reference — Definition · Usage · Engineer's Notes

A curated map of the AWS services an experienced engineer is expected to know. For each: a plain **definition**, its typical **usage**, and **engineer's notes** — the trade-offs, gotchas, and "why," which is what actually gets tested in interviews and burns you in production.

**How to read the tiers**

- **[CORE]** — you must know this deeply; you'll touch it on almost every project.
- **[KNOW]** — understand it well and know when to reach for it.
- **[AWARE]** — know what it is and when it would come up; details on demand.

**The mental model that ties it all together:** almost every AWS design is a combination of *compute* (where code runs) + *storage/data* (where state lives) + *networking* (how traffic flows) + *identity* (who can do what) + *observability* (how you know it works) + *automation* (how it's provisioned). If you can place any service into one of those buckets and explain its trade-off against its siblings, you're thinking like a senior AWS engineer.

---

## 1. Foundation, Identity & Global Structure

### IAM (Identity and Access Management) — [CORE]
**Definition.** The service that controls *who* (users, roles, services) can do *what* (actions) on *which* resources, via JSON policies.
**Usage.** Define roles for applications/services, attach least-privilege policies, and let services assume roles instead of using long-lived keys.
**Engineer's notes.** This is the single most important AWS skill. Master the difference between **users** (humans, avoid for apps), **roles** (assumed, temporary credentials — always prefer these), and **policies** (identity-based vs resource-based). Know the evaluation logic: explicit **deny** always wins, everything is denied by default. Real production apps carry **zero** access keys — they assume roles via ECS task roles, EC2 instance profiles, or EKS IRSA. Learn policy conditions (`aws:SourceIp`, `aws:PrincipalTag`) and the trust policy vs permission policy distinction. Most security incidents trace back to over-broad IAM (`"Action": "*"`).

### AWS Organizations — [KNOW]
**Definition.** Central management of multiple AWS accounts under one billing/governance umbrella.
**Usage.** Separate prod/staging/dev/security into distinct accounts; apply guardrails across all of them.
**Engineer's notes.** Multi-account is the modern best practice — blast-radius isolation beats one giant account. **Service Control Policies (SCPs)** set the *maximum* permissions an account can ever have (they don't grant, they cap). Pair with **AWS Control Tower** for a governed landing zone. Know that billing consolidates but security boundaries stay hard.

### Regions & Availability Zones — [CORE]
**Definition.** A Region is a geographic area; each contains multiple isolated **AZs** (physically separate datacenters).
**Usage.** Deploy across ≥2 AZs for high availability; pick regions for latency, data residency, and cost.
**Engineer's notes.** "Multi-AZ = high availability, Multi-Region = disaster recovery" is the one-liner to remember. Most services are *regional*; a few (IAM, Route 53, CloudFront, WAF) are *global*. Data transfer **between** regions and out to the internet costs money — cross-AZ traffic isn't free either. Not every service/instance type exists in every region.

### Amazon VPC (Virtual Private Cloud) — [CORE]
**Definition.** Your own isolated virtual network in AWS, with full control over IP ranges, subnets, and routing.
**Usage.** Place resources in public subnets (internet-facing) or private subnets (internal), controlling flow with route tables, security groups, and NACLs.
**Engineer's notes.** This is where most "why can't my app reach the database?" pain lives. Know cold: **public vs private subnets** (public has a route to an Internet Gateway), **NAT Gateway** (lets private subnets reach *out* without being reachable *in*), **Security Groups** (stateful, instance-level, allow-only) vs **NACLs** (stateless, subnet-level, allow+deny). Understand **VPC Endpoints** (reach S3/DynamoDB privately without going over the internet — a security and cost win). NAT Gateways are a common surprise cost. Never put a database in a public subnet.

### Route 53 — [KNOW]
**Definition.** AWS's managed DNS and domain registration service.
**Usage.** Route traffic to load balancers, CloudFront, or IPs; do health checks and failover routing.
**Engineer's notes.** More than DNS — its **routing policies** are the value: weighted (canary/blue-green), latency-based, geolocation, and failover (DR). **Alias records** point at AWS resources (ALB, CloudFront, S3) for free and update automatically — prefer them over CNAMEs at the zone apex. Health-check-based failover is a legit DR building block.

### CloudFront — [KNOW]
**Definition.** AWS's global CDN — caches content at edge locations close to users.
**Usage.** Serve static assets (from S3) and cache/accelerate dynamic APIs; terminate TLS at the edge.
**Engineer's notes.** Cuts latency and origin load, and it's a security layer (integrates with **WAF** and **Shield** for DDoS). Learn cache behaviors, TTLs, invalidations (which cost money — design cache keys instead of invalidating constantly). **Origin Access Control** locks an S3 bucket so it's *only* reachable via CloudFront, never directly. **Lambda@Edge / CloudFront Functions** run logic at the edge (header rewrites, auth).

### API Gateway — [KNOW]
**Definition.** Managed front door for APIs — handles routing, auth, throttling, and transformation.
**Usage.** Expose Lambda functions or backend services as REST/HTTP/WebSocket APIs.
**Engineer's notes.** Know the two flavors: **HTTP API** (cheaper, faster, fewer features — default choice) vs **REST API** (more features: request validation, API keys, WAF integration). It handles throttling, JWT/Cognito authorizers, and usage plans. For high-throughput internal service-to-service traffic, an ALB is often cheaper. The classic serverless pattern is API Gateway → Lambda → DynamoDB.

### Elastic Load Balancing (ALB / NLB) — [CORE]
**Definition.** Distributes incoming traffic across multiple targets in multiple AZs.
**Usage.** Front your ECS/EKS/EC2 services; health-check targets and route by path/host.
**Engineer's notes.** **ALB** (Layer 7 — HTTP/HTTPS, path/host routing, the default for web apps and containers) vs **NLB** (Layer 4 — TCP/UDP, ultra-low latency, static IPs, extreme throughput). ALB does TLS termination, sticky sessions, and integrates with Cognito/OIDC auth. Health checks are what make rolling deploys safe — get liveness/readiness right.

---

## 2. Compute

### EC2 (Elastic Compute Cloud) — [CORE]
**Definition.** Resizable virtual servers in the cloud.
**Usage.** Run anything that needs a full OS: legacy apps, custom workloads, self-managed software.
**Engineer's notes.** The foundation everything else abstracts over. Know **instance families** (general/compute/memory/storage/GPU optimized) and **purchasing models** — this is where huge money is saved: **On-Demand** (flexible, expensive), **Reserved/Savings Plans** (commit 1–3 yrs for ~40–70% off), **Spot** (up to 90% off but can be reclaimed — great for fault-tolerant/batch). **Instance profiles** give EC2 its IAM role. Use **user data** for bootstrap and bake **AMIs** (or use immutable containers) rather than configuring servers by hand. Increasingly you should reach for containers/serverless first and EC2 only when you truly need the OS.

### Auto Scaling Groups — [KNOW]
**Definition.** Automatically adds/removes EC2 instances to match demand and replace unhealthy ones.
**Usage.** Keep a service at N healthy instances; scale on CPU, request count, or custom metrics.
**Engineer's notes.** Self-healing (replaces failed instances) *and* elasticity. Understand target-tracking vs step vs scheduled scaling, and how ASGs pair with an ALB target group. Cooldowns and warm-up times prevent thrashing. The same idea reappears in ECS service auto scaling and Kubernetes HPA.

### ECS (Elastic Container Service) + Fargate — [CORE]
**Definition.** AWS-native container orchestration; **Fargate** runs those containers serverlessly (no EC2 to manage).
**Usage.** Run long-lived containerized microservices without managing a cluster's servers.
**Engineer's notes.** The simplest way to run containers on AWS. Know the pieces: **task definition** (the blueprint — image, CPU/mem, env, **task role**), **service** (keeps N tasks running behind an ALB), **cluster**. **Fargate vs EC2 launch type**: Fargate = no server management, pay per task, simpler; EC2 = cheaper at scale and more control. The **task role** is how the container gets AWS permissions — no keys in the image. This is the right default for most Spring Boot/Node microservices.

### EKS (Elastic Kubernetes Service) — [KNOW]
**Definition.** Managed Kubernetes control plane on AWS.
**Usage.** Run containers when you need the Kubernetes ecosystem, portability, or an existing k8s skill base.
**Engineer's notes.** Choose EKS over ECS when you need multi-cloud portability, the CNCF ecosystem (Helm, operators, service mesh), or your org already runs k8s. It's more powerful but far more operational overhead. **IRSA** (IAM Roles for Service Accounts) is the k8s equivalent of task roles. If you don't have a specific reason for Kubernetes, ECS/Fargate is less to manage.

### AWS Lambda — [CORE]
**Definition.** Run code without provisioning servers; billed per invocation and duration.
**Usage.** Event-driven glue: react to S3 uploads, queue messages, API calls, cron schedules.
**Engineer's notes.** The heart of serverless. Great for spiky, event-driven, short workloads; wrong for always-on high-throughput services (per-invocation cost, 15-min max, cold starts). Know **cold starts** and mitigations (provisioned concurrency, SnapStart for Java, GraalVM native, keeping the package small). It's stateless — persist state elsewhere (DynamoDB/S3). Concurrency limits and the event-source mapping (SQS/Kinesis/DynamoDB Streams) behavior matter. Watch out for "Lambda for everything" — orchestrating dozens of tiny functions can become a distributed-systems nightmare.

### Elastic Beanstalk — [AWARE]
**Definition.** PaaS that provisions and manages the infrastructure for your app from just the code.
**Usage.** Quick deploys for standard web apps without touching the underlying stack.
**Engineer's notes.** Convenient for small teams/prototypes but abstracts away control you often end up needing. Most serious shops move to ECS/EKS + IaC. Know it exists; rarely the modern choice.

### AWS Batch — [AWARE]
**Definition.** Managed batch computing that schedules and runs large-scale batch jobs.
**Usage.** Heavy parallel processing — genomics, rendering, ETL, ML preprocessing.
**Engineer's notes.** Handles queuing, dependencies, and provisioning (often on Spot to save money). Reach for it over hand-rolled job runners when you have large, independent, compute-heavy jobs.

---

## 3. Storage

### Amazon S3 (Simple Storage Service) — [CORE]
**Definition.** Virtually unlimited, highly durable object storage.
**Usage.** Store files, backups, data-lake data, static websites, logs — anything that isn't a database row.
**Engineer's notes.** Eleven 9's of durability; the backbone of countless architectures. Know **storage classes** (Standard → Intelligent-Tiering → Infrequent Access → Glacier → Deep Archive) and use **lifecycle policies** to cut cost automatically. Security essentials: **block public access** (on by default now — keep it), bucket policies, presigned URLs (share objects without making them public), and **SSE-KMS** encryption. **Versioning** + **S3 Object Lock** protect against deletes/ransomware. **VPC Gateway Endpoint** keeps S3 traffic off the internet. Costs come from storage + requests + data transfer out — the last one surprises people.

### EBS (Elastic Block Store) — [KNOW]
**Definition.** Persistent block-level disk volumes attached to EC2 instances.
**Usage.** Boot volumes and databases/filesystems that need low-latency block storage tied to one instance.
**Engineer's notes.** It's a *disk for one EC2 instance* (mostly single-attach), not shared storage. Know volume types: **gp3** (default SSD, tune IOPS/throughput independently — usually better value than gp2), **io2** (high-IOPS databases), **st1/sc1** (throughput HDD). Snapshots (incremental, stored in S3) are your backup/clone mechanism. Contrast with EFS (shared) and S3 (object).

### EFS (Elastic File System) — [AWARE]
**Definition.** Managed, elastic **shared** NFS filesystem, mountable by many instances at once.
**Usage.** Shared files across multiple EC2/containers/Lambda — CMS uploads, shared config, lift-and-shift apps.
**Engineer's notes.** Use when many compute nodes need the *same* POSIX filesystem simultaneously (EBS can't do that easily). More expensive per GB than S3/EBS — don't use it as a database or for things S3 handles better. **FSx** is the equivalent for Windows (SMB) or high-performance Lustre workloads.

### Storage Gateway / Glacier — [AWARE]
**Definition.** Hybrid on-prem-to-cloud storage bridging (Gateway); ultra-cheap cold archival (Glacier/Deep Archive).
**Usage.** Migrate/back up on-prem data to AWS; archive compliance data you rarely read.
**Engineer's notes.** Glacier is a *retrieval-time* trade-off — minutes to hours to get data back, in exchange for very low cost. Access it via S3 lifecycle transitions, not usually directly.

---

## 4. Databases

### RDS (Relational Database Service) — [CORE]
**Definition.** Managed relational databases (PostgreSQL, MySQL, MariaDB, Oracle, SQL Server).
**Usage.** Traditional relational workloads needing ACID transactions and SQL.
**Engineer's notes.** AWS manages patching, backups, and failover — you still design schemas and tune queries. **Multi-AZ** = synchronous standby for HA (automatic failover, *not* for read scaling). **Read replicas** = async copies for *read* scaling. Know the difference cold — it's a classic question. Prefer **IAM database authentication** or Secrets Manager over static passwords. Right-size the connection pool to the instance's `max_connections`. Use **Flyway/Liquibase** for migrations, never `ddl-auto=update` in prod.

### Aurora — [KNOW]
**Definition.** AWS's cloud-native relational engine, MySQL/PostgreSQL-compatible, with a distributed storage layer.
**Usage.** RDS workloads that need more performance, faster failover, and auto-scaling storage.
**Engineer's notes.** Storage auto-grows; up to 15 low-lag read replicas; failover in seconds. **Aurora Serverless v2** scales capacity automatically for spiky/unpredictable loads. **Global Database** gives cross-region replication for DR/low-latency reads. Costs more than plain RDS but often worth it at scale. Same SQL, better ops.

### DynamoDB — [CORE]
**Definition.** Fully managed, serverless NoSQL key-value/document database with single-digit-ms latency at any scale.
**Usage.** High-scale, access-pattern-driven workloads: user profiles, sessions, carts, IoT, event data.
**Engineer's notes.** You model around **access patterns**, not entities — get the partition key wrong and you hot-partition. Learn **single-table design**, GSIs/LSIs, and why **Scan** is a red flag (it reads everything). **On-demand** vs **provisioned** capacity (with auto scaling). **DynamoDB Streams** enable event-driven reactions to data changes. **DAX** adds a microsecond cache. TTL auto-expires items. It scales effortlessly *if* you design for it — the design is the hard part.

### ElastiCache (Redis / Memcached) — [KNOW]
**Definition.** Managed in-memory cache/store.
**Usage.** Caching, sessions, leaderboards, rate limiting, pub/sub, distributed locks.
**Engineer's notes.** Redis is the default (richer data types, persistence, replication, cluster mode); Memcached only for simple multi-threaded caching. Always set **TTLs**; design for cache misses (fall through to source of truth); beware the **thundering herd** when a hot key expires. Cluster mode vs non-cluster changes client config. It's just managed Redis — your Redis skills transfer directly.

### Redshift — [AWARE]
**Definition.** Petabyte-scale columnar data warehouse for analytics (OLAP).
**Usage.** Complex analytical queries and BI over huge historical datasets.
**Engineer's notes.** **OLAP, not OLTP** — for aggregations across billions of rows, not transactional app traffic. Columnar storage + massively parallel processing. These days many teams prefer **S3 + Athena** (query-in-place, serverless) for flexibility, reaching for Redshift when they need consistent warehouse performance. Know the OLTP vs OLAP distinction — it's fundamental.

### DocumentDB / Neptune / Timestream / Keyspaces — [AWARE]
**Definition.** Purpose-built databases: **DocumentDB** (MongoDB-compatible document), **Neptune** (graph), **Timestream** (time-series), **Keyspaces** (Cassandra-compatible).
**Usage.** Reach for the shape of data: documents, relationships/graphs, metrics over time, wide-column.
**Engineer's notes.** The AWS philosophy is "**purpose-built databases**" — pick the engine that matches the data model rather than forcing everything into one DB. Know these exist and roughly when each fits; deep expertise is role-specific.

### Amazon OpenSearch — [KNOW]
**Definition.** Managed search and analytics engine (the Elasticsearch fork) with Kibana-style dashboards.
**Usage.** Full-text search, log analytics, observability dashboards.
**Engineer's notes.** The go-to for search and centralized log analysis. Increasingly used for **vector search** in RAG/AI systems. It's operationally heavy (cluster sizing, shard management) — right-size carefully. Often paired with a relational/vector combo for hybrid search + re-ranking.

---

## 5. Messaging, Streaming & Integration

### SQS (Simple Queue Service) — [CORE]
**Definition.** Fully managed message queue that decouples producers from consumers.
**Usage.** Buffer work for async processing so slow tasks survive restarts and traffic spikes.
**Engineer's notes.** The workhorse of decoupling. **Standard** (high throughput, at-least-once, best-effort order) vs **FIFO** (exactly-once-ish, strict order, lower throughput). Always configure a **Dead Letter Queue** with `maxReceiveCount`. Consumers must be **idempotent** (at-least-once means duplicates happen). Get `visibilityTimeout` > processing time. Long polling saves money over short polling.

### SNS (Simple Notification Service) — [CORE]
**Definition.** Managed pub/sub — publish once, deliver to many subscribers.
**Usage.** Fan-out events to multiple consumers (SQS queues, Lambda, email, SMS, HTTP).
**Engineer's notes.** The **SNS → multiple SQS** fan-out is *the* canonical event-driven pattern — durable broadcast where each consumer retries independently. Use **message filter policies** so subscribers only get relevant events. SNS vs SQS: broadcast/push vs queue/pull. Combine them rather than choosing.

### EventBridge — [KNOW]
**Definition.** Serverless event bus with content-based routing, scheduling, and SaaS integrations.
**Usage.** Route domain/AWS/SaaS events to targets; run cron-scheduled jobs.
**Engineer's notes.** Where SNS is dumb fan-out, EventBridge routes on event *content* (rules matching JSON), supports **schemas**, third-party sources, and **scheduled rules** (cron). Prefer it for decoupled microservice event routing and scheduling; prefer SNS for simple, high-throughput, low-cost fan-out. The default event bus already carries AWS service events you can react to.

### Kinesis — [KNOW]
**Definition.** Real-time streaming for high-volume, ordered, replayable data.
**Usage.** Clickstreams, telemetry, logs, event sourcing, real-time analytics.
**Engineer's notes.** Unlike SQS (delete-after-consume), Kinesis **retains** records and supports **multiple independent consumers** replaying the same stream. Throughput scales with **shards**; a bad partition key creates a hot shard. Watch `IteratorAge` for lagging consumers. **Firehose** is the "just dump the stream into S3/Redshift/OpenSearch" no-code variant. SQS = task queue; Kinesis = ordered replayable stream.

### Amazon MSK (Managed Kafka) — [AWARE]
**Definition.** Managed Apache Kafka.
**Usage.** When you specifically need Kafka's ecosystem, semantics, or existing Kafka apps.
**Engineer's notes.** Choose MSK over Kinesis when you need real Kafka (Connect, Streams, exactly-once, existing tooling) or portability. More operational surface than Kinesis, which is the AWS-native, lower-ops streaming default. If your team already knows Kafka deeply, MSK removes broker management while keeping the API.

### Step Functions — [KNOW]
**Definition.** Serverless workflow orchestration — coordinate multiple services as a state machine.
**Usage.** Multi-step business processes: order fulfilment, ETL pipelines, approval flows, saga orchestration.
**Engineer's notes.** The right answer when you're tempted to chain Lambdas manually. Gives you retries, error handling, parallel branches, and visual execution history *declaratively* (JSON/ASL). **Standard** (long-running, auditable) vs **Express** (high-volume, short) workflows. It's the AWS-native way to implement the **saga pattern** for distributed transactions.

### SES (Simple Email Service) — [KNOW]
**Definition.** Managed email sending (and receiving) at scale.
**Usage.** Transactional email — receipts, password resets, notifications.
**Engineer's notes.** New accounts start in a **sandbox** (verified recipients only) until you request production access. Deliverability depends on **SPF/DKIM/DMARC** and handling **bounces/complaints** via SNS — ignoring these ruins sender reputation. Cheap and reliable for transactional mail.

### Amazon MQ — [AWARE]
**Definition.** Managed ActiveMQ/RabbitMQ.
**Usage.** Lift-and-shift apps that already speak AMQP/JMS/MQTT.
**Engineer's notes.** Use it for migrations of existing message-broker apps that need standard protocols. For *new* AWS-native work, SQS/SNS/EventBridge are cheaper and lower-ops.

---

## 6. Security & Compliance

### KMS (Key Management Service) — [CORE]
**Definition.** Managed creation and control of encryption keys.
**Usage.** Encrypt data at rest across S3/EBS/RDS and application-level secrets.
**Engineer's notes.** Central to "encryption at rest" — most services just reference a KMS key. Learn **envelope encryption** (KMS generates a data key; you encrypt locally; the 4KB direct-encrypt limit forces this for larger payloads). Enable automatic key rotation; scope IAM to specific key ARNs. **Customer-managed keys** give you control and audit over AWS-managed ones.

### Secrets Manager — [CORE]
**Definition.** Stores, retrieves, and **rotates** secrets (DB creds, API keys).
**Usage.** Keep credentials out of code/config; auto-rotate database passwords.
**Engineer's notes.** Choose over Parameter Store when you need **automatic rotation**. Costs per secret + per API call — cache at startup, don't fetch per request. Grant `GetSecretValue` on the specific ARN. The whole point is that no human ever sees or hardcodes the password.

### Systems Manager Parameter Store — [KNOW]
**Definition.** Hierarchical config/secret storage (SecureString for encrypted values).
**Usage.** Feature flags, tunables, environment config, cheaper secrets.
**Engineer's notes.** Cheaper than Secrets Manager and great for plain config, but **no built-in rotation**. Use Parameter Store for config, Secrets Manager for anything that must rotate. Knowing exactly when to pick each is a senior signal.

### Cognito — [KNOW]
**Definition.** Managed user directory and OIDC/OAuth2 identity provider.
**Usage.** App user sign-up/sign-in, MFA, social login, JWT issuance.
**Engineer's notes.** Distinguish **User Pools** (authentication — who the user is, issues JWTs) from **Identity Pools** (authorization — temporary IAM creds for AWS resources). Treat it as a standard OIDC provider and validate JWTs with your framework's resource-server support. Validate `token_use` (access vs id). Powerful but its console UX and edge cases have a learning curve.

### IAM Identity Center (SSO) — [AWARE]
**Definition.** Centralized workforce single sign-on across AWS accounts and SaaS apps.
**Usage.** Give humans temporary, role-based access across an Organization — no per-account IAM users.
**Engineer's notes.** The modern way humans should access AWS: federated, temporary credentials, no long-lived keys. Pairs with Organizations. Replaces the old pattern of IAM users per account.

### WAF · Shield · GuardDuty · Security Hub · Inspector · Macie — [KNOW/AWARE]
**Definition.** The security-tooling layer: **WAF** (web app firewall — filter malicious HTTP), **Shield** (DDoS protection), **GuardDuty** (threat detection from logs), **Security Hub** (aggregated security posture), **Inspector** (vulnerability scanning), **Macie** (sensitive-data discovery in S3).
**Usage.** Layered defense in depth around your apps and accounts.
**Engineer's notes.** Know **WAF** well (rate limiting, SQLi/XSS rules, attach to CloudFront/ALB/API Gateway) and **GuardDuty** as the always-on threat detector. The rest: know what each does so you can name the right tool. Security is a shared-responsibility model — AWS secures the cloud, *you* secure what's *in* it.

### ACM (Certificate Manager) — [KNOW]
**Definition.** Free, auto-renewing TLS/SSL certificates for AWS resources.
**Usage.** HTTPS on ALB, CloudFront, API Gateway.
**Engineer's notes.** Free and auto-renewing when used with AWS-integrated services — never manually manage certs for those again. Certs for CloudFront must live in `us-east-1`. Can't export the private key (by design), so it's AWS-integrations only.

---

## 7. Observability & Governance

### CloudWatch — [CORE]
**Definition.** The central service for metrics, logs, alarms, and dashboards.
**Usage.** Monitor everything; alert on thresholds; centralize logs.
**Engineer's notes.** Your primary observability tool. Know **Metrics** (built-in + custom via Micrometer/EMF), **Logs** (+ Logs Insights for querying — set retention or pay forever), **Alarms** (→ SNS → your pager), and **Dashboards**. Watch metric **cardinality** (unbounded tags = surprise bills). Emit **structured JSON logs** so they're queryable. Alarms are only useful when wired to a real notification path.

### CloudTrail — [KNOW]
**Definition.** Records every API call made in your account — the audit log of AWS.
**Usage.** Security forensics, compliance, "who deleted that resource?"
**Engineer's notes.** *CloudWatch = how your app behaves; CloudTrail = who did what to your account.* Keep it on in every account, deliver logs to a locked-down central S3 bucket, and never let anyone tamper with it. First place you look after an incident.

### X-Ray — [AWARE]
**Definition.** Distributed tracing across microservices.
**Usage.** Find latency bottlenecks and failures across a request's path.
**Engineer's notes.** Essential once you have many services calling each other — see the full trace, not just one service's logs. Increasingly people use **OpenTelemetry** as the vendor-neutral instrumentation layer feeding X-Ray or other backends.

### AWS Config — [AWARE]
**Definition.** Tracks resource configurations over time and evaluates them against rules.
**Usage.** Compliance auditing, drift detection, "was this ever public?"
**Engineer's notes.** Governance/compliance tool — records the *state* of resources and flags non-compliant ones (e.g. unencrypted volumes, open security groups). Complements CloudTrail (which records *actions*).

### Systems Manager (SSM) — [KNOW]
**Definition.** A suite for operating fleets: Parameter Store, Session Manager, Patch Manager, Run Command, Automation.
**Usage.** Manage servers without SSH, patch fleets, store config, run ops automation.
**Engineer's notes.** **Session Manager** lets you get a shell on an instance with *no SSH keys, no open port 22, no bastion* — a big security win, fully audited. Underrated; know it exists and reach for it instead of opening SSH.

---

## 8. Infrastructure as Code & DevOps

### CloudFormation / CDK — [CORE]
**Definition.** Define AWS infrastructure as code — **CloudFormation** (declarative YAML/JSON templates), **CDK** (real code — TypeScript/Python/Java — that synthesizes to CloudFormation).
**Usage.** Provision and version every resource reproducibly; no click-ops in production.
**Engineer's notes.** IaC is non-negotiable at senior level. CDK is loved because you get loops, types, and abstraction in a real language. Understand **stacks**, **drift**, and rollback behavior. In the wider market **Terraform** is at least as common (multi-cloud, huge ecosystem) — know both worlds. The rule: if a resource exists, it should exist *in code*.

### CodePipeline / CodeBuild / CodeDeploy — [KNOW]
**Definition.** AWS-native CI/CD: orchestration, build, and deployment.
**Usage.** Build → test → deploy pipelines entirely within AWS.
**Engineer's notes.** Know the concepts even if your shop uses **GitHub Actions/GitLab** (very common). **CodeDeploy** does the interesting part: **blue/green** and **canary** deployments to ECS/Lambda/EC2 with automatic rollback on alarm. The pattern matters more than the specific tool.

### ECR (Elastic Container Registry) — [KNOW]
**Definition.** Managed private Docker image registry.
**Usage.** Store the container images your ECS/EKS tasks run.
**Engineer's notes.** Turn on **image scanning** for vulnerabilities and lifecycle policies to prune old images (they cost storage). Cross-account/cross-region replication for multi-account setups.

---

## 9. Data, Analytics & AI (know the landscape)

### Glue · Athena · EMR · Lake Formation · QuickSight — [AWARE]
**Definition.** The analytics stack: **Glue** (serverless ETL + data catalog), **Athena** (serverless SQL over S3), **EMR** (managed Spark/Hadoop), **Lake Formation** (data-lake governance), **QuickSight** (BI dashboards).
**Usage.** Build data lakes and run analytics without standing up warehouses.
**Engineer's notes.** The modern pattern is a **data lake on S3** queried in place: land data in S3, catalog it with Glue, query with **Athena** (pay-per-scan SQL — cheap and serverless), visualize with QuickSight. Athena + S3 has displaced a lot of Redshift for flexible analytics. Know this pipeline shape even if analytics isn't your focus.

### Bedrock · SageMaker — [KNOW]
**Definition.** **Bedrock** (serverless access to foundation models — Anthropic Claude, etc. — via API) and **SageMaker** (full ML platform: build/train/deploy your own models).
**Usage.** Bedrock for GenAI features (RAG, chat, summarization) without hosting models; SageMaker for custom ML.
**Engineer's notes.** Given the market, backend engineers increasingly ship AI features via **Bedrock** — managed FMs, **Knowledge Bases** for RAG, **Guardrails**, and **Agents**, all behind an API with IAM auth (no separate keys). It's the AWS-native equivalent of calling an LLM API, with data staying in your account. Worth real familiarity now; SageMaker is deeper ML-engineering territory.

---

## 10. Cost & Account Management

### Cost Explorer · Budgets · Savings Plans — [KNOW]
**Definition.** **Cost Explorer** (visualize/analyze spend), **Budgets** (alert on spend thresholds), **Savings Plans/Reserved Instances** (commit for big discounts).
**Usage.** Understand, forecast, alert on, and reduce the bill.
**Engineer's notes.** Cost awareness *is* an engineering skill at senior level. Set budget alarms early. Know the big cost traps: **data transfer out** and **cross-AZ/cross-region** traffic, idle NAT Gateways, un-lifecycled S3 and logs, over-provisioned instances, and forgotten resources. **Tag everything** for cost allocation. Right-sizing and Savings Plans routinely cut bills 30–50%.

---

## How to prioritize your learning

If you're building toward "experienced AWS engineer," depth beats breadth in this order:

1. **IAM + VPC + networking** — everything else assumes you understand identity and how packets flow. This is where most people are weak and where senior/junior separates.
2. **Compute + containers** — EC2 fundamentals, then ECS/Fargate (and Lambda for events). Know *when* to pick each.
3. **Core data services** — S3, RDS/Aurora, DynamoDB, ElastiCache — and the trade-offs between them.
4. **Messaging & event-driven patterns** — SQS/SNS/EventBridge/Kinesis and how to combine them (fan-out, DLQs, idempotency, saga via Step Functions).
5. **Security & observability** — KMS, Secrets Manager, Cognito, CloudWatch, CloudTrail. Non-negotiable in production.
6. **IaC** — CloudFormation/CDK *and* Terraform. If it's not in code, it doesn't count.
7. **Cost** — because a design that works but bankrupts the team isn't a good design.

## The questions that reveal experience

Experienced engineers can answer the *"which one and why"* questions instantly:

- Multi-AZ vs Multi-Region · Multi-AZ vs read replicas (HA vs read scaling)
- SQS vs SNS vs Kinesis vs EventBridge vs MSK
- ECS/Fargate vs EKS vs Lambda vs EC2
- DynamoDB vs RDS · Redshift vs Athena (OLAP vs OLTP, warehouse vs query-in-place)
- Secrets Manager vs Parameter Store (rotation vs cost)
- Security Group vs NACL (stateful/instance vs stateless/subnet)
- ALB vs NLB (Layer 7 vs Layer 4)
- Why production apps carry **zero** long-lived keys (roles everywhere)
- How you make consumers **idempotent** under at-least-once delivery
- Where the money leaks (data transfer, NAT, idle resources, un-lifecycled storage)

Master the trade-offs, not just the definitions — that's the whole difference.
