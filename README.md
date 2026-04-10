# NoosphereOS
***

NoosphereOS is a **multi‑layer memory operating system for AI agents**. It gives each agent a sovereign, auditable memory stack while enabling **scoped sharing, strong RBAC, and verifiable global knowledge** across agents, clusters, and organizations.

At its core, NoosphereOS treats **memory as moral architecture**: hot memory is human‑legible and versioned; working memory is local and interpretable; long‑term memory is scalable and queryable; archival memory is immutable; and shared memory is verifiable and explicitly published.

***

## High‑level architecture

NoosphereOS defines five memory layers:

1. **L0 – Hot Memory (GitLawb + `MEMORY.md`)**
    - One GitLawb repo **per agent**, e.g. `did:gitlawb:repo:agent-α-memory`.
    - `MEMORY.md` is the authoritative, human‑readable hot memory file (anchors, sacred facts, core preferences).
    - Proposals and drafts live in staging branches or `/staging/` paths; merges to `main` are gated by policy and review.
    - Access is delegated via **UCAN capabilities** (per repo/path/branch) and exposed through memory‑specific MCP tools (`memory_get`, `memory_propose`, `memory_review`, `memory_merge`).
2. **L1 – Local Memory OS (MemPalace)**
    - Per‑agent or per‑pod MemPalace instance.
    - Loads `MEMORY.md` and builds an **episodic + semantic** memory palace: rooms/wings, temporal triples, local vector index.
    - Optimized for **local‑first recall, interpretability, and low latency**; acts as each agent’s immediate long‑term memory.
3. **L2 – Distributed Memory Substrate (Supermemory)**
    - Cluster‑ or tenant‑wide memory substrate for **cross‑session, cross‑agent** recall.
    - Stores structured episodic / observational memories derived from agent activity and `MEMORY.md` promotions.
    - Provides state‑of‑the‑art semantic + temporal retrieval via MCP or HTTP APIs.
4. **L3 – Immutable Archive (WeaveDB + Arweave/ArDrive)**
    - **Arweave/ArDrive** stores immutable snapshots and large artifacts:
        - `MEMORY.md` snapshots at milestones,
        - SOUL.md / NOOSPHERE versions,
        - PDFs, datasets, contracts, design docs.
    - **WeaveDB** provides structured indexes and metadata over permaweb content for efficient querying.
5. **L4 – Verifiable Shared Knowledge (OriginTrail DKG)**
    - Selected facts/claims, with provenance, are exported as **Knowledge Assets** on OriginTrail DKG.
    - Links NoosphereOS memory to a **decentralized knowledge graph** with verifiable proofs and cross‑org discovery.

All layers are orchestrated by the **Noosphere control plane**, which handles identity (agent DIDs), UCAN capability brokering, RBAC policies, routing, and event flows.

***

## Key principles

- **Per‑agent sovereignty:** every agent owns its own GitLawb‑backed `MEMORY.md` repository, with a complete, signed history of hot‑memory edits.
- **Human‑legible hot memory:** sacred facts, long‑lived preferences, and identity‑critical details must live in `MEMORY.md`, not only in opaque databases.
- **Promotion, not dumping:** raw interactions flow into MemPalace and Supermemory first; only **promoted** facts and anchors are written into `MEMORY.md` via a proposal/review/merge pipeline.
- **Capability‑based access:** cross‑agent memory access is delegated via UCANs on precise resources (`repo://…/MEMORY.md`, `refs/heads/memory-staging/*`), never via broad shared git credentials.
- **Layered sharing:**
    - L0/L1 are **agent‑local**;
    - L2 is **tenant/cluster‑wide**;
    - L3 is **immutable org‑level history**;
    - L4 is **globally verifiable knowledge**, explicitly exported.
- **Rollback and auditability:** any sacred fact can be traced to a git commit, proposer DID, curator DID, and permaweb snapshot; bad merges can be reverted and replayed into L1/L2.

***

## Data flow: proposing a new sacred fact

