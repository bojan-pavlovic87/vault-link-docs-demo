# VaultLink Distributed Tracing Strategy

**Document Version:** 1.0  
**Date:** October 17, 2025  
**Client:** Synertec  
**Project:** VaultLink MVP AWS Rebuild  
**Purpose:** Cross-Service Request Tracking and Observability

---

## Executive Summary

**Challenge:** VaultLink has complex payment flows spanning multiple AWS services (API Gateway  Lambda  DynamoDB  SQS  Prism  Stripe webhooks). When issues occurespecially payment-related failureswe need to trace the entire transaction across all services.

**Critical Scenario:** Payment succeeds at Stripe but webhook fails to update VaultLink. How do we correlate logs across all services to diagnose the failure?

**Recommendation:** Use **AWS X-Ray** for distributed tracing with CloudWatch Logs Insights for log correlation.

---

## The Problem

### Complex Payment Flow Example

1. **User initiates payment**  API Gateway request
2. **Lambda processes payment**  Creates Stripe Payment Intent
3. **Lambda writes to DynamoDB**  Payment record (pending status)
4. **Stripe processes card**  Payment succeeds
5. **Stripe webhook fires**  Calls VaultLink webhook endpoint
6. **Lambda handles webhook**  Updates DynamoDB (success status)
7. **Lambda sends SQS message**  Prism notification queue
8. **Lambda consumes SQS**  Delivers to Prism system

**What if step 6 fails?** Payment taken, but VaultLink thinks it's still pending. Customer charged but system not updated.

**Current Challenge:** Logs scattered across CloudWatch Log Groupshow do we connect them all for one transaction?

---

## AWS X-Ray vs Tempo+Grafana

### AWS X-Ray (Recommended for VaultLink)

**What it does:**
- Distributed tracing for AWS services and custom applications
- Automatically traces requests through API Gateway, Lambda, DynamoDB, SQS
- Creates service maps showing request flow and latencies
- Correlates traces with CloudWatch Logs

**Pros:**
- **Native AWS integration** - Zero-config tracing for managed services
- **Low overhead** - Sampling reduces performance impact
- **Cost-effective** - Pay per trace recorded (~$5 per million)
- **Built into Lambda** - Enable with environment variable
- **Service map visualization** - See dependencies automatically

**Cons:**
- **Limited retention** - 30 days from recording (service graph and trace data)
- **AWS-only** - Doesn't trace external services easily (Stripe is external subsegment)
- **Basic UI** - Less polished than Grafana
- **Sampling overhead** - Default 1 req/sec + 5% additional (configurable)

### Tempo + Grafana (Alternative)

**What it does:**
- Open-source distributed tracing backend (Tempo)
- Visualization and querying through Grafana
- Vendor-agnostic (works with any OpenTelemetry instrumentation)

**Pros:**
- **Unlimited retention** - Control your own storage
- **Better visualization** - Grafana's powerful dashboards
- **Cross-platform** - Trace beyond AWS (e.g., external APIs)
- **Open standard** - OpenTelemetry compatibility

**Cons:**
- **Self-managed** - Must deploy and maintain Tempo + Grafana
- **Manual instrumentation** - More code changes required
- **Higher complexity** - Additional infrastructure to manage
- **Cost** - EC2/Fargate for hosting, storage costs

---

## Recommendation: AWS X-Ray for MVP

**Why X-Ray:**
1. **Fast implementation** - Enable in CDK, add SDK calls, done
2. **Native AWS support** - API Gateway, Lambda, DynamoDB auto-traced
3. **Lower operational overhead** - No infrastructure to maintain
4. **Good enough for MVP** - 30-day retention sufficient for debugging
5. **Cost-effective** - Minimal cost at MVP scale

**Post-MVP consideration:** If cross-platform tracing or longer retention needed, evaluate Tempo+Grafana.

---

## How X-Ray Solves VaultLink's Problem

