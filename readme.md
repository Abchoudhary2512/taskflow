# TaskFlow 

A production-oriented **full-stack project management SaaS** built to learn modern full-stack development concepts including authentication, authorization, database design, caching, background jobs, emails, file storage, rate limiting, monitoring, and deployment.

Think of it as a simplified **Linear / Jira / Trello**.

---

##  Features

*  Email + Google authentication
*  Create and manage workspaces
*  Invite workspace members
*  Role-based access control (Owner / Admin / Member)
*  Create and manage projects
*  Task management
*  Assign tasks to members
*  Task status and priority
*  Task comments
*  File attachments
*  Search, filtering and pagination
*  Workspace dashboard
*  Activity logs
*  Redis caching
*  API rate limiting
*  Email notifications
*  Deadline reminders
*  Background jobs
*  Error monitoring
*  Production deployment

---

#  Tech Stack

| Layer           | Technology               |
| --------------- | ------------------------ |
| Framework       | Next.js + TypeScript     |
| UI              | Tailwind CSS + shadcn/ui |
| Database        | Supabase PostgreSQL      |
| ORM             | Drizzle ORM              |
| Authentication  | Supabase Auth            |
| Validation      | Zod                      |
| Cache           | Redis / Upstash          |
| Rate Limiting   | Upstash Redis            |
| Background Jobs | Inngest                  |
| Email           | Resend                   |
| Storage         | Supabase Storage         |
| Monitoring      | Sentry                   |
| Deployment      | Vercel                   |

---

#  Architecture

```text
                         User
                           │
                           ▼
                    ┌─────────────┐
                    │   Next.js   │
                    │ React / RSC │
                    └──────┬──────┘
                           │
                Server Actions / APIs
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    Supabase Auth     PostgreSQL          Redis
                           │
                      Drizzle ORM
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
              Inngest              Resend
          Background Jobs           Email
```

---

#  Complete Sequence Diagram

The following represents the lifecycle of a typical authenticated task creation request.

```mermaid
sequenceDiagram
    autonumber

    actor User

    participant UI as Next.js UI
    participant Server as Next.js Server
    participant Auth as Supabase Auth
    participant Redis as Redis / Upstash
    participant DB as PostgreSQL
    participant Inngest as Inngest
    participant Resend as Resend
    participant Sentry as Sentry

    User->>UI: Create Task

    UI->>Server: Server Action / API Request

    Server->>Auth: Validate Session
    Auth-->>Server: User + Session

    Server->>DB: Check Workspace Membership
    DB-->>Server: Role / Permissions

    Server->>Redis: Check Rate Limit
    Redis-->>Server: Allowed

    Server->>Server: Validate Input with Zod

    Server->>DB: INSERT Task
    DB-->>Server: Task Created

    Server->>Redis: Invalidate Project Cache

    Server->>Inngest: Emit task/created Event

    Server-->>UI: Return Task
    UI-->>User: Show Created Task

    Inngest->>DB: Fetch Assignee Details
    DB-->>Inngest: User Information

    Inngest->>Resend: Send Assignment Email
    Resend-->>Inngest: Email Sent

    alt Application Error
        Server->>Sentry: Capture Exception
    end
```

This teaches one of the most important full-stack concepts:

```text
Request Path
──────────────────────────────

User
 ↓
Frontend
 ↓
Server
 ↓
Authentication
 ↓
Authorization
 ↓
Rate Limiting
 ↓
Validation
 ↓
Database
 ↓
Cache Invalidation
 ↓
Background Event
 ↓
Response


Async Path
──────────────────────────────

Inngest
 ↓
Database
 ↓
Resend
 ↓
Email
```

---

#  Authentication Sequence

```mermaid
sequenceDiagram
    actor User

    participant UI as Next.js
    participant Auth as Supabase Auth
    participant DB as PostgreSQL

    User->>UI: Enter Email + Password

    UI->>Auth: Sign In

    Auth->>DB: Validate User

    DB-->>Auth: User Found

    Auth-->>UI: Session + Cookie

    UI-->>User: Redirect to Dashboard
```

Authentication determines:

> **Who is the user?**

Authorization determines:

> **What can the user do?**

---

#  Authorization Sequence

```mermaid
sequenceDiagram
    actor User

    participant Server as Next.js Server
    participant Auth as Supabase Auth
    participant DB as PostgreSQL

    User->>Server: Delete Project

    Server->>Auth: Get Current User
    Auth-->>Server: userId

    Server->>DB: Find Workspace Membership
    DB-->>Server: ADMIN

    Server->>Server: Check Permission

    alt Authorized
        Server->>DB: DELETE Project
        DB-->>Server: Success
        Server-->>User: Project Deleted
    else Unauthorized
        Server-->>User: 403 Forbidden
    end
```

Workspace roles:

| Action           | Owner | Admin | Member |
| ---------------- | ----: | ----: | -----: |
| View workspace   |     ✅ |     ✅ |      ✅ |
| Create project   |     ✅ |     ✅ |      ❌ |
| Create task      |     ✅ |     ✅ |      ✅ |
| Invite members   |     ✅ |     ✅ |      ❌ |
| Remove members   |     ✅ |     ✅ |      ❌ |
| Delete workspace |     ✅ |     ❌ |      ❌ |

Authorization must always be enforced on the **server**.

---

#  Redis Cache Sequence

