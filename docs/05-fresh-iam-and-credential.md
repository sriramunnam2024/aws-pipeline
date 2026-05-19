# Fresh IAM role + Databricks storage credential

Account `ACCOUNT_ID`, bucket `your-bucket-name`, region `YOUR_AWS_REGION`.

Do steps in order. Use **new names** so nothing stale conflicts.

| Item | New name |
|------|----------|
| IAM role | `databricks-uc-lakehouse-dev` |
| Databricks credential | `cred-s3-lakehouse-v2` |

---

## Part 1 — AWS IAM role

### 1.1 Create role (trust policy only)

IAM → **Roles** → **Create role** → **Custom trust policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::414351767826:role/unity-catalog-prod-UCMasterRole-14S5ZJVKOTYTL"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "0000"
        }
      }
    }
  ]
}
```

**Next** → skip policy checkboxes → **Role name:** `databricks-uc-lakehouse-dev` → **Create**.

Copy ARN: `arn:aws:iam::ACCOUNT_ID:role/databricks-uc-lakehouse-dev`

### 1.2 Permissions (inline policy JSON)

Role → **Permissions** → **Create inline policy** → **JSON**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::your-bucket-name"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListBucketMultipartUploads",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::your-bucket-name/*"
    },
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::ACCOUNT_ID:role/databricks-uc-lakehouse-dev"
    }
  ]
}
```

Name: `UC-S3-LakehouseDev` → **Create**.

Stop. Do not validate in Databricks until Part 2 step 2.3.

---

## Part 2 — Databricks credential

### 2.1 Create credential

Catalog → **Credentials** → **Create credential**:

- **Storage credential**
- Type: **AWS IAM Role** (read/write — not read-only)
- Name: `cred-s3-lakehouse-v2`
- IAM role ARN: `arn:aws:iam::ACCOUNT_ID:role/databricks-uc-lakehouse-dev`
- **Do not** limit to read-only

**Create** → copy **External ID** → save it.

### 2.2 Update AWS trust policy

IAM → `databricks-uc-lakehouse-dev` → **Trust relationships** → **Edit**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::414351767826:role/unity-catalog-prod-UCMasterRole-14S5ZJVKOTYTL",
          "arn:aws:iam::ACCOUNT_ID:role/databricks-uc-lakehouse-dev"
        ]
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "PASTE_EXTERNAL_ID_FROM_2.1"
        }
      }
    }
  ]
}
```

**Update policy** → wait 60 seconds.

### 2.3 Validate

Credentials → `cred-s3-lakehouse-v2` → **Test connection** / **Validate**.

Expect: Assume Role, Self Assume Role, External ID — all pass.

### 2.4 External location (SQL)

```sql
CREATE EXTERNAL LOCATION IF NOT EXISTS ext_lakehouse_landing
URL 's3://your-bucket-name/landing/'
WITH (STORAGE CREDENTIAL cred_s3_lakehouse_v2);
```

### 2.5 Test in notebook

```python
dbutils.fs.ls("s3://your-bucket-name/landing/")
```

---

## Optional cleanup

Delete old IAM role `databricks-uc-s3-landing` and old credentials after the new path works.
