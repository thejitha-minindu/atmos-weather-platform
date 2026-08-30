# Atmos REST API Contract

**Status:** Accepted design baseline  
**API version:** v1  
**Base path:** `/api/v1`  
**Default media type:** `application/json`  

This document is the implementation contract between the Atmos Next.js frontend and NestJS backend. It describes the first planned API version. It is a design baseline, not a claim that these endpoints are already implemented.

---

## 1. REST and HTTP Rules Used by Atmos

### Resources and URLs

URLs identify resources; HTTP methods describe the action. Atmos therefore uses resource-oriented URLs such as `/favorites/{favoriteId}`, not action-oriented URLs such as `/deleteFavorite`.

### HTTP method semantics

| Method | Meaning in Atmos | Safe | Idempotent |
| --- | --- | --- | --- |
| `GET` | Read data without intentionally changing application state | Yes | Yes |
| `POST` | Create a resource or initiate a non-idempotent operation | No | Usually no |
| `PATCH` | Partially update an existing resource | No | Designed to be idempotent in Atmos |
| `DELETE` | Remove a resource | No | Yes |

`GET /weather` does not create search-history records. An authenticated client records a selected location separately using `POST /history`. This preserves the safe semantics of `GET` and prevents browser retries, crawlers, or prefetching from creating user data.

### Status codes

| Status | Use |
| --- | --- |
| `200 OK` | Successful read, update, login, token refresh, or logout |
| `201 Created` | Resource successfully created |
| `204 No Content` | Resource successfully deleted and no body is needed |
| `400 Bad Request` | Malformed JSON, invalid syntax, or incompatible parameters |
| `401 Unauthorized` | Authentication is missing, invalid, or expired |
| `403 Forbidden` | Authentication is valid but the action is not allowed |
| `404 Not Found` | The requested user-owned resource does not exist |
| `409 Conflict` | The request conflicts with existing state, such as an existing email or favorite |
| `422 Unprocessable Content` | The request shape is valid but one or more values fail validation |
| `429 Too Many Requests` | A rate limit was exceeded |
| `500 Internal Server Error` | An unexpected Atmos error occurred |
| `502 Bad Gateway` | An upstream weather/geocoding provider failed or returned an unusable response |
| `503 Service Unavailable` | A required dependency is unavailable |
| `504 Gateway Timeout` | An upstream provider did not respond before the Atmos timeout |

### Idempotency

- Repeating the same `GET` has no intended data side effect.
- Repeating `PATCH /preferences` with the same body results in the same stored preferences.
- Repeating a `DELETE` leaves the resource absent. A first successful delete returns `204`; a later request may return `404`.
- Duplicate favorites are prevented by a database uniqueness rule and return `409`.

### JSON naming and nullability

- JSON property names use `camelCase`.
- Database column names and external-provider field names do not leak into the public contract.
- A property is omitted only when the contract says it is optional.
- `null` means the value is known to be unavailable or not applicable.
- Empty collections are returned as `[]`, not `null`.

### Time and dates

- Machine timestamps use ISO 8601 strings with an explicit UTC offset.
- Audit timestamps such as `createdAt` and `updatedAt` are returned in UTC with `Z`.
- Forecast timestamps include the requested location's UTC offset so that hourly and daily values can be rendered correctly.
- Daily `date` values use `YYYY-MM-DD` in the location's local calendar.

### Pagination

User collections use cursor pagination:

```json
{
  "data": [],
  "pagination": {
    "nextCursor": null,
    "hasMore": false
  }
}
```

Clients pass the opaque returned cursor using `?cursor=...`. They must not parse or construct cursor values.

### API versioning

The major API version is part of the URL: `/api/v1`.

- Backward-compatible additions, such as a new optional response field, remain in `v1`.
- Breaking changes, such as renaming or removing a field or changing its meaning, require `/api/v2`.
- Fixing a response that violates this documented contract is not considered a new version.

---

## 2. Authentication Model

Atmos uses short-lived JWT access tokens for API authorization and a longer-lived refresh token for renewing access.

