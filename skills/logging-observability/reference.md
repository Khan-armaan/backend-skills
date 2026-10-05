# Logging, Monitoring & Observability Guide

## Overview

Modern software systems require comprehensive visibility into their behavior and health. This guide covers the three pillars of system visibility: logging, monitoring, and observability.

---

## 1. Observability

**Observability** is the ability to determine the internal state of a service by examining its external outputs. It answers the question: "What is happening inside my system?"

### Three Pillars of Observability

### Logs

Detailed records of discrete events that occurred within the application, including metadata about what happened, when, and where.

### Metrics

Quantitative measurements of system behavior over time, such as request counts, error rates, and response times.

### Traces

Records showing the path a request takes through different layers and services of your application, from entry point to completion or failure.

### What Observability Tells Us

- Which layer a request is passing through
- The complete journey of a request through the system
- Where failures occur in the request lifecycle
- The internal state of services based on their external behavior
- The state of infrastructure components

**Example**: A trace might show: "Request started at API Gateway → routed to Authentication Service → processed by User Service → queried Database → failed at Payment Service with timeout error"

---

## 2. Logging

**Logging** is the practice of recording all important events and failure events with relevant metadata to provide a historical record of system behavior.

### What to Log

- **Important Events**: Successful operations, state changes, business logic execution
- **Failure Events**: Errors, exceptions, timeouts, validation failures
- **Metadata**: Timestamps, user IDs, request IDs, session information, context data

### Logging Levels

Logging levels help categorize the severity and importance of log entries:

### 1. DEBUG

- **Purpose**: Detailed information for diagnosing problems
- **Use Case**: Variable values, function entry/exit, detailed flow information
- **Environment**: Typically enabled only in development
- **Example**: `"User authentication token validated successfully for user_id: 12345"`

### 2. INFO

- **Purpose**: General informational messages about application flow
- **Use Case**: Successful operations, milestone events, configuration details
- **Environment**: Enabled in all environments
- **Example**: `"Payment processed successfully, transaction_id: TXN-98765, amount: $49.99"`

### 3. WARN

- **Purpose**: Potentially harmful situations that don't prevent operation
- **Use Case**: Deprecated API usage, fallback mechanisms triggered, retry attempts
- **Environment**: Enabled in all environments
- **Example**: `"API rate limit approaching threshold: 850/1000 requests used"`

### 4. ERROR

- **Purpose**: Error events that still allow the application to continue
- **Use Case**: Handled exceptions, failed operations, recoverable errors
- **Environment**: Enabled in all environments
- **Example**: `"Failed to send email notification, error: SMTP connection timeout, will retry"`

### 5. FATAL

- **Purpose**: Severe errors causing application termination
- **Use Case**: Unrecoverable errors, system crashes, critical resource failures
- **Environment**: Enabled in all environments
- **Example**: `"Database connection pool exhausted, application shutting down"`

### Structured vs Unstructured Logging

### Structured Logging

Logs formatted in a consistent, parseable format (typically JSON) that makes them easy to search, filter, and analyze.

**Characteristics**:

- Machine-readable format
- Consistent field names
- Easy to query and aggregate
- Ideal for production environments

**Example**:

```json
{
  "timestamp": "2026-01-10T14:32:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "request_id": "req-abc-123",
  "user_id": "usr-456",
  "message": "Payment processing failed",
  "error": {
    "type": "TimeoutException",
    "code": "GATEWAY_TIMEOUT",
    "details": "Payment gateway did not respond within 30s"
  },
  "metadata": {
    "amount": 49.99,
    "currency": "USD",
    "payment_method": "credit_card"
  }
}

```

### Unstructured Logging

Human-readable text logs without consistent formatting, typically used in development environments.

**Characteristics**:

- Human-friendly format
- Easier to read during development
- Harder to parse programmatically
- Ideal for development mode

**Example**:

```
2026-01-10 14:32:45 ERROR [payment-service] Payment processing failed for user usr-456:
TimeoutException - Payment gateway did not respond within 30s (amount: $49.99, method: credit_card)

```

---

## 3. Monitoring

**Monitoring** is the real-time checking of the health, performance, and behavior of your application and infrastructure.

### What Monitoring Tracks

- **System Health**: CPU usage, memory consumption, disk I/O, network traffic
- **Application Performance**: Response times, throughput, latency percentiles
- **Application Behavior**: Error rates, request patterns, user activity
- **Business Metrics**: Transactions completed, revenue processed, user signups

