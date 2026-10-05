================================================================================
                                 WEBHOOKS
        A Detailed Guide: Receiving, Securing, Delivering, and Syncing
================================================================================

CONTENTS
--------
  1.  What a Webhook Is: The Server Calls You
  2.  Polling Against Pushing
  3.  Where the Name Came From, and Who Sends Them
  4.  One Delivery End to End: The Receiver and GitHub's Record
  5.  The Anatomy of a Delivery, and the Timeouts
  6.  Handshakes, and a Tunnel to Your Laptop
  7.  Trusting a Delivery: The Threat
  8.  Five Ways to Prove Who Sent It, and HMAC
  9.  What Exactly Is Signed, and the Raw-Body Trap
  10. Replays and Constant-Time Comparison
  11. SSRF: The Provider's Own Danger
  12. At Least Once: Idempotency and Ordering
  13. Retries, Backoff, Jitter, and the Clock Against the Load
  14. Building Your Own: The Outbox
  15. The Dispatcher: Sign, POST, Record, Retry
  16. When a Webhook Is the Wrong Tool: Syncing
  17. The Event Log: The Right Way to Sync
  18. Checklists
  19. Resources


================================================================================
1. WHAT A WEBHOOK IS: THE SERVER CALLS YOU
================================================================================

A webhook is an HTTP request that ANOTHER system sends TO YOUR system when
something happens on its side.

Normally you are the client: you call an API, it answers. With a webhook
the roles flip. You give a provider a URL ("call me here"), and when an
event occurs - a payment succeeds, a commit is pushed, a user signs up -
the provider makes an HTTP POST to that URL with a description of the
event in the body.

    Normal API:   YOUR APP  --- GET /charges/ch_123 --->  PROVIDER
    Webhook:      PROVIDER  --- POST /webhooks/stripe -->  YOUR APP

Put simply: a webhook is a "user-defined HTTP callback". It is just HTTP -
no special protocol, no persistent connection. That simplicity is why it
became the default way SaaS products notify each other.

Three parties and terms used throughout this guide:

  PROVIDER / PRODUCER / SENDER   - the system where the event happens and
                                   which sends the request (GitHub, Stripe).
  RECEIVER / CONSUMER / ENDPOINT - your HTTP handler that accepts it.
  DELIVERY                       - one HTTP attempt to send one event to
                                   one endpoint. One event can have many
                                   delivery attempts.


================================================================================
2. POLLING AGAINST PUSHING
================================================================================

Without webhooks, the only way to learn that something changed is to ask
repeatedly - POLLING:

    every 30 s:  GET /orders?updated_since=...   -> usually "nothing new"

PROBLEMS WITH POLLING
---------------------
  - WASTE: most requests return nothing. With N customers polling every
    T seconds, the provider handles N/T requests per second forever, even
    when nothing happens.
  - LATENCY: on average you learn about a change T/2 seconds late. Making
    T smaller makes the waste worse. Latency and cost pull against each
    other.
  - RATE LIMITS: providers cap polling, so you cannot poll fast enough
    for many resources.

PUSHING (WEBHOOKS)
------------------
  - The provider sends exactly one request per event, right when it
    happens: low latency, no wasted calls.

BUT PUSH IS NOT FREE
--------------------
  - You must run a PUBLIC HTTPS endpoint that is always up.
  - You must AUTHENTICATE requests (anyone can POST to a public URL).
  - Delivery is NOT guaranteed exactly once or in order (Section 12).
  - If you are down, you depend on the provider's retry policy.
  - You cannot easily "backfill" what you missed.

Polling is a PULL model: the consumer controls pace and can always
catch up. Pushing is lower latency but hands control to the sender. The
end of this guide (Section 17) shows how the best systems combine both:
push a small notification, pull the truth.


================================================================================
3. WHERE THE NAME CAME FROM, AND WHO SENDS THEM
================================================================================

The term "web hooks" was popularized by Jeff Lindsay (progrium) in his
2007 blog post "Web hooks to revolutionize the web". His analogy was Unix
pipes and hooks: just as programs let you plug in a script to run on an
event (think git hooks), web applications should let users register a URL
to be called on an event. That would let independent web services be
composed together, the way small Unix tools are composed with pipes.

The idea took off. Today almost every platform with events sends webhooks:

  - Source control / CI:   GitHub, GitLab, Bitbucket
  - Payments:              Stripe, PayPal, Razorpay, Paddle
  - Messaging / comms:     Twilio, Slack, Discord, WhatsApp/Meta, SendGrid
  - Auth / users:          Clerk, Auth0, Supabase
  - Commerce:              Shopify, WooCommerce
  - Webhook infrastructure services that send on your behalf: Svix,
    Hookdeck, and others.

There is also an effort to standardise the format: STANDARD WEBHOOKS
(github.com/standard-webhooks), a specification for headers, signing and
verification that several providers and libraries implement. It is used as
a reference design throughout this guide.


================================================================================
4. ONE DELIVERY END TO END: THE RECEIVER AND GITHUB'S RECORD
================================================================================

Walk through a single GitHub delivery.

STEP 1 - CREATE THE HOOK
------------------------
In a repository: Settings -> Webhooks -> Add webhook:

  Payload URL   https://api.example.com/webhooks/github
  Content type  application/json
  Secret        (a long random string; used for signing - Section 8)
  Events        e.g. "Just the push event", or pick individual events

On creation GitHub immediately sends a `ping` event, so you can see whether
your endpoint is reachable.

