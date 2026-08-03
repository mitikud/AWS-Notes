# AWS for Spring Boot Engineers — A Complete, Project-Based Tutorial

A hands-on path through the AWS services that actually matter for a senior Java/Spring Boot backend engineer. Instead of 200 disconnected snippets, you build **one professional system** — `OrderFlow`, an order-processing platform — and each AWS service slots into it the way it would in production.

**Target stack**

| Component | Version |
|---|---|
| Java | 21 (LTS) |
| Spring Boot | 3.5.x |
| Spring Cloud AWS | 3.4.2 (`io.awspring.cloud`) |
| AWS SDK for Java | v2 (managed by Spring Cloud AWS BOM) |
| Build | Maven (multi-module) |
| Local AWS | LocalStack + Docker |

**The 16 services covered**

Storage & data: **S3, DynamoDB, RDS/Aurora, ElastiCache** · Messaging & events: **SQS, SNS, EventBridge, Kinesis** · Security & config: **Secrets Manager, Parameter Store, KMS, Cognito** · Compute: **Lambda, ECS/Fargate** · Comms & observability: **SES, CloudWatch**

Each service section follows the same shape: *what it is → when to use it → IAM → dependencies → config → working code → where it fits in OrderFlow → local testing → gotchas.*

---

## Part 0 — Foundations

### 0.1 Two ways to talk to AWS from Java

There are two layers, and a professional codebase uses **both**.

1. **AWS SDK for Java v2** — the low-level, fully-typed client for every service (`S3Client`, `DynamoDbClient`, `KmsClient`, …). You always have this; Spring Cloud AWS pulls it in.
2. **Spring Cloud AWS 3.x** — thin, idiomatic Spring wrappers over the SDK for the *common* services (`S3Template`, `SqsTemplate`, `@SqsListener`, `SnsTemplate`, `DynamoDbTemplate`, config import for Secrets Manager / Parameter Store). It handles auto-configuration, credential wiring, and testcontainers/LocalStack integration.

Rule of thumb: **use Spring Cloud AWS where it exists; drop to the raw SDK v2 client for everything else** (KMS, EventBridge, Kinesis, Cognito admin calls).

| | AWS SDK v2 | Spring Cloud AWS 3.x |
|---|---|---|
| Coverage | Every service | S3, SQS, SNS, DynamoDB, SES, Secrets Mgr, Parameter Store, CloudWatch |
| Style | Verbose, explicit | Spring templates + annotations |
| Auto-config | Manual bean setup | `application.yml` driven |
| Best for | KMS, EventBridge, Kinesis, niche APIs | The 80% you use daily |

### 0.2 Credentials — the chain you must understand

Never hardcode keys. The SDK's `DefaultCredentialsProvider` resolves in this order, and a senior engineer knows it cold:

1. Java system properties (`aws.accessKeyId`, `aws.secretAccessKey`)
2. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
3. Web-identity token (OIDC / IRSA on EKS)
4. Shared config file (`~/.aws/credentials`, selected by `AWS_PROFILE`)
5. **Container credentials** (ECS task role)
6. **EC2 instance profile** (IMDSv2)

**Local dev** → use a named profile in `~/.aws/credentials`.
**Production** → attach an IAM role (ECS task role or EKS IRSA). Your app carries *zero* long-lived keys. This is the single most important security habit to demonstrate in an interview.

```yaml
# application.yml — local uses a profile; prod uses the role automatically
spring:
  cloud:
    aws:
      region:
        static: us-east-1
      credentials:
        profile-name: orderflow-dev   # omit entirely in prod
```

### 0.3 Parent POM — the BOM setup

Import both the Spring Boot parent and the Spring Cloud AWS BOM so every AWS artifact version stays aligned.

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.4</version>
</parent>

