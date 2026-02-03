# Technical Design: Address Autocomplete API using Amazon Location Service

## Executive Summary

This design document outlines the implementation of a serverless Address Autocomplete API that leverages Amazon Location Service to provide real-time address suggestions for Australian addresses. The system uses AWS Lambda for compute, API Gateway for request routing, ElastiCache for caching, and integrates with PSMA and AusPost-equivalent address providers through Amazon Location Service. The architecture ensures sub-250ms median response times for cached queries, comprehensive error handling, and cost-effective scaling.

**Key Technologies:** AWS Lambda (Node.js 18.x), API Gateway REST API, Amazon Location Service, ElastiCache Redis, AWS CDK (TypeScript), CloudWatch

## Project Context

- **Project:** Address Autocomplete API (Jira: DF-2196)
- **Objective:** Provide fast, accurate address autocomplete for Australian addresses with normalized response format
- **Tech Stack:** AWS Lambda (Node.js 18.x), TypeScript, API Gateway, Amazon Location Service, ElastiCache Redis, AWS CDK
- **Team:** 2-3 backend developers, moderate AWS/serverless experience
- **Timeline:** 3 days (MVP implementation)
- **Scale:** Expected 10K requests/day initially, scaling to 100K requests/day within 6 months
- **Performance Target:** <250ms median response for cached locations, <800ms for uncached
- **Region:** ap-southeast-2 (Sydney) - Primary deployment region
- **Budget:** $50-100/month initially (within existing AWS account)
- **Constraints:**
  - Must integrate with existing auth middleware
  - Must support feature flags per environment
  - Must provide actionable error messages for debugging
  - Australian address focus (PSMA data provider)

## Requirements & Assumptions

### Requirements

1. **Functional Requirements**
   - `/address/search` endpoint accepting free-text query parameter
   - Optional locality and state filters for narrowing results
   - Return normalized address payload for frontend consumption
   - Support pagination with configurable top-N results (default: 5, max: 20)
   - Query Amazon Location Service with PSMA data provider
   - Feature flag/config toggle per environment (dev/staging/prod)

2. **Non-Functional Requirements**
   - Performance: <250ms median response for cached locations
   - Performance: <800ms p95 for uncached queries
   - Rate limiting: 100 requests/minute per API key
   - Input validation with detailed error messages
   - Comprehensive logging of provider failures
   - Unit test coverage >80%, integration tests for critical paths
   - OpenAPI 3.0 documentation
   - Integration with existing auth middleware

3. **Security Requirements**
   - API key or JWT-based authentication
   - Rate limiting per authenticated user
   - No PII logging in CloudWatch
   - Encryption in transit (TLS 1.3)
   - Encryption at rest for cache data

### Assumptions

- **Authentication:** Existing auth middleware provides validated user context via Lambda authorizer
- **Data Provider:** Amazon Location Service provides access to PSMA and AusPost-equivalent data for Australian addresses
- **Caching:** 80% of queries will be repetitive and benefit from caching (based on typical autocomplete patterns)
- **Traffic Pattern:** Mostly synchronous user input during business hours (9am-6pm AEST), minimal overnight traffic
- **Query Characteristics:** Average query length 10-30 characters, most queries are partial addresses
- **Infrastructure:** VPC and networking already provisioned, can reuse existing VPC for ElastiCache
- **Monitoring:** CloudWatch logs and metrics are standard monitoring solution

### Non-Goals (Out of Scope)

- International address support (non-Australian addresses)
- Address validation or verification (only autocomplete suggestions)
- Geocoding or reverse geocoding
- Historical address lookups
- Batch address processing
- Mobile SDK or client libraries (API only)

## Current State Analysis

This is a greenfield implementation with the following existing infrastructure:

### Existing Infrastructure
- **VPC:** Existing VPC in ap-southeast-2 with public and private subnets
- **Auth Middleware:** API Gateway Lambda authorizer validates JWT tokens and provides user context
- **Monitoring:** CloudWatch Logs, CloudWatch Metrics, X-Ray tracing enabled
- **CI/CD:** GitHub Actions for deployment automation
- **Feature Flags:** AWS AppConfig for environment-specific configuration

### Integration Points
- **API Gateway:** Must integrate with existing REST API Gateway instance
- **Auth Middleware:** Lambda authorizer provides `userId` and `apiKey` in request context
- **Logging:** Standardized structured JSON logging format required
- **Monitoring:** CloudWatch metrics with existing dashboard integration

## Proposed Design

### System Architecture

```mermaid
graph TB
    Client[Frontend Client]
    APIGW[API Gateway<br/>REST API]
    Auth[Lambda Authorizer<br/>JWT Validation]
    Lambda[Lambda Function<br/>Address Autocomplete]
    Cache[(ElastiCache Redis<br/>Serverless)]
    ALS[Amazon Location Service<br/>PSMA Data Provider]
    CW[CloudWatch<br/>Logs & Metrics]
    AppConfig[AWS AppConfig<br/>Feature Flags]

    Client -->|HTTPS| APIGW
    APIGW -->|Authorize| Auth
    Auth -->|Context| APIGW
    APIGW -->|Invoke| Lambda
    Lambda -->|Check Cache| Cache
    Lambda -->|Query| ALS
    Lambda -->|Write Cache| Cache
    Lambda -->|Logs| CW
    Lambda -->|Get Config| AppConfig

    style Lambda fill:#ff9900
    style APIGW fill:#ff4444
    style Cache fill:#cc0000
    style ALS fill:#00cc00
```

**Architecture Flow:**

1. **Request Flow:** Client → API Gateway → Lambda Authorizer (validates JWT) → Lambda Function
2. **Cache Check:** Lambda checks ElastiCache Redis for cached query result
3. **Cache Hit:** Return cached result immediately (target: <100ms)
4. **Cache Miss:** Query Amazon Location Service → Cache result → Return to client
5. **Logging:** All requests logged to CloudWatch with structured JSON
6. **Metrics:** Response time, cache hit rate, error rate tracked in CloudWatch

### Key Components

#### 1. API Gateway REST API
**Purpose:** HTTP request routing, throttling, and API management

**Technology:** API Gateway REST API (not HTTP API) for usage plans and API key support

**Responsibilities:**
- Request routing to Lambda function
- Request validation (query parameter presence)
- Rate limiting via usage plans (100 req/min per API key)
- CORS handling
- Request/response transformation
- Integration with Lambda authorizer

**Configuration:**
- Stage: dev, staging, prod
- Throttling: 100 requests/minute per API key, burst: 50
- Timeout: 29 seconds (Lambda max)
- Binary media types: application/json
- CORS: Enabled for frontend domain

#### 2. Lambda Function (Address Autocomplete Service)
**Purpose:** Core business logic for address autocomplete

**Technology:** Node.js 18.x, TypeScript, 512MB memory, 10s timeout

**Responsibilities:**
- Input validation and sanitization
- Cache key generation and lookup
- Amazon Location Service API calls
- Response normalization
- Error handling and retry logic
- Structured logging
- Metric emission

**Environment Variables:**
- `ELASTICACHE_ENDPOINT`: Redis cluster endpoint
- `LOCATION_SERVICE_PLACE_INDEX`: Amazon Location Service place index name
- `CACHE_TTL_SECONDS`: Cache TTL (default: 3600)
- `MAX_RESULTS_DEFAULT`: Default result limit (5)
- `MAX_RESULTS_LIMIT`: Maximum result limit (20)
- `APPCONFIG_APP_ID`: AWS AppConfig application ID
- `APPCONFIG_ENV_ID`: AWS AppConfig environment ID
- `APPCONFIG_CONFIG_PROFILE`: Feature flag configuration profile

#### 3. ElastiCache Redis Serverless
**Purpose:** High-performance caching layer for address queries

**Technology:** ElastiCache Redis Serverless (auto-scaling, 1GB-5GB cache)

**Responsibilities:**
- Cache address autocomplete results
- Automatic scaling based on workload
- Sub-millisecond read latency
- Automatic failover and replication

**Configuration:**
- Engine: Redis 7.x
- Mode: Serverless (automatic scaling)
- Snapshot: Daily automated snapshots
- Encryption: At-rest and in-transit enabled
- VPC: Deployed in private subnet
- Security Group: Allow inbound from Lambda security group only

**Cache Strategy:**
- Key format: `autocomplete:v1:{normalizedQuery}:{locality}:{state}`
- TTL: 1 hour (configurable via environment variable)
- Eviction: LRU (Least Recently Used)

#### 4. Amazon Location Service Place Index
**Purpose:** Address data provider for Australian addresses

**Technology:** Amazon Location Service with PSMA data provider (Esri)

**Responsibilities:**
- Provide address suggestions based on partial input
- Filter by geographic bounds (locality, state)
- Return standardized address components
- Support for Australian address formats

**Configuration:**
- Data Provider: Esri (includes PSMA data for Australia)
- Intended Use: SingleUse (per query pricing)
- Pricing Bucket: Search requests

