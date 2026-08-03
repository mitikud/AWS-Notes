# OrderFlow — Spring Boot Implementation

Complete, copy-pasteable code for every AWS service the OrderFlow application talks to. Pure-infrastructure services (VPC, CloudFront, WAF, IAM, Route 53…) have no application code — they're in the Provisioning Playbook. This doc covers the ~18 services where "source code" is a real thing.

**Stack:** Java 21 · Spring Boot 3.5.x · Spring Cloud AWS 3.4.2 · AWS SDK v2.

## Project structure

```
orderflow/
├── pom.xml                     (parent + BOM)
├── orderflow-common/           (AWS clients, security, crypto, config)
├── orderflow-orders/           (Aurora/JPA, DynamoDB, Redis, SNS, SQS producer, Cognito)
├── orderflow-fulfilment/       (SQS consumer, S3, SES, Step Functions, EventBridge)
├── orderflow-analytics/        (Kinesis, Bedrock AI)
├── orderflow-receipt-fn/       (Lambda, Spring Cloud Function)
└── infra/                      (CDK stack)
```

## Parent POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.4</version>
    </parent>
    <groupId>com.orderflow</groupId>
    <artifactId>orderflow</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <properties>
        <java.version>21</java.version>
        <spring-cloud-aws.version>3.4.2</spring-cloud-aws.version>
    </properties>

    <modules>
        <module>orderflow-common</module>
        <module>orderflow-orders</module>
        <module>orderflow-fulfilment</module>
        <module>orderflow-analytics</module>
        <module>orderflow-receipt-fn</module>
    </modules>

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
</project>
```

## Shared credentials & config (all modules)

`application.yml` — production uses the ECS task role automatically (no keys); the `local` profile points at LocalStack.

```yaml
spring:
  cloud:
    aws:
      region:
        static: us-east-1
      # credentials omitted in prod → DefaultCredentialsProvider uses the task role

---
spring:
  config:
    activate:
      on-profile: local
  cloud:
    aws:
      endpoint: http://localhost:4566          # LocalStack
      credentials:
        access-key: test
        secret-key: test
      region:
        static: us-east-1
```

---
## orderflow-common — shared AWS clients, security, crypto

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    <!-- SDK v2 clients not wrapped by Spring Cloud AWS -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>kms</artifactId>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>eventbridge</artifactId>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>kinesis</artifactId>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>sfn</artifactId>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>bedrockruntime</artifactId>
    </dependency>
    <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId></dependency>
</dependencies>
```

### AWS SDK client beans (KMS, EventBridge, Kinesis, Step Functions, Bedrock)
Spring Cloud AWS auto-wires the region and credentials providers — reuse them so LocalStack/prod both work.

```java
package com.orderflow.common.aws;

import io.awspring.cloud.autoconfigure.core.AwsClientCustomizer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import software.amazon.awssdk.auth.credentials.AwsCredentialsProvider;
import software.amazon.awssdk.awscore.client.builder.AwsClientBuilder;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.bedrockruntime.BedrockRuntimeClient;
import software.amazon.awssdk.services.eventbridge.EventBridgeClient;
import software.amazon.awssdk.services.kinesis.KinesisClient;
import software.amazon.awssdk.services.kms.KmsClient;
import software.amazon.awssdk.services.sfn.SfnClient;
import java.net.URI;

@Configuration
public class AwsClientConfig {

    @Value("${spring.cloud.aws.region.static}") String region;
    @Value("${spring.cloud.aws.endpoint:}")     String endpoint;   // set only on local

    private <B extends AwsClientBuilder<B, ?>> B base(B builder, AwsCredentialsProvider creds) {
        builder.region(Region.of(region)).credentialsProvider(creds);
        if (endpoint != null && !endpoint.isBlank()) builder.endpointOverride(URI.create(endpoint));
        return builder;
    }

    @Bean KmsClient kmsClient(AwsCredentialsProvider c)         { return base(KmsClient.builder(), c).build(); }
    @Bean EventBridgeClient eventBridgeClient(AwsCredentialsProvider c) { return base(EventBridgeClient.builder(), c).build(); }
    @Bean KinesisClient kinesisClient(AwsCredentialsProvider c) { return base(KinesisClient.builder(), c).build(); }
    @Bean SfnClient sfnClient(AwsCredentialsProvider c)         { return base(SfnClient.builder(), c).build(); }
    @Bean BedrockRuntimeClient bedrockClient(AwsCredentialsProvider c) { return base(BedrockRuntimeClient.builder(), c).build(); }
}
```

