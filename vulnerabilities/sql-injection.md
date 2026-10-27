# SQL Injection

## Tóm tắt
SQL Injection xảy ra khi ứng dụng ghép trực tiếp dữ liệu người dùng vào câu lệnh SQL mà không kiểm soát đúng cách. Kẻ tấn công có thể làm thay đổi ý nghĩa truy vấn, đọc dữ liệu trái phép, sửa dữ liệu hoặc bỏ qua cơ chế xác thực.

## Tác động
- Rò rỉ dữ liệu nhạy cảm như tài khoản, email, token hoặc thông tin thanh toán.
- Thay đổi, xóa hoặc chèn dữ liệu trái phép.
- Vượt qua đăng nhập nếu truy vấn xác thực bị thao túng.
- Làm gián đoạn dịch vụ khi truy vấn bị biến thành tác vụ tốn tài nguyên.

## Nguyên nhân phổ biến
- Nối chuỗi SQL bằng input từ người dùng.
- Không dùng prepared statement hoặc parameterized query.
- Tài khoản kết nối cơ sở dữ liệu có quyền quá rộng.
- Thông báo lỗi SQL trả về quá chi tiết cho người dùng cuối.

## Ví dụ tình huống

Một chức năng đăng nhập nhận `email` và `password` từ form. Lập trình viên ghép trực tiếp hai giá trị này vào câu truy vấn để kiểm tra thông tin xác thực. Ngoài ra, API tìm kiếm sản phẩm còn trả nguyên thông báo lỗi SQL về cho client.

### Code dễ bị tấn công (TypeScript/Express)

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";

const app = express();
app.use(express.json());

// LỖI 1: dùng tài khoản DB có quyền quá rộng (ví dụ superuser)
const pool = new Pool({ user: "postgres" });

app.post("/login", async (req: Request, res: Response) => {
  const { email, password } = req.body;

  try {
    // LỖI 2: nối chuỗi trực tiếp input của người dùng vào câu truy vấn
    const sql = `SELECT id, email, role FROM users
                 WHERE email = '${email}' AND password_hash = '${password}'`;
    const result = await pool.query(sql);

    if (result.rowCount === 0) {
      return res.status(401).json({ message: "Sai thông tin đăng nhập" });
    }
    return res.json({ user: result.rows[0] });
  } catch (err: any) {
    // LỖI 3: trả nguyên thông báo lỗi SQL về cho người dùng cuối
    return res.status(500).json({ error: err.message, query: err.query });
  }
});

app.get("/products", async (req: Request, res: Response) => {
  const { category } = req.query;
  try {
    // LỖI 2 (lặp lại): nối chuỗi với tham số trên URL
    const result = await pool.query(
      `SELECT id, name, price FROM products WHERE category = '${category}'`
    );
    return res.json(result.rows);
  } catch (err: any) {
    return res.status(500).json({ error: err.message }); // LỖI 3
  }
});

app.listen(3000);
```

### Vì sao bị khai thác

**1. Vượt qua đăng nhập (authentication bypass)**

```text
Đầu vào:
  email    = admin@example.com' --
  password = (bất kỳ)

Câu truy vấn thực tế sau khi ghép chuỗi:
  SELECT id, email, role FROM users
  WHERE email = 'admin@example.com' --' AND password_hash = '...'

Phần sau dấu -- trở thành chú thích, điều kiện kiểm tra mật khẩu bị bỏ qua.
=> Đăng nhập thành công với tài khoản admin mà không cần mật khẩu.
```

**2. Đọc dữ liệu trái phép (UNION-based)**

```text
Request:
  GET /products?category=x' UNION SELECT id, email, password_hash FROM users --

Câu truy vấn thực tế:
  SELECT id, name, price FROM products WHERE category = 'x'
  UNION SELECT id, email, password_hash FROM users --'

=> Kết quả trả về chứa email và password_hash của toàn bộ người dùng.
```

**3. Lộ thông tin qua thông báo lỗi (error-based)**

```text
Request:
  GET /products?category='

Phản hồi:
  { "error": "unterminated quoted string at or near \"'\"" }

=> Kẻ tấn công biết ứng dụng dùng PostgreSQL, biết cấu trúc truy vấn
   và có thể tinh chỉnh payload dựa trên các lỗi nhận được.
