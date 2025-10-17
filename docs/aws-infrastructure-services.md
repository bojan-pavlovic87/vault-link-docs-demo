# VaultLink AWS Infrastructure Services Specification
## Terraform/CloudFormation Implementation Guide

**Document Version:** 1.0  
**Date:** October 17, 2025  
**Prepared by:** Amdaris (Mary - Business Analyst)  
**Purpose:** Infrastructure as Code (IaC) implementation reference for VaultLink AWS rebuild

---

## Overview

This document provides a comprehensive list of all AWS services required for the VaultLink platform, organized by functional layer. Each service includes configuration requirements, dependencies, and Terraform resource considerations.

---

## Infrastructure Layers

### 1. Frontend & CDN Layer

#### **Amazon S3 (Static Assets & Documents)**
- **Purpose:** Host static assets (CSS, JS, images) and documents
- **Important:** Angular uses **SSR (Server-Side Rendering)**, the frontend application will be hosted on **Lambda or Elastic Beanstalk**
- **Note:** S3 is only used for static files in SSR scenarios.
- **Buckets Required:**
  - vaultlink-static-assets-{env} - Static files (CSS, JS, images) - served via CloudFront
  - vaultlink-customer-configs-{env} - Per-customer branding (logos, colors)
  - vaultlink-documents-{env} - Invoice PDFs, PODs, receipts
  - vaultlink-backups-{env} - Automated backup destination
  - vaultlink-logs-{env} - CloudFront and application logs

- **Configuration:**
  - Versioning: Enabled (for static-assets and customer-configs)
  - Encryption: SSE-S3 or AWS KMS
  - Public Access: Blocked (access via CloudFront only)
  - Lifecycle Policies: 
    - Documents: Transition to Glacier after 90 days, delete after 7 years
    - Logs: Delete after 90 days
    - Backups: Transition to Glacier after 30 days
  - CORS: Enabled for CloudFront origin
  - Object Lock: Enabled for compliance-critical buckets (optional)

- **Terraform Resources:**
  - aws_s3_bucket
  - aws_s3_bucket_versioning
  - aws_s3_bucket_server_side_encryption_configuration
  - aws_s3_bucket_public_access_block
  - aws_s3_bucket_lifecycle_configuration
  - aws_s3_bucket_cors_configuration
  - aws_s3_bucket_policy (CloudFront OAI access)

---

#### **Amazon CloudFront (CDN)**
- **Purpose:** Global content delivery, SSL termination, caching, WAF integration
- **Distributions Required:**
  - Primary distribution for Angular SPA (origin: S3 static-assets)
  - Optional: Separate distribution for API Gateway (if caching API responses)

- **Configuration:**
  - Origin: S3 bucket with Origin Access Identity (OAI)
  - SSL Certificate: ACM certificate for custom domain (*.vaultlink.synertec.com)
  - Price Class: Use PriceClass_100 (US, EU, Canada) or PriceClass_All based on user geography
  - Default Root Object: index.html
  - Error Pages: Custom 404, 403 redirect to index.html (SPA routing)
  - Caching Behavior:
    - Static Assets: Cache based on query strings, max-age 31536000 (1 year)
    - index.html: No cache or short TTL (60 seconds) for version updates
  - Compression: Gzip/Brotli enabled
  - HTTP  HTTPS Redirect: Enabled
  - Logging: Enabled to S3 logs bucket
  - WAF: Associate with AWS WAF WebACL

- **Terraform Resources:**
  - aws_cloudfront_distribution
  - aws_cloudfront_origin_access_identity
  - aws_acm_certificate (us-east-1 region required for CloudFront)
  - aws_acm_certificate_validation
  - aws_route53_record (DNS alias to CloudFront)

---

#### **AWS WAF (Web Application Firewall)**
- **Purpose:** DDoS protection, rate limiting, IP filtering, SQL injection/XSS blocking
- **WebACL Configuration:**
  - Scope: CLOUDFRONT (must be created in us-east-1 region)
  - Rules:
    - AWS Managed Rule: AWSManagedRulesCommonRuleSet (OWASP Top 10)
    - AWS Managed Rule: AWSManagedRulesKnownBadInputsRuleSet
    - Rate Limiting: 2000 requests per 5 minutes per IP
    - Geo-blocking: Block countries outside allowed list (optional)
    - Custom Rule: Allow CloudFront  API Gateway traffic
  - Logging: Enabled to S3 or CloudWatch Logs
  - Metrics: CloudWatch WAF metrics

- **Terraform Resources:**
  - aws_wafv2_web_acl
  - aws_wafv2_ip_set (for IP allowlists/blocklists)
  - aws_wafv2_regex_pattern_set (for custom pattern matching)
  - aws_cloudwatch_log_group (WAF logs)

---

#### **AWS Lambda@Edge (Optional)**
- **Purpose:** CloudFront ingress rules, custom headers, A/B testing, security headers
- **Functions:**
  - Viewer Request: Add security headers (CSP, HSTS, X-Frame-Options)
  - Origin Request: Modify requests to S3 (e.g., default directory index)
  - Origin Response: Custom error handling, cache control headers