- The access token is returned in the registration, login, and refresh response.
- The frontend keeps the access token in memory and sends it as `Authorization: Bearer <accessToken>`.
- The access token should not be stored in `localStorage`.
- The refresh token is stored only in a `Secure`, `HttpOnly` cookie set by the backend.
- Refresh and logout requests include cookies using `credentials: "include"`.
- Production CORS must use an explicit frontend origin and allow credentials; it must not use `*` with credentials.
- Refresh-token rotation and revocation details will be finalized with the Week 5 authentication threat model.

For endpoints marked **Optional authentication**, the request succeeds without a token. If a token is supplied, it must be valid; an invalid supplied token returns `401`.

### Authentication error distinction

- `401`: the server cannot establish a valid identity.
- `403`: the server knows the identity but the identity lacks permission.
- User-owned resources that do not belong to the current user return `404`, avoiding disclosure that another user's resource exists.

---

## 3. Cross-Cutting Response Rules

### Success response envelopes

Single-resource and aggregate endpoints return a top-level `data` property:

```json
{
  "data": {}
}
```

Collection endpoints return `data` as an array and include `pagination` when the collection is paginated.

No success envelope is returned with `204 No Content`.

### Error response format

Errors use `application/problem+json` and follow the Problem Details structure with Atmos extensions:

```json
{
  "type": "https://atmos.example/problems/validation-error",
  "title": "Validation failed",
  "status": 422,
  "detail": "One or more request values are invalid.",
  "instance": "/api/v1/weather?latitude=120&longitude=79.8612",
  "code": "VALIDATION_ERROR",
  "errors": [
    {
      "field": "latitude",
      "message": "latitude must be between -90 and 90",
      "code": "OUT_OF_RANGE"
    }
  ],
  "requestId": "req_01J6ABCDEF1234567890",
  "timestamp": "2026-08-30T08:45:12.120Z"
}
```

Rules:

- `type`, `title`, `status`, `detail`, `instance`, `code`, `requestId`, and `timestamp` are always present.
- `errors` is present only when field-level details are useful.
- Internal exception messages, stack traces, SQL details, Redis details, and raw provider responses are never returned to clients.
- The same `requestId` is included in structured server logs.

Initial stable error codes include:

| Code | Typical status |
| --- | ---: |
| `MALFORMED_REQUEST` | 400 |
| `VALIDATION_ERROR` | 422 |
| `INVALID_CREDENTIALS` | 401 |
| `TOKEN_EXPIRED` | 401 |
| `FORBIDDEN` | 403 |
| `RESOURCE_NOT_FOUND` | 404 |
| `EMAIL_ALREADY_EXISTS` | 409 |
| `FAVORITE_ALREADY_EXISTS` | 409 |
| `RATE_LIMIT_EXCEEDED` | 429 |
| `WEATHER_PROVIDER_ERROR` | 502 |
| `GEOCODING_PROVIDER_ERROR` | 502 |
| `PROVIDER_TIMEOUT` | 504 |
| `INTERNAL_ERROR` | 500 |

### Rate-limit headers

Rate-limited endpoints should return these headers when the rate-limiting implementation is added:

```text
RateLimit-Limit: 60
RateLimit-Remaining: 42
RateLimit-Reset: 1756543800
```

A `429` response also includes `Retry-After` in seconds. Exact quotas are intentionally not fixed in this initial contract; they must be configured and tested during the rate-limiting milestone (Week 7).

### Caching headers

The Redis cache is an internal implementation detail. The API may expose a diagnostic `X-Cache: HIT|MISS` header in non-production environments for testing, but clients must not depend on it. Browser/CDN cache policy will be finalized during performance work.

---

## 4. Endpoint Summary

