# HTTP Protocol & Backend Deep Dive

## DNS Lifecycle & Request Flow

### DNS Resolution Process

When you type a URL into your browser, the following DNS lifecycle occurs:

1. **Browser Cache Check**: Browser checks its own DNS cache for the IP address
2. **OS Cache Check**: If not found, checks the operating system's DNS cache
3. **Router Cache Check**: Queries the local router's DNS cache
4. **ISP DNS Server**: Requests forwarded to ISP's recursive DNS resolver
5. **Root Name Server**: If not cached, ISP queries root name servers (13 clusters worldwide)
6. **TLD Name Server**: Root server directs to Top-Level Domain server (.com, .org, etc.)
7. **Authoritative Name Server**: TLD server points to authoritative name server for the specific domain
8. **IP Address Return**: Authoritative server returns the IP address
9. **Cache & Connect**: IP is cached at multiple levels, then TCP connection established

**Example Flow:**

```java
User enters: www.example.com
→ Browser cache (miss)
→ OS cache (miss)
→ ISP DNS resolver
→ Root server → .com TLD server
→ example.com authoritative server
→ Returns: 93.184.216.34
→ Browser connects to 93.184.216.34:80
```

---

## HTTP Protocol Fundamentals

### Core Characteristics

### 1. Statelessness

HTTP is stateless, meaning each request-response cycle is independent. The server doesn't retain information about previous requests from the same client.

**Implications:**

- No built-in memory of user sessions
- Requires mechanisms like cookies, tokens, or sessions for state management
- Simplifies server design and improves scalability
- Each request must contain all necessary information

**Example:**

```java
Request 1: GET /login → Server processes, sends response
Request 2: GET /profile → Server has no memory of Request 1
```

**State Management Solutions:**

- Cookies (stored client-side, sent with each request)
- Session IDs (server maintains session data, client sends ID)
- JWT tokens (self-contained authentication)
- URL parameters or hidden form fields

### 2. Client-Server Model

Clear separation between client (requester) and server (provider):

- **Client**: Initiates requests, renders responses (browsers, mobile apps, APIs)
- **Server**: Listens for requests, processes them, sends responses
- **Benefits**: Separation of concerns, independent evolution, multiple clients can use same server

### 3. Built on TCP

HTTP runs on top of TCP (Transmission Control Protocol):

- **Connection-oriented**: Three-way handshake establishes connection
- **Reliable**: Guarantees packet delivery and order
- **Flow control**: Manages data transmission rate
- **Error checking**: Detects and retransmits corrupted packets

**TCP Three-Way Handshake:**

```java
Client → SYN → Server
Client ← SYN-ACK ← Server
Client → ACK → Server
Connection established
```

### 4. Application Layer Protocol

HTTP operates at Layer 7 (Application Layer) of the OSI model:

- Sits above TCP (Layer 4) and IP (Layer 3)
- Defines how applications communicate
- Human-readable message format
- Independent of underlying network infrastructure

---

## HTTP Version Evolution

### HTTP/0.9 (1991)

**Characteristics:**

- Extremely simple, one-line protocol
- Only GET method supported
- No headers, no status codes
- Only HTML responses
- Connection closed after each request

**Example:**

```java
GET /index.html
```

**Response:**

html

```java
<html>
  <body>Hello World</body>
</html>
```

### HTTP/1.0 (1996)

**Major Additions:**

- HTTP headers introduced
- Status codes added (200, 404, 500, etc.)
- Multiple content types (not just HTML)
- POST and HEAD methods added
- Version information in requests

**Example:**

```java
GET /page.html HTTP/1.0
User-Agent: Mozilla/5.0
Accept: text/html

HTTP/1.0 200 OK
Content-Type: text/html
Content-Length: 137

<html>...</html>
```

**Limitations:**

- New TCP connection for each request
- No persistent connections by default
- Poor performance for multiple resources

### HTTP/1.1 (1997)

**Major Improvements:**

1. **Persistent Connections**: Default keep-alive, reuse TCP connections
2. **Pipelining**: Send multiple requests without waiting for responses
3. **Chunked Transfer Encoding**: Stream responses of unknown length
4. **Host Header**: Mandatory, enables virtual hosting
5. **Additional Methods**: PUT, DELETE, OPTIONS, TRACE, CONNECT
6. **Better Caching**: Cache-Control headers
7. **Range Requests**: Download partial content

