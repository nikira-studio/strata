# Strata

**A sovereign, local-first memory architecture for AI agent systems**

**Status:** v1.0  
**Primary goals:** Deterministic, portable, inspectable, high-recall, low-context-pollution memory without excessive architectural complexity.

**Primary schema:** Two-plane model (`memory_records` + `memory_evidence`). See §22 for schema versioning guidance.

---

## 1. Purpose

This document defines the recommended memory architecture for a sovereign, local-first AI agent system.

The intent is to build a memory subsystem that is:

- reliable across long time horizons,
- inspectable by humans,
- efficient in token usage,
- capable of exact fact recall and narrative recall,
- robust to changing facts,
- portable across environments,
- resilient when optional components (embeddings, vector stores) are unavailable,
- privacy-aware by design,
- simple enough to maintain without a large platform team.

This system is designed as a replacement for naive "append everything to logs and retrieve later" memory patterns.

---

## 2. Problem Statement

Most AI memory systems fail in one or more of these ways:

1. They treat all memory as the same kind of data.
2. They stuff too much irrelevant content into prompt context.
3. They cannot distinguish current facts from outdated facts.
4. They cannot show clear provenance for remembered information.
5. They are difficult to audit or repair.
6. They rely on opaque third-party cloud memory layers.
7. They have no path from raw session content to durable structured knowledge.
8. They fail silently when retrieval components (vector stores) are unavailable.

Large context windows do **not** solve these problems by themselves. Long-context models still degrade when relevant information is buried inside long prompts, and multi-session memory remains difficult without structured external memory.

Strata addresses this by combining:

- structured exact memory,
- an evidence archive with an explicit promotion pipeline,
- narrative/document memory,
- hybrid retrieval with graceful degradation,
- explicit memory governance and privacy controls.

---

## 3. Design Principles

### 3.1 Local-first
Memory should operate fully on local hardware unless a deployer intentionally adds external systems.

### 3.2 Human-readable where practical
Important memory should be inspectable and repairable by humans without specialized tooling.

### 3.3 Exact facts are not fuzzy memory
Exact operational data should not rely solely on embeddings or semantic similarity.

### 3.4 Retrieval must be selective
The agent should receive a small evidence packet, not a raw dump of everything that might be relevant.

### 3.5 Memory must be governable
The system must support adding, updating, superseding, retracting, and auditing memory.

### 3.6 Portability matters
Memory should be easy to back up, move, restore, and inspect.

### 3.7 Retrieval must degrade gracefully
Semantic/vector retrieval is powerful but optional. Lexical FTS retrieval must always be available as a fallback. Memory must never become unavailable because an embedding backend is down.

### 3.8 Evidence first, records second
Raw session content (conversation turns, tool outputs, documents) is captured as evidence before being promoted to durable structured records. This decouples ingestion speed from analysis quality.

### 3.9 Privacy by default
Memory is private unless explicitly marked as shared. User-scoped records must not be readable by other users or by shared-scope queries without explicit permission.

---

## 4. System Overview

Strata uses four memory layers arranged from the most protected and stable to the most flexible:

### 4.1 Constitutional Memory
Small, stable, protected instructions and identity.

Examples:
- mission and purpose,
- core behavioral rules,
- tool restrictions,
- immutable or highly controlled operator directives.

**Storage:** Markdown and/or protected database records. Rarely changed; versioned and reviewed before modification.

### 4.2 Episodic Memory
A timeline of actions, decisions, outcomes, and notable events.

Examples:
- the user asked for a review,
- the agent ran a tool,
- a retrieval result was corrected,
- a task was completed,
- a preference changed.

**Storage:** `memory_evidence` table (append-only evidence archive).

### 4.3 Semantic Memory
Structured knowledge that the system can recall and reuse.

Examples:
- a hostname,
- a VLAN mapping,
- a project preference,
- a canonical relationship between systems,
- stable facts derived from documents or interactions.

**Storage:** `memory_records` table (semantic plane), promoted from evidence or written directly.

### 4.4 Narrative Memory
Human-readable source material and curated knowledge.

Examples:
- markdown notes,
- procedures,
- architecture documents,
- operator-authored summaries,
- meeting notes,
- project logs.

**Storage:** Markdown files on disk, indexed into FTS for retrieval.

---

## 5. Architecture

### 5.1 Backbone: SQLite
SQLite is the primary operational memory store.

Why:
- portable,
- embeddable,
- well understood,
- supports FTS5,
- easy to back up,
- suitable for single-user and small-team sovereign systems.

### 5.2 Two-Plane Data Model

Strata organizes durable memory into two operational planes:

**Plane 1: Semantic Plane (`memory_records`)**  
Holds normalized, durable knowledge: facts, decisions, preferences, profiles. Every write goes through the admission gate and optionally through the promotion pipeline. Records have explicit lifecycle states (active, superseded, retracted) and a supersession chain for tracking changes over time.

**Plane 2: Evidence Archive (`memory_evidence`)**  
Holds raw episodic material: conversation turns, events, tool outputs, documents, corrections. Evidence is not immediately trusted — it is captured as `pending` and moves through a promotion pipeline that decides whether to promote it to a durable record. Evidence is append-only; original evidence rows are never mutated.

These two planes are linked by `memory_links`, which records which evidence items supported the creation of which records.

### 5.3 Narrative Knowledge Base: Markdown
Markdown files provide human-readable narrative memory. Use markdown for runbooks, summaries, project notes, identity documents, architecture descriptions, and operator-authored explanations.

Markdown is not the only source of truth. It is one of the canonical memory substrates.

### 5.4 Retrieval Layer
The retrieval layer combines:
- SQLite exact queries,
- FTS5 lexical retrieval (always available),
- optional vector retrieval (when an embedding backend is healthy).

The retrieval mode is determined at query time based on embedding backend health. When the backend is unavailable or degraded, retrieval falls back to FTS-only mode automatically (§14.1).

### 5.5 Governance Layer
All durable memory writes pass through an explicit admission and update policy. This prevents junk accumulation, silent fact overwrites, uncontrolled memory drift, and false confidence in stale facts.

---

## 5a. Deployment Models

### 5a.1 Canonical Deployment: One Shared Instance for All Agents

The intended deployment of Strata is **not** one instance per agent. It is one shared instance that all agents and all users in a system write to and read from, with scope providing isolation between them.

Running a separate Strata instance per agent means:
- agents cannot share knowledge they should share (project facts, user preferences, household context),
- there is no single place to audit what the system as a whole knows,
- operational overhead multiplies — N databases to back up, monitor, and maintain,
- when preferences or facts change, they must be updated in every instance separately.

A single shared instance with scope-based isolation solves all of these. Each agent and each user gets isolated memory by default. Sharing is an explicit, controlled opt-in.

### 5a.2 Service Layer

For single-agent or single-process deployments, Strata can be accessed directly via SQLite. For multi-agent deployments, a thin HTTP service wrapper is the correct architecture.

**Why a service layer:**
- SQLite serializes concurrent writes; a service layer queues them cleanly
- Agents written in different languages or running in different processes need a shared access point
- Authentication, rate limiting, and scope enforcement become centralized

**Structure:**
A minimal Strata service exposes the agent tools as HTTP endpoints. The service holds the single SQLite file and is the only process that writes to it directly.

```
POST /memory/search
POST /memory/get
POST /memory/write
POST /memory/retract
POST /memory/confirm
POST /memory/reflect
POST /memory/compact
POST /memory/sync
POST /memory/session/start
POST /memory/session/end
GET  /health
GET  /spec
```

Every request MUST include `agent_id`. Requests on behalf of a specific user MUST also include `user_id`. The service resolves these into scope values automatically.

**Hosting:** The Strata service can be hosted in any way that makes it reachable to all agents — a background process on a local machine, a container, a VM on a private network, or any other always-on deployment method. The choice of hosting is left to the operator. What matters is that the service exposes a consistent HTTP interface and that the SQLite file it owns is not written to by any other process. All conforming implementations of Strata are interchangeable at the API level.

### 5a.3 Scope Resolution from Agent Identity

When a request arrives, the service resolves `agent_id` and `user_id` into a concrete set of scopes automatically. The caller does not need to construct scope strings manually.

**Default read scope** (what the agent can see):
```
[agent:<agent_id>, user:<user_id>, shared]
```

**Default write scope** (where the agent writes unless specified):
```
agent:<agent_id>
```

To write a user-level preference (visible to all of that user's agents):
```json
{ "agent_id": "assistant-1", "user_id": "alice", "scope": "user:alice", ... }
```

To write a shared fact (visible to all agents and all users):
```json
{ "agent_id": "assistant-1", "scope": "shared", ... }
```

The service enforces that a caller can only write to scopes it is authorized for. An agent cannot elevate its write target to a user scope it was not given access to, and MUST NOT write to another user's scope.

**Shared scope write restriction:** By default, no agent is authorized to write to `shared` scope. An attempt to write to `shared` scope without explicit operator authorization MUST be rejected with `403 Forbidden` (`SCOPE_NOT_AUTHORIZED`). The operator grants shared-scope write access by one of:

1. **Explicit allowlist:** Configure the deployment with a list of `agent_id` values permitted to write to `shared` scope. Calls from any other agent receive `403`.
2. **Operator-in-the-loop:** Route the write through operator review — the record is written with `record_status = "held"` and the `memory_write` response returns `status = "held_for_review"`. The record is not active and is not returned by default search until approved through `memory_confirm`.

This restriction prevents rogue, compromised, or misconfigured agents from polluting the shared memory tier.

**User identity — who is responsible for knowing the user's name:**

The Strata service does not ask users for their name. It does not have a sign-up flow. The `user_id` is the agent's responsibility: the agent knows (or asks) who it is serving and passes that identifier on every call.

For single-user setups, set the `DEFAULT_USER_ID` environment variable (e.g., `DEFAULT_USER_ID=alice`). The service applies this value automatically whenever a request omits `user_id`. The agent never needs to ask.

For multi-user setups, the agent must determine the user's identity before calling Strata. On first interaction with a new user, the agent SHOULD ask for a stable identifier — a first name, a username, or any consistent string — and store it locally for all future calls. Strata does not care what the value is. What matters is that the same value is used consistently across sessions and agents for the same person.

### 5a.4 Multi-Agent Scenarios

Multiple specialized agents — a research agent, a coding agent, a scheduling agent — can share a single Strata deployment. Each agent has its own memory partition. Knowledge that should be shared (project facts, system-wide preferences, decisions that affect all agents) lives in `shared` scope.

**Example — two agents serving the same user:**
- `agent:assistant-1` writes its working notes to `agent:assistant-1` scope
- `agent:assistant-2` writes its working notes to `agent:assistant-2` scope
- A project fact that both need lives in `user:alice` scope (visible to all agents serving Alice)
- A fact that applies to all users lives in `shared` scope

Neither agent can see the other's agent-scoped memories by default. If cross-agent sharing is needed for a specific piece of information, it is written to `user:<id>` scope, which all of that user's agents can read.

### 5a.5 Multi-User Isolation

Multiple users can share a single Strata instance. This is common in household deployments, small teams, or any system where several people each have their own AI agents but the operator wants a single infrastructure footprint.

**Isolation guarantee:** User-scoped records are a hard wall. A query carrying `user_id: alice` MUST NEVER return records scoped to `user:bob`. This is enforced at the query layer, not the application layer. See §21.1.

**User scope is inherited by all agents serving that user.** If Person A's agent runs a search, it automatically includes `user:person-a` in its read scope. If Person B's agent runs the same search, it automatically includes `user:person-b`. They cannot see each other's results.

**Shared scope is available to all.** The `shared` scope is intended for facts that are legitimately common knowledge across all users — system configuration, general-purpose facts, operator-defined rules. The PII gate (§21.2) MUST be applied before any record reaches `shared` scope to prevent one user's personal information from leaking into the shared tier.

### 5a.6 Scope Reference

| Scope | Readable by | Writable by | Isolation |
|---|---|---|---|
| `shared` | All agents, all users | Operator-authorized agents only | PII gate required on write |
| `user:<id>` | All agents serving that user | Agents serving that user (with explicit intent) | Hard wall between users |
| `agent:<name>` | That agent only | That agent only | Default write target |
| `tenant:<id>` | All agents/users within that tenant | Agents within that tenant | Hard wall between tenants |

The scope field on every record is the enforcement boundary. Any query, retrieval, or injection that does not apply scope filtering is a privacy violation (§21.1).

### 5a.7 Session Context *(Optional)*

An agent can declare a working context that is applied as an automatic domain/topic pre-filter on all queries. This is useful when an agent operates in distinct named contexts — different projects, different subject domains — and should not surface memories from unrelated contexts unless explicitly asked.

**Session-level declaration:** Set once at session start, applied to all queries for that session.

```json
POST /memory/session/start
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "session_context": "projects/website1"
}
```

The service maps this to a domain/topic filter: `domain = 'projects'` AND `topic = 'website1'`. All subsequent `memory_search` and `memory_get` calls in the session apply this filter automatically.

**Per-call override:** Pass `session_context` on any individual call to use a different context for just that call.

```json
POST /memory/search
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "session_context": "projects/website2",
  "query": "deployment configuration"
}
```

**Clearing context:** Pass `"session_context": null` to search across all contexts within the agent's allowed scope.

**This is a filter, not an isolation boundary.** Session context controls what the agent sees by default in a given session — it does not prevent memories from other contexts from existing in the database, and an authorized agent can always query without a context filter to see everything within its scope. For hard isolation between contexts (where context A memories must never appear in context B under any circumstances), use separate agent scopes instead.

**Context string format:** `"domain/topic"` — slash-separated, matching the `domain` and `topic` field values on records. Single-segment strings (e.g., `"health"`) match on domain only, leaving topic unrestricted.

### 5a.8 Domain and Topic Naming *(Shared Deployments)*

`domain` and `topic` are freeform strings — Strata does not enforce a fixed taxonomy. In single-agent deployments this is fine. In multi-agent or multi-user shared deployments, inconsistent naming fragments retrieval: one agent writing `domain: "code"` and another writing `domain: "coding"` for the same subject will produce split results.

**Guidance for shared deployments:** Operators SHOULD define a small, stable domain vocabulary and share it with all agents at session start (or embed it in a shared-scope record with `memory_class: "system_config"`). Agents SHOULD write `domain` from this vocabulary rather than inventing new values.

**Starter vocabulary (adapt as needed):**

| Domain | Intended use |
|---|---|
| `projects` | Named projects, product work, ongoing initiatives |
| `identity` | User profile, preferences, persona records |
| `coding` | Code style, architecture decisions, technical constraints |
| `research` | Facts, references, background knowledge |
| `tasks` | To-dos, reminders, scheduled items |
| `health` | Personal wellness, medical context |
| `finance` | Budget, spending, financial facts |
| `system` | Operator configuration, rules, deployment metadata |

`topic` is the sub-category within a domain (e.g., `domain: "projects"`, `topic: "website-redesign"`). Consistent naming matters most at the domain level — topics can be freeform within a domain without significantly harming retrieval.

`slot_key` identifies a singleton preference within a domain/topic combination (e.g., `domain: "coding"`, `topic: "style"`, `slot_key: "indentation"`). Only one active record may exist per `scope + domain + topic + slot_key` triplet. Use slot keys when there should be at most one answer per question — the admission gate enforces uniqueness automatically.

---

## 5b. API Reference

This section defines the complete HTTP interface for the Strata service. Any conforming implementation MUST expose these endpoints with these exact request and response shapes. Agents that receive only a base URL and an API key can bootstrap themselves using `GET /spec` (§5b.3) and then call any endpoint without additional instruction.

### 5b.1 Conventions

**Protocol:** HTTP/1.1 or HTTP/2. All request and response bodies are JSON.  
**Content-Type:** All requests MUST include `Content-Type: application/json`.  
**Base URL:** Operator-defined (e.g., `http://localhost:3500`). All paths below are relative to this base.

**Response envelope:** Every response uses the same wrapper:

```json
{ "ok": true, "data": { } }
```
```json
{ "ok": false, "error": { "code": "ERROR_CODE", "message": "Human-readable description" } }
```

**HTTP status codes:**

| Code | Meaning |
|---|---|
| 200 | Success |
| 400 | Missing required field or malformed request |
| 401 | Missing or invalid API key |
| 403 | Scope violation — caller not authorized for requested scope |
| 404 | Record not found within caller's scope |
| 422 | Admission gate rejected the write |
| 429 | Rate limited |
| 500 | Internal server error |

**Record object:** The following shape is returned wherever a memory record appears in a response:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "content": "Prefers tabs over spaces for all code",
  "normalized_value": "tabs",
  "memory_class": "preference",
  "scope": "agent:coding-agent",
  "domain": "coding",
  "topic": "style",
  "slot_key": "indentation",
  "confidence": 0.9,
  "importance": 0.7,
  "source_kind": "human_direct",
  "event_time": "2025-03-15T10:00:00Z",
  "created_at": "2025-03-15T10:05:00Z",
  "record_status": "active",
  "freshness_note": null
}
```

Fields that are not applicable for a given record will be `null`, not omitted.

### 5b.2 Authentication

All requests MUST include an API key in the `Authorization` header:

```
Authorization: Bearer <api-key>
```

Requests without a valid key receive `401 Unauthorized`.

**Key lifecycle — how it works:**

1. **First startup:** If no key is configured, the service automatically generates a cryptographically random 32-byte key, encodes it as a URL-safe base64 string, and persists it to `/data/strata.key`. It prints the key once to the console at startup:

   ```
   ============================================================
   Strata API Key (save this — it will not be shown again):
    sk-strata-<STRATA_API_KEY>
   ============================================================
   ```

2. **Subsequent startups:** The service reads the key from `/data/strata.key` silently. The key does not change unless the operator explicitly rotates it.

3. **Environment variable override:** If `STRATA_API_KEY` is set in the environment, it takes precedence over the key file. This is the recommended approach for containerized or automated deployments where secrets are injected at runtime rather than stored on disk.

4. **Key rotation:** To rotate the key, set a new value in `STRATA_API_KEY` or delete `/data/strata.key` and restart. Any agent using the old key will receive `401` until updated with the new key.

**From the user's perspective:** Deploy the service, copy the key from the startup output, give it to any agent alongside the base URL. That is the complete setup. The key never expires and never needs to be regenerated unless the operator chooses to rotate it.

**Identity binding — validating declared identity against credential:**

Every request body includes `agent_id` and optionally `user_id`. Implementations MUST validate the declared identity against the authenticated credential — the request body alone MUST NOT be trusted:

- **Single shared key (simple deployments):** All callers using the key are considered trusted. `agent_id` and `user_id` are accepted as declared. This is appropriate for single-user or fully trusted deployments.
- **Per-agent keys (hardened deployments):** Each agent receives its own API key issued with an explicit `agent_id` binding. A request declaring `agent_id: "coding-agent"` on a key issued for `"research-agent"` MUST be rejected with `403 Forbidden`.

**Per-agent credentials (optional hardening):**

For production multi-agent or multi-user deployments, implementations SHOULD support per-agent API key binding. Each key is issued with:

- An `agent_id` allowlist — the identity or identities this key may assert.
- A scope ceiling — the maximum scopes this key may read from or write to (e.g., write to `agent:coding-agent` only, read `shared` and `user:alice`, no access to other user scopes).

This prevents one agent from spoofing another's identity and limits blast radius if a key is compromised. The `/data/strata.key` global key continues to serve as the operator bootstrap credential. Per-agent keys are issued and managed by the operator out-of-band.

A request exceeding its key's scope ceiling receives `403 Forbidden` with error code `SCOPE_CEILING_EXCEEDED`.

### 5b.3 Service Discovery

```
GET /health
GET /spec
```

**`GET /health`** — Confirms the service is reachable and the database is healthy. No authentication required.

Response:
```json
{
  "ok": true,
  "data": {
    "status": "healthy",
    "version": "1.0",
    "db_path": "/data/strata.db",
    "retrieval_mode": "local_hybrid | hybrid | fts_only",
    "sqlite_vec": "loaded | not_loaded",
    "embedding_backend": "healthy" | "degraded" | "unavailable" | "not_configured",
    "default_user_id": "alice | null"
  }
}
```

**`GET /spec`** — Returns the full Strata specification (this document) as plain text Markdown. No authentication required. This is the primary bootstrap mechanism: an agent given only a base URL fetches this endpoint first to learn the complete API and behavioral expectations before making any memory calls.

Response: `Content-Type: text/markdown` — the contents of STRATA.md.

### 5b.4 Session Management

Sessions are optional for single-call use but required for evidence tracking, pre-compaction flush (Rule 7), and session context features (§5a.7). All conforming implementations SHOULD support session management.

---

**`POST /memory/session/start`**

Declares the start of a session. Returns a `session_id` that can be passed on subsequent calls for lineage tracking.

Request:
```json
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "session_context": "projects/website1"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `agent_id` | string | yes | Identifies the calling agent |
| `user_id` | string | no | Identifies the end user this session serves |
| `session_context` | string | no | Pre-filter context for this session (§5a.7). Format: `"domain"` or `"domain/topic"` |