| Method | Route | Access | Purpose |
| --- | --- | --- | --- |
| `GET` | `/locations/search` | Public | Resolve a text query to locations |
| `GET` | `/weather` | Public | Read normalized current, hourly, and daily weather |
| `POST` | `/auth/register` | Public | Create an account |
| `POST` | `/auth/login` | Public | Authenticate |
| `POST` | `/auth/refresh` | Refresh cookie | Rotate/renew authentication |
| `POST` | `/auth/logout` | Refresh cookie | Revoke the current refresh session |
| `GET` | `/users/me` | Authenticated | Read the current user |
| `GET` | `/favorites` | Authenticated | List favorites |
| `POST` | `/favorites` | Authenticated | Create a favorite |
| `DELETE` | `/favorites/{favoriteId}` | Authenticated | Delete a favorite |
| `GET` | `/history` | Authenticated | List search history |
| `POST` | `/history` | Authenticated | Record a selected location |
| `DELETE` | `/history/{historyId}` | Authenticated | Delete one history entry |
| `DELETE` | `/history` | Authenticated | Clear all history |
| `GET` | `/preferences` | Authenticated | Read saved preferences |
| `PATCH` | `/preferences` | Authenticated | Partially update preferences |
| `GET` | `/health` | Public | Check service liveness |
| `GET` | `/health/ready` | Public | Check readiness of required dependencies |

---

## 5. Shared Domain Types

### Units

```typescript
type UnitsSystem = 'metric' | 'imperial';

interface WeatherUnits {
  temperature: '°C' | '°F';
  windSpeed: 'km/h' | 'mph';
  precipitation: 'mm' | 'in';
}
```

Metric is the default for guest requests. The frontend supplies the registered user's saved unit system when requesting weather; the weather endpoint does not read preferences implicitly. This keeps public weather responses deterministic and cacheable by explicit request parameters.

### Location

```typescript
interface Location {
  externalId: string;
  name: string;
  admin1: string | null;
  country: string;
  countryCode: string;
  latitude: number;
  longitude: number;
  timezone: string;
}
```

`externalId` is an opaque provider-derived identifier used for correlation and deduplication. Clients must not interpret it.

### Weather condition

```typescript
interface WeatherCondition {
  code: number;
  key: string;
  label: string;
}
```

`code` is an Atmos-owned condition code contract. `key` is a stable presentation key such as `clear`, `partly-cloudy`, `rain`, or `thunderstorm`. The provider adapter maps Open-Meteo codes to this model.

---

## 6. Locations API

### `GET /api/v1/locations/search`

Searches the geocoding provider through the Atmos backend.

**Access:** Public  
**Success:** `200 OK`

#### Query parameters

| Parameter | Required | Type | Default | Validation |
| --- | --- | --- | --- | --- |
| `q` | Yes | string | — | Trimmed length 2–100 |
| `limit` | No | integer | `5` | 1–10 |
| `language` | No | string | `en` | Two-letter lowercase ISO 639-1 code |
| `countryCode` | No | string | — | Two-letter uppercase ISO 3166-1 alpha-2 code |

Unknown query parameters are rejected with `400` to catch client mistakes early.

#### Example request

```http
GET /api/v1/locations/search?q=Colombo&limit=5&language=en&countryCode=LK
```

#### Response

```json
{
  "data": [
    {
      "externalId": "provider-location-123",
      "name": "Colombo",
      "admin1": "Western Province",
      "country": "Sri Lanka",
      "countryCode": "LK",
      "latitude": 6.9271,
      "longitude": 79.8612,
      "timezone": "Asia/Colombo"
    }
  ]
}
```

No matches is a successful search and returns `200` with `"data": []`.

#### Errors

- `422 VALIDATION_ERROR`: invalid query values.
- `429 RATE_LIMIT_EXCEEDED`: client exceeded the applicable limit.
- `502 GEOCODING_PROVIDER_ERROR`: provider failed or returned invalid data.
- `504 PROVIDER_TIMEOUT`: provider timed out.

---

## 7. Weather API

### `GET /api/v1/weather`

Returns one normalized aggregate suited to the Atmos dashboard: current conditions, hourly forecast, and daily forecast.

**Access:** Public  
**Success:** `200 OK`

#### Query parameters

| Parameter | Required | Type | Default | Validation |
| --- | --- | --- | --- | --- |
| `latitude` | Yes | number | — | -90 to 90; maximum 6 decimal places |
| `longitude` | Yes | number | — | -180 to 180; maximum 6 decimal places |
| `units` | No | enum | `metric` | `metric` or `imperial` |
| `forecastDays` | No | integer | `7` | 1–16 |
| `timezone` | No | string | `auto` | `auto` or a valid IANA time-zone name |