**Example:**

```java
GET /index.html HTTP/1.1
Host: www.example.com
Connection: keep-alive
Accept-Encoding: gzip, deflate

HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234
Connection: keep-alive
Cache-Control: max-age=3600

<html>...</html>
```

**Limitations:**

- Head-of-line blocking (requests processed sequentially)
- Header overhead (repeated headers)
- Limited parallelism (browsers open 6-8 connections per domain)

### HTTP/2 (2015)

**Revolutionary Changes:**

1. **Binary Protocol**: Binary framing instead of text
2. **Multiplexing**: Multiple requests/responses over single connection
3. **Header Compression**: HPACK algorithm reduces overhead
4. **Server Push**: Server can send resources proactively
5. **Stream Prioritization**: Critical resources loaded first
6. **Single Connection**: One TCP connection per origin

**Binary Framing Layer:**

```java
HTTP/1.1:
GET /index.html HTTP/1.1\r\n
Host: example.com\r\n\r\n

HTTP/2:
[HEADERS Frame]
  Stream ID: 1
  :method: GET
  :path: /index.html
  :scheme: https
  :authority: example.com
```

**Multiplexing Example:**

```java
Single TCP Connection:
Stream 1: GET /style.css
Stream 3: GET /script.js
Stream 5: GET /image.png
(All simultaneous, no blocking)
```

**Server Push:**

```java
Client requests: GET /index.html
Server responds:
  - index.html (requested)
  - style.css (pushed)
  - script.js (pushed)
```

**Benefits:**

- Eliminates head-of-line blocking at HTTP level
- Reduces latency significantly
- Better use of single connection
- Reduced overhead from header compression

**Limitations:**

- TCP head-of-line blocking still exists
- Complex implementation
- Difficult to debug (binary format)

### HTTP/3 (2022)

**Fundamental Change: QUIC Protocol**

Built on UDP instead of TCP, using QUIC (Quick UDP Internet Connections):

**Key Features:**

1. **Built on UDP**: Eliminates TCP handshake overhead
2. **Integrated TLS 1.3**: Security built into transport layer
3. **No Head-of-Line Blocking**: Independent streams
4. **Connection Migration**: Survives network changes (Wi-Fi to cellular)
5. **0-RTT Resumption**: Faster reconnection for returning clients
6. **Improved Congestion Control**: Better packet loss handling

**Connection Establishment:**

```java
HTTP/2 (TCP + TLS):
TCP handshake (1 RTT)
+ TLS handshake (1-2 RTT)
= 2-3 RTT total

HTTP/3 (QUIC):
Combined handshake (1 RTT)
0-RTT on resumption
```

**Stream Independence:**

```java
HTTP/2 over TCP:
Stream 1: [packet lost] → ALL streams blocked

HTTP/3 over QUIC:
Stream 1: [packet lost] → only Stream 1 blocked
Streams 2, 3, 4 continue normally
```

**Benefits:**

- Faster connection establishment
- Better mobile performance
- Survives IP address changes
- True stream independence

## HTTP Request Message Structure

### Complete Request Anatomy

```java
GET /api/users/123 HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: application/json
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cache-Control: no-cache
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Cookie: session_id=abc123; preferences=dark_mode
Content-Type: application/json
Content-Length: 45

{"filter": "active", "limit": 10}
```

**Structure Breakdown:**

1. **Request Line**: `METHOD PATH HTTP_VERSION`
2. **Headers**: Key-value pairs providing metadata
3. **Blank Line**: Separates headers from body
4. **Body**: Optional data payload

---

## HTTP Headers Deep Dive

### General Headers

Apply to both requests and responses:

**Cache-Control**

```java
Cache-Control: no-cache
Cache-Control: no-store
Cache-Control: max-age=3600
Cache-Control: public, max-age=86400
Cache-Control: private, must-revalidate
```

**Connection**

```java
Connection: keep-alive
Connection: close
Connection: upgrade
```

**Date**

```java
Date: Wed, 21 Oct 2025 07:28:00 GMT
```

**Transfer-Encoding**

```java
Transfer-Encoding: chunked
Transfer-Encoding: gzip, chunked
```

**Upgrade**

```java
Upgrade: websocket
Upgrade: HTTP/2.0
```

