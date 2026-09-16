# Lab 07 — IAM with Floci

> Identity & Access Management is the backbone of every security decision. Users, groups, roles, policies, and the STS engine — all emulated locally.

**Reference notes:** `services/IAM.md`

## 1. Create a user

```bash
aws iam create-user --user-name alice
aws iam create-user --user-name bob
aws iam list-users --query 'Users[*].[UserName,CreateDate]'
```

## 2. Groups — flat collections

```bash
aws iam create-group --group-name developers
aws iam create-group --group-name admins

aws iam add-user-to-group --group-name developers --user-name alice
aws iam add-user-to-group --group-name developers --user-name bob
aws iam alias-add-group --group-name admin-group --alias developers 2>/dev/null || true
```

> **Groups cannot be nested** — only contain users. (Exam trap.) Also: groups are not an identity — they can't assume roles / be a principal.

## 3. Managed policy attached via group

```bash
# AWS-managed policy — pre-built, versioned, can't modify
aws iam attach-group-policy --group-name developers --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam list-attached-group-policies --group-name developers
```

Both `alice` and `bob` now inherit **AmazonS3ReadOnlyAccess**. Update one place → applies to all.

## 4. Create a custom inline policy (customer-managed)

```bash
cat > developer-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:DescribeInstances",
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-user-policy --user-name alice --policy-name describe-ec2 --policy-document file://developer-policy.json

aws iam list-user-policies --user-name alice
```

Every policy = **Version** + **Statement** (Effect, Action, Resource, optional Condition).

## 5. Roles — the most important concept

```bash
# EC2 service role: trust policy says "EC2 may assume me"
cat > ec2-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ec2-s3-reader \
  --assume-role-policy-document file://ec2-trust.json

# Permission policy: WHAT the role can do
aws iam attach-role-policy \
  --role-name ec2-s3-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam get-role --role-name ec2-s3-reader --query 'Role.AssumeRolePolicyDocument'
```

**Two policies on every role:**
1. **Trust policy** — WHO can assume (here: the EC2 service)
2. **Permission policy** — WHAT it can do (here: read S3)

## 6. Create an instance profile & attach the role

```bash
aws iam create-instance-profile --instance-profile-name ec2-s3-reader-profile
aws iam add-role-to-instance-profile --instance-profile-name ec2-s3-reader-profile --role-name ec2-s3-reader

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id ami-00000000 \
  --instance-type t2.micro \
  --iam-instance-profile Name=ec2-s3-reader-profile \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 describe-instance-attribute --instance-id $INSTANCE_ID --attribute iamInstanceProfile
```

> **Best practice:** grant EC2 access to S3 via a role through an instance profile — never hardcode access keys on the instance.

## 7. STS — the engine behind roles

```bash
# Assume the role yourself
cat > assume-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::000000000000:role/ec2-s3-reader"
    }
  ]
}
EOF

aws sts assume-role \
  --role-arn "arn:aws:iam::000000000000:role/ec2-s3-reader" \
  --role-session-name labsession \
  --duration-seconds 3600
```

The response has **temporary** credentials (AccessKey, SecretAccessKey, SessionToken) that expire after the duration (default 1h, up to 12h).

## 8. Permission boundaries & Deny wins

**Explicit Deny always wins over any Allow.** SCPs grant nothing — they only restrict. Boundaries are a ceiling (effective = intersection).

```bash
cat > boundary.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": "iam:*",
      "Resource": "*"
    }
  ]
}
EOF
```

> Put the boundary on a role → the role can never touch IAM, regardless of any Allow elsewhere.

## 9. Simulate policy

If available in your floci build:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::000000000000:user/alice \
  --action-names "s3:ListBucket" "ec2:DescribeInstances" \
  --query 'EvaluationResults[*].[ActionName,EvalDecision]'
```

## Cleanup

```bash
aws iam delete-user-policy --user-name alice --policy-name describe-ec2
aws iam remove-user-from-group --group-name developers --user-name alice
aws iam remove-user-from-group --group-name developers --user-name bob
aws iam delete-group --group-name developers
aws iam delete-group --group-name admins
aws iam delete-user --user-name alice
aws iam delete-user --user-name bob
aws iam remove-role-from-instance-profile --instance-profile-name ec2-s3-reader-profile --role-name ec2-s3-reader
aws iam delete-instance-profile --instance-profile-name ec2-s3-reader-profile
aws iam detach-role-policy --role-name ec2-s3-reader --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name ec2-s3-reader
```

## Exam hooks

- **Deny wins** → default deny → Allow must be explicit → SCP = outer ceiling
- IAM is **global** (not region-scoped)
- Access keys: max **2 per user**, not auto-rotated
- Cross-account = role + trust policy — no duplicate users
- Root has unrestricted access; can't be restricted by IAM (SCP/Organizations only)

## Gotchas to remember

1. Groups can't be nested
2. Permission boundaries never expand — intersection only
3. Explicit `Deny` beats any `Allow` (implicit deny ≠ explicit deny)
4. SCPs grant nothing
5. Use roles for apps/services; users for people (or legacy)