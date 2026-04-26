# DOMAIN_MODEL.md — CampusFind: Smart Campus Lost & Found System
## Assignment 9: Domain Model

---

## 1. Overview

The CampusFind domain model identifies the core entities that exist in the system, their attributes, responsibilities, and the relationships between them. This model is derived directly from the functional requirements (Assignment 4), use cases (Assignment 5), and state diagrams (Assignment 8).

---

## 2. Domain Entities

### Entity 1: User

| Field | Detail |
|---|---|
| **Description** | Represents any registered person in the system — a student, staff finder, admin, or super admin. All system interactions require an authenticated User. |
| **Attributes** | `userId: UUID`, `fullName: String`, `email: String`, `passwordHash: String`, `role: Enum[STUDENT, ADMIN, SUPER_ADMIN]`, `isVerified: Boolean`, `consentTimestamp: DateTime`, `createdAt: DateTime`, `updatedAt: DateTime` |
| **Methods** | `register()`, `verifyEmail()`, `login()`, `logout()`, `resetPassword()`, `updateProfile()`, `deactivate()` |
| **Relationships** | Submits zero or many Reports. Owns zero or many Notifications. Has zero or one active Session. Super admin manages other Users. |

**Business Rules:**
- A user's email must belong to the university domain (e.g., `@university.ac.za`).
- A user cannot log in until `isVerified = true`.
- A STUDENT cannot access any endpoint reserved for ADMIN or SUPER_ADMIN roles.
- A user's personal data must be deleted 120 days after account deactivation (POPIA).

---

### Entity 2: Report

| Field | Detail |
|---|---|
| **Description** | The central entity in the system. Represents either a lost item report or a found item report submitted by a User. All matching, notification, and handover activity revolves around pairs of Reports. |
| **Attributes** | `reportId: UUID`, `userId: UUID`, `type: Enum[LOST, FOUND]`, `itemName: String`, `category: Enum[ELECTRONICS, CLOTHING, ACCESSORIES, DOCUMENTS, KEYS, OTHER]`, `description: String`, `location: String`, `dateLostOrFound: Date`, `status: Enum[OPEN, MATCHING, MATCH_FOUND, MATCHED, HANDOVER_PENDING, RESOLVED, ESCALATED, UNCLAIMED, ARCHIVED]`, `createdAt: DateTime`, `updatedAt: DateTime` |
| **Methods** | `submit()`, `validate()`, `updateStatus()`, `archive()`, `delete()`, `triggerMatching()` |
| **Relationships** | Belongs to one User. Contains one or many Photos. Participates in zero or many Matches (as either the lost or found report). Can be linked to one Handover. |

**Business Rules:**
- A Report's `description` must be at least 20 characters.
- A Report can have a maximum of 3 Photos.
- A Report cannot be edited once its status is MATCHED or beyond.
- A Report is automatically archived 90 days after reaching RESOLVED or UNCLAIMED status.
- Only the Report's owner or an ADMIN can change its status.

---

### Entity 3: Photo

| Field | Detail |
|---|---|
| **Description** | Represents an image file uploaded with a Report. Photos are stored externally in Cloudinary; the system stores only the URL and metadata. Photos are used by the AI Matching Service for image similarity analysis. |
| **Attributes** | `photoId: UUID`, `reportId: UUID`, `cloudinaryUrl: String`, `cloudinaryPublicId: String`, `fileSizeKB: Integer`, `uploadedAt: DateTime`, `isDeleted: Boolean` |
| **Methods** | `upload()`, `validate()`, `delete()`, `fetchForAnalysis()` |
| **Relationships** | Belongs to one Report. Referenced by the MatchRecord during AI image analysis. |

**Business Rules:**
- Maximum file size is 5 MB per photo.
- Only image MIME types (jpeg, png, webp) are accepted.
- When a Report is archived and personal data is deleted, all associated Photos must be deleted from Cloudinary and `isDeleted` set to true.
- A Report can have between 0 and 3 Photos.

---

### Entity 4: MatchRecord

| Field | Detail |
|---|---|
| **Description** | Represents the AI system's suggestion that a specific lost Report and a specific found Report refer to the same item. Created automatically by the AI Matching Service; confirmed or dismissed by an Admin. |
| **Attributes** | `matchId: UUID`, `lostReportId: UUID`, `foundReportId: UUID`, `textSimilarityScore: Float`, `imageSimilarityScore: Float`, `confidenceScore: Float`, `status: Enum[GENERATED, NOTIFIED, PENDING_REVIEW, CONFIRMED, DISMISSED, STALE, CLOSED]`, `adminId: UUID`, `dismissalReason: String`, `createdAt: DateTime`, `updatedAt: DateTime` |
| **Methods** | `generate()`, `notify()`, `confirm()`, `dismiss()`, `markStale()`, `close()` |
| **Relationships** | References one lost Report and one found Report. If confirmed, is linked to one Handover. Triggers one or many Notifications. Reviewed by one Admin (User). |

**Business Rules:**
- Only matches with `confidenceScore >= 70%` are persisted and surfaced to admins.
- `confidenceScore = (textSimilarityScore × 0.6) + (imageSimilarityScore × 0.4)`.
- A MatchRecord cannot be confirmed if either associated Report has already been RESOLVED by another path.
- A dismissed MatchRecord must have a `dismissalReason` logged.
- The confidence threshold (70%) is configurable by Super Admin.

---

### Entity 5: Handover