STEP 2 - THE MINIMAL RECEIVER (Node / Express)
----------------------------------------------
    import express from "express";
    import crypto from "node:crypto";

    const app = express();

    // IMPORTANT: keep the RAW body for signature verification (Section 9)
    app.post("/webhooks/github",
      express.raw({ type: "application/json" }),
      async (req, res) => {
        const sig = req.get("X-Hub-Signature-256") ?? "";
        if (!verifyGithub(req.body, sig, process.env.GITHUB_WEBHOOK_SECRET!)) {
          return res.status(401).send("bad signature");
        }

        const event      = req.get("X-GitHub-Event");     // "push", "ping"...
        const deliveryId = req.get("X-GitHub-Delivery");  // unique GUID
        const payload    = JSON.parse(req.body.toString("utf8"));

        // Record it, enqueue work, and answer FAST (Section 5)
        await saveInbox(deliveryId, event, payload);      // idempotent insert
        res.status(202).send("accepted");
      });

    function verifyGithub(raw: Buffer, header: string, secret: string) {
      const expected = "sha256=" +
        crypto.createHmac("sha256", secret).update(raw).digest("hex");
      const a = Buffer.from(header), b = Buffer.from(expected);
      return a.length === b.length && crypto.timingSafeEqual(a, b);
    }

    app.listen(Number(process.env.PORT ?? 3000));

STEP 3 - GITHUB'S RECORD
------------------------
GitHub keeps a log of recent deliveries for each hook (Settings ->
Webhooks -> your hook -> "Recent Deliveries"). For each delivery you can
see:

  - the request: URL, headers, full JSON payload;
  - the response: status code, headers, body, and how long it took;
  - a "Redeliver" button to send the same delivery again.

This record is the single most useful debugging tool: when "the webhook
didn't work", it tells you whether GitHub sent it, what it sent, and what
your server answered. Note that GitHub does NOT automatically retry failed
deliveries - you redeliver manually or through the API, and recent
deliveries are only kept for a limited window. Check GitHub's docs for the
current retention period.


================================================================================
5. THE ANATOMY OF A DELIVERY, AND THE TIMEOUTS
================================================================================

A webhook delivery is an ordinary HTTP POST:

    POST /webhooks/github HTTP/1.1
    Host: api.example.com
    User-Agent: GitHub-Hookshot/abc1234
    Content-Type: application/json
    X-GitHub-Event: push
    X-GitHub-Delivery: 72d3162e-cc78-11e3-81ab-4c9367dc0958
    X-GitHub-Hook-ID: 123456789
    X-Hub-Signature-256: sha256=757107ea0eb2509fc211221cce984b8a37570b6d...

    {"ref":"refs/heads/main","before":"...","after":"...","commits":[...],
     "repository":{...},"pusher":{...},"sender":{...}}

THE PARTS
---------
  METHOD        Almost always POST.
  URL           The endpoint you registered.
  EVENT TYPE    Which kind of event (header like X-GitHub-Event, or a
                field in the body like Stripe's "type":"invoice.paid").
  DELIVERY/
  EVENT ID      A unique identifier - the key to idempotency (Section 12).
                GitHub: X-GitHub-Delivery. Stripe: event "id" (evt_...).
                Standard Webhooks: webhook-id.
  TIMESTAMP     When the delivery was signed (Stripe "t=", Standard
                Webhooks webhook-timestamp) - used against replays.
  SIGNATURE     Proof of origin and integrity (Sections 8-9).
  BODY          The event payload: usually JSON.

FAT VS THIN PAYLOADS
--------------------
  FAT (full) payload: the body contains the whole object as it was at
      event time. Convenient, but may be stale by the time you process it,
      and leaks more data if something goes wrong.
  THIN payload: the body says only "object X changed" (type + id). The
      receiver then fetches the current state from the API. Slightly more
      work, but always current and naturally order-insensitive. (Section
      16-17 builds on this.)

THE RESPONSE AND THE TIMEOUT
----------------------------
The provider only cares about your STATUS CODE:

  2xx            -> success, delivery done.
  anything else  -> failure (and, for providers that retry, a retry).
  no answer
  within limit   -> TIMEOUT, also a failure.

Providers give you only a few seconds. GitHub, for example, expects a
response within 10 seconds; other providers use similar or shorter limits.
If your handler does real work inline (sends emails, calls other APIs,
runs a long DB transaction) you will time out under load, the provider
will mark the delivery as failed and possibly retry - while your first
attempt is STILL RUNNING. Now you process the event twice.

THE RULE: ACKNOWLEDGE FAST, PROCESS LATER
-----------------------------------------
    receive -> verify signature -> store (durably) -> enqueue -> 2xx
                                                         |
                                     background worker <-+  does the work

  - Verify and persist synchronously (so a 2xx really means "I have it").
  - Do the actual work asynchronously (BullMQ, SQS, a DB-backed queue).
  - Return 2xx only AFTER the event is durably stored; otherwise a crash
    right after responding loses it forever.

Status code guidance:
  - 200/202/204  : stored (or already seen - duplicates also get 2xx!).
  - 400/401/403  : bad signature or malformed - do not want a retry.
  - 5xx          : temporary problem on your side - please retry.


================================================================================
6. HANDSHAKES, AND A TUNNEL TO YOUR LAPTOP
================================================================================

HANDSHAKES (ENDPOINT VERIFICATION)
----------------------------------
Some providers refuse to send real events until your endpoint proves it
is (a) yours and (b) actually able to receive their requests. This
protects the provider from being used to spam arbitrary URLs.

Common styles:

  - PING EVENT (GitHub): on creation, a harmless `ping` delivery is sent.
    Just return 2xx.

  - ECHO A CHALLENGE (Slack Events API): Slack POSTs
        {"type":"url_verification","challenge":"3eZbrw1aB..."}
    and your endpoint must reply with the challenge value.

  - VERIFY TOKEN + CHALLENGE (Meta / WhatsApp, WebSub-style hubs):
        GET /webhook?hub.mode=subscribe&hub.verify_token=MYTOKEN
                    &hub.challenge=1158201444
    Check verify_token matches what you configured; echo hub.challenge.

  - VALIDATION TOKEN (Microsoft Graph subscriptions): the service sends a
    validationToken query parameter; respond 200 with it as text/plain
    within a few seconds.

  - SIGNED CHALLENGE (e.g. Zoom's URL validation): you must return an HMAC
    of the challenge, proving you also hold the secret.

Exact formats differ per provider; always follow that provider's docs.

A TUNNEL TO YOUR LAPTOP
-----------------------
The provider is on the internet; your dev server is on localhost:3000,
behind NAT. It cannot reach you. A TUNNEL gives your local server a public
URL:

    Provider --HTTPS--> https://abc123.tunnel.example --tunnel--> localhost:3000

Options:
  - ngrok                      ngrok http 3000
  - Cloudflare Tunnel          cloudflared tunnel --url http://localhost:3000
  - smee.io (GitHub suggests it for local development; a small client
    forwards events from a smee channel to localhost)
  - Provider CLIs that forward events directly:
        stripe listen --forward-to localhost:3000/webhooks/stripe
    (prints a test signing secret to use locally)

Tips:
  - The tunnel URL often changes on restart; update the hook config (or
    use a reserved domain).
  - Use a DEV-ONLY secret, not the production one.
  - Combine with the provider's "redeliver" / "send test event" feature to
    replay the same event while you debug.


================================================================================
7. TRUSTING A DELIVERY: THE THREAT
================================================================================

Your webhook URL is a public endpoint that triggers real actions: "mark
order paid", "upgrade plan", "deploy main". Anyone who knows or guesses
the URL can send:

    curl -X POST https://api.example.com/webhooks/stripe \
         -H 'Content-Type: application/json' \
         -d '{"type":"checkout.session.completed","data":{"object":{...}}}'

If you trust the body, an attacker just got a free subscription.

The threats to defend against:

  1. FORGERY      - a request that did not come from the provider.
  2. TAMPERING    - a real request modified in transit.
  3. REPLAY       - a real, correctly signed request captured and sent
                    again later (Section 10).
  4. TIMING LEAKS - an attacker discovering the expected signature byte
                    by byte by measuring how fast you reject (Section 10).
  5. ABUSE OF THE PROVIDER - the reverse direction: your OWN webhook
                    system being tricked into calling internal URLs (SSRF,
                    Section 11).

HTTPS protects against eavesdropping and tampering on the wire, but not
against forgery: anyone can open an HTTPS connection to you. You need
proof of WHO sent it.


================================================================================
8. FIVE WAYS TO PROVE WHO SENT IT, AND HMAC
================================================================================

1) A SECRET IN THE URL
    https://api.example.com/webhooks/stripe/9f8a7c...longrandom
  + Trivial.
  - URLs end up in logs, proxies, browser history, dashboards. No
    integrity: body can be changed. No replay protection. Weakest option.

2) A SHARED TOKEN IN A HEADER (Basic auth / Bearer / custom header)
    Authorization: Bearer whk_...
  + Simple, kept out of most URL logs.
  - Same secret travels with every request; anyone who sees one request
    can forge others. Still no integrity of the body.

