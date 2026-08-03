# OrderFlow — Deep Dive: Kafka, Containers & the Alternative Services

Full provisioning + Spring Boot config/code for the services that got the "brief" treatment in the earlier docs: **MSK (Managed Kafka)**, the container stack (**ECS/Fargate**, **EKS**), the compute alternatives (**Batch**, **EC2**, **Elastic Beanstalk**), and the alternative databases and messaging (**DocumentDB**, **Keyspaces**, **Neptune**, **Timestream**, **Redshift**, **Amazon MQ**, **SageMaker**).

Same conventions as before: Java 21 · Spring Boot 3.5.x · AWS CLI for provisioning (port to CDK for production). Assumes the VPC/subnets/security-groups/`$ACCOUNT_ID` from the Provisioning Playbook already exist.

**A reminder on these being alternatives.** Several here overlap services you already have (MSK vs Kinesis, EKS vs ECS, DocumentDB vs DynamoDB, Keyspaces vs DynamoDB, Redshift vs Athena). In OrderFlow they each own a distinct sub-domain so you learn them; in a lean real system you'd pick one of each pair. The overlap notes at each section tell you which.

---

## 1. Amazon MSK — Managed Apache Kafka

**Role in OrderFlow:** the **CDC pipeline** — Debezium captures every change in Aurora and streams it through Kafka to the data lake and downstream consumers. Distinct from Kinesis (which carries raw clickstream). Also the right choice any time you need real Kafka semantics: consumer groups, log compaction, exactly-once, Kafka Connect, or an existing Kafka codebase.

**MSK vs Kinesis:** pick **Kinesis** for AWS-native simplicity and no broker management; pick **MSK** when you need the Kafka ecosystem/portability. Don't run both unless, like here, they carry genuinely different streams.

### Provisioning

MSK Serverless is the simplest (no broker sizing, IAM auth built in):
```bash
cat > msk.json <<EOF
{ "ClusterName":"orderflow-cdc",
  "VpcConfigs":[{"SubnetIds":["$PRIV_A","$PRIV_B"],"SecurityGroupIds":["$APP_SG"]}],
  "ClientAuthentication":{"Sasl":{"Iam":{"Enabled":true}}} }
EOF
aws kafka create-cluster-v2 --serverless file://msk.json

# Get the bootstrap brokers (IAM SASL endpoint) once ACTIVE
CLUSTER_ARN=$(aws kafka list-clusters-v2 --query "ClusterInfoList[?ClusterName=='orderflow-cdc'].ClusterArn" --output text)
aws kafka get-bootstrap-brokers --cluster-arn $CLUSTER_ARN --query BootstrapBrokerStringSaslIam --output text
```
The app's task role needs `kafka-cluster:Connect`, `kafka-cluster:*Topic*`, `kafka-cluster:WriteData`, `kafka-cluster:ReadData` on the cluster/topic ARNs. Also allow the `orderflow-app` security group to reach itself on 9098 (IAM) so brokers and clients connect.

Create a topic (from a client inside the VPC, or via a small admin task):
```bash
# using the standard Kafka CLI with the IAM auth jar on the classpath
kafka-topics.sh --create --topic orders.cdc --partitions 3 --replication-factor 3 \
  --bootstrap-server $BOOTSTRAP --command-config client-iam.properties
```

### Spring Boot config

**pom.xml** — Spring Kafka + the AWS MSK IAM auth provider:
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
<dependency>
    <groupId>software.amazon.msk</groupId>
    <artifactId>aws-msk-iam-auth</artifactId>
    <version>2.2.0</version>
</dependency>
```

**application.yml** — IAM auth means no passwords; the task role authenticates:
```yaml
spring:
  kafka:
    bootstrap-servers: b-1.orderflow-cdc.xxx.kafka.us-east-1.amazonaws.com:9098,b-2...:9098
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: AWS_MSK_IAM
      sasl.jaas.config: software.amazon.msk.auth.iam.IAMLoginModule required;
      sasl.client.callback.handler.class: software.amazon.msk.auth.iam.IAMClientCallbackHandler
    consumer:
      group-id: orderflow-cdc-consumers
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.orderflow.*"
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

### Producer
```java
package com.orderflow.cdc;

import lombok.RequiredArgsConstructor;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

public record OrderChange(String orderId, String field, String oldValue, String newValue) {}

@Service
@RequiredArgsConstructor
public class CdcProducer {
    private final KafkaTemplate<String, Object> kafka;

    public void publish(OrderChange change) {
        kafka.send("orders.cdc", change.orderId(), change);   // key = orderId → same partition, ordered
    }
}
```

