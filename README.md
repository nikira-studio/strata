# Strata

**The memory layer your AI assistant has been missing.**

---

Here is a scenario most people recognize: you have been working with an AI assistant for weeks. You have told it your preferred style, the name of your project, the constraints you are working within. Three days later it has forgotten all of it. You explain it again. This keeps happening.

Or this one: your assistant tells you something confidently. A week later it tells you something different about the same topic — with equal confidence. You have no idea which version is right, or whether it even knows the facts changed.

Or this: you are building something personal. Everything you tell the AI goes to a server you do not control, stored in a format you cannot inspect, governed by a policy you did not write.

Strata is the answer to all three.

---

## What Strata Is

Strata is a **memory architecture** — a complete specification for giving any AI agent a structured, trustworthy, persistent memory that runs entirely on your own hardware.

It defines exactly how memory should be stored, retrieved, governed, and maintained. Any agent that can make tool calls can use it. Any developer who can run SQLite can implement it.

If you **use** an AI assistant: Strata is what makes the difference between an assistant that starts fresh every session and one that genuinely knows you, your preferences, and your work — session after session, indefinitely.

If you **build** AI systems: Strata is the memory layer you would have to invent anyway. This is the blueprint.

---

## How to Get a Running Service

**You do not need to write any code.**

Point any capable coding AI — Claude Code, Codex, Cursor, or similar — at this repository and say:

> *"Please implement this Strata memory service using Python and FastAPI. Implement Phases 1–9 from the spec (including sqlite-vec for LOCAL_HYBRID retrieval — probe for the extension at startup and enable it automatically if available). Expose the full HTTP API and MCP server. Use `/data` as the persistent data root. Support a `DEFAULT_USER_ID` environment variable for single-user setups. Deploy it as a Docker container with a named volume mounted to `/data`."*

The AI reads `STRATA.md`, which contains the complete database schema, API definitions, tool contracts, and step-by-step implementation guidance. A capable coding agent or developer can use the spec as the implementation blueprint.

**If you prefer a different language or deployment method** (a background process, a VM, a Windows service), substitute that in the prompt. Every conforming Strata implementation exposes the same API — they are interchangeable.

**Docker persistence — important:** When running as a container, your data lives inside the container by default and will be lost if the container is removed. The service uses `/data` as its persistent root. Mount a volume there:

```yaml
# docker-compose.yml
volumes:
  - strata-data:/data
```

or with `docker run`:
```
docker run -v ./strata-data:/data ...
```

Everything — the database, the API key, the knowledge base, backups, and exports — lives under `/data`.

**Once it is running:**

1. Copy the API key printed at first startup (it is saved to `/data/strata.key` and shown once)
2. Set `DEFAULT_USER_ID` to your name or username if this is a single-user setup
3. Tell any agent: *"I have a Strata memory service running at http://your-strata-host. The API key is <STRATA_API_KEY>."*
4. The agent calls `GET http://your-strata-host/spec`, reads the full specification, and knows exactly how to use every feature — no further instruction needed

That is the complete setup. The service runs on your own hardware, your data never leaves your network, and every agent you connect to it immediately gains persistent memory.

---

## What Changes When You Have Real Memory

**Your preferences stick.** Tell your assistant once how you like things done. Strata stores it, and every future session starts already knowing it. You never re-explain your style, your constraints, or your preferences.

**Facts stay current.** When something changes — a decision gets reversed, information gets updated — Strata archives the old version and marks the new one as current. Your assistant always knows which version is true, and it knows the difference between "what was true then" and "what is true now."

**Contradictions get caught.** If your assistant is about to store something that conflicts with what it already knows, Strata flags it before writing. No silent overwrites. No confident-but-wrong answers based on stale data.

**Your data stays yours.** Everything lives in a single SQLite file on your own machine. You can open it with any database tool, copy it, back it up, move it to a new machine, or delete it entirely. No cloud dependency. No vendor lock-in.

**You can always ask why.** Every write is logged with a tamper-evident audit chain. If your assistant says something surprising, you can trace exactly where that belief came from.

---

## One Memory System for All Your Agents

Most people running AI agents today end up with the same problem: every agent has its own isolated memory, or no memory at all. A coding agent that knows your style. A research agent that doesn't. A scheduling agent that has no idea what either of the others knows. You end up re-explaining yourself to each one, and none of them can build on what the others have learned.

Strata is designed to be the single memory layer that all of them share.

