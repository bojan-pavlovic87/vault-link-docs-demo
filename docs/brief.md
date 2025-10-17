# VaultLink MVP Project Brief

**Document Version:** 1.0  
**Date:** October 17, 2025  
**Client:** Synertec  
**Focus:** Minimum Viable Product Scope & Requirements

---

## Executive Summary

VaultLink is a payment facilitation platform enabling the public to make payments using QR codes or reference numbers. Synertec has engaged Amdaris to rebuild VaultLink from the ground up on AWS, transitioning from Azure-hosted C# desktop applications to a modern cloud-native architecture.

**Project Nature:** Ground-up rebuild (not migration) - greenfield development using existing VaultLink as functional reference.

**System Scope:** VaultLink comprises 5 applications, but only **3 are in scope** for this rebuild:
1. ✅ **Statements** - View and pay statement line items
2. ✅ **Invoices** - View and pay invoices (includes cumulative invoice feature)
3. ✅ **Credit Control** - View and pay overdue statements with resolution comments
4. ❌ **NHS Patient Prefs** - Out of scope (secure NHS API requirements)
5. ❌ **NHS Comms** - Out of scope (different business requirements)

**Primary Problem:** Current Azure-hosted system has scalability limitations, high infrastructure costs, poor payment visibility, and operational overhead from maintaining multiple customer deployments.

**MVP Goal:** Rebuild core VaultLink functionality with improved UX - "same features, nicer looking" - enabling customers to view statements, check documents, and make payments through an intuitive mobile-first interface.

**Key Value Proposition:** 
- AWS serverless architecture reduces TCO by 40%
- Stripe Connect provides streamlined payment processing with PCI SAQ A compliance
- Cloud-native SaaS eliminates on-premises maintenance burden
- Modern tech stack enables 3x faster feature delivery

---

## Problem Statement

### Current Challenges

**Azure-Hosted Infrastructure:**
- High Azure infrastructure costs limiting scalability
- Current architecture:
  - Azure FrontDoor for WAF
  - 2 Windows VMs ("Concentrator": AZ-VL-CONC1/2) running C# desktop apps
  - MySQL HA cluster (2 instances, 3 tables total, 1 significant table)
  - CDN for content delivery
  - "Responder" component routing users to correct app with URL parameters
- Complex deployment and upgrade processes
- Infrastructure costs driving need for AWS migration

**Payment Processing:**
- Poor visibility into payment flows and transaction statuses
- Currently supports 4 payment providers (CardStream, WorldPay, Stripe, Elavon)
- All providers use Hosted Payment Pages (HPP) returning to Synertec-hosted URL
- Manual reconciliation requirements
- Customer frustration with payment experience
- **Note:** Stripe's specific role in new system to be confirmed (Action: Richard)

**Technical Debt:**
- C# desktop applications served via Windows VMs (non-cloud-native)
- File-based Prism integration with polling (lacks reliability and visibility)
- MySQL HA cluster used in non-relational patterns (3 tables, no relationships)
  - Contains "DYNData" field (Prism-encoded structured data, NOT JSON)
  - Only 1 significant table out of 3 total
- Customer configuration stored on file share ("P drive") with strict naming conventions
- Limited monitoring and observability
- Current performance: 264ms average latency, 500ms max, 50K peak requests at month-start

**Why Now:**
- Cost of Azure system and project cost of scaling
- Aligning with AWS development happening elsewhere in Synertec for lower cost base
- Market opportunity for improved payment UX
- Synertec's strategic shift to cloud SaaS model
- Current architecture reaching end of sustainable life
- Customer demands for modern interface
- Need for scalable-by-design architecture (capable of scaling when volumes indicate)

---

## Proposed Solution

### Technology Stack

**Frontend:**
- Angular 20+ SPA hosted on S3 + CloudFront CDN
- Angular Material or custom component library
- Mobile-first responsive design

**Backend:**
- .NET 8 on AWS Lambda for APIs and background jobs
- API Gateway for routing
- Elastic Beanstalk fallback for long-running processes

**Data & Integration:**
- DynamoDB (NoSQL) with DAX caching
- SQS for Prism integration messaging
- S3 for documents and customer configurations

**Security & Payments:**
- AWS Cognito for admin authentication
  - Customer setup and migration exercise for existing customers
  - VaultLink currently small scale - not migrating existing passwords
  - **Question:** Admin capability for customers to manage their users? (Cognito dashboard is AWS-focused, Richard pursuing with Synertec)
- Payment provider integration:
  - **Stripe Checkout** (not Stripe Elements - provides default behavior with branding capability)
  - No need for PCI SAQ A with Stripe Checkout (Stripe handles compliance)
  - **Connected Account Model:** Standard Connected Account for Synertec's customers
  - Customers manage their own Stripe accounts, Synertec doesn't want this responsibility
  - Payment methods: Whatever Stripe Checkout supports (can include payment plans in some circumstances)
  - Customers choose valid payment options via Stripe dashboard during onboarding
  - Other payment methods beyond Stripe Checkout: Out of scope
- All providers use Hosted Payment Pages (HPP) for compliance
- CloudFront + WAF for DDoS protection and IP blocking
- WAF rules for attack protection (DDoS, SQL injection, XSS)

**Operations:**
- CloudWatch for monitoring and logging
  - **Distributed Tracing:** AWS X-Ray (or Tempo if similar capability found) for cross-service request tracking
  - Critical for payment flows: Track entire transaction from request through webhook to system update
  - Example scenario: Payment made but webhook crashes - need view of entire transaction