The service normalizes coordinates to four decimal places for cache keys. The original validated coordinates may still appear in the response location metadata. Four decimal places are roughly precise enough for local weather while preventing nearly identical coordinates from fragmenting the cache.

`forecastDays` controls the number of `daily` items. The `hourly` array covers the same local-date range and can therefore contain up to 384 entries for a 16-day request.

Unknown query parameters are rejected with `400`.

#### Example request

```http
GET /api/v1/weather?latitude=6.9271&longitude=79.8612&units=metric&forecastDays=7&timezone=Asia%2FColombo
```

#### Response types

```typescript
interface WeatherResponse {
  data: {
    location: {
      latitude: number;
      longitude: number;
      timezone: string;
      utcOffsetSeconds: number;
    };
    units: WeatherUnits;
    current: CurrentWeather;
    hourly: HourlyWeather[];
    daily: DailyWeather[];
    generatedAt: string;
  };
}

interface CurrentWeather {
  observedAt: string;
  temperature: number;
  feelsLike: number;
  humidity: number;
  precipitation: number;
  rain: number;
  windSpeed: number;
  windDirection: number;
  windGust: number;
  condition: WeatherCondition;
  isDay: boolean;
}

interface HourlyWeather {
  time: string;
  temperature: number;
  feelsLike: number;
  humidity: number;
  precipitation: number;
  precipitationProbability: number | null;
  windSpeed: number;
  windDirection: number;
  windGust: number;
  uvIndex: number | null;
  condition: WeatherCondition;
  isDay: boolean;
}

interface DailyWeather {
  date: string;
  temperatureMax: number;
  temperatureMin: number;
  apparentTemperatureMax: number;
  apparentTemperatureMin: number;
  precipitationSum: number;
  precipitationProbabilityMax: number | null;
  windSpeedMax: number;
  windGustMax: number;
  uvIndexMax: number | null;
  sunrise: string;
  sunset: string;
  condition: WeatherCondition;
}
```

#### Example response

```json
{
  "data": {
    "location": {
      "latitude": 6.9271,
      "longitude": 79.8612,
      "timezone": "Asia/Colombo",
      "utcOffsetSeconds": 19800
    },
    "units": {
      "temperature": "°C",
      "windSpeed": "km/h",
      "precipitation": "mm"
    },
    "current": {
      "observedAt": "2026-08-30T14:15:00+05:30",
      "temperature": 29.1,
      "feelsLike": 33.2,
      "humidity": 78,
      "precipitation": 0,
      "rain": 0,
      "windSpeed": 14.2,
      "windDirection": 220,
      "windGust": 25.6,
      "condition": {
        "code": 2,
        "key": "partly-cloudy",
        "label": "Partly cloudy"
      },
      "isDay": true
    },
    "hourly": [
      {
        "time": "2026-08-30T15:00:00+05:30",
        "temperature": 28.8,
        "feelsLike": 32.7,
        "humidity": 79,
        "precipitation": 0.2,
        "precipitationProbability": 35,
        "windSpeed": 13.9,
        "windDirection": 224,
        "windGust": 24.8,
        "uvIndex": 4.2,
        "condition": {
          "code": 61,
          "key": "rain",
          "label": "Slight rain"
        },
        "isDay": true
      }
    ],
    "daily": [
      {
        "date": "2026-08-30",
        "temperatureMax": 30.4,
        "temperatureMin": 25.1,
        "apparentTemperatureMax": 35.2,
        "apparentTemperatureMin": 28.4,
        "precipitationSum": 4.8,
        "precipitationProbabilityMax": 70,
        "windSpeedMax": 19.4,
        "windGustMax": 34.2,
        "uvIndexMax": 7.1,
        "sunrise": "2026-08-30T06:02:00+05:30",
        "sunset": "2026-08-30T18:19:00+05:30",
        "condition": {
          "code": 61,
          "key": "rain",
          "label": "Slight rain"
        }
      }
    ],
    "generatedAt": "2026-08-30T08:45:12.120Z"
  }
}
```

