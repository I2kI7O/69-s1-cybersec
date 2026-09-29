# Cyber Security

## My Information

- Wachirawit Kantho
- 012-6

## Course Expectations

- Learn and understand the fundamentals of Cybersecurity more clearly.
- Practice using Git and GitHub for project and task management.
- Learn vulnerability analysis and cybersecurity threat prevention techniques.
- Apply the knowledge gained in real-world situations and future careers.

---

## Project Structure

```
69-s1-cybersec/
├── .env                # local secrets (gitignored - ห้าม commit)
├── .env.simple         # template สำหรับคนอื่น (commit ได้ ห้ามมี secret จริง)
├── docker-compose.yaml # db (postgres) + admin (pgAdmin) + app (Strapi)
├── api.http            # ชุดทดสอบ REST API ทุก section (REST Client / VS Code)
├── app/                # Strapi application source -> mount เข้า /opt/app/src
├── scripts/            # สคริปต์ช่วยตั้งค่า (เปิด permission)
├── data/               # volume ของ Postgres (gitignored)
├── admin_data/         # volume ของ pgAdmin (gitignored)
└── pgadmin/            # volume ของ pgAdmin (gitignored)
```

### ทำไมต้องมีโฟลเดอร์ `app/`

Docker image `prawee/strapi` มาพร้อม content type Student / Subject / Teacher / Mapping
อยู่ในตัว image อยู่แล้ว ทำให้ **schema ไม่ถูกเก็บใน git** ใครก็ตามที่ clone repo
จะไม่เห็นโครงสร้างข้อมูล และแก้โค้ดฝั่ง server ไม่ได้

โฟลเดอร์ `app/` จึงเป็นสำเนา `/opt/app/src` ของ image มาเก็บไว้ใน repo
(ไฟล์ content type ถูก copy มาแบบไม่แก้ไขอะไร) แล้วให้ `docker-compose.yaml`
mount เข้าไปที่ `/opt/app/src` ผลคือ

- schema.json / controller / route / service ของทุก content type อยู่ใน git
- เพิ่ม middleware ของตัวเองได้ โดยไม่ต้อง fork image
- ถ้าเผลอแก้ content type ผ่านหน้า Content-Type Builder
  **schema.json ใน `app/api/` จะไม่ถูกอัปเดตตาม** ต้องแก้ในรีโปเอง

---

## Getting Started

```bash
cp .env.simple .env      # แก้ค่า secret ให้เป็นของตัวเองก่อน
docker compose up -d
```

| Service  | URL                            | หมายเหตุ                          |
| -------- | ------------------------------ | --------------------------------- |
| Strapi   | http://localhost:9091          | REST API + Admin Panel (`/admin`) |
| pgAdmin  | http://localhost:8081          | user/password จาก `.env`          |
| Postgres | `localhost:54327`              | user/db = `wachi` (default)       |

Port ทั้งหมดปรับได้จาก `.env` (`APP_PORT`, `PGADMIN_DEFAULT_PORT`, `POSTGRES_PORT`)

---

## ข้อควรระวังเรื่อง Environment (env)

1. **`.env` อยู่ใน `.gitignore` แล้ว ห้าม commit เด็ดขาด**
   ค่าที่อยู่ในนั้นคือรหัสผ่าน Postgres, pgAdmin, JWT secret ของ Strapi
   ถ้า commit ไปแล้วต้อง **rotate ค่าใหม่ทั้งหมด** ไม่ใช่แค่ git rm
2. **`.env.simple` คือไฟล์เดียวที่ commit ได้** ต้องเป็น placeholder เท่านั้น
   (`change_me_*`) ห้ามใส่ค่าจริงของตัวเอง
3. **secret ทุกตัวต้องส่งผ่าน `environment:` ของ container เท่านั้น**
   ห้าม hardcode ใน `app/` หรือใส่ลง `schema.json`
4. **`docker-compose.yaml` มีค่า default เป็น `change_me` / `my...Secret`**
   เป็นแค่ตัวอย่างให้ compose ทำงานได้โดยไม่มี `.env` เท่านั้น
   ถ้า deploy จริงต้องตั้งค่าให้แข็งแรงผ่าน `.env` หรือ Docker secrets
5. **ถ้าเผย JWT secret หลุดไป** ให้เปลี่ยน `JWT_SECRET` / `ADMIN_JWT_SECRET`
   แล้ว `docker compose up -d --force-recreate app` token เก่าจะใช้ไม่ได้ทันที
6. **ค่าที่เพิ่มใหม่ในส่วนนี้** คือ `PROTOTYPE_GUARD_ENABLED` และ
   `PROTOTYPE_GUARD_MAX_DEPTH` ซึ่งควรอยู่ใน env เสมอ ไม่ใช่ hardcode ในโค้ด

