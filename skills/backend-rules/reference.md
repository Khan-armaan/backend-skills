# Backend Development Rules for LLM Assistance

This document provides comprehensive rules and guidelines for backend development. Use these rules when generating, reviewing, or modifying backend code.

---

## TABLE OF CONTENTS

1. Architecture & Layer Design
2. API Design & Routing
3. Validation & Transformation
4. Error Handling
5. Serialization & Deserialization
6. Caching
7. Security
8. Configuration Management
9. Graceful Shutdown
10. Best Practices Summary

---

## 1. ARCHITECTURE & LAYER DESIGN

### Request Lifecycle Flow

```
Entry Point → Middleware Chain → Handler Layer → Service Layer → Repository Layer → Database
                                                                                    ↓
Response ← Middleware Chain (reverse) ← Handler ← Service ← Repository ← Database
```

### Three-Layer Architecture

**Layer 1: Handler (Routes/Controllers)**
- MUST handle HTTP protocol concerns only
- MUST deserialize HTTP request body into language data structures
- MUST perform initial validation and transformation
- MUST extract request parameters (path, query, headers, body)
- MUST invoke service layer methods
- MUST serialize service responses back to HTTP format
- MUST set appropriate HTTP status codes and headers
- MUST NOT contain business logic
- MUST NOT directly access database

**Layer 2: Service (Business Logic)**
- MUST implement all business rules and workflows
- MUST coordinate operations across multiple repositories
- MUST manage transactions
- MUST perform business-level validations
- MUST be completely independent of HTTP protocol
- MUST NOT know about HTTP request/response objects
- MUST NOT import HTTP libraries
- MUST NOT execute direct database queries

**Layer 3: Repository (Data Access)**
- MUST handle all database operations and data persistence
- MUST construct database queries (SQL, ORM, query builders)
- MUST execute CRUD operations
- MUST map database rows/documents to domain models
- MUST handle database-specific errors
- MUST NOT contain business logic
- MUST NOT have HTTP concerns

### Dependency Direction Rules

```
Handler → Service → Repository → Database
```

- NEVER reverse: Repository MUST NOT call Service
- Service MUST NOT call Handler
- Dependencies flow in ONE direction only

### Directory Structure

```
project/
├── cmd/server/main.*           # Server initialization & startup
├── internal/
│   ├── handler/                # HTTP Layer (Routes)
│   │   ├── user_handler.*
│   │   └── router.*
│   ├── service/                # Business Logic Layer
│   │   └── user_service.*
│   ├── repository/             # Data Access Layer
│   │   └── user_repository.*
│   ├── middleware/             # Cross-cutting concerns
│   │   ├── cors.*
│   │   ├── logging.*
│   │   ├── auth.*
│   │   ├── request_context.*
│   │   ├── validation.*
│   │   └── error_handler.*
│   ├── model/                  # Domain Models
│   └── dto/                    # Data Transfer Objects
│       ├── request/
│       └── response/
├── pkg/                        # Public reusable packages
└── config/                     # Configuration files
```

### Middleware Execution Order

REQUEST FLOW (in order):
1. CORS Middleware - Handle preflight, set CORS headers
2. Logging Middleware - Log incoming request details
3. Request Context Middleware - Set request ID, trace ID
4. Authentication Middleware - Verify auth token, set user
5. Deserialization - Parse request body
6. Validation Middleware - Validate request structure
7. Handler Execution - Business logic executes
8. Global Error Handler - Catch and format errors

RESPONSE FLOW (reverse order)

---

## 2. API DESIGN & ROUTING

### RESTful URL Design Rules

**Resource Naming - MANDATORY:**
- MUST use plural nouns for resource names
- MUST NOT use verbs in URLs
- MUST use lowercase letters
- MUST use hyphens for multi-word URLs (NOT underscores)

```
✓ CORRECT: /users, /products, /order-history, /product-categories
✗ WRONG: /getUsers, /user, /createProduct, /product_categories, /ProductCategories
```

### HTTP Methods Mapping

