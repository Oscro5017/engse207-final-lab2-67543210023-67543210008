# INDIVIDUAL_REPORT_67543210008-8.md

## ข้อมูลผู้จัดทำ

- **ชื่อ-นามสกุล:** นายณัฐพงศ์ จินะปัญญา
- **รหัสนักศึกษา:** 67543210008-8
- **กลุ่ม:** 10
- **รายวิชา:** ENGSE207 Software Architecture

---

## ขอบเขตงานที่รับผิดชอบ

รับผิดชอบงานฝั่ง Business Logic และ Frontend ทั้งหมด ได้แก่

- **Task Service** — CRUD routes, JWT middleware, logEvent helper
- **Log Service** — Internal endpoint, GET /logs, GET /stats, Admin JWT guard
- **Frontend** — Task Board UI (index.html) และ Log Dashboard (logs.html)
- **Screenshots** — ถ่ายภาพหลักฐานการทดสอบ T1–T11 ครบทุกข้อ

---

## สิ่งที่ได้ดำเนินการด้วยตนเอง

### Task Service

- เขียน `task-service/src/middleware/authMiddleware.js` สำหรับตรวจสอบ JWT token ทุก request
- เขียน `task-service/src/middleware/jwtUtils.js` สำหรับ verify token
- เขียน `task-service/src/routes/tasks.js` ครบทุก route ได้แก่ GET, POST, PUT, DELETE
- เพิ่ม `logEvent()` helper ใน tasks.js สำหรับส่ง event `TASK_CREATED` และ `TASK_DELETED` ไปยัง Log Service
- ตั้งค่า role-based access ให้ admin เห็น tasks ทั้งหมด ส่วน member เห็นแค่ของตัวเอง
- เขียน `task-service/src/db/db.js` สำหรับเชื่อมต่อ shared PostgreSQL

### Log Service

- เขียน `log-service/src/index.js` ทั้งไฟล์ ซึ่งเป็น service ใหม่ที่ไม่มีใน Week 12
- สร้าง `POST /api/logs/internal` endpoint สำหรับรับ log จาก auth-service และ task-service ภายใน Docker network โดยไม่ต้องมี JWT
- สร้าง `GET /api/logs/` พร้อม dynamic WHERE clause สำหรับ filter ตาม service, level, event
- สร้าง `GET /api/logs/stats` สำหรับ aggregate สถิติ log แยกตาม level, service และ top events
- เพิ่ม `verifyAdminJWT` middleware เพื่อจำกัดสิทธิ์การดู logs เฉพาะ admin เท่านั้น

### Frontend

- พัฒนา `frontend/index.html` จากฐาน Week 12 โดยลบ Register tab และ Users page ออก
- เพิ่มลิงก์ไปยัง Log Dashboard และปรับ API call ให้ใช้ relative URL ผ่าน Nginx
- พัฒนา `frontend/logs.html` ใหม่ทั้งหมด ประกอบด้วย login overlay, stats card, filter bar และ log table แบบ auto-refresh ทุก 5 วินาที
- เพิ่มการตรวจสอบ role ฝั่ง frontend ว่าต้องเป็น admin เท่านั้นถึงจะเข้า Log Dashboard ได้

### การทดสอบและ Screenshots

- ทดสอบ API ทุก endpoint ด้วย curl บน HTTPS
- ถ่ายภาพ screenshots T1–T11 ครบทุกข้อตามที่โจทย์กำหนด

---

## ปัญหาที่พบและวิธีการแก้ไข

### ปัญหาที่ 1: task-service/src/db/db.js เชื่อมต่อ DB ผิด host

ตอนแรก `db.js` ของ task-service ใช้ค่า default host เป็น `task-db` และ database เป็น `task_db` ซึ่งไม่มีอยู่ใน docker-compose.yml ทำให้ task-service start แล้ว connect DB ไม่ได้ ทุก request ที่ต้องใช้ DB ก็ error 500 หมด

แก้โดยเปลี่ยน default ใน Pool config ให้ตรงกับที่กำหนดใน docker-compose

```js
const pool = new Pool({
  host:     process.env.DB_HOST     || 'postgres',
  database: process.env.DB_NAME     || 'taskboard',
  user:     process.env.DB_USER     || 'admin',
  password: process.env.DB_PASSWORD || 'secret123',
});
```

บทเรียนจากปัญหานี้คือ ใน microservices ที่ใช้ shared DB ต้องแน่ใจว่าทุก service ชี้ไปที่ container name เดียวกัน และ environment variable ต้องสอดคล้องกับที่กำหนดใน docker-compose เสมอ

### ปัญหาที่ 2: Log Service ต้องแยก internal endpoint ออกจาก public endpoint