<properties>
    <java.version>21</java.version>
    <spring-cloud-aws.version>3.4.2</spring-cloud-aws.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.awspring.cloud</groupId>
            <artifactId>spring-cloud-aws-dependencies</artifactId>
            <version>${spring-cloud-aws.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Then each module pulls only the starters it needs, e.g.:

```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-s3</artifactId>
</dependency>
```

### 0.4 LocalStack — run all of AWS on your laptop

You should **never** develop against real AWS for day-to-day coding — it's slow, costs money, and pollutes accounts. LocalStack emulates ~all of these services locally.

```yaml
# docker-compose.yml
services:
  localstack:
    image: localstack/localstack:3
    ports:
      - "4566:4566"          # single edge port for every service
    environment:
      - SERVICES=s3,sqs,sns,dynamodb,secretsmanager,ssm,ses,kms,events,kinesis,cloudwatch,logs
      - DEBUG=1
    volumes:
      - "./.localstack:/var/lib/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

Point Spring Cloud AWS at LocalStack by overriding the endpoint (dev profile only):

```yaml
# application-local.yml
spring:
  cloud:
    aws:
      region:
        static: us-east-1
      credentials:
        access-key: test        # LocalStack ignores the values but needs them present
        secret-key: test
      endpoint: http://localhost:4566   # global override for all service clients
```

Provision resources with the `awslocal` CLI (or Terraform):

```bash
awslocal s3 mb s3://orderflow-invoices
awslocal sqs create-queue --queue-name order-processing
awslocal sns create-topic --name order-events
awslocal dynamodb create-table --table-name Products \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

For integration tests, use **Testcontainers** with the `LocalStackContainer` — spun up per test run, torn down after. (Full example in §16.)

### 0.5 The OrderFlow domain

A customer browses a **product catalog**, places an **order**, pays, and receives a **receipt** — with the platform emitting **events** for analytics and sending **notifications**. Here's how each service earns its place:

| Concern | Service |
|---|---|
| Product catalog (fast, flexible reads) | **DynamoDB** |
| Orders, payments (ACID, relational) | **RDS/Aurora** (Spring Data JPA) |
| Invoices, product images, receipts | **S3** |
| Async order fulfilment | **SQS** |
| Fan-out on `OrderPlaced` | **SNS** |
| Cross-service / scheduled events | **EventBridge** |
| Clickstream & order analytics | **Kinesis** |
| DB passwords, API keys | **Secrets Manager** |
| Feature flags, tunables | **Parameter Store** |
| Encrypt PII / card tokens | **KMS** |
| User auth (JWT) | **Cognito** |
| Thumbnail / receipt generation | **Lambda** |
| Transactional email | **SES** |
| Session / catalog cache | **ElastiCache (Redis)** |
| Metrics, logs, alarms | **CloudWatch** |
| Running the whole thing | **ECS/Fargate** |

Suggested module layout:

```
orderflow/
├── pom.xml                    (parent, BOM)
├── orderflow-catalog/         (DynamoDB, ElastiCache)
├── orderflow-orders/          (RDS/JPA, SQS producer, SNS)
├── orderflow-fulfilment/      (SQS consumer, S3, SES)
├── orderflow-analytics/       (Kinesis, EventBridge)
├── orderflow-common/          (KMS, config, security/Cognito)
└── orderflow-receipt-fn/      (Lambda, Spring Cloud Function)
```

---

## Part 1 — Data & Storage

### 1. Amazon S3 — object storage

**What / when.** Durable, cheap blob storage for anything that isn't a database row: invoices, product images, receipts, backups, user uploads. In OrderFlow it stores generated PDF invoices and product photos.

**IAM (least privilege).**
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
  "Resource": "arn:aws:s3:::orderflow-invoices/*"
}
```

**Dependency.**
```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-s3</artifactId>
</dependency>
```

**Code — upload, download, presigned URLs.** `S3Template` is the idiomatic wrapper.

```java
@Service
@RequiredArgsConstructor
public class InvoiceStorageService {

    private final S3Template s3Template;
    private static final String BUCKET = "orderflow-invoices";

    /** Upload a generated invoice PDF. */
    public String store(String orderId, byte[] pdf) {
        String key = "invoices/%s.pdf".formatted(orderId);
        try (var in = new ByteArrayInputStream(pdf)) {
            S3Resource resource = s3Template.upload(BUCKET, key, in,
                    ObjectMetadata.builder().contentType("application/pdf").build());
            return resource.getLocation().toString();
        } catch (IOException e) {
            throw new StorageException("Failed to store invoice " + orderId, e);
        }
    }

    /** Time-limited link the customer can download without AWS creds. */
    public URL presignedDownload(String orderId) {
        String key = "invoices/%s.pdf".formatted(orderId);
        return s3Template.createSignedGetURL(BUCKET, key, Duration.ofMinutes(15));
    }

    public byte[] fetch(String orderId) throws IOException {
        S3Resource resource = s3Template.download(BUCKET, "invoices/%s.pdf".formatted(orderId));
        try (var in = resource.getInputStream()) {
            return in.readAllBytes();
        }
    }
}
```

You can also treat S3 as a Spring `Resource` with the `s3://` protocol — nice for streaming large files:

```java
@Value("s3://orderflow-invoices/invoices/latest.pdf")
private Resource invoice;   // works via Spring Cloud AWS ResourceLoader
```

**Local test.** `awslocal s3 mb s3://orderflow-invoices`, then run against the LocalStack endpoint.

**Gotchas.** Presigned URLs beat making buckets public — never expose customer invoices via public ACLs. Set lifecycle rules to transition old objects to Glacier. Enable SSE (see KMS, §11) for anything sensitive.