### Payment Flow with X-Ray

**Each request gets a Trace ID** that flows through all services:

```
Trace ID: 1-67891234-abcdef1234567890

 API Gateway: POST /payments
   Lambda: PaymentService.CreatePayment
      DynamoDB: PutItem (payment record)
      Stripe API: Create Payment Intent (external call)
      Response: 200 OK

 [User redirects to Stripe, payment processed]

 API Gateway: POST /webhooks/stripe
   Lambda: WebhookService.HandleStripeWebhook
      DynamoDB: UpdateItem (payment status)  FAILS HERE
      Response: 500 Error

 [SQS message never sent because webhook failed]
```

**With X-Ray, you can:**
1. Search for Trace ID or Payment ID
2. See entire request flow with timing for each step
3. Identify exactly where failure occurred
4. View associated CloudWatch Logs for that Lambda invocation
5. Correlate Stripe webhook timestamp with VaultLink processing

---

## Implementation Approach

### 1. Enable X-Ray Tracing

**Lambda Functions:**
```yaml
# CDK/CloudFormation
PaymentLambda:
  Tracing: Active  # Enables X-Ray
  Environment:
    AWS_XRAY_TRACING_NAME: VaultLink-PaymentService
```

**API Gateway:**
```yaml
ApiGateway:
  TracingEnabled: true
```

**DynamoDB:**
```yaml
# Automatically traced when called from instrumented Lambda
# No additional configuration needed
# Captured as subsegments in X-Ray trace
```

**SQS:**
```yaml
# Automatically traced when called from instrumented Lambda
# Trace context propagated via message attributes
# Consumer Lambda continues the trace
```

### 2. Instrument Custom Code

**.NET Lambda Functions:**
```csharp
// Install NuGet: AWSXRayRecorder.Handlers.AwsSdk

// Startup.cs
AWSSDKHandler.RegisterXRayForAllServices();

// PaymentService.cs
public async Task ProcessPayment(PaymentRequest request)
{
    // Automatic tracing of AWS SDK calls
    await _dynamoDb.PutItemAsync(...);
    
    // Custom subsegments for business logic
    AWSXRayRecorder.Instance.BeginSubsegment("Stripe-CreatePaymentIntent");
    var paymentIntent = await _stripeClient.CreatePaymentIntentAsync(...);
    AWSXRayRecorder.Instance.EndSubsegment();
    
    // Add metadata for searching
    AWSXRayRecorder.Instance.AddMetadata("PaymentId", paymentId);
    AWSXRayRecorder.Instance.AddMetadata("CustomerId", customerId);
}
```

### 3. Correlation IDs

**Generate correlation ID at entry point:**
```csharp
public class PaymentController
{
    public async Task<IActionResult> CreatePayment(PaymentRequest request)
    {
        var correlationId = Guid.NewGuid().ToString();
        
        // Add to X-Ray trace
        AWSXRayRecorder.Instance.AddAnnotation("CorrelationId", correlationId);
        
        // Log with structured logging
        _logger.LogInformation("Payment initiated: {CorrelationId} {CustomerId}", 
            correlationId, request.CustomerId);
        
        // Pass to downstream services
        await _sqsClient.SendMessageAsync(new SendMessageRequest
        {
            MessageAttributes = new Dictionary<string, MessageAttributeValue>
            {
                ["CorrelationId"] = new() { StringValue = correlationId }
            }
        });
    }
}
```

### 4. CloudWatch Logs Integration

X-Ray automatically links traces to CloudWatch Logs. When viewing a trace segment, you can jump directly to the logs for that Lambda invocation.

**Key Integration Features:**
- Trace segments include CloudWatch Log Group and Stream references
- Click through from X-Ray trace to exact log entries
- Correlation IDs searchable in both X-Ray and CloudWatch Logs Insights
- Lambda invocations automatically tagged with X-Ray Trace ID

---

## Querying and Debugging

