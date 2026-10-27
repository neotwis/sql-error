# Truy vấn SQL quá phức tạp gây quá tải

## Tóm tắt
Một hệ thống có thể bị quá tải khi cho phép người dùng tạo bộ lọc, sắp xếp, tìm kiếm hoặc báo cáo dẫn đến truy vấn SQL quá phức tạp. Dù không phải lúc nào cũng là xâm nhập dữ liệu, đây là rủi ro gây từ chối dịch vụ ở tầng cơ sở dữ liệu.

## Tác động
- CPU, RAM hoặc I/O của database tăng cao.
- Hàng đợi kết nối bị đầy, làm chậm toàn bộ ứng dụng.
- Khóa bảng hoặc khóa dòng kéo dài, ảnh hưởng tác vụ hợp lệ.
- Chi phí hạ tầng tăng do truy vấn không tối ưu.

## Nguyên nhân phổ biến
- Cho phép lọc tùy ý trên nhiều cột không có index phù hợp.
- Tìm kiếm dùng wildcard rộng trên tập dữ liệu lớn.
- JOIN nhiều bảng lớn không giới hạn phạm vi.
- Không có phân trang, giới hạn số dòng hoặc timeout truy vấn.

## Ví dụ tình huống

Một API tìm kiếm đơn hàng cho phép người dùng lọc theo nhiều trường, tìm theo từ khóa và sắp xếp theo cột bất kỳ. Dữ liệu gồm hàng triệu đơn hàng và hàng chục triệu dòng chi tiết đơn hàng.

### Code dễ gây quá tải (TypeScript/Express)

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";

const app = express();
const pool = new Pool();

app.get("/orders", async (req: Request, res: Response) => {
  const { keyword, status, sortBy, sortDir } = req.query;

  // LỖI 1: wildcard ở đầu và cuối, không dùng được index thông thường
  // LỖI 2: JOIN nhiều bảng lớn, không giới hạn phạm vi thời gian
  // LỖI 3: không có LIMIT/OFFSET, có thể trả về hàng triệu dòng
  // LỖI 4: cho sắp xếp theo cột bất kỳ, kể cả cột không có index
  let sql = `
    SELECT o.*, c.name, i.product_name, i.quantity
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    JOIN order_items i ON i.order_id = o.id
    WHERE (c.name ILIKE '%${keyword}%' OR i.product_name ILIKE '%${keyword}%')
  `;

  if (status) {
    sql += ` AND o.status = '${status}'`;
  }

  sql += ` ORDER BY ${sortBy ?? "o.created_at"} ${sortDir ?? "DESC"}`;

  const result = await pool.query(sql); // LỖI 5: không có timeout
  res.json(result.rows);
});