| Field | Detail |
|---|---|
| **Description** | Represents the physical process of returning a found item to its owner. Created when an Admin confirms a MatchRecord. Contains the full audit trail of the handover event. |
| **Attributes** | `handoverId: UUID`, `matchId: UUID`, `lostReportId: UUID`, `foundReportId: UUID`, `adminId: UUID`, `studentId: UUID`, `pickupLocation: String`, `pickupWindowStart: DateTime`, `pickupWindowEnd: DateTime`, `status: Enum[INITIATED, AWAITING_COLLECTION, REMINDER_1_SENT, REMINDER_2_SENT, ESCALATED, COLLECTED, UNCLAIMED, CLOSED]`, `confirmedAt: DateTime`, `isManualOverride: Boolean`, `overrideReason: String`, `createdAt: DateTime` |
| **Methods** | `initiate()`, `notifyStudent()`, `sendReminder()`, `recordCollection()`, `recordManualOverride()`, `escalate()`, `close()` |
| **Relationships** | Linked to one MatchRecord. Involves one Admin (User) and one Student (User). Triggers one or many Notifications. Updates the status of two Reports. |

**Business Rules:**
- A Handover is created only when a MatchRecord transitions to CONFIRMED status.
- Reminder 1 is sent automatically 3 days after `status = AWAITING_COLLECTION`.
- Reminder 2 is sent automatically 7 days after `status = AWAITING_COLLECTION`.
- If uncollected after 7 days, status becomes ESCALATED and the admin is alerted.
- Every Handover must have a full audit trail: admin ID, student ID, timestamps, and confirmation method.

---

### Entity 6: Notification

| Field | Detail |
|---|---|
| **Description** | Represents a message sent to a User — either an in-app notification stored in the database or an email dispatched via SendGrid. Each notification event creates one Notification record. |
| **Attributes** | `notificationId: UUID`, `userId: UUID`, `type: Enum[MATCH_ALERT, HANDOVER_SCHEDULED, REMINDER, DAILY_DIGEST, PASSWORD_RESET, VERIFICATION]`, `channel: Enum[EMAIL, IN_APP, BOTH]`, `subject: String`, `body: String`, `isRead: Boolean`, `status: Enum[QUEUED, SENDING, DELIVERED, PARTIALLY_DELIVERED, FAILED, READ, EXPIRED]`, `retryCount: Integer`, `createdAt: DateTime`, `readAt: DateTime` |
| **Methods** | `send()`, `retry()`, `markRead()`, `markExpired()`, `logFailure()` |
| **Relationships** | Belongs to one User (recipient). Associated with one MatchRecord or one Handover (the event that triggered it). |

**Business Rules:**
- A maximum of 3 retry attempts are made for failed email notifications.
- If email delivery fails after 3 retries, the IT admin is alerted.
- Unread notifications expire after 30 days.
- Students can opt out of email notifications; in-app notifications cannot be disabled.

---

### Entity 7: Session

| Field | Detail |
|---|---|
| **Description** | Represents an authenticated user session, backed by a JWT token. The Session entity tracks active tokens and supports revocation via a Redis denylist. |
| **Attributes** | `sessionId: UUID`, `userId: UUID`, `tokenHash: String`, `issuedAt: DateTime`, `expiresAt: DateTime`, `isRevoked: Boolean`, `revokedAt: DateTime`, `ipAddress: String`, `userAgent: String` |
| **Methods** | `issue()`, `validate()`, `revoke()`, `isExpired()` |
| **Relationships** | Belongs to one User. Validated by the AuthMiddleware on every protected API request. |

**Business Rules:**
- A JWT token is valid for 8 hours from `issuedAt`.
- Logging out immediately revokes the session (token added to Redis denylist).
- All sessions for a User are revoked when the account is suspended.
- A User can have multiple concurrent sessions (e.g., logged in on phone and laptop simultaneously).

---

## 3. Entity Relationship Summary

| Relationship | Type | Multiplicity | Description |
|---|---|---|---|
| User → Report | Association | 1 to 0..* | A user submits zero or many reports |
| Report → Photo | Composition | 1 to 0..3 | A report contains up to 3 photos; photos cannot exist without a report |
| Report → MatchRecord | Association | 1 to 0..* | A report can appear in multiple match suggestions |
| MatchRecord → Handover | Association | 1 to 0..1 | A confirmed match leads to at most one handover |
| Handover → Notification | Association | 1 to 1..* | A handover triggers one or more notifications |
| MatchRecord → Notification | Association | 1 to 1..* | A match triggers at least one notification |
| User → Notification | Association | 1 to 0..* | A user receives zero or many notifications |
| User → Session | Composition | 1 to 0..* | A user can have multiple active sessions |

---

## 4. Business Rules Summary

| Rule ID | Business Rule | Source |
|---|---|---|
| BR-01 | Email must match university domain on registration | FR-01 |
| BR-02 | Report description must be at least 20 characters | FR-03 |
| BR-03 | Maximum 3 photos per report, each under 5 MB | FR-03 |
| BR-04 | Only matches with confidence >= 70% are persisted | FR-05 |
| BR-05 | Confidence = text score x 0.6 + image score x 0.4 | FR-05 |
| BR-06 | Reports cannot be edited once status is MATCHED | FR-07 |
| BR-07 | Reminders sent at day 3 and day 7 of uncollected handover | FR-08 |
| BR-08 | Personal data deleted 120 days after resolution/deactivation | FR-11 |
| BR-09 | STUDENT role cannot access ADMIN endpoints (HTTP 403) | FR-12 |
| BR-10 | JWT sessions expire after 8 hours | NFR-09 |