# Server Request Lifecycle & Architecture Guide

## Overview

This document describes a clean, layered architecture for handling HTTP requests in server applications. The architecture separates concerns across distinct layers: routing, business logic, and data access.

## Request Lifecycle Flow

```
1. Entry Point (Port :8080)
   ↓
2. Middleware Chain (CORS → Logging → Auth → ...)
   ↓
3. Router/Handler Layer (HTTP-aware)
   ↓
4. Service Layer (Business Logic - HTTP-agnostic)
   ↓
5. Repository Layer (Database Operations)
   ↓
6. Response (through middleware chain in reverse)

```

## Directory Structure

```
project/
├── cmd/
│   └── server/
│       └── main.go              # Entry point, server initialization
├── internal/
│   ├── handler/                 # HTTP handlers (routing layer)
│   │   ├── user_handler.go
│   │   ├── product_handler.go
│   │   └── router.go            # Route definitions
│   │
│   ├── service/                 # Business logic layer
│   │   ├── user_service.go
│   │   └── product_service.go
│   │
│   ├── repository/              # Data access layer
│   │   ├── user_repository.go
│   │   └── product_repository.go
│   │
│   ├── middleware/              # Middleware components
│   │   ├── cors.go
│   │   ├── logging.go
│   │   ├── auth.go
│   │   ├── validation.go
│   │   └── error_handler.go
│   │
│   ├── model/                   # Data structures
│   │   ├── user.go
│   │   └── product.go
│   │
│   └── dto/                     # Data Transfer Objects
│       ├── request/
│       │   └── user_request.go
│       └── response/
│           └── user_response.go
└── pkg/
    └── database/
        └── connection.go

```

## Layer Responsibilities

### 1. Handler Layer (Routes)

**Purpose**: Handle HTTP-specific concerns - routing, request parsing, response formatting.

**Responsibilities**:

- Route HTTP requests to appropriate handlers
- Deserialize HTTP request into language-specific data structures (dict/struct)
- Validate and transform incoming data
- Call service layer methods
- Serialize service responses back to HTTP responses
- Handle HTTP status codes and headers

**Example (Go)**:

```go
// handler/user_handler.go
type UserHandler struct {
    userService *service.UserService
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    // 1. Deserialize request body into struct
    var req dto.CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // 2. Validate request
    if err := req.Validate(); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // 3. Transform to domain model
    user := req.ToUser()

    // 4. Call service layer (HTTP-agnostic)
    createdUser, err := h.userService.CreateUser(r.Context(), user)
    if err != nil {
        // Error handling middleware will catch this
        panic(err)
    }

    // 5. Serialize response
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(dto.UserResponse{User: createdUser})
}

```

**Example (Python/Flask)**:

```python
# handler/user_handler.py
from flask import Blueprint, request, jsonify

user_bp = Blueprint('user', __name__)

@user_bp.route('/users', methods=['POST'])
def create_user():
    # 1. Deserialize request into dict
    data = request.get_json()

    # 2. Validate and transform
    req = CreateUserRequest(**data)
    req.validate()

    # 3. Call service layer
    user = user_service.create_user(req.to_user())

    # 4. Serialize response
    return jsonify(UserResponse(user).to_dict()), 201

```

### 2. Service Layer (Business Logic)

**Purpose**: Contain all business logic, independent of HTTP concerns.

**Responsibilities**:

- Implement business rules and workflows
- Coordinate between multiple repositories if needed
- Handle transactions
- Perform business validations
- NO knowledge of HTTP (no request/response objects)
- Work with domain models, not DTOs

**Example (Go)**:

```go
// service/user_service.go
type UserService struct {
    userRepo *repository.UserRepository
    emailService *EmailService
}

func (s *UserService) CreateUser(ctx context.Context, user *model.User) (*model.User, error) {
    // Business logic: check if email already exists
    existing, err := s.userRepo.FindByEmail(ctx, user.Email)
    if err != nil && !errors.Is(err, repository.ErrNotFound) {
        return nil, err
    }
    if existing != nil {
        return nil, ErrEmailAlreadyExists
    }

    // Business logic: hash password
    user.PasswordHash = hashPassword(user.Password)

    // Save to database
    createdUser, err := s.userRepo.Create(ctx, user)
    if err != nil {
        return nil, err
    }

    // Business logic: send welcome email
    go s.emailService.SendWelcomeEmail(user.Email)

    return createdUser, nil
}

```

