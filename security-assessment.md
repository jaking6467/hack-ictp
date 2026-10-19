# Security Assessment: ICTP Camp 2026 — Cybering Saloon

**Target:** `https://www.ictp-camp-2026.site/`  
**Date:** 2026-10-19  
**Method:** Full-spectrum — passive recon, authenticated testing, API enumeration, client-side source analysis, Supabase direct exploitation  
**Test Account:** `s44624@bj.ac.th` (student role, Erawan town, seat_index 4)

---

## Architecture Summary

| Component     | Detail                                         |
|---------------|-------------------------------------------------|
| Framework     | Next.js (App Router, Turbopack, RSC)            |
| Hosting       | Vercel (edge: iad1 → sin1)                     |
| Backend/DB    | Supabase (project: `lpojluoriaguvzfsqfbz`)      |
| Auth          | Supabase Auth, email/password, ES256 JWT        |
| Storage       | Supabase Storage (avatars, announcements buckets)|
| Language      | Thai (`lang="th"`, locale `th_TH`)              |
| Theme         | "Cybering Saloon" — puzzle chain CTF game       |

### Database Schema (Confirmed Tables)

| Table | Direct Access | RLS | Notes |
|-------|---------------|-----|-------|
| `profiles` | SELECT on specific columns only | Own row only | id, display_name, role, email, nickname, avatar_url, seat_index, created_at |
| `puzzles` | DENIED (no GRANT) | N/A | Table exists, zero columns accessible |
| `puzzle_chains` | DENIED (no GRANT) | N/A | Table exists, zero columns accessible |
| `secrets` | DENIED (no GRANT) | N/A | Table exists, zero columns accessible |
| `golden_keys` | DENIED (no GRANT) | N/A | Table exists, zero columns accessible |
| `announcements` | Full SELECT | None (all rows visible) | 2 rows, includes creator UUID and attachments |

### RPC Functions

| Function | Access | Returns |
|----------|--------|---------|
| `get_my_puzzle` | Authenticated | `{status, seat_index, waiting_on, earned_code, current_seat}` |

### Authenticated Routes

| Route | Content |
|-------|---------|
| `/camp` | Main dashboard — camp info, member list |
| `/senior/puzzles` | Puzzle chain page — locked UI when not your turn |
| `/senior/cowhands` | Team member assignments |
| `/senior/announcements` | Announcement feed |
| `/senior/decrypt` | Decryption machine (requires all golden key pieces) |
| `/admin` | Redirects to `/camp` for student role |

---

## Critical Findings

### C1. Supabase Publishable Key Exposed in Client-Side JavaScript (HIGH)

**Location:** JS chunk `11iz8ajimq99i.js` (257KB) — Supabase client library bundle

**Key Found:**
```
sb_publishable_9P2erjv6Bf31agUe-5IGsA_3t1-YsOs
```

**Context in minified code:**
```javascript
}("https://lpojluoriaguvzfsqfbz.supabase.co","sb_publishable_9P2erjv6Bf31agUe-5IGsA_3t1-YsOs")}],11795)
```

**Impact:**
- Enables direct Supabase REST API, Auth API, Storage API, and Realtime WebSocket access
- Bypasses the Next.js server-side proxy — attackers interact with Supabase directly
- Combined with a user's access token (obtained via login), enables full API surface testing
- The key was missed by initial scans because Supabase's new `sb_publishable_` format differs from the traditional `eyJ...` JWT-format anon key

**Note:** This key is *designed* to be client-side (similar to the old anon key). The real risk depends on RLS policies and table-level GRANT configuration. In this case, the app mitigates most risk through strict GRANT restrictions (see below).

---

### C2. Supabase Realtime WebSocket — Broadcast Injection (HIGH)

**Endpoint:** `wss://lpojluoriaguvzfsqfbz.supabase.co/realtime/v1/websocket?apikey=<key>&vsn=1.0.0`

**Finding:** Any authenticated user can:
1. Connect to the Realtime WebSocket
2. Join broadcast channels
3. Send arbitrary broadcast messages to ALL connected clients

**Proof:**
```json
// Join request
{"topic":"realtime:broadcast","event":"phx_join","payload":{"config":{"broadcast":{"self":true}},"access_token":"<jwt>"},"ref":"1"}

// Broadcast message — received by all listeners
{"topic":"realtime:broadcast","event":"broadcast","payload":{"type":"broadcast","event":"refresh","payload":{"message":"injected"}},"ref":"2"}
```

**Impact:**
- The app's `LiveRefresh` component subscribes to Realtime and calls `router.refresh()` on changes
- An attacker could trigger mass page refreshes for all connected users
- Could inject custom event payloads that the app's event handlers might process
- Potential for social engineering through crafted broadcast messages

**Fix:** Implement channel-level authorization. Use Supabase's Realtime RLS policies to restrict which users can broadcast to which channels.

---

### C3. Announcement Attachments — Public Unauthenticated Access (MEDIUM)

**Bucket:** `announcements` (public read access)

