---
name: routing
description: "Backend routing: HTTP methods and intent, path, dynamic and query parameters, nested routes, API versioning strategies (URL path, header, query parameter), catch-all and 404 routes, wildcard matching, framework-specific examples, and best practices. Use when defining routes, adding API versioning, or organizing a router."
---

# Backend Routing Documentation

## Overview

Routing in backend applications defines how the server responds to client requests at specific endpoints. It combines HTTP methods with URL patterns to direct requests to the appropriate handlers.

## HTTP Methods

HTTP methods describe the intent of a request:

- **GET** - Retrieve data from the server
- **POST** - Create new resources
- **PUT** - Update/replace existing resources
- **PATCH** - Partially update existing resources
- **DELETE** - Remove resources
- **OPTIONS** - Describe communication options
- **HEAD** - Retrieve headers only (no body)

## Routing Fundamentals

**Routing = HTTP Method + URL Pattern**

The combination determines where and how to process the request.

```
GET /users → Retrieve all users
POST /users → Create a new user
GET /users/123 → Retrieve user with ID 123
DELETE /users/123 → Delete user with ID 123

```

## URL Parameters

### Path Parameters

Path parameters are part of the URL path itself and contain no variables or query syntax. They represent required values embedded in the route.

```
/users/123
/products/laptop
/categories/electronics/items/5

```

**Example Implementation:**

```jsx
// Express.js
app.get('/users/:id', (req, res) => {
  const userId = req.params.id; // "123"
});

// FastAPI (Python)
@app.get("/users/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id}

```

### Dynamic Parameters

Dynamic parameters use placeholders in the route definition that get replaced with actual values at runtime.

```
Route: /users/:userId/posts/:postId
Actual: /users/42/posts/789

userId = 42
postId = 789

```

**Multiple dynamic parameters:**

```jsx
app.get('/api/:version/users/:userId/orders/:orderId', (req, res) => {
  const { version, userId, orderId } = req.params;
});

```

### Query Parameters

Query parameters are optional variables appended to the URL after a `?` symbol, separated by `&`.

```
/users?page=2&limit=10&sort=name
/search?q=laptop&category=electronics&minPrice=500

```

**Example Implementation:**

```jsx
// Express.js
app.get('/users', (req, res) => {
  const page = req.query.page || 1;
  const limit = req.query.limit || 20;
  const sort = req.query.sort;
});

// FastAPI (Python)
@app.get("/users")
def get_users(page: int = 1, limit: int = 20, sort: str = None):
    return {"page": page, "limit": limit}

```

**Key Differences:**

| Feature | Path Parameters | Query Parameters |
| --- | --- | --- |
| Location | In the URL path | After `?` in URL |
| Required | Usually required | Optional |
| Variables | No `?` or `=` syntax | Use `key=value` pairs |
| Example | `/users/123` | `/users?id=123` |

## Nested Routes

Nested routes represent hierarchical relationships between resources.

```
/users/:userId/posts
/users/:userId/posts/:postId
/users/:userId/posts/:postId/comments
/organizations/:orgId/teams/:teamId/members

```

**Example Implementation:**

```jsx
// Express.js with Router
const userRouter = express.Router();

userRouter.get('/:userId/posts', getPosts);
userRouter.post('/:userId/posts', createPost);
userRouter.get('/:userId/posts/:postId', getPost);
userRouter.delete('/:userId/posts/:postId', deletePost);

app.use('/users', userRouter);

```

## Route Versioning

Route versioning allows multiple API versions to coexist, enabling backward compatibility.

### URL Path Versioning

```
/v1/users
/v2/users
/v1/products
/v2/products

```

**Implementation:**

```jsx
// Express.js
app.use('/v1', v1Routes);
app.use('/v2', v2Routes);

// Or with dynamic versioning
app.get('/api/:version/users', (req, res) => {
  const version = req.params.version;
  if (version === 'v1') {
    // Handle v1 logic
  } else if (version === 'v2') {
    // Handle v2 logic
  }
});

```

### Header Versioning

```
GET /users
Header: Accept: application/vnd.api.v1+json

```

### Query Parameter Versioning

```
/users?version=1
/users?api-version=2

```

## Catch-All Routes

Catch-all routes handle requests to non-existent endpoints, typically returning 404 errors or redirecting users.

### Basic 404 Handler

