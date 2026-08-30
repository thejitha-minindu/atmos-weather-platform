# Atmos System Architecture

## 1. Overview

Atmos is a production-oriented full-stack weather intelligence platform designed using a client-server architecture with separate frontend and backend applications.

The frontend is responsible for presentation and user interaction, while the backend handles business logic, external weather provider communication, authentication, persistence, caching, validation, rate limiting, error handling, and logging.

The architecture is intentionally designed to demonstrate production-oriented software engineering practices while remaining realistic for a portfolio project.

---

## 2. Architecture Goals

The architecture is designed around the following goals:

* Maintain clear separation between frontend and backend responsibilities.
* Avoid direct dependencies between the frontend and external weather providers.
* Minimize unnecessary external API requests through caching.
* Persist user-specific information reliably.
* Protect backend APIs against invalid input and excessive requests.
* Handle external provider failures gracefully.
* Keep components modular and testable.
* Support automated testing and deployment.
* Allow external services to be replaced with minimal impact on the rest of the system.
* Maintain an architecture that is realistic for a single application without unnecessary complexity.

---

## 3. High-Level Architecture

The primary system architecture is:

```text
                         ┌──────────────────────┐
                         │        User          │
                         │   Browser / Mobile   │
                         └──────────┬───────────┘
                                    │
                                  HTTPS
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │      Next.js         │
                         │      Frontend        │
                         │                      │
                         │ React + TypeScript   │
                         └──────────┬───────────┘
                                    │
                              REST / JSON
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │      NestJS          │
                         │      Backend         │
                         │                      │
                         │ Authentication       │
                         │ Weather Service      │
                         │ Geocoding            │
                         │ Favorites            │
                         │ Search History       │
                         │ Preferences          │
                         │ Validation           │
                         │ Rate Limiting        │
                         │ Logging              │
                         └──────┬───────┬───────┘
                                │       │
                    ┌───────────┘       └────────────┐
                    │                                │
                    ▼                                ▼
             ┌──────────────┐                 ┌──────────────┐
             │    Redis     │                 │ PostgreSQL   │
             │              │                 │              │
             │ Weather      │                 │ Users        │
             │ Cache        │                 │ Favorites    │
             │ Geocoding    │                 │ History      │
             │ Cache        │                 │ Preferences  │
             │ Rate Limits  │                 │              │
             └──────┬───────┘                 └──────────────┘
                    │
                    │ Cache Miss
                    ▼
             ┌────────────────┐
             │   Open-Meteo   │
             │                │
             │ Forecast API   │
             │ Geocoding API  │
             └────────────────┘
```

The frontend does not communicate directly with Open-Meteo.

All weather-related requests pass through the Atmos backend.

---

## 4. Technology Stack

### Frontend

* Next.js
* React
* TypeScript
* Tailwind CSS
* shadcn/ui
* TanStack Query
* Recharts
* Lucide

### Backend

* NestJS
* TypeScript
* REST
* Swagger / OpenAPI

### Database

* PostgreSQL
* Prisma ORM

### Cache

* Redis

### External APIs

* Open-Meteo Forecast API
* Open-Meteo Geocoding API

### Development and DevOps

* Git
* GitHub
* Docker
* Docker Compose
* GitHub Actions

---

## 5. Frontend Architecture

The frontend is implemented using Next.js and React with TypeScript.

Its primary responsibility is presentation and user interaction.

### Responsibilities

The frontend handles:

* Location search interface
* Current weather presentation
* Hourly forecast presentation
* Daily forecast presentation
* Weather charts and visualizations
* Favorite locations interface
* Search history interface
* Authentication interface
* User preference management
* Responsive layouts
* Dark/light mode
* Loading states
* Empty states
* Error states
* Client-side server-state management

The frontend should contain minimal business logic.

Business rules, persistence, authentication decisions, caching, external API communication, and rate limiting belong to the backend.

### Frontend Data Flow

```text
User Interaction
       │
       ▼
React Component
       │
       ▼
TanStack Query
       │
       ▼
Atmos REST API
       │
       ▼
Response
       │
       ▼
UI Update
```

TanStack Query provides client-side management of server state and reduces unnecessary duplicate requests from the frontend.

---

## 6. Backend Architecture

The backend is implemented using NestJS and TypeScript.

The backend acts as the central application layer between the frontend, persistent storage, cache, and external services.

### Responsibilities

The backend handles:

* REST API endpoints
* Authentication
* Authorization
* Input validation
* Weather service orchestration
* Geocoding
* Open-Meteo communication
* External provider abstraction
* Redis caching
* PostgreSQL persistence
* Rate limiting
* Error handling
* Structured logging
* Health checks
* API documentation

