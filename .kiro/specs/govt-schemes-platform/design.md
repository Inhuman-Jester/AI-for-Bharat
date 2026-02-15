# Design Document: Government Schemes Platform

## Overview

The Government Schemes Platform is a voice-first, multilingual system that bridges the gap between underprivileged rural Indians and government assistance programs. The platform addresses critical barriers including limited literacy, language diversity, poor internet connectivity, and complex bureaucratic processes.

### Core Design Principles

1. **Voice-First Architecture**: All interactions prioritize voice over text, with voice as the primary input/output modality
2. **Multilingual by Default**: Support for 10 Indian languages (Hindi, Tamil, Telugu, Bengali, Marathi, Gujarati, Kannada, Malayalam, Punjabi, Odia) at every layer
3. **Low Bandwidth Optimization**: Designed to function on connections as slow as 64 kbps
4. **Progressive Enhancement**: WhatsApp as the primary channel with web dashboard as an enhanced interface
5. **AI-Augmented Assistance**: RAG-based scheme matching and conversational AI for form filling
6. **Privacy-First**: End-to-end encryption and minimal data retention

### Technology Stack

- **WhatsApp Business API**: Primary communication channel
- **Speech-to-Text**: Multilingual ASR (Automatic Speech Recognition) service supporting Indian languages
- **Text-to-Speech**: Multilingual TTS service for voice response generation
- **LLM**: Large Language Model for natural language understanding and conversation management
- **RAG System**: Vector database + embedding model for semantic scheme matching
- **Database**: Hybrid approach - PostgreSQL for structured data, Vector DB for embeddings
- **Backend**: Python-based microservices architecture
- **Web Frontend**: React-based responsive web application
- **Message Queue**: For asynchronous processing and reliability
- **Object Storage**: For voice note storage with automatic cleanup

## Architecture

### System Architecture

The platform follows a microservices architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interfaces                          │
├──────────────────────────┬──────────────────────────────────────┤
│   WhatsApp Interface     │      Web Dashboard                   │
│   (Primary Channel)      │      (Enhanced Interface)            │
└──────────────┬───────────┴──────────────┬───────────────────────┘
               │                          │
               └──────────┬───────────────┘
                          │
               ┌──────────▼──────────┐
               │   API Gateway       │
               │   - Authentication  │
               │   - Rate Limiting   │
               │   - Load Balancing  │
               └──────────┬──────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────▼────────┐ ┌─────▼──────┐ ┌───────▼────────┐
│ Voice Service  │ │ User       │ │ Scheme         │
│ - ASR          │ │ Service    │ │ Service        │
│ - TTS          │ │ - Profiles │ │ - RAG Matching │
│ - Compression  │ │ - Auth     │ │ - Retrieval    │
└───────┬────────┘ └─────┬──────┘ └───────┬────────┘
        │                │                 │
        │         ┌──────▼──────┐          │
        │         │ Conversation│          │
        │         │ Manager     │          │
        │         │ - State     │          │
        │         │ - Context   │          │
        │         └──────┬──────┘          │
        │                │                 │
┌───────▼────────┐ ┌─────▼──────┐ ┌───────▼────────┐
│ Form Agent     │ │ Document   │ │ Message Queue  │
│ - AI Assist    │ │ Education  │ │ - Async Tasks  │
│ - Validation   │ │ Module     │ │ - Retry Logic  │
└───────┬────────┘ └─────┬──────┘ └───────┬────────┘
        │                │                 │
        └────────────────┼─────────────────┘
                         │
              ┌──────────▼──────────┐
              │   Data Layer        │
              ├─────────────────────┤
              │ PostgreSQL          │
              │ Vector DB (Schemes) │
              │ Object Storage      │
              └─────────────────────┘
```

### Component Interaction Flow

**Voice Note Processing Flow:**
1. User sends voice note via WhatsApp or Web Dashboard
2. WhatsApp Interface/Web Dashboard receives and forwards to API Gateway
3. API Gateway authenticates and routes to Voice Service
4. Voice Service compresses, stores, and transcribes voice note
5. Conversation Manager extracts information and updates User Profile
6. Scheme Service performs RAG-based matching
7. Conversation Manager generates response
8. Voice Service converts response to speech
9. WhatsApp Interface/Web Dashboard delivers response to user

**Web Dashboard Authentication Flow:**
1. User enters phone number on web dashboard
2. User Service generates and sends OTP via WhatsApp
3. User enters OTP on web dashboard
4. User Service validates OTP and issues JWT token
5. Web Dashboard uses JWT for subsequent API calls

## Components and Interfaces

### 1. WhatsApp Interface Service

**Responsibility**: Manages all WhatsApp Business API interactions

**Key Functions**:
- `receive_message(webhook_data)`: Handles incoming WhatsApp webhooks
- `send_text_message(phone_number, message, language)`: Sends text messages
- `send_voice_message(phone_number, audio_url, language)`: Sends voice messages
- `send_template_message(phone_number, template_id, params)`: Sends WhatsApp templates
- `download_media(media_id)`: Downloads voice notes from WhatsApp servers
- `acknowledge_receipt(phone_number)`: Sends acknowledgment within 30 seconds

**Interface**:
```python
class WhatsAppInterface:
    def receive_message(self, webhook_data: dict) -> Message:
        """Process incoming WhatsApp webhook and extract message"""
        
    def send_text_message(self, phone_number: str, message: str, language: str) -> bool:
        """Send text message to user"""
        
    def send_voice_message(self, phone_number: str, audio_url: str, language: str) -> bool:
        """Send voice message to user"""
        
    def download_media(self, media_id: str) -> bytes:
        """Download voice note from WhatsApp"""
        
    def acknowledge_receipt(self, phone_number: str) -> bool:
        """Send acknowledgment within 30 seconds"""
