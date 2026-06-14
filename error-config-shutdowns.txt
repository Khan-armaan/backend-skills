# Error Handling and Fault-Tolerant Systems

Building resilient systems requires anticipating failure modes and handling them gracefully. This guide covers comprehensive error handling strategies, configuration management, and graceful shutdown patterns.

## Understanding System Failures

Real-world systems face constant challenges:

- **Database queries fail** due to connection issues, deadlocks, or constraint violations
- **External APIs timeout** from network problems, rate limiting, or service outages
- **Users send invalid data** with incorrect formats, missing fields, or out-of-range values
- **Edge cases emerge** that weren't anticipated during development

The goal isn't to prevent all errors—that's impossible—but to handle them gracefully and maintain system stability.

---

## Types of Errors

### 1. Logic Errors

Logic errors occur when the application doesn't crash but produces incorrect results. These are often the most dangerous because they can go undetected.

**Characteristics:**

- Application continues running normally
- Incorrect algorithm implementation
- Missing edge case handling
- Data corruption over time
- Silent failures that compound

**Examples:**

- Calculating discounts incorrectly
- Processing timezone conversions wrong
- Off-by-one errors in pagination
- Race conditions in concurrent operations

**Prevention:**

- Comprehensive unit testing
- Property-based testing for edge cases
- Code reviews focusing on business logic
- Integration tests with realistic data
- Monitoring for anomalous outputs

### 2. Database Errors

Database operations are inherently failure-prone due to network issues, resource constraints, and data integrity requirements.

**Common Database Errors:**

**Connection Errors:**

- Database server unreachable
- Connection pool exhausted
- Authentication failures
- Network timeouts

**Constraint Violations:**

- Unique constraint violations (duplicate email addresses)
- Foreign key constraint failures (referencing non-existent records)
- Check constraint violations (invalid data ranges)
- Not-null constraint violations

**Query Errors:**

- SQL syntax errors or typos
- Invalid table or column names
- Type mismatches
- Deadlock conditions
- Transaction timeout

**Handling Strategies:**

```jsx
// Example: Database error handling with retries
async function executeWithRetry(queryFn, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await queryFn();
    } catch (error) {
      // Retry on transient errors
      if (isTransientError(error) && attempt < maxRetries) {
        await delay(Math.pow(2, attempt) * 100); // Exponential backoff
        continue;
      }

      // Classify and handle permanent errors
      if (error.code === '23505') {
        throw new ValidationError('Record already exists');
      }
      if (error.code === '23503') {
        throw new ValidationError('Referenced record not found');
      }

      throw error;
    }
  }
}

function isTransientError(error) {
  const transientCodes = ['57P03', 'ECONNRESET', 'ETIMEDOUT'];
  return transientCodes.includes(error.code);
}

```

### 3. External Service Errors

External dependencies introduce multiple failure modes beyond your control.

**Failure Types:**

- **Network failures:** DNS resolution failures, connection refused, network partitions
- **Timeouts:** Connection timeouts, read timeouts, slow responses
- **Rate limiting:** HTTP 429 responses when quota exceeded
- **Service outages:** Complete unavailability of third-party services
- **Invalid responses:** Malformed JSON, unexpected data structures

**Resilience Strategies:**

**Exponential Backoff:**

```jsx
async function callExternalAPI(url, maxRetries = 5) {
  let delay = 1000; // Start with 1 second

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, { timeout: 5000 });

      if (response.status === 429) {
        // Rate limited - respect Retry-After header
        const retryAfter = response.headers.get('Retry-After');
        await sleep(retryAfter ? parseInt(retryAfter) * 1000 : delay);
        delay *= 2; // Exponential backoff
        continue;
      }

      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }

      return await response.json();
    } catch (error) {
      if (attempt === maxRetries) {
        throw new ServiceUnavailableError('External API failed after retries');
      }

      await sleep(delay);
      delay *= 2; // Exponential backoff
    }
  }
}

```

**Circuit Breaker Pattern:**
Prevents cascading failures by stopping calls to failing services temporarily:

```jsx
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.failureCount = 0;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = Date.now();
  }

  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}

```

### 4. Input Validation Errors

User input is the primary attack vector and source of invalid data.

**Common Validation Failures:**

- Missing required fields
- Invalid data formats (email, phone, date)
- Out-of-range values (negative quantities, future birth dates)
- Type mismatches (string instead of number)
- Malicious input (SQL injection attempts, XSS payloads)

**Response:** Return HTTP 400 Bad Request with specific validation errors.

```jsx
class ValidationError extends Error {
  constructor(message, fields = {}) {
    super(message);
    this.name = 'ValidationError';
    this.statusCode = 400;
    this.fields = fields;
  }
}

// Usage
if (!email || !isValidEmail(email)) {
  throw new ValidationError('Invalid input', {
    email: 'Valid email address is required'
  });
}

```

### 5. Configuration Errors

Configuration errors should cause the application to fail immediately at startup, not at runtime.

**Fail-Fast Principle:** Validate all required configuration values before the application starts accepting requests.

```jsx
// Configuration validation at startup
function validateConfig() {
  const required = [
    'DATABASE_URL',
    'JWT_SECRET',
    'API_KEY',
    'PORT'
  ];

  const missing = required.filter(key => !process.env[key]);

  if (missing.length > 0) {
    console.error(`Missing required configuration: ${missing.join(', ')}`);
    process.exit(1); // Fail fast
  }

  // Validate formats
  if (isNaN(process.env.PORT)) {
    console.error('PORT must be a number');
    process.exit(1);
  }

  if (process.env.JWT_SECRET.length < 32) {
    console.error('JWT_SECRET must be at least 32 characters');
    process.exit(1);
  }
}

// Run before starting server
validateConfig();

```

---

## Prevention Strategies

### 1. Health Checks

Implement comprehensive health checks to detect issues before they affect users.

**Health Check Endpoints:**

```jsx
app.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    checks: {}
  };

  // Database health
  try {
    await db.query('SELECT 1');
    health.checks.database = { status: 'up' };
  } catch (error) {
    health.status = 'unhealthy';
    health.checks.database = { status: 'down', error: error.message };
  }

  // External service health
  try {
    await fetch('https://api.external.com/health', { timeout: 2000 });
    health.checks.externalAPI = { status: 'up' };
  } catch (error) {
    health.checks.externalAPI = { status: 'down' };
  }

  // Core functionality
  health.checks.core = {
    memoryUsage: process.memoryUsage(),
    uptime: process.uptime()
  };

  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
});

```

### 2. Database Performance Monitoring

Monitor query performance to detect slow queries before they cause timeouts:

```jsx
// Query timeout wrapper
async function queryWithTimeout(query, params, timeout = 5000) {
  const start = Date.now();

  try {
    const result = await Promise.race([
      db.query(query, params),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Query timeout')), timeout)
      )
    ]);

    const duration = Date.now() - start;
    if (duration > 1000) {
      console.warn(`Slow query detected: ${duration}ms`, { query });
    }

    return result;
  } catch (error) {
    console.error('Query failed', { query, duration: Date.now() - start, error });
    throw error;
  }
}

```

### 3. Service Health Monitoring

Continuously monitor external service availability:

```jsx
class ServiceMonitor {
  constructor(services) {
    this.services = services;
    this.status = {};

    // Check every 30 seconds
    setInterval(() => this.checkAll(), 30000);
    this.checkAll(); // Initial check
  }

  async checkAll() {
    for (const [name, url] of Object.entries(this.services)) {
      try {
        const start = Date.now();
        await fetch(url, { timeout: 5000 });
        this.status[name] = {
          healthy: true,
          latency: Date.now() - start,
          lastCheck: new Date()
        };
      } catch (error) {
        this.status[name] = {
          healthy: false,
          error: error.message,
          lastCheck: new Date()
        };
      }
    }
  }

  isHealthy(serviceName) {
    return this.status[serviceName]?.healthy ?? false;
  }
}

```