3) IP ALLOWLISTING
  Only accept requests from the provider's published IP ranges
  (GitHub publishes them via its meta API, for example).
  + Useful extra layer.
  - Ranges change; easy to misconfigure behind proxies/CDNs; shared
    cloud IPs; does not prove integrity. Use as defence-in-depth only.

4) MUTUAL TLS (mTLS)
  The provider presents a client certificate; you verify it.
  + Strong, transport-level, standard.
  - Operationally heavy (certificate distribution, rotation, termination
    at load balancers). Rare among SaaS webhook providers.

5) SIGNATURES OVER THE PAYLOAD (HMAC, or asymmetric)
  The provider computes a cryptographic signature over the request
  content using a key, and sends it in a header. You recompute and
  compare.
  + Proves origin AND integrity of the exact bytes. The secret itself is
    never sent. With a timestamp in the signed content, it also defeats
    replays.
  - Must be implemented carefully (Sections 9-10).
  This is what GitHub, Stripe, Shopify, Slack, Twilio, Standard Webhooks
  and most others use.

  Asymmetric variant: the provider signs with a PRIVATE key (e.g. Ed25519)
  and publishes the PUBLIC key. Receivers can verify but cannot sign, so
  a leaked verification key cannot be used to forge. Standard Webhooks
  defines an asymmetric option alongside HMAC.

HMAC IN ONE PAGE
----------------
HMAC (Hash-based Message Authentication Code, RFC 2104) combines a SECRET
KEY with a HASH FUNCTION (usually SHA-256) to produce a tag:

    tag = HMAC(key, message)
        = H( (K' XOR opad) || H( (K' XOR ipad) || message ) )

  - Only someone who knows the key can produce a valid tag for a message.
  - Changing even one byte of the message gives a completely different tag.
  - The nested construction with inner/outer padding makes it safe against
    length-extension attacks that break a naive H(key || message).

Webhook flow:

    PROVIDER                                     RECEIVER
    secret S (shared once, out of band)          same secret S
    sig = HMAC_SHA256(S, signed_content)
    send body + header(sig)        --------->    sig' = HMAC_SHA256(S, signed_content)
                                                 accept iff sig' == sig (constant time)

Concrete header formats:

  GitHub:            X-Hub-Signature-256: sha256=<hex HMAC of raw body>
  Stripe:            Stripe-Signature: t=1700000000,v1=<hex>,v1=<hex>
                     (signed content is "<t>.<raw body>")
  Standard Webhooks: webhook-id: msg_2Lh...
                     webhook-timestamp: 1700000000
                     webhook-signature: v1,<base64> [v1,<base64> ...]
                     (signed content is "<id>.<timestamp>.<raw body>")

Multiple signatures in one header exist so a provider can ROTATE secrets:
during rotation it signs with both old and new keys, and the receiver
accepts if ANY matches.