```

### 2. Voice Service

**Responsibility**: Handles all voice processing including transcription, synthesis, and compression

**Key Functions**:
- `transcribe_voice(audio_data, language)`: Converts voice to text using ASR
- `synthesize_speech(text, language)`: Converts text to voice using TTS
- `compress_audio(audio_data, target_size)`: Compresses audio for low bandwidth
- `detect_language(audio_data)`: Identifies spoken language
- `segment_long_audio(audio_data, max_duration)`: Splits audio exceeding 5 minutes

**Interface**:
```python
class VoiceService:
    def transcribe_voice(self, audio_data: bytes, language: str) -> TranscriptionResult:
        """Transcribe voice note to text within 60 seconds"""
        
    def synthesize_speech(self, text: str, language: str) -> bytes:
        """Convert text to speech in specified language"""
        
    def compress_audio(self, audio_data: bytes, max_size_kb: int = 500) -> bytes:
        """Compress audio while maintaining clarity"""
        
    def detect_language(self, audio_data: bytes) -> str:
        """Detect language from audio"""
        
    def segment_long_audio(self, audio_data: bytes, max_duration_sec: int = 300) -> list[bytes]:
        """Split audio into segments if exceeds max duration"""
```

### 3. User Service

**Responsibility**: Manages user profiles, authentication, and data privacy

**Key Functions**:
- `create_profile(extracted_info)`: Creates new user profile from voice note data
- `update_profile(user_id, new_info)`: Updates existing profile
- `get_profile(user_id)`: Retrieves user profile
- `merge_profile_updates(user_id, new_info)`: Merges new information without creating duplicates
- `delete_user_data(user_id)`: Removes all user data for privacy compliance
- `generate_otp(phone_number)`: Generates OTP for web authentication
- `validate_otp(phone_number, otp)`: Validates OTP and issues JWT token
- `encrypt_profile_data(data)`: Encrypts sensitive user information

**Interface**:
```python
class UserService:
    def create_profile(self, extracted_info: dict) -> UserProfile:
        """Create user profile from extracted information"""
        
    def update_profile(self, user_id: str, new_info: dict) -> UserProfile:
        """Update existing user profile"""
        
    def get_profile(self, user_id: str) -> UserProfile:
        """Retrieve user profile"""
        
    def merge_profile_updates(self, user_id: str, new_info: dict) -> UserProfile:
        """Merge new information into existing profile"""
        
    def delete_user_data(self, user_id: str) -> bool:
        """Delete all user data for privacy compliance"""
        
    def generate_otp(self, phone_number: str) -> str:
        """Generate OTP for web authentication"""
        
    def validate_otp(self, phone_number: str, otp: str) -> str:
        """Validate OTP and return JWT token"""
```

### 4. Conversation Manager

**Responsibility**: Maintains conversation state, context, and orchestrates multi-turn interactions

**Key Functions**:
- `extract_information(transcript, user_id)`: Extracts structured data from transcription
- `determine_intent(transcript, conversation_history)`: Identifies user intent
- `manage_conversation_state(user_id, new_message)`: Updates conversation state
- `generate_clarifying_questions(incomplete_info)`: Creates questions for missing information
- `generate_response(user_id, intent, context)`: Generates contextual responses
- `provide_menu_options(user_id, available_actions)`: Creates voice menu

**Interface**:
```python
class ConversationManager:
    def extract_information(self, transcript: str, user_id: str) -> dict:
        """Extract structured information from transcript"""
        
    def determine_intent(self, transcript: str, conversation_history: list) -> Intent:
        """Determine user intent from message"""
        
    def manage_conversation_state(self, user_id: str, new_message: Message) -> ConversationState:
        """Update and maintain conversation state"""
        
    def generate_clarifying_questions(self, incomplete_info: dict) -> str:
        """Generate questions for missing information"""
        
    def generate_response(self, user_id: str, intent: Intent, context: dict) -> str:
        """Generate contextual response"""
```

### 5. Scheme Service (RAG-based)

**Responsibility**: Manages scheme database and performs semantic matching using RAG

**Key Functions**:
- `embed_scheme(scheme_data)`: Creates vector embeddings for schemes
- `search_schemes(user_profile, top_k)`: Performs semantic search for relevant schemes
- `rank_schemes(schemes, user_profile)`: Ranks schemes by relevance score
- `filter_ineligible_schemes(schemes, user_profile)`: Removes clearly ineligible schemes
- `get_scheme_details(scheme_id)`: Retrieves detailed scheme information
- `update_scheme(scheme_id, new_data)`: Updates scheme information and re-embeds
- `add_scheme(scheme_data)`: Adds new scheme to database
- `deactivate_scheme(scheme_id)`: Marks scheme as inactive

**Interface**:
```python
class SchemeService:
    def embed_scheme(self, scheme_data: dict) -> np.ndarray:
        """Create vector embedding for scheme"""
        
    def search_schemes(self, user_profile: UserProfile, top_k: int = 5) -> list[Scheme]:
        """Search for relevant schemes using RAG"""
        
    def rank_schemes(self, schemes: list[Scheme], user_profile: UserProfile) -> list[tuple[Scheme, float]]:
        """Rank schemes by relevance score"""
        
    def filter_ineligible_schemes(self, schemes: list[Scheme], user_profile: UserProfile) -> list[Scheme]:
        """Filter out clearly ineligible schemes"""
        
    def get_scheme_details(self, scheme_id: str) -> Scheme:
        """Get detailed scheme information"""
