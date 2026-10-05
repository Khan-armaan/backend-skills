---
name: layered-architecture
description: "Server request lifecycle and layered architecture: the flow from entry point through the middleware chain to handler (controller), service (business logic) and repository (data access) layers, directory structure, responsibilities of each layer, middleware ordering, and a complete example application. Use when structuring a new backend project, deciding which layer code belongs in, writing middleware, or reviewing separation of concerns."
---

# Layered Architecture: Handler, Service, Repository

Two versions of the same guide: the first is shorter with Python/Flask examples; the second ('Server Architecture: Request Lifecycle & Layered Design') goes deeper on middleware chain architecture and data flow between layers. Headings repeat across the two versions: the first Grep match is the shorter guide, the second is the in-depth one.

## How to use this skill

The full guide is in [reference.md](reference.md) (about 1337 lines). Don't read the whole file. Pick the section you need from the list below, use Grep to find its heading in `reference.md` and get the line number, then use Read with `offset`/`limit` to load just that section.

## Sections in reference.md

- Server Request Lifecycle & Architecture Guide
  - Overview
  - Request Lifecycle Flow
  - Directory Structure
  - Layer Responsibilities
  - Middleware Chain
  - Complete Example: Putting It All Together
  - Key Principles
  - Benefits
- Server Architecture: Request Lifecycle & Layered Design
  - Overview
  - Request Lifecycle Flow
  - Directory Structure
  - Layer Responsibilities
  - Middleware Chain Architecture
  - Data Flow Through Layers
  - Complete Application Example
  - Design Principles
  - Testing Strategy
  - Benefits of This Architecture
  - Common Mistakes to Avoid
  - Conclusion