### X-Ray Console Queries

**Find all failed payment webhooks:**
```
service("VaultLink-WebhookService") 
  AND http.status = 500 
  AND annotation.PaymentProvider = "Stripe"
```

**Find slow payment processing:**
```
service("VaultLink-PaymentService") 
  AND duration >= 3
```

**Trace specific payment:**
```
annotation.PaymentId = "pay_abc123"
```

### CloudWatch Logs Insights

**Find logs for specific correlation ID:**
```
fields @timestamp, @message
| filter @message like /correlationId.*abc-123-def/
| sort @timestamp desc
```

**Payment webhook failures:**
```
fields @timestamp, service, correlationId, error
| filter service = "WebhookService" 
  AND error like /DynamoDB/
| stats count() by bin(5m)
```

---

## Critical Payment Scenarios

### Scenario 1: Webhook Delivery Failure

**Problem:** Stripe webhook never reaches VaultLink (network issue, timeout, etc.)

**Detection:**
- Stripe Dashboard shows webhook sent
- No X-Ray trace in VaultLink for that webhook
- CloudWatch shows no Lambda invocation

**Resolution:**
- Stripe retries webhooks automatically (up to 3 days)
- VaultLink should implement idempotency (check if payment already processed)
- Alert if webhook delay > 5 minutes

### Scenario 2: Webhook Processing Failure

**Problem:** Webhook received but DynamoDB update fails

**Detection:**
- X-Ray trace shows webhook Lambda invoked
- Trace segment shows DynamoDB error
- CloudWatch Logs show exception details

**Resolution:**
- Lambda DLQ captures failed invocations
- Alert triggers for webhook processing errors
- Manual intervention or retry mechanism

### Scenario 3: Prism Notification Failure

**Problem:** Payment successful, VaultLink updated, but Prism not notified

**Detection:**
- X-Ray trace shows successful payment processing
- SQS message sent but Lambda delivery function fails
- Message moves to Dead Letter Queue

**Resolution:**
- DLQ monitoring alerts team
- Replay messages from DLQ after fixing issue
- X-Ray trace correlates original payment to failed delivery

---

## Monitoring & Alerting Setup

### CloudWatch Alarms

1. **Payment Webhook Failures**
   - Metric: Lambda errors for WebhookService
   - Threshold: > 0 in 5 minutes
   - Action: SNS  PagerDuty

2. **Prism Delivery DLQ Depth**
   - Metric: SQS DLQ message count
   - Threshold: > 5 messages
   - Action: SNS  Slack #alerts

3. **X-Ray Error Rate**
   - Metric: X-Ray error percentage
   - Threshold: > 1% over 10 minutes
   - Action: SNS  Email

### X-Ray Service Map Alerts

Monitor service dependencies for:
- Latency increases (P95 > 500ms)
- Error rate spikes (> 1%)
- Throttling issues (429 responses)

### Data Retention

- **Trace data:** 30 days from recording (**HARD LIMIT - cannot be extended in X-Ray**)
- **Service graph data:** 30 days
- **No additional cost** for retention within 30-day window

**For longer retention (compliance, audit, chargebacks):**

**Option 1: Export to S3 (Recommended)**
- Set up automated export before 30-day expiry
- Query historical traces via Amazon Athena
- **Cost:** ~$0.023/GB/month (S3 Standard) + Athena query costs
- **Use case:** Long-term audit trails, compliance requirements

**Option 2: Rely on CloudWatch Logs**
- X-Ray traces linked to CloudWatch Logs (already configured)
- CloudWatch Logs retention: Configurable (1 day to 10 years)
- Search using correlation IDs in CloudWatch Logs Insights
- **Cost:** $0.50/GB ingested, $0.03/GB stored/month
- **Use case:** Log-based debugging when trace expired

