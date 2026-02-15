# Requirements Document: DignityVoice - AI Voice Call System for Patient Medication Management

## Introduction

This document specifies the requirements for "DignityVoice," an AI-powered voice call system that helps patients (especially elderly and non-tech-savvy individuals in India) manage their medications through automated voice calls, while providing healthcare providers with comprehensive monitoring data and escalation workflows. The system is built on AWS infrastructure in the ap-south-1 (Mumbai) region with a "Zero-Retention" privacy model where patient voice data is flushed immediately after summarization.

## Glossary

- **Seva**: The AI conversational assistant name used during patient calls
- **AI_Call_System**: The automated voice call system that interacts with patients via Amazon Connect over PSTN
- **Amazon_Connect**: AWS service providing PSTN telephony with DID (Direct Inward Dialing) phone numbers
- **Contact_Flow**: Amazon Connect workflow that orchestrates call logic (AI_Nurse_Flow)
- **Voice_Brain**: Lambda function that processes patient input and generates AI responses via Bedrock
- **Dialer_Agent**: Lambda function that schedules and initiates outbound calls to patients
- **Patient**: Individual receiving medication reminders and monitoring calls via standard phone (GSM/Landline)
- **Doctor**: Healthcare provider who prescribes medications and monitors patients
- **Nurse**: Healthcare staff who handles escalated calls and urgent patient needs
- **Hospital_Admin**: System administrator managing the overall platform
- **Risk_Engine**: Custom Python logic that determines patient risk groups and call schedules
- **Risk_Group**: Classification of patients (Group 1 High Risk, Group 2A/2B Medium Risk, Group 3 Low Risk)
- **Escalation**: Process of transferring patient interaction from AI to human nurse
- **Patient_Profile**: DynamoDB record containing patient information, risk group, and adherence streak
- **Adherence_Streak**: Count of consecutive days with perfect medication compliance
- **Dual_Log**: Output format containing both narrative summary and structured clinical data
- **Narrative_Summary**: Human-readable 2-3 sentence summary of the call
- **Clinical_Metrics**: Structured JSON data including medication status, symptoms, pain scale, mood
- **Flush_Protocol**: Security process that deletes patient voice data after summarization
- **Regional_Language**: Support for Indian English (en-IN) and Hindi (hi-IN)
- **Dashboard**: Next.js web interface for doctors and hospital admins to view and manage patient data
- **DID**: Direct Inward Dialing - a phone number that directly reaches the Amazon Connect instance
- **PSTN**: Public Switched Telephone Network - standard telephone infrastructure (no internet required)
- **Bulk_Import_Lambda**: Lambda function that processes CSV/Excel files for bulk patient import
- **Lookup_Patient**: Lambda function that queries DynamoDB to find patient information by phone number
- **CCP**: Contact Control Panel - Amazon Connect's embedded softphone interface for call control
- **ANI**: Automatic Number Identification - Caller ID information provided by the telephone network
- **Screen_Pop**: Automatic display of patient information when an incoming call is detected
- **METADATA#PATIENT_COUNTER**: DynamoDB item that tracks the last assigned sequential patient ID
- **Admin_Dashboard**: Web interface at /admin for hospital administrators to manage system and staff
- **Doctor_Dashboard**: Web interface at /doctor for doctors to manage patients and prescriptions
- **Nurse_Dashboard**: Web interface at /nurse for nurses to handle calls and view patient information
- **Upload_Center**: Admin Dashboard component for drag-and-drop file uploads
- **Staff_Manager**: Admin Dashboard component for creating Doctor and Nurse accounts
- **Prescription_Writer**: Doctor Dashboard component for creating and managing medication prescriptions
- **Live_Patient_View**: Nurse Dashboard component displaying patient information during calls
- **AdminGroup**: Cognito user group with full system access and staff management permissions
- **DoctorGroup**: Cognito user group with access to patient management and prescription creation
- **NurseGroup**: Cognito user group with read-only medical access and write access to notes

## Requirements

### Requirement 1: Medication Reminder Calls

**User Story:** As a patient, I want to receive automated voice calls reminding me about my medications, so that I don't forget to take them at the correct times.

#### Acceptance Criteria

