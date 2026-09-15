# SocialFlow 28 - Architecture Documentation

## Overview

**SocialFlow 28** is a production-ready, multi-tenant SaaS platform for automating social media content management and publishing. It enables users to connect social media accounts, manage content from a central library, and automatically publish content on a recurring 28-day cycle using official platform APIs.

## Core Principles

1. **Official APIs Only** - Never use scraping, password storage, or browser automation
2. **Multi-Tenant Isolation** - Complete data separation between users
3. **Security First** - OAuth 2.0, encrypted credentials, comprehensive audit logging
4. **Reliability** - Server-side job queue with retry logic and idempotency
5. **Scalability** - Stateless services, distributed job processing
6. **User Experience** - Simple, intuitive interface for non-technical business owners

## Technology Stack

### Frontend
- **Framework**: Next.js 14+ with React 18
- **Language**: TypeScript
- **Styling**: Tailwind CSS
- **UI Components**: Headless UI / Radix UI
- **State Management**: React Query / SWR for server state
- **Testing**: Playwright (E2E), Jest (Unit)

### Backend
- **Runtime**: Node.js 18+
- **Framework**: Next.js API Routes + Server Actions
- **Language**: TypeScript
- **Database**: PostgreSQL 14+
- **ORM**: Prisma
- **Job Queue**: Redis + BullMQ
- **Authentication**: NextAuth.js + custom OAuth
- **Validation**: Zod
- **Testing**: Jest + Supertest

### Infrastructure
- **Database**: PostgreSQL 14+
- **Cache/Queue**: Redis 7+
- **Storage**: AWS S3 or S3-compatible (MinIO for local development)
- **Containerization**: Docker + Docker Compose
- **Deployment**: Docker, Kubernetes-ready

## Architecture Layers

### 1. Presentation Layer (Frontend)
- Next.js pages and components
- Server Components for data fetching
- Client Components for interactivity
- Responsive design (mobile, tablet, desktop)

### 2. API Layer (Next.js API Routes + Server Actions)
- RESTful endpoints
- Server Actions for form submissions
- Request validation with Zod
- Error handling and logging
- Rate limiting middleware

### 3. Business Logic Layer
- Service classes for domain logic
- Integration services for third-party APIs
- Scheduler and job processing
- Time zone handling
- Content validation

### 4. Data Access Layer (Prisma ORM)
- Type-safe database queries
- Migration system
- Relationship management
- Connection pooling

### 5. Job Queue System (Redis + BullMQ)
- Scheduled post publishing
- Token refresh
- Google Sheets synchronization
- Notification delivery
- Retry with exponential backoff

### 6. Authentication & Authorization
- Email/password registration and login
- OAuth 2.0 integration (Google, Instagram, Facebook, LinkedIn, X, YouTube)
- Session management
- RBAC (Role-Based Access Control)
- Multi-tenant isolation

## Data Model

### Core Entities

```
Users
  ├── Organizations (multi-tenant)
  │   ├── SocialAccounts (Instagram, Facebook, LinkedIn, X, YouTube)
  │   │   └── OAuthTokens (encrypted)
  │   ├── MediaFiles (with metadata)
  │   ├── ContentItems (posts, reels, stories)
  │   ├── Campaigns (thematic groupings)
  │   ├── ContentCycles (28-day schedules)
  │   │   └── ScheduledPosts
  │   ├── PublicationAttempts (publishing history)
  │   ├── Analytics (platform-provided metrics)
  │   ├── Notifications (user alerts)
  │   └── AuditLogs (security & compliance)
  │
  ├── GoogleDriveConnections
  ├── GoogleSheetConnections
  └── UserSettings (timezone, notifications, preferences)
```

### Detailed Schema (See `SCHEMA.md` for complete definitions)

## Integration Architecture

### Platform-Agnostic Integration Framework

Each social media platform has:
1. **Authentication Service** - OAuth flow, token management
2. **Account Service** - Account info, followers, insights
3. **Publishing Service** - Post creation and scheduling
4. **Content Service** - Media upload, format validation
5. **Analytics Service** - Metrics retrieval
6. **Error Handler** - Platform-specific error handling

### Supported Platforms (Phase 1)

- Instagram (Graph API)
- Facebook Pages (Graph API)
- LinkedIn (REST API v2)
- X / Twitter (API v2)
- YouTube (YouTube Data API)
- Google Drive (Drive API v3)
- Google Sheets (Sheets API v4)

### Adding New Platforms

1. Create platform-specific service in `/services/integrations/[platform]/`
2. Implement standard interface
3. Add OAuth configuration
4. Extend database schema if needed
5. Update content validation rules
6. Add platform selector in UI

## 28-Day Scheduling System

### Scheduling Engine

The 28-day cycle is the core automation feature:

1. **Cycle Creation** - User defines 28 days of content
2. **Auto-Generation** - At end of cycle, automatically create next cycle
3. **Modes**:
   - **Repeat**: Same schedule repeats
   - **Template**: Previous schedule used as template with modifications