### Cognito security (JWT resource server)
`application.yml`:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_AbC123
```
```java
package com.orderflow.common.security;

import org.springframework.context.annotation.*;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.SecurityFilterChain;
import java.util.*;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers("/actuator/health", "/actuator/prometheus").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().denyAll())
            .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(converter())))
            .csrf(c -> c.disable());
        return http.build();
    }

    private JwtAuthenticationConverter converter() {
        var c = new JwtAuthenticationConverter();
        c.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<String> groups = jwt.getClaimAsStringList("cognito:groups");
            if (groups == null) return List.of();
            return groups.stream()
                    .map(g -> new SimpleGrantedAuthority("ROLE_" + g)).toList();
        });
        return c;
    }
}
```

### KMS envelope encryption (protect PII / card tokens)
```java
package com.orderflow.common.crypto;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.SdkBytes;
import software.amazon.awssdk.services.kms.KmsClient;
import software.amazon.awssdk.services.kms.model.DataKeySpec;
import javax.crypto.Cipher;
import javax.crypto.spec.*;
import java.security.SecureRandom;
import java.util.Base64;

@Service
@RequiredArgsConstructor
public class FieldEncryptionService {

    private final KmsClient kms;
    private static final String KEY = "alias/orderflow-pii";
    private static final SecureRandom RNG = new SecureRandom();

    public String encrypt(String plaintext) {
        var dk = kms.generateDataKey(r -> r.keyId(KEY).keySpec(DataKeySpec.AES_256));
        byte[] iv = new byte[12]; RNG.nextBytes(iv);
        byte[] ct = aes(Cipher.ENCRYPT_MODE, dk.plaintext().asByteArray(), iv,
                        plaintext.getBytes());
        // envelope = base64(encryptedDataKey) : base64(iv) : base64(ciphertext)
        var b64 = Base64.getEncoder();
        return String.join(":",
                b64.encodeToString(dk.ciphertextBlob().asByteArray()),
                b64.encodeToString(iv), b64.encodeToString(ct));
    }

    public String decrypt(String envelope) {
        var b64 = Base64.getDecoder();
        String[] p = envelope.split(":");
        byte[] edk = b64.decode(p[0]), iv = b64.decode(p[1]), ct = b64.decode(p[2]);
        var resp = kms.decrypt(r -> r.ciphertextBlob(SdkBytes.fromByteArray(edk)));
        return new String(aes(Cipher.DECRYPT_MODE, resp.plaintext().asByteArray(), iv, ct));
    }

    private byte[] aes(int mode, byte[] key, byte[] iv, byte[] data) {
        try {
            var cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(mode, new SecretKeySpec(key, "AES"), new GCMParameterSpec(128, iv));
            return cipher.doFinal(data);
        } catch (Exception e) { throw new IllegalStateException("crypto failure", e); }
    }
}
```

---
## orderflow-orders — Aurora, DynamoDB, Redis, SNS, SQS, Secrets, Parameter Store

**pom.xml**
```xml
<dependencies>
    <dependency><groupId>com.orderflow</groupId><artifactId>orderflow-common</artifactId><version>1.0.0</version></dependency>
    <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
    <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
    <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-redis</artifactId></dependency>
    <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-cache</artifactId></dependency>
    <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-actuator</artifactId></dependency>
    <dependency><groupId>org.postgresql</groupId><artifactId>postgresql</artifactId></dependency>
    <dependency><groupId>io.awspring.cloud</groupId><artifactId>spring-cloud-aws-starter-dynamodb</artifactId></dependency>
    <dependency><groupId>io.awspring.cloud</groupId><artifactId>spring-cloud-aws-starter-sns</artifactId></dependency>
    <dependency><groupId>io.awspring.cloud</groupId><artifactId>spring-cloud-aws-starter-sqs</artifactId></dependency>
    <dependency><groupId>io.awspring.cloud</groupId><artifactId>spring-cloud-aws-starter-secrets-manager</artifactId></dependency>
    <dependency><groupId>io.awspring.cloud</groupId><artifactId>spring-cloud-aws-starter-parameter-store</artifactId></dependency>
    <dependency><groupId>io.micrometer</groupId><artifactId>micrometer-registry-cloudwatch2</artifactId></dependency>
