# cmpli.ai — AWS-Native RAG & Evals Project Plan (Updated)

**Last updated:** 2025-08-29 20:31 UTC

This project plan gets you from zero to a production-oriented **RAG + Evaluations + Guardrails** setup in AWS, focused on **very large corpora** and **governance**. It pairs well with the AWS Bedrock Workshop (for fundamentals) and then adds the pieces you actually need for cmpli.ai.

> Starter repo scaffold: [cmpliai-rag-weekend.zip](sandbox:/mnt/data/cmpliai-rag-weekend.zip) — unzip and `cp env.example .env` to begin.

---

## 0) Prerequisites & Accounts

- **AWS** with access to: **Bedrock**, **Kendra (GenAI Index)**, **OpenSearch Serverless (Vector Engine)**, **S3**, **CloudWatch**, **CloudTrail**, **IAM**.
- CLI & tools: `awscli` v2, Python 3.10+, `pip`, `jq`, `git`.
- Choose **one region** (e.g., `us-east-1`) and stay in-region for all resources (HIPAA/BAA alignment).
- IAM: create roles/users with least privilege (Bedrock evals, Kendra, AOSS, S3, EventBridge).

Helpful docs:
- Bedrock **Knowledge Bases** → https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html  
- Bedrock **Evaluations** → https://docs.aws.amazon.com/bedrock/latest/userguide/model-evaluation.html  
- **Guardrails** for Bedrock → https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html  
- Kendra (GenAI) → https://docs.aws.amazon.com/kendra/latest/dg/what-is-kendra.html  
- OpenSearch Serverless Vector → https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-vector-search.html

---

## 1) Local Setup & S3

```bash
# Create and activate a virtualenv (optional but recommended)
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Configure AWS CLI
aws configure --profile default

# Create a bucket and upload sample docs
export S3_BUCKET=your-rag-lab-bucket
aws s3 mb s3://$S3_BUCKET
aws s3 sync data/docs s3://$S3_BUCKET/docs
```

Copy the environment template:
```bash
cp env.example .env
# edit .env with AWS_REGION, S3_BUCKET, etc.
```

---

## 2) Path A — **Kendra GenAI ➜ Bedrock Knowledge Base** (recommended start)

**Why:** Best out-of-box relevance for messy enterprise docs; hybrid (semantic+keyword), section/permission awareness.

**Console steps (high-level):**
1. Create a **Kendra GenAI Index**.
2. Add an **S3 data source** pointing to `s3://$S3_BUCKET/docs`.
3. Create a **Bedrock Knowledge Base** **from that Kendra index** (no re-ingest).
4. Use the KB **Test** tab to try **retrieve** vs **retrieve-and-generate** (capture good/bad examples).

**Golden set:** Edit `data/goldens/goldens.csv` with 300–500 questions, contexts, and expected answers. Upload:
```bash
aws s3 cp data/goldens/goldens.csv s3://$S3_BUCKET/goldens/goldens.csv
```

---

## 3) (Optional) Path B — **OpenSearch Serverless (Vector Engine)**

**Why:** Millisecond ANN/HNSW vector search and tight control of schema/filters/latency/QPS.

Create a Vector collection & index:
```bash
source .env
./scripts/deploy_opensearch_serverless.sh
# Then follow aws/opensearch/notes.md to add a data access policy and create an index.
```

Minimal index (HTTP example, adjust endpoint and auth):
```bash
curl -XPUT "$OPENSEARCH_ENDPOINT/my-index" -H 'content-type: application/json' -d '{
  "mappings":{"properties":{
    "text":{"type":"text"},
    "product_id":{"type":"keyword"},
    "vector":{"type":"knn_vector","dimension":1536}
  }},
  "settings":{"index":{"knn":true}}
}'
```

You can attach OpenSearch to a **second** Bedrock KB or call it directly from `src/rag_client.py` for A/B testing.

---

## 4) **Guardrails** (PII/PHI & harmful content)

Create guardrails from template and then **publish a version**:
```bash
source .env
./scripts/create_guardrails.sh
# NOTE: publish a version after creation
aws bedrock create-guardrail-version --guardrail-identifier <GUARDRAIL_ID> --region "$AWS_REGION"
```