- EventBridge for alert routing to SNS/Slack/Email
- AWS Backup for automated backups (backup verification required)
- Infrastructure as Code (CDK/CloudFormation)

### Key Benefits

**Multi-Tenant SaaS:**
- Eliminates per-customer deployments
- Centralized management
- Instant feature rollout
- Customer configs in S3 (migrated from P drive file share)
- Most customers don't customize branding (KISS principle applies)
- Lightweight upload mechanism for technical support
- Versioning support maintained from current system

**Serverless-First:**
- Pay-per-use cost model
- Auto-scaling
- No server management
- Built-in HA

**Modern Payments:**
- Stripe Connect white-label processing
- Real-time visibility
- PCI SAQ A compliance
- Multiple payment methods

**Observable:**
- Centralized CloudWatch logging
- EventBridge alert routing
- Distributed tracing
- Automated backups

---

## Target Users

### Primary User Segment: End Consumer (Third-Party Customers - 3Cs)

**Profile:**
- **Demographics:** General public, all age groups (18-80+)
- **Technical Proficiency:** Low to moderate - system must be accessible to non-technical users
- **Device Usage:** Primarily mobile phones (70-80% expected), secondary desktop/tablet
- **Context:** Receiving bills/invoices from organizations (utilities, local government, healthcare, etc.)
- **Access Method:** Receives email or SMS with URL containing VaultLink reference

**VaultLink Reference Technical Details:**
- **Key Structure:** 49-character base64 encoded string (generated from 3 salts with hashing, non-reversible)
- **URL Format:** `&H4=xxxx` plus 3-character customer ID (2 numeric + 1 alpha uppercase/lowercase)
- **Customer Limit:** ~30 customers currently, well within projected growth limits

**Current Behaviors & Workflows:**
1. Receives letter, SMS, or email with VaultLink URL (payment reference embedded)
2. Clicks link to view statement details and outstanding amounts
3. Views line items in grid representing jobs (from Prism)
4. Makes payment via Hosted Payment Page (CardStream, WorldPay, Stripe, or Elavon)
5. May submit queries or disputes with predefined reasons or free text
6. May request Proof of Delivery (POD) for specific line items

**Specific Needs & Pain Points:**
- **Convenience:** Quick access via QR scan or reference number entry
- **Trust:** Clear branding showing this is legitimate payment channel for their service provider
- **Clarity:** Easy-to-understand statement breakdowns (30-day, 60-day, 90-day aging)
- **Simplicity:** Minimal steps from landing to payment completion
- **Confirmation:** Immediate payment receipt and status visibility
- **Support:** Ability to query transactions or report issues

**Goals They're Trying to Achieve:**
- Pay bills quickly without logging in or creating accounts
- Understand what they're being charged for
- Get proof of payment immediately
- Resolve payment issues or disputes
- Access payment history when needed

---

### Secondary User Segment: Synertec Customer (Organization/Enterprise Client - SCs)

**Profile:**
- **Organization Type:** Utilities, local councils, healthcare providers, financial services
- **Roles:** Finance managers, operations managers, customer service teams
- **Technical Proficiency:** Moderate - comfortable with business software
- **Scale:** Managing thousands to hundreds of thousands of third-party customer accounts
- **Integration:** Already set up in Prism where document jobs are configured and fulfilled

**Current Behaviors & Workflows:**

**Statements Application:**
1. View line items in grid representing jobs from Prism
2. Perform reconciliation - amend invoices if customer queried
3. Request Proof of Delivery (POD) for specific jobs
4. Filter on invoice/creator note number
5. Submit items for sending payment requests to 3Cs (email or SMS)
6. Monitor payment collection and aging reports

**Invoices Application:**
1. Similar to Statements with additional "cumulative invoices" feature
2. Build up running total to reflect 3C's position
3. Send single payment request for cumulative amount

**Credit Control Application:**
1. View "overdue statements" with specialized UI
2. Request payments for overdue items
3. Add free-format comments fed back to Prism for resolution suggestions
4. Track aging and collection rates

**Customization Requirements:**
- Limited branding: 3 logo sizes, custom wording/copy (simple customization already exists - talk to Ed)
- Must provide privacy link (Synertec is GDPR data controller)
- Synertec provides cookies policy
- Most customers don't customize (not all require customization)
- **Note:** Refunds policy is customer's own responsibility, not part of VaultLink application

**Specific Needs & Pain Points:**
- **Visibility:** Real-time view of payment activity and collection rates
- **Customization:** White-label experience reflecting their brand
- **Integration:** Seamless connection with their back-office systems (Prism)
- **Reporting:** Comprehensive reconciliation and audit trails
- **Support:** Tools to quickly resolve end-consumer queries
- **Compliance:** PCI, data protection, audit requirements met automatically

**Goals Trying to Achieve:**
- Increase payment collection rates
- Reduce administrative overhead in payment processing
- Provide excellent customer experience to their end consumers
- Maintain compliance with financial regulations
- Minimize integration complexity with existing systems

---

### Tertiary User Segment: Synertec Operations Team

**Profile:** Technical support, customer success, system administrators managing system health and customer onboarding

**Needs:**
- Centralized monitoring and health dashboards
- Quick issue diagnosis tools
- Efficient customer onboarding (currently takes 4-6 weeks)
- Zero-downtime deployments using modern toolsets (Terraform)
- Support multiple environments (DEV, STAGE, PROD)
- Lightweight mechanism to upload customer branding assets
- Access to unit tests as part of CI/CD deliverables

