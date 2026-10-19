# รายงานการประเมินความปลอดภัย: ICTP Camp 2026 — Cybering Saloon

**เป้าหมาย:** `https://www.ictp-camp-2026.site/`  
**วันที่:** 19 ตุลาคม 2569  
**วิธีการ:** ครอบคลุมทั้งหมด — สำรวจแบบ passive, ทดสอบแบบมีสิทธิ์เข้าถึง, สำรวจ API/RPC/Realtime, วิเคราะห์ซอร์สโค้ดฝั่ง client, ดึงข้อมูล RSC, ปลอม JWT, ทดสอบ IDOR, brute force admin, โจมตี Supabase โดยตรง  
**บัญชีทดสอบ:** `s44624@bj.ac.th` (บทบาท student, เมือง Erawan/city_id=1, seat_index 4, cowhand="cowboy")  
**ขั้นตอน:** สคริปต์โจมตี 10+ ตัว, เรียก RPC 60+ ครั้ง, ลองรหัสผ่าน admin 40+ ครั้ง, dump ข้อมูล RSC ทั้งหมด, ดักจับ WebSocket

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

### Schema ฐานข้อมูล (ตาราง/View ที่ยืนยันแล้ว)

| ตาราง/View | เข้าถึงตรงได้ | RLS | หมายเหตุ |
|------------|--------------|-----|----------|
| `profiles` | SELECT ได้เฉพาะบาง column | แถวตัวเองเท่านั้น | id, display_name, role, email, nickname, avatar_url, seat_index, created_at |
| `cities` | **SELECT ได้ทั้งหมด** | ไม่มี | ทั้ง 6 เมือง: id, slug, name_th, name_en, blurb, map_x, map_y, accent_hex, is_active |
| `city_progress` (view) | **SELECT ได้ทั้งหมด** | ไม่มี | city_id, current_seat, solved_count, last_solved_by, last_solved_at, updated_at |
| `submissions` | ปฏิเสธ (ไม่มี GRANT) | ไม่เกี่ยว | ตารางมีอยู่ แต่ schema ถูกซ่อนทั้งหมด — column ไม่อยู่ใน PostgREST cache |
| `puzzles` | ปฏิเสธ (ไม่มี GRANT) | ไม่เกี่ยว | ตารางมีอยู่ แต่เข้าถึง column ไม่ได้เลย |
| `puzzle_chains` | ปฏิเสธ (ไม่มี GRANT) | ไม่เกี่ยว | ตารางมีอยู่ แต่เข้าถึง column ไม่ได้เลย |
| `secrets` | ปฏิเสธ (ไม่มี GRANT) | ไม่เกี่ยว | ตารางมีอยู่ แต่เข้าถึง column ไม่ได้เลย |
| `golden_keys` | ปฏิเสธ (ไม่มี GRANT) | ไม่เกี่ยว | ตารางมีอยู่ แต่เข้าถึง column ไม่ได้เลย |
| `announcements` | SELECT ได้ทั้งหมด | ไม่มี (เห็นทุกแถว) | 2 แถว, มี UUID ผู้สร้างและไฟล์แนบ |

### RPC Functions (SECURITY DEFINER)

| Function | สิทธิ์เข้าถึง | ข้อมูลที่ return |
|----------|-------------|-----------------|
| `get_my_puzzle` | ต้อง login | `{status, seat_index, waiting_on, earned_code, current_seat}` — ฉีด parameter ไม่ได้ |
| `get_my_senior` | ต้อง login | `{status}` — ข้อมูลน้อยมาก, ไม่รับ parameter |

### โครงสร้างเกม

| องค์ประกอบ | รายละเอียด |
|-----------|-----------|
| เมือง | 6 เมือง (Erawan, Mayurawentai, Nakkharat, Bandhu-Ratchasi, Kilen, Khrutha) |
| ผู้เล่น | นักเรียน 36 คน, เมืองละ 6 คน |
| Chain | Puzzle chain ตามลำดับ #1→#2→...→#6, ทีละคน |
| Golden Keys | 6 ชิ้น (เมืองละชิ้น), ได้เมื่อทำ chain ครบ |
| ถอดรหัส | ต้องมี golden key ครบทั้ง 6 ชิ้นถึงปลดล็อค |
| ความคืบหน้า | แก้ปริศนาแล้ว 7/36 ทั้งระบบ ณ เวลาทดสอบ (เมือง 3,4,5,6 มีความคืบหน้า) |

