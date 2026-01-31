# Security-Sensitive Configuration Parameters in Eclipse Milo

This document catalogs all security-sensitive configuration parameters found in the Eclipse Milo OPC UA implementation.

## Table of Contents

1. [Security Policies and Algorithms](#security-policies-and-algorithms)
2. [Message Security Configuration](#message-security-configuration)
3. [Certificate Management](#certificate-management)
4. [Trust and Validation](#trust-and-validation)
5. [Authentication and Identity](#authentication-and-identity)
6. [Server Configuration](#server-configuration)
7. [Client Configuration](#client-configuration)
8. [Cryptographic Parameters](#cryptographic-parameters)
9. [File and Storage Locations](#file-and-storage-locations)
10. [Network and Endpoint Configuration](#network-and-endpoint-configuration)

---

## Security Policies and Algorithms

### SecurityPolicy Enum
**Location:** `org.eclipse.milo.opcua.stack.core.security.SecurityPolicy`

| Parameter | Security Level | Description | Recommendation |
|-----------|---------------|-------------|----------------|
| `SecurityPolicy.None` | ❌ INSECURE | No encryption or signing | Never use in production |
| `SecurityPolicy.Basic128Rsa15` | ⚠️ DEPRECATED | SHA-1 based, PKCS#1 v1.5 padding | Migrate to SHA-256 policies |
| `SecurityPolicy.Basic256` | ⚠️ DEPRECATED | SHA-1 based algorithms | Migrate to SHA-256 policies |
| `SecurityPolicy.Basic256Sha256` | ✅ SECURE | SHA-256, AES-256, OAEP | Recommended for production |
| `SecurityPolicy.Aes128_Sha256_RsaOaep` | ✅ SECURE | SHA-256, AES-128, OAEP | Acceptable for production |
| `SecurityPolicy.Aes256_Sha256_RsaPss` | ✅ MOST SECURE | SHA-256, AES-256, PSS | Best for high-security environments |

**Security URI Strings:**
- `http://opcfoundation.org/UA/SecurityPolicy#None` - Insecure
- `http://opcfoundation.org/UA/SecurityPolicy#Basic128Rsa15` - Deprecated
- `http://opcfoundation.org/UA/SecurityPolicy#Basic256` - Deprecated
- `http://opcfoundation.org/UA/SecurityPolicy#Basic256Sha256` - Secure
- `http://opcfoundation.org/UA/SecurityPolicy#Aes256_Sha256_RsaPss` - Most Secure

### SecurityAlgorithm Enum
**Location:** `org.eclipse.milo.opcua.stack.core.security.SecurityAlgorithm`

| Algorithm | Type | Security Status |
|-----------|------|-----------------|
| `SecurityAlgorithm.None` | All | ❌ Insecure - No protection |
| `SecurityAlgorithm.HmacSha1` | Symmetric Signature | ⚠️ Deprecated - SHA-1 collision vulnerable |
| `SecurityAlgorithm.HmacSha256` | Symmetric Signature | ✅ Secure |
| `SecurityAlgorithm.Aes128` | Symmetric Encryption | ⚠️ Acceptable - AES-256 preferred |
| `SecurityAlgorithm.Aes256` | Symmetric Encryption | ✅ Secure |
| `SecurityAlgorithm.RsaSha1` | Asymmetric Signature | ⚠️ Deprecated - SHA-1 vulnerable |
| `SecurityAlgorithm.RsaSha256` | Asymmetric Signature | ✅ Secure |
| `SecurityAlgorithm.RsaSha256Pss` | Asymmetric Signature | ✅ Most Secure |
| `SecurityAlgorithm.Rsa15` | Asymmetric Encryption | ⚠️ Deprecated - PKCS#1 v1.5 vulnerable |
| `SecurityAlgorithm.RsaOaepSha1` | Asymmetric Encryption | ✅ Secure |
| `SecurityAlgorithm.RsaOaepSha256` | Asymmetric Encryption | ✅ Most Secure |

---

## Message Security Configuration

### MessageSecurityMode Enum
**Location:** `org.eclipse.milo.opcua.stack.core.types.enumerated.MessageSecurityMode`

| Mode | Security Level | Description |
|------|---------------|-------------|
| `MessageSecurityMode.None` | ❌ INSECURE | No signing or encryption |
| `MessageSecurityMode.Sign` | ⚠️ LIMITED | Integrity only, no confidentiality |
| `MessageSecurityMode.SignAndEncrypt` | ✅ SECURE | Full protection - integrity and confidentiality |

**Configuration Methods:**
- `EndpointConfiguration.Builder.setMessageSecurityMode(MessageSecurityMode)`
- `OpcUaClientConfig.Builder.setMessageSecurityMode(MessageSecurityMode)`

---

## Certificate Management

### Certificate Configuration Classes

#### CertificateManager Interface
**Location:** `org.eclipse.milo.opcua.stack.core.security.CertificateManager`

**Key Methods:**
- `getKeyPair(String applicationUri)` - Retrieves private/public key pair
- `getCertificate(String applicationUri)` - Gets X.509 certificate
- `getCertificateChain(String applicationUri)` - Gets certificate chain

**Implementations:**
- `DefaultCertificateManager` - Standard certificate manager
- File-based storage with PKCS12 keystores

#### KeyStoreCertificateStore
**Location:** `org.eclipse.milo.opcua.stack.core.security.KeyStoreCertificateStore`

**Critical Parameters:**
| Parameter | Type | Security Impact |
|-----------|------|-----------------|
| `keystorePassword` | `Supplier<char[]>` | 🔐 Protects private keys |
| `privateKeyPassword` | `Function<String, char[]>` | 🔐 Protects individual key entries |
| `keystorePath` | `Path` | 🔒 File location - must have restricted permissions |
| `keyAlias` | `String` | Identifies key in keystore |

**Example Insecure Configuration (from examples):**
```java
private static final char[] PASSWORD = "password".toCharArray(); // ❌ INSECURE
```

#### SelfSignedCertificateGenerator/Builder
**Location:** `org.eclipse.milo.opcua.stack.core.util.SelfSignedCertificate*`

**Security Parameters:**
| Parameter | Recommendation | Security Impact |
|-----------|---------------|-----------------|
| RSA Key Size | 2048+ bits (4096 for long-term) | Key strength |
| Validity Period | 1-2 years | Limits exposure window |
| Application URI | Must be unique per instance | Identity verification |
| DNS Names / IP Addresses | Match actual server endpoints | Prevents MITM |

**Methods:**
- `generateRsaKeyPair(int keySize)` - ⚠️ Use 2048+ bits minimum
- `SelfSignedCertificateBuilder.build()` - Generates certificate

---

## Trust and Validation

### Certificate Validation

#### CertificateValidator Interface
**Location:** `org.eclipse.milo.opcua.stack.core.security.CertificateValidator`

**Implementations:**
| Implementation | Purpose | Configuration |
|----------------|---------|---------------|
| `DefaultClientCertificateValidator` | Validates server certificates | Requires TrustListManager |
| `DefaultServerCertificateValidator` | Validates client certificates | Requires TrustListManager |

**Critical Configuration:**
- `setCertificateValidator(CertificateValidator)` - ❌ Must not be null or empty
- Empty validator `cert -> { }` - ❌ INSECURE - Trusts all certificates

#### TrustListManager
**Location:** `org.eclipse.milo.opcua.stack.core.security.TrustListManager`

**Implementations:**
- `FileBasedTrustListManager` - Loads from filesystem
- `MemoryTrustListManager` - In-memory trust list

**Trust Store Locations:**
- `trustedCertificates/` - Trusted CA certificates
- `issuerCertificates/` - Issuer certificates
- `rejectedCertificates/` - Quarantined certificates (FileBasedCertificateQuarantine)

---

## Authentication and Identity

### Identity Validators

#### Server-Side Validators
**Location:** `org.eclipse.milo.opcua.sdk.server.identity.*`

| Validator | Security Level | Use Case |
|-----------|---------------|----------|
| `AnonymousIdentityValidator` | ⚠️ LOW | Public read-only access |
| `UsernameIdentityValidator` | ✅ MEDIUM | Password-based auth |
| `X509IdentityValidator` | ✅ HIGH | Certificate-based auth |
| `CompositeValidator` | ✅ FLEXIBLE | Multiple auth methods |

**Critical Parameters:**
```java
// UsernameIdentityValidator
new UsernameIdentityValidator(
    boolean allowAnonymous,  // ⚠️ Set to false for security
    Predicate<AuthChallenge> // 🔐 Must validate credentials securely
)

// X509IdentityValidator  
new X509IdentityValidator(
    Predicate<X509Certificate> // 🔐 Must validate certificate
)
```

**Insecure Patterns:**
```java
// ❌ INSECURE - Always accepts
setIdentityValidator(UserTokenType.UserName, authChallenge -> true);

// ❌ INSECURE - Hardcoded credentials
"user".equals(username) && "password1".equals(password);
```

### Client-Side Identity Providers
**Location:** `org.eclipse.milo.opcua.sdk.client.api.identity.*`

| Provider | Credentials Type | Security Requirement |
|----------|------------------|---------------------|
| `AnonymousProvider` | None | No authentication |
| `UsernameProvider` | Username/password | Requires encryption (SignAndEncrypt) |
| `X509IdentityProvider` | Certificate | Certificate private key |

**Critical Parameters:**
```java
new UsernameProvider(
    String username,     // Username
    String password      // 🔐 Should be from secure source, not hardcoded
)

new X509IdentityProvider(
    X509Certificate certificate,
    PrivateKey privateKey  // 🔐 Must be protected
)
```

### UserTokenPolicy
**Location:** `org.eclipse.milo.opcua.stack.core.types.structured.UserTokenPolicy`

**Security-Critical Fields:**
| Field | Description | Security Impact |
|-------|-------------|-----------------|
| `tokenType` | UserName, Certificate, Anonymous, IssuedToken | Authentication method |
| `securityPolicyUri` | Security policy for token | 🔐 Must not be "None" for credentials |

---

## Server Configuration

### OpcUaServerConfig
**Location:** `org.eclipse.milo.opcua.sdk.server.OpcUaServerConfig`

**Security-Critical Builder Methods:**

#### Endpoints
```java
.setEndpoints(Set<EndpointConfiguration> endpoints)
```

Each `EndpointConfiguration` includes:
- `bindAddress` - Network binding
- `bindPort` - Port number
- `hostname` - Server hostname
- `path` - Endpoint path
- `certificateSupplier` - 🔐 Certificate provider
- `securityPolicy` - 🔐 Encryption policy
- `messageSecurityMode` - 🔐 Signing/encryption mode
- `tokenPolicies` - 🔐 Allowed authentication methods

#### Identity Validators
```java
.setIdentityValidator(UserTokenType tokenType, IdentityValidator validator)
```

**Must validate:**
- Username/password credentials
- X.509 certificates  
- Anonymous access permissions

#### Certificate Manager
```java
.setCertificateManager(CertificateManager)
```

Provides:
- Server certificate and private key
- Certificate chain

#### Certificate Validator
```java
.setCertificateValidator(CertificateValidator)
```

Validates client certificates

#### Application URI
```java
.setApplicationUri(String)
```

⚠️ Must match server certificate's Application URI

---

## Client Configuration

### OpcUaClientConfig
**Location:** `org.eclipse.milo.opcua.sdk.client.OpcUaClientConfig`

**Security-Critical Builder Methods:**

#### Client Certificate
```java
.setCertificate(X509Certificate)           // 🔐 Client certificate
.setKeyPair(KeyPair)                        // 🔐 Private/public key pair
```

#### Certificate Validation
```java
.setCertificateValidator(CertificateValidator)  // 🔐 Validates server certificate
```

#### Security Configuration
```java
.setEndpoint(EndpointDescription)           // Includes security settings
.setSecurityPolicy(SecurityPolicy)          // 🔐 Encryption policy
.setMessageSecurityMode(MessageSecurityMode) // 🔐 Message protection
```

#### Authentication
```java
.setIdentityProvider(IdentityProvider)      // 🔐 Authentication credentials
```

#### Session Configuration
```java
.setSessionTimeout(UInteger)                // ⚠️ Don't set too long (0 = no timeout)
```

---

## Cryptographic Parameters

### Key Generation

**RSA Key Sizes:**
| Size | Security Level | Use Case |
|------|---------------|----------|
| 1024 bits | ❌ INSECURE | Never use |
| 2048 bits | ✅ MINIMUM | Standard for 2024 |
| 4096 bits | ✅ RECOMMENDED | Long-term security |

**Code:**
```java
SelfSignedCertificateGenerator.generateRsaKeyPair(2048); // Minimum
```

### Nonce Generation
**Location:** `org.eclipse.milo.opcua.stack.core.util.NonceUtil`

**Methods:**
- `generateNonce(int length)` - Uses SecureRandom
- `getNonceLength(SecurityPolicy)` - Policy-dependent length

**Security:** Uses `java.security.SecureRandom` for cryptographic randomness

### Signature and Encryption
**Location:** `org.eclipse.milo.opcua.stack.core.security.SignatureUtil`

**Key Methods:**
- `sign(SecurityAlgorithm, PrivateKey, ByteBuffer...)` - Signs data
- `verify(SecurityAlgorithm, PublicKey, ByteBuffer, ByteBuffer)` - Verifies signature

---

## File and Storage Locations

### Default Paths (from Examples)

| Resource | Default Location | Security Requirement |
|----------|------------------|---------------------|
| Keystore | `example-server.pfx` | 🔒 0600 permissions, encrypted |
| Security PKI | `{java.io.tmpdir}/server/security/pki` | ⚠️ Don't use temp dir in production |
| Trusted Certs | `security/pki/trusted/certs/` | 🔒 Read-only access |
| Rejected Certs | `security/pki/rejected/` | Write access for quarantine |

**Security Considerations:**
- ❌ Never store keystores in `/tmp` or world-readable locations
- ✅ Use absolute paths with restricted permissions (0600 for keys, 0400 for certs)
- ✅ Protect keystore passwords in environment variables or secrets management
- ✅ Separate trust stores by environment (dev/staging/prod)

---

## Network and Endpoint Configuration

### Transport Bindings

**Supported Transports:**
- `opc.tcp://` - OPC UA TCP (binary protocol)
- `https://` - HTTPS (REST/JSON)
- `wss://` - WebSocket Secure

### EndpointConfiguration Parameters

| Parameter | Security Impact | Recommendation |
|-----------|----------------|----------------|
| `bindAddress` | Network exposure | Bind to specific interface, not 0.0.0.0 in production |
| `bindPort` | Service exposure | Use non-standard ports, firewall protection |
| `hostname` | Certificate validation | Must match certificate SAN |
| `securityPolicy` | Encryption | Never use "None" in production |
| `messageSecurityMode` | Message protection | Always use SignAndEncrypt with credentials |

### Endpoint Discovery

**Discovery URLs:**
- `opc.tcp://hostname:4840/discovery` - Default discovery endpoint

**Security:** Discovery endpoint typically uses SecurityPolicy.None but actual endpoints should use encryption

---

## Security Checklist

### Server Configuration
- [ ] SecurityPolicy is NOT "None"
- [ ] MessageSecurityMode is SignAndEncrypt for credential-based auth
- [ ] All IdentityValidators properly validate credentials
- [ ] CertificateValidator is configured and not empty
- [ ] TrustListManager includes only necessary trusted CAs
- [ ] Keystore passwords not hardcoded in source
- [ ] Session timeout is reasonable (not 0, not excessive)
- [ ] Anonymous access disabled or limited to read-only public data
- [ ] Server certificate matches hostname and Application URI

### Client Configuration  
- [ ] Client validates server certificates (CertificateValidator set)
- [ ] Trust list configured with expected server CAs only
- [ ] Client certificate and private key securely stored
- [ ] IdentityProvider credentials from secure source (not hardcoded)
- [ ] SecurityPolicy and MessageSecurityMode match server requirements
- [ ] Connection URL matches server certificate

### Cryptographic Settings
- [ ] RSA keys are 2048+ bits
- [ ] No SHA-1 based algorithms (use SHA-256+)
- [ ] AES-256 preferred over AES-128
- [ ] Certificate expiry dates validated
- [ ] Certificates signed by trusted CA (not self-signed) in production

### File and Permission Settings
- [ ] Keystores have 0600 permissions (owner read/write only)
- [ ] Certificates have 0400 permissions (owner read only)
- [ ] Trust stores in secure, permanent locations (not /tmp)
- [ ] Private keys never logged or transmitted unencrypted

---

## References

1. **OPC UA Specifications:**
   - Part 2: Security Model - https://reference.opcfoundation.org/Core/Part2/v105/
   - Part 4: Services - https://reference.opcfoundation.org/Core/Part4/v105/
   - Part 6: Mappings - https://reference.opcfoundation.org/Core/Part6/v105/

2. **Security Standards:**
   - NIST SP 800-57: Key Management - https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf
   - OWASP Top 10 - https://owasp.org/www-project-top-ten/
   - CWE Database - https://cwe.mitre.org/

3. **Eclipse Milo:**
   - GitHub Repository - https://github.com/eclipse/milo
   - Documentation - https://github.com/eclipse/milo/wiki

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**Maintained by:** Security Team
