# Lock Contention và Deadlock trong SQL

## Tóm tắt
Lock contention xảy ra khi nhiều giao dịch tranh chấp cùng tài nguyên dữ liệu. Deadlock xảy ra khi các giao dịch chờ lẫn nhau và không thể tiếp tục nếu hệ quản trị cơ sở dữ liệu không hủy một giao dịch. Cả hai đều có thể làm giảm hiệu năng hoặc gây gián đoạn dịch vụ.

## Tác động
- Tăng thời gian phản hồi của truy vấn và giao dịch.
- Giao dịch hợp lệ bị rollback do deadlock.
- Hàng đợi kết nối tăng, làm ứng dụng chậm hoặc lỗi timeout.
- Người dùng gặp lỗi không ổn định khi tải cao.

## Nguyên nhân phổ biến
- Giao dịch giữ khóa quá lâu do xử lý nhiều logic trong transaction.
- Cập nhật bảng theo thứ tự không nhất quán giữa các luồng nghiệp vụ.
- Thiếu index khiến lệnh cập nhật phải quét và khóa phạm vi lớn.
- Mức cô lập giao dịch quá cao so với nhu cầu thực tế.

## Ví dụ tình huống

Một hệ thống ngân hàng có chức năng chuyển tiền giữa hai tài khoản. Khi nhiều giao dịch chạy đồng thời, ví dụ A chuyển cho B và B chuyển cho A cùng lúc, hệ thống thỉnh thoảng báo lỗi deadlock và treo một số request. Ngoài ra, giao dịch còn gọi dịch vụ bên ngoài (gửi thông báo) khi đang giữ khóa.

### Code dễ gặp deadlock (TypeScript/Express)

```typescript
import express, { Request, Response } from "express";
import { Pool } from "pg";

const app = express();
app.use(express.json());
const pool = new Pool();

app.post("/transfer", async (req: Request, res: Response) => {
  const { fromId, toId, amount } = req.body;
  const client = await pool.connect();

  try {
    await client.query("BEGIN");

    // LỖI 1: khóa tài khoản theo thứ tự "nguồn trước, đích sau"
    // => A->B và B->A chạy cùng lúc sẽ khóa chéo nhau (deadlock)
    await client.query(
      "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE",
      [fromId]
    );
    await client.query(
      "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE",
      [toId]
    );

    await client.query(
      "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
      [amount, fromId]
    );
    await client.query(
      "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
      [amount, toId]
    );

    // LỖI 2: gọi dịch vụ bên ngoài khi transaction còn đang giữ khóa
    await fetch("https://notify.example.com/send", {
      method: "POST",
      body: JSON.stringify({ fromId, toId, amount }),
    });

    await client.query("COMMIT");
    res.json({ message: "OK" });
  } catch (err) {
    await client.query("ROLLBACK");
    // LỖI 3: không phân biệt deadlock với lỗi khác, không retry
    res.status(500).json({ message: "Chuyển tiền thất bại" });
  } finally {
    client.release();
  }
});

app.listen(3000);
```

### Vì sao xảy ra deadlock

Hai request chạy gần như đồng thời:

```text
Thời điểm | Giao dịch 1 (A -> B)         | Giao dịch 2 (B -> A)
----------|------------------------------|------------------------------
t1        | Khóa tài khoản A             |
t2        |                              | Khóa tài khoản B
t3        | Chờ khóa B (đang bị T2 giữ)  |
t4        |                              | Chờ khóa A (đang bị T1 giữ)
t5        | => Hai bên chờ nhau, deadlock; DB hủy một giao dịch
```

Việc gọi dịch vụ bên ngoài trong transaction làm thời gian giữ khóa kéo dài, khiến xác suất tranh chấp tăng mạnh khi tải cao.

### Code đã vá

