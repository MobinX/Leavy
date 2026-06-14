# Leavy LMS — Minimum Viable Product Plan

> **Version:** 0.1.0  
> **Last Updated:** June 14, 2026  
> **Scope:** Bare minimum — 2 roles, course-centric, no fluff

---

## 1. Core Concept

Leavy is a multi-institution Learning Management System. Teachers create institutions, build courses with study materials and auto-graded MCQ exams, and students enroll to learn.

**Hard constraints:**
- Only 2 user roles: **Teacher** and **Student**
- The Institution Owner **is** a Teacher (no separate Owner/Admin role)
- Firebase Auth for login
- PostgreSQL for all data
- Next.js + TypeScript frontend/backend

---

## 2. Roles & Permissions

### 2.1 Teacher
A Teacher is also the Institution Owner of the institutions they create.

| Action | Permission |
|--------|:----------:|
| Create institution | ✅ (becomes owner) |
| Edit institution settings | ✅ (own institutions only) |
| Invite another teacher to institution | ✅ (own institutions) |
| Accept/decline teacher join request | ✅ (own institutions) |
| Remove teacher from institution | ✅ (own institutions) |
| Create student IDs | ✅ (own institutions) |
| Enroll students into courses | ✅ (own courses) |
| Create courses | ✅ (own institutions) |
| Upload study material (files, YouTube links) | ✅ (own courses) |
| Create MCQ exams | ✅ (own courses) |
| View student exam results | ✅ (own courses) |
| View student roster | ✅ (own institutions) |

A Teacher can belong to multiple institutions — they own some, and are invited members of others.

### 2.2 Student

| Action | Permission |
|--------|:----------:|
| Browse available courses | ✅ (across institutions they belong to) |
| Self-enroll in courses | ✅ (open courses) |
| View enrolled course content | ✅ |
| Take MCQ exams | ✅ |
| View own exam results | ✅ |
| View own grades | ✅ |

A Student can belong to multiple institutions.

---

## 3. User Journeys

### 3.1 Teacher Creates Institution
1. Teacher signs up / logs in via Firebase Auth
2. Teacher clicks "Create Institution" → enters name, timezone
3. Institution is created → Teacher is the owner
4. Teacher lands on institution dashboard

### 3.2 Teacher Invites Another Teacher
1. Teacher A (owner of Institution X) navigates to "Members" tab
2. Enters email of Teacher B
3. Teacher B receives email → logs in → sees pending invitation
4. Teacher B accepts → now a Teacher at Institution X
5. Teacher B can create courses at Institution X

### 3.3 Teacher Requests to Join Institution
1. Teacher logs in → browses/discover institutions
2. Teacher clicks "Request to Join" on Institution Y
3. Institution Y owner receives notification
4. Owner accepts/declines the request
5. If accepted → Teacher becomes member of Institution Y

### 3.4 Teacher Creates a Student
1. Teacher navigates to "Students" tab in institution
2. Creates student: name, email, password (or email invite)
3. Student receives credentials / magic link
4. Student logs in → automatically belongs to that institution

### 3.5 Student Self-Enrolls
1. Student logs in → browses available courses
2. Clicks "Enroll" on a course
3. Student is now enrolled → course appears on dashboard

### 3.6 Teacher Creates a Course
1. Teacher selects institution → clicks "New Course"
2. Enters: title, description, cover image (optional)
3. Course created in **Draft** status
4. Teacher adds content:
   - **Study materials**: Upload files (PDF, images, docs) + add YouTube video links
   - **MCQ Exams**: Create questions with 4 options, mark correct answer, set passing score
5. Teacher sets course to **Published** → visible to students

### 3.7 Student Takes a Course
1. Student opens enrolled course → sees study materials
2. Views/downloads files, watches YouTube videos
3. When ready → clicks "Take Exam"
4. Answers all MCQ questions → submits
5. Results shown immediately (auto-graded)
6. Teacher can view all student results

---

## 4. Data Model

### 4.1 Entity Relationship

```
User ──┬── InstitutionMembership ──┬── Institution
       │   (userId, institutionId, │
       │    role: teacher|student)  │
       │                           │
       ├── Course (created by Teacher)
       │   ├── StudyMaterial (files + YouTube links)
       │   └── Exam
       │       └── Question (MCQ with 4 options)
       │
       └── CourseEnrollment (student ↔ course)
               └── ExamAttempt (student takes exam)
                       └── ExamResponse (per-question answer)
```

