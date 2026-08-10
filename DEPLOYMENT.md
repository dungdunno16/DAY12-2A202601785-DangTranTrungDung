# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đặng Trần Trung Dũng |
| Mã học viên | 2A202601785 |
| Repo | https://github.com/dungdunno16/DAY12-2A202601785-DangTranTrungDung.git |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-5c18.up.railway.app |
| Platform | Railway |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 05:46:58 GMT
server: railway-hikari
x-railway-request-id: YBctlx13QQe8TI-UCYBc-A
content-length: 57
x-hikari-trace: sin1.hs0s
x-railway-edge: sin1

{"status":"ok","service":"day12-agent","version":"1.0.0"}

HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 05:48:22 GMT
server: railway-hikari
x-railway-request-id: 9Tl9UqVtSKajHjWS0_TJvA
content-length: 31
x-hikari-trace: sin1.tr00
x-railway-edge: sin1

{"status":"ready","redis":true}

HTTP/2 401 
content-type: application/json
date: Mon, 10 Aug 2026 05:40:39 GMT
server: railway-hikari
x-railway-request-id: WjfPcz68SCuNtMjz9I3ezw
content-length: 39
x-hikari-trace: sin1.nzn2
x-railway-edge: sin1

{"detail":"invalid or missing API key"}

HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 05:43:00 GMT
server: railway-hikari
x-railway-request-id: ZddVEXlqTRix-dpvWUN5dQ
content-length: 282
x-hikari-trace: sin1.tr00
x-railway-edge: sin1
vary: accept-encoding

{"answer":"Ngắn gọn: Hello phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên.","user_id":"anonymous","history_length":0,"cost_usd":2.115e-05,"tokens":{"in":1,"out":35}}

HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 05:51:16 GMT
server: railway-hikari
x-railway-request-id: t7dhTB7CS5aEYc0TnPRhug
content-length: 279
x-hikari-trace: sin1.98a6
x-railway-edge: sin1
vary: accept-encoding

{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

200 200 200 200 200 200 200 200 200 200 429 429 429 429 429 
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl


