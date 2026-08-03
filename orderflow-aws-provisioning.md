# OrderFlow — AWS Provisioning Playbook

Step-by-step creation of every service in OrderFlow. Commands use the **AWS CLI** (reproducible and copy-pasteable); console equivalents are noted where the console is genuinely easier.

## Read this first

**CLI now, IaC later.** These imperative commands are for *learning what each resource is*. In production you never click-ops or run one-off CLI — you define everything in **AWS CDK** or **Terraform** so it's versioned, reviewable, and reproducible. Once you understand the resources below, port them to CDK (a starter stack is in the Spring Boot doc). Treat this playbook as the "what the IaC actually creates" reference.

**Order matters.** Dependencies must exist first: account guardrails → network → security/keys → data → storage → compute → messaging → analytics → observability → edge/CI. Follow the phases in order.

**Prerequisites**
```bash
aws --version                 # v2.x
aws configure --profile orderflow-dev   # set key, secret, region us-east-1, json
export AWS_PROFILE=orderflow-dev
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

**Cost warning.** A full build costs real money. NAT Gateways, RDS/Aurora, ElastiCache, OpenSearch, Redshift, MSK, and NAT are the big hourly burners. Set the budget alarm below **before anything else**, use Free Tier / Spot where possible, and run the teardown section nightly.

---

## Phase 0 — Account guardrails (do this first)

### Budget alarm — before you create anything
```bash
cat > budget.json <<'EOF'
{ "BudgetName": "orderflow-monthly", "BudgetLimit": {"Amount": "50", "Unit": "USD"},
  "TimeUnit": "MONTHLY", "BudgetType": "COST" }
EOF
cat > notify.json <<'EOF'
[ { "Notification": {"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80},
    "Subscribers":[{"SubscriptionType":"EMAIL","Address":"you@example.com"}] } ]
EOF
aws budgets create-budget --account-id $ACCOUNT_ID \
  --budget file://budget.json --notifications-with-subscribers file://notify.json
```

### Organizations (multi-account) — concept + enable
For real isolation you'd run separate accounts (`prod`, `staging`, `dev`, `security`, `analytics`). Enable Organizations from the management account, then use **Control Tower** for a governed landing zone (console-driven). For learning, one account is fine — just know the production pattern is account-per-environment with **SCPs** capping permissions.
```bash
aws organizations create-organization --feature-set ALL   # from the management account
```

### IAM Identity Center (workforce SSO)
Enable in the console (Identity Center) → create permission sets (e.g. `AdministratorAccess`, `PowerUserAccess`) → assign users to accounts. This replaces IAM users for humans; everyone gets temporary credentials.

### IAM — the role pattern you'll reuse everywhere
Every workload gets a role it *assumes* — no long-lived keys. Example: the ECS task role for the orders service.
```bash
cat > ecs-trust.json <<'EOF'
{ "Version":"2012-10-17","Statement":[
  {"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name orderflow-orders-task \
  --assume-role-policy-document file://ecs-trust.json

# Attach a scoped inline policy (least privilege — only what orders needs)
cat > orders-policy.json <<EOF
{ "Version":"2012-10-17","Statement":[
  {"Effect":"Allow","Action":["dynamodb:GetItem","dynamodb:PutItem","dynamodb:Query"],
   "Resource":"arn:aws:dynamodb:$AWS_REGION:$ACCOUNT_ID:table/orderflow-*"},
  {"Effect":"Allow","Action":["sns:Publish"],"Resource":"arn:aws:sns:$AWS_REGION:$ACCOUNT_ID:order-events"},
  {"Effect":"Allow","Action":["secretsmanager:GetSecretValue"],
   "Resource":"arn:aws:secretsmanager:$AWS_REGION:$ACCOUNT_ID:secret:orderflow/*"},
  {"Effect":"Allow","Action":["kms:Decrypt","kms:GenerateDataKey"],"Resource":"*"}]}
EOF
aws iam put-role-policy --role-name orderflow-orders-task \
  --policy-name orders-access --policy-document file://orders-policy.json
```
Repeat the pattern per service with only the actions that service needs. **Explicit deny always wins; default is deny.**

---

## Phase 1 — Network foundation (VPC)

### Create the VPC and subnets across 2 AZs
```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=orderflow-vpc}]' \
  --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

# Public subnets (ALB, NAT) in 2 AZs
PUB_A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --availability-zone ${AWS_REGION}a --query Subnet.SubnetId --output text)
PUB_B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 \
  --availability-zone ${AWS_REGION}b --query Subnet.SubnetId --output text)
