# DexHub-Public

Version: 6.8 (Node 20 + TypeScript + SQLite)
Purpose: DexHub is a local “AI OS” backend designed for durable memory, reasoning, and synthesis.
It captures and relates user knowledge in a graph of memory documents and dialectical edges — forming the substrate for a symbolic reasoning engine.

Core Stack

🧩 Core Stack
Layer	Tech	Purpose
Runtime	Node 20 + Express + Zod	ESM TypeScript backend with runtime validation

Database	SQLite (better-sqlite3)	Durable local store for docs + edges + chat history

Schema	/schema.sql + idempotent migrations	docs, edges, chats, messages, message_docs, memory_fts

Search	SQLite FTS5 (memory_fts)	Full-text semantic lookup across memory

API routes	/api/memory, /api/edges, /api/graph, /api/dialectic, /api/system	CRUD + graph analytics

Scripts	migrate, seed, verify, smoke	Local dev / health / data bootstrap


1. Prerequisites

Node.js v20+

npm (or pnpm/yarn)

SQLite (preinstalled on most systems)

Clone & Install

git clone https://github.com/yourusername/dexhub.git
cd dexhub
npm install

Environment Setup

Create a .env.local file in the project root:
PORT=5555
SQLITE_PATH=server/data/dexhub.sqlite
EMBEDDING_MODE=hash
(You can change EMBEDDING_MODE to off if embeddings are disabled.)

Database Migration

Create tables and FTS indices:
npm run migrate

(Optional) Seed with Example Data
npm run seed

Run the Server
npm run dev
→ Visit http://localhost:5555

Verify Setup

Quick health check:

npm run verify


 Dialectic Engine (Belief, Support/Attack, Tension)

This describes the purpose, math, and update loop of the dialectic engine used to maintain belief and tension for every node (doc or concept). The engine aggregates signed evidence along graph paths, applies time decay and path attenuation, and updates belief via a numerically stable Euler step. When contradictions persist, it can optionally trigger synthesis jobs (with user feedback if enabled):

## Glossary
- Node v: a doc or concept.
- Signed edges: supports, contradicts, elaborates (soft support scaled by ρ).
- Path attenuation: α^(L−1) for a length-L path (L ∈ {1,2}).
- Decay: time decay on evidence via exp(−λ·age_days).
- Belief B(v) ∈ (0,1): confidence that v is “accepted/true/useful”.
- Support mass S⁺(v), Attack mass S⁻(v): accumulated positive/negative evidence.
- Net force Δ(v) = S⁺(v) − S⁻(v).
- Tension T(v): high when both S⁺ and S⁻ are strong
## Parameters (configurable)
- α ∈ (0,1): path attenuation (default 0.7)
- ρ ∈ (0,1): elaborates-as-soft-support multiplier (default 0.5)
- λ ≥ 0: time decay per day (default 0.01)
- β > 0: belief sigmoid temperature (default 1.0)
- η ∈ (0,1]: Euler step size / learning rate (default 0.25)
- ε = 1e-8: numerical stabilizer
- Ω ∈ [0,∞): synthesis tension threshold (default 0.4)
- Γ ∈ [0,∞): friction on belief updates (default 0.05)
- HOPS ∈ {1,2}: bounded path length for aggregation (default 1)
- rhoConcept ∈ (0,1): elaborates-as-soft-support multiplier when target is concept (default 0.5)
- lambdaConcept ≥ 0: time decay per day for concept targets (default 0.01)
## Edge Contribution Weights
For an edge e: u → v with kind k and age age_days, we use calibrated probabilities for support/contradiction when available (`edge_semantics`), otherwise map from legacy weight/confidence.
```
base_support = logit(p_support(e))
base_contra  = logit(p_contra(e))
decay        = exp(−λ · age_days)

w_supp(e)   = base_support * decay
w_contra(e) = base_contra  * decay
w_elab(e)   = ρ * w_supp(e)   (if kind == elaborates)
signed(e)   = w_supp(e) − w_contra(e)
```
## Node Aggregation (per update window)
Signed path accumulation, bounded by HOPS (1 or 2):
```
S⁺(v) = Σ_paths u…→v  α^(L−1) * max(0, signed(path))
S⁻(v) = Σ_paths u…→v  α^(L−1) * max(0, −signed(path))
```
Path rules:
- Length 1: use `signed(e)` for u→v.
- Length 2: signed magnitude multiplies along u→m→v; sign flips when a contradiction is in the chain; attenuation α multiplies the magnitude.

