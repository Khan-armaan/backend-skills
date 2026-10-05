---
name: security
description: "Backend security: authentication vs authorization, password hashing (Argon2id, bcrypt, salt, pepper), stateful sessions vs stateless JWTs, OAuth2/OIDC with PKCE, rate limiting, BOLA/BFLA/IDOR authorization flaws and mitigations, XSS, CSRF, security headers, secrets handling, and a threat-modeling mindset. Use when implementing login, sessions or tokens, writing authorization checks, handling secrets, or doing a security review of backend code."
---

# Security in Backend

### Authentication vs Authorization

These are two different questions and conflating them is the root of most access-control bugs.

**Authentication (authn)** answers *"who are you?"* — proving identity. **Authorization (authz)** answers *"what are you allowed to do?"* — checking permissions. A user can be perfectly authenticated and still must be denied an action. On your auction platform: logging in is authn; checking that *this* bidder is allowed to see *that* sealed bid is authz. The mistake people make is doing authn well and then assuming an authenticated user is automatically allowed to touch any object — that's exactly the BOLA hole below.

### Password storage: hashing, salting, encryption

You've got the key insight right: **hashing is one-way, encryption is reversible.** You never encrypt passwords, because anything reversible means a stolen key = all passwords exposed. You hash so that even you can't recover the plaintext; at login you hash the submitted password and compare hashes.

The critical nuance is *which* hash. Fast general-purpose hashes (MD5, SHA-256) are wrong for passwords — a GPU can compute billions per second, so they're trivially brute-forced. Use a **slow, memory-hard, password-specific** algorithm: **Argon2id** (current best), or **bcrypt**/**scrypt**. The cost is deliberate.

**Salt** is a unique random value per password, stored alongside the hash. It defeats rainbow tables and ensures two users with the same password get different hashes. Modern algorithms (bcrypt, Argon2) generate and embed the salt for you. Optionally a **pepper** — a secret added to all passwords, kept *outside* the database (in a secret manager) — so a DB-only leak still isn't enough.

### Stateful vs stateless — and the claim that stateful is "better"

I'd push back on "stateful is better" as a flat statement; it's a tradeoff, and the framing matters because it drives your architecture.

**Stateful (server-side sessions):** the server stores the session; the client holds only an opaque random session ID. Strength: *instant revocation* — delete the session and the user is logged out everywhere immediately. Cost: a lookup (Redis/DB) on every request, and you need a shared session store across instances.

**Stateless (JWT):** the server signs a token containing the claims and stores nothing. Strength: *no lookup* — any instance verifies the signature locally, which scales beautifully across many services. Weakness: you **cannot easily revoke** a token before it expires, because the server isn't tracking it. That's the real cost.

So stateful is better *at control and revocation*; stateless is better *at horizontal scale and avoiding a central store*. The pragmatic answer most production systems land on is a **hybrid**: a short-lived stateless JWT access token (minutes) plus a long-lived **refresh token that is stored server-side** and therefore revocable. You get fast stateless verification on most requests, and you keep a revocation lever via the refresh token. For an auction platform where you may need to kill a compromised or banned account *now*, that revocability matters.

### Sessions in detail

A session is server-side proof of an already-authenticated identity. Generate the session ID as a **high-entropy random value** (never sequential, never guessable). Store it server-side keyed by that ID, with metadata: user ID, created/expires timestamps, originating IP, user-agent. Redis is the common choice because it's fast and has built-in TTL — expiration and revocation are essentially free.

The server sends the ID to the client, which stores it in a cookie. The cookie flags are where security actually lives:

- **HttpOnly: true** — JavaScript cannot read the cookie, so an XSS payload can't steal the session token.
- **Secure: true** — sent only over HTTPS, so it can't leak over plaintext.
- **SameSite: Lax** (or Strict) — the cookie isn't sent on cross-site requests, which is your primary CSRF defense today.

### JWT (the stateless path) in detail

An **access token** is short-lived (e.g. 5–15 min), sent on each request (typically `Authorization: Bearer …`), and carries claims (user ID, roles, expiry). A **refresh token** is long-lived and used *only* to mint new access tokens when they expire, so the user isn't logged out every 15 minutes.

Two things people get wrong: first, the JWT payload is **base64, not encrypted** — anyone can read it, so never put secrets in it. Second, pin the algorithm — historically attackers exploited servers that accepted `alg: none` or let them downgrade RS256 to HS256. Validate signature, issuer, audience, and expiry on every request. And as above, store refresh tokens server-side so you can revoke them.

### OAuth2 / social login / Clerk

**OAuth2** is a *delegated authorization* framework — it lets your app get permission to act on a resource without handling the user's password. "Sign in with Google" actually layers **OpenID Connect (OIDC)** on top of OAuth2 to do *authentication* (OAuth2 alone is about access, not identity). The flow you want for web/mobile is the **Authorization Code flow with PKCE**: user is redirected to Google, authenticates there, Google returns a short-lived code to your callback, your backend exchanges that code (plus the PKCE verifier) for tokens. PKCE stops an intercepted code from being usable.

Providers like **Clerk, Auth0, Supabase Auth, WorkOS** exist because rolling all of this yourself — social providers, session management, MFA, password reset, email verification — is a lot of security-critical surface. For a platform with legal/auditability stakes like real-estate bidding, using a vetted provider for authn is usually the right call so you can focus your security effort on *authorization* and the bidding logic, which no provider can do for you.

### Rate limiting and abuse prevention