- **Configuration:**
  - Runtime: Python 3.x or Node.js (edge-compatible)
  - Memory: 128MB (minimum)
  - Timeout: 5 seconds (viewer functions), 30 seconds (origin functions)
  - Region: us-east-1 (Lambda@Edge requirement)
  - IAM Role: Lambda execution role with CloudFront trigger permissions

- **Terraform Resources:**
  - aws_lambda_function 
  - aws_iam_role (Lambda@Edge execution role)
  - aws_cloudfront_distribution (associate Lambda functions)

---

### 2. API & Compute Layer

#### **AWS API Gateway (REST API)**
- **Purpose:** Central routing for all backend Lambda functions
- **Configuration:**
  - API Type: REST API (not HTTP API, for broader feature support)
  - Endpoint Type: Regional (CloudFront handles edge caching if needed)
  - Authorization: Aaws_IAM, Cognito User Pool Authorizer, Lambda Authorizer
  - Throttling: 
    - Burst Limit: 5000 requests
    - Rate Limit: 10,000 requests per second (per account)
  - CORS: Enabled for Angular SPA origin
  - Request Validation: Enabled (validate request body and parameters)
  - API Keys: Optional for third-party integrations
  - Stages: dev, staging, prod
  - Logging: CloudWatch Logs (INFO or ERROR level)
  - Metrics: CloudWatch API Gateway metrics
  - X-Ray Tracing: Enabled for distributed tracing

- **Resources/Routes:**
  - GET /statements/{accountId}  StatementService Lambda
  - POST /payments  PaymentService Lambda
  - GET /documents/{accountId}  DocumentService Lambda
  - POST /queries  QueryService Lambda
  - GET /config/{customerId}  ConfigService Lambda (customer branding)
  - POST /webhooks/stripe  StripeWebhookHandler Lambda

- **Terraform Resources:**
  - aws_api_gateway_rest_api
  - aws_api_gateway_resource
  - aws_api_gateway_method
  - aws_api_gateway_integration
  - aws_api_gateway_deployment
  - aws_api_gateway_stage
  - aws_api_gateway_method_settings
  - aws_api_gateway_authorizer
  - aws_cloudwatch_log_group (API Gateway logs)

---

#### **AWS Lambda (Compute)**
- **Purpose:** Serverless compute for API endpoints, background jobs, event processing
- **Functions Required:**

**API Functions:**
1. **StatementService**
   - Handler: VaultLink.API.StatementHandler::HandleRequest
   - Runtime: dotnet8
   - Memory: 512MB
   - Timeout: 30 seconds
   - Environment Variables: DYNAMODB_TABLE, DAX_ENDPOINT
   - Triggers: API Gateway

2. **PaymentService**
   - Handler: VaultLink.API.PaymentHandler::HandleRequest
   - Runtime: dotnet8
   - Memory: 1024MB
   - Timeout: 60 seconds
   - Environment Variables: STRIPE_SECRET_KEY, DYNAMODB_TABLE
   - Triggers: API Gateway
   - Reserved Concurrency: 50 (prevent thundering herd to Stripe)

3. **DocumentService**
   - Handler: VaultLink.API.DocumentHandler::HandleRequest
   - Runtime: dotnet8
   - Memory: 512MB
   - Timeout: 30 seconds
   - Environment Variables: S3_BUCKET, DYNAMODB_TABLE
   - Triggers: API Gateway

4. **QueryService**
   - Handler: VaultLink.API.QueryHandler::HandleRequest
   - Runtime: dotnet8
   - Memory: 256MB
   - Timeout: 15 seconds
   - Environment Variables: DYNAMODB_TABLE, SQS_QUEUE_URL
   - Triggers: API Gateway

5. **ConfigService**
   - Handler: VaultLink.API.ConfigHandler::HandleRequest
   - Runtime: dotnet8
   - Memory: 256MB
   - Timeout: 10 seconds
   - Environment Variables: S3_CONFIG_BUCKET
   - Triggers: API Gateway

6. **StripeWebhookHandler**
   - Handler: VaultLink.API.StripeWebhookHandler::HandleRequest
   - Runtime: dotnet8
   - Memory: 512MB
   - Timeout: 30 seconds
   - Environment Variables: STRIPE_WEBHOOK_SECRET, DYNAMODB_TABLE
   - Triggers: API Gateway

**Background/Event Functions:**
7. **PrismDeliveryService**
   - Handler: VaultLink.DeliveryService.PrismHandler::HandleMessage
   - Runtime: dotnet8
   - Memory: 512MB
   - Timeout: 300 seconds (5 minutes)
   - Environment Variables: PRISM_ENDPOINT, PRISM_API_KEY
   - Triggers: SQS (vaultlink-prism-delivery-queue)
   - Batch Size: 10
   - Max Concurrency: 5

