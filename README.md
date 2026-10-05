# backend-skills

Backend engineering guides packaged as [Agent Skills](https://code.claude.com/docs/en/skills) for Claude.

Each guide is a separate skill. At the start of a session the agent only sees each skill's one-line `description`, about 100 tokens per skill. It loads a skill's full content only when the task needs it: for example, `webhooks` when you ask it to verify a Stripe signature, or `caching` when you add Redis.

## Skills

| Skill | Covers |
|---|---|
| `12-factor-app` | The Twelve-Factor App methodology in depth |
| `api-design` | RESTful API design guidelines |
| `backend-rules` | Condensed, rule-style checklist for backend code covering the whole stack |
| `background-jobs` | Background jobs and asynchronous task processing |
| `caching` | Cache eviction policies (LRU, LFU, TTL) and practical caching use cases |
| `concurrency-parallelism` | Concurrency vs parallelism for backend services |
| `error-handling-config-shutdown` | Error handling and fault-tolerant systems |
| `full-text-search` | Full-text search internals and trade-offs |
| `http` | HTTP protocol deep dive for backend engineers |
| `layered-architecture` | Server request lifecycle and layered architecture |
| `logging-observability` | Logging, monitoring and observability |
| `performance` | Backend performance fundamentals |
| `routing` | Backend routing |
| `security` | Backend security |
| `serialization` | Data serialization and deserialization |
| `validation-transformation` | Request validation and transformation pipeline |
| `webhooks` | Webhooks end to end, as receiver and sender |

## How loading works

```
skills/<name>/SKILL.md      # frontmatter (name + description) and content
skills/<name>/reference.md  # large guides only
```

- **Small guides** (under ~500 lines) have all their content in `SKILL.md`, which is loaded when the skill triggers.
- **Large guides** have a short `SKILL.md` that lists the sections. The full text is in `reference.md`, and the agent greps it and reads only the section it needs. Even after a skill triggers, it doesn't pull in the whole file.

## Install

### As a Claude Code plugin (recommended)

```
/plugin marketplace add khan-armaan/backend-skills
/plugin install backend-skills@backend-skills
```

Skills are namespaced as `backend-skills:<name>`, e.g. `/backend-skills:security`.

### Copy into a project or your user directory

```bash
git clone https://github.com/khan-armaan/backend-skills
# one project:
cp -r backend-skills/skills/* your-project/.claude/skills/
# every project on this machine:
cp -r backend-skills/skills/* ~/.claude/skills/
```

### claude.ai / Claude Desktop

Zip a single skill folder (e.g. `skills/caching`) and upload it under **Settings → Capabilities → Skills**.

## Adding a new skill

1. Create `skills/<name>/SKILL.md` with frontmatter:
   ```markdown
   ---
   name: <name>
   description: "What it covers. Use when <the situations that should trigger it>."
   ---
   ```
2. Write the description carefully. It's the only part the agent sees before deciding to load the skill.
3. If the content runs past ~500 lines, put it in `reference.md` and keep `SKILL.md` as a section index.
