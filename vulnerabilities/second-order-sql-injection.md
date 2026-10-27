# Second-Order SQL Injection

## Tóm tắt
Second-Order SQL Injection xảy ra khi dữ liệu độc hại được lưu vào cơ sở dữ liệu ở một bước trước, sau đó được dùng lại trong truy vấn SQL ở bước khác mà không được tham số hóa hoặc kiểm soát.

## Tác động
- Bypass các kiểm tra ở điểm nhập ban đầu vì lỗi chỉ xuất hiện ở luồng xử lý sau.
- Ảnh hưởng đến tác vụ nền, báo cáo, quản trị hoặc đồng bộ dữ liệu.
- Có thể dẫn đến rò rỉ hoặc sửa đổi dữ liệu tùy vị trí dữ liệu được tái sử dụng.

## Nguyên nhân phổ biến
- Tin tưởng dữ liệu đã lưu trong database là an toàn.
- Tạo truy vấn động từ trường cấu hình, tên người dùng, bộ lọc hoặc metadata.
- Không áp dụng cùng tiêu chuẩn bảo vệ cho tác vụ nội bộ và tác vụ nền.

## Ví dụ tình huống

Một hệ thống thương mại điện tử cho phép người dùng đặt tên hiển thị (`display_name`) khi đăng ký. Một tác vụ nền chạy hằng đêm tổng hợp doanh thu theo từng người dùng, và nó lấy `display_name` từ bảng `users` để ghép vào câu truy vấn. Điểm đăng ký dùng parameterized query nên trông an toàn, nhưng tác vụ nền thì tin tưởng dữ liệu đã lưu.

### Code dễ bị tấn công (TypeScript/Express)

**Bước 1: Đăng ký, lưu dữ liệu an toàn nhưng không kiểm soát nội dung**

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";

const app = express();
app.use(express.json());
const pool = new Pool();

app.post("/register", async (req: Request, res: Response) => {
  const { email, displayName } = req.body;

  // Ở bước này đã tham số hóa nên KHÔNG bị SQL Injection ngay lập tức.
  // Nhưng chuỗi độc hại như: x'; DROP TABLE reports; --
  // vẫn được lưu nguyên vẹn vào database.
  await pool.query(
    "INSERT INTO users (email, display_name) VALUES ($1, $2)",
    [email, displayName]
  );

  res.status(201).json({ message: "Đăng ký thành công" });
});

app.listen(3000);
```

**Bước 2: Tác vụ nền dùng lại dữ liệu đã lưu**

```typescript
// jobs/nightlyReport.ts
import { Pool } from "pg";

const pool = new Pool();

export async function nightlyReport() {
  const { rows: users } = await pool.query("SELECT id, display_name FROM users");

  for (const user of users) {
    // LỖI: tin tưởng display_name vì "dữ liệu đến từ database nội bộ"
    // và nối thẳng vào câu truy vấn
    const sql = `
      INSERT INTO reports (owner_name, total)
      SELECT '${user.display_name}', SUM(amount)
      FROM orders
      WHERE user_id = ${user.id}
    `;
    await pool.query(sql);
  }
}
```

### Vì sao vẫn bị khai thác

```text
Luồng tấn công:

1. Kẻ tấn công đăng ký với displayName:  x'); DROP TABLE reports; --
   => /register dùng parameterized query nên lưu chuỗi này như dữ liệu thường.
   => Các kiểm tra ở điểm nhập (WAF, validation cơ bản) không thấy gì bất thường.

2. Đêm đó tác vụ nền chạy, đọc display_name từ database và ghép vào SQL:

   INSERT INTO reports (owner_name, total)
   SELECT 'x'); DROP TABLE reports; --', SUM(amount) ...

3. Chuỗi đã thoát khỏi dấu nháy và trở thành mã SQL được thực thi
   với quyền của tài khoản tác vụ nền (thường rất cao).
