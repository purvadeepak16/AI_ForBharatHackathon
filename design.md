# EduTech Platform - Design Document

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                      │
│  - Dashboard, Syllabus, Study Plans, Games, Chat        │
│  - Port: 5173 (Vite Dev Server)                         │
└────────────────┬────────────────────────────────────────┘
                 │ HTTP/REST
                 │
┌────────────────▼────────────────────────────────────────┐
│              Backend (Node.js/Express)                   │
│  - API Routes, Controllers, Services                     │
│  - Port: 5000                                            │
└─────┬──────────┬──────────────┬────────────────────────┘
      │          │              │
      │          │              │
┌─────▼──────┐  │  ┌───────────▼──────┐  ┌──────────────┐
│ Firebase   │  │  │  OpenRouter API  │  │ Python TTS   │
│ Firestore  │  │  │  (GPT Models)    │  │ Service      │
│ Auth       │  │  └──────────────────┘  │ Port: 5001   │
└────────────┘  │                        └──────────────┘
                │
        ┌───────▼────────┐
        │  File Storage  │
        │  (PDF, Audio)  │
        └────────────────┘
```

### 1.2 Technology Stack

**Frontend**:
- React 19.2.0 with React Router
- Framer Motion (animations)
- Three.js (@react-three/fiber, @react-three/drei)
- Lucide React (icons)
- Vite (build tool)

**Backend**:
- Node.js with Express.js
- Firebase Admin SDK
- Multer (file uploads)
- pdf-parse (PDF extraction)
- Axios (HTTP client)

**AI/ML**:
- OpenRouter API (GPT-3.5-turbo, GPT-4)
- Coqui TTS (Python Flask service)

**Database**:
- Firebase Firestore (NoSQL)
- Firebase Authentication



## 2. Database Schema (Firestore)

### 2.1 Collections Overview

```
firestore/
├── users/
│   └── {userId}/
│       ├── userName: string
│       ├── email: string
│       ├── userRole: "Student" | "Mentor"
│       ├── mentorSpecializations: string[]
│       └── createdAt: timestamp
│
├── syllabi/
│   └── {syllabusId}/
│       ├── userId: string
│       ├── subject: string
│       ├── source: "pdf" | "manual" | "exam"
│       ├── level: string
│       ├── units: array
│       │   └── { id, name, topics: [{ id, title, subtopics, difficulty }] }
│       ├── createdAt: timestamp
│       └── metadata: object
│
├── schedules/
│   └── {userId}/
│       └── dates/
│           └── {YYYY-MM-DD}/
│               └── tasks: array
│                   └── { id, subject, title, difficulty, estimatedHours }
│
├── dailyStudyNotes/
│   └── {userId}/
│       └── entries/
│           └── {date_signature}/
│               ├── notes: string
│               ├── prompt: string
│               ├── tasks: array
│               ├── date: string
│               └── createdAt: timestamp
│
├── quizResults/
│   └── {userId}/
│       └── quizzes/
│           └── {quizId}/
│               ├── title: string
│               ├── questions: array
│               ├── level: "basic" | "advanced" | "scenario"
│               ├── score: number
│               └── createdAt: timestamp
│
├── mentor_requests/
│   └── {studentId_mentorId}/
│       ├── studentId: string
│       ├── mentorId: string
│       ├── status: "pending" | "accepted" | "rejected"
│       ├── createdAt: string
│       └── updatedAt: string
│
├── mentor_connections/
│   └── {connectionId}/
│       ├── studentId: string
│       ├── mentorId: string
│       ├── status: "connected"
│       ├── createdAt: string
│       └── updatedAt: string
│
└── studyGroups/
    └── {groupId}/
        ├── name: string
        ├── subject: string
        ├── description: string
        ├── visibility: "public" | "private"
        ├── organizerId: string
        ├── tags: string[]
        ├── members: array
        ├── nextMeeting: string
        ├── createdAt: string
        ├── members/
        │   └── {userId}/
        │       ├── userId: string
        │       ├── role: "organizer" | "member"
        │       └── joinedAt: string
        ├── messages/
        │   └── {messageId}/
        │       ├── text: string
        │       ├── senderId: string
        │       ├── senderName: string
        │       └── createdAt: timestamp
        └── requests/
            └── {requestId}/
                ├── userId: string
                ├── status: "pending" | "approved"
                └── createdAt: string
```



## 3. API Design

### 3.1 Authentication Endpoints

```
POST /api/auth/verify
Headers: Authorization: Bearer {token}
Response: { valid: boolean, uid: string, claims: object }

GET /api/protected
Headers: Authorization: Bearer {token}
Response: { message: string, uid: string }
```

### 3.2 Syllabus Endpoints

```
POST /api/syllabus/upload
Body: FormData with PDF file
Response: { success: boolean, syllabus: object }