| HTTP Method | Operation           | Example                    |
|-------------|---------------------|----------------------------|
| GET         | Read/Retrieve       | GET /users, GET /users/{id}|
| POST        | Create              | POST /users                |
| PUT         | Update (full)       | PUT /users/{id}            |
| PATCH       | Update (partial)    | PATCH /users/{id}          |
| DELETE      | Delete              | DELETE /users/{id}         |

**Rules:**
- MUST use PATCH for partial updates (NOT PUT)
- MUST use POST with descriptive action endpoint for non-CRUD operations

```
POST /users/{userId}/send-welcome-email
POST /orders/{orderId}/cancel
POST /products/{productId}/publish
```

### Hierarchical Relationships

MUST use hierarchical URLs for relationships:

```
/users/{userId}/orders
/products/{productId}/reviews
/organizations/{orgId}/teams/{teamId}/members
```

### Pagination - MANDATORY

ALL GET APIs returning lists MUST be paginated:

```
GET /users?page=2&limit=20
```

**Required Query Parameters:**
- `page`: Page number (default: 1)
- `limit`: Items per page (default: 20)

**Required Response Structure:**
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

### Sorting

MUST implement sorting using query parameters:

```
GET /products?sortBy=price&sortOrder=desc
```

- `sortBy`: Field to sort by
- `sortOrder`: Direction (asc or desc)

### Filtering

MUST allow filtering using descriptive query parameters:

```
GET /products?category=electronics&minPrice=100&maxPrice=500
GET /users?status=active&role=admin
```

### Route Versioning

MUST implement versioning for public APIs:

```
/v1/users
/v2/users
/api/v1/users
```

### Catch-All Routes

MUST define catch-all routes LAST to handle 404:

```javascript
app.use('*', (req, res) => {
  res.status(404).json({
    error: 'Route not found',
    path: req.originalUrl
  });
});
```

### URL Parameters Rules

**Path Parameters:**
- Part of URL path, no ? or = syntax
- Usually required values
- Example: /users/123

**Query Parameters:**
- After ? in URL, using key=value pairs
- Optional by nature
- Example: /users?page=2&limit=10

---

## 3. VALIDATION & TRANSFORMATION

### Validation Pipeline Order - MANDATORY

1. Security Sanitization (first)
2. Syntactic Validation
3. Data Type Validation
4. Semantic Validation
5. Data Transformation
6. Business Logic (last)

### Syntactic Validation Rules

MUST validate:
- Presence of required fields
- Correct field naming
- Valid JSON/XML structure
- URL and path parameter format
- Pattern matching (email, phone, etc.)

**On Failure:** Return immediate error, do NOT proceed

### Data Type Validation Rules

MUST validate:
- Primitive types: string, number, boolean, null
- Complex types: arrays, objects, nested structures
- Enum values match predefined sets
- Type consistency across related fields

**Approaches:**
- Strict validation: reject if type doesn't match
- Type coercion: convert compatible types before validation

### Semantic Validation Rules

MUST validate:
- Business logic constraints
- Value ranges and boundaries
- Relationships between fields
- Domain-specific requirements
- Data consistency with system state

**Examples:**
- Start date MUST be before end date
- Age MUST be between 0 and 120
- Discount percentage MUST NOT exceed 100
- User ID MUST exist in database
- Withdrawal amount MUST NOT exceed balance

### Query Parameter Handling

**Default Value Strategy - MANDATORY:**
- Page number defaults to 1
- Limit per page defaults to 10-20
- Sort field defaults to createdAt
- Sort order defaults to descending
- Boolean flags default to false

**Validation Process:**
1. Check existence (use default if absent)
2. Validate string format
3. Transform to target type
4. Validate transformed value
5. Handle errors with clear messages

### Transformation Rules

**String Case:**
- Email addresses: lowercase
- Usernames: lowercase
- Search terms: lowercase
- Product codes: uppercase

**Number Transformation:**
- Parse string to integer/float
- Handle not-a-number values
- Verify acceptable range
- Confirm precision requirements