### Consumer
```java
package com.orderflow.cdc;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class CdcConsumer {
    private final LakeWriter lake;

    @KafkaListener(topics = "orders.cdc", groupId = "orderflow-cdc-consumers")
    public void onChange(OrderChange change) {
        log.info("CDC {} {} -> {}", change.orderId(), change.field(), change.newValue());
        lake.append(change);   // write to the S3 data lake
    }
}
```

**Enable Kafka** with `@EnableKafka` on a config class. **Gotchas:** consumers must be idempotent (Kafka is at-least-once by default); size partitions to your consumer parallelism (one consumer per partition max within a group); log compaction vs retention is a per-topic decision. For real CDC you'd run **Debezium** (via MSK Connect) rather than hand-writing change events — the producer above shows the shape your app would emit for domain events.

---

## 2. Amazon ECS + Fargate — deep dive

**Role:** runs the always-on OrderFlow microservices. The Provisioning Playbook created a cluster/service; here's the full container story — Dockerfile, task definition internals, secrets injection, health probes, graceful shutdown, and rollout.

### Dockerfile (layered, small, non-root)
Use Spring Boot's layered jars so Docker caches dependencies separately from your code:
```dockerfile
# ---- build ----
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /src
COPY . .
RUN ./mvnw -q -pl orderflow-orders -am package -DskipTests \
 && java -Djarmode=layertools -jar orderflow-orders/target/*.jar extract --destination /layers

# ---- run ----
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=build /layers/dependencies/ ./
COPY --from=build /layers/spring-boot-loader/ ./
COPY --from=build /layers/snapshot-dependencies/ ./
COPY --from=build /layers/application/ ./
USER app
EXPOSE 8080
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75","-XX:+UseZGC","org.springframework.boot.loader.launch.JarLauncher"]
```

### Spring Boot config for containers
Graceful shutdown + Actuator liveness/readiness so ECS never routes to a cold or draining task:
```yaml
server:
  shutdown: graceful          # finish in-flight requests on SIGTERM
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true         # exposes /actuator/health/liveness and /readiness
      group:
        readiness:
          include: readinessState,db,redis
```

### Task definition — the important fields
The Playbook showed the skeleton. The details that matter in production:
```json
{
  "family": "orderflow-orders",
  "cpu": "512", "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCT:role/orderflow-orders-task",
  "containerDefinitions": [{
    "name": "orders",
    "image": "ACCT.dkr.ecr.us-east-1.amazonaws.com/orderflow/orders:1.4.2",
    "portMappings": [{"containerPort": 8080}],
    "environment": [{"name": "SPRING_PROFILES_ACTIVE", "value": "prod"}],
    "secrets": [
      {"name": "SPRING_DATASOURCE_PASSWORD",
       "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCT:secret:orderflow/rds-credentials:password::"}
    ],
    "healthCheck": {
      "command": ["CMD-SHELL","wget -q -O- http://localhost:8080/actuator/health/liveness || exit 1"],
      "interval": 30, "timeout": 5, "retries": 3, "startPeriod": 60
    },
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {"awslogs-group":"/ecs/orderflow-orders","awslogs-region":"us-east-1","awslogs-stream-prefix":"orders"}
    }
  }]
}
```
Key distinctions: the **execution role** lets ECS pull the image and read the secret; the **task role** is what your *app code* uses for AWS calls. The `secrets` block injects the DB password from Secrets Manager as an env var at launch — never bake it into the image. `startPeriod` gives the JVM time to warm before health checks count.

### Rollout & scaling
The ALB target group uses `/actuator/health/readiness`; ECS does a rolling deploy (drain old tasks after new ones pass health checks) or blue/green via CodeDeploy with auto-rollback on a CloudWatch alarm. Auto scaling was set in the Playbook (target-tracking on CPU); add a second policy on the SQS backlog metric for the fulfilment service.

---

## 3. Amazon EKS — deep dive

**Role:** the shared **data-science / ML platform** cluster (SageMaker pipelines, custom training/inference) — chosen for the Kubernetes ecosystem, kept separate from the app tier on ECS.

**ECS vs EKS:** ECS is less to operate and the right default. Choose EKS for k8s portability, the CNCF ecosystem (Helm, operators, service mesh), or when the org already runs Kubernetes. Running both is only justified when two teams genuinely have different needs — as here.