### General Backend Request Flow

```text
HTTP Request
     │
     ▼
Rate Limiting
     │
     ▼
Authentication / Authorization
     │
     ▼
Input Validation
     │
     ▼
Controller
     │
     ▼
Service
     │
     ├──── Database
     │
     ├──── Redis
     │
     └──── External Provider
     │
     ▼
Response Mapping
     │
     ▼
HTTP Response
```

Not every request requires every step.

For example, public weather requests do not require authentication, while requests for user favorites do.

---

## 7. Backend Module Structure

The backend will follow NestJS's modular architecture.

The planned structure is:

```text
apps/api/src/

├── auth/
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── auth.module.ts
│   └── dto/
│
├── users/
│
├── weather/
│   ├── weather.controller.ts
│   ├── weather.service.ts
│   ├── weather.module.ts
│   ├── dto/
│   ├── interfaces/
│   └── providers/
│       └── open-meteo.provider.ts
│
├── geocoding/
│
├── favorites/
│
├── history/
│
├── preferences/
│
├── cache/
│
├── health/
│
├── common/
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
│
└── main.ts
```

Each module should own a specific area of application functionality.

---

## 8. External Weather Provider Architecture

Atmos initially uses Open-Meteo for weather and geocoding data.

The frontend must never depend directly on Open-Meteo.

The communication path is:

```text
Frontend
    │
    ▼
Atmos Backend
    │
    ▼
Weather Service
    │
    ▼
Weather Provider
    │
    ▼
Open-Meteo Provider
    │
    ▼
Open-Meteo API
```

This isolates third-party integration logic from the rest of the application.

---

## 9. Provider Abstraction

The core Weather Service should not depend directly on Open-Meteo-specific implementation details.

Instead, external weather providers should implement a common provider contract.

Conceptually:

```typescript
interface WeatherProvider {
  getWeather(
    latitude: number,
    longitude: number,
  ): Promise<WeatherData>;
}
```

The Open-Meteo integration then implements this contract:

```text
WeatherService
      │
      ▼
WeatherProvider
      ▲
      │
OpenMeteoProvider
```

This provides several benefits:

* Open-Meteo-specific logic remains isolated.
* Provider responses can be normalized.
* Provider implementations can be mocked during testing.
* Another provider can potentially be introduced later.
* The frontend does not depend on a specific external API format.

---

## 10. Weather Domain Model

Open-Meteo responses should not be returned directly to the frontend.

The provider layer should convert external responses into Atmos domain models.

For example, an internal response may conceptually resemble:

```json
{
  "location": {
    "name": "Colombo",
    "country": "Sri Lanka",
    "latitude": 6.9271,
    "longitude": 79.8612
  },
  "current": {
    "temperature": 29.1,
    "feelsLike": 32.4,
    "humidity": 78,
    "windSpeed": 14,
    "windDirection": 220,
    "uvIndex": 6
  },
  "hourly": [],
  "daily": []
}
```

The exact domain model will be finalized when the API contract is designed.

The important architectural principle is:

```text
External Provider Format
          │
          ▼
Provider Adapter
          │
          ▼
Atmos Domain Model
          │
          ▼
Frontend
```

This reduces coupling between Atmos and Open-Meteo.

---

## 11. Weather Request Flow

A normal weather request follows this process:

```text
User requests weather
        │
        ▼
Next.js Frontend
        │
        ▼
GET /api/v1/weather
        │
        ▼
NestJS Controller
        │
        ▼
Validate coordinates
        │
        ▼
Weather Service
        │
        ▼
Normalize coordinates
        │
        ▼
Generate cache key
        │
        ▼
Check Redis
        │
   ┌────┴─────┐
   │          │
 HIT         MISS
   │          │
   ▼          ▼
Return     Weather Provider
cached         │
data           ▼
           Open-Meteo
               │
               ▼
         Provider response
               │
               ▼
         Normalize data
               │
               ▼
         Save in Redis
               │
               ▼
             Return
               │
               ▼
         Next.js Frontend
               │
               ▼
              User
```

The objective is to avoid unnecessary calls to the external provider.

---

## 12. Caching Architecture

Redis is used as the primary server-side cache.

Weather information is suitable for caching because many users may request weather information for the same location while the underlying forecast data changes relatively infrequently.

### Weather Cache Key

The planned format is:

```text
weather:{latitude}:{longitude}
```

Example:

```text
weather:6.9271:79.8612
```

Coordinates will be normalized before generating cache keys.

This prevents nearly identical coordinates from unnecessarily producing separate cache entries.

