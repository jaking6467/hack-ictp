# รายงานการประเมินความปลอดภัย: ICTP Camp 2026 — Cybering Saloon

**เป้าหมาย:** `https://www.ictp-camp-2026.site/`  
**วันที่:** 19 ตุลาคม 2569  
**วิธีการ:** ครอบคลุมทั้งหมด — สำรวจแบบ passive, ทดสอบแบบมีสิทธิ์เข้าถึง, สำรวจ API, วิเคราะห์ซอร์สโค้ดฝั่ง client, โจมตี Supabase โดยตรง  
**บัญชีทดสอบ:** `s44624@bj.ac.th` (บทบาท student, เมือง Erawan, seat_index 4)

---

## สรุปสถาปัตยกรรม

| ส่วนประกอบ | รายละเอียด |
|------------|-----------|
| Framework | Next.js (App Router, Turbopack, RSC) |
| Hosting | Vercel (edge: iad1 → sin1) |
| Backend/DB | Supabase (project: `lpojluoriaguvzfsqfbz`) |
| Auth | Supabase Auth, email/password, ES256 JWT |
| Storage | Supabase Storage (buckets: avatars, announcements) |
| ภาษา | ไทย (`lang="th"`, locale `th_TH`) |
| Theme | "Cybering Saloon" — เกม CTF แบบ puzzle chain |

### Schema ฐานข้อมูล (ตารางที่ยืนยันแล้ว)

| ตาราง | เข้าถึงตรงได้ | RLS | หมายเหตุ |
|-------|--------------|-----|----------|
| `profiles` | SELECT ได้เฉพาะบาง column | แถวตัวเองเท่านั้น | id, display_name, role, email, nickname, avatar_url, seat_index, created_at |
| `puzzles` | ปฏิเสธ (ไม่มี GRANT) | ไม่เกี่ยว | ตารางมีอยู่ แต่เข้าถึง column ไม่ได้เลย |
| `puzzle_chains` | ปฏิเสธ (ไม่มี GRANT) | ไม่เกี่ยว | ตารางมีอยู่ แต่เข้าถึง column ไม่ได้เลย |
| `secrets` | ปฏิเสธ (ไม่มี GRANT) | ไม่เกี่ยว | ตารางมีอยู่ แต่เข้าถึง column ไม่ได้เลย |
| `golden_keys` | ปฏิเสธ (ไม่มี GRANT) | ไม่เกี่ยว | ตารางมีอยู่ แต่เข้าถึง column ไม่ได้เลย |
| `announcements` | SELECT ได้ทั้งหมด | ไม่มี (เห็นทุกแถว) | 2 แถว, มี UUID ผู้สร้างและไฟล์แนบ |

### RPC Functions

| Function | สิทธิ์เข้าถึง | ข้อมูลที่ return |
|----------|-------------|-----------------|
| `get_my_puzzle` | ต้อง login | `{status, seat_index, waiting_on, earned_code, current_seat}` |

### เส้นทางที่ต้อง login

| เส้นทาง | เนื้อหา |
|---------|---------|
| `/camp` | แดชบอร์ดหลัก — ข้อมูลค่าย, รายชื่อสมาชิก |
| `/senior/puzzles` | หน้า puzzle chain — ล็อคเมื่อไม่ใช่คิวของเรา |
| `/senior/cowhands` | มอบหมายสมาชิกในทีม |
| `/senior/announcements` | ฟีดประกาศ |
| `/senior/decrypt` | เครื่องถอดรหัส (ต้องมีชิ้นส่วน golden key ครบ) |
| `/admin` | redirect ไป `/camp` สำหรับ student |

---

## ช่องโหว่ร้ายแรง

### C1. Supabase Publishable Key หลุดใน JavaScript ฝั่ง Client (สูง)

**ตำแหน่ง:** JS chunk `11iz8ajimq99i.js` (257KB) — Supabase client library bundle

**Key ที่พบ:**
```
sb_publishable_9P2erjv6Bf31agUe-5IGsA_3t1-YsOs
```

