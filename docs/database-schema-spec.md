# Leavy LMS — Database Schema Specification

> **Version:** 0.1.0  
> **Last Updated:** June 14, 2026  
> **Status:** Draft — derived from 5 rounds of stakeholder interviews  
> **ORM:** Drizzle ORM (TypeScript)  
> **Database:** PostgreSQL 16+

---

## 1. Design Decisions Summary

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | **Role on `users` table** (teacher/student) | Simple. No need to infer from memberships. Teacher can also be a student. |
| 2 | **Institution memberships are teacher-only** | Students enroll directly in courses. No institution membership needed. |
| 3 | **Single institution owner** via `is_owner` flag | Owner is transferable by flipping the flag on two membership rows. |
| 4 | **Direct course enrollment** — no institution membership required for students | Simplest flow. Student sees a course → enrolls. |
| 5 | **Soft deletes everywhere** via `deleted_at TIMESTAMPTZ` | Preserves audit trail. Queries filter `WHERE deleted_at IS NULL`. |
| 6 | **MCQ options as JSONB array** + `correct_option_index` | Flexible option count per question (2–N). Both JSONB and index column for fast lookup. |
| 7 | **Study materials + exams as separate ordered sections** | Materials come first (ordered), exams follow (separately ordered). No interleaving. |
| 8 | **Multiple exams per course** | Teacher decides how many exams. Course pass = pass X% of exams. |
| 9 | **S3/Cloudflare R2** with object key storage | Store key in DB, generate presigned URLs at read time. |
| 10 | **Public institution directory** with search + category | Institutions are discoverable. Optional category field for filtering. |
| 11 | **Courses have start/end dates** | Bounded availability. Read-only after end_date. |
| 12 | **Exams have own start/end window** | Each exam independently scheduled within course window. |
| 13 | **Student sees score + pass/fail only** | No per-question detail for students. Teacher sees full breakdown. |
| 14 | **Teacher creates student accounts** with password | Backend uses Firebase Admin SDK to create auth account + DB record. |
| 15 | **No study material progress tracking** (MVP) | Only exam results tracked. Materials are free-form consumption. |

---

## 2. Entity Relationship Diagram

