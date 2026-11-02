---
hide:
    - toc
    - navigation
---

# Production RAG Architecture: Beyond Naive Vector Search

> The engineering breakdown of production retrieval augmented generation: hybrid lexical and semantic search, reciprocal rank fusion, cross encoder reranking pipelines, and token economics.

# Designing a Production-Grade Advanced RAG System

### Architecture of a Unified Intelligence Platform: ingestion, hybrid retrieval, multi-agent orchestration, access control, observability and delivery

---

## TL;DR

A production RAG system is a distributed system, not a prompt. The architecture below combines:

- **Event-driven, multi-modal ingestion** (Docling parsing, separate text / table / image paths, KEDA scale-to-zero workers)
- **Multi-stage retrieval** (semantic cache, HyDE, dense + BM25 hybrid search, Reciprocal Rank Fusion, cross-encoder reranking)
- **Supervisor-based multi-agent orchestration** (LangGraph + MCP tools)
- **An LLM gateway** for routing, fallback and cost control
- **Relationship-based (Zanzibar-style) authorization** enforced on retrieval
- **Continuous evaluation and full-stack observability** (OpenTelemetry + Grafana LGTM)
- **GitOps delivery** (GitLab CI, Helm, ArgoCD, Terraform, External Secrets)

Sections marked **Improvement** go beyond the reference diagram and describe what we would add before calling this production-ready.

---

## 1. Why naive RAG breaks in production

The tutorial version is: chunk, embed, store, top-k, stuff into prompt. It fails in an enterprise because:

| Failure mode | Consequence |
|---|---|
| Documents contain tables, charts and scans | Flattening to text destroys the facts users ask about most |
| Dense embeddings are weak on exact tokens | Ticket IDs, policy numbers and acronyms are missed |
| Short queries embed poorly | The question rarely resembles the passage that answers it |
| Retrieval ignores permissions | The most relevant chunk may be one the user must not see |
| Users want actions, not only answers | "Create a Jira ticket" is not a retrieval task |
| No measurement | Quality regressions are invisible |
| Every question costs an LLM call | Cost and latency grow linearly with traffic |

---

## 2. High-level architecture


<img src="https://raw.githubusercontent.com/subrotokumar/diagrams/576b055fbb49f3c9cd9c1765b5b87222988c0d4b/src/unified_intelligence/diagram.svg" 
classname="full-width-image" 
alt="Unified Intelligence Platform Architecture">


The core principle is **separation of planes**:

| Plane | Responsibility | Scaling profile |
|---|---|---|
| Serving | Low-latency chat and agent execution | Scales with concurrent users |
| Ingestion | Heavy parsing and embedding | Bursty; scales to zero |
| Retrieval | Search, fusion, reranking | Scales with query volume |
| Control | Permissions, evaluation, observability | Always on, lightweight |

A 10,000-page upload must never slow down someone asking a question.

---

## 3. Client and Agent Service

The **React client** talks to a single **Agent Service** (Spring in the reference diagram; FastAPI is an equivalent choice). It owns:

- Session and conversation state in PostgreSQL
- Streaming responses to the client (SSE or WebSocket)
- Entry into the agent graph
- Emitting traces, metrics and logs through OpenTelemetry

File uploads **bypass** the Agent Service: the client requests a **presigned URL** and uploads directly to object storage. This keeps large binaries off the application tier.

> **Improvement: harden the upload path.**
> - Constrain presigned URLs by `Content-Type`, maximum size (`content-length-range` with presigned POST) and short expiry.
> - Upload to a **quarantine** prefix, run malware scanning, then promote to the ingestion prefix.
> - Encrypt at rest (SSE-KMS) and tag each object with tenant and owner so the ACL can be bound at ingestion time.

> **Improvement: persist agent state.**
> LangGraph supports checkpointers. Store graph checkpoints in PostgreSQL so a conversation survives pod restarts, supports resume after human approval, and can be replayed for debugging.

---

## 4. Multi-agent orchestration

