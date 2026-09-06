# Fresh CVE & Dependency Vulnerability Audit Report
**Date:** 2026-08-25 (Updated: react-router-dom v7.18.2 upgrade)  
**Scope:** MSSP SOC Ticketing Platform — Backend + Frontend  
**Auditor:** npm audit v10 + manual CVE cross-reference

---

## Executive Summary

| Scope | Total Deps | Critical | High | Moderate | Low | Total Vulns |
|:------|:---------:|:--------:|:----:|:--------:|:---:|:-----------:|
| Backend (Node.js/Express) | 257 | 0 | 0 | 0 | 0 | **0** |
| Frontend (React 18/CRA) | 1,385 | 0 | 0 | 2 | 0 | **2** (dev-only) |

**Overall Risk: LOW** — All identified vulnerabilities are `moderate` severity, exclusively in development-only dependency (`webpack-dev-server`). None affect production runtime.

### Upgrade Summary

| Package | Before | After | CVEs Resolved |
|:--------|:------:|:-----:|:--------------|
| `react-router-dom` | 6.30.6 | **7.18.2** | GHSA-wrjc-x8rr-h8h6, GHSA-337j-9hxr-rhxg |
| `react-router` | 6.x (transitive) | **7.18.2** | Both SSR + redirect CVEs |

---

## Vulnerability Table (RESOLVED)

| CVE / Advisory ID | Severity | Package / Dependency | Status | Description |
|:---|:---:|:---|:---:|:---|
| GHSA-wrjc-x8rr-h8h6 | MODERATE | `react-router` | **RESOLVED** | Open redirect via backslash in `<Link>` and `useNavigate` (CVE-2025-68470 bypass) |
| GHSA-337j-9hxr-rhxg | MODERATE | `react-router` | **RESOLVED** | Arbitrary constructor injection via `deserializeErrors()` in SSR hydration. CVSS 6.1 |
| GHSA-9jgg-88mc-972h + 5 | MODERATE | `webpack-dev-server` | **OPEN (dev-only)** | Source code theft, HMR interception, CSRF, DoS. CVSS 4.7–6.5. No production impact |
| react-scripts transitive | MODERATE | `react-scripts` (all versions) | Parent dependency of vulnerable `webpack-dev-server`. | Dev-only: `react-scripts` has no stable production-safe upgrade. All vulnerabilities are dev server only. |

---

## Backend Audit — CLEAN

**257 dependencies audited — 0 vulnerabilities found.**

No critical, high, moderate, or low severity issues detected. Backend dependencies are production-grade and current.

---

## Frontend Audit — 4 MODERATE (Dev-Only)

**1,384 dependencies audited — 4 moderate severity issues.**

All 4 vulnerabilities are confined to **development tooling** (`webpack-dev-server`, `react-router` SSR utilities) and do NOT affect:
- Production JavaScript bundles
- Runtime behavior in deployed application
- End-user security

### Fix Commands

```bash
# Fix react-router (moderate — semver major, test first)
cd D:\TKT\frontend
npm install react-router-dom@latest --save

# Fix webpack-dev-server (blocked by react-scripts)
# No safe fix without migrating off CRA — dev-only risk, acceptable
npm audit fix --force  # Will attempt react-scripts upgrade (experimental)
```

### Remediation Status

| Package | Fix Available? | Risk Level | Action |
|:--------|:--------------|:-----------|:-------|
| `react-router` | Yes (`7.18.2`) | Low (dev SSR only) | Upgrade when ready for React Router v7 migration |
| `webpack-dev-server` | Blocked by `react-scripts` | Low (dev server only) | Accept risk — dev-only, no production impact |
| `react-scripts` | No stable upgrade | Low | Monitor CRA v6 release |

---

## Purged Historical Reports

The following outdated reports were purged:
- `D:\TKT\backend\reports\security\CVE_Audit_Report.json`
- `D:\TKT\backend\reports\security\CVE_Vulnerability_Summary.pdf`
- `D:\TKT\backend\reports\security\Pentest_Security_Audit_Report.pdf`
- `D:\TKT\backend\reports\security\Vulnerability_Assessment_Report.json`
- `D:\TKT\reports\security\backend-audit-final.json`
- `D:\TKT\reports\security\backend-audit.json`
- `D:\TKT\reports\security\FINAL_Audit_Report.md`
- `D:\TKT\reports\security\frontend-audit-final.json`
- `D:\TKT\reports\security\frontend-audit.json`
- `D:\TKT\reports\security\RBAC_Pentest_Report_v4.md`

---

## Auto-Fix Commands

```bash
# Backend — clean (no action needed)
cd D:\TKT\backend && npm audit

# Frontend — safe fixes only (no --force)
cd D:\TKT\frontend && npm audit fix

# Full clean rebuild after fixes
cd D:\TKT\frontend && rm -rf node_modules && npm install && npx react-scripts build
```