```jsx
// Express.js - Must be defined AFTER all other routes
app.use('*', (req, res) => {
  res.status(404).json({
    error: 'Route not found',
    path: req.originalUrl
  });
});

```

### Wildcard Pattern Matching

```jsx
// Catch all routes under /api
app.get('/api/*', (req, res) => {
  res.status(404).json({ error: 'API endpoint not found' });
});

// Catch specific patterns
app.get('/files/*', (req, res) => {
  const filepath = req.params[0];
  // Serve file or return 404
});

```

### Framework-Specific Examples

**Express.js:**

```jsx
// Must be last
app.use((req, res, next) => {
  res.status(404).json({
    status: 'error',
    message: 'Resource not found'
  });
});

```

**FastAPI (Python):**

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

@app.exception_handler(404)
async def not_found_handler(request: Request, exc):
    return JSONResponse(
        status_code=404,
        content={"error": "Route not found", "path": str(request.url)}
    )

```

**Django:**

```python
# urls.py
from django.urls import path, re_path
from . import views

urlpatterns = [
    # Your routes here
    re_path(r'^.*$', views.not_found),  # Catch all
]

```

## Best Practices

1. **Order matters** - Define specific routes before generic ones, and catch-all routes last
2. **Use meaningful names** - Path parameters should clearly indicate what they represent
3. **Keep it RESTful** - Follow REST conventions for resource naming
4. **Version early** - Implement versioning from the start if the API is public
5. **Validate parameters** - Always validate and sanitize path and query parameters
6. **Document thoroughly** - Maintain clear API documentation for all routes
7. **Use consistent patterns** - Maintain consistency across your routing structure

## Complete Example

```jsx
const express = require('express');
const app = express();

// Version 1 routes
const v1Router = express.Router();

v1Router.get('/users', getAllUsers);
v1Router.post('/users', createUser);
v1Router.get('/users/:userId', getUser);
v1Router.put('/users/:userId', updateUser);
v1Router.delete('/users/:userId', deleteUser);

// Nested routes
v1Router.get('/users/:userId/posts', getUserPosts);
v1Router.get('/users/:userId/posts/:postId', getUserPost);

app.use('/api/v1', v1Router);

// Version 2 routes (if needed)
const v2Router = express.Router();
v2Router.get('/users', getAllUsersV2);
app.use('/api/v2', v2Router);

// Catch-all for undefined routes (must be last)
app.use('*', (req, res) => {
  res.status(404).json({
    error: 'Not Found',
    message: `Route ${req.originalUrl} does not exist`,
    availableVersions: ['/api/v1', '/api/v2']
  });
});

app.listen(3000);

```

# Backend Routing Summary

## Core Concept

Routing = HTTP Method + URL Pattern. This combination determines where and how to process requests.

## HTTP Methods & Intent

- GET: Retrieve data
- POST: Create resources
- PUT: Replace resources
- PATCH: Partial update
- DELETE: Remove resources

## URL Parameters

### Path Parameters (No Variables)

Direct values embedded in the URL path without query syntax.

```
/users/123
/products/laptop/reviews/5

```

### Dynamic Parameters

Placeholders in route definitions replaced at runtime.

```
Route: /users/:userId/posts/:postId
Actual: /users/42/posts/789

```

### Query Parameters (Variables)

Optional key-value pairs after `?` using `=` syntax.

```
/users?page=2&limit=10&sort=name

```

**Key Difference**: Path parameters are part of the URL structure (no `?` or `=`), query parameters use variable syntax (`key=value`).

## Nested Routes

Hierarchical resource relationships.

```
/users/:userId/posts/:postId/comments
/organizations/:orgId/teams/:teamId/members

```

## Route Versioning

Multiple API versions coexisting.

```
/v1/users
/v2/users
/api/:version/users

```

## Catch-All Routes

Handle non-existent endpoints, must be defined LAST.

```jsx
app.use('*', (req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

```

## Implementation Pattern

```jsx
// Specific routes first
GET /users → all users
GET /users/:id → specific user
POST /users → create user

// Nested routes
GET /users/:id/posts
GET /users/:id/posts/:postId

// Versioning
/api/v1/users
/api/v2/users

// Catch-all LAST
app.use('*', handle404);

```

## Best Practices

1. Order: specific → generic → catch-all
2. Path params for required identifiers
3. Query params for optional filters/pagination
4. Validate all parameters
5. Version APIs early
6. Keep RESTful conventions