A single RAG chain is the wrong abstraction once the platform must do more than answer document questions. The design uses the **supervisor pattern** (LangGraph):

- **Supervisor**: classifies intent and routes.
- **RAG Agent**: knowledge questions over ingested documents.
- **Domain agents**: HR, Jira, Solution Architect, Support, Business Analyst. Each has a narrow prompt and toolset.
- **MCP servers**: expose external systems through the Model Context Protocol, so integrations do not require changes to agent code.

### The RAG agent loop

The RAG agent is a small graph: the **LLM** decides whether to call a **tool**, receives **feedback**, then either calls another tool or answers. This ReAct-style loop lets the model rewrite a failed query, perform multi-hop retrieval, or skip retrieval entirely.

**Design rule:** keep each agent's toolset small (roughly under 10 to 15 tools). Tool-selection accuracy degrades as the tool list grows, and a supervisor with specialised sub-agents scales better than one giant agent.

> **Improvement: treat write actions as high-risk.**
> - Require **human-in-the-loop approval** (LangGraph interrupts) for state-changing tools such as creating Jira tickets or HR requests.
> - Make tool calls **idempotent** (idempotency keys) because retries are inevitable.
> - Propagate the **end user's identity** to MCP servers using on-behalf-of tokens (OAuth token exchange), rather than a shared service account. Otherwise agents become a privilege-escalation path.
> - Apply least-privilege scopes per agent.

> **Improvement: query understanding before retrieval.**
> Add a lightweight router/classifier to decide: no retrieval (chit-chat), document retrieval, structured/SQL lookup, or agent action. Also add **conversation-aware query condensation**: rewrite "what about the second one?" into a standalone query using chat history.

---

## 5. Ingestion pipeline

Ingestion is where most RAG systems quietly lose quality.

### 5.1 Upload and eventing

1. Client obtains a presigned URL and uploads to **object storage** (S3-compatible).
2. The storage layer emits an **upload event** to a **message broker** (SQS, Kafka or RabbitMQ).
3. **KEDA** scales ingestion workers from zero to N based on queue depth, and back to zero when idle.

Properties gained: **back-pressure** (the queue absorbs bursts), **fault isolation** (a bad document is retried without blocking others), **cost efficiency** (no idle workers).

**Technical notes:**

- S3 can publish events natively to SQS or SNS/EventBridge. With Kafka you need a connector or a small forwarder, so SQS is usually simpler on AWS.
- With Kafka, the number of consumers that can do useful work is capped by partition count. Size partitions for your target parallelism, and set the KEDA Kafka scaler's `lagThreshold` accordingly.
- With SQS, set the **visibility timeout** longer than your worst-case parse time, or messages will be delivered twice.
- Scale-to-zero implies **cold starts** (image pull, model load). Pre-pull images, keep model weights on a shared volume or baked image layer, and set a realistic time-to-searchable SLO.

> **Improvement: make ingestion idempotent and recoverable.**
> - Key every document by a **content hash** plus a document ID; skip unchanged files and replace changed ones atomically.
> - Use a **dead-letter queue** with bounded retries and an alert; never let a poison message loop forever.
> - Track a per-document **state machine** (`uploaded → parsed → chunked → embedded → indexed → failed`) in PostgreSQL so users and operators can see status.
> - Support **deletion and re-ingestion**: when a source document is removed, all its chunks and cache entries must go too (a common GDPR / retention requirement).

### 5.2 Layout-aware parsing

Workers run each file through **Docling**, which understands reading order, headings, tables and figures. Output is split into three streams: text, tables, images.

> **Improvement: fall back to OCR and vision.** Scanned PDFs need OCR. For complex pages (slides, infographics), consider a vision-language model or page-image approach in addition to extracted text.

### 5.3 Three embedding paths

**Text path.** Text is split by a **metadata-aware chunk splitter** (document ID, section heading, page, ACL reference, source URL), embedded, and written to the **Text Collection** (pgvector).