### 4.2 Database Schema

```sql
-- ============================================
-- USERS (global identity, Firebase Auth UID)
-- ============================================
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  firebase_uid TEXT UNIQUE NOT NULL,    -- Firebase Auth UID
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- ============================================
-- INSTITUTIONS
-- ============================================
CREATE TABLE institutions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  timezone TEXT DEFAULT 'UTC',
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- ============================================
-- INSTITUTION MEMBERSHIP (User <-> Institution with role)
-- ============================================
CREATE TABLE institution_memberships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  institution_id UUID NOT NULL REFERENCES institutions(id) ON DELETE CASCADE,
  role TEXT NOT NULL CHECK (role IN ('teacher', 'student')),
  is_owner BOOLEAN DEFAULT false,       -- True for institution creator
  status TEXT DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'removed')),
  joined_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, institution_id)
);

-- ============================================
-- TEACHER INVITATIONS / JOIN REQUESTS
-- ============================================
CREATE TABLE teacher_invitations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  institution_id UUID NOT NULL REFERENCES institutions(id) ON DELETE CASCADE,
  email TEXT NOT NULL,                    -- Email of teacher being invited
  invited_by UUID NOT NULL REFERENCES users(id),
  type TEXT NOT NULL CHECK (type IN ('invitation', 'join_request')),
  -- 'invitation': institution invites teacher
  -- 'join_request': teacher requests to join
  requested_by UUID REFERENCES users(id), -- For join_request: who requested
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'declined', 'expired')),
  created_at TIMESTAMPTZ DEFAULT now(),
  resolved_at TIMESTAMPTZ
);

-- ============================================
-- COURSES
-- ============================================
CREATE TABLE courses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  institution_id UUID NOT NULL REFERENCES institutions(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  cover_image_url TEXT,
  status TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'published', 'archived')),
  enrollment_type TEXT DEFAULT 'open' CHECK (enrollment_type IN ('open', 'invite_only')),
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- ============================================
-- COURSE ENROLLMENT (Student <-> Course)
-- ============================================
CREATE TABLE course_enrollments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  student_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  enrolled_at TIMESTAMPTZ DEFAULT now(),
  completed_at TIMESTAMPTZ,
  UNIQUE(course_id, student_id)
);

-- ============================================
-- STUDY MATERIALS (files + YouTube videos)
-- ============================================
CREATE TABLE study_materials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  type TEXT NOT NULL CHECK (type IN ('file', 'youtube')),
  title TEXT NOT NULL,
  description TEXT,
  
  -- File fields
  file_url TEXT,
  file_name TEXT,
  file_size_bytes BIGINT,
  file_type TEXT,
  
  -- YouTube fields
  youtube_url TEXT,
  youtube_thumbnail_url TEXT,
  
  order_index INTEGER NOT NULL DEFAULT 0,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- ============================================
-- EXAMS
-- ============================================
CREATE TABLE exams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  duration_minutes INTEGER NOT NULL DEFAULT 30,  -- 0 = untimed
  passing_score INTEGER NOT NULL DEFAULT 50,       -- Percentage to pass
  max_attempts INTEGER DEFAULT 3,
  shuffle_questions BOOLEAN DEFAULT false,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- ============================================
-- EXAM QUESTIONS (MCQ only for MVP)
-- ============================================
CREATE TABLE exam_questions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  exam_id UUID NOT NULL REFERENCES exams(id) ON DELETE CASCADE,
  question_text TEXT NOT NULL,
  option_a TEXT NOT NULL,
  option_b TEXT NOT NULL,
  option_c TEXT NOT NULL,
  option_d TEXT NOT NULL,
  correct_option CHAR(1) NOT NULL CHECK (correct_option IN ('A', 'B', 'C', 'D')),
  points INTEGER DEFAULT 1,
  order_index INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- ============================================
-- EXAM ATTEMPTS
-- ============================================
CREATE TABLE exam_attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  exam_id UUID NOT NULL REFERENCES exams(id) ON DELETE CASCADE,
  student_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  started_at TIMESTAMPTZ DEFAULT now(),
  submitted_at TIMESTAMPTZ,
  time_spent_seconds INTEGER,
  total_score INTEGER,
  total_points INTEGER,
  passed BOOLEAN,
  status TEXT DEFAULT 'in_progress' CHECK (status IN ('in_progress', 'submitted', 'timed_out')),
  attempt_number INTEGER DEFAULT 1,
  UNIQUE(exam_id, student_id, attempt_number)
);

-- ============================================
-- EXAM RESPONSES (one per question per attempt)
-- ============================================
CREATE TABLE exam_responses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  attempt_id UUID NOT NULL REFERENCES exam_attempts(id) ON DELETE CASCADE,
  question_id UUID NOT NULL REFERENCES exam_questions(id) ON DELETE CASCADE,
  selected_option CHAR(1) CHECK (selected_option IN ('A', 'B', 'C', 'D')),
  is_correct BOOLEAN,
  score INTEGER,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- ============================================
-- INDEXES
-- ============================================
CREATE INDEX idx_membership_user ON institution_memberships(user_id);
CREATE INDEX idx_membership_institution ON institution_memberships(institution_id);
CREATE INDEX idx_course_institution ON courses(institution_id, status);
CREATE INDEX idx_enrollment_student ON course_enrollments(student_id);
CREATE INDEX idx_enrollment_course ON course_enrollments(course_id);
CREATE INDEX idx_study_material_course ON study_materials(course_id, order_index);
CREATE INDEX idx_exam_course ON exams(course_id);
CREATE INDEX idx_exam_question_exam ON exam_questions(exam_id, order_index);
CREATE INDEX idx_exam_attempt_student ON exam_attempts(student_id);
```

