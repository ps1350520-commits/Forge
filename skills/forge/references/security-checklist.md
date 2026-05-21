# Security Checklist by Domain

## Authentication & Authorization

- [ ] Passwords hashed with bcrypt/argon2 (never MD5/SHA1 for passwords)
- [ ] JWT tokens: short expiry, signed with strong secret, refresh token rotation
- [ ] OAuth: validate state param, use PKCE for public clients
- [ ] Session IDs: regenerate after login, invalidate on logout
- [ ] Role checks happen server-side, not just in the UI
- [ ] Principle of least privilege: users/services get only what they need
- [ ] Multi-factor auth for sensitive operations

## Input Validation

- [ ] Validate on server (client-side validation is UX only, not security)
- [ ] Allowlist validation over blocklist where possible
- [ ] Check types, lengths, formats, ranges
- [ ] Strip or encode HTML in user-supplied content displayed in the browser
- [ ] File uploads: check MIME type, limit file size, store outside webroot, scan for malware

## Database

- [ ] Parameterized queries or ORM always (never string concatenation for queries)
- [ ] DB user has only necessary permissions (no root/admin for app queries)
- [ ] Sensitive fields (SSN, card numbers) encrypted at rest
- [ ] Backups encrypted and tested for restore

## API Security

- [ ] Rate limiting on all endpoints, especially auth
- [ ] CORS: allowlist specific origins, not `*` for credentialed requests
- [ ] HTTPS enforced; HSTS header set
- [ ] Sensitive data not in URL params (use request body)
- [ ] API keys/tokens not logged
- [ ] GraphQL: depth limiting, query complexity limits, disable introspection in production

## Secrets Management

- [ ] No secrets in source code or version control
- [ ] Use env vars or secret managers (AWS Secrets Manager, Vault, etc.)
- [ ] Rotate secrets periodically; rotate immediately on suspected compromise
- [ ] `.env` files in `.gitignore`

## Dependencies

- [ ] `npm audit` / `pip check` / equivalent run in CI
- [ ] Dependencies pinned to specific versions in production
- [ ] Minimal dependencies — fewer packages = fewer attack surfaces
- [ ] Monitor for CVEs (Dependabot, Snyk, etc.)

## Frontend-Specific

- [ ] Content Security Policy (CSP) header configured
- [ ] `dangerouslySetInnerHTML` (React) / `innerHTML` used only with sanitized content
- [ ] Sensitive data not stored in localStorage (use httpOnly cookies)
- [ ] Subresource Integrity (SRI) for third-party scripts

## Error Handling & Logging

- [ ] Stack traces not exposed to users in production
- [ ] Error messages are informative to developers, vague to attackers
- [ ] Logs include context but NOT passwords, tokens, or PII
- [ ] Audit log for sensitive operations (logins, data exports, permission changes)

## Infrastructure

- [ ] Firewall: only necessary ports open
- [ ] SSH: key-based auth only, disable root login
- [ ] Docker: non-root user, read-only filesystem where possible, minimal base image
- [ ] TLS certificates valid and auto-renewed

## Common Vulnerability Patterns to Avoid

```
// ❌ SQL Injection
db.query(`SELECT * FROM users WHERE email = '${email}'`)

// ✅ Parameterized
db.query('SELECT * FROM users WHERE email = $1', [email])

// ❌ XSS
element.innerHTML = userInput

// ✅ Safe
element.textContent = userInput

// ❌ Hardcoded secret
const API_KEY = 'sk-abc123...'

// ✅ Environment variable
const API_KEY = process.env.OPENAI_API_KEY

// ❌ Mass assignment vulnerability
const user = await User.create(req.body) // user could set isAdmin=true

// ✅ Explicit field allowlist
const user = await User.create({ name: req.body.name, email: req.body.email })
```