```
┌──────────┐       ┌─────────────────────────┐       ┌──────────────┐
│   User   │       │ InstitutionMembership   │       │ Institution  │
│ (global) │──1:N──│  (teachers only)        │──N:1──│  (tenant)    │
├──────────┤       ├─────────────────────────┤       ├──────────────┤
│ id       │       │ id                      │       │ id           │
│ firebase │       │ user_id (FK→users)      │       │ name         │
│  _uid    │       │ institution_id (FK→inst)│       │ slug         │
│ email    │       │ is_owner BOOLEAN        │       │ category     │
│ name     │       │ deleted_at              │       │ timezone     │
│ role     │       └─────────────────────────┘       │ created_by   │
│ avatar   │                                         │ deleted_at   │
│ deleted  │       ┌─────────────────────────┐       └──────────────┘
│  _at     │       │  TeacherInvitation      │                │
└──────────┘       ├─────────────────────────┤                │
       │           │ institution_id          │────────────────┘
       │           │ email                   │
       │           │ type (invite/request)   │       ┌──────────────┐
       │           │ status                  │       │   Course     │
       │           └─────────────────────────┘       ├──────────────┤
       │                                             │ institution  │
       │           ┌─────────────────────────┐       │  _id (FK)    │
       ├──1:N──────│   CourseEnrollment      │──N:1──│ title        │
       │           ├─────────────────────────┤       │ status       │
       │           │ course_id (FK→courses)  │       │ start_date   │
       │           │ student_id (FK→users)   │       │ end_date     │
       │           │ enrolled_at             │       │ passing_exam │
       │           │ completed_at            │       │  s_percent   │
       │           │ deleted_at              │       │ created_by   │
       │           └─────────────────────────┘       │ deleted_at   │
       │                                             └──────────────┘
       │                                                    │
       │                    ┌───────────────────────────────┤
       │                    │                               │
       │           ┌────────┴────────┐           ┌──────────┴──────────┐
       │           │ StudyMaterial   │           │       Exam          │
       │           ├─────────────────┤           ├─────────────────────┤
       │           │ course_id (FK)  │           │ course_id (FK)      │
       │           │ type (file/yt)  │           │ title               │
       │           │ title           │           │ description         │
       │           │ file_key (S3)   │           │ duration_minutes    │
       │           │ file_name       │           │ passing_score       │
       │           │ youtube_url     │           │ max_attempts        │
       │           │ order_index     │           │ shuffle_questions   │
       │           │ created_by (FK) │           │ start_at            │
       │           │ deleted_at      │           │ end_at              │
       │           └─────────────────┘           │ created_by (FK)     │
       │                                         │ deleted_at          │
       │                                         └──────────┬──────────┘
       │                                                    │
       │                                          ┌─────────┴─────────┐
       │                                          │   ExamQuestion    │
       │                                          ├───────────────────┤
       │                                          │ exam_id (FK)      │
       │                                          │ question_text     │
       │                                          │ options (JSONB)   │
       │                                          │ correct_opt_idx   │
       │                                          │ points            │
       │                                          │ order_index       │
       │                                          │ deleted_at        │
       │                                          └───────────────────┘
       │
       ├──────────┐
       │          │
       │  ┌───────┴────────────┐
       │  │   ExamAttempt      │
       │  ├────────────────────┤
       └──│ student_id (FK)    │
          │ exam_id (FK)       │
          │ started_at         │
          │ submitted_at       │
          │ total_score        │
          │ total_points       │
          │ passed             │
          │ attempt_number     │
          │ status             │
          │ deleted_at         │
          └────────┬───────────┘
                   │
          ┌────────┴───────────┐
          │   ExamResponse     │
          ├────────────────────┤
          │ attempt_id (FK)    │
          │ question_id (FK)   │
          │ selected_opt_idx   │
          │ is_correct         │
          │ score              │
          │ deleted_at         │
          └────────────────────┘
```

---

## 3. Table Definitions

### 3.1 `users`

Global identity for all users. One account across all institutions.

```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  firebase_uid  TEXT UNIQUE NOT NULL,
  email         TEXT UNIQUE NOT NULL,
  name          TEXT NOT NULL,
  role          TEXT NOT NULL CHECK (role IN ('teacher', 'student')),
  avatar_url    TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at    TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_firebase_uid ON users(firebase_uid);
```

**Notes:**
- `role` is the user's primary role at registration. A teacher can also enroll as a student in other institutions' courses without changing this.
- Firebase UID is the link between Firebase Auth and our database. Always verified server-side via Firebase Admin SDK.
- Soft delete via `deleted_at`. Deleted users cannot log in, but their past exam data is preserved.

---

### 3.2 `institutions`

Tenant/organization container. Created by a teacher who becomes the owner.

```sql
CREATE TABLE institutions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          TEXT NOT NULL,
  slug          TEXT UNIQUE NOT NULL,
  category      TEXT,                            -- e.g. 'school', 'university', 'coaching', 'corporate'
  timezone      TEXT NOT NULL DEFAULT 'UTC',
  description   TEXT,                            -- Shown in public directory
  logo_url      TEXT,
  created_by    UUID NOT NULL REFERENCES users(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at    TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_institutions_slug ON institutions(slug);
CREATE INDEX idx_institutions_category ON institutions(category);
CREATE INDEX idx_institutions_name ON institutions(name);  -- For public directory search
```

**Notes:**
- `slug` is URL-safe identifier (e.g., `springfield-high-school`). Generated from name, editable.
- `category` enables filtering in the public directory (see Round 5).
- `created_by` records who created it. Ownership is tracked separately via `institution_memberships.is_owner`.

---

### 3.3 `institution_memberships`

Teacher-only membership in an institution. Not used for students.