Belief & tension:
```
Δ(v)       = S⁺(v) − S⁻(v)
Tension(v) = 2 * S⁺ * S⁻ / (S⁺ + S⁻ + ε)
Belief*(v) = σ(β · Δ(v))
Belief_next(v) = (1−η)·Belief(v) + η·Belief*(v) − Γ·(Belief(v)−0.5)
```
The friction term Γ·(Belief−0.5) biases toward neutrality to reduce oscillation. We also track EMA values:
```
EMA_belief(v)  ← (1−k)·EMA_belief(v) + k·Belief_next(v)
EMA_tension(v) ← (1−k)·EMA_tension(v) + k·Tension(v)
```
## Update Loop (sketch)

1. Load incoming edges for touched nodes; compute per-edge signed scores with decay and elaboration (use `rhoConcept`/`lambdaConcept` when target is concept).
2. Aggregate S⁺/S⁻ via 1-hop contributions; optionally add 2-hop contributions with attenuation α.
3. Update belief via the Euler step; update EMA fields. Persist in `dialectic_state`.
4. If synthesis is enabled and `Tension(v) ≥ Ω`, enqueue a synthesis job (optionally requiring user approval).

## Synthesis (Optional)

Synthesis is optional, disabled by default. When enabled, and tension crosses threshold Ω, the engine enqueues a job into a DB-backed queue. If feedback mode is enabled, jobs are queued as `queued:feedback` and must be approved before processing.

Endpoints:

- Approve job: `POST /api/dialectic/synthesis/approve` Body `{ job_id }`
- Debug list jobs: `GET /api/dialectic/_debug/jobs?limit=10`
- Worker (debug): `POST /api/dialectic/_debug/synthesis-drain` (processes one job)

## Determinism & Numerical Stability

- Clamp and bound all inputs; guard all divisions with ε.
- Avoid randomness and tie-break with stable orderings.
- HOPS=2 is optional and bounded to avoid explosion.

## Testing Checklist

- Edge sign propagation through 1- and 2-hop paths.
- Attenuation (α < 1) reduces two-hop contributions.
- Tension behavior: T=0 if one side is zero; maximal when S⁺≈S⁻.
- Decay: older edges contribute less mass.
- Stability: with Γ>0 and Δ≈0, belief converges to ~0.5.
- Synthesis trigger: enqueue at most once while inflight.

Symbolic AI Layer

The symbolic layer treats nodes and edges as explicit semantic atoms, enabling reasoning beyond embeddings.

Function	Role
Symbolic Representation	Each doc = a proposition; edges = semantic relations (logical / dialectical).

Concept Synthesis	New symbols emerge via DRE syntheses → recorded as higher-order nodes.

Energy-like Flow	Belief and tension behave like energy potentials guiding synthesis.

JEPA-inspired Predictive State	Each synthesis approximates the “next best concept” — a predictive world-model snapshot.

Non-Markov History	Context carries forward: older contradictions still echo in reasoning paths.

Together, these form DexHub’s Dialectical Memory Engine (DME) — a bridge between symbolic logic and emergent, energy-based reasoning.

Next-Phase Roadmap (summary)

Search Route: FTS-first / semantic hybrid query endpoint.

Graph Views: Community / path / richness analytics.

Synthesis Logs: Track evolving concept states over time.

Auth: Optional API-key or local-session security.

OpenAPI Docs for full self-describing API.

Local LLM Module: integrate Llama-compatible runtime for in-graph reasoning.

UI Dashboard: Memory web, tension heatmap, synthesis timeline.