1. WHEN a scheduled medication time arrives, THE AI_Call_System SHALL initiate a voice call to the patient
2. WHEN the patient answers, THE AI_Call_System SHALL ask in natural language if they have taken their medication
3. WHEN the patient confirms medication intake, THE AI_Call_System SHALL log the confirmation and end the call
4. WHEN the patient indicates they have not taken medication, THE AI_Call_System SHALL schedule a follow-up call in 30 minutes
5. WHEN the second reminder call is also missed or declined, THE AI_Call_System SHALL escalate to a human nurse via the hospital dashboard
6. WHEN the patient's voice is unclear, THE AI_Call_System SHALL activate the IVR_System for structured input
7. WHERE regional language support is configured, THE AI_Call_System SHALL communicate in the patient's preferred language

### Requirement 2: Patient-Initiated Escalation

**User Story:** As a patient, I want to speak with a human nurse at any time during an AI call, so that I can get help when I feel uncomfortable or need human assistance.

#### Acceptance Criteria

1. WHEN a patient requests human assistance during any call, THE AI_Call_System SHALL immediately escalate to a nurse
2. WHEN escalation occurs, THE AI_Call_System SHALL transfer the call with full context to the nurse dashboard
3. WHEN the nurse receives an escalated call, THE Dashboard SHALL display the patient's complete file and current call context

### Requirement 3: Daily Summary Logging Calls

**User Story:** As a doctor, I want the AI system to conduct daily check-in calls with patients to gather information about their wellbeing, so that I can monitor their condition between appointments.

#### Acceptance Criteria

1. WHEN a scheduled daily summary time arrives, THE AI_Call_System SHALL initiate a check-in call to the patient
2. WHEN the patient answers, THE AI_Call_System SHALL ask about their day, pain levels, troubles, and concerns in natural language
3. WHEN asking about medication adherence, THE AI_Call_System SHALL use simple descriptive terms (e.g., "the round yellow tablet")
4. WHEN the conversation completes, THE AI_Call_System SHALL generate a summary and store it in the Patient_File
5. WHERE regional language support is configured, THE AI_Call_System SHALL conduct the conversation in the patient's preferred language
6. WHEN the patient's voice is unclear, THE AI_Call_System SHALL activate the IVR_System
7. WHEN the patient needs assistance, THE AI_Call_System SHALL escalate to a nurse

### Requirement 4: Patient Question Answering

**User Story:** As a patient, I want to ask the AI system questions about my medications, so that I can understand my treatment better without waiting for a doctor appointment.

#### Acceptance Criteria

1. WHEN a patient asks a question about their medication, THE AI_Call_System SHALL retrieve information only from doctor-approved and hospital-approved documents
2. WHEN answering questions, THE AI_Call_System SHALL NOT use third-party information sources
3. WHEN the AI_Call_System cannot answer from approved sources, THE AI_Call_System SHALL inform the patient and offer to escalate to a nurse

### Requirement 5: AI Memory Management and Zero-Retention Privacy

**User Story:** As a hospital administrator, I want the AI system to process patient data without retaining voice recordings, so that we maintain maximum patient privacy through the "Flush Protocol."

#### Acceptance Criteria

1. WHEN the AI_Call_System initiates a call, THE AI_Call_System SHALL access the Patient_Profile only for that specific call
2. WHEN the call completes, THE AI_Call_System SHALL generate a Dual_Log containing Narrative_Summary and Clinical_Metrics
3. WHEN the Dual_Log is stored in DynamoDB, THE Flush_Protocol SHALL delete all audio recordings and transcripts from S3 storage
4. THE Flush_Protocol SHALL execute within 5 minutes of call completion
5. THE AI_Call_System SHALL NOT retain patient voice data beyond the time required for summarization
6. WHEN moving to the next patient call, THE AI_Call_System SHALL have no memory of previous patient interactions

### Requirement 6: High Risk Patient Management (Group 1)

**User Story:** As a doctor, I want high-risk patients (Group 1 - "ICU at Home") to receive 3 calls daily with immediate escalation on any missed dose, so that critical medication adherence is maintained for patients where missed doses could be fatal.

#### Acceptance Criteria

1. WHEN a patient is categorized as Risk_Group 1, THE Risk_Engine SHALL schedule 3 calls daily (morning, afternoon, evening)
2. WHEN a Risk_Group 1 call is made, THE AI_Call_System SHALL ask specifically about that time slot's medication dose
3. WHEN a Risk_Group 1 patient misses a call, THE Risk_Engine SHALL trigger an immediate escalation alert
4. WHEN a Risk_Group 1 patient responds "No" to medication intake, THE Risk_Engine SHALL trigger an immediate escalation alert
5. THE Risk_Engine SHALL maintain Risk_Group 1 status until a doctor manually changes the patient's risk group