Prefer structure-aware splitting (by heading and paragraph) with modest overlap, over fixed character windows.

> **Improvement: better chunking strategies.**
> - **Parent-child (small-to-big) retrieval:** embed small chunks for precision, but return the larger parent section to the LLM for context.
> - **Contextual retrieval:** prepend a short LLM-generated summary of the document and section to each chunk before embedding and BM25 indexing. This has been reported to substantially reduce failed retrievals, at one-off ingestion cost.
> - Store `heading_path` (e.g. `Policy > Leave > Parental`) in metadata and include it in the embedded text.

**Table path.** Tables are kept whole rather than shredded into sentences. Embed a **natural-language description** of the table (title, column names, what it shows), and store the **raw table** (Markdown or HTML) as the payload that is given to the LLM. Embedding the raw grid of numbers directly gives poor semantic vectors.

**Image path.** Images are Base64-encoded, summarised by a vision model into an **image context**, and that text is embedded. The original image stays in object storage so answers can cite or display it.

> **Improvement: consider multimodal embeddings.** Caption-then-embed is robust and keeps one text embedding space, but loses visual detail. Multimodal or late-interaction models (e.g. CLIP-style, ColPali-style page retrieval) can retrieve diagrams and charts directly. A common compromise: caption-based retrieval by default, with a multimodal index for visually heavy corpora.

**Why three collections?** Different modalities need different embedding strategies, top-k values and sometimes models or metrics. Separate collections let you tune, re-index, or swap the store per modality.

> **Improvement: embedding model governance.**
> - Store `embedding_model` and `embedding_version` on every vector. Vectors from different models are **not comparable**.
> - Plan **blue/green re-indexing**: build the new index alongside the old, evaluate, then switch an alias.
> - Tune the vector index deliberately. For pgvector HNSW, `m`, `ef_construction` and `ef_search` trade recall against latency and memory. For large corpora consider `halfvec` or quantisation.
> - Keep the dimension and distance metric (cosine vs inner product) consistent with what the embedding model was trained for.

---

## 6. Retrieval pipeline

Each stage fixes a specific failure mode.

### 6.1 Semantic cache

The query is embedded and compared with previously answered queries (Redis or ElastiCache). On a **hit**, the cached result returns immediately. On a **miss**, the pipeline continues.

**Technical cautions:**

- **Scope cache keys by tenant and permission group.** Otherwise a cached answer can leak across users.
- Use a conservative similarity threshold; near-duplicates with a different meaning ("enable X" vs "disable X") can embed very closely.
- Set TTLs and **invalidate on re-ingestion or ACL change**.
- Cache **retrieval results** (chunk IDs) separately from **final generated answers**. Retrieval caching is safer; answer caching is riskier and should be limited to non-personalised content.

### 6.2 HyDE query rewrite

**Hypothetical Document Embeddings**: the LLM writes a short hypothetical answer, and *that* is embedded instead of the raw question. Answers resemble documents more than questions do, so they land nearer the relevant chunks. It helps most with short or ambiguous queries.

**Trade-offs:** adds an LLM call (latency and cost), and can hallucinate misleading hypotheticals. Use a small, fast model, and consider applying HyDE only when initial retrieval confidence is low, or fusing HyDE results with the original-query results rather than replacing them.

> **Improvement: multi-query expansion.** Generate 2 to 4 query variants and fuse all result lists with RRF. This is often more robust than a single rewrite.

### 6.3 Hybrid search: dense plus sparse

Two searches run in parallel over the indexed chunks:

- **Dense vector search** captures meaning and paraphrase.
- **Sparse BM25 search** (Solr or OpenSearch) captures exact terms, identifiers and rare keywords.

Neither is sufficient alone. Note that this means you maintain **two indexes** and must keep them consistent (same chunk IDs, same ACL metadata, same deletion handling).

> **Improvement: apply metadata filters in both retrievers.** Tenant, ACL group, document type, date and language should be filters inside the search, not post-filters, to avoid empty or biased top-k.

### 6.4 Reciprocal Rank Fusion