```

### 6. Form Agent

**Responsibility**: Provides AI-assisted form filling with step-by-step guidance

**Key Functions**:
- `initialize_form(scheme_id, user_profile)`: Starts form filling process
- `prefill_form_fields(form, user_profile)`: Pre-fills fields from profile
- `get_next_field(form_state)`: Determines next field to fill
- `validate_field_response(field, response)`: Validates user input
- `generate_field_question(field, language)`: Creates voice question for field
- `generate_form_summary(completed_form)`: Creates summary for confirmation
- `submit_form(form_data)`: Submits completed form

**Interface**:
```python
class FormAgent:
    def initialize_form(self, scheme_id: str, user_profile: UserProfile) -> FormState:
        """Initialize form filling process"""
        
    def prefill_form_fields(self, form: Form, user_profile: UserProfile) -> Form:
        """Pre-fill form fields from user profile"""
        
    def get_next_field(self, form_state: FormState) -> FormField:
        """Get next field that needs user input"""
        
    def validate_field_response(self, field: FormField, response: str) -> ValidationResult:
        """Validate user response for field"""
        
    def generate_field_question(self, field: FormField, language: str) -> str:
        """Generate voice question for field"""
        
    def generate_form_summary(self, completed_form: Form) -> str:
        """Generate summary for user confirmation"""
```

### 7. Document Education Module

**Responsibility**: Provides guidance on mandatory documents

**Key Functions**:
- `get_document_info(document_type, language)`: Retrieves document information
- `explain_document_purpose(document_type, language)`: Explains why document is needed
- `provide_acquisition_steps(document_type, language)`: Provides step-by-step guidance
- `get_required_documents_for_scheme(scheme_id)`: Lists documents needed for scheme
- `generate_document_explanation_voice(document_type, language)`: Creates voice explanation

**Interface**:
```python
class DocumentEducationModule:
    def get_document_info(self, document_type: str, language: str) -> DocumentInfo:
        """Get information about document type"""
        
    def explain_document_purpose(self, document_type: str, language: str) -> str:
        """Explain document purpose and use cases"""
        
    def provide_acquisition_steps(self, document_type: str, language: str) -> list[str]:
        """Provide step-by-step acquisition process"""
        
    def get_required_documents_for_scheme(self, scheme_id: str) -> list[str]:
        """Get list of required documents for scheme"""
```

### 8. Web Dashboard Service

**Responsibility**: Provides web-based interface with voice recording capability

**Key Functions**:
- `authenticate_user(phone_number, otp)`: Authenticates user and returns session token
- `get_dashboard_data(user_id)`: Retrieves all dashboard data (profile, schemes, applications)
- `update_profile_web(user_id, form_data)`: Updates profile from web form
- `record_voice_note_web(user_id, audio_blob)`: Processes voice note from web
- `get_conversation_history(user_id)`: Retrieves conversation history
- `get_application_status(user_id)`: Retrieves application statuses
- `sync_data(user_id)`: Synchronizes data between WhatsApp and web interfaces

**Interface**:
```python
class WebDashboardService:
    def authenticate_user(self, phone_number: str, otp: str) -> AuthToken:
        """Authenticate user and return JWT token"""
        
    def get_dashboard_data(self, user_id: str) -> DashboardData:
        """Get all dashboard data for user"""
        
    def update_profile_web(self, user_id: str, form_data: dict) -> UserProfile:
        """Update profile from web form"""
        
    def record_voice_note_web(self, user_id: str, audio_blob: bytes) -> ProcessingResult:
        """Process voice note recorded on web"""
        
    def get_conversation_history(self, user_id: str) -> list[Message]:
        """Get conversation history"""
```

## Data Models

### UserProfile
```python
class UserProfile:
    user_id: str                    # Unique identifier (phone number hash)
    phone_number: str               # Encrypted phone number
    location: str                   # Village/District/State
    occupation: str                 # Current occupation
    problem_description: str        # Primary problem/need
    income_level: str               # Income bracket
    family_size: int                # Number of family members
    language_preference: str        # Preferred language code
    created_at: datetime            # Profile creation timestamp
    updated_at: datetime            # Last update timestamp
    consent_given: bool             # Data usage consent
    additional_info: dict           # Flexible field for extra information
```

### Scheme
```python
class Scheme:
    scheme_id: str                  # Unique scheme identifier
    name: str                       # Scheme name
    description: str                # Detailed description
    eligibility_criteria: dict      # Structured eligibility rules
    benefits: list[str]             # List of benefits
    application_process: list[str]  # Step-by-step process
    required_documents: list[str]   # Required document types
    application_deadline: datetime  # Deadline (if applicable)
    active: bool                    # Whether scheme is currently active
    embedding: np.ndarray           # Vector embedding for RAG
    version: int                    # Version number for history
    created_at: datetime
    updated_at: datetime
