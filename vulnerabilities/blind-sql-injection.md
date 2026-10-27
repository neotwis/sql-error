# Blind SQL Injection

## Tóm tắt
Blind SQL Injection là biến thể của SQL Injection trong đó ứng dụng không trả trực tiếp dữ liệu hoặc lỗi SQL, nhưng phản hồi vẫn để lộ tín hiệu gián tiếp như đúng/sai, thời gian phản hồi hoặc khác biệt nội dung trang.

## Tác động
- Trích xuất dữ liệu từng phần thông qua quan sát phản hồi.
- Gây chậm hệ thống bằng các truy vấn có điều kiện tốn thời gian.
- Khó phát hiện hơn SQL Injection thông thường vì không nhất thiết sinh lỗi rõ ràng.

## Nguyên nhân phổ biến
- Vẫn tồn tại truy vấn động không tham số hóa.
- Cơ chế xử lý lỗi bị ẩn nhưng logic phản hồi vẫn phụ thuộc vào kết quả SQL.
- Thiếu giới hạn thời gian thực thi truy vấn và giám sát bất thường.

## Ví dụ tình huống

Một chức năng kiểm tra email đã được đăng ký hay chưa. Ứng dụng không bao giờ trả dữ liệu người dùng hay lỗi SQL, chỉ trả về "đã tồn tại" hoặc "chưa tồn tại".

### Code dễ bị tấn công (TypeScript/Express)

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";

const app = express();
const pool = new Pool();

app.get("/check-email", async (req: Request, res: Response) => {
  const email = String(req.query.email ?? "");

  try {
    // LỖI: nối chuỗi trực tiếp đầu vào của người dùng vào câu truy vấn
    const query = `SELECT id FROM users WHERE email = '${email}'`;
    const result = await pool.query(query);

    // Không lộ dữ liệu hay lỗi SQL, nhưng phản hồi vẫn phụ thuộc kết quả truy vấn
    if (result.rowCount && result.rowCount > 0) {
      return res.json({ message: "Email đã tồn tại" });
    }
    return res.json({ message: "Email chưa tồn tại" });
  } catch {
    // Lỗi bị ẩn, nhưng vẫn có thể bị suy luận qua nhánh phản hồi khác
    return res.status(500).json({ message: "Lỗi hệ thống" });
  }
});

app.listen(3000);
```

### Vì sao vẫn bị khai thác

Dù không trả dữ liệu, các nhánh phản hồi khác nhau đã tạo ra một "tín hiệu đúng/sai". Nếu người gửi chèn thêm một điều kiện vào `email`, câu truy vấn trở thành:

```sql
-- Đầu vào: x' AND <điều kiện bất kỳ> --
SELECT id FROM users WHERE email = 'x' AND <điều kiện bất kỳ> --'
```

- Điều kiện đúng thì có kết quả và phản hồi là "đã tồn tại".
- Điều kiện sai thì không có kết quả và phản hồi là "chưa tồn tại".
- Điều kiện làm sai cú pháp hoặc gây lỗi thì phản hồi là "Lỗi hệ thống" (HTTP 500), đây cũng là một tín hiệu.

Quan sát sự khác biệt này nhiều lần, kẻ tấn công có thể suy ra thông tin từng phần (boolean-based). Với dạng time-based, tín hiệu là độ trễ phản hồi thay vì nội dung.

### Code đã vá

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";
import rateLimit from "express-rate-limit";

const app = express();

const pool = new Pool({
  statement_timeout: 2000,        // giới hạn thời gian thực thi truy vấn (ms)
  query_timeout: 3000,
  max: 10,                        // giới hạn số kết nối
});

// Giới hạn tần suất cho endpoint kiểm tra tồn tại
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 10,
});

app.get("/check-email", limiter, async (req: Request, res: Response) => {
  const email = String(req.query.email ?? "");

  // Phản hồi dùng chung cho mọi trường hợp
  const genericResponse = {
    message: "Nếu email hợp lệ, chúng tôi sẽ gửi hướng dẫn qua email.",
  };

  try {
    // ĐÚNG: parameterized query, đầu vào luôn được coi là dữ liệu, không phải mã SQL
    await pool.query("SELECT 1 FROM users WHERE email = $1 LIMIT 1", [email]);
  } catch (err) {
    // Chỉ ghi log nội bộ, không để lộ chi tiết ra ngoài
    console.error("check-email failed", err);
  }

  // Luôn trả cùng một nội dung và mã trạng thái
  return res.status(200).json(genericResponse);
});

app.listen(3000);
```

Những điểm đã thay đổi:
- Dùng placeholder `$1` thay vì nối chuỗi, nên đầu vào không thể thay đổi cấu trúc câu truy vấn.
- Phản hồi và mã trạng thái giống nhau trong mọi trường hợp (kể cả khi lỗi), nên không còn tín hiệu đúng/sai.
- Đặt `statement_timeout` và giới hạn số kết nối để hạn chế truy vấn chạy lâu.
- Thêm rate limiting để làm khó việc dò từng phần.

## Biện pháp phòng tránh
- Dùng parameterized query cho mọi dữ liệu đầu vào.
- Chuẩn hóa phản hồi lỗi để không tiết lộ trạng thái truy vấn.
- Đặt timeout cho truy vấn và giới hạn tài nguyên mỗi phiên/kết nối.
- Theo dõi các mẫu truy vấn lặp lại, phản hồi chậm bất thường hoặc tỷ lệ lỗi cao.
- Áp dụng rate limiting cho các endpoint kiểm tra tồn tại (email, username).
- Cấp cho tài khoản DB quyền tối thiểu cần thiết.
