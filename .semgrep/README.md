# OPC UA Security Semgrep Rules

This directory contains Semgrep rules for detecting security-sensitive configuration parameters and potential vulnerabilities in OPC UA (Eclipse Milo) implementations.

## Overview

These rules identify security issues related to:
- Insecure security policies and message security modes
- Hardcoded credentials and secrets
- Weak cryptographic configurations
- Certificate and key management issues
- Authentication and authorization weaknesses

## Rule Files

### 1. security-policy-rules.yaml
Detects insecure and deprecated security policy configurations.

**Rules:**
- `opcua-insecure-security-policy-none`: Detects use of SecurityPolicy.None (no encryption)
- `opcua-deprecated-security-policy`: Identifies deprecated policies using SHA-1
- `opcua-message-security-mode-none`: Finds MessageSecurityMode.None usage
- `opcua-message-security-mode-sign-only`: Identifies sign-only mode (no encryption)
- `opcua-security-policy-uri-none`: Detects None policy by URI string

### 2. hardcoded-secrets-rules.yaml
Identifies hardcoded credentials, passwords, and security-relevant constants.

**Rules:**
- `opcua-hardcoded-keystore-password`: Detects hardcoded keystore passwords
- `opcua-hardcoded-credentials`: Finds hardcoded username/password combinations
- `opcua-password-tochararray-literal`: Identifies password literals in method calls
- `opcua-keystore-default-password`: Detects use of default "password" in keystores
- `opcua-applicationuri-hardcoded`: Warns about hardcoded Application URIs

### 3. crypto-weaknesses-rules.yaml
Detects weak or broken cryptographic configurations.

**Rules:**
- `opcua-weak-rsa-key-size`: Identifies RSA keys smaller than 2048 bits
- `opcua-sha1-algorithm-usage`: Detects SHA-1 based algorithms (deprecated)
- `opcua-certificate-validation-disabled`: Finds disabled certificate validation
- `opcua-trust-all-certificates`: Identifies empty certificate validators
- `opcua-anonymous-identity-allowed`: Warns about anonymous authentication
- `opcua-aes128-encryption`: Suggests upgrading from AES-128 to AES-256

### 4. certificate-key-management-rules.yaml
Identifies issues in certificate and private key handling.

**Rules:**
- `opcua-private-key-file-permissions`: Warns about private key file operations
- `opcua-keystore-in-temp-directory`: Detects keystores in temporary directories
- `opcua-certificate-without-validation`: Identifies certificate usage without validation
- `opcua-self-signed-certificate-in-production`: Warns about self-signed certs in production
- `opcua-certificate-expiry-not-checked`: Detects missing expiry validation
- `opcua-keystore-pkcs12-without-encryption`: Finds unencrypted keystore storage

### 5. authentication-authorization-rules.yaml
Detects authentication and authorization security issues.

**Rules:**
- `opcua-username-identity-without-encryption`: Finds unencrypted credential transmission
- `opcua-empty-identity-validator`: Detects weak authentication validators
- `opcua-user-token-policy-without-security`: Identifies insecure token policies
- `opcua-missing-authorization-check`: Warns about methods without authorization
- `opcua-insecure-session-timeout`: Detects excessive session timeouts
- `opcua-password-in-plaintext-memory`: Identifies password storage as String

## Usage

### Running All Rules

```bash
# Run all rules on the repository
semgrep --config .semgrep/

# Run with specific output format
semgrep --config .semgrep/ --json -o results.json

# Run with SARIF output for CI/CD integration
semgrep --config .semgrep/ --sarif -o results.sarif
```

### Running Specific Rule Sets

```bash
# Check only security policy issues
semgrep --config .semgrep/security-policy-rules.yaml

# Check only for hardcoded secrets
semgrep --config .semgrep/hardcoded-secrets-rules.yaml

# Check cryptographic weaknesses
semgrep --config .semgrep/crypto-weaknesses-rules.yaml

# Check certificate management
semgrep --config .semgrep/certificate-key-management-rules.yaml

# Check authentication/authorization
semgrep --config .semgrep/authentication-authorization-rules.yaml
```

### Filtering Results

```bash
# Show only ERROR severity findings
semgrep --config .semgrep/ --severity ERROR

# Exclude example code from scanning
semgrep --config .semgrep/ --exclude "milo-examples/"

# Scan specific directories only
semgrep --config .semgrep/ opc-ua-sdk/ opc-ua-stack/
```

## Integration

### CI/CD Integration

Add to your CI/CD pipeline:

```yaml
# GitHub Actions example
- name: Run Semgrep Security Scan
  run: |
    pip install semgrep
    semgrep --config .semgrep/ --sarif -o semgrep-results.sarif
    
- name: Upload SARIF to GitHub
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: semgrep-results.sarif
```

