# security-audit-ecommerce
E-commerce platform security audit case study
# Security Audit: E-Commerce Platform

## Overview

**Scope:** Self-built web application (React SPA + FastAPI + PostgreSQL)  
**Type:** Online cosmetics store  
**Authorization:** Written permission from owner  
**Methodology:** OWASP WSTG 4.0 + automated scanning (nuclei, 13k+ templates)  
**Result:** No critical or high-severity vulnerabilities found

---

## Target & Stack

- **Frontend:** React SPA (Vite), React Router 7 (719 KB bundle)
- **Backend:** FastAPI (Python)
- **Database:** PostgreSQL (Supabase)
- **Auth:** Supabase Auth with JWT (ES256), HttpOnly secure cookies
- **CDN/Hosting:** Vercel (frontend), Railway (backend)

---

## Findings Summary

### 🔴 Critical
None.

### 🟠 Medium Severity (Total effort: ~30 min)

**M-1: Unhandled exceptions on API endpoints**
- Invalid UUID parameters return 500 (Internal Server Error) instead of 400/404
- Stack trace is not exposed, but each error is logged
- **Fix:** Validate parameter types in Pydantic schema; FastAPI will return 422 automatically on type mismatch

**M-2: Incorrect Cache-Control policy on authenticated endpoints**
- `/api/auth/me` and `/api/auth/refresh` return `Cache-Control: public, max-age=0`
- The `public` directive allows intermediate proxies to cache sensitive user data
- **Fix:** Use `Cache-Control: private, no-store` for all authenticated responses

**M-3: Incomplete Content Security Policy**
- Only `frame-ancestors 'none'` is set; missing `default-src`, `script-src`, `object-src`
- Tokens are protected in HttpOnly cookies (safe from JS theft), but CSP remains weak for form hijacking and third-party script injection
- **Fix:** Implement strict CSP with `script-src 'self'`, `default-src 'self'`, `object-src 'none'`, `base-uri 'self'`

### 🟡 Low Severity

| ID | Issue | Remediation | Effort |
|----|-------|-------------|--------|
| L-1 | HSTS missing `includeSubDomains` and `preload` | Add both flags after verifying subdomains | 5 min |
| L-2 | Platform headers leak hosting provider (x-railway-edge, x-railway-request-id) | Remove at CDN layer | 15 min |
| L-3 | Minimum password length: 6 characters | Increase to 8–12; validate against breach databases | 5 min |
| L-4 | Wildcard DNS: `*.example.kg` resolves to hosting provider | Restrict zone to only active subdomains | 15 min |
| L-5 | CORS: `Access-Control-Allow-Origin: *` on static responses | Restrict to own domain | 10 min |
| L-6 | CORS: `Access-Control-Allow-Credentials: true` without proper origin validation | Tighten origin whitelist | 5 min |
| L-7 | Email verification disabled on registration | Enable email confirmation in Supabase Auth settings | Config change |
| L-8 | Chat stores and returns HTML unfiltered (stored XSS risk in admin panel) | Sanitize HTML on output (DOMPurify); never use `dangerouslySetInnerHTML` for user data | 1–2 hours |

---

## What Was Tested ✅

### Access Control (Tested with 2 test accounts)
- **IDOR in orders:** Account A cannot read Account B's orders (404 Not Found) ✅
- **IDOR in support chat:** Chat ID spoofing ignored; server uses session context ✅
- **IDOR write:** Messages sent to the wrong account go to the correct account's chat ✅
- **Privilege escalation:** User attempting to set admin role via API gets 403 ✅
- **Admin endpoints for regular user:** All `/api/admin/*` endpoints return 403 ✅

### Business Logic
- **Negative quantity in orders:** Rejected (422 `greater_than_equal`) ✅
- **Zero quantity:** Rejected (422) ✅
- **Excessive quantity:** Rejected (422 `less_than_equal`) ✅
- **Price tampering:** Client cannot set price; server calculates from product data ✅
- **Non-existent product:** Rejected with clear error message ✅

### Authentication & Cookies
- **Cookie flags:** `HttpOnly`, `Secure`, `SameSite=lax` ✅
- **CSRF protection:** Cross-site POST cannot send cookies due to `SameSite=lax` ✅
- **Token lifespan:** Access token 1 hour, refresh cookie 30 days ✅
- **Rate limiting:** 4 attempts → 429 for 15 min; `/verify-reset-code` limited to 10 attempts ✅

### Input Validation
- **SQL injection:** Parameters are typed (Pydantic); no injection points found ✅
- **Reflected XSS:** Payloads in query parameters rejected before rendering (422 type validation) ✅
- **Form validation:** Server-side validation; clear error messages without info leakage ✅
- **Enumeration prevention:** `forgot-password` uses generic response ("If this email is registered, we sent a link") ✅

### Security Headers
| Header | Value | Status |
|--------|-------|--------|
| HSTS | `max-age=63072000` | ✅ (missing subdomains flag) |
| X-Frame-Options | `DENY` | ✅ |
| X-Content-Type-Options | `nosniff` | ✅ |
| Referrer-Policy | `strict-origin-when-cross-origin` | ✅ |
| Permissions-Policy | `camera=(), microphone=(), geolocation=()` | ✅ |
| TLS Version | 1.2, 1.3 only | ✅ |
| Certificate | Let's Encrypt wildcard, valid until Nov 2026 | ✅ |

### Secrets & Leaks
- **API keys/tokens in bundle:** None found ✅
- **Env variables exposed:** None ✅
- **Source maps:** 403 (not accessible) ✅
- **Common paths** (`/.git/`, `/admin/`, `/wp-*`, `/config.php`): All return SPA catch-all (no real files) ✅
- **Supabase Project ID:** Visible in JWT, but anon key not found in client bundle ✅

### Automated Scanning
- **nuclei (13,326 templates):** 0 findings at medium+ severity ✅
- **CMS scanners** (wpscan, joomscan, droopescan): Not applicable (self-built SPA) ✅
- **Default credentials:** Tested `admin:admin123`, `administrator:administrator`, etc. → all rejected ✅

---

## Recommendations Priority

| Effort | Count | Examples |
|--------|-------|----------|
| < 15 min | 5 | HSTS flags, password length, CORS, wildcard DNS, email verification |
| 30–120 min | 3 | M-1 (exception handling), M-2 (cache headers), M-3 (CSP) |
| Config/External | 1 | L-8 (sanitize HTML in admin panel — requires code review) |

**Total effort for all fixes: ~1 working day**

---

## Architecture Notes

- No traditional CMS present; wpscan and similar tools yield false positives
- JWT is signed by Supabase; token scheme is standard (1 hour access + refresh rotation)
- Row-Level Security (RLS) on PostgreSQL is critical: if anon key leaks, RLS is your only defense
- Admin panel implemented as `/api/admin/*` endpoints with role checks (not a separate app)

---

## Conclusion

The application demonstrates solid security practices above the average for a project of this size. Access control, CORS, rate limiting, TLS, and validation are properly implemented. The main areas for improvement are operational (cache headers, exception handling) and defense-in-depth (CSP, email verification).

**No critical vulnerabilities were found during this assessment.**