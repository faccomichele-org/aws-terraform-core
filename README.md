# aws-terraform-core

This repository defines the shared AWS foundation used by Terraform and GitHub Actions across accounts and environments.

The main design is documented here. Use [USAGE-WITH-TERRAFORM.md](./USAGE-WITH-TERRAFORM.md) for Terraform backend commands and [GITHUB-ROLE-TEMPLATE-USAGE.md](./GITHUB-ROLE-TEMPLATE-USAGE.md) for account-level GitHub Actions deployment steps.

## Architecture

The repository is split into three CloudFormation templates under `cloudformation/`:

1. `backend-setup.yaml`
Creates the central backend resources for one environment in one central AWS account.
It owns:
- the S3 bucket that stores Terraform state
- the IAM role that other accounts assume to read and write that state
- the shared artifacts S3 bucket used by GitHub Actions and peer accounts

2. `github-identity-provider.yaml`
Creates the GitHub Actions OIDC identity provider in a target AWS account.
Deploy it once per AWS account.

3. `github-terraform-policies.yaml`
Creates the common managed policies for one account and one environment.
Prerequisites:
- the manual SSM parameter `/manual/global/central-account/account-id` must already exist in the target account
- the manual SSM parameter `/manual/${Environment}/terraform/state-file/role-secret` must already exist in the target account
- `backend-setup.yaml` must already be deployed in the central account so the referenced state role and shared artifacts bucket exist

These policies let GitHub Actions:
- assume the Terraform state role in the central backend account
- read and write to the shared artifacts bucket in the central backend account

Those manual SSM values are expected to be created outside this repository before the policies stack is deployed.

## Deployment Topology

There are two deployment scopes in this design.

### Central Backend Account

Deploy `backend-setup.yaml` once per environment in a single central account.

That account is the shared home for:
- Terraform remote state
- the cross-account state access role
- shared GitHub Actions artifacts

When you add a new AWS account that must use this backend, update the `AllowedAssumeRoleAccountIDs` parameter on the central backend stack so that account can assume the state role and access the shared artifacts bucket.

### Workload Or Target Accounts

Deploy the remaining templates in every AWS account where GitHub Actions must operate.

Per account:
- `github-identity-provider.yaml`: once
- `github-terraform-policies.yaml`: once per environment

The repository-specific role is deployed from the `aws-iam-roles` repository and should be in place before the workflows that depend on it.

This separation keeps the backend centralized while keeping GitHub trust, managed policies, and repository execution roles local to the account where Terraform will run.

## Deployment Order

Use this order when onboarding a new account or environment:

1. Deploy `backend-setup.yaml` in the central account for the target environment.
2. Record and publish the manual values exposed by the backend stack outputs.
3. Deploy `github-identity-provider.yaml` in the target account.
4. Deploy `github-terraform-policies.yaml` in the target account for the same environment.
5. Deploy the repository-specific GitHub Actions role from the `aws-iam-roles` repository for each repository that needs GitHub Actions access.

## Cross-Account Contract

The central backend stack exposes two manual values through outputs:

1. the central account ID
2. the generated suffix used to reconstruct the backend state role ARN

The account-local policies stack reads those values through SSM dynamic references and uses them to build:
- the ARN of the central Terraform state role
- the name of the shared artifacts bucket in the central account

This means the per-account policies do not hardcode the central account ID or the generated CloudFormation role suffix directly in the template.

## Template Responsibilities

### `backend-setup.yaml`

Purpose:
- host backend state and shared artifacts in one place
- expose cross-account access for approved AWS accounts

Important behavior:
- state bucket is versioned and encrypted
- shared artifacts bucket is versioned, encrypted, and transitions current objects to Glacier Instant Retrieval after 30 days
- shared artifacts are retained longer in production than in non-production environments
- the state access role trust policy is driven by `AllowedAssumeRoleAccountIDs`

### `github-identity-provider.yaml`

Purpose:
- enable GitHub Actions OIDC authentication in an AWS account without long-lived credentials

Important behavior:
- deploy once per account
- reusable by multiple repository roles in the same account

### `github-terraform-policies.yaml`

Purpose:
- provide reusable managed policies for one account and one environment

Policies created:
- `${ProjectName}-tf-access-${Environment}`
- `${ProjectName}-ssm-read-${Environment}`
- `${ProjectName}-s3-artifacts-access-${Environment}`

These are intended to be attached to repository-specific roles instead of duplicating the same permissions in each role definition.

### `github-iam-role.yaml`

This template is no longer maintained in this repository.
It now lives in [aws-iam-roles](https://github.com/faccomichele/aws-iam-roles) under `cloudformation/`, where it should be deployed before the core infrastructure workflows that depend on it.

## Typical Use Case

For a repository such as `aws-iam-roles`:

1. GitHub Actions assumes the repository role created from the `aws-iam-roles` repository's `cloudformation/github-iam-role.yaml` template in the target account.
2. That role can read the SSM parameters that identify the central backend account and role suffix.
3. That role can assume the central Terraform state role.
4. Terraform uses the central S3 backend for state.
5. The same GitHub Actions role can create, update, delete, tag, and inspect IAM roles and customer-managed IAM policies in the target account.

This gives you Terraform-driven IAM management through GitHub Actions without storing static AWS credentials in GitHub.

## Repository Layout

```text
cloudformation/
  backend-setup.yaml
  github-identity-provider.yaml
  github-terraform-policies.yaml
README.md
USAGE-WITH-TERRAFORM.md
GITHUB-ROLE-TEMPLATE-USAGE.md
```

## Operational Guidance

Use the companion docs for procedural details:

- [USAGE-WITH-TERRAFORM.md](./USAGE-WITH-TERRAFORM.md): backend consumption, Terraform initialization, and state access workflow
- [GITHUB-ROLE-TEMPLATE-USAGE.md](./GITHUB-ROLE-TEMPLATE-USAGE.md): account-level GitHub OIDC, common policy, and repository role deployment commands