```

### Message
```python
class Message:
    message_id: str                 # Unique message identifier
    user_id: str                    # User identifier
    direction: str                  # 'inbound' or 'outbound'
    content_type: str               # 'voice', 'text', 'template'
    content: str                    # Text content or transcript
    audio_url: str                  # URL to audio file (if voice)
    language: str                   # Message language
    timestamp: datetime             # Message timestamp
    processed: bool                 # Whether message has been processed
    metadata: dict                  # Additional metadata
```

### ConversationState
```python
class ConversationState:
    user_id: str                    # User identifier
    current_intent: str             # Current conversation intent
    context: dict                   # Conversation context
    pending_questions: list[str]    # Questions awaiting response
    conversation_history: list[Message]  # Recent message history
    active_form: FormState          # Active form state (if any)
    last_interaction: datetime      # Last interaction timestamp
```

### FormState
```python
class FormState:
    form_id: str                    # Unique form identifier
    scheme_id: str                  # Associated scheme
    user_id: str                    # User filling the form
    fields: dict[str, FormField]    # Form fields
    completed_fields: dict[str, any]  # Completed field values
    current_field: str              # Current field being filled
    status: str                     # 'in_progress', 'completed', 'submitted'
    created_at: datetime
    updated_at: datetime
```

### FormField
```python
class FormField:
    field_id: str                   # Field identifier
    field_name: str                 # Display name
    field_type: str                 # 'text', 'number', 'date', 'choice'
    required: bool                  # Whether field is required
    validation_rules: dict          # Validation rules
    prefillable: bool               # Can be pre-filled from profile
    help_text: str                  # Help text for user
```

### TranscriptionResult
```python
class TranscriptionResult:
    transcript: str                 # Transcribed text
    language: str                   # Detected language
    confidence: float               # Confidence score (0-1)
    duration_seconds: float         # Audio duration
    processing_time_seconds: float  # Processing time
```

### DocumentInfo
```python
class DocumentInfo:
    document_type: str              # 'PAN', 'Aadhaar', 'Ration', etc.
    purpose: str                    # Why document is needed
    use_cases: list[str]            # Common use cases
    acquisition_steps: list[str]    # How to obtain
    required_for_schemes: list[str] # Schemes requiring this document
    multilingual_content: dict[str, str]  # Content in multiple languages
```

### DashboardData
```python
class DashboardData:
    user_profile: UserProfile       # User profile information
    matched_schemes: list[Scheme]   # Matched schemes with relevance scores
    applications: list[Application] # Application statuses
    conversation_history: list[Message]  # Recent conversations
    notifications: list[Notification]  # Pending notifications
```

### Application
```python
class Application:
    application_id: str             # Unique application identifier
    user_id: str                    # User identifier
    scheme_id: str                  # Applied scheme
    form_data: dict                 # Submitted form data
    status: str                     # 'draft', 'submitted', 'approved', 'rejected'
    submitted_at: datetime          # Submission timestamp
    updated_at: datetime            # Last update timestamp
```

## API Specifications

### REST API Endpoints

#### WhatsApp Webhook
```
POST /api/v1/whatsapp/webhook
Content-Type: application/json

Request Body: WhatsApp webhook payload
Response: 200 OK (acknowledgment)
```

#### Web Authentication
```
POST /api/v1/auth/request-otp
Content-Type: application/json

Request:
{
  "phone_number": "+91XXXXXXXXXX"
}

Response:
{
  "success": true,
  "message": "OTP sent via WhatsApp"
}
```

```
POST /api/v1/auth/verify-otp
Content-Type: application/json

Request:
{
  "phone_number": "+91XXXXXXXXXX",
  "otp": "123456"
}

Response:
{
  "success": true,
  "token": "jwt_token_here",
  "user_id": "user_id_here"
}
```

#### Dashboard Data
```
GET /api/v1/dashboard/{user_id}
Authorization: Bearer {jwt_token}

Response:
{
  "profile": {...},
  "matched_schemes": [...],
  "applications": [...],
  "conversation_history": [...]
}
```

#### Voice Note Submission (Web)
```
POST /api/v1/voice/submit
Authorization: Bearer {jwt_token}
Content-Type: multipart/form-data

Request:
- audio_file: audio blob
- user_id: string
- language: string (optional)

Response:
{
  "success": true,
  "message_id": "msg_id_here",
  "processing_status": "queued"
}
```

#### Profile Update (Web)
```
PUT /api/v1/profile/{user_id}
Authorization: Bearer {jwt_token}
Content-Type: application/json

Request:
{
  "location": "New Location",
  "occupation": "New Occupation",
  ...
}

Response:
{
  "success": true,
  "profile": {...}
}
```

#### Scheme Search
```
GET /api/v1/schemes/search?user_id={user_id}&top_k=5
Authorization: Bearer {jwt_token}