### Pre-commit Hook

```bash
# .git/hooks/pre-commit
#!/bin/bash
semgrep --config .semgrep/ --error
```

## Security Parameter Categories

### Critical Security Parameters Identified

1. **Security Policies**
   - SecurityPolicy.None (no encryption/signing)
   - Deprecated policies: Basic128Rsa15, Basic256 (SHA-1 based)
   - Recommended: Basic256Sha256, Aes256_Sha256_RsaPss

2. **Message Security Modes**
   - MessageSecurityMode.None (no protection)
   - MessageSecurityMode.Sign (integrity only, no confidentiality)
   - Recommended: MessageSecurityMode.SignAndEncrypt

3. **Credentials & Secrets**
   - Keystore passwords (should use environment variables)
   - User authentication credentials
   - Application URIs
   - Certificate passwords

4. **Cryptographic Settings**
   - RSA key sizes (minimum 2048 bits)
   - Hash algorithms (avoid SHA-1, use SHA-256+)
   - Symmetric encryption (prefer AES-256 over AES-128)
   - Certificate validation settings

5. **Certificate Management**
   - Certificate validation (must be enabled)
   - Trust store configuration
   - Certificate expiry checking
   - Self-signed vs CA-signed certificates
   - Private key storage and permissions

6. **Authentication & Authorization**
   - Identity validators (UsernameIdentityValidator, X509IdentityValidator)
   - Anonymous access policies
   - User token policies
   - Session timeouts
   - Authorization checks on methods

## Remediation Guidance

### Use Secure Security Policies

❌ **Bad:**
```java
EndpointConfiguration.newBuilder()
    .setSecurityPolicy(SecurityPolicy.None)
    .build();
```

✅ **Good:**
```java
EndpointConfiguration.newBuilder()
    .setSecurityPolicy(SecurityPolicy.Aes256_Sha256_RsaPss)
    .setMessageSecurityMode(MessageSecurityMode.SignAndEncrypt)
    .build();
```

### Avoid Hardcoded Credentials

❌ **Bad:**
```java
private static final char[] PASSWORD = "password".toCharArray();
```

✅ **Good:**
```java
private static final char[] PASSWORD = System.getenv("KEYSTORE_PASSWORD").toCharArray();
```

### Implement Certificate Validation

❌ **Bad:**
```java
clientConfig.setCertificateValidator(cert -> { }); // Accepts all
```

✅ **Good:**
```java
DefaultClientCertificateValidator validator = 
    new DefaultClientCertificateValidator(trustListManager);
clientConfig.setCertificateValidator(validator);
```

### Use Strong RSA Keys

❌ **Bad:**
```java
KeyPair keyPair = SelfSignedCertificateGenerator.generateRsaKeyPair(1024);
```

✅ **Good:**
```java
KeyPair keyPair = SelfSignedCertificateGenerator.generateRsaKeyPair(2048);
// Or better: 4096 for long-term security
```

### Secure Password Handling

❌ **Bad:**
```java
String password = authChallenge.getPassword(); // String in memory
```

✅ **Good:**
```java
char[] password = authChallenge.getPasswordAsCharArray();
try {
    // Use password
} finally {
    Arrays.fill(password, '\0'); // Clear immediately
}
```

## Customization

To modify rules for your environment:

1. **Adjust Severity Levels**: Change `severity:` field (ERROR, WARNING, INFO)
2. **Add Exceptions**: Use `pattern-not:` or `pattern-not-inside:` to exclude specific cases
3. **Environment-Specific Rules**: Add `metavariable-regex:` to match environment names
4. **Custom Messages**: Update `message:` field with organization-specific guidance

Example customization:

```yaml
rules:
  - id: opcua-insecure-security-policy-none
    patterns:
      - pattern: SecurityPolicy.None
      # Allow None policy in test environments
      - pattern-not-inside: |
          if (environment.equals("test")) {
            ...
          }
    severity: ERROR
```

## Contributing

When adding new rules:

1. Place them in the appropriate category file
2. Include clear `message` with remediation guidance
3. Add relevant CWE and OWASP mappings in `metadata`
4. Test against actual code before committing
5. Document the rule in this README

## References

- [OPC UA Security Specification](https://reference.opcfoundation.org/Core/Part2/v105/docs/)
- [Semgrep Documentation](https://semgrep.dev/docs/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CWE Database](https://cwe.mitre.org/)
- [Eclipse Milo Documentation](https://github.com/eclipse/milo)

## License

These Semgrep rules are provided under the same license as the Eclipse Milo project (EPL-2.0).