---

### 2. Amazon DynamoDB — managed NoSQL

**What / when.** Single-digit-millisecond key-value / document store that scales without you managing servers. Perfect for high-read, access-pattern-driven data. In OrderFlow it backs the **product catalog** and shopping carts.

**Key design mindset.** DynamoDB is *not* a relational DB — you model around access patterns, choosing a partition key (and optional sort key). Get this wrong and you'll hot-partition. This distinction is a classic interview probe.

**Dependency.**
```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-dynamodb</artifactId>
</dependency>
```

**Entity — the enhanced-client bean.**
```java
@DynamoDbBean
public class Product {
    private String id;
    private String category;
    private String name;
    private BigDecimal price;
    private int stock;

    @DynamoDbPartitionKey
    public String getId() { return id; }

    @DynamoDbSecondaryPartitionKey(indexNames = "category-index")
    public String getCategory() { return category; }
    // remaining getters/setters …
}
```

**Repository via `DynamoDbTemplate`.**
```java
@Repository
@RequiredArgsConstructor
public class ProductRepository {

    private final DynamoDbTemplate template;

    public Product save(Product p) { return template.save(p); }

    public Optional<Product> findById(String id) {
        return Optional.ofNullable(template.load(Key.builder()
                .partitionValue(id).build(), Product.class));
    }

    public List<Product> findByCategory(String category) {
        var query = QueryEnhancedRequest.builder()
                .queryConditional(QueryConditional.keyEqualTo(
                        Key.builder().partitionValue(category).build()))
                .build();
        return template.query(query, Product.class, "category-index")
                .items().stream().toList();
    }
}
```

**Gotchas.** Prefer `PAY_PER_REQUEST` billing until you know your traffic. Use **Global Secondary Indexes** for alternate access patterns rather than scans. `Scan` is a code smell in production — it reads the whole table. For conditional writes (e.g. decrement stock only if `stock > 0`), use `Expected`/condition expressions to avoid race conditions.

---

### 3. Amazon RDS / Aurora — relational database

**What / when.** Managed PostgreSQL/MySQL (Aurora = AWS's cloud-native, faster, auto-scaling flavour). Use it where you need ACID transactions and relational integrity — in OrderFlow, the **orders and payments** tables.

**Key point:** there's *no special AWS SDK* here. Your app connects with plain JDBC + Spring Data JPA/HikariCP — RDS is just a managed Postgres endpoint. What AWS gives you is the managed instance, backups, Multi-AZ failover, and read replicas.

**Dependencies.**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

**Config — pull the password from Secrets Manager, not YAML** (ties into §9):
```yaml
spring:
  datasource:
    url: jdbc:postgresql://orderflow.abc123.us-east-1.rds.amazonaws.com:5432/orderflow
    username: ${db-username}       # resolved from Secrets Manager via config import
    password: ${db-password}
    hikari:
      maximum-pool-size: 10        # tune to RDS max_connections
  jpa:
    hibernate:
      ddl-auto: validate           # never 'update' in prod — use Flyway/Liquibase
```

**Entity + repository** are ordinary Spring Data:
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    private String customerId;
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    private BigDecimal total;
    private Instant createdAt;
    // …
}

public interface OrderRepository extends JpaRepository<Order, UUID> {
    List<Order> findByCustomerIdAndStatus(String customerId, OrderStatus status);
}
```

**Pro moves.** Route heavy reads to an Aurora **read replica** with a second `DataSource` + `@Transactional(readOnly = true)` routing. Better yet, use **RDS IAM authentication** so the app requests a short-lived token instead of a static password — zero secrets. Always use a connection pool sized to the DB's connection limit (you know HikariCP well — this is where that knowledge pays off).

---

### 4. Amazon ElastiCache — managed Redis

**What / when.** In-memory cache/store for sessions, hot catalog lookups, rate limiting, and distributed locks. In OrderFlow it caches product reads (which hit DynamoDB) and holds user sessions.

**Key point:** like RDS, ElastiCache is just a managed **Redis** endpoint — you use Spring Data Redis / Lettuce, no AWS SDK required. AWS manages clustering, failover, and patching.

**Dependencies.**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

**Config.**
```yaml
spring:
  data:
    redis:
      host: orderflow-cache.abc123.ng.0001.use1.cache.amazonaws.com
      port: 6379
      ssl:
        enabled: true          # in-transit encryption on ElastiCache
```

**Declarative caching** — the cleanest win:
```java
@Service
@RequiredArgsConstructor
public class CatalogService {

    private final ProductRepository products;   // DynamoDB-backed

    @Cacheable(value = "products", key = "#id")
    public Product get(String id) {
        return products.findById(id).orElseThrow();
    }