### Requirement 7: Medium Risk Patient Management - Group 2A (Probation)

**User Story:** As a doctor, I want Medium Risk Group 2A patients to receive the same intensive monitoring as Group 1 patients, so that patients who need behavioral reinforcement maintain adherence.

#### Acceptance Criteria

1. WHEN a patient is categorized as Risk_Group 2A, THE Risk_Engine SHALL schedule 3 calls daily (morning, afternoon, evening)
2. WHEN a Risk_Group 2A patient maintains perfect adherence for 14 consecutive days, THE Risk_Engine SHALL automatically promote them to Risk_Group 2B
3. WHEN a Risk_Group 2A patient misses one medication, THE Risk_Engine SHALL reset their Adherence_Streak to 0
4. WHEN a Risk_Group 2A patient's voice sentiment is detected as "DISTRESSED", THE Risk_Engine SHALL reset their Adherence_Streak to 0

### Requirement 8: Medium Risk Patient Management - Group 2B (Trusted/Reward)

**User Story:** As a doctor, I want Medium Risk Group 2B patients to receive only one evening summary call, so that patients with proven adherence history receive less intrusive monitoring as a reward for good behavior.

#### Acceptance Criteria

1. WHEN a patient is categorized as Risk_Group 2B, THE Risk_Engine SHALL schedule 1 call daily (evening summary)
2. WHEN a Risk_Group 2B evening call is made, THE AI_Call_System SHALL ask "Did you take ALL your meds today?"
3. WHEN a Risk_Group 2B patient misses one medication, THE Risk_Engine SHALL automatically demote them to Risk_Group 2A
4. WHEN a Risk_Group 2B patient's voice sentiment is detected as "DISTRESSED", THE Risk_Engine SHALL automatically demote them to Risk_Group 2A
5. WHEN a Risk_Group 2B patient is demoted, THE Risk_Engine SHALL reset their Adherence_Streak to 0

### Requirement 9: Doctor Dashboard Access

**User Story:** As a doctor, I want to access all AI call logs and patient monitoring data through a dashboard with visual risk indicators, so that I can make better-informed diagnoses and treatment decisions.

#### Acceptance Criteria

1. WHEN a doctor logs into the Dashboard, THE Dashboard SHALL display all patients under their care in a RiskCardGrid layout
2. WHEN displaying Risk_Group 1 patients, THE Dashboard SHALL show a red border (border-red-600) with a 🚨 icon
3. WHEN displaying Risk_Group 2A patients, THE Dashboard SHALL show an orange border (border-orange-400) with a ⚠️ icon
4. WHEN displaying Risk_Group 2B patients, THE Dashboard SHALL show a yellow border (border-yellow-400)
5. WHEN displaying Risk_Group 2 patients, THE Dashboard SHALL show a progress bar indicating Adherence_Streak (X/14 days)
6. WHEN a doctor selects a patient, THE Dashboard SHALL display the PatientHistoryLog with Narrative_Summary on the left and Clinical_Metrics table on the right
7. WHEN a doctor adds a patient to the system, THE Dashboard SHALL allow assignment to a Risk_Group
8. WHEN a doctor changes a patient's Risk_Group, THE Dashboard SHALL only allow changes for patients under their care

### Requirement 10: Hospital Admin Dashboard and Nurse Escalation Management

**User Story:** As a hospital administrator, I want to manage the entire AI system and provide nurses with a real-time escalation feed, so that urgent patient needs are addressed immediately.

#### Acceptance Criteria

1. WHEN a Hospital_Admin logs into the Dashboard, THE Dashboard SHALL display all Patient_Profiles across all doctors
2. WHEN the Hospital_Admin accesses system configurations, THE Dashboard SHALL allow updates to Risk_Engine parameters
3. WHEN a call is escalated, THE Dashboard SHALL create an escalation record in DynamoDB with status "OPEN"
4. WHEN a nurse views the EscalationFeed, THE Dashboard SHALL display escalations in real-time using AWS AppSync subscriptions
5. WHEN an urgent symptom is detected (chest pain, dizziness), THE Dashboard SHALL display the escalation with bg-red-600 text-white animate-pulse styling
6. WHEN a nurse views an escalated call, THE Dashboard SHALL display complete Patient_Profile and prescription details
7. WHEN a nurse acknowledges an escalation, THE Dashboard SHALL update the escalation status to "RESOLVED"
8. WHEN a nurse determines immediate medical care is needed, THE Dashboard SHALL provide escalation options to doctors
9. WHEN the Hospital_Admin changes a patient's Risk_Group, THE Dashboard SHALL allow changes for any patient in the system