```sql
CREATE TABLE institution_memberships (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  institution_id  UUID NOT NULL REFERENCES institutions(id) ON DELETE CASCADE,
  is_owner        BOOLEAN NOT NULL DEFAULT false,
  joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at      TIMESTAMPTZ,

  UNIQUE(user_id, institution_id)
);

-- Indexes
CREATE INDEX idx_membership_user ON institution_memberships(user_id);
CREATE INDEX idx_membership_institution ON institution_memberships(institution_id);
CREATE INDEX idx_membership_owner ON institution_memberships(institution_id, is_owner)
  WHERE is_owner = true AND deleted_at IS NULL;
```

**Notes:**
- Only one row per (user_id, institution_id). 
- `is_owner = true` on exactly one row per institution (enforced at application level).
- Ownership transfer: set old owner's `is_owner` to false, set new owner's to true. Single DB transaction.
- When a teacher is removed, `deleted_at` is set. Data preserved for audit.
- A teacher can belong to many institutions. They own some, are members of others.

---

### 3.4 `teacher_invitations`

Handles both directions: institution inviting a teacher, and a teacher requesting to join.

```sql
CREATE TABLE teacher_invitations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  institution_id  UUID NOT NULL REFERENCES institutions(id) ON DELETE CASCADE,
  email           TEXT NOT NULL,
  type            TEXT NOT NULL CHECK (type IN ('invitation', 'join_request')),
  -- invitation: institution owner invites a teacher by email
  -- join_request: teacher requests to join an institution
  invited_by      UUID REFERENCES users(id),        -- For 'invitation': who sent the invite
  requested_by    UUID REFERENCES users(id),        -- For 'join_request': who requested to join
  status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'accepted', 'declined', 'expired')),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at     TIMESTAMPTZ,
  deleted_at      TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_invitation_institution ON teacher_invitations(institution_id, status);
CREATE INDEX idx_invitation_email ON teacher_invitations(email, status);
```

**Notes:**
- `invited_by` is only set when `type = 'invitation'`.
- `requested_by` is only set when `type = 'join_request'`.
- On acceptance: create `institution_memberships` row for the user (with `is_owner = false`).
- Invitations expire after 7 days (application-level logic).

---

### 3.5 `courses`

The central content container. Created by teachers within their institutions.

```sql
CREATE TABLE courses (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  institution_id        UUID NOT NULL REFERENCES institutions(id) ON DELETE CASCADE,
  title                 TEXT NOT NULL,
  description           TEXT,
  cover_image_key       TEXT,                      -- S3/R2 object key for cover image
  status                TEXT NOT NULL DEFAULT 'draft'
                          CHECK (status IN ('draft', 'published', 'archived')),
  enrollment_type       TEXT NOT NULL DEFAULT 'open'
                          CHECK (enrollment_type IN ('open', 'invite_only')),
  start_date            TIMESTAMPTZ,               -- NULL = available immediately when published
  end_date              TIMESTAMPTZ,               -- NULL = no end date
  passing_exams_percent INTEGER NOT NULL DEFAULT 100,  -- % of exams student must pass (e.g., 70)
  created_by            UUID NOT NULL REFERENCES users(id),
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at            TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_courses_institution ON courses(institution_id, status);
CREATE INDEX idx_courses_published ON courses(status, start_date, end_date)
  WHERE status = 'published' AND deleted_at IS NULL;
```

**Notes:**
- `status` lifecycle: `draft` → `published` → `archived`. `draft` can also go directly to `archived`.
- `enrollment_type`: `open` = any student can enroll. `invite_only` = only teacher-added students.
- `start_date` / `end_date`: NULL means no bound. If set, course is only visible between these dates. After `end_date`, course becomes read-only.
- `passing_exams_percent`: e.g., 70 means student must pass at least 70% of the course's exams. 100 means all exams must be passed.

---

### 3.6 `course_enrollments`

Links a student to a course. No institution membership required.

```sql
CREATE TABLE course_enrollments (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id     UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  student_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  enrolled_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at  TIMESTAMPTZ,                      -- Set when student passes the course
  deleted_at    TIMESTAMPTZ,

  UNIQUE(course_id, student_id)
);

-- Indexes
CREATE INDEX idx_enrollment_student ON course_enrollments(student_id, deleted_at);
CREATE INDEX idx_enrollment_course ON course_enrollments(course_id, deleted_at);
```