Example operation: **Agent β proposes a new sacred fact for Agent α**.

1. **Proposal**
    - Agent β calls `memory_propose` on the Noosphere Memory MCP server with `{fact, justification}` for Agent α.
    - MCP OAuth identifies β; RBAC checks β has `memory_contributor` rights for α; capability broker uses or mints a UCAN with `repo/file/propose` on `repo://did:gitlawb:α-memory/path/MEMORY.md` and a staging branch.
    - GitLawb MCP creates a staging branch, applies the patch to `MEMORY.md`, and opens a PR to `main`, annotating with proposer DID and UCAN info.
2. **Review and merge**
    - Curator agent γ (or human) calls `memory_diff` and inspects the proposed change.
    - On approval, γ calls `memory_review` then `memory_merge`.
    - RBAC enforces curator role; UCAN enforces `repo/pr/review` and `repo/pr/merge` on the relevant PR/resource.
    - GitLawb merges the PR into `main` (protected branch), creating commit `C_main` with updated `MEMORY.md`.
    - Noosphere emits `memory_promoted {agent: α, commit: C_main, fact_id, proposer: β, curator: γ}` on its event bus.
3. **MemPalace update (L1)**
    - MemPalace for Agent α subscribes to memory events or repo events; on `memory_promoted`, it fetches `MEMORY.md` at `C_main`.
    - It parses the sacred facts section, updates its palace (e.g., a “Sacred Facts” room), and marks these items as high‑priority context for recall.
4. **Supermemory update (L2)**
    - A Supermemory ingest worker listens for `memory_promoted`, computes the diff for `MEMORY.md`, and creates a **structured sacred_fact observation** with provenance (agent α, proposer β, curator γ, commit hash, timestamps).
    - It writes this observation into Supermemory’s APIs, where it becomes part of agent α’s durable, cross‑session memory.
5. **Archival snapshot (L3)**
    - For sacred facts or other policy‑marked changes, an archive worker pushes the new `MEMORY.md` version to Arweave/ArDrive and records an index in WeaveDB, tying together `arweave_tx_id`, repo DID, commit, and fact IDs.

(Optionally) 6. **DKG export (L4)**

- If the fact is meant to be globally verifiable, an exporter publishes it as a Knowledge Asset on OriginTrail DKG, linking back to the Arweave snapshot and Supermemory observation.

***

## Roles and permissions

NoosphereOS uses **RBAC + UCAN** to enforce least privilege:

- **Owner agent** – full rights over its own memory repo; can delegate to others.
- **Peer reader** – read‑only `MEMORY.md` access; no write or propose rights.
- **Contributor agent** – can propose changes (staging branches/PRs) but not merge.
- **Curator agent** – can review and merge memory PRs according to policy (especially for sacred/constitutional facts).
- **Human operator** – break‑glass authority and final override for critical changes.

All MCP access to memory tools is authenticated (OAuth 2.1), and each tool call is checked against RBAC+UCAN before hitting GitLawb or lower layers.

***

## Integration points

- **MCP‑based agents** (OpenClaw, custom MCP clients, etc.) call Noosphere Memory MCP tools instead of raw git or memory stores.
- **GitLawb** exposes repo/file/PR operations with DID + UCAN support and an MCP surface, making it a natural backend for `MEMORY.md` governance.
- **MemPalace** sits alongside MCP clients, either as a local sidecar process or integrated module, and can be exposed via its own MCP server for advanced recalls.
- **Supermemory** is reachable via MCP or direct HTTP and is treated as the cluster‑level memory substrate in NoosphereOS designs.

***

## Why use NoosphereOS?

- You want **per‑agent, inspectable memory** rather than opaque embedding stores.
- You need a **multi‑agent memory fabric** with granular permissions: some agents may read, a few may propose, only curators may merge.
- You value **immutable, permaweb‑backed archives** of constitutions, identity, and high‑impact decisions.
- You want to **share some knowledge globally**, with verifiable provenance, while keeping most memory local and private.