The example values demonstrate the contract only; they are not measured or live weather data.

#### Validation details

- Numeric query strings are transformed only when the whole value is a valid finite number.
- `NaN`, `Infinity`, empty values, and partially numeric strings are rejected.
- Latitude and longitude are required together.
- Percentages are returned in the inclusive range 0–100.
- Wind direction is returned in degrees from 0 inclusive to 360 exclusive.
- Provider missing values that the contract permits are mapped to `null`; required missing provider values produce a provider error rather than a misleading default such as `0`.

#### Errors

- `422 VALIDATION_ERROR`: invalid coordinates, units, days, or timezone.
- `429 RATE_LIMIT_EXCEEDED`: client exceeded the applicable limit.
- `502 WEATHER_PROVIDER_ERROR`: provider failed or returned unusable data.
- `504 PROVIDER_TIMEOUT`: provider timed out.

---

## 8. Authentication API

### `POST /api/v1/auth/register`

**Access:** Public  
**Success:** `201 Created`  
**Side effect:** Sets the refresh-token cookie.

#### Request

```json
{
  "name": "Thejitha Wijayanayake",
  "email": "thejitha@example.com",
  "password": "correct horse battery staple"
}
```

Validation:

- `name`: trimmed string, 2–80 characters.
- `email`: trimmed, lowercased for canonical storage, valid email syntax, maximum 254 characters.
- `password`: 12–128 characters. Permit passphrases and all printable characters; do not require arbitrary symbol/uppercase composition rules.
- Unknown body properties are rejected.

#### Response

```json
{
  "data": {
    "user": {
      "id": "8ae6782d-094d-4d67-b4cb-cd421eaa8e1a",
      "name": "Thejitha Wijayanayake",
      "email": "thejitha@example.com",
      "createdAt": "2026-08-30T08:45:12.120Z"
    },
    "accessToken": "<jwt>",
    "expiresIn": 900
  }
}
```

Errors: `409 EMAIL_ALREADY_EXISTS`, `422 VALIDATION_ERROR`.

### `POST /api/v1/auth/login`

**Access:** Public  
**Success:** `200 OK`  
**Side effect:** Sets the refresh-token cookie.

#### Request

```json
{
  "email": "thejitha@example.com",
  "password": "correct horse battery staple"
}
```

Use the same `401 INVALID_CREDENTIALS` response for an unknown email and an incorrect password. This avoids account enumeration.

The success body has the same `data` shape as registration.

### `POST /api/v1/auth/refresh`

**Access:** Valid refresh-token cookie  
**Success:** `200 OK`  
**Request body:** None  
**Side effect:** Rotates the refresh cookie.

```json
{
  "data": {
    "accessToken": "<jwt>",
    "expiresIn": 900
  }
}
```

An absent, expired, revoked, or reused refresh token returns `401`.

### `POST /api/v1/auth/logout`

**Access:** Refresh-token cookie when present  
**Success:** `200 OK`  
**Request body:** None  
**Side effect:** Revokes the current refresh session and clears the cookie.

```json
{
  "data": {
    "loggedOut": true
  }
}
```

Logout is intentionally idempotent: if the session is already absent, the response is still `200` with `loggedOut: true`.

---

## 9. Current User API

### `GET /api/v1/users/me`

**Access:** Authenticated  
**Success:** `200 OK`

```json
{
  "data": {
    "id": "8ae6782d-094d-4d67-b4cb-cd421eaa8e1a",
    "name": "Thejitha Wijayanayake",
    "email": "thejitha@example.com",
    "createdAt": "2026-08-30T08:45:12.120Z"
  }
}
```

---

## 10. Favorites API

### Favorite resource

```typescript
interface Favorite {
  id: string;
  location: Location;
  createdAt: string;
}
```

### `GET /api/v1/favorites`

**Access:** Authenticated  
**Success:** `200 OK`

Favorites are returned newest first. This initial endpoint is not paginated because Atmos will enforce a small per-user maximum.