Layer it: **per-IP** (blunt, defeats naive scripts but breaks under shared NAT/CGNAT), **per-account** (defeats targeted brute force on one user), and a **global** ceiling (protects the system itself). Implement with a token-bucket or sliding-window counter in Redis. Escalate to **CAPTCHA** when behavior looks suspicious rather than always-on.

This belongs in **auth middleware** so it's enforced before expensive logic runs. The endpoints that most need it: login, signup, password reset, OTP/verification — these are where credential stuffing and enumeration happen. Note that for your bid endpoints, you also want limits, but there the concern is more fairness/concurrency than pure abuse — different goal, same mechanism.

### The authorization failures: BOLA, BFLA, IDOR

These are the most important section for your platform, and they're #1 and #5 on the OWASP API Top 10 for a reason.

**BOLA (Broken Object Level Authorization):** a user accesses an object they don't own by changing an identifier — `GET /bids/1043` works, so they try `/bids/1044` and see someone else's bid. The fix is non-negotiable: **on every object access, verify the current user is authorized for that specific object**, server-side, in the query itself (`WHERE id = ? AND owner_id = current_user`) — not just "is logged in."

**BFLA (Broken Function Level Authorization):** a regular user invokes a function/endpoint meant for a higher role — calling `POST /admin/auctions/close` directly even though the UI never shows the button. Fix: enforce a role/permission check on every privileged function, not just hide it in the frontend.

**IDOR** is the broader umbrella — exposing internal references that can be enumerated.

On the **UUID vs sequential ID** point: switching from sequential IDs to UUIDs is good — it stops trivial enumeration. But your note already flags the trap correctly. Unguessable IDs are **defense in depth, not authorization.** A UUID in a URL can still be shared, logged, leaked in a referrer header, or sit in someone's browser history. Treating "they can't guess the URL" as protection is **security through obscurity**, which is not security. You must *still* run the authz check even with UUIDs. The UUID just buys you a second layer.

### Authorization mitigations

- **Centralize** authn/authz decisions in one policy layer or middleware, not scattered `if` checks across controllers — scattered checks are where one gets forgotten.
- **Default deny:** anything not explicitly allowed is denied. The absence of a rule should never mean "permit."
- **Test authorization explicitly:** write tests asserting user A *cannot* read/modify user B's resources. Authz bugs don't throw errors; they silently succeed, so they only surface if you test for the negative case.
- **Audit logs:** record sensitive actions (who did what to which object, when, from where). For a bidding platform with auditability requirements this is also a product feature, not just security hygiene.

### XSS (Cross-Site Scripting)

Attacker gets script to run in another user's browser — then it can read non-HttpOnly tokens, perform actions as them, or deface the page. Three flavors: **stored** (persisted in your DB and served to others — the dangerous one for a multi-user platform), **reflected** (echoed back from a request), and **DOM-based** (client-side sink).

Your note says "sanitization in validation layer" — I'd reframe: the primary defense is **context-aware output encoding** (escaping at render time), not input sanitization. Input validation helps, but the same data is safe in one context and dangerous in another. Modern frameworks (React, etc.) auto-escape by default, which is why you must be deliberate whenever you bypass that (`dangerouslySetInnerHTML`). When you genuinely must render user-supplied HTML, sanitize with a vetted library (DOMPurify). **CSP** is the backstop, and **HttpOnly cookies** mean even a successful XSS can't steal the session.

### CSRF (Cross-Site Request Forgery)

Attacker tricks an authenticated user's browser into making a state-changing request to your site — exploiting the fact that the browser *automatically attaches cookies*. This only works against cookie-based auth. Defenses: **SameSite cookies** (the modern baseline), a **CSRF token** (synchronizer or double-submit pattern) on state-changing requests, and checking **Origin/Referer**. Note the tradeoff: if you put your JWT in an `Authorization` header instead of a cookie, classic CSRF doesn't apply (the browser won't auto-attach it) — but you've now exposed yourself to XSS token theft. Cookie vs header auth is choosing *which* attack class you defend against, which is why HttpOnly + SameSite is so popular.

### Security headers

Set these at the edge/middleware so every response carries them:

- **CSP (Content-Security-Policy):** whitelists what scripts/resources can load — your strongest XSS mitigation.
- **X-Frame-Options / `frame-ancestors`:** stops your site being embedded in an attacker's iframe — defeats clickjacking.
- **HSTS (Strict-Transport-Security):** forces HTTPS, prevents downgrade.
- **X-Content-Type-Options: nosniff:** stops MIME-type sniffing.

### Secrets, config, and the debug/live distinction

Secrets — API keys, DB passwords, signing keys — never go in code or git. Use environment variables backed by a **secret manager** (Vault, AWS/GCP Secrets Manager), apply least privilege per key, and rotate them.

Your debug/live note is a real and commonly-missed issue: in production, **debug mode off**, **stack traces never shown to users** (they leak file paths, library versions, query structure — a reconnaissance gift), and an appropriate **log level** (info, not debug). The pattern is: generic error to the user, detailed error to your server logs.

### The mindset at the end — this is the most important part

Your closing notes are actually the meta-skill that makes all the above coherent, so I'll restate them as principles:

**Find the trust boundaries** — every point where data crosses from a place you don't control (the user's browser, a third-party webhook, another service) into a place you do. *All* untrusted input gets validated/encoded at that boundary.

**Name your assumptions and assume they're wrong** — "this ID belongs to the logged-in user," "only the frontend calls this endpoint," "this field is always a number." For each one ask: what happens if it's false, and is there a check that catches it?

**Defense in depth** — no single control should be the only thing between an attacker and damage. UUIDs *and* authz checks. SameSite *and* CSRF tokens. Output encoding *and* CSP. When one layer fails (and one always eventually does), another catches it.