### เส้นทางที่ต้อง login

| เส้นทาง | เนื้อหา |
|---------|---------|
| `/camp` | แดชบอร์ดหลัก — ข้อมูลค่าย, รายชื่อสมาชิก |
| `/senior/puzzles` | หน้า puzzle chain — ล็อคเมื่อไม่ใช่คิวของเรา |
| `/senior/cowhands` | มอบหมายสมาชิกในทีม |
| `/senior/announcements` | ฟีดประกาศ |
| `/senior/decrypt` | เครื่องถอดรหัส (ต้องมี golden key ครบ) — ข้อความล็อค: "ประตูสำนักงานนายอำเภอยังปิดตาย" |
| `/admin` | แดชบอร์ด admin ("แผงควบคุมพี่ค่าย") — แสดง "ไม่มีสิทธิ์" สำหรับ student |
| `/admin/puzzles`, `/admin/answers` ฯลฯ | redirect หรือแสดง permission denied ทั้งหมด |

---

## ช่องโหว่ร้ายแรง

### C1. Supabase Publishable Key หลุดใน JavaScript ฝั่ง Client (สูง)

**ตำแหน่ง:** JS chunk `11iz8ajimq99i.js` (257KB) — Supabase client library bundle

**Key ที่พบ:**
```
sb_publishable_9P2erjv6Bf31agUe-5IGsA_3t1-YsOs
```

**ผลกระทบ:**
- สามารถเข้าถึง Supabase REST API, Auth API, Storage API และ Realtime WebSocket ได้โดยตรง
- ข้าม proxy ฝั่ง Next.js server — ผู้โจมตีสื่อสารกับ Supabase ตรง
- Key นี้ถูกพลาดจากการสแกนครั้งแรก เพราะ format ใหม่ `sb_publishable_` ต่างจาก anon key แบบ JWT เดิม

**หมายเหตุ:** Key นี้ถูก *ออกแบบ* ให้อยู่ฝั่ง client แต่ในกรณีนี้แอปลดความเสี่ยงได้ดีผ่าน GRANT restrictions ที่เข้มงวด

---

### C2. Supabase Realtime WebSocket — ฉีด Broadcast ได้ (สูง)

**สิ่งที่พบ:** ผู้ใช้ที่ login แล้วทุกคนสามารถ:
1. เชื่อมต่อ Realtime WebSocket
2. เข้า broadcast channels
3. ส่ง broadcast messages ไปยัง client ที่เชื่อมต่อทั้งหมด

**ผลกระทบ:**
- Component `LiveRefresh` ของแอปสมัครรับ Realtime และเรียก `router.refresh()` เมื่อมีการเปลี่ยนแปลง
- ผู้โจมตีสามารถกระตุ้นให้ผู้ใช้ทุกคนรีเฟรชหน้าพร้อมกัน
- มีโอกาสใช้ social engineering ผ่าน broadcast messages ที่สร้างขึ้น

**วิธีแก้:** ใช้ authorization ระดับ channel ผ่าน Supabase Realtime RLS policies

---

### C3. ไฟล์แนบประกาศ — เข้าถึงได้แบบสาธารณะไม่ต้อง login (ปานกลาง)

**Bucket:** `announcements` (อ่านได้สาธารณะ)

**ผลกระทบ:** ไฟล์ทุกไฟล์ใน bucket `announcements` ใครก็อ่านได้ เส้นทางหลุดอยู่ในข้อมูลประกาศซึ่งอ่านได้ทั้งหมด

---

### C4. RPC Function `get_my_puzzle` — ชื่อนักเรียนหลุด (ปานกลาง)

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

**ผลกระทบ:** เปิดเผยชื่อเต็มของนักเรียนที่ถือ puzzle chain อยู่ — เกมสมมติว่าผู้เข้าร่วมไม่รู้ว่าใครบล็อค

---

### C5. ตาราง Announcements — ไม่มี RLS บน SELECT (ปานกลาง)