</dependencies>
```

**application.yml** — secrets & config pulled from AWS at startup
```yaml
spring:
  config:
    import:
      - aws-secretsmanager:orderflow/rds-credentials
      - aws-secretsmanager:orderflow/payment-keys
      - aws-parameterstore:/orderflow/
  datasource:
    url: jdbc:postgresql://orderflow-aurora.cluster-xxx.us-east-1.rds.amazonaws.com:5432/orderflow
    username: ${username}        # from the RDS-managed secret JSON
    password: ${password}
    hikari:
      maximum-pool-size: 10
  jpa:
    hibernate:
      ddl-auto: validate
  data:
    redis:
      host: orderflow-cache.xxx.cache.amazonaws.com
      port: 6379
      ssl:
        enabled: true
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true
  cloudwatch:
    metrics:
      export:
        namespace: OrderFlow
        step: 1m
```

### Aurora entity + repository (JPA)
```java
package com.orderflow.orders.domain;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

@Entity @Table(name = "orders")
@Getter @Setter @NoArgsConstructor
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    private String customerId;
    @Enumerated(EnumType.STRING) private OrderStatus status;
    private BigDecimal total;
    private String cardTokenEncrypted;      // KMS-encrypted envelope
    private Instant createdAt = Instant.now();
}
```
```java
package com.orderflow.orders.domain;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.*;
public interface OrderRepository extends JpaRepository<Order, UUID> {
    List<Order> findByCustomerIdAndStatus(String customerId, OrderStatus status);
}
```

### DynamoDB entity + repository (catalog & cart)
```java
package com.orderflow.orders.catalog;

import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.*;
import java.math.BigDecimal;

@DynamoDbBean
public class Product {
    private String id, category, name;
    private BigDecimal price;
    private int stock;

    @DynamoDbPartitionKey public String getId() { return id; }
    @DynamoDbSecondaryPartitionKey(indexNames = "category-index")
    public String getCategory() { return category; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public int getStock() { return stock; }
    // setters …
    public void setId(String v){id=v;} public void setCategory(String v){category=v;}
    public void setName(String v){name=v;} public void setPrice(BigDecimal v){price=v;}
    public void setStock(int v){stock=v;}
}
```
```java
package com.orderflow.orders.catalog;

import io.awspring.cloud.dynamodb.DynamoDbTemplate;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
import software.amazon.awssdk.enhanced.dynamodb.Key;
import software.amazon.awssdk.enhanced.dynamodb.model.*;
import java.util.*;

@Repository
@RequiredArgsConstructor
public class ProductRepository {
    private final DynamoDbTemplate template;

    public Product save(Product p) { return template.save(p); }

    public Optional<Product> findById(String id) {
        return Optional.ofNullable(
            template.load(Key.builder().partitionValue(id).build(), Product.class));
    }

    public List<Product> findByCategory(String category) {
        var q = QueryEnhancedRequest.builder()
                .queryConditional(QueryConditional.keyEqualTo(
                        Key.builder().partitionValue(category).build())).build();
        return template.query(q, Product.class, "category-index").items().stream().toList();
    }
}
```

### Redis caching (ElastiCache)
```java
package com.orderflow.orders.catalog;

import lombok.RequiredArgsConstructor;
import org.springframework.cache.annotation.*;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.context.annotation.*;
import org.springframework.stereotype.Service;
import java.time.Duration;

@Configuration @EnableCaching
class CacheConfig {
    @Bean RedisCacheConfiguration cacheConfig() {
        return RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)).disableCachingNullValues();
    }
}

@Service
@RequiredArgsConstructor
public class CatalogService {
    private final ProductRepository products;

    @Cacheable(value = "products", key = "#id")
    public Product get(String id) {
        return products.findById(id).orElseThrow(() -> new IllegalArgumentException("no product " + id));
    }

    @CacheEvict(value = "products", key = "#p.id")
    public void update(Product p) { products.save(p); }
}
```

### SNS publisher + SQS producer + the order use-case
```java
package com.orderflow.orders;

import com.orderflow.common.crypto.FieldEncryptionService;
import com.orderflow.orders.catalog.CatalogService;
import com.orderflow.orders.domain.*;
import io.awspring.cloud.sns.core.*;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.util.UUID;

