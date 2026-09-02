# To Be Added

Backlog of notes still to be written. These are gaps identified from an
architect's point of view — topics that are referenced across the existing notes
but never explained, or not covered at all.

Tick items off as notes are written and linked into [README.md](README.md).

---

## Security architecture

Current coverage is thin — [technical/security/jwt-and-oauth-authentication.md](technical/security/jwt-and-oauth-authentication.md)
is only JWT / OAuth / Okta mechanics. An architect is expected to own:

- [ ] **Threat modeling** — STRIDE, attack trees, trust boundaries, data-flow diagrams for security
- [ ] **OWASP Top 10** and **OWASP API Security Top 10**
- [ ] **Secrets management and key rotation** — Vault / cloud KMS, envelope encryption, encryption at rest and in transit, mTLS
- [ ] **Authorization models beyond RBAC** — ABAC, ReBAC, policy engines (OPA / Cedar)
- [ ] **Audit logging and compliance framing** — GDPR, PCI-DSS, SOC 2, ISO 42001 (the last matters given the AI-governance angle in the target role)
- [ ] **Supply-chain security** — SBOM, dependency scanning, container image scanning
- [ ] **PII handling** — tokenization, data residency

---

## Distributed systems fundamentals

Named across the notes (CAP, saga, quorum, consensus) but never explained:

- [ ] **Consensus (Raft / Paxos)** — what it does and why quorum size matters
- [ ] **Quorum math** — `R + W > N`, read-repair, hinted handoff
- [ ] **Logical clocks** — vector clocks, Lamport timestamps
- [ ] **Distributed locking and its dangers** — fencing tokens, lease expiry, split brain
- [ ] **Clock skew** — why "just use timestamps" breaks
- [ ] **Idempotency / deduplication / exactly-once** — as a first-class topic, not a footnote

---

## Other known gaps (lower priority)

From the earlier architect-POV review — see the conversation for detail:

- [ ] **API design & contracts** — versioning, pagination, RFC 9457 error model, idempotency keys, GraphQL, gRPC depth, contract testing, BFF, AsyncAPI
- [ ] **Performance & scalability engineering** — back-of-envelope capacity estimation, tail latency, load balancing (L4/L7), CDN/edge, pool sizing, autoscaling signals, profiling
- [ ] **Cloud-native & Kubernetes depth** — K8s resource model, HPA/VPA, network policies, operators, IaC/GitOps, image security, FinOps, serverless patterns
- [ ] **Architecture practice & governance** — ADRs as a practice, architecture-characteristics prioritisation, ATAM-style trade-off analysis, fitness functions, Conway's Law / Team Topologies, migration patterns (branch by abstraction, parallel run), tech-debt framework
- [ ] **Testing strategy** — test pyramid, consumer-driven contract testing, test data management, chaos/resilience testing, non-functional testing
- [ ] **Gen AI for architects** — RAG architecture depth (chunking, embeddings, vector DB selection, hybrid search, reranking), eval & guardrails, prompt injection, cost/latency/model-tiering, agentic orchestration & MCP

### Existing stubs to fill in

- [ ] [technical/gen-ai/gen-ai-basics.md](technical/gen-ai/gen-ai-basics.md) — the RAG pipeline section is marked TODO
- [ ] [technical/data/database-and-data-architecture-questions.md](technical/data/database-and-data-architecture-questions.md) — unanswered question list (much now covered by [data-architecture.md](technical/data/data-architecture.md))
- [ ] [technical/architecture/design-patterns-reading-list.md](technical/architecture/design-patterns-reading-list.md) — "Topics to Read" placeholder
- [ ] [technical/gen-ai/gen-ai-advanced-reading-list.md](technical/gen-ai/gen-ai-advanced-reading-list.md) — topics-to-read list