**Notes:**
- `student_id` references any user (a self-enrolled student, a teacher-as-student, or a teacher-created student).
- No `institution_memberships` check. Any user can enroll in any `published` + `open` course.
- `completed_at` is set when student passes enough exams per `courses.passing_exams_percent`.
- Unenrolling sets `deleted_at` (soft delete).

---

### 3.7 `study_materials`

Files and YouTube videos within a course. Materials come before exams in course layout.

```sql
CREATE TABLE study_materials (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id             UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  type                  TEXT NOT NULL CHECK (type IN ('file', 'youtube')),
  title                 TEXT NOT NULL,
  description           TEXT,

  -- File fields (when type = 'file')
  file_key              TEXT,                      -- S3/R2 object key
  file_name             TEXT,                      -- Original filename
  file_size_bytes       BIGINT,
  file_type             TEXT,                      -- MIME type (e.g., 'application/pdf')

  -- YouTube fields (when type = 'youtube')
  youtube_url           TEXT,                      -- Full YouTube URL
  youtube_thumbnail_url TEXT,                      -- Cached thumbnail from YouTube

  order_index           INTEGER NOT NULL DEFAULT 0,
  created_by            UUID NOT NULL REFERENCES users(id),
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at            TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_material_course ON study_materials(course_id, deleted_at, order_index);
```

**Notes:**
- Single table with `type` discriminator. File fields are NULL when `type = 'youtube'` and vice versa.
- `file_key` stores the S3/R2 object key (e.g., `institutions/{instId}/courses/{courseId}/materials/{uuid}.pdf`). Presigned URLs generated at read time.
- `order_index` controls display order. Materials are ordered independently of exams.
- No progress tracking for materials in MVP.

---

### 3.8 `exams`

MCQ exams within a course. Each exam has its own availability window.

```sql
CREATE TABLE exams (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id           UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  title               TEXT NOT NULL,
  description         TEXT,
  duration_minutes    INTEGER NOT NULL DEFAULT 30,   -- 0 = untimed
  passing_score       INTEGER NOT NULL DEFAULT 50,   -- Percentage to pass this exam
  max_attempts        INTEGER NOT NULL DEFAULT 3,
  shuffle_questions   BOOLEAN NOT NULL DEFAULT false, -- Randomize question order per attempt
  start_at            TIMESTAMPTZ,                   -- When exam becomes available
  end_at              TIMESTAMPTZ,                   -- Deadline (must submit by)
  order_index         INTEGER NOT NULL DEFAULT 0,    -- Order among exams in the course
  created_by          UUID NOT NULL REFERENCES users(id),
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at          TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_exam_course ON exams(course_id, deleted_at, order_index);
CREATE INDEX idx_exam_availability ON exams(course_id, start_at, end_at)
  WHERE deleted_at IS NULL;
```

**Notes:**
- `start_at`: when students can begin the exam. NULL = available immediately when course is published.
- `end_at`: deadline. Student must submit before this. If `duration_minutes > 0`, student must also finish within their personal timer.
- If a student starts at `end_at - 10min` but `duration_minutes = 30`, the exam auto-submits at `end_at` (server-enforced).
- `order_index` determines display order among exams (separate from study materials).
- `passing_score`: e.g., 50 means >=50% to pass this individual exam.
- `delete_at` soft deletes the exam. Already-submitted attempts are preserved.

---

### 3.9 `exam_questions`

MCQ questions within an exam. Options stored as JSONB for flexibility.

```sql
CREATE TABLE exam_questions (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  exam_id             UUID NOT NULL REFERENCES exams(id) ON DELETE CASCADE,
  question_text       TEXT NOT NULL,
  options             JSONB NOT NULL,                 -- Array of option objects
  correct_option_index INTEGER NOT NULL,               -- 0-based index into options array
  points              INTEGER NOT NULL DEFAULT 1,
  order_index         INTEGER NOT NULL DEFAULT 0,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at          TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_question_exam ON exam_questions(exam_id, deleted_at, order_index);
```