**Recommendation for VaultLink:**
- **MVP:** 30-day X-Ray retention sufficient for operational debugging
- **Post-MVP:** Evaluate S3 export if compliance requires 90+ day payment audit trail
- **Always:** CloudWatch Logs retention set to 90 days minimum for payment-related logs

---

## Cost Estimation

### X-Ray Pricing (Example MVP Scale)

**AWS X-Ray Free Tier (Perpetual):**
- First 100,000 traces recorded per month: FREE
- First 1,000,000 traces retrieved or scanned per month: FREE

**Paid Tier:**
- $5.00 per 1 million traces recorded ($0.000005 per trace)
- $0.50 per 1 million traces retrieved or scanned ($0.0000005 per trace)

**VaultLink MVP Assumptions:**
- 10,000 payments/month
- 5 traced requests per payment (initial request, webhook, confirmations, internal calls)
- 50,000 total traces/month
- 100 queries/day × 200 traces scanned per query = 620,000 traces scanned/month
- 50 traces retrieved per query × 100 queries × 31 days = 155,000 traces retrieved/month

**Monthly Cost Calculation:**

**Traces Recorded:**
- 50,000 traces/month - 100,000 free tier = **0 billable traces**
- Cost: **$0.00** (within free tier)

**Traces Retrieved/Scanned:**
- Total: 620,000 + 155,000 = 775,000 traces
- 775,000 - 1,000,000 free tier = **0 billable traces**
- Cost: **$0.00** (within free tier)

**Total MVP X-Ray Cost:** **$0.00/month**

---

### Extended Retention Costs (If Required)

**S3 Export Scenario (90-day compliance retention):**

**Assumptions:**
- 50,000 traces/month × 3 months = 150,000 traces in S3
- Average trace size: ~5 KB (includes segments, metadata)
- Storage: 150,000 × 5 KB = 750 MB (~0.75 GB)

**S3 Storage Cost:**
- 0.75 GB × $0.023/GB = **$0.02/month**

**Athena Query Cost (occasional historical analysis):**
- 10 queries/month scanning 750 MB each
- 7.5 GB scanned × $5/TB = **$0.04/month**

**Total Extended Retention Cost:** **~$0.06/month** (negligible)

**CloudWatch Logs Alternative (90-day retention):**
- Assuming logs already ingested for Lambda execution
- No additional ingestion cost (logs exist anyway)
- Storage: ~2 GB/month × $0.03/GB = **$0.06/month**
- CloudWatch Logs Insights queries: ~$0.005 per GB scanned

**Total CloudWatch Extended Retention:** **~$0.06-$0.10/month**
- Cost: **$0.00** (within free tier)

**Total X-Ray Cost for MVP: $0.00/month**

**At Scale (50,000 payments/month):**
- 250,000 traces recorded - 100,000 free = 150,000 × $0.000005 = **$0.75/month**
- Trace retrieval still within free tier = **$0.00**
- **Total: $0.75/month**

**Cost is negligible even at 5× MVP scale.** Can use 100% sampling rate without cost concerns.

---

## Implementation Checklist

### Phase 1: Infrastructure Setup (Week 1)
- [ ] Enable X-Ray on all Lambda functions (CDK/CloudFormation)
- [ ] Enable X-Ray on API Gateway
- [ ] Configure X-Ray sampling rules (start with default: 1 req/sec + 5%)
- [ ] Verify X-Ray IAM permissions for Lambda execution roles

### Phase 2: Code Instrumentation (Week 2-3)
- [ ] Install X-Ray SDK in .NET projects (NuGet: AWSXRayRecorder.Handlers.AwsSdk)
- [ ] Instrument payment-critical code paths
- [ ] Add correlation IDs to all requests
- [ ] Add custom subsegments for Stripe API calls
- [ ] Add metadata/annotations for payment IDs, customer IDs

### Phase 3: Monitoring Setup (Week 3-4)
- [ ] Configure CloudWatch Logs integration
- [ ] Create X-Ray service map dashboard
- [ ] Set up CloudWatch alarms for payment failures
- [ ] Create CloudWatch Logs Insights saved queries
- [ ] Configure DLQ monitoring for webhook failures