---

## Handling Errors Gracefully

### 1. Immediate Error Response

For recoverable errors, attempt recovery immediately:

**Retry Mechanisms:**

```jsx
async function sendEmailWithRetry(to, subject, body) {
  const maxRetries = 3;
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      await emailService.send({ to, subject, body });
      return { success: true };
    } catch (error) {
      lastError = error;

      if (attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        await sleep(delay);
      }
    }
  }

  // Failed after retries - queue for later processing
  await emailQueue.add({ to, subject, body });
  return { success: false, queued: true };
}

```

### 2. Non-Recoverable Errors: Containment and Graceful Degradation

When errors can't be recovered, disable the affected feature while keeping the rest of the system operational:

```jsx
class FeatureFlags {
  constructor() {
    this.flags = new Map();
  }

  isEnabled(feature) {
    return this.flags.get(feature) !== false;
  }

  disable(feature, reason) {
    console.error(`Disabling feature: ${feature}`, { reason });
    this.flags.set(feature, false);
  }
}

const features = new FeatureFlags();

// In your route handler
app.get('/recommendations', async (req, res) => {
  if (!features.isEnabled('recommendations')) {
    return res.json({
      recommendations: [],
      message: 'Recommendations temporarily unavailable'
    });
  }

  try {
    const recommendations = await getRecommendations(req.user);
    res.json({ recommendations });
  } catch (error) {
    features.disable('recommendations', error.message);
    res.json({
      recommendations: [],
      message: 'Recommendations temporarily unavailable'
    });
  }
});

```

### 3. Error Recovery Strategies

**Service Restart:**

```jsx
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);

  // Attempt graceful shutdown
  gracefulShutdown()
    .then(() => process.exit(1))
    .catch(() => process.exit(1));
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection:', reason);
  // Don't exit on unhandled rejection, just log it
});

```

**Resource Cleanup:**

```jsx
class ResourceManager {
  constructor() {
    this.resources = [];
  }

  register(resource) {
    this.resources.push(resource);
  }

  async cleanupAll() {
    // Cleanup in reverse order of acquisition
    for (let i = this.resources.length - 1; i >= 0; i--) {
      try {
        await this.resources[i].cleanup();
      } catch (error) {
        console.error('Cleanup failed:', error);
      }
    }
  }
}

```

### 4. Error Propagation Control

Control how errors bubble up through your application layers with proper context:

**Error Boundaries:**

```jsx
// Repository Layer - Low-level errors
class UserRepository {
  async findByEmail(email) {
    try {
      const result = await db.query('SELECT * FROM users WHERE email = $1', [email]);
      return result.rows[0];
    } catch (error) {
      throw new DatabaseError('Failed to query user', { email, cause: error });
    }
  }
}

// Service Layer - Business logic errors
class UserService {
  async registerUser(email, password) {
    try {
      const existing = await userRepo.findByEmail(email);
      if (existing) {
        throw new ValidationError('Email already registered', { email });
      }

      const user = await userRepo.create({ email, password });
      return user;
    } catch (error) {
      if (error instanceof ValidationError) {
        throw error; // Pass validation errors up
      }
      throw new ServiceError('User registration failed', { cause: error });
    }
  }
}

// Controller Layer - HTTP errors
app.post('/register', async (req, res, next) => {
  try {
    const user = await userService.registerUser(req.body.email, req.body.password);
    res.status(201).json({ user });
  } catch (error) {
    next(error); // Pass to global error handler
  }
});

```

**Service-Level Boundaries:**
Ensure bugs in one service don't affect others through proper decoupling:

```jsx
// Service isolation with timeouts
async function callPaymentService(data) {
  try {
    return await Promise.race([
      paymentService.process(data),
      timeout(5000) // Kill after 5 seconds
    ]);
  } catch (error) {
    // Log but don't crash
    console.error('Payment service error:', error);
    throw new ServiceError('Payment processing failed', {
      recoverable: true
    });
  }
}

```

---

## Global Error Handling

Implement a centralized error handler to ensure consistent error responses:

```jsx
// Custom Error Classes
class AppError extends Error {
  constructor(message, statusCode = 500, details = {}) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.details = details;
    this.isOperational = true;
  }
}

class ValidationError extends AppError {
  constructor(message, fields = {}) {
    super(message, 400, { fields });
  }
}

class DatabaseError extends AppError {
  constructor(message, details = {}) {
    super(message, 500, details);
    this.isOperational = false; // Not safe to continue
  }
}

class NotFoundError extends AppError {
  constructor(resource, identifier) {
    super(`${resource} not found`, 404, { resource, identifier });
  }
}

// Global Error Handler Middleware
app.use((error, req, res, next) => {
  // Log error with context
  console.error('Error occurred:', {
    name: error.name,
    message: error.message,
    stack: error.stack,
    url: req.url,
    method: req.method,
    user: req.user?.id
  });

  // Don't expose internal details in production
  const isProduction = process.env.NODE_ENV === 'production';

  if (error instanceof ValidationError) {
    return res.status(error.statusCode).json({
      error: 'Validation failed',
      message: error.message,
      fields: error.details.fields
    });
  }

  if (error instanceof NotFoundError) {
    return res.status(404).json({
      error: 'Resource not found',
      message: error.message
    });
  }

  if (error instanceof DatabaseError) {
    // Database errors might contain sensitive info
    return res.status(500).json({
      error: 'Internal server error',
      message: isProduction ? 'An error occurred' : error.message
    });
  }

  // Generic error response
  const statusCode = error.statusCode || 500;
  res.status(statusCode).json({
    error: 'Server error',
    message: isProduction ? 'An unexpected error occurred' : error.message,
    ...(isProduction ? {} : { stack: error.stack })
  });
});

```

---

## Security Considerations

**Never expose internal details in error messages:**

```jsx
// ❌ BAD - Exposes internal structure
throw new Error('Database connection failed: postgres://admin:pass123@db.internal:5432');

// ✅ GOOD - Generic, safe message
throw new DatabaseError('Database connection failed');

// ❌ BAD - Reveals system information
res.status(500).json({
  error: 'Query failed: table users does not exist',
  query: 'SELECT * FROM users WHERE id = 1'
});

// ✅ GOOD - Safe, user-friendly message
res.status(500).json({
  error: 'An error occurred while processing your request'
});

```

**Security Principles:**

- Never return HTTP 500 errors with detailed stack traces in production
- Sanitize error messages to remove database schema information
- Don't expose API keys, credentials, or internal URLs
- Log detailed errors server-side, return generic messages to clients
- Use different error messages for production vs. development

---

## Configuration Management

### Types of Configuration

**1. Application Settings**

- Log level (debug, info, warn, error)
- Server port
- Connection pool size
- Request timeout values
- Rate limiting thresholds

**2. Database Configuration**

- Host and port
- Database name
- Username and password
- Connection pool settings
- Query timeouts
- SSL/TLS settings

**3. External Services**

- API keys and secrets (OpenAI, Stripe, Razorpay)
- Service URLs and endpoints
- Timeout values
- Retry configurations

**4. Feature Flags**

- Dynamic enable/disable of features
- A/B testing configurations
- Gradual rollout percentages

**5. Observability Configuration**

- Monitoring service credentials
- Log aggregation settings
- Metrics collection intervals

**6. Security Settings**

- JWT secret and expiration
- Password hashing rounds
- CORS origins
- Rate limiting rules

**7. Performance Tuning**

- CPU and memory limits
- Worker thread counts
- Cache sizes and TTLs

### Configuration Storage

**Environment Variables (.env file):**