Response:
```json
{
  "ok": true,
  "data": {
    "session_id": "uuid",
    "agent_id": "coding-agent",
    "user_id": "alice",
    "session_context": "projects/website1",
    "started_at": "2025-03-15T10:00:00Z"
  }
}
```

---

**`POST /memory/session/end`**

Closes a session. Triggers pre-compaction flush of pending evidence (Rule 7) and finalizes session bookkeeping. SHOULD be called at the end of every session.

Request:
```json
{
  "session_id": "uuid",
  "agent_id": "coding-agent"
}
```

Response:
```json
{
  "ok": true,
  "data": {
    "session_id": "uuid",
    "evidence_flushed": 12,
    "records_promoted": 3,
    "ended_at": "2025-03-15T11:00:00Z"
  }
}
```

If the session ended unexpectedly and `session/end` was never called, the maintenance loop (§20) will recover pending evidence on the next scheduled run.

### 5b.5 memory_search

Retrieves memory records relevant to a query. Applies FTS5 and optionally vector search depending on embedding backend health (§14.1). Always applies scope filtering.

**`POST /memory/search`**

Request:
```json
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "query": "preferred indentation style",
  "session_id": "uuid",
  "session_context": "coding/style",
  "scope_override": ["agent:coding-agent", "user:alice"],
  "memory_class": ["preference", "fact"],
  "domain": "coding",
  "topic": "style",
  "min_confidence": 0.5,
  "limit": 10,
  "include_superseded": false
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `agent_id` | string | yes | — | Calling agent identity |
| `user_id` | string | no | — | End user identity; adds `user:<id>` to read scope |
| `query` | string | yes | — | Natural language search string |
| `session_id` | string | no | — | Session identifier for lineage |
| `session_context` | string | no | session default | Overrides session-level context for this call only |
| `scope_override` | array of strings | no | `[agent:<agent_id>, user:<user_id>, shared]` | Explicitly set the scopes to search |
| `memory_class` | array of strings | no | all classes | Filter results to these classes |
| `domain` | string | no | — | Filter to this domain |
| `topic` | string | no | — | Filter to this topic (only meaningful with `domain`) |
| `min_confidence` | float | no | 0.0 | Exclude records below this confidence threshold |
| `limit` | integer | no | 10 | Maximum records to return (max 100) |
| `include_superseded` | boolean | no | false | Include superseded records in results |

Response:
```json
{
  "ok": true,
  "data": {
    "results": [ <record objects> ],
    "count": 3,
    "retrieval_mode": "local_hybrid",
    "context_filtered": true
  }
}
```

| Response field | Description |
|---|---|
| `results` | Array of record objects ordered by relevance |
| `count` | Number of records returned |
| `retrieval_mode` | `"local_hybrid"` (FTS5 + sqlite-vec), `"hybrid"` (FTS5 + external embedder), or `"fts_only"` (no vector backend available) |
| `context_filtered` | `true` if a session_context filter was applied |

**Error: `QUERY_NOISE`** — The query gate (Rule 9) rejected the query as too short, trivially acknowledgment, or matching a credential pattern. The agent SHOULD skip the search and proceed. This is not a failure condition.

### 5b.6 memory_get

Fetches a specific record by ID, or an evidence packet (record plus supporting evidence) by topic.

**`POST /memory/get`**

Request — fetch by ID:
```json
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "record_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

Request — fetch evidence packet by topic:
```json
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "domain": "projects",
  "topic": "website1"
}
```

One of `record_id` or `domain`+`topic` is required.

Response — single record:
```json
{
  "ok": true,
  "data": {
    "record": <record object>,
    "evidence": null
  }
}
```

Response — evidence packet:
```json
{
  "ok": true,
  "data": {
    "record": <record object>,
    "evidence": [
      {
        "id": "uuid",
        "content": "raw evidence text",
        "evidence_type": "conversation_turn",
        "created_at": "2025-03-15T09:55:00Z"
      }
    ]
  }
}
```

### 5b.7 memory_write

Persists a new memory record. All writes pass through the admission gate (§11). The response indicates whether the record was written immediately, held for operator review, or caused supersession of an existing record.

**`POST /memory/write`**

Request:
```json
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "content": "User prefers tabs over spaces for all indentation",
  "memory_class": "preference",
  "scope": "user:alice",
  "domain": "coding",
  "topic": "style",
  "slot_key": "indentation",
  "normalized_value": "tabs",
  "event_time": "2025-03-15T10:00:00Z",
  "confidence": 0.9,
  "importance": 0.7,
  "source_kind": "human_direct",
  "session_id": "uuid"
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `agent_id` | string | yes | — | Calling agent identity |
| `user_id` | string | no | — | End user identity |
| `content` | string | yes | — | The memory content to store |
| `memory_class` | string | yes | — | One of the valid memory classes (§5b.11) |
| `scope` | string | no | `agent:<agent_id>` | Target scope for this record |
| `domain` | string | no | — | Content domain tag |
| `topic` | string | no | — | Content topic tag (use with domain) |
| `slot_key` | string | no | — | Addressable key for preferences; enables supersession by slot rather than by ID |
| `normalized_value` | string | no | — | Machine-readable form of the content (e.g., `"tabs"` for an indentation preference) |
| `event_time` | ISO 8601 string | no | — | When this fact occurred in the world (distinct from write time) |
| `confidence` | float 0.0–1.0 | no | class default | Initial confidence in this record |
| `importance` | float 0.0–1.0 | no | 0.5 | Importance weight for retrieval ranking |
| `source_kind` | string | no | `"agent_inference"` | Source authority (§5b.11) |
| `session_id` | string | no | — | Session identifier for lineage |

Response:
```json
{
  "ok": true,
  "data": {
    "status": "written",
    "record_id": "uuid",
    "superseded_id": null,
    "hold_reason": null
  }
}
```

| `status` value | Meaning |
|---|---|
| `written` | Record committed immediately |
| `held_for_review` | Record created but not active; awaiting operator confirmation via `memory_confirm` |
| `superseded_existing` | A prior record was superseded; `superseded_id` contains its ID |

When `status` is `held_for_review`, the fact is NOT yet stored as active memory. The agent SHOULD NOT treat the content as committed. The `record_id` can be passed to `memory_confirm` by an operator to approve or reject.

### 5b.8 memory_retract

Marks a record as no longer true. The record is not deleted — it is archived with `record_status = 'retracted'` and excluded from default search results. The retraction reason is logged in `memory_updates`.

**`POST /memory/retract`**

Request:
```json
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "record_id": "550e8400-e29b-41d4-a716-446655440000",
  "reason": "User changed indentation preference to spaces",
  "session_id": "uuid"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `agent_id` | string | yes | Calling agent identity |
| `user_id` | string | no | End user identity |
| `record_id` | string | yes | ID of the record to retract |
| `reason` | string | yes | Human-readable reason for retraction |
| `session_id` | string | no | Session identifier for lineage |

Response:
```json
{
  "ok": true,
  "data": {
    "retracted_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "retracted"
  }
}
```

### 5b.9 memory_confirm

Approves or rejects a record that was held for operator review. Only calls with an `operator_id` are accepted for this endpoint — standard agent calls are rejected with `403`.

**`POST /memory/confirm`**

Request:
```json
{
  "operator_id": "admin",
  "record_id": "550e8400-e29b-41d4-a716-446655440000",
  "decision": "approve",
  "reason": "Verified correct"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `operator_id` | string | yes | Operator identity (must be configured at deployment) |
| `record_id` | string | yes | ID of the held record |
| `decision` | string | yes | `"approve"` or `"reject"` |
| `reason` | string | no | Reason for the decision (logged in audit trail) |

Response:
```json
{
  "ok": true,
  "data": {
    "record_id": "550e8400-e29b-41d4-a716-446655440000",
    "decision": "approve",
    "status": "written"
  }
}
```

If `decision` is `"reject"`, `status` will be `"rejected"` and the record is permanently discarded.

### 5b.10 memory_reflect

Adjusts the confidence of an `opinion` or `belief` record based on new evidence. Positive `delta` strengthens the belief; negative weakens it. The result is clamped to `[0.0, 1.0]`.

**`POST /memory/reflect`**

Request:
```json
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "record_id": "550e8400-e29b-41d4-a716-446655440000",
  "delta": 0.15,
  "reason": "User confirmed this preference again in today's session",
  "session_id": "uuid"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `agent_id` | string | yes | Calling agent identity |
| `user_id` | string | no | End user identity |
| `record_id` | string | yes | ID of the opinion or belief record to adjust |
| `delta` | float -1.0 to 1.0 | yes | Confidence adjustment. Positive strengthens, negative weakens |
| `reason` | string | no | Reason for the adjustment (logged in audit trail) |
| `session_id` | string | no | Session identifier for lineage |

Response:
```json
{
  "ok": true,
  "data": {
    "record_id": "550e8400-e29b-41d4-a716-446655440000",
    "old_confidence": 0.6,
    "new_confidence": 0.75,
    "clamped": false
  }
}
```

`memory_reflect` only operates on records with `memory_class` of `opinion` or `belief`. Calling it on other classes returns `400 Bad Request`.

### 5b.11 Enumeration Reference

These are the valid values for enum fields. The service MUST reject any value not in these lists with `400 Bad Request`.

**`memory_class`**

| Value | Use for |
|---|---|
| `fact` | Objective, verifiable information |
| `preference` | User or agent stylistic or behavioral preferences |
| `decision` | A choice made that should influence future behavior |
| `profile` | Identity or descriptive information about a person or system |
| `event` | Something that happened at a point in time |
| `opinion` | A subjective assessment with floating confidence |
| `belief` | A working assumption the agent holds with moderate confidence |
| `scratchpad` | Temporary working notes; expires automatically after session |

**`source_kind`** (authority ranking, highest to lowest)

| Value | Meaning |
|---|---|
| `operator_authored` | Written directly by the operator |
| `human_direct` | Stated directly by the end user |
| `tool_output` | Produced by a verified tool call |
| `agent_inference` | Inferred by the agent from context |
| `episodic_inference` | Inferred from episodic evidence during promotion |
| `semantic_inference` | Inferred from semantic patterns |
| `external_import` | Imported from an external source |

**`record_status`**

| Value | Meaning |
|---|---|
| `active` | Current, returned in default search results |
| `superseded` | Replaced by a newer record; excluded from default search |
| `retracted` | Marked as no longer true; excluded from default search |
| `held` | Awaiting operator confirmation; not yet active |

### 5b.12 memory_compact

Returns the review queue without modifying anything. See §13.7 for full behavioral description.

**`POST /memory/compact`**

Request:
```json
{
  "agent_id": "coding-agent",
  "user_id": "alice",
  "limit": 20,
  "include_types": ["low_confidence", "contradiction", "stale"]
}
```

Response:
```json
{
  "ok": true,
  "data": {
    "review_items": [ <review item objects> ],
    "count": 3,
    "queue_generated_at": "2025-03-15T10:00:00Z"
  }
}
```

### 5b.13 memory_sync

Re-indexes the Markdown knowledge base on demand. Scans `$STRATA_DATA_PATH/kb/` for changed files, rebuilds `docs_fts`, and optionally runs pattern-based promotion on obvious facts or preferences found in the updated content.

**`POST /memory/sync`**

