# SOP, CORS và Preflight — Same-Origin Policy bảo vệ user qua browser, CORS nới lỏng bằng Access-Control-* headers

> Tags: #SOP #CORS #Origin #Preflight #Simple-Request #Access-Control-Allow-Origin #Access-Control-Allow-Credentials #Access-Control-Max-Age #Vary | Nguồn: `docs/06-browser-web-platform.md` câu 111/113, `docs/07-security.md` câu 121 | Mức: P0

## 1. Định nghĩa chính xác

**Same-Origin Policy (SOP)** là tường lửa cốt lõi của browser: script từ Origin A (`scheme + host + port`) **không được đọc** response/DOM/storage của Origin B. SOP bảo vệ **user** khi user đang đăng nhập `bank.com` nhưng ghé `evil.com` — ngăn `evil.com` đọc `fetch('https://bank.com/api/balance')` dù request vẫn gửi (cookie tự đính kèm).

**CORS (Cross-Origin Resource Sharing)** là cơ chế **HTTP header** cho phép server B **nới lỏng có kiểm soát** SOP cho origin cụ thể: server trả `Access-Control-Allow-Origin`, browser kiểm tra rồi mới cho JS đọc response. CORS **không chặn gửi request** — chỉ chặn **đọc response** nếu thiếu header cho phép.

**Origin** = `scheme://host:port` (port mặc định 443/80 vẫn là một phần origin). `https://app.example.com:443` khác `http://app.example.com:80` khác `https://api.example.com`.

## 2. Cơ chế hoạt động

### 2.1 SOP áp dụng ở đâu

- Chặn: `fetch`/`XHR` đọc cross-origin, `iframe` DOM, `Canvas` tainted (`toDataURL` sau khi vẽ cross-origin image không CORS), `localStorage`/`IndexedDB` per-origin.
- Không chặn: `<img>`, `<script>`, `<link>` load cross-origin (nhưng không đọc nội dung qua JS), `<form>` submit cross-origin (nhưng không đọc response qua JS).

### 2.2 CORS — hai loại request

**Simple Request** (không preflight) — thỏa **tất cả**:
- Method `GET | HEAD | POST`
- Header chỉ safelist: `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` với giá trị `application/x-www-form-urlencoded | multipart/form-data | text/plain`
- Không `ReadableStream`, không custom header (`Authorization` → không simple)
- Browser gửi thẳng request kèm `Origin: https://app.example.com`, kiểm tra `Access-Control-Allow-Origin` ở response.

**Preflight Request** — request "không simple" (`PUT`, `DELETE`, `PATCH`, `Content-Type: application/json`, `Authorization: Bearer …`, custom header):
1. Browser tự gửi **OPTIONS** preflight trước:
   ```
   OPTIONS /api/data HTTP/1.1
   Origin: https://app.example.com
   Access-Control-Request-Method: PUT
   Access-Control-Request-Headers: Content-Type, Authorization
   ```
2. Server trả 204 + CORS policy:
   ```
   HTTP/1.1 204 No Content
   Access-Control-Allow-Origin: https://app.example.com
   Access-Control-Allow-Methods: GET, PUT, DELETE
   Access-Control-Allow-Headers: Content-Type, Authorization
   Access-Control-Allow-Credentials: true
   Access-Control-Max-Age: 86400
   Vary: Origin
   ```
3. Nếu preflight pass, browser mới gửi request thật `PUT` kèm `Origin`. Nếu fail → browser block, JS nhận `TypeError: Failed to fetch` (NetworkError), không thấy status server.

### 2.3 Credentials và `*` rule

- `fetch(..., { credentials: 'include' })` gửi cookie + `Authorization`. Khi đó server **bắt buộc**:
  - `Access-Control-Allow-Origin: https://app.example.com` (origin cụ thể, **không được `*`**)
  - `Access-Control-Allow-Credentials: true`
  - Nếu server trả `Allow-Origin: *` kèm `Allow-Credentials: true` → browser **block** (spec cấm).
