# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Quang Hữu |
| Mã học viên | 2A202602756 |
| Repo | https://github.com/Anreak/K4-L3B-DAY12-NguyenQuangHuu-2A202602756-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-iedb.onrender.com |
| Platform | Render (Blueprint-managed Docker web service) |
| Ngày deploy | 2026-09-29 (ngày ghi nhận thông tin deployment) |

## Biến Môi Trường Đã Set Trên Cloud

Chỉ ghi tên biến, không ghi giá trị secret. Ảnh dashboard do chủ service cung cấp cho thấy các biến sau:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `AGENT_API_KEY` | ✅ | Secret trong Render Environment|
| `REDIS_URL` | ✅ | Redis service của Render  |
| `RATE_LIMIT_PER_MINUTE` | ✅ | Đã cấu hình  |
| `MONTHLY_BUDGET_USD` | ✅ | Đã cấu hình  |
| `LOG_LEVEL` | ✅ | Đã cấu hình  |
| `PORT` | — | platform cấp |

## Lệnh Kiểm Tra

Các lệnh PowerShell tương ứng:

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

## Kết Quả Chạy Thật
1:
HTTP/1.1 200 OK
Date: Tue, 29 Sep 2026 04:17:29 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 1f57db84-a137-418f
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a4282dac1b54fd3d-SIN
alt-svc: h3=":443"; ma=86400

{"status":"ok","service":"day12-agent","version":"1.0.0"}

2:
HTTP/1.1 200 OK
Date: Tue, 29 Sep 2026 04:18:01 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 2768f7fe-7932-4108
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a4282e76af83f87a-SIN
alt-svc: h3=":443"; ma=86400

{"status":"ready","redis":true}

3:
HTTP/1.1 401 Unauthorized
Date: Tue, 29 Sep 2026 04:18:56 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 16c64fa3-e541-425f
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a4282fce2869ce7a-SIN
alt-svc: h3=":443"; ma=86400

4:
HTTP/1.1 200 OK
Date: Tue, 29 Sep 2026 04:58:19 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
rndr-id: 372df7c0-40f5-447a
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-cache-status: DYNAMIC
CF-RAY: a428697faaf704ed-HKG
alt-svc: h3=":443"; ma=86400

{"answer":"Câu hỏi hay. test thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":1.995e-05,"tokens":{"in":1,"out":33}}

5:
PS D:\VInCode\K4-L3B-DAY12-NguyenQuangHuu-2A202602756-CloudServicesAndDeployment> 1..15 | ForEach-Object {
>>     $code = & curl.exe -s -o NUL -w "%{http_code}" -X POST "$url/ask" `
>>         -H "Content-Type: application/json" `
>>         -H "X-API-Key: $apiKey" `
>>         -H "X-User-Id: sv-test" `
>>         --data-binary "@$bodyFile"
>>     Write-Host -NoNewline "$code "
>> }
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

