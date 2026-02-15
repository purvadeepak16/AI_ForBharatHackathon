# Design Document: CodeQuest

## Overview

CodeQuest is an AI-powered gamified programming learning platform built with Flutter for cross-platform mobile deployment. The system architecture follows a serverless cloud-native approach using Firebase services for authentication, data storage, and backend functions, integrated with OpenAI's API for intelligent question generation and adaptive tutoring.

The design emphasizes:
- **Scalability**: Serverless architecture that scales automatically with user growth
- **Real-time synchronization**: Firestore real-time listeners for instant cross-device updates
- **Modularity**: Clean separation between UI, business logic, and data layers
- **Performance**: Aggressive caching and lazy loading strategies
- **Security**: Firebase security rules and server-side API key management

## Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Flutter Mobile App                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  UI Layer    │  │ Business     │  │  Data Layer  │      │
│  │  (Widgets)   │◄─┤ Logic        │◄─┤  (Repos)     │      │
│  │              │  │ (Riverpod)   │  │              │      │
│  └──────────────┘  └──────────────┘  └──────┬───────┘      │
└────────────────────────────────────────────┼────────────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │         Firebase Cloud Services                    │
                    │                                                    │
                    │  ┌──────────────┐  ┌──────────────┐              │
                    │  │ Firebase     │  │  Firestore   │              │
                    │  │ Auth         │  │  Database    │              │
                    │  └──────────────┘  └──────────────┘              │
                    │                                                    │
                    │  ┌──────────────────────────────────────┐        │
                    │  │   Firebase Cloud Functions           │        │
                    │  │  ┌────────────┐  ┌────────────┐     │        │
                    │  │  │ Question   │  │ Analytics  │     │        │
                    │  │  │ Generator  │  │ Processor  │     │        │
                    │  │  └─────┬──────┘  └────────────┘     │        │
                    │  └────────┼───────────────────────────────┘        │
                    └───────────┼────────────────────────────────────────┘
                                │
                    ┌───────────▼──────────┐
                    │   OpenAI API         │
                    │   (GPT-4)            │
                    └──────────────────────┘
```

### Component Architecture

The Flutter application follows a layered architecture with clear separation of concerns:

**Presentation Layer (UI)**
- Widgets organized by feature (auth, dashboard, game_modes, analytics)
- Stateless widgets for pure UI components
- Consumer widgets for reactive state updates via Riverpod

**Business Logic Layer**
- Riverpod providers for state management
- Service classes for business logic (GameService, AnalyticsService, AIService)
- Notifiers for complex state management (UserNotifier, ProgressNotifier)

**Data Layer**
- Repository pattern for data access abstraction
- Firebase service wrappers (AuthRepository, FirestoreRepository)
- Local caching layer using Hive for offline support

## Data Models

### User Model

```dart
class User {
  final String uid;
  final String email;
  final String? displayName;
  final String? photoUrl;
  final int xp;
  final int level;
  final int streak;
  final DateTime lastActiveDate;
  final DateTime createdAt;
  final Map<String, double> topicMastery; // topic -> mastery score (0-100)
  final List<String> earnedBadges;
  final bool leaderboardOptIn;
  
  // Computed properties
  int get xpForNextLevel => (100 * pow(level + 1, 1.5)).round();
  double get xpProgress => xp / xpForNextLevel;
}
```

### Question Model

```dart
class Question {
  final String id;
  final String topic;
  final int difficulty; // 1-5
  final String problemStatement;
  final List<String> inputExamples;
  final List<String> outputExamples;
  final String constraints;
  final String? hint;
  final GameMode gameMode;
  final int xpReward;
  final int timeLimit; // seconds
  
  // For Code Builder mode
  final List<CodeBlock>? codeBlocks;
  
  // For Bug Hunter mode
  final String? buggyCode;
  final List<int>? bugLines;
}
```

### Progress Model

```dart
class Progress {
  final String userId;
  final String questionId;
  final bool isCorrect;
  final int attemptCount;
  final int timeSpent; // seconds
  final DateTime completedAt;
  final String topic;
  final int difficulty;
  final int xpEarned;
}
```

### Mission Model

```dart
class Mission {
  final String id;
  final String userId;
  final String title;
  final String description;
  final MissionType type; // daily, weekly
  final int targetCount;
  final int currentCount;
  final int xpReward;
  final DateTime expiresAt;
  final bool isCompleted;
}
```

### Badge Model

```dart
class Badge {
  final String id;
  final String name;
  final String description;
  final String iconUrl;
  final BadgeCategory category; // streak, mastery, milestone
  final int requirement;
}
```

## Firestore Database Schema

### Collections Structure

```
users/
  {userId}/
    - uid: string
    - email: string
    - displayName: string?
    - photoUrl: string?
    - xp: number
    - level: number
    - streak: number
    - lastActiveDate: timestamp
    - createdAt: timestamp
    - topicMastery: map<string, number>
    - earnedBadges: array<string>
    - leaderboardOptIn: boolean

