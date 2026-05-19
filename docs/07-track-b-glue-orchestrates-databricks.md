# Track B — Glue Workflow orchestrates Databricks only

**No Glue ETL.** No Glue Data Catalog. Glue is only the **visual orchestrator** that calls the same entry point as Track A.

```text
Glue Workflow (schedule or manual trigger)
  → Action: Lambda start_databricks_medallion   (same as Track A)
       → Databricks Job pl_aws_medallion (bronze → silver → gold)
```

Same job, same incremental behavior. Two ways to start it:

| Trigger | Entry |
|---------|--------|
| Track A | S3 → EventBridge → Lambda |
| Track B | Glue Workflow → Lambda |

---

## Prerequisites

Complete **Track A** first:

- Lambda `start_databricks_medallion` deployed and tested  
- Databricks job `pl_aws_medallion` with max concurrent runs = 1  

---

## Glue Workflow (console)

1. **AWS Glue** → **Orchestration** → **Workflows** → **Create workflow**  
2. **Name:** `wf_trigger_databricks_medallion`  

### Start trigger

- **Type:** On demand (for learning) or **Schedule** (e.g. cron `cron(0 6 * * ? *)`)  
- Do **not** add Glue job triggers for Spark ETL  

### Action 1 — Lambda

- **Add trigger** → **Trigger type:** **Lambda** (or **Custom** depending on console version)  
- If only "Glue jobs" appear: add a **Python shell** Glue job that only calls Lambda invoke API — prefer **native Lambda step** when available  

**Alternative (always works):** Glue **Trigger** → **Lambda function** target:

- Function: `start_databricks_medallion`  
- Payload: `{}`  

### No second Glue job

Do not add crawlers, Spark jobs, or catalog steps.

---

## IAM for Glue → Lambda

Glue service role needs:

```json
{
  "Effect": "Allow",
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:YOUR_AWS_REGION:ACCOUNT_ID:function:start_databricks_medallion"
}
```

---

## Idempotency (unchanged)

- Lambda **SKIP_IF_RUNNING** still applies.  
- Databricks job still **max concurrent runs = 1**.  
- Running workflow while EventBridge also fires → guarded by Lambda + job concurrency.

---

## Interview wording

*"AWS Glue Workflow schedules/triggers a serverless controller that starts a Databricks medallion job; all transformation and schema evolution live in Unity Catalog and Delta, not in Glue ETL."*

---

## Track A vs B summary

| | Track A | Track B |
|--|---------|---------|
| **When** | File lands in S3 | Schedule or manual |
| **AWS** | EventBridge | Glue Workflow |
| **Compute** | Lambda only | Lambda only (via Glue) |
| **Databricks** | Same `pl_aws_medallion` | Same job |