**Example (Python)**:

```python
# service/user_service.py
class UserService:
    def __init__(self, user_repository, email_service):
        self.user_repo = user_repository
        self.email_service = email_service

    def create_user(self, user):
        # Business logic: check duplicates
        if self.user_repo.find_by_email(user.email):
            raise EmailAlreadyExistsError()

        # Business logic: hash password
        user.password_hash = hash_password(user.password)

        # Save to database
        created_user = self.user_repo.create(user)

        # Business logic: send email
        self.email_service.send_welcome_email(user.email)

        return created_user

```

### 3. Repository Layer (Database Operations)

**Purpose**: Handle all database operations and data persistence.

**Responsibilities**:

- Construct and execute database queries
- Map database rows to domain models
- Handle database connections and transactions
- Abstract database-specific details
- NO business logic
- Return domain models, not database-specific types

**Example (Go)**:

```go
// repository/user_repository.go
type UserRepository struct {
    db *sql.DB
}

func (r *UserRepository) Create(ctx context.Context, user *model.User) (*model.User, error) {
    query := `
        INSERT INTO users (email, password_hash, name, created_at)
        VALUES ($1, $2, $3, $4)
        RETURNING id, email, name, created_at
    `

    var created model.User
    err := r.db.QueryRowContext(
        ctx,
        query,
        user.Email,
        user.PasswordHash,
        user.Name,
        time.Now(),
    ).Scan(&created.ID, &created.Email, &created.Name, &created.CreatedAt)

    if err != nil {
        return nil, err
    }

    return &created, nil
}

func (r *UserRepository) FindByEmail(ctx context.Context, email string) (*model.User, error) {
    query := `SELECT id, email, name, password_hash, created_at FROM users WHERE email = $1`

    var user model.User
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &user.ID,
        &user.Email,
        &user.Name,
        &user.PasswordHash,
        &user.CreatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, ErrNotFound
    }
    if err != nil {
        return nil, err
    }

    return &user, nil
}

```

**Example (Python)**:

```python
# repository/user_repository.py
class UserRepository:
    def __init__(self, db_connection):
        self.db = db_connection

    def create(self, user):
        query = """
            INSERT INTO users (email, password_hash, name, created_at)
            VALUES (%s, %s, %s, %s)
            RETURNING id, email, name, created_at
        """

        cursor = self.db.cursor()
        cursor.execute(query, (user.email, user.password_hash, user.name, datetime.now()))
        row = cursor.fetchone()
        self.db.commit()

        return User(id=row[0], email=row[1], name=row[2], created_at=row[3])

    def find_by_email(self, email):
        query = "SELECT id, email, name, password_hash, created_at FROM users WHERE email = %s"

        cursor = self.db.cursor()
        cursor.execute(query, (email,))
        row = cursor.fetchone()

        if not row:
            return None

        return User(id=row[0], email=row[1], name=row[2],
                   password_hash=row[3], created_at=row[4])

```

## Middleware Chain

### Middleware Execution Order

The order of middleware is critical. Here's the recommended sequence:

```
Request Flow:
1. CORS               → Handle cross-origin requests
2. Logging            → Log request details
3. Request Context    → Set request ID, tracing
4. Auth               → Verify authentication
5. Deserialization    → Parse request body (often built into handler)
6. Validation         → Validate request data
7. [Handler Execution]
8. Global Error Handler → Catch and format errors

Response Flow (reverse):
8. Global Error Handler
7. [Handler Execution]
6. Validation
5. Serialization
4. Auth (response headers)
3. Request Context (add trace headers)
2. Logging (log response)
1. CORS (add headers)

```

### Middleware Examples

**CORS Middleware (Go)**:

```go
// middleware/cors.go
func CORS(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}

```

**Logging Middleware (Go)**:

```go
// middleware/logging.go
func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        log.Printf("Started %s %s", r.Method, r.URL.Path)

        next.ServeHTTP(w, r)

        log.Printf("Completed in %v", time.Since(start))
    })
}

```

**Auth Middleware (Go)**:

```go
// middleware/auth.go
func Auth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        user, err := validateToken(token)
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        // Add user to context
        ctx := context.WithValue(r.Context(), "user", user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

```

**Request Context Middleware (Go)**:

```go
// middleware/request_context.go
func RequestContext(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := uuid.New().String()

        ctx := context.WithValue(r.Context(), "request_id", requestID)
        w.Header().Set("X-Request-ID", requestID)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

```