### Key Metrics to Monitor

### Request Metrics

- Total request count
- Requests per second (throughput)
- Failed requests (status codes > 299)
- Success rate percentage

**Example**: "Monitor how many requests failed with status > 200 in the last hour"

### Performance Metrics

- Average response time
- P50, P95, P99 latency percentiles
- Database query duration
- External API call times

### Resource Metrics

- CPU utilization percentage
- Memory usage and available memory
- Disk space and I/O operations
- Network bandwidth usage

### Implementing Monitoring with Middleware

Middleware provides a centralized location to capture metrics and monitoring data for all requests passing through your application.

**Example Monitoring Middleware (Node.js/Express)**:

```jsx
const monitoringMiddleware = (req, res, next) => {
  const startTime = Date.now();

  // Capture response to log metrics
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    const statusCode = res.statusCode;

    // Log structured monitoring data
    logger.info({
      type: 'request_completed',
      method: req.method,
      path: req.path,
      status: statusCode,
      duration_ms: duration,
      user_id: req.user?.id,
      request_id: req.id
    });

    // Send metrics to monitoring system
    metrics.recordRequestDuration(req.path, duration);
    metrics.incrementRequestCount(req.path, statusCode);

    if (statusCode >= 400) {
      metrics.incrementErrorCount(req.path, statusCode);
    }
  });

  next();
};

```

---

## 4. Advanced Observability Concepts

### Instrumentation

**Instrumentation** is the practice of measuring different metrics and attributes of functions, services, and systems to provide visibility into their behavior.

**What to Instrument**:

- Function execution time
- Database query performance
- External API call duration
- Cache hit/miss rates
- Queue processing times
- Business operation metrics

**Example**:

```jsx
function processPayment(paymentData) {
  const span = tracer.startSpan('process_payment');
  span.setAttribute('payment.amount', paymentData.amount);
  span.setAttribute('payment.method', paymentData.method);

  try {
    const result = executePayment(paymentData);
    span.setAttribute('payment.status', 'success');
    return result;
  } catch (error) {
    span.setAttribute('payment.status', 'failed');
    span.recordException(error);
    throw error;
  } finally {
    span.end();
  }
}

```

### OpenTelemetry

**OpenTelemetry** is an open-source observability framework that provides standardized tools to properly instrument applications for logs, metrics, and traces.

**Benefits**:

- Vendor-neutral instrumentation
- Standardized data formats
- Automatic instrumentation for common frameworks
- Unified API for all observability signals
- Support for distributed tracing

**Key Components**:

- **SDK**: Libraries for instrumenting your code
- **API**: Interfaces for creating spans, metrics, and logs
- **Collectors**: Services that receive, process, and export telemetry data
- **Exporters**: Send data to observability platforms (Prometheus, Jaeger, etc.)

### Adding Errors to Traces

Traces should capture not just the path of successful requests, but also where and why failures occur.

**Example Trace with Error**:

```
Request Flow:
├─ API Gateway (200ms) ✓
├─ Authentication Service (50ms) ✓
├─ User Service (100ms) ✓
├─ Order Service (150ms) ✓
└─ Payment Service (timeout after 30s) ✗
   └─ Error: Gateway timeout - no response from payment provider

```

This trace clearly shows:

- Request started at API Gateway
- Traveled through Authentication → User → Order services successfully
- Failed at Payment Service with a timeout error
- Exact point of failure and reason

---

## 5. State Monitoring

### Service State

Monitoring the state of your services includes:

- Health status (healthy, degraded, unhealthy)
- Deployment version
- Configuration state
- Active connections
- Request queue depth
- Circuit breaker status

### Infrastructure State

Monitoring the state of your infrastructure includes:

- Server/container health
- Network connectivity
- Storage capacity and performance
- Load balancer status
- Database replication lag
- Cache availability

---

## Best Practices

### For Logging

1. Use structured logging in production
2. Include correlation IDs (request IDs) in all logs
3. Log at appropriate levels to avoid noise
4. Never log sensitive data (passwords, tokens, PII)
5. Use consistent timestamp formats (ISO 8601)

### For Monitoring

1. Implement monitoring middleware early
2. Set up alerts for critical metrics
3. Monitor both technical and business metrics
4. Use dashboards for real-time visibility
5. Establish baseline metrics for comparison

### For Observability