ผู้ใช้ทุกคนอ่านประกาศได้ทั้งหมดรวมถึง UUID ผู้สร้าง, ชื่อ, metadata ไฟล์แนบ INSERT/UPDATE/DELETE ถูกบล็อคอย่างถูกต้อง

---

### C6. เข้าถึงข้อมูล Profile ผ่าน Column-Specific Queries (ปานกลาง)

แม้ `SELECT *` ถูกปฏิเสธ แต่เลือก column เฉพาะได้ RLS จำกัดแค่แถวตัวเอง (ยืนยันแล้ว)

---

### C7. Source Maps ถูกควบคุมการเข้าถึง (ปานกลาง)

ไฟล์ `*.js.map` ทั้งหมด return 403 Forbidden ถ้า Vercel protection มีทาง bypass ซอร์สโค้ดทั้งหมดจะหลุด

---

### C8. `city_progress` View — สถานะเกมทั้งหมดหลุด (สูง)

**เรียก:** `GET /rest/v1/city_progress?select=*&order=city_id`

**Response (ทั้ง 6 เมือง):**
```json
[
  {"city_id":1, "current_seat":1, "solved_count":0, "last_solved_by":null},
  {"city_id":2, "current_seat":1, "solved_count":0, "last_solved_by":null},
  {"city_id":3, "current_seat":3, "solved_count":2, "last_solved_by":"0cc4d3d9-..."},
  {"city_id":4, "current_seat":2, "solved_count":1, "last_solved_by":"6c847a78-..."},
  {"city_id":5, "current_seat":2, "solved_count":1, "last_solved_by":"d9638ebc-..."},
  {"city_id":6, "current_seat":4, "solved_count":3, "last_solved_by":"5654641e-..."}
]
```

**ผลกระทบ:**
- เปิดเผยว่าเมืองไหนนำ/ตามในการแข่งปริศนา
- หลุด UUID ผู้แก้ (`last_solved_by`) — เทียบหาชื่อได้จากข้อมูล RSC
- เปิดเผยตำแหน่ง chain ของทั้ง 6 เมืองพร้อมกัน
- ไม่มี RLS — ผู้ใช้ทุกคนเห็นความคืบหน้าทุกเมือง

---

### C9. ตาราง `cities` — metadata ทั้งหมดเข้าถึงได้ (ปานกลาง)

ทั้ง 6 เมืองถูก return พร้อม id, slug, name_th, name_en, blurb (เรื่องราว), map_x, map_y, accent_hex, is_active

---

### C10. ฉีด JWT user_metadata — บทบาท Admin แบบสวย (ปานกลาง)

**เรียก:** `PUT /auth/v1/user` ด้วย `{"data": {"role": "admin", "is_admin": true, "is_staff": true}}`

**ผลลัพธ์:** ฉีด admin flags เข้า JWT user_metadata สำเร็จ:
```json
"user_metadata": {
  "is_admin": true,
  "is_staff": true,
  "role": "admin"
}
```

**ผลกระทบ:**
- JWT ตอนนี้มี `role: "admin"` ใน user_metadata — **แต่** แอปตรวจ `profiles.role` จากฐานข้อมูล ไม่ใช่ JWT claim
- ถ้าแอปเปลี่ยนมาตรวจ JWT วันไหน จะกลายเป็นช่องโหว่ยกระดับสิทธิ์ทันที
- endpoint `/auth/v1/user` อนุญาตแก้ user_metadata อะไรก็ได้โดยไม่มี validation
- ตอนนี้เป็นแค่ cosmetic — เส้นทาง admin ทั้งหมดยังแสดง "ไม่มีสิทธิ์"

**วิธีแก้:** ตรวจสอบ user_metadata writes ฝั่ง server — ใช้ `app_metadata` (ต้อง service_role key) สำหรับ role claims แทน

---

### C11. ข้อมูล RSC หลุด — ชื่อนักเรียนและตำแหน่ง Chain ทั้งหมด (ปานกลาง)

**สิ่งที่พบ:** RSC responses สำหรับ `/senior/puzzles` มีรายชื่อ chain ครบทั้ง 6 เมือง:
- ชื่อเต็มภาษาไทยของทุกนักเรียน
- seat_index (ตำแหน่ง chain)
- สถานะ chain ปัจจุบัน
- สถานะการเก็บ golden key ต่อเมือง