#### 5. Lambda Authorizer
**Purpose:** Authenticate and authorize API requests

**Technology:** Existing Lambda authorizer (JWT validation)

**Responsibilities:**
- Validate JWT token from Authorization header
- Extract user context (userId, apiKey, permissions)
- Return IAM policy allowing/denying request
- Cache authorization decisions (300s TTL)

**Integration:**
- Request context: `userId`, `apiKey`, `rateLimit`
- Policy: Allow/Deny based on token validity

#### 6. AWS AppConfig
**Purpose:** Feature flag management and environment-specific configuration

**Technology:** AWS AppConfig with hosted configuration

**Responsibilities:**
- Enable/disable autocomplete feature per environment
- Configure provider-specific settings
- A/B testing and canary deployments
- Runtime configuration updates without redeployment

**Configuration Schema:**
```json
{
  "features": {
    "addressAutocomplete": {
      "enabled": true,
      "maxResults": 5,
      "cacheTtlSeconds": 3600,
      "providers": {
        "amazonLocation": {
          "enabled": true,
          "placeIndex": "address-autocomplete-place-index"
        }
      }
    }
  }
}
```

### Technology Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **API Gateway** | REST API (not HTTP API) | Usage plans and API keys required for rate limiting; more mature feature set |
| **Compute** | Lambda (Node.js 18.x) | Serverless, auto-scaling, event-driven; Node.js for fast I/O operations |
| **Address Provider** | Amazon Location Service (PSMA/Esri) | Managed service, no infrastructure overhead; PSMA data for Australian addresses |
| **Cache** | ElastiCache Redis Serverless | Managed Redis, automatic scaling, sub-ms latency; serverless pricing aligns with usage |
| **IaC** | AWS CDK (TypeScript) | Type-safe infrastructure definition, better abstraction than CloudFormation |
| **Monitoring** | CloudWatch + X-Ray | Native AWS integration, standard monitoring stack |
| **Feature Flags** | AWS AppConfig | Managed service, fast retrieval, supports JSON schemas |
| **Language** | TypeScript | Type safety, better developer experience, compile-time error detection |

### Data Model

#### Request Schema

**GET/POST `/address/search`**

Query Parameters:
```typescript
interface AutocompleteRequest {
  query: string;              // Required: Free-text address query (1-200 chars)
  locality?: string;          // Optional: City/suburb filter
  state?: string;             // Optional: State code (NSW, VIC, QLD, etc.)
  maxResults?: number;        // Optional: Max results (1-20, default: 5)
}
```

Example Request:
```
GET /address/search?query=123%20George&locality=Sydney&state=NSW&maxResults=5
Authorization: Bearer <jwt_token>
```

#### Response Schema

**Success Response (200 OK):**
```typescript
interface AutocompleteResponse {
  results: AddressResult[];
  metadata: ResponseMetadata;
}

interface AddressResult {
  placeId: string;                    // Unique identifier for the place
  title: string;                      // Human-readable full address
  address: {
    addressNumber?: string;           // Street number
    street?: string;                  // Street name
    locality?: string;                // Suburb/city
    district?: string;                // District/area
    region?: string;                  // State name
    regionCode?: string;              // State code (NSW, VIC, etc.)
    postalCode?: string;              // Postcode
    country?: string;                 // Country name (Australia)
    countryCode?: string;             // Country code (AU)
    label?: string;                   // Formatted address label
  };
  position?: {
    longitude: number;
    latitude: number;
  };
  distance?: number;                  // Distance from bias position (meters)
}

interface ResponseMetadata {
  cached: boolean;                    // Whether result served from cache
  queryTime: number;                  // Query processing time (ms)
  provider: string;                   // Data provider name (amazon-location)
  resultCount: number;                // Number of results returned
}
```

Example Response:
```json
{
  "results": [
    {
      "placeId": "here:pds:place:036y1tcb-a...",
      "title": "123 George St, Sydney NSW 2000, Australia",
      "address": {
        "addressNumber": "123",
        "street": "George St",
        "locality": "Sydney",
        "region": "New South Wales",
        "regionCode": "NSW",
        "postalCode": "2000",
        "country": "Australia",
        "countryCode": "AU",
        "label": "123 George St, Sydney NSW 2000"
      },
      "position": {
        "longitude": 151.2073,
        "latitude": -33.8688
      }
    }
  ],
  "metadata": {
    "cached": true,
    "queryTime": 45,
    "provider": "amazon-location",
    "resultCount": 1
  }
}
```

**Error Response (4xx/5xx):**
```typescript
interface ErrorResponse {
  error: {
    code: string;                     // Machine-readable error code
    message: string;                  // Human-readable error message
    details?: Record<string, any>;    // Additional error context
  };
  requestId: string;                  // CloudWatch request ID for debugging
}
```

Example Error:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Query parameter must be between 1 and 200 characters",
    "details": {
      "field": "query",
      "value": "",
      "constraint": "minLength:1"
    }
  },
  "requestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### API Design

#### Authentication Endpoints

Authentication is handled by existing Lambda authorizer. The autocomplete API expects:

**Request Headers:**
```
Authorization: Bearer <jwt_token>
Content-Type: application/json
X-API-Key: <api_key> (optional, for usage plan tracking)
```

#### Address Autocomplete Endpoint

**GET/POST `/address/search`**

**Method:** GET or POST (GET preferred for caching, POST for sensitive queries)

**Rate Limit:** 100 requests/minute per API key, burst: 50

**Timeout:** 10 seconds

**Request:**
```http
GET /address/search?query=123%20George&locality=Sydney&state=NSW&maxResults=5 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
X-API-Key: abc123def456
```

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| query | string | Yes | 1-200 chars, alphanumeric + spaces | Free-text address query |
| locality | string | No | 1-100 chars | City/suburb filter |
| state | string | No | 2-3 chars, valid AU state code | State filter (NSW, VIC, QLD, SA, WA, TAS, NT, ACT) |
| maxResults | integer | No | 1-20, default: 5 | Maximum number of results |

**Responses:**

| Status | Description | Example |
|--------|-------------|---------|
| 200 OK | Successful autocomplete | See Response Schema above |
| 400 Bad Request | Invalid query parameters | `{"error": {"code": "VALIDATION_ERROR", "message": "..."}}` |
| 401 Unauthorized | Missing or invalid JWT token | `{"error": {"code": "UNAUTHORIZED", "message": "..."}}` |
| 429 Too Many Requests | Rate limit exceeded | `{"error": {"code": "RATE_LIMIT_EXCEEDED", "message": "..."}}` |
| 500 Internal Server Error | Service failure | `{"error": {"code": "INTERNAL_ERROR", "message": "..."}}` |
| 503 Service Unavailable | Location service unavailable | `{"error": {"code": "SERVICE_UNAVAILABLE", "message": "..."}}` |

**Response Headers:**
```
Content-Type: application/json
X-Cache-Status: HIT|MISS
X-Response-Time: 45
X-Request-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### Integration Patterns

#### Amazon Location Service Integration Flow

```mermaid
sequenceDiagram
    participant Client
    participant APIGateway
    participant Lambda
    participant Cache
    participant LocationService
    participant CloudWatch

    Client->>APIGateway: GET /address/search?query=123 George St
    APIGateway->>Lambda: Authorize (via Lambda Authorizer)
    Lambda->>APIGateway: Allow with user context
    APIGateway->>Lambda: Invoke with request

    Lambda->>CloudWatch: Log request received
    Lambda->>Cache: GET autocomplete:v1:123georgest

    alt Cache Hit
        Cache-->>Lambda: Cached results
        Lambda->>CloudWatch: Log cache hit
        Lambda-->>APIGateway: Return results (metadata.cached=true)
    else Cache Miss
        Cache-->>Lambda: null
        Lambda->>CloudWatch: Log cache miss
        Lambda->>LocationService: Autocomplete API call
        LocationService-->>Lambda: Raw address results
        Lambda->>Lambda: Normalize response
        Lambda->>Cache: SET autocomplete:v1:123georgest (TTL: 3600s)
        Lambda->>CloudWatch: Log provider call
        Lambda-->>APIGateway: Return results (metadata.cached=false)
    end

    APIGateway-->>Client: JSON response with metadata