Dense and sparse scores are not comparable, so **RRF** fuses by rank:

```
RRF_score(d) = Σ over rankers r of  1 / (k + rank_r(d))     (commonly k = 60)
```

It needs no score normalisation and almost no tuning. Weighted RRF can favour one retriever if evaluation shows it is stronger on your data.

### 6.5 Cross-encoder reranking

The fused candidates (e.g. top 50 to 100) go to a **cross-encoder** that reads the query and each candidate **together**, giving much more accurate relevance than bi-encoder similarity, at higher cost. Because it only sees a small candidate set, cost is manageable. The top 5 to 10 become the LLM context.

**Operational notes:** run the reranker on GPU or an optimised CPU runtime, batch candidates, and enforce a latency budget (for example, rerank only the top N, with a timeout fallback to the RRF order).

> **Improvement: post-retrieval refinement.**
> - **MMR or diversity filtering** to avoid five near-identical chunks.
> - **Context compression / deduplication** to fit the context window and cut tokens.
> - **Lost-in-the-middle mitigation:** place the strongest chunks at the start and end of the prompt.
> - **Score thresholds with an explicit "I don't know" path:** if the best reranker score is below a threshold, answer that the information was not found rather than guessing.

### Stage summary

| Stage | Problem solved | Trade-off |
|---|---|---|
| Semantic cache | Repeats, cost | Staleness; needs scoped keys |
| HyDE / multi-query | Vague short queries | Extra LLM call |
| Dense search | Paraphrase, meaning | Weak on exact terms |
| BM25 | Exact keywords, IDs | Weak on semantics |
| RRF | Merging incomparable scores | Minor rank information loss |
| Cross-encoder | Final-context precision | Added latency |

---

## 7. The LLM gateway

All model calls pass through one **gateway** fronting Azure OpenAI, AWS Bedrock, Ollama (self-hosted) and Hugging Face endpoints. A **router** selects a target by **cost, latency and context window** and performs **retry and fallback**.

Benefits: no vendor lock-in, resilience to provider outages, cost control (cheap models for rewriting and classification, strong models for synthesis), data-residency pinning for sensitive workloads, and one place for logging, PII redaction and token budgets.

**Technical notes:**

- Retries must use **exponential backoff with jitter** and respect provider rate-limit headers; add **circuit breakers** so a failing provider is skipped quickly.
- Fallbacks across providers must account for differences in **tool-calling format, context limit and tokenisation**. Normalise to one internal schema.
- Retry only **idempotent** calls. A retried generation is fine; a retried tool side-effect is not.

> **Improvement: gateway-level controls.**
> - Per-tenant and per-user **token budgets and rate limits**.
> - **Prompt caching** (provider-side) for large static system prompts.
> - **Guardrails**: input and output moderation, PII detection and redaction.
> - Structured output validation (JSON schema) with a repair-or-retry step.

---

## 8. Access control and security

### 8.1 Relationship-based access control (Zanzibar-style)

Enterprise RAG fails its first security review if retrieval ignores permissions. A **Zanzibar-style** engine models relationships such as *user ∈ group*, *group can view folder*, *folder contains document*, and answers "can user U view document D?" through graph traversal. Open-source implementations include **SpiceDB** and **OpenFGA**. Relationships are synchronised **live** from source systems (ServiceNow, Jira, HR platform).

> **Note on the diagram:** the diagram uses a Vault icon and a Zendesk icon purely as stand-ins for the permission engine and the source-of-truth systems. Vault is a secrets manager, not an authorization engine.

**Two enforcement strategies, used together:**

- **Pre-filter:** pass the user's allowed groups or resolved document set into the search as a filter, so forbidden chunks are never retrieved. Better relevance and efficiency.
- **Post-check:** verify every returned chunk against the permission engine. A safety net against stale indexes or filter bugs.

