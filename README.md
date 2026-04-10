# NoosphereOS
***

NoosphereOS is a sovereign **multi‑layer memory operating system for AI agents**. It gives each agent a sovereign, auditable memory stack while enabling **scoped sharing, strong RBAC, and verifiable global knowledge** across agents, clusters, and organizations that combines web2 operational control with web3 permanence and verifiable knowledge sharing. Its ratified architecture uses Engram as the per-agent L0 hot-memory engine with a GitLawb-tracked MEMORY.md export per agent, MemPalace as the L1 local episodic/semantic memory OS, Hindsight as the self-hosted PostgreSQL-backed L2 distributed memory substrate, WeaveDB + Arweave/ArDrive as the L3 immutable archive, and OriginTrail DKG as the L4 verifiable shared knowledge layer.

At its core, NoosphereOS treats **memory as moral architecture**: hot memory is human‑legible and versioned; working memory is local and interpretable; long‑term memory is scalable and queryable; archival memory is immutable; and shared memory is verifiable and explicitly published.

***

## High‑level architecture

NoosphereOS defines five memory layers:

1. **L0 – Hot Memory (Engram + GitLawb `MEMORY.md`)**
    - One GitLawb repo **per agent**, e.g. `did:gitlawb:repo:agent-α-memory`, remains the canonical hot‑memory repo.
    - Engram serves as the agent’s internal persistent memory store; from it, NoosphereOS generates and updates an exported `MEMORY.md` snapshot that captures anchors, sacred facts, and core preferences in a human‑readable form.
    - Proposals and drafts to change `MEMORY.md` live in staging branches or `/staging/` paths; merges into `main` are gated by NoosphereOS policy and curator/operator review.
    - Cross‑agent access to `MEMORY.md` is delegated via **UCAN capabilities** scoped to repo/path/branch and mediated by memory‑specific MCP tools (`memory_get`, `memory_propose`, `memory_review`, `memory_merge`).
2. **L1 – Local Memory OS (MemPalace)**
    - Per‑agent or per‑pod MemPalace instance.
    - Loads `MEMORY.md` and builds an **episodic + semantic** memory palace: rooms/wings, temporal triples, local vector index.
    - Optimized for **local‑first recall, interpretability, and low latency**; acts as each agent’s immediate long‑term memory.
3. **L2 – Distributed Memory Substrate (Hindsight)**
    - Cluster‑ or tenant‑wide sovereign memory substrate for **cross‑session,** **cross‑agent** recall and learning.
    - Stores structured memories derived from agent activity and `MEMORY.md` promotions, including extracted facts, entities, relationships, and temporal links.
    - Supports Hindsight’s core retain / recall / reflect model, enabling durable memory ingestion, multi‑strategy retrieval, and reflective synthesis over prior experience.
    - Provides hybrid retrieval through semantic search, keyword/BM25 matching, entity/graph traversal, and temporal reasoning, with reranking for high‑quality agent recall.
    - Exposes self‑hostable MCP and HTTP interfaces that fit NoosphereOS’s sovereign, PostgreSQL‑backed control plane.
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

- **Per‑agent sovereignty:** every agent owns its own Engram store and GitLawb‑backed `MEMORY.md` repository, with a complete, signed history of hot‑memory exports and edits.
- **Human‑legible hot memory:** sacred facts, long‑lived preferences, and identity‑critical details are curated into `MEMORY.md` for direct inspection and bootstrap context, even though richer structure lives inside Engram, MemPalace, and Hindsight.
- **Promotion, not dumping:** raw interactions flow into Engram, MemPalace, and Hindsight first; only **promoted** facts and anchors are written back to `MEMORY.md` via a proposal/review/merge pipeline.
- **Capability‑based access:** cross‑agent memory access is delegated via UCANs on precise resources (`repo://…/MEMORY.md`, `refs/heads/memory-staging/*`), never via broad shared git credentials, and all memory MCP tools enforce these capabilities.
- **Layered sharing:**  
  - L0/L1 (Engram + `MEMORY.md` + MemPalace) are **agent‑local**;  
  - L2 (Hindsight) is **tenant/cluster‑wide**;  
  - L3 (WeaveDB + Arweave/ArDrive) is **immutable org‑level history**;  
  - L4 (OriginTrail DKG) is **globally verifiable knowledge**, explicitly exported.
- **Rollback and auditability:** any sacred fact can be traced to an Engram record, a git commit, proposer DID, curator DID, a Hindsight observation, and a permaweb snapshot; bad merges can be reverted and replayed cleanly into L1/L2.

***

## Data flow: proposing a new sacred fact

Example operation: **Agent β proposes a new sacred fact for Agent α**.