**Global Error Handler (Go)**:

```go
// middleware/error_handler.go
func ErrorHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic: %v", err)

                // Determine error type and status code
                statusCode := http.StatusInternalServerError
                message := "Internal server error"

                switch e := err.(type) {
                case *ValidationError:
                    statusCode = http.StatusBadRequest
                    message = e.Error()
                case *NotFoundError:
                    statusCode = http.StatusNotFound
                    message = e.Error()
                case *UnauthorizedError:
                    statusCode = http.StatusUnauthorized
                    message = e.Error()
                }

                w.Header().Set("Content-Type", "application/json")
                w.WriteHeader(statusCode)
                json.NewEncoder(w).Encode(map[string]string{
                    "error": message,
                })
            }
        }()

        next.ServeHTTP(w, r)
    })
}

```

## Complete Example: Putting It All Together

**Entry Point (Go)**:

```go
// cmd/server/main.go
func main() {
    // Database connection
    db := database.Connect()

    // Initialize layers
    userRepo := repository.NewUserRepository(db)
    userService := service.NewUserService(userRepo)
    userHandler := handler.NewUserHandler(userService)

    // Setup router
    router := http.NewServeMux()
    router.HandleFunc("/users", userHandler.CreateUser)
    router.HandleFunc("/users/{id}", userHandler.GetUser)

    // Build middleware chain (order matters!)
    var handler http.Handler = router
    handler = middleware.ErrorHandler(handler)
    handler = middleware.Auth(handler)
    handler = middleware.RequestContext(handler)
    handler = middleware.Logging(handler)
    handler = middleware.CORS(handler)

    // Start server
    log.Println("Server starting on :8080")
    http.ListenAndServe(":8080", handler)
}

```

**Entry Point (Python/Flask)**:

```python
# app.py
from flask import Flask
from middleware import cors, logging, auth, error_handler

app = Flask(__name__)

# Database connection
db = database.connect()

# Initialize layers
user_repo = UserRepository(db)
user_service = UserService(user_repo)

# Register blueprints (routes)
from handler.user_handler import user_bp
app.register_blueprint(user_bp)

# Register middleware (order matters!)
app.before_request(cors.handle_cors)
app.before_request(logging.log_request)
app.before_request(auth.authenticate)
app.errorhandler(Exception)(error_handler.handle_error)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)

```

## Key Principles

1. **Separation of Concerns**: Each layer has a single, well-defined responsibility
2. **Dependency Direction**: Handler → Service → Repository (never reverse)
3. **HTTP Isolation**: Only handlers know about HTTP; services and repositories are HTTP-agnostic
4. **Data Flow**: Request → DTO → Domain Model → Repository → Domain Model → DTO → Response
5. **Error Handling**: Use middleware for global error handling, layer-specific errors bubble up
6. **Context Propagation**: Pass context through all layers for cancellation and request-scoped values
7. **Testability**: Each layer can be tested independently with mocks

## Benefits

- **Maintainability**: Clear separation makes code easier to understand and modify
- **Testability**: Each layer can be unit tested in isolation
- **Reusability**: Business logic in services can be reused across different handlers
- **Flexibility**: Easy to swap implementations (e.g., change database, add REST/gRPC handlers)
- **Scalability**: Clean boundaries make it easier to split into microservices later

# Server Architecture: Request Lifecycle & Layered Design

## Overview

This document describes a clean, layered architecture for handling HTTP requests in server applications. The design is language-agnostic and applies to any backend framework or programming language.

## Request Lifecycle Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Entry Point (Port :8080)                                 │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Middleware Chain                                         │
│    CORS → Logging → Request Context → Auth → ...           │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Handler Layer (Routes)                                   │
│    - Deserialize HTTP request                               │
│    - Validate input                                         │
│    - Transform to domain objects                            │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Service Layer (Business Logic)                           │
│    - Execute business rules                                 │
│    - Coordinate operations                                  │
│    - HTTP-agnostic                                          │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. Repository Layer (Database)                              │
│    - Construct queries                                      │
│    - Execute database operations                            │
│    - Map data to domain objects                             │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. Response (through middleware chain in reverse)           │
└─────────────────────────────────────────────────────────────┘