### Geocoding Cache Key

The planned format is:

```text
geocode:{normalized-query}
```

Example:

```text
geocode:colombo
```

Search strings should be normalized before cache lookup.

---

## 13. Initial Cache TTL Strategy

The initial cache policy is:

| Data              |   Initial TTL |
| ----------------- | ------------: |
| Current weather   |    10 minutes |
| Hourly forecast   | 10–15 minutes |
| Daily forecast    |    30 minutes |
| Geocoding results |      24 hours |

These values are application-level architectural decisions and may be changed after performance testing.

The application should measure cache behavior rather than assuming the initial TTL values are optimal.

---

## 14. Cache Request Flow

```text
Request
   │
   ▼
Generate Cache Key
   │
   ▼
Redis GET
   │
   ├──────── HIT ──────────► Return cached response
   │
   ▼
 MISS
   │
   ▼
External API
   │
   ▼
Normalize response
   │
   ▼
Redis SET + TTL
   │
   ▼
Return response
```

A later version of Atmos may implement stale-while-revalidate behavior.

This is considered an advanced feature rather than an initial requirement.

---

## 15. PostgreSQL Architecture

PostgreSQL stores persistent application information.

It is the source of truth for user-related data.

### Planned Persistent Data

PostgreSQL stores:

* Users
* Favorite locations
* Search history
* User preferences

Weather forecasts should not normally be permanently stored in PostgreSQL.

Weather information is temporary and should generally be cached using Redis.

---

## 16. Persistent vs Temporary Data

The basic architectural rule is:

```text
Does the data need to survive?
            │
       ┌────┴────┐
       │         │
      Yes        No
       │         │
       ▼         ▼
 PostgreSQL     Redis
```

### PostgreSQL

Use for:

* User accounts
* Favorite locations
* Search history
* User preferences

### Redis

Use for:

* Weather responses
* Geocoding responses
* Rate-limit counters
* Other temporary cached data

Redis must not be treated as the source of truth for persistent user information.

---

## 17. Planned Database Relationships

The initial conceptual model is:

```text
User
 │
 ├──────────< FavoriteLocation
 │
 ├──────────< SearchHistory
 │
 └─────────── UserPreference
```

The exact database schema will be defined separately before implementing Prisma migrations.

---

## 18. Authentication Architecture

Authentication will be introduced after the initial weather functionality is operational.

Guests will be allowed to access public weather functionality.

Authentication will be required for personalized features.

### Guest Capabilities

Guests can:

* Search locations
* View current weather
* View hourly forecasts
* View daily forecasts
* View charts
* Change temporary display settings

### Authenticated User Capabilities

Authenticated users can additionally:

* Save favorite locations
* View favorites
* Store search history
* Save preferences
* Synchronize preferences across sessions/devices

Authentication therefore exists to support meaningful user-specific functionality rather than simply being added as a portfolio feature.

---

## 19. Rate Limiting

The Atmos backend will implement API rate limiting.

The initial proposed policy is:

```text
Anonymous users:
30 requests per minute per IP

Authenticated users:
100 requests per minute per user
```

These values are starting points and may be adjusted after testing.

Redis may be used as the shared rate-limit storage mechanism.

When a client exceeds the permitted rate, the backend should return:

```text
HTTP 429 Too Many Requests
```

Rate limiting protects the backend and reduces the possibility of unnecessary external provider usage.

---

## 20. Validation

All external input must be validated before entering application services.

Examples include:

### Coordinates

```text
latitude  = -90 to 90
longitude = -180 to 180
```

### Location Searches

Search queries should be:

* Non-empty
* Length limited
* Normalized before processing

### Authentication

Email addresses, passwords, and authentication payloads must be validated.

### User Data

Favorites and preferences must be validated before persistence.

Validation should occur using backend DTOs and validation mechanisms rather than trusting frontend validation.

---

## 21. Error Handling

The backend should return predictable and structured errors.

Potential status codes include:

| Status | Purpose                                  |
| ------ | ---------------------------------------- |
| `400`  | Malformed JSON or invalid syntax         |
| `401`  | Authentication required                  |
| `403`  | Operation not permitted                  |
| `404`  | Resource not found                       |
| `409`  | Resource conflict                        |
| `422`  | Unprocessable content / validation failure |
| `429`  | Rate limit exceeded                      |
| `500`  | Unexpected internal error                |
| `502`  | Invalid/upstream provider response       |
| `503`  | Required service temporarily unavailable |
| `504`  | Upstream provider timeout                |

The backend standardizes on the RFC 7807 Problem Details specification (`application/problem+json`) with stable error codes and correlation request IDs (detailed in the API contract). A conceptual error response is:

```json
{
  "type": "https://atmos.example/problems/validation-error",
  "title": "Validation failed",
  "status": 422,
  "detail": "One or more request values are invalid.",
  "code": "VALIDATION_ERROR",
  "requestId": "req_01J6ABCDEF1234567890"
}
```

Internal implementation details, stack traces, credentials, or provider secrets must never be exposed to clients.

---

## 22. External Provider Failure Strategy

External APIs must be treated as potentially unreliable.

Possible failures include:

* Timeout
* Network failure
* Invalid response
* Provider outage
* Rate limiting
* Unexpected provider response changes

The application should:

```text
Provider Request
       │
       ▼
    Success?
    ┌──┴───┐
   Yes     No
    │       │
    ▼       ▼
 Return   Check usable
 data     cached data
             │
        ┌────┴────┐
       Yes        No
        │          │
        ▼          ▼
    Return stale   Return controlled
    data where     service error
    policy allows
```

Stale-data fallback may be introduced after the initial caching implementation.

Provider failures must be logged.

---

## 23. Logging

The backend will use structured logging.

Useful information may include:

```text
requestId
route
HTTP method
status code
response time
cache status
provider response time
provider errors
```

Example conceptual log:

```text
method=GET
route=/api/v1/weather
status=200
cache=HIT
responseTime=42ms
```

Sensitive information must not be logged.

This includes:

* Passwords
* Authentication tokens
* Cookies
* Secrets
* Private credentials

---

## 24. Observability

The project should eventually measure:

* Number of API requests
* Cache hits
* Cache misses
* Cache hit rate
* External provider requests
* Average response latency
* Provider latency
* Error rate
* Rate-limit violations

These measurements will help evaluate whether architectural features such as Redis caching actually improve the application.

Performance claims should be based on measured results rather than assumptions.

---

## 25. Health Checks

The backend exposes dedicated health endpoints:

* `GET /api/v1/health` (Liveness): Zero-dependency check to verify that the NestJS process is responsive.
* `GET /api/v1/health/ready` (Readiness): Checks whether required internal dependencies (PostgreSQL and Redis) are connected and ready to serve traffic.

Planned endpoints:

```http
GET /api/v1/health
GET /api/v1/health/ready
```

Conceptual readiness response:

```json
{
  "data": {
    "status": "ready",
    "checks": {
      "postgres": "up",
      "redis": "up"
    },
    "timestamp": "2026-08-30T08:45:12.120Z"
  }
}
```

External weather providers such as Open-Meteo are deliberately excluded from readiness checks to prevent third-party rate limits or transient network issues from affecting local service readiness probes.

---

## 26. API Architecture

Atmos uses REST for communication between the frontend and backend.

The initial API prefix is planned as:

```text
/api/v1
```

Potential resources include:

```text
/api/v1/weather
/api/v1/locations
/api/v1/auth
/api/v1/users
/api/v1/favorites
/api/v1/history
/api/v1/preferences
/api/v1/health
```

The exact API contract will be documented separately before backend implementation.

Swagger/OpenAPI will be used to document the backend API.

---

## 27. Security Principles

The application should follow these security principles:

* Never commit secrets to Git.
* Store configuration using environment variables.
* Hash passwords using an appropriate password-hashing algorithm.
* Validate all backend input.
* Configure CORS explicitly.
* Apply rate limiting.
* Use HTTPS in production.
* Protect authenticated endpoints.
* Avoid exposing internal errors.
* Avoid logging sensitive information.
* Use secure authentication token/cookie handling.
* Keep dependencies updated.

Security will be treated as part of the architecture rather than an afterthought.

---

## 28. Testing Architecture

Atmos will use multiple levels of automated testing.

### Unit Tests

Used for isolated business logic.

Examples:

* WeatherService
* CacheService
* AuthService
* GeocodingService
* FavoritesService

### Integration Tests

Used to verify interactions between components.

Examples:

```text
Backend → Redis
Backend → PostgreSQL
Service → Provider
```

### End-to-End Tests

Used to test complete user/application flows.

Examples:

```text
Search location
      ↓
Select location
      ↓
Load weather
```

and:

```text
Login
  ↓
Save favorite
  ↓
Reload
  ↓
Favorite remains
```

Caching should also have integration tests verifying cache HIT and MISS behavior.

---

## 29. Containerization

Docker will be used to provide consistent development and deployment environments.

The local development environment is expected to contain:

```text
Docker Compose

├── Frontend
├── Backend
├── PostgreSQL
└── Redis
```

The objective is to allow the application stack to eventually be started using a command similar to:

```bash
docker compose up
```

The exact container architecture may differ between development and production environments.

---

## 30. CI/CD Architecture

GitHub Actions will be used for continuous integration and deployment automation.

The planned pipeline is:

```text
Git Push / Pull Request
          │
          ▼
Install Dependencies
          │
          ▼
Lint
          │
          ▼
Type Check
          │
          ▼
Unit Tests
          │
          ▼
Integration Tests
          │
          ▼
Build
          │
          ▼
Deployment
```

Deployment should occur only after the required validation stages succeed.

The exact deployment process will depend on the selected hosting platform.

---

## 31. Repository Architecture

The planned repository structure is:

```text
atmos-weather-platform/

├── apps/
│   ├── web/
│   └── api/
│
├── packages/
│   └── shared/
│
├── docs/
│   ├── architecture.md
│   ├── features.md
│   └── adr/
│
├── infrastructure/
│
├── .github/
│   └── workflows/
│
├── docker-compose.yml
├── README.md
├── .gitignore
└── LICENSE
```

The exact structure may evolve as implementation requirements become clearer.

Major structural changes should be documented when they represent meaningful architectural decisions.

---

## 32. Architectural Decision Records

Important architectural decisions will be documented using Architecture Decision Records (ADRs).

ADRs are stored under:

```text
docs/adr/
```

Examples may include:

```text
ADR-001-separate-frontend-backend.md
ADR-002-use-postgresql.md
ADR-003-use-redis.md
ADR-004-use-rest.md
ADR-005-use-open-meteo.md
```

Each ADR should generally contain:

* Context
* Decision
* Alternatives considered
* Positive consequences
* Negative consequences

ADRs document why architectural decisions were made rather than only describing the final architecture.

---

## 33. Architectural Principles

The project will follow the following principles throughout development.

### 33.1 Separate Presentation from Business Logic

Next.js is primarily responsible for presentation and user interaction.

NestJS is responsible for application and integration logic.

### 33.2 Keep External Providers Behind the Backend

The frontend should not depend directly on Open-Meteo.

### 33.3 Normalize External Data

Third-party response structures should be converted into Atmos domain models.

### 33.4 Separate Persistent and Temporary Data

Persistent application data belongs in PostgreSQL.

Temporary performance-oriented data belongs in Redis.

### 33.5 Cache Expensive or Repeated Operations

Frequently requested weather and geocoding information should be cached where appropriate.

### 33.6 Validate at System Boundaries

Input received from users or external services should not automatically be trusted.

### 33.7 Measure Performance Improvements

Caching and optimization decisions should be evaluated using measurable results.

### 33.8 Design for Failure

External services, databases, caches, and networks may fail.

The application should handle expected failures predictably.

### 33.9 Prefer Simplicity

Technologies should only be introduced when they solve a real project requirement.

Atmos will not introduce microservices, Kubernetes, message queues, or similar infrastructure without a justified need.

### 33.10 Keep the Architecture Testable

Business logic should remain sufficiently separated from frameworks and external providers to allow meaningful automated testing.

---

## 34. Out-of-Scope Architecture

The initial Atmos architecture deliberately does not include:

* Microservices
* Kubernetes
* Kafka
* RabbitMQ
* GraphQL
* Machine-learning weather forecasting
* Custom meteorological models
* Native Android application
* Native iOS application

These technologies would increase project complexity without providing sufficient value for the current project goals.

---

## 35. Future Architectural Possibilities

After the core application is complete, possible improvements include:

* Stale-while-revalidate caching
* Multiple weather providers
* Weather-model comparison
* Performance analytics dashboard
* Distributed rate limiting
* PWA functionality
* Offline support
* Advanced observability
* Load testing
* Improved resilience strategies

These should only be considered after the core architecture is implemented, tested, deployed, and documented.

---

## 36. Architecture Summary

Atmos follows the primary request path:

```text
User
 ↓
Next.js
 ↓
NestJS
 ↓
Application Services
 ↓
┌───────────────┬─────────────────┐
│               │                 │
▼               ▼                 ▼
Redis        PostgreSQL      Weather Provider
                                  │
                                  ▼
                              Open-Meteo
```

The major architectural boundaries are:

```text
Presentation
     ↓
Next.js

Application / Business Logic
     ↓
NestJS

Persistent Data
     ↓
PostgreSQL

Temporary / Cached Data
     ↓
Redis

External Weather Data
     ↓
Open-Meteo
```

This architecture provides enough complexity to demonstrate full-stack software engineering, backend architecture, persistence, caching, security, testing, DevOps, and observability while remaining achievable as an undergraduate portfolio project.
