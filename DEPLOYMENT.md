# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Phi Nhật |
| Mã học viên | 2A202602658 |
| Repo | https://github.com/nhatgudboi/K4-L3A-DAY12-NguyenPhiNhat-2A202602658-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-5750.onrender.com |
| Platform | Render |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard Blueprint của Render, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Key Value Redis (`day12-redis`) liên kết tự động qua render.yaml |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-5750.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-5750.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-5750.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-5750.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-5750.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
1. Liveness Probe:
HTTP/1.1 200 OK
content-length: 59
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

2. Readiness Probe:
HTTP/1.1 200 OK
content-length: 32
content-type: application/json
{"status":"ready","redis":true}

3. Không có API Key:
HTTP/1.1 401 Unauthorized
content-length: 39
content-type: application/json
{"detail":"invalid or missing API key"}

4. Có API Key hợp lệ:
HTTP/1.1 200 OK
content-length: 298
content-type: application/json
{
  "answer": "Theo mình hiểu, Deploy là gì liên quan tới cách hệ thống được đóng gói và vận hành...",
  "user_id": "sv-test",
  "history_length": 0,
  "cost_usd": 0.00003015,
  "tokens": {"in": 5, "out": 49}
}

5. Rate Limiting Test (15 requests liên tiếp):
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
(10 request đầu tiên trả về 200 OK trong hạn mức, 5 request tiếp theo bị chặn với mã 429 Too Many Requests theo đúng thuật toán sliding window)
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
