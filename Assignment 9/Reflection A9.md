# REFLECTION_A9.md — Challenges in Domain Modeling and Class Diagram Design
## CampusFind: Smart Campus Lost & Found System — Assignment 9

---

## 1. Challenges in Designing the Domain Model and Class Diagram

### Abstraction: What Deserves to Be a Class?

The first and most persistent challenge was deciding what deserves to be its own class versus what should simply be an attribute on an existing class. The Photo entity was the clearest example of this tension. My initial domain model had photos represented as a `photoUrls: String[]` array attribute on the Report class — simple, and technically workable. But as I worked through the business rules (maximum 3 photos, each under 5 MB, deleted from Cloudinary when the report is archived, used independently by the AI matching service for image analysis), it became clear that photos had enough independent behaviour — `upload()`, `validate()`, `fetchForAnalysis()`, `delete()` — to justify being their own class with their own lifecycle.

The rule I settled on was: if a data item has its own attributes, its own methods, and its own lifecycle that differs from its parent, it is a class. If it is simply a value that describes something else, it is an attribute. This distinction is easy to state and genuinely difficult to apply consistently, especially for a developer working alone without a team to challenge assumptions.

The Session class presented a similar question. A JWT token is often treated as a stateless string — you validate it on each request and that is the end of it. But the requirement to support explicit revocation (FR-02: logout must invalidate the session) and admin-triggered revocation (FR-12: suspending an account revokes all tokens) means the session needs to be persisted and queried, which means it needs to be a class. Recognising when a stateless concept needs to become a stateful entity was one of the key design insights from this assignment.

### Relationships: Association vs. Composition vs. Aggregation

The second major challenge was choosing the correct relationship type between classes. The distinction between association, aggregation, and composition is straightforward in textbook examples but genuinely ambiguous in real systems.

The Report-to-Photo relationship was clearly composition: a Photo cannot exist without its parent Report, and when the Report is deleted, all its Photos must be deleted. The ownership is total and the lifecycle is shared. This is the definition of composition.

The MatchRecord-to-Report relationship was trickier. A MatchRecord references two Reports (lost and found), but it does not own them — the Reports exist independently and can participate in multiple MatchRecords simultaneously (if the AI engine generates multiple match candidates). This is association, not aggregation or composition. Getting this right required thinking carefully about ownership and lifecycle, not just about whether one class "contains" another.

### Method Definitions: Behaviour vs. Data

Defining methods for each class required distinguishing between what a class does for itself (internal behaviour) and what is done to it by external services (business workflows). For example, `Report.validate()` is internal behaviour — the Report knows whether its own fields are valid. But `ReportService.archiveExpiredReports()` is a business workflow that operates on many Reports simultaneously and coordinates with the database, the queue, and the notification service. Mixing these two concerns into a single class would have produced a bloated, hard-to-test design.

The solution was the service layer — separating domain entity classes (which hold data and self-validation) from service classes (which hold business logic and coordination). This is a standard pattern in layered architecture, but understanding why it exists — not just that it exists — was clarified by having to draw the boundary explicitly in the class diagram.

---

## 2. Alignment with Previous Assignments

The class diagram is the structural complement to the behavioural models from Assignment 8. Every `status` enumeration in the class diagram corresponds directly to a state in the Assignment 8 state diagrams. The `ReportStatus` enumeration's values (OPEN, MATCHING, MATCH_FOUND, MATCHED, HANDOVER_PENDING, RESOLVED, ESCALATED, UNCLAIMED, ARCHIVED) are exactly the states shown in the Lost Item Report state transition diagram. This alignment was not accidental — the state diagrams were used as the authoritative source for defining the valid status values in the class diagram.

The service classes in the class diagram map directly to the swimlanes in the Assignment 8 activity diagrams. The `AuthService` corresponds to the "System / API Gateway" swimlane in Workflow 1 (User Registration). The `MatchingService` corresponds to the "AI Matching Service" swimlane in Workflow 3. The `HandoverService` corresponds to the system actions in Workflow 5. The class diagram gives these swimlanes a concrete implementation structure.

The functional requirements from Assignment 4 are traceable to specific classes and methods. FR-05 (AI matching engine) is implemented by `MatchingService.runMatching()` and `MatchRecord.calculateConfidence()`. FR-08 (digital handover) is implemented by the entire Handover class and `HandoverService`. FR-11 (POPIA data deletion) is implemented by `Report.archive()`, `Photo.delete()`, and `ReportService.archiveExpiredReports()`. This traceability means that every requirement has a clear implementation path, which is what good design should guarantee.

---

## 3. Trade-offs Made

### Simplifying Inheritance in Favour of Role-Based Enumeration

An object-oriented purist would model User as an abstract base class with concrete subclasses: `Student`, `AdminUser`, and `SuperAdmin`. This would allow each subclass to have its own methods (e.g., `Student.submitReport()`, `AdminUser.confirmMatch()`). However, this inheritance hierarchy would require significant complexity: polymorphic queries, type casting, and three separate database tables (or one table with many nullable columns). For a system where the behavioural differences between roles are enforced by a single `role` attribute checked in an `AuthMiddleware`, the simpler enumeration-based approach achieves the same security outcome with far less structural complexity. This was a deliberate trade-off of OO purity for practical simplicity.

### Denormalisation in MatchRecord

As noted in the design decisions, MatchRecord stores explicit references to both lostReportId and foundReportId rather than navigating through a join table. This violates strict database normalisation but significantly improves query performance for the most frequent operation in the system (fetching match details for admin review). Performance requirements (NFR-08: matching under 60 seconds) justified this trade-off.

---

## 4. Lessons Learned About Object-Oriented Design

The most important lesson from this assignment is that object-oriented design is not about finding the "correct" model — it is about making explicit trade-offs between competing concerns (normalisation vs. performance, inheritance vs. composition, simplicity vs. extensibility) and documenting the reasoning behind those trade-offs so that future developers can understand and evolve the design intelligently.

The second lesson is that a class diagram is most valuable not as a final deliverable but as a conversation tool — a shared vocabulary that connects requirements, architecture, behaviour, and implementation into a single coherent picture. Every class in the CampusFind diagram can be traced backward to a requirement and forward to a database table, an API endpoint, and a test case. That traceability is what makes the diagram genuinely useful rather than merely decorative.