```

**Integration Steps:**

1. **Request Validation:**
   - Lambda validates query parameters
   - Sanitize input (remove special chars, normalize whitespace)
   - Check query length constraints

2. **Cache Key Generation:**
   - Normalize query: lowercase, trim, remove extra spaces
   - Format: `autocomplete:v1:{normalizedQuery}:{locality}:{state}`
   - Example: `autocomplete:v1:123georgest:sydney:nsw`

3. **Cache Lookup:**
   - Redis GET operation
   - Deserialize JSON if hit
   - Track cache hit/miss metric

4. **Location Service Query (on cache miss):**
   - Build `AutocompleteRequest`:
     ```typescript
     {
       QueryText: normalizedQuery,
       MaxResults: maxResults,
       Filter: {
         IncludeCountries: ['AUS'],
         // Additional filters based on locality/state
       },
       AdditionalFeatures: ['Core']
     }
     ```
   - Call Amazon Location Service Autocomplete API
   - Set timeout: 5 seconds
   - Retry on throttling: exponential backoff (3 attempts)

5. **Response Normalization:**
   - Map Amazon Location Service response to standard format
   - Extract address components
   - Add metadata (cached flag, query time, provider)

6. **Cache Write:**
   - Serialize normalized response to JSON
   - Redis SETEX operation (key, TTL, value)
   - TTL: 3600 seconds (1 hour)
   - Fire-and-forget (don't wait for confirmation)

7. **Metrics & Logging:**
   - CloudWatch metric: `CacheHitRate`, `ResponseTime`, `ErrorRate`
   - Structured log: Request ID, user ID, query, cache status, response time

#### Error Handling Strategy

**Error Categories:**

1. **Client Errors (4xx):**
   - Validation errors (400)
   - Authentication errors (401)
   - Rate limit errors (429)
   - Log as INFO level
   - Return actionable error message

2. **Server Errors (5xx):**
   - Location Service failures (503)
   - Cache connection errors (500)
   - Internal errors (500)
   - Log as ERROR level with full stack trace
   - Return generic message to client, detailed log to CloudWatch

**Error Handling Flow:**

```typescript
try {
  // Validate input
  validateRequest(request);

  // Try cache
  const cached = await getFromCache(cacheKey);
  if (cached) return cached;

  // Try Location Service with retry
  const results = await withRetry(
    () => locationService.autocomplete(query),
    { maxRetries: 3, backoff: 'exponential' }
  );

  // Normalize and cache
  const normalized = normalizeResults(results);
  await setCache(cacheKey, normalized); // Fire-and-forget

  return normalized;

} catch (error) {
  if (error instanceof ValidationError) {
    // 400 Bad Request
    return errorResponse(400, 'VALIDATION_ERROR', error.message);
  } else if (error instanceof ThrottlingError) {
    // 429 Too Many Requests
    return errorResponse(429, 'RATE_LIMIT_EXCEEDED', 'Please retry after 60 seconds');
  } else if (error instanceof LocationServiceError) {
    // 503 Service Unavailable
    logger.error('Location Service failed', { error, requestId });
    return errorResponse(503, 'SERVICE_UNAVAILABLE', 'Address service temporarily unavailable');
  } else {
    // 500 Internal Server Error
    logger.error('Unexpected error', { error, requestId, stack: error.stack });
    return errorResponse(500, 'INTERNAL_ERROR', 'An unexpected error occurred');
  }
}
```

**Retry Strategy:**

- **Location Service Throttling (429):** Exponential backoff with jitter (100ms, 200ms, 400ms)
- **Location Service Timeout:** 5-second timeout, fail fast, no retry
- **Cache Connection Error:** Skip cache, query Location Service directly, log error
- **Circuit Breaker:** After 50% error rate over 1 minute, fail fast for 30 seconds

### Caching Strategy

#### Cache Architecture

**Technology:** ElastiCache Redis Serverless

**Rationale:**
- Managed service, no operational overhead
- Automatic scaling (1GB-5GB based on workload)
- Sub-millisecond latency
- Built-in replication and failover
- Encryption at rest and in transit

#### Cache Key Design

**Format:** `autocomplete:v1:{normalizedQuery}:{locality?}:{state?}`

**Key Normalization:**
1. Convert query to lowercase
2. Trim whitespace
3. Replace multiple spaces with single space
4. Remove special characters except letters, numbers, spaces
5. URL-encode if necessary

**Examples:**
```
Query: "123 George St"
Key: autocomplete:v1:123georgest::

Query: "123 George St", Locality: "Sydney", State: "NSW"
Key: autocomplete:v1:123georgest:sydney:nsw

Query: "Level 2/45 Clarence St"
Key: autocomplete:v1:level245clarencest::
```

#### Cache Configuration

**TTL Strategy:**
- Default TTL: 3600 seconds (1 hour)
- Rationale: Address data is relatively static; 1-hour TTL balances freshness and cache efficiency
- Configurable via environment variable for testing

**Eviction Policy:**
- LRU (Least Recently Used)
- Redis will automatically evict least-used keys when memory limit reached

**Data Structure:**
- Simple key-value pairs
- Value: JSON-serialized AutocompleteResponse
- Typical size: 1-5KB per entry

**Memory Estimation:**
- Average entry size: 3KB
- Target 10K unique queries cached
- Memory needed: 30MB (well within 1GB minimum)

#### Cache Operations

**Read Operation:**
```typescript
async function getFromCache(key: string): Promise<AutocompleteResponse | null> {
  try {
    const startTime = Date.now();
    const cached = await redis.get(key);
    const duration = Date.now() - startTime;

    cloudwatch.putMetric('CacheLookupDuration', duration);

    if (cached) {
      cloudwatch.putMetric('CacheHit', 1);
      return JSON.parse(cached);
    } else {
      cloudwatch.putMetric('CacheMiss', 1);
      return null;
    }
  } catch (error) {
    logger.error('Cache read error', { error, key });
    cloudwatch.putMetric('CacheError', 1);
    return null; // Fail open: continue without cache
  }
}
```

**Write Operation:**
```typescript
async function setCache(key: string, value: AutocompleteResponse, ttl: number = 3600): Promise<void> {
  try {
    // Fire-and-forget: don't await
    redis.setex(key, ttl, JSON.stringify(value)).catch(error => {
      logger.error('Cache write error', { error, key });
      cloudwatch.putMetric('CacheWriteError', 1);
    });
  } catch (error) {
    logger.error('Cache write error', { error, key });
    cloudwatch.putMetric('CacheWriteError', 1);
  }
}
```

**Cache Invalidation:**
- Automatic expiration after TTL
- Manual invalidation: Not implemented in MVP (could add admin API endpoint later)
- Full cache flush: Redis FLUSHALL command (emergency only)

#### Cache Performance Monitoring

**Metrics to Track:**
- Cache hit rate (target: >70%)
- Cache read latency (target: <10ms p95)
- Cache write latency (target: <20ms p95)
- Cache memory utilization
- Eviction rate

**Alarms:**
- Cache hit rate <50% for 5 minutes
- Cache connection errors >10/minute
- Cache latency p95 >50ms

### Rate Limiting Approach

#### API Gateway Usage Plans

**Mechanism:** API Gateway Usage Plans with API Keys

**Configuration:**

**Usage Plan: Standard**
- Throttle rate: 100 requests/second per API key
- Burst: 50 requests
- Quota: 100,000 requests/month per API key

**Implementation:**
```typescript
const usagePlan = new apigateway.UsagePlan(this, 'AddressAutocompleteUsagePlan', {
  name: 'address-autocomplete-standard',
  throttle: {
    rateLimit: 100,      // Requests per second
    burstLimit: 50       // Burst capacity
  },
  quota: {
    limit: 100000,       // Monthly quota
    period: apigateway.Period.MONTH
  }
});

usagePlan.addApiStage({
  stage: api.deploymentStage
});
```

**Per-User Rate Limiting:**

In addition to API Gateway throttling, implement custom rate limiting in Lambda:

```typescript
import { RateLimiter } from 'rate-limiter-flexible';

const rateLimiter = new RateLimiter({
  storeClient: redisClient,
  keyPrefix: 'ratelimit',
  points: 100,          // Number of requests
  duration: 60,         // Per 60 seconds
  blockDuration: 60     // Block for 60 seconds if exceeded
});

async function checkRateLimit(userId: string): Promise<void> {
  try {
    await rateLimiter.consume(userId);
  } catch (error) {
    throw new RateLimitError('Rate limit exceeded. Please try again in 60 seconds.');
  }
}
```

**Rate Limit Response:**
```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1675430400

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit of 100 requests per minute exceeded",
    "details": {
      "retryAfter": 60,
      "limit": 100,
      "window": "1 minute"
    }
  },
  "requestId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### Error Handling

#### Error Categories & HTTP Status Codes

| Error Type | HTTP Status | Error Code | Retry? | Example |
|------------|-------------|------------|--------|---------|
| Validation Error | 400 | VALIDATION_ERROR | No | Invalid query parameter |
| Unauthorized | 401 | UNAUTHORIZED | No | Missing/invalid JWT token |
| Rate Limit | 429 | RATE_LIMIT_EXCEEDED | Yes (after delay) | >100 req/min |
| Location Service Throttle | 503 | SERVICE_THROTTLED | Yes (exponential backoff) | Amazon Location Service 429 |
| Location Service Unavailable | 503 | SERVICE_UNAVAILABLE | Yes (after delay) | Amazon Location Service timeout |
| Cache Error | 500 | CACHE_ERROR | No | Redis connection failed (degrades gracefully) |
| Internal Error | 500 | INTERNAL_ERROR | No | Unexpected exception |

#### Error Response Format