    @CacheEvict(value = "products", key = "#product.id")
    public void update(Product product) {
        products.save(product);
    }
}
```

Enable with `@EnableCaching` and a Redis cache manager (TTL is critical — never cache without expiry):
```java
@Bean
public RedisCacheConfiguration cacheConfig() {
    return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues();
}
```

**Gotchas.** Always set a TTL. Design for cache misses (fall through to source of truth). For cross-instance locks use Redisson. Cluster mode changes the client config — know the difference between cluster and non-cluster ElastiCache.

---

## Part 2 — Messaging & Events

### 5. Amazon SQS — managed message queue

**What / when.** A durable queue that decouples producers from consumers so slow work happens asynchronously and survives restarts. In OrderFlow, `orderflow-orders` drops an `order-processing` message; `orderflow-fulfilment` consumes it to generate the invoice, charge payment, and email the customer.

**IAM.** `sqs:SendMessage`, `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes` on the queue ARN.

**Dependency.**
```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sqs</artifactId>
</dependency>
```

**Producer — `SqsTemplate`.**
```java
@Service
@RequiredArgsConstructor
public class OrderPublisher {
    private final SqsTemplate sqsTemplate;

    public void enqueue(OrderPlacedEvent event) {
        sqsTemplate.send(to -> to
                .queue("order-processing")
                .payload(event)                        // auto-serialized to JSON
                .header("orderId", event.orderId()));
    }
}
```

**Consumer — `@SqsListener`** (Spring Cloud AWS runs a polling container, handles ack/delete on success):
```java
@Component
@RequiredArgsConstructor
public class OrderProcessingListener {

    private final FulfilmentService fulfilment;

    @SqsListener("order-processing")
    public void onOrder(OrderPlacedEvent event,
                        @Header("orderId") String orderId) {
        log.info("Processing order {}", orderId);
        fulfilment.fulfil(event);   // throw to trigger retry / DLQ
    }
}
```

**Gotchas.** Configure a **Dead Letter Queue** with a `maxReceiveCount` so poison messages don't loop forever. SQS gives *at-least-once* delivery — make consumers **idempotent** (you already use idempotency keys — apply them here). Use **FIFO queues** (`*.fifo`) only when strict ordering matters; they have lower throughput. `visibilityTimeout` must exceed your processing time or the message reappears mid-work.

---

### 6. Amazon SNS — pub/sub fan-out

**What / when.** Publish once, deliver to many subscribers (SQS queues, Lambda, email, HTTP, SMS). The **SNS→SQS fan-out** pattern is the backbone of event-driven AWS architectures. In OrderFlow, `OrderPlaced` is published to an SNS topic; fulfilment, analytics, and notification queues all subscribe.

**Dependency.**
```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sns</artifactId>
</dependency>
```

**Publisher — `SnsTemplate`.**
```java
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {
    private final SnsTemplate snsTemplate;

    public void publishOrderPlaced(OrderPlacedEvent event) {
        snsTemplate.sendNotification("order-events",
                SnsNotification.builder(event)
                        .header("eventType", "ORDER_PLACED")
                        .groupId(event.customerId())   // for FIFO topics
                        .build());
    }
}
```

Each downstream service just uses an ordinary `@SqsListener` on *its own* queue subscribed to the topic — so one publish reaches all consumers, each processing independently and retrying independently.

**SNS vs SQS (know this cold):** SQS = one queue, one logical consumer group, pull-based. SNS = broadcast to N subscribers, push-based. Combine them (SNS → many SQS) for durable fan-out; pure SNS→Lambda loses the buffering/retry that a queue gives you.

**Gotchas.** Use **message filter policies** so each subscriber only gets relevant events (e.g. analytics ignores `ORDER_CANCELLED`). SNS→SQS needs a queue access policy allowing the topic to send.

---

### 7. Amazon EventBridge — the event bus

**What / when.** A serverless event bus for routing events between AWS services, SaaS apps, and your microservices, plus **scheduled (cron) rules**. Where SNS is simple fan-out, EventBridge adds content-based routing, schemas, and scheduling. In OrderFlow it routes domain events to analytics and runs a nightly "reconcile payments" job.

**No high-level Spring Cloud AWS wrapper** — use the SDK v2 `EventBridgeClient` (declare it as a bean; the region/credentials auto-configure).

```java
@Configuration
public class EventBridgeConfig {
    @Bean
    EventBridgeClient eventBridgeClient(AwsRegionProvider region,
                                        AwsCredentialsProvider creds) {
        return EventBridgeClient.builder()
                .region(region.getRegion())
                .credentialsProvider(creds)
                .build();
    }
}
```

**Publishing a custom event.**
```java
@Service
@RequiredArgsConstructor
public class DomainEventBridgePublisher {
    private final EventBridgeClient client;
    private final ObjectMapper mapper;

