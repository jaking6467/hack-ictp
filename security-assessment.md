# Security Assessment: ICTP Camp 2026 — Cybering Saloon

**Target:** `https://www.ictp-camp-2026.site/`
**Date:** 2026-10-19
**Method:** Passive reconnaissance + header analysis + client-side source review

---

## Architecture Summary

| Component     | Detail                                         |
|---------------|-------------------------------------------------|
| Framework     | Next.js (App Router, Turbopack, RSC)            |
| Hosting       | Vercel (`server: Vercel`)                       |
| Backend/DB    | Supabase (project: `lpojluoriaguvzfsqfbz`)      |
| Auth          | School email (`s12345@bj.ac.th`) + password     |
| Language      | Thai (`lang="th"`, locale `th_TH`)              |

### Discovered Routes

| Route       | Status | Notes                        |
|-------------|--------|------------------------------|
| `/`         | 200    | Public homepage              |
| `/login`    | 200    | Auth form (email + password) |
| `/camp`     | 307→`/login` | Protected — requires auth |
| `/senior`   | 307→`/login` | Protected — requires auth |
| `/decrypt`  | 307→`/login` | Protected — requires auth |
| `/admin`    | 307→`/login` | Protected — requires auth |
| `/preview`  | 404    | Route exists in robots.txt but returns 404 |

---

## Findings

### 1. robots.txt — Route Structure Disclosure (Medium)

**File:** `https://www.ictp-camp-2026.site/robots.txt`

```
User-Agent: *
Allow: /
Disallow: /login
Disallow: /camp
Disallow: /senior
Disallow: /decrypt
Disallow: /admin
Disallow: /preview
```

**Impact:** Exposes the full protected route structure. An attacker now knows:
- `/admin` exists — admin panel behind auth
- `/decrypt` exists — likely the secret code decryption feature
- `/senior` — the "พี่รหัส" (senior/buddy) reveal page
- `/camp` — main camp dashboard
- `/preview` — possibly a staging/preview route (currently 404)

**Fix:** Remove sensitive routes from `robots.txt`. Use `noindex` meta tags or `X-Robots-Tag` headers on protected pages instead. The Disallow directive is a suggestion to crawlers, not access control — it's a roadmap for attackers.

---

### 2. Supabase Public Storage Bucket — Unauthenticated File Access (Medium)

**Exposed URLs:**
```
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/<uuid>/file.jpg
```

Three announcement files accessible without authentication:
- `0c1e0bc6-288c-4b94-85d6-f42556a01f6f`
- `9ee4f307-6e1a-4155-a7be-a443e63b2080`
- `a6681fdc-e6bb-4557-a4ab-0ed7394b7e62`

**Impact:**
- The `public` path segment means the Supabase Storage bucket `announcements` has public read access
- If UUIDs are sequential (v1) or predictable, additional files can be enumerated
- Any file uploaded to this bucket is world-readable
- The bucket path pattern is now known — other bucket names can be brute-forced

**Verify:** Check if other buckets exist (e.g., `avatars`, `puzzles`, `secrets`, `uploads`) by testing:
```
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/<bucket_name>/
```

**Fix:** If announcements are meant to be public, this is by design. But ensure no other buckets (especially ones holding puzzle answers, secret codes, or user data) use the `public` access level. Use Supabase RLS (Row Level Security) policies on storage.

---

### 3. Supabase Project ID Exposure (Low)

**Exposed:** `lpojluoriaguvzfsqfbz` in CSP headers, image URLs, and connect-src directives.

The full Supabase project URL is known:
- REST API: `https://lpojluoriaguvzfsqfbz.supabase.co/rest/v1/`
- Auth API: `https://lpojluoriaguvzfsqfbz.supabase.co/auth/v1/`
- Realtime: `wss://lpojluoriaguvzfsqfbz.supabase.co`
- Storage: `https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/`