### Requirement 11: Data Storage and Security with Flush Protocol

**User Story:** As a hospital administrator, I want patient data stored securely in AWS DynamoDB with voice recordings automatically deleted after processing, so that we maintain maximum patient privacy and comply with healthcare regulations.

#### Acceptance Criteria

1. THE AI_Call_System SHALL store all Patient_Profiles in DynamoDB using single table design
2. THE AI_Call_System SHALL store audio recordings temporarily in S3 bucket "temp-voice-storage"
3. THE S3 bucket SHALL have a lifecycle rule configured to expire objects after 1 day
4. WHEN the Flush_Protocol executes, THE Flush_Protocol SHALL read audio and transcript from S3
5. WHEN the Flush_Protocol processes data, THE Flush_Protocol SHALL send it to Amazon Bedrock for Dual_Log extraction
6. WHEN the Dual_Log is generated, THE Flush_Protocol SHALL save the JSON to DynamoDB
7. WHEN the Dual_Log is saved, THE Flush_Protocol SHALL execute hard delete (s3.delete_object) on the audio and transcript files
8. THE Flush_Protocol SHALL ensure zero patient voice data remains on the server after processing

### Requirement 12: Accessibility for Indian Patients via PSTN

**User Story:** As an elderly patient in India with limited technology skills, I want to receive calls on my standard phone (GSM/Landline) without needing internet or a smartphone, so that I can benefit from medication reminders despite my limited tech literacy.

#### Acceptance Criteria

1. THE AI_Call_System SHALL operate entirely through PSTN voice calls using Amazon Connect
2. THE AI_Call_System SHALL use a claimed Indian DID (Direct Inward Dialing) phone number (+91...)
3. THE AI_Call_System SHALL NOT require internet connectivity or smartphone applications on the patient side
4. THE AI_Call_System SHALL use Amazon Connect's built-in speech-to-text with language codes en-IN (Indian English) or hi-IN (Hindi)
5. THE AI_Call_System SHALL use Amazon Polly for text-to-speech with voice "Kajal" (bilingual) or "Aditi"
6. THE AI_Call_System SHALL use conversational and empathetic voice style
7. THE AI_Call_System SHALL keep spoken responses under 10 words for low latency on phone calls
8. THE AI_Call_System SHALL use Indian English mannerisms (e.g., "Ji", "Don't worry")
9. THE AI_Call_System SHALL provide clear, simple instructions during all interactions

### Requirement 13: Dual-Log Call Recording System

**User Story:** As a doctor, I want complete logs of all AI-patient interactions stored in both narrative and structured formats, so that I can review both the story and the clinical data of patient interactions.

#### Acceptance Criteria

1. WHEN any call completes, THE AI_Call_System SHALL generate a Dual_Log with timestamp and call type
2. WHEN generating a Dual_Log, THE AI_Call_System SHALL create a Narrative_Summary (2-3 sentence human-readable summary)
3. WHEN generating a Dual_Log, THE AI_Call_System SHALL create Clinical_Metrics JSON containing:
   - medication_status (Enum: "Fully Adherent", "Partial", "Missed", "Unknown")
   - reported_symptoms (List of strings)
   - pain_scale (Integer 0-10)
   - mood_sentiment (Enum: "Positive", "Neutral", "Distressed")
   - escalation_flag (Boolean)
4. WHEN a Dual_Log is created, THE AI_Call_System SHALL store it in DynamoDB with PK=PATIENT#{id} and SK=LOG#{date}
5. WHEN a doctor views a patient's history, THE Dashboard SHALL display all Dual_Logs in chronological order
6. THE Dashboard SHALL display Narrative_Summary in the left column and Clinical_Metrics table in the right column

### Requirement 14: Nurse Escalation Workflow with Real-Time Alerts

**User Story:** As a nurse, I want to receive escalated calls with full patient context in real-time, so that I can provide immediate assistance for urgent situations.

#### Acceptance Criteria