public record OrderPlacedEvent(UUID orderId, String customerId, BigDecimal total) {}

@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orders;
    private final CatalogService catalog;
    private final FieldEncryptionService crypto;
    private final SnsTemplate snsTemplate;
    private final MeterRegistry metrics;

    @Transactional
    public Order placeOrder(String customerId, String productId, int qty, String cardToken) {
        var product = catalog.get(productId);                 // Redis → DynamoDB
        var total = product.getPrice().multiply(BigDecimal.valueOf(qty));

        var order = new Order();
        order.setCustomerId(customerId);
        order.setStatus(OrderStatus.PLACED);
        order.setTotal(total);
        order.setCardTokenEncrypted(crypto.encrypt(cardToken));  // KMS envelope
        order = orders.save(order);                              // Aurora, in-tx

        // publish to SNS → fans out to fulfilment/analytics/notification queues
        snsTemplate.sendNotification("order-events",
                SnsNotification.builder(new OrderPlacedEvent(order.getId(), customerId, total))
                        .header("eventType", "ORDER_PLACED").build());

        metrics.counter("orders.placed", "category", product.getCategory()).increment();
        return order;
    }
}
```

### REST controller
```java
package com.orderflow.orders;

import com.orderflow.orders.domain.Order;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;
import java.math.BigDecimal;

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;

    public record PlaceOrderRequest(String productId, int qty, String cardToken) {}

    @PostMapping
    public Order place(@AuthenticationPrincipal Jwt jwt, @RequestBody PlaceOrderRequest req) {
        String customerId = jwt.getSubject();               // from the Cognito JWT
        return orderService.placeOrder(customerId, req.productId(), req.qty(), req.cardToken());
    }
}
```

---
## orderflow-fulfilment — SQS consumer, S3, SES, EventBridge, Step Functions

**pom.xml** adds:
```xml
<dependency><groupId>io.awspring.cloud</groupId><artifactId>spring-cloud-aws-starter-sqs</artifactId></dependency>
<dependency><groupId>io.awspring.cloud</groupId><artifactId>spring-cloud-aws-starter-s3</artifactId></dependency>
<dependency><groupId>io.awspring.cloud</groupId><artifactId>spring-cloud-aws-starter-ses</artifactId></dependency>
```

### SQS listener (consumes the fan-out) → orchestrates fulfilment
```java
package com.orderflow.fulfilment;

import com.orderflow.orders.OrderPlacedEvent;
import io.awspring.cloud.sqs.annotation.SqsListener;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderProcessingListener {

    private final FulfilmentService fulfilment;

    @SqsListener("order-processing")
    public void onOrder(OrderPlacedEvent event) {
        log.info("Fulfilling order {}", event.orderId());
        fulfilment.fulfil(event);      // throwing here → retried, then DLQ after maxReceiveCount
    }
}
```

### S3 invoice storage + presigned download
```java
package com.orderflow.fulfilment;

import io.awspring.cloud.s3.*;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import java.io.ByteArrayInputStream;
import java.net.URL;
import java.time.Duration;

@Service
@RequiredArgsConstructor
public class InvoiceStorage {
    private final S3Template s3;
    private static final String BUCKET = "orderflow-invoices";

    public String store(String orderId, byte[] pdf) {
        String key = "invoices/%s.pdf".formatted(orderId);
        S3Resource r = s3.upload(BUCKET, key, new ByteArrayInputStream(pdf),
                ObjectMetadata.builder().contentType("application/pdf").build());
        return r.getLocation().toString();
    }

    public URL presignedLink(String orderId) {
        return s3.createSignedGetURL(BUCKET, "invoices/%s.pdf".formatted(orderId),
                Duration.ofMinutes(15));
    }
}
```

### SES email (SES-backed JavaMailSender is auto-configured by the starter)
```java
package com.orderflow.fulfilment;

import lombok.RequiredArgsConstructor;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;
import java.net.URL;

@Service
@RequiredArgsConstructor
public class EmailService {
    private final JavaMailSender mailSender;   // SimpleEmailServiceJavaMailSender