---

## 5. API Routes

### 5.1 Auth (handled by Firebase Auth on client side)

```
POST   /api/auth/register         # Create user record in our DB after Firebase signup
GET    /api/auth/me                # Get current user profile
PATCH  /api/auth/me                # Update profile
```

### 5.2 Institutions

```
GET    /api/institutions                         # List my institutions
POST   /api/institutions                         # Create institution
GET    /api/institutions/:id                      # Get institution details
PATCH  /api/institutions/:id                      # Update (owner only)
GET    /api/institutions/:id/members              # List members
POST   /api/institutions/:id/invite              # Invite teacher to institution
```

### 5.3 Teacher Invitations / Join Requests

```
GET    /api/invitations                           # My pending invitations/requests
POST   /api/institutions/:id/join-request         # Teacher requests to join
PATCH  /api/invitations/:id/accept               # Accept invitation/request
PATCH  /api/invitations/:id/decline              # Decline invitation/request
```

### 5.4 Students

```
POST   /api/institutions/:id/students             # Teacher creates student
GET    /api/institutions/:id/students             # List all students
DELETE /api/institutions/:id/students/:studentId  # Remove student
```

### 5.5 Courses

```
GET    /api/institutions/:id/courses              # List courses (teacher: all, student: published)
POST   /api/institutions/:id/courses              # Create course (teacher)
GET    /api/courses/:id                            # Get course details
PATCH  /api/courses/:id                            # Update course (teacher)
DELETE /api/courses/:id                            # Archive course (teacher)
POST   /api/courses/:id/publish                   # Publish course (teacher)
```

### 5.6 Study Materials

```
GET    /api/courses/:id/materials                 # List materials
POST   /api/courses/:id/materials                 # Add material (teacher)
PATCH  /api/materials/:id                          # Update material (teacher)
DELETE /api/materials/:id                          # Delete material (teacher)
POST   /api/upload/presigned-url                  # Get presigned URL for file upload
```

### 5.7 Exams

```
GET    /api/courses/:id/exams                     # List exams
POST   /api/courses/:id/exams                     # Create exam (teacher)
GET    /api/exams/:id                              # Get exam (teacher: full, student: no answers)
PATCH  /api/exams/:id                              # Update exam (teacher)
DELETE /api/exams/:id                              # Delete exam (teacher)

# Questions
POST   /api/exams/:id/questions                   # Add question (teacher)
PATCH  /api/questions/:id                          # Update question (teacher)
DELETE /api/questions/:id                          # Delete question (teacher)

# Taking exams
POST   /api/exams/:id/start                       # Student starts exam → returns attempt
POST   /api/exams/:id/submit                      # Student submits exam → auto-grade
GET    /api/exams/:id/results/:attemptId          # View results
GET    /api/exams/:id/attempts                     # Teacher: all student attempts
```