```

## Directory Structure

```
project/
│
├── cmd/                          # Application entry points
│   └── server/
│       └── main.*                # Server initialization & startup
│
├── internal/                     # Private application code
│   │
│   ├── handler/                  # HTTP Layer (Routes)
│   │   ├── user_handler.*        # User-related routes
│   │   ├── product_handler.*     # Product-related routes
│   │   ├── order_handler.*       # Order-related routes
│   │   └── router.*              # Route registration & mapping
│   │
│   ├── service/                  # Business Logic Layer
│   │   ├── user_service.*        # User business logic
│   │   ├── product_service.*     # Product business logic
│   │   └── order_service.*       # Order business logic
│   │
│   ├── repository/               # Data Access Layer
│   │   ├── user_repository.*     # User database operations
│   │   ├── product_repository.*  # Product database operations
│   │   └── order_repository.*    # Order database operations
│   │
│   ├── middleware/               # Cross-cutting concerns
│   │   ├── cors.*                # CORS handling
│   │   ├── logging.*             # Request/response logging
│   │   ├── auth.*                # Authentication
│   │   ├── request_context.*     # Request ID, tracing
│   │   ├── validation.*          # Input validation
│   │   └── error_handler.*       # Global error handling
│   │
│   ├── model/                    # Domain Models
│   │   ├── user.*                # User domain entity
│   │   ├── product.*             # Product domain entity
│   │   └── order.*               # Order domain entity
│   │
│   └── dto/                      # Data Transfer Objects
│       ├── request/              # Request DTOs
│       │   ├── user_request.*
│       │   └── product_request.*
│       └── response/             # Response DTOs
│           ├── user_response.*
│           └── product_response.*
│
├── pkg/                          # Public reusable packages
│   ├── database/
│   │   └── connection.*          # Database setup
│   └── validator/
│       └── validator.*           # Validation utilities
│
└── config/                       # Configuration files
    └── config.*                  # App configuration

```

## Layer Responsibilities

### Layer 1: Handler (Routes/Controllers)

**Purpose**: Handle HTTP protocol concerns and route requests.

**Responsibilities**:

- Route incoming HTTP requests to appropriate handler functions
- Deserialize HTTP request body into language data structures:
    - Python: dict
    - Go: struct
    - Rust: struct
    - JavaScript: object
    - Java: POJO
- Perform initial validation and transformation
- Extract request parameters (path, query, headers, body)
- Invoke service layer methods
- Serialize service responses back to HTTP format
- Set HTTP status codes and headers
- Handle HTTP-specific errors (400, 401, 404, 500, etc.)

**Characteristics**:

- ✅ Knows about HTTP protocol
- ✅ Knows about request/response formats
- ✅ Converts between HTTP and domain objects
- ❌ Contains NO business logic
- ❌ Does NOT directly access database
- ❌ Should be thin and focused on HTTP concerns

**Pseudocode Example**:

```
function createUser(request, response):
    // 1. Deserialize request body into language structure
    requestData = deserialize(request.body)

    // 2. Validate request structure
    if not validate(requestData):
        return response.status(400).json({"error": "Invalid input"})

    // 3. Transform to domain model
    user = transformToUser(requestData)

    // 4. Call service layer (HTTP-agnostic)
    try:
        createdUser = userService.createUser(user)
    catch error:
        // Let error handler middleware catch this
        throw error

    // 5. Serialize and return response
    return response.status(201).json(transformToResponse(createdUser))

```

### Layer 2: Service (Business Logic)

**Purpose**: Implement all business rules and workflows.

**Responsibilities**:

- Execute business logic and rules
- Coordinate operations across multiple repositories
- Manage transactions
- Perform business-level validations
- Orchestrate complex workflows
- Handle domain events
- Completely independent of HTTP protocol

**Characteristics**:

- ✅ Contains ALL business logic
- ✅ Coordinates multiple repositories if needed
- ✅ Handles transactions
- ✅ Works with domain models
- ❌ NO knowledge of HTTP (no request/response objects)
- ❌ NO direct database queries
- ❌ Should NOT import HTTP libraries

**Pseudocode Example**:

```
class UserService:
    constructor(userRepository, emailService):
        this.userRepo = userRepository
        this.emailService = emailService

    function createUser(user):
        // Business rule: check for duplicate email
        existingUser = this.userRepo.findByEmail(user.email)
        if existingUser exists:
            throw DuplicateEmailError("Email already registered")

        // Business rule: validate age requirement
        if user.age < 18:
            throw ValidationError("User must be at least 18 years old")

        // Business logic: hash password
        user.passwordHash = hashPassword(user.password)

        // Business logic: set default values
        user.status = "ACTIVE"
        user.createdAt = getCurrentTimestamp()

        // Save to database via repository
        createdUser = this.userRepo.create(user)

        // Business logic: trigger side effects
        this.emailService.sendWelcomeEmail(user.email)

        return createdUser

    function getUserWithOrders(userId):
        // Coordinate multiple repositories
        user = this.userRepo.findById(userId)
        if not user:
            throw NotFoundError("User not found")

        orders = this.orderRepo.findByUserId(userId)
        user.orders = orders

        return user

