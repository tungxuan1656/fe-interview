# HTTP Status Codes — 401/403 vs 404/410 vs 400/422, 204/304 và Retry Semantics

> Tags: #http-status #401 #403 #404 #410 #400 #422 #204 #304 #retry #SEO | Nguồn: `docs/08-api-networking.md` câu 137-142 | Mức: P0

## 1. Định nghĩa chính xác

**Status Code** là hợp đồng 3 chữ số giữa client-server: `2xx` success, `3xx` redirect/cache, `4xx` client error, `5xx` server error. Frontend phải handle đúng để UX, cache, retry, SEO chuẩn — không dùng `200 { success: false }` cho mọi thứ.

- **401 Unauthorized** (thực chất **Unauthenticated**): **chưa xác thực** hoặc token sai/hết hạn — server không biết bạn là ai. Client phải **login/refresh** (`WWW-Authenticate: Bearer`).
- **403 Forbidden**: **đã xác thực** (biết bạn là ai) nhưng **không có quyền** — dù login lại vẫn 403 trừ khi được cấp quyền.
- **404 Not Found**: không tìm thấy hiện tại, **không nói** vĩnh viễn hay tạm — có thể sai URL/chưa tạo/đã xóa nhưng không tiết lộ. Bot sẽ thử lại.
- **410 Gone**: **đã từng tồn tại** nhưng **xóa vĩnh viễn**, không bao giờ quay lại — CDN/bot xóa cache/index nhanh, client đừng retry.
- **400 Bad Request**: **syntax sai** — JSON malformed, query param sai kiểu, header thiếu — lỗi ở **parser**, chưa tới validation logic.
- **422 Unprocessable Entity** (WebDAV, dùng rộng rãi): **syntax đúng** nhưng **semantic sai** — validation fail (`email` sai format, `price < 0`). Server hiểu nhưng không xử lý do business rule.
- **204 No Content**: success **không body** (DELETE, PUT). **304 Not Modified**: cache hit với `If-None-Match`/`If-Modified-Since` — không body, dùng cache.

## 2. Cơ chế hoạt động

### 2.1 401 vs 403

```
GET /api/orders
Authorization: Bearer expired_token
→ 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token", error_description="expired"

GET /api/admin/users
Authorization: Bearer valid_token_for_user_role
→ 403 Forbidden
{ error: "Bạn không có quyền admin" }
```
- 401 → **redirect login/refresh** (thử `refreshToken()` 1 lần, fail thì `/login?next=...`).
- 403 → **hiện "Không có quyền"** + ẩn nút, đừng retry login.
- Nhiều backend sai: trả 403 cho cả chưa login → frontend không biết đi login hay báo quyền. Chuẩn là 401 cho unauthenticated, 403 cho unauthorized.

### 2.2 404 vs 410 — SEO

|  | 404 | 410 |
|---|---|---|
| Ý nghĩa | Không tìm thấy (không rõ vĩnh viễn) | Đã xóa vĩnh viễn, biết chắc |
| Retry | Có thể thử lại | Không nên |
| SEO | Google bot **thử lại** vài lần, tốn crawl budget | Google **xóa index ngay**, tiết kiệm budget |
| Dùng khi | Sai URL, chưa tạo | Xóa sản phẩm, đổi slug vĩnh viễn, GDPR delete |

- 410 cần server **nhớ đã từng có** (thêm logic/DB flag `deleted_at`), nên nhiều team lười chỉ 404 → mất tín hiệu cho bot/CDN.
- Khi **đổi slug** vĩnh viễn: nên `301 Moved Permanently` sang slug mới (giữ SEO juice), không 410, trừ khi xóa thật.

### 2.3 400 vs 422

```
# 400 — parser không hiểu
POST /api/products
{ name: "Shoes", price: }  // JSON invalid
→ 400 { error: "Invalid JSON" }

# 422 — parser ok, validation fail
POST /api/products
{ "name":"Shoes","price":-10,"email":"not-email" }
→ 422 { errors: { price:"Giá phải >=0", email:"Email không hợp lệ" } }
```
- Tách 400/422 giúp frontend: **400 → lỗi code** (log, toast chung), **422 → hiện lỗi từng field** (`setFieldErrors`). Zod validate trước khi gửi để tránh 422.

### 2.4 204 / 304 và các 2xx/3xx/5xx