1. WHEN a call is escalated, THE Dashboard SHALL create an escalation record in DynamoDB with PK=HOSPITAL#{id} and SK=ALERT#{timestamp}
2. WHEN an escalation is created, THE Dashboard SHALL notify nurses in real-time via AWS AppSync GraphQL subscription
3. WHEN a nurse views the EscalationFeed, THE Dashboard SHALL display escalations in Kanban-style list
4. WHEN an escalation involves urgent symptoms (chest pain, dizziness), THE Dashboard SHALL display with bg-red-600 text-white animate-pulse styling
5. WHEN a nurse accepts an escalated call, THE Dashboard SHALL display the patient's complete Patient_Profile
6. WHEN a nurse accepts an escalated call, THE Dashboard SHALL display the current call context and reason for escalation
7. WHEN a nurse clicks "CALL NOW", THE Dashboard SHALL trigger a tel: link to initiate the call
8. WHEN a nurse determines urgent medical care is needed, THE Dashboard SHALL provide options to escalate to doctors or emergency services
9. WHEN a nurse completes an escalated call, THE Dashboard SHALL require the nurse to log notes in the Patient_Profile
10. WHEN a nurse acknowledges an escalation, THE Dashboard SHALL update the escalation status from "OPEN" to "RESOLVED"

### Requirement 15: AI Conversational Intelligence and Safety Constraints

**User Story:** As a hospital administrator, I want the AI assistant "Seva" to provide compassionate, safe, and medically appropriate responses, so that patients receive helpful guidance without inappropriate medical advice.

#### Acceptance Criteria

1. THE AI_Call_System SHALL use Amazon Bedrock with Anthropic Claude 3 Haiku model for conversational intelligence
2. WHEN speaking to patients, THE AI_Call_System SHALL use a warm, respectful, and patient tone
3. WHEN speaking to patients, THE AI_Call_System SHALL use Indian English mannerisms (e.g., "Ji", "Don't worry")
4. THE AI_Call_System SHALL keep spoken responses under 15 words to maintain low latency (<1.5 seconds)
5. THE AI_Call_System SHALL NOT provide medical advice beyond approved hospital documents
6. WHEN a patient mentions urgent symptoms (chest pain, dizziness), THE AI_Call_System SHALL respond "I am alerting the nurse immediately" and trigger escalation
7. WHEN a patient asks questions, THE AI_Call_System SHALL only answer from doctor-approved and hospital-approved documents
8. WHEN the AI_Call_System cannot answer from approved sources, THE AI_Call_System SHALL offer to escalate to a nurse

### Requirement 16: Call State Management and Orchestration

**User Story:** As a system architect, I want the call flow to be managed by a state machine, so that conversations follow a consistent structure and can be monitored and debugged.

#### Acceptance Criteria

1. THE AI_Call_System SHALL use AWS Step Functions to orchestrate call state transitions
2. THE AI_Call_System SHALL implement these call states: Greeting → Medication Check → Health Check → Goodbye
3. WHEN a call begins, THE AI_Call_System SHALL start in the Greeting state
4. WHEN the Greeting completes, THE AI_Call_System SHALL transition to Medication Check state
5. WHEN Medication Check completes, THE AI_Call_System SHALL transition to Health Check state
6. WHEN Health Check completes, THE AI_Call_System SHALL transition to Goodbye state
7. WHEN any state detects an escalation trigger, THE AI_Call_System SHALL transition to Escalation state immediately

### Requirement 17: Admin Bulk Patient Import System

**User Story:** As a hospital administrator, I want to upload patient data in bulk via CSV/Excel files, so that I can efficiently onboard large numbers of patients without manual data entry.

#### Acceptance Criteria

1. WHEN an admin accesses the Admin Dashboard, THE Dashboard SHALL display an upload zone with drag-and-drop functionality
2. WHEN an admin drags a CSV or Excel file to the upload zone, THE Dashboard SHALL upload the file to S3 bucket "patient-imports"
3. WHEN a file is uploaded to S3 bucket "patient-imports", THE S3 bucket SHALL trigger the Bulk_Import_Lambda function
4. WHEN the Bulk_Import_Lambda processes a CSV file, THE Bulk_Import_Lambda SHALL parse each row for patient data
5. WHEN a CSV row contains a Patient_ID field with a value, THE Bulk_Import_Lambda SHALL use that Patient_ID
6. WHEN a CSV row does not contain a Patient_ID field or the field is empty, THE Bulk_Import_Lambda SHALL auto-generate a Patient_ID using the METADATA#PATIENT_COUNTER
7. WHEN the Bulk_Import_Lambda completes processing, THE Bulk_Import_Lambda SHALL write all valid patient records to DynamoDB
8. WHEN the Bulk_Import_Lambda completes processing, THE Bulk_Import_Lambda SHALL send an email report to the admin with import statistics
9. THE email report SHALL include the total number of patients imported and the number of errors encountered
10. WHEN the Bulk_Import_Lambda encounters invalid data, THE Bulk_Import_Lambda SHALL log the error details and continue processing remaining rows
11. THE Dashboard SHALL display a progress bar during file upload showing upload percentage