    public void publish(String detailType, Object detail) {
        var entry = PutEventsRequestEntry.builder()
                .eventBusName("orderflow-bus")
                .source("orderflow.orders")
                .detailType(detailType)
                .detail(toJson(detail))
                .build();
        client.putEvents(r -> r.entries(entry));
    }
    private String toJson(Object o) { /* mapper.writeValueAsString with try/catch */ }
}
```

Consumers are **rules** (defined in Terraform/console) that match on `source`/`detail-type` and target a Lambda, SQS queue, or Step Function. Scheduled rule example (cron): target a Lambda every night at 2am with `cron(0 2 * * ? *)`.

**SNS vs EventBridge:** reach for EventBridge when you need routing on event *content*, third-party/SaaS integration, schemas, or scheduling. Reach for SNS when you just need fast, cheap fan-out.

---

### 8. Amazon Kinesis — streaming data

**What / when.** Ingest and process high-volume, ordered streams in real time — clickstreams, IoT telemetry, order events for analytics. In OrderFlow, every catalog view and order flows into a Kinesis stream that feeds real-time dashboards. Unlike SQS (delete-after-consume), Kinesis **retains** records and supports multiple independent consumers replaying the same stream.

**Producing — SDK v2 `KinesisClient`.**
```java
@Service
@RequiredArgsConstructor
public class ClickstreamProducer {
    private final KinesisClient kinesis;
    private final ObjectMapper mapper;

    public void record(ClickEvent event) {
        kinesis.putRecord(r -> r
                .streamName("orderflow-clickstream")
                .partitionKey(event.sessionId())        // controls shard distribution
                .data(SdkBytes.fromByteArray(toBytes(event))));
    }
}
```

**Consuming.** For production, use the **Kinesis Client Library (KCL) 3.x**, which handles shard leasing, checkpointing (in DynamoDB), and rebalancing across instances — don't hand-roll `getRecords`. Alternatively, **Spring Cloud Stream Kinesis binder** gives you a `@Bean Consumer<...>` programming model similar to SQS listeners.

**SQS vs Kinesis (frequent interview question):** SQS = work queue, message deleted once handled, no ordering guarantee (except FIFO), no replay. Kinesis = ordered stream per shard, retained for hours/days, **multiple** consumers, replayable — built for analytics and event sourcing, not task distribution.

**Gotchas.** Throughput scales with **shards** (1MB/s or 1000 records/s write each) — pick a good partition key to avoid a hot shard. Watch `IteratorAgeMilliseconds` to detect a lagging consumer. Consider **Kinesis Data Firehose** if you just need to dump the stream into S3/Redshift with no custom processing.

---

## Part 3 — Security, Config & Secrets

### 9. AWS Secrets Manager — secrets, rotated

**What / when.** Stores and automatically rotates sensitive values: DB passwords, third-party API keys, payment gateway tokens. In OrderFlow it holds the RDS credentials and the Stripe/Chapa key. Spring Cloud AWS surfaces secrets **as property sources at startup**, so they behave like ordinary config properties.

**Dependency.**
```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
</dependency>
```

**Config — import the secret; its JSON keys become properties.**
```yaml
spring:
  config:
    import: aws-secretsmanager:orderflow/rds-credentials
```
If the secret's JSON is `{"db-username":"orderflow","db-password":"…"}`, you reference `${db-username}` / `${db-password}` directly (as shown in the RDS section). Import multiple secrets with an optional/list syntax:
```yaml
spring:
  config:
    import:
      - aws-secretsmanager:orderflow/rds-credentials
      - optional:aws-secretsmanager:orderflow/payment-keys
```

**Gotchas.** Prefer Secrets Manager over Parameter Store when you need **automatic rotation** (it can rotate RDS creds for you). It costs per secret + per API call, so cache at startup (the config-import approach does this — it doesn't re-fetch every request). Grant `secretsmanager:GetSecretValue` on the specific secret ARN only.

---

### 10. AWS Systems Manager Parameter Store — config & flags

**What / when.** Hierarchical key-value config: feature flags, tunables, non-rotating settings, and (via `SecureString`) cheaper secrets. In OrderFlow it holds feature flags (`/orderflow/features/express-checkout`) and business tunables (free-shipping threshold).

**Dependency.**
```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-parameter-store</artifactId>
</dependency>
```

**Config — import a path; children become properties.**
```yaml
spring:
  config:
    import: aws-parameterstore:/orderflow/
