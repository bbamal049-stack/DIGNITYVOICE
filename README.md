# DignityVoice Infrastructure

AWS CDK infrastructure for the DignityVoice AI Voice Call System for Patient Medication Management.

## Overview

This CDK stack creates all required AWS infrastructure in the `ap-south-1` (Mumbai) region:

### Resources Created

1. **DynamoDB Table: `DignityVoice`**
   - Single table design with PK/SK structure
   - Three Global Secondary Indexes:
     - `GSI1-RiskGroup-Index`: Query patients by risk group for scheduled calls
     - `GSI2-Phone-Index`: Lookup patients by phone number (screen pop)
     - `GSI3-Doctor-Index`: Query all patients under a specific doctor
   - Point-in-time recovery enabled
   - AWS-managed encryption at rest
   - DynamoDB Streams enabled

2. **S3 Buckets**
   - `temp-voice-storage`: Temporary storage for patient voice recordings
     - Lifecycle policy: Auto-delete after 1 day (Flush Protocol)
     - Server-side encryption enabled
   - `patient-imports`: Storage for bulk patient CSV/Excel imports
     - Lifecycle policy: Archive to Glacier after 30 days
     - Versioning enabled

3. **Amazon Cognito User Pool**
   - Email-based authentication
   - Three user groups with role-based access:
     - `AdminGroup`: Full system access and staff management
     - `DoctorGroup`: Patient management and prescription creation
     - `NurseGroup`: Call handling and patient note access
   - Password policy enforced (8+ chars, mixed case, digits, symbols)
   - Optional MFA support

4. **Initial Data**
   - `METADATA#PATIENT_COUNTER` item initialized with `counter_value = 0`
   - Used for auto-generating sequential patient IDs (P-1, P-2, etc.)

## Requirements Validated

- **Requirement 11.1**: DynamoDB table with single table design
- **Requirement 11.2**: S3 buckets with lifecycle policies
- **Requirement 19.1**: METADATA#PATIENT_COUNTER initialization
- **Requirement 21.1**: Amazon Cognito User Pool in ap-south-1
- **Requirement 21.2**: Three Cognito groups (AdminGroup, DoctorGroup, NurseGroup)

## Prerequisites

- Node.js 18+ and npm
- AWS CLI configured with credentials
- AWS CDK CLI installed: `npm install -g aws-cdk`
- AWS account with permissions to create:
  - DynamoDB tables
  - S3 buckets
  - Cognito User Pools
  - IAM roles and policies

## Installation

```bash
# Install dependencies
npm install

# Bootstrap CDK (first time only)
cdk bootstrap aws://ACCOUNT-NUMBER/ap-south-1

# Synthesize CloudFormation template
npm run synth

# Deploy infrastructure
npm run deploy
```

## Deployment

### Deploy to AWS

```bash
# Deploy with confirmation prompts
cdk deploy

# Deploy without confirmation (CI/CD)
cdk deploy --require-approval never
```

### View Stack Outputs

After deployment, the stack outputs important resource identifiers:

```bash
# View outputs
aws cloudformation describe-stacks \
  --stack-name DignityVoiceInfrastructureStack \
  --region ap-south-1 \
  --query 'Stacks[0].Outputs'
```

Key outputs:
- `DynamoDBTableName`: DignityVoice
- `TempVoiceStorageBucketName`: temp-voice-storage
- `PatientImportsBucketName`: patient-imports
- `UserPoolId`: Cognito User Pool ID
- `UserPoolClientId`: Web client ID

## DynamoDB Schema

### Primary Key Structure

- **PK** (Partition Key): Entity type identifier
- **SK** (Sort Key): Entity-specific identifier

### Entity Patterns

| Entity | PK | SK |
|--------|----|----|
| Patient Profile | `PATIENT#{id}` | `PROFILE` |
| Call Log | `PATIENT#{id}` | `LOG#{timestamp}` |
| Prescription | `PATIENT#{id}` | `PRESCRIPTION#{date}` |
| Nurse Note | `PATIENT#{id}` | `NOTE#{timestamp}` |
| Escalation Alert | `HOSPITAL#{id}` | `ALERT#{timestamp}` |
| Metadata Counter | `METADATA` | `PATIENT_COUNTER` |

### Global Secondary Indexes

**GSI1-RiskGroup-Index**
- PK: `GSI1PK` = `RISK_GROUP#{group}`
- SK: `GSI1SK` = `NEXT_CALL#{timestamp}`
- Use case: Query patients by risk group for scheduled calls

**GSI2-Phone-Index**
- PK: `GSI2PK` = `PHONE#{phone_number}`
- SK: `GSI2SK` = `PROFILE`
- Use case: Lookup patient by phone number (screen pop)