> **Improvement: where to place the post-check.**
> The reference diagram checks permissions after the supervisor produces output. A safer placement is **before the chunks enter the LLM prompt** (after reranking, before generation). If you check only after generation, the model has already seen forbidden content and may have paraphrased it into the answer, and filtering text after the fact is unreliable. Apply the same check to results returned by **MCP tools and domain agents**, not only the RAG path.

> **Improvement: keep permissions fresh.** ACL changes (a user leaves a group) must propagate quickly. Use the permission engine's consistency tokens ("zookies" in Zanzibar terms) or short TTLs, and prefer checking at query time over copying ACLs into chunk metadata only. Copied ACLs go stale; use them for coarse pre-filtering and the engine for the final decision.

### 8.2 LLM-specific threats

> **Improvement: address the OWASP LLM Top 10 explicitly.**
> - **Indirect prompt injection:** retrieved documents are untrusted input. A malicious paragraph can instruct the model to exfiltrate data or call tools. Mitigate with clear delimiting of retrieved content, instruction hierarchy in prompts, tool-call allow-lists, human approval for sensitive actions, and output filtering.
> - **Data exfiltration via rendering:** block auto-loading of model-generated image/markdown URLs.
> - **Sensitive data in logs and traces:** redact prompts and responses in telemetry, or restrict access to them.
> - **Tenant isolation:** enforce at the index, cache and storage layers, and test it.

### 8.3 Citations

> **Improvement: return citations.** Every answer should reference the chunk IDs, document titles and page numbers it used. Citations build trust, enable audit, and make evaluation of faithfulness tractable.

---

## 9. Evaluation

The supervisor's outputs are scored on:

- **Faithfulness**: is the answer supported by the retrieved context (hallucination check)?
- **Answer relevance**: does it address the question?
- **Context precision**: was the retrieved context useful and well ranked?

> **Improvement: add retrieval-only metrics and context recall.** Evaluate the retriever separately from the generator using **Recall@k, MRR and nDCG** on a labelled set. Add **context recall** (did retrieval surface what was needed?), which requires reference answers or labelled relevant chunks. A bad answer can be a retrieval problem or a generation problem, and you need to know which.

Run evaluation in two modes:

1. **Offline, in CI:** a versioned **golden set** of questions with reference answers and relevant documents. Gate merges on regression thresholds when chunking, embeddings, prompts or models change.
2. **Online, on sampled traffic:** detect drift and new failure modes. Capture user feedback (thumbs up/down, corrections) and feed the failures back into the golden set.

**Caveat:** LLM-as-judge metrics (as in RAGAS or DeepEval) are noisy and biased. Use a strong, fixed judge model, calibrate against human labels on a sample, and track variance.

---

## 10. Observability

Every service emits traces, metrics and logs through **OpenTelemetry**. The collector fans out to the **LGTM** stack:

| Signal | Backend |
|---|---|
| Logs | Loki |
| Traces | Tempo |
| Metrics | Prometheus |
| Dashboards and alerting | Grafana |

**RAG-specific signals to instrument:**

- Per-stage latency (cache, HyDE, dense, BM25, RRF, rerank, generation) as **spans** within one trace
- Cache hit rate
- Empty-result rate and score distributions
- Token usage and cost per request, tenant and model
- Ingestion queue depth, failure rate, **time-to-searchable**
- Evaluation scores over time
- Provider error rate and fallback frequency

> **Improvement: use OpenTelemetry GenAI semantic conventions** for LLM spans (model, token counts, finish reason), and consider an LLM-focused trace tool (Langfuse, Arize Phoenix) alongside Grafana for prompt-level inspection. Define **SLOs** (for example p95 time-to-first-token, answer latency, ingestion freshness) and alert on burn rate, not raw thresholds.

---

## 11. Delivery: GitOps, IaC and secrets

The platform runs on Kubernetes (self-managed or EKS) and ships through two GitLab CI pipelines.

**Application pipeline:** lint, test, package, security scan, push the image to the registry (ECR), update the **Helm chart repository**. **ArgoCD** watches that repository and reconciles the cluster, so Git is the source of truth.