- `200 OK`: GET/PATCH có body.
- `201 Created`: POST tạo xong, kèm `Location: /products/123`.
- `204 No Content`: DELETE/PUT thành công **không body** (đừng gửi `{}`).
- `301 Moved Permanently`: đổi URL vĩnh viễn, SEO chuyển juice.
- `302 Found`: tạm.
- `304 Not Modified`: `If-None-Match: "etag"` khớp → không body, dùng cache.
- `429 Too Many Requests` + `Retry-After: 60` + `X-RateLimit-*` headers.
- `500`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` — server error, có thể retry nếu idempotent.

### 2.5 Retry semantics

**Nên retry** (tạm thời, idempotent, an toàn):
- `GET` (safe), `PUT`/`DELETE` (idempotent), `408`, `429`, `502`, `503`, `504`, network error/timeout.

**Không retry**:
- `400`, `401`, `403`, `404`, `422`, `409` — retry cũng fail (data sai).
- `POST` không idempotent — retry tạo duplicate, chỉ retry nếu có `Idempotency-Key`.

Phải **giới hạn số lần + exponential backoff + jitter + tôn trọng `Retry-After`**.

## 3. Ví dụ tối thiểu

```http
# 3.1 401 vs 403
GET /api/orders HTTP/1.1
Authorization: Bearer expired
→ 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token"

GET /api/admin/users HTTP/1.1
Authorization: Bearer valid_user_token
→ 403 Forbidden
{ "error": "Bạn không có quyền admin" }

# 3.2 404 vs 410
GET /api/products/999999
→ 404 Not Found
{ "error": "Không tìm thấy sản phẩm" }

GET /api/products/old-slug-123
→ 410 Gone
{ "error": "Sản phẩm đã xóa vĩnh viễn", "suggest": "/products" }

# 3.3 400 vs 422
POST /api/products
Content-Type: application/json
{ "price": }
→ 400 Bad Request
{ "error": "Invalid JSON" }

POST /api/products
{ "price": -10, "email": "not-email" }
→ 422 Unprocessable Entity
{ "errors": { "price": "Giá phải >=0", "email": "Email không hợp lệ" } }

# 3.4 204 / 304
DELETE /api/products/123
→ 204 No Content

GET /api/products
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
→ 304 Not Modified

# 3.5 429 + Retry-After
GET /api/products
→ 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1714000000
```

```ts
// 3.6 Frontend handle
async function handleResponse(res: Response) {
  if (res.ok) { // 200-299
    if (res.status === 204) return null;
    if (res.status === 201) console.log('Created at', res.headers.get('Location'));
    return res.json();
  }
  switch (res.status) {
    case 400: throw new Error('Request sai cú pháp');
    case 401: location.href = '/login'; break;
    case 403: showToast('Không có quyền'); break;
    case 404: showNotFound(); break;
    case 410: queryClient.removeQueries({ queryKey:['product', id] }); navigate('/products'); break;
    case 422: { const { errors } = await res.json(); showValidation(errors); break; }
    case 429: { const retryAfter = res.headers.get('Retry-After'); showToast(`Thử lại sau ${retryAfter}s`); break; }
    case 304: return cachedData;
    case 500: showToast('Lỗi server, thử lại sau'); break;
  }
}

// 3.7 Interceptor 401 refresh
async function apiFetch(url: string, opts: RequestInit = {}) {
  const res = await fetch(url, { ...opts, headers: { ...opts.headers, Authorization: `Bearer ${accessToken}` } });
  if (res.status === 401) {
    const ok = await refreshToken();
    if (ok) return apiFetch(url, opts); // retry 1 lần
    location.href = '/login?next=' + encodeURIComponent(location.pathname);
    throw new Error('Unauthorized');
  }
  if (res.status === 403) {
    showToast('Bạn không có quyền');
    throw new Error('Forbidden');
  }
  return res;
}

// 3.8 Validation trước khi gửi
import { z } from 'zod';
const schema = z.object({ name: z.string().min(1), price: z.number().min(0), email: z.string().email() });
const parsed = schema.safeParse(data);
if (!parsed.success) setFieldErrors(parsed.error.flatten().fieldErrors);

// 3.9 Retry helper — chỉ retry idempotent + 5xx/429/408
async function fetchWithRetry(url: string, opts: RequestInit & { retries?: number } = {}) {
  const { retries = 3, ...fetchOpts } = opts;
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      const res = await fetch(url, fetchOpts);
      if ([502,503,504,429,408].includes(res.status) && attempt < retries) {
        const retryAfter = res.headers.get('Retry-After');
        const delay = retryAfter ? Number(retryAfter)*1000 : Math.min(1000 * 2 ** attempt + Math.random()*1000, 10000);
        await new Promise(r=>setTimeout(r, delay));
        continue;
      }
      if (res.status >= 400 && res.status < 500) return res; // không retry 4xx
      return res;
    } catch (e) {
      const isIdempotent = ['GET','PUT','DELETE','HEAD'].includes(fetchOpts.method ?? 'GET');
      if (attempt < retries && isIdempotent) {
        await new Promise(r=>setTimeout(r, Math.min(1000 * 2 ** attempt, 10000)));
        continue;
      }
      throw e;
    }
  }
  throw new Error('Max retries');
}