**Goals:**
- Support growing customer base without linear cost scaling
- Proactively identify issues
- Reduce mean-time-to-resolution
- Reduce onboarding time from 4-6 weeks to <1 week
- Enable self-service customer configuration (post-MVP)

---

## Goals & Success Metrics

### Business Objectives

- **Reduce Infrastructure Costs** through AWS serverless architecture vs. current Azure hosting (cost of Azure system and scaling drove this project)
- **Enable Scalability** by design (not necessarily configured for scale initially, but capable when volumes indicate)
- **Increase Payment Collection Rate by 15%** through improved user experience, reduced friction in payment flow, and enhanced mobile accessibility
- **Decrease Support Ticket Volume by 50%** by improving system observability, centralizing operations, and providing better self-service capabilities for end users
- **Accelerate Feature Delivery by 3x** by moving from quarterly to monthly release cycles enabled by CI/CD pipeline and cloud-native architecture
- **Achieve 99.9% Uptime SLA** leveraging AWS managed services with built-in redundancy and automated failover
- **Onboard New Customers 5x Faster** by eliminating on-premises installation requirements and implementing self-service configuration

### User Success Metrics

**End Consumer Experience:**
- Payment completion rate >85% (from landing page to successful payment)
- Average time-to-payment <2 minutes from QR scan or reference entry
- Mobile usability score >4.5/5.0
- Payment query resolution time <24 hours
- User satisfaction (NPS) >40

**Enterprise Client Experience:**
- Customer onboarding time <1 week (vs. current 4-6 weeks)
- Payment reconciliation automation rate >95%
- System configuration changes deployable in <1 day
- Financial reporting available in real-time (vs. current end-of-day batch)
- Customer satisfaction score >4.2/5.0

**Synertec Operations Team:**
- Mean-time-to-detect (MTTD) issues <5 minutes via CloudWatch
- Mean-time-to-resolve (MTTR) incidents <2 hours
- Zero-downtime deployment success rate 100%
- Support team productivity: handle 3x customer base with same team size

### Key Performance Indicators (KPIs)

**Current Performance Baseline (Azure System):**
- **Latency:** 264ms average, 500ms maximum
- **Traffic:** Low daily requests, 50K peak at month-start
- **Page Sizes:** Low KB range (few images, minimal data)
- **Database:** Low storage requirements (Harry to confirm exact sizes)

**Target Performance (AWS System):**
- **Infrastructure Cost:** Reduce AWS costs vs. current Azure costs
- **System Availability:** 99.9% uptime measured monthly (43 minutes downtime/month)
- **API Response Time (P95):** Match or improve current 264ms average, <500ms maximum
- **Payment Processing Success Rate:** >99.5% successful HPP transactions (excluding user errors)
- **Security & Compliance:** Maintain PCI SAQ A compliance continuously (HPP model)
- **Data Integrity:** 100% message delivery guarantee for Prism integration via SQS with DLQ monitoring
- **Monitoring Coverage:** 100% of critical paths instrumented with CloudWatch metrics and alerts
- **Scalability:** Handle 50K+ peak requests without performance degradation

---

## MVP Scope & Success Criteria

### Core Features (Must Have)

**Three Applications in Scope:**

**1. Statements Application:**
- Anonymous access via VaultLink URL (49-char base64 encoded reference)
- View statement line items in grid (jobs from Prism)
- Aging breakdown visualization (30/60/90/90+ days)
- Payment processing via Stripe Checkout (not Stripe Elements)
  - Stripe Checkout provides default behavior with branding capability
  - No PCI SAQ A required (Stripe handles compliance)
  - Payment methods: Whatever Stripe Checkout supports (can include payment plans)
  - Customers choose valid options via Stripe dashboard
- Full and partial payment support
- Request Proof of Delivery (POD)
- Submit queries with predefined reasons or free text
- Filter on invoice/creator note number
- Immediate payment confirmation and receipts

