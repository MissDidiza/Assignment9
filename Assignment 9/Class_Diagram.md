# CLASS_DIAGRAM.md — CampusFind: Smart Campus Lost & Found System
## Assignment 9: Class Diagram in Mermaid.js

---

## 1. Full Class Diagram

```mermaid
classDiagram

    %% ─── ENUMERATIONS ───────────────────────────────────────────

    class UserRole {
        <<enumeration>>
        STUDENT
        ADMIN
        SUPER_ADMIN
    }

    class ReportType {
        <<enumeration>>
        LOST
        FOUND
    }

    class ReportStatus {
        <<enumeration>>
        OPEN
        MATCHING
        MATCH_FOUND
        MATCHED
        HANDOVER_PENDING
        RESOLVED
        ESCALATED
        UNCLAIMED
        ARCHIVED
    }

    class ItemCategory {
        <<enumeration>>
        ELECTRONICS
        CLOTHING
        ACCESSORIES
        DOCUMENTS
        KEYS
        OTHER
    }

    class MatchStatus {
        <<enumeration>>
        GENERATED
        NOTIFIED
        PENDING_REVIEW
        CONFIRMED
        DISMISSED
        STALE
        CLOSED
    }

    class HandoverStatus {
        <<enumeration>>
        INITIATED
        AWAITING_COLLECTION
        REMINDER_1_SENT
        REMINDER_2_SENT
        ESCALATED
        COLLECTED
        UNCLAIMED
        CLOSED
    }

    class NotificationType {
        <<enumeration>>
        MATCH_ALERT
        HANDOVER_SCHEDULED
        REMINDER
        DAILY_DIGEST
        PASSWORD_RESET
        VERIFICATION
    }

    class NotificationStatus {
        <<enumeration>>
        QUEUED
        SENDING
        DELIVERED
        PARTIALLY_DELIVERED
        FAILED
        READ
        EXPIRED
    }

    class NotificationChannel {
        <<enumeration>>
        EMAIL
        IN_APP
        BOTH
    }

    %% ─── CORE CLASSES ────────────────────────────────────────────

    class User {
        -userId : UUID
        -fullName : String
        -email : String
        -passwordHash : String
        -role : UserRole
        -isVerified : Boolean
        -emailNotificationsEnabled : Boolean
        -consentTimestamp : DateTime
        -createdAt : DateTime
        -updatedAt : DateTime
        +register(name, email, password) Boolean
        +verifyEmail(token) Boolean
        +login(email, password) Session
        +logout(sessionId) void
        +resetPassword(token, newPassword) Boolean
        +updateProfile(data) User
        +deactivate() void
        +isAdmin() Boolean
        +isSuperAdmin() Boolean
    }

    class Report {
        -reportId : UUID
        -userId : UUID
        -type : ReportType
        -itemName : String
        -category : ItemCategory
        -description : String
        -location : String
        -dateLostOrFound : Date
        -status : ReportStatus
        -createdAt : DateTime
        -updatedAt : DateTime
        +submit() Report
        +validate() Boolean
        +updateStatus(newStatus) void
        +triggerMatching() void
        +archive() void
        +softDelete() void
        +isEditable() Boolean
        +toMatchPayload() Object
    }

    class Photo {
        -photoId : UUID
        -reportId : UUID
        -cloudinaryUrl : String
        -cloudinaryPublicId : String
        -fileSizeKB : Integer
        -mimeType : String
        -uploadedAt : DateTime
        -isDeleted : Boolean
        +upload(file) Photo
        +validate() Boolean
        +delete() void
        +fetchForAnalysis() String
    }

    class MatchRecord {
        -matchId : UUID
        -lostReportId : UUID
        -foundReportId : UUID
        -textSimilarityScore : Float
        -imageSimilarityScore : Float
        -confidenceScore : Float
        -status : MatchStatus
        -adminId : UUID
        -dismissalReason : String
        -createdAt : DateTime
        -updatedAt : DateTime
        +generate(lostReport, foundReport) MatchRecord
        +calculateConfidence() Float
        +notify() void
        +confirm(adminId) void
        +dismiss(adminId, reason) void
        +markStale() void
        +close() void
        +meetsThreshold() Boolean
    }

    class Handover {
        -handoverId : UUID
        -matchId : UUID
        -lostReportId : UUID
        -foundReportId : UUID
        -adminId : UUID
        -studentId : UUID
        -pickupLocation : String
        -pickupWindowStart : DateTime
        -pickupWindowEnd : DateTime
        -status : HandoverStatus
        -confirmedAt : DateTime
        -isManualOverride : Boolean
        -overrideReason : String
        -createdAt : DateTime
        +initiate(matchId, adminId) Handover
        +notifyStudent() void
        +sendReminder(reminderNumber) void
        +recordCollection() void
        +recordManualOverride(reason) void
        +escalate() void
        +markUnclaimed() void
        +close() void
        +getDaysSinceInitiated() Integer
    }

    class Notification {
        -notificationId : UUID
        -userId : UUID
        -type : NotificationType
        -channel : NotificationChannel
        -subject : String
        -body : String
        -isRead : Boolean
        -status : NotificationStatus
        -retryCount : Integer
        -relatedEntityId : UUID
        -createdAt : DateTime
        -readAt : DateTime
        +send() void
        +retry() void
        +markRead() void
        +markExpired() void
        +logFailure(error) void
        +canRetry() Boolean
    }

    class Session {
        -sessionId : UUID
        -userId : UUID
        -tokenHash : String
        -issuedAt : DateTime
        -expiresAt : DateTime
        -isRevoked : Boolean
        -revokedAt : DateTime
        -ipAddress : String
        -userAgent : String
        +issue(userId) Session
        +validate() Boolean
        +revoke() void
        +isExpired() Boolean
        +getRemainingTTL() Integer
    }

    %% ─── SERVICE CLASSES ─────────────────────────────────────────

    class AuthService {
        <<service>>
        -userRepository : UserRepository
        -sessionRepository : SessionRepository
        -emailService : EmailService
        +register(data) User
        +verifyEmail(token) Boolean
        +login(email, password) Session
        +logout(sessionId) void
        +sendPasswordReset(email) void
        +resetPassword(token, newPassword) Boolean
        +validateToken(token) Session
    }

    class ReportService {
        <<service>>
        -reportRepository : ReportRepository
        -photoService : PhotoService
        -queueService : QueueService
        +createReport(userId, data, files) Report
        +getReportsByUser(userId) Report[]
        +getOpenReportsByType(type) Report[]
        +updateReport(reportId, data) Report
        +deleteReport(reportId) void
        +archiveExpiredReports() void
    }

    class PhotoService {
        <<service>>
        -cloudinaryClient : CloudinaryClient
        +uploadPhotos(files, reportId) Photo[]
        +deletePhoto(photoId) void
        +deletePhotosByReport(reportId) void
        +validatePhoto(file) Boolean
    }

    class MatchingService {
        <<service>>
        -aiClient : AIMatchingClient
        -matchRepository : MatchRepository
        -notificationService : NotificationService
        +runMatching(newReport) MatchRecord[]
        +analyzeReports(lost, found) Float
        +persistMatches(matches) void
        +triggerNotifications(matches) void
    }

    class NotificationService {
        <<service>>
        -notificationRepository : NotificationRepository
        -emailClient : SendGridClient
        +sendMatchAlert(userId, matchId) void
        +sendHandoverNotification(userId, handoverId) void
        +sendReminder(userId, handoverId) void
        +sendDailyDigest(adminId) void
        +retryFailed() void
    }

    class HandoverService {
        <<service>>
        -handoverRepository : HandoverRepository
        -notificationService : NotificationService
        +initiate(matchId, adminId) Handover
        +schedulePickup(handoverId, location, window) void
        +recordCollection(handoverId) void
        +recordManualOverride(handoverId, reason) void
        +processReminders() void
        +escalateOverdue() void
    }

    class StatisticsService {
        <<service>>
        -reportRepository : ReportRepository
        -cacheClient : RedisClient
        +getRecoveryRate(dateRange) Float
        +getTopLocations(dateRange) String[]
        +getTopCategories(dateRange) String[]
        +getAvgResolutionTime(dateRange) Float
        +exportCSV(dateRange) File
    }

    %% ─── RELATIONSHIPS ───────────────────────────────────────────

    %% Enumerations used by classes
    User --> UserRole : uses
    Report --> ReportType : uses
    Report --> ReportStatus : uses
    Report --> ItemCategory : uses
    MatchRecord --> MatchStatus : uses
    Handover --> HandoverStatus : uses
    Notification --> NotificationType : uses
    Notification --> NotificationStatus : uses
    Notification --> NotificationChannel : uses

    %% Core domain relationships
    User "1" --> "0..*" Report : submits
    User "1" --> "0..*" Session : authenticates via
    User "1" --> "0..*" Notification : receives

    Report "1" *-- "0..3" Photo : contains
    Report "1" --> "0..*" MatchRecord : participates in as lost
    Report "1" --> "0..*" MatchRecord : participates in as found

    MatchRecord "1" --> "0..1" Handover : leads to
    MatchRecord "1" --> "1..*" Notification : triggers

    Handover "1" --> "1..*" Notification : triggers
    Handover "1" --> "1" User : managed by admin
    Handover "1" --> "1" User : collected by student

    %% Service dependencies
    AuthService ..> User : manages
    AuthService ..> Session : manages
    ReportService ..> Report : manages
    ReportService ..> PhotoService : uses
    PhotoService ..> Photo : manages
    MatchingService ..> MatchRecord : manages
    MatchingService ..> Report : reads
    MatchingService ..> NotificationService : uses
    HandoverService ..> Handover : manages
    HandoverService ..> NotificationService : uses
    NotificationService ..> Notification : manages
    StatisticsService ..> Report : reads
```