POST /api/syllabus/parse-exam
Body: { examId: string, userId: string }
Response: { success: boolean, syllabus: object }

GET /api/syllabus/available-exams
Response: { exams: [{ id, name, subject }] }

POST /api/syllabus/save
Body: { userId, syllabus: object }
Response: { success: boolean, id: string }
```

### 3.3 Study Notes Endpoints

```
POST /api/study-notes/generate
Headers: Authorization: Bearer {token}
Body: { tasks: array, prompt: string, date: string }
Response: { 
  success: boolean, 
  cached: boolean,
  notes: string, 
  id: string 
}
```

### 3.4 Assessment Endpoints

```
POST /api/assessments/generate-quiz
Body: { 
  level: "basic" | "advanced" | "scenario",
  todaysTasks: array,
  userId: string 
}
Response: { 
  questions: [{ question, options, correctIndex, marks }],
  quizId: string 
}

POST /api/assessments/save-quiz
Body: { userId, title, questions }
Response: { success: boolean, id: string }
```

### 3.5 Mentor Endpoints

```
GET /api/mentor/list
Response: { success: boolean, mentors: array }

POST /api/mentor-requests
Headers: Authorization: Bearer {token}
Body: { mentorId: string }
Response: { success: boolean, request: object }

GET /api/mentor-requests/status
Headers: Authorization: Bearer {token}
Query: ?mentorIds=id1,id2
Response: { success: boolean, statuses: object }

GET /api/mentor/requests/incoming
Headers: Authorization: Bearer {token}
Response: { success: boolean, requests: array }

POST /api/mentor/requests/:id/accept
Headers: Authorization: Bearer {token}
Response: { success: boolean, message: string }

POST /api/mentor/requests/:id/reject
Headers: Authorization: Bearer {token}
Response: { success: boolean, message: string }

GET /api/mentor/connected-students
Headers: Authorization: Bearer {token}
Response: { success: boolean, connections: array }
```

### 3.6 Study Groups Endpoints

```
POST /api/study-groups
Headers: Authorization: Bearer {token}
Body: { name, subject, description, visibility, tags }
Response: { success: boolean, id: string, group: object }

GET /api/study-groups
Headers: Authorization: Bearer {token}
Response: { success: boolean, groups: array }

GET /api/study-groups/:id
Headers: Authorization: Bearer {token}
Response: { success: boolean, group: object, isMember: boolean }

POST /api/study-groups/:id/join
Headers: Authorization: Bearer {token}
Response: { success: boolean, message: string }

POST /api/study-groups/:id/leave
Headers: Authorization: Bearer {token}
Response: { success: boolean, message: string }

GET /api/study-groups/:id/messages
Headers: Authorization: Bearer {token}
Response: { success: boolean, messages: array }

POST /api/study-groups/:id/messages
Headers: Authorization: Bearer {token}
Body: { text: string }
Response: { success: boolean, message: object }

DELETE /api/study-groups/:id
Headers: Authorization: Bearer {token}
Response: { success: boolean, message: string }
```

### 3.7 TTS Endpoints

```
POST /api/tts/conversation
Body: { userId, conversationId }
Response: { success: boolean, audioUrl: string }

POST /api/tts/text
Body: { text: string }
Response: { success: boolean, audioUrl: string }

GET /audio/:filename
Response: Audio file (WAV)
```

### 3.8 Mind Map Endpoints

```
POST /api/mindmap/generate
Body: { topic: string, context: string }
Response: { success: boolean, mindmap: object }

POST /api/visual-mindmap/generate
Body: { topic: string, depth: number }
Response: { success: boolean, nodes: array, edges: array }
```

### 3.9 OpenRouter (AI) Endpoints

```
POST /api/openrouter/conversation
Body: { prompt: string, model: string }
Response: { success: boolean, response: string }
```



## 4. Component Architecture (Frontend)

### 4.1 Page Components

```
pages/
├── LandingPage.jsx - Unauthenticated landing
├── DashboardPage.jsx - Student dashboard
├── MentorDashboard.jsx - Mentor dashboard
├── SyllabusPage.jsx - Syllabus management
├── StudyGroupsPage.jsx - Study groups list
├── MindMapPage.jsx - Text mind maps
├── VisualMindMapPage.jsx - 3D mind maps
├── AskAIPage.jsx - AI tutor chat
├── StudyReaderPage.jsx - Reading mode
├── AssessmentsPage.jsx - Quiz dashboard
├── AssessmentTestPage.jsx - Take quiz
├── MasteryScorePage.jsx - Progress tracking
├── SchedulePage.jsx - Study calendar
├── TextToSpeechPage.jsx - TTS interface
├── TextToPodcastPage.jsx - Podcast generation
├── MentorsPage.jsx - Mentor discovery
├── DoubtTickets.jsx - Mentor tickets
├── MeetsManagement.jsx - Mentor meets
├── GamificationPage.jsx - Games dashboard
├── PeaceModePage.jsx - Wellness mode
├── ExamCountdownPage.jsx - Countdown timer
├── AchievementsPage.jsx - Badges/achievements
└── games/
    ├── ZombieGame.jsx
    ├── MemoryCardGame.jsx
    └── WhackAMoleGame.jsx