progress/
  {progressId}/
    - userId: string
    - questionId: string
    - isCorrect: boolean
    - attemptCount: number
    - timeSpent: number
    - completedAt: timestamp
    - topic: string
    - difficulty: number
    - xpEarned: number

missions/
  {missionId}/
    - userId: string
    - title: string
    - description: string
    - type: string
    - targetCount: number
    - currentCount: number
    - xpReward: number
    - expiresAt: timestamp
    - isCompleted: boolean

questions_cache/
  {questionId}/
    - topic: string
    - difficulty: number
    - problemStatement: string
    - inputExamples: array<string>
    - outputExamples: array<string>
    - constraints: string
    - hint: string?
    - gameMode: string
    - xpReward: number
    - timeLimit: number
    - createdAt: timestamp
    - usageCount: number

leaderboard/
  {userId}/
    - displayName: string
    - xp: number
    - level: number
    - rank: number
    - updatedAt: timestamp
```

### Indexing Strategy

**Composite Indexes:**
- `progress`: (userId, completedAt DESC) - for user progress history
- `progress`: (userId, topic, completedAt DESC) - for topic-specific analytics
- `leaderboard`: (xp DESC, updatedAt DESC) - for global rankings
- `missions`: (userId, expiresAt ASC, isCompleted) - for active missions

**Single-field Indexes:**
- `users.xp` (DESC) - for leaderboard queries
- `progress.completedAt` (DESC) - for recent activity
- `questions_cache.usageCount` (ASC) - for cache eviction

## API Design (Cloud Functions)

### Function: generateQuestion

**Endpoint:** `POST /api/generateQuestion`

**Request:**
```json
{
  "userId": "string",
  "topic": "string",
  "gameMode": "zombie_survival | code_builder | bug_hunter | algorithm_race",
  "difficulty": "number (1-5)"
}
```

**Response:**
```json
{
  "questionId": "string",
  "topic": "string",
  "difficulty": "number",
  "problemStatement": "string",
  "inputExamples": ["string"],
  "outputExamples": ["string"],
  "constraints": "string",
  "hint": "string?",
  "xpReward": "number",
  "timeLimit": "number",
  "codeBlocks": ["CodeBlock"]?,
  "buggyCode": "string?"
}
```

**Logic:**
1. Fetch user's current level and topic mastery from Firestore
2. Check questions_cache for suitable cached question
3. If cache miss, call OpenAI API with structured prompt
4. Parse and validate AI response
5. Store question in questions_cache
6. Return question to client

### Function: submitAnswer

**Endpoint:** `POST /api/submitAnswer`

**Request:**
```json
{
  "userId": "string",
  "questionId": "string",
  "answer": "string",
  "timeSpent": "number"
}
```

**Response:**
```json
{
  "isCorrect": "boolean",
  "explanation": "string",
  "xpEarned": "number",
  "newXp": "number",
  "newLevel": "number",
  "leveledUp": "boolean",
  "badgesEarned": ["string"],
  "streakUpdated": "boolean",
  "newStreak": "number"
}
```

**Logic:**
1. Validate answer against question solution
2. If incorrect, call OpenAI API for explanation
3. Calculate XP based on difficulty, speed, and streak
4. Update user XP, level, and streak in Firestore
5. Record progress entry
6. Update topic mastery score
7. Check for badge achievements
8. Update missions progress
9. Return comprehensive response

### Function: getPersonalizedRoadmap

**Endpoint:** `GET /api/roadmap?userId={userId}`

**Response:**
```json
{
  "weakTopics": [
    {
      "topic": "string",
      "masteryScore": "number",
      "priority": "number",
      "recommendedQuestions": "number"
    }
  ],
  "suggestedPath": ["string"],
  "estimatedTimeToMastery": "number"
}
```

**Logic:**
1. Fetch user's topic mastery scores
2. Identify topics with mastery < 70%
3. Call OpenAI API with user's progress data for personalized recommendations
4. Parse AI response and structure roadmap
5. Return prioritized learning path

### Function: generateDailyMissions

**Endpoint:** `POST /api/generateMissions` (Scheduled daily at midnight UTC)

**Logic:**
1. Query all active users
2. For each user:
   - Analyze recent progress and weak topics
   - Generate 3 missions: 1 easy, 1 medium, 1 challenging
   - Create mission documents in Firestore
3. Send push notifications for new missions

### Function: updateLeaderboard

**Endpoint:** `POST /api/updateLeaderboard` (Triggered on user XP change)

**Request:**
```json
{
  "userId": "string",
  "newXp": "number"
}
```

**Logic:**
1. Update user's leaderboard entry
2. Recalculate rank based on XP
3. Update rank field in leaderboard document
4. Use Firestore transaction for consistency

## State Management Architecture (Riverpod)

### Provider Structure

**Auth Providers:**
```dart
final authRepositoryProvider = Provider<AuthRepository>((ref) => AuthRepository());