**Date Transformation:**
- Parse various input formats
- Normalize to UTC timezone
- Validate valid calendar date
- Store timezone separately for display

**Boolean Transformation:**
- True: "true", "1", "yes", "on", "enabled"
- False: "false", "0", "no", "off", "disabled"
- Case insensitive

**Array Transformation:**
- Split comma-separated values
- Collect multiple params with same name
- Trim whitespace from elements
- Remove empty elements
- Remove duplicates if required

---

## 4. ERROR HANDLING

### Error Types Classification

**1. Logic Errors**
- Application runs but produces incorrect results
- Prevention: Unit testing, code reviews, integration tests

**2. Database Errors**
- Connection errors, constraint violations, query errors
- Handling: Retry with exponential backoff for transient errors
- Classify permanent errors and convert to validation errors

**3. External Service Errors**
- Network failures, timeouts, rate limiting, outages
- Handling: Exponential backoff, circuit breaker pattern

**4. Input Validation Errors**
- Missing fields, invalid formats, type mismatches
- Response: HTTP 400 with specific validation errors

**5. Configuration Errors**
- MUST fail fast at startup, NOT at runtime
- Validate all required config before accepting requests

### Custom Error Classes - REQUIRED

```
ValidationError  → HTTP 400 (invalid input with field details)
NotFoundError    → HTTP 404 (missing resources)
DatabaseError    → HTTP 500 (not exposed to users)
ServiceError     → HTTP 500 (internal failures)
UnauthorizedError → HTTP 401 (authentication failures)
ForbiddenError   → HTTP 403 (authorization failures)
```

### Global Error Handler - MANDATORY

MUST implement centralized error handler that:
- Logs detailed error information (stack trace, context, user)
- Returns user-friendly error messages
- Hides internal details in production
- Sets appropriate HTTP status codes
- Includes field-level validation errors when available
- Maintains consistent error response format

### Retry Strategies

**Exponential Backoff:**
- Start with 1 second delay
- Double delay each retry: 1s, 2s, 4s, 8s...
- Respect Retry-After header for rate limiting
- Maximum 3-5 retries

**Circuit Breaker Pattern:**
- Track failure count per service
- After threshold (e.g., 5 failures), open circuit
- Immediately reject requests while circuit is open
- After timeout (e.g., 60s), allow test request (half-open)
- Close circuit on success, reopen on failure

### Error Security Rules - CRITICAL

**MUST NOT expose in error messages:**
- Database connection strings or credentials
- Internal table/column names or schema details
- Stack traces in production
- API keys or secrets
- Internal server IP addresses
- Exact SQL queries
- File system paths
- Detailed system information

**MUST do:**
- Return generic error messages in production
- Log detailed errors server-side
- Use different messages for dev vs. production
- Sanitize all error messages

### Health Checks - MANDATORY

MUST implement health check endpoints:
- Application status (basic liveness)
- Database connectivity (SELECT 1)
- External service availability
- Memory usage, uptime, disk space
- Response times, queue lengths

Return HTTP 200 for healthy, HTTP 503 for unhealthy

---

## 5. SERIALIZATION & DESERIALIZATION

### Format Rules

- MUST use JSON as primary format for web APIs
- MUST set Content-Type: application/json header
- MUST handle serialization errors gracefully

### Request Processing

```
JSON String (from client)
    ↓
JSON.parse() / json.loads()
    ↓
Object/Dictionary
    ↓
Validate
    ↓
Process
```

### Response Processing

```
Object/Dictionary
    ↓
JSON.stringify() / json.dumps()
    ↓
JSON String (to client)
```

### Special Data Types

**Dates:**
- MUST use ISO 8601 format for JSON
- MUST implement custom encoders for date serialization

**Rules:**
- NEVER execute deserialized code (no eval)
- ALWAYS validate against expected schemas
- ALWAYS sanitize user input
- MUST limit payload size
- MUST use HTTPS

---

## 6. CACHING

