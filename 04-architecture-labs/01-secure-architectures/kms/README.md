# AWS Key Management Service (KMS)

## Purpose

AWS Key Management Service (KMS) is a managed service that allows you to create, manage, and control cryptographic keys used to protect data across AWS services.

KMS helps secure data stored in services such as Amazon S3, Amazon EBS, and Amazon RDS.

## Core Concepts

* **KMS Key:** A key used to perform cryptographic operations and control access to encryption.
* **Data Key:** A key used to encrypt data, commonly as part of envelope encryption.
* **Key Policy:** A resource-based policy that controls access to a KMS key.
* **IAM Policy:** Defines which KMS operations an IAM identity is allowed to request.
* **Envelope Encryption:** Encrypts data using a data key, then protects that data key using a KMS key.
* **Encryption Context:** Non-secret contextual information that can be required during supported cryptographic operations.

## How It Works

1. An application requests a cryptographic operation from KMS or an integrated AWS service.
2. KMS evaluates the applicable authorization policies and restrictions.
3. If the request is authorized, KMS performs the requested operation or returns the required cryptographic material.
4. The application or integrated service uses the result to protect or access the data.

For envelope encryption, the application encrypts data with a data key and stores the encrypted data key alongside the encrypted data.

## When to Use KMS

* Encrypting Amazon S3 objects with SSE-KMS.
* Encrypting Amazon EBS volumes.
* Encrypting supported Amazon RDS resources.
* Controlling access to encryption and decryption operations.
* Auditing supported key usage through AWS CloudTrail.

## Security Considerations

* Apply least-privilege permissions.
* Review both IAM policies and KMS key policies.
* Use customer managed keys when additional control over key policies and configuration is required.
* Avoid granting unnecessary administrative or cryptographic permissions.
* Protect encryption keys from accidental deletion or unauthorized changes.
* Remember that encryption alone does not grant access to the underlying resource.

## Important Limitations

* KMS cryptographic operations have service-specific quotas and request limits.
* Direct encryption operations have plaintext size limits; use envelope encryption for larger data.
* Disabling or deleting a key can make encrypted data unavailable.
* Cross-account access requires appropriate authorization on both the key-owning and requesting sides.