**One deployment. Every agent.** You run Strata once — a small always-on service that can live anywhere: a background process on your machine, a container, a VM, whatever fits your setup — and every agent in your setup points to it. Each agent gets its own isolated memory partition by default. Knowledge that should cross agents (project facts, preferences that apply everywhere, decisions that affect the whole system) lives in shared memory and is visible to all of them.

**Connecting an agent takes one config entry.** Strata exposes a Model Context Protocol (MCP) server alongside its HTTP API. For any MCP-compatible agent — Claude Code, Claude Desktop, or similar — you add one entry to your MCP config pointing at the Strata URL, and the Strata memory tools appear automatically. No manual tool registration, no code changes to the agent.

**Agents identify themselves on every call.** When your coding agent calls `memory_search`, it passes its identity. Strata automatically routes the query to that agent's memory, plus your user-level memory, plus anything in shared. The coding agent never sees the research agent's working notes unless that information was explicitly promoted to shared. There is no accidental cross-contamination.

**Multiple people, one instance.** If more than one person uses the system — a household, a small team — each person gets their own memory partition with a hard wall. Person A's agents see Person A's memories. Person B's agents see Person B's memories. Neither can see the other's, ever, not even by accident. The only overlap is `shared` scope, which is for genuinely common information, and nothing with personal details is allowed to reach it.

**Agents can declare a working context.** A coding agent working on Project A can tell Strata "I'm in the context of project-a" and all its searches automatically pre-filter to memories tagged for that project. Switch to Project B, declare a different context. No configuration changes, no separate instances — just a parameter on the call. An agent working across multiple domains (a health agent tracking both diet and exercise, a business agent managing multiple clients) uses the same mechanism.

**You never rebuild from scratch again.** When you add a new agent to your setup, you point it at the same Strata instance. It immediately has access to all the shared knowledge your other agents have built up. Preferences, project context, facts, decisions — all of it is already there. The new agent is not starting cold.

This is what makes Strata different from building memory into each agent individually. Individual memory is a local optimization. Strata is a system-wide capability.

---

## Why Not Just Use What Is Already Out There?

Most AI memory systems fail in one of three ways.

**No memory at all.** Every session starts cold. The agent has to be re-briefed constantly and cannot build on prior work. This is still the default for most deployed agents.

**Naive log injection.** The entire conversation history gets dumped into the context window at the start of each session. Works briefly. Degrades badly as history grows — relevant facts get buried, context gets polluted, and reasoning quality drops.

**Flat vector search.** Everything gets embedded and retrieved by similarity score. Sounds sophisticated. But it cannot tell a current fact from a stale one, cannot reason across multiple related pieces of information, and breaks entirely when the embedding service is unavailable.

Strata solves all three.

---

## How Strata Compares to Existing Memory Systems

### vs. Mem0

Mem0-style systems emphasize lightweight memory extraction and token efficiency. Strata emphasizes provenance, supersession, scope enforcement, and operator-governed lifecycle controls. It is not intended to be a drop-in replacement for every Mem0 use case; it is designed for deployments where memory must be inspectable, auditable, and locally controlled.

### vs. Letta / MemGPT

Letta and MemGPT-style systems treat memory as part of a persistent agent runtime. Strata instead treats memory as infrastructure: a standalone service that multiple agents and frameworks can share. This makes it better suited for mixed-agent environments where no single framework owns the whole system.

### vs. Zep / temporal knowledge graphs

Graph-first systems can provide rich temporal and relational memory. Strata chooses a simpler SQLite-first design with explicit timestamps, supersession chains, and evidence links. This keeps the operational footprint small while still supporting practical temporal reasoning.

| What you need | Strata's approach |
|---|---|
| Retrieval when the vector backend is down | FTS5 full-text search always available; hybrid mode degrades gracefully via circuit breaker |
| Knowing which version of a fact is current | Supersession chain — old records archived, not overwritten; freshness annotations signal when to re-verify |
| Reasoning across related pieces of memory | Domain/topic hierarchy routes queries to the right partition before search |
| Getting durable memory out of a conversation | Evidence promotion pipeline: raw turns captured, classified, and promoted to structured records automatically |
| Trusting that confident answers are actually right | Confidence scores, source authority ranking, and a nightly Reflect pass that strengthens or weakens beliefs as evidence accumulates |

---

## How Strata Improves the Agent Systems You Are Already Using

Strata slots into any agent that supports tool calls. It replaces or augments whatever memory layer that agent currently has.

### Claude Code, Codex, and coding agents

Coding agents benefit most from Strata's exact recall and preference persistence. Strata remembers project decisions, architectural constraints, code style preferences, and tool configurations across sessions — so the agent does not have to be re-briefed on every run. The scratchpad provides session-scoped working notes that expire automatically. The evidence promotion pipeline captures decisions made during a session and promotes them to durable records without requiring explicit saves.