---

## 2. Key Design Decisions

### 2.1 Composition vs. Association for Photo and Report

Photo uses **composition** with Report (`*--`) rather than simple association. This is because a Photo cannot exist independently of a Report — if a Report is deleted, all its Photos must also be deleted from Cloudinary. Composition enforces this lifecycle dependency at the design level, making it clear to any developer reading the diagram that Photo management is always the responsibility of the Report that owns it.

### 2.2 Service Layer as Separate Classes

The diagram separates domain entities (User, Report, Photo, MatchRecord, Handover, Notification, Session) from service classes (AuthService, ReportService, MatchingService, etc.). This reflects a **layered architecture** pattern where domain entities hold data and basic self-validation logic, while services hold business workflows that coordinate between multiple entities. This separation makes each class easier to test in isolation and easier to extend without breaking other parts of the system.

### 2.3 Enumerations for All Constrained Values

Every attribute with a fixed set of valid values (status fields, role, type, category, channel) is modelled as an enumeration class. This prevents invalid state values from entering the system and makes the valid state machine explicit in the class diagram itself, directly linking to the state transition diagrams from Assignment 8.

### 2.4 MatchRecord References Both Reports

The MatchRecord holds explicit foreign key references to both `lostReportId` and `foundReportId`. This is a deliberate denormalisation — while the Report entity could theoretically be navigated from the MatchRecord through its relationship lines, storing both IDs directly on the MatchRecord makes match queries significantly faster (no joins required) and makes the audit trail self-contained. This was a performance-over-normalisation trade-off justified by NFR-08 (matching must complete in under 60 seconds).

### 2.5 Alignment with Assignment 8 State Diagrams

Every `status` enumeration in this class diagram corresponds directly to the states defined in the Assignment 8 state transition diagrams. `ReportStatus` values map exactly to the Lost Item Report state diagram. `MatchStatus` maps to the AI Match Record state diagram. `HandoverStatus` maps to the Handover Record state diagram. This ensures that the class diagram and the behavioural models describe the same system without contradiction.

### 2.6 Traceability to Requirements

| Class | Key Requirements | User Stories |
|---|---|---|
| User | FR-01, FR-02, FR-12 | US-001, US-002, US-012 |
| Report | FR-03, FR-04, FR-05 | US-003, US-004, US-005 |
| Photo | FR-03, FR-04, FR-11 | US-003, US-004, US-011 |
| MatchRecord | FR-05, FR-07 | US-005, US-007 |
| Handover | FR-08 | US-008 |
| Notification | FR-06 | US-006 |
| Session | FR-02, NFR-09 | US-002 |