- `Allow-Origin: *` chỉ hợp lệ khi `credentials: 'omit'` (không cookie) — dùng cho public API/CDN.

### 2.4 `Access-Control-Max-Age` và `Vary: Origin`

- `Access-Control-Max-Age: 86400` — cache kết quả preflight 86400s, tránh OPTIONS mỗi request. Không set → mỗi request không-simple lại OPTIONS (+1 RTT).
- `Vary: Origin` — **bắt buộc** nếu `Allow-Origin` động theo request `Origin`. Nếu thiếu, CDN cache response cho `Origin: A` rồi trả cho `Origin: B` → B bị CORS fail hoặc lộ data.

### 2.5 Sơ đồ

```
Simple:   JS fetch → Browser thêm Origin → Gửi ngay → Server trả Allow-Origin → Browser check → cho đọc / block
Preflight: JS fetch (PUT+JSON) → Browser gửi OPTIONS → Server 204 Allow-Methods/Headers/Max-Age → (cache) → Gửi PUT thật → check Allow-Origin
Credentials: fetch include → Server phải Allow-Origin: <exact> + Allow-Credentials:true (không *)
```

## 3. Ví dụ tối thiểu

```http
# 3.1 Simple — Content-Type text/plain (không preflight)
GET /api/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
Content-Type: application/json

{"data":"ok"}

# 3.2 Preflight — PUT + application/json
OPTIONS /api/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: PUT, DELETE, GET
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
Access-Control-Allow-Credentials: true
Vary: Origin

# Sau đó
PUT /api/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Content-Type: application/json
Authorization: Bearer xxx

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
```

```bash
# 3.3 curl test

# Preflight
curl -i -X OPTIONS "https://api.example.com/api/data" \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: Content-Type, Authorization"

# Simple với credentials
curl -i "https://api.example.com/api/data" \
  -H "Origin: https://app.example.com" \
  --cookie "session=abc"

# Kiểm tra Vary + Max-Age
curl -s -D - "https://api.example.com/api/data" -H "Origin: https://app.example.com" -o /dev/null | grep -i "access-control\|vary"

# Sai: * với credentials (browser sẽ block)
# Server trả: Access-Control-Allow-Origin: *  + Access-Control-Allow-Credentials: true  → ❌
```

```js
// 3.4 fetch
// Simple — không preflight
fetch('https://api.example.com/api/data', { mode: 'cors' });

// Preflight — trigger OPTIONS do Content-Type: application/json
fetch('https://api.example.com/api/data', {
  method: 'PUT',
  mode: 'cors',
  credentials: 'include', // gửi cookie
  headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer xxx' },
  body: JSON.stringify({ price: 120 }),
});

// Server Express
import cors from 'cors';
app.use(cors({
  origin: 'https://app.example.com', // không *
  credentials: true,
  allowedHeaders: ['Content-Type', 'Authorization'],
  methods: ['GET', 'PUT', 'DELETE', 'PATCH'],
  maxAge: 86400,
}));
app.options('/api/data', cors()); // trả 204 cho preflight
```

```js
// 3.5 Tránh preflight không cần thiết — tránh custom header nếu có thể
// ❌ Thêm X-Request-Id custom → preflight
fetch('/api/data', { headers: { 'X-Request-Id': '123' } });
// ✅ Dùng header safelist nếu được, hoặc chấp nhận preflight + Max-Age
```

## 4. So sánh / Phân loại

| Tiêu chí | Simple Request | Preflight (OPTIONS) |
|----------|----------------|---------------------|
| Điều kiện | `GET/HEAD/POST` + header safelist + `Content-Type` là `text/plain | form-urlencoded | multipart` | `PUT/DELETE/PATCH`, `Content-Type: application/json`, `Authorization`, custom header |
| Flow | Gửi thẳng, check `Allow-Origin` ở response | Gửi `OPTIONS` trước, check `Allow-Methods/Allow-Headers`, rồi mới gửi request thật |
| Cost | 1 RTT | 2 RTT (OPTIONS + thật), giảm bằng `Max-Age` |
| Cache | Không | `Access-Control-Max-Age` cache preflight |