**Infrastructure pipeline:** merging a PR in the infra repository triggers plan and apply, with **Terraform** provisioning cloud resources.

**Secrets:** held in a vault (HashiCorp Vault or AWS Secrets Manager) and synced by the **External Secrets Operator (ESO)** into Kubernetes Secrets. **Reloader** watches ConfigMaps and Secrets and rolls the Deployment on change, so rotation needs no manual restart. The **Gateway API** (implemented here by Istio) provides ingress, mTLS and traffic policy.

> **Improvement: delivery hardening.**
> - Require manual approval and plan review before Terraform **apply** in production.
> - Sign images and generate an SBOM; enforce admission policy (cosign verification, Kyverno or OPA Gatekeeper).
> - Use **NetworkPolicies**, workload identity (IRSA on EKS) instead of static cloud keys, and separate namespaces per environment.
> - Add **HPA** for the serving tier, **PodDisruptionBudgets**, and resource requests and limits (GPU workloads especially).
> - Progressive delivery (Argo Rollouts canary) for prompt and model changes, gated by the online evaluation metrics.
> - Version **prompts, retrieval configs and model routing rules** in Git like code.

---

## 12. Key trade-offs

| Decision | Why | Cost |
|---|---|---|
| Separate ingestion and serving planes | Isolation, independent scaling | More moving parts |
| Three collections by modality | Tailored embeddings and tuning | Query fan-out, more indexes |
| Hybrid search + RRF | Semantic and lexical strengths | Two indexes to keep consistent |
| Cross-encoder reranking | Precision | Latency per query |
| Supervisor multi-agent | Scales across domains | Routing errors, harder debugging |
| LLM gateway | Portability, resilience | Another critical service |
| Zanzibar-style permissions | Correct, auditable access | Relationship sync to maintain |
| KEDA scale-to-zero | Low idle cost | Cold-start delay |
| Semantic cache | Lower latency and cost | Staleness and leakage risk |

---

## 13. Common pitfalls

1. **Skipping evaluation.** Tuning by feel causes silent regressions. Build the golden set early.
2. **Shredding tables.** Keep them whole; embed a description, store the raw table.
3. **Unscoped semantic cache.** It can leak data between users.
4. **Too many tools per agent.** Split into sub-agents.
5. **Ignoring ingestion freshness.** Define and monitor time-to-searchable.
6. **Enforcing permissions only in the UI.** Enforce in retrieval and before prompt assembly.
7. **Mixing embedding models in one index.** Version and re-index.
8. **No fallback path.** Provider outages will happen.
9. **Trusting retrieved text.** Treat documents as untrusted input (prompt injection).
10. **Over-engineering on day one.** Add stages only when evaluation shows the need.

---

## 14. Suggested build order

1. Parse, chunk, embed, hybrid retrieve, rerank, answer **with citations**.
2. Golden set and offline evaluation in CI.
3. Permissions (pre-filter and pre-prompt check).
4. Observability and SLOs.
5. LLM gateway with fallback.
6. Semantic cache and query rewriting (measure the gain first).
7. Multi-agent supervisor, MCP tools and human-approval flows.
8. Multi-modal paths (tables, then images).

## 15. Where to go next

- **Graph RAG** for questions spanning entities and relationships
- **Structured/SQL retrieval** for analytical questions
- **Feedback loops** that convert thumbs-up/down into reranker or embedding fine-tuning data
- **Long-context hybrids** that combine retrieval with large-window models
- **Per-tenant indexes** or row-level isolation for strict multi-tenancy

---

## Conclusion

An advanced RAG system is an ingestion pipeline, a multi-stage retrieval engine, an orchestration layer, a security model and an operations practice working together. Each component earns its place by fixing a real failure: layout-aware parsing because PDFs contain tables, hybrid search because users search for identifiers, reranking because vector top-k is noisy, permission checks because compliance demands them, and tracing plus evaluation because quality must be measured to be maintained.

Start with the smallest slice that works and let evaluation tell you what to add next.