The REST API returned `401` without an API key — expected behavior. However, if the `anon` key is obtained (it's a public key typically embedded in client JS), the full API surface becomes testable.

**Impact:** Enables direct API attacks against Supabase if the anon key is found or if RLS is misconfigured.

---

### 4. Login Form — Potential Brute Force (Medium)

**Observation:**
- Email field placeholder: `s12345@bj.ac.th` — predictable student ID pattern
- Student IDs follow format: `s` + 5 digits (s00001–s99999)
- No visible rate-limiting headers (`X-RateLimit-*`)
- No CAPTCHA or challenge visible in the HTML
- Uses React Server Actions for form submission

**Impact:**
- Student ID space is small (~100,000 combinations)
- If passwords were distributed (as described: "รหัสผ่านที่ได้รับ" = received password), they may be weak or follow a pattern
- Without rate limiting, credential stuffing or brute force is feasible
- Supabase Auth has built-in rate limiting, but the threshold may be generous

**Potential Attack:**
```
POST /login (React Server Action)
Body: email=s<00001-99999>@bj.ac.th, password=<common_patterns>
```

**Fix:**
1. Implement progressive rate limiting (Supabase has this — verify it's enabled)
2. Add CAPTCHA after N failed attempts
3. Use lockout after repeated failures per IP
4. Ensure error messages don't differentiate "user not found" vs "wrong password" (email enumeration)

---

### 5. Next.js Image Optimizer — Potential SSRF (Medium-High, Untested)

**Endpoint:** `/_next/image?url=<URL>&w=<width>&q=<quality>`

The Next.js image optimization endpoint proxies image requests. By default, it only allows images from `remotePatterns` configured in `next.config.js`.

**Potential Attack Vectors:**
```
/_next/image?url=http://169.254.169.254/latest/meta-data/&w=64&q=75
/_next/image?url=http://localhost:3000/api/internal&w=64&q=75
/_next/image?url=https://evil.com/redirect-to-internal&w=64&q=75
```

**Note:** Could not actively test due to sandbox restrictions. Vercel's production image optimizer typically blocks non-whitelisted domains, but misconfigurations happen.

**Fix:** Verify `images.remotePatterns` in `next.config.js` is locked down to only `lpojluoriaguvzfsqfbz.supabase.co`.

---

### 6. CSP Nonce in HTML — Proper Implementation (Informational)

**Finding:** CSP nonce rotates per request (verified with multiple fetches).

```
Request 1: nonce-OGEwYzdkMzUtNjlkZi00NjJjLTk3OGYtN2UzYzQyMjMwNTQz
Request 2: nonce-NWY0NDE4M2UtZjE1ZS00Nzc3LWE1MWYtNzk4YzZkODVmNGNi
```

The nonces are base64-encoded UUIDs (v4), providing ~122 bits of entropy per request. This is correct.

**However:** The CSP includes `'strict-dynamic'` which means any script loaded by a nonced script can itself load additional scripts. If XSS is found in a server-side rendered context, `strict-dynamic` won't prevent chained script loading.

---

### 7. Server Technology Disclosure (Low)

**Exposed Headers:**
```
server: Vercel
x-vercel-cache: MISS
x-vercel-id: iad1::sin1::n2gvw-1792419045660-b663e82efcdb
```

- Hosting: Vercel (confirmed)
- Edge region: `iad1` (US East) → `sin1` (Singapore)
- Deployment ID format revealed

**Impact:** Low. Confirms hosting stack and region. Useful for targeted attacks against Vercel's infrastructure but minimal direct risk.

---

### 8. Missing security.txt (Low)

`/.well-known/security.txt` returns 404.

**Fix:** Add a `security.txt` file with contact information for responsible disclosure.

---

### 9. Vercel Build ID Leak in RSC Payload (Low)

**Exposed:** `"b":"_i9RURT33JjUXtVGWHC_Y"` in the React Server Component flight data.

This is the Next.js build ID. It changes per deployment and can be used to:
- Detect when new deployments happen
- Fingerprint specific builds
- Potentially access build-specific cached data

---

### 10. Client-Side Route Tree Disclosure via RSC (Low)

The RSC payload reveals the full route structure:
```json
{"P":null,"c":["","login"],"q":"","i":false,"f":[[["",{"children":["(auth)",{"children":["login",...]}]}]]]}
```

This confirms:
- Route groups: `(auth)` wrapping `/login`
- Component tree structure
- Error boundary configuration

---

## What's Solid

| Control                 | Implementation                                        |
|-------------------------|-------------------------------------------------------|
| CSP                     | Strict, nonce-based, `strict-dynamic`, `object-src: none`, `base-uri: none` |
| HSTS                    | `max-age=63072000; includeSubDomains; preload` (2 years) |
| Frame protection        | `X-Frame-Options: DENY` + `frame-ancestors: 'none'`  |
| MIME sniffing           | `X-Content-Type-Options: nosniff`                     |
| COOP/CORP               | `same-origin` for both                                |
| Referrer policy         | `strict-origin-when-cross-origin`                     |
| Permissions policy      | All sensors disabled                                  |
| Cache control           | `private, no-cache, no-store, max-age=0, must-revalidate` |
| CSP nonce rotation      | Per-request UUID v4 nonces                            |
| Auth redirects          | Server-side 307 redirects (not client-side guards)    |
| Form action restriction | `form-action 'self'`                                  |
| DNS prefetch            | Disabled (`x-dns-prefetch-control: off`)              |

---

## Priority Fixes

| Priority | Finding | Action |
|----------|---------|--------|
| **1**    | robots.txt route leak | Remove `/admin`, `/decrypt`, `/senior` from Disallow list |
| **2**    | Login brute force | Verify Supabase rate limiting; add CAPTCHA after failures |
| **3**    | SSRF via image optimizer | Audit `images.remotePatterns` in `next.config.js` |
| **4**    | Supabase bucket audit | Verify no sensitive buckets are public; enable RLS |
| **5**    | Email enumeration | Use generic error messages on login failure |

---

## Scope Limitations

The following tests were **not performed** due to sandbox restrictions:
- Active SSRF testing on `/_next/image`
- Login credential brute force / error message analysis
- Supabase REST/Auth API enumeration (requires anon key)
- POST-based form testing
- XSS payload testing on input fields
- Cookie analysis (requires authenticated session)
- WebSocket (Realtime) subscription testing
- Direct Supabase RLS policy verification

These would require authenticated access or explicit authorization to test actively.
