# EduTech Platform - Requirements Document

## 1. Overview

### 1.1 Product Vision
An AI-powered educational platform that helps students prepare for competitive exams through personalized study plans, AI tutoring, gamification, mentor connections, and collaborative study groups.

### 1.2 Target Users
- **Students**: Preparing for competitive exams (JEE, NEET, GATE, etc.)
- **Mentors**: Subject matter experts providing guidance and support
- **Parents**: Monitoring student progress (future feature)

### 1.3 Core Value Proposition
- AI-driven personalized learning paths
- Gamified learning experience
- Real-time mentor support
- Collaborative study groups
- Multi-modal content delivery (text, audio, visual)

---

## 2. User Roles & Authentication

### 2.1 User Role Management
**User Story**: As a user, I can register with a specific role (Student or Mentor) so that I have access to role-appropriate features.

**Acceptance Criteria**:
- Users can sign up with Firebase Authentication
- Users select role during registration (Student/Mentor)
- Role is stored in Firestore `users` collection
- Role determines dashboard and available features
- Students see student dashboard with learning features
- Mentors see mentor dashboard with ticket/meet management

### 2.2 Authentication & Authorization
**User Story**: As a user, I can securely log in and access protected resources.

**Acceptance Criteria**:
- Firebase Authentication integration (email/password, Google, etc.)
- JWT token verification on backend
- Protected API routes require valid Bearer token
- Frontend redirects unauthenticated users to landing page
- Role-based route protection (students can't access mentor routes)

---

## 3. Syllabus Management

### 3.1 PDF Syllabus Upload
**User Story**: As a student, I can upload my course syllabus as a PDF so the system can extract topics and create a study plan.

**Acceptance Criteria**:
- File upload interface accepts PDF files
- Backend parses PDF using `pdf-parse` library
- Extracted text is cleaned and normalized
- AI (OpenRouter) structures text into units and topics
- Validation ensures extracted topics match original text
- Confidence scores assigned to each topic
- Parsed syllabus saved to Firestore

### 3.2 Manual Syllabus Entry
**User Story**: As a student, I can manually enter my syllabus topics if I don't have a PDF.

**Acceptance Criteria**:
- UI provides form to add units and topics
- Users can add/edit/delete units
- Users can add/edit/delete topics within units
- Topics can have difficulty levels and estimated hours
- Changes saved to Firestore in real-time

### 3.3 Pre-stored Exam Syllabi
**User Story**: As a student preparing for standard exams, I can select from pre-stored official syllabi (JEE, NEET, GATE, etc.).

**Acceptance Criteria**:
- System stores official syllabi in `backend/src/data/examSyllabi.js`
- Available exams: JEE Main (Physics, Chemistry, Math), GATE CS, NEET Biology, Math Olympiad
- Students can browse and select exam syllabi
- AI structures pre-stored text into units/topics
- No web scraping or external fetching required
- 100% accuracy guaranteed from official sources

### 3.4 Syllabus Editing
**User Story**: As a student, I can edit my syllabus after it's been created to add/remove topics or adjust details.

**Acceptance Criteria**:
- Syllabus editor UI with unit/topic management
- Add/remove/rename units
- Add/remove/edit topics
- Update topic difficulty and estimated hours
- Parse comma/period-separated topics into subtopics
- Changes persist to Firestore

---

## 4. Study Planning & Scheduling

### 4.1 AI-Generated Study Plans
**User Story**: As a student, I receive an AI-generated daily study plan based on my syllabus and exam date.

**Acceptance Criteria**:
- System generates daily tasks from syllabus topics
- Tasks distributed based on exam date and available time
- Each task includes subject, title, difficulty, estimated hours
- Tasks stored in Firestore under user's schedule
- Daily plan visible on dashboard

### 4.2 Study Calendar
**User Story**: As a student, I can view my study schedule in a calendar format.

**Acceptance Criteria**:
- Calendar component displays daily tasks
- Color-coded by subject or difficulty
- Click on date to see tasks for that day
- Visual indicators for completed tasks
- Supports navigation between months

### 4.3 Schedule Management
**User Story**: As a student, I can view, modify, and track my study schedule.

**Acceptance Criteria**:
- View all scheduled tasks
- Mark tasks as complete
- Reschedule tasks to different dates
- View consolidated schedule across all subjects
- Preview upcoming tasks

---

## 5. AI Learning Features

### 5.1 AI Tutor (Ask AI)
**User Story**: As a student, I can ask questions to an AI tutor and receive explanations.

**Acceptance Criteria**:
- Chat interface for asking questions
- Integration with OpenRouter API (GPT models)
- Context-aware responses based on syllabus
- Conversation history saved to Firestore
- Support for follow-up questions
- Markdown rendering for formatted responses

### 5.2 AI-Generated Study Notes
**User Story**: As a student, I can generate structured study notes for my daily tasks using AI.

**Acceptance Criteria**:
- Generate notes from today's task list
- Notes include summaries, bullet points, key definitions
- Caching mechanism prevents duplicate generation
- Notes stored in Firestore `dailyStudyNotes` collection
- Signature-based deduplication (date + tasks hash)
- Support for custom prompts

### 5.3 Mind Map Generation
**User Story**: As a student, I can generate mind maps to visualize topic relationships.

**Acceptance Criteria**:
- Text-based mind map generation from topics
- AI structures topics into hierarchical format
- Visual mind map with interactive nodes
- 3D visualization option using Three.js
- Export/save mind maps
- Navigate between related concepts

---

## 6. Assessment & Testing

### 6.1 AI-Generated Quizzes
**User Story**: As a student, I can take AI-generated quizzes based on my daily study topics.

**Acceptance Criteria**:
- Generate 10 multiple-choice questions per quiz
- Three difficulty levels: Basic, Advanced, Scenario
- Questions based on today's tasks/topics
- 4 options per question with one correct answer
- Marks assigned based on difficulty (1/2/3 points)
- Quiz saved to Firestore for later review

### 6.2 Quiz Taking & Scoring
**User Story**: As a student, I can take quizzes and see my score immediately.

**Acceptance Criteria**:
- Interactive quiz interface
- Timer for each quiz (optional)
- Submit answers and get instant feedback
- Score calculation and display
- Correct answers revealed after submission
- Quiz history saved to Firestore

### 6.3 Mastery Score Tracking
**User Story**: As a student, I can track my mastery level for each topic over time.

**Acceptance Criteria**:
- Mastery score calculated from quiz performance
- Score per topic/unit displayed
- Visual progress indicators
- Historical mastery data stored
- Identify weak areas for focused study

### 6.4 Daily Scheduled Tests
**User Story**: As a student, I receive scheduled tests at specific times to reinforce learning.

**Acceptance Criteria**:
- Tests scheduled based on study plan
- Notifications for upcoming tests
- Test generation from recent topics
- Performance tracking over time
- Adaptive difficulty based on performance

---

## 7. Gamification

### 7.1 Zombie Survival Game
**User Story**: As a student, I can play an educational zombie survival game where answering questions correctly helps me survive.

**Acceptance Criteria**:
- Game generates questions from syllabus
- Correct answers provide defense/health
- Wrong answers cause damage
- Score tracking and leaderboards
- Game state saved for continuation

### 7.2 Memory Card Game
**User Story**: As a student, I can play a memory matching game with educational content.

**Acceptance Criteria**:
- Cards contain topic-related content
- Match pairs to score points
- Difficulty levels (number of cards)
- Timer and move counter
- High score tracking

### 7.3 Whack-a-Mole Game
**User Story**: As a student, I can play whack-a-mole with educational questions.

**Acceptance Criteria**:
- Questions appear as moles
- Click correct answer to score
- Time-limited gameplay
- Increasing difficulty
- Score and accuracy tracking

### 7.4 Achievements System
**User Story**: As a student, I earn achievements and badges for completing learning milestones.

**Acceptance Criteria**:
- Achievements for quiz completion, streaks, scores
- Badge display on profile
- Achievement notifications
- Progress tracking toward next achievement
- Leaderboard integration

---

## 8. Mentor System

### 8.1 Mentor Discovery
**User Story**: As a student, I can browse available mentors and see their specializations.

**Acceptance Criteria**:
- List all mentors with role "Mentor"
- Display mentor name and specializations
- Filter mentors by subject/specialization
- View mentor profile details

### 8.2 Mentor Connection Requests
**User Story**: As a student, I can send connection requests to mentors.

**Acceptance Criteria**:
- Send connection request to any mentor
- Request stored in `mentor_requests` collection
- Status: pending, accepted, rejected
- Prevent duplicate requests (deterministic doc ID)
- View status of sent requests

### 8.3 Mentor Request Management
**User Story**: As a mentor, I can view and respond to incoming connection requests.

**Acceptance Criteria**:
- View all pending requests
- See student name and details
- Accept or reject requests
- Accepted requests create `mentor_connections` entry
- Rejected requests marked as rejected
- Requests sorted by date (newest first)

### 8.4 Connected Students Management
**User Story**: As a mentor, I can view all students connected to me.

**Acceptance Criteria**:
- List all connected students
- Display student name and email
- View connection date
- Access student progress (future)
- Sorted by connection date

### 8.5 Doubt Tickets
**User Story**: As a student, I can raise doubt tickets that my connected mentors can resolve.

**Acceptance Criteria**:
- Create ticket with subject, description, priority
- Ticket assigned to connected mentor
- Mentor can view all tickets
- Mentor can respond and resolve tickets
- Ticket status tracking (open, in-progress, resolved)
- Notification system for updates

### 8.6 Mentor Meets Scheduling
**User Story**: As a student, I can schedule one-on-one meetings with my mentor.

**Acceptance Criteria**:
- Request meeting with date/time/topic
- Mentor receives meeting request
- Mentor can accept/reject/reschedule
- Calendar integration for scheduled meets
- Meeting reminders
- Video call link generation (future)

---

## 9. Study Groups

### 9.1 Study Group Creation
**User Story**: As a student, I can create study groups for collaborative learning.

**Acceptance Criteria**:
- Create group with name, subject, description
- Set visibility (public/private)
- Add tags for discoverability
- Set next meeting date/time
- Creator becomes organizer
- Group stored in `studyGroups` collection

### 9.2 Study Group Discovery
**User Story**: As a student, I can discover and join public study groups.

**Acceptance Criteria**:
- List all public groups
- Filter by subject, tags
- View group details (members, next meeting)
- Join public groups instantly
- Request to join private groups

### 9.3 Study Group Membership
**User Story**: As a student, I can join groups and participate in discussions.

**Acceptance Criteria**:
- Join public groups directly
- Request to join private groups (requires organizer approval)
- Leave groups at any time
- View member list
- Membership stored in subcollection

### 9.4 Group Chat
**User Story**: As a group member, I can chat with other members in real-time.

**Acceptance Criteria**:
- Send text messages to group
- View message history
- Messages ordered chronologically
- Display sender name and timestamp
- Real-time updates (polling or websockets)
- Messages stored in Firestore subcollection

### 9.5 Group Management
**User Story**: As a group organizer, I can manage group settings and members.

**Acceptance Criteria**:
- Edit group details (name, description, tags)
- Approve/reject join requests for private groups
- Remove members
- Delete group (with all subcollections)
- Transfer organizer role (future)

---

## 10. Content Delivery

### 10.1 Text-to-Speech (TTS)
**User Story**: As a student, I can convert study notes to audio for listening while commuting.

**Acceptance Criteria**:
- Generate audio from text notes
- Support for conversation-style TTS (multiple speakers)
- Python Flask service with Coqui TTS
- Audio files stored locally and served via API
- Playback controls in frontend
- Download audio files

### 10.2 Text-to-Podcast
**User Story**: As a student, I can generate podcast-style audio from my study materials.

**Acceptance Criteria**:
- AI generates conversational script from notes
- Multiple speaker voices
- Natural dialogue format
- Background music (optional)
- Episode-style organization
- Downloadable podcast files

### 10.3 Study Reader
**User Story**: As a student, I can use a focused reading mode for distraction-free studying.

**Acceptance Criteria**:
- Clean reading interface
- Text highlighting and annotations
- Adjustable font size and theme
- Progress tracking
- Bookmark support
- Export notes

---

## 11. Wellness & Productivity

### 11.1 Peace Mode
**User Story**: As a student, I can activate peace mode for meditation and stress relief.

**Acceptance Criteria**:
- Calming visual interface
- Guided breathing exercises
- Ambient sounds/music
- Timer for meditation sessions
- Progress tracking
- Daily reminders

### 11.2 Exam Countdown
**User Story**: As a student, I can see a countdown to my exam date to stay motivated.

**Acceptance Criteria**:
- Display days/hours/minutes to exam
- Visual countdown timer
- Motivational messages
- Progress indicators
- Multiple exam support

---

## 12. Data Storage & Sync

### 12.1 Firestore Integration
**User Story**: As a user, all my data is securely stored and synced across devices.

**Acceptance Criteria**:
- User profiles in `users` collection
- Syllabi in `syllabi` collection
- Study plans in `schedules` collection
- Quiz results in `quizResults` collection
- Study notes in `dailyStudyNotes` collection
- Mentor connections in `mentor_connections` collection
- Study groups in `studyGroups` collection
- Real-time sync across devices

### 12.2 Caching & Performance
**User Story**: As a user, I experience fast load times through intelligent caching.

**Acceptance Criteria**:
- Study notes cached by signature (date + tasks)
- Quiz questions cached to avoid regeneration
- Firestore queries optimized with indexes
- Frontend caching for static data
- Lazy loading for large datasets

---

## 13. Non-Functional Requirements

### 13.1 Performance
- API response time < 2 seconds for most requests
- AI generation < 10 seconds for notes/quizzes
- TTS generation < 30 seconds for 1000 words
- Frontend load time < 3 seconds

### 13.2 Security
- All API routes protected with Firebase Auth
- Role-based access control enforced
- Input validation on all endpoints
- SQL injection prevention (N/A - using Firestore)
- XSS protection in frontend

### 13.3 Scalability
- Support 1000+ concurrent users
- Firestore auto-scaling
- Stateless backend for horizontal scaling
- CDN for static assets (future)

### 13.4 Reliability
- 99.9% uptime target
- Error handling on all API routes
- Graceful degradation when AI services unavailable
- Automatic retry for failed requests

### 13.5 Usability
- Responsive design (mobile, tablet, desktop)
- Intuitive navigation
- Consistent UI/UX patterns
- Accessibility compliance (WCAG 2.1 Level AA target)
- Loading states and error messages

---

## 14. Technology Stack

### 14.1 Frontend
- React 19.2.0
- React Router 7.12.0
- Framer Motion for animations
- Three.js for 3D visualizations
- Lucide React for icons
- Vite for build tooling

### 14.2 Backend
- Node.js with Express.js
- Firebase Admin SDK
- Multer for file uploads
- pdf-parse for PDF extraction
- Axios for HTTP requests

### 14.3 AI Services
- OpenRouter API (GPT-3.5-turbo, GPT-4)
- Coqui TTS (VITS model) for speech synthesis

### 14.4 Database
- Firebase Firestore (NoSQL)
- Firebase Authentication

### 14.5 Additional Services
- Python Flask for TTS service
- Local file storage for audio files

---

## 15. Future Enhancements

### 15.1 Planned Features
- Video call integration for mentor meets
- Parent dashboard for progress monitoring
- Mobile app (React Native)
- Offline mode support
- Advanced analytics and insights
- Collaborative whiteboard
- Live study sessions
- Peer-to-peer tutoring marketplace
- Integration with external learning platforms
- AI-powered essay grading
- Spaced repetition system
- Flashcard generation

### 15.2 Scalability Improvements
- Microservices architecture
- Redis caching layer
- Message queue for async tasks
- CDN for global content delivery
- Database sharding for large datasets

---

## 16. Success Metrics

### 16.1 User Engagement
- Daily active users (DAU)
- Average session duration
- Quiz completion rate
- Study group participation rate
- Mentor-student connection rate

### 16.2 Learning Outcomes
- Average quiz scores over time
- Mastery score improvements
- Topic completion rate
- Exam success rate (self-reported)

### 16.3 Platform Health
- API error rate < 1%
- Average response time
- User retention rate
- Feature adoption rate
- User satisfaction score (NPS)

---

## Document Version
- **Version**: 1.0
- **Last Updated**: 2026-02-15
- **Status**: Complete - Documenting Existing System