final authStateProvider = StreamProvider<User?>((ref) {
  return ref.watch(authRepositoryProvider).authStateChanges();
});

final currentUserProvider = StreamProvider<UserModel?>((ref) {
  final authState = ref.watch(authStateProvider);
  return authState.when(
    data: (user) => user != null 
      ? ref.watch(firestoreRepositoryProvider).getUserStream(user.uid)
      : Stream.value(null),
    loading: () => Stream.value(null),
    error: (_, __) => Stream.value(null),
  );
});
```

**Game State Providers:**
```dart
final gameServiceProvider = Provider<GameService>((ref) => GameService(ref));

final currentQuestionProvider = StateProvider<Question?>((ref) => null);

final gameSessionProvider = StateNotifierProvider<GameSessionNotifier, GameSession>(
  (ref) => GameSessionNotifier(ref),
);

class GameSessionNotifier extends StateNotifier<GameSession> {
  final Ref ref;
  
  GameSessionNotifier(this.ref) : super(GameSession.initial());
  
  Future<void> startSession(GameMode mode) async {
    // Fetch question from API
    // Update state
  }
  
  Future<void> submitAnswer(String answer) async {
    // Submit to API
    // Update state with result
    // Update user XP/level
  }
}
```

**Analytics Providers:**
```dart
final analyticsServiceProvider = Provider<AnalyticsService>((ref) => AnalyticsService(ref));

final userProgressProvider = StreamProvider.family<List<Progress>, String>((ref, userId) {
  return ref.watch(firestoreRepositoryProvider).getProgressStream(userId);
});

final topicMasteryProvider = Provider.family<Map<String, double>, String>((ref, userId) {
  final user = ref.watch(currentUserProvider).value;
  return user?.topicMastery ?? {};
});

final weeklyStatsProvider = FutureProvider.family<WeeklyStats, String>((ref, userId) async {
  return ref.watch(analyticsServiceProvider).calculateWeeklyStats(userId);
});
```

**Mission Providers:**
```dart
final activeMissionsProvider = StreamProvider.family<List<Mission>, String>((ref, userId) {
  return ref.watch(firestoreRepositoryProvider).getActiveMissions(userId);
});

final missionProgressProvider = StateNotifierProvider<MissionProgressNotifier, Map<String, int>>(
  (ref) => MissionProgressNotifier(ref),
);
```

## AI Integration Flow

### Question Generation Pipeline

1. **Client Request**: User initiates challenge in specific game mode
2. **Context Gathering**: Cloud Function fetches user profile, mastery scores, recent progress
3. **Prompt Construction**:
```
You are an expert programming tutor. Generate a coding question with these requirements:
- Topic: {topic}
- Difficulty: {difficulty}/5
- Game Mode: {gameMode}
- User Level: {userLevel}
- User's mastery in this topic: {masteryScore}%

Format the response as JSON:
{
  "problemStatement": "...",
  "inputExamples": ["..."],
  "outputExamples": ["..."],
  "constraints": "...",
  "hint": "...",
  "solution": "..."
}
```

4. **API Call**: POST to OpenAI API with GPT-4 model
5. **Response Parsing**: Validate JSON structure and content
6. **Caching**: Store in questions_cache for reuse
7. **Return**: Send question to client

### Adaptive Difficulty Algorithm

```dart
class AdaptiveDifficultyEngine {
  int calculateNextDifficulty(String topic, List<Progress> recentProgress) {
    final last10 = recentProgress.where((p) => p.topic == topic).take(10).toList();
    
    if (last10.length < 5) {
      return 2; // Default to medium
    }
    
    final accuracy = last10.where((p) => p.isCorrect).length / last10.length;
    final avgTime = last10.map((p) => p.timeSpent).reduce((a, b) => a + b) / last10.length;
    final currentDifficulty = last10.first.difficulty;
    
    if (accuracy >= 0.8 && avgTime < 120) {
      return min(5, currentDifficulty + 1);
    } else if (accuracy < 0.5 || avgTime > 300) {
      return max(1, currentDifficulty - 1);
    }
    
    return currentDifficulty;
  }
}
```

### Explanation Generation

When user submits incorrect answer:

1. **Context**: Send question, user's answer, and correct solution to OpenAI
2. **Prompt**:
```
The user attempted this coding problem:
{problemStatement}

Their answer:
{userAnswer}

Correct solution:
{correctSolution}