### Phase 4: Team Enablement (Week 4)
- [ ] Document debugging procedures for team
- [ ] Train developers on X-Ray query syntax
- [ ] Create runbook for common failure scenarios
- [ ] Test end-to-end tracing in dev/staging environments
- [ ] Conduct simulated incident response drill

---

## Additional Considerations

### Sampling Strategy
- **Critical paths (payments, webhooks):** 100% sampling (trace every request)
- **Read-only queries (statements):** 10% sampling or default (1 req/sec + 5%)
- **Health checks:** 0% sampling (no tracing)
- Configure via X-Ray sampling rules in Console or CDK

### Performance Impact
- X-Ray SDK overhead: <1ms per request
- Network call to X-Ray service: Asynchronous, non-blocking
- Lambda cold start increase: +50-100ms (one-time per container)
- **Negligible impact on user experience**

### Security Considerations
- **Do NOT trace PII** (card numbers, personal data) in annotations or metadata
- Use annotations for: PaymentId, CustomerId, CorrelationId (hashed/anonymized)
- Sensitive data in subsegment metadata only if needed for debugging
- X-Ray data encrypted at rest (AWS managed keys)

### Compliance & Audit Requirements
- **Question for Synertec:** Are there regulatory requirements for payment trace retention >30 days?
- **Financial regulations** may require 90-day to 7-year audit trails
- **Recommendation:** Confirm retention policy before finalizing S3 export strategy
- **Action:** Add to discovery questions if not already addressed

---

## Success Criteria

- **Complete Visibility:** Every payment request traceable from entry to completion
- **Fast Diagnosis:** Identify failure point within 5 minutes of incident
- **Correlation:** Link Stripe webhook events to VaultLink processing
- **Alerting:** Automated alerts for payment processing failures
- **Team Readiness:** All developers trained on X-Ray query syntax

---

## Additional Considerations

### Sampling Strategy

**Default X-Ray Sampling:**
- First request each second: 100% traced
- Additional requests: 5% sampled
- Configurable via X-Ray sampling rules

**Recommendation for VaultLink:**
- **Payment endpoints:** 100% sampling (critical path)
- **Webhook handlers:** 100% sampling (troubleshooting essential)
- **Background jobs:** 10% sampling (lower priority)
- **Health checks:** 0% sampling (noise reduction)

### Performance Impact

- X-Ray SDK overhead: <1ms per traced request
- Network overhead: Segments sent via UDP to local daemon (non-blocking)
- Lambda cold start: +50-100ms when X-Ray enabled
- No impact on Lambda execution time after cold start

### Security Considerations

- X-Ray traces may contain sensitive data (sanitize payment details)
- Use metadata (not indexed) for PII/sensitive information
- Use annotations (indexed) only for non-sensitive search fields
- IAM policies restrict X-Ray console access to authorized personnel

## Resources

- [AWS X-Ray Developer Guide](https://docs.aws.amazon.com/xray/latest/devguide/)
- [AWS X-Ray Pricing](https://aws.amazon.com/xray/pricing/)
- [X-Ray .NET SDK Documentation](https://github.com/aws/aws-xray-sdk-dotnet)
- [X-Ray Concepts (Segments, Traces, Sampling)](https://docs.aws.amazon.com/xray/latest/devguide/xray-concepts.html)
- [X-Ray Service Integrations](https://docs.aws.amazon.com/xray/latest/devguide/xray-services.html)
- [CloudWatch Logs Insights Query Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [Stripe Webhook Best Practices](https://stripe.com/docs/webhooks/best-practices)

---

**For MVP scope, see `brief.md`**  
**For infrastructure details, see `aws-infrastructure-services.md`**  
**For monitoring strategy, see `epics.md` (Epic 11)**

---

**Document End**