### Requirement 18: Nurse Smart Desk with Incoming Call Screen Pop

**User Story:** As a nurse, I want patient information to automatically appear on my screen when they call, so that I can provide immediate personalized assistance without searching for their records.

#### Acceptance Criteria

1. WHEN a nurse accesses the Nurse Dashboard, THE Dashboard SHALL embed the Amazon Connect CCP (Contact Control Panel) using Amazon Connect Streams API
2. WHEN a patient calls the hospital phone number, THE Amazon_Connect SHALL identify the caller using Caller ID (ANI)
3. WHEN Amazon_Connect identifies a Caller ID, THE Amazon_Connect SHALL invoke the Lookup_Patient Lambda function with the phone number
4. WHEN the Lookup_Patient Lambda receives a phone number, THE Lookup_Patient Lambda SHALL query DynamoDB for a matching Patient_Profile
5. WHEN a matching Patient_Profile is found, THE Lookup_Patient Lambda SHALL return the Patient_ID and patient name
6. WHEN the browser detects a ringing event via Amazon Connect Streams API, THE Dashboard SHALL automatically navigate to /nurse/patient/[id]
7. WHEN the Dashboard navigates to the patient page, THE Dashboard SHALL display the patient's complete history and current medications before the nurse answers the call
8. WHEN the Caller ID is not found in the system, THE Dashboard SHALL display a notification prompting the nurse to search manually
9. THE Dashboard SHALL provide a manual search interface for nurses to look up patients by name or ID when Caller ID is unavailable
10. THE Amazon Connect CCP SHALL remain visible in the Nurse Dashboard at all times for call control

### Requirement 19: Smart Patient ID Generation and Management

**User Story:** As a system architect, I want a flexible patient ID system that supports both auto-generated sequential IDs and custom IDs, so that the system can accommodate different hospital workflows and existing patient numbering schemes.

#### Acceptance Criteria

1. THE AI_Call_System SHALL maintain a DynamoDB item with PK=METADATA and SK=PATIENT_COUNTER to track the last assigned sequential ID
2. WHEN a doctor creates a new patient and leaves the Custom ID field blank, THE Dashboard SHALL perform an atomic update operation on METADATA#PATIENT_COUNTER to increment the counter
3. WHEN the counter is incremented from value N to N+1, THE Dashboard SHALL assign Patient_ID as "P-{N+1}" (e.g., P-1043)
4. WHEN a doctor creates a new patient and enters a custom ID, THE Dashboard SHALL check DynamoDB for uniqueness of the custom ID
5. WHEN the custom ID is unique, THE Dashboard SHALL assign the custom ID as the Patient_ID
6. WHEN the custom ID already exists, THE Dashboard SHALL display an error message and require the doctor to choose a different ID
7. WHEN the Bulk_Import_Lambda processes a CSV with a Patient_ID column, THE Bulk_Import_Lambda SHALL use the ID from the CSV column
8. WHEN the Bulk_Import_Lambda processes a CSV without a Patient_ID column or with empty Patient_ID values, THE Bulk_Import_Lambda SHALL auto-generate Patient_IDs using atomic updates to METADATA#PATIENT_COUNTER
9. THE Dashboard SHALL display the Patient_ID prominently in all patient views
10. THE AI_Call_System SHALL use Patient_ID as the primary key for all patient-related DynamoDB operations

### Requirement 20: Three-Dashboard System with Role-Based Access

**User Story:** As a hospital stakeholder, I want separate dashboards for admins, doctors, and nurses with role-appropriate functionality, so that each user type can efficiently perform their specific tasks.

#### Acceptance Criteria

