# Cal.com API v2 Backend Architecture

## Table of Contents

- [Overview](#overview)
- [Architectural Patterns](#architectural-patterns)
- [System Design](#system-design)
- [Folder Structure](#folder-structure)
- [Data Flow](#data-flow)
- [Technology Stack](#technology-stack)
- [Security Architecture](#security-architecture)
- [Best Practices](#best-practices)
- [Future Considerations](#future-considerations)

---

## Overview

The Cal.com API v2 is a modern, enterprise-grade backend system built on **NestJS** framework. It follows clean architecture principles with clear separation between open-source and commercial features, implements date-based API versioning, and uses a layered architecture for scalability and maintainability.

**Key Characteristics:**
- **Framework**: NestJS 10.x with TypeScript
- **Database**: PostgreSQL with Prisma ORM
- **Caching**: Redis for rate limiting and caching
- **Authentication**: Multi-strategy (API Keys, OAuth 2.0, Session-based)
- **Architecture**: Modular, layered, with dependency injection
- **License Model**: Open core with commercial `/ee` features

---

## Architectural Patterns

### 1. Layered Architecture (Clean Architecture)

The application follows a strict layered architecture with clear boundaries:

```
┌─────────────────────────────────────────────────────────┐
│                   Presentation Layer                     │
│              (Controllers, Guards, Filters)              │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                      │
│            (Services, Business Logic)                    │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Domain Layer                           │
│          (Entities, DTOs, Validators)                    │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                Infrastructure Layer                      │
│    (Repositories, Database, External Services)           │
└─────────────────────────────────────────────────────────┘
```

**Benefits:**
- Clear separation of concerns
- High testability through dependency injection
- Independent layer evolution
- Easy to understand and maintain

### 2. Module-Based Organization

Each feature is encapsulated in a **NestJS Module** with clear boundaries:

```typescript
@Module({
  imports: [PrismaModule, EventTypesModule, TokensModule],
  providers: [UsersRepository, UsersService],
  exports: [UsersRepository, UsersService],
})
export class UsersModule {}
```

**Key Principles:**
- **Single Responsibility**: Each module handles one domain concern
- **Explicit Dependencies**: Imports/exports clearly define module boundaries
- **Reusability**: Modules can be composed and reused
- **Isolation**: Changes in one module don't affect others

### 3. Repository Pattern

Separation of data access logic from business logic:

```typescript
@Injectable()
export class UsersRepository {
  constructor(
    private readonly dbRead: PrismaReadService,
    private readonly dbWrite: PrismaWriteService
  ) {}

  async findById(userId: number) {
    return this.dbRead.prisma.user.findUnique({
      where: { id: userId }
    });
  }
}
```

**Features:**
- **Read/Write Segregation**: Separate services for queries and mutations
- **Connection Pooling**: Optimized per operation type
- **Abstraction**: Business logic doesn't know about database details
- **Testability**: Easy to mock repositories in tests

### 4. Date-Based API Versioning

Cal.com uses a unique versioning strategy based on dates:

```
/ee/bookings/
├── 2024-04-15/    # Version released April 15, 2024
├── 2024-08-13/    # Version released August 13, 2024
└── shared/        # Common utilities
```

**Version Resolution:**
```typescript
// Client specifies version via header
Headers: { "cal-api-version": "2024-08-13" }

// Server routes to appropriate controller
@Controller({
  path: "/v2/bookings",
  version: VERSION_2024_08_13_VALUE
})
```

**Benefits:**
- Multiple versions coexist without conflicts
- Clear timeline of API evolution
- Easy deprecation strategy
- Transparent client migration path

### 5. Guard-Based Security

Declarative security using NestJS guards and decorators:

```typescript
@Post("/:bookingUid/cancel")
@UseGuards(BookingUidGuard, OptionalApiAuthGuard)
@Permissions([BOOKING_WRITE])
@PlatformPlan("SCALE")
async cancelBooking() { ... }
```

**Guard Types:**
- **Authentication Guards**: `ApiAuthGuard`, `OptionalApiAuthGuard`, `NextAuthGuard`
- **Authorization Guards**: `PermissionsGuard`, `OrganizationRolesGuard`
- **Business Logic Guards**: `BookingUidGuard`, `IsTeamInOrgGuard`
- **Billing Guards**: `PlatformPlanGuard`

**Benefits:**
- Centralized security logic
- Reusable across endpoints
- Easy to test in isolation
- Self-documenting security requirements

### 6. Service Layer Patterns

Services follow single responsibility with specialized sub-services:

```typescript
// Main orchestration service
BookingsService_2024_08_13
  ├── InputBookingsService      // Request transformation
  ├── OutputBookingsService     // Response transformation
  ├── ErrorsBookingsService     // Error handling
  └── BookingReferencesService  // Reference management
```

**Pattern Benefits:**
- Focused responsibilities
- Easier testing
- Better code organization
- Reusable components

---

## System Design

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway Layer                     │
│  • CORS Configuration                                    │
│  • Helmet Security Headers                               │
│  • API Versioning (Header-based)                         │
│  • Rate Limiting (Redis-backed)                          │
│  • Request Logging                                       │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│           Authentication & Authorization Layer           │
│  • Passport.js Strategies                                │
│  • JWT Token Validation                                  │
│  • API Key Authentication                                │
│  • OAuth 2.0 Flow                                        │
│  • Session Management (NextAuth)                         │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Business Logic Layer                   │
│  • Domain Services                                       │
│  • Workflow Orchestration                                │
│  • External Service Integration                          │
│  • Event Handling                                        │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Data Access Layer                      │
│  • Repository Pattern                                    │
│  • Query Building                                        │
│  • Transaction Management                                │
│  • Cache Management                                      │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                  Infrastructure Layer                    │
│  • PostgreSQL (Primary + Read Replicas)                  │
│  • Redis (Cache + Rate Limiting)                         │
│  • Bull Queue (Background Jobs)                          │
│  • External APIs (Stripe, Google, etc.)                  │
└─────────────────────────────────────────────────────────┘
```

### Database Architecture

**Connection Strategy:**

```typescript
// Read Operations - Optimized for queries
PrismaReadService
  ├── Connection Pool: 9 connections (default)
  ├── Idle Timeout: 300 seconds
  └── Usage: GET requests, queries, reports

// Write Operations - Optimized for mutations
PrismaWriteService
  ├── Connection Pool: 7 connections (default)
  ├── Idle Timeout: 300 seconds
  └── Usage: POST, PUT, DELETE requests

// Worker Operations - Background jobs
PrismaWorkerModule
  ├── Read Pool: 4 connections
  ├── Write Pool: 6 connections
  └── Usage: Bull queue jobs, async tasks
```

**Benefits:**
- Efficient resource utilization
- Prevents connection exhaustion
- Supports future read replica scaling
- Better performance monitoring

### Caching Strategy

**Redis Usage:**

```typescript
1. Rate Limiting
   - Store: Request counts per user/IP
   - TTL: Based on throttle window
   - Key Pattern: throttler:${userId}:${endpoint}

2. Guard Results
   - Store: Authorization decisions
   - TTL: Configurable per guard
   - Key Pattern: guard:${guardName}:${userId}:${resourceId}
   - Note: Only cache "true" results

3. Application Cache
   - Store: Calendar data, user profiles
   - TTL: Varies by data type
   - Key Pattern: cache:${type}:${id}
```

### Background Job Processing

**Bull Queue Architecture:**

```typescript
Service Layer
  ↓ [Enqueue Job]
Bull Queue (Redis)
  ↓ [Pick Job]
Worker Process
  ↓ [Execute]
PrismaWorkerModule
  ↓ [Database Operation]
PostgreSQL
```

**Job Types:**
- Email sending
- Webhook delivery
- Calendar sync
- Report generation
- Billing operations

### Authentication Flow

**Multi-Strategy Authentication:**

```typescript
1. API Key Flow
   ┌─────────────────────────────────────┐
   │ Client includes API Key in header   │
   └────────────┬────────────────────────┘
                ↓
   ┌─────────────────────────────────────┐
   │ ApiAuthGuard validates key          │
   │ ├── Extract from Authorization      │
   │ ├── Validate prefix (cal_)          │
   │ └── Lookup in database              │
   └────────────┬────────────────────────┘
                ↓
   ┌─────────────────────────────────────┐
   │ Load user & permissions             │
   └─────────────────────────────────────┘

2. OAuth 2.0 Flow
   ┌─────────────────────────────────────┐
   │ Client includes Bearer token        │
   └────────────┬────────────────────────┘
                ↓
   ┌─────────────────────────────────────┐
   │ ApiAuthGuard validates token        │
   │ ├── Verify JWT signature            │
   │ ├── Check expiration                │
   │ └── Validate OAuth client           │
   └────────────┬────────────────────────┘
                ↓
   ┌─────────────────────────────────────┐
   │ Load user & scoped permissions      │
   └─────────────────────────────────────┘

3. Session Flow (NextAuth)
   ┌─────────────────────────────────────┐
   │ Client includes session cookie      │
   └────────────┬────────────────────────┘
                ↓
   ┌─────────────────────────────────────┐
   │ NextAuthGuard validates session     │
   │ ├── Decrypt session token           │
   │ ├── Verify signature                │
   │ └── Check expiration                │
   └────────────┬────────────────────────┘
                ↓
   ┌─────────────────────────────────────┐
   │ Load user from session              │
   └─────────────────────────────────────┘
```

---

## Folder Structure

### Root Directory Structure

```
apps/api/v2/
├── src/                          # Source code
│   ├── ee/                       # Enterprise Edition (Commercial)
│   ├── modules/                  # Core modules (Open Source)
│   ├── lib/                      # Shared libraries
│   ├── middleware/               # HTTP middleware
│   ├── filters/                  # Exception filters
│   ├── config/                   # Configuration
│   ├── swagger/                  # API documentation
│   ├── app.module.ts            # Root module
│   ├── app.ts                   # Bootstrap configuration
│   └── main.ts                  # Application entry point
│
├── test/                         # E2E test suites
├── scripts/                      # Build and deployment scripts
├── .env.example                 # Environment template
├── Dockerfile                   # Container configuration
├── docker-compose.yaml          # Local development setup
├── nest-cli.json                # NestJS CLI configuration
├── tsconfig.json                # TypeScript configuration
├── jest.config.ts               # Unit test configuration
└── jest-e2e.ts                  # E2E test configuration
```

### Enterprise Edition Structure (`/ee`)

Commercial license features organized by domain:

```
ee/
├── bookings/
│   ├── 2024-04-15/              # Versioned endpoints
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── inputs/
│   │   ├── outputs/
│   │   └── guards/
│   ├── 2024-08-13/              # Latest version
│   └── shared/                  # Common utilities
│
├── calendars/
│   ├── controllers/
│   ├── services/
│   │   ├── calendars.service.ts
│   │   └── calendars-cache.service.ts
│   ├── processors/
│   └── calendars.repository.ts
│
├── event-types/
│   ├── event-types_2024_04_15/
│   └── event-types_2024_06_14/
│
├── schedules/
│   ├── schedules_2024_04_15/
│   └── schedules_2024_06_11/
│
├── gcal/                        # Google Calendar integration
├── me/                          # User profile endpoints
├── provider/                    # Provider-specific features
└── platform-endpoints-module.ts # EE module aggregator
```

### Core Modules Structure (`/modules`)

Open-source features following consistent patterns:

```
modules/
├── api-keys/
│   ├── controllers/
│   │   └── api-keys.controller.ts
│   ├── services/
│   │   └── api-keys.service.ts
│   ├── inputs/
│   │   ├── create-api-key.input.ts
│   │   └── update-api-key.input.ts
│   ├── outputs/
│   │   └── api-key.output.ts
│   ├── api-keys-repository.ts
│   └── api-keys.module.ts
│
├── auth/
│   ├── decorators/              # Custom decorators
│   │   ├── get-user/
│   │   ├── get-optional-user/
│   │   ├── permissions/
│   │   ├── billing/
│   │   └── roles/
│   ├── guards/                  # Authorization guards
│   │   ├── api-auth/
│   │   ├── next-auth/
│   │   ├── permissions/
│   │   ├── optional-api-auth/
│   │   └── billing/
│   ├── strategies/              # Passport strategies
│   │   ├── api-auth/
│   │   └── next-auth/
│   └── auth.module.ts
│
├── users/
│   ├── services/
│   │   └── users.service.ts
│   ├── inputs/
│   │   ├── create-user.input.ts
│   │   ├── update-user.input.ts
│   │   ├── create-managed-user.input.ts
│   │   └── get-users.input.ts
│   ├── outputs/
│   │   └── user.output.ts
│   ├── validators/              # Custom validators
│   │   ├── timezone-validator.ts
│   │   ├── locale-validator.ts
│   │   └── theme-validator.ts
│   ├── users.repository.ts
│   └── users.module.ts
│
├── organizations/
│   ├── bookings/                # Org-scoped bookings
│   ├── teams/                   # Org teams
│   ├── users/                   # Org users
│   ├── event-types/             # Org event types
│   ├── conferencing/            # Org conferencing
│   ├── delegation-credentials/
│   ├── routing-forms/
│   ├── schedules/
│   ├── stripe/
│   ├── webhooks/
│   └── attributes/
│
├── teams/
│   ├── event-types/
│   ├── memberships/
│   ├── schedules/
│   ├── teams/
│   └── verified-resources/
│
├── webhooks/
│   ├── controllers/
│   ├── services/
│   ├── guards/
│   ├── decorators/
│   ├── pipes/
│   ├── inputs/
│   └── outputs/
│
├── billing/
│   ├── controllers/
│   ├── services/
│   └── interfaces/
│
├── slots/                       # Availability slots
│   ├── slots-2024-04-15/
│   └── slots-2024-09-04/
│
├── prisma/                      # Database module
│   ├── prisma-read.service.ts
│   ├── prisma-write.service.ts
│   ├── prisma.module.ts
│   └── prisma-worker.module.ts
│
├── redis/                       # Cache module
├── jwt/                         # JWT utilities
├── tokens/                      # Token management
├── kysely/                      # Type-safe SQL
└── endpoints.module.ts          # Core module aggregator
```

### Shared Libraries (`/lib`)

Reusable utilities and common functionality:

```
lib/
├── api-key/                     # API key utilities
├── atoms/                       # Atomic operations
│   └── decorators/
├── docs/                        # OpenAPI/Swagger helpers
├── enums/                       # Shared enumerations
├── inputs/                      # Common DTOs
├── modules/                     # Reusable business modules
│   ├── regular-booking.module.ts
│   ├── recurring-booking.module.ts
│   ├── instant-booking.module.ts
│   ├── available-slots.module.ts
│   └── booking-cancel.module.ts
├── pagination/                  # Pagination utilities
├── passport/                    # Passport.js setup
│   └── strategies/
├── repositories/                # Base repositories
├── response/                    # Response formatters
├── roles/                       # Role definitions
├── services/                    # Shared services
├── safe-parse/                  # Safe parsing utilities
├── is-origin-allowed/           # CORS helpers
├── api-versions.ts             # Version constants
├── throttler-guard.ts          # Rate limiting
└── logger.ts                   # Logging utilities
```

### Middleware Structure

```
middleware/
├── body/
│   ├── json.body.middleware.ts     # JSON parsing
│   └── raw.body.middleware.ts      # Raw body (webhooks)
│
├── request-ids/
│   ├── request-id.middleware.ts    # ID generation
│   └── request-id.interceptor.ts   # ID injection
│
├── app.logger.middleware.ts        # Request logging
├── app.redirects.middleware.ts     # Redirect handling
└── app.rewrites.middleware.ts      # URL rewriting
```

### Exception Filters

```
filters/
├── http-exception.filter.ts        # Generic HTTP errors
├── prisma-exception.filter.ts      # Database errors
├── zod-exception.filter.ts         # Validation errors
├── trpc-exception.filter.ts        # tRPC errors
└── calendar-service-exception.filter.ts  # Calendar API errors
```

### Standard Module Pattern

Every domain module follows this consistent structure:

```
module-name/
├── controllers/                     # HTTP endpoints
│   ├── module-name.controller.ts
│   └── e2e/                        # E2E test specs
│       ├── feature-1.e2e-spec.ts
│       └── feature-2.e2e-spec.ts
│
├── services/                        # Business logic
│   ├── module-name.service.ts      # Main service
│   ├── input.service.ts            # Input transformation
│   ├── output.service.ts           # Output transformation
│   └── errors.service.ts           # Error handling
│
├── repositories/                    # Data access (optional)
│   └── module-name.repository.ts
│
├── guards/                          # Custom guards (optional)
│   └── custom-guard.ts
│
├── inputs/                          # Request DTOs
│   ├── create-entity.input.ts
│   ├── update-entity.input.ts
│   └── get-entities.input.ts
│
├── outputs/                         # Response DTOs
│   ├── entity.output.ts
│   └── entities.output.ts
│
├── validators/                      # Custom validators (optional)
│   └── custom-validator.ts
│
├── pipes/                          # Custom pipes (optional)
│   └── transform.pipe.ts
│
└── module-name.module.ts           # Module definition
```

---

## Data Flow

### Complete Request Processing Pipeline

```
1. HTTP Request Arrives
   ├── Method: POST /v2/bookings
   ├── Headers: { "cal-api-version": "2024-08-13", "Authorization": "..." }
   └── Body: { eventTypeId, start, end, ... }

2. Middleware Chain Execution
   ├── RawBodyMiddleware
   │   └── Skip (only for /billing/webhook)
   ├── JsonBodyMiddleware
   │   └── Parse JSON → req.body
   ├── RequestIdMiddleware
   │   └── Generate UUID → req.id
   ├── AppLoggerMiddleware
   │   └── Log request details
   ├── RedirectsMiddleware
   │   └── Handle redirects (if applicable)
   └── RewriterMiddleware
       └── Rewrite URLs (if applicable)

3. Global Interceptors & Guards
   ├── CustomThrottlerGuard
   │   ├── Check rate limit in Redis
   │   ├── Key: throttler:${userId}:${endpoint}
   │   └── Allow or reject
   └── API Versioning
       ├── Extract version from header
       └── Route to correct controller version

4. Controller-Level Guards
   ├── ApiAuthGuard / OptionalApiAuthGuard
   │   ├── Extract credentials
   │   ├── Validate against database
   │   ├── Load user & permissions
   │   └── Attach to request
   ├── PermissionsGuard
   │   ├── Check required permissions
   │   └── Allow or reject
   ├── PlatformPlanGuard
   │   ├── Check billing plan
   │   └── Allow or reject
   └── Domain Guards
       └── BookingUidGuard, TeamGuard, etc.

5. Controller Method Execution
   ├── Parameter Extraction
   │   ├── @Param() → URL parameters
   │   ├── @Query() → Query strings
   │   ├── @Body() → Request body
   │   └── @GetUser() → Authenticated user
   ├── Validation Pipeline
   │   ├── class-validator decorators
   │   ├── Custom validators
   │   └── Throw BadRequestException if fails
   └── Pipe Transformations
       └── CreateBookingInputPipe

6. Service Layer Processing
   ├── Input Service
   │   ├── Transform request DTO
   │   ├── Validate business rules
   │   └── Prepare for processing
   ├── Main Service
   │   ├── Orchestrate business logic
   │   ├── Call repositories
   │   ├── Call external services
   │   └── Handle transactions
   └── Output Service
       ├── Transform domain models
       └── Prepare response DTO

7. Repository Layer
   ├── Query Building
   │   └── Prisma query composition
   ├── Database Execution
   │   ├── PrismaReadService (SELECT)
   │   └── PrismaWriteService (INSERT/UPDATE/DELETE)
   └── Result Mapping
       └── Transform to domain models

8. External Service Integration (if needed)
   ├── CalendarsService
   │   └── Create calendar events
   ├── EmailModule
   │   └── Send confirmation emails
   ├── WebhooksModule
   │   └── Trigger webhooks
   └── BillingModule
       └── Process payments

9. Response Preparation
   ├── Output Service Transformation
   ├── ResponseInterceptor
   │   └── Add request ID to response
   └── Serialization
       └── JSON.stringify

10. Exception Handling (if error occurs)
    ├── Domain-specific Filters
    ├── PrismaExceptionFilter (Database errors)
    ├── ZodExceptionFilter (Validation errors)
    ├── HttpExceptionFilter (HTTP errors)
    ├── TRPCExceptionFilter (tRPC errors)
    ├── CalendarServiceExceptionFilter (External API errors)
    └── SentryGlobalFilter (Last resort + logging)

11. HTTP Response Sent
    ├── Status: 200 OK
    ├── Headers: { "x-request-id": "..." }
    └── Body: { "status": "success", "data": {...} }
```

### Detailed Booking Creation Flow

```typescript
/**
 * Example: POST /v2/bookings - Create a new booking
 */

┌─────────────────────────────────────────────────────────┐
│ 1. Client Request                                        │
│    POST /v2/bookings                                     │
│    Headers: {                                            │
│      "cal-api-version": "2024-08-13",                   │
│      "Authorization": "Bearer token..."                 │
│    }                                                     │
│    Body: {                                               │
│      eventTypeId: 123,                                   │
│      start: "2024-01-01T10:00:00Z",                     │
│      end: "2024-01-01T11:00:00Z",                       │
│      attendee: { email: "..." }                          │
│    }                                                     │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 2. Middleware Processing                                 │
│    JsonBodyMiddleware: Parse body                        │
│    RequestIdMiddleware: Generate "req-abc-123"          │
│    AppLoggerMiddleware: Log request                      │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 3. Version Resolution                                    │
│    Extract "2024-08-13" from header                     │
│    Route to BookingsController_2024_08_13               │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 4. Guard Execution                                       │
│    OptionalApiAuthGuard:                                 │
│    ├── Extract Bearer token                              │
│    ├── Verify JWT signature                              │
│    ├── Load user from database                           │
│    └── Attach to request.user                            │
│    PermissionsGuard:                                     │
│    └── Check BOOKING_WRITE permission                    │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 5. Controller: createBooking()                          │
│    @Body(CreateBookingInputPipe) body                   │
│    - Validate input structure                            │
│    - Transform to CreateBookingInput DTO                 │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 6. InputBookingsService                                 │
│    - Validate business rules                             │
│    - Check required fields                               │
│    - Transform attendee data                             │
│    - Determine booking type                              │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 7. BookingsService_2024_08_13                           │
│    Determine booking type:                               │
│    ├── Regular → RegularBookingModule                    │
│    ├── Recurring → RecurringBookingModule                │
│    └── Instant → InstantBookingModule                    │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 8. Repository Operations                                 │
│    EventTypesRepository:                                 │
│    └── Fetch event type details                          │
│    UsersRepository:                                      │
│    └── Validate host availability                        │
│    BookingsRepository:                                   │
│    └── Create booking record                             │
│    (All use PrismaWriteService)                         │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 9. External Integrations (Parallel)                     │
│    CalendarsService:                                     │
│    └── Create Google Calendar event                      │
│    EmailModule:                                          │
│    └── Queue confirmation email                          │
│    WebhooksModule:                                       │
│    └── Trigger booking.created webhook                   │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 10. Billing Processing (Async)                          │
│     BookingsService.billBooking()                        │
│     └── Stripe charge via BillingModule                  │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 11. OutputBookingsService                               │
│     Transform booking entity to output DTO:              │
│     - Format dates                                       │
│     - Include event type details                         │
│     - Include attendees                                  │
│     - Include booking references                         │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 12. Response                                             │
│     Status: 200 OK                                       │
│     Headers: { "x-request-id": "req-abc-123" }          │
│     Body: {                                              │
│       "status": "success",                               │
│       "data": {                                          │
│         "id": 456,                                       │
│         "uid": "booking-xyz",                            │
│         "eventType": {...},                              │
│         "attendees": [...]                               │
│       }                                                  │
│     }                                                    │
└─────────────────────────────────────────────────────────┘
```

### Database Access Patterns

```typescript
// READ Operations
Controller
  → Service
    → Repository.findX()
      → PrismaReadService.prisma.model.findX()
        → PostgreSQL Read Connection Pool (9 connections)
          → Read Replica (if configured)

// WRITE Operations
Controller
  → Service
    → Repository.create/update/delete()
      → PrismaWriteService.prisma.model.X()
        → PostgreSQL Write Connection Pool (7 connections)
          → Primary Database

// TRANSACTIONAL Operations
Service
  → PrismaWriteService.prisma.$transaction([
      operation1,
      operation2,
      operation3
    ])
  → All operations succeed or all rollback
```

### Cache Flow with Redis

```typescript
// Guard Cache Pattern
@UseGuards(CustomGuard)
async endpoint() {
  // Guard execution:
  const cacheKey = `guard:${guardName}:${userId}:${resourceId}`;
  const cached = await redis.get(cacheKey);

  if (cached === 'true') {
    return true; // Cache hit
  }

  // Execute guard logic
  const result = await guardLogic();

  if (result === true) {
    // Only cache successful results
    await redis.set(cacheKey, 'true', 'EX', ttl);
  }

  return result;
}

// Application Cache Pattern
async getEventType(id: number) {
  const cacheKey = `event-type:${id}`;
  const cached = await redis.get(cacheKey);

  if (cached) {
    return JSON.parse(cached);
  }

  const eventType = await repository.findById(id);
  await redis.set(cacheKey, JSON.stringify(eventType), 'EX', 300);

  return eventType;
}
```

### Background Job Flow

```typescript
// Enqueue Job
Service Layer
  ↓
await queue.add('send-email', {
  to: 'user@example.com',
  template: 'booking-confirmation',
  data: bookingData
}, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 2000 }
});
  ↓
Bull Queue → Redis

// Process Job (Separate Worker)
Worker Process
  ↓
@Process('send-email')
async handleEmail(job: Job) {
  const emailService = new EmailService();
  await emailService.send(job.data);
}
  ↓
Uses PrismaWorkerModule
  ↓
Separate Connection Pool (4 read, 6 write)
```

---

## Technology Stack

### Core Framework & Language

```yaml
Runtime & Language:
  - Node.js: JavaScript runtime
  - TypeScript 5.x: Type-safe language
  - NestJS 10.x: Application framework
    Features:
      - Dependency injection
      - Modular architecture
      - Decorator-based
      - Express adapter

Build Tools:
  - nest-cli: NestJS CLI
  - ts-node: TypeScript execution
  - tsconfig-paths: Path mapping
```

### Database & ORM

```yaml
Database:
  - PostgreSQL: Primary database
    Configuration:
      - Read/write splitting
      - Connection pooling
      - Transaction support

ORM & Query Builders:
  - Prisma: Primary ORM
    Features:
      - Type-safe queries
      - Migration management
      - Multi-database support
      - Connection pooling
  - Kysely: Type-safe SQL builder
    Use case: Complex queries
  - pg (node-postgres): Connection driver
    Features:
      - Native PostgreSQL driver
      - Connection pooling
```

### Caching & Queue

```yaml
Cache & Session:
  - Redis (ioredis client)
    Use cases:
      - Rate limiting storage
      - Session storage
      - Guard result caching
      - Application caching
    Configuration:
      - TLS in production
      - Connection pooling

Background Jobs:
  - Bull: Redis-based queue
    Features:
      - Job retry logic
      - Delayed jobs
      - Job prioritization
      - Job events
```

### Authentication & Authorization

```yaml
Authentication:
  - Passport.js: Authentication middleware
  - passport-jwt: JWT strategy
  - next-auth: Session management
  - @nestjs/jwt: JWT utilities
  - @nestjs/passport: NestJS integration

OAuth & Tokens:
  - OAuth 2.0 flow implementation
  - Access token management
  - Refresh token rotation
```

### Validation & Transformation

```yaml
Request Validation:
  - class-validator: Decorator-based validation
    Features:
      - Built-in validators
      - Custom validators
      - Nested validation
  - class-transformer: Object mapping
    Features:
      - Type transformation
      - Serialization
      - Deserialization

Schema Validation:
  - Zod: Runtime type checking
    Use cases:
      - Complex validation logic
      - Schema inference
```

### HTTP & Middleware

```yaml
HTTP Server:
  - Express: Web framework
  - @nestjs/platform-express: NestJS adapter

Security:
  - Helmet: Security headers
    Features:
      - XSS protection
      - Content security policy
      - HSTS
  - CORS: Cross-origin configuration
  - cookie-parser: Cookie handling

Body Parsing:
  - body-parser: JSON/URL-encoded
  - Custom raw body middleware: Webhooks
```

### API Documentation

```yaml
Documentation:
  - @nestjs/swagger: OpenAPI generation
    Features:
      - Automatic schema generation
      - Interactive UI
      - Type-safe documentation
      - Multiple versions support
```

### Monitoring & Logging

```yaml
Logging:
  - Winston: Logging framework
    Features:
      - Multiple transports
      - Log levels
      - Structured logging
  - nest-winston: NestJS integration
  - @axiomhq/winston: Log aggregation

Error Tracking:
  - @sentry/nestjs: Error monitoring
  - @sentry/node: Core SDK
  - @sentry/profiling-node: Performance monitoring
    Features:
      - Error tracking
      - Performance monitoring
      - Release tracking
      - Breadcrumbs
```

### External Integrations

```yaml
Payment:
  - Stripe: Payment processing
    Features:
      - Payment intents
      - Subscriptions
      - Webhooks
      - Customer management

Calendar:
  - googleapis: Google Calendar API
    Features:
      - Event creation
      - Availability checking
      - Calendar sync
  - @microsoft/microsoft-graph-types-beta: Microsoft Calendar

CRM:
  - jsforce: Salesforce integration
```

### Testing

```yaml
Test Framework:
  - Jest: Unit & E2E testing
    Configuration:
      - Coverage reporting
      - Parallel execution
      - Watch mode
  - @nestjs/testing: Testing utilities
  - supertest: HTTP assertions
  - @golevelup/ts-jest: Enhanced mocking

Test Types:
  - Unit tests: Individual components
  - Integration tests: Module interactions
  - E2E tests: Full request/response cycle
```

### Development Tools

```yaml
Code Quality:
  - ESLint: Linting
  - Prettier: Code formatting
  - TypeScript: Type checking

Utilities:
  - lodash: Utility functions
  - luxon: Date/time handling
  - uuid: UUID generation
  - dotenv: Environment variables
  - concurrently: Run multiple commands
```

---

## Security Architecture

### Multi-Layer Security

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Network Security                               │
│ - CORS configuration                                     │
│ - Helmet security headers                                │
│ - TLS/SSL encryption                                     │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 2: Rate Limiting                                   │
│ - Redis-backed throttling                                │
│ - Per-user limits                                        │
│ - Per-endpoint limits                                    │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Authentication                                  │
│ - API Key validation                                     │
│ - JWT token verification                                 │
│ - OAuth 2.0 flow                                         │
│ - Session management                                     │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 4: Authorization                                   │
│ - Permission-based access                                │
│ - Role-based access control                              │
│ - Organization-scoped resources                          │
│ - Resource ownership validation                          │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 5: Input Validation                                │
│ - DTO validation                                         │
│ - Custom validators                                      │
│ - SQL injection prevention                               │
│ - XSS prevention                                         │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 6: Business Logic Security                        │
│ - Domain-specific guards                                 │
│ - Resource access validation                             │
│ - Feature gating (billing plans)                         │
└─────────────────────────────────────────────────────────┘
```

### Authentication Strategies

**1. API Key Authentication**

```typescript
// Usage
Headers: {
  "x-cal-secret-key": "cal_live_abc123..."
}

// Validation
ApiAuthGuard
  ├── Extract key from header
  ├── Validate prefix "cal_"
  ├── Lookup in database
  │   SELECT * FROM ApiKey WHERE hashedKey = hash(key)
  ├── Check expiration
  ├── Load associated user
  └── Attach permissions to request
```

**2. OAuth 2.0 Flow**

```typescript
// Token Request
POST /oauth/token
Body: {
  grant_type: "authorization_code",
  code: "...",
  client_id: "...",
  client_secret: "..."
}

// Token Usage
Headers: {
  "Authorization": "Bearer access_token..."
}

// Validation
ApiAuthGuard
  ├── Extract Bearer token
  ├── Verify JWT signature
  ├── Check expiration
  ├── Load OAuth client
  ├── Load user
  └── Apply scoped permissions
```

**3. Session Authentication**

```typescript
// Session Cookie
Cookies: {
  "next-auth.session-token": "..."
}

// Validation
NextAuthGuard
  ├── Extract session cookie
  ├── Decrypt session token
  ├── Verify signature
  ├── Check expiration
  ├── Load user from session
  └── Attach to request
```

### Authorization Model

**Permission-Based Access Control:**

```typescript
// Endpoint Declaration
@Permissions([BOOKING_READ, BOOKING_WRITE])
@UseGuards(ApiAuthGuard, PermissionsGuard)
async getBookings() { ... }

// Permission Check Flow
PermissionsGuard.canActivate()
  ├── Get required permissions from decorator
  ├── Get user permissions from request
  │   (loaded by ApiAuthGuard)
  ├── Check if user has all required permissions
  │   user.permissions.includes(BOOKING_READ) &&
  │   user.permissions.includes(BOOKING_WRITE)
  └── Allow or throw ForbiddenException
```

**Role-Based Access Control:**

```typescript
// Team/Organization Roles
enum MembershipRole {
  OWNER,
  ADMIN,
  MEMBER
}

// Role Check
@MembershipRoles([MembershipRole.OWNER, MembershipRole.ADMIN])
@UseGuards(MembershipRolesGuard)
async updateTeam() { ... }

// Validation
MembershipRolesGuard.canActivate()
  ├── Get required roles from decorator
  ├── Get user membership
  │   FROM Membership WHERE userId = ? AND teamId = ?
  ├── Check if user has required role
  └── Allow or deny
```

### Rate Limiting Strategy

```typescript
// Configuration
ThrottlerModule.forRootAsync({
  useFactory: (redis: RedisService) => ({
    throttlers: [
      {
        name: "default",
        ttl: seconds(60),
        limit: 120  // 120 requests per minute
      }
    ],
    storage: new ThrottlerStorageRedisService(redis)
  })
})

// Custom Rate Limits
@EndpointThrottle({
  short: { ttl: seconds(10), limit: 5 },
  medium: { ttl: seconds(60), limit: 30 },
  long: { ttl: seconds(3600), limit: 500 }
})

// Redis Keys
throttler:${userId}:${endpoint}:${window}
```

### Input Validation

**1. DTO Validation**

```typescript
export class CreateBookingInput {
  @IsInt()
  @IsNotEmpty()
  eventTypeId: number;

  @IsISO8601()
  @IsNotEmpty()
  start: string;

  @IsISO8601()
  @IsNotEmpty()
  end: string;

  @ValidateNested()
  @Type(() => AttendeeInput)
  attendee: AttendeeInput;
}
```

**2. Custom Validators**

```typescript
@Injectable()
export class TimezoneValidator implements ValidatorConstraintInterface {
  validate(timezone: string): boolean {
    return isValidTimezone(timezone);
  }

  defaultMessage(): string {
    return 'Invalid timezone format';
  }
}
```

**3. Prisma-Level Protection**

- Parameterized queries (SQL injection prevention)
- Type-safe query building
- Automatic escaping

### Security Best Practices Implemented

✅ **OWASP Top 10 Protection:**
- SQL Injection: Prisma parameterized queries
- XSS: Helmet CSP headers + input sanitization
- CSRF: Token validation for state-changing operations
- Authentication: Multi-strategy authentication
- Authorization: Multi-layer permission checks
- Sensitive Data Exposure: Encryption at rest & transit
- XXE: Disabled XML parsing
- Broken Access Control: Resource ownership validation
- Security Misconfiguration: Environment-based configs
- Insufficient Logging: Winston + Sentry monitoring

✅ **Additional Security Measures:**
- API key prefix validation
- JWT signature verification
- Request ID tracking
- Rate limiting per user/endpoint
- Redis cache poisoning prevention
- Database connection pooling
- Secret management via env vars
- License key validation
- Origin validation for CORS

---

## Best Practices

### 1. Module Organization

**DO:**
```typescript
// Clear module boundaries
@Module({
  imports: [PrismaModule, EventTypesModule],
  providers: [BookingsService, BookingsRepository],
  exports: [BookingsService],  // Only export what's needed
})
export class BookingsModule {}
```

**DON'T:**
```typescript
// Circular dependencies
@Module({
  imports: [UsersModule],  // UsersModule imports BookingsModule
  exports: [BookingsService],
})
export class BookingsModule {}
```

### 2. Repository Pattern

**DO:**
```typescript
@Injectable()
export class UsersRepository {
  constructor(
    private readonly dbRead: PrismaReadService,
    private readonly dbWrite: PrismaWriteService
  ) {}

  async findById(id: number): Promise<User | null> {
    return this.dbRead.prisma.user.findUnique({ where: { id } });
  }

  async create(data: CreateUserInput): Promise<User> {
    return this.dbWrite.prisma.user.create({ data });
  }
}
```

**DON'T:**
```typescript
// Direct Prisma usage in services
@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaClient) {}

  async getUser(id: number) {
    return this.prisma.user.findUnique({ where: { id } });
  }
}
```

### 3. DTO Validation

**DO:**
```typescript
export class CreateBookingInput {
  @IsInt()
  @Min(1)
  eventTypeId: number;

  @IsISO8601()
  @IsNotEmpty()
  start: string;

  @ValidateNested()
  @Type(() => AttendeeInput)
  attendee: AttendeeInput;
}
```

**DON'T:**
```typescript
// Manual validation in service
async createBooking(data: any) {
  if (!data.eventTypeId || typeof data.eventTypeId !== 'number') {
    throw new Error('Invalid eventTypeId');
  }
  // ...
}
```

### 4. Error Handling

**DO:**
```typescript
// Use exception filters
@Catch(PrismaClientKnownRequestError)
export class PrismaExceptionFilter implements ExceptionFilter {
  catch(exception: PrismaClientKnownRequestError, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse();

    if (exception.code === 'P2002') {
      return response.status(409).json({
        status: 'error',
        message: 'Duplicate entry'
      });
    }
  }
}
```

**DON'T:**
```typescript
// Try-catch everywhere
async createUser(data: CreateUserInput) {
  try {
    return await this.repository.create(data);
  } catch (error) {
    if (error.code === 'P2002') {
      throw new ConflictException('User exists');
    }
    throw error;
  }
}
```

### 5. Guard Usage

**DO:**
```typescript
@Post('/:bookingUid/cancel')
@UseGuards(BookingUidGuard, ApiAuthGuard)
@Permissions([BOOKING_WRITE])
async cancelBooking(
  @Param('bookingUid') bookingUid: string,
  @GetUser() user: User
) {
  return this.bookingsService.cancel(bookingUid, user);
}
```

**DON'T:**
```typescript
// Manual authorization checks
async cancelBooking(bookingUid: string, user: User) {
  const booking = await this.repository.findByUid(bookingUid);
  if (booking.userId !== user.id) {
    throw new ForbiddenException();
  }
  // ...
}
```

### 6. Service Layer

**DO:**
```typescript
// Thin controllers, fat services
@Controller('/bookings')
export class BookingsController {
  constructor(private readonly bookingsService: BookingsService) {}

  @Post('/')
  async create(@Body() input: CreateBookingInput, @GetUser() user: User) {
    return this.bookingsService.create(input, user);
  }
}

// Service contains business logic
@Injectable()
export class BookingsService {
  async create(input: CreateBookingInput, user: User) {
    // Validate
    // Transform
    // Orchestrate
    // Return
  }
}
```

**DON'T:**
```typescript
// Fat controllers
@Controller('/bookings')
export class BookingsController {
  @Post('/')
  async create(@Body() input: CreateBookingInput) {
    // Business logic in controller
    const eventType = await this.eventTypesRepo.find(input.eventTypeId);
    if (!eventType) throw new NotFoundException();
    // ... more logic
  }
}
```

### 7. Testing

**DO:**
```typescript
describe('BookingsService', () => {
  let service: BookingsService;
  let repository: MockType<BookingsRepository>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        BookingsService,
        {
          provide: BookingsRepository,
          useFactory: () => createMock<BookingsRepository>(),
        },
      ],
    }).compile();

    service = module.get(BookingsService);
    repository = module.get(BookingsRepository);
  });

  it('should create a booking', async () => {
    repository.create.mockResolvedValue(mockBooking);
    const result = await service.create(input);
    expect(result).toEqual(mockBooking);
  });
});
```

### 8. Environment Configuration

**DO:**
```typescript
// Centralized configuration
export default (): AppConfig => ({
  api: {
    port: Number(getEnv("API_PORT", "5555")),
    url: getEnv("API_URL", "http://localhost"),
  },
  db: {
    readUrl: getEnv("DATABASE_READ_URL"),
    writeUrl: getEnv("DATABASE_WRITE_URL"),
  },
});
```

**DON'T:**
```typescript
// Scattered process.env access
const port = process.env.API_PORT || 5555;
const dbUrl = process.env.DATABASE_URL;
```

### 9. API Versioning

**DO:**
```typescript
// Date-based versioning with clear modules
/ee/bookings/
  ├── 2024-04-15/
  │   └── bookings.controller.ts  // @Controller({ version: VERSION_2024_04_15 })
  └── 2024-08-13/
      └── bookings.controller.ts  // @Controller({ version: VERSION_2024_08_13 })
```

**DON'T:**
```typescript
// Conditional logic in same controller
@Get()
async getBooking(@Headers('version') version: string) {
  if (version === '2024-04-15') {
    return this.getBookingV1();
  } else {
    return this.getBookingV2();
  }
}
```

### 10. Logging

**DO:**
```typescript
@Injectable()
export class BookingsService {
  private readonly logger = new Logger(BookingsService.name);

  async create(input: CreateBookingInput) {
    this.logger.log('Creating booking', { eventTypeId: input.eventTypeId });

    try {
      const booking = await this.repository.create(input);
      this.logger.log('Booking created', { bookingId: booking.id });
      return booking;
    } catch (error) {
      this.logger.error('Failed to create booking', error.stack);
      throw error;
    }
  }
}
```

---

## Future Considerations

### 1. Domain-Driven Design (DDD)

**Current State:**
- Module-based organization
- Service-oriented architecture
- Repository pattern implemented

**Potential Enhancement:**
```typescript
// Introduce domain entities
export class Booking {
  private constructor(
    private readonly id: number,
    private readonly eventTypeId: number,
    private status: BookingStatus
  ) {}

  static create(data: CreateBookingData): Booking {
    // Business rules
    return new Booking(null, data.eventTypeId, BookingStatus.PENDING);
  }

  confirm(): void {
    if (this.status !== BookingStatus.PENDING) {
      throw new DomainException('Only pending bookings can be confirmed');
    }
    this.status = BookingStatus.CONFIRMED;
  }

  cancel(reason: string): BookingCancelledEvent {
    this.status = BookingStatus.CANCELLED;
    return new BookingCancelledEvent(this.id, reason);
  }
}
```

**Benefits:**
- Rich domain models
- Business logic encapsulation
- Domain events for decoupling

### 2. Event-Driven Architecture

**Current State:**
- Synchronous service calls
- Direct dependencies
- Limited event handling

**Potential Enhancement:**
```typescript
// Domain events
export class BookingCreatedEvent {
  constructor(
    public readonly bookingId: number,
    public readonly eventTypeId: number,
    public readonly userId: number
  ) {}
}

// Event handlers
@Injectable()
export class BookingCreatedHandler {
  @OnEvent('booking.created')
  async handle(event: BookingCreatedEvent) {
    await this.emailService.sendConfirmation(event.bookingId);
    await this.webhookService.trigger('booking.created', event);
    await this.analyticsService.track(event);
  }
}
```

**Benefits:**
- Loose coupling
- Easier to add features
- Better scalability
- Audit trail

### 3. CQRS Pattern

**Current State:**
- Read/write split at database level
- Combined service methods

**Potential Enhancement:**
```typescript
// Command side
export class CreateBookingCommand {
  constructor(public readonly data: CreateBookingInput) {}
}

@CommandHandler(CreateBookingCommand)
export class CreateBookingHandler {
  async execute(command: CreateBookingCommand): Promise<void> {
    // Write to database
    // Publish events
  }
}

// Query side
export class GetBookingQuery {
  constructor(public readonly bookingId: number) {}
}

@QueryHandler(GetBookingQuery)
export class GetBookingHandler {
  async execute(query: GetBookingQuery): Promise<BookingView> {
    // Read from optimized view
  }
}
```

**Benefits:**
- Optimized read/write models
- Better scalability
- Independent evolution

### 4. API Gateway Pattern

**Current State:**
- Monolithic NestJS application
- Single entry point

**Potential Enhancement:**
```
┌─────────────────────────────────────────┐
│           API Gateway                    │
│  • Routing                               │
│  • Rate limiting                         │
│  • Authentication                        │
│  • Request transformation                │
└─────────────────────────────────────────┘
           ↓          ↓          ↓
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Bookings │ │  Users   │ │ Events   │
    │ Service  │ │ Service  │ │ Service  │
    └──────────┘ └──────────┘ └──────────┘
```

**Benefits:**
- Service independence
- Easier scaling
- Technology flexibility

### 5. Circuit Breaker for External Services

**Current State:**
- Direct external service calls
- Basic error handling

**Potential Enhancement:**
```typescript
@Injectable()
export class GoogleCalendarService {
  private circuitBreaker: CircuitBreaker;

  constructor() {
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,
      resetTimeout: 60000,
      fallback: this.fallbackCalendarCreation
    });
  }

  async createEvent(data: EventData) {
    return this.circuitBreaker.execute(() =>
      this.googleCalendarApi.create(data)
    );
  }

  private async fallbackCalendarCreation(data: EventData) {
    // Queue for later processing
    await this.queue.add('create-calendar-event', data);
  }
}
```

**Benefits:**
- Improved resilience
- Graceful degradation
- Better error handling

### 6. Microservices Architecture

**Current State:**
- Modular monolith
- Shared database

**Potential Enhancement:**
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Bookings   │────▶│   Message    │◀────│    Users     │
│   Service    │     │     Bus      │     │   Service    │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    ▲                     │
       │                    │                     │
       ▼                    │                     ▼
┌──────────────┐           │             ┌──────────────┐
│  Bookings DB │           │             │   Users DB   │
└──────────────┘           │             └──────────────┘
                            │
                   ┌────────┴─────────┐
                   │   Event Types    │
                   │     Service      │
                   └──────────────────┘
```

**Benefits:**
- Independent deployment
- Technology flexibility
- Team autonomy
- Fault isolation

### 7. GraphQL API Layer

**Current State:**
- REST API only
- Multiple endpoints for related data

**Potential Enhancement:**
```typescript
@Resolver(() => Booking)
export class BookingsResolver {
  @Query(() => Booking)
  async booking(@Args('id') id: number) {
    return this.bookingsService.findById(id);
  }

  @ResolveField(() => EventType)
  async eventType(@Parent() booking: Booking) {
    return this.eventTypesService.findById(booking.eventTypeId);
  }

  @ResolveField(() => [Attendee])
  async attendees(@Parent() booking: Booking) {
    return this.attendeesService.findByBookingId(booking.id);
  }
}
```

**Benefits:**
- Flexible queries
- Reduced over-fetching
- Better client experience

### 8. Observability Enhancements

**Current State:**
- Winston logging
- Sentry error tracking
- Request ID tracking

**Potential Enhancement:**
```typescript
// OpenTelemetry integration
import { trace, context } from '@opentelemetry/api';

@Injectable()
export class BookingsService {
  async create(input: CreateBookingInput) {
    const span = trace.getTracer('bookings').startSpan('create_booking');

    try {
      span.setAttribute('event_type_id', input.eventTypeId);

      const booking = await this.repository.create(input);

      span.addEvent('booking_created', { booking_id: booking.id });
      span.setStatus({ code: SpanStatusCode.OK });

      return booking;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message
      });
      throw error;
    } finally {
      span.end();
    }
  }
}
```

**Benefits:**
- Distributed tracing
- Better performance insights
- Easier debugging

---

## Appendix

### A. Key File References

**Core Application Files:**
- Entry point: `src/main.ts:19`
- App module: `src/app.module.ts:26`
- Bootstrap: `src/app.ts:27`
- Configuration: `src/config/app.ts:5`

**Module Examples:**
- Bookings (latest): `src/ee/bookings/2024-08-13/bookings.module.ts:50`
- Users: `src/modules/users/users.module.ts:8`
- Auth: `src/modules/auth/auth.module.ts:15`

**Architecture Components:**
- Repository pattern: `src/modules/users/users.repository.ts:16`
- Guard example: `src/modules/auth/guards/api-auth/api-auth.guard.ts`
- Middleware chain: `src/app.module.ts:86-108`

### B. Environment Variables

Required configuration (see `.env.example`):
```bash
# Database
DATABASE_READ_URL=postgresql://...
DATABASE_WRITE_URL=postgresql://...
DATABASE_READ_POOL_MAX=9
DATABASE_WRITE_POOL_MAX=7

# Redis
REDIS_URL=redis://localhost:6379

# API
API_PORT=5555
API_URL=http://localhost
CALCOM_LICENSE_KEY=...

# Auth
NEXTAUTH_SECRET=...

# Stripe
STRIPE_API_KEY=...
STRIPE_WEBHOOK_SECRET=...
```

### C. Useful Commands

```bash
# Development
yarn dev                    # Start with docker
yarn dev:no-docker         # Start without docker
yarn dev:build:watch       # Watch platform packages

# Testing
yarn test                  # Unit tests
yarn test:e2e             # E2E tests
yarn test:e2e:watch       # E2E tests in watch mode

# Build
yarn build                # Production build

# Database
yarn prisma generate      # Generate Prisma client
yarn prisma migrate dev   # Run migrations
```

### D. Documentation Links

- NestJS: https://docs.nestjs.com
- Prisma: https://www.prisma.io/docs
- Passport.js: https://www.passportjs.org
- Redis: https://redis.io/docs
- Bull: https://docs.bullmq.io

---

**Document Version:** 1.0
**Last Updated:** 2025-01-21
**Maintained By:** Engineering Team
