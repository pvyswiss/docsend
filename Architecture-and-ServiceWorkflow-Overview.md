# DocSend Architecture and Service Workflow Overview

> Self-Hosted Deployment Guide - Version 1.0
> Last Updated: March 2026

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Architecture](#2-system-architecture)
3. [External Services](#3-external-services)
4. [Data Flow Diagrams](#4-data-flow-diagrams)
5. [Trigger.dev Workflows](#5-triggerdev-workflows)
6. [Redis and Cache Strategy](#6-redis-and-cache-strategy)
7. [API Endpoints Overview](#7-api-endpoints-overview)
8. [Database Schema Summary](#8-database-schema-summary)
9. [Environment Variables](#9-environment-variables)
10. [Security Considerations](#10-security-considerations)
11. [Future Integrations](#11-future-integrations)
12. [Deployment Guide](#12-deployment-guide)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Introduction

### Purpose

This document provides a comprehensive overview of the DocSend (Papermark) architecture for self-hosted deployment. It covers the system architecture, external service integrations, data flows, and deployment requirements.

### Overview

DocSend is a document sharing and dataroom platform built with Next.js, PostgreSQL, and various external services for email, AI, analytics, and background job processing.

---

## 2. System Architecture

### High-Level Container Diagram

```mermaid
graph TB
    subgraph "Docker Compose (Host)"
        subgraph "Internal Network"
            A["App Container<br/>Next.js :3000"] --> B["PostgreSQL<br/>:5432"]
            A --> C["Redis<br/>:6379"]
        end
    end
    
    subgraph "External Services"
        A --> D["Resend<br/>(Email)"]
        A --> E["OpenAI<br/>(AI/ML)"]
        A --> F["Tinybird<br/>(Analytics)"]
        A --> G["Hanko<br/>(Passkeys)"]
        A --> H["Trigger.dev<br/>(Background Jobs)"]
        A --> I["Minio/S3<br/>(File Storage)"]
    end
    
    subgraph "Optional/Future"
        A --> J["Slack OAuth<br/>(Integration)"]
        A --> K["Matrix/PVYmessenger<br/>(Future Bridge)"]
    end
```

### Component Overview

| Component | Technology | Port | Purpose |
|-----------|------------|------|---------|
| **App** | Next.js 14 | 3000 | Web application & API |
| **PostgreSQL** | postgres:16-alpine | 5432 | Primary database |
| **Redis** | redis:7-alpine | 6379 | Cache, rate limiting, job queues |

### Service Communication

```mermaid
sequenceDiagram
    participant User
    participant App as Next.js App
    participant DB as PostgreSQL
    participant Redis
    participant S3 as Minio/S3
    participant Email as Resend
    participant AI as OpenAI
    participant Analytics as Tinybird
    participant Jobs as Trigger.dev

    User->>App: Upload Document
    App->>S3: Store file
    App->>DB: Save metadata
    App->>Jobs: Trigger processing job
    Jobs->>AI: Process document
    Jobs->>Analytics: Record event
    Jobs->>Email: Send confirmation
    DB-->>App: Confirm save
    App-->>User: Show success
```

---

## 3. External Services

### Service Mapping Table

| Service | Purpose | Status | Self-Hosted Alternative |
|---------|---------|--------|------------------------|
| **PostgreSQL** | Primary database | ✅ Required | Self-hosted (docker-compose) |
| **Redis** | Cache, rate limiting, job queues | ✅ Required | Self-hosted (docker-compose) |
| **Minio/S3** | File storage | ✅ Required | s3.pingsalabim.com |
| **Resend** | Email sending | ✅ Required | API key provided |
| **OpenAI** | AI features (document Q&A) | ✅ Required | API key provided |
| **Tinybird** | Analytics & event tracking | ✅ Required | Account configured |
| **Hanko** | Passkey authentication | ✅ Required | Account configured |
| **Trigger.dev** | Background job processing | ✅ Required | Account configured |
| **Slack** | Team notifications | ⚠️ Dummy | Can map to Matrix later |
| **Upstash QStash** | Cron scheduling | ⚠️ Optional | Use Trigger.dev instead |
| **Google OAuth** | Social login | ❌ Optional | Disabled |
| **LinkedIn OAuth** | Social login | ❌ Optional | Disabled |
| **PostHog** | Analytics | ❌ Disabled | Using Tinybird instead |

### Service Details

#### Resend (Email)
- **Purpose**: Transactional emails (document viewed, invites, notifications)
- **Required**: Yes
- **Configuration**: `RESEND_API_KEY` environment variable

#### OpenAI (AI/ML)
- **Purpose**: Document Q&A, AI-powered features
- **Required**: Yes
- **Configuration**: `OPENAI_API_KEY` environment variable

#### Tinybird (Analytics)
- **Purpose**: Track page views, clicks, video views, webhook events
- **Required**: Yes
- **Configuration**: `TINYBIRD_TOKEN`, `TINYBIRD_URL`
- **Data Sources**: page_views__v3, video_views__v1, click_events__v1, webhook_events__v1

#### Hanko (Passkeys)
- **Purpose**: Passwordless authentication via passkeys
- **Required**: Yes
- **Configuration**: `HANKO_API_KEY`, `NEXT_PUBLIC_HANKO_TENANT_ID`

#### Trigger.dev (Background Jobs)
- **Purpose**: Async job processing (exports, conversions, notifications)
- **Required**: Yes
- **Configuration**: `TRIGGER_SECRET_KEY`, `TRIGGER_API_URL`

#### Minio/S3 (Storage)
- **Purpose**: Document and media file storage
- **Required**: Yes
- **Configuration**: `NEXT_PRIVATE_UPLOAD_*` environment variables

---

## 4. Data Flow Diagrams

### Document Upload Flow

```mermaid
flowchart TD
    A[User uploads document] --> B[App validates file]
    B --> C[Upload to Minio/S3]
    C --> D[Save to PostgreSQL]
    D --> E[Trigger document processing]
    E --> F[OpenAI processing]
    F --> G[Tinybird: Record event]
    G --> H[Resend: Send confirmation]
    H --> I[User sees success]
```

### Link View Tracking Flow

```mermaid
flowchart TD
    A[Visitor clicks link] --> B[App validates link]
    B --> C[Check rate limit Redis]
    C --> D{Allowed?}
    D -->|No| E[Return 429]
    D -->|Yes| F[Record Tinybird event]
    F --> G[Send webhook if enabled]
    G --> H[Send email notification]
    H --> I[Return document/link]
```

### Background Job Flow (Trigger.dev)

```mermaid
flowchart TD
    A[User triggers action] --> B[App calls Trigger.dev]
    B --> C[Trigger.dev queues job]
    C --> D[Job executes async]
    D --> E{Job Type}
    E -->|Export| F[Query Tinybird]
    E -->|Convert| G[Process with FFmpeg/Python]
    E -->|Notify| H[Send via Resend]
    F --> I[Upload to S3]
    G --> I
    I --> J[Send completion email]
```

### Email Notification Flow

```mermaid
sequenceDiagram
    participant User as User
    participant App as Next.js API
    participant Redis as Redis
    participant Resend as Resend API
    
    User->>App: Trigger notification
    App->>Redis: Check rate limit
    Redis-->>App: Allowed
    App->>Resend: Send email
    Resend-->>App: Success
    App-->>User: Notification sent
```

---

## 5. Trigger.dev Workflows

### Background Jobs Overview

| Trigger Name | File | Purpose | Uses |
|--------------|------|---------|------|
| `export-visits` | `lib/trigger/export-visits.ts` | Export visitor data to CSV | Tinybird, Redis, Resend, S3 |
| `bulk-download` | `lib/trigger/bulk-download.ts` | Create ZIP of multiple files | Redis job store, S3 |
| `convert-files` | `lib/trigger/convert-files.ts` | PDF to image conversion | Python, S3 |
| `pdf-to-image-route` | `lib/trigger/pdf-to-image-route.ts` | PDF page rendering | S3 |
| `optimize-video-files` | `lib/trigger/optimize-video-files.ts` | Video transcoding | FFmpeg, S3 |
| `send-scheduled-email` | `lib/trigger/send-scheduled-email.ts` | Scheduled email notifications | Resend |
| `dataroom-upload-notification` | `lib/trigger/dataroom-upload-notification.ts` | Notify on dataroom uploads | Resend |
| `dataroom-change-notification` | `lib/trigger/dataroom-change-notification.ts` | Notify on dataroom changes | Resend, Redis |
| `cleanup-expired-exports` | `lib/trigger/cleanup-expired-exports.ts` | Clean old exports | Vercel Blob, Redis |
| `pause-reminder-notification` | `lib/trigger/pause-reminder-notification.ts` | Billing reminders | Resend |
| `automatic-unpause` | `lib/trigger/automatic-unpause.ts` | Auto-unpause teams | Prisma |
| `conversation-message-notification` | `lib/trigger/conversation-message-notification.ts` | Chat notifications | Resend |

### Trigger.dev Configuration

```typescript
// trigger.config.ts
export default defineConfig({
  project: "proj_plmsfqvqunboixacjjus",
  dirs: ["./lib/trigger", "./ee/features/ai/lib/trigger"],
  maxDuration: timeout.None,
  build: {
    extensions: [
      prismaExtension({ schema: "prisma/schema/schema.prisma" }),
      ffmpeg(),
      pythonExtension({ scripts: ["./**/*.py"] }),
    ],
  },
});
```

---

## 6. Redis and Cache Strategy

### Redis Usage Breakdown

| Usage | File | Description | Local Redis Compatible |
|-------|------|-------------|----------------------|
| **Rate Limiting** | `lib/redis.ts` | Limit API requests per team | ✅ Yes |
| **TUS Upload Locker** | `lib/files/tus-redis-locker.ts` | Lock files during upload | ✅ Yes |
| **Job Store** | `lib/redis-job-store.ts` | Store background job state | ✅ Yes |
| **Download Job Store** | `lib/redis-download-job-store.ts` | Track bulk downloads | ✅ Yes |
| **Domain Middleware** | `lib/middleware/domain.ts` | Cache domain configs | ✅ Yes |
| **Slack Tokens** | `lib/integrations/slack/install.ts` | Store OAuth tokens | ✅ Yes |
| **Dataroom Queue** | `lib/redis/dataroom-notification-queue.ts` | Notification queuing | ✅ Yes |
| **QStash** | `lib/cron/index.ts` | Cron job scheduling | ❌ No - requires Upstash |

### Local Redis Configuration

```yaml
# docker-compose.yml
redis:
  image: redis:7-alpine
  command: redis-server --appendonly yes --bind 0.0.0.0 --protected-mode no
  volumes:
    - redis_data:/data
  networks:
    - docsend-network
```

### QStash Alternatives

QStash is used for:
- Cron job scheduling
- Async webhook delivery

**Alternatives:**
1. **Trigger.dev** - Already integrated, use for all background jobs
2. **Local Cron** - Add cron entries to container
3. **Upstash QStash** - Create free account (10k jobs/month)

---

## 7. API Endpoints Overview

### Main API Routes

| Path | Method | Description | Auth |
|------|--------|-------------|------|
| `/api/teams/*` | CRUD | Team management | ✅ |
| `/api/documents/*` | CRUD | Document management | ✅ |
| `/api/links/*` | CRUD | Link management | ✅ |
| `/api/datarooms/*` | CRUD | Dataroom management | ✅ |
| `/api/views/*` | POST | Track document views | ❌ |
| `/api/record_view` | POST | Record page view | ❌ |
| `/api/record_click` | POST | Record click event | ❌ |
| `/api/record_video_view` | POST | Record video view | ❌ |
| `/api/webhooks/*` | CRUD | Webhook management | ✅ |
| `/api/integrations/slack/*` | CRUD | Slack integration | ✅ |
| `/api/ai/*` | Various | AI features | ✅ |

### Authentication

| Method | Provider | Status |
|--------|----------|--------|
| Email/Password | NextAuth | ✅ |
| Passkeys | Hanko | ✅ |
| Google OAuth | NextAuth | ❌ Optional |
| LinkedIn OAuth | NextAuth | ❌ Optional |
| Slack OAuth | Slack | ⚠️ Dummy |

---

## 8. Database Schema Summary

### Core Tables

```
teams
├── id (UUID)
├── name (String)
├── plan (String) - e.g., "datarooms-premium"
├── stripeId (String, nullable)
├── createdAt (DateTime)
└── updatedAt (DateTime)

users
├── id (UUID)
├── email (String)
├── name (String, nullable)
├── teamId (UUID, FK)
├── createdAt (DateTime)
└── updatedAt (DateTime)

documents
├── id (UUID)
├── teamId (UUID, FK)
├── name (String)
├── storageKey (String)
├── versions (JSON)
├── createdAt (DateTime)
└── updatedAt (DateTime)

links
├── id (UUID)
├── documentId (UUID, FK, nullable)
├── dataroomId (UUID, FK, nullable)
├── teamId (UUID, FK)
├── expiresAt (DateTime, nullable)
├── password (String, nullable)
├── createdAt (DateTime)
└── updatedAt (DateTime)

datarooms
├── id (UUID)
├── teamId (UUID, FK)
├── name (String)
├── createdAt (DateTime)
└── updatedAt (DateTime)

invitations
├── id (UUID)
├── email (String)
├── teamId (UUID, FK)
├── expiresAt (DateTime)
├── createdAt (DateTime)
└── updatedAt (DateTime)
```

### Key Relationships

```mermaid
erDiagram
    TEAMS ||--o{ USERS : "has"
    TEAMS ||--o{ DOCUMENTS : "owns"
    TEAMS ||--o{ LINKS : "creates"
    TEAMS ||--o{ DATAROOMS : "manages"
    DOCUMENTS ||--o{ LINK : "has"
    DATAROOMS ||--o{ LINK : "has"
    TEAMS ||--o{ INVITATIONS : "sends"
```

---

## 9. Environment Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `POSTGRES_PRISMA_URL` | Database connection string | `postgresql://user:pass@postgres:5432/docsend` |
| `NEXTAUTH_SECRET` | NextAuth secret | Generate strong random string |
| `NEXTAUTH_URL` | App URL | `https://dataroom.pvy.swiss` |
| `NEXT_PUBLIC_BASE_URL` | Public base URL | `https://dataroom.pvy.swiss` |
| `NEXT_PUBLIC_APP_BASE_HOST` | App host | `dataroom.pvy.swiss` |
| `RESEND_API_KEY` | Resend API key | `re_...` |
| `OPENAI_API_KEY` | OpenAI API key | `sk-...` |
| `TINYBIRD_TOKEN` | Tinybird token | `p.eyJ...` |
| `TINYBIRD_URL` | Tinybird API URL | `https://api.europe-west2.gcp.tinybird.co` |
| `HANKO_API_KEY` | Hanko API key | From Hanko dashboard |
| `NEXT_PUBLIC_HANKO_TENANT_ID` | Hanko tenant ID | `https://xxx.hanko.io` |
| `TRIGGER_SECRET_KEY` | Trigger.dev key | `tr_...` |
| `TRIGGER_API_URL` | Trigger.dev URL | `https://api.trigger.dev` |
| `NEXT_PRIVATE_UPLOAD_*` | S3/Minio config | Minio credentials |
| `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY` | Encryption key | Generate strong random string |
| `INTERNAL_API_KEY` | Internal API key | Generate strong random string |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `SLACK_CLIENT_ID` | Slack OAuth | Dummy |
| `SLACK_CLIENT_SECRET` | Slack OAuth | Dummy |
| `GOOGLE_CLIENT_ID` | Google OAuth | Disabled |
| `GOOGLE_CLIENT_SECRET` | Google OAuth | Disabled |
| `NEXT_PUBLIC_WEBHOOK_BASE_HOST` | Webhook host | Disabled |

### Build-Time Variables (GitLab CI)

These must be passed as `--build-arg` during Docker build:

```
OPENAI_API_KEY
UPSTASH_REDIS_REST_URL
UPSTASH_REDIS_REST_TOKEN
UPSTASH_REDIS_REST_LOCKER_URL
UPSTASH_REDIS_REST_LOCKER_TOKEN
TINYBIRD_TOKEN
TINYBIRD_URL
NEXT_PUBLIC_BASE_URL
NEXT_PUBLIC_APP_BASE_HOST
NEXT_PUBLIC_WEBHOOK_BASE_HOST
TRIGGER_SECRET_KEY
TRIGGER_API_URL
HANKO_API_KEY
NEXT_PUBLIC_HANKO_TENANT_ID
SLACK_CLIENT_ID
SLACK_CLIENT_SECRET
```

---

## 10. Security Considerations

### Authentication

- **NextAuth.js** for session management
- **Hanko Passkeys** for passwordless auth
- **JWT tokens** for API authentication

### Data Encryption

- **Document passwords** encrypted with `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY`
- **S3/Mino** connections over HTTPS
- **PostgreSQL** connections over internal network only

### Network Security

```mermaid
flowchart TB
    subgraph "Public"
        A[User Browser]
    end
    
    subgraph "DMZ/Proxy"
        B[Pangolin Proxy]
    end
    
    subgraph "Internal Network"
        C[App Container]
        D[PostgreSQL]
        E[Redis]
    end
    
    A --> B
    B --> C
    C --> D
    C --> E
    
    D -.->|Internal Only| F[No External Access]
    E -.->|Internal Only| F
```

### Secrets Management

- **Never commit** secrets to git
- Use GitLab CI/CD variables for build-time secrets
- Use docker-compose `.env` file for runtime secrets

---

## 11. Future Integrations

### Matrix/PVYmessenger Bridge

Since you have PVYmessenger (Matrix), here's the integration plan:

#### Current Slack Flow
```
Event → App → Slack API → Slack Channel
```

#### Future Matrix Flow
```
Event → App → Matrix Webhook → PVYmessenger → Matrix Room
```

#### Implementation Steps
1. Create Matrix webhook or bot
2. Add webhook URL to settings
3. Replace Slack notifications with Matrix messages

### Feature Roadmap

| Feature | Priority | Effort | Notes |
|---------|----------|--------|-------|
| Matrix integration | High | Medium | Replace Slack notifications |
| Upstash QStash | Low | Low | Use Trigger.dev instead |
| Custom domain support | Medium | Low | Already supported |
| SAML/SSO | Medium | Medium | Available in ee/ |
| Custom branding | Medium | Low | Available in settings |

---

## 12. Deployment Guide

### Prerequisites

1. **Docker host** (Proxmox/Alpine recommended)
2. **GitLab registry** access
3. **Domain** with DNS configured

### Quick Start

```bash
# Pull the image
docker-compose pull

# Start services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f app
```

### Health Checks

```yaml
# docker-compose.yml healthchecks
postgres:
  test: ["CMD-SHELL", "pg_isready -U docsend"]
  interval: 5s
  timeout: 5s
  retries: 5

redis:
  test: ["CMD", "redis-cli", "ping"]
  interval: 5s
  timeout: 5s
  retries: 5
```

### First Run Setup

1. Start containers: `docker-compose up -d`
2. Wait for healthy: `docker-compose ps`
3. Run migrations: `docker-compose exec app npx prisma migrate deploy`
4. Access app: `http://localhost:3000`
5. Create first team via UI

---

## 13. Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Build fails with missing env vars | Add all `--build-arg` variables to CI/CD |
| Redis connection errors | Check `UPSTASH_REDIS_REST_URL` format |
| Email not sending | Verify `RESEND_API_KEY` is valid |
| Slow analytics | Check Tinybird data sources exist |
| Passkeys not working | Verify Hanko credentials |
| Background jobs not running | Check Trigger.dev dashboard |

### Debug Commands

```bash
# Check container logs
docker-compose logs -f app

# Check Redis connectivity
docker-compose exec app redis-cli ping

# Check database connectivity
docker-compose exec app npx prisma db ping

# Check environment variables
docker-compose exec app env | grep -E "^(NEXT|RESEND|OPENAI|TINYBIRD|HANKO|TRIGGER)"

# Rebuild and restart
docker-compose down && docker-compose build --no-cache && docker-compose up -d
```

### Monitoring

| Service | Dashboard |
|---------|-----------|
| App | `http://dataroom.pvy.swiss/api/health` |
| PostgreSQL | Via pgAdmin or CLI |
| Redis | `redis-cli INFO` |
| Trigger.dev | trigger.dev dashboard |
| Tinybird | tinybird.co dashboard |

---

## Appendix A: File Structure

```
docsend/
├── app/                    # Next.js App Router
├── components/            # React components
├── lib/                   # Core libraries
│   ├── trigger/          # Trigger.dev jobs
│   ├── tinybird/         # Analytics
│   ├── redis/            # Redis utilities
│   ├── emails/           # Email templates
│   └── integrations/    # Third-party integrations
├── pages/                # Next.js Pages Router (legacy)
├── prisma/               # Database schema
├── ee/                   # Enterprise features
│   ├── stripe/          # Billing
│   └── features/        # Advanced features
├── docker-compose.yml   # Container orchestration
├── Dockerfile           # App container
├── env.tst              # Environment template
└── .gitlab-ci.yml       # CI/CD pipeline
```

---

## Appendix B: API Rate Limits

| Endpoint | Limit | Window |
|----------|-------|--------|
| Document views | 100 | Per IP / minute |
| API general | 1000 | Per team / hour |
| Link creation | 100 | Per team / hour |
| Bulk downloads | 10 | Per team / day |

---

*Document generated for self-hosted DocSend deployment*