| Header | Mục đích | Giá trị |
|--------|----------|---------|
| `Access-Control-Allow-Origin` | Origin được phép đọc | `https://app.example.com` hoặc `*` (chỉ khi không credentials) |
| `Access-Control-Allow-Credentials` | Cho phép gửi cookie/Authorization | `true` (khi đó `Allow-Origin` không được `*`) |
| `Access-Control-Allow-Methods` | Method cho phép (preflight) | `GET, PUT, DELETE` |
| `Access-Control-Allow-Headers` | Header cho phép (preflight) | `Content-Type, Authorization` |
| `Access-Control-Max-Age` | Cache preflight bao lâu | `86400` (s) |
| `Vary: Origin` | Cache key thêm Origin | Bắt buộc khi `Allow-Origin` động |

| SOP | CORS |
|-----|------|
| Mặc định block đọc cross-origin | Nới lỏng có kiểm soát qua header server |
| Bảo vệ user khỏi `evil.com` đọc `bank.com` | Cho phép `app.example.com` đọc `api.example.com` khi server đồng ý |
| Không cấu hình | Server phải cấu hình `Allow-Origin` + `Vary` |

| `Allow-Origin: *` | `Allow-Origin: https://app.example.com` |
|-------------------|------------------------------------------|
| Dùng cho public API không cookie | Dùng khi `credentials: include` |
| Không kèm `Allow-Credentials: true` | Bắt buộc kèm `Vary: Origin` |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không `Allow-Origin: *` khi có credentials**: spec cấm `*` + `Allow-Credentials: true` → browser block. Nhiều bug production do copy `*` cho API private.
- **Không bỏ `Vary: Origin`**: nếu server echo `Origin` động mà không `Vary`, CDN cache sai origin → user B nhận `Allow-Origin: A` → CORS fail, hoặc ngược lại lộ data.
- **Không tránh preflight bằng `text/plain` hack**: đổi `application/json` thành `text/plain` để thành simple rồi tự parse → mất semantic, server phải handle, không đáng. Thay vào đó set `Max-Age` để cache OPTIONS.
- **Không dùng wildcard cho `Allow-Headers: *` khi preflight cần explicit**: một số browser/CDN yêu cầu list rõ `Content-Type, Authorization`.
- **Không phụ thuộc CORS để bảo mật**: CORS chỉ chặn **đọc** ở browser, không chặn **gửi** (curl/postman vẫn gửi được). Bảo vệ thật phải bằng auth/session/CSRF token, không phải CORS.
- **Khi nào không cần CORS**: cùng origin qua **proxy** (`/api` → `api.example.com` ở Next.js rewrites / Nginx) thì không cross-origin, không cần header. Trade-off: tốn server hop nhưng tránh config CORS phức tạp.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `Allow-Origin: *` với `credentials: include` → block**
  - Triệu chứng: `Access to fetch ... has been blocked by CORS policy: The value of 'Access-Control-Allow-Origin' ... must not be wildcard ... when credentials mode is 'include'`.
  - Fix: server echo origin cụ thể: `Access-Control-Allow-Origin: https://app.example.com` + `Allow-Credentials: true` + `Vary: Origin`.
  - Đo: Network → Response Headers, Console CORS error.

- **Lỗi 2: Thiếu `Vary: Origin` → CDN cache sai**
  - Triệu chứng: `app.example.com` pass, `admin.example.com` fail dù server cho cả hai (do CDN trả cache của A cho B).
  - Fix: thêm `Vary: Origin`.
  - Đo: `curl -H "Origin: https://a.com" -I https://api/data` vs `curl -H "Origin: https://b.com"` → so sánh `Allow-Origin`, check `X-Cache: HIT`.

- **Lỗi 3: Preflight không cache → mỗi PUT lại OPTIONS**
  - Triệu chứng: Waterfall Network mỗi `fetch PUT` có thêm `OPTIONS` 200ms.
  - Fix: `Access-Control-Max-Age: 86400` (24h). Lưu ý Chrome cap ~2h (7200s) dù set lớn hơn.
  - Đo: Network → `OPTIONS` request, Header `Access-Control-Max-Age`, `Size (from preflight cache)` khi cached.