Explain what went wrong in simple terms and guide them to the correct approach.
Use beginner-friendly language and include a code example.
```

3. **Parse**: Extract explanation text
4. **Return**: Display to user with formatting

## Components and Interfaces

### Core Services

**AuthService**
```dart
class AuthService {
  final FirebaseAuth _auth;
  
  Future<UserCredential> signUpWithEmail(String email, String password);
  Future<UserCredential> signInWithEmail(String email, String password);
  Future<UserCredential> signInWithGoogle();
  Future<void> signOut();
  Stream<User?> authStateChanges();
}
```

**GameService**
```dart
class GameService {
  final Ref ref;
  final ApiClient _apiClient;
  
  Future<Question> generateQuestion(String topic, GameMode mode, int difficulty);
  Future<SubmitAnswerResponse> submitAnswer(String questionId, String answer, int timeSpent);
  Future<void> requestHint(String questionId);
}
```

**AnalyticsService**
```dart
class AnalyticsService {
  final FirestoreRepository _firestore;
  
  Future<WeeklyStats> calculateWeeklyStats(String userId);
  Future<Map<String, double>> calculateTopicMastery(String userId);
  Future<List<String>> identifyWeakTopics(String userId);
  Stream<List<Progress>> getProgressStream(String userId);
}
```

**MissionService**
```dart
class MissionService {
  final FirestoreRepository _firestore;
  
  Future<List<Mission>> getActiveMissions(String userId);
  Future<void> updateMissionProgress(String missionId, int progress);
  Future<void> completeMission(String missionId);
}
```

### Repository Interfaces

**FirestoreRepository**
```dart
class FirestoreRepository {
  final FirebaseFirestore _firestore;
  
  // User operations
  Future<UserModel> getUser(String userId);
  Stream<UserModel> getUserStream(String userId);
  Future<void> updateUser(String userId, Map<String, dynamic> data);
  
  // Progress operations
  Future<void> recordProgress(Progress progress);
  Stream<List<Progress>> getProgressStream(String userId);
  Future<List<Progress>> getRecentProgress(String userId, int limit);
  
  // Mission operations
  Stream<List<Mission>> getActiveMissions(String userId);
  Future<void> updateMission(String missionId, Map<String, dynamic> data);
  
  // Leaderboard operations
  Future<List<LeaderboardEntry>> getTopUsers(int limit);
  Future<LeaderboardEntry> getUserRank(String userId);
}
```

**CacheRepository**
```dart
class CacheRepository {
  final Box _box;
  