```
`/orderflow/features/express-checkout` → property `features.express-checkout`. Combine with Spring `@ConfigurationProperties` and (with Spring Cloud Context) `@RefreshScope` to pick up changes without redeploying.

```java
@Component
@ConfigurationProperties(prefix = "features")
public class FeatureFlags {
    private boolean expressCheckout;
    // getter/setter
}
```

**Secrets Manager vs Parameter Store:** Parameter Store is cheaper and great for plain config + occasional secrets, but has **no built-in rotation**. Use Secrets Manager for anything that must rotate (DB creds); use Parameter Store for everything else. Knowing exactly when to pick each is a strong senior signal.

---

### 11. AWS KMS — encryption keys

**What / when.** Managed encryption keys for encrypting data at rest and in transit, and for envelope encryption of application-level secrets. In OrderFlow it encrypts stored card-token references and PII columns, and provides the key behind S3/RDS encryption.

**No high-level wrapper — use SDK v2 `KmsClient`.** The common professional pattern is **envelope encryption**: KMS generates a data key, you encrypt the payload locally with the plaintext data key (fast, no size limit), then store the KMS-encrypted data key alongside the ciphertext.

```java
@Service
@RequiredArgsConstructor
public class FieldEncryptionService {

    private final KmsClient kms;
    private static final String KEY_ID = "alias/orderflow-pii";

    /** Envelope-encrypt a sensitive field. */
    public EncryptedField encrypt(byte[] plaintext) {
        var dk = kms.generateDataKey(r -> r.keyId(KEY_ID)
                .keySpec(DataKeySpec.AES_256));
        byte[] ciphertext = aesGcmEncrypt(dk.plaintext().asByteArray(), plaintext);
        return new EncryptedField(ciphertext,
                dk.ciphertextBlob().asByteArray());   // store both
    }

    public byte[] decrypt(EncryptedField field) {
        var resp = kms.decrypt(r -> r
                .ciphertextBlob(SdkBytes.fromByteArray(field.encryptedDataKey())));
        return aesGcmDecrypt(resp.plaintext().asByteArray(), field.ciphertext());
    }
    // aesGcm* : standard javax.crypto AES/GCM/NoPadding helpers
}
```

**Gotchas.** For small values you can call `kms.encrypt` directly, but the **4KB limit** forces envelope encryption for anything larger — hence the pattern above. Rotate keys (KMS supports automatic annual rotation). Scope IAM to specific key ARNs and actions (`kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey`). Most "encryption at rest" (S3 SSE-KMS, RDS, EBS) just needs you to *reference* a KMS key in config — no app code at all.

---

### 12. Amazon Cognito — user authentication

**What / when.** Managed user directory + OAuth2/OIDC identity provider: sign-up, sign-in, MFA, social login, and JWT issuance. In OrderFlow, customers authenticate against a Cognito User Pool; your APIs validate the resulting JWT. The clean pattern is to treat Cognito as a **standard OIDC provider** and use Spring Security's resource-server support — no AWS SDK needed on the hot path.

**Dependency.**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

**Config — point at your User Pool's issuer; JWKS validation is automatic.**
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_AbC123
```