**ตัวอย่าง (เมือง 1 — Erawan):**
```
Chain: #1 ศราวิณณ์ (active) → #2 พิชญพงษ์ → #3 ธัชนนท์ → #4 ศุภโชติ (เรา) → #5 ณัฏฐกฤต → #6 ณัฐชนน
```

**ผลกระทบ:** ความยุติธรรมของเกม puzzle chain ขึ้นอยู่กับ partial anonymity — RSC ส่งข้อมูลมากกว่าที่ UI แสดง

---

### C12. สมัครรับ Realtime Changes ได้ทุกตาราง (ปานกลาง)

สมัครรับ PostgreSQL changes บนทุกตาราง: puzzles, submissions, profiles, cities, golden_keys, puzzle_chains, announcements, * (wildcard) — ทั้งหมดสำเร็จ

**ผลกระทบ:** เมื่อมีคนส่งคำตอบ puzzle หรือได้รหัส, INSERT/UPDATE event อาจถูกดักจับแบบ real-time

---

### C13. ยืนยัน Admin Account ผ่าน Password Reset (ต่ำ)

- `admin@ictp-camp.co.th` → **มีอยู่** (admin UUID: `15135102-69fd-4d31-9ff8-d9f6f59d4a63`)
- ลอง brute force 40+ รหัสผ่านกับ 5 บัญชี staff/admin — ล้มเหลวทั้งหมด ไม่มี account lockout

---

## ช่องโหว่ระดับปานกลาง/ต่ำ

### 14. Auth Settings เข้าถึงได้สาธารณะ (ต่ำ)

ยืนยัน: เปิดแค่ email auth, ปิดการสมัคร (`disable_signup: true`)

---

### 15. JWKS Public Key หลุด (ต่ำ)

ES256 public key เข้าถึงได้ — มาตรฐานสำหรับ JWT verification

---

### 16. Error Messages เปิดเผยการมีอยู่ของ Table/Column (ต่ำ)

PostgreSQL error messages หลุด schema ฐานข้อมูล — สำรวจตาราง/column ได้จาก 403 vs 404 vs 400

---

### 17. SSRF ผ่าน Image Optimizer — ถูกบล็อคอย่างถูกต้อง (ข้อมูล)

ทุก SSRF payloads return 400 Bad Request ไม่สามารถโจมตีได้

---

### 18. robots.txt เปิดเผยโครงสร้าง Route (ต่ำ)

เปิดเผย: `/admin`, `/decrypt`, `/senior`, `/camp`, `/preview`, `/login`

---

### 19. ไม่มี API Routes (ข้อมูล)

แอปใช้เฉพาะ React Server Actions และ RSC — ไม่มี REST API routes เลย ลดพื้นที่โจมตีมาก

---

## สิ่งที่ปลอดภัยดี