Request:
```json
{
  "agent_id": "coding-agent",
  "paths": ["kb/projects", "kb/notes"],
  "promote": true
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `agent_id` | string | yes | — | Calling agent identity |
| `paths` | array of strings | no | all of `kb/` | Subdirectories to scan, relative to `STRATA_DATA_PATH` |
| `promote` | boolean | no | false | Run pattern-based promotion on updated content |

Response:
```json
{
  "ok": true,
  "data": {
    "files_scanned": 12,
    "files_updated": 3,
    "records_promoted": 1,
    "docs_fts_rebuilt": false
  }
}
```

`docs_fts_rebuilt` is `true` only if the FTS index was fully rebuilt (e.g., after significant changes). Incremental updates are `false`.

---

## 5c. Agent Integration Guide

This section tells any agent exactly how to use Strata correctly. An agent that follows these steps will integrate properly with any conforming Strata implementation. No other documentation is required beyond this spec and the base URL of the service.

### 5c.0 User Identity Setup

Before making any calls on behalf of a user, you need a `user_id`. This section tells you how to get one.

**Single-user deployment (most personal setups):** The operator set `DEFAULT_USER_ID` when deploying the service. You do not need to ask — omitting `user_id` from your requests is fine and the service will apply the default. Confirm this by checking `GET /health`; if `default_user_id` appears in the response, it is set.

**Multi-user deployment:** You must know which user you are serving. On first interaction with a new user, ask:

> *"What name or username should I use to remember things for you?"*

Store the answer locally and pass it as `user_id` on every Strata call for that user. The value can be anything — a first name, a handle, a UUID — as long as it is used consistently. If the value changes between sessions, Strata treats it as a different person and the memory does not connect.

**The rule:** `user_id` is the agent's responsibility, not the service's. Strata enforces isolation between users; the agent is responsible for knowing which user is being served.

### 5c.1 Bootstrapping from a URL

If you have been given a Strata service URL and nothing else:

1. Call `GET <base_url>/health` to confirm the service is reachable.
2. Call `GET <base_url>/spec` to retrieve this specification. Read it fully before proceeding.
3. You now know everything needed to use the service.

Minimum required information to operate: **base URL** and **API key**. Nothing else.

### 5c.2 Session Lifecycle

**At the start of every session:**

1. Call `POST /memory/session/start` with your `agent_id`, your `user_id` (if serving a specific user), and `session_context` (if you are working in a specific domain or project context).
2. Call `POST /memory/search` with a brief orientation query — for example: `"current goals, active preferences, recent decisions"` — to load relevant context into your working memory.
3. Inject the returned records into your system context before processing the first user message.

**During a session:**

4. Before answering any question where prior knowledge might be relevant, call `POST /memory/search` with a specific query.
5. When you learn a fact, preference, decision, or profile detail that should persist beyond this session, call `POST /memory/write`.
6. When you discover that a stored fact is no longer true, call `POST /memory/retract`.
7. When new evidence strengthens or weakens a belief or opinion, call `POST /memory/reflect`.
8. For temporary working notes that should not persist after the session, write with `"memory_class": "scratchpad"`. These expire automatically.

**At the end of every session:**

9. Call `POST /memory/session/end` with your `session_id`. This flushes pending evidence and finalizes session bookkeeping (required by Rule 7). If the session ends unexpectedly and this call is never made, the maintenance loop will recover on the next scheduled run.
10. Optionally call `POST /memory/compact` to surface the review queue. If items are returned, consider calling `memory_reflect` or `memory_retract` on the most important ones before closing the session.

### 5c.3 When to Search

Search before: answering factual questions, making recommendations, beginning a new task on a known project, referring to a user's stated preferences, or any situation where prior context would affect your response.

Do not search for: trivially short inputs (one or two words), acknowledgment phrases ("ok", "thanks", "got it"), or strings that look like credentials or tokens. The query gate will reject these automatically — if you receive a `QUERY_NOISE` error, skip the search and proceed normally.

### 5c.4 When to Write

Write when you learn something that should outlast this session:

- A user states a preference ("I always want short explanations")
- A decision is made ("We are going with PostgreSQL for this project")
- A new fact is established ("The staging server is at 192.168.1.50")
- A profile detail is confirmed ("This user is a senior engineer")

Do not write conversational filler, trivial acknowledgments, or information that is already in the database. Check search results before writing — if the fact already exists and is still accurate, there is no need to write it again.

Use `slot_key` for preferences that should overwrite a prior value of the same preference rather than accumulate (e.g., `"slot_key": "response_length"` ensures there is only one active preference for response length at a time).

### 5c.5 When to Retract

Retract when something that was previously true is no longer true. Always provide a `reason` — this becomes part of the audit trail and helps future agents understand why the record was archived.

After retracting, write the new fact with a fresh `memory_write` call if applicable.

### 5c.6 Handling a Held Write

If `memory_write` returns `"status": "held_for_review"`, the record has NOT been committed to active memory. Do not treat it as stored. If the context is important, inform the user or operator that the write is pending confirmation. An operator must call `POST /memory/confirm` to approve or reject it.

### 5c.7 Handling Scope

You do not need to construct scope strings manually. The service resolves your `agent_id` and `user_id` into the correct scope automatically.

- Default read: your agent scope + your user scope (if provided) + shared
- Default write: your agent scope

To write something the user's other agents should also see, explicitly pass `"scope": "user:<user_id>"` in the write request.

To write something all agents and all users should see (shared facts, system-wide configuration), explicitly pass `"scope": "shared"`. Note: the PII gate (§21.2) will reject shared-scope writes containing personal information.

You MUST NOT pass a `scope` value for a user other than the one you are currently serving.

### 5c.8 Working with Session Context

If you are working on a specific project or domain within a session, declare it at session start:

```json
{ "session_context": "projects/website1" }
```

All searches in that session will automatically pre-filter to memories tagged with that domain/topic. To search across all contexts for a specific call, pass `"session_context": null` on that call.

To check what context you are currently in, it is returned in the `session/start` response and can be stored in your working memory for the session.

---

## 5d. MCP Integration

The Model Context Protocol (MCP) is an open standard for exposing tools and resources to AI agents. A Strata MCP server allows any MCP-compatible client — Claude Code, Claude Desktop, or any MCP-aware agent — to use Strata's memory tools automatically without manual HTTP configuration. The agent discovers the tools at connection time and calls them exactly as it would any other tool.

The MCP interface and the HTTP API expose the same underlying operations. Calls via MCP and calls via HTTP MUST produce identical results.

### 5d.1 MCP Server Endpoint

The Strata service MUST expose an MCP endpoint alongside the HTTP API:

```
GET  /mcp          — MCP server manifest (tool definitions and input schemas)
POST /mcp          — MCP tool call handler
GET  /mcp/sse      — Server-sent events stream (for streaming-capable clients)
```

Authentication uses the same `Authorization: Bearer <api-key>` header as the HTTP API.

### 5d.2 Client Configuration

Adding Strata to any MCP-compatible client requires a single configuration entry. No additional setup is needed — the client fetches the tool manifest at connection time and the tools become immediately available.

**Claude Code / Claude Desktop** (`~/.claude/mcp.json` or equivalent):

```json
{
  "mcpServers": {
    "strata": {
      "type": "http",
      "url": "http://your-strata-host/mcp",
      "headers": {
        "Authorization": "Bearer <STRATA_API_KEY>"
      }
    }
  }
}
```

That is the complete client-side configuration. The agent now has the full set of Strata memory tools.

### 5d.3 Agent Identity in MCP Context

In HTTP API calls, `agent_id` and `user_id` are passed in the request body. In MCP calls, these are resolved server-side:

- `agent_id` is derived from the MCP client's `client_info.name` field provided during the MCP handshake.
- `user_id` is configured in the Strata server's MCP settings (set once per deployment, or per client registration) and applied to all calls from that client.

The agent does not pass `agent_id` or `user_id` as tool parameters — scope resolution is automatic.

### 5d.4 Tool Manifest

The MCP server MUST serve the following tool definitions at `GET /mcp`. Input schemas follow JSON Schema draft-07.

---

**`memory_search`**
```json
{
  "name": "memory_search",
  "description": "Search memory for facts, preferences, decisions, and history relevant to a query. Call this before answering questions where prior knowledge might be relevant.",
  "inputSchema": {
    "type": "object",
    "required": ["query"],
    "properties": {
      "query": { "type": "string", "description": "Natural language search string" },
      "session_context": { "type": "string", "description": "Domain/topic pre-filter. Format: 'domain' or 'domain/topic'" },
      "memory_class": { "type": "array", "items": { "type": "string" }, "description": "Restrict to these memory classes" },
      "domain": { "type": "string" },
      "topic": { "type": "string" },
      "min_confidence": { "type": "number", "minimum": 0, "maximum": 1, "default": 0 },
      "limit": { "type": "integer", "minimum": 1, "maximum": 100, "default": 10 },
      "include_superseded": { "type": "boolean", "default": false }
    }
  }
}
```

**`memory_get`**
```json
{
  "name": "memory_get",
  "description": "Fetch a specific memory record by ID, or retrieve an evidence packet for a domain/topic.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "record_id": { "type": "string", "description": "UUID of a specific record" },
      "domain": { "type": "string", "description": "Domain for evidence packet lookup" },
      "topic": { "type": "string", "description": "Topic for evidence packet lookup" }
    }
  }
}
```

**`memory_write`**
```json
{
  "name": "memory_write",
  "description": "Persist a fact, preference, decision, profile detail, or event to memory. Use when you learn something that should outlast this session.",
  "inputSchema": {
    "type": "object",
    "required": ["content", "memory_class"],
    "properties": {
      "content": { "type": "string", "description": "The memory content to store" },
      "memory_class": { "type": "string", "enum": ["fact", "preference", "decision", "profile", "event", "opinion", "belief", "scratchpad"] },
      "scope": { "type": "string", "description": "Target scope. Defaults to agent scope. Use 'user:<id>' for user-level preferences." },
      "domain": { "type": "string" },
      "topic": { "type": "string" },
      "slot_key": { "type": "string", "description": "Addressable key for preferences. Ensures only one active record per slot." },
      "normalized_value": { "type": "string", "description": "Machine-readable form of the content" },
      "event_time": { "type": "string", "format": "date-time", "description": "When this fact occurred in the world" },
      "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
      "importance": { "type": "number", "minimum": 0, "maximum": 1, "default": 0.5 },
      "source_kind": { "type": "string", "default": "agent_inference" }
    }
  }
}
```

**`memory_retract`**
```json
{
  "name": "memory_retract",
  "description": "Mark a memory record as no longer true. The record is archived, not deleted. Always provide a reason.",
  "inputSchema": {
    "type": "object",
    "required": ["record_id", "reason"],
    "properties": {
      "record_id": { "type": "string", "description": "UUID of the record to retract" },
      "reason": { "type": "string", "description": "Why this record is no longer true" }
    }
  }
}
```

**`memory_confirm`**
```json
{
  "name": "memory_confirm",
  "description": "Approve or reject a memory record that was held for operator review.",
  "inputSchema": {
    "type": "object",
    "required": ["record_id", "decision"],
    "properties": {
      "record_id": { "type": "string" },
      "decision": { "type": "string", "enum": ["approve", "reject"] },
      "reason": { "type": "string" }
    }
  }
}
```

**`memory_reflect`**
```json
{
  "name": "memory_reflect",
  "description": "Adjust the confidence of an opinion or belief record based on new evidence. Positive delta strengthens, negative weakens.",
  "inputSchema": {
    "type": "object",
    "required": ["record_id", "delta"],
    "properties": {
      "record_id": { "type": "string" },
      "delta": { "type": "number", "minimum": -1, "maximum": 1, "description": "Confidence adjustment. Positive strengthens, negative weakens." },
      "reason": { "type": "string" }
    }
  }
}
```

**`memory_compact`**
```json
{
  "name": "memory_compact",
  "description": "Return the review queue: records that may need attention (low confidence, contradictions, stale facts, unresolved holds). Read-only — call memory_retract, memory_reflect, or memory_confirm to act on results.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "limit": { "type": "integer", "minimum": 1, "maximum": 100, "default": 20 },
      "include_types": {
        "type": "array",
        "items": { "type": "string", "enum": ["low_confidence", "contradiction", "stale", "scratchpad_candidate", "unresolved_hold"] },
        "description": "Filter to specific flag reasons. Omit for all types."
      }
    }
  }
}
```

**`memory_sync`**
```json
{
  "name": "memory_sync",
  "description": "Re-index the Markdown knowledge base on demand. Call after manually editing files in /data/kb/ to ensure docs_fts stays current.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "paths": { "type": "array", "items": { "type": "string" }, "description": "Subdirectories to scan, relative to STRATA_DATA_PATH. Defaults to all of kb/." },
      "promote": { "type": "boolean", "default": false, "description": "Run pattern-based promotion on updated Markdown content." }
    }
  }
}
```

### 5d.5 Relationship to HTTP API

The MCP interface is a convenience layer over the HTTP API. Every MCP tool call is translated internally to the equivalent HTTP API operation with the same validation, admission gate, scope enforcement, and audit logging. There is no MCP-only behavior.

Implementations that support MCP MUST pass the same test suite as the HTTP API (§25). MCP is listed as part of the Phase 3 implementation (§24).

---

## 6. Non-Goals

The initial Strata implementation is **not** intended to be:

- a full graph database platform,
- a petabyte-scale document warehouse,
- a cloud memory service,
- a fully autonomous self-writing knowledge engine,
- a replacement for application-specific state machines,
- a blockchain or external cryptographic ledger (see §12a for optional lightweight integrity support).

Strata is a pragmatic, sovereign memory substrate, not a universal knowledge operating system.

---

## 7. Memory Classes and Storage Rules

### 7.1 Constitutional Memory
**Storage:** Markdown files and/or protected records in the semantic plane  
**Characteristics:** small, versioned, rarely changed, reviewed before modification.

### 7.2 Episodic Memory
**Storage:** `memory_evidence` table  
**Characteristics:** append-only, timestamped, auditable. May be promoted to semantic records.

### 7.3 Semantic Memory
**Storage:** `memory_records` table  
**Characteristics:** updatable via supersession chain, source-linked, confidence-scored, slot-addressed for preferences.

### 7.4 Narrative Memory
**Storage:** Markdown files on disk, indexed in FTS  
**Characteristics:** human-readable, longer-form, context-rich.

---

## 8. Recommended Directory Layout

```text
/data/                  ← persistent root (mount point for container deployments)
  strata.db             ← primary SQLite database
  strata.key            ← auto-generated API key (§5b.2)
  kb/                   ← Markdown knowledge base
    identity/
    runbooks/
    projects/
    summaries/
    notes/
  exports/
  backups/
```

**Environment variables:**

| Variable | Default | Description |
|---|---|---|
| `STRATA_DATA_PATH` | `/data` (container) / `./data` (local) | Root directory for all persistent data |
| `STRATA_DB_PATH` | `$STRATA_DATA_PATH/strata.db` | SQLite database path (override for custom locations) |
| `STRATA_API_KEY` | read from `strata.key` | API key override (for secrets injection at runtime) |
| `DEFAULT_USER_ID` | unset | Single-user convenience: used when `user_id` is omitted from requests |

**Container deployments:** Mount a named volume or host directory to `/data`. This directory contains everything that must survive container restarts: the database, the API key, the knowledge base, and all exports.

```yaml
# docker-compose.yml example
volumes:
  - strata-data:/data