1. WHEN an admin logs into the system, THE Dashboard SHALL display the Admin Dashboard at route /admin
2. WHEN the Admin Dashboard loads, THE Dashboard SHALL display an Upload Center with drag-and-drop functionality for CSV/Excel files
3. WHEN a file is being uploaded, THE Admin Dashboard SHALL display a progress bar showing upload percentage
4. WHEN the Admin Dashboard displays import history, THE Dashboard SHALL show an error log table with details of failed imports
5. WHEN an admin accesses the Staff Manager, THE Admin Dashboard SHALL provide functionality to add Doctor and Nurse accounts
6. WHEN an admin creates a Doctor or Nurse account, THE Dashboard SHALL send a Cognito invitation email to the new user
7. WHEN a doctor logs into the system, THE Dashboard SHALL display the Doctor Dashboard at route /doctor
8. WHEN the Doctor Dashboard displays the Add Patient modal, THE Dashboard SHALL provide a toggle between "Auto-Generate ID" and "Custom ID" options
9. WHEN a doctor uses the Prescription Writer, THE Dashboard SHALL provide a dynamic form with fields for medication name, dosage, frequency, and duration
10. WHEN a doctor creates a prescription, THE Dashboard SHALL provide a Risk Level dropdown with options for Group 1, Group 2A, Group 2B, and Group 3
11. WHEN a nurse logs into the system, THE Dashboard SHALL display the Nurse Dashboard at route /nurse
12. WHEN the Nurse Dashboard loads, THE Dashboard SHALL display the Amazon Connect CCP softphone in the bottom right corner
13. WHEN a nurse views a patient, THE Dashboard SHALL display a Live Patient View with a header showing patient name, age, and risk badge
14. WHEN the Live Patient View loads, THE Dashboard SHALL provide tabs for History, Meds, and Notes
15. WHEN a nurse selects the History tab, THE Dashboard SHALL display a timeline of all AI call logs with Dual_Log format
16. WHEN a nurse selects the Meds tab, THE Dashboard SHALL display all current prescriptions with medication details
17. WHEN a nurse selects the Notes tab, THE Dashboard SHALL provide a text area for the nurse to write and save notes
18. THE Dashboard SHALL ensure all three dashboard types are responsive and accessible on desktop and tablet devices

### Requirement 21: Authentication and Role-Based Authorization with Amazon Cognito

**User Story:** As a hospital administrator, I want secure authentication with role-based access control, so that users can only access features appropriate to their role and patient data is protected.

#### Acceptance Criteria

1. THE AI_Call_System SHALL use Amazon Cognito User Pools for authentication in the ap-south-1 region
2. THE AI_Call_System SHALL create three Cognito Groups: AdminGroup, DoctorGroup, and NurseGroup
3. WHEN a user is assigned to AdminGroup, THE Dashboard SHALL grant full access to all /admin routes
4. WHEN a user is assigned to AdminGroup, THE Dashboard SHALL allow creation of Doctor and Nurse accounts
5. WHEN a user is assigned to AdminGroup, THE Dashboard SHALL allow viewing and editing of all patient records across all doctors
6. WHEN a user is assigned to DoctorGroup, THE Dashboard SHALL grant access to /doctor routes only
7. WHEN a user is assigned to DoctorGroup, THE Dashboard SHALL allow viewing and editing of medical records for patients under their care
8. WHEN a user is assigned to DoctorGroup, THE Dashboard SHALL allow creation and modification of prescriptions for their patients
9. WHEN a user is assigned to DoctorGroup, THE Dashboard SHALL allow assignment and modification of Risk_Group for their patients
10. WHEN a user is assigned to NurseGroup, THE Dashboard SHALL grant access to /nurse routes only
11. WHEN a user is assigned to NurseGroup, THE Dashboard SHALL provide read-only access to medical records and prescriptions
12. WHEN a user is assigned to NurseGroup, THE Dashboard SHALL provide write access to the Notes section for all patients
13. WHEN a user is assigned to NurseGroup, THE Dashboard SHALL allow viewing of escalation alerts and patient call history
14. WHEN a user attempts to access a route without proper authorization, THE Dashboard SHALL redirect to an unauthorized error page
15. WHEN a user logs in, THE Dashboard SHALL verify their Cognito Group membership and route them to the appropriate dashboard
16. THE Dashboard SHALL use Cognito JWT tokens for API authentication with Lambda functions
17. THE Dashboard SHALL implement token refresh logic to maintain user sessions securely