| การควบคุม | การ implement | ระดับ |
|----------|-------------|-------|
| GRANT ระดับตาราง | puzzles, secrets, golden_keys, puzzle_chains, submissions ล็อคทั้งหมด | ดีเยี่ยม |
| SECURITY DEFINER functions | เนื้อหา puzzle ส่งผ่าน `get_my_puzzle`/`get_my_senior` เท่านั้น — ฉีด parameter ไม่ได้ | ดีเยี่ยม |
| RLS บน profiles | แถวตัวเองเท่านั้น, UPDATE/INSERT ถูกบล็อคที่ GRANT level | ดีเยี่ยม |
| RLS บน announcements writes | INSERT/UPDATE/DELETE ถูกบล็อคทั้งหมด | ดี |
| Storage upload RLS | path traversal ไปโฟลเดอร์ผู้ใช้อื่นถูกบล็อค (403) | ดีเยี่ยม |
| Puzzle chain ฝั่ง server | ไม่สามารถ bypass จาก client — บังคับ chain ใน SECURITY DEFINER, ไม่มีเนื้อหาใน RSC | ดีเยี่ยม |
| CSP | Nonce-based เข้มงวด, `strict-dynamic`, rotate nonce ทุก request | ดีเยี่ยม |
| HSTS | max-age 2 ปี พร้อม includeSubDomains และ preload | ดีเยี่ยม |
| ป้องกัน SSRF | Image optimizer ตรวจสอบ URL อย่างถูกต้อง | ดี |
| ปิดการสมัคร | ไม่สามารถสร้างบัญชีใหม่ | ดี |
| Source maps | ไม่ได้ deploy ใน production (เข้าถึงได้ 0 ไฟล์) | ดีเยี่ยม |
| ป้องกัน Frame | `X-Frame-Options: DENY` + CSP `frame-ancestors: 'none'` | ดีเยี่ยม |
| ไม่มี API routes | 100% server actions — ไม่มี action IDs ที่ค้นพบได้, ไม่มี REST endpoints | ดีเยี่ยม |
| Auth ของ Admin | ตรวจ role ที่ฐานข้อมูล (profiles.role), ไม่ใช่ JWT claims — immune ต่อ metadata injection | ดีเยี่ยม |
| รหัสผ่านแข็งแรง | รหัสผ่าน admin รอดจากการลอง 40+ ครั้ง | ดี |
| แยก role | app_metadata (ต้อง service_role เท่านั้น) vs user_metadata (ผู้ใช้แก้ได้) แยกอย่างถูกต้อง | ดี |

---

## สรุปการโจมตี

### สิ่งที่ทำได้ (ข้อมูลหลุด & ช่องโหว่ที่ยืนยันแล้ว)
1. ดึง Supabase publishable key จาก client-side JS (format ใหม่ `sb_publishable_`)
2. สำรวจพบ 9 ตาราง/views ฐานข้อมูลและระดับการเข้าถึง
3. **เข้าถึง `city_progress` view** — สถานะเกมทั้ง 6 เมือง (จำนวนที่แก้, UUID ผู้แก้, ตำแหน่ง chain)
4. **เข้าถึง `cities` table** — ทั้ง 6 เมืองพร้อม metadata ครบ
5. เข้าถึงข้อมูลประกาศและไฟล์แนบทั้งหมดโดยไม่ต้อง login
6. ฉีด broadcast messages ผ่าน Supabase Realtime WebSocket
7. **สมัครรับ Realtime changes บนทุกตาราง** รวมถึง puzzles, golden_keys, puzzle_chains
8. **ฉีด admin role เข้า JWT user_metadata** (`is_admin: true, is_staff: true, role: "admin"`)
9. อ่านข้อมูล profile ตัวเองผ่าน Supabase API ตรง (ข้ามแอป)
10. ยืนยันสถานะ puzzle chain และระบุชื่อเต็มนักเรียนที่บล็อค
11. **ดึงชื่อนักเรียนและตำแหน่ง chain ทั้งหมด** จากข้อมูล RSC flight ของทั้ง 6 เมือง
12. ยืนยัน admin account: `admin@ictp-camp.co.th` (UUID: `15135102-69fd-4d31-9ff8-d9f6f59d4a63`)
13. ยืนยันโครงสร้าง storage bucket และ list ไฟล์ได้

### สิ่งที่ทำไม่ได้ (โจมตี 10+ ขั้นตอน)
1. **อ่านเนื้อหา puzzle, คำตอบ, secrets หรือข้อมูล golden key ใดๆ เลย** — SECURITY DEFINER + GRANT บล็อคทั้งหมด
2. **ยกระดับบทบาทจาก student เป็น admin** — profiles.role UPDATE ถูกบล็อคที่ GRANT level, JWT injection เป็นแค่ cosmetic
3. แก้ไขข้อมูลในฐานข้อมูล — GRANT/RLS บล็อค writes ทั้งหมด
4. เข้าถึง profile ผู้ใช้อื่น — RLS จำกัดแค่แถวตัวเอง, IDOR ทั้งหมดล้มเหลว (403)
5. อัปโหลดไฟล์ไปเส้นทาง storage ของผู้ใช้อื่น
6. ข้ามลำดับ puzzle chain
7. เข้าถึง source maps (ไม่ได้ deploy) หรือโค้ดฝั่ง server
8. โจมตี SSRF
9. สร้างบัญชีใหม่ (ปิดการสมัคร)
10. เข้าถึงเส้นทางหรือฟังก์ชัน admin — ตรวจ role ที่ DB ไม่ใช่ JWT
11. **Brute force รหัสผ่าน admin** — ลอง 40+ ครั้งกับ 5 บัญชี ล้มเหลวทั้งหมด ไม่มี lockout
12. **หา server action IDs** — ค้นไม่เจอในทุก JS chunk หรือ RSC response
13. **ค้นพบ RPC functions ที่ไม่มีเอกสาร** — ลองชื่อ 60+ ตัว มีแค่ 2 ตัว
14. **ปลอม RPC parameters** — `get_my_puzzle` ไม่รับ parameter ใดๆ, return แค่ข้อมูลตัวเอง
15. **เข้าถึง schema ตาราง submissions** — columns ถูกซ่อนจาก PostgREST schema cache

