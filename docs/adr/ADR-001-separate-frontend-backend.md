# ADR-001: Separate Frontend and Backend Applications

## Status

Accepted

## Context

Atmos is a production-oriented full-stack weather intelligence platform.

A simple weather application could allow the frontend to communicate directly with a third-party weather API such as Open-Meteo. This would be sufficient for retrieving and displaying weather information, but Atmos is intended to support functionality beyond basic weather-data retrieval.

The project requires or plans to support:

* Backend REST API design
* Authentication and authorization
* Persistent user data
* Favorite locations
* Search history
* User preferences
* Redis caching
* Rate limiting
* Input validation
* Centralized error handling
* Structured logging
* Performance monitoring
* Health checks
* External weather provider abstraction
* Automated testing

If the Next.js frontend communicated directly with Open-Meteo, third-party integration logic would become coupled to the client application.

It would also make several cross-cutting concerns more difficult to manage consistently, including:

* Server-side caching
* Rate limiting
* Authentication
* Authorization
* Request validation
* Logging
* Monitoring
* Provider failure handling
* Response normalization
* External provider replacement

The architecture therefore requires a decision about whether Atmos should:

1. Communicate with Open-Meteo directly from the frontend.
2. Use Next.js server functionality as the application backend.
3. Maintain a separate standalone backend application.

---

## Decision

Atmos will use separate frontend and backend applications.

The frontend will be implemented using Next.js.

The backend will be implemented using NestJS.

The frontend will communicate only with the Atmos REST API for application data that requires backend processing.

The NestJS backend will be responsible for communicating with external weather services such as Open-Meteo.

The primary architecture will be:

```text
User
  │
  ▼
Next.js Frontend
  │
  │ HTTPS / REST
  ▼
NestJS Backend
  │
  ├──────────────► Redis
  │
  ├──────────────► PostgreSQL
  │
  └──────────────► Open-Meteo
```

For weather requests, the typical flow will be:

```text
User
  │
  ▼
Next.js Frontend
  │
  ▼
Atmos REST API
  │
  ▼
NestJS Controller
  │
  ▼
Weather Service
  │
  ▼
Redis Cache
  │
  ├──── HIT ────► Return cached data
  │
  └──── MISS
         │
         ▼
  Weather Provider
         │
         ▼
  OpenMeteoProvider
         │
         ▼
     Open-Meteo
```

The frontend should not depend directly on the Open-Meteo response format.

The backend will retrieve external weather information and transform it into Atmos-specific domain models before returning it to the frontend.

---

## Responsibilities

### Next.js Frontend

The frontend will primarily be responsible for:

* User interface
* User interaction
* Location search interface
* Weather presentation
* Weather charts
* Authentication interface
* Favorite locations interface
* Search history interface
* User preferences interface
* Responsive design
* Dark/light mode
* Loading states
* Error states
* Client-side server-state management

The frontend should contain minimal application business logic.

---

### NestJS Backend

The backend will primarily be responsible for:

* REST API endpoints
* Business logic
* Authentication
* Authorization
* Input validation
* Weather service orchestration
* Geocoding
* External API communication
* Weather provider abstraction
* Response normalization
* Redis caching
* PostgreSQL persistence
* Rate limiting
* Error handling
* Structured logging
* Health checks
* API documentation

---

## Provider Boundary

The backend should not tightly couple core application services to Open-Meteo.

Instead, Atmos will introduce a provider abstraction.

Conceptually:

```text
WeatherService
      │
      ▼
WeatherProvider
      ▲
      │
OpenMeteoProvider
```

A conceptual provider interface may resemble:

```typescript
interface WeatherProvider {
  getWeather(
    latitude: number,
    longitude: number,
  ): Promise<WeatherData>;
}
```

The exact interface will be finalized during implementation.

This means that `WeatherService` should depend on the provider contract rather than directly depending on Open-Meteo implementation details.

---

## Data Flow

External provider data should follow this flow:

```text
Open-Meteo
    │
    ▼
OpenMeteoProvider
    │
    ▼
Response Normalization
    │
    ▼
Atmos Domain Model
    │
    ▼
WeatherService
    │
    ▼
Controller
    │
    ▼
Frontend
```

Raw Open-Meteo responses should not normally be exposed directly to frontend components.

This creates a boundary between Atmos and its external provider.

---

## Positive Consequences

### Centralized External API Integration

All communication with Open-Meteo is handled by the backend.

Provider-specific behavior is not distributed throughout frontend components.

### Server-Side Caching

The backend can use Redis to cache weather and geocoding information.

Multiple clients requesting the same location can benefit from the same cached information.

### Centralized Rate Limiting

Requests can be rate-limited consistently through the backend.

This protects Atmos infrastructure and reduces unnecessary external provider requests.

### Authentication and Authorization

Authenticated functionality can be controlled centrally.

Examples include:

* Favorite locations
* Search history
* User preferences

### Persistent User Data

The backend can manage PostgreSQL persistence independently from the frontend.

### Response Normalization

Open-Meteo responses can be transformed into Atmos domain models before being returned to clients.

