# Step 3 — Unity Catalog on S3

Configure in Databricks SQL (replace placeholders in the UI):

- Storage credential (IAM role for UC)
- External location: `s3://YOUR_BUCKET/landing/`
- Schema `cloud_practice_bronze` + volume `autoloader_meta`

## Notebooks

1. `bronze_load.ipynb` — set `s3_bucket` and `landing_prefix` widgets
2. `silver_load.ipynb`
3. `gold_load.ipynb`
