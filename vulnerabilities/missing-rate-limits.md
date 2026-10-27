# Thiếu Rate Limit cho thao tác SQL nặng

## Tóm tắt
Khi các endpoint kích hoạt truy vấn SQL nặng không có rate limit, một người dùng hoặc bot có thể gửi nhiều yêu cầu liên tiếp làm quá tải database. Đây là rủi ro vận hành thường gặp ở tìm kiếm, báo cáo, thống kê và export dữ liệu.

## Tác động
- Tài nguyên database bị chiếm dụng bởi một nhóm nhỏ yêu cầu.
- Người dùng hợp lệ bị chậm hoặc không thể truy cập dịch vụ.
- Tăng nguy cơ timeout, lỗi kết nối và tự động mở rộng tốn kém.
- Khó phân biệt giữa tăng trưởng thật và lạm dụng nếu thiếu quan sát.

## Nguyên nhân phổ biến
- Không giới hạn số lần gọi endpoint theo tài khoản, IP hoặc token.
- Không phân loại độ nặng của truy vấn để áp dụng quota khác nhau.
- Cho phép chạy báo cáo đồng bộ nhiều lần trong thời gian ngắn.
- Thiếu circuit breaker khi database đã quá tải.

## Ví dụ tình huống

Một hệ thống quản lý bán hàng có endpoint xuất báo cáo doanh thu theo năm. Truy vấn tổng hợp trên hàng chục triệu dòng, mỗi lần chạy mất khoảng 8 đến 10 giây. Endpoint chỉ yêu cầu đăng nhập, không giới hạn số lần gọi. Một người dùng (hoặc script bị lỗi, hoặc bot dùng token bị lộ) gọi liên tục nhiều request song song.

### Code dễ bị quá tải (TypeScript/Express)

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";

const app = express();
const pool = new Pool(); // mặc định tối đa 10 kết nối

// LỖI 1: không có rate limit, mỗi người dùng gọi bao nhiêu lần cũng được
// LỖI 2: báo cáo nặng chạy đồng bộ ngay trong request
// LỖI 3: không cache, cùng một báo cáo bị tính lại nhiều lần
// LỖI 4: không có timeout, không kiểm tra tải của database
app.get("/reports/revenue", async (req: Request, res: Response) => {
  const year = Number(req.query.year);

  const result = await pool.query(
    `SELECT date_trunc('month', o.created_at) AS month,
            c.region,
            SUM(i.quantity * i.unit_price) AS revenue
     FROM orders o
     JOIN customers c ON c.id = o.customer_id
     JOIN order_items i ON i.order_id = o.id
     WHERE EXTRACT(YEAR FROM o.created_at) = $1
     GROUP BY 1, 2
     ORDER BY 1`,
    [year]
  );

  res.json(result.rows);
});

app.listen(3000);
```

### Vì sao gây quá tải

```text
Kịch bản: 1 người dùng gửi 20 request song song, mỗi request chạy 10 giây

- Pool chỉ có 10 kết nối => 10 request đầu chiếm hết, 10 request còn lại xếp hàng.
- Trong 10 giây đó, mọi người dùng khác đều không lấy được kết nối.
- CPU và I/O của database chạm trần vì 10 truy vấn tổng hợp chạy cùng lúc.
- Request của người dùng hợp lệ bị timeout, ứng dụng báo lỗi diện rộng.
- Nếu hệ thống tự động mở rộng, chi phí tăng mà vấn đề không được giải quyết.
```

Mệnh đề `EXTRACT(YEAR FROM ...)` còn khiến database không dùng được index trên `created_at`, làm mỗi truy vấn nặng hơn mức cần thiết.

### Code đã vá

```typescript
import express, { Request, Response, NextFunction } from "express";
import { Pool } from "pg";
import rateLimit from "express-rate-limit";

const app = express();

const pool = new Pool({
  max: 10,
  statement_timeout: 15000, // tối đa 15 giây mỗi truy vấn
  query_timeout: 16000,
});

// 1. Rate limit riêng cho endpoint nặng, tính theo người dùng (không chỉ IP)
const heavyLimiter = rateLimit({
  windowMs: 10 * 60 * 1000, // 10 phút
  max: 5,                   // tối đa 5 báo cáo mỗi 10 phút
  keyGenerator: (req) => (req as any).user?.id ?? req.ip,
  standardHeaders: true,    // trả header RateLimit-* cho client
  legacyHeaders: false,
  message: { message: "Bạn yêu cầu báo cáo quá nhiều lần, vui lòng thử lại sau" },
});

// 2. Giới hạn số báo cáo chạy đồng thời trên toàn hệ thống (bulkhead)
const MAX_CONCURRENT_HEAVY = 3;
let runningHeavy = 0;

function heavyConcurrencyGuard(req: Request, res: Response, next: NextFunction) {
  if (runningHeavy >= MAX_CONCURRENT_HEAVY) {
    res.setHeader("Retry-After", "30");
    return res.status(429).json({ message: "Hệ thống đang xử lý nhiều báo cáo, thử lại sau" });
  }
  runningHeavy++;
  res.on("finish", () => runningHeavy--);
  res.on("close", () => {
    // đảm bảo giảm bộ đếm cả khi client ngắt kết nối
    if (!res.writableEnded) runningHeavy--;
  });
  next();
}

// 3. Circuit breaker đơn giản: tạm từ chối khi database đang quá tải
let consecutiveFailures = 0;
let circuitOpenUntil = 0;

function circuitBreaker(req: Request, res: Response, next: NextFunction) {
  if (Date.now() < circuitOpenUntil) {
    res.setHeader("Retry-After", "30");
    return res.status(503).json({ message: "Dịch vụ báo cáo tạm thời quá tải" });
  }
  next();
}

