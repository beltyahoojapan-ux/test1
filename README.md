# BankBangna v14 — ระบบรับประมูล/ฝากซื้อสินค้าจากญี่ปุ่น

ระบบเดียวครบตั้งแต่ลูกค้าวางลิงก์ → คำนวณราคาถึงมือที่ไทย → ส่งคำสั่งซื้อ →
เจ้าหน้าที่ตรวจสอบและติดตามสถานะจนส่งถึงลูกค้า

```
ลูกค้า                     เจ้าหน้าที่
  │ วางลิงก์                   │
  ├─ ดึงข้อมูล + เดาประเภท      │
  ├─ คำนวณ รอบ1 + รอบ2         │
  ├─ ใบเสนอราคา (ยืน 24 ชม.)   │
  └─ ส่งคำสั่งซื้อ ──────────────┤ ตรวจสอบ ยืนยันประเภท/ค่าส่งจริง
                               ├─ เปลี่ยนสถานะตามขั้นตอน
                               └─ ระบบเก็บสถิติความแม่นของกฎ/การประเมิน
```

---

## 1. โครงสร้างโปรเจกต์

```
bankbangna/
├── backend/
│   ├── main.py          API ทั้งหมด (FastAPI) + เสิร์ฟหน้าเว็บ
│   ├── config.py        ค่าตั้งระบบทั้งหมด อ่านจาก .env
│   ├── db.py            connection pool + สร้างตาราง + ข้อมูลตั้งต้น
│   ├── schema.sql       โครงสร้างฐานข้อมูล (รันซ้ำได้ปลอดภัย)
│   ├── schemas.py       ตรวจความถูกต้องของข้อมูลขาเข้า (Pydantic)
│   ├── pricing.py       สูตรราคา — ฟังก์ชันบริสุทธิ์ ทดสอบได้โดยไม่ต้องมี DB
│   ├── rule_engine.py   ตรรกะกฎ if-else — ฟังก์ชันบริสุทธิ์
│   ├── rules_manager.py จัดการกฎ + version + rollback + audit
│   ├── classifier.py    เดาประเภทสินค้าจากชื่อ
│   ├── scraper.py       ดึงข้อมูลจาก 9 เว็บญี่ปุ่น + กัน SSRF
│   ├── analytics.py     สถิติ 3 มิติ + ข้อเสนอแนะ
│   └── security.py      รหัสผ่าน / JWT / สิทธิ์ / rate limit / security headers
├── frontend/
│   ├── index.html       หน้าลูกค้า
│   ├── admin.html       หลังบ้าน
│   └── assets/          CSS, JS, ฟอนต์ (self-host ไม่พึ่ง CDN)
├── tests/               ชุดทดสอบ 45 ข้อ
├── deploy/
│   ├── Caddyfile        reverse proxy + HTTPS อัตโนมัติ
│   ├── backup.sh        สำรองฐานข้อมูล
│   └── smoke_test.sh    ตรวจระบบหลัง deploy
├── Dockerfile
├── docker-compose.yml
└── .env.example
```

**หลักการแยกชั้น:** `pricing.py` และ `rule_engine.py` ไม่รู้จักฐานข้อมูลและไม่รู้จัก HTTP
จึงทดสอบสูตรเงินได้ด้วย `pytest` ในไม่กี่มิลลิวินาที และแก้สูตรได้โดยไม่กระทบส่วนอื่น

---

## 2. รันบนเครื่องตัวเอง

```bash
# ต้องมี PostgreSQL 14+ อยู่แล้ว
createdb bankbangna

cp .env.example .env     # แก้ DATABASE_URL, JWT_SECRET, BOOTSTRAP_ADMIN_*
pip install -r requirements-dev.txt
uvicorn backend.main:app --reload --port 8000
```

เปิด `http://localhost:8000` (ลูกค้า) และ `http://localhost:8000/admin.html` (หลังบ้าน)
ตารางฐานข้อมูลถูกสร้างอัตโนมัติตอนเริ่มระบบ

