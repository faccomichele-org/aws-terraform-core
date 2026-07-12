# Using The Central Backend With Terraform

This document covers the command-level workflow for consuming the centralized Terraform backend created by `cloudformation/backend-setup.yaml`.

For the system design and deployment topology, see [README.md](./README.md).

## Preconditions

Before using this backend from a Terraform project:

1. The central backend account must already have `cloudformation/backend-setup.yaml` deployed for the target environment.
2. The AWS account running Terraform must be included in `AllowedAssumeRoleAccountIDs` on that central backend stack.
3. The AWS principal used by Terraform must have permission to assume the central backend state role.
4. If GitHub Actions is running Terraform, the account-local role should already include the managed policies created by `cloudformation/github-terraform-policies.yaml`.
5. If the workflow depends on the repository-specific GitHub Actions role, that role should already have been deployed from the `aws-iam-roles` repository's `cloudformation/github-iam-role.yaml` template.
6. The manual SSM parameters `/manual/global/central-account/account-id` and `/manual/${Environment}/terraform/state-file/role-secret` must already exist in the target account before the policies stack is deployed.

## Required Backend Values

The backend configuration needs:

- the state bucket name in the central account
- the target state key for the current Terraform project
- the AWS region
- the ARN of the central backend state role

The exact values come from the deployed backend stack and its environment.

## Example Backend Configuration

```hcl
terraform {
  backend "s3" {
    bucket = "aws-terraform-core-state-files-dev-123456789012"
    key    = "aws-iam-roles/terraform.tfstate"
    region = "eu-west-1"

    assume_role = {
      role_arn = "arn:aws:iam::123456789012:role/terraform-core-backend-setup-dev-TfStateRole-ABCDEFG12345"
    }

    encrypt      = true
    use_lockfile = true
  }
}
```

## Terraform Initialization

Initialize Terraform normally once the backend block is in place:

```bash
terraform init
```

If you prefer to keep backend values outside source control, use a backend config file:

```hcl
bucket = "aws-terraform-core-state-files-dev-123456789012"
key    = "aws-iam-roles/terraform.tfstate"
region = "eu-west-1"

assume_role = {
  role_arn = "arn:aws:iam::123456789012:role/terraform-core-backend-setup-dev-TfStateRole-ABCDEFG12345"
}

encrypt      = true
use_lockfile = true
```

Then run:

```bash
terraform init -backend-config=backend-config.tfbackend
```

## GitHub Actions Flow

When Terraform runs from GitHub Actions in a target account, the expected flow is:

1. GitHub Actions assumes the repository role created from the `aws-iam-roles` repository's `cloudformation/github-iam-role.yaml` template.
2. That role has the managed policies from `cloudformation/github-terraform-policies.yaml` attached.
3. Those policies allow the workflow to:
   - read the SSM values that identify the central backend account and role suffix
   - assume the central backend state role
   - access the shared artifacts bucket in the central account
4. Terraform uses the central S3 backend through the assumed backend role.

## Shared Artifacts Bucket

The central backend stack also creates a shared artifacts bucket for cross-account GitHub Actions usage.

Use it for files that should live alongside the centralized backend model, such as:

- generated Terraform plan artifacts
- packaged files reused across account pipelines
- workflow outputs that need to be exchanged between automation steps or accounts

The shared artifacts bucket is separate from the Terraform state bucket and is accessed through the `${ProjectName}-s3-artifacts-access-${Environment}` managed policy.

## Verification Commands

Verify the caller identity used by Terraform or GitHub Actions:

```bash
aws sts get-caller-identity
```

Verify access to the central shared artifacts bucket:

```bash
aws s3 ls s3://aws-terraform-core-shared-artifacts-dev-123456789012/
```

Verify the state location after `terraform apply`:

```bash
aws s3 ls s3://aws-terraform-core-state-files-dev-123456789012/aws-iam-roles/
```

## Key Rules

1. Use a unique `key` per Terraform project.
2. Keep backend state in the central account only.
3. Do not create per-repository backend buckets when using this model.
4. Update the central backend stack whenever a new AWS account must participate.
5. Treat the account-local GitHub policies and roles as consumers of the central backend, not replacements for it.