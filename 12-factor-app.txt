================================================================================
                        THE TWELVE-FACTOR APP
             A Detailed Guide: History, Factors, Revision, Practice
================================================================================

CONTENTS
--------
  0. Introduction: Why the Twelve Factors Still Run Our Lives
  1. Where the Twelve Factors Came From: Heroku, 2011, Software Erosion
  2. Who Owns It Now: The 2024 Open-Sourcing and the Community Revision
  3. Factor I    - Codebase
  4. Factor II   - Dependencies
  5. Factor III  - Config (including the open-source litmus test)
  6. Factor IV   - Backing Services
  7. Factor V    - Build, Release, Run
  8. Factor VI   - Processes (stateless, sticky sessions, WebSocket exception)
  9. Factor VII  - Port Binding
 10. Factor VIII - Concurrency
 11. Factor IX   - Disposability
 12. Factor X    - Dev/Prod Parity
 13. Factor XI   - Logs
 14. Factor XII  - Admin Processes
 15. The Real Procfile Formation: Boot, One-Off Job, Graceful Shutdown
 16. Quick-Reference Cheat Sheet
 17. Resources


================================================================================
0. INTRODUCTION: WHY THE TWELVE FACTORS STILL RUN OUR LIVES
================================================================================

The Twelve-Factor App is a methodology for building software-as-a-service
applications that:

  * use declarative formats for setup automation, so a new developer (or a
    new machine) can join the project with minimal time and cost;
  * have a clean contract with the underlying operating system, giving
    maximum portability between execution environments;
  * are suitable for deployment on modern cloud platforms, removing the need
    for hand-managed servers;
  * minimize divergence between development and production, enabling
    continuous deployment;
  * can scale up without significant changes to tooling, architecture, or
    development practices.

Even if you have never read the document, you almost certainly follow it.
Every time you:

  - put a DATABASE_URL in an environment variable,
  - write logs to stdout and let Docker / Kubernetes collect them,
  - build a container image once and promote the same image through
    staging to production,
  - scale "web" pods horizontally behind a load balancer,
  - run a database migration as a separate job before rolling out,

...you are applying twelve-factor ideas. Kubernetes, Docker, Heroku, Render,
Fly.io, Railway, Cloud Run, AWS ECS and most PaaS products are built around
the assumptions the methodology describes. Understanding the "why" behind
each factor lets you know when a rule is load-bearing and when it is safe
to bend.


================================================================================
1. WHERE THE TWELVE FACTORS CAME FROM: HEROKU, 2011, SOFTWARE EROSION
================================================================================

THE AUTHORS AND THE CONTEXT
---------------------------
The methodology was written by Adam Wiggins, a co-founder of Heroku, and
published at 12factor.net around 2011. It distilled what Heroku's team had
learned from watching an enormous number of applications being built,
deployed, operated and scaled on their platform.

Heroku in 2011 was one of the first mainstream Platform-as-a-Service
products. You pushed code with `git push heroku master`, and the platform
built it, ran it in lightweight containers called "dynos", routed traffic to
it, and let you scale with a single command. For that to work for
*arbitrary* customer apps, the apps had to obey a contract: read config
from the environment, bind to a port given to you, don't write to local
disk and expect it to persist, log to stdout, be killable at any time.

The twelve factors are, essentially, that contract written down in a
language-agnostic way, so that it applies to Ruby, Python, Java, Node, Go,
or anything else.

THE TWO PROBLEMS IT ATTACKS
---------------------------
The original introduction frames the goal as addressing:

  1. The dynamics of an application's organic growth over time.
  2. The dynamics of collaboration between developers working on the
     codebase.
  3. Avoiding the cost of "software erosion".

SOFTWARE EROSION
----------------
"Software erosion" (sometimes called "software rot" or "bit rot") is the
gradual decay of a running application even when nobody touches its code.
The world around it keeps moving:

  - the OS gets security patches,
  - system libraries get upgraded or removed,
  - language runtimes go end-of-life,
  - the server it lives on gets replaced,
  - the one engineer who knew how to deploy it leaves.

An app that depends on implicit things - a globally installed binary, a
config file that someone hand-edited on server #3, a cron job that lives
only in one crontab - erodes fast. When it finally breaks, nobody can
reproduce its environment.

The twelve factors fight erosion by making everything the app depends on
EXPLICIT and REPRODUCIBLE: dependencies declared, config externalized,
builds repeatable, processes disposable. Heroku's related writing on
erosion-resistance argued that apps should be insulated from the
platform underneath them, so the platform can be patched and upgraded
continuously without breaking the apps on top.

THE INFLUENCES
--------------
The document openly credits earlier thinking, notably Martin Fowler's
books "Patterns of Enterprise Application Architecture" and "Refactoring".
It was also strongly shaped by Unix philosophy: processes, streams,
environment variables, and small composable tools.


================================================================================
2. WHO OWNS IT NOW: THE 2024 OPEN-SOURCING AND THE COMMUNITY REVISION
================================================================================

For more than a decade the methodology was effectively frozen: a static
website owned by Heroku (and later Salesforce, which acquired Heroku in
2010). The text aged. It never mentioned containers, Kubernetes, service
meshes, secret managers, OpenTelemetry, or serverless functions.

THE OPEN-SOURCING (12 NOVEMBER 2024)
------------------------------------
On 12 November 2024 the Twelve-Factor App was announced as open source
(see the announcement at 12factor.net/blog/open-source-announcement).
The content moved to a public GitHub organization:

    https://github.com/twelve-factor/twelve-factor

The stated intent is to let the broader community - not a single vendor -
maintain and modernize the methodology, through public discussion,
issues and pull requests, with a governance process for accepting changes.