1. Instrument all critical paths in your application
2. Use OpenTelemetry for standardized instrumentation
3. Implement distributed tracing across services
4. Correlate logs, metrics, and traces using common identifiers
5. Add context and metadata to all observability signals

---

## Summary

**Logging** provides detailed records of what happened in your system.

**Monitoring** gives you real-time insights into system health and performance.

**Observability** enables you to understand the internal state of your system by examining its external outputs through logs, metrics, and traces.

Together, these three pillars provide comprehensive visibility into your application's behavior, performance, and health, enabling you to quickly identify, diagnose, and resolve issues while ensuring optimal system operation.

# LLM Reference Guide: Logging, Monitoring & Observability

## Quick Reference for AI Implementation

This document provides a concise reference for implementing logging, monitoring, and observability in applications.

---

## Core Concepts

### 1. Observability

**Definition**: Ability to determine internal state of a service by examining external outputs.

**Three Pillars**:

- **Logs**: Discrete event records with metadata
- **Metrics**: Quantitative measurements over time
- **Traces**: Request path through system layers

**Purpose**: Understand what's happening inside the system by analyzing logs, metrics, and traces.

---

### 2. Logging

**Definition**: Recording important events and failures with metadata.

**Log Components**:

- Event description
- Timestamp (ISO 8601 format)
- Severity level
- Context metadata (user_id, request_id, session_id)
- Error details (if applicable)

---

### 3. Monitoring

**Definition**: Real-time checking of system health, performance, and behavior.

**Key Areas**:

- Health status
- Performance metrics
- Resource utilization
- Error rates
- Request throughput

---

## Logging Levels (Use in Order)

| Level | Severity | Use Case | Example |
| --- | --- | --- | --- |
| **DEBUG** | Lowest | Diagnostic details, variable values | `"Validating user token: eyJhbGc..."` |
| **INFO** | Low | Normal operations, milestones | `"User logged in successfully, user_id: 123"` |
| **WARN** | Medium | Potential issues, non-critical problems | `"API rate limit at 90%, user_id: 456"` |
| **ERROR** | High | Operation failures, handled exceptions | `"Payment failed: timeout, will retry"` |
| **FATAL** | Critical | System crashes, unrecoverable errors | `"Database connection lost, shutting down"` |

---

## Structured vs Unstructured Logging

### Structured Logging (Production)

**Format**: JSON
**Purpose**: Machine-readable, queryable, aggregatable

```json
{
  "timestamp": "2026-01-10T14:32:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "request_id": "req-abc-123",
  "user_id": "usr-456",
  "message": "Payment processing failed",
  "error": {
    "type": "TimeoutException",
    "code": "GATEWAY_TIMEOUT",
    "message": "Payment gateway timeout after 30s"
  },
  "metadata": {
    "amount": 49.99,
    "currency": "USD",
    "payment_method": "credit_card"
  }
}

```

### Unstructured Logging (Development)

**Format**: Plain text
**Purpose**: Human-readable, quick debugging

```
2026-01-10 14:32:45 ERROR [payment-service] Payment failed for user usr-456:
TimeoutException - Gateway timeout after 30s (amount: $49.99)

```

---

## Monitoring Implementation

### Use Middleware Pattern

**Purpose**: Centralized request/response tracking

**What to Track**:

- Request method, path, headers
- Response status code
- Response time/duration
- User/session identification
- Error occurrences

### Example Middleware Template

```jsx
function monitoringMiddleware(req, res, next) {
  const startTime = Date.now();
  const requestId = generateRequestId();
  req.requestId = requestId;

  res.on('finish', () => {
    const duration = Date.now() - startTime;
    const logData = {
      timestamp: new Date().toISOString(),
      level: res.statusCode >= 400 ? 'ERROR' : 'INFO',
      request_id: requestId,
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration_ms: duration,
      user_id: req.user?.id || 'anonymous'
    };

    // Log structured data
    logger.log(logData);

    // Record metrics
    metrics.recordRequest(req.path, res.statusCode, duration);

    // Alert on errors
    if (res.statusCode >= 500) {
      alerts.notify('server_error', logData);
    }
  });

  next();
}

```

---

## Metrics to Track

### Request Metrics

- Total request count
- Requests per second (RPS)
- Failed requests (status >= 400)
- Success rate percentage
- **Key Rule**: Monitor requests with status > 200 for failures

### Performance Metrics

- Average response time
- P50, P95, P99 latency percentiles
- Throughput (requests/second)
- Error rate percentage