```

### 4.2 Reusable Components

```
components/
├── SyllabusIngestion.jsx - PDF upload & parsing
├── SyllabusEditor.jsx - Edit syllabus
├── DailyPlan.jsx - Today's tasks
├── StudyCalendar.jsx - Calendar view
├── ScheduleView.jsx - Schedule management
├── SchedulePreview.jsx - Quick schedule view
├── ConsolidatedScheduleView.jsx - All schedules
├── TodaysPlanSummary.jsx - Task summary
├── MindMap.jsx - Text mind map
├── VisualMindMap.jsx - 3D mind map
├── MindNode.jsx - Mind map node
├── TeacherAssistant.jsx - AI chat interface
├── ChatMessage.jsx - Chat message bubble
├── TestLevelsDisplay.jsx - Quiz difficulty selector
├── ConnectMentorButton.jsx - Mentor connection
├── MentorListModal.jsx - Mentor browser
├── MentorConnectionRequests.jsx - Request management
├── GroupChatPanel.jsx - Study group chat
├── GroupMeetPreview.jsx - Meeting preview
├── GamesGrid.jsx - Game cards grid
├── GameCard.jsx - Individual game card
├── PeaceMode.jsx - Meditation interface
├── Toast.jsx - Notification toast
├── ErrorBoundary.jsx - Error handling
└── ui/
    ├── button.jsx
    ├── card.jsx
    └── input.jsx
```

### 4.3 Context Providers

```
context/
├── AuthContext.jsx
│   - currentUser
│   - userData
│   - login/logout
│   - role-based routing
│
└── StudyMapContext.jsx
    - syllabus state
    - study plan state
    - progress tracking
```

### 4.4 Layout Components

```
layout/
└── MainLayout.jsx
    - Navigation sidebar
    - Header
    - Role-based menu
    - Logout functionality
```



## 5. Service Layer Design

### 5.1 Backend Services

```javascript
// services/firestoreService.js
- saveGeneratedQuiz(userId, quizData)
- getUserSchedule(userId, date)
- saveStudyNotes(userId, notes)
- getSyllabus(userId)
- saveSyllabus(userId, syllabus)

// services/pdfParser.js
- parsePDF(buffer)
- cleanText(rawText)
- extractStructure(text)

// services/aiExtractor.js
- extractTopicsFromText(text, subject)
- structureSyllabus(rawText)
- validateExtraction(topics, originalText)

// services/syllabusService.js
- normalizeSyllabus({ subject, source, units })
- addUnit(syllabus, unitName)
- removeUnit(syllabus, unitId)
- addTopic(syllabus, unitId, topic)
- removeTopic(syllabus, unitId, topicId)
- updateTopic(syllabus, unitId, topicId, updates)
- validateExtractedTopics(topics, text)
- parseSubtopics(title)

// services/scheduleService.js
- generateDailyPlan(syllabus, examDate)
- distributeTopics(topics, days)
- calculateDifficulty(topic)

// services/testGenerationService.js
- generateQuiz(topics, level)
- createQuestion(topic, difficulty)
- validateAnswers(questions, userAnswers)

// services/mentorRequestService.js
- createRequest(studentId, mentorId)
- acceptRequest(requestId)
- rejectRequest(requestId)
- getConnectedStudents(mentorId)
```

### 5.2 Python TTS Service

```python
# tts_service/app.py
from flask import Flask, request, send_file
from TTS.api import TTS
import os

app = Flask(__name__)
tts = TTS(model_name="tts_models/en/vctk/vits")

@app.route('/tts', methods=['POST'])
def generate_tts():
    text = request.json.get('text')
    filename = f"{uuid4()}.wav"
    tts.tts_to_file(text=text, file_path=f"generated_audio/{filename}")
    return {"audioUrl": f"/audio/{filename}"}

@app.route('/audio/<filename>')
def serve_audio(filename):
    return send_file(f"generated_audio/{filename}")

@app.route('/health')
def health():
    return {"status": "ok"}
