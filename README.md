# Terraform `state mv`: Moving an IAM Policy Between States

This project demonstrates how to move an existing AWS IAM policy (`aws_iam_policy.custom_policy`) from one Terraform state file to another using `terraform state mv`, without deleting or recreating the resource in AWS.

## Repository Structure

```
.
├── tf_code_1/        # Source configuration (state: tf_code_1.tfstate)
├── tf_code_2/        # Destination configuration (state: tf_code_2.tfstate)
├── .gitignore
└── README.md
```

## Overview

| Item | Value |
|------|-------|
| Resource moved | `aws_iam_policy.custom_policy` |
| Source state | `tf_code_1.tfstate` |
| Destination state | `tf_code_2.tfstate` |
| State bucket | `cmtr-n9nbj5cz-backend-bucket-1792054263` |
| Region | `eu-west-1` |
| Terraform version | `>= 1.5.7` |

## Prerequisites

- Terraform `>= 1.5.7`
- AWS credentials with access to the state bucket and IAM
- Git

## Steps Performed

### 1. Initialize both configurations

```bash
cd tf_code_1
terraform init \
  -backend-config='bucket=cmtr-n9nbj5cz-backend-bucket-1792054263' \
  -backend-config='key=tf_code_1.tfstate' \
  -backend-config='region=eu-west-1'

cd ../tf_code_2
terraform init \
  -backend-config='bucket=cmtr-n9nbj5cz-backend-bucket-1792054263' \
  -backend-config='key=tf_code_2.tfstate' \
  -backend-config='region=eu-west-1'
```

### 2. Pull both remote states locally

```bash
cd ../tf_code_1 && terraform state pull > ../src.tfstate
cd ../tf_code_2 && terraform state pull > ../dst.tfstate
```

### 3. Move the resource

```bash
cd ../tf_code_1
terraform state mv \
  -state=../src.tfstate \
  -state-out=../dst.tfstate \
  aws_iam_policy.custom_policy aws_iam_policy.custom_policy
```

### 4. Push the updated states back to the remote backend

The destination is pushed first so the resource is never untracked.

```bash
cd ../tf_code_2 && terraform state push ../dst.tfstate
cd ../tf_code_1 && terraform state push ../src.tfstate
```

### 5. Update the configuration

- Removed the `aws_iam_policy.custom_policy` resource block from `tf_code_1`.
- Added the identical resource block to `tf_code_2`.
- Each resource is now declared only in the configuration that manages it.

### 6. Validate

Run in both `tf_code_1` and `tf_code_2`:

```bash
terraform fmt
terraform validate
terraform plan
```

Expected result: `No changes. Your infrastructure matches the configuration.`

## Verification

```bash
# Should list the policy
cd tf_code_2 && terraform state list

# Should NOT list the policy
cd ../tf_code_1 && terraform state list
```

Also confirm in the AWS Console (IAM > Policies) that `custom_policy` still exists and was not recreated.

## Notes

- `terraform apply` was never run before both plans came back clean.
- Local state copies (`src.tfstate`, `dst.tfstate`) are excluded via `.gitignore` and are not committed.
- `.terraform.lock.hcl` is committed to pin provider versions.

## Troubleshooting

| Problem | Likely cause |
|---------|--------------|
| Plan shows changes in `tf_code_2` | Resource block differs from the real policy; copy it exactly from the old config |
| Plan wants to destroy in `tf_code_1` | Resource block still present in the config after the move |
| `state push` serial/lineage error | Re-pull the state and redo the move; use `-force` only as a last resort |