```typescript
import express, { Request, Response } from "express";
import { Pool, PoolClient } from "pg";

const app = express();
app.use(express.json());

const pool = new Pool({
  max: 10,
  statement_timeout: 5000, // giới hạn thời gian thực thi mỗi câu lệnh
});

const DEADLOCK_CODE = "40P01";
const SERIALIZATION_CODE = "40001";
const MAX_RETRIES = 3;

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

async function transfer(
  client: PoolClient,
  fromId: number,
  toId: number,
  amount: number
) {
  await client.query("BEGIN");
  // Không chờ khóa quá lâu, tránh giữ kết nối vô hạn
  await client.query("SET LOCAL lock_timeout = '2s'");

  // 1. Luôn khóa theo thứ tự id tăng dần, bất kể chiều chuyển tiền
  const [firstId, secondId] = fromId < toId ? [fromId, toId] : [toId, fromId];
  await client.query("SELECT 1 FROM accounts WHERE id = $1 FOR UPDATE", [firstId]);
  await client.query("SELECT 1 FROM accounts WHERE id = $1 FOR UPDATE", [secondId]);

  // 2. Kiểm tra số dư và cập nhật bằng điều kiện, tránh số dư âm
  const debit = await client.query(
    "UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
    [amount, fromId]
  );
  if (debit.rowCount === 0) {
    throw new Error("INSUFFICIENT_FUNDS");
  }
  await client.query(
    "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
    [amount, toId]
  );

  // 3. Ghi sự kiện vào bảng outbox cùng transaction, không gọi dịch vụ ngoài
  await client.query(
    "INSERT INTO outbox (type, payload) VALUES ('transfer_done', $1)",
    [JSON.stringify({ fromId, toId, amount })]
  );

  await client.query("COMMIT");
}

app.post("/transfer", async (req: Request, res: Response) => {
  const { fromId, toId, amount } = req.body;

  if (fromId === toId || !(amount > 0)) {
    return res.status(400).json({ message: "Dữ liệu không hợp lệ" });
  }

  // 4. Retry có backoff cho lỗi deadlock có thể khôi phục
  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    const client = await pool.connect();
    try {
      await transfer(client, fromId, toId, amount);
      return res.json({ message: "OK" });
    } catch (err: any) {
      await client.query("ROLLBACK");

      if (err.message === "INSUFFICIENT_FUNDS") {
        return res.status(422).json({ message: "Số dư không đủ" });
      }

      const retryable = err.code === DEADLOCK_CODE || err.code === SERIALIZATION_CODE;
      if (retryable && attempt < MAX_RETRIES) {
        await sleep(50 * 2 ** attempt + Math.random() * 50); // backoff + jitter
        continue;
      }

      console.error("transfer failed", { code: err.code, attempt });
      return res.status(503).json({ message: "Hệ thống đang bận, vui lòng thử lại" });
    } finally {
      client.release();
    }
  }
});

// Worker riêng đọc outbox và gửi thông báo, nằm ngoài transaction chuyển tiền
async function processOutbox() {
  const { rows } = await pool.query(
    `SELECT id, payload FROM outbox
     WHERE sent = false ORDER BY id LIMIT 20
     FOR UPDATE SKIP LOCKED`
  );
  for (const row of rows) {
    await fetch("https://notify.example.com/send", {
      method: "POST",
      body: JSON.stringify(row.payload),
    });
    await pool.query("UPDATE outbox SET sent = true WHERE id = $1", [row.id]);
  }
}
setInterval(processOutbox, 2000);

app.listen(3000);
```

Index hỗ trợ để lệnh cập nhật chỉ khóa đúng dòng cần thiết:

```sql
-- id là khóa chính nên đã có index; bảng outbox cần index cho worker
CREATE INDEX idx_outbox_pending ON outbox (id) WHERE sent = false;
```

Theo dõi các tiến trình đang chờ khóa và giữ khóa lâu (PostgreSQL):

```sql
SELECT
  blocked.pid        AS blocked_pid,
  blocked.query      AS blocked_query,
  blocking.pid       AS blocking_pid,
  blocking.query     AS blocking_query,
  now() - blocking.xact_start AS blocking_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY (pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';
```

Những điểm đã thay đổi:
- Luôn khóa tài khoản theo thứ tự `id` tăng dần, loại bỏ khả năng khóa chéo giữa A->B và B->A.
- Transaction ngắn gọn, bỏ lệnh gọi dịch vụ bên ngoài và chuyển sang mẫu outbox.
- Đặt `lock_timeout` và `statement_timeout` để không chờ vô hạn.
- Retry có backoff và jitter cho lỗi deadlock (`40P01`) và serialization failure (`40001`).
- Kiểm tra số dư ngay trong câu `UPDATE` bằng điều kiện `balance >= $1`.
- Phân biệt rõ lỗi nghiệp vụ (hết tiền) với lỗi hệ thống.

## Biện pháp phòng tránh
- Giữ transaction ngắn và tránh gọi dịch vụ bên ngoài trong transaction.
- Cập nhật tài nguyên theo thứ tự nhất quán trong toàn hệ thống.
- Bổ sung index cho điều kiện WHERE trong UPDATE và DELETE.
- Thiết kế cơ chế retry an toàn cho lỗi deadlock có thể khôi phục.
- Theo dõi lock wait, deadlock report và truy vấn giữ khóa lâu.
- Bật `log_lock_waits` và đặt `deadlock_timeout` phù hợp để ghi nhận tranh chấp khóa.
- Chỉ dùng mức cô lập cao (`SERIALIZABLE`) khi thật sự cần, và luôn kèm cơ chế retry.