```

### Layer 3: Repository (Data Access)

**Purpose**: Handle all database operations and data persistence.

**Responsibilities**:

- Construct database queries (SQL, ORM, query builders)
- Execute CRUD operations (Create, Read, Update, Delete)
- Map database rows/documents to domain models
- Handle database connections and connection pools
- Abstract database-specific implementation details
- Provide a clean interface for data access

**Characteristics**:

- ✅ All database interaction happens here
- ✅ Constructs and executes queries
- ✅ Maps data between database and domain models
- ✅ Handles database-specific errors
- ❌ NO business logic
- ❌ NO HTTP concerns
- ❌ Should NOT validate business rules

**Pseudocode Example**:

```
class UserRepository:
    constructor(databaseConnection):
        this.db = databaseConnection

    function create(user):
        query = """
            INSERT INTO users (email, password_hash, name, age, status, created_at)
            VALUES (?, ?, ?, ?, ?, ?)
            RETURNING id, email, name, age, status, created_at
        """

        result = this.db.execute(
            query,
            [user.email, user.passwordHash, user.name, user.age, user.status, user.createdAt]
        )

        // Map database result to domain model
        return mapRowToUser(result)

    function findByEmail(email):
        query = "SELECT * FROM users WHERE email = ?"

        result = this.db.execute(query, [email])

        if result is empty:
            return null

        return mapRowToUser(result)

    function findById(userId):
        query = "SELECT * FROM users WHERE id = ?"

        result = this.db.execute(query, [userId])

        if result is empty:
            return null

        return mapRowToUser(result)

    function update(userId, userData):
        query = """
            UPDATE users
            SET name = ?, email = ?, updated_at = ?
            WHERE id = ?
        """

        this.db.execute(
            query,
            [userData.name, userData.email, getCurrentTimestamp(), userId]
        )

        return this.findById(userId)

    function delete(userId):
        query = "DELETE FROM users WHERE id = ?"
        this.db.execute(query, [userId])

```

## Middleware Chain Architecture

### Middleware Execution Order

Middleware executes in a specific order for both requests and responses:

```
REQUEST FLOW (Incoming):
┌────────────────────────────────────────────────┐
│ 1. CORS Middleware                             │ ← Handle preflight, set CORS headers
│ 2. Logging Middleware                          │ ← Log incoming request details
│ 3. Request Context Middleware                  │ ← Set request ID, trace ID
│ 4. Authentication Middleware                   │ ← Verify auth token, set user
│ 5. Deserialization (often in handler)          │ ← Parse request body
│ 6. Validation Middleware                       │ ← Validate request structure
│ 7. Handler Execution                           │ ← Business logic executes
│ 8. Global Error Handler                        │ ← Catch and format errors
└────────────────────────────────────────────────┘

RESPONSE FLOW (Outgoing - reverse order):
┌────────────────────────────────────────────────┐
│ 8. Global Error Handler                        │ ← Format errors if any
│ 7. Handler Execution                           │ ← Return response
│ 6. Validation (response validation optional)   │
│ 5. Serialization                               │ ← Convert to JSON/XML
│ 4. Authentication                              │ ← Add auth response headers
│ 3. Request Context                             │ ← Add trace headers
│ 2. Logging                                     │ ← Log response status/time
│ 1. CORS                                        │ ← Add CORS response headers
└────────────────────────────────────────────────┘

```

### Individual Middleware Descriptions

### 1. CORS Middleware

**Purpose**: Handle Cross-Origin Resource Sharing

```
function corsMiddleware(request, response, next):
    // Set CORS headers
    response.setHeader("Access-Control-Allow-Origin", "*")
    response.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
    response.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization")

    // Handle preflight requests
    if request.method == "OPTIONS":
        response.status(200).end()
        return

    next()