1. **Proposal (Engram + GitLawb staging)**  
    - Agent β writes the candidate fact and rationale into its own Engram store (e.g., as an observation or reflection) and then calls `memory_propose` on the Noosphere Memory MCP server with `{fact, justification}` targeting Agent α.
    - MCP OAuth identifies β; RBAC verifies β has `memory_contributor` rights for α; the capability broker uses or mints a UCAN with `repo/file/propose` on `repo://did:gitlawb:α-memory/path/MEMORY.md` and a staging branch.
    - Via GitLawb MCP, Noosphere creates a staging branch, applies a patch to `MEMORY.md`, and opens a PR to `main`, annotating it with proposer DID, UCAN CID, and an internal `fact_id`.

2. **Review and merge (L0 governance)**  
    - Curator agent γ (or a human operator) calls `memory_diff` and inspects the proposed change to `MEMORY.md` in the context of existing sacred facts and Engram state.
    - On approval, γ calls `memory_review` then `memory_merge`. RBAC enforces curator role; UCAN enforces `repo/pr/review` and `repo/pr/merge` on the specific PR/resource.
    - GitLawb merges the PR into `main` (protected branch), creating commit `C_main` with the updated `MEMORY.md`.
    - Noosphere emits `memory_promoted {agent: α, commit: C_main, fact_id, proposer: β, curator: γ}` on its event bus.  

3. **Engram + MemPalace update (L0/L1)**  
    - An Engram sync worker for Agent α listens for `memory_promoted`, fetches `MEMORY.md` at `C_main`, and writes a structured “sacred_fact” record back into α’s Engram store with full provenance.
    - MemPalace for Agent α also subscribes to memory events or repo events; on `memory_promoted`, it fetches `MEMORY.md` at `C_main`, parses the sacred facts section, updates its palace (e.g., a “Sacred Facts” room), and marks these items as high‑priority context for recall.

4. **Hindsight update (L2)**  
    - A Hindsight ingest worker listens for `memory_promoted`, computes the diff for `MEMORY.md`, and creates a **structured sacred_fact memory** for Agent α, capturing content, entities, relationships, timestamps, and provenance (β, γ, commit hash, Engram IDs).
    - It calls Hindsight’s retain API (HTTP or client library), where the new memory is stored in PostgreSQL-backed tables and indexed for semantic, keyword, temporal, and graph-based recall.
    - This sacred fact now participates in Hindsight’s retain / recall / reflect pipeline and is available for cross‑session, cross‑agent queries under NoosphereOS policy.

5. **Archival snapshot (L3)**  
    - For sacred facts or other policy-marked changes, an archive worker pushes the new `MEMORY.md` version at `C_main` to Arweave/ArDrive and records an index in WeaveDB, tying together `arweave_tx_id`, repo DID, commit, `fact_id`, Engram record IDs, and Hindsight memory IDs.

6. **DKG export (L4, optional)**  
    - If the fact is meant to be globally verifiable, an exporter publishes it as a Knowledge Asset on OriginTrail DKG, referencing the Arweave snapshot and the corresponding Hindsight memory (and optionally Engram IDs) as sources.
    - Access to this Knowledge Asset (public, org-scoped, or group-scoped) is determined by NoosphereOS export policy and DKG ACLs.

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

- **MCP‑based agents** (OpenClaw, custom MCP clients, etc.) call Noosphere’s Memory MCP server for `memory_get / memory_propose / memory_review / memory_merge` instead of talking directly to git, Engram, MemPalace, or Hindsight.
- **GitLawb** exposes repo/file/PR operations with DID + UCAN support and an MCP surface, making it the natural backend for per‑agent `MEMORY.md` governance and signed history.
- **Engram** runs per agent as the internal hot‑memory engine (SQLite/FTS‑backed), and can be accessed directly by local runtimes or via its own MCP server, while Noosphere periodically exports curated state into `MEMORY.md`.
- **MemPalace** sits alongside MCP clients (typically as a local sidecar) and can also be exposed via an MCP server, providing high‑quality local episodic/semantic recall that incorporates `MEMORY.md` anchors.
- **Hindsight** is reachable via HTTP or client libraries (and via an MCP wrapper where needed) and is treated as the cluster‑level, PostgreSQL‑backed L2 substrate for structured retain / recall / reflect across agents and sessions.

***

## Why use NoosphereOS?

- You want **per‑agent, inspectable memory** rather than opaque embedding stores.
- You need a **multi‑agent memory fabric** with granular permissions: some agents may read, a few may propose, only curators may merge.
- You value **immutable, permaweb‑backed archives** of constitutions, identity, and high‑impact decisions.
- You want to **share some knowledge globally**, with verifiable provenance, while keeping most memory local and private.