```json
{
  "data": [
    {
      "id": "69735226-1ace-4071-b5f5-41ad45fd22f8",
      "location": {
        "externalId": "provider-location-123",
        "name": "Colombo",
        "admin1": "Western Province",
        "country": "Sri Lanka",
        "countryCode": "LK",
        "latitude": 6.9271,
        "longitude": 79.8612,
        "timezone": "Asia/Colombo"
      },
      "createdAt": "2026-08-30T08:45:12.120Z"
    }
  ]
}
```

### `POST /api/v1/favorites`

**Access:** Authenticated  
**Success:** `201 Created`

#### Request

```json
{
  "location": {
    "externalId": "provider-location-123",
    "name": "Colombo",
    "admin1": "Western Province",
    "country": "Sri Lanka",
    "countryCode": "LK",
    "latitude": 6.9271,
    "longitude": 79.8612,
    "timezone": "Asia/Colombo"
  }
}
```

The location snapshot allows Atmos to render a favorite without re-running geocoding. The backend validates every field and normalizes coordinates. A uniqueness constraint prevents the same user from saving the same normalized location twice.

Initial maximum: 20 favorites per user. Exceeding the maximum returns `422` with code `FAVORITE_LIMIT_REACHED`. The response contains the created `Favorite` in `data`.

### `DELETE /api/v1/favorites/{favoriteId}`

**Access:** Authenticated  
**Success:** `204 No Content`

`favoriteId` must be a UUID. A missing favorite, including one owned by another user, returns `404 RESOURCE_NOT_FOUND`.

---

## 11. Search History API

History records locations that a registered user intentionally selects. Autocomplete keystrokes are not recorded.

### History resource

```typescript
interface HistoryEntry {
  id: string;
  location: Location;
  searchedAt: string;
}
```

### `GET /api/v1/history`

**Access:** Authenticated  
**Success:** `200 OK`

#### Query parameters

| Parameter | Required | Type | Default | Validation |
| --- | --- | --- | --- | --- |
| `limit` | No | integer | `20` | 1–50 |
| `cursor` | No | string | — | Non-empty opaque cursor, maximum 500 characters |

Results are ordered by `searchedAt` descending and use the standard pagination envelope.

### `POST /api/v1/history`

**Access:** Authenticated  
**Success:** `201 Created`

The request body is the same `{ "location": Location }` shape used to create a favorite. The response contains the created `HistoryEntry`.

To prevent noisy history, the service may coalesce a repeated selection of the same normalized location within a short defined interval. If implemented, that behavior must be documented and tested before release.

### `DELETE /api/v1/history/{historyId}`

**Access:** Authenticated  
**Success:** `204 No Content`

### `DELETE /api/v1/history`

**Access:** Authenticated  
**Success:** `204 No Content`

Deletes all history belonging to the current user. It never affects other users.

---

## 12. Preferences API

### Preference resource

```typescript
interface Preferences {
  units: 'metric' | 'imperial';
  theme: 'light' | 'dark' | 'system';
  updatedAt: string;
}
```

### `GET /api/v1/preferences`

**Access:** Authenticated  
**Success:** `200 OK`

```json
{
  "data": {
    "units": "metric",
    "theme": "system",
    "updatedAt": "2026-08-30T08:45:12.120Z"
  }
}
```

Defaults are created with the user account, so this endpoint does not return `404` for a valid user.

### `PATCH /api/v1/preferences`

**Access:** Authenticated  
**Success:** `200 OK`

```json
{
  "units": "imperial",
  "theme": "dark"
}
```

Validation:

- Both fields are optional, but at least one recognized field must be present.
- `units` is `metric` or `imperial`.
- `theme` is `light`, `dark`, or `system`.
- Explicit `null` values and unknown properties are rejected.

The response contains the complete updated `Preferences` resource.

---

## 13. Health API

### `GET /api/v1/health`

**Access:** Public  
**Purpose:** Liveness; confirms that the NestJS process can answer HTTP requests.  
**Success:** `200 OK`

