# Truy vấn không giới hạn kết quả

## Tóm tắt
Truy vấn không giới hạn kết quả xảy ra khi API hoặc chức năng xuất dữ liệu cho phép trả về quá nhiều dòng cùng lúc. Điều này có thể làm cạn bộ nhớ ứng dụng, bão hòa mạng hoặc gây áp lực lớn lên cơ sở dữ liệu.

## Tác động
- Tăng độ trễ hoặc timeout cho người dùng hợp lệ.
- Cạn RAM ở ứng dụng khi cố nạp toàn bộ kết quả vào bộ nhớ.
- Làm nghẽn băng thông giữa ứng dụng và database.
- Dễ bị lạm dụng để tạo tải cao lặp lại.

## Nguyên nhân phổ biến
- Thiếu LIMIT, OFFSET hoặc cursor-based pagination.
- Endpoint export dữ liệu không có giới hạn hoặc không chạy bất đồng bộ.
- Không giới hạn tần suất gọi API theo người dùng hoặc IP.
- Không có ngưỡng tối đa cho khoảng thời gian lọc dữ liệu.

## Ví dụ tình huống

Một hệ thống quản lý giao dịch có hai endpoint: danh sách giao dịch (`/transactions`) và xuất CSV (`/transactions/export`). Bảng `transactions` có hàng chục triệu dòng. Cả hai endpoint đều không phân trang, không giới hạn khoảng thời gian, và export nạp toàn bộ kết quả vào bộ nhớ rồi mới ghép thành chuỗi CSV.

### Code dễ bị quá tải (TypeScript/Express)

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";

const app = express();
const pool = new Pool();

// LỖI 1: không có LIMIT, trả về toàn bộ bảng
// LỖI 2: không bắt buộc khoảng thời gian
app.get("/transactions", async (req: Request, res: Response) => {
  const { userId } = req.query;
  const result = await pool.query(
    "SELECT * FROM transactions WHERE user_id = $1", // SELECT * kéo cả cột không cần thiết
    [userId]
  );
  res.json(result.rows); // serialize hàng triệu dòng thành JSON trong RAM
});

// LỖI 3: export đồng bộ, nạp toàn bộ kết quả vào bộ nhớ
// LỖI 4: không rate limit, không giới hạn số lần export
app.get("/transactions/export", async (req: Request, res: Response) => {
  const result = await pool.query("SELECT * FROM transactions");

  const csv = result.rows
    .map((r) => `${r.id},${r.user_id},${r.amount},${r.created_at.toISOString()}`)
    .join("\n");

  res.setHeader("Content-Type", "text/csv");
  res.send(csv);
});

app.listen(3000);
```

### Vì sao gây quá tải

```text
Kịch bản: bảng có 20 triệu dòng, mỗi dòng ~200 byte

- pool.query() nạp toàn bộ ~4 GB kết quả vào RAM của tiến trình Node.js
  => vượt giới hạn heap, tiến trình bị crash (out of memory).
- Chuỗi CSV được tạo thêm một bản sao nữa trong bộ nhớ => RAM tăng gấp đôi.
- Hàng GB dữ liệu đi qua mạng giữa database và ứng dụng, chiếm hết băng thông.
- Kết nối database bị giữ trong suốt quá trình, các request khác phải xếp hàng.
- Chỉ cần vài request export đồng thời là cả hệ thống ngừng phản hồi.
```

### Code đã vá

**Danh sách giao dịch: phân trang kiểu cursor (keyset) với giới hạn bắt buộc**

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";
import rateLimit from "express-rate-limit";

const app = express();

const pool = new Pool({
  max: 10,
  statement_timeout: 5000,
});

const listLimiter = rateLimit({ windowMs: 60 * 1000, max: 60 });
const exportLimiter = rateLimit({ windowMs: 60 * 60 * 1000, max: 3 });

const DEFAULT_PAGE_SIZE = 20;
const MAX_PAGE_SIZE = 100;
const MAX_RANGE_DAYS = 90;

app.get("/transactions", listLimiter, async (req: Request, res: Response) => {
  const userId = Number(req.query.userId);
  if (!Number.isInteger(userId)) {
    return res.status(400).json({ message: "userId không hợp lệ" });
  }

  // 1. Giới hạn kích thước trang có ngưỡng trên
  const limit = Math.min(
    MAX_PAGE_SIZE,
    Math.max(1, Number(req.query.limit) || DEFAULT_PAGE_SIZE)
  );

  // 2. Bắt buộc khoảng thời gian và giới hạn độ dài khoảng
  const from = new Date(String(req.query.from));
  const to = new Date(String(req.query.to));
  const rangeDays = (to.getTime() - from.getTime()) / 86_400_000;
  if (isNaN(rangeDays) || rangeDays < 0 || rangeDays > MAX_RANGE_DAYS) {
    return res
      .status(400)
      .json({ message: `Khoảng thời gian phải hợp lệ và tối đa ${MAX_RANGE_DAYS} ngày` });
  }

  // 3. Cursor: dùng (created_at, id) của dòng cuối trang trước
  const cursorTime = req.query.cursorTime ? new Date(String(req.query.cursorTime)) : null;
  const cursorId = req.query.cursorId ? Number(req.query.cursorId) : null;

  const params: unknown[] = [userId, from, to];
  let cursorClause = "";
  if (cursorTime && cursorId !== null) {
    params.push(cursorTime, cursorId);
    cursorClause = `AND (created_at, id) < ($4, $5)`;
  }
  params.push(limit + 1); // lấy dư 1 dòng để biết còn trang sau không

  try {
    // 4. Chỉ chọn cột cần thiết, thay SELECT *
    const result = await pool.query(
      `SELECT id, amount, status, created_at
       FROM transactions
       WHERE user_id = $1 AND created_at BETWEEN $2 AND $3
       ${cursorClause}
       ORDER BY created_at DESC, id DESC
       LIMIT $${params.length}`,
      params
    );

    const hasMore = result.rows.length > limit;
    const rows = hasMore ? result.rows.slice(0, limit) : result.rows;
    const last = rows[rows.length - 1];

    return res.json({
      data: rows,
      nextCursor: hasMore ? { cursorTime: last.created_at, cursorId: last.id } : null,
    });
  } catch (err) {
    console.error("transactions query failed", err);
    return res.status(503).json({ message: "Hệ thống đang bận, vui lòng thử lại sau" });
  }
});
```