### Provisioning (eksctl does the heavy lifting)
```bash
eksctl create cluster --name orderflow-ml --region us-east-1 \
  --vpc-private-subnets $PRIV_A,$PRIV_B --fargate

# OIDC provider (required for IRSA) + a service account bound to an IAM role
eksctl utils associate-iam-oidc-provider --cluster orderflow-ml --approve
eksctl create iamserviceaccount --cluster orderflow-ml --namespace orderflow \
  --name orders-sa --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/orderflow-orders-access \
  --approve
```
**IRSA** (IAM Roles for Service Accounts) is the EKS equivalent of an ECS task role — pods get AWS permissions via the annotated service account, still zero long-lived keys.

### Kubernetes manifests

**Deployment** — note the graceful-shutdown and probe wiring mirrors the ECS setup:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders, namespace: orderflow }
spec:
  replicas: 2
  selector: { matchLabels: { app: orders } }
  template:
    metadata: { labels: { app: orders } }
    spec:
      serviceAccountName: orders-sa          # IRSA → AWS permissions
      terminationGracePeriodSeconds: 30
      containers:
        - name: orders
          image: ACCT.dkr.ecr.us-east-1.amazonaws.com/orderflow/orders:1.4.2
          ports: [{ containerPort: 8080 }]
          env:
            - { name: SPRING_PROFILES_ACTIVE, value: prod }
          envFrom:
            - secretRef: { name: orders-secrets }
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits:   { cpu: "500m", memory: "1Gi" }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 30
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 45
```

**Service + Horizontal Pod Autoscaler:**
```yaml
apiVersion: v1
kind: Service
metadata: { name: orders, namespace: orderflow }
spec:
  selector: { app: orders }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: orders-hpa, namespace: orderflow }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: orders }
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```
Expose externally with an **AWS Load Balancer Controller** `Ingress` (provisions an ALB), the k8s equivalent of the ECS ALB. The Spring Boot config (graceful shutdown, actuator probes) is identical to ECS — the app doesn't care which orchestrator runs it, which is exactly the point of containers.

---

## 4. Compute alternatives

### AWS Batch — bulk jobs
**Role:** nightly bulk work (catalog image re-processing, data-lake compaction). Runs containerized jobs on a Fargate/EC2 compute environment.
```bash
aws batch create-compute-environment --compute-environment-name orderflow-batch \
  --type MANAGED --state ENABLED \
  --compute-resources 'type=FARGATE,maxvCpus=16,subnets=['$PRIV_A'],securityGroupIds=['$APP_SG']'
aws batch create-job-queue --job-queue-name orderflow-batch-q --priority 1 \
  --compute-environment-order 'order=1,computeEnvironment=orderflow-batch'
aws batch register-job-definition --job-definition-name image-reprocess --type container \
  --platform-capabilities FARGATE \
  --container-properties '{"image":"ACCT.dkr.ecr.us-east-1.amazonaws.com/orderflow/image-worker:latest","resourceRequirements":[{"type":"VCPU","value":"1"},{"type":"MEMORY","value":"2048"}],"jobRoleArn":"arn:aws:iam::ACCT:role/orderflow-batch-job","executionRoleArn":"arn:aws:iam::ACCT:role/ecsTaskExecutionRole"}'
```
The worker is a plain Spring Boot app that runs once and exits — a `CommandLineRunner`, not a web server:
```java
@SpringBootApplication
public class ImageWorkerApplication {
    public static void main(String[] args) {
        System.exit(SpringApplication.exit(SpringApplication.run(ImageWorkerApplication.class, args)));
    }
    @Bean
    CommandLineRunner run(ImageProcessor processor) {
        return args -> processor.reprocessBatch(System.getenv("BATCH_PREFIX"));  // then JVM exits
    }
}
```
Submit: `aws batch submit-job --job-name nightly --job-queue orderflow-batch-q --job-definition image-reprocess`. Trigger it from an EventBridge scheduled rule.

### EC2 — the backing compute (brief)
EC2 mostly appears *under* Batch/ECS here. If you need a raw instance (a licensed third-party connector, say), the pattern is: launch with an **instance profile** (its IAM role), bootstrap via **user data**, never open SSH — use **SSM Session Manager**.
```bash
aws ec2 run-instances --image-id ami-xxxxxxxx --instance-type t3.small \
  --iam-instance-profile Name=orderflow-connector \
  --subnet-id $PRIV_A --security-group-ids $APP_SG \
  --user-data file://bootstrap.sh