```

Điểm mấu chốt: lỗ hổng nằm ở nơi dữ liệu được **dùng lại**, không phải nơi dữ liệu **được nhập vào**. Tác vụ nền thường chạy với quyền cao và ít bị giám sát hơn, nên hậu quả có thể nặng hơn SQL Injection thông thường.

### Code đã vá

**Bước 1: Xác thực dữ liệu trước khi lưu (lớp phòng thủ bổ sung)**

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";

const app = express();
app.use(express.json());
const pool = new Pool();

// Cho phép chữ, số, khoảng trắng và một số ký tự thông dụng, giới hạn độ dài
const DISPLAY_NAME_RE = /^[\p{L}\p{N} ._-]{1,50}$/u;

app.post("/register", async (req: Request, res: Response) => {
  const { email, displayName } = req.body;

  if (typeof displayName !== "string" || !DISPLAY_NAME_RE.test(displayName)) {
    return res.status(400).json({ message: "Tên hiển thị không hợp lệ" });
  }

  await pool.query(
    "INSERT INTO users (email, display_name) VALUES ($1, $2)",
    [email, displayName]
  );

  res.status(201).json({ message: "Đăng ký thành công" });
});

app.listen(3000);
```

**Bước 2: Tác vụ nền tham số hóa mọi giá trị, kể cả dữ liệu từ database**

```typescript
// jobs/nightlyReport.ts
import { Pool } from "pg";

// Tài khoản DB riêng cho tác vụ nền, quyền tối thiểu
const jobPool = new Pool({
  user: "report_job",       // chỉ có SELECT trên orders/users, INSERT trên reports
  statement_timeout: 30000,
});

export async function nightlyReport() {
  const { rows: users } = await jobPool.query("SELECT id, display_name FROM users");

  for (const user of users) {
    // ĐÚNG: dữ liệu từ database vẫn được coi là không đáng tin
    await jobPool.query(
      `INSERT INTO reports (owner_name, total)
       SELECT $1, COALESCE(SUM(amount), 0)
       FROM orders
       WHERE user_id = $2`,
      [user.display_name, user.id]
    );
  }
}
```

Hoặc tốt hơn, gộp thành một câu lệnh duy nhất, không cần vòng lặp và không còn chỗ để ghép chuỗi:

```typescript
export async function nightlyReport() {
  await jobPool.query(
    `INSERT INTO reports (owner_name, total)
     SELECT u.display_name, COALESCE(SUM(o.amount), 0)
     FROM users u
     LEFT JOIN orders o ON o.user_id = u.id
     GROUP BY u.id, u.display_name`
  );
}
```

Phân quyền cho tài khoản tác vụ nền (PostgreSQL):

```sql
CREATE ROLE report_job LOGIN PASSWORD '...';

GRANT SELECT ON users, orders TO report_job;
GRANT INSERT ON reports TO report_job;
-- Không cấp DROP, ALTER, DELETE, hay quyền trên các bảng khác
```

Trường hợp cần tên bảng hoặc cột động (không thể tham số hóa bằng `$n`), dùng allowlist và định danh được escape:

```typescript
import { escapeIdentifier } from "pg"; // hoặc dùng pg-format

const ALLOWED_COLUMNS = new Set(["created_at", "total_amount"]);

function safeColumn(name: string): string {
  if (!ALLOWED_COLUMNS.has(name)) throw new Error("Cột không được phép");
  return name; // đã nằm trong allowlist cố định
}
```

Những điểm đã thay đổi:
- Tham số hóa truy vấn ở tác vụ nền, coi dữ liệu từ database là không đáng tin.
- Gộp vòng lặp thành một câu lệnh `INSERT ... SELECT`, loại bỏ việc ghép chuỗi.
- Xác thực định dạng và độ dài `display_name` trước khi lưu, làm lớp bảo vệ bổ sung.
- Dùng tài khoản DB riêng cho tác vụ nền với quyền tối thiểu, hạn chế thiệt hại nếu có lỗi sót.
- Đặt `statement_timeout` cho tác vụ nền.
- Với tên bảng/cột động, dùng allowlist thay vì nối chuỗi từ dữ liệu bất kỳ.

## Biện pháp phòng tránh
- Tham số hóa truy vấn ở mọi nơi, kể cả khi dữ liệu đến từ database nội bộ.
- Xác thực và chuẩn hóa dữ liệu trước khi lưu và trước khi tái sử dụng.
- Rà soát các chức năng tạo báo cáo, tìm kiếm nâng cao và tác vụ batch.
- Tách quyền giữa tài khoản ứng dụng, tài khoản quản trị và tài khoản tác vụ nền.
- Dùng ORM hoặc query builder và cấm ghép chuỗi SQL bằng lint rule (ví dụ `eslint-plugin-sql`, Semgrep).
- Đưa tác vụ nền, báo cáo và script quản trị vào phạm vi code review và kiểm thử bảo mật như code của API chính.
- Ghi log và cảnh báo khi tác vụ nền thực thi câu lệnh DDL hoặc truy cập bảng bất thường.
