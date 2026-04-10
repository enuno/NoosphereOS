# NoosphereOS — Architecture Diagrams

This document contains the full set of architecture diagrams for the NoosphereOS multi-agent memory operating system. For narrative descriptions of each component, see [README.md](./README.md).

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [Layer Definitions and Stack](#2-layer-definitions-and-stack)
3. [Control Plane and Agent Runtime](#3-control-plane-and-agent-runtime)
4. [L0: Hot Memory — Engram + GitLawb + MEMORY.md](#4-l0-hot-memory--engram--gitlawb--memorymd)
5. [L0 Permission Model — UCAN + RBAC](#5-l0-permission-model--ucan--rbac)
6. [L1: Local Memory OS — MemPalace](#6-l1-local-memory-os--mempalace)
7. [L2: Distributed Memory Substrate — Hindsight](#7-l2-distributed-memory-substrate--hindsight)
8. [L3: Immutable Archive — WeaveDB + Arweave/ArDrive](#8-l3-immutable-archive--weavedb--arweaveardrive)
9. [L4: Verifiable Shared Knowledge — OriginTrail DKG](#9-l4-verifiable-shared-knowledge--origintrail-dkg)
10. [Data Flow: Sacred Fact Promotion (L0 → L1 → L2 → L3)](#10-data-flow-sacred-fact-promotion-l0--l1--l2--l3)
11. [MCP Tool Surface and Authorization Flow](#11-mcp-tool-surface-and-authorization-flow)
12. [Multi-Agent Sharing and RBAC Policy Matrix](#12-multi-agent-sharing-and-rbac-policy-matrix)

---

## 1. High-Level Architecture

```text
                      ┌──────────────────────────────────────────┐
                      │         HUMAN OPERATORS / UIs            │
                      │  - Inspect MEMORY.md diffs               │
                      │  - Approve/deny memory promotions        │
                      │  - View Arweave / DKG snapshots          │
                      └──────────────────────────────────────────┘
                                           ▲
                                           │  review / governance
                                           │
┌──────────────────────────────────────────────────────────────────────────┐
│                        NOOSPHERE CONTROL PLANE                          │
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  Identity & Keys │  │  UCAN Capability │  │  RBAC + Policy       │  │
│  │  (Agent DIDs)    │  │  Broker          │  │  Engine              │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘  │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Memory Router                                                   │   │
│  │  Decides target layer(s) for each read/write operation           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Event Bus — NATS JetStream                                      │   │
│  │  noosphere.memory.promoted.{agent_did}                           │   │
│  │  noosphere.memory.proposal.created.{agent_did}                   │   │
│  │  noosphere.l0.engram.synced.{agent_did}                          │   │
│  │  noosphere.l1.mempalace.updated.{agent_did}                      │   │
│  │  noosphere.l2.hindsight.retained.{agent_did}                     │   │
│  │  noosphere.l3.archive.snapshot.created.{agent_did}               │   │
│  │  noosphere.l4.dkg.export_requested.{agent_did}                   │   │
│  │  noosphere.memory.reverted.{agent_did}                           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
    │                      │                      │                    │
    │ MCP (OAuth 2.1)      │ MCP (OAuth 2.1)      │ async events       │ async events
    ▼                      ▼                      ▼                    ▼

┌───────────────────┐   ┌────────────────────────────────────────────────┐
│  AGENT RUNTIME(S) │   │  NOOSPHERE MEMORY MCP SERVER                  │
│  OpenClaw         │   │  Exposes memory_get / propose / review / merge │
│  moltbot          │──▶│  Enforces RBAC + UCAN per call                 │
│  custom agents    │◀──│  Hides raw GitLawb / Engram / storage details  │
└───────────────────┘   └────────────────────────────────────────────────┘
         │                              │
         │                              │
         ▼                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│                         MEMORY LAYERS                                  │
│                                                                        │
│  L0  Engram + MEMORY.md + GitLawb repos (hot memory, per agent)       │
│  L1  MemPalace             (local episodic/semantic OS, per agent)     │
│  L2  Hindsight             (distributed substrate, per tenant/org)     │
│  L3  WeaveDB + Arweave     (immutable archive, per org)                │
│  L4  OriginTrail DKG       (verifiable shared knowledge, selective)    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Layer Definitions and Stack

```text
  GRANULARITY ◀────────────────────────────────────────────▶ BREADTH

  L0 │ Engram + MEMORY.md + GitLawb │ per-agent │ hot, human-readable, signed history
     │
  L1 │ MemPalace                    │ per-agent/pod │ local episodic + semantic OS
     │
  L2 │ Hindsight                    │ per-tenant/cluster │ sovereign distributed
     │                               │                    │ retain/recall/reflect
  L3 │ WeaveDB + Arweave            │ per-org/global │ immutable archive, artifact store
     │
  L4 │ OriginTrail DKG              │ shared/global │ verifiable, cross-org knowledge graph


  SCOPE OF SHARING:
  ─────────────────────────────────────────────────────────────────
  L0: Agent-local (Engram store + GitLawb MEMORY.md; shared only via explicit UCAN delegation)
  L1: Agent-local (no remote sync by default)
  L2: Tenant/org-wide (multi-agent access with policy; self-hosted PostgreSQL)
  L3: Immutable, org-level records (append-only, no deletion)
  L4: Selective public publication (explicit DKG export)
  ─────────────────────────────────────────────────────────────────
```

---

## 3. Control Plane and Agent Runtime

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                        NOOSPHERE CONTROL PLANE                          │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  IDENTITY SERVICE                                               │   │
│   │  - Manages agent DIDs (did:gitlawb:agent:...)                   │   │
│   │  - Key generation, rotation, export                             │   │
│   │  - DID resolution for peers, curators, operators                │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  UCAN CAPABILITY BROKER                                         │   │
│   │  - Mints delegations: resource / ability / expiry / audience    │   │
│   │  - Validates UCAN chains on every tool invocation               │   │
│   │  - Handles revocation checks                                    │   │
│   │  - Resource URIs:                                               │   │
│   │      repo://did:gitlawb:repo:AGENT-memory/path/MEMORY.md        │   │
│   │      repo://did:gitlawb:repo:AGENT-memory/ref/refs/heads/...    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  RBAC + POLICY ENGINE                                           │   │
│   │  Roles: owner | peer_reader | contributor | curator | operator  │   │
│   │  Policies: per-agent, per-group, per-memory-class               │   │
│   │  Sacred/constitutional facts → stricter approval required       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  MEMORY ROUTER                                                  │   │
│   │  On read:  L0 (anchor/MEMORY.md) → L1 (local) → L2 (Hindsight) │   │
│   │  On write: stages to Engram/L1/L2, promotes to L0 via proposal  │   │
│   │  On milestone: triggers L3 snapshot + optional L4 export        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  EVENT BUS — NATS JetStream (ratified 2026-04-10)               │   │
│   │                                                                 │   │
│   │  Deployment topology:                                           │   │
│   │    - Production: 3-node NATS cluster, JetStream replication=3   │   │
│   │    - Dev/single-node: embedded NATS server (single binary)      │   │
│   │    - Edge (ServerDomes/DePIN): NATS Leaf Nodes connecting to    │   │
│   │      central cluster; agents publish locally, cluster replicates│   │
│   │                                                                 │   │
│   │  JetStream streams:                                             │   │
│   │    NOOSPHERE_MEMORY     — subjects: noosphere.memory.>          │   │
│   │    NOOSPHERE_L0         — subjects: noosphere.l0.>              │   │
│   │    NOOSPHERE_L1         — subjects: noosphere.l1.>              │   │
│   │    NOOSPHERE_L2         — subjects: noosphere.l2.>              │   │
│   │    NOOSPHERE_L3         — subjects: noosphere.l3.>              │   │
│   │    NOOSPHERE_L4         — subjects: noosphere.l4.>              │   │
│   │                                                                 │   │
│   │  Subject namespace (per-agent scoping):                         │   │
│   │    noosphere.memory.proposal.created.{agent_did}                │   │
│   │    noosphere.memory.promoted.{agent_did}                        │   │
│   │    noosphere.memory.reverted.{agent_did}                        │   │
│   │    noosphere.l0.engram.synced.{agent_did}                       │   │
│   │    noosphere.l1.mempalace.updated.{agent_did}                   │   │
│   │    noosphere.l2.hindsight.retained.{agent_did}                  │   │
│   │    noosphere.l3.archive.snapshot.created.{agent_did}            │   │
│   │    noosphere.l4.dkg.export_requested.{agent_did}                │   │
│   │                                                                 │   │
│   │  Consumer model:                                                │   │
│   │    - All workers use durable pull consumers (work-queue)        │   │
│   │    - Delivery: at-least-once; idempotency keys per message      │   │
│   │    - Dead-letter: NOOSPHERE_DLQ stream for failed deliveries    │   │
│   │                                                                 │   │
│   │  Message envelope — CloudEvents (application/cloudevents+json): │   │
│   │    { specversion, type, source, id, time,                       │   │
│   │      datacontenttype, data: { ...payload } }                    │   │
│   │                                                                 │   │
│   │  Built-in KV buckets (JetStream KV):                           │   │
│   │    KV_AGENT_REGISTRY   — agent DID → metadata, status          │   │
│   │    KV_UCAN_REVOCATIONS — revoked UCAN CIDs                      │   │
│   │    KV_SESSION_STATE    — transient per-agent session state      │   │
│   │                                                                 │   │
│   │  Security:                                                      │   │
│   │    - TLS on all connections                                     │   │
│   │    - NKEYS + JWT-based auth (maps to agent DID namespaces)      │   │
│   │    - Per-subject publish/subscribe ACLs enforced by server      │   │
│   │    - Agents may only publish to noosphere.*.*.{own_agent_did}   │   │
│   │    - Cross-agent subscriptions require operator-issued JWT      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4. L0: Hot Memory — Engram + GitLawb + MEMORY.md

```text
┌──────────────────────────────────────────────────────────────────────────┐
│  L0 — PER-AGENT HOT MEMORY                                              │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  ENGRAM — Internal Per-Agent Memory Store                         │  │
│  │                                                                   │  │
│  │  Storage: SQLite + FTS5 (full-text search)                        │  │
│  │  Interfaces: MCP server | HTTP API | CLI | TUI                    │  │
│  │                                                                   │  │
│  │  MCP Tools:                                                       │  │
│  │    mem_save          ← agent writes a new memory                  │  │
│  │    mem_search        ← semantic + keyword recall                  │  │
│  │    mem_context       ← returns ranked memories for current task   │  │
│  │    mem_session_summary ← end-of-session compression               │  │
│  │                                                                   │  │
│  │  Capabilities:                                                    │  │
│  │    - Contradiction detection + resolution                         │  │
│  │    - Context compaction recovery                                  │  │
│  │    - Cross-session persistence (survives agent restarts)          │  │
│  │    - Structured observation records with timestamps               │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                          │                                               │
│                          │  Noosphere promotion pipeline                │
│                          │  (curated subset only)                       │
│                          ▼                                               │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  GitLawb REPO — did:gitlawb:repo:agent-α-memory                  │  │
│  │                                                                   │  │
│  │  Branch: main (PROTECTED — no direct push except owner/operator)  │  │
│  │  ├── MEMORY.md          ← canonical exported hot memory           │  │
│  │  │                         (sacred facts, anchors, core prefs)    │  │
│  │  └── /staging/          ← proposals, patches, drafts              │  │
│  │       └── <actor-did>/  ← per-proposer staging area               │  │
│  │                                                                   │  │
│  │  MEMORY.md STRUCTURE (example):                                   │  │
│  │                                                                   │  │
│  │    # Agent α — Hot Memory                                         │  │
│  │    last_updated: 2026-04-10T08:00:00Z                             │  │
│  │    commit: C_main_abc123                                          │  │
│  │    engram_snapshot: engram://α/snapshot/xyz                       │  │
│  │                                                                   │  │
│  │    ## Sacred Facts                                                │  │
│  │    - [fact-001] <content> (promoted by β, curated by γ)          │  │
│  │    - [fact-002] <content> (promoted by agent, auto-curated)      │  │
│  │                                                                   │  │
│  │    ## Anchors                                                     │  │
│  │    - Current task: ...                                            │  │
│  │    - Core identity: ...                                           │  │
│  │    - Risk constraints: ...                                        │  │
│  │                                                                   │  │
│  │  GitLawb Repo Events (bridged to NATS by Noosphere Event Bus):    │  │
│  │    - pr_opened, pr_reviewed, pr_merged, commit_created            │  │
│  │    - Each event includes: author DID, UCAN CID, commit SHA        │  │
│  │    - Published to: noosphere.memory.proposal.created.{agent_did}  │  │
│  │                    noosphere.memory.promoted.{agent_did}          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  KEY PRINCIPLE: Engram is the primary store. MEMORY.md is a curated,   │
│  human-legible export — never written directly by agents, only via      │
│  the Noosphere proposal/review/merge pipeline.                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 5. L0 Permission Model — UCAN + RBAC

```text
  RESOURCE: repo://did:gitlawb:repo:agent-α-memory/path/MEMORY.md

  ┌─────────────────────────────────────────────────────────────────────┐
  │  ROLE               │ ABILITIES GRANTED                             │
  ├─────────────────────┼───────────────────────────────────────────────┤
  │  owner agent        │ repo/file/read                                │
  │                     │ repo/file/propose                             │
  │                     │ repo/ref/push (main + staging)                │
  │                     │ repo/pr/review                                │
  │                     │ repo/pr/merge                                 │
  │                     │ repo/event/subscribe                          │
  │                     │ engram/write (own store)                      │
  ├─────────────────────┼───────────────────────────────────────────────┤
  │  peer_reader        │ repo/file/read (MEMORY.md only)               │
  │                     │ repo/event/subscribe                          │
  ├─────────────────────┼───────────────────────────────────────────────┤
  │  contributor agent  │ repo/file/read                                │
  │                     │ repo/file/propose (staging/* only)            │
  │                     │ engram/write (own store, not target agent's)  │
  ├─────────────────────┼───────────────────────────────────────────────┤
  │  curator agent      │ repo/file/read                                │
  │                     │ repo/file/propose                             │
  │                     │ repo/pr/review                                │
  │                     │ repo/pr/merge (policy-gated)                  │
  ├─────────────────────┼───────────────────────────────────────────────┤
  │  human operator     │ ALL abilities + break-glass push to main      │
  └─────────────────────┴───────────────────────────────────────────────┘

  UCAN ATTENUATION RULES:
  ─────────────────────────────────────────────────────────────────────
  1. Audience restriction: each token issued to exactly one agent DID
  2. Path restriction:    /path/MEMORY.md only (not whole repo)
  3. Branch restriction:  staging/<actor-did>/* (not main)
  4. Time restriction:    short expiry (10–60 min for ops tasks)
  5. Action restriction:  propose ≠ merge; read ≠ write
  6. NO wildcard (*) authority granted to non-owner agents
  ─────────────────────────────────────────────────────────────────────
```

---

## 6. L1: Local Memory OS — MemPalace

```text
┌──────────────────────────────────────────────────────────────────────────┐
│  L1 — PER-AGENT MEMPALACE INSTANCE                                      │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  PALACE STRUCTURE (rooms/wings)                               │     │
│   │                                                               │     │
│   │  ┌─────────────────┐   ┌─────────────────┐                   │     │
│   │  │  Sacred Facts   │   │  Projects /     │                   │     │
│   │  │  (from MEMORY.md│   │  Tasks          │                   │     │
│   │  │  promotions)    │   │                 │                   │     │
│   │  └─────────────────┘   └─────────────────┘                   │     │
│   │  ┌─────────────────┐   ┌─────────────────┐                   │     │
│   │  │  Episodic       │   │  Semantic /     │                   │     │
│   │  │  Conversations  │   │  Entity Graph   │                   │     │
│   │  └─────────────────┘   └─────────────────┘                   │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                                                                          │
│   Storage backends (local, no external deps):                           │
│   ├── SentenceTransformers (local embeddings)                           │
│   ├── ChromaDB (local vector store)                                     │
│   └── SQLite (temporal triple store / entity graph)                     │
│                                                                          │
│   Triggers for reload/update:                                           │
│   ├── Startup: loads MEMORY.md from GitLawb main branch                 │
│   ├── On memory_promoted event: NATS pull consumer on                   │
│   │     noosphere.memory.promoted.{agent_did}                          │
│   │     Re-parses sacred facts, updates palace rooms                   │
│   ├── On Engram sync: NATS pull consumer on                             │
│   │     noosphere.l0.engram.synced.{agent_did}                         │
│   │     Incorporates latest observation records                         │
│   └── On task completion: writes new episodic entries locally           │
│       Publishes: noosphere.l1.mempalace.updated.{agent_did}             │
│                                                                          │
│   Recall interface:                                                      │
│   ├── MCP tools: memory_search, memory_reflect                          │
│   └── Direct SDK for co-located agent runtimes                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 7. L2: Distributed Memory Substrate — Hindsight

```text
┌──────────────────────────────────────────────────────────────────────────┐
│  L2 — HINDSIGHT (sovereign self-hosted, per tenant / org)               │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  INGEST (retain)                                              │     │
│   │  - NATS durable pull consumer on:                             │     │
│   │      noosphere.memory.promoted.{agent_did}                    │     │
│   │      noosphere.l0.engram.synced.{agent_did}                   │     │
│   │  - Accepts structured memories from Engram sync workers       │     │
│   │  - Extracts: facts, entities, relationships, temporal links   │     │
│   │  - Builds structured observations:                            │     │
│   │      {subject, type, content, entities, source,               │     │
│   │       provenance, confidence, timestamp}                      │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                          │                                               │
│                          ▼                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  MEMORY STORE (PostgreSQL-backed)                             │     │
│   │  ├── Semantic index (vector embeddings)                       │     │
│   │  ├── BM25 / keyword index (full-text)                         │     │
│   │  ├── Entity + relationship graph (knowledge graph)            │     │
│   │  └── Temporal index (time-aware recall, decay, ordering)      │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                          │                                               │
│                          ▼                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  RECALL + REFLECT API                                         │     │
│   │  Retrieval strategies (fused, reranked):                      │     │
│   │    - Semantic search (embedding similarity)                   │     │
│   │    - Keyword/BM25 (exact and near-exact match)                │     │
│   │    - Graph traversal (entity → relation → entity)             │     │
│   │    - Temporal reasoning (before/after/during, recency decay)  │     │
│   │  Reflect: synthesizes prior memories into updated summaries   │     │
│   │  Scoped by: agent_id / user_id / tenant                       │     │
│   │  Returns: top-k memories + provenance + source commit CID     │     │
│   │  Interfaces: HTTP API | Python client | MCP wrapper           │     │
│   │  On retain complete: publishes to NATS                        │     │
│   │    noosphere.l2.hindsight.retained.{agent_did}                │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                                                                          │
│   Memory types handled:                                                  │
│   ├── sacred_fact        (from MEMORY.md + Engram promotions)           │
│   ├── episodic           (task logs, conversation summaries)             │
│   ├── preference         (derived from interactions)                    │
│   └── external_doc       (from ingested files / tools / searches)       │
│                                                                          │
│   Deployment:                                                            │
│   ├── Self-hosted: Docker Compose or Kubernetes                         │
│   ├── Embedded PostgreSQL or external cluster                           │
│   └── No cloud dependency — fully sovereign                             │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 8. L3: Immutable Archive — WeaveDB + Arweave/ArDrive

```text
┌──────────────────────────────────────────────────────────────────────────┐
│  L3 — PERMAWEB ARCHIVE LAYER                                            │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ARWEAVE / ARDRIVE (permanent storage)                          │    │
│  │  Stored assets:                                                 │    │
│  │    - MEMORY.md snapshots at milestones                          │    │
│  │    - SOUL.md / NOOSPHERE config versions                        │    │
│  │    - Constitutions and policy documents                         │    │
│  │    - PDFs, datasets, contracts, design docs                     │    │
│  │  Tags on each asset:                                            │    │
│  │    { agent_did, commit_sha, engram_snapshot_id,                 │    │
│  │      hindsight_obs_ids, type, promoted_at }                     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                          │                                               │
│                          │ arweave_tx_id                                 │
│                          ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  WEAVEDB (structured index on permaweb)                         │    │
│  │  Schema per record:                                             │    │
│  │    {                                                            │    │
│  │      arweave_tx_id,                                             │    │
│  │      agent_did,                                                 │    │
│  │      commit_sha,                                                │    │
│  │      engram_snapshot_id,                                        │    │
│  │      hindsight_memory_ids: [...],                               │    │
│  │      fact_ids: [...],                                           │    │
│  │      type: "sacred_fact_snapshot" | "constitution" | ...,       │    │
│  │      promoted_at,                                               │    │
│  │      proposer_did,                                              │    │
│  │      curator_did                                                │    │
│  │    }                                                            │    │
│  │  Queryable by agents and operators for reference lookups        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Trigger: archive worker maintains durable NATS pull consumer on        │
│    noosphere.memory.promoted.> (all agents, sacred_fact policy=true)    │
│  On completion: publishes noosphere.l3.archive.snapshot.created.{did}   │
│  Policy: sacred_fact=true OR type=constitution → always snapshot        │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 9. L4: Verifiable Shared Knowledge — OriginTrail DKG

```text
┌──────────────────────────────────────────────────────────────────────────┐
│  L4 — ORIGINTRAIL DKG LAYER (opt-in, selective export)                  │
│                                                                          │
│  Input from L3: arweave_tx_id, WeaveDB record CID                       │
│  Input from L2: Hindsight memory ID, observation provenance             │
│  Input from L0: Engram snapshot ID, MEMORY.md commit SHA                │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  KNOWLEDGE ASSET (per exported fact or claim)                   │    │
│  │  {                                                              │    │
│  │    subject: agent_did / entity,                                 │    │
│  │    claim: <semantic triple or JSON-LD statement>,               │    │
│  │    provenance: {proposer, curator, promoted_at, commit},        │    │
│  │    sources: [arweave_tx_id, hindsight_memory_id,                │    │
│  │              engram_snapshot_id],                               │    │
│  │    access: public | org-permissioned | agent-group              │    │
│  │  }                                                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Use cases:                                                              │
│   - Cross-org agent knowledge sharing with verifiable provenance        │
│   - Audits and compliance proofs                                         │
│   - Noosphere "commons": shared world-model for a multi-org swarm       │
│                                                                          │
│  Trigger: DKG exporter maintains durable NATS pull consumer on          │
│    noosphere.l4.dkg.export_requested.> (explicit operator/agent action) │
│  Policy:  explicit opt-in only — NO automatic full-memory DKG export    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Data Flow: Sacred Fact Promotion (L0 → L1 → L2 → L3)

```text
  OPERATION: Agent β proposes new sacred fact for Agent α
  ─────────────────────────────────────────────────────────────────────────

  [Agent β Runtime]
       │  1. Identifies candidate sacred fact (e.g., long-lived constraint)
       │     Writes candidate into β's own Engram store as an observation
       │
       ▼
  [Noosphere Memory MCP Server]
       │  2. Receives memory_propose({ agent: α, fact, justification })
       │     - OAuth 2.1 authenticates β
       │     - RBAC: β has "contributor" role for α?
       │     - Capability broker: mints UCAN
       │         resource: repo://did:gitlawb:α-memory/path/MEMORY.md
       │         ability:  repo/file/propose
       │         branch:   refs/heads/memory-staging/<β-did>/...
       │         expiry:   10 minutes
       │
       ▼
  [GitLawb Repo: did:gitlawb:α-memory]
       │  3. Creates staging branch
       │     Applies patch to MEMORY.md (appends fact + metadata)
       │     Opens PR: head=staging/β-did/..., base=main
       │     PR metadata: {proposer: β-did, ucan_cid, fact_id}
       │
       ▼
  [NATS JetStream — stream: NOOSPHERE_MEMORY]
       │  4. Publishes: noosphere.memory.proposal.created.{α-did}
       │     CloudEvents envelope: {type, source, id, time,
       │       data: {agent: α, proposer: β, repo_did, pr_id}}
       │
       ▼
  [Curator Agent γ / Human Operator]
       │  5. Pull consumer receives message from NOOSPHERE_MEMORY stream
       │     Calls memory_diff(pr_id) → inspects change in context
       │       of existing sacred facts and Engram state
       │
       │  6. Approves:
       │     Calls memory_review({ pr_id, decision: "approve" })
       │       - RBAC: γ has "curator" role
       │       - UCAN: repo/pr/review on this PR
       │     Calls memory_merge({ pr_id })
       │       - UCAN: repo/pr/merge (policy-gated for sacred facts)
       │
       ▼
  [GitLawb Repo: did:gitlawb:α-memory]
       │  7. Merges PR to main (protected branch)
       │     Creates commit C_main { updated MEMORY.md }
       │     GitLawb webhook bridge triggers NATS publish
       │
       ▼
  [NATS JetStream — stream: NOOSPHERE_MEMORY]
       │  8. Publishes: noosphere.memory.promoted.{α-did}
       │     CloudEvents envelope: {type, source, id, time,
       │       data: {agent: α, commit: C_main, fact_id,
       │              proposer: β, curator: γ}}
       │
       ├──────────────────────────────────────────────────┐
       │                                                  │
       ▼  (parallel, async pull consumers)                ▼  (parallel, async)

  [Engram Sync Worker — Agent α]               [MemPalace — Agent α]
       │  9a. Pull consumer on                      │  9b. Pull consumer on
       │      noosphere.memory.promoted.{α-did}     │      noosphere.memory.promoted.{α-did}
       │      Fetches MEMORY.md at C_main           │      Fetches MEMORY.md at C_main
       │      Writes structured sacred_fact         │      Parses sacred facts section
       │      record into α's Engram store:         │      Updates palace:
       │        {content, entities, provenance:     │        - "Sacred Facts" room
       │         β-did, γ-did, commit, ts}          │        - New triples inserted
       │      Resolves contradictions if any        │        - High-priority recall flag
       │      Publishes:                            │      Publishes:
       │        noosphere.l0.engram.synced.{α-did}  │        noosphere.l1.mempalace.updated.{α-did}
       │
       ▼  (parallel, async pull consumer)

  [Hindsight Ingest Worker — L2]
       │  9c. Pull consumers on:
       │        noosphere.memory.promoted.{α-did}
       │        noosphere.l0.engram.synced.{α-did}
       │      Reads MEMORY.md diff at C_main + Engram record
       │      Builds structured sacred_fact memory:
       │        {subject: α,
       │         type: sacred_fact,
       │         content: X,
       │         entities: [...],
       │         relationships: [...],
       │         source: gitlawb://α@C_main,
       │         engram_id: <engram record ID>,
       │         provenance: {proposer: β, curator: γ, ts}}
       │      Calls Hindsight retain API (HTTP / Python client)
       │      Hindsight indexes memory across:
       │        - Semantic (vector) index
       │        - BM25 keyword index
       │        - Entity/relationship graph
       │        - Temporal index
       │      Publishes:
       │        noosphere.l2.hindsight.retained.{α-did}
       │
       ▼  (if sacred_fact policy = archive)

  [Archive Worker]
       │  10. Pull consumer on noosphere.memory.promoted.{α-did}
       │      Fetches MEMORY.md at C_main
       │      Writes to Arweave/ArDrive:
       │        tags: {agent: α, commit: C_main, type: sacred_fact_snapshot,
       │               engram_snapshot_id, hindsight_memory_ids}
       │      Writes index row to WeaveDB:
       │        {arweave_tx_id, agent: α, commit: C_main,
       │         fact_ids, engram_snapshot_id, hindsight_memory_ids, ...}
       │      Publishes:
       │        noosphere.l3.archive.snapshot.created.{α-did}
       │
       ▼  (only if dkg_export_requested)

  [DKG Exporter]
        11. Pull consumer on noosphere.l4.dkg.export_requested.{α-did}
            Publishes Knowledge Asset on OriginTrail DKG
            Links: arweave_tx_id + hindsight_memory_id + engram_snapshot_id
            Access: org-permissioned or public (per policy)
```

---

## 11. MCP Tool Surface and Authorization Flow

```text
  NOOSPHERE MEMORY MCP SERVER — TOOL SURFACE

  ┌──────────────────┬──────────────────────────────────────────────────┐
  │  MCP Tool        │  Action                                          │
  ├──────────────────┼──────────────────────────────────────────────────┤
  │  memory_get      │  Read MEMORY.md at a given agent / revision      │
  │  memory_diff     │  Show diff between base and proposed change      │
  │  memory_propose  │  Create staging branch/PR for MEMORY.md change   │
  │  memory_review   │  Approve or reject a pending PR                  │
  │  memory_merge    │  Merge approved PR to protected main branch      │
  │  memory_subscribe│  Subscribe to commit/PR events for an agent repo │
  └──────────────────┴──────────────────────────────────────────────────┘

  INTEGRATION POINTS:

  ├── MCP-based agents (OpenClaw, moltbot, custom) call Noosphere Memory
  │   MCP tools instead of talking directly to Engram, GitLawb, or Hindsight.
  │
  ├── GitLawb: DID + UCAN-native repo/file/PR operations; canonical backend
  │   for per-agent MEMORY.md governance and signed history.
  │
  ├── Engram: per-agent internal hot-memory engine (SQLite/FTS); accessed
  │   by local runtimes or via its own MCP server; Noosphere exports
  │   curated state periodically into MEMORY.md.
  │
  ├── MemPalace: local sidecar with MCP server for high-quality local
  │   episodic/semantic recall; incorporates MEMORY.md anchors and
  │   Engram sync records.
  │
  ├── Hindsight: reachable via HTTP or Python client (MCP wrapper optional);
  │   cluster-level L2 substrate for sovereign retain/recall/reflect
  │   across agents and sessions.
  │
  └── NATS JetStream: event bus for all async lifecycle events across the
      memory pipeline; workers use durable pull consumers per stream;
      GitLawb webhook bridge publishes into NATS subject namespace.

  AUTHORIZATION CHAIN PER TOOL CALL:

  Agent ──▶ MCP Client
              │
              │  (TLS + OAuth 2.1 Bearer Token)
              ▼
  Noosphere Memory MCP Server
              │
              ├─ 1. Validate OAuth token → resolve principal DID
              │
              ├─ 2. RBAC check:
              │      - Does this principal have the required role
              │        for the requested agent's memory?
              │      - Is this memory class allowed for this role?
              │        (sacred facts → curator+ only for merge)
              │
              ├─ 3. UCAN check:
              │      - Does principal have a valid UCAN delegation
              │        for the exact resource/ability/branch?
              │      - Is the delegation unexpired and unrevoked?
              │        (revocation checked against NATS KV_UCAN_REVOCATIONS)
              │      - Is the chain of trust intact?
              │
              ├─ 4. If all pass → forward to GitLawb MCP operation
              │      with the agent's capability token
              │
              └─ 5. Log: principal, tool, resource, decision,
                       UCAN CID, timestamp → Audit log + NATS event
```

---

## 12. Multi-Agent Sharing and RBAC Policy Matrix

```text
  SCENARIO: Multi-agent swarm accessing Agent α's memory

  ┌─────────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
  │  Role           │ memory_get   │ memory_propose│ memory_review│ memory_merge │
  ├─────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
  │  owner agent    │ ✅ always     │ ✅ always     │ ✅ always     │ ✅ always     │
  │  peer_reader    │ ✅ MEMORY.md  │ ❌            │ ❌            │ ❌            │
  │  contributor    │ ✅ MEMORY.md  │ ✅ staging    │ ❌            │ ❌            │
  │  curator agent  │ ✅ always     │ ✅ always     │ ✅ always     │ ✅ policy-gated│
  │  human operator │ ✅ always     │ ✅ always     │ ✅ always     │ ✅ always     │
  └─────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘

  SHARING MODES:

  Private (default)
    Agent α's Engram store and MEMORY.md are accessible only to owner
    and explicitly delegated peers. No agent can propose or read without
    a UCAN delegation.

  Group-Shared
    A group policy grants peer_reader or contributor rights to a set of
    agent DIDs. Managed via RBAC group roles + batch UCAN issuance
    (e.g., "mining-agent-group").

  Org-Wide L2 Access
    Hindsight (L2) may be queried by any agent in the tenant with an
    appropriate scope token — no per-agent repo delegation required for
    read. Scoped by agent_id / tenant in the Hindsight recall API.

  Global / Verifiable (opt-in)
    Only explicitly exported Knowledge Assets are visible on OriginTrail
    DKG. No agent's full Engram store or MEMORY.md is ever automatically
    public.

  ROLLBACK POLICY:
  ─────────────────────────────────────────────────────────────────────
  1. Revert bad commit on MEMORY.md main branch (git revert C_bad)
  2. Publish to NATS: noosphere.memory.reverted.{agent_did}
       {agent, reverted_commit, reason, operator_did}
  3. Engram sync worker (pull consumer on memory.reverted) marks
       affected records as "superseded"
  4. MemPalace re-fetches MEMORY.md → removes reverted facts from palace
  5. Hindsight marks affected memories as "superseded" via update API
  6. WeaveDB row retains record for audit; Arweave snapshot is permanent
     but annotated as superseded in WeaveDB index
  ─────────────────────────────────────────────────────────────────────
```

---

*Generated for the [NoosphereOS](https://github.com/enuno/NoosphereOS) project.*

*Architecture ratified April 2026. Sovereign stack: **Engram + GitLawb MEMORY.md (L0)**, **MemPalace (L1)**, **Hindsight (L2)**, **WeaveDB + Arweave/ArDrive (L3)**, **OriginTrail DKG (L4)**. Event bus: **NATS JetStream**.*

*NoosphereOS is a fully sovereign multi-agent memory operating system combining web2 operational control (Engram, GitLawb, MemPalace, Hindsight, NATS JetStream) with web3 permanence and verifiable knowledge sharing (Arweave/ArDrive, WeaveDB, OriginTrail DKG).*
