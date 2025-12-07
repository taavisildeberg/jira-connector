# Security Advisory

## Overview

This document outlines security vulnerabilities identified in the jira-connector library and provides guidance for users and maintainers.

## Current Vulnerabilities

### Critical: Deprecated `request` Library

**Affected Versions**: All versions using `request@2.88.2` or earlier

**Description**: The library depends on the deprecated and unmaintained `request` package, which has known security vulnerabilities.

**Vulnerabilities**:
1. **Server-Side Request Forgery (SSRF)** - CVE-2023-28155
   - CVSS Score: 6.1 (MEDIUM)
   - Impact: Attackers could potentially manipulate HTTP requests
   - GitHub Advisory: GHSA-p8p7-x288-28g6

2. **Unsafe Random in form-data** - GHSA-fjxv-7rqg-78g4
   - Severity: CRITICAL
   - Impact: Weak boundary generation in multipart form data
   - CWE-330: Use of Insufficiently Random Values

3. **Prototype Pollution in tough-cookie** - GHSA-72xf-g2v4-qvf3
   - CVSS Score: 6.5 (MEDIUM)
   - Impact: Potential for prototype pollution attacks
   - CWE-1321: Improperly Controlled Modification of Object Prototype Attributes

### Medium: Deprecated `uuid` Library

**Affected Versions**: All versions using `uuid@3.4.0`

**Description**: The uuid package uses `Math.random()` which is not cryptographically secure.

**Impact**: UUIDs generated for OAuth tokens may be predictable, potentially compromising authentication security.

**Recommendation**: Upgrade to uuid@7.x or higher

### Low: Unsupported `har-validator`

**Affected Versions**: All versions using `har-validator@5.1.5`

**Description**: The har-validator package is no longer maintained.

**Impact**: Potential unpatched vulnerabilities and compatibility issues with newer Node.js versions.

## Risk Assessment

| Vulnerability | Severity | Exploitability | Impact | Risk Level |
|--------------|----------|----------------|---------|-----------|
| request SSRF | Medium-High | Medium | High | **HIGH** |
| form-data random | Critical | Low | High | **HIGH** |
| tough-cookie prototype pollution | Medium | Medium | Medium | **MEDIUM** |
| uuid weak random | Medium | Low | Medium | **MEDIUM** |
| har-validator EOL | Low | Low | Low | **LOW** |

**Overall Risk**: **HIGH**

## Mitigation Strategies

### For Users (Immediate Actions)

1. **Network Security**
   - Use network-level controls to restrict outbound connections from your application
   - Implement allow-lists for Jira API endpoints
   - Use firewalls to prevent SSRF attacks

2. **Input Validation**
   - Validate and sanitize all user inputs before passing to jira-connector
   - Never allow user-controlled URLs to be passed directly to API methods

3. **Monitoring**
   - Monitor for unusual API requests
   - Implement logging for all Jira API calls
   - Set up alerts for unexpected network activity

4. **Consider Alternatives**
   - Evaluate alternative Jira libraries that use modern dependencies
   - Consider implementing your own wrapper using secure HTTP clients

### For Maintainers (Long-term Solutions)

1. **Immediate Priority**: Replace `request` library
   - Recommended: Migrate to `axios` or `node-fetch`
   - This is a breaking change but necessary for security

2. **Update Dependencies**
   ```bash
   npm install axios@latest
   npm uninstall request
   npm install uuid@latest
   ```

3. **Add Security Testing**
   - Implement dependency scanning in CI/CD
   - Use tools like `npm audit`, Snyk, or Dependabot
   - Set up automated security alerts

4. **Implement Security Best Practices**
   - Add request timeouts to prevent hanging requests
   - Implement rate limiting
   - Add input validation for all API methods
   - Use strict Content-Type checking

## Reporting Security Issues

If you discover a security vulnerability in jira-connector, please report it by:

1. **DO NOT** open a public GitHub issue
2. Use GitHub's Security Advisory feature (Recommended): Navigate to the repository's "Security" tab and click "Report a vulnerability"
3. Alternatively, open a private issue by contacting repository maintainers through GitHub
4. Provide detailed information about the vulnerability
5. Allow reasonable time for a fix before public disclosure

## Security Checklist for Users

- [ ] Review all code that uses jira-connector
- [ ] Implement network-level security controls
- [ ] Enable request logging and monitoring
- [ ] Validate all user inputs before passing to library
- [ ] Use environment variables for credentials (never hardcode)
- [ ] Implement proper error handling to avoid credential leaks
- [ ] Keep the library updated to the latest version
- [ ] Subscribe to security advisories for this repository
- [ ] Consider running security scans on your application
- [ ] Review your OAuth implementation for token security

## Timeline

- **2025-12-07**: Vulnerabilities identified in code review
- **[TBD]**: Security advisory published
- **[TBD]**: Fix version planned
- **[TBD]**: Fix version released

## References

- [CVE-2023-28155 - request SSRF](https://nvd.nist.gov/vuln/detail/CVE-2023-28155)
- [GHSA-p8p7-x288-28g6 - request deprecation](https://github.com/advisories/GHSA-p8p7-x288-28g6)
- [GHSA-fjxv-7rqg-78g4 - form-data unsafe random](https://github.com/advisories/GHSA-fjxv-7rqg-78g4)
- [GHSA-72xf-g2v4-qvf3 - tough-cookie prototype pollution](https://github.com/advisories/GHSA-72xf-g2v4-qvf3)
- [request deprecation notice](https://github.com/request/request/issues/3142)

## Additional Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [npm Security Best Practices](https://docs.npmjs.com/security-best-practices)

---

**Last Updated**: December 7, 2025  
**Status**: ACTIVE - Vulnerabilities not yet patched
