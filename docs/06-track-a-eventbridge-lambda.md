# Track A — EventBridge + Lambda → Databricks job

Orchestration stays in **AWS**. Medallion logic stays in **Databricks** (incremental: Auto Loader checkpoints + MERGE).

```text
S3 landing/*.json (or manual job)
  → EventBridge
  → Lambda start_databricks_medallion
  → Databricks Job (bronze → silver → gold)
```

No Glue compute. No duplicate ETL.

---

## 0. Idempotency rules (read first)

| Layer | How re-runs stay safe |
|--------|----------------------|
| **Bronze** | Auto Loader + `checkpointLocation` on UC volume — already ingested files skipped |
| **Silver / gold** | `MERGE` on `order_id`, `customer_id`, `line_id`; country table overwrite by design |
| **Job** | Databricks job **Max concurrent runs = 1** |
| **Triggers** | Prefer **one event per upload batch** (see EventBridge note below) |

**EventBridge gotcha:** `aws_upload.sh` uploads `.csv` and `.json` → **two S3 events** → two Lambda invocations.

Mitigations (pick one):

1. **EventBridge filter:** suffix `.json` only (one trigger per `generate_landing` run).
2. **Lambda guard:** if job already `RUNNING`, exit 0 (no second start).
3. Both (recommended).

---

## 1. Databricks — job workflow (UI)

**Workflows** → **Create job** → name: `pl_aws_medallion`

| Setting | Value |
|---------|--------|
| **Max concurrent runs** | `1` |
| **Timeout** | e.g. 60 min total |

### Task 1 — bronze

- **Type:** Notebook  
- **Path:** `/Users/you@example.com/aws_practice/aws_bronze_load` (your workspace path)  
- **Cluster / compute:** your working SQL warehouse or job compute  
- **Parameters:** defaults in notebook widgets (`aws_lakehouse`, `your-bucket-name`, …)

### Task 2 — silver

- **Depends on:** bronze  
- **Notebook:** `aws_silver_load`

### Task 3 — gold

- **Depends on:** silver  
- **Notebook:** `aws_gold_load`

**Save** → copy **Job ID** (number in URL or job settings).

### Auth for API (no secrets in git)

**Account console** → **User management** → **Service principals** → create `sp-aws-orchestration`

- Generate **OAuth secret** (or PAT if your tier allows)  
- Grant SP on workspace + `USE CATALOG` on `aws_lakehouse`  
- Store in **AWS Secrets Manager** as JSON:

```json
{
  "databricks_host": "https://dbc-xxxx.cloud.databricks.com",
  "databricks_token": "YOUR_PAT"
}
```

Secret name example: `databricks/pl_aws_medallion`

---

## 2. AWS — Lambda (console)

### 2.1 Secrets Manager

Create secret `databricks/pl_aws_medallion` (key/value or JSON above).

### 2.2 IAM role for Lambda

Trust: `lambda.amazonaws.com`

Permissions:

- `secretsmanager:GetSecretValue` on that secret  
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`  
- (Optional) `dynamodb:*` if you add dedupe table later  

### 2.3 Function

- **Name:** `start_databricks_medallion`  
- **Runtime:** Python 3.12  
- **Env vars:**

| Key | Value |
|-----|--------|
| `DATABRICKS_SECRET_ARN` | ARN of secret |
| `DATABRICKS_JOB_ID` | Job ID from step 1 |
| `SKIP_IF_RUNNING` | `true` |

- **Code:** copy from `infra/lambda/start_databricks_medallion/lambda_function.py` in this repo (deploy via zip or inline editor).

### 2.4 Test Lambda

Test event `{}` → expect `run_id` in response body.

---

## 3. AWS — EventBridge (console)

### 3.1 Rule

- **Name:** `s3-landing-new-file`  
- **Event bus:** default  
- **Rule type:** Rule with an event pattern  

**Pattern (JSON):**

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": ["your-bucket-name"] },
    "object": {
      "key": [{ "suffix": ".json" }]
    }
  }
}
```

Using **`.json` only** avoids double fire from `.csv` + `.json` in one upload.

### 3.2 Target

- **Target:** Lambda `start_databricks_medallion`  
- **Retry:** default  

### 3.3 S3 → EventBridge

**S3** → bucket → **Properties** → **Event notifications** → create notification for `landing/` prefix → send to **EventBridge** (enable EventBridge on bucket if prompted).

---

## 4. End-to-end test

1. `python generate_landing.py --cloud aws` + `aws_upload.sh`  
2. Confirm EventBridge invocations (CloudWatch → Lambda logs)  
3. Databricks **Runs** tab → one `pl_aws_medallion` success  
4. Row counts stable on re-run (bronze no duplicate raw rows for same files; silver/gold MERGE)

---

## 5. Manual run (same data path)

Databricks → Jobs → `pl_aws_medallion` → **Run now**  

Same job definition as Lambda — no extra logic branch.

---

## Cost

EventBridge + Lambda: ~$0 at demo volume. You pay **Databricks** when the job runs.