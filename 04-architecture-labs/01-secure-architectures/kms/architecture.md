# KMS Architecture

## 1. Purpose

This document explains how AWS KMS integrates with IAM and Amazon S3 to protect encrypted data.

The architecture focuses on authorization, encryption, and the relationship between an application and a KMS key.

## 2. Architecture Diagram

```text
                  AWS Account
                       |
                       v
              +------------------+
              |   EC2 Instance   |
              |                  |
              |   IAM Role       |
              +------------------+
                       |
                       | 1. GetObject request
                       v
              +------------------+
              |   Amazon S3      |
              |                  |
              |   SSE-KMS        |
              +------------------+
                       |
                       | 2. KMS decryption
                       |    through service integration
                       v
              +------------------+
              |    AWS KMS       |
              |                  |
              |  KMS Key         |
              |  Key Policy      |
              +------------------+
                       |
                       v
              Decryption permitted
              when authorization
              requirements are met
```

## 3. Components

### EC2 Instance

Runs the application that requests access to encrypted objects.

The application receives temporary credentials through an attached IAM role rather than storing long-term access keys.

### IAM Role

Defines the permissions available to the application.

Depending on the use case, the role may require:

* `s3:GetObject` to read objects.
* `kms:Decrypt` to support reading SSE-KMS encrypted objects.

### Amazon S3

Stores the encrypted objects.

With SSE-KMS, S3 integrates with KMS to protect object data encryption keys.

### AWS KMS

Manages the KMS key and authorizes supported cryptographic operations.

The applicable key policy, IAM permissions, grants, and other restrictions determine whether the required operation is permitted.

## 4. Authorization Flow

1. The application obtains temporary credentials for its EC2 IAM role.
2. The application requests an encrypted object from S3.
3. S3 evaluates whether the request is authorized.
4. For SSE-KMS encrypted objects, the necessary KMS authorization must also be satisfied.
5. KMS performs the required cryptographic operation when authorized.
6. S3 returns the object to the application if all required checks succeed.

Note: The application normally requests the object from S3. S3 handles the KMS integration; the application does not necessarily call KMS directly.

## 5. Key Policy vs IAM Policy

### IAM Policy

Defines which KMS operations the role is allowed to request.

### KMS Key Policy

Controls access to the specific KMS key.

An IAM Allow does not automatically guarantee access to a KMS key. The key policy must support the applicable authorization path, and other restrictions must also be satisfied.

## 6. Envelope Encryption

For envelope encryption:

1. KMS generates a data key.
2. The plaintext data key encrypts the data outside KMS.
3. The encrypted data key is stored with the encrypted data.
4. During decryption, the encrypted data key is sent to KMS.
5. If authorized, KMS returns the plaintext data key.
6. The data key decrypts the encrypted data.

For SSE-KMS, the integrated AWS service manages this process.

## 7. Security Considerations

* Follow least-privilege access.
* Restrict KMS permissions to the required keys and operations.
* Avoid broad permissions such as `kms:*` for application roles.
* Review cross-account key policies and IAM permissions when applicable.
* Use CloudTrail to audit supported KMS API activity.
* Understand the impact of disabling or deleting a key.

## 8. Design Trade-offs

### Customer Managed Keys

Advantages:

* Greater control over key policies.
* More control over configuration and rotation settings.
* Useful for specific access-control and auditing requirements.

Trade-offs:

* Additional configuration and operational responsibility.
* Misconfiguration can prevent applications from accessing encrypted data.
* Disabling or deleting a key can disrupt dependent workloads.

### AWS Managed Keys

Advantages:

* Simpler setup for supported integrated services.
* AWS manages key administration and rotation behavior.

Trade-offs:

* Less control over key policies and configuration than customer managed keys.

## 9. Key Takeaway

Secure access to SSE-KMS encrypted S3 objects requires both resource authorization and the necessary cryptographic authorization.

IAM controls identity permissions, while the KMS key policy and other applicable authorization mechanisms control use of the encryption key.