**Accessible files:**
```
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/9ee4f307-6e1a-4155-a7be-a443e63b2080/file.jpg (73KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/a6681fdc-e6bb-4557-a4ab-0ed7394b7e62/file.jpg (90KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/0c1e0bc6-288c-4b94-85d6-f42556a01f6f/file.jpg (161KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/b5d23d66-2adc-49d7-aec0-01ecb018e449/file.jpg (97KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/dae7f1a9-6dfa-4249-8563-8ff71d0e8b4c/file.jpg (94KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/27787a3a-772c-4b14-9a68-b33e511148fb/file.png (548KB)
```

**Impact:** Any file in the `announcements` bucket is world-readable. File paths are UUIDv4 (not enumerable), but paths are leaked in the announcement data which is also fully readable.

---

### C4. RPC Function `get_my_puzzle` — Information Leak (MEDIUM)

**Call:** `POST /rest/v1/rpc/get_my_puzzle` with authentication

**Response:**
```json
{
  "status": "locked",
  "seat_index": 4,
  "waiting_on": "ศราวิณณ์ ตันติวรรธณ์",
  "earned_code": null,
  "current_seat": 1
}
```

**Impact:**
- Reveals the full name of the student currently holding the puzzle chain (`waiting_on`)
- Reveals the current chain position (`current_seat: 1`)
- Combined with profile data, enables social engineering to pressure specific students
- The puzzle chain game assumes participants don't know who is blocking them

---

### C5. Announcements Table — No RLS on SELECT (MEDIUM)

**Finding:** The `announcements` table has no Row-Level Security for SELECT operations. All authenticated users (and potentially unauthenticated users with just the anon key) can read all announcements including:
- Creator UUID (`created_by`)
- Creator display name
- Full attachment metadata (paths, sizes, types)
- Creation and publication timestamps

**Note:** INSERT, UPDATE, DELETE are properly blocked by RLS (tested — all return 0 rows affected).

---

### C6. Profile Data Accessible via Column-Specific Queries (MEDIUM)

**Finding:** While `SELECT *` is denied on the `profiles` table (no GRANT), specific column selection bypasses the table-level restriction through what appears to be a view or RLS policy:

```
GET /rest/v1/profiles?select=id,display_name,role,email,nickname,avatar_url,seat_index,created_at
→ 200 OK (Content-Range: 0-0/1)
```

**Data returned:**
```json
{
  "id": "36205a45-e9e2-41a8-91af-c760c15ee0db",
  "display_name": "ศุภโชติ ศรีประทิน",
  "role": "student",
  "email": "s44624@bj.ac.th",
  "nickname": "เจ",
  "avatar_url": "36205a45-e9e2-41a8-91af-c760c15ee0db/5a82fe70-5860-48e0-9a54-68ca9fbda1ab.webp",
  "seat_index": 4,
  "created_at": "2026-09-19T17:20:17.882542+00:00"
}
```

**Impact:** Profile email and seat_index (puzzle chain position) are readable. RLS limits to own row only (verified), but the email pattern reveals the student ID format.

---

### C7. Source Maps Exist but Access-Controlled (MEDIUM)

**Finding:** All `*.js.map` files return **403 Forbidden** (not 404).

```
11iz8ajimq99i.js.map → 403
01v516rjx45kz.js.map → 403
1p7j8akj4t_bx.js.map → 403
```

**Impact:** Source maps are generated and deployed (likely for Vercel error tracking) but properly access-controlled. However:
- If Vercel's protection mechanism has a bypass (e.g., `x-vercel-protection-bypass` header), the full server-side source code would be exposed
- Source maps would reveal the Supabase service_role key, all RPC function logic, and RLS policy implementations

**Fix:** Either disable source map upload entirely, or ensure they're only accessible to Vercel's error tracking infrastructure.

---

## Medium/Low Findings

### 8. Auth Settings Publicly Accessible (LOW)

**Endpoint:** `GET /auth/v1/settings` (with publishable key)

**Reveals:**
- All auth providers (only email enabled)
- `disable_signup: true`
- `mailer_autoconfirm: false`
- `phone_autoconfirm: false`

**Impact:** Confirms attack surface is limited to email auth only. Signup is disabled, preventing account creation attacks.

---

### 9. JWKS Public Key Exposure (LOW)

**Endpoint:** `/.well-known/jwks.json`

```json
{
  "keys": [{"kty":"EC","crv":"P-256","kid":"537de384-1dc2-49f5-9c9f-ea04f7ce476d","x":"...","y":"..."}]
}
```

**Impact:** The ES256 public key is accessible. This is standard for JWT verification but enables:
- JWT signature verification outside the app
- Key rotation detection
- Token validity analysis

---

### 10. Error Messages Reveal Table/Column Existence (LOW)

**Examples:**
```
403: "Grant SELECT ON public.profiles TO authenticated" → confirms table name and GRANT state
400: "column puzzles.question does not exist" → confirms column doesn't exist (but table does)
400: "column puzzles.chain_order does not exist" → ditto
```