# Private subnets (ECS, RDS, cache)
PRIV_A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 \
  --availability-zone ${AWS_REGION}a --query Subnet.SubnetId --output text)
PRIV_B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 \
  --availability-zone ${AWS_REGION}b --query Subnet.SubnetId --output text)
```

### Internet Gateway + NAT Gateway + routes
```bash
IGW_ID=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Public route table → IGW
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_A
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_B

# NAT Gateway (lets private subnets reach out) — THIS COSTS ~$32/mo, delete when idle
EIP=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id $PUB_A --allocation-id $EIP \
  --query NatGateway.NatGatewayId --output text)
# wait, then private route table → NAT
PRIV_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $PRIV_RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_ID
aws ec2 associate-route-table --route-table-id $PRIV_RT --subnet-id $PRIV_A
aws ec2 associate-route-table --route-table-id $PRIV_RT --subnet-id $PRIV_B
```

### Security groups (stateful, instance-level)
```bash
ALB_SG=$(aws ec2 create-security-group --group-name orderflow-alb --description "ALB" \
  --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0

APP_SG=$(aws ec2 create-security-group --group-name orderflow-app --description "ECS tasks" \
  --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 8080 --source-group $ALB_SG

DB_SG=$(aws ec2 create-security-group --group-name orderflow-db --description "RDS/cache" \
  --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 5432 --source-group $APP_SG
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 6379 --source-group $APP_SG
```

### VPC endpoints (keep S3/DynamoDB traffic off the internet — security + cost)
```bash
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --service-name com.amazonaws.$AWS_REGION.s3 \
  --route-table-ids $PRIV_RT --vpc-endpoint-type Gateway
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --service-name com.amazonaws.$AWS_REGION.dynamodb \
  --route-table-ids $PRIV_RT --vpc-endpoint-type Gateway
```

### ACM certificate + Route 53
```bash
# Hosted zone (if you own the domain)
aws route53 create-hosted-zone --name orderflow.com --caller-reference $(date +%s)
# Certificate (DNS-validated) — for CloudFront it MUST be in us-east-1
aws acm request-certificate --domain-name orderflow.com \
  --subject-alternative-names '*.orderflow.com' --validation-method DNS --region us-east-1
```
Then add the CNAME validation record ACM gives you to the hosted zone. Certs on AWS-integrated services auto-renew.

---

## Phase 2 — Data layer

### KMS keys (create before data — data references them for encryption)
```bash
KEY_ID=$(aws kms create-key --description "orderflow-pii" \
  --query KeyMetadata.KeyId --output text)
aws kms create-alias --alias-name alias/orderflow-pii --target-key-id $KEY_ID
aws kms enable-key-rotation --key-id $KEY_ID
```

### Aurora PostgreSQL (orders/payments — ACID)
```bash
# DB subnet group across the private subnets
aws rds create-db-subnet-group --db-subnet-group-name orderflow-db-subnets \
  --db-subnet-group-description "private" --subnet-ids $PRIV_A $PRIV_B

# Aurora cluster (encrypted with our KMS key), then an instance
aws rds create-db-cluster --db-cluster-identifier orderflow-aurora \
  --engine aurora-postgresql --engine-version 16.4 \
  --master-username orderflow --manage-master-user-password \
  --db-subnet-group-name orderflow-db-subnets --vpc-security-group-ids $DB_SG \
  --storage-encrypted --kms-key-id alias/orderflow-pii
aws rds create-db-instance --db-instance-identifier orderflow-aurora-1 \
  --db-cluster-identifier orderflow-aurora --engine aurora-postgresql \
  --db-instance-class db.t4g.medium
```
`--manage-master-user-password` makes RDS store the password **in Secrets Manager and rotate it** — no password ever in your hands. Add a read replica later with another `create-db-instance` in the same cluster.

### DynamoDB (catalog, carts, sessions, idempotency)
```bash
aws dynamodb create-table --table-name orderflow-products \
  --attribute-definitions AttributeName=id,AttributeType=S AttributeName=category,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --global-secondary-indexes '[{"IndexName":"category-index","KeySchema":[{"AttributeName":"category","KeyType":"HASH"}],"Projection":{"ProjectionType":"ALL"}}]' \
  --billing-mode PAY_PER_REQUEST

aws dynamodb create-table --table-name orderflow-carts \
  --attribute-definitions AttributeName=customerId,AttributeType=S \
  --key-schema AttributeName=customerId,KeyType=HASH --billing-mode PAY_PER_REQUEST

# Enable Streams (feeds Lambda) on the catalog table
aws dynamodb update-table --table-name orderflow-products \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

### ElastiCache Redis (cache, sessions, locks)
```bash
aws elasticache create-cache-subnet-group --cache-subnet-group-name orderflow-cache-subnets \
  --cache-subnet-group-description "private" --subnet-ids $PRIV_A $PRIV_B
aws elasticache create-replication-group --replication-group-id orderflow-cache \
  --replication-group-description "orderflow" --engine redis \
  --cache-node-type cache.t4g.micro --num-node-groups 1 --replicas-per-node-group 1 \
  --cache-subnet-group-name orderflow-cache-subnets --security-group-ids $DB_SG \
  --transit-encryption-enabled
```

### OpenSearch (search, logs, vectors)
```bash
aws opensearch create-domain --domain-name orderflow-search \
  --engine-version OpenSearch_2.13 \
  --cluster-config InstanceType=t3.small.search,InstanceCount=1 \
  --ebs-options EBSEnabled=true,VolumeType=gp3,VolumeSize=10 \
  --encryption-at-rest-options Enabled=true \
  --node-to-node-encryption-options Enabled=true
```

### The purpose-built extras (brief — alternatives / niche)
```bash
# DocumentDB (reviews / CMS documents) — Mongo-compatible
aws docdb create-db-cluster --db-cluster-identifier orderflow-reviews \
  --engine docdb --master-username orderflow --manage-master-user-password \
  --db-subnet-group-name orderflow-db-subnets --vpc-security-group-ids $DB_SG
# Neptune (reco graph): aws neptune create-db-cluster --engine neptune ...
# Timestream (metrics):  aws timestream-write create-database --database-name orderflow-metrics
# Keyspaces (Cassandra): create keyspace/table via CQL or console
```
Provision these only when you reach the AI/analytics phase; each is an hourly cost.

---

## Phase 3 — Config & secrets

### Secrets Manager (payment keys — Aurora creds are already managed above)
```bash
aws secretsmanager create-secret --name orderflow/payment-keys \
  --secret-string '{"stripe-key":"sk_test_xxx","chapa-key":"CHASECK_xxx"}' \
  --kms-key-id alias/orderflow-pii
```

### Parameter Store (feature flags, tunables)
```bash
aws ssm put-parameter --name /orderflow/features/express-checkout --value "true" --type String
aws ssm put-parameter --name /orderflow/pricing/free-ship-threshold --value "50" --type String
aws ssm put-parameter --name /orderflow/payment/webhook-secret --value "whsec_xxx" --type SecureString
```

### Cognito (customer auth)
```bash
POOL_ID=$(aws cognito-idp create-user-pool --pool-name orderflow-users \
  --auto-verified-attributes email \
  --policies '{"PasswordPolicy":{"MinimumLength":8,"RequireUppercase":true,"RequireNumbers":true}}' \
  --query UserPool.Id --output text)

aws cognito-idp create-user-pool-client --user-pool-id $POOL_ID \
  --client-name orderflow-web --no-generate-secret \
  --allowed-o-auth-flows code --allowed-o-auth-scopes openid email profile \
  --allowed-o-auth-flows-user-pool-client \
  --callback-urls https://orderflow.com/callback

# Groups → become roles in your app
aws cognito-idp create-group --user-pool-id $POOL_ID --group-name ADMIN
echo "Issuer: https://cognito-idp.$AWS_REGION.amazonaws.com/$POOL_ID"   # goes in application.yml
```

---

## Phase 4 — Storage

### S3 buckets (invoices, images, data lake, static site, logs)
```bash
for B in invoices images datalake logs; do
  aws s3api create-bucket --bucket orderflow-$B-$ACCOUNT_ID --region $AWS_REGION
  aws s3api put-public-access-block --bucket orderflow-$B-$ACCOUNT_ID \
    --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
  aws s3api put-bucket-encryption --bucket orderflow-$B-$ACCOUNT_ID \
    --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms","KMSMasterKeyID":"alias/orderflow-pii"}}]}'
  aws s3api put-bucket-versioning --bucket orderflow-$B-$ACCOUNT_ID \
    --versioning-configuration Status=Enabled
done

# Lifecycle: move invoices to Glacier after 90 days
aws s3api put-bucket-lifecycle-configuration --bucket orderflow-invoices-$ACCOUNT_ID \
  --lifecycle-configuration '{"Rules":[{"ID":"archive","Status":"Enabled","Filter":{"Prefix":"invoices/"},"Transitions":[{"Days":90,"StorageClass":"GLACIER"}]}]}'
```

### ECR (container images)
```bash
for SVC in orders fulfilment catalog notification admin; do
  aws ecr create-repository --repository-name orderflow/$SVC \
    --image-scanning-configuration scanOnPush=true
done
aws ecr get-login-password | docker login --username AWS \
  --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

---

## Phase 5 — Compute

### ALB + target group
```bash
ALB_ARN=$(aws elbv2 create-load-balancer --name orderflow-alb \
  --subnets $PUB_A $PUB_B --security-groups $ALB_SG \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
TG_ARN=$(aws elbv2 create-target-group --name orderflow-orders-tg \
  --protocol HTTP --port 8080 --vpc-id $VPC_ID --target-type ip \
  --health-check-path /actuator/health \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### ECS cluster + task definition + Fargate service
```bash
aws ecs create-cluster --cluster-name orderflow

cat > taskdef.json <<EOF
{ "family":"orderflow-orders","networkMode":"awsvpc","requiresCompatibilities":["FARGATE"],
  "cpu":"512","memory":"1024",
  "executionRoleArn":"arn:aws:iam::$ACCOUNT_ID:role/ecsTaskExecutionRole",
  "taskRoleArn":"arn:aws:iam::$ACCOUNT_ID:role/orderflow-orders-task",
  "containerDefinitions":[{
    "name":"orders","image":"$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/orderflow/orders:latest",
    "portMappings":[{"containerPort":8080}],
    "logConfiguration":{"logDriver":"awslogs","options":{
      "awslogs-group":"/ecs/orderflow-orders","awslogs-region":"$AWS_REGION","awslogs-stream-prefix":"orders"}},
    "environment":[{"name":"SPRING_PROFILES_ACTIVE","value":"prod"}]
  }] }
EOF
aws logs create-log-group --log-group-name /ecs/orderflow-orders
aws ecs register-task-definition --cli-input-json file://taskdef.json

aws ecs create-service --cluster orderflow --service-name orders \
  --task-definition orderflow-orders --desired-count 2 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$PRIV_A,$PRIV_B],securityGroups=[$APP_SG]}" \
  --load-balancers targetGroupArn=$TG_ARN,containerName=orders,containerPort=8080
```

### Auto scaling (ECS service)
```bash
aws application-autoscaling register-scalable-target --service-namespace ecs \
  --resource-id service/orderflow/orders --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 10
aws application-autoscaling put-scaling-policy --service-namespace ecs \
  --resource-id service/orderflow/orders --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu70 --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{"TargetValue":70,"PredefinedMetricSpecification":{"PredefinedMetricType":"ECSServiceAverageCPUUtilization"}}'
```

### Lambda (receipt/thumbnail generator)
```bash
# package your jar/zip first, then:
aws lambda create-function --function-name orderflow-receipt \
  --runtime java21 --handler org.springframework.cloud.function.adapter.aws.FunctionInvoker \
  --role arn:aws:iam::$ACCOUNT_ID:role/orderflow-lambda \
  --timeout 30 --memory-size 1024 \
  --environment 'Variables={SPRING_CLOUD_FUNCTION_DEFINITION=generateReceipt}' \
  --zip-file fileb://receipt-fn.zip
# trigger from S3 object-created events
aws s3api put-bucket-notification-configuration --bucket orderflow-invoices-$ACCOUNT_ID \
  --notification-configuration '{"LambdaFunctionConfigurations":[{"LambdaFunctionArn":"arn:aws:lambda:'$AWS_REGION':'$ACCOUNT_ID':function:orderflow-receipt","Events":["s3:ObjectCreated:*"]}]}'
```

### The compute alternatives (brief)
```bash
# EC2 + Batch compute env (bulk image jobs on Spot)
aws batch create-compute-environment --compute-environment-name orderflow-batch \
  --type MANAGED --compute-resources type=FARGATE_SPOT,maxvCpus=16,subnets=$PRIV_A,securityGroupIds=$APP_SG
# EKS (ML platform):        eksctl create cluster --name orderflow-ml --fargate
# Elastic Beanstalk (site): eb init && eb create orderflow-site
```

---

## Phase 6 — Messaging & events

### SQS (+ DLQ)
```bash
DLQ_URL=$(aws sqs create-queue --queue-name order-processing-dlq --query QueueUrl --output text)
DLQ_ARN=$(aws sqs get-queue-attributes --queue-url $DLQ_URL --attribute-names QueueArn \
  --query Attributes.QueueArn --output text)
aws sqs create-queue --queue-name order-processing \
  --attributes '{"RedrivePolicy":"{\"deadLetterTargetArn\":\"'$DLQ_ARN'\",\"maxReceiveCount\":\"3\"}","VisibilityTimeout":"60"}'
```

### SNS (+ SNS→SQS fan-out)
```bash
TOPIC_ARN=$(aws sns create-topic --name order-events --query TopicArn --output text)
Q_ARN=$(aws sqs get-queue-attributes --queue-url \
  $(aws sqs get-queue-url --queue-name order-processing --query QueueUrl --output text) \
  --attribute-names QueueArn --query Attributes.QueueArn --output text)
aws sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint $Q_ARN \
  --attributes '{"RawMessageDelivery":"true","FilterPolicy":"{\"eventType\":[\"ORDER_PLACED\"]}"}'
# (also add a queue policy allowing SNS to send to the queue)
```

### EventBridge (bus, custom rule, scheduled job)
```bash
aws events create-event-bus --name orderflow-bus
# scheduled nightly reconciliation → Lambda
aws events put-rule --name nightly-reconcile --schedule-expression 'cron(0 2 * * ? *)'
aws events put-targets --rule nightly-reconcile \
  --targets 'Id=1,Arn=arn:aws:lambda:'$AWS_REGION':'$ACCOUNT_ID':function:orderflow-reconcile'
```

### Kinesis (clickstream)
```bash
aws kinesis create-stream --stream-name orderflow-clickstream --shard-count 1
```

### Step Functions (fulfilment saga)
```bash
cat > saga.json <<'EOF'
{ "Comment":"fulfilment saga","StartAt":"Charge",
  "States":{
    "Charge":{"Type":"Task","Resource":"arn:aws:lambda:REGION:ACCT:function:charge","Next":"Invoice","Catch":[{"ErrorEquals":["States.ALL"],"Next":"Refund"}]},
    "Invoice":{"Type":"Task","Resource":"arn:aws:lambda:REGION:ACCT:function:invoice","Next":"Notify"},
    "Notify":{"Type":"Task","Resource":"arn:aws:lambda:REGION:ACCT:function:notify","End":true},
    "Refund":{"Type":"Task","Resource":"arn:aws:lambda:REGION:ACCT:function:refund","End":true}}}
EOF
aws stepfunctions create-state-machine --name orderflow-fulfilment \
  --definition file://saga.json --role-arn arn:aws:iam::$ACCOUNT_ID:role/orderflow-sfn
```

### SES (email — verify identity first)
```bash
aws sesv2 create-email-identity --email-identity orders@orderflow.com
# verify the domain + set up DKIM in the console; new accounts start in the sandbox
```

### Alternatives (brief)
```bash
# MSK (Kafka/CDC):  aws kafka create-cluster-v2 --cluster-name orderflow-cdc ...
# Amazon MQ:        aws mq create-broker --broker-name orderflow-mq --engine-type RABBITMQ ...
```

---

## Phase 7 — Analytics & AI

### Glue (catalog + crawler over the S3 lake)
```bash
aws glue create-database --database-input '{"Name":"orderflow_lake"}'
aws glue create-crawler --name orderflow-crawler \
  --role arn:aws:iam::$ACCOUNT_ID:role/orderflow-glue \
  --database-name orderflow_lake \
  --targets '{"S3Targets":[{"Path":"s3://orderflow-datalake-'$ACCOUNT_ID'/"}]}'
aws glue start-crawler --name orderflow-crawler
```

### Athena (serverless SQL over the lake)
```bash
aws athena create-work-group --name orderflow \
  --configuration '{"ResultConfiguration":{"OutputLocation":"s3://orderflow-datalake-'$ACCOUNT_ID'/athena-results/"}}'
# then: aws athena start-query-execution --query-string "SELECT ..." --work-group orderflow
```

### Bedrock (GenAI — enable model access, then call)
```bash
# In the console: Bedrock → Model access → request access to Claude models (one-time).
# Then invoke from anywhere with an IAM role that has bedrock:InvokeModel:
aws bedrock-runtime invoke-model --model-id anthropic.claude-3-5-sonnet-20241022-v2:0 \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":256,"messages":[{"role":"user","content":"hi"}]}' \
  --cli-binary-format raw-in-base64-out out.json
```

### The heavy analytics stack (brief — provision only when needed)
```bash
# Lake Formation: register the S3 location + grant table permissions (console-driven)
# Redshift Serverless: aws redshift-serverless create-namespace/workgroup ...
# EMR Serverless:      aws emr-serverless create-application --type SPARK ...
# QuickSight:          sign up in console, point at Athena/Redshift
# SageMaker:           aws sagemaker create-domain / create-endpoint ...
```

---

## Phase 8 — Observability & security posture

### CloudWatch (log group, alarm, dashboard)
```bash
aws logs create-log-group --log-group-name /orderflow/orders
aws logs put-retention-policy --log-group-name /orderflow/orders --retention-in-days 30

# Alarm: SQS backlog climbing
aws cloudwatch put-metric-alarm --alarm-name orders-queue-backlog \
  --namespace AWS/SQS --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=order-processing \
  --statistic Average --period 300 --threshold 100 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 --alarm-actions $TOPIC_ARN
```

### CloudTrail (account-wide audit → locked S3 bucket)
```bash
aws cloudtrail create-trail --name orderflow-audit \
  --s3-bucket-name orderflow-logs-$ACCOUNT_ID --is-multi-region-trail
aws cloudtrail start-logging --name orderflow-audit
```

### X-Ray, Config, Systems Manager, and the security services (enable)
```bash
# X-Ray: add the ADOT/X-Ray sidecar or SDK to services; no resource to pre-create
aws configservice put-configuration-recorder --configuration-recorder name=default,roleARN=arn:aws:iam::$ACCOUNT_ID:role/aws-config
aws guardduty create-detector --enable
aws securityhub enable-security-hub
aws inspector2 enable --resource-types ECR EC2
aws macie2 enable-macie
# Systems Manager Session Manager needs the SSM agent (built into ECS/AL2) + an IAM role — no open SSH
```

---

## Phase 9 — Edge & CI/CD

### CloudFront + WAF + Shield (front the app)
```bash
# WAF WebACL (rate limit + managed rules)
aws wafv2 create-web-acl --name orderflow-waf --scope CLOUDFRONT --region us-east-1 \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=orderflow \
  --rules '[{"Name":"rate","Priority":1,"Statement":{"RateBasedStatement":{"Limit":2000,"AggregateKeyType":"IP"}},"Action":{"Block":{}},"VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"rate"}}]'
# CloudFront distribution → origin = the SPA S3 bucket / the ALB (console or a distribution-config.json)
# Shield Standard is automatic; Shield Advanced is a paid subscription for extra DDoS protection
```

### API Gateway (HTTP API → ALB/services)
```bash
API_ID=$(aws apigatewayv2 create-api --name orderflow-api --protocol-type HTTP \
  --query ApiId --output text)
# add a JWT authorizer pointing at the Cognito issuer, routes, and integrations to the ALB/VPC link
```

### CodePipeline / CodeBuild / CodeDeploy (or GitHub Actions)
`buildspec.yml` for CodeBuild:
```yaml
version: 0.2
phases:
  build:
    commands:
      - mvn -B verify                       # runs Testcontainers/LocalStack tests
      - docker build -t $REPO:$TAG .
      - aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY
      - docker push $REPO:$TAG
artifacts:
  files: [imagedefinitions.json]
```
Pipeline: Source (GitHub/CodeCommit) → Build (CodeBuild) → Deploy (CodeDeploy blue/green to ECS). CodeDeploy shifts traffic and auto-rolls-back on a CloudWatch alarm.

---

## Teardown (run when done — stops the bill)

Delete in reverse dependency order. The expensive things first:
```bash
aws ecs update-service --cluster orderflow --service orders --desired-count 0
aws ecs delete-service --cluster orderflow --service orders --force
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws rds delete-db-instance --db-instance-identifier orderflow-aurora-1 --skip-final-snapshot
aws rds delete-db-cluster --db-cluster-identifier orderflow-aurora --skip-final-snapshot
aws elasticache delete-replication-group --replication-group-id orderflow-cache
aws opensearch delete-domain --domain-name orderflow-search
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_ID        # stops the ~$32/mo bleed
aws ec2 release-address --allocation-id $EIP
# then DynamoDB tables, S3 buckets (empty first), queues, topics, VPC last
```
**The rule:** if you're not actively using it, tear it down — especially NAT Gateway, Aurora, ElastiCache, OpenSearch, Redshift, and MSK. Use `cdk destroy` / `terraform destroy` once you've moved to IaC — one command instead of this list.

---

## Next: convert to IaC

Everything above should ultimately be one `cdk deploy`. A CDK stack replaces this entire playbook with reviewable, versioned TypeScript/Java — see the companion Spring Boot doc for a starter `OrderFlowStack`. Imperative CLI is for *understanding*; IaC is for *operating*.
