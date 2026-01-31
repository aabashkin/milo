# Quick Reference: Security-Sensitive Parameters

This is a quick reference guide for developers working with Eclipse Milo OPC UA.

## ⚠️ Most Critical Parameters to Monitor

### 1. Security Policy (NEVER use "None" in production)
```java
// ❌ INSECURE
.setSecurityPolicy(SecurityPolicy.None)

// ✅ SECURE
.setSecurityPolicy(SecurityPolicy.Aes256_Sha256_RsaPss)
.setSecurityPolicy(SecurityPolicy.Basic256Sha256)
```

### 2. Message Security Mode (Always use SignAndEncrypt with credentials)
```java
// ❌ INSECURE
.setMessageSecurityMode(MessageSecurityMode.None)

// ✅ SECURE
.setMessageSecurityMode(MessageSecurityMode.SignAndEncrypt)
```

### 3. Hardcoded Credentials (NEVER hardcode passwords)
```java
// ❌ INSECURE
private static final char[] PASSWORD = "password".toCharArray();

// ✅ SECURE
private static final char[] PASSWORD = System.getenv("KEYSTORE_PASSWORD").toCharArray();
```

### 4. Certificate Validation (ALWAYS validate certificates)
```java
// ❌ INSECURE - Trusts all certificates
.setCertificateValidator(cert -> { })

// ✅ SECURE - Proper validation
DefaultClientCertificateValidator validator = 
    new DefaultClientCertificateValidator(trustListManager);
.setCertificateValidator(validator)
```

### 5. RSA Key Size (Minimum 2048 bits)
```java
// ❌ INSECURE
generateRsaKeyPair(1024)

// ✅ SECURE
generateRsaKeyPair(2048)  // Minimum
generateRsaKeyPair(4096)  // Recommended for long-term
```

## 🔍 Running Security Scans

### Quick Scan
```bash
# Scan entire repository
semgrep --config .semgrep/

# Scan only ERROR severity
semgrep --config .semgrep/ --severity ERROR

# Scan specific module
semgrep --config .semgrep/ opc-ua-sdk/
```

### Pre-commit Hook
```bash
# Add to .git/hooks/pre-commit
semgrep --config .semgrep/ --error --quiet
```

## 📋 Security Checklist

Before deploying OPC UA server/client:

- [ ] SecurityPolicy is NOT "None"
- [ ] MessageSecurityMode is SignAndEncrypt (when using credentials)
- [ ] No hardcoded passwords in source code
- [ ] Certificate validation is enabled and configured
- [ ] RSA keys are 2048+ bits
- [ ] No SHA-1 based algorithms in production
- [ ] Keystores stored in secure locations (not /tmp)
- [ ] Keystore file permissions are 0600 or stricter
- [ ] Session timeout is reasonable (not 0, not excessive)
- [ ] Anonymous access is disabled or limited appropriately

## 🎯 Rule Categories

| Category | Rules | Severity |
|----------|-------|----------|
| Security Policies | 5 | ERROR/WARNING |
| Hardcoded Secrets | 5 | ERROR |
| Crypto Weaknesses | 6 | ERROR/WARNING/INFO |
| Certificate Management | 6 | ERROR/WARNING |
| Authentication/Authorization | 6 | ERROR/WARNING |

## 📚 More Information

- Full documentation: [SECURITY_PARAMETERS.md](../SECURITY_PARAMETERS.md)
- Rule details: [.semgrep/README.md](.semgrep/README.md)
- OPC UA Security: https://reference.opcfoundation.org/Core/Part2/v105/

## 🚀 CI/CD Integration

### GitHub Actions
```yaml
- name: Security Scan
  run: |
    pip install semgrep
    semgrep --config .semgrep/ --sarif -o results.sarif
    
- uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: results.sarif
```

### GitLab CI
```yaml
semgrep:
  image: semgrep/semgrep
  script:
    - semgrep --config .semgrep/ --json -o semgrep.json
  artifacts:
    reports:
      sast: semgrep.json
```

## ⚡ Quick Fixes

### Fix 1: Remove Insecure Policy
```diff
- .setSecurityPolicy(SecurityPolicy.None)
+ .setSecurityPolicy(SecurityPolicy.Basic256Sha256)
```

### Fix 2: Use Environment Variables
```diff
- private static final char[] PASSWORD = "password".toCharArray();
+ private static final char[] PASSWORD = 
+     System.getenv("KEYSTORE_PASSWORD").toCharArray();
```

### Fix 3: Enable Certificate Validation
```diff
- .setCertificateValidator(cert -> { })
+ .setCertificateValidator(
+     new DefaultClientCertificateValidator(trustListManager))
```
