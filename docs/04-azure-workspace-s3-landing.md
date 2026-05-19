# Azure Databricks workspace -> AWS S3 landing

Use Azure workspace catalog `dbw_lakehouse_dev` to read S3 via Unity Catalog.

## 1. AWS IAM role

Create role `databricks-uc-s3-landing` with S3 access to `YOUR_BUCKET/landing/*`.

Trust policy: copy from Databricks Account Console when adding AWS storage credential.

## 2. Account console

accounts.azuredatabricks.net -> Catalog -> Storage credentials -> Add -> AWS IAM role -> paste Role ARN.

## 3. SQL (workspace)

```sql
USE CATALOG dbw_lakehouse_dev;

CREATE STORAGE CREDENTIAL IF NOT EXISTS cred_aws_s3_landing
WITH (AWS_IAM_ROLE = 'arn:aws:iam::ACCOUNT:role/databricks-uc-s3-landing');

CREATE EXTERNAL LOCATION IF NOT EXISTS ext_aws_s3_landing
URL 's3://YOUR_BUCKET/landing/'
WITH (STORAGE CREDENTIAL cred_aws_s3_landing);

CREATE SCHEMA IF NOT EXISTS aws_practice_bronze;
CREATE SCHEMA IF NOT EXISTS aws_practice_silver;
CREATE SCHEMA IF NOT EXISTS aws_practice_gold;
CREATE VOLUME IF NOT EXISTS aws_practice_bronze.autoloader_meta;
```

Grant your user USE CREDENTIAL, READ/WRITE FILES on external location, schema/volume on bronze.

## 4. Notebook

Import `src/notebooks/aws_practice/00_s3_connection_test.ipynb` to workspace folder `aws_practice`.

Set widget `s3_bucket` to your bucket (no secrets in repo).