  Future<void> cacheQuestion(Question question);
  Question? getCachedQuestion(String questionId);
  Future<void> cacheUserData(UserModel user);
  UserModel? getCachedUser(String userId);
  Future<void> clearCache();
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property Reflection

After analyzing all acceptance criteria, I've identified several areas where properties can be consolidated:

**Consolidation Opportunities:**
1. **Round-trip properties** (1.4, 7.4, 12.5): All test data persistence to Firestore. Can be combined into a single comprehensive persistence property.
2. **XP calculation properties** (7.1, 7.2): Both test XP calculation with different factors. Can be combined into one property covering all XP calculation factors.
3. **Analytics calculation properties** (11.2, 11.3, 11.4): All test aggregation of progress data. Can be combined into a comprehensive analytics calculation property.
4. **Leaderboard properties** (10.1, 10.4): Both test leaderboard ranking logic. Can be combined into one comprehensive ranking property.
5. **Error handling properties** (14.1, 14.3): Both test error message generation. Can be combined into one property about error responses.

**Properties to Keep Separate:**
- Authentication flows (different providers have different logic)
- Game mode behaviors (each mode has unique mechanics)
- Adaptive difficulty (different triggers for increase/decrease)
- Badge awards (milestone-based, needs specific examples)

### Correctness Properties

**Property 1: Authentication Round-Trip**
*For any* valid email and password combination, creating an account then logging in with those credentials should successfully authenticate the user and return their profile data.
**Validates: Requirements 1.1, 1.3**

**Property 2: Profile Data Persistence**
*For any* user profile update (XP, level, streak, badges, mastery scores), writing the update to Firestore then reading it back should return the updated values.
**Validates: Requirements 1.4, 7.4, 12.5**

**Property 3: Profile Completeness**
*For any* authenticated user, their profile should contain all required fields: uid, email, XP, level, streak, topicMastery map, and earnedBadges array.
**Validates: Requirements 1.5**

**Property 4: Invalid Credentials Rejection**
*For any* invalid email/password combination, authentication attempts should fail with a descriptive error message and prevent access.
**Validates: Requirements 1.6**

**Property 5: Question Structure Completeness**
*For any* generated question, it should contain all required fields: problemStatement, inputExamples, outputExamples, constraints, difficulty, topic, and xpReward, all with non-empty values.
**Validates: Requirements 2.2**

**Property 6: Question Difficulty Appropriateness**
*For any* question generation request, the returned question's difficulty should be within ±1 level of the user's current mastery score for that topic.
**Validates: Requirements 2.1**

**Property 7: Incorrect Answer Explanation**
*For any* incorrect answer submission, the system should return a non-empty explanation containing both error identification and correct solution guidance.
**Validates: Requirements 3.1**

**Property 8: Explanation Code Examples**
*For any* explanation generated for an incorrect answer, the explanation text should contain code examples (identifiable by code block markers or syntax patterns).
**Validates: Requirements 3.3**

**Property 9: Topic Accuracy Calculation**
*For any* set of progress records for a specific topic, the calculated accuracy percentage should equal (correct answers / total answers) * 100.
**Validates: Requirements 4.1**

**Property 10: Per-Topic Difficulty Independence**
*For any* user with progress in multiple topics, changing the difficulty for one topic should not affect the difficulty levels of other topics.
**Validates: Requirements 4.4**

**Property 11: Difficulty Change Notification**
*For any* difficulty adjustment (increase or decrease), a notification event should be triggered with the topic and new difficulty level.
**Validates: Requirements 4.5**

**Property 12: Weak Topic Identification**
*For any* user's topic mastery scores, topics with mastery below 60% should be identified as weak topics and included in the learning roadmap.
**Validates: Requirements 5.1, 5.2**

**Property 13: Roadmap Priority Ordering**
*For any* learning roadmap, weak topics should be ordered by ascending mastery score (lowest mastery first).
**Validates: Requirements 5.3**

**Property 14: XP Calculation Completeness**
*For any* completed challenge, the awarded XP should be calculated based on difficulty, time spent, and current streak, with higher difficulty and active streaks increasing XP.
**Validates: Requirements 6.6, 7.1, 7.2**

**Property 15: Level-Up Threshold**
*For any* user level N, when accumulated XP reaches or exceeds 100 * ((N+1) ^ 1.5), the user's level should increment to N+1.
**Validates: Requirements 7.3, 7.6**

**Property 16: XP Progress Calculation**
*For any* user with current XP and level, the progress percentage toward next level should equal (current XP / XP required for next level) * 100.
**Validates: Requirements 7.5**

**Property 17: Streak Increment on Daily Activity**
*For any* user who completes at least one challenge in a calendar day (UTC), their streak should increment by 1 if the previous activity was yesterday, or reset to 1 if there was a gap.
**Validates: Requirements 8.1**

**Property 18: Mission Completion Rewards**
*For any* completed mission, the mission should be marked as complete (isCompleted = true) and the specified XP reward should be added to the user's total XP.
**Validates: Requirements 9.2**

**Property 19: Weekly Goal Generation Relevance**
*For any* generated weekly goals, at least 50% of the goals should target topics from the user's weak topics list.
**Validates: Requirements 9.3**

**Property 20: Leaderboard Ranking Correctness**
*For any* set of users, the leaderboard should be ordered by total XP in descending order, with each user's rank equal to their position in this ordering (1-indexed).
**Validates: Requirements 10.1, 10.4**

**Property 21: Leaderboard Real-Time Updates**
*For any* user XP change, the leaderboard should reflect the updated XP and recalculated rank within the next query.
**Validates: Requirements 10.2**

**Property 22: Leaderboard Result Limiting**
*For any* leaderboard query, the returned results should contain at most 100 users.
**Validates: Requirements 10.3**

**Property 23: Leaderboard Privacy Controls**
*For any* leaderboard query, only users with leaderboardOptIn = true should appear in the public results.
**Validates: Requirements 10.5**

**Property 24: Analytics Data Accuracy**
*For any* user's progress records, the calculated analytics (accuracy percentage, average time, total time) should match the aggregated values from the raw progress data.
**Validates: Requirements 11.1, 11.2, 11.3, 11.4**

**Property 25: Weak Topic Highlighting**
*For any* analytics dashboard data, topics with mastery score below 60% should be flagged as weak topics.
**Validates: Requirements 11.5**

**Property 26: Real-Time Analytics Updates**
*For any* new progress record added, querying analytics immediately after should include the new record in all calculations.
**Validates: Requirements 11.6**

**Property 27: Badge Award on Milestone**
*For any* user reaching a defined milestone (first challenge, 10 challenges, 100 challenges, 7-day streak, 30-day streak, level 10, topic mastery), the corresponding badge should be added to their earnedBadges array.
**Validates: Requirements 12.1**

**Property 28: Badge Persistence**
*For any* earned badge, it should appear in the user's earnedBadges array and persist across sessions.
**Validates: Requirements 12.3**

**Property 29: API Error Response**
*For any* failed API request, the system should return an error response containing a user-friendly message describing the failure reason.
**Validates: Requirements 14.1, 14.3**

**Property 30: Error Logging**
*For any* error that occurs in the system, an error log entry should be created with timestamp, error type, and context information.
**Validates: Requirements 14.4**

**Property 31: Cache Hit Reduces Network Requests**
*For any* data request where valid cached data exists, the system should return cached data without making a network request to Firestore.
**Validates: Requirements 15.3**

**Property 32: Firestore Security Rules Enforcement**
*For any* unauthorized data access attempt (user trying to read/write another user's data), the request should be rejected with a permission denied error.
**Validates: Requirements 16.3**

**Property 33: Input Validation**
*For any* user input containing potentially malicious content (SQL injection patterns, script tags, etc.), the validation should reject the input before processing.
**Validates: Requirements 16.4**

**Property 34: Touch Target Minimum Size**
*For any* interactive UI element (button, link, input), the touch target size should be at least 44x44 pixels.
**Validates: Requirements 18.5**

## Error Handling

### Error Categories

**Authentication Errors:**
- Invalid credentials: Return specific error code and message
- Network failures: Retry with exponential backoff (3 attempts)
- OAuth failures: Display provider-specific error message

**API Errors:**
- OpenAI API failures: Retry up to 3 times with exponential backoff
- Rate limiting: Return 429 status with retry-after header
- Timeout: Return 408 status with timeout message
- Validation errors: Return 400 status with field-specific error details

**Data Errors:**
- Firestore permission denied: Return 403 with clear message
- Document not found: Return 404 with resource identifier
- Write conflicts: Retry with transaction
- Quota exceeded: Return 429 with quota information

**Client Errors:**
- Network offline: Queue operations locally, sync when online
- Cache corruption: Clear cache and refetch data
- Invalid state: Reset to safe default state

### Error Response Format

```dart
class ErrorResponse {
  final String code;
  final String message;
  final String? userMessage;
  final Map<String, dynamic>? details;
  final bool isRetryable;
  
  ErrorResponse({
    required this.code,
    required this.message,
    this.userMessage,
    this.details,
    this.isRetryable = false,
  });
}
```

### Retry Strategy

```dart
class RetryPolicy {
  static const maxAttempts = 3;
  static const baseDelay = Duration(seconds: 1);
  
  static Future<T> executeWithRetry<T>(
    Future<T> Function() operation,
    bool Function(dynamic error) shouldRetry,
  ) async {
    int attempt = 0;
    while (true) {
      try {
        return await operation();
      } catch (e) {
        attempt++;
        if (attempt >= maxAttempts || !shouldRetry(e)) {
          rethrow;
        }
        await Future.delayed(baseDelay * pow(2, attempt - 1));
      }
    }
  }
}
```

### Error Logging

All errors are logged with structured data:
```dart
class ErrorLogger {
  static void logError(
    dynamic error,
    StackTrace stackTrace, {
    String? context,
    Map<String, dynamic>? metadata,
  }) {
    final logEntry = {
      'timestamp': DateTime.now().toIso8601String(),
      'error': error.toString(),
      'stackTrace': stackTrace.toString(),
      'context': context,
      'metadata': metadata,
      'userId': getCurrentUserId(),
      'platform': Platform.operatingSystem,
      'appVersion': getAppVersion(),
    };
    
    // Send to Firebase Crashlytics
    FirebaseCrashlytics.instance.recordError(error, stackTrace);
    
    // Send to custom logging service
    LoggingService.instance.log(logEntry);
  }
}
```

## Testing Strategy

### Dual Testing Approach

CodeQuest employs both unit testing and property-based testing for comprehensive coverage:

**Unit Tests** focus on:
- Specific examples of core functionality
- Edge cases (empty inputs, boundary values, null handling)
- Error conditions and exception handling
- Integration points between components
- UI widget rendering and interactions

**Property-Based Tests** focus on:
- Universal properties that hold for all inputs
- Data integrity across operations
- Calculation correctness across ranges
- State consistency after operations
- Round-trip properties for serialization/deserialization

### Property-Based Testing Configuration

**Library:** We'll use the `test` package with custom property-based testing utilities, or integrate `dart_check` for more advanced property testing.

**Configuration:**
- Minimum 100 iterations per property test
- Each test tagged with feature name and property number
- Tag format: `@Tags(['feature:codequest-app', 'property:1'])`

**Example Property Test Structure:**
```dart
@Tags(['feature:codequest-app', 'property:2'])
test('Property 2: Profile Data Persistence', () async {
  // Run 100 iterations with random data
  for (int i = 0; i < 100; i++) {
    // Generate random user profile update
    final userId = generateRandomUserId();
    final update = generateRandomProfileUpdate();
    
    // Write update
    await firestoreRepo.updateUser(userId, update);
    
    // Read back
    final retrieved = await firestoreRepo.getUser(userId);
    
    // Verify all fields match
    expect(retrieved.xp, equals(update['xp']));
    expect(retrieved.level, equals(update['level']));
    expect(retrieved.streak, equals(update['streak']));
    // ... verify all fields
  }
});
```

### Test Organization

```
test/
  unit/
    auth/
      auth_service_test.dart
      auth_repository_test.dart
    game/
      game_service_test.dart
      question_generator_test.dart
    analytics/
      analytics_service_test.dart
      mastery_calculator_test.dart
  property/
    auth_properties_test.dart
    game_properties_test.dart
    analytics_properties_test.dart
    leaderboard_properties_test.dart
  integration/
    game_flow_test.dart
    mission_completion_test.dart
  widget/
    dashboard_widget_test.dart
    game_mode_widget_test.dart
```

### Coverage Goals

- Unit test coverage: >80% of business logic
- Property test coverage: All 34 correctness properties
- Integration test coverage: All critical user flows
- Widget test coverage: All major UI components

### Continuous Testing

- Run unit tests on every commit
- Run property tests on pull requests
- Run integration tests before deployment
- Monitor test execution time and optimize slow tests

## Security Design

### Authentication Security

**Firebase Authentication** handles:
- Password hashing (bcrypt with salt)
- OAuth token management
- Session management
- Account recovery

**Additional Security Measures:**
- Email verification required before full access
- Password strength requirements (min 8 chars, mixed case, numbers)
- Rate limiting on login attempts (5 attempts per 15 minutes)
- Session timeout after 30 days of inactivity

### Data Access Control

**Firestore Security Rules:**

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users can only read/write their own data
    match /users/{userId} {
      allow read: if request.auth != null && request.auth.uid == userId;
      allow write: if request.auth != null && request.auth.uid == userId;
    }
    
    // Progress records are user-specific
    match /progress/{progressId} {
      allow read: if request.auth != null && 
                     resource.data.userId == request.auth.uid;
      allow create: if request.auth != null && 
                       request.resource.data.userId == request.auth.uid;
    }
    
    // Missions are user-specific
    match /missions/{missionId} {
      allow read, update: if request.auth != null && 
                             resource.data.userId == request.auth.uid;
    }
    
    // Leaderboard is read-only for clients
    match /leaderboard/{userId} {
      allow read: if request.auth != null;
      allow write: if false; // Only Cloud Functions can write
    }
    
    // Questions cache is read-only for clients
    match /questions_cache/{questionId} {
      allow read: if request.auth != null;
      allow write: if false; // Only Cloud Functions can write
    }
  }
}
```

### API Security

**Cloud Functions Security:**
- All functions require authentication
- API keys stored in environment variables
- OpenAI API key never exposed to client
- Rate limiting per user (100 requests per hour)
- Input validation on all endpoints
- CORS configured for specific origins only

**Input Validation:**
```dart
class InputValidator {
  static bool isValidEmail(String email) {
    return RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(email);
  }
  