Response:
{
  "schemes": [
    {
      "scheme_id": "...",
      "name": "...",
      "relevance_score": 0.95,
      ...
    }
  ]
}
```

### Message Queue Events

#### VoiceNoteReceived
```json
{
  "event_type": "voice_note_received",
  "user_id": "user_id",
  "message_id": "msg_id",
  "audio_url": "storage_url",
  "language": "hi",
  "timestamp": "2024-01-01T00:00:00Z"
}
```

#### ProfileUpdated
```json
{
  "event_type": "profile_updated",
  "user_id": "user_id",
  "updated_fields": ["location", "occupation"],
  "timestamp": "2024-01-01T00:00:00Z"
}
```

#### SchemeMatched
```json
{
  "event_type": "scheme_matched",
  "user_id": "user_id",
  "schemes": ["scheme_id_1", "scheme_id_2"],
  "timestamp": "2024-01-01T00:00:00Z"
}
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I've identified the following redundancies and consolidations:

**Redundancies Eliminated:**
- Properties 1.1 and 1.2 (workflow triggering and acknowledgment) can be combined into a single property about voice note reception handling
- Properties 6.1, 6.2, and 6.3 (multilingual support) overlap significantly and can be consolidated into comprehensive language consistency properties
- Properties 3.3 and 3.4 (profile storage and duplicate prevention) can be combined into a single profile persistence property
- Properties 4.4 and 14.5 (scheme information completeness) are redundant and can be unified
- Properties 15.4 and 15.7 (web voice note processing parity) are essentially the same property
- Properties 2.1 and 13.1 (processing time requirements) can be consolidated into a single timing property

**Properties Combined:**
- Language support properties (2.2, 6.1) merged into comprehensive multilingual support property
- Profile update properties (3.4, 14.6, 14.7) merged into profile synchronization property
- Error handling properties (1.5, 12.1) merged into retry mechanism property
- Compression properties (7.1, 15.10) merged into audio compression property

### Voice Note Reception and Processing

**Property 1: Voice note acknowledgment timing**
*For any* voice note received via WhatsApp or web interface, the Platform should acknowledge receipt within 30 seconds and automatically trigger the processing workflow.
**Validates: Requirements 1.1, 1.2**

**Property 2: Long audio segmentation**
*For any* voice note exceeding 5 minutes in duration, the Platform should segment it into chunks of 5 minutes or less for processing.
**Validates: Requirements 1.4, 15.3**

**Property 3: Transcription timing**
*For any* voice note under normal load conditions, the Language_Processor should complete transcription within 60 seconds of receipt.
**Validates: Requirements 2.1, 13.1**

**Property 4: Multilingual transcription support**
*For any* voice note in Hindi, Tamil, Telugu, Bengali, Marathi, Gujarati, Kannada, Malayalam, Punjabi, or Odia, the Language_Processor should successfully transcribe it to text.
**Validates: Requirements 1.3, 2.2, 6.1**

**Property 5: Information extraction completeness**
*For any* transcribed voice note, the Platform should extract at least one of the key information fields (location, occupation, problem type, or income status) when present in the transcript.
**Validates: Requirements 2.3**

**Property 6: Problem categorization**
*For any* voice note transcript containing a problem description, the Platform should identify and assign a primary problem category.
**Validates: Requirements 2.4**

**Property 7: Retry mechanism for failures**
*For any* voice note that fails to upload or transcribe due to network or processing errors, the Platform should request the user to resend.
**Validates: Requirements 1.5, 12.1**

### User Profile Management

**Property 8: Automatic profile creation**
*For any* set of extracted key information from a voice note, the Platform should automatically create a User_Profile if one doesn't exist for that user.
**Validates: Requirements 3.1**

**Property 9: Profile field completeness**
*For any* created User_Profile, it should include fields for location, occupation, problem description, income level, family size, and language preference (even if some are initially empty).
**Validates: Requirements 3.2**

**Property 10: Profile persistence and uniqueness**
*For any* user sending multiple voice notes, the Platform should maintain exactly one User_Profile per user, updating the existing profile rather than creating duplicates.
**Validates: Requirements 3.3, 3.4**

**Property 11: Profile validation**
*For any* extracted information, the Platform should validate completeness before creating or updating a User_Profile, requesting clarification if insufficient information is present.
**Validates: Requirements 2.5, 3.5**

**Property 12: Profile synchronization across interfaces**
*For any* profile update made through WhatsApp or web interface, the changes should be immediately reflected in both interfaces.
**Validates: Requirements 14.6, 14.7**

### Scheme Matching and Retrieval

**Property 13: Automatic scheme search trigger**
*For any* User_Profile creation or update, the RAG_Model should automatically search the Scheme_Database for relevant schemes.
**Validates: Requirements 4.1**

**Property 14: Scheme ranking by relevance**
*For any* set of matching schemes, the RAG_Model should rank them by relevance score in descending order, with higher scores indicating better eligibility match.
**Validates: Requirements 4.2**

**Property 15: Top-k scheme retrieval**
*For any* scheme search, the Platform should return at most 5 schemes (or fewer if less than 5 relevant schemes exist).
**Validates: Requirements 4.3**

**Property 16: Scheme information completeness**
*For any* scheme displayed to a user (via WhatsApp or web), the Platform should include scheme name, benefits, eligibility criteria, application process, and required documents.
**Validates: Requirements 4.4, 14.5**

**Property 17: Ineligibility filtering**
*For any* scheme search, the Platform should filter out schemes for which the user is clearly ineligible based on profile data (e.g., age, location, income restrictions).
**Validates: Requirements 4.5**