---

## REST API

ชุด request ทั้งหมดอยู่ใน `api.http` (เปิดด้วย REST Client ของ VS Code หรือ IntelliJ)

### 1. ADMIN OPERATIONS

Strapi Admin Panel — Login / Register / Forgot Password / Reset Password / Profile

### 2. USER OPERATIONS

Users & Permissions plugin — Login / Register / Forgot Password / Reset Password / Profile

### 3. CONTENT OPERATIONS

Content type ทั้ง 3 ตัวมี CRUD ครบชุด (Create / List All / List with ID / Update / Delete)
รวม 15 endpoint

| Content type | ใน schema.json                   | endpoint หลัก             |
| ------------ | -------------------------------- | ------------------------ |
| `student`    | `name`, `mobile`, `cardId`       | `/api/students`          |
| `subject`    | `name`                           | `/api/subjects`          |
| `teacher`    | `name`, `mappings` (relation)    | `/api/teachers`          |

#### 3.1 เปิด permission ก่อนทดสอบ

Content type ที่สร้างใหม่จะ **private by default** ทุก request จะได้ 403 Forbidden
จนกว่าจะเปิด permission ใน Users & Permissions

**วิธีที่ 1 — หน้าเว็บ (ปกติที่สุด)**

1. เข้า `http://localhost:9091/admin`
2. Settings > Users & Permissions
3. เลือก Public Role (ถ้าอยากให้เรียกได้โดยไม่ต้อง login) และ/หรือ
   Authenticated Role
4. เปิดสวิตช์ `create`, `find`, `findOne`, `update`, `delete`
   ของ Student, Subject, Teacher
5. Save

**วิธีที่ 2 — สคริปต์ (ทำซ้ำได้ เหมือนกดสวิตช์ในหน้าเว็บ)**

```bash
docker compose exec -T db psql -U wachi -d wachi < scripts/enable-content-api-permissions.sql
```

> ⚠️ การเปิด `create` / `update` / `delete` ให้ Public Role เท่ากับเปิดให้
> คนที่ไม่มี token เขียนข้อมูลได้ทั้งหมด เหมาะกับการสาธิตเท่านั้น
> ระบบจริงควรเปิดเฉพาะ `find` / `findOne` ให้ Public
> และให้ที่เหลืออยู่กับ Authenticated role

#### 3.2 เรื่อง body ของ Strapi v4

- `POST` / `PUT` ต้องห่อ payload ด้วยคีย์ `data` เสมอ
  ไม่งั้นได้ `400 Missing "data" payload in the request body`
- ฟิลด์ที่ไม่ประกาศใน `schema.json` จะถูกตัดทิ้งเงียบ ๆ ไม่ error
- `Student` มี `beforeCreate` lifecycle ที่เขียนทับ `mobile` ด้วย `md5`
  ค่า mobile ที่ได้กลับมาจึงยาว 32 ตัวอักษร ไม่ใช่ 10 ตัวตามที่ส่งเข้าไป
  (schema ประกาศ `maxLength: 10` แต่การตรวจความยาวเกิด**ก่อน** lifecycle ทำงาน
  ค่าที่ถูกเขียนทับจึงไม่ผ่านการตรวจซ้ำ)
- `md5` ไม่ใช่การเข้ารหัสที่เหมาะกับข้อมูลส่วนบุคคล
  มันถอดกลับได้เร็วและไม่มี salt
- อย่า hardcode `id` ในไฟล์ทดสอบ ให้ใช้
  `{{createStudent.response.body.data.id}}` แทน

---

## Prototype Pollution

### ผลการทดสอบกับ Strapi 4.16.2 (Strapi 4.16.2 + qs 6.11.1 + Node 18)

| # | จุดที่โจมตี                    | ก่อนใส่ middleware | หลังใส่ middleware |
| - | ----------------------------- | ------------------ | ------------------ |
| 1 | `?__proto__[polluted]=yes`    | 200 (ถูก `qs` ตัด) | 400                |
| 2 | `?constructor[prototype][..]` | 200 (ถูก `qs` ตัด) | 400                |
| 3 | `?fields[0]=__proto__`        | **500**            | 400                |
| 4 | `?populate=constructor`       | **500**            | 400                |
| 5 | body `data.__proto__`         | **500**            | 400                |
| 6 | body `data.constructor`       | **500**            | 400                |
| 7 | body `__proto__` ระดับ top   | 200                | 400                |
| 8 | object ซ้อนลึกเกิน 12 ชั้น     | 500                | 400                |

สรุป