```typescript
interface ErrorResponse {
  error: {
    code: string;                     // Machine-readable error code
    message: string;                  // Human-readable message
    details?: Record<string, any>;    // Additional context for debugging
  };
  requestId: string;                  // CloudWatch request ID
  timestamp: string;                  // ISO 8601 timestamp
}
```

#### Logging Strategy

**Structured Logging Format:**

```typescript
interface LogEntry {
  timestamp: string;                  // ISO 8601
  level: 'DEBUG' | 'INFO' | 'WARN' | 'ERROR';
  requestId: string;
  userId?: string;
  event: string;                      // Event name (e.g., 'CACHE_HIT', 'LOCATION_SERVICE_ERROR')
  message: string;
  duration?: number;                  // Operation duration (ms)
  metadata?: Record<string, any>;     // Additional context
  error?: {
    name: string;
    message: string;
    stack?: string;
  };
}
```

**Example Logs:**

**Success (Cache Hit):**
```json
{
  "timestamp": "2026-02-03T14:30:00.123Z",
  "level": "INFO",
  "requestId": "a1b2c3d4-e5f6-7890",
  "userId": "user_12345",
  "event": "CACHE_HIT",
  "message": "Address autocomplete served from cache",
  "duration": 45,
  "metadata": {
    "query": "123 george st",
    "cacheKey": "autocomplete:v1:123georgest::",
    "resultCount": 5
  }
}
```

**Error (Location Service Failure):**
```json
{
  "timestamp": "2026-02-03T14:30:00.123Z",
  "level": "ERROR",
  "requestId": "b2c3d4e5-f6a7-8901",
  "userId": "user_67890",
  "event": "LOCATION_SERVICE_ERROR",
  "message": "Amazon Location Service API call failed",
  "duration": 5023,
  "metadata": {
    "query": "invalid address",
    "statusCode": 503,
    "retryAttempt": 3
  },
  "error": {
    "name": "ServiceUnavailableError",
    "message": "Service temporarily unavailable",
    "stack": "ServiceUnavailableError: Service temporarily unavailable\n    at ..."
  }
}
```

**CloudWatch Insights Queries:**

**Cache Hit Rate:**
```
fields @timestamp, event, duration
| filter event in ["CACHE_HIT", "CACHE_MISS"]
| stats count() as total, sum(event = "CACHE_HIT") as hits by bin(1m)
| fields hits / total * 100 as hit_rate_pct
```

**Error Rate by Type:**
```
fields @timestamp, event, error.name
| filter level = "ERROR"
| stats count() by error.name
```

**p95 Response Time:**
```
fields @timestamp, duration
| filter event = "REQUEST_COMPLETE"
| stats pct(duration, 95) as p95, pct(duration, 50) as p50, avg(duration) as avg
```

### Lambda Code Structure

#### Project Structure

```
address-autocomplete/
├── src/
│   ├── handlers/
│   │   └── autocomplete.ts          # Lambda entry point
│   ├── services/
│   │   ├── location-service.ts      # Amazon Location Service client
│   │   ├── cache-service.ts         # Redis cache client
│   │   └── config-service.ts        # AppConfig client
│   ├── models/
│   │   ├── request.ts               # Request models
│   │   ├── response.ts              # Response models
│   │   └── errors.ts                # Custom error classes
│   ├── utils/
│   │   ├── validation.ts            # Input validation
│   │   ├── normalization.ts         # Query normalization
│   │   ├── logger.ts                # Structured logger
│   │   └── metrics.ts               # CloudWatch metrics
│   └── config/
│       └── constants.ts             # Constants and config
├── tests/
│   ├── unit/
│   │   ├── services/                # Service unit tests
│   │   └── utils/                   # Utility unit tests
│   └── integration/
│       └── autocomplete.test.ts     # Integration tests
├── infrastructure/
│   ├── lib/
│   │   ├── address-autocomplete-stack.ts
│   │   ├── api-gateway-construct.ts
│   │   ├── lambda-construct.ts
│   │   ├── elasticache-construct.ts
│   │   └── location-service-construct.ts
│   ├── bin/
│   │   └── app.ts
│   └── cdk.json
├── package.json
├── tsconfig.json
├── jest.config.js
└── README.md
```

#### Key Implementation Files

**handlers/autocomplete.ts:**
```typescript
import { APIGatewayProxyEvent, APIGatewayProxyResult, Context } from 'aws-lambda';
import { LocationService } from '../services/location-service';
import { CacheService } from '../services/cache-service';
import { ConfigService } from '../services/config-service';
import { validateRequest, normalizeQuery } from '../utils/validation';
import { logger } from '../utils/logger';
import { metrics } from '../utils/metrics';
import { AutocompleteRequest, AutocompleteResponse } from '../models/request';
import { ValidationError, RateLimitError } from '../models/errors';

const locationService = new LocationService(process.env.LOCATION_SERVICE_PLACE_INDEX!);
const cacheService = new CacheService(process.env.ELASTICACHE_ENDPOINT!);
const configService = new ConfigService();

export const handler = async (
  event: APIGatewayProxyEvent,
  context: Context
): Promise<APIGatewayProxyResult> => {
  const requestId = context.requestId;
  const startTime = Date.now();

  logger.info('Request received', { requestId, path: event.path });

  try {
    // Get feature flag configuration
    const config = await configService.getConfig();
    if (!config.features.addressAutocomplete.enabled) {
      return errorResponse(503, 'FEATURE_DISABLED', 'Address autocomplete is currently disabled');
    }

    // Validate request
    const request = validateRequest(event.queryStringParameters);
    const userId = event.requestContext.authorizer?.userId;

    logger.info('Request validated', { requestId, userId, query: request.query });

    // Generate cache key
    const cacheKey = generateCacheKey(request);

    // Try cache first
    const cached = await cacheService.get(cacheKey);
    if (cached) {
      logger.info('Cache hit', { requestId, cacheKey });
      metrics.putMetric('CacheHit', 1);

      const response = {
        ...cached,
        metadata: {
          ...cached.metadata,
          cached: true,
          queryTime: Date.now() - startTime
        }
      };

      return successResponse(response);
    }

    logger.info('Cache miss', { requestId, cacheKey });
    metrics.putMetric('CacheMiss', 1);

    // Query Amazon Location Service
    const results = await locationService.autocomplete(request);

    // Normalize response
    const response: AutocompleteResponse = {
      results: results.ResultItems.map(normalizeResult),
      metadata: {
        cached: false,
        queryTime: Date.now() - startTime,
        provider: 'amazon-location',
        resultCount: results.ResultItems.length
      }
    };

    // Cache response (fire-and-forget)
    cacheService.set(cacheKey, response, config.features.addressAutocomplete.cacheTtlSeconds)
      .catch(err => logger.error('Cache write failed', { error: err, requestId }));

    logger.info('Request complete', { requestId, duration: Date.now() - startTime });
    metrics.putMetric('RequestDuration', Date.now() - startTime);

    return successResponse(response);

  } catch (error) {
    return handleError(error, requestId);
  }
};

function generateCacheKey(request: AutocompleteRequest): string {
  const normalized = normalizeQuery(request.query);
  const locality = request.locality?.toLowerCase() || '';
  const state = request.state?.toLowerCase() || '';
  return `autocomplete:v1:${normalized}:${locality}:${state}`;
}

function successResponse(data: any): APIGatewayProxyResult {
  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(data)
  };
}

function errorResponse(statusCode: number, code: string, message: string, details?: any): APIGatewayProxyResult {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      error: { code, message, details },
      requestId: context.requestId,
      timestamp: new Date().toISOString()
    })
  };
}

function handleError(error: any, requestId: string): APIGatewayProxyResult {
  if (error instanceof ValidationError) {
    logger.info('Validation error', { error: error.message, requestId });
    return errorResponse(400, 'VALIDATION_ERROR', error.message, error.details);
  } else if (error instanceof RateLimitError) {
    logger.warn('Rate limit exceeded', { requestId });
    return errorResponse(429, 'RATE_LIMIT_EXCEEDED', error.message);
  } else {
    logger.error('Unexpected error', { error, requestId, stack: error.stack });
    metrics.putMetric('ErrorCount', 1);
    return errorResponse(500, 'INTERNAL_ERROR', 'An unexpected error occurred');
  }
}
```

**services/location-service.ts:**
```typescript
import { LocationClient, AutocompleteCommand } from '@aws-sdk/client-geo-places';
import { logger } from '../utils/logger';
import { AutocompleteRequest } from '../models/request';

export class LocationService {
  private client: LocationClient;
  private placeIndex: string;

  constructor(placeIndex: string) {
    this.client = new LocationClient({ region: process.env.AWS_REGION });
    this.placeIndex = placeIndex;
  }

  async autocomplete(request: AutocompleteRequest, retries = 3): Promise<any> {
    const command = new AutocompleteCommand({
      QueryText: request.query,
      MaxResults: request.maxResults || 5,
      Filter: {
        IncludeCountries: ['AUS'],
        ...(request.locality && { Locality: request.locality }),
        ...(request.state && { Region: request.state })
      },
      AdditionalFeatures: ['Core']
    });

    try {
      const response = await this.client.send(command);
      logger.debug('Location Service response', { resultCount: response.ResultItems?.length });
      return response;
    } catch (error: any) {
      if (error.name === 'ThrottlingException' && retries > 0) {
        const delay = Math.pow(2, 3 - retries) * 100; // Exponential backoff
        logger.warn('Location Service throttled, retrying', { retries, delay });
        await new Promise(resolve => setTimeout(resolve, delay));
        return this.autocomplete(request, retries - 1);
      }
      logger.error('Location Service error', { error });
      throw error;
    }
  }
}
```