Secret hygiene:
  - Generate with a CSPRNG, at least 32 random bytes.
  - One secret per endpoint (not one per provider account).
  - Store in a secret manager; support rotation.


================================================================================
9. WHAT EXACTLY IS SIGNED, AND THE RAW-BODY TRAP
================================================================================

The signature is over BYTES - the exact bytes the provider sent - not
over "the JSON object". This is where most bugs come from.

THE RAW-BODY TRAP
-----------------
Most frameworks parse JSON automatically:

    app.use(express.json());   // body is now a JS object

If you then do:

    const sig = hmac(secret, JSON.stringify(req.body));   // WRONG

you are signing a RE-SERIALIZED version, which can differ from the
original in many ways:

  - whitespace and newlines ( {"a":1} vs { "a": 1 } );
  - key order;
  - unicode escaping ("é" vs "é"), escaped slashes ("\/");
  - number formatting (1.0 vs 1, large integers losing precision);
  - trailing newline.

The result: valid deliveries fail verification ("works with test events,
fails in production"), and a tempting "fix" is to disable verification.

THE FIX: VERIFY AGAINST THE RAW BYTES, THEN PARSE
-------------------------------------------------
  Express:   app.post("/webhooks/x", express.raw({type: "application/json"}), handler)
             (and make sure a global express.json() does not run first for
             this route), or use express.json({ verify: (req,_,buf) =>
             { req.rawBody = buf } }).
  NestJS:    NestFactory.create(AppModule, { rawBody: true }) and read
             req.rawBody in the controller.
  Next.js (App Router):  const raw = await req.text();  // before JSON.parse
  Fastify:   add a content-type parser that keeps the buffer.
  Python/Flask: request.get_data()  |  Django: request.body

Other pieces of the trap:
  - Do not let middleware decompress/re-encode or change charset first.
  - Use the SAME encoding the provider used for the signature (hex vs
    base64) and the same prefix ("sha256=", "v1,").
  - For signatures that include a timestamp or ID, build the signed string
    EXACTLY as documented: e.g. `${id}.${timestamp}.${rawBody}`.
  - If a proxy or API gateway sits in front, ensure it passes the body
    through unchanged.

STANDARD WEBHOOKS VERIFICATION (illustrative)
---------------------------------------------
    function verifyStandard(raw: Buffer, h: Headers, secretB64: string) {
      const id = h.get("webhook-id")!, ts = h.get("webhook-timestamp")!;
      if (Math.abs(Date.now()/1000 - Number(ts)) > 300) return false; // 5 min
      const key = Buffer.from(secretB64.replace(/^whsec_/, ""), "base64");
      const expected = crypto.createHmac("sha256", key)
        .update(`${id}.${ts}.`).update(raw).digest("base64");
      return h.get("webhook-signature")!.split(" ").some(part => {
        const [ver, sig] = part.split(",");
        return ver === "v1" && safeEqual(sig, expected);
      });
    }

In production, prefer the provider's official SDK (stripe.webhooks.
constructEvent, @octokit/webhooks, the standardwebhooks libraries, svix):
they handle these details correctly.


================================================================================
10. REPLAYS AND CONSTANT-TIME COMPARISON
================================================================================

REPLAY ATTACKS
--------------
A valid signature proves the provider created this exact request at some
point. It does NOT prove it was created just now. An attacker who obtains
one real delivery (from a log, a misconfigured proxy, a leaked HAR file)
can POST it again and again, and the signature will still verify.

Defences:

  1. SIGN A TIMESTAMP and enforce a TOLERANCE WINDOW.
     The timestamp is inside the signed content, so it cannot be changed
     without breaking the signature. Reject if it is too old (or too far
     in the future).
       Stripe: t= in Stripe-Signature, default tolerance 5 minutes in its
               libraries.
       Standard Webhooks: webhook-timestamp, typically a few minutes.
     Keep server clocks in sync (NTP), or valid deliveries get rejected.

  2. REMEMBER IDs YOU HAVE SEEN (dedupe), at least for the length of the
     tolerance window - better, permanently, via your idempotency table
     (Section 12). A replay inside the window is then a harmless
     duplicate.

  Note: GitHub's X-Hub-Signature-256 covers only the body, with no signed
  timestamp, so for GitHub the delivery GUID dedupe is your main defence.

CONSTANT-TIME COMPARISON
------------------------
A normal string comparison (`a === b`, `strcmp`) returns as soon as it
finds the first differing character. The time it takes leaks HOW MANY
leading characters matched. In theory an attacker can submit guesses and
measure response times to discover the correct signature one byte at a
time (a TIMING ATTACK).

The fix: compare in CONSTANT TIME - always examine every byte:

    Node:    crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b))
             (throws if lengths differ - check length first)
    Python:  hmac.compare_digest(a, b)
    Go:      hmac.Equal(a, b)   /  subtle.ConstantTimeCompare
    Ruby:    Rack::Utils.secure_compare / ActiveSupport::SecurityUtils
    Java:    MessageDigest.isEqual(a, b)

    function safeEqual(a: string, b: string) {
      const x = Buffer.from(a), y = Buffer.from(b);
      return x.length === y.length && crypto.timingSafeEqual(x, y);
    }

Over the internet the noise makes such attacks hard, but the fix is one
line, so always do it.


================================================================================
11. SSRF: THE PROVIDER'S OWN DANGER
================================================================================

So far: dangers to the RECEIVER. Once you SEND webhooks yourself, you face
a different danger.

Your product lets customers enter "any URL you like" and then your servers
make HTTP requests to it. That is the definition of SERVER-SIDE REQUEST
FORGERY (SSRF) exposure. A malicious customer registers:

    http://169.254.169.254/latest/meta-data/iam/security-credentials/
                         (cloud instance metadata - credentials!)
    http://localhost:6379/            (your Redis)
    http://10.0.3.17:8080/admin/delete (an internal admin service)
    http://internal-db.svc.cluster.local:5432/

