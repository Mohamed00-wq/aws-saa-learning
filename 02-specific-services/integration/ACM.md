# AWS Certificate Manager  SSL/TLS Certificates

## Purpose

AWS Certificate Manager (ACM) is the service for **provisioning, managing, and deploying public and private SSL/TLS certificates** on AWS. It handles the certificate **lifecycle automatically**  issuance, domain validation, and **automatic renewal** (certificates never expire on you). ACM integrates natively with **Elastic Load Balancing (ALB), CloudFront, API Gateway, Elastic Beanstalk, and CloudFormation**, so you pick a certificate from a drop-down instead of installing cert files. For the SAA exam it's the default answer for **HTTPS/TLS in front of load-balanced or CDN-hosted apps**.

## Main use cases

- **HTTPS termination** on **Application Load Balancers** and **CloudFront distributions**
- **TLS for API Gateway** custom domain names and for Elastic Beanstalk environments
- **Wildcard and SAN certificates** covering many subdomains (e.g. `*.example.com` + `example.com`)
- **Certificate rotation without downtime** (auto-renewal + redeploy to integrated services)
- External TLS (EC2, containers, on-prem) with **exportable public certificates** (new since June 2025) or **imported third-party certificates**
- **Private/internal PKI** for services inside a VPC via **AWS Private Certificate Authority (Private CA)** managed through ACM

## Key features

- **Public certificates are free** when used with integrated AWS services (ALB, CloudFront, API Gateway, ELB, Elastic Beanstalk)
- **Two issuance options**  **non-exportable** (integrated services only, $0) and **exportable public certificates** ($15 per FQDN, $149 per wildcard, paid on issuance/renewal, valid 395 days, usable on EC2/containers/on-prem)
- **Domain validation**  **DNS validation (recommended)** by adding a CNAME record, or email validation. DNS validation also enables **automatic renewal**
- **Automatic renewal** attempted ~45 days before expiry for ACM-issued certificates (no manual renewal)
- **Issued by Amazon Trust Services** (public CA)  trusted by all major browsers
- **Imports** third-party CA certificates (PEM) and lets you monitor their expiry  importing is free
- **Managed private keys** stored encrypted (managed by ACM/KMS), auditable via CloudTrail
- **Reduced ECC (EC_prime256v1, EC_secp384r1)** and RSA key options certs valid up to 395 days (exportable) / ~198 days (integrated standard)

## When to use

- Any scenario with "**ALB + HTTPS**" or "**CloudFront + custom domain with TLS**"
- Need a **free, auto-renewing, trusted public certificate** for an integrated AWS service
- Centralized management and **no more manual cert expiry alerts**
- Terminating TLS at AWS edge/load level, with backend traffic re-encrypted inside the VPC

## Important limitation

- **ACM is regional**  a certificate must live in the **same Region as the resource using it (ALB, API Gateway)**. The major exception is **CloudFront, which is global and requires the certificate in us-east-1 (N. Virginia)**  the classic exam trap
- **Imported certificates are NOT auto-renewed**  ACM only renews ACM-issued certificates. You must import a replacement before expiry
- Certs **created before June 17, 2025 are non-exportable** (can't get the private key). ACM certs traditionally **couldn't be installed directly on EC2 instances** (the new exportable option addresses that)
- Requires a **public domain you control** for validation certs become **pending validation** and ACM gives up after ~72 hours if validation never completes
- Free tier only applies to integrated AWS services  exportable certs, Private CA, and extra API calls are billed

## SAA relevance

- "**HTTPS in front of an ALB**" -> request ACM cert (same Region), attach to HTTPS listener backend instances use their own/internal cert
- "**CloudFront SSL/TLS custom certificate**" -> **ACM cert must be in us-east-1**, created for the CloudFront distribution
- "**Auto-renewing free certificate**" -> ACM public cert with **DNS validation** (Route 53 helps here)
- "**Import vs managed**" -> imported certs are **not renewed automatically** by AWS
- "**Classic limit to remember**" -> ACM is **regional** except **CloudFront (us-east-1)** and certs **cannot be used directly on EC2** unless exportable
- Exam trap: ACM **public certs are free** for integrated services and **renew automatically**  the corners where it breaks are region mismatch, imported certs, and EC2-direct use cases.