```bash
# Application
NODE_ENV=production
PORT=3000
LOG_LEVEL=info

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
DB_POOL_SIZE=20
DB_TIMEOUT=5000

# External Services
OPENAI_API_KEY=sk-xxx
STRIPE_SECRET_KEY=sk_live_xxx
RAZORPAY_KEY_ID=rzp_live_xxx
RAZORPAY_KEY_SECRET=xxx

# Security
JWT_SECRET=your-secret-key-min-32-characters
JWT_EXPIRES_IN=7d

# Feature Flags
ENABLE_RECOMMENDATIONS=true
ENABLE_ANALYTICS=false

```

### Configuration Validation

**Comprehensive validation at startup:**

```jsx
const { z } = require('zod');

const configSchema = z.object({
  // Application
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.string().regex(/^\d+$/).transform(Number),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']),

  // Database
  DATABASE_URL: z.string().url().startsWith('postgresql://'),
  DB_POOL_SIZE: z.string().regex(/^\d+$/).transform(Number).default('10'),
  DB_TIMEOUT: z.string().regex(/^\d+$/).transform(Number).default('5000'),

  // External Services
  OPENAI_API_KEY: z.string().min(1),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),

  // Security
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  JWT_EXPIRES_IN: z.string().default('7d'),

  // Feature Flags
  ENABLE_RECOMMENDATIONS: z.string().transform(val => val === 'true').default('true'),
});

function loadConfig() {
  try {
    require('dotenv').config();
    const config = configSchema.parse(process.env);
    return config;
  } catch (error) {
    console.error('Configuration validation failed:');
    if (error instanceof z.ZodError) {
      error.errors.forEach(err => {
        console.error(`  ${err.path.join('.')}: ${err.message}`);
      });
    }
    process.exit(1);
  }
}

// Load and validate config before starting
const config = loadConfig();

```

---

## Graceful Shutdown

Graceful shutdown ensures that in-flight requests complete and resources are properly released when the application stops.

### Process Lifecycle Management

**Understanding Signals:**

- **SIGTERM**: Graceful shutdown request (default from `kill` command)
- **SIGINT**: Interrupt signal (Ctrl+C)
- **SIGKILL**: Forceful termination (cannot be caught)

### Implementing Graceful Shutdown

```jsx
class GracefulShutdown {
  constructor(server) {
    this.server = server;
    this.isShuttingDown = false;
    this.connections = new Set();

    // Track active connections
    server.on('connection', (conn) => {
      this.connections.add(conn);
      conn.on('close', () => this.connections.delete(conn));
    });
  }

  async shutdown(signal) {
    if (this.isShuttingDown) {
      console.log('Shutdown already in progress');
      return;
    }

    this.isShuttingDown = true;
    console.log(`Received ${signal}, starting graceful shutdown...`);

    // 1. Stop accepting new connections
    this.server.close(() => {
      console.log('Server closed to new connections');
    });

    // 2. Set a deadline for existing requests
    const shutdownTimeout = setTimeout(() => {
      console.error('Shutdown timeout exceeded, forcing exit');
      this.connections.forEach(conn => conn.destroy());
      process.exit(1);
    }, 30000); // 30 second deadline

    // 3. Connection draining - wait for existing requests
    console.log(`Waiting for ${this.connections.size} active connections...`);
    await this.waitForConnections();

    // 4. Cleanup resources in reverse order of acquisition
    await this.cleanupResources();

    clearTimeout(shutdownTimeout);
    console.log('Graceful shutdown complete');
    process.exit(0);
  }

  async waitForConnections() {
    return new Promise((resolve) => {
      const check = setInterval(() => {
        if (this.connections.size === 0) {
          clearInterval(check);
          console.log('All connections closed');
          resolve();
        } else {
          console.log(`${this.connections.size} connections remaining...`);
        }
      }, 1000);
    });
  }

  async cleanupResources() {
    console.log('Cleaning up resources...');

    try {
      // Close database connections
      if (global.dbPool) {
        console.log('Closing database pool...');
        await global.dbPool.end();
      }

      // Close Redis connections
      if (global.redisClient) {
        console.log('Closing Redis connection...');
        await global.redisClient.quit();
      }

      // Close message queues
      if (global.messageQueue) {
        console.log('Closing message queue...');
        await global.messageQueue.close();
      }

      // Close file handles
      // Close network connections
      // Any other cleanup needed

      console.log('Resource cleanup complete');
    } catch (error) {
      console.error('Error during cleanup:', error);
      // Continue with shutdown even if cleanup fails
    }
  }
}

// Setup graceful shutdown
const gracefulShutdown = new GracefulShutdown(server);

process.on('SIGTERM', () => gracefulShutdown.shutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown.shutdown('SIGINT'));

// Handle uncaught errors
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
  gracefulShutdown.shutdown('uncaughtException');
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection at:', promise, 'reason:', reason);
  gracefulShutdown.shutdown('unhandledRejection');
});

```

### Shutdown Checklist

1. **Stop accepting new requests** - Server stops listening for new connections
2. **Connection draining** - Allow 30-60 seconds for existing requests to complete
3. **Resource cleanup** - Close resources in reverse order of acquisition:
    - Message queue connections
    - Redis/cache connections
    - Database connection pool
    - File handles
    - Network sockets
4. **Exit process** - Exit with appropriate code (0 for success, 1 for error)

---

## Best Practices Summary

**Error Handling:**

- Classify errors by type and handle them appropriately
- Use custom error classes for different scenarios
- Implement retry logic with exponential backoff for transient failures
- Use circuit breakers to prevent cascading failures
- Never expose internal details in error messages
- Log detailed errors server-side, return generic messages to clients

**Prevention:**

- Implement comprehensive health checks
- Monitor database query performance
- Validate configuration at startup (fail-fast)
- Use feature flags for graceful degradation
- Test edge cases and failure scenarios

**Configuration:**

- Store configuration in environment variables
- Validate all configuration at startup
- Use schema validation libraries (zod, joi)
- Never commit secrets to version control
- Document all configuration options

**Graceful Shutdown:**

- Register signal handlers for SIGTERM and SIGINT
- Stop accepting new connections immediately
- Allow existing requests to complete (with timeout)
- Clean up resources in reverse order
- Log shutdown progress for debugging

Remember: The goal is not to prevent all errors, but to handle them gracefully while maintaining system stability and security.

# Error Handling and Fault-Tolerant Systems: Executive Summary

Building resilient systems requires anticipating failure modes and handling them gracefully. This guide provides a comprehensive overview of error handling strategies, configuration management, and graceful shutdown patterns.

## Understanding System Failures

Real-world systems face constant challenges:

- **Database queries fail** due to connection issues, deadlocks, or constraint violations
- **External APIs timeout** from network problems, rate limiting, or service outages
- **Users send invalid data** with incorrect formats, missing fields, or out-of-range values
- **Edge cases emerge** that weren't anticipated during development

The goal isn't to prevent all errors—that's impossible—but to handle them gracefully and maintain system stability.

---

## Types of Errors

### 1. Logic Errors

Logic errors are the most dangerous because the application continues running but produces incorrect results.

**Characteristics:**

- Application doesn't crash but generates wrong outputs
- Incorrect algorithm implementation
- Missing edge case handling
- Data corruption that compounds over time
- Silent failures that go undetected

**Examples:** Incorrect discount calculations, timezone conversion bugs, off-by-one errors in pagination, race conditions in concurrent operations

**Prevention:** Comprehensive unit testing, property-based testing, thorough code reviews focusing on business logic, integration tests with realistic data, monitoring for anomalous outputs

### 2. Database Errors

Database operations are inherently failure-prone due to network issues, resource constraints, and data integrity requirements.

**Connection Errors:**

