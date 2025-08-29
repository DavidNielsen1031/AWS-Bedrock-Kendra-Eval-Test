# cmpliai-rag-weekend — AWS-Native RAG Accuracy & Evaluations

> **Goal:** Stand up a production-oriented **RAG** pipeline on AWS, measure it with **Bedrock Evaluations**, and harden it with **Guardrails**—optimized for **very large corpora** and **regulated** environments.

If you want a starter scaffold with scripts and templates, grab this zip and unzip it locally:
- [cmpliai-rag-weekend.zip](sandbox:/mnt/data/cmpliai-rag-weekend.zip)

---

## What you’ll build
- **Knowledge Base (KB)** in Amazon **Bedrock** with citations.
- Two retriever options you can A/B:
  - **Kendra GenAI Index** (best out-of-box accuracy over messy enterprise docs).
  - **OpenSearch Serverless (Vector Engine)** (low-latency ANN/HNSW + metadata filters).
- **Bedrock Evaluations** (automatic + LLM-as-judge) over a **golden set**.
- **Guardrails** for PII/PHI masking/blocking and harmful content controls.
- Optional **human evaluation** flow for high-risk samples.

---

## Repo structure
```
.
├─ data/
│  ├─ docs/                     # put sample PDFs/text here (sync to S3)
│  └─ goldens/goldens.csv       # golden set template (id, query, reference_context, expected_answer)
├─ aws/
│  ├─ bedrock/
│  │  ├─ eval_job_request.json  # evaluation job skeleton (script patches placeholders)
│  │  └─ guardrails.json        # example guardrails policy (edit before use)
│  └─ opensearch/
│     └─ notes.md               # CLI notes to create collection/index
├─ scripts/
│  ├─ create_guardrails.sh
│  ├─ run_bedrock_eval.sh
│  └─ deploy_opensearch_serverless.sh
├─ src/
│  ├─ query_rewriter.py         # small-model query rewriting
│  └─ rag_client.py             # tiny client for Bedrock KB (retrieve-and-generate)
├─ docs/
│  └─ results.md                # log metrics (baseline vs tuned)
├─ env.example                  # copy to .env and fill in values
└─ requirements.txt
```

---

## Prerequisites
- AWS account with access to **Bedrock**, **Kendra**, **OpenSearch Serverless (Vector)**, **S3**, **CloudWatch**, **CloudTrail**.
- CLI/tools: `awscli` v2, Python 3.10+, `pip`, `jq`, `git`.
- IAM roles/policies with least privilege for the above services.
- One AWS region (e.g., `us-east-1`) for all resources. Configure **KMS/CMKs** per your BAA.

Helpful docs:
- **Knowledge Bases (Bedrock):** https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html
- **Evaluations (Bedrock):** https://docs.aws.amazon.com/bedrock/latest/userguide/model-evaluation.html
- **Guardrails:** https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
- **Kendra GenAI:** https://docs.aws.amazon.com/kendra/latest/dg/what-is-kendra.html
- **OpenSearch Serverless Vector:** https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-vector-search.html

---

## Quick start (5 steps)

1) **Setup & data**
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

cp env.example .env   # fill AWS_REGION, S3_BUCKET, KB_ID, etc.
aws s3 mb s3://$S3_BUCKET
aws s3 sync data/docs s3://$S3_BUCKET/docs
```

2) **Path A (recommended): Kendra GenAI → KB**
- Create **Kendra GenAI Index**; add S3 data source pointing to your bucket.
- Create a **Bedrock Knowledge Base** *from that Kendra index* (no re-ingest).
- Use KB “Test” tab to compare **retrieve** vs **retrieve-and-generate** and note citations.

3) **(Optional) Path B: OpenSearch Serverless (Vector)**
```bash
source .env
./scripts/deploy_opensearch_serverless.sh
# Then follow aws/opensearch/notes.md to add a data access policy and create an index.
```
Attach OpenSearch to a second KB or call it directly from `src/rag_client.py` for A/B.

4) **Guardrails (PII/PHI)**
```bash
source .env
./scripts/create_guardrails.sh
# IMPORTANT: publish a guardrail version after creation
aws bedrock create-guardrail-version --guardrail-identifier <GUARDRAIL_ID> --region "$AWS_REGION"
```
Start with **mask** email/phone; **block** SSN; review categories as needed.

5) **Evaluations (RAG)**
```bash
# Fill your golden set (300–500 rows) then upload:
aws s3 cp data/goldens/goldens.csv s3://$S3_BUCKET/goldens/goldens.csv

# Kick off a KB evaluation (script patches placeholders in the JSON request):
source .env
./scripts/run_bedrock_eval.sh
```
Results land in S3; log baseline in `docs/results.md`.

---

## Architecture
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

---

## Tuning playbook (iterate → evaluate → log)
- **Hybrid retrieval:** combine vector + keyword; apply **metadata filters** (e.g., product, version, effective_date).
- **Chunking:** use heading/section-aware chunks; enable neighbor expansion for cross-section answers.
- **Query rewriting:** run `src/query_rewriter.py` to extract IDs/keywords/synonyms.
- **k & rerank:** sweep `k` and consider lightweight re-ranking when answers are long.
- **Re-evaluate after each change** and record the delta in `docs/results.md`.

---

## Human evaluation (optional, recommended)
Use a private workforce (SageMaker Ground Truth) for a small, high-risk slice (e.g., claims/appeals). Store adjudication notes by example ID and fold them into the golden set and guardrail policy.

---

## CI & nightly (next step)
- **PRs (CodeBuild):** run library-style evals (SageMaker Clarify FMEval) and fail on regression.
- **Nightly (EventBridge):** trigger a Bedrock **evaluation job**, write report to S3, post deltas to Slack/Jira (Lambda webhook).

Flow:
```
PR -> CodeBuild (FMEval) -> Pass/Fail
Nightly -> EventBridge -> Bedrock Eval -> S3 -> Lambda -> Slack/Jira
```

---

## Troubleshooting
- Guardrails “not working”: you must **publish a version** and attach it.
- Eval job errors: check IAM permissions and S3 paths; inspect the patched JSON request.
- Kendra slow under load: offload heavy-QPS routes to OpenSearch for latency/QPS control.
- Keep a 20–50 item **smoke golden set** for fast iteration, plus a 300–500 item set for nightly.

---

## Clean up
Delete test **KBs**, **Kendra index**, **OpenSearch collection**, **guardrail versions**, and **S3** data when done. Verify **CloudWatch**, **CloudTrail**, and IAM artifacts are removed if not needed.

---

## Handy CLI
```bash
# Upload docs & goldens
aws s3 sync data/docs s3://$S3_BUCKET/docs
aws s3 cp data/goldens/goldens.csv s3://$S3_BUCKET/goldens/goldens.csv

# Start an evaluation job (example)
aws bedrock create-evaluation-job --region "$AWS_REGION" --cli-input-json file://<patched>.json

# Create OpenSearch Serverless collection (Vector)
aws opensearchserverless create-collection --name "$OPENSEARCH_COLLECTION_NAME" --type VECTORSEARCH --region "$AWS_REGION"

# Publish a Guardrail version
aws bedrock create-guardrail-version --guardrail-identifier <GUARDRAIL_ID> --region "$AWS_REGION"
```