Log Service มี endpoint สองแบบที่มีสิทธิ์ต่างกัน คือ `/api/logs/internal` ที่ services อื่น call ได้โดยไม่ต้อง JWT และ `/api/logs/` ที่ต้องการ JWT ระดับ admin

ปัญหาคือถ้าเปิด `/api/logs/internal` ให้เรียกได้จากภายนอกก็จะเป็นช่องโหว่ แก้โดยให้ Nginx block path นี้จากภายนอกด้วย

```nginx
location = /api/logs/internal {
    return 403;
}
```

และใช้ `verifyAdminJWT` middleware แยกต่างหากจาก `verifyJWT` ปกติ เพื่อให้ GET /logs/ เข้าถึงได้เฉพาะ admin เท่านั้น

---

## สิ่งที่ได้เรียนรู้จากงานนี้

### เชิงเทคนิค

- **JWT flow ในทางปฏิบัติ:** เข้าใจว่า token ถูกสร้างที่ auth-service และถูกตรวจสอบที่ task-service และ log-service โดยใช้ JWT_SECRET เดียวกัน ทำให้เห็นว่า secret ต้องถูก share ผ่าน environment variable และต้องเปลี่ยนทุกครั้งก่อน deploy จริง
- **Lightweight Logging vs Full Observability:** การเก็บ log ลง PostgreSQL ผ่าน REST API เป็นวิธีที่ง่ายและเข้าใจง่าย แต่ถ้าระบบมี traffic สูงจะกลายเป็น bottleneck เพราะทุก event ต้อง HTTP call ไปหา log-service การใช้ message queue หรือ centralized logging อย่าง Loki เหมาะกว่าสำหรับ production
- **Role-based Access Control:** การแยก admin กับ member ต้องทำทั้งฝั่ง backend middleware และฝั่ง frontend เพื่อ UX ที่ดี แต่ security จริงๆ ต้องพึ่ง backend เป็นหลัก

### เชิงสถาปัตยกรรม

- **Shared Database Trade-off:** การใช้ DB เดียวช่วยให้ง่ายตอน development แต่ทำให้ services ผูกกันอยู่ที่ schema ถ้าอยากแยก service จริงๆ ต้องแยก DB ด้วย
- **Service Communication ภายใน Docker Network:** การที่ services คุยกันด้วยชื่อ service โดยตรง เช่น `http://log-service:3003` ทำให้ไม่ต้องผ่าน Nginx และลด latency รวมถึงไม่ต้องมี JWT สำหรับ internal call
- **Fire-and-forget Logging:** การที่ `logEvent()` ใช้ try-catch และไม่ throw error ออกมา ทำให้ระบบหลักไม่หยุดทำงานแม้ log-service จะล่ม เป็น pattern ที่สำคัญมากในระบบจริง

### เชิงการทำงานเป็นทีม

- การแบ่งงานตาม service boundary ทำให้ conflict น้อยมาก เพราะแต่ละคนดูแลไฟล์คนละชุด
- การ commit แยกชื่อกันเพื่อแสดง contribution ชัดเจนเป็นสิ่งสำคัญในการทำงานจริง

---

## แนวทางการพัฒนาต่อไปใน Set 2

**Task Service**
- เพิ่ม pagination สำหรับ GET /tasks/ เพราะตอนนี้ดึงมาทั้งหมดซึ่งไม่เหมาะถ้า task มีจำนวนมาก
- เพิ่ม filter task ตาม status และ priority

**Log Service**
- เปลี่ยนจาก synchronous HTTP call เป็น async message queue เพื่อลด latency ของ main service
- เพิ่ม retention policy เพื่อลบ log เก่าออกอัตโนมัติ
- พิจารณาแยก Log Service ออกเป็น database ของตัวเองเพื่อไม่ให้กระทบ main DB

**Frontend**
- เพิ่ม pagination สำหรับ Log Dashboard
- เพิ่ม chart แสดงสถิติ log แบบ visual
- ปรับ error handling ให้ชัดเจนขึ้นเมื่อ token หมดอายุ

**สถาปัตยกรรม**
- แยก database ให้แต่ละ service มี DB ของตัวเอง (database-per-service pattern)
- เพิ่ม health check endpoint และ monitoring ที่ครอบคลุมขึ้น
- Deploy บน Railway Cloud และทดสอบ HTTPS ด้วย certificate จริง

---

> รายงานฉบับนี้จัดทำเพื่อประกอบการส่งงาน Final Lab Set 1
> รายวิชา ENGSE207 Software Architecture มหาวิทยาลัยเทคโนโลยีราชมงคลล้านนา