**`options` JSONB structure:**

```json
[
  { "text": "Paris", "id": "opt_0" },
  { "text": "London", "id": "opt_1" },
  { "text": "Berlin", "id": "opt_2" },
  { "text": "Madrid", "id": "opt_3" }
]
```

- `correct_option_index = 0` means the first option ("Paris") is correct.
- Each option has a `text` (display) and `id` (stable identifier for shuffling).
- The `id` allows frontend to track which option was selected even after shuffling.
- `correct_option_index` is duplicated as a column (not just inside JSONB) for fast server-side auto-grading without parsing JSONB.

---

### 3.10 `exam_attempts`

A student's attempt at an exam. Tracks the overall result.

```sql
CREATE TABLE exam_attempts (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  exam_id             UUID NOT NULL REFERENCES exams(id) ON DELETE CASCADE,
  student_id          UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  started_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  submitted_at        TIMESTAMPTZ,
  time_spent_seconds  INTEGER,                       -- Calculated on submit
  total_score         INTEGER,                       -- Sum of correct response scores
  total_points        INTEGER,                       -- Sum of all question points
  passed              BOOLEAN,                       -- (total_score / total_points) * 100 >= exam.passing_score
  status              TEXT NOT NULL DEFAULT 'in_progress'
                        CHECK (status IN ('in_progress', 'submitted', 'timed_out')),
  attempt_number      INTEGER NOT NULL DEFAULT 1,
  deleted_at          TIMESTAMPTZ,

  UNIQUE(exam_id, student_id, attempt_number)
);

-- Indexes
CREATE INDEX idx_attempt_exam ON exam_attempts(exam_id, deleted_at);
CREATE INDEX idx_attempt_student ON exam_attempts(student_id, deleted_at);
```

**Notes:**
- Unique on `(exam_id, student_id, attempt_number)` prevents duplicate submission.
- `attempt_number` is computed as `MAX(attempt_number) + 1` for the (exam, student) pair.
- `status = 'timed_out'` when the client's timer expires and the server receives no explicit submit. The server also validates submit time against `started_at + duration_minutes`.
- `passed` is computed during auto-grading: `(total_score / total_points) * 100 >= exam.passing_score`.
- A student can retake up to `exam.max_attempts` times.

---

### 3.11 `exam_responses`

Per-question answer within an exam attempt.

```sql
CREATE TABLE exam_responses (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  attempt_id          UUID NOT NULL REFERENCES exam_attempts(id) ON DELETE CASCADE,
  question_id         UUID NOT NULL REFERENCES exam_questions(id) ON DELETE CASCADE,
  selected_option_index INTEGER,                     -- 0-based index. NULL = unanswered
  is_correct          BOOLEAN,
  score               INTEGER NOT NULL DEFAULT 0,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at          TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_response_attempt ON exam_responses(attempt_id);
```

**Notes:**
- `selected_option_index` is the student's chosen option. Matches `exam_questions.correct_option_index` structure.
- `selected_option_index = NULL` means the student did not answer this question (score = 0).
- `is_correct` is set during auto-grading: `selected_option_index == correct_option_index`.
- `score` = `exam_questions.points` if correct, 0 otherwise. No partial credit in MVP.

---

## 4. Course Pass/Fail Logic

A student passes a course when they pass at least `courses.passing_exams_percent`% of the course's exams.

```
passed_exams = COUNT(exam_attempts WHERE passed = true AND deleted_at IS NULL)
total_exams = COUNT(exams WHERE deleted_at IS NULL)

IF (passed_exams / total_exams) * 100 >= passing_exams_percent
  → course_enrollments.completed_at = now()
  → Student has "passed" the course
```

**Example:** Course has 4 exams, `passing_exams_percent = 50`. Student only needs to pass 2 of 4 exams.  
**Example:** Course has 3 exams, `passing_exams_percent = 100`. Student must pass all 3 exams.

This is computed on each exam submission. Recalculated on exam deletion/addition by the teacher.