8. **CloudWatchAlarmHandler**
   - Handler: VaultLink.Monitoring.AlarmHandler::HandleAlarm
   - Runtime: dotnet8
   - Memory: 256MB
   - Timeout: 60 seconds
   - Environment Variables: SNS_TOPIC_ARN, SLACK_WEBHOOK_URL
   - Triggers: EventBridge (CloudWatch Alarm events)

**Provisioned Concurrency (Optional):**
- PaymentService: 5 instances (reduce cold starts for critical path)
- StatementService: 3 instances

- **Common Configuration:**
  - VPC: Optional (only if accessing resources in VPC, e.g., RDS)
  - Dead Letter Queue: SQS queue for failed invocations
  - Environment Encryption: AWS KMS
  - Layers: Shared libraries, AWS SDK, observability tools
  - X-Ray Tracing: Enabled
  - Reserved Concurrent Executions: Configure per function based on load

- **Terraform Resources:**
  - aws_lambda_function
  - aws_lambda_permission (API Gateway invoke permission)
  - aws_lambda_event_source_mapping (SQS trigger)
  - aws_iam_role (Lambda execution role)
  - aws_iam_role_policy_attachment
  - aws_lambda_provisioned_concurrency_config
  - aws_lambda_layer_version (shared layers)
  - aws_cloudwatch_log_group (Lambda logs)

---

#### **AWS Elastic Beanstalk (Fallback)**
- **Purpose:** Host long-running .NET processes if Lambda 15-min limit insufficient
- **Configuration:**
  - Platform: .NET on Linux (or .NET Core on Linux)
  - Environment Type: Load-balanced
  - Instance Type: t3.medium (start small, scale up)
  - Capacity: 
    - Min: 2 instances
    - Max: 10 instances
    - Auto Scaling: CPU > 70% scale up
  - Load Balancer: Application Load Balancer (ALB)
  - Health Checks: HTTP /health endpoint
  - Deployment: Rolling with batch (10% at a time)
  - Monitoring: Enhanced CloudWatch monitoring
  - VPC: Private subnets with NAT Gateway egress

- **Terraform Resources:**
  - aws_elastic_beanstalk_application
  - aws_elastic_beanstalk_environment
  - aws_elastic_beanstalk_configuration_template
  - aws_iam_instance_profile
  - aws_security_group

---

### 3. Data Layer

#### **Amazon DynamoDB (Primary Database)**
- **Purpose:** NoSQL database for application data
- **Tables Required:**

1. **VaultLinkMain (Single-Table Design - Recommended)**
   - Partition Key: PK (String) - e.g., CUSTOMER#123, STATEMENT#456
   - Sort Key: SK (String) - e.g., METADATA, DOCUMENT#789
   - Attributes: JSON-serialized data per entity type
   - Billing Mode: On-Demand (start), evaluate Provisioned later
   - Global Secondary Indexes:
     - GSI1: GSI1PK (PK), GSI1SK (SK) - Inverted index for queries
     - GSI2: GSI2PK (PK), GSI2SK (SK) - Date-based queries
   - Point-in-Time Recovery: Enabled
   - Encryption: AWS managed key (or customer-managed KMS key)
   - TTL: Enabled on ExpiresAt attribute (for temporary data)
   - Streams: Enabled (for event-driven architectures, auditing)
   - CloudWatch Contributor Insights: Enabled

OR

**Multi-Table Design (Alternative):**
2. **Customers**
   - PK: CustomerId (String)
   - Attributes: CustomerName, BrandingConfig, PrismEndpoint, etc.

3. **Statements**
   - PK: StatementId (String)
   - SK: CustomerId (String)
   - GSI: CustomerId-InvoiceDate-index

4. **Documents**
   - PK: DocumentId (String)
   - SK: StatementId (String)
   - GSI: StatementId-CreatedDate-index

5. **Queries**
   - PK: QueryId (String)
   - SK: CustomerId-DocumentId (composite)
   - GSI: CustomerId-Status-index

6. **AuditLog**
   - PK: EventId (String)
   - SK: Timestamp (Number)
   - TTL: ExpiresAt (90 days)

- **Common Configuration:**
  - Backup: AWS Backup plan (daily, retain 30 days)
  - CloudWatch Alarms: Read/Write throttles, consumed capacity
  - Tags: Environment, Project, CostCenter

- **Terraform Resources:**
  - aws_dynamodb_table
  - aws_dynamodb_table_item (seed data)
  - aws_appautoscaling_target (if using provisioned capacity)
  - aws_appautoscaling_policy

---

#### **Amazon DAX (DynamoDB Accelerator)**
- **Purpose:** In-memory cache for DynamoDB read-heavy operations
- **Configuration:**
  - Cluster Name: vaultlink-dax-{env}
  - Node Type: dax.t3.small (start), scale to dax.r5.large if needed
  - Number of Nodes: 3 (multi-AZ)
  - Subnet Group: Private subnets across multiple AZs
  - Security Group: Allow inbound from Lambda security group on port 8111
  - Parameter Group: Default (adjust TTL if needed)
  - Maintenance Window: Sunday 02:00-03:00 UTC
  - Notification Topic: SNS topic for DAX events