### Form Filling and Assistance

**Property 18: Step-by-step form guidance**
*For any* scheme application form, the Form_Agent should guide users through fields sequentially, one field at a time.
**Validates: Requirements 5.1**

**Property 19: Form field pre-filling**
*For any* form field that can be pre-filled from the User_Profile, the Form_Agent should automatically populate it with the corresponding profile data.
**Validates: Requirements 5.2**

**Property 20: Language-specific form prompts**
*For any* form field requiring user input, the Form_Agent should ask for information via voice message in the user's preferred language.
**Validates: Requirements 5.3**

**Property 21: Form field validation**
*For any* user response to a form field, the Form_Agent should validate it against field requirements before accepting and moving to the next field.
**Validates: Requirements 5.4**

**Property 22: Form completion summary**
*For any* completed form, the Form_Agent should provide a summary of all filled fields for user confirmation before submission.
**Validates: Requirements 5.5**

**Property 23: Form progress preservation**
*For any* form submission failure, the Platform should save the user's progress and allow them to resume from where they left off.
**Validates: Requirements 12.3**

### Multilingual Support

**Property 24: Language consistency across interactions**
*For any* user with an identified language preference, all Platform interactions (voice and text) should use that language consistently.
**Validates: Requirements 6.2**

**Property 25: Text-to-speech conversion**
*For any* text response generated by the Platform, it should be converted to a voice message in the user's preferred language before delivery.
**Validates: Requirements 6.3**

**Property 26: Conversation context preservation**
*For any* multi-turn conversation, the Platform should maintain context from previous exchanges when generating responses.
**Validates: Requirements 6.4, 11.2**

**Property 27: Language confirmation on uncertainty**
*For any* voice note where language detection confidence is below a threshold, the Platform should ask the user to confirm their preferred language.
**Validates: Requirements 6.5**

### Low Bandwidth Optimization

**Property 28: Audio compression**
*For any* voice message (incoming or outgoing), the Platform should compress it to minimize data transfer while maintaining sufficient clarity for understanding.
**Validates: Requirements 7.1, 15.10**

**Property 29: Message queuing on poor connectivity**
*For any* message that fails to deliver due to network issues, the Platform should queue it for delivery when connection improves.
**Validates: Requirements 7.3**

**Property 30: Text fallback on voice failure**
*For any* voice message that fails to deliver due to bandwidth constraints, the Platform should fall back to sending a text-based version.
**Validates: Requirements 7.4**

**Property 31: Message size limits**
*For any* response message, the Platform should ensure voice messages don't exceed 500 KB and text messages don't exceed 10 KB.
**Validates: Requirements 7.5**

### Document Education

**Property 32: Language-specific document explanations**
*For any* document information request, the Document_Education_Module should provide explanations in the user's preferred language.
**Validates: Requirements 8.1**

**Property 33: Document type coverage**
*For any* request for information about PAN Card, Aadhaar Card, Ration Card, Income Certificate, Caste Certificate, or Bank Account documents, the Document_Education_Module should provide complete information.
**Validates: Requirements 8.2**

**Property 34: Document explanation completeness**
*For any* document type explanation, the Platform should include purpose, use cases, and step-by-step acquisition process.
**Validates: Requirements 8.3**

**Property 35: Voice-based document guidance**
*For any* document explanation, the Platform should provide it as a voice message (in addition to or instead of text).
**Validates: Requirements 8.4**

**Property 36: Proactive document notification**
*For any* scheme matched to a user that requires specific documents, the Platform should proactively inform the user and provide guidance on obtaining those documents.
**Validates: Requirements 8.5**

### Scheme Database Management

**Property 37: Scheme data completeness**
*For any* scheme stored in the Scheme_Database, it should include name, description, eligibility criteria, benefits, application process, and required documents.
**Validates: Requirements 9.1**

**Property 38: Scheme CRUD operations**
*For any* scheme, the Platform should support adding it to the database, updating its information, and removing it (marking as inactive).
**Validates: Requirements 9.2, 9.5**

**Property 39: Cascading scheme updates**
*For any* scheme whose details are updated, the Platform should refresh all affected user recommendations to reflect the new information.
**Validates: Requirements 9.3**

**Property 40: Scheme version history**
*For any* scheme update, the Scheme_Database should maintain a version history record of the previous state.
**Validates: Requirements 9.4**

### Privacy and Security

**Property 41: Data encryption**
*For any* User_Profile data, the Platform should encrypt it both at rest (in database) and in transit (during API calls).
**Validates: Requirements 10.1**

**Property 42: Voice note automatic deletion**
*For any* voice note stored by the Platform, it should be automatically deleted after 30 days unless the user explicitly requests retention.
**Validates: Requirements 10.3**

**Property 43: Complete data deletion**
*For any* user requesting profile deletion, the Platform should remove all associated data including profile, voice notes, conversation history, and applications.
**Validates: Requirements 10.5**

### Conversation Flow

**Property 44: Clarifying questions on unclear intent**
*For any* user message where intent cannot be determined with confidence, the Platform should ask clarifying questions via voice message.
**Validates: Requirements 11.3**

**Property 45: Voice menu for multiple options**
*For any* conversation state where multiple actions are available, the Platform should provide menu options via voice.
**Validates: Requirements 11.4**