```

### Elastic Beanstalk — the marketing microsite (brief)
Beanstalk is fine for a small standalone Spring Boot app where you don't want to manage ECS. It runs a fat jar directly.
```bash
eb init orderflow-site --platform "Corretto 21" --region us-east-1
eb create orderflow-site-env --instance-type t3.small
eb deploy      # uploads target/*.jar
```
Add a `Procfile` (`web: java -jar application.jar --server.port=5000`) and `.ebextensions/` for env vars. Beanstalk gives you the ALB, auto scaling, and health checks without writing a task definition — convenient, but you trade away control, which is why the main services use ECS.

---

## 5. Alternative databases (Spring config for each)

### DocumentDB — product reviews (MongoDB-compatible)
```bash
aws docdb create-db-cluster --db-cluster-identifier orderflow-reviews \
  --engine docdb --master-username orderflow --manage-master-user-password \
  --db-subnet-group-name orderflow-db-subnets --vpc-security-group-ids $DB_SG
aws docdb create-db-instance --db-instance-identifier orderflow-reviews-1 \
  --db-cluster-identifier orderflow-reviews --engine docdb --db-instance-class db.t3.medium
```
Spring Data MongoDB — download the Amazon RDS CA bundle (`global-bundle.pem`) and point the driver at it:
```xml
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-mongodb</artifactId></dependency>
```
```yaml
spring:
  data:
    mongodb:
      uri: mongodb://orderflow:${docdb-password}@orderflow-reviews.cluster-xxx.docdb.amazonaws.com:27017/reviews?tls=true&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false
      # JVM: -Djavax.net.ssl.trustStore=/opt/certs/rds-truststore.jks
```
```java
@Document("reviews")
public record Review(@Id String id, String productId, String customerId, int rating, String text) {}

public interface ReviewRepository extends MongoRepository<Review, String> {
    List<Review> findByProductIdOrderByRatingDesc(String productId);
}
```
**Overlap:** DocumentDB here only because reviews are document-shaped. Most teams keep reviews in DynamoDB or Aurora and skip it.

### Keyspaces — high-write event store (Cassandra-compatible)
```bash
# Create keyspace/table via CQL against cassandra.us-east-1.amazonaws.com:9142 (TLS)
```
Spring Data Cassandra with the AWS SigV4 auth plugin (or service-specific credentials):
```xml
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-cassandra</artifactId></dependency>
<dependency><groupId>software.aws.mcs</groupId><artifactId>aws-sigv4-auth-cassandra-java-driver-plugin</artifactId><version>4.0.9</version></dependency>
```
```yaml
spring:
  cassandra:
    contact-points: cassandra.us-east-1.amazonaws.com:9142
    local-datacenter: us-east-1
    keyspace-name: orderflow_events
    ssl: true
```
```java
@Table("click_events")
public record ClickEvent(@PrimaryKeyColumn(name="session_id", type=PARTITIONED) String sessionId,
                         @PrimaryKeyColumn(name="ts", type=CLUSTERED) Instant ts,
                         String productId, String action) {}
public interface ClickEventRepo extends CassandraRepository<ClickEvent, String> {}
```
**Overlap:** Keyspaces vs DynamoDB — both wide/key-value at scale. Use Keyspaces only if you specifically want the Cassandra API/CQL.

### Neptune — recommendation graph
```bash
aws neptune create-db-cluster --db-cluster-identifier orderflow-graph --engine neptune \
  --db-subnet-group-name orderflow-db-subnets --vpc-security-group-ids $DB_SG
```
Query with the Gremlin driver (Neptune speaks Gremlin/SPARQL, no Spring Data module):
```xml
<dependency><groupId>org.apache.tinkerpop</groupId><artifactId>gremlin-driver</artifactId><version>3.7.2</version></dependency>
```
```java
@Bean
GraphTraversalSource graph() {
    Cluster cluster = Cluster.build("orderflow-graph.cluster-xxx.neptune.amazonaws.com")
            .port(8182).enableSsl(true).create();
    return traversal().withRemote(DriverRemoteConnection.using(cluster));
}
// "bought together" — customers who bought X also bought:
List<Object> recos = g.V().has("product","id",productId)
        .in("bought").out("bought").dedup().limit(10).values("id").toList();