// 3.10 Exponential backoff + jitter + Retry-After
function exponentialBackoff(attempt: number, base=1000, max=30000) {
  const exp = Math.min(base * 2 ** attempt, max);
  return Math.random() * exp; // full jitter 0..exp
}
```

```js
// 3.11 Curl
curl -i https://api.example.com/api/products/999 -H "Authorization: Bearer expired"
curl -i -X DELETE https://api.example.com/api/products/123 -H "Authorization: Bearer valid"
curl -i -X POST https://api.example.com/api/products -H "Content-Type: application/json" -d '{ "price": -1 }'
curl -i https://api.example.com/api/products -H "If-None-Match: \"abc123\""
```

## 4. So sánh / Phân loại

| Status | Ý nghĩa | Body? | Retry? | Frontend action |
|--------|---------|-------|--------|-----------------|
| **200** | OK, có body | Có | Không | Render |
| **201** | Created | Có + `Location` | Không | Redirect/detail |
| **204** | No Content (DELETE) | **Không** | Không | Xóa cache |
| **301** | Moved Permanently | Không | Không | Update URL, SEO juice |
| **304** | Not Modified | **Không** | Không | Dùng cache (`If-None-Match`) |
| **400** | Bad Request (syntax) | Có | Không | Log, toast chung |
| **401** | Unauthenticated | Có | Không (trừ refresh) | Refresh → login |
| **403** | Forbidden (đã auth, không quyền) | Có | Không | Ẩn nút, toast quyền |
| **404** | Not Found (không rõ vĩnh viễn) | Có | Có thể | NotFound + search |
| **410** | Gone (xóa vĩnh viễn) | Có | Không | Xóa cache, redirect, SEO xóa index |
| **422** | Validation semantic fail | Có `errors` | Không | Hiện lỗi từng field |
| **429** | Too Many Requests | Có | Có sau `Retry-After` | Backoff |
| **500/502/503/504** | Server error | Có | Có nếu idempotent | Backoff + jitter |

| 401 vs 403 | 404 vs 410 | 400 vs 422 |
|------------|------------|------------|
| 401: chưa biết bạn là ai → login | 404: không biết có từng tồn tại không | 400: parser không hiểu (JSON invalid) |
| 403: biết bạn là ai, không có quyền → đừng login lại | 410: biết chắc đã xóa vĩnh viễn | 422: parser hiểu, validation fail |

| 204 vs 304 |
|------------|
| 204: success không body (DELETE) | 304: cache hit, không body, dùng cache |

| Nên retry | Không retry |
|-----------|-------------|
| `GET`/`PUT`/`DELETE` + `408/429/502/503/504` + network error | `400/401/403/404/422/409` + `POST` không `Idempotency-Key` |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng `200 { success: false }` cho mọi thứ**: mất semantic, `fetch`/`axios` interceptor, CDN, retry, cache không hoạt động đúng. Dùng status chuẩn để `res.ok` (200-299) và `switch(res.status)` hoạt động.
- **Không trả 403 cho chưa login**: frontend không biết redirect login hay báo quyền. Tuân thủ 401 (unauthenticated) vs 403 (unauthorized).
- **Không trả 404 khi muốn che giấu resource**: nếu `GET /admin/users/123` cho user thường, trả **404 thay vì 403** để không tiết lộ tồn tại resource (security through obscurity cho IDOR).
- **Không trả 410 nếu không chắc vĩnh viễn**: 410 cần server nhớ `deleted_at`, nếu có thể restore (soft delete) thì 404 an toàn hơn. 410 sai → bot xóa index oan.
- **Không dùng 301 khi chỉ tạm**: `301` cache vĩnh viễn ở browser/CDN, đổi lại khó. Tạm dùng `302`.
- **Không retry vô hạn**: thundering herd khi server 503 → 1000 client retry cùng lúc DDoS. Phải **max 3 + backoff + jitter + `Retry-After` + circuit breaker**.
- **Không retry `POST` mặc định**: chỉ `GET/PUT/DELETE` + 5xx/429. `POST` chỉ retry với `Idempotency-Key`.
- **Khi nào dùng `409 Conflict` thay 400/422**: khi version conflict (`If-Match` fail) hoặc trùng unique (`email đã tồn tại`) — semantic rõ hơn.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Backend trả 403 cho cả chưa login → frontend không redirect**
  - Triệu chứng: hết hạn token nhưng không về `/login`, chỉ toast "Không có quyền".
  - Fix: backend 401 cho unauthenticated + `WWW-Authenticate: Bearer`.
  - Đo: Network → Status, Response `WWW-Authenticate`.

- **Lỗi 2: Chỉ trả 404, không 410 → SEO tốn crawl budget**
  - Triệu chứng: xóa 1000 sản phẩm, Google vẫn crawl 404 hàng tuần.
  - Fix: sản phẩm xóa vĩnh viễn → `410 Gone` + xóa cache.
  - Đo: Google Search Console → Coverage, `curl -I https://shop/products/old` → `410`.