**บริบทในโค้ดที่ minify แล้ว:**
```javascript
}("https://lpojluoriaguvzfsqfbz.supabase.co","sb_publishable_9P2erjv6Bf31agUe-5IGsA_3t1-YsOs")}],11795)
```

**ผลกระทบ:**
- สามารถเข้าถึง Supabase REST API, Auth API, Storage API และ Realtime WebSocket ได้โดยตรง
- ข้าม proxy ฝั่ง Next.js server — ผู้โจมตีสื่อสารกับ Supabase ตรง
- รวมกับ access token ของผู้ใช้ (ได้จากการ login) สามารถทดสอบ API surface ทั้งหมดได้
- Key นี้ถูกพลาดจากการสแกนครั้งแรก เพราะ format ใหม่ `sb_publishable_` ต่างจาก anon key แบบ JWT เดิมที่ขึ้นต้นด้วย `eyJ`

**หมายเหตุ:** Key นี้ถูก *ออกแบบ* ให้อยู่ฝั่ง client (คล้ายกับ anon key เดิม) ความเสี่ยงจริงขึ้นอยู่กับ RLS policies และ GRANT ระดับตาราง ในกรณีนี้ แอปลดความเสี่ยงได้ดีผ่าน GRANT restrictions ที่เข้มงวด (ดูด้านล่าง)

---

### C2. Supabase Realtime WebSocket — ฉีด Broadcast ได้ (สูง)

**Endpoint:** `wss://lpojluoriaguvzfsqfbz.supabase.co/realtime/v1/websocket?apikey=<key>&vsn=1.0.0`

**สิ่งที่พบ:** ผู้ใช้ที่ login แล้วทุกคนสามารถ:
1. เชื่อมต่อ Realtime WebSocket
2. เข้า broadcast channels
3. ส่ง broadcast messages ไปยัง client ที่เชื่อมต่อทั้งหมด

**หลักฐาน:**
```json
// คำขอเข้าร่วม
{"topic":"realtime:broadcast","event":"phx_join","payload":{"config":{"broadcast":{"self":true}},"access_token":"<jwt>"},"ref":"1"}

// ข้อความ broadcast — ทุกคนที่ listen อยู่ได้รับ
{"topic":"realtime:broadcast","event":"broadcast","payload":{"type":"broadcast","event":"refresh","payload":{"message":"injected"}},"ref":"2"}
```

**ผลกระทบ:**
- Component `LiveRefresh` ของแอปสมัครรับ Realtime และเรียก `router.refresh()` เมื่อมีการเปลี่ยนแปลง
- ผู้โจมตีสามารถกระตุ้นให้ผู้ใช้ทุกคนรีเฟรชหน้าพร้อมกัน
- สามารถฉีด event payload ที่กำหนดเองที่ event handlers ของแอปอาจประมวลผล
- มีโอกาสใช้ social engineering ผ่าน broadcast messages ที่สร้างขึ้น

**วิธีแก้:** ใช้ authorization ระดับ channel — ใช้ Supabase Realtime RLS policies จำกัดว่าใครสามารถ broadcast ไปยัง channel ไหนได้

---

### C3. ไฟล์แนบประกาศ — เข้าถึงได้แบบสาธารณะไม่ต้อง login (ปานกลาง)

**Bucket:** `announcements` (อ่านได้สาธารณะ)

**ไฟล์ที่เข้าถึงได้:**
```
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/9ee4f307-6e1a-4155-a7be-a443e63b2080/file.jpg (73KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/a6681fdc-e6bb-4557-a4ab-0ed7394b7e62/file.jpg (90KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/0c1e0bc6-288c-4b94-85d6-f42556a01f6f/file.jpg (161KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/b5d23d66-2adc-49d7-aec0-01ecb018e449/file.jpg (97KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/dae7f1a9-6dfa-4249-8563-8ff71d0e8b4c/file.jpg (94KB)
https://lpojluoriaguvzfsqfbz.supabase.co/storage/v1/object/public/announcements/27787a3a-772c-4b14-9a68-b33e511148fb/file.png (548KB)
```

