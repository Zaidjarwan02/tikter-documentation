# Security Audit & Penetration Test Report
**Date:** 2026-08-31  
**Scope:** MSSP SOC Ticketing Platform — Full Application  
**Tester:** Automated (PowerShell + curl) + Manual Code Review

---

## Executive Summary

| Metric | Before | After |
|--------|--------|-------|
| **Total Tests** | 64 | 64 |
| **PASS** | 35 | 60 |
| **FAIL** | 4 | 0 |
| **WARN** | 25 | 4 |
| **Security Score** | 54.7% | **93.7%** |

### Vulnerability Breakdown

| Severity | Before | After | Status |
|----------|--------|-------|--------|
| HIGH | 2 | 0 | **ALL FIXED** |
| MEDIUM | 3 | 1 | **2 FIXED** |
| LOW | 1 | 0 | **FIXED** |

---

## CVE / Vulnerability Table

| ID | CVE/CWE | Severity | Description | Status | Fix Applied |
|:---|:--------|:--------:|:------------|:------:|:------------|
| SEC-001 | CWE-942 | **HIGH** | CORS reflects any origin with credentials — any malicious site can make authenticated API calls | **FIXED** | Removed development bypass; strict origin allowlist always enforced |
| SEC-002 | CWE-942 | **HIGH** | CORS preflight accepts evil.com — OPTIONS returns Allow-Origin: evil.com | **FIXED** | Same fix as SEC-001; Socket.io CORS also hardened |
| SEC-007 | CWE-614 | **MEDIUM** | Auth cookies missing `Secure` flag — tokens transmitted over HTTP | **FIXED** | `secure: true` + `sameSite: 'strict'` on all auth cookies |
| SEC-010 | CWE-614 | **MEDIUM** | Refresh token cookie path mismatch — `clearCookie` path was `/api/auth/refresh` | **FIXED** | All clearCookie calls now use consistent `path: '/'` with proper flags |
| RL-002 | CWE-770 | **MEDIUM** | No rate limiting on `/api/auth/forgot-password` endpoint | **ACCEPTED** | Endpoint returns 404 (not implemented); no attack surface |
| SEC-006 | CWE-693 | **LOW** | Missing `Permissions-Policy` header | **FIXED** | Added: `camera=(), microphone=(), geolocation=(), payment=(), usb=(), magnetometer=(), gyroscope=(), accelerometer=()` |
| BUG-001 | N/A | **MEDIUM** | `getRoutingDepartments` not exported from ticketController | **FIXED** | Added to module.exports |

---

## npm Dependency Audit

| Scope | Total Deps | Critical | High | Moderate | Low | Total |
|:------|:---------:|:--------:|:----:|:--------:|:---:|:-----:|
| Backend (22 deps) | 257 | 0 | 0 | 0 | 0 | **0** |
| Frontend (14 deps) | 1,385 | 0 | 0 | 2 (dev-only) | 0 | **2** |

### Dependency Vulnerabilities

| CVE / Advisory | Severity | Package | Status | Impact |
|:---------------|:--------:|:--------|:------:|:-------|
| GHSA-9jgg-88mc-972h | MODERATE | `webpack-dev-server` | **OPEN (dev-only)** | Source code theft in dev server — no production impact |
| react-scripts transitive | MODERATE | `react-scripts` | **ACCEPTED** | Parent dependency; no production-safe upgrade available |

**Note:** Both moderate vulnerabilities are development-only (`webpack-dev-server`). They do NOT affect:
- Production JavaScript bundles
- Server-side code
- User data at rest or in transit

---

## Security Headers Verified

| Header | Value | Status |
|:-------|:------|:------:|
| Content-Security-Policy | `default-src 'self'; script-src 'self'; ...` | ✅ |
| X-Frame-Options | `DENY` | ✅ |
| X-Content-Type-Options | `nosniff` | ✅ |
| Strict-Transport-Security | `max-age=31536000; includeSubDomains; preload` | ✅ |
| Referrer-Policy | `strict-origin-when-cross-origin` | ✅ |
| X-XSS-Protection | `0` (modern CSP preferred) | ✅ |
| Permissions-Policy | `camera=(), microphone=(), ...` | ✅ (NEW) |
| Cross-Origin-Opener-Policy | `same-origin` | ✅ |
| Cross-Origin-Resource-Policy | `cross-origin` | ✅ |
| X-DNS-Prefetch-Control | `off` | ✅ |
| X-Download-Options | `noopen` | ✅ |
| X-Permitted-Cross-Domain-Policies | `none` | ✅ |

---

## Cookie Security Verified

| Cookie | HttpOnly | Secure | SameSite | Path | Status |
|:-------|:--------:|:------:|:--------:|:-----|:------:|
| access_token | ✅ | ✅ | Strict | / | ✅ |
| refresh_token | ✅ | ✅ | Strict | / | ✅ |

---

## CORS Policy Verified

| Origin | Result | Status |
|:-------|:------:|:------:|
| `http://localhost:3000` | Allowed | ✅ |
| `http://localhost:4000` | Allowed | ✅ |
| `http://127.0.0.1:3000` | Allowed | ✅ |
| `http://10.x.x.x:*` | Allowed (LAN) | ✅ |
| `http://172.16-31.x.x:*` | Allowed (LAN) | ✅ |
| `http://192.168.x.x:*` | Allowed (LAN) | ✅ |
| `https://evil.com` | **Rejected (500)** | ✅ |
| No origin (server-to-server) | Allowed | ✅ |

---

## Rate Limiting Verified

| Endpoint | Limit | Window | Status |
|:---------|:------|:-------|:------:|
| `/api/auth/login` | 5 attempts | 15 min | ✅ |
| `/api/auth/*` (strict) | 3 attempts | 1 hour | ✅ |
| `/api/*` (global) | 100/10000 requests | 15 min | ✅ |

---

## Input Validation Tests

| Test | Result |
|:-----|:------:|
| SQL Injection (login) | ✅ PASS — Parameterized queries |
| SQL Injection (search) | ✅ PASS — No SQL error |
| XSS (ticket title) | ✅ PASS — Input sanitized |
| Path traversal | ✅ PASS — 404 blocked |
| Null byte injection | ✅ PASS — Rejected |
| NoSQL injection | ✅ PASS — Rejected |
| Command injection | ✅ PASS — No execution |
| SSTI template injection | ✅ PASS — Not evaluated |

---

## Files Modified

| File | Changes |
|:-----|:--------|
| `backend/src/server.js` | CORS strict origin always enforced; Permissions-Policy header added |
| `backend/src/utils/tokenUtils.js` | Cookie flags: `secure: true`, `sameSite: 'strict'`, consistent clearCookie paths |
| `backend/src/controllers/ticketController.js` | Added `getRoutingDepartments` to exports |

---

*Report generated: 2026-08-31*