```bash
pytest tests -q                       # ทดสอบสูตรราคาและกฎ
sh deploy/smoke_test.sh               # ทดสอบ API + ความปลอดภัย
```

---

## 3. ขึ้น VPS (Docker)

ต้องมี: VPS Ubuntu 22.04+ RAM 2 GB ขึ้นไป, โดเมนที่ชี้ A record มาที่ IP ของ VPS แล้ว

```bash
# --- บน VPS ---
curl -fsSL https://get.docker.com | sh

git clone <repo> /opt/bankbangna && cd /opt/bankbangna
cp .env.example .env
nano .env
#   DOMAIN=app.bankbangna.com
#   POSTGRES_PASSWORD=$(openssl rand -base64 24)
#   JWT_SECRET=$(openssl rand -hex 32)
#   BOOTSTRAP_ADMIN_EMAIL=you@yourdomain.com
#   BOOTSTRAP_ADMIN_PASSWORD=<รหัสผ่านที่แข็งแรง>

docker compose up -d --build
docker compose logs -f api        # ดูว่าขึ้นเรียบร้อย
```

Caddy ขอใบรับรอง HTTPS จาก Let's Encrypt ให้อัตโนมัติภายในไม่กี่วินาที จากนั้น:

```bash
sh deploy/smoke_test.sh https://app.bankbangna.com
```

**เข้าใช้ครั้งแรก:** ล็อกอินที่ `https://โดเมน/admin.html` ด้วยบัญชี bootstrap →
ไปแท็บ **อัตราแลกเปลี่ยน** ตั้งเรตขายจริง → แท็บ **กฎตรวจสอบ** ปรับกฎตามหน้างาน
→ ลบ `BOOTSTRAP_ADMIN_*` ออกจาก `.env`

**ตั้งสำรองข้อมูลรายวัน:**
```bash
crontab -e
0 3 * * * cd /opt/bankbangna && sh deploy/backup.sh >> /var/log/bb-backup.log 2>&1
```

**ปิดพอร์ตที่ไม่ใช้:**
```bash
ufw allow 22 && ufw allow 80 && ufw allow 443 && ufw enable
```
(Postgres ไม่ได้เปิดออกนอกเครื่องอยู่แล้ว — เข้าถึงได้เฉพาะจาก container)

**อัปเดตเวอร์ชันใหม่:**
```bash
cd /opt/bankbangna && git pull && docker compose up -d --build
```

---

## 4. สูตรราคา

### รอบ 1 — จ่ายที่ญี่ปุ่น
```
ค่าบริการ   = ราคาสินค้า × 15%
รวมเงินเยน  = ราคาสินค้า + ค่าบริการ + ค่าส่งในญี่ปุ่น + ค่าแพ็กที่โอซาก้า (500 เยน ทุกชิ้น)
รอบ 1 (บาท) = รวมเงินเยน × เรตขาย
```

### รอบ 2 — ค่านำเข้าไทย (บาท)

| ประเภท | LEVEL_1 | LEVEL_2 | LEVEL_3 | HEAD_AFF |
|---|---:|---:|---:|---:|
| สินค้าทั่วไป | 300 | 200 | 100 | 300 |
| ล้อ (ชุด 4 วง) | 4,500 | 4,500 | 4,000 | 4,000 |
| ยาง (เส้น) | 1,200 | 1,200 | 1,000 | 1,000 |
| เครื่องยนต์ | 10,000 | 10,000 | 10,000 | 10,000 |

**Off-road** (ล้อ/ยาง): `1,700 × จำนวนวง` แทนตารางข้างบน

> จำนวนวงตั้งต้น: ล้อ = 4, ยาง = 1 — v13 ตั้ง 4 ทุกกรณี ทำให้ยาง Off-road 1 เส้น
> ถูกคิด 6,800 บาทแทน 1,700 บาท

ยอดรวม = รอบ 1 + รอบ 2 · ทุกตัวเลขคิดด้วย `Decimal` ไม่ใช่ `float`

---

## 5. กฎตรวจสอบสินค้า

กฎหนึ่งข้อ = เงื่อนไขหลายข้อ (ต้องตรง **ทุกข้อ**) + ผลลัพธ์หนึ่งอย่าง