- Database server unreachable
- Connection pool exhausted
- Authentication failures
- Network timeouts

**Constraint Violations:**

- Unique constraints (duplicate email addresses)
- Foreign key constraints (referencing non-existent records)
- Check constraints (invalid data ranges)
- Not-null constraints

**Query Errors:**

- SQL syntax errors or typos
- Invalid table or column names
- Type mismatches
- Deadlock conditions
- Transaction timeouts

**Handling Strategies:** Implement retry logic for transient errors with exponential backoff. Classify permanent errors (like constraint violations) and convert them to appropriate validation errors. Never retry non-transient errors.

### 3. External Service Errors

External dependencies introduce multiple failure modes beyond your control.

**Failure Types:**

- **Network failures:** DNS resolution failures, connection refused, network partitions
- **Timeouts:** Connection timeouts, read timeouts, slow responses
- **Rate limiting:** HTTP 429 responses when quota exceeded
- **Service outages:** Complete unavailability of third-party services
- **Invalid responses:** Malformed JSON, unexpected data structures

**Resilience Strategies:**

**Exponential Backoff:** When receiving rate limit errors (HTTP 429) or transient failures, wait progressively longer between retry attempts. Start with 1 second, then 2, 4, 8, etc. Respect the Retry-After header if provided.

**Circuit Breaker Pattern:** Prevents cascading failures by stopping calls to failing services temporarily. After a threshold of failures (e.g., 5 consecutive failures), the circuit "opens" and immediately rejects requests for a timeout period (e.g., 60 seconds). After the timeout, allow one test request (half-open state) to check if the service has recovered.

### 4. Input Validation Errors

User input is the primary attack vector and source of invalid data.

**Common Validation Failures:**

- Missing required fields
- Invalid data formats (email, phone, date)
- Out-of-range values (negative quantities, future birth dates)
- Type mismatches (string instead of number)
- Malicious input (SQL injection attempts, XSS payloads)

**Response:** Always return HTTP 400 Bad Request with specific, helpful validation errors that guide the user to fix their input.

### 5. Configuration Errors

Configuration errors should cause the application to fail immediately at startup, not at runtime.

**Fail-Fast Principle:** Validate all required configuration values before the application starts accepting requests. Check for missing values, invalid formats, insufficient security (like short JWT secrets), and type mismatches. If configuration is invalid, log clear error messages and exit immediately with a non-zero status code.

---

## Prevention Strategies

### 1. Health Checks

Implement comprehensive health checks to detect issues before they affect users.

**Health Check Components:**

- **Application status:** Basic liveness check confirming the app is running
- **Database health:** Run a simple query (SELECT 1) to verify database connectivity
- **External service health:** Ping external APIs to check availability
- **Core functionality:** Memory usage, uptime, disk space
- **Performance metrics:** Response times, queue lengths, connection pool status

Return HTTP 200 for healthy systems, HTTP 503 for unhealthy systems. Include detailed status for each component to aid debugging.

### 2. Database Performance Monitoring

Monitor query performance to detect slow queries before they cause timeouts.

**Monitoring Strategies:**

- Set query timeout limits (e.g., 5 seconds max)
- Log slow queries that exceed thresholds (e.g., > 1 second)
- Track query duration and frequency
- Alert on queries approaching timeout limits
- Monitor connection pool exhaustion

This helps identify problematic queries early and optimize them before they impact users.

### 3. Service Health Monitoring

Continuously monitor external service availability with periodic health checks.

**Implementation:**

- Check external services every 30-60 seconds
- Track response latency and availability
- Maintain service health status in memory
- Use health status to enable graceful degradation
- Alert when services are consistently down

This allows your system to proactively disable features that depend on failing services.

---

## Handling Errors Gracefully

### 1. Immediate Error Response

For recoverable errors, attempt recovery immediately using retry mechanisms.

**Retry with Exponential Backoff:**

- For transient failures (network issues, temporary service unavailability)
- Retry 3-5 times with increasing delays between attempts
- If all retries fail, consider queueing the operation for later processing
- Common use case: Sending emails, processing payments, calling external APIs

### 2. Non-Recoverable Errors: Containment and Graceful Degradation

When errors can't be recovered, disable the affected feature while keeping the rest of the system operational.

**Feature Flag Approach:**

- Maintain a registry of feature flags in memory
- When a feature repeatedly fails, automatically disable it
- Return friendly messages to users explaining the feature is temporarily unavailable
- Continue serving other features normally
- Re-enable features automatically once the underlying issue is resolved

**Example:** If the recommendations service fails, disable recommendations and return empty results with a friendly message, but allow the rest of the application to function normally.

### 3. Error Recovery Strategies

**Service Restart:**

- Catch uncaught exceptions and unhandled promise rejections
- Attempt graceful shutdown before restarting
- Use process managers (PM2, systemd) to automatically restart crashed processes
- Log crash information for debugging

**Resource Cleanup:**

- Maintain a registry of acquired resources (database connections, file handles, network sockets)
- On error, clean up resources in reverse order of acquisition
- Ensure cleanup happens even when errors occur during cleanup itself
- Prevent resource leaks that accumulate over time

### 4. Error Propagation Control

Control how errors bubble up through application layers with proper context and boundaries.

**Layer-Based Error Handling:**

**Repository Layer (Database):**

- Catches low-level database errors
- Wraps them in domain-specific error types
- Adds context like which query failed and why

**Service Layer (Business Logic):**

- Catches repository errors and validation errors
- Adds business context
- Converts technical errors to business errors
- Decides which errors can be recovered

**Controller Layer (HTTP):**

- Catches all errors from lower layers
- Converts to appropriate HTTP responses
- Passes errors to global error handler

**Service-Level Boundaries:**
Ensure bugs in one service don't affect others through proper isolation:

- Set timeouts for service-to-service calls
- Use circuit breakers between services
- Handle service failures gracefully without crashing the caller
- Log errors but don't propagate them beyond service boundaries

---

## Global Error Handling

Implement a centralized error handler to ensure consistent error responses across your application.

**Custom Error Classes:**
Create specific error types for different scenarios:

- **ValidationError** - HTTP 400 for invalid user input with field-level details
- **NotFoundError** - HTTP 404 for missing resources
- **DatabaseError** - HTTP 500 for database failures (not exposed to users)
- **ServiceError** - HTTP 500 for internal service failures
- **UnauthorizedError** - HTTP 401 for authentication failures
- **ForbiddenError** - HTTP 403 for authorization failures

**Global Error Handler Middleware:**
Catch all errors that bubble up from any layer and convert them to appropriate HTTP responses. The handler should:

- Log detailed error information (stack trace, request context, user info) for debugging
- Return user-friendly error messages
- Hide internal implementation details in production
- Set appropriate HTTP status codes
- Include field-level validation errors when available
- Maintain consistent error response format across the application

**Error Flow:**

1. Repository throws DatabaseError
2. Service catches it, adds business context, throws ServiceError
3. Controller catches it, passes to next()
4. Global error handler catches it, logs details, returns safe HTTP response

---

## Security Considerations

**Never expose internal details in error messages.** This is critical for security.

**What NOT to expose:**

- Database connection strings or credentials
- Internal table/column names or schema details
- Stack traces in production
- API keys or secrets
- Internal server IP addresses or hostnames
- Exact SQL queries that failed
- File system paths
- Detailed system information

**What TO do:**

- Return generic error messages to users in production
- Log detailed errors server-side for debugging
- Use different error messages for development vs. production environments
- Sanitize error messages to remove sensitive information
- Never return HTTP 500 errors with detailed stack traces in production

**Example Safe Responses:**

- ❌ BAD: "Database connection failed: postgres://admin:pass123@db.internal:5432"
- ✅ GOOD: "Database connection failed"
- ❌ BAD: "Query failed: table users does not exist"
- ✅ GOOD: "An error occurred while processing your request"