- **Terraform Resources:**
  - aws_dax_cluster
  - aws_dax_subnet_group
  - aws_dax_parameter_group
  - aws_security_group

---

#### **Amazon ElastiCache (Redis - Optional)**
- **Purpose:** Session storage if Cognito insufficient, rate limiting, distributed locks
- **Configuration:**
  - Engine: Redis 7.x
  - Node Type: cache.t3.micro (start)
  - Number of Nodes: 2 (primary + replica for HA)
  - Subnet Group: Private subnets
  - Security Group: Allow Lambda/Beanstalk on port 6379
  - Automatic Failover: Enabled (Multi-AZ)
  - Encryption: In-transit and at-rest
  - Backup: Daily, retain 7 days
  - Maintenance Window: Sunday 03:00-04:00 UTC

- **Terraform Resources:**
  - aws_elasticache_replication_group
  - aws_elasticache_subnet_group
  - aws_elasticache_parameter_group
  - aws_security_group

---

### 4. Integration & Messaging Layer

#### **Amazon SQS (Simple Queue Service)**
- **Purpose:** Message queuing for Prism integration, decoupling services
- **Queues Required:**

1. **vaultlink-prism-delivery-queue**
   - Type: Standard (FIFO if ordering required)
   - Visibility Timeout: 300 seconds (match Lambda timeout)
   - Message Retention: 4 days (default), extend to 14 days if needed
   - Max Message Size: 256 KB
   - Receive Wait Time: 20 seconds (long polling)
   - Dead Letter Queue: vaultlink-prism-dlq
   - Max Receive Count: 3 (after 3 failures  DLQ)
   - Encryption: AWS KMS
   - CloudWatch Alarms: Queue depth > 100, age of oldest message > 1 hour

2. **vaultlink-prism-dlq (Dead Letter Queue)**
   - Type: Standard
   - Retention: 14 days (max)
   - Alarms: Any message in DLQ  alert

3. **vaultlink-internal-events (Optional)**
   - Purpose: Internal event bus for async processing
   - Type: Standard

- **Terraform Resources:**
  - aws_sqs_queue
  - aws_sqs_queue_policy
  - aws_lambda_event_source_mapping (SQS  Lambda)
  - aws_cloudwatch_metric_alarm

---

#### **Amazon EventBridge (Event Bus)**
- **Purpose:** Event routing from CloudWatch alarms to notification targets
- **Configuration:**
  - Default Event Bus: Use default (or create custom bus)
  - Rules:
    1. **CloudWatch Alarm  SNS**
       - Event Pattern: CloudWatch Alarm state change (ALARM)
       - Target: SNS topic (vaultlink-alerts)
    2. **Lambda Errors  Slack**
       - Event Pattern: Lambda function errors
       - Target: Lambda function (posts to Slack webhook)
    3. **DynamoDB Streams  EventBridge (Optional)**
       - Event Pattern: DynamoDB item changes
       - Target: Lambda function for audit logging

- **Terraform Resources:**
  - aws_cloudwatch_event_rule
  - aws_cloudwatch_event_target
  - aws_cloudwatch_event_bus (if custom bus)

---

#### **Amazon SNS (Simple Notification Service)**
- **Purpose:** Push notifications to SMS, Email, Lambda
- **Topics Required:**

1. **vaultlink-alerts**
   - Subscriptions:
     - SMS: +44... (on-call engineer phone)
     - Email: ops@amdaris.com
     - Lambda: CloudWatchAlarmHandler (routes to Slack)
   - Delivery Status Logging: CloudWatch Logs
   - KMS Encryption: Enabled

2. **vaultlink-user-notifications (Optional - Phase 2)**
   - Purpose: Send payment confirmations, query updates to end-users
   - Subscriptions: SQS queue for batch processing

- **Terraform Resources:**
  - aws_sns_topic
  - aws_sns_topic_subscription
  - aws_sns_topic_policy

---

#### **Amazon SES (Simple Email Service)**
- **Purpose:** Send transactional emails (receipts, alerts)
- **Configuration:**
  - Verified Domains: synertec.com, vaultlink.synertec.com
  - Verified Emails: noreply@vaultlink.synertec.com
  - Configuration Set: Track bounces, complaints, deliveries
  - Sending Limits: Start in sandbox (200 emails/day), request production access
  - Reputation Dashboard: Monitor bounce/complaint rates
  - Dedicated IP: Optional (if high volume, >100k emails/month)

- **Terraform Resources:**
  - aws_ses_domain_identity
  - aws_ses_domain_identity_verification
  - aws_ses_email_identity
  - aws_ses_configuration_set
  - aws_route53_record (DKIM, SPF, DMARC records)

---