```json
{
  "data": {
    "status": "ok",
    "timestamp": "2026-08-30T08:45:12.120Z"
  }
}
```

This endpoint should not query PostgreSQL, Redis, or Open-Meteo.

### `GET /api/v1/health/ready`

**Access:** Public  
**Purpose:** Readiness; checks whether Atmos can serve application traffic.  
**Success:** `200 OK` when required dependencies are ready.  
**Failure:** `503 Service Unavailable` when a required dependency is unavailable.

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

Open-Meteo is not called by readiness checks. Repeated readiness probes must not consume provider capacity or make deployment health depend on a third party.

Production deployments may expose only the overall status publicly and keep detailed dependency information internal.

---

## 14. Validation Policy

The NestJS global validation configuration should eventually enforce these rules:

- Transform explicitly supported primitive query parameters.
- Reject unknown request-body fields (`whitelist: true`, `forbidNonWhitelisted: true`).
- Reject unknown query parameters through DTO validation.
- Validate nested objects.
- Stop malformed JSON at `400`; use `422` for semantically invalid values.
- Set sensible string and collection length limits before calling providers or databases.
- Normalize only after validation; normalization must not turn an invalid input into a valid one silently.

Validation is required at the API boundary even if the frontend already validates. Frontend validation improves UX; backend validation protects the system and defines the trusted boundary.

---

## 15. Public vs Authenticated Matrix

| Capability | Guest | Registered user |
| --- | :---: | :---: |
| Search locations | Yes | Yes |
| Read weather and forecasts | Yes | Yes |
| Use temporary unit/theme choices in the frontend | Yes | Yes |
| Register/login/refresh/logout | Yes | Yes |
| Read current account | No | Yes |
| Create/list/delete favorites | No | Yes |
| Record/list/delete search history | No | Yes |
| Read/update synchronized preferences | No | Yes |
| Check liveness/readiness | Yes | Yes |

Atmos has one application role in v1: user. There is no admin role or role-based authorization requirement in the current feature scope.

---

## 16. Deliberate Non-Decisions and Deferred Details

The contract intentionally does not fix details that require implementation evidence or a later threat/performance decision:

- Exact rate-limit quotas.
- Exact Redis TTL values beyond the initial architecture estimates.
- Browser/CDN cache-control behavior.
- Refresh-token lifetime, storage schema, and reuse-detection details.
- Search-history retention period and repeat-selection coalescing interval.
- Production health-check detail exposure.
- A separate endpoint for weather sections. The v1 aggregate is the initial recommendation; measurements can justify later changes.

These are not missing route definitions. They are engineering parameters that should be decided when the relevant capability is implemented and tested.

---

## 17. OpenAPI and Implementation Acceptance Criteria

When NestJS implementation begins, this contract is accepted only when:

1. Swagger exposes all implemented v1 routes and DTO schemas.
2. Example requests match validation behavior.
3. Success and error responses match the documented envelopes.
4. Public and authenticated endpoints have the documented guards.
5. Provider DTOs are mapped to Atmos domain models.
6. No raw Open-Meteo response type is exported to controllers or frontend code.
7. Automated tests cover representative success, validation, authorization, conflict, provider-failure, and timeout cases.
8. Contract changes are reflected here before dependent frontend work is merged.

---

## 18. Key Design Decisions

- Use URL-based major versioning with `/api/v1`.
- Use resource-oriented REST routes and standard method semantics.
- Use a single aggregate weather endpoint for the initial dashboard.
- Resolve text locations separately, then request weather using coordinates.
- Keep weather reads side-effect free; record history explicitly.
- Return Atmos-owned weather models rather than Open-Meteo responses.
- Make requested units explicit so weather output remains deterministic and cacheable.
- Use cursor pagination for potentially growing user history.
- Use Problem Details-compatible error responses with stable Atmos error codes.
- Use access-token Bearer authentication plus an HttpOnly refresh cookie as the Week 5 implementation baseline.
- Keep liveness separate from dependency readiness.

No new ADR is required for the general endpoint shapes. The authentication token-storage design is security-sensitive and should receive its own ADR when the Week 5 threat model confirms the final approach.