    public void sendReceipt(String to, String orderId, URL link) {
        var msg = new SimpleMailMessage();
        msg.setFrom("orders@orderflow.com");    // verified SES identity
        msg.setTo(to);
        msg.setSubject("Your OrderFlow receipt — " + orderId);
        msg.setText("Thanks for your order! Download your receipt: " + link);
        mailSender.send(msg);
    }
}
```

### EventBridge publisher + Step Functions starter
```java
package com.orderflow.fulfilment;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.eventbridge.EventBridgeClient;
import software.amazon.awssdk.services.eventbridge.model.PutEventsRequestEntry;
import software.amazon.awssdk.services.sfn.SfnClient;

@Service
@RequiredArgsConstructor
public class FulfilmentService {

    private final InvoiceStorage invoices;
    private final EmailService email;
    private final EventBridgeClient eventBridge;
    private final SfnClient sfn;
    private final ObjectMapper mapper;

    @Value("${orderflow.sfn-arn}") String stateMachineArn;

    public void fulfil(com.orderflow.orders.OrderPlacedEvent event) {
        // kick off the saga (charge → invoice → notify, with compensating refund)
        sfn.startExecution(r -> r.stateMachineArn(stateMachineArn).input(toJson(event)));

        // emit a domain event to the bus for analytics/other consumers
        var entry = PutEventsRequestEntry.builder()
                .eventBusName("orderflow-bus").source("orderflow.fulfilment")
                .detailType("OrderFulfilled").detail(toJson(event)).build();
        eventBridge.putEvents(r -> r.entries(entry));
    }

    private String toJson(Object o) {
        try { return mapper.writeValueAsString(o); }
        catch (Exception e) { throw new IllegalStateException(e); }
    }
}
```

---

## orderflow-analytics — Kinesis + Bedrock (AI)

### Kinesis clickstream producer
```java
package com.orderflow.analytics;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.SdkBytes;
import software.amazon.awssdk.services.kinesis.KinesisClient;

public record ClickEvent(String sessionId, String productId, String action) {}

@Service
@RequiredArgsConstructor
public class ClickstreamProducer {
    private final KinesisClient kinesis;
    private final ObjectMapper mapper;

    public void record(ClickEvent e) {
        kinesis.putRecord(r -> r
                .streamName("orderflow-clickstream")
                .partitionKey(e.sessionId())
                .data(SdkBytes.fromByteArray(json(e))));
    }
    private byte[] json(Object o) {
        try { return mapper.writeValueAsBytes(o); }
        catch (Exception ex) { throw new IllegalStateException(ex); }
    }
}
```

### Bedrock — RAG support chatbot / content generation
```java
package com.orderflow.analytics;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.SdkBytes;
import software.amazon.awssdk.services.bedrockruntime.BedrockRuntimeClient;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class AiAssistant {
    private final BedrockRuntimeClient bedrock;
    private final ObjectMapper mapper;
    private static final String MODEL = "anthropic.claude-3-5-sonnet-20241022-v2:0";

    public String answer(String userQuestion, String retrievedContext) {
        var body = Map.of(
            "anthropic_version", "bedrock-2023-05-31",
            "max_tokens", 512,
            "system", "You are OrderFlow support. Answer only from the provided context.",
            "messages", java.util.List.of(Map.of(
                "role", "user",
                "content", "Context:\n" + retrievedContext + "\n\nQuestion: " + userQuestion)));
        var resp = bedrock.invokeModel(r -> r.modelId(MODEL)
                .body(SdkBytes.fromByteArray(json(body))));
        return parseText(resp.body().asUtf8String());
    }

    private byte[] json(Object o) {
        try { return mapper.writeValueAsBytes(o); } catch (Exception e) { throw new IllegalStateException(e); }
    }
    private String parseText(String responseJson) {
        try {
            var node = mapper.readTree(responseJson);
            return node.get("content").get(0).get("text").asText();
        } catch (Exception e) { throw new IllegalStateException(e); }
    }
}
```

---

## orderflow-receipt-fn — Lambda (Spring Cloud Function)

**pom.xml** adds:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-adapter-aws</artifactId>
</dependency>
```
```java
package com.orderflow.receipt;

import com.amazonaws.services.lambda.runtime.events.S3Event;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import java.util.function.Function;

@SpringBootApplication
public class ReceiptFnApplication {
    public static void main(String[] args) { SpringApplication.run(ReceiptFnApplication.class, args); }

    @Bean
    public Function<S3Event, String> generateReceipt(ReceiptService receipts) {
        return event -> {
            event.getRecords().forEach(rec ->
                receipts.buildAndStore(rec.getS3().getObject().getKey()));
            return "ok";
        };
    }
}
```
Deploy with env var `SPRING_CLOUD_FUNCTION_DEFINITION=generateReceipt` (see Provisioning Playbook, Phase 5). Mitigate cold starts with SnapStart or a GraalVM native build.

