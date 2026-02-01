# Security Best Practices in Production

## 1. "Security is not a Feature, it's a Requirement"
A breach costs millions. Developers often create vulnerabilities via negligence.

## 2. Authentication vs Authorization
**AuthN (Who are you?)**: Login (OIDC/OAuth2).
**AuthZ (What can you do?)**: Permissions (RBAC/ABAC).

### 2.1 Handling Passwords
**NEVER** store plain text.
**Use**: `BCrypt` or `Argon2`. These algorithms are "slow by design" to resist GPU brute-forcing.

## 3. The OWASP Top 10 (Critical Fixes)

### 3.1 Injection (SQL / Log)
**Vulnerability**: `query = "SELECT * FROM users WHERE name = '" + userInput + "'"`
**Attack**: Input `'; DROP TABLE users; --`
**Fix**: Using **Prepared Statements** (JPA/Hibernate does this by default).
*   *Log Injection*: Don't log user input directly. They can inject fake log lines to confuse Audit tools. sanitation required.

### 3.2 Broken Access Control (IDOR)
**Vulnerability**: User A calls `/api/orders/555`. 555 belongs to User B. The backend returns it because "User A is logged in".
**Fix**: **Data Authorization**.
```java
@PreAuthorize("@securityService.isOwner(#orderId)")
public Order getOrder(Long orderId) { ... }
```

### 3.3 Sensitive Data Exposure
**Vulnerability**: Storing API Keys / DB Passwords in Git code.
**Fix**: **Secrets Management**.
1.  **Env Variables**: Inject at runtime.
2.  **Vault (HashiCorp)**: Centralized secret storage. App authenticates to Vault to fetch DB pass.

## 4. JWT (JSON Web Tokens) Best Practices
Stateless Auth for Microservices.

1.  **Signing**: Use RS256 (Asymmetric). Auth Service signs with Private Key. Microservices verify with Public Key.
2.  **Expiration**: Short TTL (e.g., 15 mins).
3.  **Revocation**: JWTs cannot be revoked!
    *   *Workaround*: To "logout", you must blacklist the JTI (Token ID) in Redis for the remaining TTL duration.

## 5. Dependency Scanning (Software Supply Chain)
**Issue**: Using a library with a CVE (e.g., Log4Shell).
**Solution**: **Snyk / OWASP Dependency Check** in CI/CD pipeline. Block build if High Severity CVE found.