### 5. Authentication & Authorization

#### **AWS Cognito (User Pools)**
- **Purpose:** Authentication for admin/staff users
- **Configuration:**
  - User Pool: vaultlink-admin-users-{env}
  - Sign-in Options: Email, Username
  - Password Policy:
    - Minimum Length: 12 characters
    - Require: Uppercase, lowercase, numbers, symbols
    - Temporary Password Expiry: 7 days
  - MFA: Required for all users (TOTP or SMS)
  - Account Recovery: Email only (no SMS for security)
  - Email Configuration: SES (for custom from address)
  - Triggers: Pre-signup Lambda (validate email domain)
  - App Clients:
    - Angular SPA: Authorization Code flow with PKCE
    - API Gateway: Verify JWT tokens
  - User Pool Domain: vaultlink-auth-{env}.auth.{region}.amazoncognito.com
  - Custom Domain: auth.vaultlink.synertec.com (requires ACM certificate)

- **Identity Pools (Optional):**
  - Purpose: Temporary AWS credentials for direct S3 access (if needed)
  - Authenticated Role: Read-only S3 access to user's documents

- **Terraform Resources:**
  - aws_cognito_user_pool
  - aws_cognito_user_pool_client
  - aws_cognito_user_pool_domain
  - aws_cognito_identity_pool
  - aws_cognito_identity_pool_roles_attachment
  - aws_iam_role (Cognito authenticated/unauthenticated roles)

---

### 6. Monitoring & Observability

#### **Amazon CloudWatch (Logs, Metrics, Alarms)**
- **Log Groups:**
  - /aws/lambda/{function-name} (auto-created by Lambda)
  - /aws/apigateway/vaultlink-api-{env}
  - /aws/waf/vaultlink-webacl
  - /aws/elasticbeanstalk/vaultlink-app-{env}
  - Retention: 90 days (compliance requirement)
  - Encryption: KMS

- **Metrics & Alarms:**
  1. **API Gateway**
     - 5XXError > 10 in 5 minutes  ALARM
     - Latency P95 > 1000ms  ALARM
  2. **Lambda**
     - Errors > 5 in 5 minutes  ALARM
     - Throttles > 0  ALARM
     - Duration P95 > (Timeout - 10%)  WARNING
  3. **DynamoDB**
     - Read/WriteThrottleEvents > 0  ALARM
     - ConsumedReadCapacity > 80% provisioned  WARNING
  4. **SQS**
     - ApproximateAgeOfOldestMessage > 3600 seconds  ALARM
     - ApproximateNumberOfMessagesVisible > 1000  WARNING
  5. **Custom Application Metrics**
     - PaymentSuccessRate < 99%  ALARM
     - PrismDeliveryFailureRate > 1%  ALARM

- **Dashboards:**
  - **Operational Dashboard**: API health, Lambda performance, database metrics
  - **Business Dashboard**: Payment volume, success rates, user activity
  - **Security Dashboard**: WAF blocks, authentication failures, suspicious activity

- **Terraform Resources:**
  - aws_cloudwatch_log_group
  - aws_cloudwatch_metric_alarm
  - aws_cloudwatch_dashboard
  - aws_cloudwatch_log_metric_filter (custom metrics from logs)

---

#### **AWS X-Ray (Distributed Tracing)**
- **Purpose:** End-to-end request tracing across Lambda, API Gateway, DynamoDB
- **Configuration:**
  - API Gateway: X-Ray tracing enabled
  - Lambda: Active tracing enabled on all functions
  - SDK Instrumentation: AWS SDK calls automatically traced
  - Custom Segments: Add application-specific trace data
  - Sampling Rules: 5% of requests (adjust based on volume)
  - Retention: 30 days

- **Terraform Resources:**
  - aws_xray_sampling_rule
  - IAM permissions: xray:PutTraceSegments, xray:PutTelemetryRecords

---

### 7. Backup & Disaster Recovery

#### **AWS Backup**
- **Purpose:** Centralized backup management for DynamoDB, S3
- **Backup Plans:**
  1. **Daily Backup Plan**
     - Resources: DynamoDB tables, S3 buckets (customer-configs, documents)
     - Schedule: Daily at 01:00 UTC
     - Retention: 30 days
     - Vault: Default backup vault with KMS encryption
     - Lifecycle: Move to cold storage after 30 days (if supported)

  2. **Weekly Backup Plan**
     - Resources: Same as daily
     - Schedule: Sunday at 00:00 UTC
     - Retention: 90 days

- **Backup Vault:**
  - Name: vaultlink-backup-vault-{env}
  - Encryption: AWS KMS customer-managed key
  - Vault Lock: Optional (enforce retention, prevent deletion)

- **Terraform Resources:**
  - aws_backup_plan
  - aws_backup_vault
  - aws_backup_selection
  - aws_iam_role (AWS Backup service role)

---

### 8. Security & Compliance

