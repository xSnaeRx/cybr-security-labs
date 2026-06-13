# AWS IAM Enumeration & Secrets Manager Lab

**Platform:** [Cybr.com](https://cybr.com)  
**Topic:** AWS IAM Enumeration, Privilege Mapping, and Secrets Extraction  
**Tools Used:** AWS CLI, base64  
**Environment:** WSL2 (Ubuntu) on Windows

---

## Overview

This lab covers foundational AWS enumeration techniques using the AWS CLI. The objective is to map out IAM users, groups, roles, and policies — then leverage discovered permissions to extract secrets from AWS Secrets Manager. This mirrors real-world cloud attack paths where an attacker gains initial access via exposed credentials and works to enumerate what they can reach.

---

## Lab Objectives

- Configure and verify AWS CLI credentials
- Enumerate IAM users, groups, roles, and attached policies
- Discover and exploit Secrets Manager access
- Decode extracted secret values

---

## Step 1: Verify AWS CLI and Configure Credentials

Confirm the AWS CLI is installed and configure credentials for the lab environment.

```bash
aws --version
aws configure
```

Verify the configured identity to confirm which account and user you are operating as:

```bash
aws sts get-caller-identity
```

> **Why this matters:** `get-caller-identity` is typically the first command run after obtaining credentials. It reveals the Account ID, User ID, and ARN — all useful context for planning next steps.

---

## Step 2: Enumerate IAM Users

List all IAM users in the account:

```bash
aws iam list-users
```

Check what inline policies are attached directly to a specific user:

```bash
aws iam list-user-policies --user-name Julie
```

Retrieve the full policy document to understand what permissions it grants:

```bash
aws iam get-user-policy --user-name Julie --policy-name AllowReadSecretsManager
```

> **Key finding:** The `AllowReadSecretsManager` policy name immediately signals that this user has Secrets Manager read access — a high-value permission worth pursuing.

---

## Step 3: Enumerate IAM Groups and Managed Policies

Identify what groups exist and which group(s) a user belongs to:

```bash
aws iam list-groups
aws iam list-groups-for-user --user-name Mary
aws iam list-groups-for-user --user-name Joel
```

Check managed policies attached directly to users:

```bash
aws iam list-attached-user-policies --user-name Mary
aws iam list-attached-user-policies --user-name Joel
```

> **Why this matters:** Permissions can flow to a user through direct policy attachments, group membership, or role assumption. Checking all three paths gives a complete picture of what an identity can actually do.

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

Retrieve the full policy document for the role's inline policy:

```bash
aws iam get-role-policy --role-name SupportRole --policy-name AllowS3FullAccessForRole
```

> **Why this matters:** Roles with overly broad policies are a common misconfiguration. Understanding role permissions reveals what an attacker could do if they can assume that role.

---

## Step 5: Enumerate AWS Secrets Manager

With Secrets Manager read access confirmed, list all available secrets:

```bash
aws secretsmanager list-secrets
```

Investigate specific secrets — check version history, resource policies, and metadata:

```bash
aws secretsmanager list-secret-version-ids --secret-id sm-enumerate-password
aws secretsmanager describe-secret --secret-id sm-enumerate-password
aws secretsmanager get-resource-policy --secret-id sm-enumerate-password
aws secretsmanager get-resource-policy --secret-id sm-enumerate-api-key
```

---

## Step 6: Extract Secret Values

Retrieve the plaintext values of both secrets:

```bash
aws secretsmanager get-secret-value --secret-id sm-enumerate-password
aws secretsmanager get-secret-value --secret-id sm-enumerate-api-key
```

One of the returned values was base64-encoded. Decode it:

```bash
echo Y3lici1sYWJzLWZha2UtYXBpLWtleS0xMTIy | base64 -d
# Output: cybr-labs-fake-api-key-1122
```

> **Key finding:** The decoded value reveals a (simulated) API key stored in Secrets Manager. In a real engagement, this could enable lateral movement or unauthorized access to third-party services.

---

## Key Takeaways

| Technique | Command | Purpose |
|---|---|---|
| Identity verification | `sts get-caller-identity` | Confirm who you are operating as |
| User enumeration | `iam list-users` | Map all IAM users in the account |
| Inline policy discovery | `iam list-user-policies` / `get-user-policy` | Find and read a user's inline policies |
| Managed policy discovery | `iam list-attached-user-policies` | Find managed policies attached to a user |
| Group mapping | `iam list-groups-for-user` | Check group-based permission inheritance |
| Role enumeration | `iam list-role-policies` / `get-role-policy` | Understand role permissions |
| Secret discovery | `secretsmanager list-secrets` | Identify available secrets |
| Secret extraction | `secretsmanager get-secret-value` | Retrieve plaintext secret values |
| Decoding | `base64 -d` | Decode base64-encoded secret values |

---

## Defensive Notes

- **Least privilege:** Users should only hold permissions required for their specific job function. Broad Secrets Manager read access across all secrets is rarely necessary.
- **Secrets rotation:** Secrets Manager supports automatic rotation — secrets with no rotation policy are a finding.
- **CloudTrail monitoring:** Every command in this lab generates a CloudTrail event. Detection rules watching for `secretsmanager:GetSecretValue` or bulk IAM enumeration calls from unexpected principals would flag this activity.
- **Resource-based policies:** Secrets Manager resource policies can restrict which principals can access a given secret, providing defense in depth beyond IAM alone.

---

## References

- [Cybr.com](https://cybr.com) — Lab platform
- [AWS IAM Documentation](https://docs.aws.amazon.com/iam/)
- [AWS Secrets Manager Documentation](https://docs.aws.amazon.com/secretsmanager/)
- [MITRE ATT&CK T1087.004 — Cloud Account Discovery](https://attack.mitre.org/techniques/T1087/004/)
- [MITRE ATT&CK T1552.004 — Private Keys / Credentials in Cloud Storage](https://attack.mitre.org/techniques/T1552/)