**Export: tác vụ nền hoặc streaming, không nạp toàn bộ vào RAM**

```typescript
import { pipeline } from "node:stream/promises";
import { Transform } from "node:stream";
import QueryStream from "pg-query-stream";

const MAX_EXPORT_ROWS = 1_000_000;

app.get("/transactions/export", exportLimiter, async (req: Request, res: Response) => {
  const userId = Number(req.query.userId);
  if (!Number.isInteger(userId)) {
    return res.status(400).json({ message: "userId không hợp lệ" });
  }

  const client = await pool.connect();
  try {
    res.setHeader("Content-Type", "text/csv");
    res.setHeader("Content-Disposition", 'attachment; filename="transactions.csv"');
    res.write("id,amount,status,created_at\n");

    // Con trỏ phía server: đọc từng lô nhỏ (500 dòng), không nạp hết vào RAM
    const query = new QueryStream(
      `SELECT id, amount, status, created_at
       FROM transactions
       WHERE user_id = $1
       ORDER BY id
       LIMIT ${MAX_EXPORT_ROWS}`,
      [userId],
      { batchSize: 500 }
    );
    const dbStream = client.query(query);

    const toCsv = new Transform({
      objectMode: true,
      transform(row, _enc, cb) {
        cb(null, `${row.id},${row.amount},${row.status},${row.created_at.toISOString()}\n`);
      },
    });

    // pipeline tự xử lý backpressure và dọn dẹp khi client ngắt kết nối
    await pipeline(dbStream, toCsv, res);
  } catch (err) {
    console.error("export failed", err);
    if (!res.headersSent) {
      res.status(500).json({ message: "Lỗi hệ thống" });
    }
  } finally {
    client.release();
  }
});

app.listen(3000);
```

Với dữ liệu rất lớn, nên chuyển sang mô hình bất đồng bộ: nhận yêu cầu, xử lý nền, ghi file lên kho lưu trữ rồi gửi liên kết tải về.

```typescript
app.post("/transactions/export", exportLimiter, async (req: Request, res: Response) => {
  const jobId = await exportQueue.add("transactions", {
    userId: (req as any).user.id,
    from: req.body.from,
    to: req.body.to,
  });
  return res.status(202).json({ jobId, statusUrl: `/exports/${jobId}` });
});
```

Index hỗ trợ phân trang keyset:

```sql
-- Khớp đúng với ORDER BY và điều kiện lọc
CREATE INDEX idx_transactions_user_created
  ON transactions (user_id, created_at DESC, id DESC);
```

Những điểm đã thay đổi:
- Bắt buộc `LIMIT` với kích thước trang mặc định và ngưỡng tối đa.
- Dùng cursor (keyset) thay cho `OFFSET`, hiệu năng ổn định ở trang sâu.
- Bắt buộc khoảng thời gian lọc và giới hạn độ dài khoảng.
- Thay `SELECT *` bằng danh sách cột cần thiết.
- Export dùng streaming với `pg-query-stream`, RAM không tăng theo số dòng; có trần số bản ghi.
- Rate limit riêng, chặt hơn cho endpoint export.
- Đặt `statement_timeout`; với export lớn chuyển sang hàng đợi và tải file.

## Biện pháp phòng tránh
- Bắt buộc phân trang và đặt kích thước trang tối đa.
- Dùng streaming hoặc tác vụ nền cho export lớn thay vì trả đồng bộ.
- Giới hạn khoảng thời gian truy vấn và số bản ghi tối đa.
- Áp dụng rate limit, quota và giám sát tải theo endpoint.
- Dùng cursor-based pagination cho bảng lớn thay cho `OFFSET` sâu.
- Chỉ trả về các cột cần thiết, tránh `SELECT *`.
- Cảnh báo khi response vượt ngưỡng kích thước hoặc số dòng bất thường.
