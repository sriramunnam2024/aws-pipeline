# Step 2 — Manual upload

Upload tooling is **outside this repo** (git-safe).

## Generate files

In local `mock_landing`:

```bash
python generate_landing.py --cloud aws
```

## Upload

**CLI:** `mock_landing/aws_upload.sh` with env vars from `mock_landing/env.example` (copy to `.env`, gitignored).

**Console:** upload `landing_data/aws/*.csv` and `*.json` to `s3://BUCKET/landing/`.

## Next

`docs/03-unity-catalog-s3.md`