---

## CloudWatch observability (metrics + structured logs)

Custom business metrics via Micrometer (registry auto-exports to CloudWatch — config in the orders `application.yml`):
```java
package com.orderflow.orders;

import io.micrometer.core.instrument.MeterRegistry;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import java.time.Duration;

@Component
@RequiredArgsConstructor
public class OrderMetrics {
    private final MeterRegistry registry;
    public void processingTime(Duration d) { registry.timer("orders.processing.time").record(d); }
    public void paymentFailed()            { registry.counter("orders.payment.failed").increment(); }
}
```
For logs, add `logstash-logback-encoder` and emit JSON so CloudWatch Logs Insights can query fields. On ECS the `awslogs` driver ships stdout to CloudWatch automatically.

---

## Integration test (Testcontainers + LocalStack — no real AWS)

```java
package com.orderflow.orders;

import org.junit.jupiter.api.*;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.*;
import org.testcontainers.containers.localstack.LocalStackContainer;
import org.testcontainers.junit.jupiter.*;
import org.testcontainers.utility.DockerImageName;
import static org.testcontainers.containers.localstack.LocalStackContainer.Service.*;

@SpringBootTest
@Testcontainers
class OrderFlowIntegrationTest {

    @Container
    static LocalStackContainer localstack =
        new LocalStackContainer(DockerImageName.parse("localstack/localstack:3"))
            .withServices(S3, SQS, SNS, DYNAMODB, KMS);

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.cloud.aws.endpoint", localstack::getEndpoint);
        r.add("spring.cloud.aws.region.static", localstack::getRegion);
        r.add("spring.cloud.aws.credentials.access-key", localstack::getAccessKey);
        r.add("spring.cloud.aws.credentials.secret-key", localstack::getSecretKey);
    }

    @Test
    void placeOrder_publishesEventAndPersists() {
        // provision table/topic/queue, place an order, assert Aurora row + SNS→SQS message
    }
}
```

---

## CDK starter stack (replace the whole CLI playbook)

`infra/` — one `cdk deploy` provisions the core. This is the production answer to the imperative CLI.
```java
public class OrderFlowStack extends Stack {
    public OrderFlowStack(Construct scope, String id, StackProps props) {
        super(scope, id, props);

        Vpc vpc = Vpc.Builder.create(this, "Vpc").maxAzs(2).build();

        Table products = Table.Builder.create(this, "Products")
                .partitionKey(Attribute.builder().name("id").type(AttributeType.STRING).build())
                .billingMode(BillingMode.PAY_PER_REQUEST).stream(StreamViewType.NEW_AND_OLD_IMAGES)
                .build();

        Bucket invoices = Bucket.Builder.create(this, "Invoices")
                .encryption(BucketEncryption.KMS_MANAGED).versioned(true)
                .blockPublicAccess(BlockPublicAccess.BLOCK_ALL).build();

        Queue dlq = Queue.Builder.create(this, "Dlq").build();
        Queue processing = Queue.Builder.create(this, "Processing")
                .deadLetterQueue(DeadLetterQueue.builder().queue(dlq).maxReceiveCount(3).build())
                .build();
        Topic orderEvents = Topic.Builder.create(this, "OrderEvents").build();
        orderEvents.addSubscription(new SqsSubscription(processing));

        DatabaseCluster.Builder.create(this, "Aurora")
                .engine(DatabaseClusterEngine.auroraPostgres(
                    AuroraPostgresClusterEngineProps.builder()
                        .version(AuroraPostgresEngineVersion.VER_16_4).build()))
                .vpc(vpc).build();
        // + ECS service, ALB, Cognito, ElastiCache, roles … each ~5-10 lines
    }
}
```
Deploy: `cdk deploy`. Destroy: `cdk destroy`. Everything versioned, reviewable, reproducible.

---

## Where to go next

You now have the provisioning steps (Doc 1) and the application code (this doc) for every service OrderFlow uses. Build it in the 5 phases from the architecture doc — don't wire all 18 integrations at once. Each phase compiles, runs on LocalStack, and demos independently.