### System Metrics

- CPU utilization %
- Memory usage (used/total)
- Disk I/O operations
- Network bandwidth
- Active connections

---

## Distributed Tracing

### Trace Components

**Span**: Single operation unit
**Trace**: Collection of spans showing complete request flow

### Example Trace Flow

```
Trace ID: trace-12345
├─ [Span 1] API Gateway (50ms) ✓
│  └─ span_id: span-001, status: OK
├─ [Span 2] Auth Service (30ms) ✓
│  └─ span_id: span-002, status: OK
├─ [Span 3] User Service (100ms) ✓
│  └─ span_id: span-003, status: OK
├─ [Span 4] Database Query (200ms) ✓
│  └─ span_id: span-004, status: OK
└─ [Span 5] Payment Service (30000ms) ✗
   └─ span_id: span-005, status: ERROR
   └─ error: "Gateway timeout - no response after 30s"

```

**Analysis**: Request started at API Gateway → Auth Service → User Service → Database Query → Failed at Payment Service with timeout

---

## Instrumentation

### Definition

Practice of measuring metrics and attributes of functions/services.

### What to Instrument

1. **Function execution time**: How long operations take
2. **External API calls**: Duration, success/failure rates
3. **Database queries**: Query time, row counts
4. **Cache operations**: Hit/miss rates
5. **Queue processing**: Message processing time
6. **Business operations**: Transactions, signups, payments

### Instrumentation Template

```jsx
function instrumentedFunction(params) {
  const span = tracer.startSpan('function_name');
  span.setAttributes({
    'param.value': params.value,
    'user.id': params.userId
  });

  const startTime = Date.now();

  try {
    const result = executeOperation(params);

    // Record success metrics
    span.setStatus({ code: 'OK' });
    span.setAttribute('result.status', 'success');
    metrics.recordSuccess('function_name');

    return result;
  } catch (error) {
    // Record failure metrics
    span.setStatus({ code: 'ERROR', message: error.message });
    span.recordException(error);
    metrics.recordFailure('function_name', error.type);

    logger.error({
      function: 'function_name',
      error: error.message,
      stack: error.stack,
      params: params
    });

    throw error;
  } finally {
    const duration = Date.now() - startTime;
    span.setAttribute('duration_ms', duration);
    span.end();

    metrics.recordDuration('function_name', duration);
  }
}

```

---

## OpenTelemetry

### What It Provides

- Standardized instrumentation APIs
- Vendor-neutral data collection
- Automatic instrumentation for frameworks
- Unified approach to logs, metrics, traces

### Core Components

1. **SDK**: Libraries for code instrumentation
2. **API**: Interfaces for creating observability signals
3. **Collectors**: Receive, process, export telemetry
4. **Exporters**: Send data to platforms (Prometheus, Jaeger, Datadog, etc.)

### Basic Setup Pattern

```jsx
// Initialize OpenTelemetry
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'my-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0'
  }),
  traceExporter: new JaegerExporter(),
  metricReader: new PrometheusExporter()
});

sdk.start();

```

---

## Error Handling in Traces

### Adding Errors to Spans

```jsx
try {
  // Operation
} catch (error) {
  span.recordException(error);
  span.setStatus({
    code: SpanStatusCode.ERROR,
    message: error.message
  });
  span.setAttribute('error.type', error.constructor.name);
  span.setAttribute('error.handled', true);
}

```

### Error Context to Include

- Error type/class
- Error message
- Stack trace
- Operation being performed
- Input parameters
- User/session context
- Timestamp

---

## State Monitoring

### Service State

Monitor and expose:

- Health status: `healthy`, `degraded`, `unhealthy`
- Readiness: Can accept traffic
- Liveness: Is running
- Startup: Has finished initialization
- Version/build information
- Configuration state
- Dependencies status

### Infrastructure State

Monitor:

- Container/pod health
- Node resource availability
- Network connectivity
- Storage capacity
- Database connection pool status
- Cache cluster status
- Message queue depth

---

## Implementation Checklist

### Logging

- [ ]  Implement structured logging in production
- [ ]  Use appropriate log levels
- [ ]  Include request_id in all logs
- [ ]  Add user context to logs
- [ ]  Never log sensitive data (passwords, tokens, PII)
- [ ]  Use ISO 8601 timestamps
- [ ]  Implement log rotation/retention

### Monitoring