#### **AWS KMS (Key Management Service)**
- **Purpose:** Encryption keys for S3, DynamoDB, SQS, SNS, logs
- **Keys Required:**
  1. **vaultlink-data-key**
     - Purpose: Encrypt DynamoDB tables, S3 buckets
     - Key Policy: Allow Lambda/Beanstalk decrypt, allow admins manage
     - Rotation: Automatic annual rotation enabled

  2. **vaultlink-logs-key**
     - Purpose: Encrypt CloudWatch Logs
     - Key Policy: Allow CloudWatch Logs service

  3. **vaultlink-backup-key**
     - Purpose: Encrypt AWS Backup vaults
     - Key Policy: Allow AWS Backup service

- **Terraform Resources:**
  - aws_kms_key
  - aws_kms_alias
  - aws_kms_key_policy

---

#### **AWS Secrets Manager**
- **Purpose:** Store Stripe API keys, Prism credentials, database passwords
- **Secrets:**
  1. **vaultlink/{env}/stripe-secret-key**
     - Value: Stripe secret key
     - Rotation: Manual (Stripe doesn't support auto-rotation)

  2. **vaultlink/{env}/stripe-webhook-secret**
     - Value: Stripe webhook signing secret

  3. **vaultlink/{env}/prism-api-key**
     - Value: Prism integration API key

  4. **vaultlink/{env}/slack-webhook-url**
     - Value: Slack incoming webhook URL for alerts

- **Configuration:**
  - Encryption: KMS customer-managed key
  - Rotation: Enabled (where supported)
  - Replica: Optional multi-region replication

- **Terraform Resources:**
  - aws_secretsmanager_secret
  - aws_secretsmanager_secret_version
  - aws_secretsmanager_secret_rotation

---

#### **AWS Certificate Manager (ACM)**
- **Purpose:** SSL/TLS certificates for CloudFront, API Gateway, ALB
- **Certificates:**
  1. ***.vaultlink.synertec.com** (us-east-1 for CloudFront)
     - Validation: DNS (Route 53)
     - Renewal: Automatic

  2. **api.vaultlink.synertec.com** (regional for API Gateway)
     - Validation: DNS

  3. **auth.vaultlink.synertec.com** (for Cognito custom domain)
     - Validation: DNS

- **Terraform Resources:**
  - aws_acm_certificate
  - aws_acm_certificate_validation
  - aws_route53_record (DNS validation)

---

#### **AWS IAM (Identity & Access Management)**
- **Roles Required:**
  1. **Lambda Execution Roles** (per function or shared)
     - Permissions: CloudWatch Logs, DynamoDB, S3, SQS, Secrets Manager, X-Ray

  2. **Elastic Beanstalk Instance Profile**
     - Permissions: S3, CloudWatch, DynamoDB

  3. **AWS Backup Service Role**
     - Managed Policy: AWSBackupServiceRolePolicyForBackup

  4. **Cognito Authenticated Role**
     - Permissions: Limited S3 read access

  5. **EventBridge Role**
     - Permissions: Invoke Lambda, publish to SNS

- **Policies:**
  - Least Privilege: Each role has minimal permissions
  - Resource-Based Policies: S3 buckets, SQS queues, Lambda functions
  - Boundary Policies: Prevent privilege escalation

- **Terraform Resources:**
  - aws_iam_role
  - aws_iam_policy
  - aws_iam_role_policy_attachment
  - aws_iam_instance_profile

---

#### **AWS CloudTrail**
- **Purpose:** Audit log of all AWS API calls
- **Configuration:**
  - Trail Name: vaultlink-audit-trail-{env}
  - S3 Bucket: vaultlink-cloudtrail-{env}
  - Log File Validation: Enabled
  - Multi-Region: Enabled
  - Management Events: Read/Write
  - Data Events: S3 object-level API (optional, for sensitive buckets)
  - CloudWatch Logs Integration: Stream to CloudWatch for real-time analysis
  - Encryption: SSE-KMS
  - Retention: 90 days in CloudWatch, indefinite in S3 (with Glacier transition)

- **Terraform Resources:**
  - aws_cloudtrail
  - aws_s3_bucket (CloudTrail logs)
  - aws_cloudwatch_log_group

---

### 9. Networking (Optional - if VPC required)

#### **Amazon VPC**
- **Purpose:** Isolated network for Elastic Beanstalk, RDS (if used), ElastiCache
- **Configuration:**
  - CIDR Block: 10.0.0.0/16
  - Subnets:
    - Public Subnets: 10.0.1.0/24, 10.0.2.0/24 (ALB, NAT Gateway)
    - Private Subnets: 10.0.11.0/24, 10.0.12.0/24 (Beanstalk, ElastiCache, DAX)
  - Availability Zones: 2 AZs minimum (eu-west-2a, eu-west-2b)
  - Internet Gateway: Attached to VPC
  - NAT Gateway: 1 per public subnet (HA)
  - Route Tables:
    - Public: 0.0.0.0/0  IGW
    - Private: 0.0.0.0/0  NAT Gateway
  - VPC Endpoints: S3, DynamoDB (Gateway endpoints, no data transfer charges)
  - Flow Logs: Enabled to CloudWatch Logs (traffic analysis, security)

- **Terraform Resources:**
  - aws_vpc
  - aws_subnet
  - aws_internet_gateway
  - aws_nat_gateway
  - aws_eip (for NAT)
  - aws_route_table
  - aws_route_table_association
  - aws_vpc_endpoint
  - aws_flow_log

---

#### **Security Groups**
- **Groups Required:**
  1. **alb-sg** (Application Load Balancer)
     - Inbound: 443 from 0.0.0.0/0 (HTTPS)
     - Outbound: All to beanstalk-sg

  2. **beanstalk-sg** (Elastic Beanstalk instances)
     - Inbound: 8080 from alb-sg
     - Outbound: All to 0.0.0.0/0 (API calls, DynamoDB, S3)

  3. **lambda-sg** (if Lambda in VPC)
     - Inbound: None
     - Outbound: All to 0.0.0.0/0

  4. **elasticache-sg**
     - Inbound: 6379 from beanstalk-sg, lambda-sg
     - Outbound: None

  5. **dax-sg**
     - Inbound: 8111 from beanstalk-sg, lambda-sg
     - Outbound: None

- **Terraform Resources:**
  - aws_security_group
  - aws_security_group_rule

---

### 10. DNS & Domain Management

#### **Amazon Route 53**
- **Purpose:** DNS management for custom domains
- **Hosted Zones:**
  - vaultlink.synertec.com (public hosted zone)

- **Records:**
  - A (Alias): vaultlink.synertec.com  CloudFront distribution
  - A (Alias): api.vaultlink.synertec.com  API Gateway custom domain
  - A (Alias): auth.vaultlink.synertec.com  Cognito custom domain
  - CNAME: _acme-challenge.vaultlink.synertec.com  ACM validation
  - TXT: vaultlink.synertec.com  SPF record (SES)
  - TXT: _dmarc.vaultlink.synertec.com  DMARC policy (SES)
  - CNAME: {selector}._domainkey.vaultlink.synertec.com  DKIM (SES)

- **Health Checks:**
  - HTTPS check on vaultlink.synertec.com
  - Alarm: Route 53 health check failure  SNS notification

- **Terraform Resources:**
  - aws_route53_zone
  - aws_route53_record
  - aws_route53_health_check

---

## Infrastructure as Code Structure

### Recommended Terraform Module Organization:

\\\
terraform/
 modules/
    frontend/           # S3, CloudFront, WAF
    api/                # API Gateway, Lambda functions
    compute/            # Elastic Beanstalk (if needed)
    data/               # DynamoDB, DAX, ElastiCache
    messaging/          # SQS, SNS, EventBridge, SES
    auth/               # Cognito
    monitoring/         # CloudWatch, X-Ray
    security/           # KMS, Secrets Manager, IAM
    backup/             # AWS Backup
    networking/         # VPC, subnets, security groups (optional)
 environments/
    dev/
       main.tf
       variables.tf
       outputs.tf
       terraform.tfvars
    staging/
    prod/
 backend.tf              # S3 backend for Terraform state
 provider.tf             # AWS provider configuration
\\\

---

## Multi-Environment Configuration

### Environment-Specific Variables:
\\\hcl
# terraform.tfvars (per environment)
environment         = \"prod\"
aaws_region          = \"eu-west-2\"
project_name        = \"vaultlink\"

# Frontend
cloudfront_price_class = \"PriceClass_All\"  # dev: PriceClass_100

# Compute
lambda_memory_statement   = 512
lambda_memory_payment     = 1024
lambda_provisioned_concurrency = 5  # dev: 0

# Data
dynamodb_billing_mode = \"PAY_PER_REQUEST\"  # prod: \"PROVISIONED\"
dax_node_type        = \"dax.r5.large\"      # dev: \"dax.t3.small\"
dax_node_count       = 3                     # dev: 1

# Monitoring
log_retention_days   = 90  # dev: 7
alarm_notification_emails = [\"ops@amdaris.com\"]

# Backup
backup_retention_days = 30  # dev: 7

# Tags
tags = {
  Environment = \"prod\"
  Project     = \"VaultLink\"
  ManagedBy   = \"Terraform\"
  CostCenter  = \"Engineering\"
  Owner       = \"Amdaris\"
}
\\\

---

## Deployment Sequence

### Recommended Terraform Apply Order:

1. **Foundation** (can be applied in parallel after VPC)
   - VPC & Networking (if using)
   - KMS keys
   - S3 buckets (backend state, logs)
   - IAM roles (service roles)

2. **Security**
   - Secrets Manager secrets
   - ACM certificates
   - WAF WebACL (must be in us-east-1 for CloudFront)
   - Security Groups

3. **Data Layer**
   - DynamoDB tables
   - DAX cluster (depends on DynamoDB, VPC)
   - ElastiCache (optional, depends on VPC)

4. **Authentication**
   - Cognito User Pool
   - Cognito User Pool Client

5. **Compute**
   - Lambda functions (depends on IAM, Secrets Manager)
   - Lambda layers
   - Elastic Beanstalk (optional, depends on VPC, security groups)

6. **API**
   - API Gateway REST API
   - API Gateway resources, methods, integrations
   - API Gateway deployment & stage

7. **Messaging**
   - SQS queues
   - SNS topics
   - EventBridge rules
   - SES domain identities

8. **Frontend**
   - S3 bucket policy (CloudFront OAI)
   - CloudFront distribution (depends on S3, ACM, WAF)

9. **Monitoring**
   - CloudWatch Log Groups
   - CloudWatch Alarms
   - CloudWatch Dashboards
   - X-Ray sampling rules

10. **Backup**
    - AWS Backup vault
    - AWS Backup plans
    - AWS Backup selections

11. **DNS**
    - Route 53 records (depends on CloudFront, API Gateway)
    - Route 53 health checks

---

## Cost Estimation (Monthly - Rough Estimates)

**Development Environment:**
- S3: ~$10
- CloudFront: ~$20
- Lambda: ~$50 (low usage)
- DynamoDB: ~$25 (on-demand)
- SQS/SNS: ~$5
- CloudWatch: ~$10
- **Total: ~$120/month**

**Production Environment:**
- S3: ~$50 (includes backups, documents)
- CloudFront: ~$200 (global distribution)
- WAF: ~$50
- Lambda: ~$500 (moderate usage + provisioned concurrency)
- DynamoDB: ~$200 (on-demand, 10M reads/month)
- DAX: ~$200 (3x dax.t3.small nodes)
- SQS/SNS: ~$20
- Cognito: ~$50 (1000 MAUs)
- CloudWatch: ~$50
- Secrets Manager: ~$5
- KMS: ~$10
- AWS Backup: ~$30
- ACM: Free
- **Total: ~$1,365/month**

*Note: Costs will vary based on actual usage. Monitor with AWS Cost Explorer.*

---

## Security Best Practices Checklist

- [ ] Enable encryption at rest (S3, DynamoDB, EBS, logs)
- [ ] Enable encryption in transit (HTTPS, TLS 1.2+)
- [ ] Use KMS customer-managed keys for sensitive data
- [ ] Enable MFA for Cognito users
- [ ] Implement least-privilege IAM policies
- [ ] Enable CloudTrail in all regions
- [ ] Enable VPC Flow Logs (if using VPC)
- [ ] Enable GuardDuty for threat detection (optional but recommended)
- [ ] Enable AWS Config for compliance monitoring (optional)
- [ ] Rotate secrets regularly (Secrets Manager)
- [ ] Enable S3 versioning and object lock for critical buckets
- [ ] Configure WAF rules to block common attacks
- [ ] Implement rate limiting (API Gateway throttling, WAF rate rules)
- [ ] Enable X-Ray tracing for security analysis
- [ ] Set up CloudWatch Alarms for security events
- [ ] Regular security audits with AWS Security Hub (optional)

---

## Compliance & Audit Requirements

**PCI SAQ A Compliance:**
- [ ] Stripe Elements used (no card data touches VaultLink)
- [ ] HTTPS enforced everywhere
- [ ] Access logs enabled (CloudFront, API Gateway, S3)
- [ ] Regular vulnerability scans (AWS Inspector or third-party)

**ISO27001 Alignment:**
- [ ] Risk assessment documented
- [ ] Security controls mapped to AWS services
- [ ] Audit trail via CloudTrail
- [ ] Backup and disaster recovery tested

**GDPR:**
- [ ] Data encryption at rest and in transit
- [ ] Data retention policies enforced (S3 lifecycle, DynamoDB TTL)
- [ ] Right to be forgotten: Implement data deletion workflows
- [ ] Data residency: Ensure AWS region compliance (EU: eu-west-2)

---

## Next Steps for Implementation

1. **Set up Terraform backend:**
   - Create S3 bucket for Terraform state: vaultlink-terraform-state-{account-id}
   - Enable versioning and encryption
   - Create DynamoDB table for state locking: vaultlink-terraform-locks

2. **Initialize Git repository:**
   - Create terraform/ directory structure
   - Add .gitignore for *.tfstate, *.tfvars (secrets)
   - Commit module structure

3. **Create base modules:**
   - Start with modules/frontend (S3, CloudFront)
   - Test deployment to dev environment
   - Iterate through modules in dependency order

4. **Set up CI/CD for Terraform:**
   - GitHub Actions workflow for 	erraform plan on PR
   - Require approval for 	erraform apply to prod
   - Automated 	erraform fmt and 	flint checks

5. **Document runbooks:**
   - Deployment procedures
   - Rollback procedures
   - Disaster recovery procedures
   - Secrets rotation procedures

---

**Document End**