**services/cache-service.ts:**
```typescript
import Redis from 'ioredis';
import { logger } from '../utils/logger';

export class CacheService {
  private client: Redis;

  constructor(endpoint: string) {
    this.client = new Redis({
      host: endpoint,
      port: 6379,
      tls: { rejectUnauthorized: true },
      retryStrategy: (times) => Math.min(times * 50, 2000),
      maxRetriesPerRequest: 3
    });

    this.client.on('error', (err) => {
      logger.error('Redis connection error', { error: err });
    });
  }

  async get(key: string): Promise<any | null> {
    try {
      const value = await this.client.get(key);
      return value ? JSON.parse(value) : null;
    } catch (error) {
      logger.error('Cache get error', { error, key });
      return null; // Fail open
    }
  }

  async set(key: string, value: any, ttl: number = 3600): Promise<void> {
    try {
      await this.client.setex(key, ttl, JSON.stringify(value));
    } catch (error) {
      logger.error('Cache set error', { error, key });
      // Don't throw - cache failures shouldn't break the API
    }
  }
}
```

### Infrastructure as Code (AWS CDK)

#### CDK Stack Structure

**infrastructure/lib/address-autocomplete-stack.ts:**
```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as elasticache from 'aws-cdk-lib/aws-elasticache';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as logs from 'aws-cdk-lib/aws-logs';
import { Construct } from 'constructs';

export class AddressAutocompleteStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const environment = this.node.tryGetContext('environment') || 'dev';

    // VPC (use existing or create new)
    const vpc = ec2.Vpc.fromLookup(this, 'ExistingVPC', {
      vpcId: this.node.tryGetContext('vpcId')
    });

    // Security Groups
    const lambdaSecurityGroup = new ec2.SecurityGroup(this, 'LambdaSG', {
      vpc,
      description: 'Security group for Address Autocomplete Lambda',
      allowAllOutbound: true
    });

    const cacheSecurityGroup = new ec2.SecurityGroup(this, 'CacheSG', {
      vpc,
      description: 'Security group for ElastiCache Redis'
    });

    cacheSecurityGroup.addIngressRule(
      lambdaSecurityGroup,
      ec2.Port.tcp(6379),
      'Allow Lambda to access Redis'
    );

    // ElastiCache Redis Serverless
    const cacheSubnetGroup = new elasticache.CfnSubnetGroup(this, 'CacheSubnetGroup', {
      description: 'Subnet group for Address Autocomplete cache',
      subnetIds: vpc.privateSubnets.map(subnet => subnet.subnetId),
      cacheSubnetGroupName: `address-autocomplete-${environment}`
    });

    const redisCache = new elasticache.CfnServerlessCache(this, 'RedisCache', {
      serverlessCacheName: `address-autocomplete-${environment}`,
      engine: 'redis',
      description: 'Redis cache for address autocomplete',
      securityGroupIds: [cacheSecurityGroup.securityGroupId],
      subnetIds: vpc.privateSubnets.map(subnet => subnet.subnetId),
      dailySnapshotTime: '03:00',
      snapshotRetentionLimit: 7
    });

    // Amazon Location Service Place Index
    const placeIndex = new cdk.aws_location.CfnPlaceIndex(this, 'PlaceIndex', {
      indexName: `address-autocomplete-${environment}`,
      dataSource: 'Esri', // Includes PSMA data for Australia
      description: 'Place index for Australian address autocomplete',
      pricingPlan: 'RequestBasedUsage'
    });

    // Lambda Function
    const autocompleteLambda = new lambda.Function(this, 'AutocompleteFunction', {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'handlers/autocomplete.handler',
      code: lambda.Code.fromAsset('../dist'), // Built TypeScript code
      functionName: `address-autocomplete-${environment}`,
      description: 'Address autocomplete using Amazon Location Service',
      memorySize: 512,
      timeout: cdk.Duration.seconds(10),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      securityGroups: [lambdaSecurityGroup],
      environment: {
        ELASTICACHE_ENDPOINT: redisCache.attrEndpointAddress,
        LOCATION_SERVICE_PLACE_INDEX: placeIndex.indexName,
        CACHE_TTL_SECONDS: '3600',
        MAX_RESULTS_DEFAULT: '5',
        MAX_RESULTS_LIMIT: '20',
        APPCONFIG_APP_ID: this.node.tryGetContext('appConfigAppId'),
        APPCONFIG_ENV_ID: environment,
        APPCONFIG_CONFIG_PROFILE: 'address-autocomplete-config',
        NODE_ENV: environment
      },
      logRetention: logs.RetentionDays.ONE_WEEK,
      tracing: lambda.Tracing.ACTIVE
    });

    // Grant permissions
    autocompleteLambda.addToRolePolicy(new iam.PolicyStatement({
      effect: iam.Effect.ALLOW,
      actions: [
        'geo:SearchPlaceIndexForSuggestions',
        'geo:GetPlace',
        'geo-places:Autocomplete'
      ],
      resources: [placeIndex.attrArn]
    }));

    autocompleteLambda.addToRolePolicy(new iam.PolicyStatement({
      effect: iam.Effect.ALLOW,
      actions: ['appconfig:GetConfiguration'],
      resources: ['*'] // Scope down to specific AppConfig resource
    }));

    // API Gateway
    const api = new apigateway.RestApi(this, 'AddressAutocompleteAPI', {
      restApiName: `address-autocomplete-${environment}`,
      description: 'Address Autocomplete API using Amazon Location Service',
      deployOptions: {
        stageName: environment,
        tracingEnabled: true,
        loggingLevel: apigateway.MethodLoggingLevel.INFO,
        dataTraceEnabled: true,
        metricsEnabled: true
      },
      defaultCorsPreflightOptions: {
        allowOrigins: apigateway.Cors.ALL_ORIGINS,
        allowMethods: ['GET', 'POST', 'OPTIONS'],
        allowHeaders: ['Content-Type', 'Authorization', 'X-API-Key']
      }
    });

    // API Resources
    const addressResource = api.root.addResource('address');
    const searchResource = addressResource.addResource('search');

    // Lambda Integration
    const lambdaIntegration = new apigateway.LambdaIntegration(autocompleteLambda, {
      proxy: true,
      integrationResponses: [{
        statusCode: '200',
        responseParameters: {
          'method.response.header.Access-Control-Allow-Origin': "'*'"
        }
      }]
    });

    // Request Validator
    const requestValidator = new apigateway.RequestValidator(this, 'RequestValidator', {
      restApi: api,
      validateRequestParameters: true,
      validateRequestBody: false
    });

    // GET /address/search
    searchResource.addMethod('GET', lambdaIntegration, {
      requestValidator,
      requestParameters: {
        'method.request.querystring.query': true,  // Required
        'method.request.querystring.locality': false,
        'method.request.querystring.state': false,
        'method.request.querystring.maxResults': false
      },
      methodResponses: [{
        statusCode: '200',
        responseParameters: {
          'method.response.header.Access-Control-Allow-Origin': true
        }
      }]
    });

    // Usage Plan
    const usagePlan = api.addUsagePlan('StandardUsagePlan', {
      name: `address-autocomplete-standard-${environment}`,
      throttle: {
        rateLimit: 100,
        burstLimit: 50
      },
      quota: {
        limit: 100000,
        period: apigateway.Period.MONTH
      }
    });

    usagePlan.addApiStage({
      stage: api.deploymentStage
    });

    // CloudWatch Alarms
    const errorAlarm = autocompleteLambda.metricErrors({
      period: cdk.Duration.minutes(5),
      statistic: 'Sum'
    }).createAlarm(this, 'ErrorAlarm', {
      threshold: 10,
      evaluationPeriods: 1,
      alarmDescription: 'Alert when Lambda errors exceed 10 in 5 minutes'
    });

    const durationAlarm = autocompleteLambda.metricDuration({
      period: cdk.Duration.minutes(5),
      statistic: 'Average'
    }).createAlarm(this, 'DurationAlarm', {
      threshold: 5000, // 5 seconds
      evaluationPeriods: 2,
      alarmDescription: 'Alert when average Lambda duration exceeds 5s'
    });

    // Outputs
    new cdk.CfnOutput(this, 'ApiEndpoint', {
      value: api.url,
      description: 'API Gateway endpoint URL'
    });

    new cdk.CfnOutput(this, 'ApiId', {
      value: api.restApiId,
      description: 'API Gateway ID'
    });

    new cdk.CfnOutput(this, 'CacheEndpoint', {
      value: redisCache.attrEndpointAddress,
      description: 'ElastiCache Redis endpoint'
    });

    new cdk.CfnOutput(this, 'PlaceIndexName', {
      value: placeIndex.indexName,
      description: 'Amazon Location Service Place Index name'
    });
  }
}
```

