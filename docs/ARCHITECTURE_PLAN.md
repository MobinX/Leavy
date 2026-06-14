# Leavy LMS — Complete Architecture & Implementation Plan

> **Status:** Planning Phase  
> **Version:** 0.1.0  
> **Last Updated:** June 14, 2026

---

## Table of Contents

1. [Business Requirements Analysis](#1-business-requirements-analysis)
2. [Missing Requirements & Gaps](#2-missing-requirements--gaps)
3. [Risks & Edge Cases](#3-risks--edge-cases)
4. [Additional Production Features](#4-additional-production-features)
5. [User Journeys](#5-user-journeys)
6. [Permission & Role Architecture](#6-permission--role-architecture)
7. [Institution Membership Architecture](#7-institution-membership-architecture)
8. [Course Lifecycle Architecture](#8-course-lifecycle-architecture)
9. [Exam Lifecycle Architecture](#9-exam-lifecycle-architecture)
10. [Homework Lifecycle Architecture](#10-homework-lifecycle-architecture)
11. [Notification Architecture](#11-notification-architecture)
12. [Reporting Architecture](#12-reporting-architecture)
13. [Analytics Architecture](#13-analytics-architecture)
14. [Deployment Architecture](#14-deployment-architecture)
15. [Scalability Strategy](#15-scalability-strategy)
16. [Security Strategy](#16-security-strategy)
17. [MVP Scope](#17-mvp-scope)
18. [Post-MVP Roadmap](#18-post-mvp-roadmap)
19. [Architectural Tradeoffs](#19-architectural-tradeoffs)
20. [Implementation Considerations](#20-implementation-considerations)

---

## 1. Business Requirements Analysis

### 1.1 Core Entities & Relationships

```
Institution ──┬── Course ──┬── Class (YouTube video embeds)
              │            ├── Note (PDF, Docs, Images, files)
              │            ├── Homework ──┬── Submission
              │            │              └── Grade + Feedback
              │            └── Exam ──┬── Question (MCQ / Short / Long)
              │                       └── StudentResponse ── Grade
              │
              ├── Teacher (via InstitutionMembership)
              └── Student (via InstitutionMembership)
```

### 1.2 Constraint Summary

| Domain | Constraint |
|--------|-----------|
| Video | YouTube embeds only. No internal hosting, processing, or storage. |
| Real-time | None. Standard request-response. No WebSockets, no live chat, no streaming. |
| Identity | Global users. One account across all institutions. |
| Tenant model | Multi-tenant with strict data isolation per institution. |
| Scale | Potentially thousands of institutions, hundreds of thousands of users, millions of submissions. |
| Stack | Next.js (frontend), TypeScript (backend), PostgreSQL (database). |

### 1.3 Domain Language

- **Institution**: The tenant/organization container (school, university, coaching center).
- **Course**: The central entity; owned by an institution; contains all learning content.
- **Class**: A learning session within a course, backed by an embedded YouTube video.
- **Note**: Supplementary study material (files).
- **Homework**: Teacher-created assignment with deadline, submissions, and grading.
- **Exam**: Teacher-created assessment with multiple question types.

---

## 2. Missing Requirements & Gaps

The following critical requirements are absent from the brief and must be addressed:

### 2.1 Identity & Authentication

| Gap | Severity | Recommendation |
|-----|----------|---------------|
| **Authentication mechanism** | Critical | Implement magic-link email auth + optional OAuth (Google, Microsoft). Educational institutions use Google Workspace and Microsoft 365 extensively. |
| **SSO / SAML support** | High (post-MVP) | Many schools/universities require SAML-based SSO. Architecture must accommodate this without rework. |
| **Account merging** | High | A user may sign up with personal email then get invited via school email. Must support account merging/unification. |
| **Password-based auth** | Medium | Magic-link is simpler and more secure; password auth can be added later. |

### 2.2 Authorization & Roles

| Gap | Severity | Recommendation |
|-----|----------|---------------|
| **Institution-level RBAC** | Critical | Beyond Teacher/Student: Institution Admin, Course Coordinator, Teaching Assistant, Observer. |
| **Cross-institution role independence** | Critical | A user can be Teacher at Institution A and Student at Institution B simultaneously. |
| **Super admin / platform admin** | High | A platform-level admin role for support operations, not tied to any institution. |

### 2.3 Data & Storage

| Gap | Severity | Recommendation |
|-----|----------|---------------|
| **File storage strategy** | Critical | PDFs, images, homework submissions need storage. Use S3-compatible object storage with presigned URLs. |
| **CDN for assets** | High | Profile images, course thumbnails, uploaded notes need CDN delivery. |
| **Video metadata tracking** | Medium | YouTube links need metadata (title, duration, thumbnail) cached server-side. |
| **File size limits & quotas** | High | Per-institution storage quotas for SaaS tiering. |

### 2.4 Business / SaaS

| Gap | Severity | Recommendation |
|-----|----------|---------------|
| **Billing & subscriptions** | Critical | Stripe integration for institution subscriptions, usage-based pricing tiers. |
| **Institution provisioning** | Critical | Self-serve signup flow or managed onboarding. |
| **Trial period / freemium** | Medium | Free tier for small coaching centers to drive adoption. |
| **Rate limiting per tenant** | High | Prevent one institution from degrading platform performance. |

### 2.5 Pedagogy & Assessment

| Gap | Severity | Recommendation |
|-----|----------|---------------|
| **Question banks / pools** | High | Teachers need reusable question banks across exams. |
| **Exam time limits** | High | Timed exams with auto-submit on expiry. |
| **Randomized question ordering** | Medium | Shuffle questions per student to reduce cheating. |
| **Gradebook / weighted grading** | High | Weighted categories, curving, grade export (CSV/Excel). |
| **Plagiarism detection** | Low (post-MVP) | Integration with external plagiarism checkers. |

### 2.6 Compliance & Legal

| Gap | Severity | Recommendation |
|-----|----------|---------------|
| **GDPR compliance** | Critical | Data export, account deletion, consent management, data residency options. |
| **FERPA (US education privacy)** | Critical | Student educational records privacy protections. |
| **COPPA (children's privacy)** | High | If platform serves users under 13, COPPA compliance is mandatory. |
| **Audit logging** | High | Immutable logs for grade changes, permission changes, data access. |
| **Data retention policies** | Medium | Configurable retention for submissions and student data. |

### 2.7 Notifications

| Gap | Severity | Recommendation |
|-----|----------|---------------|
| **Email notifications** | Critical | Assignment deadlines, grading published, course enrollment, institution invites. |
| **In-app notification center** | High | Persistent notification bell with read/unread state. |
| **Notification preferences** | Medium | Per-user opt-in/opt-out for notification types. |

### 2.8 Course & Content Management

| Gap | Severity | Recommendation |
|-----|----------|---------------|
| **Course publishing workflow** | High | Draft → Published → Archived states. |
| **Course duplication** | Medium | Clone a course with its structure for new semesters. |
| **Content versioning** | Low | Track changes to notes and exam questions. |
| **Course prerequisites** | Medium | Enforce prerequisite completion before enrollment. |

---

## 3. Risks & Edge Cases

### 3.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Thundering herd on exam deadlines** | High | High | Queue-based submission ingestion. API returns 202 Accepted immediately; background workers persist. Rate-limit per exam. |
| **YouTube dependency** | Medium | Medium | YouTube embeds may show ads, be blocked by school firewalls, or videos may be deleted. Cache metadata; surface "video unavailable" gracefully. |
| **Cross-tenant data leakage (IDOR)** | Critical | Low | Row-Level Security (RLS) at database level + middleware-level tenant scoping as defense-in-depth. |
| **Database contention on large reports** | Medium | Medium | Read replicas for reporting queries; materialized views for dashboards. |
| **File upload reliability** | Medium | Medium | Chunked uploads for large files; retry logic; presigned URLs with expiration. |
| **N+1 query problems** | Medium | High | Strict use of DataLoader patterns; eager loading where appropriate; query monitoring. |

### 3.2 Business Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **YouTube policy changes** | Medium | Low | Abstract video provider behind an interface for future provider swaps (Vimeo, Wistia). |
| **Institution data export demands** | Medium | Medium | Build comprehensive data export from day one (course data, grades, submissions). |
| **Competing with established LMS (Canvas, Moodle)** | High | N/A | Focus on SMB/coaching center niche; emphasize ease of use and multi-tenant simplicity. |

### 3.3 Edge Cases

#### Timezone & Deadlines
- **Problem**: A teacher in IST sets homework deadline "11:59 PM". A student in PST sees a different deadline.
- **Resolution**: Store all timestamps in UTC. Store each institution's timezone. Display deadlines in the student's local timezone (browser-detected) with explicit timezone label. Grace period of 60 seconds after deadline before marking as late.

#### Concurrent Submissions
- **Problem**: Student double-clicks "Submit" on slow connection. Two identical submissions created.
- **Resolution**: Idempotency keys on submission endpoints. Debounce UI button after first click. Unique constraint on (student_id, homework_id, attempt_number).

#### Global Identity Conflicts
- **Problem**: User signs up as "teacher" with personal email; institution admin invites same person's work email. Two accounts exist.
- **Resolution**: Account linking/merging flow. Email verification ensures ownership before merging.

#### Institution Lifecycle
- **Problem**: Institution stops paying. What happens to data? What about users who also belong to other institutions?
- **Resolution**: Soft-delete institution (flag as suspended). Users retain their global accounts and access to other active institutions. Data retained for 90-day grace period, then export offered, then deleted.

#### Exam Timing
- **Problem**: Student starts a 60-minute exam at 11:55 PM on the deadline day. Exam should be 60 minutes, but deadline has passed.
- **Resolution**: "Start-by" time (must begin exam before this time) vs "Submit-by" time (must finish by this time). Clearly communicate both to students.

#### Grading Disputes
- **Problem**: Student disputes an auto-graded MCQ or manually-graded essay.
- **Resolution**: Grade appeal workflow. Student flags → Teacher reviews → Grade stands or is adjusted. All grade changes are audit-logged.

#### Deleted Content
- **Problem**: Teacher deletes a note that students have bookmarked. Teacher deletes an exam question that was already answered.
- **Resolution**: Soft-delete all content. Submissions reference immutable snapshots of questions at time of exam.

#### Large File Uploads
- **Problem**: Student uploads 500MB video as homework submission.
- **Resolution**: File size limits per homework config. Chunked upload via presigned URLs. Progress indicators. Quota enforcement.

---

## 4. Additional Production Features

Beyond the MVP brief, a production-grade LMS needs:

### 4.1 Assessment & Grading
- **Question banks**: Reusable question pools with tagging and difficulty levels.
- **Randomized exams**: Pull N questions from a bank; randomize order per student.
- **Partial credit**: Award partial points for partially correct answers.
- **Rubrics**: Structured grading criteria for essays and long answers.
- **Grade curving**: Apply curves (bell curve, linear shift) to exam results.
- **Grade appeals**: Formal dispute resolution workflow.
- **Bulk grading**: Grade multiple submissions efficiently from a single view.

### 4.2 Content Delivery
- **Course progress tracking**: Visual progress bar per course (X/Y classes watched, Z/W homework submitted).
- **Completion certificates**: Auto-generated PDF certificates on course completion.
- **Drip content**: Schedule content release dates.
- **Prerequisites**: Enforce course prerequisites.

### 4.3 Communication
- **Announcements**: Institution-wide and course-level announcements.
- **Discussion forums** (low priority): Async threaded discussions per course.
- **Feedback templates**: Saved feedback snippets for common grading comments.

### 4.4 Administration
- **Bulk user import**: CSV upload for adding students/teachers.
- **Custom branding**: Per-institution logo, colors, domain (CNAME).
- **API access**: REST API for institution integrations (LTI, SIS sync).
- **Webhooks**: Event-driven callbacks for external system integration.

### 4.5 Mobile
- **PWA**: Progressive Web App for mobile access (no native app needed for MVP).
- **Offline support**: Cache course notes for offline reading.

---

## 5. User Journeys

### 5.1 Institution Admin Onboarding
1. Admin signs up at `leavy.com` → creates institution profile (name, type, timezone).
2. Admin selects subscription plan → enters payment via Stripe.
3. Admin is directed to institution dashboard.
4. Admin invites teachers via email.
5. Admin creates course categories/departments (optional).
6. Admin views analytics dashboard (empty initially).

### 5.2 Teacher Workflow
1. Teacher receives email invitation → creates account (or logs into existing).
2. Teacher accepts institution membership → lands on institution dashboard.
3. Teacher creates a course: title, description, cover image, start/end dates.
4. Teacher adds content:
   - **Classes**: Uploads video to YouTube → pastes link → adds title/description.
   - **Notes**: Uploads PDFs, documents, images.
   - **Homework**: Creates assignment with description, files, deadline, max score.
   - **Exams**: Creates exam with MCQ, short answer, long answer questions.
5. Teacher publishes course (or keeps as draft).
6. Students enroll (self-enroll or admin adds them).
7. Teacher monitors progress, grades submissions, views analytics.

### 5.3 Student Workflow
1. Student receives invitation or self-registers → creates global account.
2. Student joins institution → sees enrolled courses on dashboard.
3. Student opens a course → sees structured content:
   - Classes with video embeds → watches, marks as complete.
   - Notes → views/downloads.
   - Homework → reads instructions → submits (text + files) → sees grade/feedback later.
   - Exams → starts timed exam → answers questions → submits → sees results (auto-graded MCQs immediately; manual grades later).
4. Student views performance dashboard: grades, progress across courses.

### 5.4 Cross-Institution User
1. Teacher A teaches at School X and Coaching Center Y.
2. Teacher A logs in → sees both institutions on dashboard.
3. Teacher A switches context to School X → sees School X courses.
4. Teacher A switches context to Coaching Center Y → sees Coaching Center Y courses.
5. No data leakage between institutions.

### 5.5 Grading Workflow
1. Teacher opens homework/exam → sees list of submissions.
2. Teacher clicks a submission → sees student's answers.
3. For auto-graded (MCQ): score already computed; teacher can override.
4. For manual grading: teacher reads answer, assigns score, writes feedback.
5. Teacher publishes grades → student receives notification.
6. Student views grade + feedback on their dashboard.

---

## 6. Permission & Role Architecture

### 6.1 Role Hierarchy (per Institution)

```
Platform Super Admin (global, outside institutions)
  └── Institution Owner (one per institution)
        └── Institution Admin (multiple per institution)
              ├── Course Coordinator (manages specific courses)
              ├── Teacher (teaches assigned courses)
              │     └── Teaching Assistant (assists specific courses)
              ├── Student (enrolls in courses)
              └── Observer (read-only, e.g., parent, auditor)
```

### 6.2 Role Definitions

| Role | Scope | Key Permissions |
|------|-------|----------------|
| **Platform Super Admin** | Global | Manage all institutions, view platform analytics, support operations. |
| **Institution Owner** | Single institution | Manage billing, delete institution, transfer ownership, manage admins. |
| **Institution Admin** | Single institution | Manage users, create courses, view all reports, configure settings. |
| **Course Coordinator** | Specific courses | Manage course content, enroll students, view course analytics. |
| **Teacher** | Assigned courses | Create/edit content, grade submissions, view student progress. |
| **Teaching Assistant** | Assigned courses | Grade submissions, answer questions, cannot create/edit content. |
| **Student** | Enrolled courses | View content, submit homework, take exams, view own grades. |
| **Observer** | Specific courses/students | Read-only access to content and grades (e.g., parents, auditors). |

### 6.3 Permission Model

Use **scoped RBAC** (Role-Based Access Control with institution/course scoping):

```
Permission check: can(user, action, resource)

Where:
- user.roles[institutionId] → [role1, role2, ...]
- action → "course.create", "submission.grade", "report.view", etc.
- resource.tenantId → must match current institution context
- resource.courseId → must be in user's assigned courses
```

**Key principles:**
- Permissions are always scoped to an institution context.
- A user's roles are per-institution (independence principle).
- Course-level permissions further restrict teachers/TAs to their assigned courses.
- Database-level Row-Level Security (RLS) as safety net.

### 6.4 Permission Matrix (Abbreviated)

| Action | Super Admin | Inst. Owner | Inst. Admin | Teacher | TA | Student | Observer |
|--------|:-----------:|:-----------:|:-----------:|:-------:|:--:|:-------:|:--------:|
| Create institution | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Manage billing | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Invite/remove users | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Create course | ❌ | ✅ | ✅ | ✅¹ | ❌ | ❌ | ❌ |
| Edit course content | ❌ | ✅ | ✅ | ✅² | ❌ | ❌ | ❌ |
| Grade submissions | ❌ | ✅ | ✅ | ✅² | ✅² | ❌ | ❌ |
| View all reports | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| View own course reports | ❌ | ✅ | ✅ | ✅² | ✅² | ❌ | ✅³ |
| Submit homework | ❌ | ❌ | ❌ | ❌ | ❌ | ✅⁴ | ❌ |
| Take exam | ❌ | ❌ | ❌ | ❌ | ❌ | ✅⁴ | ❌ |
| View own grades | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅³ |

¹ Teachers can create courses but require admin approval to publish (configurable).  
² Only in assigned courses.  
³ Observer sees grades for their assigned students only.  
⁴ Only in enrolled courses.

---

## 7. Institution Membership Architecture

### 7.1 Data Model

```
┌─────────────┐       ┌──────────────────────────┐       ┌──────────────┐
│    User     │       │  InstitutionMembership   │       │ Institution  │
│  (global)   │───────│                          │───────│  (tenant)    │
├─────────────┤  1:N  ├──────────────────────────┤  N:1  ├──────────────┤
│ id          │       │ id                       │       │ id           │
│ email       │       │ userId                   │       │ name         │
│ name        │       │ institutionId            │       │ slug         │
│ avatarUrl   │       │ role                     │       │ timezone     │
│ createdAt   │       │ joinedAt                 │       │ settings     │
│ ...         │       │ isActive                 │       │ status       │
└─────────────┘       │ invitedBy (userId)       │       │ ...          │
                      └──────────────────────────┘       └──────────────┘
                                 │
                                 │ N:1
                                 ▼
                      ┌──────────────────────┐
                      │  CourseEnrollment    │
                      ├──────────────────────┤
                      │ id                   │
                      │ courseId             │
                      │ membershipId         │
                      │ enrolledAt           │
                      │ completedAt          │
                      │ progress             │
                      └──────────────────────┘
```

### 7.2 Membership Lifecycle

```
Invitation Sent ──► Pending ──► Accepted (Active)
                       │
                       ▼
                   Declined / Expired

Active ──► Suspended ──► Active (reinstated)
Active ──► Removed (revoked)
```

### 7.3 Design Decisions

1. **Invitation-based joining**: Institution admins invite users by email. If email exists → user gets notification. If email doesn't exist → invitation email with signup link.
2. **Self-enrollment for students**: Optional per-course setting allowing students to join via link or code.
3. **Role assignment at membership level**: A user can have different roles in different institutions.
4. **Active/inactive toggle**: Admins can suspend a user without deleting membership (preserves data).
5. **Membership carries enrollment**: When a user is removed from an institution, their course enrollments are soft-deleted. Data is preserved for audit.

### 7.4 Context Switching

The application must maintain an "active institution context" for the user:

- **API**: All requests include `X-Institution-Id` header (set after user selects institution).
- **Middleware**: Validates user has active membership in that institution.
- **Database**: All queries include `WHERE institution_id = $currentInstitutionId`.
- **UI**: Institution switcher in top navigation bar.
- **Default**: Last-used institution (stored in user preferences).

---

## 8. Course Lifecycle Architecture

### 8.1 Course States

```
         ┌──────────┐
         │  Draft   │  Teacher creates, edits privately
         └────┬─────┘
              │ Publish
              ▼
         ┌──────────┐
         │Published │  Visible to enrolled students
         └────┬─────┘
              │ End Date passes / Manual
              ▼
         ┌──────────┐
         │Completed │  Read-only for students, teachers can still grade
         └────┬─────┘
              │ Archive
              ▼
         ┌──────────┐
         │ Archived │  Hidden from all; data preserved for records
         └──────────┘

Transitions also available:
  Draft → Archived (delete without publishing)
  Published → Archived (emergency takedown)
  Archived → Draft (restore)
```

### 8.2 Course Structure

```
Course
├── Metadata (title, description, cover image, tags, category)
├── Settings (enrollment type, prerequisites, completion criteria)
├── Modules (ordered sections)
│   ├── Module 1: "Introduction"
│   │   ├── Class 1 (YouTube embed)
│   │   ├── Note 1 (PDF)
│   │   └── Quiz (mini-exam)
│   ├── Module 2: "Advanced Topics"
│   │   ├── Class 2 (YouTube embed)
│   │   ├── Homework 1
│   │   └── Note 2
│   └── Module 3: "Assessment"
│       ├── Exam 1 (Midterm)
│       └── Exam 2 (Final)
└── Enrollment (list of students)
```

### 8.3 Modules

Modules provide logical grouping within a course. They are ordered and can be:
- **Sequential**: Students must complete Module 1 before accessing Module 2 (drip).
- **Free-form**: All modules available immediately.

### 8.4 Course Completion

A course is marked "completed" for a student when:
- All required content is viewed/submitted (configurable rules).
- Minimum grade threshold met (configurable).
- Teacher manually marks complete.

---

## 9. Exam Lifecycle Architecture

### 9.1 Exam States

```
Draft ──► Published ──► In Progress ──► Grading ──► Completed
                    │                                 │
                    └───── Closed (early) ◄────────────┘
```

### 9.2 Exam Configuration

```typescript
interface ExamConfig {
  title: string;
  description: string;
  durationMinutes: number;        // Time limit. 0 = untimed
  startAt: Date;                  // When exam becomes available
  endAt: Date;                    // Deadline (submit-by time)
  startByAt?: Date;               // Latest time student can begin (defaults to endAt)
  shuffleQuestions: boolean;      // Randomize order per student
  shuffleOptions: boolean;        // Randomize MCQ option order
  showResults: 'immediately' | 'after_grading' | 'after_deadline' | 'never';
  allowRetakes: boolean;
  maxRetakes: number;
  passingScore: number;           // Percentage to pass
  questions: ExamQuestion[];
}
```

### 9.3 Question Types

| Type | Auto-graded | Implementation |
|------|:-----------:|---------------|
| **Multiple Choice (Single)** | Yes | One correct option from list |
| **Multiple Choice (Multi)** | Yes | Multiple correct options; partial credit possible |
| **True/False** | Yes | Binary choice |
| **Short Answer** | No* | Text input; teacher grades manually |
| **Long Answer / Essay** | No | Rich text; rubric-based grading |
| **Fill in the Blank** | Yes | Exact or fuzzy string match |
| **File Upload** | No | Student uploads file (e.g., code, design) |

*Short answer could support keyword-based auto-grading as enhancement.

### 9.4 Exam Taking Flow

```
Student clicks "Start Exam"
  → System checks: within time window? within retake limit?
  → Creates ExamAttempt record with startTime
  → Renders questions (randomized if configured)
  → Timer counts down (client-side, server-enforced)
  → Student answers questions (auto-save drafts every 30s)
  → Student clicks "Submit" or timer expires (auto-submit)
  → System queues submission for processing
  → Auto-graded questions scored immediately
  → Manual questions queued for teacher grading
  → Student sees "Submitted" confirmation + auto-graded results (if configured)
```

### 9.5 Anti-Cheating Measures (MVP-appropriate)

1. **Timer enforcement**: Server validates submission time against exam start time + duration. Late submissions rejected.
2. **Question randomization**: Shuffle question order and MCQ options per student.
3. **No backward navigation** (configurable): Prevent returning to previous questions.
4. **Single active attempt**: Prevent multiple concurrent exam sessions.
5. **IP logging**: Log IP address of exam start/submit (for audit, not blocking).

Post-MVP enhancements: Browser lockdown, proctoring integrations, plagiarism detection.

### 9.6 Submission Processing

```
POST /api/exams/:id/submit
  → Validate: student enrolled? within time? within retake limit?
  → Generate idempotency key (prevents duplicate submissions)
  → Create submission payload
  → Push to Redis queue
  → Return 202 Accepted with attemptId

Background Worker:
  → Consumes queue
  → Auto-grades MCQ/TrueFalse/FillInBlank
  → Stores responses in DB
  → Updates attempt status
  → Triggers notification to teacher (for manual grading)
```

---

## 10. Homework Lifecycle Architecture

### 10.1 Homework States

```
Draft ──► Published ──► Closed ──► Grading Complete
```

### 10.2 Homework Configuration

```typescript
interface HomeworkConfig {
  title: string;
  description: string;
  instructions: string;           // Rich text
  attachments: FileAttachment[];  // Teacher-provided files
  dueDate: Date;
  allowLateSubmissions: boolean;
  latePenaltyPercent?: number;   // e.g., 10% per day late
  maxScore: number;
  submissionType: 'text' | 'file' | 'both';
  allowedFileTypes?: string[];   // e.g., ['.pdf', '.docx', '.zip']
  maxFileSizeMB?: number;
  gradingType: 'points' | 'rubric' | 'pass_fail';
  rubric?: RubricCriteria[];
  allowResubmission: boolean;
  maxResubmissions: number;
}
```

### 10.3 Submission Flow

```
Student opens homework
  → Reads instructions, downloads attachments
  → Writes response / uploads files
  → Saves draft (optional)
  → Submits
  → System records submission with timestamp
  → Student sees confirmation
  → If late: system calculates penalty

Teacher reviews:
  → Opens submission list
  → Reads student response / downloads files
  → Assigns score, writes feedback
  → Publishes grade (or saves draft)
  → Student receives notification
  → Student views grade + feedback
```

### 10.4 Feedback & Grading

- **Inline annotations** (future): Annotate student text responses.
- **Rubric-based**: Click rubric criteria levels; score auto-calculates.
- **Feedback templates**: Saved snippets for common comments.
- **Private comments**: Visible only to student.
- **Grade publishing**: Grades can be drafted and published in bulk.

---

## 11. Notification Architecture

### 11.1 Notification Channels

| Channel | Use Case | Priority |
|---------|----------|----------|
| **Email** | Invitations, deadline reminders, grade published, announcements | Critical |
| **In-app** | Persistent notification center; read/unread state | High |
| **SMS** (post-MVP) | Parent notifications, urgent alerts | Low |
| **Push** (post-MVP) | PWA push notifications for mobile | Low |

### 11.2 Notification Events

| Event | Recipient | Channel |
|-------|-----------|---------|
| Institution invitation | User (email) | Email |
| Enrollment confirmed | Student | Email + In-app |
| Homework deadline (24h before) | Student | Email + In-app |
| Homework deadline (1h before) | Student | In-app |
| Homework submitted | Teacher (assigned) | In-app |
| Homework graded | Student | Email + In-app |
| Exam scheduled | Student | Email + In-app |
| Exam graded | Student | Email + In-app |
| Course published | Enrolled students | In-app |
| Announcement posted | Course students | Email + In-app |
| Grade appeal filed | Teacher | In-app |
| Grade appeal resolved | Student | Email + In-app |

### 11.3 Architecture

```
Application Event
       │
       ▼
┌──────────────┐
│ Event Emitter │  (in-process for MVP; message queue for scale)
└──────┬───────┘
       │
       ├──► Notification Service
       │      ├── Create in-app notification (DB)
       │      ├── Check user preferences
       │      └── Dispatch email (via email provider: Resend, SendGrid, SES)
       │
       └──► (Future: SMS, Push services)
```

### 11.4 User Preferences

```typescript
interface NotificationPreferences {
  email: {
    gradingNotifications: boolean;
    deadlineReminders: boolean;
    announcements: boolean;
    invitations: boolean;  // Always on
  };
  inApp: {
    gradingNotifications: boolean;
    deadlineReminders: boolean;
    announcements: boolean;
  };
  digestFrequency: 'instant' | 'daily' | 'weekly';
}
```

---

## 12. Reporting Architecture

### 12.1 Report Types

#### Institution-Level Reports
| Report | Data | Audience |
|--------|------|----------|
| **Enrollment Summary** | Students per course, enrollment trends | Admins |
| **Teacher Activity** | Courses created, content added, grades given | Admins |
| **Revenue Report** | Subscription revenue (from Stripe) | Owner |
| **Completion Rates** | Course completion percentages | Admins |

#### Course-Level Reports
| Report | Data | Audience |
|--------|------|----------|
| **Student Progress** | Per-student progress: classes watched, homework submitted, grades | Teacher |
| **Grade Distribution** | Histogram of grades for homework/exam | Teacher |
| **Content Engagement** | Video watch counts, note downloads | Teacher |
| **Submission Timeline** | Submissions over time (catch late pattern) | Teacher |

#### Student-Level Reports
| Report | Data | Audience |
|--------|------|----------|
| **Performance Dashboard** | Grades across courses, GPA-like metrics | Student |
| **Progress Report** | Detailed per-course progress | Student, Observer |
| **Grade Transcript** | Historical grades across all courses | Student |

### 12.2 Architecture

```
┌──────────────────────────────────────────────┐
│              Reporting Service               │
│                                              │
│  Sync Reports (small, real-time):            │
│    → Direct DB query (scoped to tenant)     │
│    → Cached for 5 minutes (Redis)           │
│                                              │
│  Async Reports (large, institution-wide):    │
│    → Request queued                         │
│    → Background job generates CSV/PDF       │
│    → Result uploaded to S3 (presigned URL)  │
│    → User notified when ready               │
│                                              │
│  Scheduled Reports:                          │
│    → Cron jobs (weekly/monthly summaries)   │
│    → Auto-emailed to admins                 │
└──────────────────────────────────────────────┘
```

### 12.3 Export Formats
- **CSV**: Grades, enrollment lists, submission data.
- **PDF**: Formatted grade reports, transcripts, certificates.
- **Excel**: Multi-sheet workbooks with charts.

---

## 13. Analytics Architecture

### 13.1 Metrics

#### For Students
- **Course Progress**: Percentage of content completed.
- **Grade Trend**: Line chart of grades over time.
- **Strength/Weakness**: Topics with highest/lowest scores.
- **Time Spent**: Estimated time on platform (derived from activity timestamps).
- **Ranking** (optional, per-institution setting): Percentile within course.

#### For Teachers
- **Class Performance**: Average, median, distribution per homework/exam.
- **Question Analysis**: Which questions students struggle with most.
- **Engagement Metrics**: Video views, note downloads, submission timeliness.
- **At-Risk Students**: Students below grade threshold or with low engagement.
- **Comparative**: Current cohort vs previous cohorts.

#### For Institution Admins
- **Platform Adoption**: Active users over time.
- **Course Effectiveness**: Completion rates, average grades.
- **Teacher Performance**: Grading turnaround time, student satisfaction (future).
- **Retention**: Student re-enrollment rates.

### 13.2 Architecture

```
Operational DB (PostgreSQL)
       │
       │ ETL Pipeline (batch, hourly/daily)
       ▼
Analytics Store (PostgreSQL read replica or separate analytics DB)
       │
       ├── Materialized Views (pre-computed aggregates)
       │
       ├── Dashboard Queries (with caching)
       │
       └── Export Pipeline (CSV/PDF)
```

**MVP approach**: Use PostgreSQL materialized views refreshed periodically. No need for a separate analytics database at MVP scale.

**Post-MVP**: Consider dedicated analytics pipeline (ClickHouse, TimescaleDB) for high-cardinality event data.

---

## 14. Deployment Architecture

### 14.1 Infrastructure

```
                          ┌──────────────┐
                          │   Cloudflare  │  DNS, CDN, DDoS Protection
                          └──────┬───────┘
                                 │
                    ┌────────────┴────────────┐
                    │     Load Balancer       │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
     ┌────────────┐    ┌────────────┐    ┌────────────┐
     │ Next.js    │    │ Next.js    │    │ Next.js    │
     │ Server 1   │    │ Server 2   │    │ Server N   │
     └─────┬──────┘    └─────┬──────┘    └─────┬──────┘
           │                 │                 │
           └─────────────────┼─────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │ PostgreSQL │ │   Redis    │ │  S3/        │
     │ (Primary)  │ │  (Cache/   │ │  Object     │
     │            │ │   Queue)   │ │  Storage    │
     └─────┬──────┘ └────────────┘ └────────────┘
           │
     ┌─────┴──────┐
     │ PostgreSQL │
     │ (Read      │
     │  Replica)  │
     └────────────┘
```

### 14.2 Technology Choices

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Hosting** | Vercel (frontend) + Railway/Render/Fly.io (backend) | Vercel for optimal Next.js hosting. Railway for simple managed Postgres + Redis. |
| **Alternative** | AWS (ECS + RDS + ElastiCache + S3) | More control, better at scale, but higher operational complexity. |
| **Database** | PostgreSQL 16+ | Required. Use managed (RDS, Railway, Supabase). |
| **Cache** | Redis | Session store, rate limiting, job queue, caching. |
| **Storage** | AWS S3 or Cloudflare R2 | Object storage for files, with presigned URLs for direct upload. |
| **CDN** | Cloudflare | Global CDN, DDoS protection, SSL. |
| **Email** | Resend or SendGrid | Transactional email delivery. |
| **Monitoring** | Sentry (errors) + Grafana/Prometheus (metrics) or Datadog | Observability. |
| **Logging** | Structured JSON logs → centralized logging (Axiom, Logtail, or ELK) | Audit trail and debugging. |
| **CI/CD** | GitHub Actions | Automated testing, linting, deployment. |

### 14.3 Environments

| Environment | Purpose |
|-------------|---------|
| **Development** | Local development with Docker Compose (Postgres, Redis). |
| **Staging** | Production-like environment for QA and preview. |
| **Production** | Live environment with full monitoring and backups. |

### 14.4 Docker Compose (Development)

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: leavy
      POSTGRES_USER: leavy
      POSTGRES_PASSWORD: leavy_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  # Optional: local S3 emulator
  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    command: server /data --console-address ":9001"

volumes:
  pgdata:
```

---

## 15. Scalability Strategy

### 15.1 Scaling Phases

#### Phase 1: MVP (up to ~100 institutions, ~10K users)
- Single PostgreSQL instance (sufficient).
- Single Next.js server (or Vercel serverless).
- Redis for cache and lightweight queues.
- No read replicas.
- Simple materialized views for analytics.

#### Phase 2: Growth (~500 institutions, ~100K users)
- PostgreSQL read replicas for reporting queries.
- Dedicated background worker service for job processing.
- Redis cluster for higher throughput.
- Connection pooling (PgBouncer).
- CDN for static assets and uploaded files.

#### Phase 3: Scale (~5000+ institutions, ~500K+ users)
- Database sharding by tenant_id (Citus or manual partitioning).
- Separate analytics database (ClickHouse/TimescaleDB).
- Multiple app server instances behind load balancer.
- Queue-based submission processing decoupled from API.
- Rate limiting per tenant at API gateway level.
- Consider edge caching for read-heavy content.

### 15.2 Key Scaling Patterns

1. **Queue-first submission ingestion**: Exam and homework submissions go to queue first, not DB. This decouples API responsiveness from DB write capacity.
2. **CQRS-lite for reporting**: Separate read models (materialized views) from write models (operational tables).
3. **Tenant-based partitioning**: All major tables partitioned by `institution_id` for query isolation and easier data management.
4. **Connection pooling**: PgBouncer in front of PostgreSQL to handle high connection counts from serverless functions.
5. **Background job processing**: All non-request-critical work (grading, report generation, email sending, analytics ETL) runs in background workers.

### 15.3 Database Indexing Strategy

Critical indexes for performance:
```sql
-- Institution membership lookups (most common query)
CREATE INDEX idx_membership_user_institution ON institution_memberships(user_id, institution_id);

-- Course listing per institution
CREATE INDEX idx_course_institution ON courses(institution_id, status);

-- Submissions per homework (grading view)
CREATE INDEX idx_submission_homework ON homework_submissions(homework_id, submitted_at);

-- Submissions per student (dashboard)
CREATE INDEX idx_submission_student ON homework_submissions(student_id, homework_id);

-- Exam responses for grading
CREATE INDEX idx_exam_response_attempt ON exam_responses(exam_attempt_id);
```

---

## 16. Security Strategy

### 16.1 Defense in Depth

```
Layer 1: Network
  ├── Cloudflare DDoS protection
  ├── HTTPS only (HSTS)
  └── Firewall / security groups

Layer 2: Application
  ├── Input validation & sanitization
  ├── CORS policy (strict origin)
  ├── CSP headers
  ├── Rate limiting (global + per-tenant)
  ├── CSRF protection (Next.js built-in)
  └── Helmet.js security headers

Layer 3: Authentication
  ├── Magic-link email auth (no passwords to leak)
  ├── JWT with short expiry + refresh tokens
  ├── Session invalidation on role change
  └── 2FA (post-MVP for admins)

Layer 4: Authorization
  ├── Middleware: tenant context validation
  ├── Service layer: permission checks per action
  └── Database: Row-Level Security (RLS)

Layer 5: Data
  ├── Encryption at rest (PostgreSQL TDE, S3 encryption)
  ├── Encryption in transit (TLS everywhere)
  ├── PII field-level encryption (student records)
  └── Database backups (encrypted, off-site)
```

### 16.2 Row-Level Security (RLS)

PostgreSQL RLS is the **safety net** preventing cross-tenant data leakage:

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE courses ENABLE ROW LEVEL SECURITY;
ALTER TABLE homework_submissions ENABLE ROW LEVEL SECURITY;
ALTER TABLE exam_responses ENABLE ROW LEVEL SECURITY;
-- ... all tenant-scoped tables

-- Policy: User can only see data from their current institution
CREATE POLICY tenant_isolation ON courses
  FOR ALL
  TO authenticated_user
  USING (institution_id = current_setting('app.current_institution_id')::uuid);

-- The app sets this at the start of each transaction:
-- SET app.current_institution_id = '...';
-- SET app.current_user_id = '...';
```

### 16.3 API Security

- **All endpoints require authentication** (except auth endpoints).
- **Tenant context header**: `X-Institution-Id` validated against user's memberships.
- **Idempotency keys**: Required for mutation endpoints (POST/PUT/DELETE).
- **Input validation**: Zod schemas for all request bodies.
- **Output filtering**: Never return `tenant_id` or internal IDs in responses. Return only data scoped to current tenant.

### 16.4 Data Privacy & Compliance

| Regulation | Requirements | Implementation |
|------------|-------------|----------------|
| **GDPR** | Right to access, rectify, delete data. Data portability. | User settings: export all data; delete account (anonymize or hard-delete). |
| **FERPA** | Protect student education records. Limit access to authorized personnel. | Strict RBAC; audit logs for all grade/record access. |
| **COPPA** | Parental consent for users under 13. | Age gate during registration; parental consent workflow (future). |

### 16.5 Audit Logging

```typescript
interface AuditLog {
  id: string;
  timestamp: Date;
  actorId: string;       // User who performed action
  institutionId: string; // Tenant context
  action: string;        // e.g., "grade.change", "user.invite", "course.delete"
  resource: string;      // e.g., "homework_submission:abc123"
  details: object;       // Before/after values, reason
  ip: string;
  userAgent: string;
}
```

- Audit logs are append-only (immutable).
- Stored in a separate audit database/table.
- Not accessible via application API (only platform super admins).
- Retained for minimum 7 years (compliance).

---

## 17. MVP Scope

### 17.1 MVP Must-Have

#### Authentication & Identity
- [x] Magic-link email authentication
- [x] Global user accounts
- [x] Basic profile (name, email, avatar)

#### Institution Management
- [x] Institution creation (manual, no self-serve)
- [x] Institution settings (name, timezone)
- [x] Invite teachers by email
- [x] Invite/enroll students by email

#### Course Management
- [x] Create/edit/delete courses (Draft/Published/Archived)
- [x] Course modules (ordered sections)
- [x] YouTube video classes within modules
- [x] File notes (PDF, images, documents)
- [x] Student enrollment in courses

#### Homework
- [x] Create homework with description, files, deadline
- [x] Student submission (text + file upload)
- [x] Teacher grading (points + feedback)
- [x] Grade visibility for students

#### Exams
- [x] Create exams with MCQ + short answer + long answer
- [x] Timed exams with auto-submit
- [x] Auto-grading for MCQs
- [x] Manual grading for short/long answer
- [x] Grade visibility for students

#### Student Dashboard
- [x] Enrolled courses list
- [x] Course progress (basic: content completed/total)
- [x] Grades per course (homework + exam scores)

#### Teacher Dashboard
- [x] Assigned courses
- [x] Submission queue (ungraded homework/exams)
- [x] Basic student roster

#### Admin Dashboard
- [x] Institution overview (users, courses)
- [x] User management (invite, remove, change role)

#### Infrastructure
- [x] PostgreSQL with RLS
- [x] Redis for sessions and caching
- [x] File storage (S3/R2)
- [x] Email delivery

### 17.2 MVP Explicitly Excluded (but architected for)
- Self-serve institution signup (manual onboarding)
- Billing/subscriptions (free during beta)
- SAML/SSO
- Advanced gradebook (weighting, curving)
- Question banks
- Course prerequisites
- Drip content
- Announcements
- In-app notification center
- Mobile PWA
- Parent accounts
- Plagiarism detection
- LTI integration
- Custom branding/white-label

### 17.3 MVP Success Criteria
- A teacher can create a course with classes, notes, homework, and exams.
- Students can enroll, view content, submit work, and see grades.
- Institution admin can manage users and view basic reports.
- Multi-tenancy: Data from Institution A is completely invisible to Institution B.
- Cross-institution users: A user can be Teacher at School X and Student at Coaching Center Y with no data leakage.

---

## 18. Post-MVP Roadmap

### Phase 2: Growth (3-6 months post-MVP)
1. **Self-serve onboarding**: Institution signup flow with trial period.
2. **Billing & subscriptions**: Stripe integration with tiered plans.
3. **In-app notification center**: Bell icon with unread count.
4. **Advanced gradebook**: Weighted categories, grade curving, CSV export.
5. **Question banks**: Reusable question pools with tagging.
6. **Bulk operations**: Bulk user import, bulk grading.
7. **Basic analytics**: Student performance trends, course effectiveness.
8. **Email templates**: Customizable notification emails per institution.

### Phase 3: Maturity (6-12 months)
1. **SSO/SAML integration**: For school/university districts.
2. **PWA**: Progressive Web App with offline support.
3. **Parent/Observer accounts**: Limited access for parents.
4. **LTI 1.3 integration**: Interoperability with external tools.
5. **Discussion forums**: Async threaded discussions per course.
6. **Plagiarism detection**: Integration with external service.
7. **Custom branding**: Per-institution logo, colors, CNAME.
8. **API & Webhooks**: Public API for institution integrations.
9. **Advanced analytics**: Predictive analytics, at-risk student detection.
10. **Certificate generation**: Auto-generated PDF completion certificates.

### Phase 4: Scale (12+ months)
1. **Database sharding**: Horizontal scaling with Citus or manual partitioning.
2. **Multi-region deployment**: Data residency options for GDPR.
3. **White-label**: Complete white-label solution for large institutions.
4. **AI features**: AI-assisted grading suggestions, content recommendations.
5. **SIS integration**: Student Information System sync (PowerSchool, etc.).
6. **SCORM/xAPI**: eLearning standard compliance.
7. **Marketplace**: Shared course marketplace across institutions (opt-in).

---

## 19. Architectural Tradeoffs

### 19.1 Database: Shared vs. Per-Tenant

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| **Shared DB + RLS** | Simple, cost-effective, easy global identity, single connection pool | Noisy neighbor risk; one bad query affects all tenants | ✅ **MVP choice** |
| **Database per tenant** | Perfect isolation, easy per-tenant backups, compliance-friendly | Impossible to scale cost-effectively to thousands of tenants; hard global identity | ❌ Not viable at scale |
| **Hybrid/Sharded** | Best of both worlds: isolate large tenants, pool small ones | Complex operations; migration tools needed | ✅ **Post-MVP target** |

### 19.2 Auth: Managed vs. Custom

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| **Managed (Auth0/Clerk/WorkOS)** | SSO/SAML out of box, MFA, compliance certs | Expensive at scale ($0.02-0.07/MAU); vendor lock-in | Post-MVP |
| **Custom (NextAuth/Lucia)** | Free, full control, no vendor lock-in | Must build SSO/SAML manually; compliance burden | ✅ **MVP choice** |
| **Hybrid** | Custom auth with SSO add-on via WorkOS | Slightly more complex | Ideal long-term |

### 19.3 Frontend: SSR vs. SPA vs. Hybrid

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| **Full SSR (Next.js App Router)** | SEO, fast initial load | Expensive server compute; slow for interactive dashboards | Partial |
| **SPA (Client Components)** | Fast interactions, low server cost | Slow initial load; poor SEO | Partial |
| **Hybrid** | ISR/SSG for public pages; client components for dashboards | Slightly more complex architecture | ✅ **Recommended** |

### 19.4 File Storage: S3 Presigned URLs vs. Server Proxy

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| **Presigned URLs** | Offloads bandwidth, scales infinitely, cheaper | Slightly more complex client logic; URL expiration | ✅ **Recommended** |
| **Server proxy** | Simple client code; no URL expiration | Server becomes bottleneck for large files; expensive bandwidth | ❌ Not scalable |

### 19.5 Job Queue: Redis vs. Dedicated (BullMQ/Inngest)

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| **Redis + BullMQ** | Already using Redis; mature library | Redis memory pressure if queue grows large | ✅ **MVP choice** |
| **Inngest** | Managed, durable, great DX | New dependency; cost at scale | Post-MVP |
| **SQS/RabbitMQ** | Battle-tested, durable | Operational overhead | Enterprise scale |

### 19.6 Monorepo vs. Multi-Repo

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| **Monorepo (Turborepo/Nx)** | Shared types, single CI, easy refactoring | Larger clone; tooling complexity | ✅ **Recommended** |
| **Multi-repo** | Independent deploy cycles; smaller clones | Type sharing pain; cross-repo PRs | ❌ Premature for MVP |

---

## 20. Implementation Considerations

### 20.1 Monorepo Structure

```
leavy/
├── apps/
│   ├── web/                    # Next.js frontend
│   │   ├── app/                # App Router pages
│   │   ├── components/         # Shared UI components
│   │   └── lib/                # Frontend utilities
│   └── api/                    # Backend API (could be Next.js API routes or separate)
│       ├── routes/             # API route handlers
│       ├── services/           # Business logic
│       ├── middleware/          # Auth, tenant context, rate limiting
│       └── jobs/               # Background job definitions
├── packages/
│   ├── database/               # Prisma/Drizzle schemas, migrations, seeds
│   ├── shared-types/           # TypeScript types shared frontend/backend
│   ├── email/                  # Email templates (react-email)
│   ├── config/                 # ESLint, TypeScript config, Tailwind
│   └── utils/                  # Shared utility functions
├── docker/
│   └── docker-compose.yml      # Development infrastructure
├── docs/
│   └── ARCHITECTURE_PLAN.md    # This document
└── turbo.json                  # Turborepo configuration
```

### 20.2 Key Dependencies

#### Frontend
```json
{
  "next": "^14",
  "react": "^18",
  "typescript": "^5",
  "tailwindcss": "^3",
  "shadcn/ui": "UI components",
  "react-hook-form": "Form handling",
  "zod": "Validation",
  "tanstack/react-query": "Server state",
  "zustand": "Client state",
  "react-player": "YouTube embeds",
  "uploadthing" or "@aws-sdk/client-s3": "File uploads",
  "recharts": "Charts/analytics",
  "date-fns": "Date handling with timezone support"
}
```

#### Backend
```json
{
  "next": "^14",
  "next-auth": "Authentication",
  "drizzle-orm": "ORM (lightweight, TypeScript-native)",
  "pg": "PostgreSQL driver",
  "ioredis": "Redis client",
  "bullmq": "Job queue",
  "zod": "API validation",
  "resend": "Email delivery",
  "@aws-sdk/client-s3": "S3 operations",
  "@aws-sdk/s3-request-presigner": "Presigned URLs"
}
```

#### Dev Tools
```json
{
  "turbo": "Monorepo orchestration",
  "prettier": "Formatting",
  "eslint": "Linting",
  "vitest": "Testing",
  "playwright": "E2E testing",
  "drizzle-kit": "Schema migrations"
}
```

### 20.3 Database Schema (Core Tables)

```sql
-- Users (global identity)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  avatar_url TEXT,
  email_verified BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Institutions (tenants)
CREATE TABLE institutions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  timezone TEXT DEFAULT 'UTC',
  settings JSONB DEFAULT '{}',
  status TEXT DEFAULT 'active', -- active, suspended, deleted
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Institution memberships (User <-> Institution, carries role)
CREATE TABLE institution_memberships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  institution_id UUID NOT NULL REFERENCES institutions(id) ON DELETE CASCADE,
  role TEXT NOT NULL, -- owner, admin, teacher, student, observer
  status TEXT DEFAULT 'active', -- active, suspended, removed
  joined_at TIMESTAMPTZ DEFAULT now(),
  invited_by UUID REFERENCES users(id),
  UNIQUE(user_id, institution_id)
);

-- Courses
CREATE TABLE courses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  institution_id UUID NOT NULL REFERENCES institutions(id),
  title TEXT NOT NULL,
  description TEXT,
  cover_image_url TEXT,
  status TEXT DEFAULT 'draft', -- draft, published, completed, archived
  settings JSONB DEFAULT '{}',
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Course modules
CREATE TABLE course_modules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  order_index INTEGER NOT NULL,
  is_sequential BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Content items within modules (polymorphic via type + reference)
CREATE TABLE module_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  module_id UUID NOT NULL REFERENCES course_modules(id) ON DELETE CASCADE,
  item_type TEXT NOT NULL, -- 'class', 'note', 'homework', 'exam'
  item_id UUID NOT NULL,   -- references respective table
  order_index INTEGER NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Classes (YouTube videos)
CREATE TABLE classes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  youtube_url TEXT NOT NULL,
  youtube_thumbnail_url TEXT,
  duration_seconds INTEGER,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Notes (files)
CREATE TABLE notes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  file_url TEXT NOT NULL,
  file_name TEXT NOT NULL,
  file_size_bytes BIGINT,
  file_type TEXT,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Homework
CREATE TABLE homework (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  instructions TEXT,
  due_date TIMESTAMPTZ NOT NULL,
  allow_late BOOLEAN DEFAULT false,
  late_penalty_percent INTEGER DEFAULT 0,
  max_score INTEGER NOT NULL DEFAULT 100,
  submission_type TEXT DEFAULT 'both',
  allowed_file_types TEXT[],
  max_file_size_mb INTEGER DEFAULT 10,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Homework submissions
CREATE TABLE homework_submissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  homework_id UUID NOT NULL REFERENCES homework(id) ON DELETE CASCADE,
  student_id UUID NOT NULL REFERENCES users(id),
  text_response TEXT,
  file_url TEXT,
  file_name TEXT,
  submitted_at TIMESTAMPTZ DEFAULT now(),
  is_late BOOLEAN DEFAULT false,
  score INTEGER,
  feedback TEXT,
  graded_by UUID REFERENCES users(id),
  graded_at TIMESTAMPTZ,
  status TEXT DEFAULT 'submitted', -- draft, submitted, graded, returned
  UNIQUE(homework_id, student_id) -- One submission per student per homework
);

-- Exams
CREATE TABLE exams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  duration_minutes INTEGER NOT NULL DEFAULT 60,
  start_at TIMESTAMPTZ NOT NULL,
  end_at TIMESTAMPTZ NOT NULL,
  start_by_at TIMESTAMPTZ,
  shuffle_questions BOOLEAN DEFAULT false,
  shuffle_options BOOLEAN DEFAULT false,
  show_results TEXT DEFAULT 'after_grading',
  allow_retakes BOOLEAN DEFAULT false,
  max_retakes INTEGER DEFAULT 1,
  passing_score INTEGER DEFAULT 50,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Exam questions
CREATE TABLE exam_questions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  exam_id UUID NOT NULL REFERENCES exams(id) ON DELETE CASCADE,
  question_type TEXT NOT NULL, -- mcq_single, mcq_multi, true_false, short_answer, long_answer, fill_blank
  question_text TEXT NOT NULL,
  options JSONB,               -- For MCQ: [{text, isCorrect}]
  correct_answer TEXT,         -- For short answer/fill in blank
  points INTEGER DEFAULT 1,
  order_index INTEGER NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Exam attempts
CREATE TABLE exam_attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  exam_id UUID NOT NULL REFERENCES exams(id),
  student_id UUID NOT NULL REFERENCES users(id),
  started_at TIMESTAMPTZ DEFAULT now(),
  submitted_at TIMESTAMPTZ,
  time_spent_seconds INTEGER,
  total_score INTEGER,
  status TEXT DEFAULT 'in_progress', -- in_progress, submitted, graded
  attempt_number INTEGER DEFAULT 1,
  UNIQUE(exam_id, student_id, attempt_number)
);

-- Exam responses
CREATE TABLE exam_responses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  attempt_id UUID NOT NULL REFERENCES exam_attempts(id) ON DELETE CASCADE,
  question_id UUID NOT NULL REFERENCES exam_questions(id),
  student_answer TEXT,
  is_correct BOOLEAN,
  score INTEGER,
  graded_by UUID REFERENCES users(id),
  graded_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Course enrollments
CREATE TABLE course_enrollments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  student_id UUID NOT NULL REFERENCES users(id),
  membership_id UUID NOT NULL REFERENCES institution_memberships(id),
  enrolled_at TIMESTAMPTZ DEFAULT now(),
  completed_at TIMESTAMPTZ,
  progress_percent INTEGER DEFAULT 0,
  UNIQUE(course_id, student_id)
);

-- Content progress (tracking which items student has completed)
CREATE TABLE content_progress (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  enrollment_id UUID NOT NULL REFERENCES course_enrollments(id) ON DELETE CASCADE,
  module_item_id UUID NOT NULL REFERENCES module_items(id),
  completed BOOLEAN DEFAULT false,
  completed_at TIMESTAMPTZ,
  UNIQUE(enrollment_id, module_item_id)
);

-- Notifications
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  institution_id UUID REFERENCES institutions(id),
  type TEXT NOT NULL,
  title TEXT NOT NULL,
  body TEXT,
  data JSONB DEFAULT '{}',
  read BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### 20.4 API Route Design (RESTful)

```
# Auth
POST   /api/auth/send-magic-link
GET    /api/auth/verify-magic-link?token=
GET    /api/auth/session
DELETE /api/auth/session

# Users
GET    /api/users/me
PATCH  /api/users/me
GET    /api/institutions/:instId/users                # List institution members
POST   /api/institutions/:instId/users/invite         # Invite user

# Institutions
GET    /api/institutions                              # List user's institutions
POST   /api/institutions                              # Create (super admin)
GET    /api/institutions/:instId                      # Get details
PATCH  /api/institutions/:instId                      # Update settings

# Courses
GET    /api/institutions/:instId/courses              # List courses
POST   /api/institutions/:instId/courses              # Create course
GET    /api/institutions/:instId/courses/:courseId    # Get course
PATCH  /api/institutions/:instId/courses/:courseId    # Update course
DELETE /api/institutions/:instId/courses/:courseId    # Soft-delete

# Course Content (Classes, Notes)
GET    /api/courses/:courseId/modules                 # Get modules + items
POST   /api/courses/:courseId/modules                 # Create module
PATCH  /api/courses/:courseId/modules/:modId          # Update module
DELETE /api/courses/:courseId/modules/:modId          # Delete module
POST   /api/courses/:courseId/modules/:modId/items    # Add item (class/note/homework/exam)

# Classes
POST   /api/courses/:courseId/classes
PATCH  /api/courses/:courseId/classes/:classId
DELETE /api/courses/:courseId/classes/:classId

# Notes
POST   /api/courses/:courseId/notes
PATCH  /api/courses/:courseId/notes/:noteId
DELETE /api/courses/:courseId/notes/:noteId

# Homework
POST   /api/courses/:courseId/homework
PATCH  /api/courses/:courseId/homework/:hwId
DELETE /api/courses/:courseId/homework/:hwId
GET    /api/homework/:hwId/submissions                 # Teacher: list submissions
POST   /api/homework/:hwId/submit                      # Student: submit
GET    /api/homework/:hwId/my-submission               # Student: view own
PATCH  /api/submissions/:subId/grade                   # Teacher: grade

# Exams
POST   /api/courses/:courseId/exams
PATCH  /api/courses/:courseId/exams/:examId
DELETE /api/courses/:courseId/exams/:examId
POST   /api/exams/:examId/start                        # Student: begin attempt
POST   /api/exams/:examId/submit                       # Student: submit attempt
GET    /api/exams/:examId/results/:attemptId           # Student: view results
GET    /api/exams/:examId/submissions                  # Teacher: list submissions
PATCH  /api/exam-responses/:responseId/grade           # Teacher: grade response

# Enrollments
POST   /api/courses/:courseId/enroll                   # Enroll student
DELETE /api/courses/:courseId/enroll/:studentId        # Unenroll
GET    /api/courses/:courseId/students                 # List enrolled

# Dashboard / Analytics
GET    /api/student/dashboard                          # Student dashboard data
GET    /api/teacher/dashboard                          # Teacher dashboard data
GET    /api/institutions/:instId/dashboard             # Admin dashboard

# Files
POST   /api/upload/presigned-url                       # Get presigned upload URL
GET    /api/files/:fileId/download                     # Get presigned download URL

# Notifications
GET    /api/notifications                              # List user notifications
PATCH  /api/notifications/:id/read                     # Mark as read
POST   /api/notifications/read-all                     # Mark all as read
```

### 20.5 Implementation Order (MVP)

1. **Week 1-2: Foundation**
   - Monorepo setup (Turborepo)
   - Database schema + migrations (Drizzle)
   - Authentication (magic link via next-auth)
   - Core middleware (tenant context, auth)

2. **Week 3-4: Institution & User Management**
   - Institution CRUD
   - Membership system (invite, accept, roles)
   - User profile & settings

3. **Week 5-7: Course System**
   - Course CRUD with draft/published/archived states
   - Module system (ordered content)
   - Class creation (YouTube embeds)
   - Notes (file uploads with presigned URLs)
   - Enrollment system

4. **Week 8-9: Homework**
   - Homework creation (teacher)
   - Submission + file upload (student)
   - Grading + feedback (teacher)

5. **Week 10-11: Exams**
   - Exam creation with question types
   - Timed exam taking flow
   - Auto-grading (MCQ) + manual grading
   - Result display

6. **Week 12-13: Dashboards & Notifications**
   - Student dashboard (courses, progress, grades)
   - Teacher dashboard (submissions, roster)
   - Admin dashboard (overview)
   - Email notifications (deadlines, grades)

7. **Week 14: Polish & Launch**
   - RLS enforcement on all tables
   - Rate limiting
   - Error handling & loading states
   - Responsive design
   - Deployment setup

---

## Appendix A: Decision Log

| Decision | Options | Chosen | Rationale |
|----------|---------|--------|-----------|
| ORM | Prisma vs Drizzle | Drizzle | Lighter, better TypeScript inference, better SQL control |
| Auth | NextAuth vs Lucia vs Clerk | NextAuth (custom) | Free, flexible, good Next.js integration |
| UI Library | Tailwind + shadcn/ui vs MUI vs Chakra | shadcn/ui | Modern, customizable, copy-paste components |
| API Style | REST vs GraphQL vs tRPC | REST | Standard, simpler caching and documentation |
| Job Queue | BullMQ vs Inngest vs SQS | BullMQ | Already using Redis; mature |
| File Storage | S3 vs Cloudflare R2 vs UploadThing | S3 (AWS SDK) | Industry standard; presigned URL support |
| Email | Resend vs SendGrid vs SES | Resend | Best DX; react-email integration |

---

## Appendix B: Open Questions (for stakeholder discussion)

1. **Self-serve vs. managed onboarding**: Should institutions be able to sign up themselves, or is this a managed sales process initially? (Affects: billing integration priority, admin tooling)

2. **Free tier strategy**: What limits define the free tier? Number of students? Courses? Storage? (Affects: SaaS architecture, quota system)

3. **Data residency**: Are there geographic constraints on where data must be stored? (Affects: deployment architecture, cloud provider choice)

4. **White-label timeline**: Is white-labeling (custom domain, branding) a near-term or long-term requirement? (Affects: multi-tenant routing, SSL architecture)

5. **Mobile priority**: How important is mobile access for MVP? PWA sufficient, or should native apps be on the roadmap? (Affects: API design, responsive priority)

6. **Integration requirements**: Any existing systems (SIS, payment gateways, analytics) that must be integrated from day one? (Affects: webhook/API design)

7. **Content migration**: Is there existing course content to import from another LMS? (Affects: import tool prioritization)

8. **Compliance timeline**: How soon is GDPR/FERPA/COPPA compliance needed? Is legal review available? (Affects: data model, audit logging priority)

---

*End of Architecture Plan. This document serves as the blueprint for all subsequent implementation phases. No code should be written that contradicts the architectural decisions herein without updating this document first.*
