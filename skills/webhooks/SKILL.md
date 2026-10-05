---
name: webhooks
description: "Webhooks end to end, as receiver and sender: polling vs push, delivery anatomy and timeouts, verification handshakes and local tunnels, HMAC signature verification and the raw-body trap, replay protection and constant-time comparison, SSRF risks for senders, at-least-once delivery with idempotency and ordering, retries with backoff and jitter, the transactional outbox and dispatcher pattern, and when to use an event log for syncing instead. Use when receiving webhooks (Stripe, GitHub, etc.), building a webhook delivery system, or verifying webhook signatures."
---

# Webhooks

A long numbered guide. Section 18 (Checklists) gives receiver and sender checklists; read it first when reviewing an implementation.

## How to use this skill

The full guide is in [reference.md](reference.md) (about 1169 lines). Don't read the whole file. Pick the section you need from the list below, use Grep to find its heading in `reference.md` and get the line number, then use Read with `offset`/`limit` to load just that section.

## Sections in reference.md

- 1. WHAT A WEBHOOK IS: THE SERVER CALLS YOU
- 2. POLLING AGAINST PUSHING
- 3. WHERE THE NAME CAME FROM, AND WHO SENDS THEM
- 4. ONE DELIVERY END TO END: THE RECEIVER AND GITHUB'S RECORD
- 5. THE ANATOMY OF A DELIVERY, AND THE TIMEOUTS
- 6. HANDSHAKES, AND A TUNNEL TO YOUR LAPTOP
- 7. TRUSTING A DELIVERY: THE THREAT
- 8. FIVE WAYS TO PROVE WHO SENT IT, AND HMAC
- 9. WHAT EXACTLY IS SIGNED, AND THE RAW-BODY TRAP
- 10. REPLAYS AND CONSTANT-TIME COMPARISON
- 11. SSRF: THE PROVIDER'S OWN DANGER
- 12. AT LEAST ONCE: IDEMPOTENCY AND ORDERING
- 13. RETRIES, BACKOFF, JITTER, AND THE CLOCK AGAINST THE LOAD
- 14. BUILDING YOUR OWN: THE OUTBOX
- 15. THE DISPATCHER: SIGN, POST, RECORD, RETRY
- 16. WHEN A WEBHOOK IS THE WRONG TOOL: SYNCING
- 17. THE EVENT LOG: THE RIGHT WAY TO SYNC
- 18. CHECKLISTS
- 19. RESOURCES