**Deployment Commands:**

```bash
# Install dependencies
npm install

# Build TypeScript
npm run build

# Synthesize CloudFormation template
cdk synth --context environment=dev

# Deploy to dev
cdk deploy --context environment=dev --context vpcId=vpc-12345

# Deploy to prod
cdk deploy --context environment=prod --context vpcId=vpc-67890
```

### OpenAPI Specification

**openapi.yaml:**
```yaml
openapi: 3.0.3
info:
  title: Address Autocomplete API
  description: Address autocomplete API using Amazon Location Service with PSMA data for Australian addresses
  version: 1.0.0
  contact:
    name: API Support
    email: support@example.com

servers:
  - url: https://api.example.com/dev
    description: Development
  - url: https://api.example.com/prod
    description: Production

security:
  - BearerAuth: []
  - ApiKeyAuth: []

paths:
  /address/search:
    get:
      summary: Search for address autocomplete suggestions
      description: Returns address autocomplete suggestions based on partial query input
      operationId: searchAddresses
      tags:
        - Address Autocomplete
      parameters:
        - name: query
          in: query
          description: Free-text address query (partial or full address)
          required: true
          schema:
            type: string
            minLength: 1
            maxLength: 200
          example: "123 George St"
        - name: locality
          in: query
          description: Optional city/suburb filter
          required: false
          schema:
            type: string
            minLength: 1
            maxLength: 100
          example: "Sydney"
        - name: state
          in: query
          description: Optional state code filter (NSW, VIC, QLD, SA, WA, TAS, NT, ACT)
          required: false
          schema:
            type: string
            enum: [NSW, VIC, QLD, SA, WA, TAS, NT, ACT]
          example: "NSW"
        - name: maxResults
          in: query
          description: Maximum number of results to return
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 20
            default: 5
          example: 5
      responses:
        '200':
          description: Successful response with address suggestions
          headers:
            X-Cache-Status:
              description: Cache hit or miss
              schema:
                type: string
                enum: [HIT, MISS]
            X-Response-Time:
              description: Response time in milliseconds
              schema:
                type: integer
            X-Request-Id:
              description: Request ID for debugging
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AutocompleteResponse'
              examples:
                cached_result:
                  summary: Cached result
                  value:
                    results:
                      - placeId: "here:pds:place:036y1tcb-abc123"
                        title: "123 George St, Sydney NSW 2000, Australia"
                        address:
                          addressNumber: "123"
                          street: "George St"
                          locality: "Sydney"
                          region: "New South Wales"
                          regionCode: "NSW"
                          postalCode: "2000"
                          country: "Australia"
                          countryCode: "AU"
                          label: "123 George St, Sydney NSW 2000"
                        position:
                          longitude: 151.2073
                          latitude: -33.8688
                    metadata:
                      cached: true
                      queryTime: 45
                      provider: "amazon-location"
                      resultCount: 1
        '400':
          description: Bad request - invalid parameters
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                error:
                  code: "VALIDATION_ERROR"
                  message: "Query parameter must be between 1 and 200 characters"
                  details:
                    field: "query"
                    constraint: "minLength:1"
                requestId: "a1b2c3d4-e5f6-7890"
                timestamp: "2026-02-03T14:30:00.123Z"
        '401':
          description: Unauthorized - missing or invalid authentication
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: Too many requests - rate limit exceeded
          headers:
            Retry-After:
              description: Seconds to wait before retrying
              schema:
                type: integer
            X-RateLimit-Limit:
              schema:
                type: integer
            X-RateLimit-Remaining:
              schema:
                type: integer
            X-RateLimit-Reset:
              schema:
                type: integer
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '503':
          description: Service unavailable
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT token authentication
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
      description: API key for usage plan tracking

  schemas:
    AutocompleteResponse:
      type: object
      required:
        - results
        - metadata
      properties:
        results:
          type: array
          items:
            $ref: '#/components/schemas/AddressResult'
        metadata:
          $ref: '#/components/schemas/ResponseMetadata'

    AddressResult:
      type: object
      required:
        - placeId
        - title
        - address
      properties:
        placeId:
          type: string
          description: Unique identifier for the place
          example: "here:pds:place:036y1tcb-abc123"
        title:
          type: string
          description: Human-readable full address
          example: "123 George St, Sydney NSW 2000, Australia"
        address:
          $ref: '#/components/schemas/Address'
        position:
          $ref: '#/components/schemas/Position'
        distance:
          type: number
          description: Distance from bias position in meters
          example: 1250.5

    Address:
      type: object
      properties:
        addressNumber:
          type: string
          example: "123"
        street:
          type: string
          example: "George St"
        locality:
          type: string
          example: "Sydney"
        district:
          type: string
          example: "Sydney"
        region:
          type: string
          example: "New South Wales"
        regionCode:
          type: string
          example: "NSW"
        postalCode:
          type: string
          example: "2000"
        country:
          type: string
          example: "Australia"
        countryCode:
          type: string
          example: "AU"
        label:
          type: string
          example: "123 George St, Sydney NSW 2000"

    Position:
      type: object
      required:
        - longitude
        - latitude
      properties:
        longitude:
          type: number
          format: double
          example: 151.2073
        latitude:
          type: number
          format: double
          example: -33.8688

    ResponseMetadata:
      type: object
      required:
        - cached
        - queryTime
        - provider
        - resultCount
      properties:
        cached:
          type: boolean
          description: Whether result was served from cache
        queryTime:
          type: integer
          description: Query processing time in milliseconds
        provider:
          type: string
          description: Data provider name
          example: "amazon-location"
        resultCount:
          type: integer
          description: Number of results returned

    ErrorResponse:
      type: object
      required:
        - error
        - requestId
        - timestamp
      properties:
        error:
          type: object
          required:
            - code
            - message
          properties:
            code:
              type: string
              description: Machine-readable error code
              enum:
                - VALIDATION_ERROR
                - UNAUTHORIZED
                - RATE_LIMIT_EXCEEDED
                - SERVICE_UNAVAILABLE
                - INTERNAL_ERROR
            message:
              type: string
              description: Human-readable error message
            details:
              type: object
              description: Additional error context
              additionalProperties: true
        requestId:
          type: string
          description: CloudWatch request ID for debugging
        timestamp:
          type: string
          format: date-time
          description: Error timestamp in ISO 8601 format
```

### Observability

#### CloudWatch Metrics

**Custom Metrics:**

| Metric Name | Unit | Description | Alarm Threshold |
|-------------|------|-------------|-----------------|
| CacheHitRate | Percent | Cache hit rate | <50% for 5 min |
| CacheLookupDuration | Milliseconds | Cache read latency | p95 >50ms |
| LocationServiceDuration | Milliseconds | Amazon Location Service API latency | p95 >800ms |
| RequestDuration | Milliseconds | Total request duration | p95 >250ms (cached), p95 >800ms (uncached) |
| ErrorCount | Count | Total errors | >10 in 5 min |
| ValidationErrorCount | Count | Validation errors | Info only |
| RateLimitExceededCount | Count | Rate limit violations | >50 in 5 min |

**Lambda Metrics (Built-in):**
- Invocations
- Errors
- Duration (p50, p95, p99, max)
- Throttles
- ConcurrentExecutions

**API Gateway Metrics (Built-in):**
- Count (requests)
- 4XXError
- 5XXError
- Latency (p50, p95, p99)

#### CloudWatch Dashboard

```typescript
const dashboard = new cloudwatch.Dashboard(this, 'AddressAutocompleteDashboard', {
  dashboardName: `address-autocomplete-${environment}`
});

dashboard.addWidgets(
  new cloudwatch.GraphWidget({
    title: 'Request Rate',
    left: [
      api.metricCount({ statistic: 'Sum', period: Duration.minutes(1) })
    ]
  }),
  new cloudwatch.GraphWidget({
    title: 'Error Rate',
    left: [
      api.metric4XXError({ statistic: 'Sum', period: Duration.minutes(1) }),
      api.metric5XXError({ statistic: 'Sum', period: Duration.minutes(1) })
    ]
  }),
  new cloudwatch.GraphWidget({
    title: 'Cache Hit Rate',
    left: [
      new cloudwatch.MathExpression({
        expression: 'hits / (hits + misses) * 100',
        usingMetrics: {
          hits: new cloudwatch.Metric({ namespace: 'AddressAutocomplete', metricName: 'CacheHit' }),
          misses: new cloudwatch.Metric({ namespace: 'AddressAutocomplete', metricName: 'CacheMiss' })
        }
      })
    ]
  }),
  new cloudwatch.GraphWidget({
    title: 'Response Time (p95)',
    left: [
      autocompleteLambda.metricDuration({ statistic: 'p95', period: Duration.minutes(1) })
    ]
  })
);
```