---

## 5. Index Summary

| Table | Index | Columns | Purpose |
|-------|-------|---------|---------|
| users | `idx_users_email` | `email` | Login lookup |
| users | `idx_users_firebase_uid` | `firebase_uid` | Firebase token verification |
| institutions | `idx_institutions_slug` | `slug` | URL routing |
| institutions | `idx_institutions_category` | `category` | Directory filtering |
| institutions | `idx_institutions_name` | `name` | Directory search |
| institution_memberships | `idx_membership_user` | `user_id` | "My institutions" query |
| institution_memberships | `idx_membership_institution` | `institution_id` | "Institution members" query |
| institution_memberships | `idx_membership_owner` | `institution_id, is_owner` (partial) | Fast owner lookup |
| teacher_invitations | `idx_invitation_institution` | `institution_id, status` | Pending invites for institution |
| teacher_invitations | `idx_invitation_email` | `email, status` | "My invitations" for a user |
| courses | `idx_courses_institution` | `institution_id, status` | Course listing |
| courses | `idx_courses_published` | `status, start_date, end_date` (partial) | Discoverable courses |
| course_enrollments | `idx_enrollment_student` | `student_id, deleted_at` | "My courses" query |
| course_enrollments | `idx_enrollment_course` | `course_id, deleted_at` | "Course students" query |
| study_materials | `idx_material_course` | `course_id, deleted_at, order_index` | Ordered materials listing |
| exams | `idx_exam_course` | `course_id, deleted_at, order_index` | Ordered exams listing |
| exams | `idx_exam_availability` | `course_id, start_at, end_at` (partial) | Active exams check |
| exam_questions | `idx_question_exam` | `exam_id, deleted_at, order_index` | Ordered questions |
| exam_attempts | `idx_attempt_exam` | `exam_id, deleted_at` | Teacher result view |
| exam_attempts | `idx_attempt_student` | `student_id, deleted_at` | Student result view |
| exam_responses | `idx_response_attempt` | `attempt_id` | Per-attempt response lookup |

---

## 6. Drizzle ORM Considerations

### 6.1 UUID Primary Keys
Use `uuid().primaryKey().defaultRandom()` for all tables.

### 6.2 Enums
PostgreSQL native enums via `pgEnum`:
- `user_role` → `'teacher'`, `'student'`
- `course_status` → `'draft'`, `'published'`, `'archived'`
- `enrollment_type` → `'open'`, `'invite_only'`
- `material_type` → `'file'`, `'youtube'`
- `invitation_type` → `'invitation'`, `'join_request'`
- `invitation_status` → `'pending'`, `'accepted'`, `'declined'`, `'expired'`
- `attempt_status` → `'in_progress'`, `'submitted'`, `'timed_out'`

### 6.3 Relations
Define using `relations()` from `drizzle-orm`:
- `users` → `institution_memberships`, `course_enrollments`, `exam_attempts`, `courses`, `study_materials`, `exams`
- `institutions` → `institution_memberships`, `courses`
- `courses` → `course_enrollments`, `study_materials`, `exams`
- `exams` → `exam_questions`, `exam_attempts`
- `exam_attempts` → `exam_responses`

### 6.4 Schema File Structure
```
packages/database/src/
├── schema/
│   ├── users.ts
│   ├── institutions.ts
│   ├── memberships.ts
│   ├── invitations.ts
│   ├── courses.ts
│   ├── enrollments.ts
│   ├── materials.ts
│   ├── exams.ts
│   ├── questions.ts
│   ├── attempts.ts
│   ├── responses.ts
│   └── enums.ts
├── relations.ts        # All drizzle relations
├── index.ts            # Re-exports
└── migrate.ts          # Migration runner
```

---

## 7. Data Integrity Rules

### 7.1 Ownership
- Exactly one `institution_memberships` row per institution must have `is_owner = true` and `deleted_at IS NULL`.
- When transferring ownership, both rows updated atomically.