  static bool isValidAnswer(String answer) {
    // Sanitize and validate code input
    if (answer.length > 10000) return false;
    if (containsMaliciousPatterns(answer)) return false;
    return true;
  }
  
  static bool containsMaliciousPatterns(String input) {
    final maliciousPatterns = [
      RegExp(r'<script', caseSensitive: false),
      RegExp(r'javascript:', caseSensitive: false),
      RegExp(r'on\w+\s*=', caseSensitive: false),
    ];
    return maliciousPatterns.any((pattern) => pattern.hasMatch(input));
  }
}
```

### Data Privacy

**GDPR Compliance:**
- User consent for data collection
- Right to access personal data
- Right to delete account and all data
- Data export functionality
- Privacy policy and terms of service

**Account Deletion:**
```dart
Future<void> deleteUserAccount(String userId) async {
  // Delete from Authentication
  await FirebaseAuth.instance.currentUser?.delete();
  
  // Delete user document
  await FirebaseFirestore.instance.collection('users').doc(userId).delete();
  
  // Delete all progress records
  final progressQuery = await FirebaseFirestore.instance
      .collection('progress')
      .where('userId', isEqualTo: userId)
      .get();
  for (var doc in progressQuery.docs) {
    await doc.reference.delete();
  }
  
  // Delete all missions
  final missionsQuery = await FirebaseFirestore.instance
      .collection('missions')
      .where('userId', isEqualTo: userId)
      .get();
  for (var doc in missionsQuery.docs) {
    await doc.reference.delete();
  }
  
  // Remove from leaderboard
  await FirebaseFirestore.instance
      .collection('leaderboard')
      .doc(userId)
      .delete();
}
```

## Performance Optimization

### Caching Strategy

**Local Cache (Hive):**
- User profile data (TTL: 5 minutes)
- Recent questions (TTL: 1 hour)
- Analytics data (TTL: 5 minutes)
- Leaderboard snapshot (TTL: 1 minute)

**Cache Implementation:**
```dart
class CacheManager {
  static const userCacheTTL = Duration(minutes: 5);
  static const questionCacheTTL = Duration(hours: 1);
  