### 5.8 Enrollments

```
POST   /api/courses/:id/enroll                    # Student self-enrolls
POST   /api/courses/:id/enroll/:studentId         # Teacher enrolls a student
DELETE /api/courses/:id/enroll/:studentId         # Unenroll student
GET    /api/courses/:id/students                  # List enrolled students
```

### 5.9 Dashboard

```
GET    /api/dashboard/student                     # Student dashboard
GET    /api/dashboard/teacher                     # Teacher dashboard
GET    /api/institutions/:id/dashboard            # Institution overview (teacher/owner)
```

---

## 6. Frontend Pages

### 6.1 Auth Pages
- `/login` — Firebase Auth sign-in (email/password, Google)
- `/register` — Sign-up + role selection (Teacher or Student)
- `/onboarding` — Post-registration: teacher creates first institution

### 6.2 Teacher Pages
- `/dashboard` — My institutions, courses, recent activity
- `/institutions/:id` — Institution dashboard (members, courses, students)
- `/institutions/:id/members` — Manage teachers, pending invitations
- `/institutions/:id/students` — Student roster, create student
- `/institutions/:id/courses/new` — Create course
- `/courses/:id` — Course detail (materials, exam, students)
- `/courses/:id/edit` — Edit course content
- `/courses/:id/exam/:examId/results` — View all student exam results

### 6.3 Student Pages
- `/dashboard` — My institutions, enrolled courses, progress
- `/courses/:id` — Course content (materials + exam)
- `/courses/:id/exam/:examId` — Take exam
- `/courses/:id/exam/:examId/results` — View my exam results
- `/courses/browse` — Browse available courses to enroll

### 6.4 Shared
- `/settings` — Profile, notification preferences
- `/invitations` — Pending teacher invitations / join requests

---

## 7. Course Structure (Flat, No Modules)

In the MVP, a course is a flat list of content:

```
Course: "Introduction to Python"
├── 📄 Study Material: "Python Basics" (PDF file)
├── 📄 Study Material: "Variables and Types" (PDF file)
├── 🎥 Study Material: "Python Setup Guide" (YouTube video)
├── 🎥 Study Material: "Your First Program" (YouTube video)
└── 📝 Exam: "Python Basics Quiz" (10 MCQ questions, 30 min, 60% to pass)
```

No modules, no sections, no drip content — just an ordered list of materials followed by an exam.

---

## 8. Exam Flow

### 8.1 Teacher Creates Exam
1. Teacher opens course → clicks "Create Exam"
2. Enters: title, description, duration, passing score, max attempts
3. Adds MCQ questions one by one:
   - Question text
   - Option A, B, C, D
   - Mark correct option (A/B/C/D)
   - Points per question (default: 1)
4. Can reorder questions, edit, delete
5. Exam is automatically available when course is published

### 8.2 Student Takes Exam
1. Student opens course → sees exam listed
2. Clicks "Start Exam" → timer begins (if timed)
3. Questions appear one at a time or all at once
4. Student selects answer for each question
5. Clicks "Submit" or timer expires (auto-submit)
6. **Instant results**: Score, pass/fail, per-question correct/incorrect shown

### 8.3 Auto-Grading
- Each correct answer = question's point value
- Total score = sum of correct answers
- Pass = `(total_score / total_points) * 100 >= passing_score`
- No partial credit, no manual grading needed

---

## 9. Student ID Creation Flow

Teachers can create student accounts directly:

1. Teacher navigates to institution → "Students" tab → "Create Student"
2. Enters: name, email
3. System creates Firebase Auth account (or uses custom token)
4. Student receives email with login instructions
5. Student is automatically added to the institution as a Student

---

## 10. Teacher Joining Institution — Two Directions

### 10.1 Institution Invites Teacher
```
Institution Owner → "Members" → "Invite Teacher" → Enter email
→ Teacher gets email → Logs in → Sees invitation → Accepts
→ Teacher is now a member
```

### 10.2 Teacher Requests to Join
```
Teacher → "Join Institution" → Enters institution name/code
→ Request sent → Institution Owner gets notification
→ Owner accepts → Teacher is now a member
```

---