4. **Publication Queue** - Jobs created for each scheduled post
5. **Job Processing** - Background worker publishes at scheduled time
6. **Error Recovery** - Retries with exponential backoff
7. **Duplicate Prevention** - Detects and prevents duplicate publications

### Time Zone Handling

- All timestamps stored in UTC
- User's local timezone configurable
- Display conversion on frontend
- DST handled automatically by system

## Security Architecture

### Authentication & Authorization
- Email verification for registration
- Secure password hashing (bcrypt)
- JWT-based sessions
- CSRF protection on state-changing operations
- Rate limiting per IP and per user

### Data Protection
- All OAuth tokens encrypted at rest (AES-256)
- Secure OAuth flows (PKCE where applicable)
- HTTPS only
- Secure cookies (httpOnly, secure, sameSite)
- Input validation and sanitization
- Output encoding against XSS

### Multi-Tenant Isolation
- Row-level security policies in database
- Organization-scoped queries in all endpoints
- User cannot access data from other organizations
- API endpoints verify authorization

### Audit & Compliance
- Comprehensive audit logging
- User action tracking
- API error logging
- Token refresh logging
- Failed authentication attempts logged
- GDPR-friendly data export/deletion

### Secrets Management
- Environment variables for all credentials
- `.env.example` without secrets
- Never commit `.env` or credentials
- Docker secrets support for production

## API Design

### Conventions
- RESTful endpoints
- JSON request/response
- Standard HTTP status codes
- Pagination with limit/offset
- Error responses with error codes and messages

### Versioning
- API version in header: `Accept: application/json;version=1`
- Backward compatibility maintained
- Deprecation warnings in responses

### Rate Limiting
- Per-user: 1000 requests/hour for standard endpoints
- Per-user: 100 requests/hour for heavy operations (publishing)
- Per-IP: 10,000 requests/hour
- Exponential backoff recommended for clients

## Deployment Strategy

### Local Development
```
docker-compose up -d
npm install
npx prisma migrate dev
npm run dev
```

### Staging
- Separate PostgreSQL instance
- Separate Redis instance
- Separate S3 bucket
- Test OAuth applications

### Production
- Managed PostgreSQL (AWS RDS, Supabase, etc.)
- Managed Redis (AWS ElastiCache, Upstash, etc.)
- S3-compatible storage
- Multi-region deployment ready
- Automated backups
- Health checks and monitoring
- Log aggregation

## Performance Considerations

### Database
- Indexes on frequently queried fields
- Connection pooling (Prisma)
- Query optimization
- Archive old analytics data

### Caching
- Redis for session storage
- Redis for job queue
- User-level caching with React Query

### Job Processing
- Parallel job workers
- Prioritized queue
- Dead letter queue for failed jobs
- Automatic retry with backoff

### Media
- Lazy loading on frontend
- Image optimization with Next.js Image
- Thumbnail generation
- CDN delivery (optional)

## Development Workflow

### Branching Strategy
- `main` - production-ready code
- `develop` - integration branch
- `feature/*` - feature branches
- `bugfix/*` - bug fix branches

### Pull Request Process
1. Create feature branch from `develop`
2. Implement changes with tests
3. Create PR with clear description
4. Code review
5. Merge to `develop`
6. Merge to `main` for release

### Testing Strategy
- Unit tests for business logic
- Integration tests for APIs
- E2E tests for critical user flows
- Test database isolation per test
- Mock external APIs (Google, social media)

## Monitoring & Observability

### Logging
- Structured JSON logging
- Log levels: debug, info, warn, error
- Centralized log aggregation
- Request tracking with correlationId

### Metrics
- Publishing success/failure rates
- API response times
- Job queue depth
- Database query performance
- Error rates by type

### Alerts
- Failed publication attempts
- Token refresh failures
- Queue processing delays
- High error rates
- Resource exhaustion

## Roadmap

### Phase 1: MVP (This Implementation)
- User authentication
- Content library
- Social account connections
- 28-day scheduling
- Basic analytics
- Email notifications

### Phase 2: Enhanced Features
- Approval workflows
- Advanced analytics dashboards
- Content collaboration
- Bulk operations
- SMS notifications

### Phase 3: Scale & Optimize
- Admin dashboard
- API for third-party integrations
- White-label support
- Advanced reporting
- Webhook support

### Phase 4: Enterprise
- SSO integration
- Advanced security features
- Custom integrations
- Dedicated support

## Documentation Structure

- `README.md` - Getting started guide
- `ARCHITECTURE.md` - This document
- `SECURITY.md` - Security implementation details
- `API.md` - API endpoint documentation
- `SCHEMA.md` - Database schema details
- `SETUP.md` - Local development setup
- `DEPLOYMENT.md` - Production deployment guide
- `INTEGRATIONS.md` - Social media integration guide