#### X-Ray Tracing

Enabled for Lambda and API Gateway to trace:
- API Gateway → Lambda invocation
- Lambda → ElastiCache calls
- Lambda → Amazon Location Service calls
- Lambda → AppConfig calls

**Trace Annotations:**
```typescript
import AWSXRay from 'aws-xray-sdk-core';

AWSXRay.captureAsyncFunc('cacheCheck', async (subsegment) => {
  subsegment?.addAnnotation('cacheKey', cacheKey);
  const result = await cacheService.get(cacheKey);
  subsegment?.addMetadata('cacheHit', !!result);
  subsegment?.close();
  return result;
});
```

## Cost Estimates

### Monthly Cost Breakdown (Conservative Estimates)

**Assumptions:**
- 50,000 requests/month (initially)
- 70% cache hit rate
- Average 5 results per query
- ap-southeast-2 (Sydney) pricing

| Service | Usage | Unit Cost | Monthly Cost |
|---------|-------|-----------|--------------|
| **Amazon Location Service** | 15,000 searches (30% cache miss) | $5.00/1K searches | $75.00 |
| **API Gateway** | 50,000 requests | $3.50/million | $0.18 |
| **Lambda** | 50K invocations, 512MB, 250ms avg | $0.20/million requests + $0.0000166667/GB-sec | $1.04 |
| **ElastiCache Redis Serverless** | 1-2GB cache, 50K requests | ~$0.125/GB-hour + data processing | $20-30 |
| **CloudWatch Logs** | 5GB logs/month | $0.50/GB ingested | $2.50 |
| **CloudWatch Metrics** | 20 custom metrics | $0.30/metric | $6.00 |
| **Data Transfer** | 5GB outbound | $0.114/GB | $0.57 |
| **NAT Gateway** | Existing shared resource | N/A | $0 (shared) |
| **VPC Endpoints** | Location Service VPC endpoint (optional) | $0.01/hour | $7.30 |
| | | **Total** | **~$112.59/month** |

**Cost at Scale (500,000 requests/month):**

| Service | Usage | Monthly Cost |
|---------|-------|--------------|
| Amazon Location Service | 150K searches | $750.00 |
| API Gateway | 500K requests | $1.75 |
| Lambda | 500K invocations | $10.40 |
| ElastiCache Redis | 3-5GB cache | $50-70 |
| CloudWatch | Logs + Metrics | $20.00 |
| Data Transfer | 50GB | $5.70 |
| | **Total** | **~$838-858/month** |

**Cost Optimization Strategies:**

1. **Increase Cache Hit Rate:**
   - Target 80%+ cache hit rate reduces Location Service costs by 10%
   - Optimize cache TTL (currently 1 hour)

2. **Optimize Lambda Memory:**
   - Test with 256MB instead of 512MB (50% cost reduction)
   - May increase duration slightly but reduce GB-sec cost

3. **Use Reserved Capacity (ElastiCache):**
   - If traffic is predictable, reserved instances save 30-50%

4. **Batch Queries (future optimization):**
   - Client-side debouncing reduces API calls
   - Implement query batching for multiple concurrent requests

5. **Monitor and Adjust:**
   - Review CloudWatch Cost Explorer weekly
   - Set budget alerts at $100, $200, $500/month

### Performance Budget

| Metric | Target | Current Estimate | Headroom |
|--------|--------|------------------|----------|
| p50 Response Time (cached) | <100ms | ~50ms | 50ms |
| p95 Response Time (cached) | <250ms | ~150ms | 100ms |
| p50 Response Time (uncached) | <500ms | ~400ms | 100ms |
| p95 Response Time (uncached) | <800ms | ~650ms | 150ms |
| Cache Hit Rate | >70% | ~70% | 0% (needs optimization) |
| Error Rate | <1% | <0.5% | 0.5% |

## Implementation Plan

### Phase 1: Foundation (Day 1 - 8 hours)

**Step 1: Project Setup (1 hour)**
- Initialize CDK project and TypeScript configuration
- Set up project structure (handlers, services, models, utils)
- Install dependencies (aws-sdk, ioredis, jest, typescript)
- Configure ESLint, Prettier, Jest

**Files to create:**
- `infrastructure/package.json`, `tsconfig.json`, `cdk.json`
- `src/package.json`, `tsconfig.json`, `jest.config.js`
- `.eslintrc.js`, `.prettierrc`

**Step 2: Infrastructure Setup (3 hours)**
- Create CDK stack with VPC lookup
- Deploy ElastiCache Redis Serverless
- Create Amazon Location Service Place Index (Esri/PSMA)
- Set up security groups for Lambda and ElastiCache
- Deploy base Lambda function (hello world)
- Create API Gateway with placeholder endpoint

**Files to create:**
- `infrastructure/lib/address-autocomplete-stack.ts`
- `infrastructure/bin/app.ts`

**Commands:**
```bash
cdk bootstrap --context environment=dev
cdk synth --context environment=dev --context vpcId=vpc-12345
cdk deploy --context environment=dev --context vpcId=vpc-12345
```

**Step 3: Core Services Implementation (4 hours)**
- Implement `LocationService` with Amazon Location Service SDK
- Implement `CacheService` with ioredis client
- Implement `ConfigService` with AppConfig integration
- Create data models (request, response, errors)
- Implement validation utils and normalization utils
- Create structured logger and metrics utils

**Files to create:**
- `src/services/location-service.ts`
- `src/services/cache-service.ts`
- `src/services/config-service.ts`
- `src/models/request.ts`, `response.ts`, `errors.ts`
- `src/utils/validation.ts`, `normalization.ts`, `logger.ts`, `metrics.ts`

**Testing:**
- Unit tests for validation and normalization
- Integration test for Location Service (manual via AWS console)
- Integration test for ElastiCache connection

### Phase 2: Core Features (Day 2 - 8 hours)

**Step 4: Lambda Handler Implementation (3 hours)**
- Implement autocomplete Lambda handler
- Add request validation logic
- Integrate cache check and write
- Integrate Amazon Location Service calls
- Add error handling with proper HTTP status codes
- Implement structured logging
- Add CloudWatch metrics emission

**Files to create:**
- `src/handlers/autocomplete.ts`

**Testing:**
- Unit tests for handler logic (mocked services)
- Integration test with local Lambda runtime (SAM Local or LocalStack)

**Step 5: API Gateway Integration (2 hours)**
- Configure API Gateway resources and methods
- Add request parameter validation
- Set up Lambda integration with proxy
- Configure CORS headers
- Add API Gateway stage variables
- Deploy and test end-to-end

**Files to update:**
- `infrastructure/lib/address-autocomplete-stack.ts` (API Gateway section)

**Testing:**
- End-to-end test via curl or Postman
- Test CORS preflight requests
- Test error responses (400, 401, 429, 500)

**Step 6: Rate Limiting & Feature Flags (3 hours)**
- Configure API Gateway usage plans
- Add API key management
- Implement custom Lambda rate limiting (ioredis)
- Create AppConfig configuration profile
- Implement feature flag checks in Lambda
- Add environment-specific configuration

**Files to create:**
- `infrastructure/lib/appconfig-construct.ts`
- `src/config/feature-flags.json`

**Testing:**
- Test rate limiting (100 req/min threshold)
- Test feature flag enable/disable
- Verify API key throttling

### Phase 3: Testing, Documentation & Deployment (Day 3 - 8 hours)

**Step 7: Comprehensive Testing (4 hours)**
- Write unit tests for all services (>80% coverage)
- Write integration tests for API endpoint
- Test cache hit/miss scenarios
- Test error handling (Location Service failures)
- Load test with 100 concurrent requests
- Validate performance targets (<250ms cached)

**Files to create:**
- `tests/unit/services/*.test.ts`
- `tests/unit/utils/*.test.ts`
- `tests/integration/autocomplete.test.ts`
- `tests/load/autocomplete-load.test.ts` (using artillery or k6)

**Commands:**
```bash
npm test -- --coverage
npm run test:integration
npm run test:load
```

**Step 8: OpenAPI Documentation (2 hours)**
- Create OpenAPI 3.0 specification
- Document all endpoints, parameters, responses
- Add request/response examples
- Generate Postman collection from OpenAPI spec
- Create README with API usage instructions

**Files to create:**
- `docs/openapi.yaml`
- `docs/API_USAGE.md`
- `README.md`

**Step 9: Monitoring & Deployment (2 hours)**
- Create CloudWatch dashboard
- Set up CloudWatch alarms (error rate, duration, cache hit rate)
- Configure X-Ray tracing
- Deploy to staging environment
- Run smoke tests
- Deploy to production
- Document deployment process

**Files to create:**
- `infrastructure/lib/monitoring-construct.ts`
- `docs/DEPLOYMENT.md`
- `docs/MONITORING.md`

**Commands:**
```bash
cdk deploy --context environment=staging
# Run smoke tests
cdk deploy --context environment=prod
```