```

### 2. Logging Middleware

**Purpose**: Log request and response information

```
function loggingMiddleware(request, response, next):
    startTime = getCurrentTime()
    requestId = getRequestId(request)

    log("Request started", {
        method: request.method,
        path: request.path,
        requestId: requestId
    })

    // Continue to next middleware
    next()

    // This executes after response (if framework supports it)
    duration = getCurrentTime() - startTime
    log("Request completed", {
        method: request.method,
        path: request.path,
        status: response.statusCode,
        duration: duration,
        requestId: requestId
    })

```

### 3. Request Context Middleware

**Purpose**: Set up request-scoped data (IDs, tracing)

```
function requestContextMiddleware(request, response, next):
    // Generate unique request ID
    requestId = generateUUID()

    // Add to request context
    request.context.set("requestId", requestId)
    request.context.set("startTime", getCurrentTime())

    // Add to response headers for client tracing
    response.setHeader("X-Request-ID", requestId)

    next()

```

### 4. Authentication Middleware

**Purpose**: Verify user identity

```
function authenticationMiddleware(request, response, next):
    // Extract auth token from header
    authHeader = request.getHeader("Authorization")

    if not authHeader:
        return response.status(401).json({
            "error": "Authentication required"
        })

    // Validate token
    token = extractToken(authHeader)
    user = validateToken(token)

    if not user:
        return response.status(401).json({
            "error": "Invalid or expired token"
        })

    // Add user to request context
    request.context.set("user", user)

    next()

```

### 5. Validation Middleware

**Purpose**: Validate request structure and data

```
function validationMiddleware(schema):
    return function(request, response, next):
        // Validate request body against schema
        errors = validateAgainstSchema(request.body, schema)

        if errors.length > 0:
            return response.status(400).json({
                "error": "Validation failed",
                "details": errors
            })

        next()

```

### 6. Global Error Handler Middleware

**Purpose**: Catch and format all errors

```
function globalErrorHandler(error, request, response, next):
    // Log error
    log("Error occurred", {
        error: error.message,
        stack: error.stack,
        requestId: request.context.get("requestId")
    })

    // Determine status code and message
    statusCode = 500
    message = "Internal server error"

    if error is ValidationError:
        statusCode = 400
        message = error.message
    else if error is NotFoundError:
        statusCode = 404
        message = error.message
    else if error is UnauthorizedError:
        statusCode = 401
        message = error.message
    else if error is ForbiddenError:
        statusCode = 403
        message = error.message

    // Return formatted error response
    response.status(statusCode).json({
        "error": message,
        "requestId": request.context.get("requestId"),
        "timestamp": getCurrentTimestamp()
    })

```

## Data Flow Through Layers

### Request Processing

```
HTTP Request
    ↓
[Middleware Chain]
    ↓
Handler receives HTTP Request
    ↓
Deserialize to language data structure (dict/struct/object)
    ↓
Validate request structure
    ↓
Transform to Domain Model
    ↓
Call Service with Domain Model
    ↓
Service executes business logic
    ↓
Service calls Repository with Domain Model
    ↓
Repository constructs database query
    ↓
Repository executes query
    ↓
Repository maps result to Domain Model
    ↓
Repository returns Domain Model to Service
    ↓
Service returns Domain Model to Handler
    ↓
Handler transforms Domain Model to Response DTO
    ↓
Serialize to HTTP Response
    ↓
[Middleware Chain - Reverse]
    ↓
HTTP Response

```

### Data Transformation Flow

```
HTTP JSON → Request DTO → Domain Model → Repository → Database
                                                          ↓
HTTP JSON ← Response DTO ← Domain Model ← Repository ← Database

```

## Complete Application Example

### Entry Point (Server Initialization)

```
function main():
    // 1. Load configuration
    config = loadConfig()

    // 2. Initialize database connection
    database = createDatabaseConnection(config.databaseUrl)

    // 3. Initialize repositories
    userRepository = new UserRepository(database)
    orderRepository = new OrderRepository(database)
    productRepository = new ProductRepository(database)

    // 4. Initialize services
    emailService = new EmailService(config.emailConfig)
    userService = new UserService(userRepository, emailService)
    orderService = new OrderService(orderRepository, productRepository, userRepository)

    // 5. Initialize handlers
    userHandler = new UserHandler(userService)
    orderHandler = new OrderHandler(orderService)

    // 6. Setup router
    router = createRouter()

    // User routes
    router.post("/users", userHandler.createUser)
    router.get("/users/:id", userHandler.getUser)
    router.put("/users/:id", userHandler.updateUser)
    router.delete("/users/:id", userHandler.deleteUser)

    // Order routes
    router.post("/orders", orderHandler.createOrder)
    router.get("/orders/:id", orderHandler.getOrder)

    // 7. Build middleware chain (ORDER MATTERS!)
    app = createApp()

    // Apply middleware in order
    app.use(corsMiddleware)
    app.use(loggingMiddleware)
    app.use(requestContextMiddleware)
    app.use(authenticationMiddleware)
    app.use(router)
    app.use(globalErrorHandler)

    // 8. Start server
    server = createServer(app)
    server.listen(8080)

    log("Server started on port 8080")