### Cache Eviction Policies

**No Eviction:**
- Use for: Critical data, config files, bounded datasets
- Cache never removes entries, fails on capacity

**LRU (Least Recently Used):**
- Use for: General caching, temporal access patterns
- Removes items not accessed longest

**LFU (Least Frequently Used):**
- Use for: Data with clear popularity patterns
- Removes least accessed items

**TTL (Time To Live):**
- Use for: Stale data, sessions, rate limiting, API responses
- Removes after set time period

### Use Case Guidelines

**Database Query Caching (Complex Joins):**
- Policy: LRU + TTL
- TTL: 5-15 minutes
- Invalidate on writes to relevant tables

**Session API Caching:**
- Policy: TTL + LRU
- TTL: Match session expiration (e.g., 30 min)
- Include user metadata to avoid lookups

**Rate Limiting:**
- Policy: TTL (primary)
- TTL: Rate window (e.g., 60 sec for per-minute limits)
- Use atomic increment operations

### Cache Best Practices

1. Monitor hit rates, eviction rates, memory usage
2. Start with conservative (shorter) TTLs
3. Test under different load patterns
4. Plan for cache failures with fallbacks
5. Use appropriate cache layers (Memory, Redis, CDN)
6. Invalidate proactively, don't rely solely on TTL
7. Ensure policies work across distributed instances

---

## 7. SECURITY

### Input Sanitization - MANDATORY

- MUST strip HTML tags (prevent XSS)
- MUST escape special characters
- MUST remove script tags and event handlers
- MUST validate file upload content types
- MUST encode output appropriately for context
- Apply IMMEDIATELY upon request entry

### SQL Injection Prevention - MANDATORY

- MUST use parameterized queries
- MUST validate input against SQL keywords
- MUST escape special SQL characters
- MUST use ORM tools that prevent injection
- MUST apply least privilege to database accounts

### NoSQL Injection Prevention

- Validate query operators against whitelist
- Sanitize query objects
- Enforce strict type checking
- NEVER use string concatenation for queries

### Authentication Rules

- Verify token validity and signature (JWT)
- Check session existence and expiration
- Validate API keys against authorized database
- Enforce rate limits per authenticated user

### Authorization Rules

- Verify user permission for requested resource
- Check role-based access control
- Validate resource ownership
- Enforce fine-grained permissions (read, write, delete)

### Size and Rate Limiting - MANDATORY

**Request Size Limits:**
- Maximum request body size
- Maximum query parameters
- Maximum field value length
- Maximum array sizes

**Rate Limiting:**
- Requests per time period per user
- Requests per time period per IP
- Enforce pagination for result sets
- Throttle expensive operations

### CSRF Protection

- Validate CSRF tokens in state-changing requests
- Check origin and referer headers
- Use same-site cookie attributes
- Require re-authentication for sensitive operations

---

## 8. CONFIGURATION MANAGEMENT

### Configuration Types

1. **Application Settings:** Log level, port, pool size, timeouts
2. **Database Config:** Host, credentials, pool settings, timeouts
3. **External Services:** API keys, URLs, retry configs
4. **Feature Flags:** Enable/disable features, A/B testing
5. **Observability:** Monitoring credentials, log settings
6. **Security:** JWT secret, hashing rounds, CORS origins
7. **Performance:** CPU/memory limits, worker counts, cache sizes

### Configuration Rules - MANDATORY

- MUST store all config in environment variables
- MUST NEVER hardcode values in source code
- MUST NEVER commit .env files to version control
- MUST use .gitignore to exclude .env files
- MUST use separate configs for dev, staging, production
- MUST rotate secrets regularly
- SHOULD use secret management services for production

### Configuration Validation - MANDATORY

MUST validate at startup (fail-fast):
- Missing required values
- Invalid formats (URLs, numbers, booleans)
- Insufficient security (weak passwords, short secrets)
- Type mismatches
- Out-of-range values

Use schema validation libraries (Zod, Joi, Yup)

---

## 9. GRACEFUL SHUTDOWN

