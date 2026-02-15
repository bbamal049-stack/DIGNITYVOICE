# Design Document: DignityVoice - AI Voice Call System for Patient Medication Management

## Overview

DignityVoice is a comprehensive AI-powered healthcare platform built on AWS infrastructure in the ap-south-1 (Mumbai) region. The system enables automated medication reminders and patient monitoring through PSTN voice calls, while providing healthcare providers with three specialized dashboards (Admin, Doctor, Nurse) for patient management, bulk data import, and real-time escalation handling.

The architecture follows a serverless, event-driven design using AWS services including Amazon Connect for telephony, Lambda for business logic, DynamoDB for data storage, S3 for file storage, Cognito for authentication, and Bedrock for AI conversational intelligence. The system implements a "Zero-Retention" privacy model where patient voice data is immediately flushed after summarization.

## Architecture

### High-Level System Components

1. **Telephony Layer**: Amazon Connect with PSTN integration for voice calls
2. **AI Processing Layer**: Lambda functions with Amazon Bedrock (Claude 3 Haiku) for conversational AI
3. **Data Layer**: DynamoDB single-table design with S3 for temporary audio storage
4. **Authentication Layer**: Amazon Cognito User Pools with role-based access control
5. **Frontend Layer**: Next.js web application with three role-specific dashboards
6. **Bulk Import Layer**: S3-triggered Lambda for CSV/Excel processing
7. **Real-Time Layer**: AWS AppSync for GraphQL subscriptions and live escalation feeds

### AWS Services and Region

- **Region**: ap-south-1 (Mumbai)
- **Amazon Connect**: PSTN telephony with Indian DID (+91...)
- **AWS Lambda**: Serverless compute for business logic
- **Amazon DynamoDB**: NoSQL database for patient data and logs
- **Amazon S3**: Object storage for file uploads and temporary audio
- **Amazon Cognito**: User authentication and authorization
- **Amazon Bedrock**: AI model hosting (Claude 3 Haiku)
- **AWS Step Functions**: Call state orchestration
- **AWS AppSync**: Real-time GraphQL API for escalations
- **Amazon SES**: Email service for import reports and notifications
- **Amazon Polly**: Text-to-speech with Indian voices (Kajal, Aditi)


## Components and Interfaces

### 1. Amazon Connect Contact Flow (AI_Nurse_Flow)

**Purpose**: Orchestrates inbound and outbound call logic, integrating with Lambda for AI processing.

**Configuration**:
- Contact Flow Name: AI_Nurse_Flow
- Language: en-IN (Indian English) or hi-IN (Hindi)
- Speech Recognition: Amazon Lex V2 or native Connect STT
- Text-to-Speech: Amazon Polly (Voice: Kajal or Aditi)
- Call Recording: Enabled with S3 storage in "temp-voice-storage" bucket

**Flow Logic**:
1. **Greeting Block**: Play welcome message "Hello, this is Seva from [Hospital Name]"
2. **Get Customer Input**: Capture patient speech with 5-second timeout
3. **Invoke Lambda**: Call Voice_Brain Lambda with patient input and call context
4. **Play Prompt**: Speak AI-generated response using Polly
5. **Check for Escalation**: If escalation flag is true, transfer to nurse queue
6. **Loop or End**: Continue conversation or end call based on state

**Escalation Transfer**:
- Transfer Type: Queue transfer to "Nurse-Escalation-Queue"
- Transfer Attributes: Include Patient_ID, escalation_reason, call_context
- Timeout: 30 seconds before voicemail

### 2. Voice_Brain Lambda Function

**Purpose**: Processes patient speech input and generates AI responses using Amazon Bedrock.

**Runtime**: Python 3.11
**Memory**: 512 MB
**Timeout**: 30 seconds

**Input Event**:
```json
{
  "patient_id": "P-1043",
  "user_input": "Yes, I took my medicine",
  "call_type": "medication_reminder",
  "call_state": "medication_check",
  "conversation_history": []
}
```