### Request Headers

**Host** (Required in HTTP/1.1)

`Host: www.example.com
Host: api.example.com:8080`

**User-Agent**

`User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
User-Agent: curl/7.68.0`

**Accept Headers**

`Accept: text/html,application/json;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.9,es;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Charset: utf-8, iso-8859-1;q=0.5`

**Authorization**

`Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Authorization: Digest username="user", realm="realm", nonce="nonce"`

**Cookie**

`Cookie: session_id=abc123; user_pref=dark_mode; auth_token=xyz789`

**Referer**

`Referer: https://www.google.com/search?q=example`

**Origin**

`Origin: https://www.example.com`

*If- Conditional Headers**

`If-Modified-Since: Wed, 21 Oct 2025 07:28:00 GMT
If-None-Match: "686897696a7c876b7e"
If-Match: "686897696a7c876b7e"
If-Unmodified-Since: Wed, 21 Oct 2025 07:28:00 GMT
If-Range: "686897696a7c876b7e"`

### Representation Headers

Describe the body content:

**Content-Type**

`Content-Type: text/html; charset=utf-8
Content-Type: application/json
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Type: application/x-www-form-urlencoded
Content-Type: image/png`

**Content-Length**

`Content-Length: 348`

**Content-Encoding**

`Content-Encoding: gzip
Content-Encoding: compress
Content-Encoding: deflate
Content-Encoding: br (Brotli)`

**Content-Language**

`Content-Language: en-US
Content-Language: es-ES, en-US`

**Content-Location**

`Content-Location: /documents/report.pdf`

### Security Headers

**Strict-Transport-Security (HSTS)**

`Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`

**Content-Security-Policy**

`Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' https://fonts.googleapis.com`

**X-Frame-Options**

`X-Frame-Options: DENY
X-Frame-Options: SAMEORIGIN
X-Frame-Options: ALLOW-FROM https://example.com`

**X-Content-Type-Options**

`X-Content-Type-Options: nosniff`

**X-XSS-Protection**

`X-XSS-Protection: 1; mode=block`

**Referrer-Policy**

`Referrer-Policy: no-referrer
Referrer-Policy: strict-origin-when-cross-origin`

**Permissions-Policy**

`Permissions-Policy: geolocation=(self), microphone=()`

## HTTP Methods

### GET

**Purpose**: Retrieve data from server

**Characteristics:**