**Security Principles:**

- Treat all error messages as potential information leaks
- Assume attackers will try to extract information from error responses
- Use consistent, generic error messages for similar failure types
- Log everything server-side, expose nothing to clients

---

## Configuration Management

### Types of Configuration

**1. Application Settings**

- Log level (debug, info, warn, error)
- Server port
- Connection pool size
- Request timeout values
- Rate limiting thresholds

**2. Database Configuration**

- Host and port
- Database name
- Username and password
- Connection pool settings (min/max connections, idle timeout)
- Query timeouts
- SSL/TLS settings

**3. External Services**

- API keys and secrets (OpenAI, Stripe, Razorpay, payment gateways)
- Service URLs and endpoints
- Timeout values for each service
- Retry configurations

**4. Feature Flags**

- Dynamic enable/disable of features
- A/B testing configurations
- Gradual rollout percentages
- Beta feature access

**5. Observability Configuration**

- Monitoring service credentials (New Relic, Datadog)
- Log aggregation settings (Splunk, ELK)
- Metrics collection intervals
- APM configuration

**6. Security Settings**

- JWT secret and expiration time
- Password hashing rounds (bcrypt)
- CORS allowed origins
- Rate limiting rules
- Session timeout values
- Encryption keys

**7. Performance Tuning**

- CPU and memory limits
- Worker thread/process counts
- Cache sizes and TTLs
- Connection pool sizes
- Request queue limits

### Configuration Storage

**Environment Variables (.env file):**
Store all configuration as environment variables, never hardcode values in source code. Use a .env file for local development and environment variables in production.

**Structure:**
Organize by category (Application, Database, External Services, Security, Features). Use clear, descriptive names with consistent prefixes (DB_, API_, FEATURE_). Include comments explaining non-obvious values.

**Security:**

- Never commit .env files to version control
- Use .gitignore to exclude .env files
- Rotate secrets regularly
- Use separate configurations for development, staging, and production
- Consider secret management services (AWS Secrets Manager, HashiCorp Vault) for sensitive production credentials

### Configuration Validation

**Fail-Fast at Startup:**
Validate all configuration before starting the application. Check for:

- Missing required values
- Invalid formats (URLs, numbers, booleans)
- Insufficient security (weak passwords, short secrets)
- Type mismatches
- Out-of-range values

**Validation Library:**
Use schema validation libraries like Zod, Joi, or Yup to define expected configuration structure and automatically validate at startup.

**Benefits:**

- Catch configuration errors immediately, not at runtime
- Clear error messages showing exactly what's wrong
- Self-documenting configuration requirements
- Type safety for configuration values
- Automatic type conversion (string "3000" → number 3000)

---

## Graceful Shutdown

Graceful shutdown ensures that in-flight requests complete and resources are properly released when the application stops.

### Process Lifecycle Management

**Understanding Unix Signals:**
Applications receive signals from the operating system to control their lifecycle:

- **SIGTERM**: Graceful shutdown request (default from `kill` command, Kubernetes, Docker)
- **SIGINT**: Interrupt signal (Ctrl+C in terminal)
- **SIGKILL**: Forceful termination (cannot be caught or handled)

Applications should register handlers for SIGTERM and SIGINT to perform graceful shutdown.

### Graceful Shutdown Process

**Step 1: Stop Accepting New Connections**
When a shutdown signal is received, immediately stop the server from accepting new connections. Existing connections continue to be processed.

**Step 2: Connection Draining (30-60 seconds)**
Allow time for existing requests to complete. Set a reasonable deadline (typically 30-60 seconds). If requests don't complete within the deadline, forcefully terminate them.

**Step 3: Resource Cleanup (Reverse Order)**
Clean up resources in the reverse order they were acquired:

1. Close message queue connections (stop consuming new messages)
2. Close Redis/cache connections
3. Close database connection pool
4. Close file handles
5. Close network sockets

Clean up each resource even if previous cleanup steps fail. Log any cleanup errors but continue with remaining cleanup.

**Step 4: Exit Process**
Exit with appropriate status code:

- Exit 0 for successful graceful shutdown
- Exit 1 for errors or forced shutdown

### Implementation Checklist

**Register Signal Handlers:**
Listen for SIGTERM and SIGINT signals and trigger graceful shutdown when received.

**Set Shutdown Deadline:**
Use a timeout (30-60 seconds) to prevent shutdown from hanging indefinitely. After the deadline, forcefully close all connections and exit.

**Track Active Connections:**
Maintain a set of active connections so you know when all requests have completed.

**Stop Accepting New Requests:**
Call server.close() to stop accepting new connections while allowing existing ones to finish.

**Wait for Existing Requests:**
Monitor active connections and wait for them to complete naturally. Log progress periodically.

**Clean Up Resources:**
Close all acquired resources in reverse order. Handle errors during cleanup gracefully without stopping the cleanup process.

**Exit Cleanly:**
Once cleanup is complete or the deadline is reached, exit the process with an appropriate status code.

### Best Practices

**Use Process Managers:**

- PM2, systemd, or Docker/Kubernetes handle process restarts
- They send SIGTERM before SIGKILL
- Configure appropriate grace periods

**Handle Uncaught Errors:**
Register handlers for uncaughtException and unhandledRejection that trigger graceful shutdown, preventing the process from staying in an undefined state.

**Log Shutdown Progress:**
Log each step of the shutdown process (signal received, connections draining, resources cleaned up, exit) to help debug shutdown issues.

**Test Graceful Shutdown:**
Regularly test your shutdown process in development and staging environments. Ensure it completes within the deadline and doesn't leave resources hanging.

**Configure Load Balancers:**
Ensure your load balancer respects health checks and stops sending traffic to shutting-down instances.

---

## Best Practices Summary

### Error Handling

**Classification and Handling:**

- Classify errors by type (logic, database, external service, validation, configuration)
- Handle each type appropriately with specific strategies
- Use custom error classes for different scenarios
- Implement global error handler middleware for consistency

**Retry and Recovery:**

- Implement retry logic with exponential backoff for transient failures
- Use circuit breakers to prevent cascading failures
- Set appropriate timeouts for all external calls
- Queue failed operations for later processing when appropriate

**Error Boundaries:**

- Create clear boundaries between application layers
- Add context as errors bubble up through layers
- Isolate services so failures don't cascade
- Use timeouts to prevent hanging operations

**Logging and Monitoring:**

- Log detailed errors server-side with full context
- Return generic, safe error messages to clients
- Never expose internal system details
- Monitor error rates and patterns

### Prevention

**Health Checks:**

- Implement comprehensive health check endpoints
- Monitor database connectivity and performance
- Check external service availability
- Track system resources (memory, CPU, disk)

**Validation:**

- Validate configuration at startup (fail-fast principle)
- Validate user input at API boundaries
- Use schema validation libraries
- Provide clear, helpful validation error messages

**Monitoring:**

- Track query performance and set alerts
- Monitor external service health continuously
- Log slow operations before they become timeouts
- Set up alerts for abnormal patterns

### Configuration Management

**Organization:**

- Store all configuration in environment variables
- Never hardcode values in source code
- Organize by category with consistent naming
- Use separate configurations for each environment

**Security:**

- Never commit secrets to version control
- Rotate secrets regularly
- Use secret management services for production
- Validate security-related configuration (min lengths, formats)

**Validation:**

- Validate all configuration at startup before accepting requests
- Use schema validation libraries for type safety
- Provide clear error messages for invalid configuration
- Exit immediately on configuration errors (fail-fast)

### Graceful Shutdown

**Signal Handling:**