- [ ]  Add monitoring middleware to all routes
- [ ]  Track request/response metrics
- [ ]  Monitor error rates
- [ ]  Set up performance metrics
- [ ]  Create dashboards for visualization
- [ ]  Configure alerts for critical metrics
- [ ]  Monitor both system and business metrics

### Observability

- [ ]  Implement distributed tracing
- [ ]  Use OpenTelemetry for standardization
- [ ]  Instrument critical code paths
- [ ]  Add error context to traces
- [ ]  Correlate logs/metrics/traces with request_id
- [ ]  Set up trace sampling for high-volume services
- [ ]  Export telemetry to observability platform

---

## Code Patterns for LLM

### 1. Logger Instance

```jsx
const logger = require('./logger');

logger.debug('Debug message', { context: 'additional data' });
logger.info('Info message', { userId: 123 });
logger.warn('Warning message', { threshold: 90 });
logger.error('Error message', { error: err.message });
logger.fatal('Fatal message', { reason: 'DB connection lost' });

```

### 2. Middleware Pattern

```jsx
app.use((req, res, next) => {
  req.startTime = Date.now();
  req.requestId = generateId();

  res.on('finish', () => {
    logRequest(req, res);
    recordMetrics(req, res);
  });

  next();
});

```

### 3. Tracing Pattern

```jsx
const tracer = require('./tracer');

async function processRequest(data) {
  const span = tracer.startSpan('process_request');

  try {
    span.setAttribute('request.id', data.id);
    const result = await doWork(data);
    span.setStatus({ code: 'OK' });
    return result;
  } catch (error) {
    span.recordException(error);
    span.setStatus({ code: 'ERROR' });
    throw error;
  } finally {
    span.end();
  }
}

```

### 4. Metrics Recording

```jsx
// Counter
metrics.increment('requests.total');
metrics.increment('requests.failed', { status: 500 });

// Gauge
metrics.gauge('memory.usage', process.memoryUsage().heapUsed);

// Histogram
metrics.histogram('request.duration', duration);

// Summary
metrics.summary('response.time', responseTime);

```

---

## Common Patterns by Language

### Node.js/Express

- Use Winston or Pino for logging
- Use Prometheus client for metrics
- Use OpenTelemetry Node SDK
- Middleware for request tracking

### Python/Flask

- Use structlog for structured logging
- Use prometheus_client for metrics
- Use OpenTelemetry Python SDK
- Decorators for instrumentation

### Go

- Use zap or logrus for logging
- Use prometheus/client_golang
- Use OpenTelemetry Go SDK
- Middleware functions for HTTP handlers

### Java/Spring Boot

- Use Logback or Log4j2 with JSON layout
- Use Micrometer for metrics
- Use OpenTelemetry Java SDK
- Use Spring Interceptors

---

## Quick Decision Tree

**When to use each log level?**

- DEBUG → Detailed diagnostic info
- INFO → Normal operation confirmation
- WARN → Potentially harmful situation
- ERROR → Error event, app continues
- FATAL → Severe error, app terminates

**When to use structured vs unstructured?**

- Structured → Production, need to query/aggregate
- Unstructured → Development, human readability

**When to create a span?**

- Significant operation (>10ms typical)
- External service call
- Database query
- Critical business logic

**When to record a metric?**

- Every request/response
- Resource usage checks
- Business events
- Error occurrences

---

## Best Practices Summary

1. **Always** include request_id/trace_id for correlation
2. **Always** use structured logging in production
3. **Always** instrument critical paths
4. **Never** log sensitive data (PII, credentials)
5. **Never** ignore errors silently
6. **Use** consistent naming conventions
7. **Use** appropriate log levels
8. **Monitor** both success and failure cases
9. **Alert** on anomalies, not just failures
10. **Review** logs and metrics regularly

---

## Resources & Tools

### Logging Libraries

- Node.js: Winston, Pino, Bunyan
- Python: structlog, python-json-logger
- Go: zap, logrus
- Java: Logback, Log4j2

### Metrics Libraries

- Prometheus client libraries
- StatsD
- Micrometer (Java)

### Tracing Tools

- OpenTelemetry
- Jaeger
- Zipkin
- AWS X-Ray

### Observability Platforms

- Datadog
- New Relic
- Grafana Stack (Loki, Tempo, Prometheus)
- Elastic Stack (ELK)
- Honeycomb

---

## End of Reference Guide

Use this document as a quick reference when implementing logging, monitoring, and observability features in applications.
