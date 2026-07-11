# aws-terraform-core

This repository defines the shared AWS foundation used by Terraform and GitHub Actions across accounts and environments.

The main design is documented here. Use [USAGE-WITH-TERRAFORM.md](./USAGE-WITH-TERRAFORM.md) for Terraform backend commands and [GITHUB-ROLE-TEMPLATE-USAGE.md](./GITHUB-ROLE-TEMPLATE-USAGE.md) for account-level GitHub Actions deployment steps.

## Architecture

The repository is split into four CloudFormation templates under `cloudformation/`:

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
These policies let GitHub Actions:
- read the manual SSM parameters that point to the central backend account and role secret
- assume the Terraform state role in the central backend account
- read and write to the shared artifacts bucket in the central backend account

4. `github-iam-role.yaml`
Creates the repository-specific GitHub Actions IAM role for one account and one environment.
This role trusts GitHub OIDC, attaches the common managed policies from `github-terraform-policies.yaml`, and adds repository-specific permissions on top. In the current template, the inline policy is set up so the role can manage IAM roles and customer-managed IAM policies in the local account. That makes it suitable for the `aws-iam-roles` repository to deploy IAM resources through GitHub Actions with Terraform.

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
- `github-iam-role.yaml`: once per repository and per environment

This separation keeps the backend centralized while keeping GitHub trust, managed policies, and repository execution roles local to the account where Terraform will run.

## Deployment Order

Use this order when onboarding a new account or environment:

1. Deploy `backend-setup.yaml` in the central account for the target environment.
2. Record and publish the manual values exposed by the backend stack outputs.
3. Deploy `github-identity-provider.yaml` in the target account.
4. Deploy `github-terraform-policies.yaml` in the target account for the same environment.
5. Deploy `github-iam-role.yaml` in the target account for each repository that needs GitHub Actions access.

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

Purpose:
- create the GitHub Actions execution role for a specific repository in a specific account and environment

Important behavior:
- trust is scoped to `repo:${Organization}/${ProjectName}:*`
- common backend-related permissions come from managed policies
- repository-specific permissions are defined inline in this template
- the current inline policy allows managing IAM roles and customer-managed IAM policies in the local account

## Typical Use Case

For a repository such as `aws-iam-roles`:

1. GitHub Actions assumes the repository role created by `github-iam-role.yaml` in the target account.
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
  github-iam-role.yaml
README.md
USAGE-WITH-TERRAFORM.md
GITHUB-ROLE-TEMPLATE-USAGE.md
```

## Operational Guidance

Use the companion docs for procedural details:

- [USAGE-WITH-TERRAFORM.md](./USAGE-WITH-TERRAFORM.md): backend consumption, Terraform initialization, and state access workflow
- [GITHUB-ROLE-TEMPLATE-USAGE.md](./GITHUB-ROLE-TEMPLATE-USAGE.md): account-level GitHub OIDC, common policy, and repository role deployment commands