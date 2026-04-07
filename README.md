# SwitchesTasks — Portfolio Showcase

> A task management platform where Git activity drives task lifecycle automatically.
> Push a branch → task moves to `IN_PROGRESS`. Merge a PR → `COMPLETED`. No manual updates needed.

**Private source repos** · **Full microservices architecture** · **TypeScript end-to-end**

---

## What is SwitchesTasks?

SwitchesTasks is a Kanban-style project management tool built for software teams. The core innovation is that task state transitions are triggered automatically by GitHub events — developers never need to manually update task status.

It was built as a full production-grade system: 7 independent microservices, each with its own database, communicating over HTTP, orchestrated with Docker Compose.

---

## Architecture

```
                        ┌─────────────────┐
                        │    Frontend      │
                        │  React + Zustand │
                        └────────┬────────┘
                                 │
                        ┌────────▼────────┐
                        │   API Gateway   │  :3000
                        │ Express + Redis │
                        │  JWT · Rate limit│
                        └────────┬────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                       │
 ┌────────▼───────┐   ┌──────────▼──────┐   ┌──────────▼──────┐
 │  Auth Service  │   │ User-Team Svc   │   │  Task Service   │
 │  :3001         │   │  :3002          │   │  :3003          │
 │  JWT · bcrypt  │   │  Roles · Teams  │   │  State machine  │
 └────────────────┘   └─────────────────┘   └─────────────────┘
                                                      ▲
 ┌──────────────────────────┐             ┌───────────┴─────────┐
 │ GitHub Integration Svc   │────────────▶│ Notification Service │
 │  :3004                   │             │  :3005               │
 │  Webhook · Auto-assign   │             │  Redis · In-app      │
 └──────────────────────────┘             └─────────────────────┘
```

### Services

| Service | Role |
|---|---|
| `api-gateway` | Single entry point — JWT validation, rate limiting, reverse proxy |
| `auth` | Register/Login, JWT issuance, GitHub OAuth |
| `user-team` | Users, teams, roles, GitHub repo linking |
| `task` | Task CRUD, state machine, audit log |
| `github-integration` | Receives GitHub webhooks, triggers task transitions |
| `notification` | In-app notifications with read/unread state |
| `frontend` | React SPA with role-based UI and Kanban board |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Node.js, TypeScript, Express |
| Frontend | React 19, TypeScript, Vite, Zustand |
| Database | PostgreSQL 16 — one isolated DB per service |
| Cache | Redis 7 |
| ORM | Prisma |
| Auth | JWT, bcrypt, GitHub OAuth |
| Containers | Docker, Docker Compose |
| Testing | Vitest, fast-check (property-based testing) |

---

## Key Feature: Git-Driven Task Automation

The core of the system is the webhook processor in the GitHub Integration service.
When a developer pushes a branch named `feature/TASK-42-my-feature`, the system:

1. Extracts the task ID from the branch name
2. Resolves the GitHub username to an internal user ID
3. Auto-assigns the task to the developer
4. Transitions the task state to `IN_PROGRESS`

```typescript
// github-integration/src/services/webhookProcessor.ts

async function handlePush(event: WebhookEvent): Promise<void> {
  const branchName = (event.payload.ref as string).replace('refs/heads/', '');
  const taskId = extractTaskId(event); // extracts TASK-42 from branch name
  if (!taskId) return;

  // Auto-assign: resolve GitHub username → internal userId
  const pusherLogin = event.payload.pusher?.name ?? '';
  const assigneeId = await resolveUserByGithub(pusherLogin);
  if (assigneeId) {
    await assignTask(taskId, assigneeId);
  }

  // Transition task state automatically
  await transitionTask(taskId, 'IN_PROGRESS', `Branch push: ${branchName}`, `push:${event.repoFullName}`);
  await linkGitRef(taskId, 'BRANCH', branchName, event.repoFullName);
}

async function handlePullRequest(event: WebhookEvent): Promise<void> {
  const { action, pull_request: pr } = event.payload;
  const taskId = extractTaskId(event);
  if (!taskId) return;

  if (action === 'opened') {
    await transitionTask(taskId, 'IN_REVIEW', `PR opened: ${pr.title}`, `pull_request:opened`);
  } else if (action === 'closed' && pr.merged) {
    await transitionTask(taskId, 'COMPLETED', `PR merged: ${pr.title}`, `pull_request:merged`);
  } else if (action === 'closed' && !pr.merged) {
    await transitionTask(taskId, 'IN_PROGRESS', `PR rejected: ${pr.title}`, `pull_request:closed`);
  }
}
```