```

**4. Thay đổi hoặc xóa dữ liệu**

Vì tài khoản DB có quyền rộng, kẻ tấn công có thể dùng stacked queries (nếu driver cho phép) để chạy thêm lệnh `UPDATE`, `DELETE` hoặc `DROP`. Quyền càng rộng, thiệt hại càng lớn.

### Code đã vá

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";
import bcrypt from "bcrypt";
import rateLimit from "express-rate-limit";

const app = express();
app.use(express.json());

// 1. Tài khoản DB riêng cho ứng dụng, quyền tối thiểu (xem phần SQL bên dưới)
const pool = new Pool({
  user: "app_user",
  statement_timeout: 5000,
});

const loginLimiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 10 });

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
const ALLOWED_CATEGORIES = ["books", "electronics", "clothing", "toys"];

app.post("/login", loginLimiter, async (req: Request, res: Response) => {
  const { email, password } = req.body;

  // 2. Kiểm tra kiểu dữ liệu, độ dài và định dạng
  if (
    typeof email !== "string" || email.length > 254 || !EMAIL_RE.test(email) ||
    typeof password !== "string" || password.length < 8 || password.length > 128
  ) {
    return res.status(400).json({ message: "Dữ liệu không hợp lệ" });
  }

  try {
    // 3. Parameterized query: input luôn là dữ liệu, không bao giờ là mã SQL
    const result = await pool.query(
      "SELECT id, email, role, password_hash FROM users WHERE email = $1",
      [email]
    );

    // 4. So sánh mật khẩu bằng hash, không so sánh trong câu SQL
    const user = result.rows[0];
    const ok = user && (await bcrypt.compare(password, user.password_hash));

    if (!ok) {
      return res.status(401).json({ message: "Sai thông tin đăng nhập" });
    }
    return res.json({ user: { id: user.id, email: user.email, role: user.role } });
  } catch (err) {
    // 5. Ghi log nội bộ, trả thông báo chung cho client
    console.error("login failed", err);
    return res.status(500).json({ message: "Lỗi hệ thống" });
  }
});

app.get("/products", async (req: Request, res: Response) => {
  const category = String(req.query.category ?? "");

  // Allowlist cho giá trị có tập hợp cố định
  if (!ALLOWED_CATEGORIES.includes(category)) {
    return res.status(400).json({ message: "Danh mục không hợp lệ" });
  }

  try {
    const result = await pool.query(
      "SELECT id, name, price FROM products WHERE category = $1",
      [category]
    );
    return res.json(result.rows);
  } catch (err) {
    console.error("products query failed", err);
    return res.status(500).json({ message: "Lỗi hệ thống" });
  }
});

app.listen(3000);
```

Phân quyền tối thiểu cho tài khoản ứng dụng (PostgreSQL):

```sql
CREATE ROLE app_user LOGIN PASSWORD '...';

-- Chỉ cấp đúng quyền cần dùng trên đúng bảng
GRANT SELECT ON products TO app_user;
GRANT SELECT ON users TO app_user;
GRANT INSERT, UPDATE ON orders TO app_user;

-- Không cấp: SUPERUSER, CREATE, DROP, ALTER, hay quyền trên bảng nhạy cảm không liên quan
-- Nếu có thể, tách tài khoản chỉ đọc (read-only) cho các endpoint chỉ truy vấn
```

Kiểm thử bảo mật tự động cho endpoint nhận input động (Jest + supertest):

```typescript
import request from "supertest";
import { app } from "./app";

const PAYLOADS = [
  "' OR '1'='1",
  "admin@example.com' --",
  "x' UNION SELECT NULL --",
  "'; DROP TABLE users; --",
];

describe("SQL Injection", () => {
  test.each(PAYLOADS)("POST /login từ chối payload: %s", async (payload) => {
    const res = await request(app)
      .post("/login")
      .send({ email: payload, password: "whatever123" });

    expect([400, 401]).toContain(res.status);
    expect(JSON.stringify(res.body)).not.toMatch(/syntax|pg_|postgres|SELECT/i);
  });

  test.each(PAYLOADS)("GET /products từ chối payload: %s", async (payload) => {
    const res = await request(app).get("/products").query({ category: payload });

    expect(res.status).toBe(400);
    expect(JSON.stringify(res.body)).not.toMatch(/syntax|pg_|postgres|SELECT/i);
  });
});
```

Những điểm đã thay đổi:
- Dùng parameterized query (`$1`) thay cho nối chuỗi, input không thể thay đổi cấu trúc câu truy vấn.
- Không so sánh mật khẩu trong SQL, chỉ lấy bản ghi theo email rồi dùng `bcrypt.compare`.
- Kiểm tra kiểu, độ dài, định dạng; dùng allowlist cho giá trị có tập cố định.
- Dùng tài khoản DB riêng với quyền tối thiểu, hạn chế thiệt hại nếu còn lỗi sót.
- Ẩn chi tiết lỗi SQL khỏi phản hồi, chỉ ghi log nội bộ.
- Thêm rate limit cho đăng nhập và `statement_timeout` cho truy vấn.
- Bổ sung kiểm thử tự động với các payload SQL Injection phổ biến.

## Biện pháp phòng tránh
- Luôn dùng prepared statement, parameterized query hoặc ORM an toàn.
- Kiểm tra kiểu dữ liệu, độ dài và định dạng input ở tầng ứng dụng.
- Áp dụng nguyên tắc đặc quyền tối thiểu cho tài khoản cơ sở dữ liệu.
- Ẩn chi tiết lỗi SQL khỏi phản hồi công khai; ghi log nội bộ có kiểm soát.
- Bổ sung kiểm thử bảo mật cho các endpoint nhận input động.
- Với tên bảng hoặc cột động (không thể tham số hóa), dùng allowlist cố định thay vì nối chuỗi từ input.
- Dùng lint rule hoặc công cụ quét (Semgrep, CodeQL) để phát hiện nối chuỗi SQL trong CI.
- Cân nhắc WAF như lớp bảo vệ bổ sung, không dùng thay cho tham số hóa.