- **`qs` 6.11.1 ปิดช่องโหว่ query string ไปแล้ว** (`allowPrototypes = false`,
  แก้ CVE-2022-24999) ทำให้ `?__proto__[...]` ไม่มีผล — นับเป็นชั้นป้องกันแรกแล้ว
  guard ของเราจึงต้องดู querystring ดิบ ไม่ใช่ `ctx.query` ที่ผ่าน `qs` แล้ว
- **request body เป็นช่องโหว่จริง** เพราะ `JSON.parse` สร้าง own property
  ชื่อ `__proto__` ได้ เมื่อส่งเข้าไปจะวิ่งถึง `yup` แล้วพังเป็น unhandled
  `TypeError: field.resolve is not a function` → ตอบ 500
  ซึ่งเป็นทั้ง DoS และการรั่วของ error
- **Strapi ไม่ validate ค่าใน `fields` / `populate` ก่อนเอาไปต่อกับ SQL**
  ทำให้ error จาก PostgreSQL หลุดถึง log (ดู log จะเจอ
  `column t0.__proto__ does not exist`)

### วิธีป้องกันที่ใส่ให้

`app/middlewares/prototype-pollution-guard.js` ต่อเข้ากับ Koa ใน
`bootstrap()` ของ `app/index.js` (ลงทะเบียนหลัง `strapi::body` เพื่อให้เห็น body แล้ว
แต่ก่อนที่ router จะเรียก controller)

ตรวจ 3 อย่าง

1. **raw querystring** — decode แล้วหา `__proto__` / `constructor` / `prototype`
   ทำขั้นนี้ก่อนเพราะ `qs` ตัด key อันตรายทิ้งตั้งแต่ตอน parse
   middleware จึงไม่เห็นมันถ้าดูแค่ `ctx.query` (ต้องดูข้อความดิบด้วย)
2. **ชื่อ key และค่า** ที่เป็น `__proto__` / `constructor` / `prototype`
   ไม่ว่าจะอยู่ระดับไหน ใน query หรือ body รวมถึง key ที่ซ้อนอยู่ใน array
3. **ความลึก** ของ object ที่ซ้อนกัน เกิน `PROTOTYPE_GUARD_MAX_DEPTH` (ค่าเริ่มต้น 12)
   กัน payload ที่ทำให้ process ค้าง/ช้า

ถ้าเข้าเงื่อนไขจะตอบ `400` พร้อมบอกว่าโดนที่ `query` หรือ `body`

```json
{
  "data": null,
  "error": {
    "status": 400,
    "name": "ValidationError",
    "message": "Rejected unsafe property name (prototype pollution guard)",
    "details": { "source": "body", "key": "__proto__" }
  }
}
```

### ข้อจำกัดของ guard

- **ตรวจค่าที่เป็น string ด้วย** เพื่อกันกรณี `fields[0]=__proto__`
  ทำให้ payload ที่ค่าเท่ากับ `"constructor"` พอดีจะถูกปฏิเสธไปด้วย
  ถ้าอยากปิด ให้ตั้ง `PROTOTYPE_GUARD_ENABLED=false` ใน `.env`
- **decode querystring แค่รอบเดียว** แปลว่า `?%255F%255Fproto%255F%255F[x]=1`
  จะผ่านไปได้ แต่**ไม่เป็นช่องโหว่** เพราะ `qs` ก็ decode รอบเดียวเช่นกัน
  คีย์จึงกลายเป็นข้อความ `%5F%5Fproto%5F%5F` ที่ Strapi เพิ่งมองข้าม
  ยืนยันแล้วว่า request นี้ยังคืนข้อมูลปกติ ไม่มีการเขียนทับ `Object.prototype`
  (ถ้าอยากกันทุกกรณีให้เพิ่มรอบ decode ที่ซ้ำ แต่จะได้ false positive มากขึ้น)
- เป็น **defense in depth** ไม่ใช่การแก้ที่ต้นเหตุ ต้องอัปเดต `qs` / Strapi ต่อไปด้วย
- ไม่ได้ป้องกัน SQL injection หรือข้อผิดพลาดด้าน business logic
  ชั้นถัดไปคือการ validate ที่ controller/service

### ประเด็นความปลอดภัยอื่นที่เจอระหว่างทดสอบ

- **Mass assignment ผ่าน `id`**: `POST` ที่ส่ง `data.id` มาด้วยจะถูกใช้เป็น
  primary key จริง Strapi core API ไม่ได้กรอง `id` ออกจาก payload
- **ให้ Public role เขียนข้อมูลได้** ดูหมายเหตุในหัวข้อ 3.1
- **md5 บนเบอร์โทรศัพท์** ควรเปลี่ยนเป็น hash ที่ออกแบบมาใช้กับข้อมูลส่วนบุคคล
  (argon2 / bcrypt / scrypt) หรือเก็บเป็นข้อมูลดิบแต่เข้ารหัสที่ชั้นอื่นแทน