**GSI3-Doctor-Index**
- PK: `GSI3PK` = `DOCTOR#{doctor_id}`
- SK: `GSI3SK` = `PATIENT#{patient_id}`
- Use case: Query all patients under a specific doctor

## S3 Bucket Policies

### temp-voice-storage

- **Purpose**: Temporary storage for patient voice recordings
- **Lifecycle**: Auto-delete after 1 day (Flush Protocol)
- **Encryption**: S3-managed (SSE-S3)
- **Public Access**: Blocked
- **CORS**: Enabled for web uploads

### patient-imports

- **Purpose**: Bulk patient CSV/Excel imports
- **Lifecycle**: Archive to Glacier after 30 days
- **Encryption**: S3-managed (SSE-S3)
- **Versioning**: Enabled
- **Public Access**: Blocked
- **CORS**: Enabled for web uploads

## Cognito User Groups

### AdminGroup (Precedence: 1)
- Full system access
- Staff management (create Doctor/Nurse accounts)
- View/edit all patient records
- System configuration

### DoctorGroup (Precedence: 2)
- Patient management (create, view, edit)
- Prescription creation
- View patients under their care only
- Risk group assignment

### NurseGroup (Precedence: 3)
- Call handling and escalation management
- Read-only access to medical records
- Write access to patient notes
- Screen pop functionality

## Security Features

1. **Encryption at Rest**
   - DynamoDB: AWS-managed encryption
   - S3: Server-side encryption (SSE-S3)

2. **Encryption in Transit**
   - All API calls use HTTPS
   - Cognito enforces secure connections

3. **Access Control**
   - IAM roles with least privilege
   - Cognito groups for role-based access
   - S3 buckets block all public access

4. **Data Protection**
   - DynamoDB point-in-time recovery
   - S3 versioning for imports
   - Lifecycle policies for data retention

5. **Privacy (Flush Protocol)**
   - Voice recordings auto-deleted after 1 day
   - No long-term storage of patient voice data

## Cost Optimization

- **DynamoDB**: On-demand billing (pay per request)
- **S3**: Lifecycle policies reduce storage costs
  - temp-voice-storage: 1-day retention
  - patient-imports: Glacier archival after 30 days
- **Cognito**: Free tier covers up to 50,000 MAUs

## Monitoring

CloudWatch metrics are automatically enabled for:
- DynamoDB read/write capacity
- S3 bucket operations
- Cognito authentication events

## Cleanup

To delete all resources:

```bash
# Destroy stack (will prompt for confirmation)
cdk destroy

# Note: Buckets and DynamoDB table have RETAIN policy
# Manual deletion required for data safety
```

## Next Steps

After infrastructure deployment:

1. **Create Admin User**
   ```bash
   aws cognito-idp admin-create-user \
     --user-pool-id <UserPoolId> \
     --username admin@hospital.com \
     --user-attributes Name=email,Value=admin@hospital.com Name=name,Value="Admin User" \
     --region ap-south-1
   
   aws cognito-idp admin-add-user-to-group \
     --user-pool-id <UserPoolId> \
     --username admin@hospital.com \
     --group-name AdminGroup \
     --region ap-south-1
   ```

2. **Deploy Lambda Functions** (Task 2)
   - Voice_Brain Lambda
   - Dialer_Agent Lambda
   - Flush_Protocol Lambda
   - Bulk_Import_Lambda
   - Lookup_Patient Lambda

3. **Configure Amazon Connect** (Task 3)
   - Claim Indian DID phone number
   - Create AI_Nurse_Flow contact flow
   - Configure call recording to temp-voice-storage

4. **Deploy Frontend Application** (Task 4)
   - Next.js web application
   - Configure Cognito authentication
   - Deploy to AWS Amplify or Vercel

## Troubleshooting

### Deployment Fails

```bash
# Check CDK version
cdk --version

# Update CDK
npm install -g aws-cdk@latest

# Clear CDK cache
rm -rf cdk.out
```

### Counter Not Initialized

```bash
# Manually create counter item
aws dynamodb put-item \
  --table-name DignityVoice \
  --item '{"PK":{"S":"METADATA"},"SK":{"S":"PATIENT_COUNTER"},"counter_value":{"N":"0"}}' \
  --region ap-south-1
```

### Bucket Name Conflicts

If bucket names are already taken, modify bucket names in the stack:
- Add a unique suffix (e.g., `temp-voice-storage-{account-id}`)
- Update Lambda functions to use new bucket names

## Support

For issues or questions:
- Review CloudFormation events in AWS Console
- Check CloudWatch Logs for custom resource execution
- Verify IAM permissions for deployment role

## License

MIT