**ผลกระทบ:** ไฟล์ทุกไฟล์ใน bucket `announcements` ใครก็อ่านได้ เส้นทางไฟล์เป็น UUIDv4 (เดาไม่ได้) แต่เส้นทางหลุดอยู่ในข้อมูลประกาศซึ่งอ่านได้ทั้งหมดเช่นกัน

---

### C4. RPC Function `get_my_puzzle` — ข้อมูลหลุด (ปานกลาง)

**เรียก:** `POST /rest/v1/rpc/get_my_puzzle` พร้อม authentication

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

**ผลกระทบ:**
- เปิดเผยชื่อเต็มของนักเรียนที่ถือ puzzle chain อยู่ตอนนี้ (`waiting_on`)
- เปิดเผยตำแหน่งปัจจุบันของ chain (`current_seat: 1`)
- รวมกับข้อมูล profile สามารถใช้ social engineering กดดันนักเรียนคนนั้นได้
- เกม puzzle chain สมมติว่าผู้เข้าร่วมไม่รู้ว่าใครเป็นคนบล็อคพวกเขา

---

### C5. ตาราง Announcements — ไม่มี RLS บน SELECT (ปานกลาง)

**สิ่งที่พบ:** ตาราง `announcements` ไม่มี Row-Level Security สำหรับ SELECT ผู้ใช้ที่ login แล้วทุกคน (และอาจจะผู้ใช้ที่ไม่ได้ login ด้วย anon key เท่านั้น) สามารถอ่านประกาศทั้งหมดรวมถึง:
- UUID ผู้สร้าง (`created_by`)
- ชื่อแสดงของผู้สร้าง
- metadata ไฟล์แนบทั้งหมด (เส้นทาง, ขนาด, ชนิด)
- เวลาสร้างและเผยแพร่

**หมายเหตุ:** INSERT, UPDATE, DELETE ถูกบล็อคโดย RLS อย่างถูกต้อง (ทดสอบแล้ว — ทั้งหมด return 0 แถว)

---

### C6. เข้าถึงข้อมูล Profile ผ่าน Column-Specific Queries (ปานกลาง)

**สิ่งที่พบ:** แม้ `SELECT *` จะถูกปฏิเสธบนตาราง `profiles` (ไม่มี GRANT) แต่การเลือก column เฉพาะข้ามข้อจำกัดระดับตารางได้ผ่านสิ่งที่ดูเหมือนจะเป็น view หรือ RLS policy:

```
GET /rest/v1/profiles?select=id,display_name,role,email,nickname,avatar_url,seat_index,created_at
→ 200 OK (Content-Range: 0-0/1)
```

**ข้อมูลที่ได้:**
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

**ผลกระทบ:** email และ seat_index (ตำแหน่ง puzzle chain) ของ profile อ่านได้ RLS จำกัดแค่แถวตัวเอง (ยืนยันแล้ว) แต่รูปแบบ email เปิดเผย format รหัสนักเรียน

---

### C7. Source Maps มีอยู่แต่ถูกควบคุมการเข้าถึง (ปานกลาง)

**สิ่งที่พบ:** ไฟล์ `*.js.map` ทั้งหมด return **403 Forbidden** (ไม่ใช่ 404)

```
11iz8ajimq99i.js.map → 403
01v516rjx45kz.js.map → 403
1p7j8akj4t_bx.js.map → 403
```

**ผลกระทบ:** Source maps ถูก generate และ deploy (น่าจะสำหรับ Vercel error tracking) แต่ถูกควบคุมการเข้าถึงอย่างถูกต้อง อย่างไรก็ตาม:
- ถ้ากลไกป้องกันของ Vercel มีทาง bypass (เช่น header `x-vercel-protection-bypass`) ซอร์สโค้ดฝั่ง server ทั้งหมดจะหลุด
- Source maps จะเปิดเผย Supabase service_role key, logic ของ RPC function ทั้งหมด และ RLS policy implementations

**วิธีแก้:** ปิด source map upload ทั้งหมด หรือให้แน่ใจว่าเข้าถึงได้จาก Vercel error tracking infrastructure เท่านั้น

---

