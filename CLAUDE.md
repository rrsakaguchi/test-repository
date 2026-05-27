## Prisma Schema

```prisma
// =====================================================================
//  TaskBoard — Prisma Schema
//  PostgreSQL 16 / timezone='Asia/Tokyo' / TIMESTAMPTZ
//  Single source of DB schema. BE/FE の型は packages/shared (Zod) で共有。
// =====================================================================

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ---------------------------------------------------------------------
//  Enums
// ---------------------------------------------------------------------

enum UserRole {
  ADMIN
  MEMBER
}

enum UserStatus {
  ACTIVE
  INACTIVE
}

enum TaskStatus {
  TODO
  IN_PROGRESS
  DONE
}

enum TaskAssignmentStatus {
  UNASSIGNED
  REQUESTED
  ACCEPTED
}

enum NotificationType {
  TASK_REQUESTED
  TASK_ACCEPTED
  TASK_DECLINED
  TASK_REQUEST_CANCELLED
  PROJECT_MEMBER_ADDED
}

enum NotificationChannel {
  EMAIL
  IN_APP
}

enum NotificationStatus {
  PENDING
  SENT
  FAILED
}

// ---------------------------------------------------------------------
//  User
// ---------------------------------------------------------------------

model User {
  id           String     @id @default(cuid())
  email        String     @unique
  name         String
  passwordHash String     @map("password_hash")
  role         UserRole   @default(MEMBER)
  status       UserStatus @default(ACTIVE)
  avatarUrl    String?    @map("avatar_url")
  deletedAt    DateTime?  @map("deleted_at") @db.Timestamptz(3)
  createdAt    DateTime   @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt    DateTime   @updatedAt @map("updated_at") @db.Timestamptz(3)

  // Relations
  projectMembers       ProjectMember[]
  assignedTasks        Task[]          @relation("TaskAssignee")
  requestedToTasks     Task[]          @relation("TaskRequestedTo")
  requestedByTasks     Task[]          @relation("TaskRequestedBy")
  comments             Comment[]
  invitationsSent      Invitation[]    @relation("InvitedBy")
  notificationsReceived Notification[] @relation("NotificationRecipient")
  notificationsActed   Notification[]  @relation("NotificationActor")

  @@index([status])
  @@index([role])
  @@index([deletedAt])
  @@map("users")
}

// ---------------------------------------------------------------------
//  Invitation
// ---------------------------------------------------------------------

model Invitation {
  id           String    @id @default(cuid())
  email        String
  role         UserRole
  tokenHash    String    @unique @map("token_hash")
  invitedById  String    @map("invited_by_id")
  expiresAt    DateTime  @map("expires_at") @db.Timestamptz(3)
  acceptedAt   DateTime? @map("accepted_at") @db.Timestamptz(3)
  createdAt    DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt    DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)

  invitedBy User @relation("InvitedBy", fields: [invitedById], references: [id], onDelete: Restrict)

  @@index([email])
  @@index([acceptedAt])
  @@index([expiresAt])
  @@map("invitations")
}

// ---------------------------------------------------------------------
//  Project
// ---------------------------------------------------------------------

model Project {
  id          String    @id @default(cuid())
  name        String
  description String?
  archivedAt  DateTime? @map("archived_at") @db.Timestamptz(3)
  createdAt   DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt   DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)

  members       ProjectMember[]
  tasks         Task[]
  notifications Notification[]

  @@index([archivedAt])
  @@map("projects")
}

// ---------------------------------------------------------------------
//  ProjectMember (join table, composite PK)
// ---------------------------------------------------------------------

model ProjectMember {
  projectId String   @map("project_id")
  userId    String   @map("user_id")
  joinedAt  DateTime @default(now()) @map("joined_at") @db.Timestamptz(3)

  project Project @relation(fields: [projectId], references: [id], onDelete: Cascade)
  user    User    @relation(fields: [userId], references: [id], onDelete: Restrict)

  @@id([projectId, userId])
  @@index([userId])
  @@map("project_members")
}

// ---------------------------------------------------------------------
//  Task
//
//  Invariants (Service layer enforced, optional DB CHECK constraint):
//    UNASSIGNED : assignee_id IS NULL AND requested_to_id IS NULL AND requested_by_id IS NULL
//    REQUESTED  : assignee_id IS NULL AND requested_to_id IS NOT NULL AND requested_by_id IS NOT NULL
//    ACCEPTED   : assignee_id IS NOT NULL AND requested_to_id IS NULL AND requested_by_id IS NULL
//
//  position: 0-based integer, unique within (project_id, status) by convention only
//            (no DB unique constraint; last-writer-wins for R-05 responsiveness).
// ---------------------------------------------------------------------

model Task {
  id               String               @id @default(cuid())
  projectId        String               @map("project_id")
  title            String
  description      String?
  status           TaskStatus           @default(TODO)
  assignmentStatus TaskAssignmentStatus @default(UNASSIGNED) @map("assignment_status")
  assigneeId       String?              @map("assignee_id")
  requestedToId    String?              @map("requested_to_id")
  requestedById    String?              @map("requested_by_id")
  requestMessage   String?              @map("request_message")
  dueDate          DateTime?            @map("due_date") @db.Date
  position         Int                  @default(0)
  createdAt        DateTime             @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt        DateTime             @updatedAt @map("updated_at") @db.Timestamptz(3)

  project       Project        @relation(fields: [projectId], references: [id], onDelete: Cascade)
  assignee      User?          @relation("TaskAssignee", fields: [assigneeId], references: [id], onDelete: SetNull)
  requestedTo   User?          @relation("TaskRequestedTo", fields: [requestedToId], references: [id], onDelete: SetNull)
  requestedBy   User?          @relation("TaskRequestedBy", fields: [requestedById], references: [id], onDelete: SetNull)
  comments      Comment[]
  notifications Notification[]

  @@index([projectId, status, position])
  @@index([assigneeId, status])
  @@index([requestedToId])
  @@index([dueDate])
  @@map("tasks")
}

// ---------------------------------------------------------------------
//  Comment (logical delete via deletedAt; body replaced with "" in API)
// ---------------------------------------------------------------------

model Comment {
  id        String    @id @default(cuid())
  taskId    String    @map("task_id")
  authorId  String    @map("author_id")
  body      String
  isEdited  Boolean   @default(false) @map("is_edited")
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz(3)
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)

  task   Task @relation(fields: [taskId], references: [id], onDelete: Cascade)
  author User @relation(fields: [authorId], references: [id], onDelete: Restrict)

  @@index([taskId, createdAt])
  @@index([authorId])
  @@map("comments")
}

// ---------------------------------------------------------------------
//  Notification (history + Outbox for pg-boss email jobs)
// ---------------------------------------------------------------------

model Notification {
  id           String              @id @default(cuid())
  recipientId  String              @map("recipient_id")
  type         NotificationType
  channel      NotificationChannel @default(EMAIL)
  status       NotificationStatus  @default(PENDING)
  payload      Json                @default("{}") @db.JsonB
  taskId       String?             @map("task_id")
  projectId    String?             @map("project_id")
  actorId      String?             @map("actor_id")
  readAt       DateTime?           @map("read_at") @db.Timestamptz(3)
  sentAt       DateTime?           @map("sent_at") @db.Timestamptz(3)
  failedAt     DateTime?           @map("failed_at") @db.Timestamptz(3)
  errorMessage String?             @map("error_message")
  jobId        String?             @map("job_id")
  createdAt    DateTime            @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt    DateTime            @updatedAt @map("updated_at") @db.Timestamptz(3)

  recipient User     @relation("NotificationRecipient", fields: [recipientId], references: [id], onDelete: Cascade)
  actor     User?    @relation("NotificationActor", fields: [actorId], references: [id], onDelete: SetNull)
  task      Task?    @relation(fields: [taskId], references: [id], onDelete: Cascade)
  project   Project? @relation(fields: [projectId], references: [id], onDelete: SetNull)

  @@index([recipientId, createdAt])
  @@index([status, createdAt])
  @@map("notifications")
}
```