```



## 6. Authentication & Authorization Flow

### 6.1 User Authentication

```
1. User signs up/logs in via Firebase Auth
2. Frontend receives ID token
3. Token stored in localStorage/sessionStorage
4. Token sent in Authorization header for all API calls
5. Backend verifies token using Firebase Admin SDK
6. User ID extracted from verified token
7. Request proceeds with authenticated user context
```

### 6.2 Role-Based Access Control

```javascript
// Middleware: auth.js
export default async function auth(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  
  try {
    const decoded = await admin.auth().verifyIdToken(token);
    req.user = { id: decoded.uid, email: decoded.email };
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}

// Role check in controllers
const userDoc = await db.collection('users').doc(userId).get();
const userRole = userDoc.data()?.userRole;

if (userRole === 'Mentor') {
  // Mentor-specific logic
} else {
  // Student-specific logic
}
```

### 6.3 Frontend Route Protection

```javascript
// App.jsx
function AppContent() {
  const { currentUser, userData } = useAuth();
  
  if (!currentUser) {
    return <LandingPage />;
  }
  
  const isMentor = userData?.userRole === 'Mentor';
  const defaultRoute = isMentor ? '/mentor/dashboard' : '/dashboard';
  
  return (
    <Routes>
      {isMentor && (
        <>
          <Route path="/mentor/dashboard" element={<MentorDashboard />} />
          <Route path="/mentor/tickets" element={<DoubtTickets />} />
        </>
      )}
      
      {!isMentor && (
        <>
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="/syllabus" element={<SyllabusPage />} />
        </>
      )}
      
      <Route path="*" element={<Navigate to={defaultRoute} />} />
    </Routes>
  );
}
```



## 7. Data Flow Diagrams

### 7.1 Syllabus Upload & Parsing Flow

```
User uploads PDF
    ↓
Frontend sends FormData to /api/syllabus/upload
    ↓
Backend receives file via Multer
    ↓
pdf-parse extracts raw text
    ↓
Text cleaning & normalization
    ↓
OpenRouter AI structures text into units/topics
    ↓
Validation: compare extracted topics with original text
    ↓
Assign confidence scores
    ↓
Save to Firestore (syllabi collection)
    ↓
Return structured syllabus to frontend
    ↓
Display in SyllabusEditor component
```

### 7.2 Quiz Generation Flow

```
User clicks "Generate Quiz"
    ↓
Frontend sends today's tasks + level to /api/assessments/generate-quiz
    ↓
Backend builds AI prompt with topics and difficulty guidance
    ↓
OpenRouter API generates 10 MCQ questions
    ↓
Backend parses and validates JSON response
    ↓
Fallback to mock questions if parsing fails
    ↓
Save quiz to Firestore (quizResults collection)
    ↓
Return questions to frontend
    ↓
Display in AssessmentTestPage
    ↓
User submits answers
    ↓
Calculate score and save results
```

### 7.3 Study Notes Generation Flow

```
User requests study notes
    ↓
Frontend sends tasks + date to /api/study-notes/generate
    ↓
Backend calculates signature (hash of tasks + date)
    ↓
Check Firestore cache (dailyStudyNotes/{userId}/entries/{signature})
    ↓
If cached: return immediately
    ↓
If not cached:
    ↓
Build prompt from tasks
    ↓
Call OpenRouter API
    ↓
Extract response text
    ↓
Save to Firestore with signature
    ↓
Return notes to frontend
```

### 7.4 TTS Generation Flow

```
User enters text and clicks "Generate Audio"
    ↓
Frontend sends text to /api/tts/text
    ↓
Backend forwards to Python TTS service (port 5001)
    ↓
Python service:
    - Loads Coqui TTS model (VITS)
    - Generates WAV file
    - Saves to generated_audio/ directory
    ↓
Returns audio filename
    ↓
Backend returns audio URL to frontend
    ↓
Frontend displays audio player with /audio/{filename}
    ↓
User plays/downloads audio
```

### 7.5 Mentor Connection Flow

```
Student browses mentors (/api/mentor/list)
    ↓
Student clicks "Connect" on mentor profile
    ↓
Frontend sends POST /api/mentor-requests { mentorId }
    ↓
Backend creates request with deterministic ID: {studentId}_{mentorId}
    ↓
Request saved to mentor_requests collection (status: pending)
    ↓
Mentor views incoming requests (/api/mentor/requests/incoming)
    ↓
Mentor clicks "Accept"
    ↓
Backend updates request status to "accepted"
    ↓
Creates mentor_connections entry
    ↓
Student can now raise tickets and schedule meets
```

### 7.6 Study Group Chat Flow

```
User joins study group
    ↓
Frontend loads messages (/api/study-groups/:id/messages)
    ↓
Messages displayed in chronological order
    ↓
User types message and clicks send
    ↓
POST /api/study-groups/:id/messages { text }
    ↓
Backend verifies membership
    ↓
Saves message with server timestamp
    ↓
Returns message object
    ↓
Frontend appends to chat
    ↓
(Optional) Polling or WebSocket for real-time updates
```



## 8. AI Integration Design

### 8.1 OpenRouter API Integration

**Configuration**:
```javascript
const OPENROUTER_API_KEY = process.env.OPENROUTER_API_KEY;
const OPENROUTER_URL = 'https://openrouter.ai/api/v1/chat/completions';
const DEFAULT_MODEL = 'openai/gpt-3.5-turbo';
```

**Request Format**:
```javascript
{
  model: 'openai/gpt-3.5-turbo',
  messages: [
    { role: 'user', content: 'Your prompt here' }
  ],
  temperature: 0.7,
  max_tokens: 2000
}
```

**Use Cases**:
1. Syllabus structuring from PDF text
2. Quiz question generation
3. Study notes generation
4. Mind map generation
5. Conversational AI tutor
6. Podcast script generation

### 8.2 Prompt Engineering Patterns

**Syllabus Extraction**:
```
"Parse the following syllabus text into structured JSON format.
Extract units and topics with this structure:
[
  {
    "name": "Unit Name",
    "topics": [
      { "title": "Topic Title", "difficulty": "easy|medium|hard" }
    ]
  }
]

Text: {syllabusText}

Return ONLY valid JSON, no markdown, no explanation."
```

**Quiz Generation**:
```
"Generate exactly 10 multiple-choice quiz questions about: {topics}

Requirements:
- Each question has 4 options
- Difficulty level: {level}
- Marks per question: {marks}
- Questions test {complexity_guidance}

Return ONLY JSON array:
[
  {
    "question": "Question text?",
    "options": ["Option 1", "Option 2", "Option 3", "Option 4"],
    "correctIndex": 0,
    "marks": 1
  }
]"
```

**Study Notes**:
```
"Today's Study Plan:
1. {subject}: {topic} ({difficulty})
2. ...

Please summarize these into structured study notes:
- For each item: short summary, 3 bullet points, 1 key definition
- Use clear headings and formatting
- Focus on exam-relevant content"
```

### 8.3 AI Response Handling

**Parsing Strategy**:
```javascript
function extractAssistantText(json) {
  // Handle various response formats
  if (json?.choices?.[0]?.message?.content) {
    return json.choices[0].message.content;
  }
  // Fallback handling...
}

function cleanAIResponse(text) {
  // Remove markdown code blocks
  text = text.replace(/```json\s*/g, '').replace(/```\s*/g, '');
  
  // Extract JSON array
  const arrayMatch = text.match(/\[[\s\S]*\]/);
  if (arrayMatch) {
    return JSON.parse(arrayMatch[0]);
  }
  
  throw new Error('Invalid AI response format');
}
```

**Error Handling**:
```javascript
try {
  const response = await callOpenRouter(prompt);
  const parsed = cleanAIResponse(response);
  return parsed;
} catch (err) {
  console.error('AI generation failed:', err);
  // Return mock/fallback data
  return generateMockData();
}
```

### 8.4 Caching Strategy

**Study Notes Caching**:
```javascript
function buildSignature(tasks, prompt) {
  const source = JSON.stringify(tasks) + '|' + prompt;
  return crypto.createHash('sha256').update(source).digest('hex').slice(0, 12);
}

const docId = `${date}_${signature}`;
const cached = await firestore.collection('dailyStudyNotes')
  .doc(userId)
  .collection('entries')
  .doc(docId)
  .get();

if (cached.exists) {
  return cached.data().notes; // Return cached
}

// Generate new notes...
```

**Benefits**:
- Reduces API costs
- Faster response times
- Consistent results for same inputs
- Offline capability for cached data



## 9. Security Design

### 9.1 Authentication Security

**Token Management**:
- Firebase ID tokens expire after 1 hour
- Automatic token refresh on frontend
- Tokens stored in memory (not localStorage for production)
- HTTPS-only in production

**Backend Verification**:
```javascript
// Every protected route
const token = req.headers.authorization?.replace('Bearer ', '');
const decoded = await admin.auth().verifyIdToken(token, true);
// checkRevoked: true ensures token hasn't been revoked
```

### 9.2 Authorization Security

**Role Verification**:
```javascript
async function requireRole(req, res, next, allowedRoles) {
  const userId = req.user.id;
  const userDoc = await db.collection('users').doc(userId).get();
  const userRole = userDoc.data()?.userRole;
  
  if (!allowedRoles.includes(userRole)) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  
  next();
}
```

**Resource Ownership**:
```javascript
// Ensure user can only access their own data
const syllabus = await db.collection('syllabi').doc(syllabusId).get();
if (syllabus.data().userId !== req.user.id) {
  return res.status(403).json({ error: 'Forbidden' });
}
```

### 9.3 Input Validation

**Request Validation**:
```javascript
// Validate required fields
if (!req.body.mentorId) {
  return res.status(400).json({ error: 'mentorId required' });
}

// Validate data types
if (!Array.isArray(tasks)) {
  return res.status(400).json({ error: 'tasks must be an array' });
}

// Sanitize user input
const cleanText = text.trim().substring(0, 10000); // Limit length
```

**File Upload Security**:
```javascript
const upload = multer({
  limits: { fileSize: 10 * 1024 * 1024 }, // 10MB max
  fileFilter: (req, file, cb) => {
    if (file.mimetype === 'application/pdf') {
      cb(null, true);
    } else {
      cb(new Error('Only PDF files allowed'));
    }
  }
});
```

### 9.4 CORS Configuration

```javascript
const ALLOWED_ORIGINS = [
  'http://localhost:5173',
  'http://127.0.0.1:5173',
  process.env.FRONTEND_ORIGIN
];

const corsOptions = {
  origin: ALLOWED_ORIGINS,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true
};

app.use(cors(corsOptions));
```

### 9.5 Rate Limiting (Future)

```javascript
// To be implemented
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/api/', limiter);
```

### 9.6 Data Privacy

**PII Handling**:
- User emails stored only in Firebase Auth
- Display names stored in Firestore
- No sensitive data in logs
- Firestore security rules enforce access control

**Firestore Security Rules** (to be implemented):
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users can only read/write their own data
    match /syllabi/{syllabusId} {
      allow read, write: if request.auth != null 
        && resource.data.userId == request.auth.uid;
    }
    
    // Study groups: public readable, members can write
    match /studyGroups/{groupId} {
      allow read: if resource.data.visibility == 'public' 
        || request.auth.uid in resource.data.members;
      allow write: if request.auth.uid == resource.data.organizerId;
    }
  }
}
```



## 10. Performance Optimization

### 10.1 Frontend Optimization

**Code Splitting**:
```javascript
// Lazy load routes
const DashboardPage = lazy(() => import('./pages/DashboardPage'));
const GamificationPage = lazy(() => import('./pages/GamificationPage'));

<Suspense fallback={<LoadingSpinner />}>
  <Routes>
    <Route path="/dashboard" element={<DashboardPage />} />
  </Routes>
</Suspense>
```

**Memoization**:
```javascript
// Expensive computations
const sortedTasks = useMemo(() => {
  return tasks.sort((a, b) => a.difficulty - b.difficulty);
}, [tasks]);

// Prevent unnecessary re-renders
const MemoizedComponent = memo(({ data }) => {
  return <div>{data}</div>;
});
```

**Debouncing**:
```javascript
// Search input
const debouncedSearch = useCallback(
  debounce((query) => {
    fetchResults(query);
  }, 300),
  []
);
```

### 10.2 Backend Optimization

**Database Indexing**:
```javascript
// Firestore composite indexes needed:
// - studyGroups: visibility + createdAt
// - mentor_requests: mentorId + status
// - mentor_requests: studentId + status
// - studyGroups/members: userId
```

**Query Optimization**:
```javascript
// Limit results
const groups = await db.collection('studyGroups')
  .where('visibility', '==', 'public')
  .limit(50)
  .get();

// Use specific fields
const users = await db.collection('users')
  .select('userName', 'userRole')
  .get();
```

**Caching**:
```javascript
// In-memory cache for frequently accessed data
const cache = new Map();

function getCachedData(key, fetchFn, ttl = 300000) {
  const cached = cache.get(key);
  if (cached && Date.now() - cached.timestamp < ttl) {
    return cached.data;
  }
  
  const data = await fetchFn();
  cache.set(key, { data, timestamp: Date.now() });
  return data;
}
```

### 10.3 API Optimization

**Batch Requests**:
```javascript
// Instead of N requests for N mentors
const mentorIds = ['id1', 'id2', 'id3'];
const statuses = await getMentorRequestStatuses(mentorIds);
// Returns: { id1: 'pending', id2: 'accepted', id3: 'none' }
```

**Pagination** (to be implemented):
```javascript
GET /api/study-groups?page=1&limit=20
GET /api/mentor/requests/incoming?page=1&limit=10
```

**Response Compression**:
```javascript
import compression from 'compression';
app.use(compression());
```

### 10.4 Asset Optimization

**Image Optimization**:
- Use WebP format for images
- Lazy load images below the fold
- Responsive images with srcset

**Bundle Optimization**:
```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'three-vendor': ['three', '@react-three/fiber', '@react-three/drei']
        }
      }
    }
  }
}
```

### 10.5 TTS Service Optimization

**Model Caching**:
```python
# Load model once at startup
tts = TTS(model_name="tts_models/en/vctk/vits")

# Reuse for all requests
@app.route('/tts', methods=['POST'])
def generate_tts():
    # Model already loaded
    tts.tts_to_file(text=text, file_path=path)
```

**Audio File Management**:
```python
# Cleanup old files periodically
def cleanup_old_files():
    for file in os.listdir('generated_audio'):
        if file_age > 24_hours:
            os.remove(file)
```



## 11. Error Handling & Resilience

### 11.1 Frontend Error Handling

**Error Boundary**:
```javascript
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };
  
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error, errorInfo) {
    console.error('Error caught:', error, errorInfo);
    // Log to error tracking service
  }
  
  render() {
    if (this.state.hasError) {
      return <ErrorFallback error={this.state.error} />;
    }
    return this.props.children;
  }
}
```

**API Error Handling**:
```javascript
async function fetchWithErrorHandling(url, options) {
  try {
    const response = await fetch(url, options);
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || 'Request failed');
    }
    
    return await response.json();
  } catch (err) {
    console.error('API Error:', err);
    showToast('error', err.message);
    throw err;
  }
}
```

**Toast Notifications**:
```javascript
function showToast(type, message) {
  // Display user-friendly error messages
  toast.show({
    type: 'error' | 'success' | 'info',
    message: message,
    duration: 3000
  });
}
```

### 11.2 Backend Error Handling

**Global Error Handler**:
```javascript
app.use((err, req, res, next) => {
  console.error('Unhandled error:', err);
  
  res.status(err.status || 500).json({
    error: err.message || 'Internal server error',
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
});
```

**Controller Error Patterns**:
```javascript
export async function controllerFunction(req, res) {
  try {
    // Validate input
    if (!req.body.requiredField) {
      return res.status(400).json({ error: 'requiredField is required' });
    }
    
    // Business logic
    const result = await someOperation();
    
    // Success response
    return res.json({ success: true, data: result });
    
  } catch (err) {
    console.error('Controller error:', err);
    return res.status(500).json({ 
      error: 'Internal server error',
      details: err.message 
    });
  }
}
```

**AI Service Fallbacks**:
```javascript
async function generateQuizWithFallback(topics, level) {
  try {
    // Try AI generation
    const questions = await generateWithAI(topics, level);
    return { questions, aiGenerated: true };
  } catch (err) {
    console.warn('AI generation failed, using mock data:', err);
    // Fallback to mock questions
    const mockQuestions = generateMockQuestions(topics, level);
    return { questions: mockQuestions, mock: true };
  }
}
```

### 11.3 Retry Logic

**Exponential Backoff**:
```javascript
async function retryWithBackoff(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (err) {
      if (i === maxRetries - 1) throw err;
      
      const delay = Math.pow(2, i) * 1000; // 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Usage
const data = await retryWithBackoff(() => fetchFromAPI(url));
```

### 11.4 Graceful Degradation

**Feature Flags**:
```javascript
const features = {
  aiGeneration: process.env.OPENROUTER_API_KEY ? true : false,
  ttsService: process.env.TTS_SERVICE_URL ? true : false,
  studyGroups: true
};

if (!features.aiGeneration) {
  console.warn('AI features disabled - no API key');
  // Use mock data or disable feature
}
```

**Service Health Checks**:
```javascript
// Check TTS service availability
async function checkTTSHealth() {
  try {
    const response = await fetch('http://localhost:5001/health');
    return response.ok;
  } catch {
    return false;
  }
}

// Disable TTS button if service unavailable
const ttsAvailable = await checkTTSHealth();
```

### 11.5 Logging & Monitoring

**Structured Logging**:
```javascript
function log(level, message, metadata = {}) {
  const logEntry = {
    timestamp: new Date().toISOString(),
    level,
    message,
    ...metadata
  };
  
  console[level](JSON.stringify(logEntry));
  
  // Send to monitoring service (future)
  // sendToMonitoring(logEntry);
}

// Usage
log('info', 'Quiz generated', { userId, quizId, level });
log('error', 'AI generation failed', { error: err.message, userId });
```

**Performance Monitoring**:
```javascript
function measurePerformance(name, fn) {
  const start = Date.now();
  const result = await fn();
  const duration = Date.now() - start;
  
  log('info', `Performance: ${name}`, { duration });
  
  return result;
}

// Usage
const quiz = await measurePerformance('generateQuiz', () => 
  generateQuiz(topics, level)
);
```



## 12. Multilingual Support Design

### 12.1 Internationalization (i18n) Architecture

**Language Detection**:
```javascript
// Detect user's preferred language
const userLanguage = navigator.language || navigator.userLanguage;
const supportedLanguages = ['en', 'hi', 'ta', 'te', 'kn', 'ml', 'bn', 'mr'];

function getLanguage() {
  const stored = localStorage.getItem('preferredLanguage');
  if (stored && supportedLanguages.includes(stored)) {
    return stored;
  }
  
  const browserLang = userLanguage.split('-')[0];
  return supportedLanguages.includes(browserLang) ? browserLang : 'en';
}
```

**Translation System**:
```javascript
// i18n/translations.js
const translations = {
  en: {
    dashboard: {
      title: 'Dashboard',
      todaysPlan: "Today's Study Plan",
      generateQuiz: 'Generate Quiz'
    },
    syllabus: {
      upload: 'Upload Syllabus',
      addUnit: 'Add Unit'
    }
  },
  hi: {
    dashboard: {
      title: 'डैशबोर्ड',
      todaysPlan: 'आज की अध्ययन योजना',
      generateQuiz: 'क्विज़ बनाएं'
    },
    syllabus: {
      upload: 'पाठ्यक्रम अपलोड करें',
      addUnit: 'इकाई जोड़ें'
    }
  },
  ta: {
    dashboard: {
      title: 'டாஷ்போர்டு',
      todaysPlan: 'இன்றைய படிப்பு திட்டம்',
      generateQuiz: 'வினாடி வினா உருவாக்கு'
    }
  }
  // Add more languages...
};

// Hook for translations
function useTranslation() {
  const [language, setLanguage] = useState(getLanguage());
  
  const t = (key) => {
    const keys = key.split('.');
    let value = translations[language];
    
    for (const k of keys) {
      value = value?.[k];
    }
    
    return value || key;
  };
  
  return { t, language, setLanguage };
}
```

### 12.2 Content Localization

**AI-Generated Content Translation**:
```javascript
// Backend: Translate AI responses
async function translateContent(text, targetLanguage) {
  if (targetLanguage === 'en') return text;
  
  const prompt = `Translate the following educational content to ${targetLanguage}. 
  Maintain formatting and technical terms:
  
  ${text}`;
  
  const response = await callOpenRouter(prompt);
  return response;
}

// Usage in study notes generation
export async function generateStudyNotesWithCache(req, res) {
  const { tasks, language = 'en' } = req.body;
  
  // Generate notes in English
  const notes = await generateNotes(tasks);
  
  // Translate if needed
  const localizedNotes = await translateContent(notes, language);
  
  return res.json({ notes: localizedNotes });
}
```

**Syllabus Language Support**:
```javascript
// Store syllabus in multiple languages
const syllabus = {
  id: 'syllabus-123',
  subject: 'Physics',
  language: 'en',
  translations: {
    hi: { subject: 'भौतिकी', units: [...] },
    ta: { subject: 'இயற்பியல்', units: [...] }
  },
  units: [...]
};
```

**TTS Multi-Language Support**:
```python
# Python TTS service with language support
from TTS.api import TTS

# Load models for different languages
models = {
    'en': TTS(model_name="tts_models/en/vctk/vits"),
    'hi': TTS(model_name="tts_models/hi/cv/vits"),
    'ta': TTS(model_name="tts_models/ta/cv/vits")
}

@app.route('/tts', methods=['POST'])
def generate_tts():
    text = request.json.get('text')
    language = request.json.get('language', 'en')
    
    tts_model = models.get(language, models['en'])
    filename = f"{uuid4()}.wav"
    
    tts_model.tts_to_file(text=text, file_path=f"generated_audio/{filename}")
    return {"audioUrl": f"/audio/{filename}"}
```

### 12.3 UI Language Switching

**Language Selector Component**:
```javascript
function LanguageSelector() {
  const { language, setLanguage } = useTranslation();
  
  const languages = [
    { code: 'en', name: 'English', flag: '🇬🇧' },
    { code: 'hi', name: 'हिन्दी', flag: '🇮🇳' },
    { code: 'ta', name: 'தமிழ்', flag: '🇮🇳' },
    { code: 'te', name: 'తెలుగు', flag: '🇮🇳' },
    { code: 'kn', name: 'ಕನ್ನಡ', flag: '🇮🇳' },
    { code: 'ml', name: 'മലയാളം', flag: '🇮🇳' },
    { code: 'bn', name: 'বাংলা', flag: '🇮🇳' },
    { code: 'mr', name: 'मराठी', flag: '🇮🇳' }
  ];
  
  const handleChange = (newLang) => {
    setLanguage(newLang);
    localStorage.setItem('preferredLanguage', newLang);
    // Reload content in new language
    window.location.reload();
  };
  
  return (
    <select value={language} onChange={(e) => handleChange(e.target.value)}>
      {languages.map(lang => (
        <option key={lang.code} value={lang.code}>
          {lang.flag} {lang.name}
        </option>
      ))}
    </select>
  );
}
```

### 12.4 RTL (Right-to-Left) Support

**Direction Handling**:
```javascript
const rtlLanguages = ['ar', 'ur', 'he'];

function getTextDirection(language) {
  return rtlLanguages.includes(language) ? 'rtl' : 'ltr';
}

// Apply to root element
useEffect(() => {
  document.documentElement.dir = getTextDirection(language);
  document.documentElement.lang = language;
}, [language]);
```

**RTL-Aware Styling**:
```css
/* Use logical properties for RTL support */
.container {
  margin-inline-start: 20px; /* Instead of margin-left */
  padding-inline-end: 10px;  /* Instead of padding-right */
}

[dir="rtl"] .icon {
  transform: scaleX(-1); /* Flip icons in RTL */
}
```