```mermaid
sequenceDiagram
    actor User

    participant Server as Next.js
    participant Redis as Redis
    participant DB as PostgreSQL

    User->>Server: Open Dashboard

    Server->>Redis: GET workspace:123:dashboard

    alt Cache Hit
        Redis-->>Server: Cached Data
        Server-->>User: Dashboard
    else Cache Miss
        Redis-->>Server: null

        Server->>DB: Query Dashboard Data
        DB-->>Server: Dashboard Data

        Server->>Redis: SET Cached Data + TTL

        Server-->>User: Dashboard
    end
```

Concepts learned:

```text
Cache Hit
Cache Miss
TTL
Cache Invalidation
Cache Keys
Performance Optimization
```

---

#  Rate Limiting Sequence

```mermaid
sequenceDiagram
    actor User

    participant API as Next.js API
    participant Redis as Redis

    User->>API: POST /comments

    API->>Redis: Increment Request Counter

    Redis-->>API: Request Count

    alt Under Limit
        API-->>User: Request Accepted
    else Limit Exceeded
        API-->>User: 429 Too Many Requests
    end
```

Example policy:

```text
POST /comments

20 requests
per user
per minute
```

---

#  Background Job Sequence

Email delivery should not unnecessarily increase request latency.

```mermaid
sequenceDiagram
    actor User

    participant Server as Next.js
    participant DB as PostgreSQL
    participant Inngest as Inngest
    participant Resend as Resend

    User->>Server: Assign Task

    Server->>DB: Update Task
    DB-->>Server: Updated

    Server->>Inngest: Emit task/assigned

    Server-->>User: Task Assigned

    Note over Inngest,Resend: Runs asynchronously

    Inngest->>DB: Get Assignee Information
    DB-->>Inngest: Email + Task

    Inngest->>Resend: Send Email

    alt Email Failed
        Resend-->>Inngest: Error
        Inngest->>Inngest: Retry Job
        Inngest->>Resend: Retry Email
    else Email Sent
        Resend-->>Inngest: Success
    end
```

This introduces:

```text
Event-driven architecture
Background processing
Retries
Failure handling
Idempotency
Async workflows
```

---

# File Upload Sequence

```mermaid
sequenceDiagram
    actor User

    participant UI as Next.js UI
    participant Server as Next.js Server
    participant Storage as Supabase Storage
    participant DB as PostgreSQL

    User->>UI: Select Attachment

    UI->>Server: Upload Request

    Server->>Server: Validate File

    Server->>Storage: Upload File
    Storage-->>Server: Storage Path

    Server->>DB: Save Attachment Metadata
    DB-->>Server: Attachment Created

    Server-->>UI: Upload Complete
    UI-->>User: Display Attachment
```

The actual file goes to **object storage** while PostgreSQL stores metadata such as:

```text
id
task_id
file_name
storage_path
file_size
mime_type
uploaded_by
created_at
```

---

#  Database Design

Main tables:

```text
users
workspaces
workspace_members
projects
tasks
comments
attachments
invitations
activity_logs
```

Relationships:

```text
User
 │
 └── WorkspaceMember
          │
          ▼
      Workspace
          │
          ├── Projects
          │     │
          │     └── Tasks
          │           ├── Comments
          │           └── Attachments
          │
          ├── Invitations
          │
          └── ActivityLogs
```

---

#  Project Structure

```text
taskflow/
│
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   ├── signup/
│   │   └── forgot-password/
│   │
│   ├── dashboard/
│   │
│   ├── workspace/
│   │   └── [workspaceId]/
│   │       ├── projects/
│   │       ├── members/
│   │       └── settings/
│   │
│   └── api/
│       ├── inngest/
│       ├── upload/
│       └── health/
│
├── components/
├── actions/
│
├── db/
│   ├── index.ts
│   ├── schema/
│   └── migrations/
│
├── lib/
│   ├── supabase/
│   ├── redis.ts
│   ├── resend.ts
│   ├── auth.ts
│   ├── permissions.ts
│   └── rate-limit.ts
│
├── inngest/
│   ├── functions/
│   └── client.ts
│
├── emails/
├── validators/
├── types/
│
├── drizzle.config.ts
├── .env.local
├── package.json
└── README.md
```

---

#  Development Roadmap

```text
Phase 1
Next.js + TypeScript + Tailwind + shadcn
                ↓
Phase 2
Supabase PostgreSQL + Drizzle
                ↓
Phase 3
Supabase Authentication
                ↓
Phase 4
Workspace / Project / Task CRUD
                ↓
Phase 5
RBAC + Authorization
                ↓
Phase 6
Redis Cache + Rate Limiting
                ↓
Phase 7
Resend Emails
                ↓
Phase 8
Inngest Background Jobs
                ↓
Phase 9
Supabase Storage
                ↓
Phase 10
Sentry + Security + Optimization
                ↓
             Vercel
```

---

#  Full Application Lifecycle

By the end of the project, you should understand this complete flow:

```text
                    USER
                      │
                      ▼
                  React UI
                      │
                      ▼
                  Next.js
                      │
            ┌─────────┴─────────┐
            ▼                   ▼
      Authentication        Rate Limit
            │                   │
            └─────────┬─────────┘
                      ▼
                Authorization
                      │
                      ▼
                  Validation
                      │
                      ▼
                Business Logic
                      │
            ┌─────────┴─────────┐
            ▼                   ▼
        PostgreSQL             Redis
            │                   │
            └─────────┬─────────┘
                      ▼
                 Emit Event
                      │
                      ▼
                   Inngest
                      │
              ┌───────┴───────┐
              ▼               ▼
           Resend          Scheduled
           Email             Jobs

                 + Sentry
              Observability
```

The goal is not just to know **Next.js**.

The goal is to understand what happens from the moment a user clicks a button until the request travels through **frontend → server → authentication → authorization → cache → database → background infrastructure → external services → production monitoring**.