Your dispatcher, sitting inside your network, happily calls them. Your
delivery log may even show the response body back to the attacker.

DEFENCES
--------
  - Resolve DNS and BLOCK private, loopback, link-local and other special
    ranges: 127.0.0.0/8, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16,
    169.254.0.0/16, 100.64.0.0/10, 0.0.0.0/8, ::1, fc00::/7, fe80::/10,
    plus IPv4-mapped IPv6 forms.
  - Check the IP you ACTUALLY CONNECT TO, not just the one you resolved
    when validating: attackers use DNS REBINDING (resolve to a public IP
    at validation time, private IP at request time). Pin the resolved IP
    for the connection.
  - Do NOT follow redirects (or re-validate each hop): a public URL can
    302 to http://169.254.169.254/.
  - Allow only http(s) and normal ports (443, optionally 80 or a short
    list).
  - Run the dispatcher through an EGRESS PROXY in an isolated network
    segment with no route to internal services. Stripe open-sourced
    SMOKESCREEN, an egress proxy built exactly for this: it sits between
    your webhook senders and the internet and denies connections to
    internal/private addresses by policy.
  - Use IMDSv2 / block metadata endpoints from these workers.
  - Do not reflect full response bodies to customers; truncate them.
  - Tight timeouts and response size limits (a "tarpit" endpoint can
    otherwise hold your workers forever).


================================================================================
12. AT LEAST ONCE: IDEMPOTENCY AND ORDERING
================================================================================

DELIVERY GUARANTEES
-------------------
  AT MOST ONCE   - send once, never retry. May lose events.
  AT LEAST ONCE  - retry until acknowledged. Never loses (within the
                   retry window), but MAY DUPLICATE.
  EXACTLY ONCE   - not achievable over an unreliable network between two
                   independent systems. (If the provider sends, you process
                   and return 200, and the response is lost, the provider
                   cannot tell "processed" from "never arrived" - it must
                   retry.)

Practically every webhook provider is AT LEAST ONCE. Therefore:

  EVERY WEBHOOK HANDLER MUST BE IDEMPOTENT.
  Processing the same event twice must have the same effect as once.

Duplicates come from: timeouts (you were still processing), lost
responses, provider-side retries after partial failure, manual
redeliveries, and the provider's own internal at-least-once machinery.

IDEMPOTENCY: THE INBOX TABLE
----------------------------
Use the provider's unique event/delivery ID as a key, enforced by the
database:

    CREATE TABLE webhook_inbox (
      provider      text        NOT NULL,
      event_id      text        NOT NULL,
      event_type    text        NOT NULL,
      payload       jsonb       NOT NULL,
      received_at   timestamptz NOT NULL DEFAULT now(),
      processed_at  timestamptz,
      PRIMARY KEY (provider, event_id)
    );

    INSERT INTO webhook_inbox (provider, event_id, event_type, payload)
    VALUES ($1,$2,$3,$4)
    ON CONFLICT (provider, event_id) DO NOTHING;
    -- 0 rows inserted => duplicate: return 2xx and do nothing else

