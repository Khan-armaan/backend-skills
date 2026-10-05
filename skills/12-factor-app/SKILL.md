---
name: 12-factor-app
description: "The Twelve-Factor App methodology in depth: codebase, dependencies, config in environment variables, backing services, build/release/run, stateless processes and sticky sessions, port binding, concurrency via the process model, disposability and graceful shutdown, dev/prod parity, logs as event streams, and admin processes, plus a Procfile walkthrough and cheat sheet. Use when designing cloud-native or containerized services, reviewing configuration, deployment or scaling setup, or deciding how an app should handle state, env vars, restarts and logs."
---

# The Twelve-Factor App

A detailed guide to the twelve factors: where they came from, each factor's rule, why it matters, common violations, and modern notes. Section 16 is a quick-reference cheat sheet; read it first for a fast answer.

## How to use this skill

The full guide is in [reference.md](reference.md) (about 1218 lines). Don't read the whole file. Pick the section you need from the list below, use Grep to find its heading in `reference.md` and get the line number, then use Read with `offset`/`limit` to load just that section.

## Sections in reference.md

- 0. INTRODUCTION: WHY THE TWELVE FACTORS STILL RUN OUR LIVES
- 1. WHERE THE TWELVE FACTORS CAME FROM: HEROKU, 2011, SOFTWARE EROSION
- 2. WHO OWNS IT NOW: THE 2024 OPEN-SOURCING AND THE COMMUNITY REVISION
- 3. FACTOR I - CODEBASE
- 4. FACTOR II - DEPENDENCIES
- 5. FACTOR III - CONFIG
- 6. FACTOR IV - BACKING SERVICES
- 7. FACTOR V - BUILD, RELEASE, RUN
- 8. FACTOR VI - PROCESSES
- 9. FACTOR VII - PORT BINDING
- 10. FACTOR VIII - CONCURRENCY
- 11. FACTOR IX - DISPOSABILITY
- 12. FACTOR X - DEV/PROD PARITY
- 13. FACTOR XI - LOGS
- 14. FACTOR XII - ADMIN PROCESSES
- 15. THE REAL PROCFILE FORMATION: BOOT, ONE-OFF JOB, GRACEFUL SHUTDOWN
- 16. QUICK-REFERENCE CHEAT SHEET
- 17. RESOURCES