- Register handlers for SIGTERM and SIGINT
- Implement proper shutdown sequence
- Set reasonable deadlines (30-60 seconds)
- Log shutdown progress

**Resource Management:**

- Stop accepting new connections immediately
- Allow existing requests to complete
- Clean up resources in reverse order of acquisition
- Handle cleanup errors gracefully

**Process Management:**

- Use process managers for automatic restarts
- Handle uncaught exceptions and unhandled rejections
- Exit with appropriate status codes
- Test shutdown process regularly

### Security

**Error Messages:**

- Never expose database schema or queries
- Hide stack traces in production
- Sanitize all error messages
- Log detailed errors server-side only

**Information Leakage:**

- Don't reveal internal IP addresses or hostnames
- Hide API keys and credentials
- Use generic error messages for similar failure types
- Treat all errors as potential security vulnerabilities

---

## Key Takeaways

The goal of fault-tolerant systems is not to prevent all errors—that's impossible—but to:

1. **Anticipate common failure modes** and handle them gracefully
2. **Fail fast** for configuration errors at startup
3. **Recover automatically** from transient failures with retries and circuit breakers
4. **Degrade gracefully** when recovery isn't possible by disabling features
5. **Maintain security** by never exposing internal details
6. **Clean up properly** during shutdown to prevent resource leaks
7. **Monitor continuously** to detect issues before they impact users
8. **Learn from failures** by logging detailed information for debugging

Building resilient systems is an ongoing process of identifying failure points, implementing appropriate handling strategies, and continuously monitoring and improving based on real-world behavior.

# Error Handling and Fault-Tolerant Systems: Executive Summary

Building resilient systems requires anticipating failure modes and handling them gracefully. This guide provides a comprehensive overview of error handling strategies, configuration management, and graceful shutdown patterns.

## Understanding System Failures

Real-world systems face constant challenges:

- **Database queries fail** due to connection issues, deadlocks, or constraint violations
- **External APIs timeout** from network problems, rate limiting, or service outages
- **Users send invalid data** with incorrect formats, missing fields, or out-of-range values
- **Edge cases emerge** that weren't anticipated during development

The goal isn't to prevent all errors—that's impossible—but to handle them gracefully and maintain system stability.

---

## Types of Errors

### 1. Logic Errors

Logic errors occur when the application doesn't crash but produces incorrect results. These are often the most dangerous because they can go undetected.

**Characteristics:**

- Application continues running normally
- Incorrect algorithm implementation
- Missing edge case handling
- Data corruption over time
- Silent failures that compound

**Examples:**

- Calculating discounts incorrectly
- Processing timezone conversions wrong
- Off-by-one errors in pagination
- Race conditions in concurrent operations

**Prevention:**

- Comprehensive unit testing
- Property-based testing for edge cases
- Code reviews focusing on business logic
- Integration tests with realistic data
- Monitoring for anomalous outputs

### 2. Database Errors

Database operations are inherently failure-prone due to network issues, resource constraints, and data integrity requirements.

**Common Database Errors:**

**Connection Errors:**

- Database server unreachable
- Connection pool exhausted
- Authentication failures
- Network timeouts

**Constraint Violations:**

- Unique constraint violations (duplicate email addresses)
- Foreign key constraint failures (referencing non-existent records)
- Check constraint violations (invalid data ranges)
- Not-null constraint violations

**Query Errors:**

- SQL syntax errors or typos
- Invalid table or column names
- Type mismatches
- Deadlock conditions
- Transaction timeout

**Handling Strategies:**

```jsx
// Example: Database error handling with retries
async function executeWithRetry(queryFn, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await queryFn();
    } catch (error) {
      // Retry on transient errors
      if (isTransientError(error) && attempt < maxRetries) {
        await delay(Math.pow(2, attempt) * 100); // Exponential backoff
        continue;
      }

      // Classify and handle permanent errors
      if (error.code === '23505') {
        throw new ValidationError('Record already exists');
      }
      if (error.code === '23503') {
        throw new ValidationError('Referenced record not found');
      }

      throw error;
    }
  }
}

function isTransientError(error) {
  const transientCodes = ['57P03', 'ECONNRESET', 'ETIMEDOUT'];
  return transientCodes.includes(error.code);
}

```

### 3. External Service Errors

External dependencies introduce multiple failure modes beyond your control.

**Failure Types:**

- **Network failures:** DNS resolution failures, connection refused, network partitions
- **Timeouts:** Connection timeouts, read timeouts, slow responses
- **Rate limiting:** HTTP 429 responses when quota exceeded
- **Service outages:** Complete unavailability of third-party services
- **Invalid responses:** Malformed JSON, unexpected data structures

**Resilience Strategies:**

**Exponential Backoff:**

```jsx
async function callExternalAPI(url, maxRetries = 5) {
  let delay = 1000; // Start with 1 second

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, { timeout: 5000 });

      if (response.status === 429) {
        // Rate limited - respect Retry-After header
        const retryAfter = response.headers.get('Retry-After');
        await sleep(retryAfter ? parseInt(retryAfter) * 1000 : delay);
        delay *= 2; // Exponential backoff
        continue;
      }

      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }

      return await response.json();
    } catch (error) {
      if (attempt === maxRetries) {
        throw new ServiceUnavailableError('External API failed after retries');
      }

      await sleep(delay);
      delay *= 2; // Exponential backoff
    }
  }
}

```

**Circuit Breaker Pattern:**
Prevents cascading failures by stopping calls to failing services temporarily:

```jsx
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.failureCount = 0;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = Date.now();
  }

  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}

```

### 4. Input Validation Errors

User input is the primary attack vector and source of invalid data.

**Common Validation Failures:**

- Missing required fields
- Invalid data formats (email, phone, date)
- Out-of-range values (negative quantities, future birth dates)
- Type mismatches (string instead of number)
- Malicious input (SQL injection attempts, XSS payloads)

**Response:** Return HTTP 400 Bad Request with specific validation errors.

```jsx
class ValidationError extends Error {
  constructor(message, fields = {}) {
    super(message);
    this.name = 'ValidationError';
    this.statusCode = 400;
    this.fields = fields;
  }
}

// Usage
if (!email || !isValidEmail(email)) {
  throw new ValidationError('Invalid input', {
    email: 'Valid email address is required'
  });
}

```

### 5. Configuration Errors

Configuration errors should cause the application to fail immediately at startup, not at runtime.

**Fail-Fast Principle:** Validate all required configuration values before the application starts accepting requests.

```jsx
// Configuration validation at startup
function validateConfig() {
  const required = [
    'DATABASE_URL',
    'JWT_SECRET',
    'API_KEY',
    'PORT'
  ];

  const missing = required.filter(key => !process.env[key]);

  if (missing.length > 0) {
    console.error(`Missing required configuration: ${missing.join(', ')}`);
    process.exit(1); // Fail fast
  }

  // Validate formats
  if (isNaN(process.env.PORT)) {
    console.error('PORT must be a number');
    process.exit(1);
  }

  if (process.env.JWT_SECRET.length < 32) {
    console.error('JWT_SECRET must be at least 32 characters');
    process.exit(1);
  }
}

// Run before starting server
validateConfig();

```

---

## Prevention Strategies

### 1. Health Checks

Implement comprehensive health checks to detect issues before they affect users.

**Health Check Endpoints:**

```jsx
app.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    checks: {}
  };

  // Database health
  try {
    await db.query('SELECT 1');
    health.checks.database = { status: 'up' };
  } catch (error) {
    health.status = 'unhealthy';
    health.checks.database = { status: 'down', error: error.message };
  }

  // External service health
  try {
    await fetch('https://api.external.com/health', { timeout: 2000 });
    health.checks.externalAPI = { status: 'up' };
  } catch (error) {
    health.checks.externalAPI = { status: 'down' };
  }

  // Core functionality
  health.checks.core = {
    memoryUsage: process.memoryUsage(),
    uptime: process.uptime()
  };

  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
});

```