Set at least: **mask email/phone**, **block SSN**, review content categories per policy.

---

## 5) **RAG Evaluations** (automatic + LLM-as-judge)

Kick off a **Knowledge Base evaluation** against your golden set (the script patches placeholders):
```bash
source .env
./scripts/run_bedrock_eval.sh
```

What you’ll get:
- **Metrics** (e.g., Precision, Recall, Groundedness) in S3 report outputs.
- Optional **LLM-as-judge** scoring.
- Use the results to compare **Kendra vs OpenSearch**, chunk sizes, `k`, query rewriting, and metadata filters.

**Re-run after each change**; log deltas in `docs/results.md`.

---

## 6) **Tuning for Accuracy**

- **Hybrid retrieval:** Prefer vector + keyword; use metadata filters (e.g., product/version/effective_date).
- **Chunking:** Use section/heading-aware chunks; try neighbor expansion when answers span sections.
- **Query rewriting:** Use `src/query_rewriter.py` (Bedrock small model) to extract keywords/IDs and synonyms.
- **Top-k & rerank:** Sweep `k` and consider lightweight re-ranking for long answers.
- **Quality loop:** Re-evaluate and track wins/losses; keep examples of **fails** for targeted fixes.

---

## 7) **Human Evaluation** (optional but recommended for high-risk)

- Use **SageMaker Ground Truth** (private workforce) for a small % of queries (claims/appeals).
- Store adjudication notes with example IDs so they feed back into your golden set and guardrail policy.

---

## 8) **CI & Nightly Jobs (Next Step)**

- **CI (CodeBuild):** Run library-style evals with **SageMaker Clarify FMEval** on PRs; fail on regression.
- **Nightly (EventBridge):** Trigger a Bedrock **evaluation job** on the golden set and post summary deltas to Slack/Jira (Lambda webhook).

Pseudo-flow:
```
PR -> CodeBuild (FMEval) -> Pass/Fail gate
Nightly -> EventBridge -> Bedrock Eval -> S3 Report -> Lambda -> Slack/Jira
```

---

## 9) **Clean Up**

Tear down to avoid charges:
- Delete **KBs**, **Kendra index**, **OpenSearch collection**, **Guardrail versions**, and the **S3** data if this was a lab.
- Verify **CloudWatch** alarms, **CloudTrail** logs, and IAM roles/policies are removed if unneeded.

---

## Troubleshooting & Tips

- If Kendra results feel slow at scale, push heavy-traffic routes to OpenSearch for QPS/latency control.
- Guardrails not applying? Ensure you **published a version** and attached it to the runtime/eval as needed.
- Evaluation job errors usually indicate **IAM** or **S3 path** issues; double-check `aws/bedrock/eval_job_request.json` after the script patches it.
- Keep a small **“smoke” golden set** (20–50 items) for quick sanity checks, and a larger (300–500) set for nightly runs.

---

## What “Good” Looks Like for cmpli.ai

- **Citations** reliably point to correct pages/sections.
- **RAG eval metrics**: Groundedness ≥ 0.80; Precision/Recall ≥ 0.75; PII = 0 (tune to your risk posture).
- **Ops hooks**: Slack/Jira notifications with before/after metrics and samples.
- **Knowledge** base steadily improves as you iterate chunking, metadata, and retrieval parameters.

---

### Appendix: Handy CLI Snippets

```bash
# Upload docs and goldens
aws s3 sync data/docs s3://$S3_BUCKET/docs
aws s3 cp data/goldens/goldens.csv s3://$S3_BUCKET/goldens/goldens.csv

# Start a Bedrock evaluation job from the patched JSON
aws bedrock create-evaluation-job --region "$AWS_REGION" --cli-input-json file://patched.json

# Create OpenSearch Serverless collection (Vector)
aws opensearchserverless create-collection --name "$OPENSEARCH_COLLECTION_NAME" --type VECTORSEARCH --region "$AWS_REGION"

# Publish a Guardrail version after creation
aws bedrock create-guardrail-version --guardrail-identifier <GUARDRAIL_ID> --region "$AWS_REGION"
```