## ช่องโหว่ระดับปานกลาง/ต่ำ

### 8. Auth Settings เข้าถึงได้สาธารณะ (ต่ำ)

**Endpoint:** `GET /auth/v1/settings` (ด้วย publishable key)

**เปิดเผย:**
- Auth providers ทั้งหมด (เปิดแค่ email)
- `disable_signup: true`
- `mailer_autoconfirm: false`
- `phone_autoconfirm: false`

**ผลกระทบ:** ยืนยันว่า attack surface จำกัดเฉพาะ email auth เท่านั้น การสมัครถูกปิด ป้องกันการโจมตีด้วยการสร้างบัญชี

---

### 9. JWKS Public Key หลุด (ต่ำ)

**Endpoint:** `/.well-known/jwks.json`

```json
{
  "keys": [{"kty":"EC","crv":"P-256","kid":"537de384-1dc2-49f5-9c9f-ea04f7ce476d","x":"...","y":"..."}]
}
```

**ผลกระทบ:** ES256 public key เข้าถึงได้ เป็นมาตรฐานสำหรับ JWT verification แต่เปิดโอกาสให้:
- ตรวจสอบ JWT signature นอกแอป
- ตรวจจับ key rotation
- วิเคราะห์ความถูกต้องของ token

---

### 10. Error Messages เปิดเผยการมีอยู่ของ Table/Column (ต่ำ)

**ตัวอย่าง:**
```
403: "Grant SELECT ON public.profiles TO authenticated" → ยืนยันชื่อตารางและสถานะ GRANT
400: "column puzzles.question does not exist" → ยืนยันว่า column ไม่มี (แต่ตารางมี)
400: "column puzzles.chain_order does not exist" → เช่นกัน
```

**ผลกระทบ:** PostgreSQL error messages หลุด schema ฐานข้อมูล ผู้โจมตีสามารถสำรวจ:
- ตารางไหนมีอยู่ (403 vs 404)
- column ไหนมีอยู่ (400 "does not exist" vs 200)
- การตั้งค่า GRANT ต่อ role

---

### 11. SSRF ผ่าน Image Optimizer — ถูกบล็อคอย่างถูกต้อง (ข้อมูล)

ทุก SSRF payloads ที่ส่งไป `/_next/image` return **400 Bad Request**:
- `http://169.254.169.254/latest/meta-data/` → 400
- `file:///etc/passwd` → 400
- `http://localhost:3000/.env` → 400

Image optimizer ของ Vercel ตรวจสอบ URL อย่างถูกต้อง ไม่สามารถโจมตีได้

---

### 12. robots.txt เปิดเผยโครงสร้าง Route (ต่ำ)

เปิดเผย: `/admin`, `/decrypt`, `/senior`, `/camp`, `/preview`, `/login`

---

### 13. ไม่มี API Routes (ข้อมูล)

ทุกเส้นทาง `/api/*` return HTML หลักของแอป แอปใช้ **เฉพาะ** React Server Actions และ RSC — ไม่มี REST API routes เลย ลดพื้นที่โจมตีได้อย่างมาก

---

## สิ่งที่ปลอดภัยดี

| การควบคุม | การ implement | ระดับ |
|----------|-------------|-------|
| GRANT ระดับตาราง | puzzles, secrets, golden_keys, puzzle_chains ล็อคทั้งหมด | ดีเยี่ยม |
| RLS บน profiles | แถวตัวเองเท่านั้น, จำกัดระดับ column | ดี |
| RLS บน announcements writes | INSERT/UPDATE/DELETE ถูกบล็อคทั้งหมด | ดี |
| Storage upload RLS | path traversal ไปโฟลเดอร์ผู้ใช้อื่นถูกบล็อค (403) | ดีเยี่ยม |
| Puzzle chain ฝั่ง server | ไม่สามารถ bypass จาก client — บังคับ chain ใน SECURITY DEFINER functions | ดีเยี่ยม |
| CSP | Nonce-based เข้มงวด, `strict-dynamic`, rotate อย่างถูกต้อง | ดีเยี่ยม |
| HSTS | max-age 2 ปี พร้อม includeSubDomains และ preload | ดีเยี่ยม |
| ป้องกัน SSRF | Image optimizer ตรวจสอบ URL อย่างถูกต้อง | ดี |
| ปิดการสมัคร | ไม่สามารถสร้างบัญชีใหม่ | ดี |
| ป้องกัน Source map | 403 (ไม่ใช่ 404) — ควบคุมการเข้าถึง | ดี |
| ป้องกัน Frame | `X-Frame-Options: DENY` + CSP `frame-ancestors: 'none'` | ดีเยี่ยม |
| ไม่มี API routes | 100% server actions — ไม่มี REST endpoints ให้โจมตี | ดีเยี่ยม |