### OpenClaw, Hermes, and similar agentic frameworks

These frameworks provide orchestration and tool routing but typically delegate memory to a simple vector store or session context. Strata replaces that backend with a small set of standard memory tools the agent uses naturally:

- `memory_search` — retrieve relevant facts, preferences, and history
- `memory_get` — fetch a specific record or evidence packet on a topic
- `memory_write` — persist a fact, preference, decision, or event
- `memory_retract` — mark a record as no longer true
- `memory_confirm` — approve or reject a pending write (operator-in-the-loop)
- `memory_reflect` — adjust confidence in a belief based on new evidence

An agent that currently calls `search_vector_db(query)` replaces it with `memory_search(query)` and immediately gains: fallback resilience, scope isolation, freshness annotations, source authority ranking, and a full audit trail.

### Multi-agent systems

When multiple specialized agents collaborate — planner, researcher, coder, reviewer — Strata's scope model gives each agent its own memory partition by default. Shared memory is explicitly designated. Sub-agents cannot acquire broader memory access than their delegating agent. The integrity chain provides a tamper-evident audit log across the whole system.

### Any application where users return across sessions

If your application involves someone who comes back — a recurring user, a long-running project, a personal assistant — Strata gives it memory. Profile records persist identity and preferences. Fact records persist operational knowledge. Opinion and belief records persist the agent's evolving understanding. The privacy model ensures one user's memory is never visible to another.

---

## Architecture at a Glance

```
Constitutional  ──────────────────────────────────  Markdown + protected records
Memory                                               (identity, rules, directives)

Semantic        ──────────────────────────────────  memory_records
Memory          facts, preferences, decisions,       (active, superseded, retracted)
                profiles, opinions, beliefs          domain/topic hierarchy
                                                     bi-temporal: event_time + created_at

Episodic        ──────────────────────────────────  memory_evidence
Memory          conversation turns, tool outputs,    (pending → promoted/ignored)
                events, corrections                  append-only, idempotent ingestion

Narrative       ──────────────────────────────────  Markdown files + docs_fts
Memory          runbooks, architecture docs,
                meeting notes, summaries
```

Every write passes through an **admission gate** — impact scoring, supersession detection, conflict checking. Raw evidence passes through a **promotion pipeline** — negation filtering, pattern classification, structured record creation. Every retrieval runs through a **query gate** and returns a curated result set with freshness annotations, not a raw dump.

---

## Getting Started

The full specification is in **[STRATA.md](./STRATA.md)**. It is self-contained: SQL schema, trigger definitions, algorithm pseudocode, governance rules, complete HTTP API reference with request/response schemas, and a phased implementation roadmap.

**For builders:** Give STRATA.md to any capable AI or developer. Everything needed to build a fully conforming implementation from scratch is in that document — no external dependencies, no ambiguity about what to build.

**For users of an already-built Strata service:** Give any agent the base URL and API key. The agent calls `GET /spec` to retrieve the full specification, reads it, and knows exactly how to use every feature. Nothing else is required.

### Implementation Phases

1. **Phase 1 — Backbone**: SQLite schema, schema versioning, `memory_updates` audit log
2. **Phase 1a — Integrity Chain**: hash-chained audit rows (add with Phase 1)
3. **Phase 2 — Admission Gate**: supersession chain, slot-key preferences, impact scoring
4. **Phase 3 — Retrieval**: FTS5 with auto-sync triggers, query gate, all agent tools, HTTP API, MCP server
5. **Phase 4 — Evidence Promotion**: session ingestion with cursors, negation gate, provenance links
6. **Phase 5 — Hybrid Retrieval**: sqlite-vec `LOCAL_HYBRID` (probe at startup, zero external deps); external `HYBRID` for users with a preferred embedder (Ollama, etc.)
7. **Phase 6 — Maintenance**: nightly loop, contradiction detection, Markdown sync, Reflect pass, review queue
8. **Phase 7 — Lifecycle**: export, repair, selective erasure
9. **Phase 8 — Optional**: LLM reranker, post-turn extraction, relational model, external ledger

**Most users should target Phases 1–6.** Phases 1–4 are the governed memory core. Phase 5 adds semantic search with zero external infrastructure. Phase 6 adds the maintenance loop that keeps memory healthy over time. See `STRATA.md §24` for the full phased implementation guide.

---

## License

Released into the public domain under the [Unlicense](./LICENSE). Build freely, stay sovereign.