KUBECON NORTH AMERICA 2024
--------------------------
Around the same time, Vish Abrams (Chief Architect at Heroku) gave a
sponsored keynote at KubeCon + CloudNativeCon North America 2024 titled
"The Twelve-Factor App ..." explaining the move. The core message: the
twelve factors became the de facto foundation of cloud-native development,
the ecosystem has changed enormously since 2011, and the methodology
should now evolve in the open alongside the cloud-native community.

WHAT THE REVISION IS LOOKING AT
-------------------------------
The revision is ongoing work, so check the GitHub repository for the
current state. The areas that are commonly discussed as needing an update
include:

  * CONFIG AND SECRETS - The original "store config in env vars" rule is
    widely criticized when applied to secrets (see the Diogo Monica article
    in Resources). Modern practice favours secret managers, mounted
    secret files, and short-lived credentials.
  * WORKLOAD IDENTITY - Instead of long-lived API keys in config, apps can
    prove who they are to backing services using platform-issued identity
    (e.g. cloud IAM roles, SPIFFE-style identities, OIDC tokens).
  * OBSERVABILITY - "Logs as event streams" is still right, but modern
    systems also emit metrics and traces (e.g. OpenTelemetry). Telemetry
    as a whole, not just logs, is in scope.
  * CONTAINERS AND ORCHESTRATION - Language about "dynos", "process
    managers", and Procfiles maps onto container images, pods,
    deployments, jobs and init containers.
  * DEPENDENCIES BEYOND THE LANGUAGE - Container base images, system
    packages, and supply-chain concerns (lockfiles, SBOMs, provenance).
  * BACKING SERVICES - Managed cloud services, service discovery and
    bindings are richer than "a URL in an env var".

Throughout this guide, "ORIGINAL" means the 2011 text, and "MODERN NOTE"
means how the idea is usually applied today.


================================================================================
3. FACTOR I - CODEBASE
   "One codebase tracked in revision control, many deploys"
================================================================================

THE RULE
--------
A twelve-factor app is always tracked in a version control system (Git,
historically also Subversion/Mercurial). There is a ONE-TO-ONE relationship
between a codebase and an app:

  * If there are multiple codebases, it is not an app - it is a
    distributed system. Each component in a distributed system is an app,
    and each can individually comply with the twelve factors.
  * If multiple apps share the same code, that is a violation. The shared
    code should be factored out into libraries and included via the
    dependency manager (Factor II).

A "DEPLOY" is a running instance of the app: production, staging, each
developer's local copy. There is ONE codebase but MANY deploys. Deploys may
run different commits (staging is ahead of production; your laptop is
ahead of staging), but they are all versions of the same codebase.

WHY IT MATTERS
--------------
  - If production is built from a different repo, branch-fork, or
    "the copy on the server", you lose the ability to know what is
    running.
  - Copy-pasted code between two apps diverges silently; bug fixes land
    in one and not the other.

COMMON VIOLATIONS
-----------------
  - Editing files directly on the production server.
  - Two services that `cp -r` the same "utils" folder.
  - A separate "prod" repository manually synced from "dev".
  - One repo that contains five unrelated apps with no clear boundary.

MONOREPOS
---------
Monorepos are not automatically a violation. The spirit of Factor I is
that each deployable app has a single, well-defined source of truth, and
shared code is consumed as a versioned dependency (workspace packages,
internal libraries). A monorepo with clear per-app build targets and
shared packages fits the spirit well.

MODERN NOTE
-----------
Today the "codebase" is usually a Git repo, and the immutable artifact is
a container image tagged with the commit SHA. Many deploys = many
environments (preview/PR environments, staging, prod) running images
built from that one repo.


================================================================================
4. FACTOR II - DEPENDENCIES
   "Explicitly declare and isolate dependencies"
================================================================================

THE RULE
--------
A twelve-factor app NEVER relies on the implicit existence of system-wide
packages. It:

  1. DECLARES all dependencies, completely and exactly, via a dependency
     declaration manifest; and
  2. ISOLATES them during execution, so no implicit dependencies "leak in"
     from the surrounding system.

Both are required; one without the other is not enough.

EXAMPLES BY ECOSYSTEM
---------------------
    Language   Declaration                        Isolation
    --------   --------------------------------   ---------------------------
    Ruby       Gemfile + Gemfile.lock             bundle exec
    Python     requirements.txt / pyproject +     virtualenv / venv / uv /
               lockfile                           poetry
    Node.js    package.json + package-lock.json   local node_modules
               (or pnpm-lock / yarn.lock)
    Go         go.mod + go.sum                    module cache, static binary
    Java       pom.xml / build.gradle             build tool classpath
    Rust       Cargo.toml + Cargo.lock            cargo
    .NET       *.csproj / packages.lock.json      NuGet restore

SYSTEM TOOLS
------------
The original text explicitly calls out system tools: a twelve-factor app
does not assume `curl` or `ImageMagick` exist on the host. Even if they are
present on most systems today, there is no guarantee they will exist on
the next machine or be the version you expect. If the app needs to shell
out to a tool, that tool should be vendored into the app or declared.

WHY IT MATTERS
--------------
  - SETUP: A new developer clones the repo, runs one deterministic command
    (`npm ci`, `bundle install`, `pip install -r ...`), and is ready.
  - REPRODUCIBILITY: The same versions are installed on every machine.
  - EROSION RESISTANCE: OS upgrades on the host do not silently change the
    library versions the app runs against.

LOCKFILES
---------
A manifest with ranges (e.g. "^4.18.0") is a declaration. A LOCKFILE pins
the exact resolved tree and is what makes builds reproducible. Commit
lockfiles for applications. Use `npm ci` (not `npm install`) in CI.

