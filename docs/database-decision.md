# VaultLink Database Decision: NoSQL vs SQL

**Document Version:** 1.0  
**Date:** October 17, 2025  
**Client:** Synertec  
**Project:** VaultLink MVP AWS Rebuild  
**Purpose:** Database Technology Assessment

---

## Executive Summary

**Recommendation:** Use **NoSQL (DynamoDB)** for VaultLink MVP.

DynamoDB is well-suited for VaultLink's access patterns, serverless architecture, and multi-tenant requirements. The benefits of auto-scaling, low latency, and natural multi-tenant isolation outweigh the limitations for this use case.

---

## Why NoSQL (DynamoDB) Fits VaultLink

### Strong Alignment with Requirements

**1. Access Patterns Match NoSQL Strengths**
- Primary use case: Retrieve statement by reference number/QR code (simple key-value lookup)
- Multi-tenant data isolation naturally modeled with partition keys
- Read-heavy workload (end-users viewing statements)

**2. Serverless Architecture Integration**
- AWS Lambda + DynamoDB is a proven, battle-tested combination
- No server management required
- Auto-scaling built-in
- Pay-per-request pricing aligns with variable B2B2C usage patterns

**3. Performance at Scale**
- Sub-10ms latency for statement retrieval
- Unlimited throughput capacity
- Meets <3s page load and <500ms API response requirements

**4. Multi-Tenant Security**
- Row-level security via partition keys (CustomerId)
- Natural data isolation per customer
- IAM policies enforce tenant boundaries

**5. Schema Flexibility**
- Customer-specific configurations vary per tenant (white-label branding)
- Document metadata can evolve without database migrations
- Payment transaction records benefit from flexible JSON structure

---

## Key Considerations

### Areas Requiring Planning

**1. Query Flexibility**
- **Challenge:** Complex queries like "all payments in date range" require Global Secondary Indexes (GSIs)
- **Solution:** Design GSIs upfront for known access patterns

**2. No Complex Joins**
- **Challenge:** NoSQL requires denormalization or multiple queries
- **Solution:** Store frequently accessed data together or aggregate in application layer

**3. Reporting & Analytics**
- **Challenge:** Complex aggregations and ad-hoc queries are limited
- **Solution:** Use DynamoDB Streams to export data to S3/Athena for analytics

**4. Team Learning Curve**
- **Challenge:** Team must shift from relational to NoSQL thinking
- **Solution:** Invest 2-3 days in single-table design workshop before implementation

---

## VaultLink Access Patterns Summary

| Use Case | DynamoDB Fit | Approach |
|----------|--------------|----------|
| Retrieve statement by reference | Excellent | Direct key lookup |
| List statements for customer | Excellent | Query by partition key |
| Get payment history | Good | Query with sort key filter |
| Customer configuration/branding | Excellent | Single-item retrieval |
| Document metadata | Excellent | Query by statement ID |
| Admin reporting/analytics | Moderate | Stream to S3/Athena |

---

## When SQL Would Be Better

Consider **RDS/Aurora** instead if:
- Frequent ad-hoc queries with unpredictable patterns
- Complex reporting is a core MVP requirement
- Multi-table ACID transactions are critical
- Regulatory mandate for SQL database

**For VaultLink:** None of these apply. Access patterns are well-defined and predictable.

---

## Recommended Approach

### Single-Table Design

Use one DynamoDB table with composite keys:

**Structure:**
- **Partition Key (PK):** Customer identifier or entity type
- **Sort Key (SK):** Entity identifier with type prefix
- **Attributes:** Flexible data per entity

**Example Entities:**
- Customer metadata: `CUSTOMER#123 / METADATA`
- Statement: `CUSTOMER#123 / STATEMENT#456`
- Payment: `CUSTOMER#123 / STATEMENT#456#PAYMENT#789`
- Document: `CUSTOMER#123 / STATEMENT#456#DOCUMENT#101`

### Required Global Secondary Indexes

1. **GSI-StatementRef:** Lookup by reference number
2. **GSI-PaymentDate:** List payments by date range
3. **GSI-UserEmail:** Admin user lookup

---

## Implementation Plan

### Phase 1: Foundation (Week 1-2)
- Conduct single-table design workshop
- Document all access patterns
- Design primary key structure and GSIs
- Create DynamoDB table with infrastructure as code

### Phase 2: Development (Week 3-14)
- Implement data access layer
- Test with realistic data volumes
- Monitor and optimize query performance

### Phase 3: Analytics (Week 15-17)
- Configure DynamoDB Streams
- Set up export pipeline to S3
- Create Athena queries for reporting

---

## Success Criteria

- **Performance:** <10ms query latency
- **Scalability:** Handle 10x traffic without changes
- **Multi-Tenancy:** 100% data isolation between customers
- **Team Readiness:** Developers comfortable with patterns by Week 3

---

## Resources

- [The DynamoDB Book by Alex DeBrie](https://www.dynamodbbook.com/)
- [AWS DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [Single-Table Design Video (Rick Houlihan)](https://www.youtube.com/watch?v=HaEPXoXVf2k)

---

**For MVP scope, see `brief.md`**  
**For infrastructure details, see `aws-infrastructure-services.md`**  
**For epic breakdown, see `epics.md`**

---

**Document End**
