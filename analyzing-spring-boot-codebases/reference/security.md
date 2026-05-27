# Security Review

## Review areas

Assess these categories:

- access control and authorization
- authentication and token handling
- password storage and encoding
- CSRF and CORS posture
- endpoint exposure
- secrets in source control or config
- SQL, JPQL, and command injection risk
- SSRF risk in outbound HTTP calls
- unsafe deserialization or object mapping
- actuator and admin surface exposure
- sensitive data serialization and logging
- file upload validation
- dependency vulnerabilities

## What to flag

### P0
- hardcoded production secrets
- open admin or actuator endpoints with sensitive exposure
- broken auth on protected business endpoints
- confirmed injection vulnerabilities
- unsafe file handling leading to remote code or data compromise

### P1
- overly broad `permitAll`
- CSRF disabled where browser session flows exist
- wildcard CORS in sensitive environments
- weak password hashing or insecure custom crypto
- token leakage in logs

### P2
- missing method-level authorization
- incomplete validation at boundaries
- insecure defaults that are not clearly exploitable
- excessive data exposure in DTOs or entity serialization

### P3
- security hardening improvements
- missing headers or observability improvements

## OWASP summary

Map findings to:

- Broken Access Control
- Cryptographic Failures
- Injection
- Insecure Design
- Security Misconfiguration
- Vulnerable Components
- Identification and Authentication Failures
- Software and Data Integrity Failures
- Security Logging and Monitoring Failures
- SSRF

## Evidence rules

Always include:

- file path
- line number when possible
- why the issue matters
- exact remediation