```

or bind-mount:
```
docker run -v ./strata-data:/data ...
```

**Local deployments:** The service defaults to `./data` relative to its working directory. No additional configuration needed.

Conventions:
- `strata.db` is the primary SQLite database and the only file any process should write to directly.
- `kb/` holds human-editable Markdown documents. Changes here are picked up by the maintenance loop (§20.11) or on-demand via `memory_sync`.
- `backups/` receives nightly or scheduled database backups.
- `exports/` receives output from lifecycle export operations (§23).

---

## 9. SQLite Schema

### 9.1 Enumerations

Implementations MUST enforce these at the application layer — SQLite does not enforce TEXT enums natively. Reject unknown values at write time with a descriptive error. New values may be added only via schema migration (§22).

**`memory_records.memory_class`**
```
fact | preference | decision | profile | event | note | summary | rule | constraint | opinion | belief | scratchpad | system_config
```
The `fact` type is for operationally relied-upon exact knowledge. `preference` is for stable behavioral preferences (use `slot_key` for slot-addressed supersession — see §9.2). `decision` is for committed choices. `profile` is for identity or environmental information. `event` is for something that happened at a point in time. `note` and `summary` are for less structured semantic content. `rule` and `constraint` are for behavioral directives. `opinion` is for subjective assessments or beliefs held by the agent that are not objectively verifiable facts — they carry a floating confidence score that is adjusted by the Reflect layer (§20.10) as supporting or contradicting evidence accumulates. `belief` is a stronger, higher-stakes opinion that has been confirmed across multiple sessions. `scratchpad` is for temporary working notes that expire automatically after the session. `system_config` is for operator-defined configuration, taxonomy, deployment policy, or shared system metadata. Implementations MAY extend this set; new values MUST be documented in local deployment configuration.

**`memory_records.status`**
```
active | superseded | retracted | held
```
- `active` — current, returned in default search results
- `superseded` — replaced by a newer record; excluded from default search
- `retracted` — marked as no longer true; excluded from default search
- `held` — accepted for operator review but not active and not returned by default search

**`memory_evidence.evidence_type`**
```
conversation_turn | document | tool_execution | tool_failure |
agent_action | skill_execution | correction | session_start | session_end
```

**`memory_evidence.promotion_status`**
```
pending | promoted | ignored | rejected | normalized
```
- `pending` — awaiting promotion review
- `promoted` — produced at least one `memory_records` row
- `ignored` — suppressed (e.g., negation gate or trivial content)
- `rejected` — reviewed and explicitly excluded
- `normalized` — processed and indexed but not promoted to a full record

**`memory_evidence.speaker_role`** *(for conversation turns)*
```
user | assistant | system | tool
```

**`memory_updates.update_type`**
```
insert | update | supersede | retract | restore |
reject_volatile | hold_for_review | compaction_flush |
integrity_check | schema_migration | retrieval_mode_decision | belief_update
```
`belief_update` records in-place confidence adjustments to `opinion` and `belief` records made by the Reflect layer (§20.10). Unlike `supersede` (which creates a new row), `belief_update` captures confidence drift on the same record. The `prior_value_json` MUST include the old confidence value; `new_value_json` MUST include the new value and the reason for the change (corroborating or contradicting evidence ID).

**`memory_links.link_type`**
```
supports | contradicts | refines | retracts
```

**`scratchpad_entries.kind`**
```
note | draft | working_context | candidate_fact
```

**`scope`** *(applies to all scoped tables)*

`scope` is a logical partition key identifying which agent, user, tenant, or role owns a memory record. Strata defines three scope tiers:

- **Shared scope** — system-wide, readable by all agents and users. Use `'shared'` as the value. Writable only by privileged sources (operator correction, structured import). PII MUST NOT appear in shared-scope records — pass through a PII check at write time (§21.2).
- **User scope** — private to a specific human user. Format: `'user:<id>'` (e.g., `'user:alice'`, `'user:42'`). Retrieval MUST filter to the requesting user's scope by default. Cross-user reads require explicit elevated permission.
- **Role/agent scope** — for multi-agent deployments. Format: `'agent:<name>'` or `'role:<name>'`. Use to separate memory by agent role (e.g., `'agent:planner'`, `'agent:researcher'`).

Additional tiers for advanced deployments:
- Tenant scope: `'tenant:<id>'`
- Combined: `'tenant:<id>:user:<id>'`

Single-agent, single-user deployments: use `'shared'` for all records.

**`source_kind`** *(inline field on records and evidence)*
```
operator_correction | structured_record | markdown_doc |
episodic_inference | semantic_inference | tool_output |
external_import | evidence_promotion | explicit_write
```

---

### 9.2 `memory_records` — Semantic Plane

The primary durable knowledge store. Holds all structured semantic memory: facts, preferences, decisions, and profiles.

```sql
CREATE TABLE memory_records (
  id                   INTEGER PRIMARY KEY,
  scope                TEXT NOT NULL,           -- see §9.1 scope
  owner                TEXT,                    -- user_id or NULL for shared
  memory_class        TEXT NOT NULL,           -- see §9.1 enum (memory_class)
  slot_key             TEXT,                    -- slot address for supersession (§9.2.1)
  domain               TEXT,                   -- high-level category (§9.2.2): e.g. 'network', 'user_preferences'
  topic                TEXT,                   -- sub-category within domain: e.g. 'vlans', 'response_style'
  title                TEXT NOT NULL,           -- short label, first 120 chars of content
  content              TEXT NOT NULL,           -- full text of this memory
  normalized_value     TEXT,                    -- canonical form for exact-match queries
  status               TEXT NOT NULL DEFAULT 'active',  -- see §9.1 enum
  confidence           REAL DEFAULT 1.0,        -- 0.0–1.0; for opinion/belief types, adjusted by Reflect layer
  importance           INTEGER DEFAULT 5,       -- 1–10 subjective weight
  event_time           TEXT,                   -- ISO8601 when the real-world event/fact occurred (§9.2.3)
  valid_from           TEXT,                    -- ISO8601 start of validity window
  valid_to             TEXT,                    -- ISO8601 end of validity window
  last_confirmed_at    TEXT,                    -- ISO8601 last time this was verified
  source_kind          TEXT,                    -- see §9.1 enum
  source_ref           TEXT,                    -- free-form reference to origin
  supersedes_record_id INTEGER,                -- FK to the record this replaces
  created_at           TEXT NOT NULL,           -- ISO8601 transaction time (when written to DB)
  updated_at           TEXT NOT NULL,
  FOREIGN KEY(supersedes_record_id) REFERENCES memory_records(id)
);
```

#### 9.2.1 Slot-Key Semantics for Preferences

The `slot_key` field provides addressable supersession for preferences. It identifies the "slot" that a preference occupies (e.g., `'response_style'`, `'code_format'`, `'verbosity'`).

When a new preference is written for an existing `(scope, owner, slot_key)` combination with a different value:
1. The existing `active` record for that slot is marked `superseded`.
2. A new `active` record is inserted with `supersedes_record_id` pointing to the old one.
3. A `memory_updates` row records the supersession atomically.

This preserves the full history while ensuring only one `active` record exists per slot per owner. Non-preference record types MAY also use `slot_key` for similar addressable supersession.

#### 9.2.2 Domain and Topic Hierarchy

The `domain` and `topic` fields implement a two-level content hierarchy that enables hierarchical routing at retrieval time without requiring a full graph database.

- `domain` — the high-level category of knowledge. Use lowercase slugs. Examples: `network`, `user_preferences`, `project:sage`, `security`, `infrastructure`, `procedures`.
- `topic` — the sub-category within the domain. Examples: `vlans`, `response_style`, `authentication`, `deployment`, `api_design`.

**Convention:** Use a colon separator for compound values when needed: `domain='project:sage'`, `topic='memory_system'`. Avoid deep nesting — two levels is the intended maximum for a SQLite-backed system.

**Routing benefit:** The query planner (§15) uses domain and topic to narrow candidate collection before FTS or vector search, achieving the same multi-hop traversal efficiency that ByteRover's Context Tree provides, without requiring a separate hierarchical store. A query classified as "VLAN configuration" routes to `domain='network'`, `topic='vlans'` before expanding to full-text search.

Implementations SHOULD set `domain` on every write. `topic` is optional but recommended for records where the domain is broad.

#### 9.2.3 Bi-Temporal Fields

`memory_records` carries two distinct timestamps to support bi-temporal queries:

| Field | Meaning | Also called |
|---|---|---|
| `event_time` | When the real-world fact or event occurred | "Event Time" |
| `created_at` | When this record was written to the database | "Transaction Time" |

The distinction matters for temporal reasoning. Example: a user says "yesterday I decided to use PostgreSQL." The `event_time` should be set to yesterday; `created_at` is automatically set to now. Without this distinction, a temporal query ("what was true last week?") would incorrectly date the fact to when it was recorded, not when it happened.

**Rules:**
- `event_time` is optional. If not known, leave NULL — the system treats `created_at` as the best available temporal anchor.
- `event_time` MUST NOT be in the future unless the record represents a planned/scheduled fact.
- At retrieval time, when `event_time` IS NOT NULL and differs significantly from `created_at`, both timestamps SHOULD be surfaced in the evidence packet with clear labels.
- The `valid_from`/`valid_to` fields capture the *validity window* of the fact (when it was true), which is distinct from `event_time` (when the event that created this fact occurred).

---

### 9.3 `memory_evidence` — Evidence Archive

The append-only episodic store. Captures raw session content, tool outputs, documents, and events before (optionally) promoting them to semantic records.

```sql
CREATE TABLE memory_evidence (
  id                   INTEGER PRIMARY KEY,
  scope                TEXT NOT NULL,
  owner                TEXT,
  evidence_type        TEXT NOT NULL,           -- see §9.1 enum
  source_kind          TEXT,
  source_ref           TEXT,
  session_id           TEXT,                    -- originating session identifier
  message_id           INTEGER,                 -- position within session (for ordering)
  timestamp            TEXT,                    -- ISO8601 when the event was observed
  speaker_role         TEXT,                    -- see §9.1 enum (for conversation turns)
  title                TEXT,
  content              TEXT NOT NULL,
  promotion_status     TEXT NOT NULL DEFAULT 'pending'
                       CHECK (promotion_status IN
                         ('pending','promoted','ignored','rejected','normalized')),
  promotion_checked_at TEXT,                    -- ISO8601 when promotion was last evaluated
  metadata_json        TEXT,                    -- JSON blob for type-specific fields
  created_at           TEXT NOT NULL,
  UNIQUE(scope, session_id, message_id)         -- idempotency key for session ingestion
);
```

**Evidence is append-only.** Original evidence rows MUST NOT be mutated (except `promotion_status` and `promotion_checked_at`, which may be updated by the promotion runner). To correct or retract an evidence item, write a new evidence row with `evidence_type='correction'` and reference the original ID in `metadata_json`. See Appendix A.

The `UNIQUE(scope, session_id, message_id)` constraint makes session ingestion idempotent: calling ingest twice on the same session produces no duplicate rows. Use `message_id` as the sequential message position within a session; use `evidence_ingestion_cursors` (§9.6) to track the ingestion watermark.

---

### 9.4 `memory_links` — Evidence-Record Provenance

Records which evidence items supported, contradicted, or refined which semantic records. Provides a complete audit trail from raw evidence to durable knowledge.

```sql
CREATE TABLE memory_links (
  record_id    INTEGER NOT NULL,
  evidence_id  INTEGER NOT NULL,
  link_type    TEXT NOT NULL,           -- see §9.1 enum
  created_at   TEXT NOT NULL,
  PRIMARY KEY (record_id, evidence_id, link_type),
  FOREIGN KEY(record_id)   REFERENCES memory_records(id) ON DELETE CASCADE,
  FOREIGN KEY(evidence_id) REFERENCES memory_evidence(id) ON DELETE CASCADE
);
```

---

### 9.5 `memory_embeddings` — Vector Store

Optional. Stores vector embeddings for semantic/hybrid retrieval. When present, embeddings are stored alongside the records/evidence they represent using a polar encoding scheme that reduces storage size.

```sql
CREATE TABLE memory_embeddings (
  id             INTEGER PRIMARY KEY,
  memory_id      TEXT NOT NULL,         -- e.g. 'record:42' or 'evidence:7'
  layer          TEXT NOT NULL,         -- 'record' | 'evidence'
  target_type    TEXT,                  -- 'record' | 'evidence' (redundant with layer; for indexed lookups)
  target_id      INTEGER,
  radius         REAL NOT NULL,         -- polar encoding: L2 norm of the original vector
  embedding_blob BLOB NOT NULL,         -- packed float32 unit-normalized vector
  dim            INTEGER NOT NULL,      -- vector dimension (document this at deployment time)
  model          TEXT NOT NULL,         -- embedding model name (e.g. 'nomic-embed-text')
  created_at     TEXT NOT NULL,
  UNIQUE(memory_id, layer),
  UNIQUE(target_type, target_id)
    WHERE target_type IS NOT NULL AND target_id IS NOT NULL
);
```

**Polar encoding:** Rather than storing the raw float vector, Strata stores a `radius` (the L2 norm) and a unit-normalized blob. To reconstruct: `vector = unit_blob * radius`. This reduces storage while preserving full fidelity. The `radius` also provides a fast magnitude filter before full similarity computation.

**Recommended defaults:** 384 dimensions for local models (e.g., nomic-embed-text, mxbai-embed-large via Ollama/llama.cpp); 1536 for OpenAI text-embedding-3-small. Document the chosen dimension in your deployment configuration.

**sqlite-vec (recommended local backend):** When the `sqlite-vec` extension is available, create an additional `vec0` virtual table for in-process ANN search. This keeps all vector data inside the single SQLite file with zero external dependencies and SIMD-accelerated cosine similarity.

```sql
-- Create only when sqlite-vec extension is loaded (check at startup — see §14.2)
CREATE VIRTUAL TABLE IF NOT EXISTS memory_vecs USING vec0(
  embedding_blob float[384]   -- dimension MUST match memory_embeddings.dim for this deployment
);
```

The `rowid` of `memory_vecs` MUST match the `id` of the corresponding `memory_embeddings` row. On every embedding write, insert into both tables atomically.

**Search with sqlite-vec:**
```sql
SELECT me.memory_id, mv.distance
FROM memory_vecs mv
JOIN memory_embeddings me ON me.id = mv.rowid
WHERE mv.embedding_blob MATCH :query_vec
  AND k = :limit