---

## สรุปการโจมตี

### สิ่งที่ทำได้
1. ดึง Supabase publishable key จาก client-side JS
2. สำรวจพบ 6 ตารางฐานข้อมูลและโครงสร้าง column
3. เข้าถึงข้อมูลประกาศและไฟล์แนบทั้งหมดโดยไม่ต้อง login
4. ฉีด broadcast messages ผ่าน Supabase Realtime WebSocket
5. อ่านข้อมูล profile ตัวเองผ่าน Supabase API ตรง (ข้ามแอป)
6. ยืนยันสถานะ puzzle chain และระบุชื่อนักเรียนที่บล็อค
7. ยืนยันโครงสร้าง storage bucket และความสามารถในการ list ไฟล์

### สิ่งที่ทำไม่ได้
1. อ่านเนื้อหา puzzle, คำตอบ, secrets หรือข้อมูล golden key ใดๆ เลย
2. ยกระดับบทบาทจาก student เป็น admin/staff/senior
3. แก้ไขข้อมูลในฐานข้อมูล (RLS บล็อค writes ทั้งหมด)
4. เข้าถึง profile ผู้ใช้อื่น (RLS จำกัดแค่แถวตัวเอง)
5. อัปโหลดไฟล์ไปเส้นทาง storage ของผู้ใช้อื่น
6. ข้ามลำดับ puzzle chain
7. เข้าถึง source maps หรือโค้ดฝั่ง server
8. โจมตี SSRF
9. สร้างบัญชีใหม่ (ปิดการสมัคร)
10. เข้าถึงเส้นทางหรือฟังก์ชัน admin

---

## ลำดับการแก้ไข

| ลำดับ | ช่องโหว่ | ความรุนแรง | การดำเนินการ |
|-------|---------|-----------|-------------|
| **P0** | ฉีด broadcast ผ่าน Realtime | สูง | เพิ่ม auth policies ระดับ channel |
| **P1** | Publishable key ใน client JS | สูง | ย้ายการเรียก Supabase ทั้งหมดไปฝั่ง server; ลบ client library bundle |
| **P2** | `get_my_puzzle` ข้อมูลหลุด | ปานกลาง | ลบชื่อ `waiting_on` ออกจาก response |
| **P3** | Bucket ประกาศสาธารณะ | ปานกลาง | เปลี่ยนเป็นต้อง login ถ้าไฟล์แนบมีข้อมูลสำคัญ |
| **P4** | Announcements ไม่มี RLS บน SELECT | ปานกลาง | เพิ่ม RLS policy บังคับ authentication |
| **P5** | Error message หลุด schema | ต่ำ | ตั้งค่า PostgREST ให้ return generic errors |
| **P6** | Deploy source maps | ปานกลาง | ลบ source maps จาก production build |
| **P7** | เข้าถึง profile ผ่าน column เฉพาะ | ปานกลาง | จำกัด GRANT ระดับ column ให้เฉพาะ column ที่จำเป็น |

---

## โครงสร้าง JWT

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

## เครื่องมือที่ใช้

- Playwright (headless Chromium) สำหรับ browser automation ที่ต้อง login
- ทดสอบ Supabase REST/Auth/Storage/Realtime API โดยตรง
- วิเคราะห์ JavaScript chunk ฝั่ง client
- สำรวจ column ผ่าน PostgREST ด้วย error-based inference
- ทดสอบ WebSocket protocol