### 7.2 Enrollment Validation
- Student must not be deleted (`users.deleted_at IS NULL`).
- Course must be `published` and within `start_date`/`end_date` range (if set).
- For `invite_only` courses, enrollment must be initiated by a teacher.

### 7.3 Exam Attempt Validation
- Student must be enrolled in the course (`course_enrollments` with `deleted_at IS NULL`).
- Exam must not be deleted.
- Current time must be between `exam.start_at` and `exam.end_at` (if set).
- `attempt_number` must be <= `exam.max_attempts`.
- No concurrent in-progress attempt for same (exam, student).

### 7.4 Soft Delete Propagation
- Deleting a course soft-deletes its study materials and exams (cascade at application level).
- Deleting an exam soft-deletes its questions (cascade at application level).
- Existing exam attempts and responses are preserved (historical data).
- Deleting an institution soft-deletes its courses and teacher memberships.

---

## 8. Soft Delete Query Pattern

Every query against soft-deletable tables MUST include:

```sql
WHERE deleted_at IS NULL
```

**Helper (Drizzle):**
```typescript
// In where clause
import { isNull } from 'drizzle-orm';

.where(isNull(table.deletedAt))
```

Consider a Drizzle custom base query or repository pattern to enforce this automatically.

---

## 9. Firebase Auth Integration

### 9.1 Registration Flow
1. Client calls Firebase Auth `createUserWithEmailAndPassword`.
2. Client sends Firebase ID token to `POST /api/auth/register`.
3. Backend verifies token via Firebase Admin SDK.
4. Backend extracts `uid`, `email`, `name`.
5. Backend creates `users` row with `firebase_uid`, `email`, `name`, `role`.
6. Returns user record.

### 9.2 Teacher Creates Student
1. Teacher's client sends student email + password to `POST /api/institutions/:id/students`.
2. Backend calls Firebase Admin SDK `createUser({ email, password })`.
3. Backend creates `users` row with `role = 'student'`.
4. No `institution_memberships` row (students don't need it).
5. Returns student record.

### 9.3 Auth Middleware
All protected API routes:
1. Extract Firebase ID token from `Authorization: Bearer <token>` header.
2. Verify via Firebase Admin SDK.
3. Look up `users` row by `firebase_uid`.
4. Check `users.deleted_at IS NULL`.
5. Attach `{ userId, role }` to request context.

---

## 10. File Storage (S3/R2) Pattern

### 10.1 Object Key Convention
```
institutions/{institutionId}/courses/{courseId}/materials/{uuid}.{ext}
institutions/{institutionId}/courses/{courseId}/covers/{uuid}.{ext}
```

### 10.2 Upload Flow
1. Client calls `POST /api/upload/presigned-url` with file metadata.
2. Backend generates an S3 presigned PUT URL (5-minute expiry).
3. Client uploads directly to S3 via the presigned URL.
4. Client confirms upload → Backend creates `study_materials` row with `file_key`.

### 10.3 Download/Access Flow
1. Client requests `GET /api/materials/:id/download`.
2. Backend generates presigned GET URL (15-minute expiry) from `study_materials.file_key`.
3. Client receives URL and fetches directly from S3.

---

## 11. Table Size Estimates (MVP)

| Table | Est. Rows (100 institutions) | Growth Driver |
|-------|------------------------------|---------------|
| users | 10,000 | New signups |
| institutions | 100 | Teacher creation |
| institution_memberships | 500 | Teacher invites |
| teacher_invitations | 200 | Invite/request activity |
| courses | 1,000 | Teacher course creation |
| course_enrollments | 50,000 | Student enrollment |
| study_materials | 10,000 | Content uploads |
| exams | 2,000 | Exam creation |
| exam_questions | 40,000 | 20 questions/exam avg |
| exam_attempts | 100,000 | Student attempts |
| exam_responses | 2,000,000 | 20 responses/attempt avg |

**Largest table:** `exam_responses` at ~2M rows. Well within single PostgreSQL instance capacity.

---

*End of Database Schema Specification. All decisions derived from 5 rounds of stakeholder interviews. No code should be written that contradicts this spec without updating this document first.*