ORDER BY mv.distance;
```

**Fallback:** If the `sqlite-vec` extension is not loaded, similarity computation falls back to the application layer using the `embedding_blob` stored in `memory_embeddings`. The schema and data are identical; only the search path differs. This means a deployment can start without sqlite-vec and enable it later with no schema migration — just load the extension and create the `memory_vecs` virtual table.

---

### 9.6 `evidence_ingestion_cursors` — Ingestion Watermarks

Tracks how far session ingestion has progressed for each scope+session pair. Enables idempotent incremental ingestion without re-reading already-processed messages.

```sql
CREATE TABLE evidence_ingestion_cursors (
  scope           TEXT NOT NULL,
  session_id      TEXT NOT NULL,
  last_message_id INTEGER NOT NULL DEFAULT 0,
  updated_at      TEXT NOT NULL,
  PRIMARY KEY (scope, session_id)
);
```

Ingestion implementations SHOULD read the cursor before processing, ingest only messages with `message_id > last_message_id`, then update the cursor after a successful batch. This makes repeated ingestion calls idempotent.

---

### 9.7 `scratchpad_entries` — Ephemeral Notes

Short-lived session-scoped scratch space. Intended for working context, draft facts, and candidate decisions that haven't been committed to durable memory. Not part of the promotion pipeline.

```sql
CREATE TABLE scratchpad_entries (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  scope        TEXT NOT NULL,
  session_id   TEXT NOT NULL,
  kind         TEXT NOT NULL DEFAULT 'note',   -- see §9.1 enum
  content      TEXT NOT NULL,
  content_hash TEXT,                           -- SHA-256 of content for deduplication
  created_at   TEXT NOT NULL,
  expires_at   TEXT NOT NULL                  -- ISO8601; TTL enforced at read and maintenance time
);
```

**TTL policy:** `expires_at` MUST be set on every write. Minimum TTL: 60 seconds. Maximum TTL: 86400 seconds (24 hours). The maintenance loop (§20) purges expired entries. Retrieving scratchpad entries MUST filter out expired rows.

Scratchpad entries are not included in `memory_updates` audit rows. They are inherently ephemeral and their expiry is expected behavior.

---

### 9.8 `privacy_audit` — Access Log

Records read and write access to memory. Supports compliance, debugging, and cross-user access review.

```sql
CREATE TABLE privacy_audit (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp     TEXT NOT NULL,
  accessor      TEXT NOT NULL,    -- who performed the operation (user_id or agent identifier)
  accessed_user TEXT,             -- whose data was accessed (NULL for shared scope)
  resource_path TEXT NOT NULL,    -- e.g. 'memory/record/42' or 'memory/search/shared'
  action        TEXT NOT NULL,    -- 'read' | 'write' | 'delete'
  reason        TEXT,             -- for system-initiated operations
  session_id    TEXT
);
```

Implementations SHOULD write a `privacy_audit` row for every `memory_records` read or write and every `memory_search` call. Rows MUST NOT include raw sensitive content — `resource_path` is a path identifier, not a data copy.

For retrieval audit rows: omit the raw query text from `resource_path` to avoid logging potentially sensitive search terms. Use a path like `'memory/search/<scope>'` with operational metadata in `reason`.

---

### 9.9 `memory_updates` — Audit Ledger

The central governance log. Every durable memory write (insert, update, supersession, retraction, schema migration) produces a row here.

```sql
CREATE TABLE memory_updates (
  id               INTEGER PRIMARY KEY,
  update_type      TEXT NOT NULL,   -- see §9.1 enum
  target_table     TEXT NOT NULL,   -- 'memory_records' | 'memory_evidence' | etc.
  target_id        INTEGER,
  prior_value_json TEXT,            -- JSON snapshot of row before change
  new_value_json   TEXT,            -- JSON snapshot of row after change
  reason           TEXT,
  source_id        INTEGER,         -- optional FK to a sources table if used
  created_at       TEXT NOT NULL,

  -- Impact classification (§11.3)
  impact_score     REAL,            -- 0.0–1.0

  -- Lightweight integrity chain (§12a)
  row_hash         TEXT,            -- SHA-256 of this row's canonical content
  prev_hash        TEXT,            -- row_hash of the immediately prior row

  -- Session lineage (nullable — populated by session-aware runtimes)
  request_id       TEXT,            -- originating session or intent ID
  correlation_id   TEXT,            -- e.g. 'preference:response_style'
  causation_id     TEXT             -- record/evidence ID that triggered this write
);
```

---

### 9.10 Schema Metadata

```sql
CREATE TABLE schema_meta (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

INSERT INTO schema_meta (key, value) VALUES ('schema_version', '2');

CREATE TABLE applied_migrations (
  id             INTEGER PRIMARY KEY,
  migration_file TEXT NOT NULL UNIQUE,
  applied_at     TEXT NOT NULL
);
```

`schema_meta` tracks the schema version. `applied_migrations` tracks which migration files have been applied, enabling safe incremental upgrades on existing databases.

---

### 9.11 FTS5 Tables and Sync Triggers

Lexical full-text search over semantic records and evidence. Both tables use external-content FTS5 synchronized to their source tables via triggers.

```sql
CREATE VIRTUAL TABLE memory_records_fts USING fts5(
  title,
  content,
  normalized_value,
  memory_class,
  scope       UNINDEXED,
  owner       UNINDEXED,
  status      UNINDEXED,
  content='memory_records',
  content_rowid='id',
  tokenize='porter unicode61'
);

CREATE VIRTUAL TABLE memory_evidence_fts USING fts5(
  title,
  content,
  evidence_type,
  speaker_role,
  scope            UNINDEXED,
  owner            UNINDEXED,
  promotion_status UNINDEXED,
  content='memory_evidence',
  content_rowid='id',
  tokenize='porter unicode61'
);
```

**Auto-sync triggers (REQUIRED):** External-content FTS tables must be kept in sync with their source tables. Strata requires these triggers rather than relying solely on maintenance-loop rebuilds:

```sql
-- memory_records FTS sync
CREATE TRIGGER memory_records_ai AFTER INSERT ON memory_records BEGIN
  INSERT INTO memory_records_fts(rowid, title, content, normalized_value,
    memory_class, scope, owner, status)
  VALUES (new.id, new.title, new.content,
    COALESCE(new.normalized_value, ''), new.memory_class,
    new.scope, COALESCE(new.owner, ''), new.status);
END;

CREATE TRIGGER memory_records_ad AFTER DELETE ON memory_records BEGIN
  INSERT INTO memory_records_fts(memory_records_fts, rowid, title, content,
    normalized_value, memory_class, scope, owner, status)
  VALUES ('delete', old.id, old.title, old.content,
    COALESCE(old.normalized_value, ''), old.memory_class,
    old.scope, COALESCE(old.owner, ''), old.status);
END;

CREATE TRIGGER memory_records_au AFTER UPDATE ON memory_records BEGIN
  INSERT INTO memory_records_fts(memory_records_fts, rowid, title, content,
    normalized_value, memory_class, scope, owner, status)
  VALUES ('delete', old.id, old.title, old.content,
    COALESCE(old.normalized_value, ''), old.memory_class,
    old.scope, COALESCE(old.owner, ''), old.status);
  INSERT INTO memory_records_fts(rowid, title, content, normalized_value,
    memory_class, scope, owner, status)
  VALUES (new.id, new.title, new.content,
    COALESCE(new.normalized_value, ''), new.memory_class,
    new.scope, COALESCE(new.owner, ''), new.status);
END;

-- memory_evidence FTS sync (same pattern)
CREATE TRIGGER memory_evidence_ai AFTER INSERT ON memory_evidence BEGIN
  INSERT INTO memory_evidence_fts(rowid, title, content, evidence_type,
    speaker_role, scope, owner, promotion_status)
  VALUES (new.id, COALESCE(new.title, ''), new.content, new.evidence_type,
    COALESCE(new.speaker_role, ''), new.scope,
    COALESCE(new.owner, ''), new.promotion_status);
END;

CREATE TRIGGER memory_evidence_ad AFTER DELETE ON memory_evidence BEGIN
  INSERT INTO memory_evidence_fts(memory_evidence_fts, rowid, title, content,
    evidence_type, speaker_role, scope, owner, promotion_status)
  VALUES ('delete', old.id, COALESCE(old.title, ''), old.content,
    old.evidence_type, COALESCE(old.speaker_role, ''), old.scope,
    COALESCE(old.owner, ''), old.promotion_status);
END;

CREATE TRIGGER memory_evidence_au AFTER UPDATE ON memory_evidence BEGIN
  INSERT INTO memory_evidence_fts(memory_evidence_fts, rowid, title, content,
    evidence_type, speaker_role, scope, owner, promotion_status)
  VALUES ('delete', old.id, COALESCE(old.title, ''), old.content,
    old.evidence_type, COALESCE(old.speaker_role, ''), old.scope,
    COALESCE(old.owner, ''), old.promotion_status);
  INSERT INTO memory_evidence_fts(rowid, title, content, evidence_type,
    speaker_role, scope, owner, promotion_status)
  VALUES (new.id, COALESCE(new.title, ''), new.content, new.evidence_type,
    COALESCE(new.speaker_role, ''), new.scope,
    COALESCE(new.owner, ''), new.promotion_status);
END;
```

**FTS index health check:** Before serving results, implementations SHOULD verify that FTS row counts are at least a minimum ratio (recommended: 0.5) of the corresponding source table counts. If degraded, trigger an immediate rebuild:

```sql
INSERT INTO memory_records_fts(memory_records_fts) VALUES('rebuild');
INSERT INTO memory_evidence_fts(memory_evidence_fts) VALUES('rebuild');
```

Log the repair as a `memory_updates` row with `update_type='integrity_check'`.

For `docs_fts` (markdown document indexing): this table is standalone (not external-content) because narrative documents live on disk, not in a SQLite table. Index documents into `docs_fts` explicitly at ingest time.

```sql
CREATE VIRTUAL TABLE docs_fts USING fts5(
  title,
  body,
  domain,
  source_path,
  tokenize='porter unicode61'
);
```

---

### 9.12 Recommended Indexes

```sql
-- memory_records: primary retrieval paths
CREATE INDEX idx_records_scope_type_status  ON memory_records(scope, owner, memory_class, status);
CREATE INDEX idx_records_slot_status        ON memory_records(slot_key, status);
CREATE INDEX idx_records_updated            ON memory_records(updated_at DESC);
CREATE INDEX idx_records_valid_to           ON memory_records(valid_to) WHERE valid_to IS NOT NULL;
-- Hierarchical routing (§9.2.2)
CREATE INDEX idx_records_domain_topic       ON memory_records(domain, topic, status);
CREATE INDEX idx_records_domain_scope       ON memory_records(scope, domain, memory_class, status);
-- Bi-temporal queries (§9.2.3)
CREATE INDEX idx_records_event_time         ON memory_records(event_time) WHERE event_time IS NOT NULL;

-- memory_evidence: promotion queue and session queries
CREATE INDEX idx_evidence_scope_status_created
  ON memory_evidence(scope, owner, promotion_status, created_at DESC);
CREATE INDEX idx_evidence_session_message   ON memory_evidence(session_id, message_id);

-- memory_links: provenance lookups
CREATE INDEX idx_links_record               ON memory_links(record_id);
CREATE INDEX idx_links_evidence             ON memory_links(evidence_id);

-- memory_embeddings: vector lookups
CREATE INDEX idx_embeddings_memory_id       ON memory_embeddings(memory_id);
CREATE INDEX idx_embeddings_layer           ON memory_embeddings(layer);
CREATE INDEX idx_embeddings_target          ON memory_embeddings(target_type, target_id);

-- scratchpad: session retrieval and expiry
CREATE INDEX idx_scratchpad_session         ON scratchpad_entries(scope, session_id, created_at DESC);
CREATE INDEX idx_scratchpad_expiry          ON scratchpad_entries(expires_at);

-- privacy_audit: access review
CREATE INDEX idx_privacy_accessor           ON privacy_audit(accessor, timestamp);
CREATE INDEX idx_privacy_accessed           ON privacy_audit(accessed_user, timestamp);
CREATE INDEX idx_privacy_session            ON privacy_audit(session_id);

-- memory_updates: audit and review queue
CREATE INDEX idx_updates_created            ON memory_updates(created_at, update_type);
CREATE INDEX idx_updates_target             ON memory_updates(target_table, target_id);
CREATE INDEX idx_updates_impact             ON memory_updates(impact_score, created_at);
```

---

### 9.13 Optional Module A: `tasks`

Task tracking is a natural fit for memory-governed agent systems, but `tasks` is the table most likely to collide with an existing host application schema.

```sql
CREATE TABLE tasks (
  id          INTEGER PRIMARY KEY,
  title       TEXT NOT NULL,
  description TEXT,
  status      TEXT NOT NULL DEFAULT 'open',  -- open | in_progress | blocked | completed | cancelled
  priority    TEXT DEFAULT 'normal',          -- low | normal | high | critical
  scope       TEXT DEFAULT 'shared',
  source_kind TEXT,
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL,
  due_at      TEXT
);

CREATE INDEX idx_tasks_status_priority ON tasks(status, priority, scope);
```

Implementers SHOULD evaluate whether to include `tasks` as-is, rename it (e.g., `strata_tasks`, `memory_tasks`), or omit it entirely and use the host application's existing task system. See §27 (Conformance) for guidance.

---

### 9.14 Optional Module B: `entities` and `relationships`

For deployments that need structured relational knowledge — network topologies, organizational structures, dependency graphs.

```sql
CREATE TABLE entities (
  id             INTEGER PRIMARY KEY,
  entity_type    TEXT NOT NULL,
  canonical_name TEXT NOT NULL,
  aliases_json   TEXT,    -- JSON array of alternate names
  scope          TEXT DEFAULT 'shared',
  status         TEXT DEFAULT 'active',  -- active | merged | deprecated
  created_at     TEXT NOT NULL
);

CREATE TABLE relationships (
  id                INTEGER PRIMARY KEY,
  subject_entity_id INTEGER NOT NULL,
  predicate         TEXT NOT NULL,
  object_entity_id  INTEGER NOT NULL,
  confidence        REAL DEFAULT 1.0,
  source_kind       TEXT,
  created_at        TEXT NOT NULL,
  valid_from        TEXT,
  valid_to          TEXT,
  FOREIGN KEY(subject_entity_id) REFERENCES entities(id),
  FOREIGN KEY(object_entity_id)  REFERENCES entities(id)
);
```

Aliases are stored as a JSON array in `aliases_json` rather than a normalized `entity_aliases` table. Alias resolution is handled at the application layer before queries. If alias search at query time is required, introduce a separate `entity_aliases` table in a future version.

---

## 10. Evidence Promotion Pipeline

The promotion pipeline is the bridge between raw episodic evidence and durable semantic records. It runs on `memory_evidence` rows with `promotion_status = 'pending'` and decides whether to promote them to `memory_records`.

### 10.1 Promotion Lifecycle

```
memory_evidence.promotion_status:

  pending
    ├── [negation gate hit]    → ignored
    ├── [pattern match found]  → promoted  (+ memory_records row created + memory_links row)
    ├── [no match]             → ignored   (or normalized, for future use)
    └── [explicit reject]      → rejected
```

The promotion runner processes pending evidence in creation order (FIFO). It SHOULD run:
- after each session ends (on session-end evidence),
- as a background task on a configurable interval,
- during the maintenance loop (§20).

### 10.2 Negation Gate

The negation gate is applied before pattern classification. Evidence containing negation language MUST be marked `ignored` and MUST NOT be promoted to a durable record.

Negation indicators (implementations SHOULD match these case-insensitively as whole words or phrases):
- `decided against`
- `will not` / `won't`
- `do not prefer` / `don't prefer`
- `no longer true`
- `not true`
- `changed my mind`
- `scratch that`

The negation gate exists to prevent contradictions from entering the semantic record plane. If an agent says "actually, I don't prefer verbose responses," that evidence should be ignored and the existing preference superseded via an explicit `memory_write` call rather than through auto-promotion of the negation statement.

### 10.3 Pattern Classification

If the negation gate is passed, classify the evidence by matching against configurable patterns. The following are recommended default pattern groups:

| Record Type | Example Trigger Phrases |
|---|---|
| `decision` | "we decided", "decided to", "final approach", "going with", "chosen" |
| `preference` | "prefer", "likes", "wants", "response style", "always use" |
| `profile` | "my name is", "I use", "I work on", "primary environment", "I am" |
| `fact` | (no pattern — use post-turn LLM extraction or explicit `memory_write` for facts) |

Pattern matching is intentionally conservative. When no pattern matches, the evidence is marked `ignored` — it is not silently dropped but is preserved in the archive for future re-evaluation. More sophisticated extraction (including fact extraction) should use the optional Post-Turn Extraction feature (§19).

Implementations MAY make the pattern set configurable per deployment.

### 10.4 Confidence Assignment

Assign confidence to promoted records based on the classification source:

| Source | Default Confidence |
|---|---|
| Explicit `memory_write` call | 1.0 |
| Decision pattern match | 0.85 |
| Preference pattern match | 0.80 |
| Profile pattern match | 0.75 |
| LLM extraction (post-turn) | 0.70–0.90 (model-assigned) |
| Semantic inference | 0.60 |

### 10.5 Slot-Key Assignment for Preferences

When promoting a `preference` record, derive the `slot_key` from the dominant subject of the preference (e.g., "response style" → `slot_key = 'response_style'`). Check for an existing `active` preference with the same `(scope, owner, slot_key)`. If one exists with a different `normalized_value`, supersede it atomically (§9.2.1). Write a `memory_updates` row with `update_type = 'supersede'`.

### 10.6 Provenance Linking

After creating a `memory_records` row from evidence, write a `memory_links` row:
```
record_id   = <new memory_records.id>
evidence_id = <source memory_evidence.id>
link_type   = 'supports'
```

Update the evidence row: `promotion_status = 'promoted'`, `promotion_checked_at = <now>`.

---

## 11. Admission Gate

Every durable write to `memory_records` or `memory_evidence` passes through the admission gate.

The gate operates on two paths:

**Hot path:** Invoked during an active agent session. Must be fast and non-blocking. Writes that require human review are placed as `hold_for_review` rather than blocking the session.

**Background path:** Invoked by the maintenance loop and bulk import operations. May include slower contradiction detection and consolidation.

### 11.1 Memory Classes at Ingestion

| Class | Destination |
|---|---|
| Exact fact, preference, decision, profile | `memory_records` (direct write) |
| Conversation turn, tool output, event | `memory_evidence` (pending, for promotion) |
| Narrative summary | Markdown file + `docs_fts` index |
| Volatile context | Scratchpad (`scratchpad_entries`) or not persisted |

### 11.2 Admission Outcomes

| Outcome | Meaning |
|---|---|
| `accept_new` | Written as new active record |
| `accept_update` | Updated in place (only for mutable fields) |
| `accept_supersede` | Old record superseded, new record written |
| `accept_retract` | Record marked retracted |
| `reject_volatile` | Rejected as not worth persisting |
| `hold_for_review` | Parked for operator confirmation (high impact or conflict) |

### 11.3 Impact Score

Every `memory_updates` write SHOULD compute an `impact_score` at admission time:

```
impact_score = (0.5 × persistence) + (0.3 × reach) + (0.2 × sensitivity)
```

| Component | 0.0 | 0.5 | 1.0 |
|---|---|---|---|
| **persistence** | Ephemeral / volatile | Session-durable | Permanent / constitutional |
| **reach** | Local only | Shared scope | External egress or broadcast |
| **sensitivity** | Generic text | Operational data | PII / credentials / secrets |

Reference scores for common write types:

| Write type | score |
|---|---|
| Volatile context (rejected) | 0.00 |
| Evidence (conversation turn) | 0.25 |
| Semantic record (normal fact) | 0.50 |
| Preference write | 0.54 |
| Record supersession | 0.56 |
| Record retraction | 0.58 |
| Sensitive record (PII domain) | 0.70 |
| Constitutional memory write | 0.80 |
| Constitutional memory retraction | 1.00 |

High-impact writes (`impact_score >= 0.7`) MAY trigger `hold_for_review` unless they come from a trusted source (operator correction, structured import).

---

## 12. Governance Rules

### Rule 1: Records Are Never Silently Overwritten
When a semantic record changes, the old record is marked `superseded`, a new record is inserted, and `supersedes_record_id` links them. Silent in-place overwrites are prohibited.

### Rule 2: Inferred Memories Are Labeled
If a memory is inferred (from evidence promotion, LLM extraction, or semantic inference) rather than directly confirmed, it MUST carry a lower confidence score or `uncertain`-equivalent status. Inferred and confirmed memories must be distinguishable.

### Rule 3: Source Authority Matters
Precedence for resolving conflicts:
1. Explicit operator correction
2. Structured record from a trusted source
3. Operator-curated markdown
4. Recent episodic evidence
5. Pattern-promoted inference
6. Semantic inference

Higher-authority sources SHOULD be able to supersede lower-authority records without requiring operator confirmation.

### Rule 4: Stale Memory Is Marked, Not Blindly Deleted
Old information may still be useful for temporal reasoning. Before removing records, mark them `stale` (or set `valid_to`) and surface them via freshness annotations (§14.8). Consider removal only after operator review.

### Rule 5: Protected Domains Require Stricter Writes
Credentials, secrets, PII, and sensitive operational details MUST require stronger admission gates, masking at retrieval, and restricted display. Writes to these domains SHOULD require `impact_score >= 0.7` confirmation.

### Rule 6: Narrative and Exact Facts Must Not Be Conflated
A markdown note can mention a fact, but the canonical authoritative value MUST live in `memory_records` if the system must rely on it operationally. Narrative is supplementary context, not the source of truth.

### Rule 7: Pre-Compaction Flush
Before any context compaction event, the agent runtime MUST flush all session-accumulated memory writes to persistent storage. Unwritten session context MUST NOT be silently dropped. Flush with `update_type = 'compaction_flush'` before compaction proceeds.

### Rule 8: Negation Gate
Evidence containing negation or retraction language MUST be suppressed before promotion and MUST NOT be auto-promoted to a durable record. See §10.2 for the full gate definition. Suppressed evidence is marked `ignored` and preserved in the archive — it is not deleted.

### Rule 9: Query Gate
Memory search MUST NOT be triggered for queries that cannot yield useful results. Implementations SHOULD skip searches when:
- the query is shorter than a minimum useful length (recommended: 10 characters),
- the query contains no alphanumeric characters,
- the query matches a credential pattern (e.g., API keys, bearer tokens, long hex strings),
- the query is a trivial acknowledgment ("ok", "thanks", "yes", "no", "got it").

Log skipped queries in `memory_updates` with `update_type = 'retrieval_mode_decision'` and `reason = 'query_gate_skip'` for observability.

---

## 12a. Lightweight Integrity Chain *(Optional Hardening)*

The integrity chain is **recommended but not required for conformance**. It is operational hardening for deployments that need tamper evidence.

On every write to `memory_updates`, the implementation SHOULD:

1. Retrieve the `row_hash` of the most recent existing row (the "tail").
2. Set `prev_hash` = that tail hash (or `NULL` for the genesis row).
3. Compute `row_hash` = SHA-256 of the canonical serialization of this row's content.

**Canonical serialization:** JSON with keys sorted alphabetically, no whitespace (consistent with JCS / RFC 8785).

**Fields included in the hash:** All content fields — `update_type`, `target_table`, `target_id`, `prior_value_json`, `new_value_json`, `reason`, `source_id`, `created_at`, `impact_score`, `prev_hash`, `request_id`, `correlation_id`, `causation_id`. Exclude `id` and `row_hash` itself.

```python
# Pseudocode
def compute_row_hash(row: dict) -> str:
    fields = {
        "causation_id":     row.get("causation_id"),
        "correlation_id":   row.get("correlation_id"),
        "created_at":       row["created_at"],
        "impact_score":     row.get("impact_score"),
        "new_value_json":   row.get("new_value_json"),
        "prev_hash":        row.get("prev_hash"),
        "prior_value_json": row.get("prior_value_json"),
        "reason":           row.get("reason"),
        "request_id":       row.get("request_id"),
        "source_id":        row.get("source_id"),
        "target_id":        row.get("target_id"),
        "target_table":     row["target_table"],
        "update_type":      row["update_type"],
    }
    canonical = json.dumps(fields, sort_keys=True, separators=(',', ':'))
    return sha256(canonical.encode('utf-8')).hexdigest()
```

Chain verification during maintenance:

```python
def verify_chain(rows: list[dict]) -> bool:  # rows ordered by id ASC
    prev = None
    for row in rows:
        if row["prev_hash"] != prev:
            return False   # chain broken
        if row["row_hash"] != compute_row_hash(row):
            return False   # row tampered
        prev = row["row_hash"]
    return True
```

A broken chain SHOULD be logged as an `integrity_violation` event and flagged for operator review.

---

## 13. Agent Tool Interface

Strata exposes memory capabilities to the agent through five tools. Runtimes integrate these into the agent's toolset. Each tool is language-agnostic — the interface is defined by its input/output shape.

### 13.1 `memory_search`

Search memory across all layers using a query string. Subject to the Query Gate (Rule 9).

**Input:**
```json
{
  "query": "string",
  "layers": ["records", "evidence", "docs"],   // optional; defaults to all
  "memory_class": "string",                     // optional filter on memory_records.memory_class
  "scope": "string",                           // optional; defaults to caller's scope
  "include_history": false,                    // if true, includes superseded/retracted
  "keyword_only": false,                       // if true, force FTS_ONLY mode
  "limit": 10                                  // optional; default 5
}
```

**Default retrieval filters** (unless `include_history: true`):
- `memory_records.status = 'active'`
- `memory_evidence.promotion_status NOT IN ('ignored', 'rejected')`
- Records where `valid_to IS NOT NULL AND valid_to < now()` are excluded

**Output:**
```json
{
  "results": [
    {
      "layer": "records | evidence | docs",
      "id": "integer",
      "summary": "string",
      "confidence": 0.0,
      "source_kind": "string",
      "created_at": "ISO8601",
      "status": "string",
      "freshness_caveat": "string | null"       // see §14.8
    }
  ],
  "retrieval_mode": "local_hybrid | hybrid | fts_only"
}
```

Internally executes the retrieval pipeline (§14). Does not assemble an evidence packet — use `memory_get` with `topic` for that.

---

### 13.2 `memory_get`

Retrieve a specific record by ID, or fetch a curated evidence packet for a topic.

**Input (by ID):**
```json
{
  "layer": "records | evidence | docs",
  "id": "integer"
}
```

**Input (evidence packet):**
```json
{
  "topic": "string",
  "scope": "string"   // optional
}
```

**Output (by ID):** Full row content for the requested record.

**Output (evidence packet):**
```json
{
  "records": [...],
  "evidence": [...],
  "narrative_summary": "string",
  "provenance": [{"source_kind": "string", "source_ref": "string"}],
  "retrieval_mode": "local_hybrid | hybrid | fts_only"
}
```

---

### 13.3 `memory_write`

Write a new memory item. Passes through the admission gate (§11).

**Input:**
```json
{
  "layer": "records | evidence | scratchpad",
  "content": { ... },           // layer-appropriate fields (see examples below)
  "reason": "string",
  "source_kind": "string",
  "scope": "string",            // defaults to caller's scope
  "request_id": "string",       // optional session lineage
  "correlation_id": "string",
  "causation_id": "string"
}
```

**Example `content` for a semantic record:**
```json
{
  "memory_class": "fact",
  "title": "switch-core-01 IP address",
  "content": "switch-core-01 has IP address 10.10.1.5",
  "normalized_value": "10.10.1.5",
  "confidence": 1.0
}
```

**Example `content` for a preference:**
```json
{
  "memory_class": "preference",
  "slot_key": "response_style",
  "title": "Response style preference",
  "content": "User prefers concise responses without preamble",
  "normalized_value": "concise",
  "confidence": 0.9
}
```

**Example `content` for evidence:**
```json
{
  "evidence_type": "tool_execution",
  "title": "nmap scan on VLAN 10",
  "content": "Ran nmap scan on 10.10.1.0/24, found 6 hosts up",
  "session_id": "session-abc123",
  "message_id": 42,
  "timestamp": "2026-04-27T14:22:00Z"
}
```

**Output:**
```json
{
  "outcome": "accept_new | accept_update | accept_supersede | reject_volatile | hold_for_review",
  "id": "integer | null",
  "impact_score": 0.0,
  "message": "string"
}
```

---

### 13.4 `memory_retract`

Mark a semantic record as retracted. Does not delete.

**Input:**
```json
{
  "layer": "records",
  "id": "integer",
  "reason": "string"
}
```

**Output:**
```json
{
  "outcome": "retracted | hold_for_review",
  "impact_score": 0.0,
  "message": "string"
}
```

High-impact retractions (`impact_score >= 0.7`) MAY require operator confirmation via `memory_confirm`.

---

### 13.5 `memory_confirm`

Approve or reject a `hold_for_review` write. For operator-in-the-loop workflows.

**Input:**
```json
{
  "update_id": "integer",
  "decision": "approve | reject",
  "reason": "string"
}
```

**Output:**
```json
{
  "outcome": "approved | rejected",
  "message": "string"
}
```

---

### 13.6 `memory_reflect`

Adjust the confidence of an `opinion` or `belief` record in response to new corroborating or contradicting evidence. This is the primary interface to the Reflect layer (§20.10).

**Input:**
```json
{
  "record_id": "integer",             // ID of the opinion/belief record to reflect on
  "direction": "strengthen | weaken | neutral",
  "magnitude": 0.1,                   // optional; 0.0–0.3, default 0.1
  "evidence_id": "integer | null",    // optional: evidence that triggered this reflection
  "reason": "string"
}
```

**Behavior:**
- `strengthen`: `new_confidence = min(1.0, confidence + magnitude)`
- `weaken`: `new_confidence = max(0.0, confidence - magnitude)`
- `neutral`: no confidence change; logs the reflection as an observation without update

The record is updated in-place (confidence and `updated_at`). A `memory_updates` row is written with `update_type = 'belief_update'`, `prior_value_json = {"confidence": <old>}`, `new_value_json = {"confidence": <new>, "direction": "<dir>", "evidence_id": <id>}`.

If `evidence_id` is provided, a `memory_links` row is written connecting the evidence to the record with `link_type = 'supports'` (for strengthen) or `link_type = 'contradicts'` (for weaken).

If confidence drops below 0.3 and `direction = 'weaken'`, the implementation SHOULD also mark the record `status = 'uncertain'` (where supported) or surface it in the review queue.

**Output:**
```json
{
  "outcome": "updated | no_change | hold_for_review",
  "prior_confidence": 0.0,
  "new_confidence": 0.0,
  "message": "string"
}
```

This tool is not subject to the Query Gate (Rule 9). It is a write operation, not a search.

---

### 13.7 `memory_compact`

Returns the current review queue — records that may need attention — so the agent or operator can take action using existing tools. `memory_compact` is read-only: it writes nothing and modifies nothing. All follow-up actions use `memory_retract`, `memory_reflect`, or `memory_confirm`.

**Input:**
```json
{
  "limit": 20,
  "include_types": ["low_confidence", "contradiction", "stale", "scratchpad_candidate"]
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `limit` | integer | no | 20 | Maximum review items to return |
| `include_types` | array | no | all types | Filter to specific flag reasons |

**Flag reason values:**

| Value | Triggers when |
|---|---|
| `low_confidence` | `opinion` or `belief` with `confidence < 0.5` |
| `contradiction` | Active records with conflicting `memory_links` (`contradicts` links) |
| `stale` | `importance >= 0.7` and `updated_at` older than 30 days |
| `scratchpad_candidate` | `scratchpad_entries` with substantive content that may warrant promotion |
| `unresolved_hold` | Records with `status = 'held'` awaiting `memory_confirm` |

**Output:**
```json
{
  "review_items": [
    {
      "record_id": "uuid",
      "content": "User prefers dark mode interfaces",
      "memory_class": "belief",
      "confidence": 0.38,
      "flag_reason": "low_confidence",
      "suggested_action": "strengthen | weaken | verify | retract | promote",
      "age_days": 47
    }
  ],
  "count": 3,
  "queue_generated_at": "2025-03-15T10:00:00Z"
}
```

The agent SHOULD review these items and call `memory_reflect` to adjust confidence, `memory_retract` to remove stale items, or `memory_confirm` to resolve held records. Nothing happens automatically — this tool surfaces what the maintenance loop found; action is the agent's or operator's choice.

**When to call:** Optionally at session end (§5c.2), or any time the agent wants to proactively maintain memory quality. The maintenance loop (§20.8) generates the review queue on its own schedule; `memory_compact` simply exposes it.

---

## 14. Retrieval Model

### 14.1 Retrieval Modes

Strata defines three retrieval modes. The mode is determined at query time:

| Mode | Description |
|---|---|
| `LOCAL_HYBRID` | FTS5 + sqlite-vec in-process ANN search. Zero external dependencies. **Recommended default when sqlite-vec is available.** |
| `HYBRID` | FTS5 + external embedding backend vector search. For deployments using a remote embedding API. |
| `FTS_ONLY` | FTS5 lexical search only. Used when no vector backend is available. Always available. |

**Mode selection priority (highest wins):**
1. Caller passes `keyword_only: true` → always `FTS_ONLY`
2. Environment/config override (e.g., `STRATA_RETRIEVAL_MODE=fts_only`) → honor it
3. sqlite-vec extension loaded and healthy → `LOCAL_HYBRID`
4. External embedding backend healthy (circuit breaker closed) → `HYBRID`
5. All vector backends unavailable → `FTS_ONLY`

Log every mode decision as a `memory_updates` row with `update_type = 'retrieval_mode_decision'`. Include the trigger reason in `reason`. Do NOT include the raw query or user ID in the audit row — use only operational metadata.

### 14.2 Embedding Backend Circuit Breaker

Protects the hot retrieval path from cascading failures when the embedding backend is slow or unavailable. Also handles sqlite-vec extension availability at startup.

**sqlite-vec startup probe (run once at service start):**
```python
def probe_sqlite_vec(conn) -> bool:
    try:
        conn.execute("SELECT vec_version()")
        return True
    except Exception:
        return False
```

If the probe returns `True`, set the retrieval mode default to `LOCAL_HYBRID` for the lifetime of this service instance. No further probing needed — the extension either loaded at startup or it did not.

**External backend circuit breaker (applies to `HYBRID` mode only):**

**Configuration (recommended defaults):**
- Probe cache TTL: 60 seconds (re-probe only after cache expires)
- Circuit open threshold: 3 consecutive probe failures
- Half-open interval: 300 seconds (time before allowing a probe after circuit opens)

**Behavior:**
- On each retrieval call, if probe cache is valid and healthy: use `HYBRID`.
- If probe cache is expired: run a liveness probe (a minimal embedding request against a known string).
- If probe succeeds: reset failure count, update cache, use `HYBRID`.
- If probe fails: increment failure count. If count >= threshold, open circuit. Use `FTS_ONLY`.
- After half-open interval: allow a single probe. If it succeeds, close circuit. If it fails, restart interval.

```
State machine:
  CLOSED → (3 failures) → OPEN → (300s) → HALF_OPEN → (probe ok) → CLOSED
                                                       → (probe fail) → OPEN
```

### 14.3 Query Gate

Before executing any retrieval, apply the Query Gate (Rule 9, §12). Skip search and return an empty result set when:
- query length < minimum useful length (recommended: 10 characters),
- query has no alphanumeric characters,
- query matches a credential pattern (API keys, bearer tokens, hex strings > 32 chars),
- query is a trivial acknowledgment.

Log skips in `memory_updates` for observability.

### 14.4 Query Sanitization

Before passing query text to FTS5, sanitize it:
1. Strip any embedded personalization or context markers (e.g., if the runtime appends context to query strings, strip the marker and everything after it before searching).
2. Extract alphanumeric tokens only (strip punctuation and special characters).
3. Cap to a maximum of 12 tokens to bound FTS5 query complexity and prevent injection.

```python
# Pseudocode
def sanitize_fts_query(query: str) -> str:
    tokens = re.findall(r'[A-Za-z0-9_]+', query)
    return ' '.join(tokens[:12])
```

This prevents FTS5 syntax injection and bounds performance.

### 14.5 Stage 1: Candidate Collection

Collect candidates from multiple sources in parallel:

1. **Exact SQLite queries** — `normalized_value` exact match, slot-key lookup, ID lookup.
2. **FTS5 lexical search** — `memory_records_fts`, `memory_evidence_fts`, `docs_fts`. Use BM25 ranking (`bm25(memory_records_fts)`).
3. **Vector similarity** — `LOCAL_HYBRID`: sqlite-vec ANN via `memory_vecs`; `HYBRID`: cosine similarity from `memory_embeddings` via external backend.
4. **Markdown retrieval** — `docs_fts` search on indexed narrative documents.

Apply default filters at this stage (unless `include_history: true`):
- `memory_records.status = 'active'`
- Records where `valid_to < now()` are excluded
- Scope filter: queries default to caller's scope

### 14.6 Stage 2: Signal Fusion and Reranking

Strata uses **Reciprocal Rank Fusion (RRF)** to merge multiple retrieval signals into a single ranked list. RRF is the required merging strategy for all multi-signal retrievals.

**RRF formula:**

```
RRF_score(d) = Σ  1 / (k + rank_i(d))
               i
```

Where `d` is a candidate record, `rank_i(d)` is the 1-based rank of `d` in signal list `i` (records absent from a list contribute 0), and `k = 60` (standard constant).

**Core signals (always computed):**

| Signal | Source | Notes |
|---|---|---|
| BM25 lexical | FTS5 `bm25()` score | Always available |
| Confidence | `memory_records.confidence` | Rank descending |
| Temporal proximity | `COALESCE(event_time, created_at)` | Rank by recency descending |

**Extended signal (HYBRID mode):**

| Signal | Source | Notes |
|---|---|---|
| Semantic similarity | Cosine distance from `memory_embeddings` | Only when embedding backend healthy |

**Optional signals (enable for higher accuracy):**

| Signal | Source | Notes |
|---|---|---|
| Link density | Count of `memory_links` rows pointing to this record | Node importance proxy |
| Domain/topic match | Exact match on `domain`/`topic` vs. session context | Positional boost |

**Algorithm pseudocode:**

```python
def rrf_merge(signal_lists: list[list[Record]], k: int = 60) -> list[Record]:
    scores = {}
    active = [lst for lst in signal_lists if lst]
    for lst in active:
        for rank, record in enumerate(lst, start=1):
            scores[record.id] = scores.get(record.id, 0) + 1.0 / (k + rank)
    all_records = {r.id: r for lst in active for r in lst}
    return sorted(all_records.values(), key=lambda r: scores[r.id], reverse=True)
```

**Post-fusion adjustments (apply after RRF, before returning):**
- Exact `normalized_value` match: add a fixed boost of `1/(k+1)` (equivalent to rank-1 in an additional signal list)
- Record under contradiction review: multiply score by 0.5
- Record type mismatch with query classification: multiply score by 0.8
- Slot diversity: if multiple records share the same `slot_key`, keep only the highest-scoring one

### 14.7 Stage 3: Evidence Packet Assembly

Return a compact, curated packet to the reasoning model. Do NOT dump raw logs or entire document contents into context.

Recommended packet contents:
- 1–3 exact records (highest-ranked, active only),
- 1–2 recent evidence items (if evidence layer is included),
- 1 short narrative summary (if docs layer is included),
- source/provenance metadata for internal traceability,
- `retrieval_mode` field indicating which mode was used: `local_hybrid`, `hybrid`, or `fts_only`.

### 14.8 Freshness Annotations

Attach a `freshness_caveat` to results that may be stale. The caveat is a human-readable string included in the result for the agent to surface when relevant.

Recommended staleness thresholds:

| Record type | Caveat after N days |
|---|---|
| `event` / tool execution | 1 day |
| `preference` | 30 days |
| `fact` / general | 2 days |

Caveat text by age:
- Under threshold: no caveat (`null`)
- Past threshold, under 30 days: `"[N days ago] May be outdated — verify before asserting as fact."`
- 30+ days: `"[N days ago — potentially stale] Point-in-time observation. Verify against current state before asserting as fact."`

Freshness annotations are advisory. The agent SHOULD use them to decide whether to re-verify information before acting on it.

### 14.9 Optional LLM Reranker

When result count exceeds a minimum threshold (recommended: 4), an optional LLM reranker can be invoked to select the best results from a capped candidate set.

**Configuration:**
- Activates when candidate count > threshold (default: 4)
- Maximum candidates passed to the LLM (default: 10)
- Timeout: abort and return original results if LLM takes longer than N seconds (default: 3s)
- Always fall back to Stage 2 ranking on any error or malformed response

The reranker prompt asks the LLM to select the most relevant results from the candidate list and return a JSON array of selected IDs. Parse the response strictly; on parse failure, use original ranking.

This feature is opt-in. Most deployments should not need it; Stage 2 ranking is sufficient for well-structured memory.

### 14.10 FTS Index Health Check

Before executing an FTS query, verify that the FTS indexes are healthy:

```python
# Pseudocode
def check_fts_health(conn, min_ratio=0.5) -> bool:
    records_count = conn.execute("SELECT count(*) FROM memory_records").fetchone()[0]
    fts_count = conn.execute("SELECT count(*) FROM memory_records_fts").fetchone()[0]
    if records_count > 0 and fts_count < records_count * min_ratio:
        return False  # degraded
    return True
```

If degraded, repair before querying:
```sql
INSERT INTO memory_records_fts(memory_records_fts) VALUES('rebuild');
```

Log the repair as a `memory_updates` row with `update_type = 'integrity_check'` and `reason = 'fts_index_repair'`.

---

## 15. Query Planning

Before retrieval, classify the incoming query to route it efficiently.

| Query class | Primary retrieval path |
|---|---|
| Exact fact lookup | `normalized_value` exact match → domain/topic filter → FTS → vector |
| Temporal lookup | `event_time` + `valid_from`/`valid_to` windows + evidence `timestamp` ordering |
| Bi-temporal lookup | Filter by `event_time` range for "what was true before X"; filter by `created_at` range for "what did we know before X" |
| Preference lookup | Slot-key lookup (`slot_key`, `owner`, `scope`) |
| Procedural lookup | `docs_fts` (markdown knowledge base) |
| Broad synthesis | Domain/topic filter → hybrid search + reranking → evidence packet |
| Relationship lookup | Optional Module B (entities/relationships) + narrative supplements |
| Profile/identity lookup | `memory_class = 'profile'` filter + scope filter |
| Opinion/belief lookup | `memory_class IN ('opinion', 'belief')` + domain filter + confidence ordering |
| Cross-topic synthesis | Domain filter first (e.g., `domain='network'`), then full-text within that domain |

For exact fact lookups: prefer structured queries first. Only fall back to FTS or vector when exact queries return no results.

For preference lookups: always use slot-key lookup for performance; FTS is the fallback when slot-key is unknown.

**Hierarchical routing:** When a query can be classified into a domain (and optionally a topic), apply that filter at Stage 1 before FTS or vector search. This dramatically narrows the candidate set and mirrors the retrieval efficiency of ByteRover's Context Tree. Domain classification can use simple keyword matching on the query string (e.g., "VLAN" → `domain='network'`, "response style" → `domain='user_preferences'`) or a lightweight LLM classification step.

**Bi-temporal routing:** Distinguish between two temporal query types:
- *"What was true at time T?"* → filter by `event_time <= T AND (valid_to IS NULL OR valid_to >= T)`
- *"What did we know as of time T?"* → filter by `created_at <= T`

---

## 16. Optional Relational Model

Strata intentionally avoids a heavy external graph dependency.

If Optional Module B is enabled, the `entities` and `relationships` tables support:
- person → project
- device → network segment
- file → project
- task → owner
- system → dependency

If future needs require multi-hop graph exploration at large scale, an external graph layer can be added. It is not required for the initial implementation.

---

## 17. Session Injection and Deduplication

### 17.1 Session Injection Rules

When a session begins, the runtime MAY inject recent memory into the agent's context. Injection must be controlled:

- Inject only what is relevant to the current task or topic.
- Never inject the full evidence log or all active records.
- Use the evidence packet format (§14.7) as the default injection shape.
- Constitutional memory is always available but SHOULD NOT be re-injected verbatim unless the session requires it.
- Apply freshness annotations (§14.8) to injected facts.

### 17.2 Deduplication

Before injecting memory into a session:

1. Compare incoming candidates against the current session context.
2. If a record or evidence item was already injected earlier in the same session, skip it.
3. Track injected IDs in a session-scoped dedup set (ephemeral; not persisted to the database).
4. When the same topic appears in multiple layers, prefer the highest-authority, most-recent record and suppress lower-priority duplicates.

Deduplication state is not persisted between sessions.

---

## 18. Idempotent Session Ingestion

When ingesting session messages into `memory_evidence`:

1. Read the `evidence_ingestion_cursors` row for `(scope, session_id)`. If not found, treat `last_message_id = 0`.
2. Insert only `memory_evidence` rows with `message_id > last_message_id`. The `UNIQUE(scope, session_id, message_id)` constraint provides a safety net.
3. After a successful batch, update `evidence_ingestion_cursors` with the new `last_message_id`.
4. This makes the ingest operation fully idempotent: calling it multiple times on the same session is safe.

Use separate cursor scopes for different ingestion processes (e.g., `'__post_turn_extractor__'`) so they track their own read watermarks independently.

---

## 19. Optional Post-Turn Extraction

After each agentic turn, an optional LLM-driven extraction pass can identify and write durable memories from the turn's content.

**When to use:** When the pattern-based promotion pipeline (§10) is insufficient for extracting facts from natural language. The LLM extractor can identify structured facts that regex patterns cannot.

**How it works:**
1. After the turn completes, fetch recent session messages (up to the turn boundary).
2. Send them to the LLM with an extraction prompt requesting a JSON array of memory candidates: `[{"type": "fact|preference|decision|profile", "content": "...", "confidence": 0.0–1.0}]`.
3. For each valid candidate, write it via the `memory_write` tool path (through the admission gate).
4. Skip if the main agentic loop already wrote memory during the turn (avoid duplicates).
5. Track progress via `evidence_ingestion_cursors` with a dedicated scope key (e.g., `__extractor__`).

**Safeguards:**
- Parse the LLM response strictly; discard malformed or empty outputs.
- Apply a maximum candidate count per turn (recommended: 10).
- Do not re-process turns already covered by the cursor.
- Apply the negation gate (Rule 8) to extracted candidates before writing.

This feature is **opt-in**. Disable it in resource-constrained or latency-sensitive deployments.

---

## 20. Maintenance Loop

Run a scheduled maintenance process during low-activity windows (e.g., nightly).

### 20.1 Deduplication
- Identify exact duplicate `memory_records` rows (same `scope`, `owner`, `memory_class`, `content`). Keep the most recent; retract duplicates. Log in `memory_updates`.
- Flag near-duplicates (similar content, same slot key) for operator review.

### 20.2 Promotion Pass
- Run the promotion pipeline (§10) on all remaining `pending` evidence rows.
- This catches evidence that was not processed at session end.

### 20.3 Summary Consolidation
- Identify long episodic sequences that can be summarized.
- Write a `memory_records` row with `memory_class = 'summary'` referencing the source evidence IDs.
- Preserve links via `memory_links`.

### 20.4 FTS Reindexing
- Verify FTS index health (§14.10). Rebuild if degraded.
- Refresh `docs_fts` with any newly added markdown documents.
- Refresh vector indexes if embeddings are enabled.

### 20.5 Contradiction Detection
- Identify conflicts such as:
  - two `active` records with the same `slot_key` for the same owner,
  - a preference and a contradicting profile record for the same owner,
  - facts with overlapping `valid_from`/`valid_to` windows claiming different values.
- Flag conflicts for operator review.

### 20.6 Preference Decay
- Adjust `confidence` or `importance` weighting for preferences not confirmed in a long time.
- Do not delete; mark as potentially stale via `freshness_caveat` at retrieval time.

### 20.7 Scratchpad Purge
- Delete all `scratchpad_entries` where `expires_at < now()`.

### 20.8 Review Queue Generation
- Generate a review list for:
  - low-confidence records (`confidence < 0.6`),
  - unresolved contradictions,
  - stale high-impact records (`importance >= 8` and `updated_at` older than 30 days),
  - broken integrity chain entries (if chain is enabled).

### 20.9 Integrity Chain Verification *(if enabled)*
- Verify `row_hash` / `prev_hash` chain over all `memory_updates` rows.
- Log any violations as evidence rows with `evidence_type = 'correction'` and `metadata_json` referencing the broken link.
- Add to the review queue.

### 20.10 Reflect Pass — Opinion and Belief Maintenance

The Reflect pass is the Strata equivalent of Hindsight's CARA layer. It examines `opinion` and `belief` records and adjusts their confidence based on newly accumulated evidence since the last reflect pass. This is what transforms Strata from a static fact store into a system with a consistent, evolving perspective.

**When to run:**
- After the evidence promotion pass (§20.2) completes — new evidence may corroborate or contradict existing opinions.
- Optionally as a dedicated background agent invoked via `memory_reflect` (§13.6) after high-activity sessions.

**Algorithm:**

For each `active` record with `memory_class IN ('opinion', 'belief')`:

1. **Gather supporting evidence:** Query `memory_links` for evidence rows with `link_type = 'supports'` or `link_type = 'contradicts'` written since `last_confirmed_at` (or since the record's `created_at` if never confirmed).
2. **Count signals:** `support_count` = rows with `link_type = 'supports'`; `contradict_count` = rows with `link_type = 'contradicts'`.
3. **Adjust confidence:**
   - Net positive (`support_count > contradict_count`): call `memory_reflect` with `direction = 'strengthen'`, `magnitude = min(0.15, 0.05 * net_positive)`.
   - Net negative (`contradict_count > support_count`): call `memory_reflect` with `direction = 'weaken'`, `magnitude = min(0.20, 0.05 * net_negative)`.
   - Balanced or no new signals: call `memory_reflect` with `direction = 'neutral'`.
4. **Update `last_confirmed_at`** to now after processing.
5. **Promotion rule:** When a `opinion` record reaches `confidence >= 0.85` and has been active for at least 30 days, the Reflect pass MAY upgrade it to `memory_class = 'belief'` (via supersession). This represents a subjective assessment that has been consistently corroborated over time.
6. **Demotion rule:** When a `belief` record drops below `confidence = 0.30`, the Reflect pass SHOULD supersede it with a new `opinion` record at the current confidence level.

**Observer pattern (optional, async):** Implementations MAY run a lightweight "Observer" agent that watches session activity in real time and calls `memory_reflect` immediately after a piece of evidence strongly corroborates or contradicts a known opinion, rather than waiting for the nightly batch.

The Reflect pass MUST NOT modify `fact`, `preference`, `decision`, or `profile` records — confidence updates for those types go through the normal `supersede` path.

### 20.11 Markdown Sync

Re-index the Markdown knowledge base to keep `docs_fts` current with any files added, edited, or deleted since the last maintenance run.

**Steps:**
1. Walk `$STRATA_DATA_PATH/kb/` recursively. For each `.md` file, compare `mtime` against the last-indexed timestamp (stored in `schema_meta` as `last_kb_sync`).
2. For files newer than `last_kb_sync`: re-index into `docs_fts` (delete existing rows for that path, re-insert).
3. For files that have been deleted since last sync: remove their `docs_fts` rows.
4. Update `last_kb_sync` in `schema_meta`.

**Optional promotion pass:** If `promote = true` (or enabled in deployment config), run lightweight pattern classification (§10.3) on newly indexed content. Obvious facts, preferences, and decisions found in Markdown may be promoted to `memory_records` via the normal admission gate. The Markdown file path is stored as `source_id` on promoted records for provenance.

**On-demand vs. scheduled:** The nightly maintenance loop runs this step automatically. The `POST /memory/sync` endpoint (§5b.13) and `memory_sync` MCP tool (§5d.4) allow on-demand invocation — useful when a user has just edited files in `/data/kb/` and wants the changes reflected immediately.

**Bidirectional use:** Markdown files in `/data/kb/` are human-editable by design. Users and operators can edit them directly; this sync step ensures the structured search layer stays consistent with those edits. The structured `memory_records` plane remains the governed source of truth for facts and preferences; Markdown is the narrative and human-readable layer.

---

## 21. Security and Privacy Controls

### 21.1 Scope Enforcement
All queries MUST apply a scope filter by default. Shared-scope records are readable by all. User-scoped records are readable only by agents serving that user, or by an operator with explicit elevated permission.

Scope is the PRIMARY mechanism for preventing memory bleed between users in multi-user deployments. Any retrieval that skips scope filtering is a privacy violation.

**Hard cross-user isolation rule:** A query carrying `user_id: X` MUST NEVER return records scoped to `user: Y` where Y ≠ X. This is non-negotiable. It is enforced at the query layer, not left to the application to remember. The SQL predicate on every retrieval MUST include a scope clause that makes cross-user access structurally impossible, not just unlikely.

See §5a for the full deployment model, scope resolution rules, and scope reference table.

### 21.2 PII Gate for Shared Memory
Before writing any record to `shared` scope, the implementation SHOULD scan the content for PII indicators (email addresses, phone numbers, SSNs, IP addresses that identify individuals, etc.). Reject or quarantine records that contain PII before they reach shared scope. User-scoped records may contain PII.

### 21.3 Sensitive Domain Handling
Sensitive memory (credentials, secrets, access tokens, security configurations) MUST:
- be masked at retrieval time (return `[REDACTED]` or `[MASKED]` rather than the raw value),
- require stronger write controls (impact_score >= 0.7 threshold),
- be excluded from vector indexing and FTS indexing,
- be logged in `privacy_audit` on every access.

### 21.4 Least-Context Principle
Even when access is allowed, retrieve the minimum necessary content. Prefer masked or summarized representations of sensitive facts. Do not return raw sensitive values unless explicitly required for the task.

### 21.5 Privacy Audit Logging
Write a `privacy_audit` row for every memory read and write. The row MUST NOT include raw memory content — only the resource path, action, accessor, and session context.

### 21.6 Hard Deletion
Hard deletion of individual records is permitted only on explicit operator instruction (e.g., GDPR erasure requests). Before deleting:
1. Log the deletion in `memory_updates` with `update_type = 'retract'` and `reason` describing the erasure request (not the deleted value itself).
2. Write a `privacy_audit` row with `action = 'delete'`.
3. Remove the row.

For markdown documents: remove from `/data/kb/`, update `docs_fts`, and write the deletion to `memory_updates`.

---

### 21.7 Agentic Memory Security Threats

Agents with memory write access face a distinct threat model from traditional applications. The following threats are specific to memory-backed autonomous agents and MUST be addressed in production deployments.

| Threat | Description | Strata Mitigation |
|---|---|---|
| **Memory Poisoning (MINJA)** | Adversary feeds the agent subtly false facts or behavioral overrides across multiple interactions; these are stored in memory and triggered later | Temporal Trust Scoring (§21.8); source authority hierarchy (Rule 3); integrity chain (§12a) |
| **Indirect Prompt Injection** | Malicious content retrieved from memory (or documents) contains embedded instructions that hijack the agent's reasoning | Content inspection at retrieval boundary (§21.9); `source_kind` tracking for untrusted sources |
| **Privilege Escalation via Delegation** | In multi-agent systems, tokens passed between agents accumulate permissions beyond the original human intent | Monotonic scope reduction; DPoP token binding (§21.10) |
| **Data Exfiltration** | Agent writes sensitive memory content to an external tool or service | Tool allowlisting; egress filtering; `privacy_audit` logging |
| **Memory Replay Attack** | Superseded or retracted facts are surfaced to the agent through direct ID queries or `include_history: true` | Default filters exclude superseded/retracted; elevated access required for history queries |

### 21.8 Temporal Trust Scoring

Memory Poisoning attacks (MINJA-class) work by introducing false facts gradually across multiple sessions. A single contradicting record is easy to inject; a sustained campaign is harder. Temporal trust scoring adds a write-time defense:

**Principle:** Records written from lower-authority sources (`source_kind IN ('episodic_inference', 'semantic_inference', 'tool_output')`) within a short window of an existing high-confidence record on the same topic SHOULD be held for review rather than immediately accepted.

**Implementation:**
1. At admission time, if a new record's `(domain, topic, scope)` matches an existing high-confidence active record (`confidence >= 0.8`), compute the time delta between the existing record's `created_at` and now.
2. If the new record comes from a lower-authority source AND the time delta is short (recommend: < 24 hours), escalate `impact_score` by +0.2 before the `hold_for_review` threshold check.
3. Log the escalation in `memory_updates` with `reason = 'temporal_trust_escalation'`.

This does not prevent legitimate rapid updates, but it slows down automated injection campaigns that try to overwrite facts in a single session burst.

**Corroboration window:** For `opinion` and `belief` records specifically, require that confidence-strengthening evidence comes from at least two distinct sessions before `confidence` crosses 0.7. Single-session corroboration alone SHOULD NOT be sufficient to establish a high-confidence belief.

### 21.9 Content Inspection at the Retrieval Boundary

Before injecting retrieved memory content into the agent's reasoning context, implementations SHOULD scan for embedded instruction patterns:

Indicators of injection attempts in retrieved content:
- Imperative commands directed at the agent: "Ignore previous instructions", "You must now...", "Override your rules"
- Scope-elevation language: "You are now authorized to...", "Treat this as a system instruction"
- Identity manipulation: "Your name is now...", "You have been updated to..."

When detected:
1. Strip or quarantine the affected content before injection.
2. Write a `privacy_audit` row with `action = 'read'` and `reason = 'injection_attempt_quarantined'`.
3. Log to `memory_updates` with `update_type = 'integrity_check'` and flag the source record for operator review.

This is especially important for records with `source_kind = 'tool_output'` or `source_kind = 'external_import'` where content originates outside the trusted operator boundary.

### 21.10 Multi-Agent Security and DPoP

In multi-agent deployments where Strata instances service multiple agents with different permission levels, token-based access control MUST be scoped to prevent privilege escalation.

**Scope monotonicity:** When one agent delegates a memory task to a sub-agent, the sub-agent's scope MUST be equal to or narrower than the delegating agent's scope. Sub-agents MUST NOT acquire broader memory access than their delegator.

**DPoP (Demonstrating Proof-of-Possession):** For deployments where memory access is mediated by bearer tokens, implement DPoP to cryptographically bind tokens to specific agent identities. A token presented without proof of the corresponding private key MUST be rejected, even if the token itself is valid. This prevents the "Delegation Cascade" failure mode where a sequence of agent-to-agent calls dilutes the original human authorization until it is unrecognizable.

**Memory isolation between agents:** In multi-agent deployments:
- Each agent's writes SHOULD carry a `scope` value encoding its identity (e.g., `agent:planner`, `agent:researcher`).
- Cross-agent memory reads MUST be explicit — agents SHOULD NOT have default read access to other agents' scoped memories.
- `shared` scope is the only memory tier readable by all agents by default.

---

## 22. Schema Versioning and Migration

### 22.1 Version Tracking

Schema version is tracked in `schema_meta`:

```sql
INSERT INTO schema_meta (key, value) VALUES ('schema_version', '2');
```

Applied migrations are tracked in `applied_migrations` (§9.10). Implementations check this table to determine which migration files have already been applied before running them.

### 22.2 Migration Policy

- Schema changes MUST be additive (new columns, new tables, new enum values).
- Column additions MUST include a DEFAULT value.
- Destructive changes (column removal, type changes) require a major version bump and a documented migration path.
- Each migration MUST write a `memory_updates` row with `update_type = 'schema_migration'` recording the version transition and timestamp.
- Migrations MUST be executed inside a transaction. On failure, roll back.
- Migration files are named `NNN_description.sql` with sequential numeric prefixes.

### 22.3 Migration Procedure

Before running a migration:
1. Back up `strata.db` to `/data/backups/pre-migration-v<N>-<timestamp>.db`.
2. Run migration in a transaction.
3. Update `schema_meta.schema_version`.
4. Write the `schema_migration` audit row.
5. Insert a row into `applied_migrations`.
6. Verify before committing.

### 22.4 FTS Migration Playbook

**Rebuilding a corrupt or out-of-sync FTS index:**
```sql
INSERT INTO memory_records_fts(memory_records_fts) VALUES('rebuild');
INSERT INTO memory_evidence_fts(memory_evidence_fts) VALUES('rebuild');
```

Safe to run at any time. Rescans the content table from scratch.

**If triggers were not created at initial setup:**
The triggers defined in §9.11 MUST be added as a schema update. After adding them, run a rebuild to ensure the FTS index reflects all existing rows.

**Verifying sync:**
```sql
SELECT count(*) FROM memory_records;
SELECT count(*) FROM memory_records_fts;
-- Row counts should match. Discrepancy > threshold: run rebuild.
```

---

## 23. Memory Lifecycle: Export, Repair, and Forget

### 23.1 Export

The system SHOULD support exporting memory to human-readable formats.

Export targets:
- JSON (full table dump or filtered by scope, memory_class, date range).
- Markdown (narrative-formatted summary per topic or entity).

Exports write to `/data/exports/`. Exports SHOULD include provenance metadata so exported records remain auditable.

### 23.2 Repair

When corruption or inconsistency is detected:
1. Run the maintenance loop (§20) to generate the review queue.
2. Identify affected records by layer and ID.
3. For semantic records: restore from the `supersedes_record_id` chain if available.
4. For evidence: write a new `correction` evidence row referencing the corrupted item. Do not mutate the original.
5. For the `memory_updates` chain: identify the first broken link; records after the break point should be reviewed for re-ingestion.

Repair operations MUST themselves produce `memory_updates` rows with appropriate `reason`.

### 23.3 Forget (Selective Erasure)

When a user or operator requests removal:

1. Use `memory_retract` for semantic records.
2. For evidence: write a `correction` evidence row with `metadata_json.operation = 'retract'` referencing the original. The original row is preserved (append-only).
3. For markdown documents: remove from `/data/kb/`, update `docs_fts`, write a `memory_updates` row.
4. For PII hard-deletion: see §21.6.

---

## 24. Implementation Phases

### Phase 1: Backbone
Implement:
- SQLite schema (all core tables, schema_meta, applied_migrations),
- application-enforced enum validation,
- markdown KB directory layout,
- basic `memory_write` (direct record insert, no promotion pipeline yet),
- `memory_updates` audit row on every write.

### Phase 1a: Integrity Chain *(recommended alongside Phase 1)*
Implement:
- `row_hash` / `prev_hash` computation on every `memory_updates` write,
- `impact_score` calculation at admission time,
- lineage field population when called from a session-aware context.

Small enough to build alongside Phase 1. Deferring means retrofitting a chain onto existing rows later.

### Phase 2: Admission Gate
Implement:
- memory class routing (record vs. evidence vs. scratchpad),
- supersede/retract logic with supersedes_record_id chain,
- slot-key semantics for preferences (§9.2.1),
- confidence assignment,
- source authority rules,
- hot-path vs. background-path gate behavior.

### Phase 3: Retrieval
Implement:
- FTS5 search with auto-sync triggers,
- FTS health check and repair,
- exact `normalized_value` lookup,
- evidence packet builder,
- agent tool interface (§13) — all seven tools,
- Query Gate (Rule 9),
- query sanitization (§14.4),
- scope-filtered queries,
- HTTP API (§5b) and MCP server (§5d),
- `POST /memory/sync` endpoint and `memory_sync` MCP tool (§5b.13, §5d.4).

### Phase 4: Evidence Promotion
Implement:
- evidence ingestion with cursors (§18),
- negation gate (Rule 8, §10.2),
- pattern classification (§10.3),
- slot-key preference supersession via promotion,
- memory_links provenance (§9.4),
- scratchpad TTL management.

### Phase 5: Hybrid Retrieval
Implement:
- sqlite-vec extension probe at startup; create `memory_vecs` vec0 virtual table if available (§9.5, §14.2) — **recommended default path**,
- polar-encoded vector storage in `memory_embeddings` for application-layer fallback (§9.5),
- `LOCAL_HYBRID` mode (FTS5 + sqlite-vec) as the default when extension loads,
- circuit breaker for external embedding backend (`HYBRID` mode) as an optional alternative,
- RRF signal fusion (§14.6),
- freshness annotations (§14.8),
- `memory_compact` tool and `POST /memory/compact` endpoint (§13.7, §5b.12).

### Phase 6: Maintenance
Implement:
- nightly maintenance loop (§20) including all steps through §20.11,
- contradiction detection,
- FTS reindexing,
- Markdown sync (§20.11),
- scratchpad purge,
- review queue generation,
- integrity chain verification (if enabled).

### Phase 7: Lifecycle Operations
Implement:
- export (JSON and markdown),
- repair workflow,
- forget / selective erasure.

### Phase 8: Optional Enhancements
Add when warranted:
- LLM reranker (§14.9),
- post-turn extraction (§19),
- Opinion/Belief Network with Reflect pass (§20.10) — recommended alongside Phase 6,
- Temporal trust scoring for agentic security (§21.8),
- Content inspection at retrieval boundary (§21.9),
- DPoP and multi-agent scope enforcement (§21.10) — required for multi-agent deployments,
- Optional Module A (tasks),
- Optional Module B (entities and relationships),
- external Ledger integration (for cryptographic audit trail).

---

## 25. Evaluation Plan

Do not trust memory because it feels intelligent. Test it.

### 25.1 Exact Recall
- "What IP did we assign to the primary switch?"
- "Which endpoint requires authentication?"

### 25.2 Temporal Reasoning
- "What was the hostname before the migration?"
- "What changed between January and March?"

### 25.3 Knowledge Updates
- Old preference vs. current preference: does the current one win?
- Superseded facts: are they excluded from default results?

### 25.4 Multi-Session Continuity
- Use a fact learned weeks ago correctly in a current task.

### 25.5 Abstention
- Correctly report that the system does not know something.
- Avoid hallucinating missing exact facts.

### 25.6 Staleness Awareness
- Does the agent use the freshness caveat to decide whether to re-verify?
- Are preferences older than 30 days annotated appropriately?

### 25.7 Resilience
- With the embedding backend offline: does retrieval fall back to FTS without error?
- After a circuit break: does the system recover automatically when the backend comes back?

### 25.8 Privacy Correctness
- User-scoped records from user A are not returned in queries from user B.
- PII content is not written to shared scope.
- Sensitive records are masked at retrieval time.

### 25.9 Tool Interface Correctness
- `memory_write` applies the admission gate correctly.
- `memory_retract` triggers `hold_for_review` for high-impact items.
- `memory_search` returns only active, non-retracted records by default.
- `memory_write` with a preference supersedes the existing slot correctly.

### 25.10 Promotion Pipeline Correctness
- Negation-pattern evidence is not promoted.
- Preference evidence is promoted and supersedes the previous slot value.
- Promoted records have `memory_links` rows pointing back to their source evidence.

### 25.11 Bi-Temporal Reasoning
- Write a record with `event_time = yesterday, created_at = now`. Query "what was true yesterday?" — the record is returned. Query "what did we know an hour ago?" — the record is NOT returned (it was written just now, `created_at` is now).
- Write a fact, then supersede it with a new value. Query "what was the value before the update?" using `event_time` range — the original is returned with correct timestamp. Query "what is current?" — only the new value is returned.
- Simulate a LongMemEval-style "Knowledge Update" test: write an old constraint, then write a superseding instruction. Verify the agent correctly ignores the old constraint and uses the new one.

### 25.12 Opinion and Belief Network
- Write an `opinion` record with `confidence = 0.6`. Write three supporting evidence items. Run the Reflect pass. Verify `confidence` has increased (target ≥ 0.75).
- Write a `belief` record with `confidence = 0.9`. Write two contradicting evidence items. Run the Reflect pass. Verify `confidence` has decreased and the record is surfaced in the review queue if it drops below threshold.
- Verify that `opinion` and `belief` records are returned in `memory_search` with their confidence scores, and that lower-confidence records are ranked below higher-confidence ones.
- Write an `opinion` that remains corroborated across 30 days of simulated sessions. Verify it is promoted to `belief` by the Reflect pass.
- Verify that `fact`, `preference`, `decision`, and `profile` records are NOT modified by the Reflect pass.

### 25.13 Hierarchical Domain Routing
- Write 100 records across 5 domains. Issue a domain-specific query (e.g., "VLAN routing" → `domain='network'`). Verify that Stage 1 candidate collection is limited to the target domain and does not pull records from other domains.
- Issue a cross-domain query (no domain classification possible). Verify full-corpus search is triggered as expected.
- Write records with and without `domain` set. Verify domain-filtered queries only return records where `domain` matches, and undomained queries return records from all domains.

### 25.14 Agentic Security
- Attempt to write a fact from `source_kind = 'tool_output'` that contradicts an existing `confidence = 0.9` record within the same session. Verify temporal trust escalation fires and the write is held for review.
- Inject a prompt-injection string into a document indexed in `docs_fts`. Retrieve it. Verify the injection attempt is detected and quarantined before it reaches the context.
- In a multi-agent scenario, verify that a sub-agent cannot read records scoped to a peer agent without explicit cross-scope permission.

---

## 26. Recommended Defaults

For a local-first sovereign deployment:

| Feature | Default |
|---|---|
| SQLite backend | enabled |
| FTS5 with triggers | enabled (all three tables) |
| Markdown KB | enabled |
| Integrity chain (Phase 1a) | enabled (recommended from day one) |
| Admission gate | enabled |
| Evidence promotion pipeline | enabled |
| Session ingestion cursors | enabled |
| Scratchpad TTL | 3600s (1 hour) |
| Agent tool interface | enabled |
| Privacy audit logging | enabled |
| Nightly maintenance loop | enabled |
| sqlite-vec extension | enabled if available; `LOCAL_HYBRID` mode when loaded |
| Embedding/vector retrieval | `LOCAL_HYBRID` via sqlite-vec when extension available; `FTS_ONLY` otherwise |
| LLM reranker | disabled until needed |
| Post-turn extraction | disabled until needed |
| Reflect pass (opinion/belief) | enabled (runs after promotion pass) |
| Temporal trust scoring | enabled |
| Content inspection at retrieval | enabled |
| Domain/topic tagging | encouraged (set on all writes) |
| Optional Module A (tasks) | conditional (omit if host app has its own) |
| Optional Module B (entities) | disabled unless relational queries are needed |
| FTS health check min ratio | 0.5 |
| Circuit breaker threshold | 3 failures |
| Circuit breaker half-open | 300 seconds |
| Probe cache TTL | 60 seconds |
| Freshness caveat: events | 1 day |
| Freshness caveat: preferences | 30 days |
| Freshness caveat: facts | 2 days |

---

## 27. Conformance

### 27.1 Minimum Conforming Implementation

A conforming Strata implementation MUST include:

- Core schema: `memory_records`, `memory_evidence`, `memory_links`, `memory_updates`, `schema_meta`, `applied_migrations`
- FTS5 tables with auto-sync triggers: `memory_records_fts`, `memory_evidence_fts`
- Application-enforced enum validation on all write paths
- Admission gate with impact scoring
- Evidence ingestion with idempotency (UNIQUE constraint on `memory_evidence`)
- Negation gate (Rule 8)
- Query Gate (Rule 9)
- Default retrieval filters (exclude retracted/superseded/expired)
- Scope enforcement on all read paths
- Governance rules 1–9
- Agent tool interface (`memory_search`, `memory_get`, `memory_write`, `memory_retract`, `memory_confirm`, `memory_reflect`, `memory_compact`)
- Schema versioning (`schema_meta`) and migration tracking (`applied_migrations`)
- Lifecycle operations (export, repair, forget)

**Note on optional table columns:** The `memory_updates` schema includes `row_hash` and `prev_hash`. These columns MUST exist but MAY remain NULL if the integrity chain (§12a) is not enabled. The `memory_embeddings` table MAY be omitted entirely if vector retrieval is not implemented.

### 27.2 Optional Conforming Extensions

| Extension | Section | When to add |
|---|---|---|
| `scratchpad_entries` | §9.7 | When ephemeral session notes are needed |
| `privacy_audit` | §9.8 | When access auditing is required |
| sqlite-vec `memory_vecs` + `LOCAL_HYBRID` retrieval | §9.5, §14.1 | Recommended for all local deployments; enables zero-ops in-file vector search |
| `memory_embeddings` + external `HYBRID` retrieval | §9.5, §14.1 | When using a remote embedding API instead of sqlite-vec |
| `memory_sync` / Markdown bidirectional sync | §20.11, §5b.13 | When users edit Markdown files directly |
| Evidence promotion pipeline | §10 | Recommended; required for automated evidence → record conversion |
| Integrity chain (row_hash/prev_hash) | §12a | Recommended; required when tamper evidence is needed |
| Optional Module A (`tasks`) | §9.13 | When no existing task system is present |
| Optional Module B (`entities`, `relationships`) | §9.14 | When relational knowledge graph queries are needed |
| LLM reranker | §14.9 | When ranking quality needs improvement |
| Post-turn extraction | §19 | When pattern-based promotion is insufficient |
| External Ledger integration | §12a | When cryptographic audit trail or PITR is required |
| `evidence_ingestion_cursors` | §9.6 | Required when implementing session ingestion |
| Opinion/Belief types + `memory_reflect` + Reflect pass | §9.1, §13.6, §20.10 | When agents need consistent beliefs across sessions |
| Domain/topic fields + hierarchical routing | §9.2.2, §15 | Recommended for all deployments with diverse knowledge domains |
| `event_time` bi-temporality | §9.2.3, §15 | When temporal reasoning over historical states is needed |
| Temporal trust scoring | §21.8 | When operating in adversarial or high-stakes environments |
| Content inspection at retrieval | §21.9 | When indexing externally-sourced content (documents, tool outputs) |
| DPoP and multi-agent scope enforcement | §21.10 | Required for multi-agent deployments |

### 27.3 Recommended Default Profile

For a new local-first sovereign system, build:

- All minimum conforming components
- `scratchpad_entries` (§9.7)
- `privacy_audit` (§9.8)
- Integrity chain (Phase 1a)
- Evidence promotion pipeline (Phase 4)
- `evidence_ingestion_cursors` (§9.6)
- Domain/topic fields on all `memory_records` writes
- `event_time` populated for all writes where the event time is known
- Reflect pass alongside the nightly maintenance loop
- Nightly maintenance loop
- Narrative KB (`docs_fts`)
- FTS retrieval required
- sqlite-vec `LOCAL_HYBRID` recommended when available
- external embedding backends optional

This covers the majority of single-agent and small-team use cases without unnecessary complexity.

---

## 28. What Strata Is Not

Strata is not:
- a magic memory plugin,
- a substitute for application design,
- a promise of perfect recall,
- a justification to store everything forever,
- an excuse to bypass source validation,
- a blockchain or external cryptographic ledger,
- a replacement for explicit `memory_write` calls (promotion pipeline supplements it, does not replace it),
- a model-level attention mechanism (hybrid KV+FW attention is a transformer architecture concern, not a memory storage concern — Strata handles the persistence layer below that).

It is a disciplined substrate for building a trustworthy, governable memory layer — one that can stand alone or integrate cleanly with a broader agent governance framework.

---

## 29. Final Recommendation

Build Strata in this order:

1. **SQLite backbone** with schema versioning (`schema_meta`, `applied_migrations`), enum enforcement, and `memory_updates` audit on every write.
2. **Integrity chain** (`row_hash` / `prev_hash`) — add it now; retrofitting is painful.
3. **Semantic plane** (`memory_records`) with admission gate, supersession chain, and slot-key semantics.
4. **Evidence archive** (`memory_evidence`) with idempotent session ingestion, ingestion cursors, and the negation gate.
5. **FTS retrieval** with auto-sync triggers, health check, query sanitization, and Query Gate.
6. **Agent tool interface** with scope enforcement, freshness annotations, and the full evidence packet format. Include HTTP API (§5b) and MCP server (§5d) from the start.
7. **Promotion pipeline** connecting evidence to records via `memory_links`.
8. **Maintenance loop** for nightly consolidation, contradiction detection, Markdown sync, and review queue generation.
9. **Hybrid retrieval** — probe for sqlite-vec at startup and enable `LOCAL_HYBRID` automatically when the extension loads. This requires no external services and no additional configuration — just load the extension. Add the `memory_vecs` vec0 virtual table and update the retrieval path. For users with a preferred external embedder (Ollama, etc.), support `HYBRID` mode as a configurable option.
10. **Optional enhancements** only when actual need arises: LLM reranker, post-turn extraction, relational model, external ledger.

**Recommended implementation scope:** For most users, steps 1–9 represent the complete useful system. An AI coding agent building Strata SHOULD default to implementing all nine steps. Step 9 requires no external infrastructure when sqlite-vec is available and should not be deferred. Steps beyond 9 are genuinely optional and should be added only when the user specifically needs them.

**Environment to configure at deployment:**
- `STRATA_DATA_PATH` — persistent data root (`/data` in containers, `./data` locally)
- `STRATA_API_KEY` — API key override (if not using auto-generated key from `strata.key`)
- `DEFAULT_USER_ID` — for single-user deployments, set this so agents never need to ask

This sequence builds each layer on a solid foundation, ensures every write is auditable from day one, and keeps retrieval available even when optional components fail. The result is a memory system that is trustworthy, portable, inspectable, and sovereign — capable of serving a single agent or scaling to a multi-user, multi-agent deployment.

---

## Appendix A: Append-Only Event and Evidence Patterns

This appendix provides concrete patterns for the most commonly misimplemented operations: correcting and retracting evidence. The `memory_evidence` table is strictly append-only — original rows MUST NOT be mutated (except `promotion_status` and `promotion_checked_at`).

### A.1 Correcting Evidence

**Scenario:** A conversation turn was ingested with an incorrect summary.

**Step 1 — The original evidence row (do not touch):**
```json
{
  "id": 42,
  "evidence_type": "conversation_turn",
  "title": "Tool execution summary",
  "content": "Ran scan on VLAN 20",
  "session_id": "session-abc",
  "message_id": 7,
  "promotion_status": "promoted"
}
```

**Step 2 — Write a new correction evidence row:**
```json
{
  "evidence_type": "correction",
  "title": "Correction to evidence 42",
  "content": "Correction: scan was on VLAN 10, not VLAN 20",
  "session_id": "session-abc",
  "message_id": null,
  "metadata_json": "{\"corrects_evidence_id\": 42, \"field\": \"content\",
                     \"correct_value\": \"Ran scan on VLAN 10\"}",
  "promotion_status": "ignored"
}
```

**Step 3 — Write a `memory_updates` row:**
```json
{
  "update_type": "update",
  "target_table": "memory_evidence",
  "target_id": 42,
  "prior_value_json": "{\"content\": \"Ran scan on VLAN 20\"}",
  "new_value_json": "{\"via_correction_evidence_id\": 43}",
  "reason": "Correction: wrong VLAN recorded",
  "impact_score": 0.25
}
```

The original row (id=42) is never touched. Consumers must check for `correction` evidence referencing the original ID to find the authoritative value.

---

### A.2 Retracting Evidence

**Scenario:** An evidence item was recorded that should not have been.

**Step 1 — The original evidence row (do not touch).**

**Step 2 — Write a correction evidence row expressing retraction:**
```json
{
  "evidence_type": "correction",
  "title": "Retraction of evidence 77",
  "content": "Evidence 77 was a test run, not a real operation",
  "metadata_json": "{\"retracts_evidence_id\": 77, \"reason\": \"test run incorrectly logged\"}",
  "promotion_status": "ignored"
}
```

**Step 3 — Write a `memory_updates` row with `update_type = 'retract'`.**

The original row remains in the table. Its logical retraction is expressed through the correction row and the `memory_updates` record.

---

### A.3 Key Rules Summary

| Operation | Do | Do NOT |
|---|---|---|
| Correct evidence content | Write new `correction` evidence + `memory_updates` row | Modify the original content |
| Retract evidence | Write new `correction` evidence + `memory_updates` row | Delete original row or change its type |
| Update `promotion_status` | Update in place | Rewrite the entire row |

---

## Appendix B: Evidence Promotion Examples

### B.1 Preference Promotion

**Input evidence:**
```
content: "I prefer concise responses without long preambles"
evidence_type: conversation_turn
speaker_role: user
```

**Negation gate:** No negation patterns found. Proceed.

**Pattern match:** `prefer` matches `_PREFERENCE_PATTERNS`. Record type: `preference`.

**Slot-key derivation:** "response style / length" → `slot_key = 'response_style'`

**Existing record check:** Active preference found with `slot_key = 'response_style'`, `normalized_value = 'verbose'`.

**Action:**
1. Mark existing record `status = 'superseded'`
2. Insert new `memory_records` row: `memory_class='preference'`, `slot_key='response_style'`, `normalized_value='concise'`, `supersedes_record_id = <old_id>`, `confidence=0.80`
3. Write `memory_links` row: `(record_id=<new_id>, evidence_id=<source_id>, link_type='supports')`
4. Update evidence: `promotion_status = 'promoted'`
5. Write `memory_updates` row: `update_type='supersede'`

---

### B.2 Negation Suppression

**Input evidence:**
```
content: "Actually, I decided against using the verbose format"
```

**Negation gate:** `decided against` matches. Evidence marked `promotion_status = 'ignored'`. No record created.

**Follow-up:** If the intent was to retract an existing preference, the agent SHOULD call `memory_retract` explicitly with the specific record ID rather than relying on auto-promotion.

---

### B.3 Decision Promotion

**Input evidence:**
```
content: "We decided to use PostgreSQL for the production database"
```

**Negation gate:** No match. Proceed.

**Pattern match:** `we decided` matches `_DECISION_PATTERNS`. Record type: `decision`.

**Action:** Insert new `memory_records` row with `memory_class='decision'`, `confidence=0.85`. Write `memory_links`. Mark evidence `promoted`.

---

## Appendix C: Retrieval Mode Decision Tree

```
Incoming memory_search call
         │
         ▼
Is query_gate triggered? (too short, credential-like, trivial)
         │
    YES  │  NO
         │   ▼
  Return  │  keyword_only=true in request?
  empty   │       │
  result  │  YES  │  NO
          │       │   ▼
          │ FTS   │  STRATA_RETRIEVAL_MODE=fts_only in env?
          │ ONLY  │       │
          │       │  YES  │  NO
          │       │       │   ▼
          │       │ FTS   │  Is embedding health cached and healthy?
          │       │ ONLY  │       │
          │       │       │  YES  │  NO (cache expired or degraded)
          │       │       │       │   ▼
          │       │       │       │  Run liveness probe
          │       │       │       │       │
          │       │       │       │  OK   │  FAIL
          │       │       │       │       │   ▼
          │       │       │ HYBRID │  Increment failure count
          │       │       │       │  Count >= threshold?
          │       │       │       │  YES → Open circuit → FTS ONLY
          │       │       │       │  NO  → FTS ONLY (this request)
          │
          └──── Log mode decision to memory_updates ──────────────►
```

---

*Strata is released into the public domain under the Unlicense. Build freely, stay sovereign.*
