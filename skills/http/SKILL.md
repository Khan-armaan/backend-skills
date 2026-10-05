---
name: http
description: "HTTP protocol deep dive for backend engineers: DNS and request flow, HTTP/1.1 vs HTTP/2 vs HTTP/3, request message structure, headers, methods, idempotency, CORS preflight flows, status codes, HTTP caching (Cache-Control, ETag), content negotiation, compression, keep-alive, and handling large requests and responses (streaming, chunking, multipart uploads). Use when choosing status codes or headers, debugging CORS, setting cache headers, or reasoning about protocol-level behaviour."
---

# HTTP Protocol Deep Dive

A single long reference organized by protocol topic. Pick the section you need from the list below.

## How to use this skill

The full guide is in [reference.md](reference.md) (about 1302 lines). Don't read the whole file. Pick the section you need from the list below, use Grep to find its heading in `reference.md` and get the line number, then use Read with `offset`/`limit` to load just that section.

## Sections in reference.md

- HTTP Protocol & Backend Deep Dive
  - DNS Lifecycle & Request Flow
  - HTTP Protocol Fundamentals
  - HTTP Version Evolution
  - HTTP Request Message Structure
  - HTTP Headers Deep Dive
  - HTTP Methods
  - Idempotent vs Non-Idempotent Methods
  - CORS Request Flows
  - HTTP Response Status Codes
  - HTTP Caching
  - Content Negotiation
  - HTTP Compression
  - Keep-Alive & Persistent Connections
  - Handling Large Requests and Responses
