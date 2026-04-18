# mnemospark-app

Static web shell for **`https://app.mnemospark.ai`**: ls-web session exchange, file list, and multi-download. This repository is **independent** from the marketing site (`mnemospark-website`).

## Layout

- `app/` — static assets (`index.html`, favicon, etc.)
- `infra/cloudformation/app.yaml` — dedicated S3 bucket, CloudFront + OAC, optional GitHub OIDC deploy role
- `.github/workflows/deploy.yml` — deploy stack, sync `app/` to S3, invalidate CloudFront

## GitHub Actions secrets

Configure secrets on the **`prod`** [GitHub Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) for this repository (the workflow uses `environment: prod`):

| Secret | Purpose |
|--------|---------|
| `AWS_ROLE_ARN_APP` | IAM role ARN for OIDC (`sts:AssumeRoleWithWebIdentity`). Trust must allow this repo, `refs/heads/main`, and `environment:prod` in the OIDC subject (see `GitHubEnvironmentName` in `infra/cloudformation/app.yaml`). |
| `ACM_CERTIFICATE_ARN` | Public ACM certificate in **us-east-1** that covers `app.mnemospark.ai`. |

The deploy role trust policy should match the defaults in `infra/cloudformation/app.yaml` (`GitHubOrg` / `GitHubRepo` / `GitHubBranch` / `GitHubEnvironmentName`), or adjust those parameters when deploying the stack manually. If the IAM role was created earlier with a different environment name in the trust policy, update the role trust to include `repo:<org>/<repo>:environment:prod` or redeploy the stack with matching parameters.

## One-time: CloudFormation stack

Use the same AWS account as production. Ensure the GitHub OIDC provider exists in IAM (reuse from other stacks if already configured).

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --stack-name mnemospark-app \
  --template-file infra/cloudformation/app.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ProjectName=mnemospark-app \
    AppDomainName=app.mnemospark.ai \
    AcmCertificateArn=<YOUR_ACM_CERT_ARN> \
    GitHubOrg=pawlsclick \
    GitHubRepo=mnemospark-app \
    GitHubBranch=main \
    GitHubEnvironmentName=prod \
    GitHubOidcProviderArn=<YOUR_GITHUB_OIDC_PROVIDER_ARN>
```

Record outputs:

- `AppDnsTarget` — set DNS (see below)
- `GitHubDeployRoleArn` — use as `AWS_ROLE_ARN_APP` if you create the role via this stack

For day-to-day CI, the workflow passes `CreateGitHubDeployRole=false` and expects the role to already exist (same pattern as `mnemospark-website` prod deploy).

## DNS (Porkbun)

Create a **`app`** **CNAME** pointing to the stack output **`AppDnsTarget`** (CloudFront domain name).

## Migrating from mnemospark-website

If you previously deployed **`mnemospark-website-app`** from the marketing repo, that stack and its GitHub workflow are obsolete. After this repo is live:

1. Point `app` DNS at the **new** distribution (`AppDnsTarget` from stack `mnemospark-app`), or reuse the same ACM alias only if you intentionally update the same distribution (not typical when moving repos).
2. Delete the old CloudFormation stack **`mnemospark-website-app`** when you no longer need it, after DNS cutover.
3. Remove **`AWS_ROLE_ARN_APP`** from the `mnemospark-website` repo if it was only used for the removed workflow.

## Day-to-day

1. Edit files under `app/`
2. Merge to `main`
3. GitHub Actions runs `.github/workflows/deploy.yml`