```

### Timestream — operational time-series
```bash
aws timestream-write create-database --database-name orderflow-metrics
aws timestream-write create-table --database-name orderflow-metrics --table-name order-rate
```
SDK v2 clients (no Spring wrapper):
```xml
<dependency><groupId>software.amazon.awssdk</groupId><artifactId>timestreamwrite</artifactId></dependency>
<dependency><groupId>software.amazon.awssdk</groupId><artifactId>timestreamquery</artifactId></dependency>
```
```java
@Service @RequiredArgsConstructor
public class MetricsWriter {
    private final TimestreamWriteClient tsw;
    public void recordOrderRate(double ordersPerMin) {
        var record = Record.builder().measureName("orders_per_min")
                .measureValue(String.valueOf(ordersPerMin))
                .measureValueType(MeasureValueType.DOUBLE)
                .time(String.valueOf(System.currentTimeMillis())).build();
        tsw.writeRecords(r -> r.databaseName("orderflow-metrics").tableName("order-rate").records(record));
    }
}
```

### Redshift — the BI warehouse
```bash
aws redshift-serverless create-namespace --namespace-name orderflow-wh
aws redshift-serverless create-workgroup --workgroup-name orderflow-wg \
  --namespace-name orderflow-wh --base-capacity 8 \
  --subnet-ids $PRIV_A $PRIV_B --security-group-ids $DB_SG
```
Connect over JDBC (analytics reads — not app-transaction traffic):
```xml
<dependency><groupId>com.amazon.redshift</groupId><artifactId>redshift-jdbc42</artifactId><version>2.1.0.30</version></dependency>
```
```java
@Bean
DataSource redshift() {
    var ds = new HikariDataSource();
    ds.setJdbcUrl("jdbc:redshift://orderflow-wg.xxx.us-east-1.redshift-serverless.amazonaws.com:5439/dev");
    ds.setMaximumPoolSize(4);   // warehouses want few, long-lived connections
    return ds;
}
// use JdbcTemplate for aggregate BI queries; keep it OUT of the request hot path
```
**Overlap:** Redshift vs Athena — a lean start is often just Athena over S3; add Redshift when you need consistent warehouse performance.

---

## 6. Amazon MQ — legacy broker bridge (RabbitMQ)

**Role:** a JMS/AMQP bridge to a legacy partner system that can't speak SQS.
```bash
aws mq create-broker --broker-name orderflow-mq --engine-type RABBITMQ \
  --engine-version 3.13 --host-instance-type mq.t3.micro --deployment-mode SINGLE_INSTANCE \
  --users Username=orderflow,Password=<from-secrets-manager> --publicly-accessible false \
  --subnet-ids $PRIV_A --security-groups $APP_SG
```
Standard Spring AMQP — Amazon MQ is just managed RabbitMQ:
```xml
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-amqp</artifactId></dependency>
```
```yaml
spring:
  rabbitmq:
    addresses: amqps://b-xxx.mq.us-east-1.amazonaws.com:5671
    username: orderflow
    password: ${mq-password}
```
```java
@Component @RequiredArgsConstructor
public class PartnerBridge {
    private final RabbitTemplate rabbit;
    public void forward(Object msg) { rabbit.convertAndSend("partner.exchange", "orders", msg); }

    @RabbitListener(queues = "partner.replies")
    public void onReply(String reply) { /* handle legacy partner reply */ }
}
```
**Overlap:** for new AWS-native work use SQS/SNS/EventBridge; Amazon MQ is for lift-and-shift of apps already speaking AMQP/JMS/MQTT.

---

## 7. SageMaker — custom model inference

**Role:** hosts the custom **fraud** and **recommendation** models. Training/endpoint creation is provisioning; the app just invokes the endpoint (like Bedrock).
```xml
<dependency><groupId>software.amazon.awssdk</groupId><artifactId>sagemakerruntime</artifactId></dependency>
```
```java
@Service @RequiredArgsConstructor
public class FraudScorer {
    private final SageMakerRuntimeClient sm;
    private final ObjectMapper mapper;

    public double score(Map<String,Object> features) {
        var body = SdkBytes.fromByteArray(json(features));
        var resp = sm.invokeEndpoint(r -> r.endpointName("orderflow-fraud")
                .contentType("application/json").body(body));
        return Double.parseDouble(resp.body().asUtf8String().trim());  // e.g. 0.0–1.0 risk
    }
    private byte[] json(Object o){ try { return mapper.writeValueAsBytes(o);} catch(Exception e){throw new IllegalStateException(e);} }
}
```
**EMR, Lake Formation, QuickSight** are data-platform/ops services with no Spring Boot application code — they're provisioned (Playbook Phase 7) and operated through their own consoles/APIs, feeding data to and from the S3 lake the app writes to.

---

## Where these fit the build plan

All of this is **Phase 3–4** work in the OrderFlow build plan (search/reviews/reco, then the analytics & AI platform) — you don't need any of it to have a working storefront. Add each only when its sub-domain becomes real, and remember the overlap notes: in a lean production system you'd pick one of each alternative pair (Kinesis *or* MSK, ECS *or* EKS, DynamoDB *or* DocumentDB/Keyspaces, Redshift *or* Athena) rather than running all of them.