**Step 10: Handoff & Documentation (Optional)**
- Create runbook for operational issues
- Document troubleshooting steps
- Create alerts and escalation procedures
- Knowledge transfer session with team

**Files to create:**
- `docs/RUNBOOK.md`
- `docs/TROUBLESHOOTING.md`

## Testing Strategy

### Unit Tests

**Coverage Target:** >80% code coverage

**Test Files:**
- `tests/unit/services/location-service.test.ts`
- `tests/unit/services/cache-service.test.ts`
- `tests/unit/services/config-service.test.ts`
- `tests/unit/utils/validation.test.ts`
- `tests/unit/utils/normalization.test.ts`
- `tests/unit/handlers/autocomplete.test.ts`

**Example: validation.test.ts**
```typescript
import { validateRequest } from '../../src/utils/validation';
import { ValidationError } from '../../src/models/errors';

describe('validateRequest', () => {
  it('should validate valid request', () => {
    const params = { query: '123 George St', maxResults: '5' };
    const result = validateRequest(params);
    expect(result.query).toBe('123 George St');
    expect(result.maxResults).toBe(5);
  });

  it('should throw ValidationError for missing query', () => {
    expect(() => validateRequest({})).toThrow(ValidationError);
  });

  it('should throw ValidationError for query too long', () => {
    const longQuery = 'a'.repeat(201);
    expect(() => validateRequest({ query: longQuery })).toThrow(ValidationError);
  });

  it('should validate state codes', () => {
    const params = { query: 'test', state: 'NSW' };
    const result = validateRequest(params);
    expect(result.state).toBe('NSW');
  });

  it('should throw ValidationError for invalid state code', () => {
    const params = { query: 'test', state: 'INVALID' };
    expect(() => validateRequest(params)).toThrow(ValidationError);
  });
});
```

### Integration Tests

**Test Scenarios:**
1. **Cache Hit:** Query → Cache hit → Return cached result
2. **Cache Miss:** Query → Cache miss → Location Service → Cache write → Return result
3. **Location Service Failure:** Query → Cache miss → Location Service error → Return 503
4. **Rate Limiting:** Exceed 100 req/min → Return 429
5. **Authentication Failure:** Invalid JWT → Return 401

**Example: autocomplete.test.ts**
```typescript
import { handler } from '../../src/handlers/autocomplete';
import { APIGatewayProxyEvent, Context } from 'aws-lambda';

describe('Autocomplete Handler Integration', () => {
  it('should return results for valid query', async () => {
    const event: APIGatewayProxyEvent = {
      queryStringParameters: { query: '123 George St', state: 'NSW' },
      requestContext: {
        requestId: 'test-request-id',
        authorizer: { userId: 'user-123' }
      }
    } as any;

    const context: Context = {
      requestId: 'test-request-id',
      functionName: 'test-function'
    } as any;

    const response = await handler(event, context);

    expect(response.statusCode).toBe(200);
    const body = JSON.parse(response.body);
    expect(body.results).toBeInstanceOf(Array);
    expect(body.metadata.provider).toBe('amazon-location');
  });

  it('should return 400 for missing query parameter', async () => {
    const event: APIGatewayProxyEvent = {
      queryStringParameters: {},
      requestContext: { requestId: 'test-request-id' }
    } as any;

    const response = await handler(event, {} as Context);

    expect(response.statusCode).toBe(400);
    const body = JSON.parse(response.body);
    expect(body.error.code).toBe('VALIDATION_ERROR');
  });
});
```

### Load Tests

**Tool:** Artillery or k6

**Test Scenario:**
- Ramp up: 0 → 100 requests/second over 2 minutes
- Sustained: 100 requests/second for 5 minutes
- Target: p95 response time <250ms (cached), <800ms (uncached)

**artillery.yml:**
```yaml
config:
  target: https://api.example.com/dev
  phases:
    - duration: 120
      arrivalRate: 1
      rampTo: 100
      name: Ramp up
    - duration: 300
      arrivalRate: 100
      name: Sustained load
  plugins:
    expect: {}

scenarios:
  - name: Address Autocomplete
    flow:
      - get:
          url: "/address/search?query=123%20George%20St&state=NSW"
          headers:
            Authorization: "Bearer {{ $env.JWT_TOKEN }}"
          expect:
            - statusCode: 200
            - hasProperty: results
            - contentType: json
```

**Run:**
```bash
JWT_TOKEN=<token> artillery run tests/load/artillery.yml
```

### End-to-End Tests

**Manual Testing Checklist:**
- [ ] GET /address/search with valid query returns 200
- [ ] GET /address/search with locality filter returns filtered results
- [ ] GET /address/search with state filter returns filtered results
- [ ] GET /address/search with maxResults=10 returns 10 results
- [ ] GET /address/search without auth returns 401
- [ ] GET /address/search with invalid query returns 400
- [ ] Cache hit returns metadata.cached=true
- [ ] Cache miss returns metadata.cached=false
- [ ] Rate limit exceeded returns 429
- [ ] CloudWatch logs show structured JSON format
- [ ] CloudWatch metrics show CacheHit/CacheMiss
- [ ] X-Ray trace shows all service calls

## Risks & Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Amazon Location Service Pricing** | High | Medium | Monitor daily costs via CloudWatch; set budget alert at $100/month; implement caching to reduce API calls by 70%+ |
| **Location Service Rate Limiting** | Medium | Medium | Implement exponential backoff retry (3 attempts); use circuit breaker after 50% error rate; cache aggressively |
| **ElastiCache Connection Failures** | Medium | Low | Fail open strategy (skip cache, query Location Service); add health checks; connection retry with backoff |
| **Cold Start Latency** | Medium | Medium | Use Lambda SnapStart (not available for Node.js 18 yet) or Provisioned Concurrency for production; optimize bundle size |
| **VPC NAT Gateway Costs** | Medium | Low | Already using existing NAT Gateway (shared cost); monitor data transfer costs |
| **PSMA Data Coverage Gaps** | Low | Low | Esri provider includes PSMA data for Australia; handle empty results gracefully; log coverage gaps for analysis |
| **Cache Memory Exhaustion** | Low | Low | ElastiCache Serverless auto-scales 1GB-5GB; monitor eviction rate; adjust TTL if needed |
| **JWT Token Validation Latency** | Low | Low | Lambda authorizer caches policy decisions (300s TTL); optimize authorizer code |
| **Feature Flag Fetch Latency** | Low | Low | AppConfig uses local cache with 30s refresh; fetch asynchronously in background |
| **CloudWatch Logs Cost Overrun** | Low | Medium | Set 7-day retention; filter out debug logs in production; use log sampling for high-volume |

## Open Questions

1. **Q:** Should we support reverse geocoding (lat/lng → address)?
   **Decision Needed By:** Post-MVP (not in scope for 3-day implementation)
   **Options:** Yes (separate endpoint) vs No (out of scope)
   **Recommendation:** No for MVP, add later if needed

2. **Q:** Should we cache negative results (no matches found)?
   **Decision Needed By:** Day 1
   **Options:** Yes (reduce Location Service calls) vs No (simpler implementation)
   **Recommendation:** Yes, cache with shorter TTL (300s)

3. **Q:** What's the existing auth middleware JWT payload format?
   **Decision Needed By:** Day 1 (before Lambda authorizer integration)
   **Action:** Review existing Lambda authorizer code and documentation

4. **Q:** Should we support multiple data providers (fallback to AusPost if PSMA unavailable)?
   **Decision Needed By:** Post-MVP
   **Options:** Yes (resilience) vs No (complexity)
   **Recommendation:** No for MVP, Amazon Location Service has built-in redundancy

5. **Q:** Do we need audit logging for compliance?
   **Decision Needed By:** Day 2 (before production deployment)
   **Options:** Yes (log all queries to S3) vs No (CloudWatch only)
   **Action:** Check with compliance team

6. **Q:** Should maxResults default be configurable via AppConfig?
   **Decision Needed By:** Day 2
   **Options:** Yes (flexible) vs No (hardcoded in Lambda)
   **Recommendation:** Yes, use AppConfig for runtime configuration

## References

- [Amazon Location Service Autocomplete API Documentation](https://docs.aws.amazon.com/location/latest/APIReference/API_geoplaces_Autocomplete.html)
- [Amazon Location Service Developer Guide](https://docs.aws.amazon.com/location/latest/developerguide/autocomplete.html)
- [ElastiCache Redis Serverless Documentation](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/serverless.html)
- [API Gateway Usage Plans and Rate Limiting](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [AWS Well-Architected Framework - Serverless Applications](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/welcome.html)
- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
- [AWS CDK TypeScript Reference](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-construct-library.html)
- [ioredis Documentation](https://github.com/redis/ioredis)
- [AWS X-Ray SDK for Node.js](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-nodejs.html)

---

**Document Version:** 1.0
**Last Updated:** 2026-02-03
**Author:** Technical Lead
**Status:** Ready for Implementation
**Review:** Pending team review and approval