- **Lỗi 4: `Content-Type: application/json` tưởng simple → không, nó trigger preflight**
  - Triệu chứng: `POST` JSON luôn có `OPTIONS`, tưởng bug.
  - Fix: hiểu spec — chỉ `text/plain|form-urlencoded|multipart` mới simple. `application/json` → preflight là đúng. Đừng đổi thành `text/plain` hack.

- **Lỗi 5: Thiếu `Allow-Headers: Authorization` → preflight fail**
  - Triệu chứng: `Request header field authorization is not allowed by Access-Control-Allow-Headers`.
  - Fix: server `Allow-Headers: Content-Type, Authorization`.

- **Lỗi 6: `OPTIONS` trả 200 với body thay vì 204**
  - Triệu chứng: vẫn pass nhưng tốn bandwidth, một số lib expect 204.
  - Fix: `204 No Content` cho OPTIONS.

- **Công cụ**:
  - DevTools **Network** → filter `OPTIONS`, check `Status 204`, Request Headers `Access-Control-Request-*`, Response `Access-Control-Allow-*`, `Vary`.
  - **Console** → CORS error chi tiết.
  - `curl -X OPTIONS -H "Origin: https://app.example.com" -H "Access-Control-Request-Method: PUT" -H "Access-Control-Request-Headers: Content-Type" -i https://api/data`
  - Application → không liên quan, CORS là network header.

## 7. Câu hỏi tự kiểm tra

1. Định nghĩa Origin và vì sao `https://app.example.com` khác `https://api.example.com` dù cùng `example.com`? SOP bảo vệ ai khỏi điều gì?
2. Phân biệt Simple Request vs Preflight — điều kiện nào trigger `OPTIONS`? `Content-Type: application/json` có phải simple không? `Authorization` header thì sao?
3. Vì sao `Access-Control-Allow-Origin: *` không dùng được với `credentials: include`? `Vary: Origin` và `Access-Control-Max-Age` để làm gì trong thực chiến CDN/preflight cache?

<details>
<summary>Đáp án 30s</summary>

1. **Origin = scheme + host + port** (`https://app.example.com:443`). SOP so sánh exact 3 phần, nên `app` vs `api` khác host → cross-origin. SOP bảo vệ **user** (không phải server): ngăn `evil.com` dùng JS đọc response của `bank.com` khi browser tự gửi cookie `bank.com` kèm request, tránh lộ `balance`/DOM/storage.
2. **Simple**: `GET/HEAD/POST` + header safelist (`Accept`, `Accept-Language`, `Content-Language`, `Content-Type` chỉ `text/plain|form-urlencoded|multipart`) + không custom header. **Preflight**: mọi thứ khác (`PUT/DELETE/PATCH`, `Content-Type: application/json`, `Authorization`, `X-Custom`) → browser gửi `OPTIONS` với `Access-Control-Request-Method/Headers`, server trả `Allow-Methods/Allow-Headers/Max-Age` 204 rồi mới gửi thật. `application/json` và `Authorization` đều **không simple** → luôn preflight.
3. Spec CORS cấm `Allow-Origin: *` khi `Allow-Credentials: true` vì `*` nghĩa là cho mọi origin đọc response kèm cookie → lộ session. Khi `credentials: include`, server phải echo `Allow-Origin: https://app.example.com` cụ thể + `Allow-Credentials: true`. `Vary: Origin` báo CDN cache key thêm `Origin`, nếu thiếu CDN trả nhầm `Allow-Origin: A` cho `B`. `Max-Age: 86400` cache kết quả OPTIONS 24h, tránh +1 RTT mỗi request không-simple.

</details>

---
*Tham khảo chi tiết: `docs/06-browser-web-platform.md` — Câu 111, 113. `docs/07-security.md` — Câu 121. Spec: [Fetch — CORS protocol](https://fetch.spec.whatwg.org/#http-cors-protocol), [MDN — CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).*