// 4. Cache kết quả báo cáo (ví dụ minh họa dùng Map, thực tế nên dùng Redis)
const cache = new Map<string, { expiresAt: number; data: unknown }>();
const CACHE_TTL_MS = 10 * 60 * 1000;

app.get(
  "/reports/revenue",
  heavyLimiter,
  circuitBreaker,
  heavyConcurrencyGuard,
  async (req: Request, res: Response) => {
    const year = Number(req.query.year);
    if (!Number.isInteger(year) || year < 2000 || year > 2100) {
      return res.status(400).json({ message: "Năm không hợp lệ" });
    }

    const cacheKey = `revenue:${year}`;
    const cached = cache.get(cacheKey);
    if (cached && cached.expiresAt > Date.now()) {
      return res.json(cached.data);
    }

    const startedAt = Date.now();
    try {
      // Dùng khoảng thời gian thay cho EXTRACT để tận dụng index created_at
      const result = await pool.query(
        `SELECT date_trunc('month', o.created_at) AS month,
                c.region,
                SUM(i.quantity * i.unit_price) AS revenue
         FROM orders o
         JOIN customers c ON c.id = o.customer_id
         JOIN order_items i ON i.order_id = o.id
         WHERE o.created_at >= make_date($1, 1, 1)
           AND o.created_at <  make_date($1 + 1, 1, 1)
         GROUP BY 1, 2
         ORDER BY 1`,
        [year]
      );

      consecutiveFailures = 0;
      cache.set(cacheKey, { expiresAt: Date.now() + CACHE_TTL_MS, data: result.rows });

      // 5. Ghi log phục vụ điều tra lạm dụng
      console.info("heavy_query", {
        userId: (req as any).user?.id,
        ip: req.ip,
        endpoint: "/reports/revenue",
        params: { year },
        durationMs: Date.now() - startedAt,
      });

      return res.json(result.rows);
    } catch (err) {
      consecutiveFailures++;
      if (consecutiveFailures >= 3) {
        circuitOpenUntil = Date.now() + 30_000; // mở circuit 30 giây
      }
      console.error("revenue report failed", err);
      return res.status(503).json({ message: "Không thể tạo báo cáo, vui lòng thử lại sau" });
    }
  }
);

app.listen(3000);
```

Với báo cáo rất nặng, nên chuyển sang mô hình bất đồng bộ: nhận yêu cầu, đưa vào hàng đợi rồi trả về mã công việc để client kiểm tra sau.

```typescript
// Nhận yêu cầu, trả về ngay và xử lý nền
app.post("/reports/revenue", heavyLimiter, async (req: Request, res: Response) => {
  const year = Number(req.body.year);
  const jobId = await reportQueue.add("revenue", { year, userId: (req as any).user.id });
  return res.status(202).json({ jobId, statusUrl: `/reports/jobs/${jobId}` });
});

app.get("/reports/jobs/:id", async (req: Request, res: Response) => {
  const job = await reportQueue.getJob(req.params.id);
  if (!job) return res.status(404).json({ message: "Không tìm thấy công việc" });
  return res.json({ state: await job.getState(), result: job.returnvalue ?? null });
});
```

Index hỗ trợ cho truy vấn báo cáo, và materialized view cho dữ liệu tổng hợp:

```sql
CREATE INDEX idx_orders_created_at ON orders (created_at);
CREATE INDEX idx_order_items_order_id ON order_items (order_id);

-- Tính sẵn doanh thu theo tháng và khu vực, làm mới theo lịch
CREATE MATERIALIZED VIEW mv_monthly_revenue AS
SELECT date_trunc('month', o.created_at) AS month,
       c.region,
       SUM(i.quantity * i.unit_price) AS revenue
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN order_items i ON i.order_id = o.id
GROUP BY 1, 2;

CREATE UNIQUE INDEX ON mv_monthly_revenue (month, region);
-- Làm mới không chặn truy vấn đọc:
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_revenue;
```

Những điểm đã thay đổi:
- Thêm rate limit riêng cho endpoint nặng, tính theo người dùng thay vì chỉ theo IP.
- Giới hạn số báo cáo chạy đồng thời để một nhóm yêu cầu không chiếm hết connection pool.
- Thêm circuit breaker, tạm từ chối khi database liên tục lỗi hoặc timeout.
- Cache kết quả để các yêu cầu lặp lại không chạm vào database.
- Đặt `statement_timeout`, kiểm tra và chuẩn hóa tham số đầu vào.
- Viết lại điều kiện lọc theo khoảng thời gian để dùng được index.
- Ghi log người dùng, tham số và thời gian thực thi để điều tra lạm dụng.
- Với báo cáo lớn, chuyển sang hàng đợi hoặc materialized view thay vì chạy đồng bộ.

## Biện pháp phòng tránh
- Áp dụng rate limit và quota riêng cho endpoint tốn tài nguyên.
- Dùng hàng đợi, cache hoặc lịch chạy cho báo cáo lớn.
- Từ chối hoặc trì hoãn yêu cầu khi database vượt ngưỡng tải an toàn.
- Ghi log người dùng, tham số truy vấn và thời gian thực thi để điều tra lạm dụng.
- Phân loại endpoint theo độ nặng (nhẹ, trung bình, nặng) và gán quota khác nhau cho từng nhóm.
- Trả mã `429` kèm header `Retry-After` để client biết khi nào được thử lại.
- Đặt cảnh báo khi một người dùng hoặc IP vượt ngưỡng bất thường.