**Property 46: Process completion summary**
*For any* completed process (profile creation, scheme matching, form submission), the Platform should summarize the outcome and provide next steps.
**Validates: Requirements 11.5**

### Error Handling

**Property 47: Error messages in user language**
*For any* error condition, the Platform should provide error messages in simple language via voice in the user's preferred language.
**Validates: Requirements 12.4**

**Property 48: Critical error notification**
*For any* critical system error, the Platform should notify administrators and inform the user of expected resolution time.
**Validates: Requirements 12.5**

**Property 49: Overload handling with queue notification**
*For any* request received when system load exceeds capacity, the Platform should queue the request and inform the user of expected wait time.
**Validates: Requirements 13.3**

### Web Dashboard

**Property 50: OTP-based authentication**
*For any* web dashboard access attempt, the Platform should authenticate the user using their phone number and a valid OTP sent via WhatsApp.
**Validates: Requirements 14.2**

**Property 51: Dashboard data completeness**
*For any* authenticated user accessing the web dashboard, it should display their profile information, matched schemes, application status, and conversation history.
**Validates: Requirements 14.3**

**Property 52: Web-WhatsApp language parity**
*For any* language supported in the WhatsApp interface, the Web_Dashboard should also support it.
**Validates: Requirements 14.4**

**Property 53: Web voice note processing parity**
*For any* voice note recorded and submitted through the web interface, the Platform should process it using the same workflow and provide the same acknowledgment as WhatsApp voice notes.
**Validates: Requirements 15.4, 15.7**

## Error Handling

### Error Categories and Responses

**1. Network and Connectivity Errors**
- **Voice note upload failure**: Queue message, request resend after 3 failed attempts
- **Poor connectivity**: Switch to text mode, queue voice messages for later delivery
- **Timeout errors**: Retry with exponential backoff (max 3 attempts)

**2. Processing Errors**
- **Transcription failure**: Request user to resend voice note with clearer audio
- **Language detection failure**: Ask user to confirm language preference
- **Information extraction failure**: Ask clarifying questions to gather missing information

**3. Data Validation Errors**
- **Incomplete profile data**: Request specific missing information
- **Invalid form field input**: Explain validation requirements and request correction
- **Duplicate profile detection**: Merge with existing profile, inform user

**4. Business Logic Errors**
- **No matching schemes found**: Inform user, suggest alternative resources or broader search
- **Scheme no longer active**: Notify user, suggest similar active schemes
- **Ineligible for selected scheme**: Explain eligibility criteria, suggest alternatives

**5. System Errors**
- **Database connection failure**: Queue operations, retry with backoff
- **External API failure** (WhatsApp, ASR, TTS): Use fallback mechanisms, notify administrators
- **Critical system failure**: Notify administrators immediately, inform user of expected resolution

### Error Recovery Strategies

**Graceful Degradation:**
- Voice → Text fallback when TTS fails
- Synchronous → Asynchronous processing when load is high
- Detailed → Simple responses when bandwidth is limited

**Retry Logic:**
- Network operations: 3 retries with exponential backoff
- Transcription: 2 retries, then request resend
- Form submission: Unlimited retries with saved progress

**User Communication:**
- All errors explained in simple language
- Voice delivery preferred, text as fallback
- Expected resolution time provided when known
- Alternative actions suggested when available

## Testing Strategy

### Dual Testing Approach

The platform requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests** focus on:
- Specific examples demonstrating correct behavior
- Edge cases (empty inputs, boundary values, special characters)
- Error conditions and exception handling
- Integration points between components
- Browser compatibility for web features

**Property-Based Tests** focus on:
- Universal properties that hold for all inputs
- Comprehensive input coverage through randomization
- Invariants that must be maintained
- Round-trip properties (e.g., encrypt/decrypt, serialize/deserialize)
- Metamorphic properties (e.g., compression reduces size while maintaining clarity)

### Property-Based Testing Configuration

**Testing Library**: Use `hypothesis` for Python components

**Test Configuration**:
- Minimum 100 iterations per property test (due to randomization)
- Each test must reference its design document property
- Tag format: `# Feature: govt-schemes-platform, Property {number}: {property_text}`

**Example Property Test Structure**:
```python
from hypothesis import given, strategies as st

# Feature: govt-schemes-platform, Property 10: Profile persistence and uniqueness
@given(
    user_id=st.text(min_size=10, max_size=20),
    voice_notes=st.lists(st.dictionaries(...), min_size=2, max_size=5)
)
def test_profile_uniqueness(user_id, voice_notes):
    """For any user sending multiple voice notes, 
    only one profile should exist"""
    for note in voice_notes:
        process_voice_note(user_id, note)
    
    profiles = get_profiles_for_user(user_id)
    assert len(profiles) == 1
```

### Test Coverage Requirements

**Component-Level Testing:**
- Voice Service: Transcription accuracy, TTS quality, compression ratios
- User Service: Profile CRUD, authentication, encryption
- Conversation Manager: Intent detection, context preservation, response generation
- Scheme Service: RAG matching accuracy, ranking correctness, filtering
- Form Agent: Pre-filling, validation, step-by-step guidance
- Document Education: Content completeness, language support
- Web Dashboard: Authentication flow, data synchronization, voice recording