---

## Task State Machine

Every task follows a strict state machine enforced at the service level.
State transitions are validated before any update — invalid transitions throw immediately.

```typescript
// task/src/services/taskService.ts

const VALID_TRANSITIONS: Record<TaskState, TaskState[]> = {
  BACKLOG:     ['TODO', 'BLOCKED', 'CANCELLED'],
  TODO:        ['IN_PROGRESS', 'BLOCKED', 'CANCELLED'],
  IN_PROGRESS: ['IN_REVIEW', 'BLOCKED', 'CANCELLED'],
  IN_REVIEW:   ['COMPLETED', 'IN_PROGRESS', 'BLOCKED', 'CANCELLED'],
  BLOCKED:     ['BACKLOG', 'TODO', 'IN_PROGRESS', 'IN_REVIEW', 'CANCELLED'],
  COMPLETED:   [],
  CANCELLED:   [],
};

export function applyTransition(current: TaskState, target: TaskState): void {
  if (TERMINAL_STATES.has(current)) {
    throw new ConflictError(`Task is in terminal state '${current}'`);
  }
  if (!VALID_TRANSITIONS[current].includes(target)) {
    throw new ValidationError(
      `Invalid transition: '${current}' → '${target}'. Valid: ${VALID_TRANSITIONS[current].join(', ')}`
    );
  }
}
```

CTO/COO roles can bypass the state machine via an override endpoint, with mandatory justification logged to the audit trail.

---

## Role-Based Access Control

Six roles with distinct permissions enforced at the UI level:

| Role | Capabilities |
|---|---|
| `CEO` | Full overview dashboard, KPIs, team management, task transitions if assigned |
| `CTO` | Full board access, state overrides, webhook management, task transitions if assigned |
| `COO` | Same as CTO |
| `Lead_Dev` | Create tasks, manage team assignments, task transitions |
| `Dev` | View board, transition own assigned tasks via Git or UI |
| `Viewer` | Read-only access |

---

## Audit Log

Every state transition — whether triggered by a developer, a manager, or a GitHub webhook — is recorded in an immutable audit log with actor, source, previous state, new state, and reason.

```typescript
await tx.auditLog.create({
  data: {
    taskId: task.id,
    actorId,
    actorType,           // USER | SYSTEM
    eventSource,         // MANUAL | GITHUB
    previousState: task.state,
    newState: targetState,
    reason,
    triggeringEvent,     // e.g. "push:org/repo" or "MANUAL_OVERRIDE"
  },
});
```

---

## Infrastructure

All services run in Docker containers on a shared bridge network.
Each service has its own isolated PostgreSQL database — no shared schema, no cross-service DB queries.

```
docker compose up --build
```

Prisma migrations run automatically on startup via dedicated migration containers.
ngrok is included for local webhook testing with GitHub.

---

## What I Learned Building This

- Designing a microservices system where services communicate only via HTTP (no shared DB)
- Implementing a formal state machine with validation at the service boundary
- Building a webhook-driven automation pipeline with HMAC signature verification
- Role-based UI with React — same backend, different experience per role
- Property-based testing with fast-check to validate state machine invariants
- Docker Compose orchestration with health checks and dependency ordering

---

*Source code is private. Available for review upon request.*
#   s w i t c h e s - t a s k s - p o r t f o l i o  
 