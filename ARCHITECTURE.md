# Architecture — laravel-ca-crt (Certificate Management)

## Overview

`laravel-ca-crt` handles the full lifecycle of X.509 certificates: issuance, signing, validation, renewal, revocation, export, and chain building. It provides purpose-specific certificate builders (server TLS, client mTLS, S/MIME, code signing) and includes Microsoft AD CS template support. All certificate operations are logged via `CA\Log` for structured audit trails. It depends on `laravel-ca` (core), `laravel-ca-key` (key access), `laravel-ca-csr` (CSR processing), and `laravel-ca-log` (audit logging).

## Directory Structure

```
src/
├── CrtServiceProvider.php             # Registers signer, validator, manager, exporter, chain builder
├── Builders/
│   ├── AdcsCertificateBuilder.php     # Builder for AD CS-style certificates
│   ├── ClientCertificateBuilder.php   # Builder for client/mTLS certificates
│   ├── CodeSigningCertificateBuilder.php # Builder for code signing certificates
│   ├── ServerCertificateBuilder.php   # Builder for server TLS certificates
│   └── SmimeCertificateBuilder.php    # Builder for S/MIME email certificates
├── Console/
│   └── Commands/
│       ├── CrtIssueCommand.php        # Issue a certificate (ca-crt:issue)
│       ├── CrtListCommand.php         # List certificates with filtering
│       ├── CrtRevokeCommand.php       # Revoke a certificate with reason
│       ├── CrtRenewCommand.php        # Renew an expiring certificate
│       ├── CrtExportCommand.php       # Export certificate in various formats
│       ├── CrtVerifyCommand.php       # Verify a certificate chain
│       └── CrtExpiringScanCommand.php # Scan for certificates nearing expiry
├── Contracts/
│   ├── CertificateManagerInterface.php    # Contract for the main certificate service
│   ├── CertificateSignerInterface.php     # Contract for certificate signing
│   ├── CertificateValidatorInterface.php  # Contract for certificate validation
│   └── MicrosoftTemplateInterface.php     # Contract for Microsoft AD CS templates
├── Events/
│   ├── CertificateIssued.php          # Fired when a certificate is issued
│   ├── CertificateRenewed.php         # Fired when a certificate is renewed
│   ├── CertificateRevoked.php         # Fired when a certificate is revoked
│   └── CertificateExpiring.php        # Fired when approaching expiry threshold
├── Facades/
│   └── CaCrt.php                      # Facade resolving CertificateManagerInterface
├── Http/
│   ├── Controllers/
│   │   └── CertificateController.php  # REST API for certificate operations
│   ├── Requests/
│   │   ├── IssueCertificateRequest.php    # Validation for certificate issuance
│   │   ├── RenewCertificateRequest.php    # Validation for renewal
│   │   └── RevokeCertificateRequest.php   # Validation for revocation
│   └── Resources/
│       └── CertificateResource.php    # JSON API resource for certificate serialization
├── Models/
│   ├── Certificate.php                # Eloquent model for the certificate entity
│   └── CertificateChain.php          # Eloquent model linking certificates in a chain
├── Services/
│   ├── CertificateManager.php         # Full lifecycle: issue, renew, revoke, export, validate
│   ├── CertificateSigner.php          # Signs certificates using CA private key via phpseclib
│   ├── CertificateValidator.php       # Validates certificate chains, expiry, revocation status
│   ├── CertificateRenewer.php         # Handles renewal logic: new key or same key, new serial
│   ├── CertificateExporter.php        # Exports to PEM, DER, PKCS7, Base64
│   └── ChainBuilder.php              # Builds and validates certificate chains from leaf to root
└── Templates/
    ├── TemplateResolver.php           # Maps template names to builder/template classes
    └── MicrosoftTemplates/
        ├── WebServerTemplate.php      # Microsoft Web Server template
        ├── ComputerTemplate.php       # Microsoft Computer template
        ├── UserTemplate.php           # Microsoft User template
        ├── CodeSigningTemplate.php    # Microsoft Code Signing template
        ├── DomainControllerTemplate.php # Microsoft Domain Controller template
        └── SmimeTemplate.php          # Microsoft S/MIME template
```

## Service Provider

`CrtServiceProvider` registers the following:

| Category | Details |
|---|---|
| **Config** | Merges `config/ca-crt.php`; publishes under tag `ca-crt-config` |
| **Singletons** | `ChainBuilder`, `CertificateExporter`, `CertificateSignerInterface` (resolved to `CertificateSigner`), `CertificateValidatorInterface` (resolved to `CertificateValidator`), `CertificateRenewer`, `CertificateManagerInterface` (resolved to `CertificateManager`) |
| **Alias** | `ca-crt` points to `CertificateManagerInterface` |
| **Migrations** | `ca_certificates`, `ca_certificate_chains` tables |
| **Commands** | `ca-crt:issue`, `ca-crt:list`, `ca-crt:revoke`, `ca-crt:renew`, `ca-crt:export`, `ca-crt:verify`, `ca-crt:expiring-scan` |
| **Routes** | API routes under configurable prefix (default `api/ca/certificates`) |

## Key Classes

**CertificateManager** -- The central orchestration service composing the signer, validator, exporter, renewer, chain builder, key manager, and serial generator. It provides the full certificate lifecycle: issue from CSR, revoke with reason code, renew (optionally with new key), export in multiple formats, and validate against a trust chain.

**CertificateSigner** -- Signs X.509 certificates using the issuing CA's private key via phpseclib. Assigns serial numbers, sets validity periods, embeds extensions (Key Usage, SAN, Basic Constraints, AIA, CRL Distribution Points), and produces DER-encoded signed certificates.

**ChainBuilder** -- Traverses parent-child CA relationships to build a complete certificate chain from a leaf certificate up to the root CA. Used for PEM bundle exports and chain validation.

**CertificateRenewer** -- Handles certificate renewal by generating a new serial number and optionally a new key pair. Copies the original certificate's DN and extensions, but updates validity dates and signs with the current CA key.

**TemplateResolver** -- Maps template name strings (e.g., "WebServer", "User", "CodeSigning") to concrete template classes that define the X.509 extensions and Key Usage for that certificate purpose. Supports both standard and Microsoft AD CS-style templates.

## Design Decisions

- **Purpose-specific builders**: Rather than a single monolithic certificate builder, each certificate purpose (server TLS, client, S/MIME, code signing, AD CS) has a dedicated builder class. Each builder pre-configures the correct Key Usage, Extended Key Usage, and SAN types for its purpose.

- **Microsoft AD CS compatibility**: The `MicrosoftTemplates` directory provides Microsoft-compatible certificate templates with the correct OIDs (`1.3.6.1.4.1.311.*`), enabling interoperability with Active Directory environments.

- **Chain as a first-class model**: Certificate chains are stored explicitly in `CertificateChain` rather than computed on the fly, ensuring consistent chain resolution and enabling pre-built PEM bundles.

- **Revocation with reason codes**: Revocation always records a reason code (from RFC 5280: keyCompromise, caCompromise, affiliationChanged, etc.) for CRL and OCSP reporting.

- **Structured audit logging via CA\Log**: All certificate lifecycle operations (issuance, renewal, revocation, validation, signing, export, chain building) are logged through `CaLog` facade. Successful operations use domain-specific methods (`certificateIssued`, `certificateRevoked`, `log`), while errors are captured with `CaLog::critical()` before re-throwing exceptions. This provides a comprehensive audit trail without altering existing control flow.

## PHP 8.4 Features Used

- **`readonly` constructor promotion**: Used in `CertificateManager`, `CertificateSigner`, `CertificateRenewer` for immutable dependencies.
- **Named arguments**: Extensively used in constructor injection and event dispatch.
- **`match` expressions**: Used in template resolution and export format selection.
- **`final` classes**: Service classes prevent unintended inheritance.
- **Strict types**: Every file declares `strict_types=1`.

## Extension Points

- **CertificateSignerInterface**: Replace the signing implementation for hardware-based signing (HSM, cloud KMS).
- **CertificateValidatorInterface**: Bind custom validation logic (e.g., OCSP stapling checks, CT log verification).
- **MicrosoftTemplateInterface**: Implement to add custom Microsoft AD CS templates.
- **Events**: Listen to `CertificateIssued`, `CertificateRevoked`, `CertificateRenewed`, `CertificateExpiring` for audit and automation.
- **Config**: Customize default validity periods, export formats, and route settings via `config/ca-crt.php`.
