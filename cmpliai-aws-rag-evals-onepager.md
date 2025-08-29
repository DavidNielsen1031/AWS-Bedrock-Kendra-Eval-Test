# cmpli.ai — AWS‑Native AI Evals & RAG Tooling (One‑Pager)

**Goal:** Improve RAG search accuracy on a very large corpus and govern model quality/safety — fully in AWS (HIPAA‑eligible, region‑pinned, KMS‑encrypted).

## Stack (what each piece does)
- **Amazon Bedrock Knowledge Bases (KB)** — RAG over your docs; supports *retrieve* and *retrieve‑and‑generate* with citations; pluggable backends (Kendra GenAI, OpenSearch Serverless, S3 Vectors).
- **Retriever (pick 1 primary; keep 1 backup)**
  - **Kendra GenAI Index** *(primary for accuracy/enterprise sources)*: Hybrid semantic+keyword, section/heading‑aware, permission‑aware; strong out‑of‑box relevance.
  - **OpenSearch Serverless — Vector Engine** *(primary when latency/QPS control matters)*: Millisecond ANN/HNSW vector search with metadata filters; you own schema, scaling, and cost.
  - *(Optional)* **S3 Vectors via KB** for ultra‑large/long‑tail content at low storage/query cost (accept lower QPS).
- **Amazon Bedrock Evaluations** — Batch & model‑as‑judge evals for RAG (precision/recall, groundedness), side‑by‑side comparisons; results to S3; callable from CI (CodeBuild/CodePipeline).
- **Guardrails for Bedrock** — PII/PHI detection, masking/blocking, harmful‑content controls; metrics/alarms via CloudWatch.
- **SageMaker Clarify FMEval (+ optional Ground Truth)** — Library‑style evals in CI; human review for high‑risk items (claims/appeals).
- **Platform & Ops** — **S3** (docs/embeddings/eval data), **KMS** (CMK), **CloudWatch/CloudTrail**, **IAM**, **EventBridge + Lambda** (post‑eval → Slack/Jira).

## Flow (high level)
User/Job → **KB** (Retriever: Kendra or OpenSearch) → **FM (Generate)** → **Guardrails** → **Answer + Citations**  
Nightly/PR: **Golden set in S3** → **Bedrock Evaluations / FMEval** → **Results to S3** → **EventBridge** → **Slack/Jira**

```mermaid
flowchart LR
  subgraph Client["Prompt or API"]
  Q[User Query]
  end
  Q --> RW[Query Rewriter (Bedrock small model)]
  RW --> BR[Bedrock Knowledge Bases]

  subgraph Retrieval["Retrieval Store"]
    Kendra[Kendra GenAI Index]:::opt
    OSS[OpenSearch Serverless Vector]:::opt
  end
  BR <---> Kendra
  BR <---> OSS

  BR --> FM[Bedrock FM (RAG generate)]
  FM --> GR[Guardrails]
  GR --> Ans[Answer + Citations]

  Q -->|Goldens| Eval[Bedrock Evaluations]
  FM --> Eval
  Kendra --> Eval
  OSS --> Eval

  classDef opt stroke-dasharray: 4 2;
```

## Why this stack
- **Accuracy:** Kendra hybrid + section‑aware chunking; or OpenSearch with tuned ANN + metadata filters.
- **Governance:** First‑party evals, encryption/KMS, region pinning, CloudTrail; fewer vendors to assess.
- **Speed to value:** Console to start; CLI/SDK for CI; easy A/B retriever comparison on same KB.

## Assumptions (to validate)
- All PHI stays in one AWS region under our **BAA** using **CMKs**.
- Corpus primarily in **S3** (connectors not critical for pilot).
- RAG answers must include **traceable citations** to source pages/sections.
- **Kendra** latency/QPS acceptable for pilot; otherwise heavy‑traffic routes pivot to **OpenSearch**.
- SMEs available for **weekly human‑in‑the‑loop** scoring on a small high‑risk slice.

## Open questions / decisions
1. **Retriever choice:** Start with Kendra or OpenSearch for latency/cost control? (Pilot both on the same golden set.)
2. **Regions & residency:** Single region vs multi‑region? Any EU requirements?
3. **Guardrails policy:** Exact PII mask/block rules (e.g., SSN block; email/phone mask).
4. **Golden set:** Size (300–500), owner, refresh cadence.
5. **CI gating thresholds:** e.g., Groundedness ≥ 0.80; Precision/Recall ≥ 0.75; PII = 0.
6. **Connector needs:** SharePoint/Confluence/DB connectors (pushes us toward Kendra) vs S3‑only.
7. **Cost envelope:** Monthly caps for indexing, eval runs, and retrieval QPS per environment.

## Immediate next steps
- **Week 1:** KB on **Kendra GenAI**, ingest 5–10 GB slice; 300–500‑row golden set; baseline **Bedrock RAG eval**.
- **Week 2:** Spin up **OpenSearch Serverless** index; add metadata filters + query rewriting; re‑run eval; compare.
- **Week 3:** Finalize **Guardrails** policy; add **CI gate** (CodeBuild + FMEval) and nightly eval with Slack/Jira notifications.
