# DignityVoice Infrastructure Architecture

## Overview

This document describes the AWS infrastructure architecture for the DignityVoice AI Voice Call System deployed in the `ap-south-1` (Mumbai) region.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AWS Region: ap-south-1 (Mumbai)                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Amazon Cognito User Pool                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │   │
│  │  │  AdminGroup  │  │ DoctorGroup  │  │  NurseGroup  │          │   │
│  │  │ (Precedence  │  │ (Precedence  │  │ (Precedence  │          │   │
│  │  │      1)      │  │      2)      │  │      3)      │          │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │   │
│  │                                                                   │   │
│  │  Authentication & Authorization                                  │   │
│  │  - Email-based sign-in                                          │   │
│  │  - Role-based access control                                    │   │
│  │  - MFA support (optional)                                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                     │
│                                    │ JWT Tokens                          │
│                                    ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Frontend Application                        │   │
│  │                    (Next.js - Not in this stack)                │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │   │
│  │  │    Admin     │  │    Doctor    │  │    Nurse     │          │   │
│  │  │  Dashboard   │  │  Dashboard   │  │  Dashboard   │          │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                     │
│                                    │ API Calls                           │
│                                    ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    AWS Lambda Functions                          │   │
│  │                    (Not in this stack)                           │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │   │
│  │  │ Voice_Brain  │  │Dialer_Agent  │  │Flush_Protocol│          │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │   │
│  │  ┌──────────────┐  ┌──────────────┐                            │   │
│  │  │Bulk_Import   │  │Lookup_Patient│                            │   │
│  │  └──────────────┘  └──────────────┘                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                     │
│                                    │ Read/Write                          │
│                                    ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Amazon DynamoDB                               │   │
│  │                    Table: DignityVoice                           │   │
│  │                                                                   │   │
│  │  Primary Key: PK (Partition), SK (Sort)                         │   │
│  │                                                                   │   │
│  │  ┌───────────────────────────────────────────────────────────┐  │   │
│  │  │ Global Secondary Indexes (GSI)                            │  │   │
│  │  │                                                             │  │   │
│  │  │  GSI1-RiskGroup-Index                                      │  │   │
│  │  │  - PK: GSI1PK (RISK_GROUP#{group})                        │  │   │
│  │  │  - SK: GSI1SK (NEXT_CALL#{timestamp})                     │  │   │
│  │  │  - Use: Query patients by risk group for scheduled calls  │  │   │
│  │  │                                                             │  │   │
│  │  │  GSI2-Phone-Index                                          │  │   │
│  │  │  - PK: GSI2PK (PHONE#{phone_number})                      │  │   │
│  │  │  - SK: GSI2SK (PROFILE)                                   │  │   │
│  │  │  - Use: Lookup patient by phone (screen pop)              │  │   │
│  │  │                                                             │  │   │
│  │  │  GSI3-Doctor-Index                                         │  │   │
│  │  │  - PK: GSI3PK (DOCTOR#{doctor_id})                        │  │   │
│  │  │  - SK: GSI3SK (PATIENT#{patient_id})                      │  │   │
│  │  │  - Use: Query all patients under a doctor                 │  │   │
│  │  └───────────────────────────────────────────────────────────┘  │   │
│  │                                                                   │   │
│  │  Features:                                                        │   │
│  │  - On-demand billing                                             │   │
│  │  - Point-in-time recovery                                        │   │
│  │  - AWS-managed encryption                                        │   │
│  │  - DynamoDB Streams enabled                                      │   │
│  │                                                                   │   │
│  │  Entities Stored:                                                │   │
│  │  - Patient Profiles                                              │   │
│  │  - Call Logs (Dual_Log format)                                  │   │
│  │  - Prescriptions                                                 │   │
│  │  - Nurse Notes                                                   │   │
│  │  - Escalation Alerts                                             │   │
│  │  - Doctor/Nurse Profiles                                         │   │
│  │  - METADATA#PATIENT_COUNTER (initialized to 0)                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        Amazon S3 Buckets                         │   │
│  │                                                                   │   │
│  │  ┌───────────────────────────────────────────────────────────┐  │   │
│  │  │ temp-voice-storage                                         │  │   │
│  │  │                                                             │  │   │
│  │  │ Purpose: Temporary storage for patient voice recordings   │  │   │
│  │  │                                                             │  │   │
│  │  │ Features:                                                  │  │   │
│  │  │ - S3-managed encryption (SSE-S3)                          │  │   │
│  │  │ - Block all public access                                 │  │   │
│  │  │ - Lifecycle: Auto-delete after 1 day (Flush Protocol)    │  │   │
│  │  │ - CORS enabled for web uploads                            │  │   │
│  │  │                                                             │  │   │
│  │  │ Used by:                                                   │  │   │
│  │  │ - Amazon Connect (call recordings)                        │  │   │
│  │  │ - Flush_Protocol Lambda (read & delete)                   │  │   │
│  │  └───────────────────────────────────────────────────────────┘  │   │
│  │                                                                   │   │
│  │  ┌───────────────────────────────────────────────────────────┐  │   │
│  │  │ patient-imports                                            │  │   │
│  │  │                                                             │  │   │
│  │  │ Purpose: Bulk patient CSV/Excel file uploads              │  │   │
│  │  │                                                             │  │   │
│  │  │ Features:                                                  │  │   │
│  │  │ - S3-managed encryption (SSE-S3)                          │  │   │
│  │  │ - Block all public access                                 │  │   │
│  │  │ - Versioning enabled                                      │  │   │
│  │  │ - Lifecycle: Archive to Glacier after 30 days            │  │   │
│  │  │ - CORS enabled for web uploads                            │  │   │
│  │  │                                                             │  │   │
│  │  │ Used by:                                                   │  │   │
│  │  │ - Admin Dashboard (file uploads)                          │  │   │
│  │  │ - Bulk_Import_Lambda (triggered on upload)                │  │   │
│  │  └───────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## Data Flow Diagrams

### 1. Patient Creation Flow (Auto-Generated ID)

```
┌──────────────┐
│   Doctor     │
│  Dashboard   │
└──────┬───────┘
       │ 1. Create patient (no custom ID)
       ▼
┌──────────────┐
│   Lambda     │
│  Function    │
└──────┬───────┘
       │ 2. Atomic update: ADD counter_value :1
       ▼
┌──────────────────────────────────────┐
│         DynamoDB                     │
│  PK: METADATA                        │
│  SK: PATIENT_COUNTER                 │
│  counter_value: N → N+1              │
└──────┬───────────────────────────────┘
       │ 3. Return new counter value (e.g., 1043)
       ▼
┌──────────────┐
│   Lambda     │
│  Assigns:    │
│  P-1043      │
└──────┬───────┘
       │ 4. Create patient record
       ▼
┌──────────────────────────────────────┐
│         DynamoDB                     │
│  PK: PATIENT#P-1043                  │
│  SK: PROFILE                         │
│  patient_id: P-1043                  │
│  name: "Ramesh Kumar"                │
│  ...                                 │
└──────────────────────────────────────┘
```

### 2. Bulk Import Flow

```
┌──────────────┐
│    Admin     │
│  Dashboard   │
└──────┬───────┘
       │ 1. Upload CSV file
       ▼
┌──────────────────────────────────────┐
│         S3: patient-imports          │
│  Key: imports/patients_jan_2024.csv │
└──────┬───────────────────────────────┘
       │ 2. S3 Event Notification
       ▼
┌──────────────────────────────────────┐
│    Bulk_Import_Lambda                │
│                                      │
│  For each row:                       │
│  - If Patient_ID exists: use it      │
│  - If empty: auto-generate via       │
│    METADATA#PATIENT_COUNTER          │
│  - Validate data                     │
│  - Write to DynamoDB                 │
└──────┬───────────────────────────────┘
       │ 3. Write patient records
       ▼
┌──────────────────────────────────────┐
│         DynamoDB                     │
│  Multiple patient records created    │
└──────┬───────────────────────────────┘
       │ 4. Send email report
       ▼
┌──────────────┐
│  Amazon SES  │
│  (Email)     │
└──────────────┘
```

### 3. Screen Pop Flow (Nurse Dashboard)

```
┌──────────────┐
│   Patient    │
│   Calls      │
└──────┬───────┘
       │ 1. Incoming call with Caller ID
       ▼
┌──────────────────────────────────────┐
│      Amazon Connect                  │
│  Captures ANI: +919876543210         │
└──────┬───────────────────────────────┘
       │ 2. Invoke Lookup_Patient Lambda
       ▼
┌──────────────────────────────────────┐
│    Lookup_Patient Lambda             │
│  Query GSI2-Phone-Index              │
└──────┬───────────────────────────────┘
       │ 3. Query by phone
       ▼
┌──────────────────────────────────────┐
│         DynamoDB                     │
│  GSI2PK: PHONE#+919876543210         │
│  GSI2SK: PROFILE                     │
│  Returns: Patient#P-1043             │
└──────┬───────────────────────────────┘
       │ 4. Return patient info
       ▼
┌──────────────────────────────────────┐
│    Nurse Dashboard                   │
│  Auto-navigate to:                   │
│  /nurse/patient/P-1043               │
│  Display patient history             │
└──────────────────────────────────────┘
```

### 4. Flush Protocol Flow

```
┌──────────────┐
│   Patient    │
│   Call Ends  │
└──────┬───────┘
       │ 1. Call recording saved
       ▼
┌──────────────────────────────────────┐
│    S3: temp-voice-storage            │
│  Key: recordings/call-12345.mp3      │
│  Key: transcripts/call-12345.txt     │
└──────┬───────────────────────────────┘
       │ 2. S3 Event Notification
       ▼
┌──────────────────────────────────────┐
│    Flush_Protocol Lambda             │
│                                      │
│  1. Download audio & transcript      │
│  2. Send to Bedrock for analysis     │
│  3. Generate Dual_Log                │
│  4. Save to DynamoDB                 │
│  5. DELETE audio & transcript        │
└──────┬───────────────────────────────┘
       │ 3. Write Dual_Log
       ▼
┌──────────────────────────────────────┐
│         DynamoDB                     │
│  PK: PATIENT#P-1043                  │
│  SK: LOG#2024-01-15T08:30:00Z        │
│  narrative_summary: "..."            │
│  clinical_metrics: {...}             │
└──────────────────────────────────────┘
       │ 4. Hard delete from S3
       ▼
┌──────────────────────────────────────┐
│    S3: temp-voice-storage            │
│  Files deleted (Zero-Retention)      │
└──────────────────────────────────────┘
```

## Security Architecture

### Authentication & Authorization

```
┌──────────────┐
│     User     │
└──────┬───────┘
       │ 1. Login with email/password
       ▼
┌──────────────────────────────────────┐
│      Amazon Cognito                  │
│  - Verify credentials                │
│  - Check group membership            │
│  - Generate JWT tokens               │
└──────┬───────────────────────────────┘
       │ 2. JWT with group claims
       │    cognito:groups: ["DoctorGroup"]
       ▼
┌──────────────────────────────────────┐
│      Frontend Application            │
│  - Store tokens securely             │
│  - Include in API requests           │
│  - Route guards check groups         │
└──────┬───────────────────────────────┘
       │ 3. API call with Authorization header
       ▼
┌──────────────────────────────────────┐
│      Lambda Function                 │
│  - Verify JWT signature              │
│  - Extract group claims              │
│  - Enforce access control            │
└──────────────────────────────────────┘
```

### Data Encryption

**At Rest:**
- DynamoDB: AWS-managed encryption (KMS)
- S3: Server-side encryption (SSE-S3)
- Cognito: Encrypted by default

**In Transit:**
- All API calls use HTTPS/TLS 1.2+
- Cognito enforces secure connections
- S3 enforces SSL for uploads

### Network Security

- S3 buckets: Block all public access
- DynamoDB: VPC endpoints (recommended for production)
- Cognito: Secure token exchange
- Lambda: VPC integration (optional)

## Scalability

### DynamoDB
- **Billing Mode**: On-demand (auto-scales)
- **Read/Write Capacity**: Unlimited
- **Partition Strategy**: Single table design with efficient access patterns
- **GSI Indexes**: Optimized for common query patterns

### S3
- **Scalability**: Unlimited storage
- **Request Rate**: 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD per second per prefix
- **Lifecycle Policies**: Automatic data management

### Cognito
- **User Pool**: Scales automatically
- **Token Caching**: Reduces authentication overhead
- **Group Management**: Efficient role-based access

## High Availability

### DynamoDB
- **Multi-AZ**: Automatic replication across 3 AZs
- **Point-in-Time Recovery**: Enabled
- **Backup**: Continuous backups for 35 days

### S3
- **Durability**: 99.999999999% (11 9's)
- **Availability**: 99.99%
- **Versioning**: Enabled for patient-imports

### Cognito
- **Multi-AZ**: Automatic replication
- **SLA**: 99.9% uptime

## Monitoring & Observability

### CloudWatch Metrics (Automatic)

**DynamoDB:**
- ConsumedReadCapacityUnits
- ConsumedWriteCapacityUnits
- UserErrors
- SystemErrors

**S3:**
- NumberOfObjects
- BucketSizeBytes
- AllRequests
- 4xxErrors, 5xxErrors

**Cognito:**
- SignInSuccesses
- SignInThrottles
- TokenRefreshSuccesses

### Recommended Alarms

1. **DynamoDB Throttling**
   - Metric: UserErrors
   - Threshold: > 10 in 5 minutes

2. **S3 Upload Failures**
   - Metric: 4xxErrors
   - Threshold: > 50 in 5 minutes

3. **Cognito Authentication Failures**
   - Metric: SignInThrottles
   - Threshold: > 100 in 5 minutes

## Cost Optimization

### DynamoDB
- **On-Demand Billing**: Pay only for what you use
- **No idle capacity costs**
- **Estimated**: $1.25 per million write requests, $0.25 per million read requests

### S3
- **Lifecycle Policies**: Reduce storage costs
  - temp-voice-storage: 1-day retention
  - patient-imports: Glacier after 30 days
- **Estimated**: $0.023 per GB/month (Standard), $0.004 per GB/month (Glacier)

### Cognito
- **Free Tier**: 50,000 MAUs
- **Estimated**: Free for most use cases

### Total Estimated Monthly Cost
- **1000 patients, 3000 calls/month**: $40-80/month
- **10000 patients, 30000 calls/month**: $200-400/month

## Disaster Recovery

### Backup Strategy

**DynamoDB:**
- Point-in-time recovery: 35 days
- On-demand backups: Manual snapshots
- Cross-region replication: Optional

**S3:**
- Versioning: Enabled for patient-imports
- Cross-region replication: Optional
- Lifecycle policies: Automatic archival

**Cognito:**
- User export: Manual via CLI
- Backup user pool configuration: Infrastructure as Code (CDK)

### Recovery Procedures

**RTO (Recovery Time Objective):** < 4 hours
**RPO (Recovery Point Objective):** < 1 hour

1. **DynamoDB Table Loss**
   - Restore from point-in-time recovery
   - Estimated time: 30-60 minutes

2. **S3 Bucket Loss**
   - Restore from versioning (patient-imports)
   - temp-voice-storage: No recovery needed (1-day retention)

3. **Cognito User Pool Loss**
   - Redeploy from CDK stack
   - Import users from backup
   - Estimated time: 2-3 hours

## Compliance & Privacy

### HIPAA Compliance (Recommended)
- Enable AWS Business Associate Addendum (BAA)
- Use AWS KMS for encryption
- Enable CloudTrail for audit logging
- Implement VPC endpoints for private access

### Data Retention
- **Voice Recordings**: 1 day (Flush Protocol)
- **Call Logs**: Indefinite (DynamoDB)
- **Patient Imports**: 30 days (then Glacier)

### Privacy Features
- Zero-Retention voice data
- Encryption at rest and in transit
- Role-based access control
- Audit logging via CloudTrail

## Future Enhancements

1. **Multi-Region Deployment**
   - DynamoDB Global Tables
   - S3 Cross-Region Replication
   - Route 53 for failover

2. **Enhanced Security**
   - AWS WAF for API protection
   - GuardDuty for threat detection
   - AWS Config for compliance monitoring

3. **Performance Optimization**
   - DynamoDB Accelerator (DAX) for caching
   - CloudFront for frontend distribution
   - Lambda@Edge for edge computing

4. **Advanced Monitoring**
   - X-Ray for distributed tracing
   - Custom CloudWatch dashboards
   - Automated alerting with SNS

## References

- [AWS CDK Documentation](https://docs.aws.amazon.com/cdk/)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [Cognito Security Best Practices](https://docs.aws.amazon.com/cognito/latest/developerguide/managing-security.html)
