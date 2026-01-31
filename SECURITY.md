# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 0.3.x   | :white_check_mark: |
| 0.2.x   | :x:                |
| 0.1.x   | :x:                |

## Reporting a Vulnerability

https://www.eclipse.org/security/

## Security Scanning

This repository includes automated security scanning using Semgrep to detect security-sensitive configuration issues in OPC UA implementations.

### Running Security Scans

```bash
# Install Semgrep
pip install semgrep

# Run security scan
semgrep --config .semgrep/

# Run on specific severity
semgrep --config .semgrep/ --severity ERROR
```

### Security Resources

- **Security Parameter Guide**: [SECURITY_PARAMETERS.md](SECURITY_PARAMETERS.md) - Complete catalog of security-sensitive OPC UA configuration parameters
- **Semgrep Rules**: [.semgrep/](.semgrep/) - 28 security rules across 5 categories
- **Findings Summary**: [SEMGREP_FINDINGS_SUMMARY.md](SEMGREP_FINDINGS_SUMMARY.md) - Current security scan results
- **Quick Reference**: [.semgrep/QUICK_REFERENCE.md](.semgrep/QUICK_REFERENCE.md) - Developer security checklist

### Key Security Configurations

⚠️ **Critical**: The following configurations must be set securely in production:

1. **SecurityPolicy**: Never use `SecurityPolicy.None` (use `Basic256Sha256` or `Aes256_Sha256_RsaPss`)
2. **MessageSecurityMode**: Always use `SignAndEncrypt` when transmitting credentials
3. **Credentials**: Never hardcode passwords or keystore passwords in source code
4. **Certificate Validation**: Always implement proper certificate chain validation
5. **RSA Keys**: Use minimum 2048-bit keys (4096-bit recommended)

See [SECURITY_PARAMETERS.md](SECURITY_PARAMETERS.md) for complete details.