### 2. Database Performance Monitoring

Monitor query performance to detect slow queries before they cause timeouts:

```jsx
// Query timeout wrapper
async function queryWithTimeout(query, params, timeout = 5000) {
  const start = Date.now();

  try {
    const result = await Promise.race([
      db.query(query, params),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Query timeout')), timeout)
      )
    ]);

    const duration = Date.now() - start;
    if (duration > 1000) {
      console.warn(`Slow query detected: ${duration}ms`, { query });
    }

    return result;
  } catch (error) {
    console.error('Query failed', { query, duration: Date.now() - start, error });
    throw error;
  }
}

```

### 3. Service Health Monitoring

Continuously monitor external service availability:

```jsx
class ServiceMonitor {
  constructor(services) {
    this.services = services;
    this.status = {};

    // Check every 30 seconds
    setInterval(() => this.checkAll(), 30000);
    this.checkAll(); // Initial check
  }

  async checkAll() {
    for (const [name, url] of Object.entries(this.services)) {
      try {
        const start = Date.now();
        await fetch(url, { timeout: 5000 });
        this.status[name] = {
          healthy: true,
          latency: Date.now() - start,
          lastCheck: new Date()
        };
      } catch (error) {
        this.status[name] = {
          healthy: false,
          error: error.message,
          lastCheck: new Date()
        };
      }
    }
  }

  isHealthy(serviceName) {
    return this.status[serviceName]?.healthy ?? false;
  }
}

```

---

## Handling Errors Gracefully

### 1. Immediate Error Response

For recoverable errors, attempt recovery immediately:

**Retry Mechanisms:**

```jsx
async function sendEmailWithRetry(to, subject, body) {
  const maxRetries = 3;
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      await emailService.send({ to, subject, body });
      return { success: true };
    } catch (error) {
      lastError = error;

      if (attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        await sleep(delay);
      }
    }
  }

  // Failed after retries - queue for later processing
  await emailQueue.add({ to, subject, body });
  return { success: false, queued: true };
}

```

### 2. Non-Recoverable Errors: Containment and Graceful Degradation

When errors can't be recovered, disable the affected feature while keeping the rest of the system operational:

```jsx
class FeatureFlags {
  constructor() {
    this.flags = new Map();
  }

  isEnabled(feature) {
    return this.flags.get(feature) !== false;
  }

  disable(feature, reason) {
    console.error(`Disabling feature: ${feature}`, { reason });
    this.flags.set(feature, false);
  }
}

const features = new FeatureFlags();

// In your route handler
app.get('/recommendations', async (req, res) => {
  if (!features.isEnabled('recommendations')) {
    return res.json({
      recommendations: [],
      message: 'Recommendations temporarily unavailable'
    });
  }

  try {
    const recommendations = await getRecommendations(req.user);
    res.json({ recommendations });
  } catch (error) {
    features.disable('recommendations', error.message);
    res.json({
      recommendations: [],
      message: 'Recommendations temporarily unavailable'
    });
  }
});

```

### 3. Error Recovery Strategies

**Service Restart:**

```jsx
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);

  // Attempt graceful shutdown
  gracefulShutdown()
    .then(() => process.exit(1))
    .catch(() => process.exit(1));
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection:', reason);
  // Don't exit on unhandled rejection, just log it
});

```

**Resource Cleanup:**

```jsx
class ResourceManager {
  constructor() {
    this.resources = [];
  }

  register(resource) {
    this.resources.push(resource);
  }

  async cleanupAll() {
    // Cleanup in reverse order of acquisition
    for (let i = this.resources.length - 1; i >= 0; i--) {
      try {
        await this.resources[i].cleanup();
      } catch (error) {
        console.error('Cleanup failed:', error);
      }
    }
  }
}

```

### 4. Error Propagation Control

Control how errors bubble up through your application layers with proper context:

**Error Boundaries:**

```jsx
// Repository Layer - Low-level errors
class UserRepository {
  async findByEmail(email) {
    try {
      const result = await db.query('SELECT * FROM users WHERE email = $1', [email]);
      return result.rows[0];
    } catch (error) {
      throw new DatabaseError('Failed to query user', { email, cause: error });
    }
  }
}

// Service Layer - Business logic errors
class UserService {
  async registerUser(email, password) {
    try {
      const existing = await userRepo.findByEmail(email);
      if (existing) {
        throw new ValidationError('Email already registered', { email });
      }

      const user = await userRepo.create({ email, password });
      return user;
    } catch (error) {
      if (error instanceof ValidationError) {
        throw error; // Pass validation errors up
      }
      throw new ServiceError('User registration failed', { cause: error });
    }
  }
}

// Controller Layer - HTTP errors
app.post('/register', async (req, res, next) => {
  try {
    const user = await userService.registerUser(req.body.email, req.body.password);
    res.status(201).json({ user });
  } catch (error) {
    next(error); // Pass to global error handler
  }
});

```

**Service-Level Boundaries:**
Ensure bugs in one service don't affect others through proper decoupling:

```jsx
// Service isolation with timeouts
async function callPaymentService(data) {
  try {
    return await Promise.race([
      paymentService.process(data),
      timeout(5000) // Kill after 5 seconds
    ]);
  } catch (error) {
    // Log but don't crash
    console.error('Payment service error:', error);
    throw new ServiceError('Payment processing failed', {
      recoverable: true
    });
  }
}

```

---

## Global Error Handling

Implement a centralized error handler to ensure consistent error responses:

```jsx
// Custom Error Classes
class AppError extends Error {
  constructor(message, statusCode = 500, details = {}) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.details = details;
    this.isOperational = true;
  }
}

class ValidationError extends AppError {
  constructor(message, fields = {}) {
    super(message, 400, { fields });
  }
}

class DatabaseError extends AppError {
  constructor(message, details = {}) {
    super(message, 500, details);
    this.isOperational = false; // Not safe to continue
  }
}

class NotFoundError extends AppError {
  constructor(resource, identifier) {
    super(`${resource} not found`, 404, { resource, identifier });
  }
}

// Global Error Handler Middleware
app.use((error, req, res, next) => {
  // Log error with context
  console.error('Error occurred:', {
    name: error.name,
    message: error.message,
    stack: error.stack,
    url: req.url,
    method: req.method,
    user: req.user?.id
  });

  // Don't expose internal details in production
  const isProduction = process.env.NODE_ENV === 'production';

  if (error instanceof ValidationError) {
    return res.status(error.statusCode).json({
      error: 'Validation failed',
      message: error.message,
      fields: error.details.fields
    });
  }

  if (error instanceof NotFoundError) {
    return res.status(404).json({
      error: 'Resource not found',
      message: error.message
    });
  }

  if (error instanceof DatabaseError) {
    // Database errors might contain sensitive info
    return res.status(500).json({
      error: 'Internal server error',
      message: isProduction ? 'An error occurred' : error.message
    });
  }

  // Generic error response
  const statusCode = error.statusCode || 500;
  res.status(statusCode).json({
    error: 'Server error',
    message: isProduction ? 'An unexpected error occurred' : error.message,
    ...(isProduction ? {} : { stack: error.stack })
  });
});

```

---

## Security Considerations

**Never expose internal details in error messages:**

```jsx
// ❌ BAD - Exposes internal structure
throw new Error('Database connection failed: postgres://admin:pass123@db.internal:5432');

// ✅ GOOD - Generic, safe message
throw new DatabaseError('Database connection failed');

// ❌ BAD - Reveals system information
res.status(500).json({
  error: 'Query failed: table users does not exist',
  query: 'SELECT * FROM users WHERE id = 1'
});

// ✅ GOOD - Safe, user-friendly message
res.status(500).json({
  error: 'An error occurred while processing your request'
});

```