**Processing Logic**:
1. Retrieve Patient_Profile from DynamoDB (PK=PATIENT#{id})
2. Load approved hospital documents from S3 for RAG context
3. Build prompt with patient context, conversation history, and safety constraints
4. Call Amazon Bedrock (Claude 3 Haiku) with max_tokens=50 for low latency
5. Parse AI response for escalation triggers (chest pain, dizziness, "nurse")
6. Update call state in Step Functions
7. Return response with escalation flag

**Output**:
```json
{
  "ai_response": "Great! I'm happy to hear that.",
  "escalation_flag": false,
  "next_state": "health_check",
  "detected_sentiment": "Positive"
}
```

**Safety Constraints**:
- Maximum response length: 15 words
- No medical advice beyond approved documents
- Immediate escalation on urgent symptoms
- Indian English mannerisms (Ji, Don't worry)


### 3. Dialer_Agent Lambda Function

**Purpose**: Schedules and initiates outbound calls to patients based on Risk_Group and medication schedules.

**Runtime**: Python 3.11
**Memory**: 256 MB
**Timeout**: 60 seconds
**Trigger**: EventBridge scheduled rule (runs every 15 minutes)

**Processing Logic**:
1. Query DynamoDB for all patients with GSI: GSI1PK=RISK_GROUP#{group}, GSI1SK=NEXT_CALL#{timestamp}
2. Filter patients where NEXT_CALL <= current_time
3. For each patient, determine call type (medication_reminder or daily_summary)
4. Invoke Amazon Connect StartOutboundVoiceContact API
5. Update patient's NEXT_CALL timestamp based on Risk_Group schedule
6. Log call initiation in DynamoDB

**Risk Group Schedules**:
- Group 1 (High Risk): 3 calls daily at 08:00, 14:00, 20:00 IST
- Group 2A (Probation): 3 calls daily at 08:00, 14:00, 20:00 IST
- Group 2B (Trusted): 1 call daily at 20:00 IST
- Group 3 (Low Risk): 1 call daily at 20:00 IST

**Amazon Connect API Call**:
```python
connect_client.start_outbound_voice_contact(
    InstanceId='connect-instance-id',
    ContactFlowId='ai-nurse-flow-id',
    DestinationPhoneNumber=patient_phone,
    Attributes={
        'patient_id': patient_id,
        'call_type': call_type,
        'risk_group': risk_group
    }
)
```

### 4. Flush_Protocol Lambda Function

**Purpose**: Processes call recordings, generates Dual_Log summaries, and deletes voice data for privacy.

**Runtime**: Python 3.11
**Memory**: 1024 MB
**Timeout**: 300 seconds (5 minutes)
**Trigger**: S3 Event Notification on "temp-voice-storage" bucket (ObjectCreated)

**Processing Logic**:
1. Receive S3 event with audio file key
2. Download audio file and transcript from S3
3. Build Bedrock prompt for Dual_Log extraction
4. Call Amazon Bedrock with full transcript
5. Parse response into Narrative_Summary and Clinical_Metrics JSON
6. Write Dual_Log to DynamoDB (PK=PATIENT#{id}, SK=LOG#{timestamp})
7. Execute hard delete on S3 objects (audio and transcript)
8. Verify deletion and log completion

**Bedrock Prompt Template**:
```
Analyze this patient call transcript and generate:
1. Narrative Summary (2-3 sentences)
2. Clinical Metrics JSON with:
   - medication_status: "Fully Adherent" | "Partial" | "Missed" | "Unknown"
   - reported_symptoms: [list of strings]
   - pain_scale: 0-10
   - mood_sentiment: "Positive" | "Neutral" | "Distressed"
   - escalation_flag: boolean

Transcript: {transcript_text}
```

**Dual_Log DynamoDB Schema**:
```json
{
  "PK": "PATIENT#P-1043",
  "SK": "LOG#2024-01-15T14:30:00Z",
  "call_type": "medication_reminder",
  "narrative_summary": "Patient confirmed taking morning medication...",
  "clinical_metrics": {
    "medication_status": "Fully Adherent",
    "reported_symptoms": [],
    "pain_scale": 2,
    "mood_sentiment": "Positive",
    "escalation_flag": false
  },
  "call_duration_seconds": 45
}
```


### 5. Bulk_Import_Lambda Function (NEW)

**Purpose**: Processes CSV/Excel files uploaded by admins to bulk import patient data.

**Runtime**: Python 3.11
**Memory**: 1024 MB
**Timeout**: 900 seconds (15 minutes)
**Trigger**: S3 Event Notification on "patient-imports" bucket (ObjectCreated)

**Processing Logic**:
1. Receive S3 event with uploaded file key
2. Download file from S3 and detect format (CSV or Excel)
3. Parse file using pandas library
4. For each row:
   - Check if Patient_ID column exists and has value
   - If Patient_ID exists: Use provided ID
   - If Patient_ID missing/empty: Call get_next_patient_id() for auto-generation
   - Validate required fields (name, phone, doctor_id)
   - Check for duplicate Patient_ID in DynamoDB
   - Write patient record to DynamoDB if valid
   - Log error if validation fails
5. Generate import report with statistics
6. Send email report via Amazon SES to admin
7. Clean up temporary files

**Auto-ID Generation Function**:
```python
def get_next_patient_id():
    response = dynamodb.update_item(
        TableName='DignityVoice',
        Key={'PK': 'METADATA', 'SK': 'PATIENT_COUNTER'},
        UpdateExpression='ADD counter_value :inc',
        ExpressionAttributeValues={':inc': 1},
        ReturnValues='UPDATED_NEW'
    )
    counter = response['Attributes']['counter_value']
    return f"P-{counter}"
```

**Expected CSV Format**:
```csv
Patient_ID,Name,Phone,Age,Doctor_ID,Risk_Group,Preferred_Language
P-1001,Ramesh Kumar,+919876543210,65,DOC-123,Group 1,en-IN
,Priya Sharma,+919876543211,58,DOC-123,Group 2A,hi-IN
ICU-BED-5,Anil Verma,+919876543212,72,DOC-124,Group 1,en-IN
```

**Email Report Template**:
```
Subject: Patient Import Complete - [Filename]

Import Summary:
- Total Rows: 500
- Successfully Imported: 498
- Errors: 2

Error Details:
- Row 45: Missing required field 'Phone'
- Row 203: Duplicate Patient_ID 'P-1001'

Import completed at: 2024-01-15 14:30:00 IST
```

**DynamoDB Patient Record**:
```json
{
  "PK": "PATIENT#P-1043",
  "SK": "PROFILE",
  "patient_id": "P-1043",
  "name": "Ramesh Kumar",
  "phone": "+919876543210",
  "age": 65,
  "doctor_id": "DOC-123",
  "risk_group": "Group 1",
  "preferred_language": "en-IN",
  "adherence_streak": 0,
  "created_at": "2024-01-15T14:30:00Z",
  "created_by": "ADMIN#admin@hospital.com"
}
```


### 6. Lookup_Patient Lambda Function (NEW)

**Purpose**: Queries DynamoDB to find patient information by phone number for screen pop functionality.

**Runtime**: Python 3.11
**Memory**: 256 MB
**Timeout**: 10 seconds
**Trigger**: Invoked by Amazon Connect Contact Flow or API Gateway

**Input Event**:
```json
{
  "phone_number": "+919876543210"
}
```

**Processing Logic**:
1. Normalize phone number (remove spaces, ensure +91 prefix)
2. Query DynamoDB using GSI: GSI2PK=PHONE#{phone}, GSI2SK=PROFILE
3. If patient found, return patient_id and name
4. If not found, return null with appropriate message
5. Log lookup attempt for audit trail

**Output**:
```json
{
  "found": true,
  "patient_id": "P-1043",
  "name": "Ramesh Kumar",
  "risk_group": "Group 1",
  "doctor_id": "DOC-123"
}
```

**DynamoDB GSI Configuration**:
- GSI Name: GSI2-Phone-Index
- Partition Key: GSI2PK (String)
- Sort Key: GSI2SK (String)
- Projection: ALL

**Patient Record with GSI Attributes**:
```json
{
  "PK": "PATIENT#P-1043",
  "SK": "PROFILE",
  "GSI2PK": "PHONE#+919876543210",
  "GSI2SK": "PROFILE",
  "patient_id": "P-1043",
  "name": "Ramesh Kumar",
  "phone": "+919876543210"
}
```

### 7. Risk_Engine Module

**Purpose**: Determines patient risk groups, manages adherence streaks, and handles automatic promotions/demotions.

**Implementation**: Python module imported by Voice_Brain and Dialer_Agent Lambdas

**Core Functions**:

**calculate_next_call_time(risk_group, current_time)**:
```python
def calculate_next_call_time(risk_group, current_time):
    schedules = {
        'Group 1': ['08:00', '14:00', '20:00'],
        'Group 2A': ['08:00', '14:00', '20:00'],
        'Group 2B': ['20:00'],
        'Group 3': ['20:00']
    }
    times = schedules[risk_group]
    # Find next scheduled time after current_time
    next_time = find_next_occurrence(times, current_time)
    return next_time
```

**update_adherence_streak(patient_id, medication_status, sentiment)**:
```python
def update_adherence_streak(patient_id, medication_status, sentiment):
    if medication_status == 'Fully Adherent' and sentiment != 'Distressed':
        increment_streak(patient_id)
    else:
        reset_streak(patient_id)
    
    check_for_promotion_demotion(patient_id)
```

**check_for_promotion_demotion(patient_id)**:
```python
def check_for_promotion_demotion(patient_id):
    patient = get_patient(patient_id)
    
    # Promotion: 2A -> 2B after 14 days perfect adherence
    if patient['risk_group'] == 'Group 2A' and patient['adherence_streak'] >= 14:
        update_risk_group(patient_id, 'Group 2B')
        send_notification(patient_id, 'Promoted to Group 2B')
    
    # Demotion: 2B -> 2A on any missed medication or distress
    if patient['risk_group'] == 'Group 2B':
        if patient['adherence_streak'] == 0:
            update_risk_group(patient_id, 'Group 2A')
            send_notification(patient_id, 'Demoted to Group 2A')
```


### 8. Frontend Architecture (Next.js Application)

**Framework**: Next.js 14 with App Router
**Styling**: Tailwind CSS
**State Management**: React Context + SWR for data fetching
**Authentication**: AWS Amplify with Cognito integration
**Real-Time**: AWS AppSync GraphQL subscriptions

**Route Structure**:
```
/
├── /login (public)
├── /admin (AdminGroup only)
│   ├── /upload
│   ├── /staff
│   └── /patients
├── /doctor (DoctorGroup only)
│   ├── /patients
│   ├── /add-patient
│   └── /prescriptions
└── /nurse (NurseGroup only)
    ├── /escalations
    └── /patient/[id]
```

**Authentication Flow**:
1. User visits /login and enters credentials
2. Amplify calls Cognito authenticateUser()
3. Cognito returns JWT tokens with group claims
4. Frontend extracts groups from token: `token.payload['cognito:groups']`
5. Route guard checks group membership and redirects to appropriate dashboard
6. API calls include JWT in Authorization header

**Protected Route Component**:
```typescript
function ProtectedRoute({ children, allowedGroups }) {
  const { user, groups } = useAuth();
  
  if (!user) return <Navigate to="/login" />;
  
  const hasAccess = groups.some(g => allowedGroups.includes(g));
  if (!hasAccess) return <UnauthorizedPage />;
  
  return children;
}
```

### 9. Admin Dashboard Components (NEW)

**Upload Center Component**:
```typescript
// components/admin/UploadCenter.tsx
function UploadCenter() {
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);
  
  const handleDrop = async (files) => {
    const file = files[0];
    setUploading(true);
    
    // Upload to S3 with progress tracking
    await Storage.put(`imports/${file.name}`, file, {
      progressCallback: (p) => {
        setProgress((p.loaded / p.total) * 100);
      }
    });
    
    setUploading(false);
    showSuccessMessage('File uploaded successfully');
  };
  
  return (
    <div className="border-2 border-dashed border-gray-300 rounded-lg p-8">
      <Dropzone onDrop={handleDrop}>
        <p>Drag & drop CSV/Excel files here</p>
      </Dropzone>
      {uploading && <ProgressBar value={progress} />}
    </div>
  );
}
```

**Staff Manager Component**:
```typescript
// components/admin/StaffManager.tsx
function StaffManager() {
  const [role, setRole] = useState('Doctor');
  
  const createStaffAccount = async (formData) => {
    const response = await API.post('api', '/admin/create-staff', {
      body: {
        email: formData.email,
        name: formData.name,
        role: role, // 'Doctor' or 'Nurse'
        group: role === 'Doctor' ? 'DoctorGroup' : 'NurseGroup'
      }
    });
    
    // Backend creates Cognito user and sends invitation
    showSuccessMessage(`Invitation sent to ${formData.email}`);
  };
  
  return (
    <form onSubmit={createStaffAccount}>
      <input name="email" type="email" required />
      <input name="name" type="text" required />
      <select value={role} onChange={(e) => setRole(e.target.value)}>
        <option value="Doctor">Doctor</option>
        <option value="Nurse">Nurse</option>
      </select>
      <button type="submit">Create Account</button>
    </form>
  );
}
```

**Error Log Table Component**:
```typescript
// components/admin/ErrorLogTable.tsx
function ErrorLogTable({ importId }) {
  const { data: errors } = useSWR(`/admin/import-errors/${importId}`);
  
  return (
    <table className="min-w-full divide-y divide-gray-200">
      <thead>
        <tr>
          <th>Row Number</th>
          <th>Error Message</th>
          <th>Data</th>
        </tr>
      </thead>
      <tbody>
        {errors?.map(error => (
          <tr key={error.row_number}>
            <td>{error.row_number}</td>
            <td className="text-red-600">{error.message}</td>
            <td className="font-mono text-sm">{JSON.stringify(error.data)}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```


### 10. Doctor Dashboard Components (NEW)

**Add Patient Modal Component**:
```typescript
// components/doctor/AddPatientModal.tsx
function AddPatientModal({ isOpen, onClose }) {
  const [idMode, setIdMode] = useState('auto'); // 'auto' or 'custom'
  const [customId, setCustomId] = useState('');
  
  const createPatient = async (formData) => {
    const payload = {
      name: formData.name,
      phone: formData.phone,
      age: formData.age,
      risk_group: formData.risk_group,
      preferred_language: formData.language,
      id_mode: idMode,
      custom_id: idMode === 'custom' ? customId : null
    };
    
    const response = await API.post('api', '/doctor/patients', { body: payload });
    
    if (response.error === 'DUPLICATE_ID') {
      showError('This Patient ID already exists');
      return;
    }
    
    showSuccess(`Patient created with ID: ${response.patient_id}`);
    onClose();
  };
  
  return (
    <Modal isOpen={isOpen} onClose={onClose}>
      <form onSubmit={createPatient}>
        <input name="name" placeholder="Patient Name" required />
        <input name="phone" placeholder="+91..." required />
        <input name="age" type="number" required />
        
        <div className="flex gap-4">
          <label>
            <input 
              type="radio" 
              checked={idMode === 'auto'} 
              onChange={() => setIdMode('auto')} 
            />
            Auto-Generate ID
          </label>
          <label>
            <input 
              type="radio" 
              checked={idMode === 'custom'} 
              onChange={() => setIdMode('custom')} 
            />
            Custom ID
          </label>
        </div>
        
        {idMode === 'custom' && (
          <input 
            value={customId} 
            onChange={(e) => setCustomId(e.target.value)}
            placeholder="e.g., ICU-BED-5" 
            required 
          />
        )}
        
        <select name="risk_group" required>
          <option value="Group 1">Group 1 - High Risk</option>
          <option value="Group 2A">Group 2A - Probation</option>
          <option value="Group 2B">Group 2B - Trusted</option>
          <option value="Group 3">Group 3 - Low Risk</option>
        </select>
        
        <select name="language" required>
          <option value="en-IN">English (India)</option>
          <option value="hi-IN">Hindi</option>
        </select>
        
        <button type="submit">Create Patient</button>
      </form>
    </Modal>
  );
}
```

**Prescription Writer Component**:
```typescript
// components/doctor/PrescriptionWriter.tsx
function PrescriptionWriter({ patientId }) {
  const [medications, setMedications] = useState([
    { name: '', dosage: '', frequency: '', duration: '' }
  ]);
  
  const addMedication = () => {
    setMedications([...medications, { name: '', dosage: '', frequency: '', duration: '' }]);
  };
  
  const savePrescription = async () => {
    const response = await API.post('api', '/doctor/prescriptions', {
      body: {
        patient_id: patientId,
        medications: medications,
        created_at: new Date().toISOString()
      }
    });
    
    showSuccess('Prescription saved successfully');
  };
  
  return (
    <div className="space-y-4">
      {medications.map((med, index) => (
        <div key={index} className="grid grid-cols-4 gap-4">
          <input 
            placeholder="Medication Name"
            value={med.name}
            onChange={(e) => updateMedication(index, 'name', e.target.value)}
          />
          <input 
            placeholder="Dosage (e.g., 500mg)"
            value={med.dosage}
            onChange={(e) => updateMedication(index, 'dosage', e.target.value)}
          />
          <select 
            value={med.frequency}
            onChange={(e) => updateMedication(index, 'frequency', e.target.value)}
          >
            <option value="">Frequency</option>
            <option value="Once daily">Once daily</option>
            <option value="Twice daily">Twice daily</option>
            <option value="Three times daily">Three times daily</option>
          </select>
          <input 
            placeholder="Duration (e.g., 30 days)"
            value={med.duration}
            onChange={(e) => updateMedication(index, 'duration', e.target.value)}
          />
        </div>
      ))}
      
      <button onClick={addMedication} className="text-blue-600">
        + Add Another Medication
      </button>
      
      <button onClick={savePrescription} className="bg-blue-600 text-white px-4 py-2">
        Save Prescription
      </button>
    </div>
  );
}
```


### 11. Nurse Dashboard Components (NEW)

**Amazon Connect CCP Integration**:
```typescript
// components/nurse/ConnectCCP.tsx
import 'amazon-connect-streams';

function ConnectCCP() {
  const ccpRef = useRef(null);
  const router = useRouter();
  
  useEffect(() => {
    // Initialize Amazon Connect Streams API
    connect.core.initCCP(ccpRef.current, {
      ccpUrl: 'https://your-instance.awsapps.com/connect/ccp-v2',
      loginPopup: false,
      softphone: {
        allowFramedSoftphone: true
      }
    });
    
    // Listen for incoming contacts
    connect.contact((contact) => {
      contact.onConnecting(async () => {
        const attributes = contact.getAttributes();
        const phoneNumber = contact.getInitialConnection().getEndpoint().phoneNumber;
        
        // Call Lookup_Patient Lambda
        const response = await API.post('api', '/nurse/lookup-patient', {
          body: { phone_number: phoneNumber }
        });
        
        if (response.found) {
          // Screen pop: Navigate to patient page
          router.push(`/nurse/patient/${response.patient_id}`);
        } else {
          // Show manual search prompt
          showNotification('Caller ID not found. Please search manually.');
        }
      });
    });
  }, []);
  
  return (
    <div 
      ref={ccpRef} 
      className="fixed bottom-4 right-4 w-80 h-96 border rounded-lg shadow-lg bg-white"
    />
  );
}
```

**Live Patient View Component**:
```typescript
// components/nurse/LivePatientView.tsx
function LivePatientView({ patientId }) {
  const { data: patient } = useSWR(`/nurse/patients/${patientId}`);
  const [activeTab, setActiveTab] = useState('history');
  
  const riskBadgeStyles = {
    'Group 1': 'bg-red-600 text-white',
    'Group 2A': 'bg-orange-400 text-white',
    'Group 2B': 'bg-yellow-400 text-black',
    'Group 3': 'bg-green-400 text-white'
  };
  
  return (
    <div className="space-y-4">
      {/* Header */}
      <div className="bg-white p-6 rounded-lg shadow">
        <div className="flex items-center justify-between">
          <div>
            <h1 className="text-2xl font-bold">{patient?.name}</h1>
            <p className="text-gray-600">Age: {patient?.age} | ID: {patient?.patient_id}</p>
          </div>
          <span className={`px-4 py-2 rounded-full ${riskBadgeStyles[patient?.risk_group]}`}>
            {patient?.risk_group}
          </span>
        </div>
      </div>
      
      {/* Tabs */}
      <div className="border-b border-gray-200">
        <nav className="flex space-x-8">
          <button 
            onClick={() => setActiveTab('history')}
            className={activeTab === 'history' ? 'border-b-2 border-blue-600' : ''}
          >
            History
          </button>
          <button 
            onClick={() => setActiveTab('meds')}
            className={activeTab === 'meds' ? 'border-b-2 border-blue-600' : ''}
          >
            Medications
          </button>
          <button 
            onClick={() => setActiveTab('notes')}
            className={activeTab === 'notes' ? 'border-b-2 border-blue-600' : ''}
          >
            Notes
          </button>
        </nav>
      </div>
      
      {/* Tab Content */}
      <div className="bg-white p-6 rounded-lg shadow">
        {activeTab === 'history' && <HistoryTimeline patientId={patientId} />}
        {activeTab === 'meds' && <MedicationList patientId={patientId} />}
        {activeTab === 'notes' && <NurseNotes patientId={patientId} />}
      </div>
    </div>
  );
}
```

**History Timeline Component**:
```typescript
// components/nurse/HistoryTimeline.tsx
function HistoryTimeline({ patientId }) {
  const { data: logs } = useSWR(`/nurse/patients/${patientId}/logs`);
  
  return (
    <div className="space-y-4">
      {logs?.map(log => (
        <div key={log.SK} className="border-l-4 border-blue-600 pl-4">
          <div className="flex justify-between">
            <span className="font-semibold">{log.call_type}</span>
            <span className="text-gray-500">{formatDate(log.SK)}</span>
          </div>
          
          {/* Narrative Summary */}
          <p className="text-gray-700 mt-2">{log.narrative_summary}</p>
          
          {/* Clinical Metrics Table */}
          <table className="mt-4 w-full text-sm">
            <tbody>
              <tr>
                <td className="font-semibold">Medication Status:</td>
                <td>{log.clinical_metrics.medication_status}</td>
              </tr>
              <tr>
                <td className="font-semibold">Pain Scale:</td>
                <td>{log.clinical_metrics.pain_scale}/10</td>
              </tr>
              <tr>
                <td className="font-semibold">Mood:</td>
                <td>{log.clinical_metrics.mood_sentiment}</td>
              </tr>
              {log.clinical_metrics.reported_symptoms.length > 0 && (
                <tr>
                  <td className="font-semibold">Symptoms:</td>
                  <td>{log.clinical_metrics.reported_symptoms.join(', ')}</td>
                </tr>
              )}
            </tbody>
          </table>
        </div>
      ))}
    </div>
  );
}
```

**Nurse Notes Component**:
```typescript
// components/nurse/NurseNotes.tsx
function NurseNotes({ patientId }) {
  const { data: notes, mutate } = useSWR(`/nurse/patients/${patientId}/notes`);
  const [newNote, setNewNote] = useState('');
  
  const saveNote = async () => {
    await API.post('api', '/nurse/notes', {
      body: {
        patient_id: patientId,
        note: newNote,
        timestamp: new Date().toISOString()
      }
    });
    
    setNewNote('');
    mutate(); // Refresh notes list
    showSuccess('Note saved');
  };
  
  return (
    <div className="space-y-4">
      {/* Previous Notes */}
      <div className="space-y-2">
        {notes?.map(note => (
          <div key={note.timestamp} className="bg-gray-50 p-4 rounded">
            <p className="text-sm text-gray-600">{formatDate(note.timestamp)}</p>
            <p className="mt-1">{note.note}</p>
            <p className="text-xs text-gray-500 mt-2">By: {note.nurse_name}</p>
          </div>
        ))}
      </div>
      
      {/* New Note Input */}
      <div>
        <textarea 
          value={newNote}
          onChange={(e) => setNewNote(e.target.value)}
          placeholder="Add a note about this patient..."
          className="w-full h-32 p-4 border rounded"
        />
        <button 
          onClick={saveNote}
          className="mt-2 bg-blue-600 text-white px-4 py-2 rounded"
        >
          Save Note
        </button>
      </div>
    </div>
  );
}
```


## Data Models

### DynamoDB Single Table Design

**Table Name**: DignityVoice
**Region**: ap-south-1
**Billing Mode**: On-Demand

**Primary Key**:
- Partition Key (PK): String
- Sort Key (SK): String

**Global Secondary Indexes**:

1. **GSI1-RiskGroup-Index**:
   - PK: GSI1PK (String) - Format: RISK_GROUP#{group}
   - SK: GSI1SK (String) - Format: NEXT_CALL#{timestamp}
   - Purpose: Query patients by risk group for scheduled calls

2. **GSI2-Phone-Index**:
   - PK: GSI2PK (String) - Format: PHONE#{phone_number}
   - SK: GSI2SK (String) - Format: PROFILE
   - Purpose: Lookup patients by phone number for screen pop

3. **GSI3-Doctor-Index**:
   - PK: GSI3PK (String) - Format: DOCTOR#{doctor_id}
   - SK: GSI3SK (String) - Format: PATIENT#{patient_id}
   - Purpose: Query all patients under a specific doctor

### Entity Schemas

**1. Patient Profile**:
```json
{
  "PK": "PATIENT#P-1043",
  "SK": "PROFILE",
  "GSI1PK": "RISK_GROUP#Group 1",
  "GSI1SK": "NEXT_CALL#2024-01-15T08:00:00Z",
  "GSI2PK": "PHONE#+919876543210",
  "GSI2SK": "PROFILE",
  "GSI3PK": "DOCTOR#DOC-123",
  "GSI3SK": "PATIENT#P-1043",
  "patient_id": "P-1043",
  "name": "Ramesh Kumar",
  "phone": "+919876543210",
  "age": 65,
  "doctor_id": "DOC-123",
  "risk_group": "Group 1",
  "adherence_streak": 7,
  "preferred_language": "en-IN",
  "created_at": "2024-01-10T10:00:00Z",
  "updated_at": "2024-01-15T07:00:00Z"
}
```

**2. Call Log (Dual_Log)**:
```json
{
  "PK": "PATIENT#P-1043",
  "SK": "LOG#2024-01-15T08:30:00Z",
  "call_type": "medication_reminder",
  "call_duration_seconds": 45,
  "narrative_summary": "Patient confirmed taking morning medication on time. Reported feeling well with no new symptoms. Pain level remains at 2/10.",
  "clinical_metrics": {
    "medication_status": "Fully Adherent",
    "reported_symptoms": [],
    "pain_scale": 2,
    "mood_sentiment": "Positive",
    "escalation_flag": false
  },
  "ai_model": "claude-3-haiku",
  "created_at": "2024-01-15T08:30:00Z"
}
```

**3. Prescription**:
```json
{
  "PK": "PATIENT#P-1043",
  "SK": "PRESCRIPTION#2024-01-10",
  "prescription_id": "RX-5678",
  "doctor_id": "DOC-123",
  "medications": [
    {
      "name": "Metformin",
      "dosage": "500mg",
      "frequency": "Twice daily",
      "duration": "30 days",
      "description": "Round yellow tablet"
    },
    {
      "name": "Lisinopril",
      "dosage": "10mg",
      "frequency": "Once daily",
      "duration": "30 days",
      "description": "Small white pill"
    }
  ],
  "created_at": "2024-01-10T14:00:00Z",
  "created_by": "DOC-123"
}
```

**4. Escalation Alert**:
```json
{
  "PK": "HOSPITAL#HOSP-001",
  "SK": "ALERT#2024-01-15T08:35:00Z",
  "alert_id": "ALERT-9876",
  "patient_id": "P-1043",
  "patient_name": "Ramesh Kumar",
  "escalation_reason": "Patient reported chest pain",
  "urgency": "HIGH",
  "call_context": {
    "call_type": "daily_summary",
    "reported_symptoms": ["chest pain", "shortness of breath"],
    "pain_scale": 8
  },
  "status": "OPEN",
  "created_at": "2024-01-15T08:35:00Z",
  "resolved_at": null,
  "resolved_by": null
}
```

**5. Nurse Note**:
```json
{
  "PK": "PATIENT#P-1043",
  "SK": "NOTE#2024-01-15T09:00:00Z",
  "note_id": "NOTE-4321",
  "nurse_id": "NURSE-456",
  "nurse_name": "Priya Sharma",
  "note": "Spoke with patient after escalation. Chest pain was due to indigestion. Advised to take antacid. Will monitor closely.",
  "created_at": "2024-01-15T09:00:00Z"
}
```

**6. Metadata Counter**:
```json
{
  "PK": "METADATA",
  "SK": "PATIENT_COUNTER",
  "counter_value": 1043,
  "last_updated": "2024-01-15T10:30:00Z"
}
```

**7. Doctor Profile**:
```json
{
  "PK": "DOCTOR#DOC-123",
  "SK": "PROFILE",
  "doctor_id": "DOC-123",
  "name": "Dr. Anil Mehta",
  "email": "anil.mehta@hospital.com",
  "phone": "+919876543200",
  "specialization": "Cardiology",
  "cognito_username": "anil.mehta@hospital.com",
  "created_at": "2024-01-01T00:00:00Z"
}
```

**8. Nurse Profile**:
```json
{
  "PK": "NURSE#NURSE-456",
  "SK": "PROFILE",
  "nurse_id": "NURSE-456",
  "name": "Priya Sharma",
  "email": "priya.sharma@hospital.com",
  "phone": "+919876543201",
  "shift": "Morning",
  "cognito_username": "priya.sharma@hospital.com",
  "created_at": "2024-01-01T00:00:00Z"
}
```

**9. Bulk Import Record**:
```json
{
  "PK": "IMPORT#2024-01-15",
  "SK": "BATCH#12345",
  "import_id": "IMPORT-12345",
  "filename": "patients_jan_2024.csv",
  "uploaded_by": "ADMIN#admin@hospital.com",
  "total_rows": 500,
  "successful_imports": 498,
  "failed_imports": 2,
  "errors": [
    {
      "row_number": 45,
      "error": "Missing required field 'Phone'",
      "data": {"Name": "John Doe", "Age": 60}
    },
    {
      "row_number": 203,
      "error": "Duplicate Patient_ID 'P-1001'",
      "data": {"Patient_ID": "P-1001", "Name": "Jane Smith"}
    }
  ],
  "status": "COMPLETED",
  "created_at": "2024-01-15T10:00:00Z",
  "completed_at": "2024-01-15T10:05:00Z"
}
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Patient ID Uniqueness

*For any* patient creation operation (manual or bulk import), the assigned Patient_ID should be unique across the entire system, and duplicate ID assignment should be rejected with an error.

**Validates: Requirements 17.6, 19.4, 19.5, 19.6**

### Property 2: Atomic Counter Increment

*For any* auto-generated Patient_ID, the METADATA#PATIENT_COUNTER should be atomically incremented, and the resulting ID should be exactly "P-{counter_value}" where counter_value is the new incremented value.

**Validates: Requirements 19.2, 19.3, 19.8**

### Property 3: Bulk Import Idempotency

*For any* CSV file uploaded to S3, processing the same file multiple times should not create duplicate patient records, and the system should detect and skip existing Patient_IDs.

**Validates: Requirements 17.3, 17.4, 17.7**

### Property 4: Screen Pop Caller ID Lookup

*For any* incoming call with a valid Caller ID (ANI), if the phone number exists in DynamoDB, the Lookup_Patient Lambda should return the correct Patient_ID and name within 2 seconds.

**Validates: Requirements 18.3, 18.4, 18.5**

### Property 5: Role-Based Route Access

*For any* authenticated user, attempting to access a route should succeed if and only if the user's Cognito group membership includes the required group for that route.

**Validates: Requirements 21.3, 21.6, 21.10, 21.14**

### Property 6: Nurse Notes Write Permission

*For any* user in NurseGroup, they should have write access to the Notes section for all patients, while having read-only access to medical records and prescriptions.

**Validates: Requirements 21.12, 21.13**

### Property 7: Admin Staff Creation

*For any* user in AdminGroup, creating a new Doctor or Nurse account should result in a Cognito user being created with the appropriate group membership and an invitation email being sent.

**Validates: Requirements 21.4, 20.6**

### Property 8: Risk Group Call Scheduling

*For any* patient with a specific Risk_Group, the Dialer_Agent should schedule calls according to the correct frequency: Group 1 and 2A receive 3 calls daily, while Group 2B and 3 receive 1 call daily.

**Validates: Requirements 6.1, 7.1, 8.1**

### Property 9: Adherence Streak Promotion

*For any* patient in Risk_Group 2A, maintaining perfect adherence (medication_status = "Fully Adherent" and mood_sentiment != "Distressed") for 14 consecutive days should automatically promote them to Risk_Group 2B.

**Validates: Requirements 7.2**

### Property 10: Adherence Streak Demotion

*For any* patient in Risk_Group 2B, a single missed medication (medication_status != "Fully Adherent") or distressed sentiment (mood_sentiment = "Distressed") should immediately demote them to Risk_Group 2A and reset their adherence_streak to 0.

**Validates: Requirements 8.3, 8.4, 8.5**

### Property 11: Flush Protocol Execution

*For any* completed call, the Flush_Protocol should generate a Dual_Log with both narrative_summary and clinical_metrics, store it in DynamoDB, and delete all audio recordings and transcripts from S3 within 5 minutes.

**Validates: Requirements 5.2, 5.3, 5.4, 11.7, 11.8**

### Property 12: Escalation Real-Time Notification

*For any* escalation event created in DynamoDB, all connected nurses should receive a real-time notification via AWS AppSync GraphQL subscription within 2 seconds.

**Validates: Requirements 14.2, 14.3**

### Property 13: Urgent Symptom Escalation

*For any* call where the patient mentions urgent symptoms (chest pain, dizziness, shortness of breath), the AI_Call_System should immediately set escalation_flag to true and create an escalation record with urgency = "HIGH".

**Validates: Requirements 15.6, 14.4**

### Property 14: CSV Patient ID Handling

*For any* CSV row processed by Bulk_Import_Lambda, if the Patient_ID column exists and has a non-empty value, that value should be used; otherwise, a new ID should be auto-generated using the counter.

**Validates: Requirements 17.5, 17.6, 19.7**

### Property 15: Doctor Patient Scope

*For any* doctor viewing or editing patient data, they should only be able to access patients where the patient's doctor_id matches their own doctor_id, unless they are in AdminGroup.

**Validates: Requirements 9.8, 21.7, 21.8**

### Property 16: Import Error Reporting

*For any* bulk import operation, all validation errors should be logged with row number and error message, and the final email report should accurately reflect the count of successful and failed imports.

**Validates: Requirements 17.8, 17.9, 17.10**

### Property 17: Amazon Connect CCP Visibility

*For any* nurse logged into the Nurse Dashboard, the Amazon Connect CCP should be visible and functional at all times, allowing call control operations.

**Validates: Requirements 18.1, 18.10, 20.12**

### Property 18: Custom ID Uniqueness Check

*For any* doctor creating a patient with a custom ID, the system should query DynamoDB for existing Patient_IDs and reject the creation if a duplicate is found, displaying an appropriate error message.

**Validates: Requirements 19.4, 19.5, 19.6**

### Property 19: Prescription Dynamic Form

*For any* doctor creating a prescription, the Prescription Writer should allow adding multiple medications dynamically, with each medication having fields for name, dosage, frequency, and duration.

**Validates: Requirements 20.9**

### Property 20: Live Patient View Tabs

*For any* nurse viewing a patient, the Live Patient View should provide three tabs (History, Meds, Notes) with appropriate content: History shows call logs in Dual_Log format, Meds shows prescriptions, and Notes shows nurse notes with write access.

**Validates: Requirements 20.14, 20.15, 20.16, 20.17**


## Error Handling

### 1. Bulk Import Error Handling

**Scenario**: Invalid data in CSV file during bulk import

**Handling Strategy**:
- Continue processing remaining rows after encountering an error
- Log each error with row number, error message, and problematic data
- Accumulate all errors in memory during processing
- Include error details in final email report
- Store import record in DynamoDB with error array

**Error Types**:
- Missing required fields (name, phone, doctor_id)
- Invalid phone number format
- Duplicate Patient_ID
- Invalid Risk_Group value
- Invalid language code

**Example Error Log**:
```python
errors = []
for row_num, row in enumerate(csv_rows):
    try:
        validate_and_create_patient(row)
    except ValidationError as e:
        errors.append({
            'row_number': row_num,
            'error': str(e),
            'data': row
        })
        continue  # Continue processing next row
```

### 2. Patient Lookup Failure (Screen Pop)

**Scenario**: Incoming call with Caller ID not found in system

**Handling Strategy**:
- Return `{"found": false}` from Lookup_Patient Lambda
- Display notification in Nurse Dashboard: "Caller ID not found. Please search manually."
- Provide manual search interface with name/ID search
- Log lookup attempt for audit trail
- Do not block call handling

**Fallback UI**:
```typescript
if (!lookupResult.found) {
  showNotification({
    type: 'warning',
    message: 'Caller ID not recognized. Use search to find patient.',
    action: 'Open Search'
  });
}
```

### 3. Duplicate Patient ID Error

**Scenario**: Doctor attempts to create patient with existing custom ID

**Handling Strategy**:
- Check DynamoDB for existing Patient_ID before creation
- If duplicate found, return error response with status 409 (Conflict)
- Display user-friendly error message in modal
- Keep modal open with form data preserved
- Suggest alternative ID or auto-generation

**API Response**:
```json
{
  "error": "DUPLICATE_ID",
  "message": "Patient ID 'ICU-BED-5' already exists. Please choose a different ID.",
  "existing_patient": {
    "patient_id": "ICU-BED-5",
    "name": "Existing Patient Name"
  }
}
```

### 4. Counter Increment Race Condition

**Scenario**: Multiple concurrent patient creations attempting to increment counter

**Handling Strategy**:
- Use DynamoDB atomic update with ADD operation
- DynamoDB guarantees atomicity at item level
- Each concurrent request gets unique incremented value
- No additional locking mechanism needed
- Retry on ProvisionedThroughputExceededException

**Implementation**:
```python
try:
    response = dynamodb.update_item(
        TableName='DignityVoice',
        Key={'PK': 'METADATA', 'SK': 'PATIENT_COUNTER'},
        UpdateExpression='ADD counter_value :inc',
        ExpressionAttributeValues={':inc': 1},
        ReturnValues='UPDATED_NEW'
    )
    return f"P-{response['Attributes']['counter_value']}"
except ClientError as e:
    if e.response['Error']['Code'] == 'ProvisionedThroughputExceededException':
        time.sleep(0.1)
        return get_next_patient_id()  # Retry
    raise
```

### 5. Cognito User Creation Failure

**Scenario**: Admin attempts to create staff account but Cognito operation fails

**Handling Strategy**:
- Wrap Cognito API calls in try-catch
- Handle specific error codes (UsernameExistsException, InvalidParameterException)
- Display appropriate error message to admin
- Do not create partial records in DynamoDB
- Log error for admin review

**Error Handling**:
```python
try:
    cognito.admin_create_user(
        UserPoolId=user_pool_id,
        Username=email,
        UserAttributes=[
            {'Name': 'email', 'Value': email},
            {'Name': 'name', 'Value': name}
        ],
        DesiredDeliveryMediums=['EMAIL']
    )
    
    cognito.admin_add_user_to_group(
        UserPoolId=user_pool_id,
        Username=email,
        GroupName=group_name
    )
except cognito.exceptions.UsernameExistsException:
    return {'error': 'USER_EXISTS', 'message': 'User with this email already exists'}
except Exception as e:
    logger.error(f'Cognito error: {str(e)}')
    return {'error': 'COGNITO_ERROR', 'message': 'Failed to create user account'}
```

### 6. Unauthorized Route Access

**Scenario**: User attempts to access route without proper Cognito group membership

**Handling Strategy**:
- Implement route guard middleware in Next.js
- Extract Cognito groups from JWT token
- Check if user's groups include required group for route
- Redirect to /unauthorized page with clear message
- Log unauthorized access attempts

**Route Guard**:
```typescript
export function withAuth(Component, allowedGroups) {
  return function AuthenticatedComponent(props) {
    const { user, groups } = useAuth();
    
    if (!user) {
      return <Navigate to="/login" />;
    }
    
    const hasAccess = groups.some(g => allowedGroups.includes(g));
    if (!hasAccess) {
      return (
        <UnauthorizedPage 
          message="You do not have permission to access this page."
          requiredRole={allowedGroups.join(' or ')}
        />
      );
    }
    
    return <Component {...props} />;
  };
}
```

### 7. S3 Upload Failure

**Scenario**: Admin's CSV file upload to S3 fails due to network or permission issues

**Handling Strategy**:
- Display upload progress bar with percentage
- Catch upload errors and display user-friendly message
- Provide retry button
- Log error details for debugging
- Do not trigger Lambda if upload incomplete

**Upload Error Handling**:
```typescript
try {
  await Storage.put(`imports/${file.name}`, file, {
    progressCallback: (progress) => {
      setUploadProgress((progress.loaded / progress.total) * 100);
    }
  });
  showSuccess('File uploaded successfully');
} catch (error) {
  console.error('Upload error:', error);
  showError('Upload failed. Please check your connection and try again.');
  setUploadProgress(0);
}
```

### 8. AppSync Subscription Connection Loss

**Scenario**: Nurse's WebSocket connection to AppSync drops, missing escalation notifications

**Handling Strategy**:
- Implement automatic reconnection with exponential backoff
- Display connection status indicator in UI
- On reconnection, fetch missed escalations from DynamoDB
- Show notification: "Connection restored. Checking for missed alerts."
- Maintain local queue of unacknowledged escalations

**Reconnection Logic**:
```typescript
useEffect(() => {
  const subscription = API.graphql(
    graphqlOperation(onCreateEscalation)
  ).subscribe({
    next: (data) => handleNewEscalation(data),
    error: (error) => {
      console.error('Subscription error:', error);
      setConnectionStatus('disconnected');
      
      // Retry with exponential backoff
      setTimeout(() => {
        setConnectionStatus('reconnecting');
        // Re-subscribe
      }, retryDelay);
    }
  });
  
  return () => subscription.unsubscribe();
}, []);
```

### 9. Lambda Timeout During Bulk Import

**Scenario**: Large CSV file (10,000+ rows) exceeds Lambda 15-minute timeout

**Handling Strategy**:
- Process CSV in batches of 500 rows
- Use Step Functions to orchestrate batch processing
- Each Lambda invocation processes one batch
- Store progress in DynamoDB
- Send partial success email if timeout occurs
- Provide resume capability for failed batches

**Batch Processing**:
```python
def process_batch(batch_number, rows):
    for row in rows:
        try:
            create_patient(row)
        except Exception as e:
            log_error(batch_number, row, e)
    
    update_progress(batch_number, 'COMPLETED')
```

### 10. Bedrock API Rate Limiting

**Scenario**: High call volume causes Bedrock API throttling

**Handling Strategy**:
- Implement exponential backoff retry logic
- Queue requests in SQS if rate limit exceeded
- Display "Processing..." message to patient during retry
- Set maximum retry attempts (3)
- Escalate to nurse if AI unavailable after retries
- Monitor CloudWatch metrics for rate limit alarms

**Retry Logic**:
```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type(ThrottlingException)
)
def call_bedrock(prompt):
    return bedrock.invoke_model(
        modelId='anthropic.claude-3-haiku',
        body=json.dumps({'prompt': prompt})
    )
```


## Testing Strategy

### Dual Testing Approach

The system will employ both unit testing and property-based testing to ensure comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs
- Both approaches are complementary and necessary for comprehensive coverage

Unit tests focus on concrete scenarios and integration points, while property tests validate general correctness across randomized inputs. Together, they provide confidence in both specific behaviors and universal invariants.

### Property-Based Testing Configuration

**Library Selection**:
- **Python (Lambda functions)**: Hypothesis
- **TypeScript (Frontend)**: fast-check

**Configuration**:
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `# Feature: ai-voice-medication-system, Property {number}: {property_text}`

**Example Property Test (Python)**:
```python
from hypothesis import given, strategies as st
import pytest

# Feature: ai-voice-medication-system, Property 1: Patient ID Uniqueness
@given(
    patient_data=st.lists(
        st.fixed_dictionaries({
            'name': st.text(min_size=1),
            'phone': st.from_regex(r'\+91[0-9]{10}'),
            'age': st.integers(min_value=18, max_value=120)
        }),
        min_size=2,
        max_size=100
    )
)
def test_patient_id_uniqueness(patient_data):
    """For any set of patient creation operations, all assigned Patient_IDs should be unique."""
    created_ids = []
    
    for patient in patient_data:
        patient_id = create_patient(patient)
        assert patient_id not in created_ids, f"Duplicate ID generated: {patient_id}"
        created_ids.append(patient_id)
```

**Example Property Test (TypeScript)**:
```typescript
import fc from 'fast-check';

// Feature: ai-voice-medication-system, Property 5: Role-Based Route Access
describe('Role-Based Route Access', () => {
  it('should allow access only to users with correct group membership', () => {
    fc.assert(
      fc.property(
        fc.record({
          route: fc.constantFrom('/admin', '/doctor', '/nurse'),
          userGroups: fc.array(fc.constantFrom('AdminGroup', 'DoctorGroup', 'NurseGroup'))
        }),
        ({ route, userGroups }) => {
          const routeGroupMap = {
            '/admin': ['AdminGroup'],
            '/doctor': ['DoctorGroup', 'AdminGroup'],
            '/nurse': ['NurseGroup', 'AdminGroup']
          };
          
          const hasAccess = checkRouteAccess(route, userGroups);
          const shouldHaveAccess = routeGroupMap[route].some(g => userGroups.includes(g));
          
          expect(hasAccess).toBe(shouldHaveAccess);
        }
      ),
      { numRuns: 100 }
    );
  });
});
```

### Unit Testing Strategy

**Lambda Functions**:
- Test each Lambda function with mocked AWS service calls
- Use moto library for mocking AWS services (DynamoDB, S3, SES)
- Test error handling paths explicitly
- Verify correct DynamoDB query patterns
- Test atomic counter increment logic

**Example Unit Test**:
```python
import boto3
from moto import mock_dynamodb
import pytest

@mock_dynamodb
def test_bulk_import_handles_missing_patient_id():
    """Test that bulk import auto-generates ID when Patient_ID column is empty."""
    # Setup
    dynamodb = boto3.resource('dynamodb', region_name='ap-south-1')
    table = create_test_table(dynamodb)
    
    # Test data with missing Patient_ID
    csv_data = [
        {'Name': 'John Doe', 'Phone': '+919876543210', 'Age': 60},
        {'Patient_ID': '', 'Name': 'Jane Smith', 'Phone': '+919876543211', 'Age': 55}
    ]
    
    # Execute
    result = process_bulk_import(csv_data)
    
    # Verify
    assert result['successful_imports'] == 2
    assert result['failed_imports'] == 0
    
    # Check that auto-generated IDs were assigned
    patients = table.scan()['Items']
    assert all(p['patient_id'].startswith('P-') for p in patients)
```

**Frontend Components**:
- Use React Testing Library for component tests
- Test user interactions (clicks, form submissions)
- Mock API calls with MSW (Mock Service Worker)
- Test conditional rendering based on user roles
- Verify navigation and routing logic

**Example Component Test**:
```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { AddPatientModal } from './AddPatientModal';

describe('AddPatientModal', () => {
  it('should show custom ID input when custom mode is selected', () => {
    render(<AddPatientModal isOpen={true} onClose={() => {}} />);
    
    const customRadio = screen.getByLabelText('Custom ID');
    fireEvent.click(customRadio);
    
    const customInput = screen.getByPlaceholderText('e.g., ICU-BED-5');
    expect(customInput).toBeInTheDocument();
  });
  
  it('should display error when duplicate ID is submitted', async () => {
    // Mock API to return duplicate error
    server.use(
      rest.post('/api/doctor/patients', (req, res, ctx) => {
        return res(ctx.status(409), ctx.json({ error: 'DUPLICATE_ID' }));
      })
    );
    
    render(<AddPatientModal isOpen={true} onClose={() => {}} />);
    
    // Fill form and submit
    fireEvent.change(screen.getByPlaceholderText('Patient Name'), {
      target: { value: 'Test Patient' }
    });
    fireEvent.click(screen.getByText('Create Patient'));
    
    // Verify error message
    await waitFor(() => {
      expect(screen.getByText(/already exists/i)).toBeInTheDocument();
    });
  });
});
```

### Integration Testing

**API Gateway + Lambda Integration**:
- Test complete request/response flow
- Verify JWT token validation
- Test CORS configuration
- Verify error responses match API contract

**Amazon Connect + Lambda Integration**:
- Test Contact Flow invocation of Lambda
- Verify Lambda response format for Connect
- Test escalation transfer logic
- Verify call attribute passing

**S3 + Lambda Trigger Integration**:
- Test S3 event notification delivery
- Verify Lambda processes correct S3 object
- Test file parsing with various CSV formats
- Verify cleanup of processed files

### End-to-End Testing

**Critical User Flows**:

1. **Admin Bulk Import Flow**:
   - Upload CSV file
   - Verify Lambda processing
   - Check DynamoDB records created
   - Verify email report sent

2. **Nurse Screen Pop Flow**:
   - Simulate incoming call with Caller ID
   - Verify Lookup_Patient invocation
   - Check automatic navigation to patient page
   - Verify patient data displayed

3. **Doctor Patient Creation Flow**:
   - Create patient with auto-generated ID
   - Verify counter increment
   - Create patient with custom ID
   - Verify uniqueness check

4. **Role-Based Access Flow**:
   - Login as each role (Admin, Doctor, Nurse)
   - Attempt to access all routes
   - Verify appropriate access/denial
   - Check redirect behavior

### Performance Testing

**Load Testing Scenarios**:

1. **Concurrent Patient Creation**:
   - Simulate 100 concurrent patient creations
   - Verify no duplicate IDs generated
   - Measure counter increment performance

2. **Bulk Import Large Files**:
   - Test with 10,000 row CSV
   - Measure processing time
   - Verify memory usage stays within Lambda limits

3. **Screen Pop Response Time**:
   - Measure Lookup_Patient Lambda latency
   - Target: < 500ms for 95th percentile
   - Test with 1000+ patient records

4. **AppSync Subscription Scalability**:
   - Connect 50 concurrent nurse clients
   - Send escalation notifications
   - Verify all clients receive within 2 seconds

### Security Testing

**Authentication Tests**:
- Verify JWT token validation
- Test expired token handling
- Verify refresh token flow
- Test invalid token rejection

**Authorization Tests**:
- Verify group-based access control
- Test cross-role data access attempts
- Verify patient data scoping by doctor_id
- Test admin override permissions

**Data Privacy Tests**:
- Verify Flush_Protocol deletes audio files
- Test S3 lifecycle policy enforcement
- Verify no PII in CloudWatch logs
- Test encryption at rest and in transit

### Monitoring and Observability

**CloudWatch Metrics**:
- Lambda invocation counts and errors
- DynamoDB read/write capacity usage
- S3 bucket object counts
- Bedrock API call counts and latency

**CloudWatch Alarms**:
- Lambda error rate > 5%
- DynamoDB throttling events
- S3 upload failures
- Bedrock rate limit exceeded

**X-Ray Tracing**:
- Enable for all Lambda functions
- Trace complete request flows
- Identify performance bottlenecks
- Monitor external service calls

**Custom Metrics**:
- Patient creation rate
- Bulk import success rate
- Screen pop response time
- Escalation notification latency