Then a worker processes rows where processed_at IS NULL. Make the work
itself idempotent too:
  - Prefer state-setting operations ("set status = paid") over increments
    ("balance += 10").
  - Put the business change and "mark processed" in the SAME transaction.
  - When calling other APIs, pass an idempotency key derived from the
    event ID (e.g. Stripe's Idempotency-Key header).

A check-then-act like "SELECT; if not found then process" is NOT enough:
two concurrent duplicate deliveries can both pass the check. Rely on a
unique constraint or a lock.

ORDERING
--------
Webhooks are generally NOT delivered in order. Event B (created after A)
can arrive first because A was retried, because deliveries go out in
parallel, or because networks reorder. Stripe, for example, explicitly
does not guarantee event order.

Example failure: "subscription.updated (status=active)" and later
"subscription.updated (status=canceled)" arrive in reverse order. Naively
applying both in arrival order leaves the user ACTIVE forever.

Strategies:

  1. FETCH CURRENT STATE (thin events). Treat the webhook as "something
     about subscription sub_123 changed", then GET /subscriptions/sub_123
     and store whatever the API says now. Order no longer matters - the
     last fetch wins with the latest truth.
  2. VERSION / TIMESTAMP GUARDS. Store the object's version or
     updated_at; only apply an update if it is newer:
        UPDATE subscriptions SET status=$1, src_updated_at=$2
        WHERE id=$3 AND (src_updated_at IS NULL OR src_updated_at < $2);
     (Event creation time can tie or be coarse; a monotonically increasing
     version or sequence number is better when available.)
  3. STATE MACHINES. Reject impossible transitions (e.g. "paid" -> "pending").
  4. HANDLE "CHILD BEFORE PARENT": an event for an object you have never
     seen (invoice for an unknown customer). Fetch the parent, or retry
     later (return 5xx / requeue).

Per-key ordering can be enforced on your side by processing all events for
the same object through a single partition/queue, but you still cannot
fix the order in which the provider SENT them; strategy 1 or 2 is the
robust answer.


================================================================================
13. RETRIES, BACKOFF, JITTER, AND THE CLOCK AGAINST THE LOAD
================================================================================

When a delivery fails (non-2xx, timeout, connection error), a provider
that retries will try again later.

WHY NOT RETRY IMMEDIATELY, OVER AND OVER?
-----------------------------------------
If the receiver is down because it is overloaded, hammering it with
immediate retries makes things worse - and when it comes back, a flood of
queued retries can knock it over again (a "thundering herd").

EXPONENTIAL BACKOFF
-------------------
Wait longer after each failure:

    delay(n) = base * 2^n     (capped at some maximum)
    e.g. 1 s, 2 s, 4 s, 8 s, 16 s ... cap at 1 h

JITTER
------
If thousands of deliveries failed at the same moment (your outage), plain
exponential backoff makes them all retry at the SAME moments too.
Randomize:

    FULL JITTER:   delay = random(0, min(cap, base * 2^n))
    EQUAL JITTER:  d = min(cap, base * 2^n); delay = d/2 + random(0, d/2)

This spreads the retry load over time.

REAL SCHEDULES
--------------
  - Svix publishes a fixed schedule; at the time of writing its docs list
    attempts at roughly: immediately, 5 seconds, 5 minutes, 30 minutes,
    2 hours, 5 hours, 10 hours, 10 hours (each after the previous
    failure) - about a day and a bit in total. After the final failure
    the message is marked failed and the endpoint may eventually be
    disabled.
  - Stripe retries failed live-mode deliveries for up to 3 days with
    exponential backoff, and emails you about failing endpoints.
  - GitHub does not automatically retry; you redeliver.
  Always check current provider docs - schedules change.

THE CLOCK AGAINST THE LOAD
--------------------------
Every retry schedule is a trade-off between two forces:

  THE CLOCK - you want the event to land as soon as possible after the
  receiver recovers. Long gaps late in the schedule mean an endpoint that
  recovered 1 minute after a failed attempt waits hours for the next one.

  THE LOAD - you want to avoid hammering a struggling receiver and to
  avoid spending your own resources on endpoints that are dead.

Total retry WINDOW also matters: it is how long a receiver can be down
without losing data. After the window, events are gone unless there is
another way to catch up (Section 17).

Good practice for providers:
  - Respect `Retry-After` on 429/503 responses.
  - Treat 410 Gone as "endpoint removed": stop.
  - Treat most 4xx as non-retryable (but 408/409/425/429 are often
    retryable), all 5xx and network errors as retryable.
  - After sustained failure, DISABLE the endpoint and notify the owner,
    rather than retrying forever.
  - Offer manual / bulk "replay since time T" in the dashboard.
  - Isolate endpoints so one slow customer does not delay everyone
    (per-endpoint queues, concurrency limits, circuit breakers).


================================================================================
14. BUILDING YOUR OWN: THE OUTBOX
================================================================================

Now flip sides: your app (e.g. a project board app) needs to SEND
webhooks: "card.created", "card.moved", "board.archived".

THE NAIVE VERSION, AND WHY IT IS WRONG
--------------------------------------
    await db.query("INSERT INTO cards ...");      // 1. commit change
    await fetch(customerUrl, { method: "POST", body });  // 2. send webhook

Failure modes:
  - Crash between 1 and 2: the card exists, the webhook is never sent.
  - Send first, then the DB write fails/rolls back: you announced an event
    that never happened.
  - The customer's endpoint is slow: YOUR user's request now waits for
    THEIR server. A dead endpoint breaks your API latency.

This is the DUAL-WRITE PROBLEM: two systems (your DB and the network) with
no shared transaction.

THE TRANSACTIONAL OUTBOX PATTERN
--------------------------------
Write the event to an OUTBOX table IN THE SAME DATABASE TRANSACTION as the
business change. Sending happens later, from that table.

    BEGIN;
      INSERT INTO cards (id, board_id, title, ...) VALUES (...);
      INSERT INTO outbox (id, event_type, aggregate_id, payload, created_at)
      VALUES (gen_random_uuid(), 'card.created', $cardId, $json, now());
    COMMIT;

Either both rows exist or neither. The event can no longer be lost or
invented. The user's request finishes as soon as the commit is done.

    CREATE TABLE outbox (
      id            uuid PRIMARY KEY,
      seq           bigserial,                 -- global order for the log
      event_type    text  NOT NULL,
      aggregate_id  text  NOT NULL,
      payload       jsonb NOT NULL,
      created_at    timestamptz NOT NULL DEFAULT now()
    );

Then FAN OUT: for each subscription interested in the event, create a
delivery row (either in the same transaction or in a separate step):

    CREATE TABLE webhook_endpoints (
      id         uuid PRIMARY KEY,
      account_id uuid NOT NULL,
      url        text NOT NULL,
      secret     text NOT NULL,          -- store encrypted
      events     text[] NOT NULL,        -- subscribed event types
      enabled    boolean NOT NULL DEFAULT true
    );

    CREATE TABLE webhook_deliveries (
      id              uuid PRIMARY KEY,
      event_id        uuid NOT NULL REFERENCES outbox(id),
      endpoint_id     uuid NOT NULL REFERENCES webhook_endpoints(id),
      status          text NOT NULL DEFAULT 'pending', -- pending|succeeded|failed
      attempt         int  NOT NULL DEFAULT 0,
      next_attempt_at timestamptz NOT NULL DEFAULT now(),
      last_status     int,
      last_error      text,
      last_response   text,             -- truncated
      UNIQUE (event_id, endpoint_id)
    );

The event ID in the outbox becomes the webhook-id that receivers use for
idempotency - identical across every retry of that event.

Alternatives for reading the outbox: polling the table (simplest, works
well with an index) or change data capture (e.g. Debezium reading the
Postgres WAL) for high volume.


================================================================================
15. THE DISPATCHER: SIGN, POST, RECORD, RETRY
================================================================================

The DISPATCHER is a background worker (separate process type) that drains
pending deliveries.

1. CLAIM WORK SAFELY (multiple dispatchers can run)
---------------------------------------------------
    SELECT * FROM webhook_deliveries
    WHERE status = 'pending' AND next_attempt_at <= now()
    ORDER BY next_attempt_at
    LIMIT 50
    FOR UPDATE SKIP LOCKED;

SKIP LOCKED lets many workers pull different rows without blocking each
other. (Or use a queue like BullMQ/SQS keyed by delivery ID.)

2. SIGN
-------
    const id  = event.id;                        // stable across retries
    const ts  = Math.floor(Date.now() / 1000);   // fresh per attempt
    const body = JSON.stringify(envelope);        // serialize ONCE
    const sig = crypto.createHmac("sha256", key)
                  .update(`${id}.${ts}.${body}`).digest("base64");

    headers = {
      "content-type":      "application/json",
      "user-agent":        "BoardApp-Webhooks/1.0",
      "webhook-id":        id,
      "webhook-timestamp": String(ts),
      "webhook-signature": `v1,${sig}`,
    }

Sign the EXACT string you send (no re-serialization afterwards).
Following the Standard Webhooks spec means receivers can use existing
libraries.

3. POST
-------
  - Through the SSRF-safe egress path (Section 11).
  - Short timeout (e.g. 5-15 s connect+response), no redirects.
  - Limit the response size you read.

4. RECORD
---------
Store every attempt: time, status code, duration, truncated response body
or error. This is your version of GitHub's "Recent Deliveries" page -
customers (and your support team) will need it.

    2xx  -> status='succeeded'
    else -> attempt = attempt + 1
            if attempt >= MAX: status='failed' (+ alert / maybe disable endpoint)
            else: next_attempt_at = now() + backoffWithJitter(attempt)

5. RETRY
--------
The row simply becomes eligible again at next_attempt_at; the loop picks
it up. Because the event ID is stable, receivers dedupe correctly.

A dispatcher skeleton:

    async function tick() {
      const rows = await claimDue(50);              // FOR UPDATE SKIP LOCKED
      await Promise.all(rows.map(async d => {
        const started = Date.now();
        try {
          const res = await safeFetch(d.url, signedRequest(d));
          await record(d, res.status, Date.now() - started, await snippet(res));
          if (res.ok) return markSucceeded(d);
          if (!isRetryable(res.status)) return markFailed(d);
          return scheduleRetry(d, retryAfter(res) ?? backoff(d.attempt));
        } catch (err) {
          await record(d, null, Date.now() - started, String(err));
          return scheduleRetry(d, backoff(d.attempt));
        }
      }));
    }

    function backoff(attempt: number) {
      const cap = 6 * 60 * 60 * 1000, base = 5_000;
      const d = Math.min(cap, base * 2 ** attempt);
      return Math.random() * d;                      // full jitter
    }

EXTRAS A GOOD SENDER PROVIDES
-----------------------------
  - Per-endpoint secret, rotation with overlap (multiple signatures).
  - Event type filtering, a "send test event" button, manual replay.
  - Per-endpoint concurrency limits and circuit breakers.
  - Auto-disable after prolonged failure, with email notification.
  - Versioned payload schemas; document every event type.
  - Optionally: use a managed service (Svix, Hookdeck, etc.) instead of
    building all of the above.


================================================================================
16. WHEN A WEBHOOK IS THE WRONG TOOL: SYNCING
================================================================================

A very common use of webhooks is SYNCING: keeping a copy of the provider's
data in your database. Examples: mirror Clerk users into your `users`
table; mirror Stripe subscriptions into `subscriptions`.

Webhooks are a poor foundation for this on their own:

  - LOSS: if your endpoint is down longer than the retry window, or a bug
    returned 2xx without saving, events are gone. There is no "what did I
    miss?" query.
  - ORDER: events arrive out of order (Section 12), so naive replay
    produces wrong state.
  - DUPLICATES: at least once.
  - LATENCY / RACE CONDITIONS: the webhook is asynchronous. Example: a
    user signs up in your frontend and is immediately redirected to a
    page that queries YOUR users table - but the user.created webhook
    hasn't arrived yet. The page breaks. Clerk's documentation on syncing
    data makes this point: webhooks are asynchronous and not suited to
    flows that need the data immediately; for those, use the session
    token / API directly and treat webhooks as eventually consistent.
  - BOOTSTRAP: webhooks only tell you about CHANGES. They cannot fill an
    empty database with everything that already exists.
  - DRIFT: any missed event means your copy is silently wrong forever,
    unless you periodically reconcile.

Before syncing via webhooks, ask: do I need a copy at all? Often you can
just call the provider's API (with caching) when you need the data.

If you do need a copy, the robust approach is the hybrid:

  1. Treat webhooks as a NUDGE ("something changed, go look"), not as the
     source of truth.
  2. On nudge, FETCH current state from the API and upsert it.
  3. Run a periodic RECONCILIATION job that walks the provider's list API
     and fixes drift.
  4. Better still, consume an EVENT LOG if the provider offers one
     (next section).

Brandur Leach's writing on webhooks (see Resources) argues this from the
provider side: webhooks are operationally hard for both sides, and an
API that lets consumers read events at their own pace is often better.


================================================================================
17. THE EVENT LOG: THE RIGHT WAY TO SYNC
================================================================================

THE IDEA
--------
The provider exposes its events as an ORDERED, DURABLE, PAGINATED LOG
that consumers READ with a CURSOR:

    GET /events?after=evt_000123&limit=100
    -> {
         "data": [ {"id":"evt_000124","seq":124,"type":"card.moved",...},
                   ... ],
         "next_cursor": "evt_000223",
         "has_more": true
       }

The consumer stores its cursor ("I have processed everything up to seq
223") and keeps reading from there. This is how Kafka consumers, database
replication (reading the WAL), and change feeds work. Stripe, for example,
exposes a list of recent events via its API, which you can page through
to recover missed events within its retention window.

WHY IT FIXES THE WEBHOOK PROBLEMS
---------------------------------
  - NO LOSS: the log is durable; if you are down for a day, you resume
    from your cursor and catch up.
  - ORDER: the log has a defined order (a sequence number), so you apply
    events in the right order.
  - DUPLICATES: you advance the cursor in the same transaction as the
    changes you apply; re-reading after a crash is harmless if handlers
    are idempotent.
  - BACKPRESSURE: the consumer reads at its own pace. A slow consumer
    never overloads itself, and the provider never maintains per-endpoint
    retry state.
  - BOOTSTRAP: combine with a full list/snapshot, then follow the log
    from the snapshot point.
  - NO PUBLIC ENDPOINT required, and no SSRF exposure for the provider.

COMBINING THE TWO: WEBHOOK AS A DOORBELL
----------------------------------------
Pure polling of the log brings back the latency vs cost trade-off of
Section 2. The best design uses both:

    1. Provider appends to its event log (fed by the outbox, Section 14 -
       the outbox's `seq` column IS the log).
    2. Provider sends a tiny webhook: "new events available" (or a thin
       event with just an ID).
    3. Consumer, on webhook OR on a timer (safety net, e.g. every minute),
       reads the log from its cursor until has_more = false.

    Missed webhook?    -> the timer catches up.
    Duplicate webhook? -> reading the log is idempotent.
    Out-of-order?      -> the log is ordered.

The webhook only carries "when to look"; the log carries "what happened".
Push for latency, pull for correctness.

CONSUMER SKETCH
---------------
    async function syncFromLog() {
      let cursor = await getCursor("board-app");
      for (;;) {
        const page = await api.get(`/events?after=${cursor}&limit=100`);
        await db.tx(async tx => {
          for (const e of page.data) await applyEvent(tx, e);  // idempotent
          await setCursor(tx, "board-app", page.next_cursor);
        });
        cursor = page.next_cursor;
        if (!page.has_more) break;
      }
    }
    // trigger: on webhook delivery (debounced) AND every 60 s

DESIGN NOTES FOR PROVIDERS
--------------------------
  - Give each event a monotonically increasing sequence per log (per
    account or global).
  - Define and document RETENTION (how far back can a cursor go?), and
    what happens if a consumer falls further behind (snapshot + resume).
  - Beware gaps from concurrent transactions committing sequence numbers
    out of order: only expose events up to a "safe" point, or assign
    sequence numbers at commit/relay time.


================================================================================
18. CHECKLISTS
================================================================================

RECEIVER CHECKLIST
------------------
  [ ] HTTPS endpoint, one per provider, secret per endpoint.
  [ ] Verify signature against the RAW body, with the provider's SDK
      where possible.
  [ ] Constant-time comparison.
  [ ] Enforce timestamp tolerance (where the provider signs one).
  [ ] Dedupe by event/delivery ID with a unique constraint.
  [ ] Store durably, then return 2xx quickly; process in a worker.
  [ ] Return 2xx for duplicates; 4xx for bad signatures; 5xx to ask for
      a retry.
  [ ] Handle out-of-order events: fetch current state or version guards.
  [ ] Handle unknown event types gracefully (2xx and ignore).
  [ ] Log delivery IDs; never log secrets.
  [ ] Reconciliation job / event log for anything you sync.
  [ ] Tunnel + provider "redeliver" for local development.

SENDER CHECKLIST
----------------
  [ ] Transactional outbox; no dual writes.
  [ ] Separate dispatcher process; FOR UPDATE SKIP LOCKED or a queue.
  [ ] Stable event ID across retries; fresh timestamp per attempt.
  [ ] HMAC-SHA256 signing (consider Standard Webhooks format);
      multiple signatures for rotation.
  [ ] SSRF protection: IP checks at connect time, no redirects, egress
      proxy (e.g. Smokescreen), size and time limits.
  [ ] Exponential backoff with jitter; respect Retry-After; cap total
      window; auto-disable + notify.
  [ ] Per-endpoint isolation and concurrency limits.
  [ ] Delivery log visible to customers; manual replay; test events.
  [ ] Documented event types and payload versions.
  [ ] Offer an event log API for consumers that need to sync.


================================================================================
19. RESOURCES
================================================================================

  * Jeff Lindsay - "Web hooks to revolutionize the web" (2007)
      https://progrium.github.io/blog/2007/05/03/web-hooks-to-revolutionize-the-web/
  * GitHub - Webhook events and payloads
      https://docs.github.com/en/webhooks/webhook-events-and-payloads
  * GitHub - Validating webhook deliveries
      https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries
  * Stripe - Receive events in your webhook endpoint
      https://docs.stripe.com/webhooks
  * Standard Webhooks - the specification
      https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md
  * Svix - Retry schedule
      https://docs.svix.com/retries
  * Brandur Leach - Webhooks, operability and idempotency
      https://brandur.org/webhooks
  * Stripe - Smokescreen (SSRF-preventing egress proxy)
      https://github.com/stripe/smokescreen
  * Clerk - Syncing Clerk data to your application
      https://clerk.com/docs/guides/development/webhooks/syncing
  * RFC 2104 - HMAC: Keyed-Hashing for Message Authentication
      https://www.rfc-editor.org/rfc/rfc2104
  * The board app used in the video is a private repository; the receiver
    and producer are shown in full in the video. Code in this file is
    illustrative and written independently.

  Source video timestamps
      0:00   What a webhook is: the server calls you
      3:21   Polling against pushing
      6:29   Where the name came from, and who sends them
      8:29   One delivery, end to end: the receiver
      9:56   The anatomy of a delivery, and the timeouts
      12:53  Handshakes, and a tunnel to your laptop
      15:25  Trusting a delivery: the threat
      16:52  Five ways to prove it, and HMAC
      20:30  What exactly is signed, and the raw-body trap
      22:44  Replays, and constant-time comparison
      25:16  SSRF: the provider's own danger
      27:39  At least once: idempotency and ordering
      34:31  Retries, backoff, and the clock against the load
      38:58  Building your own: the outbox
      43:10  The dispatcher: sign, POST, record, retry
      48:19  When a webhook is the wrong tool: syncing
      52:54  The event log: the right way to sync

================================================================================
                                   END
================================================================================