**Integration Testing:**
- End-to-end voice note processing flow
- WhatsApp webhook handling
- Web dashboard authentication and data access
- Cross-interface synchronization (WhatsApp ↔ Web)
- Form submission workflow
- Error recovery scenarios

**Performance Testing:**
- Voice note processing time under various loads
- Concurrent user handling (target: 1000 concurrent users)
- Database query performance
- API response times
- Bandwidth optimization effectiveness

**Security Testing:**
- Encryption verification (at rest and in transit)
- Authentication and authorization
- OTP generation and validation
- Data deletion completeness
- Input sanitization and validation

### Testing Data Requirements

**Multilingual Test Data:**
- Voice samples in all 10 supported languages
- Transcripts with various accents and dialects
- Edge cases: code-switching, background noise, unclear speech

**User Profile Variations:**
- Different locations (rural, urban, various states)
- Various occupations and income levels
- Different family sizes and problem types
- Complete and incomplete profiles

**Scheme Database:**
- Active and inactive schemes
- Various eligibility criteria combinations
- Different benefit types and application processes
- Schemes with overlapping eligibility

**Network Conditions:**
- Simulated low bandwidth (64 kbps)
- Intermittent connectivity
- High latency scenarios
- Connection drops during operations

### Continuous Testing

**Automated Test Execution:**
- Run unit tests on every commit
- Run property tests on every pull request
- Run integration tests nightly
- Run performance tests weekly

**Monitoring and Alerting:**
- Track test failure rates
- Monitor property test counterexamples
- Alert on performance degradation
- Track error rates by category

## Implementation Notes

### Technology Recommendations

**Speech Processing:**
- ASR: Google Cloud Speech-to-Text or Azure Speech Services (both support Indian languages)
- TTS: Google Cloud Text-to-Speech or Azure TTS
- Audio compression: Opus codec (excellent quality at low bitrates)

**RAG System:**
- Vector Database: Pinecone, Weaviate, or Qdrant
- Embedding Model: multilingual-e5-large or sentence-transformers
- LLM: GPT-4 or Claude for conversation management and form assistance

**Backend:**
- Framework: FastAPI (Python) for high performance async APIs
- Message Queue: Redis or RabbitMQ for reliability
- Database: PostgreSQL with pgvector extension for hybrid storage
- Caching: Redis for session state and conversation context

**Frontend:**
- Framework: React with TypeScript
- UI Library: Material-UI or Chakra UI for responsive design
- Audio Recording: RecordRTC or MediaRecorder API
- State Management: Redux or Zustand

**Infrastructure:**
- Cloud Provider: AWS, GCP, or Azure
- Container Orchestration: Kubernetes for scalability
- Object Storage: S3, GCS, or Azure Blob for voice notes
- CDN: CloudFront or Cloudflare for web dashboard

### Scalability Considerations

**Horizontal Scaling:**
- Stateless API services for easy replication
- Message queue for asynchronous processing
- Database read replicas for query performance
- Caching layer to reduce database load

**Performance Optimization:**
- Voice note processing: Parallel processing of segments
- Scheme matching: Pre-computed embeddings, cached results
- Web dashboard: Lazy loading, pagination, data prefetching
- API: Rate limiting, request batching, connection pooling

**Cost Optimization:**
- Voice note storage: Automatic cleanup after 30 days
- Transcription: Batch processing during off-peak hours
- Caching: Reduce API calls to external services
- Compression: Minimize bandwidth costs

### Security Best Practices

**Data Protection:**
- Encryption: AES-256 for data at rest, TLS 1.3 for data in transit
- Key Management: Use cloud provider's key management service
- Access Control: Role-based access control (RBAC)
- Audit Logging: Log all data access and modifications

**Authentication:**
- OTP: 6-digit codes with 5-minute expiry
- JWT: Short-lived tokens (1 hour) with refresh mechanism
- Rate Limiting: Prevent brute force attacks on OTP validation
- Session Management: Secure session storage, automatic timeout

**Privacy Compliance:**
- Data Minimization: Collect only necessary information
- User Consent: Explicit consent for data collection and usage
- Right to Deletion: Implement complete data deletion
- Data Portability: Allow users to export their data

### Monitoring and Observability

**Metrics to Track:**
- Voice note processing time (p50, p95, p99)
- Transcription accuracy rate
- Scheme matching relevance scores
- Form completion rates
- Error rates by category
- User engagement metrics

**Logging:**
- Structured logging with correlation IDs
- Log levels: DEBUG, INFO, WARNING, ERROR, CRITICAL
- Sensitive data masking in logs
- Centralized log aggregation

**Alerting:**
- Processing time exceeds thresholds
- Error rate spikes
- System resource utilization
- External API failures
- Security incidents

### Deployment Strategy

**Phased Rollout:**
1. **Phase 1**: WhatsApp interface with basic voice processing
2. **Phase 2**: RAG-based scheme matching and recommendations
3. **Phase 3**: Form filling assistance and document education
4. **Phase 4**: Web dashboard with voice recording
5. **Phase 5**: Advanced features and optimizations

**Feature Flags:**
- Enable/disable features without deployment
- A/B testing for conversation flows
- Gradual rollout to user segments
- Quick rollback on issues

**Disaster Recovery:**
- Regular database backups (daily full, hourly incremental)
- Multi-region deployment for high availability
- Automated failover mechanisms
- Documented recovery procedures