## 11. Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | Next.js 14 (App Router) + React 18 + TypeScript |
| **Styling** | Tailwind CSS + shadcn/ui components |
| **Auth** | Firebase Authentication (email/password + Google) |
| **Backend API** | Next.js API Routes |
| **Database** | PostgreSQL + Drizzle ORM |
| **File Storage** | Firebase Storage or AWS S3 (presigned URLs) |
| **Cache** | Redis (optional for MVP — skip if complexity too high) |
| **Email** | Firebase Auth built-in email + Resend for custom emails |
| **Forms** | react-hook-form + zod validation |
| **Data Fetching** | TanStack React Query |
| **Monorepo** | Optional — single Next.js app for MVP simplicity |

---

## 12. Implementation Order (8 Weeks)

### Week 1: Foundation
- [ ] Next.js project setup with TypeScript + Tailwind + shadcn/ui
- [ ] Firebase Auth integration (signup, login, logout)
- [ ] User registration flow (create user record in PostgreSQL after Firebase signup)
- [ ] Drizzle ORM setup + database migrations
- [ ] Core middleware (auth check, institution context)

### Week 2: Institutions & Membership
- [ ] Institution CRUD (create, edit, view)
- [ ] Institution membership (add/remove teachers & students)
- [ ] Teacher invitation flow (invite + accept)
- [ ] Teacher join request flow (request + accept/decline)
- [ ] Student ID creation by teacher

### Week 3: Courses
- [ ] Course CRUD (create, edit, publish, archive)
- [ ] Course listing (teacher: all, student: published)
- [ ] Course detail page with content display
- [ ] Enrollment flow (self-enroll + teacher-enroll)

### Week 4: Study Materials
- [ ] File upload (presigned URLs → S3/Firebase Storage)
- [ ] YouTube video embed (react-player)
- [ ] Material reordering (drag-and-drop or numeric)
- [ ] Student view: download files, watch videos

### Week 5: Exams — Teacher Side
- [ ] Exam CRUD
- [ ] MCQ question builder UI
- [ ] Question management (add, edit, reorder, delete)
- [ ] Exam settings (duration, passing score, max attempts)

### Week 6: Exams — Student Side
- [ ] Exam taking UI (question display, option selection, timer)
- [ ] Auto-grading engine (server-side)
- [ ] Exam submission + instant results
- [ ] Result display (score, pass/fail, per-question feedback)

### Week 7: Dashboards
- [ ] Student dashboard (enrolled courses, exam results, progress)
- [ ] Teacher dashboard (my courses, recent submissions, quick stats)
- [ ] Institution dashboard (members, courses, students overview)

### Week 8: Polish & Launch
- [ ] Error handling & loading states
- [ ] Responsive design pass
- [ ] Email notifications (basic: invitation, enrollment)
- [ ] Deployment setup (Vercel)
- [ ] Testing & bug fixes

---

## 13. MVP Success Criteria

- [ ] A user can sign up as Teacher or Student via Firebase Auth
- [ ] A Teacher can create an institution
- [ ] A Teacher can invite another Teacher to their institution (and vice versa via join request)
- [ ] A Teacher can create student accounts
- [ ] A Teacher can create a course with file-based study materials and YouTube videos
- [ ] A Teacher can create an MCQ exam, set a passing score, and publish it
- [ ] A Student can browse courses, self-enroll, and access study materials
- [ ] A Student can take an MCQ exam and see instant results
- [ ] A Teacher can view all student exam results for their courses
- [ ] A Student can view their own exam results and course progress
- [ ] Multi-institution: Teacher/Student can belong to multiple institutions with no data leakage

---

## 14. What's NOT in MVP

- ❌ Modules / sections within courses (flat list of materials only)
- ❌ Homework assignments (only exams)
- ❌ Short answer / long answer questions (MCQ only)
- ❌ Manual grading (100% auto-graded)
- ❌ Question banks / pools
- ❌ Drip content / prerequisites
- ❌ Course completion certificates
- ❌ In-app notification center (email only for MVP)
- ❌ Analytics / reporting dashboards
- ❌ Billing / subscriptions
- ❌ PWA / mobile app
- ❌ Custom branding
- ❌ Institution Admin / Owner as separate roles (Owner = Teacher)
- ❌ Announcements
- ❌ Discussion forums

---

*End of MVP Plan. This is the blueprint for the 8-week build. Keep it simple.*