| ผลลัพธ์ | ความหมาย |
|---|---|
| `WARN` | แสดงคำเตือน ลูกค้ายังสั่งซื้อได้ |
| `REQUIRE_REVIEW` | สั่งซื้อได้ แต่ติดธงให้เจ้าหน้าที่ตรวจก่อน |
| `BLOCK` | ห้ามสั่งซื้อ และหยุดประเมินกฎที่เหลือ |

ช่องที่ใช้ได้: `item_class` `customer_level` `is_offroad` `wheel_qty` `item_price_jpy`
`domestic_shipping_jpy` `total_jpy` `grand_total_thb` `size_tier` `source_site` `title`

เงื่อนไข: `eq` `ne` `in` `not_in` `gt` `gte` `lt` `lte` `contains` `contains_any`

กฎทุกข้อมีประวัติเวอร์ชันและย้อนกลับได้ · ทดลองกฎใน Sandbox ก่อนเปิดใช้จริงได้เสมอ

---

## 6. สถานะออเดอร์

```
รอตรวจสอบ ─┬─→ รอชำระเงิน ─┬─→ กำลังประมูล ─┬─→ ซื้อสำเร็จ ─→ รอแพ็คที่โอซาก้า
           │               │                └─→ ประมูลไม่ได้       │
           └─→ ยกเลิก      └─→ ยกเลิก                              ↓
                                       ส่งให้ลูกค้าแล้ว ←─ ถึงไทยแล้ว ←─ กำลังเดินทาง
```

ระบบปฏิเสธการข้ามขั้น (เช่น รอชำระเงิน → ส่งแล้ว) ผู้ดูแลระบบข้ามได้ด้วย `force`
ทุกการเปลี่ยนสถานะถูกบันทึกพร้อมผู้ทำและเวลา

---

## 7. ความปลอดภัย

| เรื่อง | วิธีจัดการ |
|---|---|
| SSRF (ลิงก์หลอกให้ยิงเข้าเครือข่ายภายใน) | เทียบโดเมนแบบตรงตัว, ปฏิเสธ IP ภายใน, ตรวจทุก redirect, จำกัดขนาด/เวลา |
| XSS (ชื่อสินค้าญี่ปุ่นแทรกสคริปต์) | ไม่ใช้ `innerHTML` กับข้อมูลใด ๆ + CSP `script-src 'self'` |
| SQL injection | ใช้ parameterized query ทุกจุด ชื่อคอลัมน์มาจาก whitelist เท่านั้น |
| ขโมย session | cookie `HttpOnly` + `Secure` + `SameSite=strict` (JavaScript อ่านไม่ได้) |
| CSRF | ทุก request ที่แก้ข้อมูลต้องมี header `X-Requested-With` |
| เดารหัสผ่าน | bcrypt + จำกัด 10 ครั้ง/15 นาที/IP และ 20 ครั้ง/ชม./อีเมล |
| ยิง API รัว | จำกัดการดึงลิงก์ 30 ครั้ง/ชม./IP (ปรับได้ใน `.env`) |
| ลูกค้าแก้ราคาตัวเอง | ระดับราคามาจากบัญชีในฐานข้อมูล ไม่รับจาก client |
| ใบเสนอราคาเก่าถูกนำมาใช้ | ใบเสนอราคาหมดอายุใน 24 ชม. และผูกกับเจ้าของ |
| ไม่รู้ว่าใครแก้อะไร | `audit_logs` เก็บทุกการแก้กฎ/ออเดอร์/ผู้ใช้/เรต พร้อม IP |

**สิ่งที่ยังควรทำเพิ่มก่อนรับเงินจริง:** ระบบชำระเงินและการยืนยันสลิป,
แจ้งเตือนลูกค้าทาง LINE/อีเมลเมื่อสถานะเปลี่ยน, และการตั้ง `Retry-After` ร่วมกับ
Redis หากขยายเป็นหลายเครื่อง (ตอนนี้ rate limit เก็บในหน่วยความจำของ process เดียว)