### Signal Handling - MANDATORY

Register handlers for:
- SIGTERM: Graceful shutdown request
- SIGINT: Interrupt signal (Ctrl+C)

### Shutdown Process

1. **Stop accepting new connections** immediately
2. **Connection draining** (30-60 second timeout)
3. **Resource cleanup** in REVERSE order:
   - Close message queue connections
   - Close Redis/cache connections
   - Close database connection pool
   - Close file handles
   - Close network sockets
4. **Exit process** with appropriate code (0 = success, 1 = error)

### Shutdown Rules

- Set reasonable deadline (30-60 seconds)
- Force terminate if deadline exceeded
- Log shutdown progress
- Handle cleanup errors gracefully
- Continue cleanup even if individual steps fail
- Use process managers (PM2, systemd) for auto-restart

---

## 10. BEST PRACTICES SUMMARY

### Architecture

1. Keep handlers thin, move logic to services
2. Always go through service → repository for database
3. Extract data from HTTP and pass domain objects
4. Keep services HTTP-agnostic
5. Keep repositories focused on data access only
6. Use global error handler for consistent responses

### API Design

1. Use plural nouns for resources
2. Use hyphens for multi-word URLs
3. Represent hierarchies in URL structure
4. Use PATCH for partial updates
5. Paginate all list endpoints
6. Support sorting and filtering via query params
7. Use POST for custom non-CRUD actions
8. Maintain consistency across endpoints
9. Provide sensible defaults
10. Avoid abbreviations, keep names intuitive

### Validation

1. Fail fast - reject invalid requests early
2. Provide clear, actionable error messages
3. Security first - never trust client input
4. Use schema validation libraries
5. Whitelist allowed values, don't blacklist
6. Log validation failures
7. Cache validation rules/schemas
8. Apply same patterns across endpoints

### Error Handling

1. Classify errors by type
2. Implement retry with exponential backoff
3. Use circuit breakers for external services
4. Create clear boundaries between layers
5. Add context as errors bubble up
6. Log detailed errors server-side only
7. Return generic messages to clients

### Security

1. Sanitize all input immediately
2. Use parameterized queries always
3. Validate authentication on every request
4. Check authorization for every resource
5. Implement rate limiting
6. Limit request sizes
7. Never expose internal details in errors

### Performance

1. Use appropriate caching strategies
2. Set query timeouts
3. Monitor for slow queries
4. Use connection pooling
5. Implement pagination
6. Optimize expensive operations

---

## QUICK REFERENCE CHECKLIST

Before deploying any endpoint, verify:

□ Handler only handles HTTP concerns
□ Business logic is in service layer
□ Database access is in repository layer
□ URL follows REST conventions (plural nouns, hyphens)
□ Correct HTTP method for operation
□ List endpoints are paginated
□ Input validation implemented
□ Proper error handling with correct HTTP codes
□ Sensitive data not exposed in errors
□ Authentication/authorization checks
□ Rate limiting configured
□ Request size limits set
□ Configuration validated at startup
□ Health check endpoint available
□ Graceful shutdown implemented
□ Logging implemented for debugging
□ Security measures (sanitization, parameterized queries)

---

## ERROR RESPONSE FORMAT STANDARD

All error responses MUST follow this format:

```json
{
  "error": "Error type or category",
  "message": "Human-readable description",
  "fields": {
    "fieldName": "Specific field error"
  },
  "requestId": "unique-request-id",
  "timestamp": "ISO-8601-timestamp"
}
```

HTTP Status Codes:
- 400: Bad Request (validation errors)
- 401: Unauthorized (authentication failed)
- 403: Forbidden (authorization failed)
- 404: Not Found (resource doesn't exist)
- 409: Conflict (duplicate resource)
- 422: Unprocessable Entity (semantic validation)
- 429: Too Many Requests (rate limited)
- 500: Internal Server Error (unexpected errors)
- 503: Service Unavailable (dependency down)

---
 
## END OF BACKEND RULES DOCUMENT