### Provider Replacement

If another weather provider is introduced later, much of the application can remain unchanged.

For example:

```text
Current:

WeatherService
      ↓
OpenMeteoProvider


Possible Future:

WeatherService
      ↓
WeatherProvider
      ↓
AnotherWeatherProvider
```

### Centralized Error Handling

External provider failures can be converted into predictable application errors.

### Better Observability

The backend provides a central location for measuring:

* Request counts
* Response times
* Cache hits
* Cache misses
* Provider requests
* Provider latency
* Provider failures
* Rate-limit violations

### Improved Testability

Provider implementations can be mocked when testing business logic.

### Independent Backend Skills

Using a standalone backend allows the project to demonstrate dedicated backend architecture, REST API development, persistence, caching, security, and testing.

---

## Negative Consequences

### Increased Complexity

The system contains two independently running applications rather than a single Next.js application.

### Additional Deployment Requirements

Both the frontend and backend must be deployed and configured.

### Additional Infrastructure

The backend also depends on services such as:

* PostgreSQL
* Redis

### Additional Network Request

Frontend requests must travel through the backend before reaching external weather services when cached information is unavailable.

This introduces some additional latency.

### API Contract Coordination

The frontend and backend must agree on request and response formats.

Changes to API contracts must be managed carefully.

### More Development Work

Authentication, caching, validation, logging, and API design must be explicitly implemented rather than relying entirely on frontend framework functionality.

---

## Alternatives Considered

### Alternative 1: Direct Open-Meteo Calls from the Frontend

Architecture:

```text
User
  ↓
Next.js
  ↓
Open-Meteo
```

#### Advantages

* Simple architecture
* Faster initial development
* Fewer services to deploy
* Less backend code

#### Disadvantages

* External API logic becomes coupled to the frontend.
* Centralized Redis caching becomes more difficult.
* Centralized rate limiting becomes more difficult.
* Provider abstraction is weaker.
* Logging and monitoring are more limited.
* Backend engineering opportunities are reduced.
* Provider response structures may leak into UI components.

#### Decision

Rejected.

This approach would be sufficient for a basic weather application but does not satisfy the engineering objectives of Atmos.

---

### Alternative 2: Next.js Route Handlers as the Backend

Architecture:

```text
User
  ↓
Next.js
  ↓
Next.js Route Handlers
  ↓
Redis / PostgreSQL / Open-Meteo
```

#### Advantages

* Single application
* Simpler deployment
* Less infrastructure complexity
* Server-side logic remains possible
* Can still support authentication and persistence

#### Disadvantages

* Frontend and backend application concerns remain more closely coupled.
* The project would provide less experience designing a standalone backend API.
* Independent backend deployment and scaling would be less explicit.
* The architectural boundary between frontend and backend would be less pronounced.

#### Decision

Not selected.

This is a technically valid architecture and may be preferable for some production applications.

However, Atmos intentionally uses a standalone NestJS backend because the project also aims to demonstrate dedicated backend architecture and API-development skills.

---

### Alternative 3: Microservices

Possible architecture:

```text
Frontend
   │
   ├── Weather Service
   ├── Authentication Service
   ├── User Service
   └── Geocoding Service
```

#### Advantages

* Strong service separation
* Independent deployment
* Independent scaling

#### Disadvantages

* Significant infrastructure complexity
* Distributed-system concerns
* More deployment overhead
* More difficult local development
* More complicated observability
* Unnecessary for the expected application scale

#### Decision

Rejected.

A modular monolithic NestJS backend provides sufficient separation for Atmos without unnecessary distributed-system complexity.

---

## Consequences for Future Development

All future frontend weather functionality should communicate with the Atmos backend instead of communicating directly with Open-Meteo.

Provider-specific implementation details should remain inside the provider layer.

The following dependency direction should be preserved:

```text
Frontend
   │
   ▼
Atmos API
   │
   ▼
Application Services
   │
   ▼
Provider Abstraction
   │
   ▼
External Provider
```

The application should avoid introducing dependencies in the opposite direction.

For example, frontend components should not import or depend on Open-Meteo-specific response structures.

---

## When This Decision Should Be Revisited

This decision may be reconsidered if:

* The project requirements change significantly.
* Maintaining a standalone backend creates unjustified operational complexity.
* The application becomes primarily static/client-side.
* A different backend architecture provides a measurable advantage.
* Deployment constraints make the current architecture impractical.

The decision should not be changed solely to introduce a newer technology.

Any significant replacement of this architecture should be documented in a new ADR that supersedes this decision.

---

## Summary

Atmos will use:

```text
Next.js
   │
   │ REST
   ▼
NestJS
   │
   ├── Redis
   ├── PostgreSQL
   └── WeatherProvider
          │
          ▼
      Open-Meteo
```

The primary reason for this decision is to create a clear separation between presentation and application logic while providing centralized caching, persistence, authentication, validation, rate limiting, error handling, observability, and external provider abstraction.

The additional complexity of maintaining separate frontend and backend applications is accepted because these capabilities are important to the engineering objectives of Atmos.