---

## ลำดับการแก้ไข

| ลำดับ | ช่องโหว่ | ความรุนแรง | การดำเนินการ |
|-------|---------|-----------|-------------|
| **P0** | `city_progress` view เปิดให้ทุกคน | สูง | เพิ่ม RLS หรือ revoke GRANT — สถานะเกมไม่ควรให้นักเรียนทุกคนอ่านได้ |
| **P1** | ฉีด broadcast + สมัครทุกตาราง | สูง | เพิ่ม auth policies ระดับ channel; จำกัด subscription เฉพาะ channel ที่แอปใช้ |
| **P2** | ฉีด JWT user_metadata | ปานกลาง | ตรวจสอบ/sanitize user_metadata writes; บล็อค field role/admin/staff |
| **P3** | Publishable key ใน client JS | สูง | ย้ายการเรียก Supabase ทั้งหมดไปฝั่ง server; ลบ client library bundle |
| **P4** | `get_my_puzzle` หลุดชื่อ | ปานกลาง | ลบชื่อเต็ม `waiting_on` ออกจาก response |
| **P5** | RSC หลุดชื่อ/chain นักเรียน | ปานกลาง | ส่งเฉพาะ chain ของเมืองผู้ใช้ใน RSC ไม่ใช่ทั้ง 6 เมือง |
| **P6** | Bucket ประกาศสาธารณะ | ปานกลาง | เปลี่ยนเป็นต้อง login ถ้าไฟล์แนบมีข้อมูลสำคัญ |
| **P7** | Announcements ไม่มี RLS บน SELECT | ปานกลาง | เพิ่ม RLS policy บังคับ authentication |
| **P8** | ไม่มี account lockout | ปานกลาง | implement rate limiting / lockout หลังลองผิด N ครั้ง |
| **P9** | Error message หลุด schema | ต่ำ | ตั้งค่า PostgREST ให้ return generic errors |
| **P10** | `cities` table เปิดให้ทุกคน | ต่ำ | เพิ่ม RLS ถ้า metadata เมืองควรถูกจำกัด |

---

## โครงสร้าง JWT (หลังฉีด Metadata)

```json
{
  "alg": "ES256",
  "kid": "537de384-1dc2-49f5-9c9f-ea04f7ce476d",
  "iss": "https://lpojluoriaguvzfsqfbz.supabase.co/auth/v1",
  "sub": "36205a45-e9e2-41a8-91af-c760c15ee0db",
  "aud": "authenticated",
  "role": "authenticated",
  "email": "s44624@bj.ac.th",
  "app_metadata": {"provider": "email", "providers": ["email"]},
  "user_metadata": {
    "display_name": "ศุภโชติ ศรีประทิน",
    "email_verified": true,
    "is_admin": true,
    "is_staff": true,
    "nickname": "เจ",
    "role": "admin"
  },
  "aal": "aal1"
}
```

**หมายเหตุ:** `user_metadata.role = "admin"` และ `is_admin = true` ถูกฉีดสำเร็จผ่าน `PUT /auth/v1/user` แอปเพิกเฉยถูกต้อง — authorization ตรวจ `profiles.role` ในฐานข้อมูล ซึ่งแก้ไม่ได้โดยไม่มี service_role privileges

---

## เวกเตอร์โจมตีที่หมดแล้ว