  Future<UserModel?> getCachedUser(String userId) async {
    final box = await Hive.openBox('users');
    final cached = box.get(userId);
    
    if (cached == null) return null;
    
    final cacheTime = DateTime.parse(cached['timestamp']);
    if (DateTime.now().difference(cacheTime) > userCacheTTL) {
      await box.delete(userId);
      return null;
    }
    
    return UserModel.fromJson(cached['data']);
  }
  
  Future<void> cacheUser(String userId, UserModel user) async {
    final box = await Hive.openBox('users');
    await box.put(userId, {
      'timestamp': DateTime.now().toIso8601String(),
      'data': user.toJson(),
    });
  }
}
```

**Firestore Query Optimization:**
- Use composite indexes for complex queries
- Limit query results (pagination)
- Use `where` clauses to filter at database level
- Avoid reading entire collections

### Lazy Loading

**Dashboard:**
- Load user profile immediately
- Lazy load analytics graphs on scroll
- Lazy load leaderboard on tab switch
- Lazy load mission details on expansion

**Game Modes:**
- Preload next question while user answers current
- Load game mode assets on demand
- Cache frequently used code blocks

### Image Optimization

- Use WebP format for images
- Implement progressive loading
- Cache badge icons locally
- Use CDN for static assets

### Network Optimization

**Request Batching:**
```dart
class BatchRequestManager {
  final List<Future Function()> _pendingRequests = [];
  Timer? _batchTimer;
  