**2. Invoices Application:**
- All Statements features PLUS:
- Cumulative invoices feature (running total for 3C's position)
- Single payment request for cumulative amount

**3. Credit Control Application:**
- View overdue statements with specialized UI
- Request payments for overdue items
- Add free-format comments (fed back to Prism for resolution suggestions)
- Track aging and collection rates

**Document Management (All Apps):**
- View, search, filter documents (invoices, credits, PODs)
- Time-based grouping (Last Week, +2 Weeks Ago, etc.)
- Batch operations (Agree, Query, Copy, POD)
- Documents contain DYNData field (Prism-encoded structured data)

**Query & Dispute (All Apps):**
- Submit queries on invoices with predefined options
- Free-text "Other" option
- Query notifications sent to Prism via SQS
- Query tracking and status visibility

**Multi-Tenancy:**
- Customer-specific branding (3 logo sizes, custom wording/copy)
- Simple customization already exists (logos and some text - talk to Ed for details)
- Branding stored in S3 (migrated from P drive file share)
- Most customers don't customize (KISS principle)
- Support for versioning (maintain from current system)
- Required links per customer:
  - Privacy link (SC supplies - Synertec is GDPR data controller)
  - Cookies policy (Synertec provides)
  - **Note:** Refunds policy is customer's own responsibility (not part of VaultLink)
- "Powered by Synertec" footer
- Per-customer configuration without code changes
- Lightweight upload mechanism for Synertec technical support

**Integration:**
- Replace file-based Prism polling with SQS messaging
- Lambda delivery functions for robust, visible communication
- Retry logic and dead-letter queues
- Payment confirmations sent to Prism
- Query notifications sent to Prism with free-format comments
- Message format to be specified during project
- Authentication mechanism TBD
- **Note:** Synertec will handle Prism-side changes

**Operations:**
- Mobile-first responsive Angular SPA (3 separate apps)
- CloudFront CDN for fast global delivery (replacing Azure CDN)
- WAF for DDoS protection, IP blocking, attack prevention
- Cognito authentication for admin/staff
- HTTPS enforcement via CloudFront + WAF
- CloudWatch monitoring + AWS X-Ray distributed tracing
  - Cross-service request tracking for payment flows
  - Track entire transaction (e.g., payment  webhook  system update)
- EventBridge alerting to support (email, SMS)
- Automated AWS Backup with verification
- Deploy using Terraform for multiple environments (DEV, STAGE, PROD)
- **Deploy in Europe (UK customers)** - affects AWS Region choice for data residency
- Unit tests delivered as part of CI/CD setup
- **E2E Testing:** Playwright (Synertec uses Playwright, Amdaris will provide tests as part of deliverables)
- Analytics integration (to be defined when we know what's achievable)

### Out of Scope for MVP

- **NHS Patient Prefs Application** (separate NHS API requirements)
- **NHS Comms Application** (different business requirements)
- Native mobile apps (web only, but mobile-responsive)
- Real-time push notifications
- Advanced analytics dashboards (basic analytics links to be defined when achievable)
- Customer self-service configuration portal
- **Recurring payments/subscriptions** (Stripe Checkout can include payment plans in some circumstances - customers choose options)
- Multi-language support (English only)
- Payment plans/installments (out of scope for this project)
- Integrations beyond Prism
- API marketplace
- Custom reporting builder
- SAML/OIDC SSO (basic Cognito only)
- PWA/offline mode
- Customer communication platform within VaultLink (SC sends emails/SMS with VaultLink URLs)
- Accessibility features beyond best-practice (NHS apps have basic accessibility, in-scope apps currently don't)
- **Other payment methods beyond Stripe Checkout** (out of scope)
- **Admin capability for customers to manage users** (Richard pursuing with Synertec - Cognito dashboard is AWS-focused)
- **Geographic blocking for non-EU/UK customers** (geo-blocking only for non-EU/UK)

### Success Criteria

**Functional:**
- All core features implemented and tested
- Feature parity with existing VaultLink
- Harry Pestell (PO) approval of all flows

**Performance:**
- API response <500ms (P95)
- Page load <3s on 3G mobile
- Support 500 concurrent users

**User Acceptance:**
- Pilot customer >80% payment completion rate
- "Nicer looking" goal validated vs. current system

**Technical Quality:**
- Zero critical security vulnerabilities
- 100% critical path test coverage
- 99% uptime during 2-week pilot
- 99.9% Prism message delivery

**Operational:**
- CloudWatch dashboards functional
- AWS X-Ray distributed tracing operational (cross-service request tracking)
- Alert procedures documented and tested
- Synertec team trained
- Disaster recovery tested
- **Backup verification completed**

**Compliance:**
- **Stripe Checkout compliance** (Stripe handles PCI compliance - no SAQ A required)
- Security audit passed with **SAST evidence** (DAST done by Synertec, out of scope for Amdaris)
- Data encryption verified
- GDPR-compliant data handling
- **ISO27001 support:** Amdaris will support Synertec's ISO27001 (out of scope for Amdaris to implement)
- **Penetration testing:** Out of scope for Amdaris (Synertec responsibility)

---

## Technical Overview

### Performance Requirements
- API Response: <500ms (P95) payment, <200ms statement retrieval
- Page Load: <3s on 3G mobile, <1s cached
- Throughput: 500 concurrent users (MVP), scalable to 5000+
- Database: DynamoDB single-digit ms latency, DAX sub-ms caching

**Technology Choices

**Frontend:** Angular 20+, Signals state management, Angular Material/custom UI, **Playwright for E2E testing** (Synertec uses Playwright)

**Backend:** .NET 8, Lambda for APIs, Elastic Beanstalk fallback, ASP.NET Core Web API, xUnit testing

**Data:** DynamoDB (NoSQL) - ✅ **Validated in `database-decision.md`** (strong alignment with access patterns, serverless architecture, multi-tenant security), DAX caching, ElastiCache Redis (optional)

**Infrastructure:** AWS pure cloud, **Europe region (UK customers - affects data residency)**, CloudFront CDN, API Gateway, CDK/CloudFormation IaC

**Testing Platforms:**
- **Automated testing:** All suggested platforms (comprehensive browser/device coverage)
- **Manual testing:** 1 mobile + 1 desktop platform

###Security & Compliance
- **Stripe Checkout compliance** (Stripe handles PCI - no SAQ A required from VaultLink)
- **ISO27001 alignment:** Amdaris supports Synertec's ISO27001, not implementing for them (out of scope)
- Data encryption at rest (DynamoDB, S3) and in transit (TLS 1.2+)
- Cognito MFA for admin/staff
- CloudTrail audit logs (90-day retention)
- **SAST in CI/CD pipeline** (DAST by Synertec - not typically part of CI/CD, Synertec handles)
- **Penetration testing:** Synertec responsibility (out of scope for Amdaris)
- **Geographic considerations:** Deploy in Europe for UK customers (data residency)

### Key Integrations

**Prism:**
- SQS-based messaging
- Lambda delivery function
- TBD: message format, auth, endpoint
- 3 retries + exponential backoff → DLQ

**Stripe Connect:**
- Stripe Elements for PCI compliance
- Webhook for payment status
- Connected account multi-tenancy
- TBD: fee structure, account model

**Stripe Checkout:**
- **Connected Account Model:** Standard Connected Account
- Customers manage their own Stripe accounts (Synertec doesn't manage)
- Customers choose payment options during onboarding (Richard can explain)
- Webhook for payment status updates
- Payment methods: Whatever Stripe Checkout supports (can include payment plans)
- **No Stripe Elements needed** - using Stripe Checkout for default behavior with branding
- **No PCI SAQ A required** - Stripe handles compliance

**AWS Services:**
- Cognito (auth), Lambda (compute), DynamoDB (data)
- S3 (storage), CloudFront (CDN), API Gateway (routing)
- SQS (messaging), SNS (notifications), EventBridge (events)
- CloudWatch (monitoring), **X-Ray (distributed tracing)**, Backup (DR)

## Technical Considerations

### Platform Requirements

- **Target Platforms:** Web application (primary), mobile-responsive design
- **Browser/OS Support:** 
  - **Modern Browsers:** Chrome 90+, Firefox 88+, Safari 14+, Edge 90+ (last 2 years)
  - **Mobile Browsers:** iOS Safari 14+, Chrome Mobile 90+, Samsung Internet
  - **Progressive Enhancement:** Core payment flow functional on older browsers, enhanced UX on modern browsers
  - **Operating Systems:** Platform-agnostic (web-based), no OS-specific dependencies
- **Performance Requirements:**
  - **API Response Time:** <500ms (P95) for payment initiation, <200ms for statement retrieval
  - **Page Load Time:** <3 seconds initial load on 3G mobile connection, <1 second cached repeat visits
  - **Time to Interactive (TTI):** <5 seconds on mobile
  - **Throughput:** Support 500 concurrent users initially, scale to 5000+ with no architectural changes
  - **Database Performance:** DynamoDB single-digit millisecond latency, DAX sub-millisecond caching

### Technology Preferences

**Frontend Stack:**
- **Framework:** Angular (latest LTS version, currently v20+)
- **State Management:** Angular Signals + Computed Signals (Angular 20+), consider NgRx for complex state if needed
- **UI Components:** Angular Material or custom component library aligned with \"nicer looking\" requirement
- **Build Tool:** Angular CLI with optimized production builds
- **Package Management:** npm
- **Testing:** Jasmine/Karma for unit tests, Cypress for E2E tests

**Backend Stack:**
- **Language:** .NET 8 (C# latest LTS)
- **Compute:** Hybrid approach
  - **AWS Lambda** for API endpoints (API Gateway integration)
  - **AWS Lambda** for background jobs (SQS consumers, scheduled tasks)
  - **AWS Lambda** for delivery systems (Prism integration)
  - **Elastic Beanstalk/EC2** reserved for long-running processes if Lambda 15-min limit insufficient
- **API Framework:** ASP.NET Core Web API or AWS Lambda ASP.NET Core adapter
- **Package Management:** NuGet
- **Testing:** xUnit for unit tests, integration test suite for Lambda functions

**Database & Caching:**
- **Primary Database:** Amazon DynamoDB (NoSQL)
  - Design tables around access patterns (single-table design consideration)
  - Global Secondary Indexes (GSIs) for alternate query patterns
  - On-demand billing initially, evaluate provisioned capacity at scale
- **Caching Layer:** DAX (DynamoDB Accelerator) for read-heavy operations
  - Consider for statement retrieval and document lookups
  - Evaluate cost/benefit during performance testing
- **Session Storage:** ElastiCache (Redis) if stateful sessions needed beyond Cognito

**Hosting/Infrastructure:**
- **Cloud Provider:** AWS (no hybrid, pure AWS)
- **Regions:** Single region initially (eu-west-2 London or us-east-1), evaluate multi-region for HA later
- **CDN:** CloudFront for static asset delivery and API caching
- **WAF:** AWS WAF integrated with CloudFront for DDoS protection and custom security rules
- **Load Balancing:** API Gateway handles API traffic, CloudFront handles static assets
- **Infrastructure as Code:** AWS CDK or CloudFormation for reproducible environments

### Architecture Considerations

**Repository Structure:**
\\\
vaultlink-monorepo/
 frontend/                 # Angular SPA
    src/
    angular.json
    package.json
 backend/                  # .NET backend
    VaultLink.API/       # API Gateway + Lambda functions
    VaultLink.Core/      # Domain logic
    VaultLink.Infrastructure/  # AWS integrations
    VaultLink.DeliveryService/  # Prism integration Lambda
    VaultLink.sln
 infrastructure/           # IaC (CDK/CloudFormation)
    stacks/
    constructs/
 docs/                     # Architecture & discovery docs
 .github/workflows/        # CI/CD pipelines
\\\

**Service Architecture:**
- **API Gateway** as single entry point for all backend requests
- **Lambda Functions** organized by bounded context:
  - \StatementService\ - statement retrieval and calculations
  - \PaymentService\ - Stripe Checkout integration
  - \DocumentService\ - document management and retrieval
  - \QueryService\ - query submission and tracking
  - \PrismDeliveryService\ - SQS consumer  Prism sender
- **SQS Queues:**
  - vaultlink-prism-delivery-queue\ (standard queue for message delivery)
  - vaultlink-prism-dlq\ (dead-letter queue for failed deliveries)
- **S3 Buckets:**
  - vaultlink-static-assets\ (frontend SPA, CloudFront origin)
  - vaultlink-customer-configs\ (per-customer branding, configs)
  - vaultlink-documents\ (invoice PDFs, PODs)
  - vaultlink-backups\ (automated backup destination)

**Integration Requirements:**
- **Prism:** SQS-based messaging with Lambda delivery function
  - Message format: TBD during discovery (JSON assumed)
  - Authentication: TBD (IAM roles, API keys, or credentials)
  - Retry policy: 3 attempts with exponential backoff, then DLQ
- **Stripe Checkout:** 
  - Standard Connected Account for Synertec's customers
  - Customers manage own accounts (Synertec doesn't want this responsibility)
  - Customers choose payment options during onboarding (Richard can explain)
  - Use Stripe Checkout for default behavior with branding
  - Webhook endpoint for payment status updates
- **Cognito:**
  - User pool for admin/staff authentication
  - Optional public user pool if authenticated end-user features added post-MVP
  - SSO integration (SAML/OIDC) deferred to Phase 2

**Security/Compliance:**
- **Stripe Checkout Compliance:** Stripe handles PCI compliance (no SAQ A required for VaultLink)
- **ISO27001 Alignment:** Amdaris supports Synertec's ISO27001 (not implementing, out of scope)
- **Data Encryption:**
  - At rest: DynamoDB encryption, S3 server-side encryption (SSE-S3 or KMS)
  - In transit: TLS 1.2+ enforced via CloudFront and API Gateway
- **Authentication/Authorization:**
  - Cognito for admin/staff access with MFA
  - JWT tokens for API authorization
  - IAM roles for service-to-service (Lambda  DynamoDB, Lambda  SQS)
- **Security Testing:**
  - **SAST** (Static Application Security Testing) in CI/CD pipeline with evidence
  - **DAST** (Dynamic Application Security Testing) - Synertec responsibility (not typically in CI/CD)
  - Dependency vulnerability scanning (Dependabot)
- **Penetration Testing:** Synertec responsibility (out of scope for Amdaris)
- **Logging & Audit:**
  - CloudWatch Logs for all Lambda functions
  - CloudTrail for AWS API audit logs
  - VPC Flow Logs if private networking required
  - Retention: 90 days minimum for compliance

---

## Constraints & Assumptions

### Constraints

**Budget & Timeline:**
- AWS costs expected 40% lower than current on-prem + Azure
- Discovery: 2 weeks | MVP: TBD | Pilot: 2-4 weeks | Training: 6 months

**Team:**
- **Amdaris:** Richard (Architect), Bojan (Principal Dev), PM (TBD), Devs (TBD)
- **Synertec:** Harry (PO), Technical team (TBD)

**Technical:**
- Pure AWS (no hybrid cloud)
- Lambda 15-min limit, cold starts
- DynamoDB NoSQL access patterns critical
- Prism compatibility (no changes to Prism)
- Last 2 years browser support
- PCI SAQ A compliance mandatory

### Assumptions

**Users:**
- QR code familiarity 
- Prefer anonymous flows over accounts
- Peak usage first week of month

**Technical:**
- Current VaultLink well-documented via UI mockups
- MySQL→DynamoDB modeling viable
- Prism SQS/Lambda delivery compatible
- Stripe Connect supports required flows
- .NET 9 Lambda cold starts acceptable with provisioned concurrency

**Business:**
- Customers accept cloud SaaS migration
- Web responsive sufficient for MVP (no native apps)
- Synertec handles customer communication
- No major regulatory changes during development
---

## Risks & Critical Questions

### High Priority Risks

**1. DynamoDB Schema Design (HIGH)**
- NoSQL access patterns critical and hard to change post-launch
- **Mitigation:** Heavy discovery investment, prototype queries, plan GSIs upfront

**2. Prism Integration Unknown (HIGH)**
- Limited info on message format, auth, capabilities
- **Mitigation:** Prioritize Prism questions in discovery, flexible adapter design

**3. Multi-Tenant Data Isolation (HIGH)**
- Security breach exposing customer data catastrophic
- **Mitigation:** Database-level tenant isolation, row-level security, penetration testing

**4. Scope Creep (HIGH)**
- "Feature parity" may expand with undocumented features
- **Mitigation:** Freeze scope post-discovery, formal change process, defer to Phase 2

**5. Lambda Cold Starts (MEDIUM)**
- .NET cold starts may violate <500ms requirement
- **Mitigation:** Provisioned concurrency for critical paths, Beanstalk fallback

**6. Customer Migration Resistance (MEDIUM)**
- On-prem customers may resist cloud migration
- **Mitigation:** Clear communication, emphasize AWS/Stripe security, pilot program


### Critical Discovery Questions

**Current System Understanding (5 questions):**
1. What are the exact database sizes for the MySQL cluster? (Action: Harry to confirm)
2. What is the current monthly Azure infrastructure cost breakdown?
3. What are peak traffic patterns beyond month-start 50K requests?
4. Which customers currently use customization vs. default branding?
5. What is the breakdown of traffic by application (Statements vs. Invoices vs. Credit Control)?

**Architecture (7 questions):**
1. DynamoDB access patterns? (read vs write, query patterns, relationships)
2. Single-table or multi-table design?
3. Expected Lambda concurrency? (provisioned concurrency needs)
4. VPC for Lambda or AWS managed?
5. S3 bucket structure for customer assets?
6. Angular SSR or pure SPA?
7. Auth flow: admin/staff vs. anonymous users?

**Prism Integration (8 questions):** 
1. Message format? (JSON, XML, CSV, custom - currently DYNData encoded)
2. Message schema/structure per event type?
3. Authentication method? (API keys, OAuth, certs, IAM)
4. Endpoint type? (REST, SOAP, file drop, DB insert)
5. Prism unavailable handling? (retry, SLA, escalation)
6. Acknowledgment/confirmation needed?
7. Message throughput? (per minute/hour)
8. Ordering requirements? (FIFO queue needed)
9. Who handles Prism-side changes? (Confirmed: Synertec)
10. How to handle free-format comments from Credit Control app?

**Payment Provider Integration (6 questions):**
1. **What is Stripe's specific role?** ✅ **ANSWERED:**
   - **Stripe Checkout** (not Elements)
   - Standard Connected Account model
   - Customers manage own accounts
   - Customers choose payment options during onboarding
   - Payment methods: Whatever Stripe Checkout supports
   - Can include payment plans in some circumstances
   - No PCI SAQ A required (Stripe handles compliance)
2. ~~Do all 4 providers need support?~~ ✅ **ANSWERED:** No, **Stripe Checkout only** for MVP
3. Which provider is most commonly used? ✅ **ANSWERED:** Stripe (only one in scope)
4. What is the current Synertec-hosted URL that HPPs return to?
5. Will this URL change or remain the same in AWS?
6. What data is returned from Stripe Checkout to confirm payment success/failure?
2. Platform fee structure? (%, fixed, combo)
3. Payment methods beyond cards?
4. Recurring payments/subscriptions exist?
5. Refund/dispute handling?
6. Webhook event strategy?

**User Experience (6 questions):**
1. Complete feature list from current VaultLink?
2. Most-used workflows?
3. Known UX pain points to fix?
4. Hidden features/workarounds users rely on?
5. End-user demographics? (age, tech proficiency, devices)
6. Accessibility requirements? (WCAG level, assistive tech) - ✅ **Best-practice only**

**Operations (6 questions):**
1. Branding customization level per customer? ✅ **ANSWERED:** Simple (logos and some text - talk to Ed)
2. Customer-specific business rules? (terms, aging, fees)
3. Configuration options per customer?
4. Update rollout strategy? (all-at-once, staged, opt-in)
5. CloudWatch metrics and alert thresholds?
6. Backup/DR RTO/RPO requirements? - **Include backup verification**

**Compliance (7 questions):**
1. ~~PCI SAQ A requirements?~~ ✅ **ANSWERED:** Not required - Stripe Checkout handles compliance
2. ISO27001 controls? ✅ **ANSWERED:** Amdaris supports Synertec's ISO27001 (not implementing, out of scope)
3. Data retention and deletion policies (GDPR)?
4. Audit logging requirements?
5. Geographic data residency? ✅ **ANSWERED:** Deploy in Europe for UK customers
6. Penetration testing? ✅ **ANSWERED:** Synertec responsibility (out of scope for Amdaris)
7. **Payment trace retention requirements?** 
   - X-Ray has 30-day hard limit (cannot be extended)
   - For 90+ day compliance: Export to S3 (~$0.06/month) or CloudWatch Logs
   - **Question:** Do financial regulations require >30 days payment audit trail?
   - **Action:** Confirm retention policy before finalizing export strategy

---

## Next Steps

### Immediate Actions (Week 1-2)

1. **Finalize Discovery Brief** - Review and approve with Synertec stakeholders

2. **Discovery Workshop** - 2-day stakeholder session to address critical questions

3. **Access Current System** - Amdaris team needs VaultLink access for functional analysis

4. **Prism Documentation** - Obtain API specs, message formats, auth details, sample data

5. **Stripe Financial Model** - Validate fee structure and account model with finance team

### Architecture Phase (Week 3-4)

**Owner:** Richard (Architect), Bojan (Principal Dev)

**Deliverables:**
- Architecture document (system design, data model, API specs)
- DynamoDB schema and access patterns
- Lambda function catalog
- Security and compliance mapping
- POCs: Lambda cold start optimization, DynamoDB validation, Stripe/Prism prototypes

### PRD Phase (Week 3-4, Parallel)

**Owner:** Product Manager, Harry (PO)

**Deliverables:**
- Product Requirements Document
- Detailed feature specs
- User stories with acceptance criteria
- UI/UX specifications

### Development Planning (Week 5)

**Deliverables:**
- Story point estimation
- Sprint planning and MVP timeline
- Team allocation
- Risk mitigation plan
- Development standards

**Outcome:** Committed delivery timeline, Sprint 0 ready to begin

### Continuous Activities

- Bi-weekly stakeholder demos
- Bi-weekly retrospectives
- Regular architecture reviews
- Ongoing user research
- Iterative security testing

---

## QA & Testing Approach

### Testing Boundaries

**Prism Integration Testing:**
- Synertec **cannot easily stand up a dedicated Prism** environment for Amdaris development
- **System boundaries:** File or queue-based inputs and outputs
- **Approach:**
  1. Amdaris defines integration tests with expected inputs and results
  2. Amdaris clearly documents test cases and expected outcomes
  3. Synertec QA team agrees on test criteria
  4. UAT organized by Synertec with Amdaris team support

**Deliverables:**
- Unit tests delivered as part of CI/CD setup (confirmed by Richard)
- Integration test suite with documented inputs/outputs
- E2E test automation where Prism not required
- UAT test plan for Synertec execution

### Testing Responsibilities

**Amdaris:**
- Unit tests (all functions/components)
- Integration tests (within AWS boundaries)
- E2E tests (VaultLink application flows) - **Using Playwright** (Synertec uses Playwright)
- Performance/load testing
- Security testing (**SAST with evidence** - DAST by Synertec)
- Test documentation and handover
- **Automated testing across all suggested platforms**
- **Manual testing on 1 mobile + 1 desktop platform**

**Synertec:**
- UAT organization and execution
- Prism integration validation
- Customer acceptance testing
- **Penetration testing** (Synertec handles - out of scope for Amdaris)
- **DAST** (Dynamic Application Security Testing - Synertec responsibility)

### Test Data Management

- Mock Prism message formats for development
- Synthetic test data for all 3 applications (Statements, Invoices, Credit Control)
- Customer configuration samples (with/without customization)
- Payment provider sandbox accounts (all 4 providers if required)

---

## Actions Tracking

### Open Actions from Tech Walkthrough Meeting

**Action Items:**

1. **Harry Pestelle**
   - [ ] Provide access to GitHub repo for "Concentrator" app (functionality review)
   - [ ] Confirm exact database sizes for MySQL cluster
   - Status: Pending

2. **Cheryl Penny**
   - [ ] Provide existing test cases as background/starting point for Amdaris
   - Status: Pending

3. **Richard Livingstone**
   - [x] **Confirm Stripe's specific role** ✅ **COMPLETED:**
     - Stripe Checkout (not Elements)
     - Standard Connected Account model
     - Customers manage own accounts, choose payment options during onboarding
     - No PCI SAQ A required
   - [ ] Confirm architecture assumptions and capacity metrics surfaced
   - [ ] Revisit/confirm costings based on scenario discussed
   - [ ] **Pursuing with Synertec:** Admin capability for customers to manage users (Cognito dashboard AWS-focused)
   - Status: Partial completion

4. **Amdaris (Ed)**
   - [ ] Supply list of supported mobile/other devices to be targeted
   - Status: Pending

5. **Synertec**
   - [ ] Supply requirements for analytics links to be included
   - Status: Pending

### Decisions Pending

- [x] ~~Single payment provider?~~ ✅ **DECIDED:** Stripe Checkout only (not all 4 providers)
- [x] ~~Accessibility requirements?~~ ✅ **DECIDED:** Best-practice only
- [ ] Analytics platform integration requirements - **To be defined when we know what's achievable**
- [x] ~~Penetration testing scope?~~ ✅ **DECIDED:** Synertec responsibility (out of scope for Amdaris)
- [ ] Device support matrix (mobile browsers, versions) - **Automated testing: all platforms, Manual: 1 mobile + 1 desktop**
- [ ] **Admin capability for customers to manage users?** (Richard pursuing with Synertec)
- [x] ~~DynamoDB validation~~ ✅ **VALIDATED:** NoSQL is appropriate (see `database-decision.md` for comprehensive analysis)
- [ ] **Customer branding details** (Talk to Ed about existing simple customization)

---

## Document Summary

This MVP-focused brief provides:

✅ **Business Context** - Problem, solution, stakeholders  
✅ **User Understanding** - Three segments with needs and pain points  
✅ **System Scope** - 3 applications in scope (Statements, Invoices, Credit Control)  
✅ **Current System Details** - Azure architecture, performance baselines, technical constraints  
✅ **Technical Approach** - AWS serverless, DynamoDB, SQS/Prism integration, payment providers  
✅ **MVP Scope** - Core features, out of scope, success metrics  
✅ **Risks** - Identified and mitigated  
✅ **Discovery** - 38+ critical questions to answer  
✅ **Timeline** - 20-week phased approach  
✅ **QA Approach** - Testing boundaries and responsibilities  
✅ **Actions Tracking** - Open items from tech walkthrough meeting

**Related Documents:**
- `aws-infrastructure-services.md` - Complete AWS services specification
- `epics.md` - 16 epics broken down by phase
- `branching-strategy.md` - Git workflow and CI/CD
- `testing-strategy.md` - Comprehensive testing approach
- `database-decision.md` - ✅ **NoSQL/DynamoDB validation and justification**
- `ui-framework-decision.md` - Angular Material recommendation
- `distributed-tracing.md` - X-Ray observability strategy
- `post-mvp-roadmap.md` - Phase 2+ features

**Key Takeaways for Kickoff:**

1. **This is Azure → AWS migration, not on-premises rebuild**
2. **Only 3 of 5 VaultLink apps in scope** (NHS apps excluded)
3. **Stripe Checkout only** (Standard Connected Account, customers manage own accounts)
4. **No PCI SAQ A required** (Stripe Checkout handles compliance)
5. **VaultLink reference structure** is 49-char base64 with specific URL format
6. **Prism testing requires special approach** (cannot stand up dedicated environment)
7. **Current performance baseline:** 264ms avg, 500ms max, 50K peak at month-start
8. **Customization is minimal** (logos and some text - talk to Ed)
9. **Refunds policy is customer responsibility** (not part of VaultLink application)
10. **Unit tests + Playwright E2E tests** are part of CI/CD deliverable
11. **Deploy in Europe** for UK customer data residency
12. **ISO27001 & penetration testing** are Synertec's responsibility (Amdaris supports, doesn't implement)
13. **SAST by Amdaris, DAST by Synertec**
14. **Distributed tracing critical** for payment flows (AWS X-Ray)
15. ~~**Validate NoSQL in tech design**~~ ✅ **VALIDATED:** DynamoDB appropriate (see `database-decision.md`)  
✅ **Success Criteria** - Goals, metrics, KPIs  
✅ **MVP Scope** - In/out-of-scope features with success criteria  
✅ **Technical Foundation** - Stack, performance, security, integrations  
✅ **Risk Awareness** - High-priority risks and 38 critical questions  
✅ **Next Steps** - Clear path from discovery to development

**For detailed post-MVP roadmap**, see `post-mvp-roadmap.md`

**For AWS infrastructure specifications**, see `aws-infrastructure-services.md`

---

**Document End**
