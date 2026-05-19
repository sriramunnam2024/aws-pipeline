# AWS CLI setup

Install [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) (MSI or `winget install Amazon.AWSCLI`).

```powershell
aws --version
aws configure
aws sts get-caller-identity
```

Store **access keys only** in `%UserProfile%\.aws\credentials` — not in this repo.

Set bucket/region in local `.env` from `.env.example`, or export before running upload scripts in **mock_landing**.