app.listen(3000);
```

Ngoài ra, code này còn nối chuỗi trực tiếp đầu vào nên đồng thời dính SQL Injection. Phần dưới đây tập trung vào vấn đề quá tải.

### Vì sao gây quá tải

- `ILIKE '%...%'` buộc database quét toàn bộ bảng (sequential scan) trên cả `customers` và `order_items`.
- JOIN hai bảng lớn rồi mới lọc tạo ra tập dữ liệu trung gian rất lớn, tốn RAM và I/O.
- Không có `LIMIT`, một request có thể kéo hàng triệu dòng vào bộ nhớ của cả database lẫn ứng dụng.
- Sắp xếp theo cột không có index buộc database phải sort trên đĩa.
- Không có timeout nên truy vấn chậm giữ kết nối rất lâu. Chỉ cần vài request đồng thời là connection pool cạn, toàn bộ ứng dụng bị treo.

### Code đã vá

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";
import rateLimit from "express-rate-limit";

const app = express();

const pool = new Pool({
  max: 10,                    // giới hạn số kết nối
  statement_timeout: 3000,    // tối đa 3 giây mỗi truy vấn
  query_timeout: 4000,
});

const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 30, // 30 request/phút mỗi IP
});

// Danh sách cho phép (allowlist) cho cột sắp xếp, các cột này đã có index
const SORTABLE_COLUMNS: Record<string, string> = {
  createdAt: "o.created_at",
  total: "o.total_amount",
};

const ALLOWED_STATUS = ["pending", "paid", "shipped", "cancelled"];
const MAX_PAGE_SIZE = 50;
const MAX_RANGE_DAYS = 90;

app.get("/orders", limiter, async (req: Request, res: Response) => {
  // 1. Phân trang bắt buộc, có giới hạn trên
  const page = Math.max(1, Number(req.query.page) || 1);
  const pageSize = Math.min(MAX_PAGE_SIZE, Number(req.query.pageSize) || 20);

  // 2. Bắt buộc có khoảng thời gian và giới hạn độ dài khoảng
  const from = new Date(String(req.query.from));
  const to = new Date(String(req.query.to));
  const rangeDays = (to.getTime() - from.getTime()) / 86_400_000;

  if (isNaN(rangeDays) || rangeDays < 0 || rangeDays > MAX_RANGE_DAYS) {
    return res
      .status(400)
      .json({ message: `Khoảng thời gian phải hợp lệ và tối đa ${MAX_RANGE_DAYS} ngày` });
  }

  // 3. Allowlist cho sắp xếp và trạng thái
  const sortColumn = SORTABLE_COLUMNS[String(req.query.sortBy)] ?? "o.created_at";
  const sortDir = req.query.sortDir === "asc" ? "ASC" : "DESC";

  const params: unknown[] = [from, to];
  let where = "o.created_at BETWEEN $1 AND $2";

  const status = String(req.query.status ?? "");
  if (status) {
    if (!ALLOWED_STATUS.includes(status)) {
      return res.status(400).json({ message: "Trạng thái không hợp lệ" });
    }
    params.push(status);
    where += ` AND o.status = $${params.length}`;
  }

  // 4. Tìm kiếm theo tiền tố (dùng được index), yêu cầu tối thiểu 3 ký tự
  const keyword = String(req.query.keyword ?? "").trim();
  if (keyword) {
    if (keyword.length < 3) {
      return res.status(400).json({ message: "Từ khóa tối thiểu 3 ký tự" });
    }
    params.push(`${keyword}%`);
    where += ` AND c.name ILIKE $${params.length}`;
  }

  params.push(pageSize, (page - 1) * pageSize);

  // 5. Không JOIN order_items ở danh sách, chỉ lấy thông tin tóm tắt
  const sql = `
    SELECT o.id, o.status, o.total_amount, o.created_at, c.name AS customer_name
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    WHERE ${where}
    ORDER BY ${sortColumn} ${sortDir}
    LIMIT $${params.length - 1} OFFSET $${params.length}
  `;

  try {
    const result = await pool.query(sql, params);
    return res.json({ page, pageSize, data: result.rows });
  } catch (err) {
    console.error("orders query failed", err);
    return res.status(503).json({ message: "Hệ thống đang bận, vui lòng thử lại sau" });
  }
});

app.listen(3000);
```

Index hỗ trợ cho các truy vấn trên:

```sql
CREATE INDEX idx_orders_created_at ON orders (created_at);
CREATE INDEX idx_orders_status_created_at ON orders (status, created_at);
CREATE INDEX idx_customers_name_pattern ON customers (name text_pattern_ops);
```

Kiểm tra kế hoạch thực thi trước khi đưa lên production:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.status, o.total_amount, o.created_at, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY o.created_at DESC
LIMIT 20 OFFSET 0;
```

Những điểm đã thay đổi:
- Bắt buộc phân trang và giới hạn `pageSize` tối đa.
- Bắt buộc khoảng thời gian có giới hạn, thu hẹp tập dữ liệu ngay từ đầu.
- Dùng allowlist cho cột sắp xếp và trạng thái, chỉ cho phép cột đã có index.
- Tìm kiếm theo tiền tố thay vì wildcard hai đầu, kèm độ dài tối thiểu.
- Bỏ JOIN `order_items` khỏi danh sách, chỉ tải chi tiết khi người dùng mở một đơn.
- Dùng parameterized query cho mọi giá trị đầu vào.
- Đặt `statement_timeout`, giới hạn connection pool và rate limit.

## Biện pháp phòng tránh
- Đặt giới hạn bắt buộc cho phân trang, kích thước kết quả và độ phức tạp bộ lọc.
- Tạo index dựa trên mẫu truy vấn thực tế và kiểm tra kế hoạch thực thi.
- Dùng cache, materialized view hoặc hàng đợi cho báo cáo nặng.
- Thiết lập statement timeout, connection pool hợp lý và rate limit theo người dùng.
- Theo dõi slow query log và cảnh báo khi tài nguyên tăng bất thường.
