# AWS IAM Enumeration & Secrets Manager Lab

**Platform:** [Cybr.com](https://cybr.com)  
**Topic:** AWS IAM Enumeration, Privilege Mapping, and Secrets Extraction  
**Tools Used:** AWS CLI, base64  
**Environment:** WSL2 (Ubuntu) on Windows

---

## Overview

This lab covers foundational AWS enumeration techniques using the AWS CLI. The goal is to map out IAM users, groups, roles, and policies — then leverage discovered permissions to extract secrets from AWS Secrets Manager. This mirrors real-world cloud attack paths where an attacker gains initial access via exposed credentials and attempts to enumerate their permissions and escalate.

---

## Lab Objectives

- Configure and verify AWS CLI credentials
- Enumerate IAM users, groups, roles, and attached policies
- Discover and exploit Secrets Manager access
- Decode extracted secret values

---

## Step 1: Verify AWS CLI and Configure Credentials

First, confirm the AWS CLI is installed and configure credentials for the lab environment.

```bash
aws --version
aws configure
```

Verify the configured identity to confirm which account and user you're operating as:

```bash
aws sts get-caller-identity
```

> **Why this matters:** `get-caller-identity` is often the first command an attacker runs after obtaining credentials. It reveals the Account ID, User ID, and ARN — all useful for planning next steps.

You can also configure and use named profiles to simulate operating as different users:

```bash
aws configure --profile xsnaerx
aws sts get-caller-identity --profile xsnaerx
```

---

## Step 2: Enumerate IAM Users

List all IAM users in the account:

```bash
aws iam list-users
```

Check what inline policies are attached directly to a specific user:

```bash
aws iam list-user-policies --user-name Julie --profile xsnaerx
```

Retrieve the full policy document for an inline policy to understand what permissions it grants:

```bash
aws iam get-user-policy --user-name Julie --profile xsnaerx --policy-name AllowReadSecretsManager
```

> **Key finding:** The `AllowReadSecretsManager` policy name signals that this user has Secrets Manager read access — a high-value permission to pursue.

---

## Step 3: Enumerate IAM Groups and Managed Policies

Identify what groups exist and which group(s) a user belongs to:

```bash
aws iam list-groups
aws iam list-groups-for-user --user-name Mary
aws iam list-groups-for-user --user-name Joel
```

Check managed policies attached to users directly:

```bash
aws iam list-attached-user-policies --user-name Mary
aws iam list-attached-user-policies --user-name Joel
```

> **Why this matters:** Users can receive permissions through direct attachments, group membership, or role assumption. Checking all three paths gives a complete picture of what an identity can do.

---

## Step 4: Enumerate IAM Roles and Role Policies

List all roles in the account:

```bash
aws iam list-roles
```

Check what inline policies are attached to a specific role:

```bash
aws iam list-role-policies --role-name SupportRole
```

Retrieve the full policy document for a role's inline policy:

```bash
aws iam get-role-policy --role-name SupportRole --policy-name AllowS3FullAccessForRole
```

> **Why this matters:** Roles with overly broad policies (like full S3 access) are common misconfigurations. Understanding role policies reveals what an attacker could do if they can assume the role.

---

## Step 5: Enumerate AWS Secrets Manager

With the `xsnaerx` profile confirmed to have Secrets Manager read access, list available secrets:

```bash
aws secretsmanager list-secrets --profile xsnaerx
```

Investigate specific secrets — check version history, resource policies, and metadata:

```bash
aws secretsmanager list-secret-version-ids --secret-id sm-enumerate-password --profile xsnaerx
aws secretsmanager describe-secret --secret-id sm-enumerate-password --profile xsnaerx
aws secretsmanager get-resource-policy --secret-id sm-enumerate-password --profile xsnaerx
aws secretsmanager get-resource-policy --secret-id sm-enumerate-api-key --profile xsnaerx
```

---

## Step 6: Extract Secret Values

Retrieve the plaintext values of both secrets:

```bash
aws secretsmanager get-secret-value --secret-id sm-enumerate-password --profile xsnaerx
aws secretsmanager get-secret-value --secret-id sm-enumerate-api-key --profile xsnaerx
```

One of the returned values was base64-encoded. Decode it:

```bash
echo Y3lici1sYWJzLWZha2UtYXBpLWtleS0xMTIy | base64 -d
# Output: cybr-labs-fake-api-key-1122
```

> **Key finding:** The decoded value confirms a (simulated) API key stored in Secrets Manager. In a real engagement, this could be used for further lateral movement or access to third-party services.

---

## Key Takeaways

| Technique | Command Used | Purpose |
|---|---|---|
| Identity verification | `sts get-caller-identity` | Confirm who you are operating as |
| User enumeration | `iam list-users` | Map all IAM users |
| Policy discovery | `iam list-user-policies` / `get-user-policy` | Find what permissions a user has |
| Group mapping | `iam list-groups-for-user` | Check group-based permissions |
| Role enumeration | `iam list-role-policies` / `get-role-policy` | Understand role permissions |
| Secret discovery | `secretsmanager list-secrets` | Identify available secrets |
| Secret extraction | `secretsmanager get-secret-value` | Retrieve plaintext credentials |
| Decoding | `base64 -d` | Decode encoded secret values |

---

## Defensive Notes

- **Least privilege:** Users should only have the specific permissions required for their job function. The `AllowReadSecretsManager` policy granted broad read access to secrets.
- **Secrets rotation:** Secrets Manager secrets should have automatic rotation enabled.
- **CloudTrail monitoring:** All of the enumeration commands above generate CloudTrail log entries. A SIEM or detection rule watching for `secretsmanager:GetSecretValue` from unexpected principals would catch this activity.
- **Resource-based policies:** Secrets Manager resource policies can restrict which principals can access a secret, adding a second layer of control.

---

## References

- [Cybr.com](https://cybr.com) — Lab platform
- [AWS IAM Documentation](https://docs.aws.amazon.com/iam/)
- [AWS Secrets Manager Documentation](https://docs.aws.amazon.com/secretsmanager/)
- [MITRE ATT&CK T1552.004 — Cloud Instance Metadata API](https://attack.mitre.org/techniques/T1552/004/)
- [MITRE ATT&CK T1087.004 — Cloud Account Discovery](https://attack.mitre.org/techniques/T1087/004/)
