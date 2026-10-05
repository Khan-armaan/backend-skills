---
name: validation-transformation
description: "Request validation and transformation pipeline: where validation sits in middleware, types of validation (type, syntactic, semantic, business rules), query parameter handling, transformation and normalization, data integrity and security validation, middleware implementation examples, processing order and the error response format for validation failures. Use when validating request bodies, params or query strings, writing validation middleware or schemas, or sanitizing input."
---

# Validation and Transformation Pipeline

Two parts: a practical guide with middleware and query-parameter code examples, then a 'Conceptual Framework' version that explains the same pipeline in more depth. Some headings appear in both parts; matches after the 'Conceptual Framework' heading belong to the in-depth version.

## How to use this skill

The full guide is in [reference.md](reference.md) (about 976 lines). Don't read the whole file. Pick the section you need from the list below, use Grep to find its heading in `reference.md` and get the line number, then use Read with `offset`/`limit` to load just that section.

## Sections in reference.md

- Validation and Transformation Pipeline for Backend Development
  - Overview
  - Architecture
  - Middleware Implementation Pattern
  - Types of Validation
  - Query Parameter Handling
  - Transformation Pipeline
  - Backend Validation for Data Integrity and Security
  - Middleware Implementation Example
  - Query Parameter Validation Example
  - Best Practices
  - Error Response Format
  - Conclusion
- Validation and Transformation Pipeline - Conceptual Framework
  - Purpose and Context
  - Core Concept
  - Pipeline Architecture
  - Validation Types
  - Query Parameter Special Handling
  - Transformation Pipeline
  - Data Integrity Validation
  - Security Validation
  - Error Response Structure
  - Processing Order and Best Practices
  - Integration Points
  - Summary
