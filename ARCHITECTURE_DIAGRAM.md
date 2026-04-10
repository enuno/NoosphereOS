# NoosphereOS — Architecture Diagrams

This document contains the full set of architecture diagrams for the NoosphereOS multi-agent memory operating system. For narrative descriptions of each component, see [README.md](./README.md).

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [Layer Definitions and Stack](#2-layer-definitions-and-stack)
3. [Control Plane and Agent Runtime](#3-control-plane-and-agent-runtime)
4. [L0: Hot Memory — GitLawb + MEMORY.md](#4-l0-hot-memory--gitlawb--memorymd)
5. [L0 Permission Model — UCAN + RBAC](#5-l0-permission-model--ucan--rbac)
6. [L1: Local Memory OS — MemPalace](#6-l1-local-memory-os--mempalace)
7. [L2: Distributed Memory Substrate — Supermemory](#7-l2-distributed-memory-substrate--supermemory)
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
│  │  Event Bus                                                       │   │
│  │  memory_promoted | memory_proposal_created | archive_snapshot    │   │
│  │  memory_replicated | dkg_exported                                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
    │                      │                      │                    │
    │ MCP (OAuth 2.1)      │ MCP (OAuth 2.1)      │ async events       │ async events
    ▼                      ▼                      ▼                    ▼

┌───────────────────┐   ┌────────────────────────────────────────────────┐
│  AGENT RUNTIME(S) │   │  NOOSPHERE MEMORY MCP SERVER                  │
│  OpenClaw         │   │  Exposes memory_get / propose / review / merge │
│  moltbot          │──▶│  Enforces RBAC + UCAN per call                 │
│  custom agents    │◀──│  Hides raw GitLawb / storage details           │
└───────────────────┘   └────────────────────────────────────────────────┘
         │                              │
         │                              │
         ▼                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│                         MEMORY LAYERS                                  │
│                                                                        │
│  L0  MEMORY.md + GitLawb repos (hot memory, per agent)                │
│  L1  MemPalace             (local episodic/semantic OS, per agent)     │
│  L2  Supermemory           (distributed substrate, per tenant/org)     │
│  L3  WeaveDB + Arweave     (immutable archive, per org)                │
│  L4  OriginTrail DKG       (verifiable shared knowledge, selective)    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Layer Definitions and Stack

```text
  GRANULARITY ◀────────────────────────────────────────────▶ BREADTH

  L0 │ MEMORY.md + GitLawb │ per-agent │ hot, human-readable, signed history
     │
  L1 │ MemPalace           │ per-agent/pod │ local episodic + semantic OS
     │
  L2 │ Supermemory         │ per-tenant/cluster │ distributed, cross-agent recall
     │
  L3 │ WeaveDB + Arweave   │ per-org/global │ immutable archive, artifact store
     │
  L4 │ OriginTrail DKG     │ shared/global │ verifiable, cross-org knowledge graph


  SCOPE OF SHARING:
  ─────────────────────────────────────────────────────────────────
  L0: Agent-local (shared only via explicit UCAN delegation)
  L1: Agent-local (no remote sync by default)
  L2: Tenant/org-wide (multi-agent access with policy)
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
│   │  On read:  L0 (anchor) → L1 (local) → L2 (distributed)         │   │
│   │  On write: stages to L1/L2, promotes to L0 via proposal         │   │
│   │  On milestone: triggers L3 snapshot + optional L4 export        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  EVENT BUS                                                      │   │
│   │  Topics:                                                        │   │
│   │    memory_proposal_created  {agent, proposer, repo_did, pr_id}  │   │
│   │    memory_promoted          {agent, commit, fact_id, curator}   │   │
│   │    local_memory_updated     {agent, commit, anchors}            │   │
│   │    archive_snapshot_created {agent, commit, arweave_tx_id}      │   │
│   │    dkg_exported             {agent, knowledge_asset_id}         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4. L0: Hot Memory — GitLawb + MEMORY.md

```text
┌──────────────────────────────────────────────────────────────────────────┐
│  L0 — PER-AGENT GitLawb REPO                                            │
│  Repo DID: did:gitlawb:repo:agent-α-memory                              │
│                                                                          │
│  Branch: main (PROTECTED — no direct push except owner/operator)        │
│  ├── MEMORY.md          ← canonical hot memory (sacred facts,           │
│  │                         anchors, core preferences, constraints)       │
│  └── /staging/          ← proposals, patches, drafts from agents        │
│       └── <actor-did>/  ← per-proposer staging area                     │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  MEMORY.md STRUCTURE (example)                                    │  │
│  │                                                                   │  │
│  │  # Agent α — Hot Memory                                           │  │
│  │  last_updated: 2026-04-10T08:00:00Z                               │  │
│  │  commit: C_main_abc123                                            │  │
│  │                                                                   │  │
│  │  ## Sacred Facts                                                  │  │
│  │  - [fact-001] <content> (promoted by β, curated by γ)            │  │
│  │  - [fact-002] <content> (promoted by agent, auto-curated)        │  │
│  │                                                                   │  │
│  │  ## Anchors                                                       │  │
│  │  - Current task: ...                                              │  │
│  │  - Core identity: ...                                             │  │
│  │  - Risk constraints: ...                                          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  GitLawb Repo Events (subscribed by Noosphere Event Bus):               │
│    - pr_opened, pr_reviewed, pr_merged, commit_created                  │
│    - Each event includes: author DID, UCAN CID, commit SHA              │
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
  ├─────────────────────┼───────────────────────────────────────────────┤
  │  peer_reader        │ repo/file/read (MEMORY.md only)               │
  │                     │ repo/event/subscribe                          │
  ├─────────────────────┼───────────────────────────────────────────────┤
  │  contributor agent  │ repo/file/read                                │
  │                     │ repo/file/propose (staging/* only)            │
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
│   ├── On memory_promoted event: re-parses sacred facts, updates rooms   │
│   └── On task completion: writes new episodic entries locally           │
│                                                                          │
│   Recall interface:                                                      │
│   ├── MCP tools: memory_search, memory_reflect                          │
│   └── Direct SDK for co-located agent runtimes                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 7. L2: Distributed Memory Substrate — Supermemory

```text
┌──────────────────────────────────────────────────────────────────────────┐
│  L2 — SUPERMEMORY CLUSTER (per tenant / org)                            │
│                                                                          │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  INGEST                                                       │     │
│   │  - Subscribes to memory_promoted, task_completed, episodic    │     │
│   │  - Builds structured observational memories:                  │     │
│   │      {subject, type, content, source, provenance, timestamp}  │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                          │                                               │
│                          ▼                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  MEMORY STORE                                                 │     │
│   │  - Distributed vector DB (semantic search)                    │     │
│   │  - Relational / doc DB (structured memory, provenance)        │     │
│   │  - Temporal index (time-aware recall and decay)               │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                          │                                               │
│                          ▼                                               │
│   ┌───────────────────────────────────────────────────────────────┐     │
│   │  RECALL API                                                   │     │
│   │  - MCP server: memory tools for cross-agent queries           │     │
│   │  - Scoped by: agent_id / user_id / tenant                     │     │
│   │  - Returns: top-k memories + provenance + source commit CID   │     │
│   └───────────────────────────────────────────────────────────────┘     │
│                                                                          │
│   Memory types handled:                                                  │
│   ├── sacred_fact        (from MEMORY.md promotions)                    │
│   ├── episodic           (task logs, conversation summaries)             │
│   ├── preference         (derived from interactions)                    │
│   └── external_doc       (from ingested files / tools / searches)       │
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
│  │    { agent_did, commit_sha, type, promoted_at }                 │    │
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
│  │      fact_ids: [...],                                           │    │
│  │      type: "sacred_fact_snapshot" | "constitution" | ...,       │    │
│  │      promoted_at,                                               │    │
│  │      proposer_did,                                              │    │
│  │      curator_did                                                │    │
│  │    }                                                            │    │
│  │  Queryable by agents and operators for reference lookups        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Trigger: archive worker subscribes to memory_promoted events           │
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
│  Input from L2: Supermemory observation ID, provenance                  │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  KNOWLEDGE ASSET (per exported fact or claim)                   │    │
│  │  {                                                              │    │
│  │    subject: agent_did / entity,                                 │    │
│    │    claim: <semantic triple or JSON-LD statement>,              │    │
│  │    provenance: {proposer, curator, promoted_at, commit},        │    │
│  │    sources: [arweave_tx_id, supermemory_obs_id],                │    │
│  │    access: public | org-permissioned | agent-group              │    │
│  │  }                                                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Use cases:                                                              │
│   - Cross-org agent knowledge sharing with verifiable provenance        │
│   - Audits and compliance proofs                                         │
│   - Noosphere "commons": shared world-model for a multi-org swarm       │
│                                                                          │
│  Trigger: exporter subscribes to dkg_export_requested events            │
│  Policy:  explicit opt-in only — NO automatic full-memory DKG export    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Data Flow: Sacred Fact Promotion (L0 → L1 → L2 → L3)

```text
  OPERATION: Agent β proposes new sacred fact for Agent α
  ─────────────────────────────────────────────────────────────────────────

  [Agent β Runtime]
       │  1. Identifies candidate sacred fact
       │     (e.g., long-lived constraint, identity anchor)
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
  [Noosphere Event Bus]
       │  4. Emits: memory_proposal_created
       │     {agent: α, proposer: β, repo_did, pr_id}
       │
       ▼
  [Curator Agent γ / Human Operator]
       │  5. Receives notification
       │     Calls memory_diff(pr_id) → inspects change
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
       │     Emits repo events: pr_merged, commit_created(C_main)
       │
       ▼
  [Noosphere Event Bus]
       │  8. Emits: memory_promoted
       │     {agent: α, commit: C_main, fact_id, proposer: β, curator: γ}
       │
       ├──────────────────────────────────────────────────┐
       │                                                  │
       ▼  (parallel, async)                               ▼  (parallel, async)

  [MemPalace — Agent α]                        [Supermemory Ingest Worker]
       │  9a. Subscribes to memory_promoted         │  9b. Subscribes to memory_promoted
       │      Fetches MEMORY.md at C_main           │      Reads MEMORY.md diff at C_main
       │      Parses sacred facts section           │      Builds structured observation:
       │      Updates palace:                       │        {subject: α,
       │        - "Sacred Facts" room               │         type: sacred_fact,
       │        - New triples inserted              │         content: X,
       │        - High-priority recall flag set     │         source: gitlawb://α@C_main
       │      Emits: local_memory_updated           │         provenance: {β, γ, ts}}
       │                                            │      Writes to Supermemory API
       │                                            │      Supermemory indexes fact
       │                                            │      (semantic + temporal)
       │
       ▼  (if sacred_fact policy = archive)

  [Archive Worker]
       │  10. Fetches MEMORY.md at C_main
       │      Writes to Arweave/ArDrive:
       │        tags: {agent: α, commit: C_main, type: sacred_fact_snapshot}
       │      Writes index row to WeaveDB:
       │        {arweave_tx_id, agent: α, commit: C_main, fact_ids, ...}
       │      Emits: archive_snapshot_created {arweave_tx_id}
       │
       ▼  (only if dkg_export_requested)

  [DKG Exporter]
        11. Publishes Knowledge Asset on OriginTrail DKG
            Links: arweave_tx_id + supermemory_obs_id
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
              │      - Is the chain of trust intact?
              │
              ├─ 4. If all pass → forward to GitLawb MCP operation
              │      with the agent's capability token
              │
              └─ 5. Log: principal, tool, resource, decision,
                       UCAN CID, timestamp → Audit log / Event bus
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
    Agent α's memory is readable only by owner + explicitly delegated peers.
    No other agent can propose or read without a UCAN delegation.

  Group-Shared
    A group policy grants peer_reader or contributor rights to a set of agent DIDs.
    Managed via RBAC group roles + batch UCAN issuance (e.g., "mining-agent-group").

  Org-Wide L2 Access
    Supermemory (L2) may be queried by any agent in the tenant with appropriate
    scope token — no per-agent repo delegation required for read.

  Global / Verifiable (opt-in)
    Only explicitly exported Knowledge Assets are visible on OriginTrail DKG.
    No agent's full memory is ever automatically public.

  ROLLBACK POLICY:
  ─────────────────────────────────────────────────────────────────────
  1. Revert bad commit on MEMORY.md main branch (git revert C_bad)
  2. Emit memory_reverted { agent, reverted_commit, reason }
  3. MemPalace re-fetches MEMORY.md → removes reverted facts from palace
  4. Supermemory marks affected observations as "superseded"
  5. WeaveDB row retains record for audit; Arweave snapshot is permanent
     but can be annotated as superseded in WeaveDB index
  ─────────────────────────────────────────────────────────────────────
```

---

*Generated for the [NoosphereOS](https://github.com/enuno/NoosphereOS) project.*
*Architecture ratified April 2026. Components: GitLawb (L0), MemPalace (L1), Supermemory (L2), WeaveDB + Arweave/ArDrive (L3), OriginTrail DKG (L4).*