**Impact:** PostgreSQL error messages leak the database schema. An attacker can enumerate:
- Which tables exist (403 vs 404)
- Which columns exist (400 "does not exist" vs 200)
- GRANT configuration per role

---

### 11. SSRF via Image Optimizer — Properly Blocked (INFORMATIONAL)

All SSRF payloads against `/_next/image` returned **400 Bad Request**:
- `http://169.254.169.254/latest/meta-data/` → 400
- `file:///etc/passwd` → 400
- `http://localhost:3000/.env` → 400

Vercel's image optimizer properly validates URLs. Not exploitable.

---

### 12. robots.txt Route Structure Disclosure (LOW)

Exposes: `/admin`, `/decrypt`, `/senior`, `/camp`, `/preview`, `/login`

---

### 13. No API Routes (INFORMATIONAL)

All `/api/*` paths return the main HTML application. The app uses **exclusively** React Server Actions and RSC — no REST API routes exist. This significantly reduces the attack surface.

---

## What's Properly Secured

| Control | Implementation | Rating |
|---------|---------------|--------|
| Table-level GRANT | puzzles, secrets, golden_keys, puzzle_chains completely locked | Excellent |
| RLS on profiles | Own row only, column-level restrictions | Good |
| RLS on announcements writes | INSERT/UPDATE/DELETE all blocked | Good |
| Storage upload RLS | Path traversal to other users' folders blocked (403) | Excellent |
| Server-side puzzle chain | No client-side bypass possible — chain enforcement in SECURITY DEFINER functions | Excellent |
| CSP | Strict nonce-based, `strict-dynamic`, proper rotation | Excellent |
| HSTS | 2-year max-age with includeSubDomains and preload | Excellent |
| SSRF protection | Image optimizer properly validates URLs | Good |
| Signup disabled | Cannot create new accounts | Good |
| Source map protection | 403 (not 404) — access controlled | Good |
| Frame protection | `X-Frame-Options: DENY` + CSP `frame-ancestors: 'none'` | Excellent |
| No API routes | 100% server actions — no REST endpoints to attack | Excellent |

---

## Attack Summary

### What We Achieved
1. Extracted the Supabase publishable key from client-side JS
2. Enumerated 6 database tables and their column structures
3. Accessed all announcement data and attachments without authentication
4. Injected broadcast messages via Supabase Realtime WebSocket
5. Read our own profile data via direct Supabase API (bypassing the app)
6. Confirmed puzzle chain status and identified blocking student's name
7. Confirmed storage bucket structure and file listing capabilities

### What We Could NOT Achieve
1. Read ANY puzzle content, answers, secrets, or golden key data
2. Escalate role from student to admin/staff/senior
3. Modify any database records (all writes blocked by RLS)
4. Access other users' profiles (RLS limits to own row)
5. Upload files to other users' storage paths
6. Bypass the puzzle chain order enforcement
7. Access source maps or server-side code
8. Perform SSRF attacks
9. Create new accounts (signup disabled)
10. Access admin routes or functionality

---

## Priority Remediation

| Priority | Finding | Severity | Action |
|----------|---------|----------|--------|
| **P0** | Realtime broadcast injection | HIGH | Add channel-level auth policies |
| **P1** | Publishable key in client JS | HIGH | Move ALL Supabase calls server-side; remove client library bundle |
| **P2** | `get_my_puzzle` info leak | MEDIUM | Remove `waiting_on` name from response |
| **P3** | Public announcement bucket | MEDIUM | Switch to authenticated-only if attachments contain sensitive info |
| **P4** | Announcements no RLS on SELECT | MEDIUM | Add RLS policy requiring authentication |
| **P5** | Error message schema leak | LOW | Configure PostgREST to return generic errors |
| **P6** | Source maps deployed | MEDIUM | Remove source maps from production build |
| **P7** | Column-specific profile access | MEDIUM | Restrict column-level GRANT to only necessary columns |

---

## JWT Structure

```json
{
  "iss": "https://lpojluoriaguvzfsqfbz.supabase.co/auth/v1",
  "sub": "36205a45-e9e2-41a8-91af-c760c15ee0db",
  "aud": "authenticated",
  "role": "authenticated",
  "email": "s44624@bj.ac.th",
  "app_metadata": {"provider": "email", "providers": ["email"]},
  "user_metadata": {"display_name": "ศุภโชติ ศรีประทิน", "email_verified": true, "nickname": "เจ"},
  "aal": "aal1",
  "session_id": "b374b557-720d-42b7-8417-9011e286cd76"
}
```

Algorithm: ES256 (ECDSA P-256)  
Key ID: `537de384-1dc2-49f5-9c9f-ea04f7ce476d`

---

## Tools Used

- Playwright (headless Chromium) for authenticated browser automation
- Direct Supabase REST/Auth/Storage/Realtime API testing
- Client-side JavaScript chunk analysis
- PostgREST column enumeration via error-based inference
- WebSocket protocol testing