- Safe (doesn't modify server state)
- Idempotent (multiple identical requests have same effect)
- Cacheable
- Parameters in URL query string
- No request body (generally)

**Example:**

`GET /api/users?role=admin&page=2 HTTP/1.1
Host: api.example.com
Accept: application/json`

### POST

**Purpose**: Submit data to server, create resources

**Characteristics:**

- Not safe (modifies server state)
- Not idempotent (multiple requests may create multiple resources)
- Not cacheable by default
- Data in request body
- Can trigger side effects

**Example:**

```java
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Content-Length: 78

{
  "name": "John Doe",
  "email": "john@example.com",
  "role": "admin"
}
```

### PUT

**Purpose**: Update or replace entire resource

**Characteristics:**

- Not safe
- Idempotent (multiple identical requests produce same result)
- Replaces entire resource
- Client specifies resource URI

**Example:**

```java
PUT /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "id": 123,
  "name": "John Doe Updated",
  "email": "john.doe@example.com",
  "role": "admin",
  "active": true
}
```

### PATCH

**Purpose**: Partial update of resource

**Characteristics:**

- Not safe
- Not necessarily idempotent
- Updates only specified fields
- More efficient than PUT for small changes

**Example:**

```java
PATCH /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "email": "newemail@example.com"
}
```

### DELETE

**Purpose**: Remove resource

**Characteristics:**

- Not safe
- Idempotent (deleting same resource multiple times has same effect)
- May return deleted resource or confirmation

**Example:**

```java
DELETE /api/users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer token123
```

### HEAD

**Purpose**: Retrieve headers only, no body

**Characteristics:**

- Safe
- Idempotent
- Used to check if resource exists, get metadata
- Identical to GET but without response body

**Example:**

```java
HEAD /api/large-file.zip HTTP/1.1
Host: files.example.com

Response:
HTTP/1.1 200 OK
Content-Type: application/zip
Content-Length: 104857600
Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT
```

### OPTIONS

**Purpose**: Describe communication options for resource

**Characteristics:**

- Safe
- Idempotent
- Used for CORS preflight requests
- Returns allowed methods and headers

**Example:**

```java
OPTIONS /api/users HTTP/1.1
Host: api.example.com
Origin: https://frontend.example.com

Response:
HTTP/1.1 204 No Content
Allow: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
```

### TRACE

**Purpose**: Perform message loop-back test

**Characteristics:**

- Safe
- Idempotent
- Echoes received request
- Often disabled for security

### CONNECT

**Purpose**: Establish tunnel to server

**Characteristics:**

- Used for HTTPS through proxy
- Creates TCP tunnel
- Enables SSL/TLS connections through proxy

---

## Idempotent vs Non-Idempotent Methods

### Idempotent Methods

**Definition**: Multiple identical requests have the same effect as a single request

**Idempotent Methods:**

- GET: Reading data multiple times doesn't change it
- PUT: Replacing resource with same data produces same result
- DELETE: Deleting already-deleted resource results in same state
- HEAD, OPTIONS: Only retrieve information

**Example:**

```java
DELETE /api/users/123 (first call) → Resource deleted
DELETE /api/users/123 (second call) → Resource still deleted (same state)
DELETE /api/users/123 (third call) → Resource still deleted (same state)
```

### Non-Idempotent Methods

**Definition**: Multiple identical requests may produce different effects

**Non-Idempotent Methods:**

- POST: Each request may create new resource
- PATCH: Depending on implementation, may have different effects

**Example:**

```java
POST /api/orders (first call) → Order #1 created
POST /api/orders (second call) → Order #2 created
POST /api/orders (third call) → Order #3 created
```

**Importance:**

- Retry safety: Idempotent methods safe to retry on network errors
- Caching: Only idempotent, safe methods typically cached
- API design: Influences how endpoints are structured

## CORS Request Flows

### Simple Request Flow

**Criteria for Simple Request:**

- Method: GET, HEAD, or POST
- Only simple headers (Accept, Accept-Language, Content-Language, Content-Type)
- Content-Type: application/x-www-form-urlencoded, multipart/form-data, or text/plain

**Flow:**

```java
1. Browser sends request with Origin header
   GET /api/data HTTP/1.1
   Host: api.example.com
   Origin: https://frontend.example.com

2. Server processes and responds with CORS headers
   HTTP/1.1 200 OK
   Access-Control-Allow-Origin: https://frontend.example.com
   Access-Control-Allow-Credentials: true
   Content-Type: application/json
   
   {"data": "response"}

3. Browser checks CORS headers and allows/blocks based on match
```

### Preflighted Request Flow

**Triggers Preflight:**

- Methods: PUT, DELETE, PATCH, or custom methods
- Custom headers
- Content-Type: application/json

**Flow:**

```java
1. Browser sends OPTIONS preflight request
   OPTIONS /api/users HTTP/1.1
   Host: api.example.com
   Origin: https://frontend.example.com
   Access-Control-Request-Method: DELETE
   Access-Control-Request-Headers: Authorization, Content-Type

2. Server responds to preflight
   HTTP/1.1 204 No Content
   Access-Control-Allow-Origin: https://frontend.example.com
   Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
   Access-Control-Allow-Headers: Authorization, Content-Type
   Access-Control-Max-Age: 86400
   Access-Control-Allow-Credentials: true

3. If preflight successful, browser sends actual request
   DELETE /api/users/123 HTTP/1.1
   Host: api.example.com
   Origin: https://frontend.example.com
   Authorization: Bearer token123

4. Server responds to actual request
   HTTP/1.1 200 OK
   Access-Control-Allow-Origin: https://frontend.example.com
   Content-Type: application/json
   
   {"message": "User deleted"}
```

**Preflight Caching:**

- Access-Control-Max-Age header caches preflight response
- Reduces overhead for subsequent requests
- Typical values: 3600 (1 hour) to 86400 (24 hours)

---

## HTTP Response Status Codes

### 1xx Informational

**100 Continue**

- Server received request headers
- Client should send request body

**101 Switching Protocols**

- Server switching protocols as requested (WebSocket upgrade)

**103 Early Hints**

- Hints to preload resources while server prepares response

### 2xx Success

**200 OK**

- Standard success response
- Response body contains requested data

**201 Created**

- Resource successfully created
- Location header contains URI of new resource

**202 Accepted**

- Request accepted but processing not complete
- Used for asynchronous operations

**204 No Content**

- Success but no content to return
- Common for DELETE, PUT operations

**206 Partial Content**

- Partial resource delivered (Range request)

### 3xx Redirection

**301 Moved Permanently**

- Resource permanently moved to new URI
- Update bookmarks, SEO impact

**302 Found**

- Temporary redirect
- Original URI should be used for future requests

**304 Not Modified**

- Resource not modified since last request
- Use cached version

**307 Temporary Redirect**

- Like 302 but method must not change

**308 Permanent Redirect**

- Like 301 but method must not change

### 4xx Client Errors

**400 Bad Request**

- Malformed request syntax
- Invalid request data

**401 Unauthorized**

- Authentication required
- WWW-Authenticate header indicates method

**403 Forbidden**

- Server understood request but refuses
- Authentication won't help

**404 Not Found**

- Resource doesn't exist

**405 Method Not Allowed**

- HTTP method not supported for resource
- Allow header lists valid methods

**409 Conflict**

- Request conflicts with current state
- Common in concurrent update scenarios

**413 Payload Too Large**

- Request body exceeds server limits

**415 Unsupported Media Type**

- Content-Type not supported

**422 Unprocessable Entity**

- Validation errors in request data

**429 Too Many Requests**

- Rate limit exceeded
- Retry-After header suggests wait time

### 5xx Server Errors

**500 Internal Server Error**

- Generic server error
- Something went wrong on server

**502 Bad Gateway**

- Invalid response from upstream server
- Common with proxies, load balancers

**503 Service Unavailable**

- Server temporarily overloaded or down
- Retry-After header suggests wait time

**504 Gateway Timeout**

- Upstream server didn't respond in time

## HTTP Caching

### Cache-Control Directives

**Request Directives:**

```java
Cache-Control: no-cache (force revalidation)
Cache-Control: no-store (don't cache at all)
Cache-Control: max-age=0 (must revalidate)
Cache-Control: max-stale=3600 (accept stale up to 1 hour)
Cache-Control: min-fresh=600 (must be fresh for 10 minutes)
Cache-Control: only-if-cached (only return cached)
```

**Response Directives:**

```java
Cache-Control: public (cacheable by any cache)
Cache-Control: private (only browser cache)
Cache-Control: no-cache (revalidate before use)
Cache-Control: no-store (don't cache)
Cache-Control: max-age=3600 (fresh for 1 hour)
Cache-Control: s-maxage=7200 (shared cache max age)
Cache-Control: must-revalidate (revalidate when stale)
Cache-Control: proxy-revalidate (proxies must revalidate)
Cache-Control: immutable (never changes, don't revalidate)
```

### Validation Headers

**ETag (Entity Tag)**

```java
Response:
ETag: "686897696a7c876b7e"
ETag: W/"686897696a7c876b7e" (weak validator)

Conditional Request:
If-None-Match: "686897696a7c876b7e"

Server Response:
304 Not Modified (if match)
200 OK + new content (if no match)
```

**Last-Modified**

```java
Response:
Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT

Conditional Request:
If-Modified-Since: Wed, 21 Oct 2025 07:28:00 GMT

Server Response:
304 Not Modified (if not modified)
200 OK + new content (if modified)
```

### Caching Strategies

**Aggressive Caching (Static Assets)**

`Cache-Control: public, max-age=31536000, immutable`

- Files with hash in filename (app.a3f2b1.js)
- Never change, can cache forever

**No Caching (Dynamic Content)**

`Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Expires: 0`

- User-specific data
- Frequently changing content

**Revalidation (Semi-Static Content)**

`Cache-Control: public, max-age=3600, must-revalidate
ETag: "abc123"`

- Content that changes occasionally
- Balance between freshness and performance

**Private Caching (User-Specific)**

`Cache-Control: private, max-age=600`

- User dashboards, profiles
- Only browser cache, not CDN

---

## Content Negotiation

### Accept Headers

**Content Type Negotiation:**

```java
Request:
Accept: application/json, application/xml;q=0.9, text/html;q=0.8, */*;q=0.1

Response:
Content-Type: application/json
```

- q values indicate preference (0.0-1.0)
- Server chooses best match

**Language Negotiation:**

```java
Request:
Accept-Language: en-US, en;q=0.9, es;q=0.8, fr;q=0.7

Response:
Content-Language: en-US
```

**Encoding Negotiation:**

```java
Request:
Accept-Encoding: gzip, deflate, br

Response:
Content-Encoding: br
```

**Character Set Negotiation:**

```java
Request:
Accept-Charset: utf-8, iso-8859-1;q=0.5

Response:
Content-Type: text/html; charset=utf-8
```

### Vary Header

Indicates which headers affect response variation:

```java
Vary: Accept-Encoding
Vary: Accept-Language, Accept-Encoding
Vary: User-Agent
```

- Helps caches store different versions
- Critical for proper CDN caching

---

## HTTP Compression

### Compression Algorithms

**Gzip**

`Content-Encoding: gzip`

- Most widely supported
- Good compression ratio
- Moderate CPU usage

**Deflate**

`Content-Encoding: deflate`

- Less common
- Similar to gzip

**Brotli**

`Content-Encoding: br`

- Better compression than gzip (15-25% smaller)
- Higher CPU usage
- Supported by modern browsers
- Best for static assets

### Compression Flow

```java
1. Client declares support:
   Accept-Encoding: gzip, deflate, br

2. Server compresses response:
   Content-Encoding: br
   Content-Length: 2847 (compressed size)
   
3. Browser decompresses automatically
```

### What to Compress

**Compress:**

- Text files (HTML, CSS, JavaScript)
- JSON, XML
- SVG images
- Fonts (not WOFF2, already compressed)

**Don't Compress:**

- Already compressed images (JPEG, PNG)
- Videos (MP4, WebM)
- Already compressed archives (ZIP)
- Very small files (overhead not worth it)

**Example Configuration:**

```java
Minimum size: 1KB
MIME types: 
  - text/html
  - text/css
  - application/javascript
  - application/json
  - image/svg+xml
```

---

## Keep-Alive & Persistent Connections

### HTTP/1.0 vs HTTP/1.1

**HTTP/1.0 (No Keep-Alive):**

```java
Request 1:
TCP handshake → Request → Response → Connection closed

Request 2:
TCP handshake → Request → Response → Connection closed
```

- New connection for each request
- High latency overhead

**HTTP/1.1 (Keep-Alive Default):**

```java
TCP handshake
Request 1 → Response 1
Request 2 → Response 2
Request 3 → Response 3
Connection kept open
```

### Keep-Alive Headers

```java
HTTP/1.1 200 OK
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

- **timeout**: Seconds connection stays open when idle
- **max**: Maximum requests before closing connection

### Benefits

- Reduced latency (no repeated TCP handshakes)
- Less CPU and memory usage
- Reduced network congestion
- Better performance for multiple requests

### Limitations

- Server resources held longer
- Head-of-line blocking (HTTP/1.1)
- Connection limits per domain

---

## Handling Large Requests and Responses

### Multipart Requests

Used for file uploads with form data:

```java
POST /upload HTTP/1.1
Host: example.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Length: 12345

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="username"

john_doe
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="avatar"; filename="photo.jpg"
Content-Type: image/jpeg

[binary image data]
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="description"

My profile photo
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

**Structure:**

- Boundary delimiter separates parts
- Each part has its own headers
- Supports mixed content types
- Can upload multiple files simultaneously

### Chunked Transfer Encoding

For responses of unknown length:

```java
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
7\r\n
Network\r\n
0\r\n
\r\n
```

**Format:**

- Chunk size in hexadecimal
- CRLF (\r\n)
- Chunk data
- CRLF
- Repeat until chunk size is 0

**Benefits:**

- Stream data without knowing total size
- Start sending before generating complete response
- Useful for dynamic content generation
- Live data streams

### Range Requests (Partial Content)

Download parts of large files:

```java
Request:
GET /video.mp4 HTTP/1.1
Host: example.com
Range: bytes=0-1023

Response:
HTTP/1.1 206 Partial Content
Content-Type: video/mp4
Content-Range: bytes 0-1023/10485760
Content-Length: 1024

[first 1024 bytes of video]
```

**Use Cases:**

- Resume interrupted downloads
- Video seeking/scrubbing
- Parallel downloads (
