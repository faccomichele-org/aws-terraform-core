# GitHub Actions Account Setup And Role Deployment

This document covers the command-level setup for GitHub Actions access in each AWS account.

For the overall design and template responsibilities, see [README.md](./README.md).

## What Gets Deployed Per Account

Each target AWS account uses three layers from this repository, plus one repository-specific role template from the companion `aws-iam-roles` repository:

1. `cloudformation/github-identity-provider.yaml`
Deployed once per account to create the GitHub OIDC provider.

2. `cloudformation/github-terraform-policies.yaml`
Deployed once per account and per environment to create the shared managed policies used by repository roles.

3. The repository-specific role template from `https://github.com/faccomichele/aws-iam-roles`
Deployed once per repository and per environment from that repository's `cloudformation/github-iam-role.yaml` file.

## Deployment Order

For a new account, use this order:

1. Deploy the GitHub OIDC provider.
2. Deploy the common Terraform and artifacts policies for each environment.
3. Deploy the repository-specific GitHub Actions role from the `aws-iam-roles` repository for each repository.

## Step 1: Deploy The OIDC Provider

```bash
aws cloudformation deploy \
  --stack-name terraform-core-github-identity-provider \
  --template-file cloudformation/github-identity-provider.yaml \
  --parameter-overrides \
    ProjectName=aws-terraform-core \
    Organization=faccomichele \
  --capabilities CAPABILITY_NAMED_IAM
```

Deploy this only once in each AWS account.

## Step 2: Deploy The Common Managed Policies

Run once per environment in the same AWS account:

Before deploying, make sure these manual SSM parameters already exist in the target account, because the policies stack resolves them at deploy/runtime:

- `/manual/global/central-account/account-id`
- `/manual/${Environment}/terraform/state-file/role-secret`

```bash
aws cloudformation deploy \
  --stack-name terraform-core-github-terraform-policies-dev \
  --template-file cloudformation/github-terraform-policies.yaml \
  --parameter-overrides \
    ProjectName=aws-terraform-core \
    Organization=faccomichele \
    Environment=dev \
  --capabilities CAPABILITY_NAMED_IAM
```

This stack creates:

- `aws-terraform-core-tf-access-dev`
- `aws-terraform-core-ssm-read-dev`
- `aws-terraform-core-s3-artifacts-access-dev`

Repeat for `stg` and `prod` as needed.

## Step 3: Deploy The Repository Role

Deploy one role per repository and per environment from a checkout of the `aws-iam-roles` repository. The template lives at `cloudformation/github-iam-role.yaml` in that repository:

```bash
aws cloudformation deploy \
  --stack-name terraform-core-github-iam-role-dev-aws-iam-roles \
  --template-file cloudformation/github-iam-role.yaml \
  --parameter-overrides \
    ProjectName=aws-iam-roles \
    Organization=faccomichele \
    Environment=dev \
  --capabilities CAPABILITY_NAMED_IAM
```

## What The Repository Role Provides

The role created by `aws-iam-roles/cloudformation/github-iam-role.yaml`:

- trusts GitHub Actions via the local OIDC provider
- attaches the common managed policies created by `github-terraform-policies.yaml`
- includes inline IAM permissions for role and customer-managed policy administration in the local account

With the current template, the role can:

- list and inspect IAM roles and policies with IAM APIs that require `Resource: *`
- create, update, delete, tag, and untag IAM roles in the local account
- attach and detach managed policies to roles
- manage inline role policies
- create, version, delete, and tag customer-managed IAM policies in the local account

This is intended for repositories such as `aws-iam-roles` that use Terraform from GitHub Actions to manage IAM as code.

## Example GitHub Actions Workflow

```yaml
name: Terraform

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/terraform-core-github-iam-role-dev-aws-iam-roles-GHARole-ABCDE12345
          aws-region: eu-west-1

      - name: Verify identity
        run: aws sts get-caller-identity

      - name: Terraform init
        run: terraform init

      - name: Terraform apply
        run: terraform apply -auto-approve
```

## Verification Commands

Check the OIDC provider exists in the account:

```bash
aws iam list-open-id-connect-providers
```

Check the common policies exist:

```bash
aws iam list-policies --scope Local --query "Policies[?contains(PolicyName, 'aws-terraform-core')].[PolicyName,Arn]"
```

Check the repository role trust and attached policies:

```bash
aws iam get-role --role-name <role-name>
aws iam list-attached-role-policies --role-name <role-name>
```

## Common Operational Notes

1. Keep the backend centralized in the dedicated backend account.
2. Deploy the OIDC provider separately in every account where GitHub Actions must run.
3. Deploy the common policies separately for each environment in each account.
4. Deploy the repository role separately for each repository and environment combination.
5. If a new AWS account must use the centralized backend, remember to update the central `backend-setup.yaml` stack so that account is added to `AllowedAssumeRoleAccountIDs`.