| เวกเตอร์ | จำนวนครั้ง | ผลลัพธ์ |
|---------|----------|---------|
| SELECT ตารางตรง | ทั้ง 9 ตาราง | 4 เข้าถึงได้, 5 ล็อคทั้งหมด |
| ค้นหา RPC function | 60+ ชื่อ | มีแค่ `get_my_puzzle` กับ `get_my_senior` |
| ฉีด RPC parameter | หลาย combo | ไม่รับ parameter — ใช้ JWT sub ภายใน |
| ฉีด JWT metadata | role, is_admin, is_staff | ฉีดสำเร็จ — cosmetic เท่านั้น, แอปตรวจ DB |
| UPDATE role ใน profile | PATCH, UPSERT, ตรง | ทั้งหมดถูกบล็อคที่ GRANT level (403) |
| Brute force admin | 40+ รหัสผ่าน × 5 บัญชี | ล้มเหลวทั้งหมด, ไม่มี lockout |
| หา source maps | ทุก JS chunk + .map | 0 source maps ใน production |
| Next.js internals | .env, data routes, build manifest | ไม่มีข้อมูลสำคัญหลุด |
| IDOR ผ่าน user IDs | 5 UUIDs ที่รู้ | อ่าน profile ทั้งหมดถูกบล็อค (403) |
| GraphQL | pg_graphql extension | ไม่ได้เปิด — dead end |
| Storage signed URL forgery | 6 bucket/path combos | ทั้งหมดถูกปฏิเสธหรือว่าง |
| หา server actions | สแกนทุก JS chunk | 0 action IDs ค้นพบ |
| RSC data mining | 6 หน้า RSC responses | มีแค่ status data, ไม่มีเนื้อหา puzzle |
| ดักจับ Realtime data | สมัครทุกตาราง | subscription ทำงาน, ไม่มี data ระหว่างทดสอบ |
| Schema ตาราง submissions | 30+ ชื่อ column | ทั้งหมดถูกซ่อนจาก PostgREST schema cache |

---

## การประเมินภาพรวม

**ระดับความปลอดภัย: A- (แข็งแรง)**

แอปนี้แสดงให้เห็นความปลอดภัยระดับฐานข้อมูลที่แข็งแรงมาก แนวทาง defense-in-depth — รวม GRANT restrictions ระดับตาราง, Row-Level Security, SECURITY DEFINER functions และ server-side rendering เท่านั้น — ป้องกันการดึงข้อมูลและยกระดับสิทธิ์ทั้งหมดที่พยายาม

**คำตอบ puzzle และ admin access เข้าไม่ถึงอย่างแท้จริง** หากไม่ได้ Supabase service_role key หรือรหัสผ่าน admin เวกเตอร์ทฤษฎีที่เหลือ:
1. หา service_role key ผ่านช่องโหว่แยก (ซอร์สโค้ดฝั่ง server หลุด, Vercel env exposure)
2. Social engineering admin ให้บอกรหัสผ่าน
3. โจมตี zero-day ใน Next.js หรือ Supabase ที่ยังไม่มีใครค้นพบ
4. เข้าถึง infrastructure ของ Vercel/Supabase โดยตรง

จุดอ่อนหลักเป็น information leaks (city_progress, ชื่อนักเรียนใน RSC, JWT metadata injection) และ Realtime broadcast injection — ไม่มีอันไหนทำให้ข้อมูลเกมหลักถูก compromise

---

## เครื่องมือที่ใช้

- Playwright (headless Chromium) สำหรับ browser automation และดักจับ network
- ทดสอบ Supabase REST/Auth/Storage/Realtime API โดยตรง (10+ ขั้นตอน)
- วิเคราะห์ JavaScript chunk ฝั่ง client (หา source maps, ดึง action IDs)
- สำรวจ column/table/view ผ่าน PostgREST ด้วย error-based inference
- ทดสอบ WebSocket protocol (ฉีด broadcast, สมัครรับ channel)
- แยกวิเคราะห์ข้อมูล RSC flight data
- วิเคราะห์ JWT และฉีด metadata
- สำรวจ admin account ผ่าน password reset
- Brute force admin (40+ รูปแบบรหัสผ่าน)