**Security Principles:**

- Never return HTTP 500 errors with detailed stack traces in production
- Sanitize error messages to remove database schema information
- Don't expose API keys, credentials, or internal URLs
- Log detailed errors server-side, return generic messages to clients
- Use different error messages for production vs. development

---

## Configuration Management

### Types of Configuration

**1. Application Settings**

- Log level (debug, info, warn, error)
- Server port
- Connection pool size
- Request timeout values
- Rate limiting thresholds

**2. Database Configuration**

- Host and port
- Database name
- Username and password
- Connection pool settings
- Query timeouts
- SSL/TLS settings

**3. External Services**

- API keys and secrets (OpenAI, Stripe, Razorpay)
- Service URLs and endpoints
- Timeout values
- Retry configurations

**4. Feature Flags**

- Dynamic enable/disable of features
- A/B testing configurations
- Gradual rollout percentages

**5. Observability Configuration**

- Monitoring service credentials
- Log aggregation settings
- Metrics collection intervals

**6. Security Settings**

- JWT secret and expiration
- Password hashing rounds
- CORS origins
- Rate limiting rules

**7. Performance Tuning**

- CPU and memory limits
- Worker thread counts
- Cache sizes and TTLs

### Configuration Storage

**Environment Variables (.env file):**

```bash
# Application
NODE_ENV=production
PORT=3000
LOG_LEVEL=info

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
DB_POOL_SIZE=20
DB_TIMEOUT=5000

# External Services
OPENAI_API_KEY=sk-xxx
STRIPE_SECRET_KEY=sk_live_xxx
RAZORPAY_KEY_ID=rzp_live_xxx
RAZORPAY_KEY_SECRET=xxx

# Security
JWT_SECRET=your-secret-key-min-32-characters
JWT_EXPIRES_IN=7d

# Feature Flags
ENABLE_RECOMMENDATIONS=true
ENABLE_ANALYTICS=false

```

### Configuration Validation

**Comprehensive validation at startup:**

```jsx
const { z } = require('zod');

const configSchema = z.object({
  // Application
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.string().regex(/^\d+$/).transform(Number),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']),

  // Database
  DATABASE_URL: z.string().url().startsWith('postgresql://'),
  DB_POOL_SIZE: z.string().regex(/^\d+$/).transform(Number).default('10'),
  DB_TIMEOUT: z.string().regex(/^\d+$/).transform(Number).default('5000'),

  // External Services
  OPENAI_API_KEY: z.string().min(1),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),

  // Security
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  JWT_EXPIRES_IN: z.string().default('7d'),

  // Feature Flags
  ENABLE_RECOMMENDATIONS: z.string().transform(val => val === 'true').default('true'),
});

function loadConfig() {
  try {
    require('dotenv').config();
    const config = configSchema.parse(process.env);
    return config;
  } catch (error) {
    console.error('Configuration validation failed:');
    if (error instanceof z.ZodError) {
      error.errors.forEach(err => {
        console.error(`  ${err.path.join('.')}: ${err.message}`);
      });
    }
    process.exit(1);
  }
}

// Load and validate config before starting
const config = loadConfig();

```

---

## Graceful Shutdown

Graceful shutdown ensures that in-flight requests complete and resources are properly released when the application stops.

### Process Lifecycle Management

**Understanding Signals:**

- **SIGTERM**: Graceful shutdown request (default from `kill` command)
- **SIGINT**: Interrupt signal (Ctrl+C)
- **SIGKILL**: Forceful termination (cannot be caught)

### Implementing Graceful Shutdown

```jsx
class GracefulShutdown {
  constructor(server) {
    this.server = server;
    this.isShuttingDown = false;
    this.connections = new Set();

    // Track active connections
    server.on('connection', (conn) => {
      this.connections.add(conn);
      conn.on('close', () => this.connections.delete(conn));
    });
  }

  async shutdown(signal) {
    if (this.isShuttingDown) {
      console.log('Shutdown already in progress');
      return;
    }

    this.isShuttingDown = true;
    console.log(`Received ${signal}, starting graceful shutdown...`);

    // 1. Stop accepting new connections
    this.server.close(() => {
      console.log('Server closed to new connections');
    });

    // 2. Set a deadline for existing requests
    const shutdownTimeout = setTimeout(() => {
      console.error('Shutdown timeout exceeded, forcing exit');
      this.connections.forEach(conn => conn.destroy());
      process.exit(1);
    }, 30000); // 30 second deadline

    // 3. Connection draining - wait for existing requests
    console.log(`Waiting for ${this.connections.size} active connections...`);
    await this.waitForConnections();

    // 4. Cleanup resources in reverse order of acquisition
    await this.cleanupResources();

    clearTimeout(shutdownTimeout);
    console.log('Graceful shutdown complete');
    process.exit(0);
  }

  async waitForConnections() {
    return new Promise((resolve) => {
      const check = setInterval(() => {
        if (this.connections.size === 0) {
          clearInterval(check);
          console.log('All connections closed');
          resolve();
        } else {
          console.log(`${this.connections.size} connections remaining...`);
        }
      }, 1000);
    });
  }

  async cleanupResources() {
    console.log('Cleaning up resources...');

    try {
      // Close database connections
      if (global.dbPool) {
        console.log('Closing database pool...');
        await global.dbPool.end();
      }

      // Close Redis connections
      if (global.redisClient) {
        console.log('Closing Redis connection...');
        await global.redisClient.quit();
      }

      // Close message queues
      if (global.messageQueue) {
        console.log('Closing message queue...');
        await global.messageQueue.close();
      }

      // Close file handles
      // Close network connections
      // Any other cleanup needed

      console.log('Resource cleanup complete');
    } catch (error) {
      console.error('Error during cleanup:', error);
      // Continue with shutdown even if cleanup fails
    }
  }
}

// Setup graceful shutdown
const gracefulShutdown = new GracefulShutdown(server);

process.on('SIGTERM', () => gracefulShutdown.shutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown.shutdown('SIGINT'));

// Handle uncaught errors
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
  gracefulShutdown.shutdown('uncaughtException');
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection at:', promise, 'reason:', reason);
  gracefulShutdown.shutdown('unhandledRejection');
});

```

### Shutdown Checklist

1. **Stop accepting new requests** - Server stops listening for new connections
2. **Connection draining** - Allow 30-60 seconds for existing requests to complete
3. **Resource cleanup** - Close resources in reverse order of acquisition:
    - Message queue connections
    - Redis/cache connections
    - Database connection pool
    - File handles
    - Network sockets
4. **Exit process** - Exit with appropriate code (0 for success, 1 for error)

---

## Best Practices Summary

**Error Handling:**

- Classify errors by type and handle them appropriately
- Use custom error classes for different scenarios
- Implement retry logic with exponential backoff for transient failures
- Use circuit breakers to prevent cascading failures
- Never expose internal details in error messages
- Log detailed errors server-side, return generic messages to clients

**Prevention:**

- Implement comprehensive health checks
- Monitor database query performance
- Validate configuration at startup (fail-fast)
- Use feature flags for graceful degradation
- Test edge cases and failure scenarios

**Configuration:**

- Store configuration in environment variables
- Validate all configuration at startup
- Use schema validation libraries (zod, joi)
- Never commit secrets to version control
- Document all configuration options

**Graceful Shutdown:**

- Register signal handlers for SIGTERM and SIGINT
- Stop accepting new connections immediately
- Allow existing requests to complete (with timeout)
- Clean up resources in reverse order
- Log shutdown progress for debugging

Remember: The goal is not to prevent all errors, but to handle them gracefully while maintaining system stability and security.