- **Lỗi 3: Trả `200 { success:false }` thay 422 → validation không hiện field**
  - Triệu chứng: `res.ok` true, frontend không vào nhánh `422`.
  - Fix: trả `422` với `errors: { field: msg }`.
  - Đo: Network → Status phải `422`, không `200`.

- **Lỗi 4: `204` trả body → client parse fail**
  - Triệu chứng: `await res.json()` trên `204` → `SyntaxError: Unexpected end`.
  - Fix: `204` không body, frontend `if (res.status===204) return null`.
  - Đo: Network → `Content-Length: 0`.

- **Lỗi 5: `304` không gửi `ETag`/`If-None-Match`**
  - Triệu chứng: mỗi `GET` 200 full body thay vì 304.
  - Fix: server `ETag: "abc"` + `Cache-Control`, client `If-None-Match`, hoặc `fetch` để browser tự.
  - Đo: Request Headers `If-None-Match`, Response `304`.

- **Lỗi 6: Retry `POST` không `Idempotency-Key` → duplicate payment**
  - Triệu chứng: timeout → retry → trừ tiền 2 lần.
  - Fix: `Idempotency-Key` + server lưu 24h.
  - Đo: Network → 2 `POST` cùng key → cùng `id`.

- **Lỗi 7: Không tôn trọng `Retry-After` → bị ban**
  - Triệu chứng: `429` nhưng retry ngay 1s → `429` tiếp, IP bị block.
  - Fix: parse `Retry-After` (giây hoặc date), delay đúng.
  - Đo: Response Headers `Retry-After`, `X-RateLimit-*`.

- **Công cụ**:
  - DevTools **Network** → Status, Headers (`Location`, `ETag`, `Retry-After`, `WWW-Authenticate`), Preview.
  - `curl -i -H "Authorization: Bearer xxx" https://api/...`
  - `curl -H "If-None-Match: \"abc\"" -i https://api/products`
  - TanStack Query Devtools, Axios interceptor logs.
  - SEO: Google Search Console, `curl -I` check `404` vs `410` vs `301`.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt `401` vs `403` — khi nào redirect login/refresh, khi nào hiện "Không có quyền" và ẩn nút? Vì sao tên `401 Unauthorized` lại thực chất là `Unauthenticated`?
2. `404` vs `410` khác gì về retry và SEO (Google xóa index, crawl budget)? Khi nào nên `301` thay vì `410` khi đổi slug?
3. `400` vs `422` khác gì (parser syntax vs validation semantic) và `204` vs `304` (không body) dùng khi nào? Nêu retry semantics: method nào + status nào được retry, `POST` khi nào không được, và `Retry-After` + backoff/jitter để làm gì?

<details>
<summary>Đáp án 30s</summary>

1. **401**: chưa xác thực (token thiếu/sai/hết hạn, server không biết bạn là ai) → thử `refreshToken()` 1 lần, fail thì `location.href='/login'`, response có `WWW-Authenticate: Bearer`. **403**: đã xác thực nhưng không có quyền (user gọi `DELETE /admin`) → đừng login lại, hiện toast + ẩn nút. Tên `401 Unauthorized` là lịch sử HTTP/1.0, thực chất là **Unauthenticated** (chưa biết ai), còn 403 mới là **Unauthorized** (biết ai nhưng không cho).
2. **404**: không tìm thấy, không nói vĩnh viễn, bot sẽ thử lại, tốn crawl budget. **410**: đã xóa vĩnh viễn, biết chắc không quay lại, bot **xóa index ngay** và CDN xóa cache, client đừng retry. Đổi slug vĩnh viễn nên `301 Moved Permanently` sang slug mới (giữ SEO juice) thay vì `410`; chỉ `410` khi xóa thật không thay thế.
3. **400**: syntax sai (JSON malformed, parser chưa tới logic) → lỗi code. **422**: syntax đúng nhưng semantic fail (validation `price < 0`) → hiện `errors` từng field. **204**: success không body (DELETE). **304**: cache hit với `If-None-Match` → không body, dùng cache. **Retry**: chỉ `GET/PUT/DELETE` (idempotent) + `408/429/502/503/504`/network error; không retry `400/401/403/404/422/409` và `POST` không `Idempotency-Key` (sợ duplicate). `Retry-After` (giây hoặc date) + exponential backoff `base*2^attempt` + jitter random để tránh thundering herd (1000 client cùng retry 1 lúc DDoS).

</details>

---
*Tham khảo chi tiết: `docs/08-api-networking.md` — Câu 137-142. Spec: [MDN — HTTP Status](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status), [RFC 7231 §6](https://httpwg.org/specs/rfc7231.html#status.codes).*