**Security config — map Cognito groups to roles.**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/orders/**").authenticated()
                .anyRequest().permitAll())
            .oauth2ResourceServer(oauth -> oauth
                .jwt(jwt -> jwt.jwtAuthenticationConverter(cognitoConverter())));
        return http.build();
    }

    private JwtAuthenticationConverter cognitoConverter() {
        var converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<String> groups = jwt.getClaimAsStringList("cognito:groups");
            return groups == null ? List.of()
                : groups.stream()
                    .map(g -> new SimpleGrantedAuthority("ROLE_" + g))
                    .collect(Collectors.toList());
        });
        return converter;
    }
}
```

For **admin operations** (programmatic user creation, etc.) use the SDK v2 `CognitoIdentityProviderClient`. For login flows in a SPA/mobile app, use the Hosted UI or Amplify on the client and let the backend just validate tokens.

**Gotchas.** Validate the `token_use` claim (`access` vs `id`) — don't authorize with an ID token. Cache JWKS (Spring does this). Know the difference between **User Pools** (authn — who you are) and **Identity Pools** (authz to AWS resources via temporary IAM creds).

---

## Part 4 — Serverless & Compute

### 13. AWS Lambda — serverless functions (Spring Cloud Function)

**What / when.** Run code without managing servers, billed per invocation — ideal for event-driven, spiky, or glue workloads. In OrderFlow, a Lambda generates the **PDF receipt** and thumbnails when an object lands in S3, triggered by S3/SQS/EventBridge events. **Spring Cloud Function** lets you write plain `Function`/`Consumer` beans and deploy the *same* code to Lambda or as an HTTP endpoint.

**Dependencies.**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-adapter-aws</artifactId>
</dependency>
```

**The function is just a bean.**
```java
@SpringBootApplication
public class ReceiptFnApplication {
    public static void main(String[] args) {
        SpringApplication.run(ReceiptFnApplication.class, args);
    }

    @Bean
    public Function<S3Event, String> generateReceipt(ReceiptService receipts) {
        return event -> {
            event.getRecords().forEach(rec -> {
                String key = rec.getS3().getObject().getKey();
                receipts.buildAndStore(key);
            });
            return "ok";
        };
    }
}
```

The Lambda handler is provided by the adapter (`org.springframework.cloud.function.adapter.aws.FunctionInvoker`); you set the bean name via the `SPRING_CLOUD_FUNCTION_DEFINITION=generateReceipt` env var.

**Cold starts matter.** A full Spring Boot JVM Lambda has noticeable cold-start latency. Mitigations a senior candidate should name: **AWS Lambda SnapStart** for Java, **GraalVM native image** (`spring-boot-starter-native` / `native-maven-plugin`) for millisecond starts, provisioned concurrency, and keeping the deployment jar lean.

**When *not* to use Lambda.** Long-running, steady-traffic services belong on ECS/Fargate — Lambda's 15-minute cap and per-invocation model make it wrong for your always-on APIs. Use Lambda for event reactions and cron glue.

---

### 14. Amazon ECS / Fargate — running the services

**What / when.** ECS orchestrates containers; **Fargate** runs them serverlessly (no EC2 to patch). This is where your always-on Spring Boot microservices actually live. (EKS is the Kubernetes alternative — pick it when you need k8s portability/ecosystem; Fargate-on-ECS is simpler for a focused AWS shop.)

**Dockerfile — layered, small, non-root.**
```dockerfile
# Build with Spring Boot layered jars for better caching
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/orderflow-orders.jar app.jar
RUN addgroup -S app && adduser -S app -G app
USER app
EXPOSE 8080
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75","-jar","app.jar"]
```

**The deployment path (know the pieces):**
1. Build image, push to **ECR** (Elastic Container Registry).
2. Define an **ECS Task Definition** — image, CPU/memory, env vars, and crucially the **task IAM role** (this is what gives the container its AWS permissions — no keys in the image).
3. Run it as an **ECS Service** behind an **Application Load Balancer**, with a target group hitting `/actuator/health`.
4. Auto-scale on CPU/memory or a custom CloudWatch metric.

**Config that ties it together:** the task role is your credential source (§0.2), Secrets Manager/Parameter Store feed config, and CloudWatch collects logs/metrics. Health checks come from Spring Boot Actuator:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true          # liveness/readiness groups for ECS/k8s
```

**Gotchas.** Set container memory to leave JVM headroom (`MaxRAMPercentage`). Wire liveness vs readiness properly so ECS doesn't route traffic before the app is warm. Ship logs to CloudWatch via the `awslogs` driver. Use rolling or blue/green deploys via CodeDeploy.

---

## Part 5 — Communication & Observability

### 15. Amazon SES — transactional email

**What / when.** Send high-deliverability transactional email — order confirmations, receipts, password resets. In OrderFlow, fulfilment emails the customer their receipt with a presigned S3 link. Spring Cloud AWS integrates SES with Spring's `MailSender`, so it feels like ordinary Spring email.

**Dependency.**
```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-ses</artifactId>
</dependency>
```

**Code — inject the auto-configured sender.**
```java
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final JavaMailSender mailSender;   // backed by SES (SimpleEmailServiceJavaMailSender)

    public void sendReceipt(String to, String orderId, URL receiptLink) {
        var msg = new SimpleMailMessage();
        msg.setFrom("orders@orderflow.com");   // must be a verified SES identity
        msg.setTo(to);
        msg.setSubject("Your OrderFlow receipt — " + orderId);
        msg.setText("Thanks for your order! Download your receipt: " + receiptLink);
        mailSender.send(msg);
    }
}
```
For HTML + attachments use `MimeMessageHelper` with the same sender.

**Gotchas.** New SES accounts are **sandboxed** — you can only send to verified addresses until you request production access. Verify your domain and set up **SPF/DKIM/DMARC** for deliverability. Handle **bounces and complaints** via an SNS topic subscription — ignoring these tanks your sender reputation. For bulk/marketing email, SES is fine but consider dedicated tooling.

---

### 16. Amazon CloudWatch — metrics, logs, alarms

**What / when.** The observability backbone: application/infra **metrics**, centralized **logs**, **alarms**, and dashboards. In OrderFlow every service ships JVM + business metrics (orders/min, queue depth, payment failures) and structured logs.

**Metrics via Micrometer** (Spring Boot's metrics facade → CloudWatch registry):
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-cloudwatch2</artifactId>
</dependency>
```
```yaml
management:
  cloudwatch:
    metrics:
      export:
        namespace: OrderFlow
        step: 1m
```

**Custom business metrics** — the kind that make dashboards useful:
```java
@Service
@RequiredArgsConstructor
public class OrderMetrics {
    private final MeterRegistry registry;

    public void orderPlaced(String category) {
        registry.counter("orders.placed", "category", category).increment();
    }
    public void recordProcessingTime(Duration d) {
        registry.timer("orders.processing.time").record(d);
    }
}
```

**Logs.** On ECS, the `awslogs` driver streams stdout to CloudWatch Logs — emit **structured JSON logs** (via `logstash-logback-encoder`) so you can query them with CloudWatch Logs Insights. Alarms watch metrics (e.g. `orders.processing.time` p99 > 5s, or SQS `ApproximateNumberOfMessagesVisible` climbing) and page you via SNS.

**Testcontainers integration test** (ties the whole tutorial together — this is how you test AWS code without touching real AWS):
```java
@SpringBootTest
@Testcontainers
class OrderFlowIntegrationTest {

    @Container
    static LocalStackContainer localstack = new LocalStackContainer(
            DockerImageName.parse("localstack/localstack:3"))
            .withServices(S3, SQS, DYNAMODB);

    @DynamicPropertySource
    static void awsProps(DynamicPropertyRegistry r) {
        r.add("spring.cloud.aws.endpoint", localstack::getEndpoint);
        r.add("spring.cloud.aws.region.static", localstack::getRegion);
        r.add("spring.cloud.aws.credentials.access-key", localstack::getAccessKey);
        r.add("spring.cloud.aws.credentials.secret-key", localstack::getSecretKey);
    }

    @Test
    void placingOrder_storesInvoiceAndEnqueuesProcessing() {
        // arrange bucket/queue, POST an order, assert S3 object + SQS message exist
    }
}
```

**Gotchas.** Custom metrics cost per metric — don't explode cardinality with unbounded tag values (e.g. per-user tags). Set log retention (default is *never expire* = growing bill). Prefer **EMF (Embedded Metric Format)** or OpenTelemetry for high-volume metrics. Alarms are only useful if wired to a real notification channel.

---

## Deployment & CI/CD — putting it in production

The professional loop:

1. **Infrastructure as Code** — define every resource (VPC, RDS, DynamoDB tables, queues, roles) in **Terraform** or **AWS CDK**. Never click-ops production.
2. **CI** — GitHub Actions/CodeBuild: `mvn verify` (with Testcontainers/LocalStack), build layered Docker image, push to **ECR**.
3. **CD** — update the ECS service (rolling or blue/green via CodeDeploy). Config/secrets come from Secrets Manager + Parameter Store at boot; the task role provides credentials.
4. **Observe** — CloudWatch dashboards + alarms → SNS → your on-call channel.

---

## Suggested learning order

Build OrderFlow incrementally — don't try all 16 at once:

1. **Week 1 — core plumbing:** S3 + SQS + SNS with LocalStack. Get the fan-out working end to end.
2. **Week 2 — data:** DynamoDB (catalog) + RDS/JPA (orders) + ElastiCache caching.
3. **Week 3 — security & config:** Secrets Manager + Parameter Store + Cognito + KMS.
4. **Week 4 — events & serverless:** EventBridge + Kinesis + a Lambda receipt function.
5. **Week 5 — ship it:** SES + CloudWatch + Dockerize + ECS/Fargate + Terraform.

Each week produces something demonstrable for a portfolio and gives you concrete stories for interviews.

## Interview-ready talking points (the "why," not just the "how")

These are the trade-off questions that separate senior from mid-level:

- **SQS vs SNS vs Kinesis vs EventBridge** — queue vs fan-out vs replayable stream vs content-routed bus. Be able to pick the right one and justify it.
- **DynamoDB vs RDS** — access-pattern modeling vs relational/ACID; when a single-table DynamoDB design beats joins, and when it doesn't.
- **Secrets Manager vs Parameter Store** — rotation vs cost.
- **Lambda vs ECS/Fargate** — event-driven/spiky vs always-on; cold starts and how to fix them (SnapStart, GraalVM native).
- **Credentials** — the provider chain, and why production apps carry zero long-lived keys (task roles / IRSA).
- **Idempotency & at-least-once delivery** — how you make SQS/SNS consumers safe (you already do this with idempotency keys).
- **Envelope encryption with KMS** — why the 4KB limit forces it.

---

*Built for a stable, professional stack: Java 21 · Spring Boot 3.5.x · Spring Cloud AWS 3.4.2 · AWS SDK v2. When Spring Boot 4 / Spring Cloud AWS 4.x stabilise, the concepts carry over unchanged — only a few dependency versions move.*