```

## Design Principles

### 1. Separation of Concerns

Each layer has a single, well-defined responsibility:

- **Handler**: HTTP protocol
- **Service**: Business logic
- **Repository**: Data persistence

### 2. Dependency Direction

Dependencies flow in one direction only:

```
Handler → Service → Repository → Database

```

Never reverse: Repository should never call Service, Service should never call Handler.

### 3. HTTP Isolation

Only the Handler layer knows about HTTP:

- Services receive domain objects, not HTTP requests
- Services return domain objects, not HTTP responses
- Repositories work with domain objects and database data only

### 4. Single Responsibility

- Handler: One route handler per endpoint
- Service: One service method per business operation
- Repository: One repository method per database operation

### 5. Domain Model Purity

Domain models should:

- Represent business entities
- Be independent of HTTP concerns
- Be independent of database structure
- Contain only business-relevant fields

### 6. Error Handling Strategy

```
Repository errors → Service errors → Handler errors → Middleware
        ↓                ↓                ↓                ↓
    DBError      BusinessError      HTTPError      FormattedResponse

```

### 7. Transaction Boundaries

Transactions should be managed at the Service layer:

```
function transferMoney(fromUserId, toUserId, amount):
    transaction = startTransaction()
    try:
        userRepository.deduct(fromUserId, amount, transaction)
        userRepository.add(toUserId, amount, transaction)
        transaction.commit()
    catch error:
        transaction.rollback()
        throw error

```

### 8. Context Propagation

Pass context through all layers for:

- Request cancellation
- Request tracing
- User information
- Request-scoped data

```
Handler → Service → Repository
  ctx  →   ctx   →     ctx

```

## Testing Strategy

### Unit Testing by Layer

**Handler Tests**:

- Mock the Service layer
- Test request deserialization
- Test validation logic
- Test response serialization
- Test error handling

**Service Tests**:

- Mock the Repository layer
- Test business logic
- Test business rule validation
- Test error scenarios
- Test transaction handling

**Repository Tests**:

- Use in-memory or test database
- Test query construction
- Test data mapping
- Test CRUD operations
- Test database error handling

### Integration Testing

- Test full request flow through all layers
- Use test database
- Verify end-to-end functionality

## Benefits of This Architecture

1. **Maintainability**: Clear boundaries make code easy to understand and modify
2. **Testability**: Each layer can be tested independently
3. **Reusability**: Business logic can be reused across different interfaces (REST, GraphQL, CLI)
4. **Flexibility**: Easy to swap implementations (e.g., change from SQL to NoSQL)
5. **Scalability**: Clean separation makes it easier to scale or split into microservices
6. **Team Collaboration**: Different teams can work on different layers simultaneously
7. **Code Quality**: Enforces best practices and prevents mixing of concerns

## Common Mistakes to Avoid

❌ **Don't**: Put business logic in handlers
✅ **Do**: Keep handlers thin, move logic to services

❌ **Don't**: Access database directly from handlers
✅ **Do**: Always go through service → repository

❌ **Don't**: Pass HTTP request/response objects to services
✅ **Do**: Extract data and pass domain objects

❌ **Don't**: Put HTTP concerns in services
✅ **Do**: Keep services completely HTTP-agnostic

❌ **Don't**: Put business logic in repositories
✅ **Do**: Keep repositories focused on data access only

❌ **Don't**: Skip middleware error handling
✅ **Do**: Use global error handler for consistent error responses

## Conclusion

This layered architecture provides a robust foundation for building maintainable, scalable server applications. By maintaining clear boundaries between HTTP handling, business logic, and data access, you create code that is easier to understand, test, and evolve over time.
