# Step 1 — S3 landing bucket

Create a private bucket in your region (Console or Terraform).

## Layout

```text
s3://<your-bucket>/landing/<timestamp>.csv
s3://<your-bucket>/landing/<timestamp>.json
```

## IAM

Use an IAM user (not root) with `ListBucket` and `PutObject`/`GetObject` on `landing/*`.

Store bucket name and region in local `.env` from `.env.example` (gitignored).

## Next

`docs/02-manual-upload.md`