MODERN NOTE
-----------
Containers made "isolation" almost universal: the Dockerfile is a
declaration of the whole OS-level dependency set, and the container is the
isolation boundary. That extends Factor II from language libraries to
system packages:

    FROM node:22-slim
    RUN apt-get update && apt-get install -y --no-install-recommends \
        imagemagick && rm -rf /var/lib/apt/lists/*
    COPY package.json package-lock.json ./
    RUN npm ci --omit=dev

Supply-chain security (pinned base image digests, SBOMs, signature
verification, dependency scanning) is a natural extension of "explicitly
declare".


================================================================================
5. FACTOR III - CONFIG
   "Store config in the environment"
================================================================================

WHAT IS "CONFIG"?
-----------------
Config is everything that is LIKELY TO VARY BETWEEN DEPLOYS:

  - resource handles to the database, cache, queue and other backing
    services (URLs, hostnames, ports);
  - credentials for external services (S3, Stripe, Twilio, OpenAI ...);
  - per-deploy values such as the canonical hostname of the deploy.

Config is NOT internal application configuration that is the same in every
deploy, e.g. route definitions, dependency-injection wiring, or how modules
are connected. That kind of "config" belongs in the code.

THE RULE
--------
Strict separation of config from code. Config varies substantially across
deploys; code does not. Constants in code that hold deploy-specific values
are a violation.

THE OPEN-SOURCE LITMUS TEST
---------------------------
The original document gives a simple test:

    "Could the codebase be made open source at any moment, without
     compromising any credentials?"

If the answer is no - if pushing the repo to a public GitHub would leak a
database password, an API key, or a production hostname - then config is
not properly separated from code.

This is a great practical test. Ask it in code review. Run secret scanners
(gitleaks, trufflehog, GitHub secret scanning) to enforce it.

WHY ENVIRONMENT VARIABLES?
--------------------------
The original text argues for env vars specifically, and against two other
common approaches:

  1. CONFIG FILES CHECKED INTO THE REPO (config/database.yml) - easy to
     accidentally commit secrets; scattered across formats and places.
  2. CONFIG FILES NOT CHECKED IN - better, but tend to be language- or
     framework-specific, scattered, and inconsistent.
  3. NAMED "ENVIRONMENTS" GROUPED IN CODE (development, test, staging,
     production) - does not scale. As the number of deploys grows
     (joes-staging, qa-2, pr-1234), you get a combinatorial explosion of
     named groups.

Env vars, by contrast:

  - are easy to change between deploys without changing code;
  - are unlikely to be checked into the repo by accident;
  - are a language- and OS-agnostic standard;
  - are GRANULAR: each var is independent and orthogonal; you do not group
    them into named "environments".

EXAMPLE
-------
    # Bad: hardcoded in code
    const db = new Pool({ host: "prod-db.internal", password: "hunter2" });

    # Good: read from the environment
    const db = new Pool({ connectionString: process.env.DATABASE_URL });

    # Locally: a .env file (git-ignored) loaded by a dev tool
    DATABASE_URL=postgres://localhost:5432/app_dev
    REDIS_URL=redis://localhost:6379
    PORT=3000

Good practice:
  - Validate config at startup and fail fast with a clear message if a
    required variable is missing (e.g. with a schema: zod, pydantic, envalid,
    @nestjs/config + Joi).
  - Commit a `.env.example` listing the variable NAMES, never values.
  - Never log full config at startup.

THE CRITIQUE: ENV VARS AND SECRETS
----------------------------------
Diogo Monica's well-known 2017 post "Why you shouldn't use ENV variables
for secret data" (see Resources) points out real problems with putting
SECRETS (not ordinary config) in env vars:

  - The environment is implicitly inherited by every child process the
    app spawns - including third-party tools - violating least privilege.
  - Crash reporters, error pages and debug endpoints often dump the whole
    environment into logs or bug trackers.
  - Env vars of a container can be read via tools like `docker inspect`
    and, on Linux, via /proc/<pid>/environ by sufficiently privileged
    users.
  - There is no access control, audit trail, or rotation built in.

Recommended alternatives for secrets:

  - A secret manager (HashiCorp Vault, AWS Secrets Manager, GCP Secret
    Manager, Azure Key Vault, Doppler, Infisical, 1Password ...).
  - Secrets mounted as files (Docker/Kubernetes secrets) on an in-memory
    filesystem, readable only by the app, ideally read once and cleared.
  - Short-lived, automatically rotated credentials obtained via workload
    identity, so there is no long-lived secret at all.

A balanced modern reading of Factor III:
  - Keep config OUT of code (the litmus test still holds).
  - Plain, non-sensitive config: env vars are fine.
  - Secrets: prefer a secret manager / mounted files / identity-based
    credentials, injected at runtime by the platform.


================================================================================
6. FACTOR IV - BACKING SERVICES
   "Treat backing services as attached resources"
================================================================================

THE RULE
--------
A BACKING SERVICE is any service the app consumes over the network as part
of its normal operation:

  - datastores (PostgreSQL, MySQL, MongoDB, CouchDB),
  - caches (Redis, Memcached),
  - message queues (RabbitMQ, Kafka, SQS, BullMQ on Redis),
  - SMTP / email services (Postmark, SendGrid, SES),
  - object storage (S3), search (Elasticsearch/OpenSearch),
  - third-party APIs (Stripe, Twilio, Google Maps, LLM APIs, New Relic).

The code of a twelve-factor app makes NO DISTINCTION between LOCAL and
THIRD-PARTY services. Both are ATTACHED RESOURCES, accessed via a URL or
other locator/credentials stored in config.

THE CONSEQUENCE
---------------
A deploy should be able to swap a local MySQL for Amazon RDS, or a local
SMTP server for a third-party mail service, WITHOUT ANY CODE CHANGE - only
a config change.

Resources can be attached and detached at will. If a database fails due to
a hardware issue, an operator can spin up a new database restored from
backup, detach the old one and attach the new one, again with no code
change.

    Production deploy
       |-- DATABASE_URL --> managed Postgres (resource #1)
       |-- REDIS_URL    --> managed Redis    (resource #2)
       |-- SMTP_URL     --> Postmark         (resource #3)
       '-- S3_BUCKET    --> AWS S3           (resource #4)

    Local dev deploy
       |-- DATABASE_URL --> Postgres in docker-compose
       |-- REDIS_URL    --> Redis in docker-compose
       |-- SMTP_URL     --> Mailpit / MailHog
       '-- S3_BUCKET    --> MinIO

WHY IT MATTERS
--------------
  - Loose coupling: the app does not know or care where a service runs.
  - Disaster recovery and migrations become config changes.
  - Each environment can use whatever instance makes sense.

PRACTICAL TIPS
--------------
  - Wrap each backing service behind a small client/adapter module.
  - Use the same protocol locally as in production (Postgres locally if
    Postgres in prod; see Factor X).
  - Handle the service being temporarily unavailable: timeouts, retries
    with backoff, circuit breakers, readiness checks.


================================================================================
7. FACTOR V - BUILD, RELEASE, RUN
   "Strictly separate build and run stages"
================================================================================

THE THREE STAGES
----------------
A codebase is transformed into a running deploy through three stages:

  1. BUILD STAGE
     Converts a code repo at a specific commit into an executable bundle
     called a BUILD. It fetches vendored dependencies (Factor II) and
     compiles binaries and assets.
       Output: a build artifact (today: usually a container image).

  2. RELEASE STAGE
     Takes the build and COMBINES it with the deploy's current CONFIG
     (Factor III). The result - build + config - is a RELEASE, ready for
     immediate execution.
       Output: release v42 = image@sha256:abc + config set #17

  3. RUN STAGE (also "runtime")
     Runs the app in the execution environment by launching some set of
     the app's processes (Factor VI) against a selected release.

    [ code @ commit ] --build--> [ BUILD ] --+ config--> [ RELEASE vN ] --run--> processes

THE RULES
---------
  - STRICT SEPARATION: It is impossible to make changes to code at
    runtime, since there is no way to propagate them back to the build
    stage. No SSH-ing into production to edit a file.
  - Every release has a UNIQUE ID (a timestamp such as
    2011-04-06-20:32:17 or an incrementing number such as v100).
  - Releases are an APPEND-ONLY LEDGER: a release cannot be mutated once
    created. Any change must create a new release.
  - ROLLBACK = point the run stage at a previous release.
  - The BUILD stage is initiated by developers when new code is deployed,
    and can be complex (errors surface in front of a developer).
    The RUN stage should have as few moving parts as possible, because it
    can happen automatically (server reboot, crashed process restarted by
    the process manager) at 4 a.m. when nobody is around.

MODERN MAPPING
--------------
  Build   -> CI pipeline runs tests and `docker build`, pushes
             registry/app:<git-sha>  (BUILD ONCE).
  Release -> A deployment record: Kubernetes Deployment / Helm release /
             ECS task definition revision / Heroku release vN, referencing
             the image digest plus ConfigMaps/Secrets.
  Run     -> The orchestrator starts containers from that release.

Key modern principle: BUILD ONCE, PROMOTE EVERYWHERE. The exact same image
that passed staging goes to production; only the config differs. Never
rebuild per environment.

ANTI-PATTERNS
-------------
  - Running `npm install` or `git pull` on the production server at boot.
  - Baking environment-specific config (API URLs, keys) into the image.
  - Using mutable tags such as `:latest` as the release identifier.
  - Hot-patching code inside a running container.


================================================================================
8. FACTOR VI - PROCESSES
   "Execute the app as one or more stateless processes"
================================================================================

THE RULE
--------
The app is executed in the execution environment as one or more PROCESSES.
In the simplest case, that is a single script run with `python app.py`. In
the complex case, it is many process types (web, worker, clock) each
running many instances.

Twelve-factor processes are STATELESS and SHARE-NOTHING. Any data that
needs to persist must be stored in a STATEFUL BACKING SERVICE, typically a
database (Factor IV).

THE MEMORY / FILESYSTEM CAVEAT
------------------------------
The process's memory space or filesystem CAN be used as a brief,
single-transaction cache. For example: downloading a large file, operating
on it, and storing the result in the database, all within one request or
job.

But the app NEVER ASSUMES that anything cached in memory or on disk will be
available on a future request or job:

  - with many processes running, a future request will likely be served
    by a DIFFERENT process;
  - even with one process, a restart (deploy, config change, the platform
    relocating the process) wipes all local state.

Asset packagers that compile assets at RUNTIME violate this; assets should
be compiled during the BUILD stage (Factor V).

STICKY SESSIONS
---------------
"Sticky sessions" means the load balancer pins a user to the same process
so that session data cached in that process's memory is available on the
next request. The original text is blunt: sticky sessions are a VIOLATION
of twelve-factor and should never be used or relied upon.

Why they are harmful:
  - If that process dies or is replaced during a deploy, the user's
    session is lost.
  - Load becomes uneven; you cannot freely scale in/out.
  - Autoscaling and rolling deploys fight against stickiness.

The fix: store session state in a datastore that offers time-expiration,
such as Memcached or Redis (or use signed, stateless tokens such as
JWTs/signed cookies where appropriate).

    BAD:  user -> LB (sticky) -> web.2  (session in web.2's RAM)
    GOOD: user -> LB (any)    -> web.N  -> Redis (session store)

THE WEBSOCKET EXCEPTION
-----------------------
WebSockets (and Server-Sent Events, long-polling, gRPC streams) are the
classic case where "stateless and share-nothing" seems to break: a
WebSocket is a LONG-LIVED CONNECTION bound to ONE particular process for
its entire lifetime. The process necessarily holds connection state.

How to reconcile it with Factor VI:

  1. Treat the connection itself as the only process-local state.
     Accept that the connection lives on one process, but do not store
     anything there that cannot be rebuilt.
  2. Keep durable/shared state in backing services. Room membership,
     presence, message history -> Redis/DB.
  3. Fan out across processes with a pub/sub backplane. If user A is
     connected to web.1 and user B to web.3, a message from A is
     published to Redis Pub/Sub (or NATS, Kafka ...), and every process
     delivers it to its own locally connected clients. (Socket.IO's Redis
     adapter is a well-known example.)
  4. Design clients to RECONNECT. When a process is killed (deploy,
     scale-in, crash), clients reconnect with backoff to any other process
     and re-sync state from the backing service.
  5. Load balancer affinity, when used for the WebSocket upgrade handshake
     or for polling-transport fallbacks, is a TRANSPORT CONCERN - not a
     place to keep application state.

In other words: the connection is sticky by nature, but your DATA must not
be. If killing any process loses nothing but connections that clients will
automatically re-establish, you still honour the spirit of Factor VI.


================================================================================
9. FACTOR VII - PORT BINDING
   "Export services via port binding"
================================================================================

THE RULE
--------
Web apps are sometimes executed inside a webserver container: PHP apps
inside Apache's mod_php, Java apps inside Tomcat. The twelve-factor app is
instead COMPLETELY SELF-CONTAINED and does not rely on runtime injection of
a webserver into the execution environment.

The web app EXPORTS HTTP AS A SERVICE BY BINDING TO A PORT, and listens for
requests coming in on that port.

    Local:       http://localhost:5000/
    Production:  a routing layer forwards public-hostname requests to the
                 port-bound web processes.

This is typically done by declaring a webserver library as a dependency
(Factor II):

    Python   -> Gunicorn / Uvicorn
    Ruby     -> Puma / Thin
    Java     -> Jetty / embedded Tomcat (Spring Boot)
    Node.js  -> built-in http module (Express, Fastify, NestJS)
    Go       -> net/http

The port number comes from CONFIG (usually the PORT env var), set by the
platform:

    const port = Number(process.env.PORT ?? 3000);
    app.listen(port, "0.0.0.0");

NOT JUST HTTP
-------------
Nearly any kind of server software can be run by a process binding to a
port and awaiting incoming requests: XMPP, the Redis protocol, gRPC, raw
TCP, etc.

APPS AS BACKING SERVICES
------------------------
The port-binding approach means one app can become the BACKING SERVICE for
another app (Factor IV) simply by providing its URL as a resource handle in
the consuming app's config. This is the foundation of service-oriented and
microservice architectures.

MODERN NOTE
-----------
Containers make this the default: EXPOSE a port, and the orchestrator maps
and routes to it. Bind to 0.0.0.0 (not 127.0.0.1) inside containers, or
the router cannot reach you. Health-check endpoints (/healthz, /readyz)
are served on the same or a dedicated port.


================================================================================
10. FACTOR VIII - CONCURRENCY
    "Scale out via the process model"
================================================================================

THE RULE
--------
In the twelve-factor app, PROCESSES ARE FIRST-CLASS CITIZENS. Inspired by
the Unix process model for running service daemons, the developer
architects the app to handle different workloads by assigning each type of
work to a PROCESS TYPE:

  - HTTP requests     -> web process type
  - long-running jobs -> worker process type
  - scheduled jobs    -> clock / scheduler process type

THE PROCESS FORMATION
---------------------
The array of process types and the number of processes of each type is
called the PROCESS FORMATION.

    Process type    Count
    ------------    -----
    web             4
    worker          8
    clock           1      (singleton!)

This does not exclude in-process concurrency. An individual process can
still use threads (JVM), an event loop (Node.js, Python async), or green
threads/goroutines internally. But a single machine/VM can only grow so
large (VERTICAL scaling), so the app must also be able to span multiple
processes on multiple machines (HORIZONTAL scaling).

The share-nothing, horizontally partitionable nature of twelve-factor
processes (Factor VI) is what makes adding more concurrency a simple,
reliable operation: "add more processes".

DO NOT DAEMONIZE
----------------
Twelve-factor processes should never daemonize or write PID files.
Instead, rely on the operating system's PROCESS MANAGER (originally
Upstart / systemd, Foreman in development, or a distributed process
manager on a cloud platform such as Heroku's dyno manager) to:

  - manage output streams,
  - respond to crashed processes (restart them),
  - handle user-initiated restarts and shutdowns.

MODERN MAPPING
--------------
  - Process type   -> Kubernetes Deployment (or ECS service) per type
  - Process count  -> replicas / desired count
  - Scaling        -> `kubectl scale`, Horizontal Pod Autoscaler (HPA),
                      KEDA (scale workers on queue depth)
  - Process manager-> kubelet / container runtime / systemd

Example: in a NestJS + BullMQ system, run the API as `web` and the queue
consumers as a separate `worker` process type; scale workers on queue
length independently of HTTP traffic.

SINGLETONS
----------
Some process types must run exactly once (a cron "clock", a leader). Keep
them tiny, or use leader election / distributed locks / an external
scheduler so that scaling the formation cannot accidentally run the same
scheduled job N times.


================================================================================
11. FACTOR IX - DISPOSABILITY
    "Maximize robustness with fast startup and graceful shutdown"
================================================================================

THE RULE
--------
Twelve-factor processes are DISPOSABLE: they can be started or stopped at a
moment's notice. This enables fast elastic scaling, rapid deployment of
code or config changes, and robustness of production deploys.

FAST STARTUP
------------
Processes should strive to minimize startup time. Ideally, a process takes
a few seconds from the launch command until it is up and ready to receive
requests or jobs.

Fast startup gives:
  - agility for releases and scaling up;
  - robustness: the process manager can move processes to new physical
    machines easily when warranted.

Tips: avoid heavy work at boot (no migrations on every boot, no asset
compilation, no giant cache warm-ups); lazy-load where possible; make
readiness explicit with a readiness probe.

GRACEFUL SHUTDOWN ON SIGTERM
----------------------------
Processes shut down gracefully when they receive a SIGTERM from the
process manager.

  WEB PROCESS:
    1. Stop listening on the service port (refuse new requests).
    2. Allow in-flight requests to finish.
    3. Close DB pools / connections.
    4. Exit.
  Implicit in this model: HTTP requests are short (no more than a few
  seconds), or, for long polling, the client reconnects seamlessly when
  the connection is lost.

  WORKER PROCESS:
    Return the current job to the work queue. For example, on RabbitMQ the
    worker can send a NACK; on Beanstalkd the job returns to the queue
    automatically when the worker disconnects. Lock-based systems (e.g.
    Delayed Job) need to release their lock on the job record.
  Implicit in this model: all jobs are REENTRANT, typically achieved by
  wrapping results in a transaction, or making the operation IDEMPOTENT.

ROBUSTNESS AGAINST SUDDEN DEATH
-------------------------------
Processes should also be robust against SUDDEN death - hardware failure,
OOM kill, SIGKILL - where there is no graceful shutdown at all. The
recommended approach is a robust queueing backend that returns jobs to the
queue when clients disconnect or time out. Either way, a twelve-factor app
is architected to handle unexpected, non-graceful terminations.

The original text points to "crash-only design" as the logical conclusion
of this idea: if the system can recover from a crash correctly, then a
crash is just a fast way to stop.

KUBERNETES SPECIFICS
--------------------
  - On pod termination Kubernetes sends SIGTERM, waits
    `terminationGracePeriodSeconds` (default 30 s), then sends SIGKILL.
  - Make sure your app is PID 1 or that the init process forwards
    signals: use exec-form `CMD ["node","dist/main.js"]`, not
    `CMD npm start` through a shell that swallows SIGTERM (or use tini /
    dumb-init).
  - A short `preStop` sleep can help the load balancer remove the pod from
    rotation before the app stops accepting connections.
  - Mark readiness as failing as soon as shutdown begins.


================================================================================
12. FACTOR X - DEV/PROD PARITY
    "Keep development, staging, and production as similar as possible"
================================================================================

THE THREE GAPS
--------------
Historically there have been substantial gaps between development (a
developer making live edits to a local deploy) and production (a running
deploy accessed by end users). The original document names three gaps:

  1. THE TIME GAP
     A developer may work on code that takes days, weeks, or even months
     to go into production.
  2. THE PERSONNEL GAP
     Developers write code, ops engineers deploy it.
  3. THE TOOLS GAP
     Developers may use a stack like Nginx, SQLite and macOS, while
     production uses Apache, MySQL and Linux.

The twelve-factor app is designed for CONTINUOUS DEPLOYMENT by keeping
these gaps small:

    Gap           Traditional app          Twelve-factor app
    -----------   ----------------------   ---------------------------------
    Time          Weeks between deploys    Hours (or minutes)
    Personnel     Devs write, ops deploy   Devs who wrote it are closely
                                           involved in deploying and watching
                                           it in production
    Tools         Divergent                As similar as possible

BACKING SERVICE PARITY
----------------------
Backing services are where developers are most tempted to diverge: SQLite
locally and PostgreSQL in production, an in-memory cache locally and Redis
in production. Adapters (ORMs, cache abstraction layers) make it LOOK safe.

The twelve-factor developer RESISTS this urge. Tiny incompatibilities
(SQL dialect differences, transaction semantics, JSON operators, locking,
case sensitivity, eviction behaviour) crop up and cause code that worked
and passed tests in development or staging to fail in production. Those
failures create friction that disincentivizes continuous deployment.

Lightweight local services are less compelling than they once were. Modern
backing services (Postgres, Redis, RabbitMQ ...) are easy to install and
run locally, especially with containers:

    # docker-compose.yml (dev)
    services:
      db:    { image: postgres:16, ports: ["5432:5432"] }
      redis: { image: redis:7,     ports: ["6379:6379"] }

Adapters are still useful for portability to NEW backing services, but
all deploys (dev, staging, prod) should use the SAME TYPE AND VERSION of
each backing service.

MODERN NOTE
-----------
  - Containers + docker-compose / devcontainers / Tilt / Skaffold shrink
    the tools gap dramatically.
  - Testcontainers lets tests run against real Postgres/Redis.
  - Ephemeral preview environments per pull request shrink the time gap.
  - "You build it, you run it" / DevOps culture shrinks the personnel gap.
  - Infrastructure as code (Terraform, Pulumi, Helm) keeps staging and
    production structurally identical.


================================================================================
13. FACTOR XI - LOGS
    "Treat logs as event streams"
================================================================================

WHAT LOGS ARE
-------------
Logs provide visibility into the behaviour of a running app. In server
environments they are commonly written to a file on disk ("logfile"), but
that is only an output format.

LOGS ARE THE STREAM OF AGGREGATED, TIME-ORDERED EVENTS collected from the
output streams of all running processes and backing services. In raw form
they are typically text with one event per line (backtraces may span
lines). Logs have no fixed beginning or end; they flow continuously as
long as the app is operating.

THE RULE
--------
A twelve-factor app NEVER CONCERNS ITSELF WITH ROUTING OR STORAGE OF ITS
OUTPUT STREAM. It should not attempt to write to or manage logfiles.
Instead, each running process writes its event stream, UNBUFFERED, to
STDOUT.

  - In local development, the developer views the stream in the terminal
    in the foreground.
  - In staging or production, each process's stream is CAPTURED BY THE
    EXECUTION ENVIRONMENT, collated together with all other streams from
    the app, and routed to one or more final destinations for viewing and
    long-term archival. These destinations are not visible to or
    configurable by the app; they are completely managed by the execution
    environment.

Open-source log routers named in the original: Logplex (Heroku) and
Fluentd. Today: Fluent Bit, Vector, Promtail/Alloy, the OpenTelemetry
Collector, CloudWatch agents, etc.

WHAT YOU CAN DO WITH THE STREAM
-------------------------------
The event stream can be routed to a file, watched via a realtime `tail`, or
- most significantly - sent to a log indexing and analysis system such as
Splunk, or a general-purpose data warehouse such as Hadoop/Hive (in 2011
terms; today: Elasticsearch/OpenSearch, Loki, Datadog, BigQuery ...). These
systems allow:

  - finding specific events in the past;
  - large-scale graphing of trends (e.g. requests per minute);
  - active alerting according to user-defined heuristics (e.g. alert when
    errors per minute exceed a threshold).

MODERN PRACTICE
---------------
  - STRUCTURED LOGS: emit one JSON object per line, with level, timestamp,
    message, request ID / trace ID, user/tenant ID, etc.
      {"level":"info","ts":"2026-10-05T10:15:00Z","msg":"order created",
       "orderId":"o_123","traceId":"4bf92f..."}
  - Use a fast logger (pino, winston, structlog, zap, Serilog) configured
    to write to stdout.
  - Correlate logs with TRACES and METRICS (OpenTelemetry) - the
    "observability" extension the community revision discusses.
  - Never log secrets or full request bodies containing personal data.
  - Do not log to files inside containers; the file disappears with the
    container and fills the disk.


================================================================================
14. FACTOR XII - ADMIN PROCESSES
    "Run admin/management tasks as one-off processes"
================================================================================

THE RULE
--------
The process formation (Factor VIII) is the set of processes used to do the
app's regular business (serving web requests, processing jobs). Separately,
developers often need to do one-off administrative or maintenance tasks,
such as:

  - running database migrations
    (`rake db:migrate`, `python manage.py migrate`, `npx prisma migrate
    deploy`, `npm run typeorm migration:run`);
  - running a console / REPL to execute arbitrary code or inspect models
    against the live database (`rails console`, `python manage.py shell`);
  - running one-time scripts committed to the app's repo
    (`php scripts/fix_bad_records.php`).

One-off admin processes should be run IN AN IDENTICAL ENVIRONMENT as the
regular long-running processes of the app:

  - they run against the SAME RELEASE (same code, same config);
  - admin code MUST SHIP WITH APPLICATION CODE to avoid synchronization
    issues;
  - the SAME DEPENDENCY ISOLATION techniques are used
    (`bundle exec rake db:migrate`, the virtualenv's python, etc.).

Twelve-factor strongly favours languages that provide a REPL shell out of
the box and that make it easy to run one-off scripts.

  - Local: invoke the command directly in the app's checkout.
  - Production: use SSH or another remote command execution mechanism
    provided by the deploy's execution environment to run the process
    (e.g. `heroku run`, `kubectl run/exec`, `kubectl create job`,
    `aws ecs run-task`, `fly ssh console`).

MODERN NOTE
-----------
  - Migrations as a RELEASE PHASE / pre-deploy job (Heroku `release:`
    process type, Helm pre-upgrade hook, Kubernetes Job, Argo CD sync
    hook) so they run exactly once per release, before new web
    processes take traffic.
  - Write migrations to be BACKWARD COMPATIBLE (expand -> migrate ->
    contract), because during a rolling deploy old and new code run
    against the same schema at the same time.
  - Prefer committed, reviewed scripts over ad-hoc console sessions in
    production; audit who ran what.


================================================================================
15. THE REAL PROCFILE FORMATION: BOOT, ONE-OFF JOB, GRACEFUL SHUTDOWN
================================================================================

This section ties the factors together the way the closing demo does: a
small task app (see "tasker" in Resources) run as a real formation. The
example below is illustrative - adapt names and commands to the actual
repository.

THE PROCFILE
------------
A Procfile is a plain-text file in the repo root that declares process
types and the command that starts each one. It was popularized by Heroku
and by Foreman (and clones like Honcho, Overmind, node-foreman), and the
same idea maps directly to docker-compose services or Kubernetes
Deployments/Jobs.

    # Procfile
    release: node dist/scripts/migrate.js
    web:     node dist/main.js
    worker:  node dist/worker.js

  - `web`     -> binds to $PORT and serves HTTP           (Factors VII, VIII)
  - `worker`  -> consumes the job queue from Redis         (Factor VIII)
  - `release` -> one-off admin process run per release     (Factor XII)

And the environment (Factor III) - local example, git-ignored:

    # .env
    PORT=5000
    DATABASE_URL=postgres://postgres:postgres@localhost:5432/tasker
    REDIS_URL=redis://localhost:6379
    LOG_LEVEL=info

STEP 1 - BUILD ONCE (Factor V, II)
----------------------------------
    $ npm ci                         # exact deps from the lockfile
    $ npm run build                  # compile TS -> dist/
    # or, in CI:
    $ docker build -t registry/tasker:$(git rev-parse --short HEAD) .

STEP 2 - RUN THE ONE-OFF JOB (Factor XII)
-----------------------------------------
Run migrations with the SAME build and SAME config as the app, as a
separate, short-lived process:

    $ foreman run node dist/scripts/migrate.js
    # Heroku:     heroku run node dist/scripts/migrate.js
    # Kubernetes: a Job using the same image + same ConfigMap/Secret

It starts, does its work, writes progress to stdout, and exits 0. It is
not part of the long-running formation.

STEP 3 - BOOT THE FORMATION (Factors VI, VII, VIII, XI)
-------------------------------------------------------
    $ foreman start -m web=2,worker=3

    10:00:01 web.1    | listening on port 5000
    10:00:01 web.2    | listening on port 5001
    10:00:01 worker.1 | waiting for jobs on queue "tasks"
    10:00:01 worker.2 | waiting for jobs on queue "tasks"
    10:00:01 worker.3 | waiting for jobs on queue "tasks"

What to notice:
  - Boot takes about a second (disposability: fast startup).
  - Every process logs to stdout; the process manager interleaves and
    prefixes the streams (logs as event streams). The app does not open a
    single log file.
  - Each web process got its own port from the environment (port binding).
  - Scaling is "change the numbers" (concurrency via the process model).
  - Any web process can serve any request because sessions/state live in
    Postgres/Redis (stateless processes).

STEP 4 - SEND TRAFFIC AND JOBS
------------------------------
    $ curl -X POST localhost:5000/tasks -d '{"title":"write notes"}'
    web.1    | {"level":"info","msg":"task created","id":"t_1"}
    worker.2 | {"level":"info","msg":"processing","job":"notify","id":"t_1"}

The web process only enqueues; the worker does the slow part. If more
jobs pile up, scale `worker` without touching `web`.

STEP 5 - GRACEFUL SHUTDOWN (Factor IX)
--------------------------------------
Press Ctrl-C (or the platform sends SIGTERM during a deploy/scale-in):

    ^C SIGINT received - sending SIGTERM to all processes
    web.1    | SIGTERM: closing server, draining 1 in-flight request
    web.1    | closed DB pool, bye
    worker.2 | SIGTERM: finishing current job t_1 (or returning it to queue)
    worker.2 | closed queue connection, bye
    ...      | exited with code 0

A minimal Node/TypeScript shutdown handler:

    const server = app.listen(Number(process.env.PORT ?? 3000));

    async function shutdown(signal: string) {
      console.log(JSON.stringify({ level: "info", msg: `${signal}: draining` }));
      server.close(async () => {          // stop accepting, finish in-flight
        await db.end();                   // release backing services
        await redis.quit();
        process.exit(0);
      });
      setTimeout(() => process.exit(1), 25_000).unref(); // hard stop < grace
    }
    process.on("SIGTERM", () => shutdown("SIGTERM"));
    process.on("SIGINT",  () => shutdown("SIGINT"));

For a worker (BullMQ example): call `await worker.close()`, which stops
taking new jobs and waits for the active one to finish; jobs that are
interrupted by a hard kill are detected as stalled and retried - which is
why job handlers must be IDEMPOTENT.

STEP 6 - KILL ONE PROCESS HARD (Factor IX, VI)
----------------------------------------------
    $ kill -9 <pid of worker.2>

The queue notices the lost lock and the job is retried by another worker.
Nothing is lost, because no state lived only in worker.2. This is the
"robust against sudden death" property in action.

THE SAME FORMATION IN KUBERNETES TERMS
--------------------------------------
    Procfile line     Kubernetes object
    ---------------   ----------------------------------------------------
    release: ...      Job (or Helm pre-upgrade hook) using image:<sha>
    web: ...          Deployment (replicas: 2) + Service + Ingress,
                      readiness/liveness probes, containerPort from config
    worker: ...       Deployment (replicas: 3), scaled by HPA/KEDA
    .env              ConfigMap (plain config) + Secret / external secrets
    stdout            Collected by node log agent -> Loki/ELK/Datadog
    SIGTERM           Sent on pod deletion; terminationGracePeriodSeconds


================================================================================
16. QUICK-REFERENCE CHEAT SHEET
================================================================================

  #    Factor               One-line rule                     Modern shorthand
  ---- -------------------- --------------------------------- -------------------
  I    Codebase             One repo per app, many deploys    Git repo -> image
  II   Dependencies         Declare + isolate everything      Lockfile + Dockerfile
  III  Config               Config in env, not code           Env + secret manager
  IV   Backing services     Attached resources via URLs       Swap by config only
  V    Build, release, run  Strictly separated stages         Build once, promote
  VI   Processes            Stateless, share-nothing          State in Redis/DB
  VII  Port binding         Self-contained, bind to $PORT     EXPOSE + Service
  VIII Concurrency          Scale out by process type         Replicas / HPA
  IX   Disposability        Fast boot, graceful SIGTERM       Probes + grace period
  X    Dev/prod parity      Small time/people/tools gaps      Compose, previews
  XI   Logs                 Unbuffered stream to stdout       JSON logs + OTel
  XII  Admin processes      One-offs in same release/env      Jobs / release phase

SELF-AUDIT QUESTIONS
--------------------
  [ ] Could I open-source this repo right now without leaking anything?
  [ ] Can a new dev go from `git clone` to running app with 1-2 commands?
  [ ] Is the production artifact the exact same one that passed staging?
  [ ] Can I kill -9 any process and lose nothing but in-flight work that
      is automatically retried?
  [ ] Can I double the web or worker count with one command?
  [ ] Do dev and prod use the same database engine and major version?
  [ ] Does the app write logs only to stdout?
  [ ] Are migrations run once per release, from code in the repo?
  [ ] Are secrets kept out of plain env vars where it matters?


================================================================================
17. RESOURCES
================================================================================

  * The Twelve-Factor App (original text)
      https://12factor.net

  * The open-source project and community revision
      https://github.com/twelve-factor/twelve-factor

  * Open-sourcing announcement (12 Nov 2024)
      https://12factor.net/blog/open-source-announcement

  * Vish Abrams at KubeCon NA 2024 - Sponsored Keynote:
    "The Twelve-Factor App, ..."
      https://www.youtube.com/watch?v=_V_s4VeJvjU

  * Diogo Monica - "Why you shouldn't use ENV variables for secret data"
      https://blog.diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/

  * The task app used in the closing demo ("tasker")
      https://github.com/sriniously/tasker

  * Source video timestamps
      0:00     The twelve-factor app, and why it runs our lives
      5:07     Where it came from: Heroku, 2011, software erosion
      12:05    Who owns it now: open-sourced, and being revised
      14:23    I.    Codebase
      20:58    II.   Dependencies
      28:10    III.  Config
      38:06    IV.   Backing services
      43:06    V.    Build, release, run
      50:16    VI.   Processes
      56:39    VII.  Port binding
      1:02:09  VIII. Concurrency
      1:09:59  IX.   Disposability
      1:16:48  X.    Dev/prod parity
      1:24:04  XI.   Logs
      1:29:28  XII.  Admin processes, and the live formation

================================================================================
                                   END
================================================================================