  void addRequest(Future Function() request) {
    _pendingRequests.add(request);
    
    _batchTimer?.cancel();
    _batchTimer = Timer(Duration(milliseconds: 100), _executeBatch);
  }
  
  Future<void> _executeBatch() async {
    final requests = List.from(_pendingRequests);
    _pendingRequests.clear();
    
    await Future.wait(requests.map((r) => r()));
  }
}
```

**Compression:**
- Enable gzip compression for API responses
- Compress large payloads before transmission

## Deployment Architecture

### Development Environment

```
Flutter App (Local)
    ↓
Firebase Emulator Suite
    ├── Auth Emulator
    ├── Firestore Emulator
    └── Functions Emulator
```

### Production Environment

```
Flutter App (iOS/Android)
    ↓
Firebase Production
    ├── Firebase Authentication
    ├── Cloud Firestore
    ├── Cloud Functions (Node.js)
    │   └── OpenAI API Integration
    ├── Firebase Hosting (Web Assets)
    └── Firebase Crashlytics
```

### CI/CD Pipeline

**GitHub Actions Workflow:**
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter test
      - run: flutter analyze
      
  deploy-functions:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
      - run: npm install -g firebase-tools
      - run: firebase deploy --only functions
      
  build-android:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: subosito/flutter-action@v2
      - run: flutter build apk --release
      - uses: actions/upload-artifact@v2
        with:
          name: android-release
          path: build/app/outputs/flutter-apk/app-release.apk
```

### Environment Configuration

**Development:**
- Firebase project: codequest-dev
- OpenAI API: Development key with lower rate limits
- Debug logging enabled

**Production:**
- Firebase project: codequest-prod
- OpenAI API: Production key with higher rate limits
- Error logging only
- Analytics enabled

### Monitoring and Observability

**Firebase Crashlytics:**
- Automatic crash reporting
- Custom error logging
- User session tracking

**Firebase Performance Monitoring:**
- App startup time
- Screen rendering performance
- Network request latency
- Custom traces for critical operations

**Cloud Functions Monitoring:**
- Execution time
- Error rate
- Invocation count
- Memory usage

**Alerts:**
- Error rate > 5%
- API latency > 5 seconds
- Crash rate > 1%
- OpenAI API quota approaching limit

## Conclusion

This design document provides a comprehensive technical blueprint for CodeQuest, an AI-powered gamified programming learning platform. The architecture leverages Firebase's serverless infrastructure for scalability, implements robust security measures, and employs property-based testing for correctness verification.

Key design decisions:
- **Serverless architecture** for automatic scaling and reduced operational overhead
- **Real-time synchronization** for seamless cross-device experience
- **Aggressive caching** for optimal performance
- **Comprehensive testing strategy** combining unit and property-based tests
- **Security-first approach** with Firestore rules and input validation
- **Modular component design** for maintainability and extensibility

The system is designed to be built incrementally during a hackathon while maintaining production-ready quality and scalability for future growth.
