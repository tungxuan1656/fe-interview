# 08. API & Networking - 16 Câu Hỏi Senior

> 16 câu hỏi về API và networking (Câu 135-150) - từ RESTful, status code, retry, race condition đến pagination và optimistic update. Senior thiết kế API cho UX mượt, không chỉ cho đúng spec.

## Mục lục

- [Câu 135: RESTful là gì? 6 nguyên tắc thiết kế API](#câu-135-restful-là-gì-6-nguyên-tắc-thiết-kế-api)
- [Câu 136: PUT vs PATCH vs POST - idempotent và partial update](#câu-136-put-vs-patch-vs-post---idempotent-và-partial-update)
- [Câu 137: Status Code hay dùng - 2xx, 3xx, 4xx, 5xx](#câu-137-status-code-hay-dùng---2xx-3xx-4xx-5xx)
- [Câu 138: 401 Unauthorized vs 403 Forbidden](#câu-138-401-unauthorized-vs-403-forbidden)
- [Câu 139: 404 Not Found vs 410 Gone](#câu-139-404-not-found-vs-410-gone)
- [Câu 140: 400 Bad Request vs 422 Unprocessable Entity](#câu-140-400-bad-request-vs-422-unprocessable-entity)
- [Câu 141: Retry - khi nào nên và khi nào không nên retry?](#câu-141-retry---khi-nào-nên-và-khi-nào-không-nên-retry)
- [Câu 142: Exponential Backoff, Jitter và Retry-After](#câu-142-exponential-backoff-jitter-và-retry-after)
- [Câu 143: Race Condition - stale response và cách fix](#câu-143-race-condition---stale-response-và-cách-fix)
- [Câu 144: Autocomplete - debounce, cache và race condition](#câu-144-autocomplete---debounce-cache-và-race-condition)
- [Câu 145: Request Cancellation và AbortController](#câu-145-request-cancellation-và-abortcontroller)
- [Câu 146: Offset Pagination vs Cursor Pagination](#câu-146-offset-pagination-vs-cursor-pagination)
- [Câu 147: Cursor Pagination thực chiến và Infinite Scroll](#câu-147-cursor-pagination-thực-chiến-và-infinite-scroll)
- [Câu 148: Optimistic Update và Rollback](#câu-148-optimistic-update-và-rollback)
- [Câu 149: ETag, Conditional Request và Cache API](#câu-149-etag-conditional-request-và-cache-api)
- [Câu 150: API Contract - versioning, idempotency-key và rate limit](#câu-150-api-contract---versioning-idempotency-key-và-rate-limit)

---

### Câu 135: RESTful là gì? 6 nguyên tắc thiết kế API

**Trả lời Senior:**
REST (Representational State Transfer) là **kiến trúc cho API** dùng HTTP làm application protocol, không phải chỉ CRUD qua HTTP. RESTful API coi mọi thứ là **resource** (danh từ), thao tác qua **method** (động từ), và **stateless**.

6 constraints của REST (Fielding):

1.  **Client-Server**: tách UI và data.
2.  **Stateless**: mỗi request chứa đủ info, server không lưu session client.
3.  **Cacheable**: response đánh dấu cache được hay không (`Cache-Control`).
4.  **Uniform Interface**: resource định danh bằng URI, thao tác qua method, self-descriptive message, HATEOAS.
5.  **Layered System**: qua proxy, CDN, gateway không ảnh hưởng.
6.  **Code on Demand** (optional): server trả code cho client chạy.

Thực chiến RESTful:

- **Danh từ số nhiều**: `/products`, `/users/123/orders` (không `/getProducts`).
- **Method đúng**: `GET /products` (list), `POST /products` (tạo), `GET /products/123` (chi tiết), `PUT /products/123` (thay toàn bộ), `PATCH /products/123` (sửa 1 phần), `DELETE /products/123`.
- **Status code đúng**: `200`, `201`, `204`, `400`, `404`.
- **Filter/sort/paginate qua query**: `GET /products?category=shoes&sort=price&order=asc&page=2&limit=20`.
- **Version**: `/api/v1/products`.
- **Stateless**: `Authorization: Bearer <token>` mỗi request, không session server.

```http
# ✅ RESTful
GET    /api/v1/products?category=shoes&limit=20&cursor=abc
POST   /api/v1/products          { name: "Shoes", price: 100 } -> 201 { id: "123" }
GET    /api/v1/products/123      -> 200 { id: "123", name: "Shoes" }
PATCH  /api/v1/products/123      { price: 120 } -> 200
DELETE /api/v1/products/123      -> 204
GET    /api/v1/users/123/orders

# ❌ Không RESTful
GET /api/getProducts
POST /api/createProduct
GET /api/products?action=delete&id=123
POST /api/products/123/updatePrice
```

```typescript
// Frontend fetch RESTful
const res = await fetch('/api/v1/products?category=shoes&limit=20', {
  headers: { Authorization: `Bearer ${token}` },
});
if (!res.ok) throw new Error(await res.text());
const { data, nextCursor } = await res.json();

// HATEOAS (nâng cao) - response chứa link
// { data: { id: "123" }, links: { self: "/products/123", orders: "/users/123/orders" } }
```

**Trade-off:** REST đơn giản, cache được, nhưng với query phức tạp (nested, graph) thì GraphQL linh hoạt hơn. Dùng REST cho CRUD, GraphQL cho data graph.

**Câu hỏi đào sâu:** Vì sao REST phải stateless? HATEOAS có thực sự cần không? REST khác RPC thế nào?

---

### Câu 136: PUT vs PATCH vs POST - idempotent và partial update

**Trả lời Senior:**
Ba method đều tạo/sửa, nhưng khác **idempotent** và **payload**:

- **POST**: **không idempotent**, tạo resource mới, server quyết id. `POST /products` 2 lần tạo 2 product. Dùng cho create, hoặc action không fit REST (`POST /orders/123/cancel`).
- **PUT**: **idempotent**, thay **toàn bộ** resource, client quyết id. `PUT /products/123 { name, price, desc }` - gửi thiếu field thì field đó bị xóa/null. Gọi 10 lần kết quả như 1 lần.
- **PATCH**: **không idempotent (thường)**, sửa **một phần**, chỉ gửi field đổi. `PATCH /products/123 { price: 120 }` - chỉ đổi price, giữ lại name/desc. Tiết kiệm bandwidth, nhưng phải xử lý merge.

Idempotent nghĩa là gọi nhiều lần cùng request thì server state như gọi 1 lần (không tạo duplicate, không cộng dồn).

```http
# POST - tạo mới, server sinh id
POST /products
{ "name": "Shoes", "price": 100 } -> 201 { "id": "abc", "name": "Shoes" }
# Gọi lại -> 201 { "id": "def" } (2 sản phẩm)

# PUT - thay toàn bộ, idempotent
PUT /products/abc
{ "name": "Shoes", "price": 120, "desc": "Nice" } -> 200
# Gọi lại y hệt -> 200, vẫn 1 sản phẩm, price 120

# PATCH - sửa 1 phần
PATCH /products/abc
{ "price": 130 } -> 200 { "name": "Shoes", "price": 130, "desc": "Nice" }
# Chỉ price đổi, name/desc giữ

# JSON Patch (RFC 6902) cho PATCH chuẩn
PATCH /products/abc
Content-Type: application/json-patch+json
[
  { "op": "replace", "path": "/price", "value": 130 },
  { "op": "add", "path": "/tags/0", "value": "sale" }
]
```

```typescript
// Frontend
// Tạo
await fetch('/api/products', { method: 'POST', body: JSON.stringify({ name: 'Shoes', price: 100 }) });
// Sửa toàn bộ (ít dùng)
await fetch('/api/products/123', { method: 'PUT', body: JSON.stringify(product) });
// Sửa 1 phần (hay dùng)
await fetch('/api/products/123', { method: 'PATCH', body: JSON.stringify({ price: 130 }) });

// Lưu ý: PUT/PATCH cần If-Match để tránh lost update (optimistic locking)
await fetch('/api/products/123', {
  method: 'PATCH',
  headers: { 'If-Match': etag }, // 412 nếu đã bị sửa bởi người khác
  body: JSON.stringify({ price: 130 }),
});
```

**Trade-off:** `PUT` an toàn retry (idempotent) nhưng tốn băng thông. `PATCH` tiết kiệm nhưng phải handle `null` vs `undefined` (xóa hay giữ?). Nhiều team chỉ dùng `POST` + `PATCH` cho đơn giản.

**Câu hỏi đào sâu:** Vì sao PUT idempotent còn PATCH thường không? Khi nào dùng `application/merge-patch+json` vs `json-patch+json`?

---

### Câu 137: Status Code hay dùng - 2xx, 3xx, 4xx, 5xx

**Trả lời Senior:**
Status code là **hợp đồng** giữa client và server, frontend phải handle đúng để UX chuẩn.

- **2xx Success**: `200 OK` (GET, PATCH thành công, có body), `201 Created` (POST tạo xong, có `Location: /products/123`), `204 No Content` (DELETE, PUT thành công không body).
- **3xx Redirect**: `301 Moved Permanently` (đổi URL vĩnh viễn, SEO), `302 Found` (tạm), `304 Not Modified` (cache hit, body rỗng, dùng với `If-None-Match`).
- **4xx Client Error**: `400 Bad Request` (syntax sai), `401 Unauthorized` (chưa auth), `403 Forbidden` (đã auth nhưng không quyền), `404 Not Found`, `409 Conflict` (trùng, version conflict), `422 Unprocessable Entity` (validation fail), `429 Too Many Requests` (rate limit, kèm `Retry-After`).
- **5xx Server Error**: `500 Internal Server Error`, `502 Bad Gateway` (proxy lỗi), `503 Service Unavailable` (quá tải/maintain).

```typescript
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
    case 409: showToast('Trùng dữ liệu, vui lòng refresh'); break;
    case 422: {
      const { errors } = await res.json();
      showValidation(errors); // { email: "đã tồn tại" }
      break;
    }
    case 429: {
      const retryAfter = res.headers.get('Retry-After');
      showToast(`Thử lại sau ${retryAfter}s`);
      break;
    }
    case 500: showToast('Lỗi server, thử lại sau'); break;
    case 304: return cachedData; // Not Modified
  }
}

// Fetch với status
const res = await fetch('/api/products', { method: 'POST', body: JSON.stringify(data) });
if (res.status === 201) { /* created */ }
if (res.status === 422) { /* validation */ }
```

**Trade-off:** Đừng dùng `200` cho mọi thứ kèm `{ success: false }` - mất semantic, không tận dụng `fetch`/`axios` interceptor. Dùng code chuẩn để CDN, retry, cache hoạt động đúng.

**Câu hỏi đào sâu:** Vì sao `201` nên kèm `Location` header? `429` khác `503` thế nào?

---

### Câu 138: 401 Unauthorized vs 403 Forbidden

**Trả lời Senior:**
Hai code hay nhầm nhất, khác ở **đã xác thực chưa**:

- **401 Unauthorized**: **chưa xác thực** hoặc token sai/hết hạn. Server không biết bạn là ai. Client phải **đi login/refresh**. Dù tên là Unauthorized, thực chất là **Unauthenticated**. Response nên có `WWW-Authenticate: Bearer`.
- **403 Forbidden**: **đã xác thực** (biết bạn là ai) nhưng **không có quyền** làm hành động đó. Ví dụ user thường gọi `DELETE /admin/users`. Dù có login lại cũng vẫn 403, trừ khi được cấp quyền.

Frontend handle khác nhau: 401 → redirect login/refresh, 403 → hiện "Không có quyền" + ẩn nút.

```http
# 401 - chưa đăng nhập hoặc token hết hạn
GET /api/orders
Authorization: Bearer expired_token
-> 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token", error_description="expired"

# 403 - đã đăng nhập nhưng không đủ quyền
GET /api/admin/users
Authorization: Bearer valid_token_for_user_role
-> 403 Forbidden
{ "error": "Bạn không có quyền admin" }
```

```typescript
// Interceptor
async function apiFetch(url: string, opts: RequestInit = {}) {
  const res = await fetch(url, { ...opts, headers: { ...opts.headers, Authorization: `Bearer ${accessToken}` } });
  if (res.status === 401) {
    // Thử refresh
    const ok = await refreshToken();
    if (ok) return apiFetch(url, opts); // retry 1 lần
    location.href = '/login?next=' + encodeURIComponent(location.pathname);
    throw new Error('Unauthorized');
  }
  if (res.status === 403) {
    // Đừng retry, hiện UI
    showToast('Bạn không có quyền thực hiện hành động này');
    throw new Error('Forbidden');
  }
  return res;
}

// UI: ẩn nút nếu không có quyền
function DeleteButton({ canDelete }: { canDelete: boolean }) {
  if (!canDelete) return null; // hoặc disabled + tooltip "Cần quyền admin"
  return <button onClick={handleDelete}>Xóa</button>;
}
```

**Trade-off:** Nhiều backend trả 403 cho cả chưa login (sai), làm frontend không biết đi login hay báo quyền. Chuẩn là 401 cho unauthenticated, 403 cho unauthorized.

**Câu hỏi đào sâu:** Vì sao 401 tên là Unauthorized nhưng lại là Unauthenticated? Khi nào backend nên trả 404 thay vì 403 để che giấu resource?

---

### Câu 139: 404 Not Found vs 410 Gone

**Trả lời Senior:**
Cả hai đều "không tìm thấy", nhưng khác **vĩnh viễn hay tạm**:

- **404 Not Found**: server **không tìm thấy** resource hiện tại, nhưng **không nói** là có từng tồn tại hay sẽ có lại. Có thể do sai URL, chưa tạo, hoặc đã xóa nhưng server không muốn tiết lộ. Client có thể thử lại sau, hoặc do user gõ sai.
- **410 Gone**: resource **đã từng tồn tại** nhưng **đã xóa vĩnh viễn** và **không bao giờ quay lại**, server biết chắc. Báo cho client **đừng thử lại**, và cho **CDN/bot** xóa cache, Google **gỡ index** nhanh hơn 404. Dùng khi xóa sản phẩm, xóa bài, đổi slug vĩnh viễn.

Frontend: 404 → hiện NotFound + gợi ý search, 410 → hiện "Đã xóa vĩnh viễn" + redirect tới danh mục.

```http
# 404 - không biết có từng tồn tại không
GET /api/products/999999 -> 404 Not Found
{ "error": "Không tìm thấy sản phẩm" }

# 410 - đã xóa vĩnh viễn
GET /api/products/old-slug-123 -> 410 Gone
{ "error": "Sản phẩm đã xóa vĩnh viễn", "suggest": "/products" }
# Header: Cache-Control: no-cache (đừng cache 410 quá lâu nếu có thể restore)
```

```typescript
// Frontend handle
async function getProduct(id: string) {
  const res = await fetch(`/api/products/${id}`);
  if (res.status === 404) {
    showNotFound(`Sản phẩm ${id} không tồn tại`);
    // Gợi ý search hoặc redirect
  }
  if (res.status === 410) {
    showGone(`Sản phẩm đã bị xóa vĩnh viễn`);
    // Xóa khỏi cache local, redirect
    queryClient.removeQueries({ queryKey: ['product', id] });
    navigate('/products');
  }
  return res.json();
}

// SEO: với 410, Google bot sẽ xóa index nhanh hơn, không tốn crawl budget
// Với 404, Google sẽ thử lại vài lần trước khi xóa
```

| | 404 | 410 |
|---|---|---|
| Ý nghĩa | Không tìm thấy (không rõ vĩnh viễn) | Đã xóa vĩnh viễn |
| Retry | Có thể thử lại | Không nên |
| SEO | Bot thử lại | Bot xóa index ngay |
| Dùng khi | Sai URL, chưa tạo | Xóa vĩnh viễn, đổi slug |

**Trade-off:** 410 chính xác hơn nhưng cần server nhớ đã từng có gì (thêm logic). Nhiều team lười chỉ trả 404 cho mọi case, mất tín hiệu cho bot/CDN.

**Câu hỏi đào sâu:** Vì sao 410 tốt hơn 404 cho SEO khi xóa sản phẩm? Khi nào nên 301 redirect thay vì 410?

---

### Câu 140: 400 Bad Request vs 422 Unprocessable Entity

**Trả lời Senior:**
Cả hai đều lỗi do client, nhưng khác **lỗi gì**:

- **400 Bad Request**: **syntax sai**, server không hiểu request. Ví dụ JSON malformed (`{ name: }`), query param sai kiểu (`?page=abc`), header thiếu. Lỗi ở **tầng parser**, chưa tới validation logic.
- **422 Unprocessable Entity** (WebDAV, nhưng dùng rộng rãi): **syntax đúng** (JSON parse được) nhưng **semantic sai** - validation fail. Ví dụ `email` không đúng format, `password` quá ngắn, `price < 0`. Server hiểu request nhưng không xử lý được vì business rule.

Nhiều API (đặc biệt REST) dùng 400 cho cả hai, nhưng tách ra giúp frontend handle chính xác: 400 → lỗi code, 422 → hiện lỗi validation cho user.

```http
# 400 - JSON sai
POST /api/products
Content-Type: application/json
{ name: "Shoes", price: }  // JSON invalid
-> 400 Bad Request
{ "error": "Invalid JSON" }

# 400 - thiếu field bắt buộc ở parser
POST /api/products
{} -> 400 { "error": "Missing required field: name" } // nếu coi là bad request

# 422 - validation semantic
POST /api/products
{ "name": "Shoes", "price": -10, "email": "not-email" }
-> 422 Unprocessable Entity
{
  "errors": {
    "price": "Giá phải >= 0",
    "email": "Email không hợp lệ"
  }
}
```

```typescript
// Frontend
async function createProduct(data: Product) {
  const res = await fetch('/api/products', { method: 'POST', body: JSON.stringify(data), headers: { 'Content-Type': 'application/json' } });
  if (res.status === 400) {
    // Lỗi code, không hiện cho user, log
    console.error('Bad Request', await res.text());
    showToast('Dữ liệu gửi sai định dạng');
  }
  if (res.status === 422) {
    const { errors } = await res.json();
    // Hiện lỗi từng field
    setFieldErrors(errors); // { price: "Giá phải >=0" }
  }
}

// Zod validation trước khi gửi để tránh 422
import { z } from 'zod';
const schema = z.object({ name: z.string().min(1), price: z.number().min(0), email: z.string().email() });
const parsed = schema.safeParse(data);
if (!parsed.success) setFieldErrors(parsed.error.flatten().fieldErrors);
```

**Trade-off:** Tách 400/422 rõ ràng nhưng cần team thống nhất. Nhiều framework (Laravel, Rails) mặc định 422 cho validation, Express hay trả 400. Quan trọng là document.

**Câu hỏi đào sâu:** Vì sao 422 ban đầu của WebDAV nhưng lại phổ biến cho REST? Khi nào nên trả 400 kèm `errors` chi tiết thay vì 422?

---

### Câu 141: Retry - khi nào nên và khi nào không nên retry?

**Trả lời Senior:**
Retry không phải luôn tốt - retry sai làm **nhân đôi side effect** (trừ tiền 2 lần, tạo 2 đơn).

**Nên retry**: lỗi **tạm thời**, **idempotent**, và **an toàn**:

- `GET` (idempotent, safe) - luôn retry được.
- `PUT`/`DELETE` (idempotent) - retry được.
- Network error (`fetch failed`, `timeout`), `408 Request Timeout`, `429 Too Many Requests`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`.

**Không nên retry**: lỗi **client** hoặc **non-idempotent**:

- `400`, `401`, `403`, `404`, `422` - retry cũng fail, do data sai.
- `POST` tạo resource (không idempotent) - retry tạo duplicate nếu server đã xử lý nhưng response mất. Chỉ retry POST nếu có **Idempotency-Key**.
- `409 Conflict` - cần user giải quyết.

Và phải **giới hạn số lần + backoff**, không retry vô hạn.

```typescript
// Retry helper
async function fetchWithRetry(url: string, opts: RequestInit & { retries?: number } = {}) {
  const { retries = 3, ...fetchOpts } = opts;
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      const res = await fetch(url, fetchOpts);
      // Chỉ retry 5xx, 429, 408
      if ([502, 503, 504, 429, 408].includes(res.status) && attempt < retries) {
        await sleep(backoff(attempt));
        continue;
      }
      // 4xx không retry
      if (res.status >= 400 && res.status < 500) return res;
      return res;
    } catch (e) {
      // Network error - retry nếu còn attempt và là GET/PUT/DELETE
      const isIdempotent = ['GET', 'PUT', 'DELETE', 'HEAD'].includes(fetchOpts.method ?? 'GET');
      if (attempt < retries && isIdempotent) {
        await sleep(backoff(attempt));
        continue;
      }
      throw e;
    }
  }
  throw new Error('Max retries');
}
function backoff(attempt: number) { return Math.min(1000 * 2 ** attempt, 10000); }
function sleep(ms: number) { return new Promise(r => setTimeout(r, ms)); }

// POST với Idempotency-Key để retry an toàn
await fetch('/api/orders', {
  method: 'POST',
  headers: { 'Idempotency-Key': crypto.randomUUID(), 'Content-Type': 'application/json' },
  body: JSON.stringify(order),
});
// Server lưu key 24h, nếu trùng thì trả lại response cũ, không tạo mới
```

**Trade-off:** Retry tăng reliability nhưng tăng load server khi đang quá tải (thundering herd). Phải có **circuit breaker** và **backoff**.

**Câu hỏi đào sâu:** Vì sao POST không nên retry mặc định? `Idempotency-Key` hoạt động thế nào để POST retry an toàn?

---

### Câu 142: Exponential Backoff, Jitter và Retry-After

**Trả lời Senior:**
Retry ngay lập tức khi server 503 sẽ **DDoS chính mình** (1000 client cùng retry giây 1). **Exponential backoff** tăng delay theo cấp số nhân, **jitter** random để tránh **thundering herd** (đàn cùng retry 1 lúc), và tôn trọng `Retry-After` header.

Công thức: `delay = min(base * 2^attempt + jitter, max)`. Jitter có 2 loại: **full jitter** (`random(0, delay)`) và **equal jitter** (`delay/2 + random(0, delay/2)`).

`Retry-After` là header server trả khi `429`/`503`: `Retry-After: 60` (giây) hoặc `Retry-After: Wed, 21 Oct 2026 07:28:00 GMT` (date). Client phải ưu tiên nó hơn backoff tính.

```typescript
// Exponential backoff + full jitter
function exponentialBackoff(attempt: number, base = 1000, max = 30000) {
  const exp = Math.min(base * 2 ** attempt, max);
  const jitter = Math.random() * exp; // full jitter 0..exp
  // hoặc equal jitter: exp/2 + Math.random() * exp/2
  return jitter;
}

// Retry tôn trọng Retry-After
async function fetchWithBackoff(url: string, opts: RequestInit = {}, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const res = await fetch(url, opts);
    if (res.ok) return res;
    if (![429, 502, 503, 504].includes(res.status) || attempt === maxRetries) return res;

    let delay: number;
    const retryAfter = res.headers.get('Retry-After');
    if (retryAfter) {
      // Retry-After có thể là giây hoặc date
      const seconds = Number(retryAfter);
      if (!isNaN(seconds)) delay = seconds * 1000;
      else delay = new Date(retryAfter).getTime() - Date.now();
    } else {
      delay = exponentialBackoff(attempt);
    }
    console.log(`Retry ${attempt + 1} sau ${Math.round(delay)}ms`);
    await new Promise(r => setTimeout(r, delay));
  }
  throw new Error('Max retries');
}

// Ví dụ timeline: attempt 0 -> ~0-1s, attempt 1 -> ~0-2s, attempt 2 -> ~0-4s (random)
// 3 client cùng fail sẽ retry lệch nhau nhờ jitter, không cùng 1s

// AWS, Google API đều dùng exponential backoff + jitter
```

**Trade-off:** Backoff làm tổng thời gian tăng, nhưng giảm 90% load khi server recovery. Jitter làm delay không đều, nhưng tránh spike. Luôn set `max` để không đợi 5 phút.

**Câu hỏi đào sâu:** Full jitter vs equal jitter khác gì? Vì sao không dùng fixed delay 1s cho mọi retry?

---

### Câu 143: Race Condition - stale response và cách fix

**Trả lời Senior:**
Race condition xảy ra khi **2 request cùng loại, response về không theo thứ tự gửi**, làm UI hiện data cũ (stale). Ví dụ user gõ "a" → fetch `?q=a`, rồi gõ "ab" → fetch `?q=ab`, nhưng `?q=a` về sau → UI hiện kết quả "a" dù input là "ab".

Nguyên nhân: network latency ngẫu nhiên, không đảm bảo FIFO.

Fix:

1.  **Bỏ qua stale**: gắn `requestId` tăng dần, chỉ update nếu `id === latestId`.
2.  **Cancel request cũ**: `AbortController` abort fetch trước khi gửi mới.
3.  **Dùng SWR/TanStack Query**: tự dedupe và chỉ nhận latest.

```typescript
// ❌ Race
let query = 'a';
fetch(`/api/search?q=${query}`).then(r => r.json()).then(data => setResults(data));
query = 'ab';
fetch(`/api/search?q=${query}`).then(r => r.json()).then(data => setResults(data));
// Nếu "a" về sau, setResults("a") ghi đè "ab"

// ✅ Fix 1: requestId
let latestId = 0;
async function search(q: string) {
  const id = ++latestId;
  const res = await fetch(`/api/search?q=${q}`);
  const data = await res.json();
  if (id === latestId) setResults(data); // chỉ latest mới update
}

// ✅ Fix 2: AbortController
let controller: AbortController | null = null;
async function search(q: string) {
  controller?.abort();
  controller = new AbortController();
  try {
    const res = await fetch(`/api/search?q=${q}`, { signal: controller.signal });
    const data = await res.json();
    setResults(data);
  } catch (e) {
    if ((e as Error).name === 'AbortError') return; // bỏ qua
    throw e;
  }
}

// ✅ Fix 3: TanStack Query tự handle
const { data } = useQuery({ queryKey: ['search', query], queryFn: () => fetch(`/api/search?q=${query}`).then(r=>r.json()) });
// TanStack chỉ set data của queryKey mới nhất

// Với useEffect
useEffect(() => {
  let ignore = false;
  fetch(`/api/search?q=${query}`).then(r=>r.json()).then(data=>{ if(!ignore) setResults(data); });
  return () => { ignore = true; };
}, [query]);
```

**Trade-off:** `requestId` đơn giản nhưng vẫn tốn network (request cũ vẫn chạy). `Abort` tiết kiệm network nhưng cần handle `AbortError`. TanStack Query là chuẩn cho app lớn.

**Câu hỏi đào sâu:** Vì sao `let ignore = false` trong `useEffect` cleanup lại fix được race? `AbortController` khác `ignore flag` thế nào về network?

---

### Câu 144: Autocomplete - debounce, cache và race condition

**Trả lời Senior:**
Autocomplete là **bài tập tổng hợp** của debounce + cache + race + cancel. Yêu cầu: gõ mượt, ít request, không stale, có cache.

Flow Senior:

1.  **Debounce 200-300ms**: đợi user ngừng gõ mới fetch, giảm 80% request.
2.  **Cache**: `Map<string, Result>` hoặc TanStack Query, gõ lại "ab" thì lấy cache ngay, không fetch.
3.  **Cancel + Race**: abort request cũ, chỉ nhận latest (như câu 143).
4.  **Min length + loading + empty**: chỉ fetch khi `query.length >= 2`, hiện skeleton, không fetch khi rỗng.

```typescript
function useAutocomplete(query: string, delay = 300) {
  const [results, setResults] = useState<string[]>([]);
  const [loading, setLoading] = useState(false);
  const cache = useRef<Map<string, string[]>>(new Map());
  const controller = useRef<AbortController | null>(null);

  const debouncedQuery = useDebounce(query, delay); // hook debounce

  useEffect(() => {
    if (debouncedQuery.length < 2) { setResults([]); return; }
    if (cache.current.has(debouncedQuery)) { setResults(cache.current.get(debouncedQuery)!); return; }

    controller.current?.abort();
    controller.current = new AbortController();
    setLoading(true);

    fetch(`/api/search?q=${encodeURIComponent(debouncedQuery)}`, { signal: controller.current.signal })
      .then(r => r.json())
      .then(data => {
        cache.current.set(debouncedQuery, data);
        setResults(data);
      })
      .catch(e => { if ((e as Error).name !== 'AbortError') console.error(e); })
      .finally(() => setLoading(false));

    return () => controller.current?.abort();
  }, [debouncedQuery]);

  return { results, loading };
}

// useDebounce
function useDebounce<T>(value: T, delay: number) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);
  return debounced;
}

// Với TanStack Query - gọn hơn
function Autocomplete({ query }: { query: string }) {
  const debounced = useDebounce(query, 300);
  const { data, isFetching } = useQuery({
    queryKey: ['autocomplete', debounced],
    queryFn: ({ signal }) => fetch(`/api/search?q=${debounced}`, { signal }).then(r=>r.json()),
    enabled: debounced.length >= 2,
    staleTime: 5 * 60 * 1000, // cache 5 phút
  });
  return <>{isFetching ? <Spinner/> : <List items={data ?? []}/>}</>;
}
```

**Trade-off:** Debounce 300ms cân bằng mượt và tiết kiệm, nhưng với search nhạy (như Google) có thể 150ms + cache. Cache Map không giới hạn sẽ phình, nên LRU hoặc TanStack `gcTime`.

**Câu hỏi đào sâu:** Vì sao autocomplete cần `minLength` 2? Làm sao highlight match mà không XSS?

---

### Câu 145: Request Cancellation và AbortController

**Trả lời Senior:**
`fetch` không cancel được mặc định, phải dùng **AbortController** - chuẩn browser để hủy request, timeout, và cleanup khi unmount. Quan trọng cho autocomplete, navigate, và tránh setState on unmounted.

Cơ chế: `controller = new AbortController()`, `signal` truyền vào `fetch`, `controller.abort()` sẽ reject fetch với `AbortError`, browser **hủy network** (không tốn bandwidth). Một controller có thể abort nhiều fetch cùng signal, và có thể `reason`.

Dùng cho: **unmount cancel**, **race cancel**, **timeout**, **user cancel**.

```typescript
// 1. Unmount cancel - tránh setState sau unmount
useEffect(() => {
  const controller = new AbortController();
  fetch('/api/products', { signal: controller.signal })
    .then(r => r.json())
    .then(setProducts)
    .catch(e => { if (e.name !== 'AbortError') console.error(e); });
  return () => controller.abort(); // unmount -> cancel
}, []);

// 2. Timeout với AbortSignal.timeout (mới) hoặc manual
// Modern:
const res = await fetch('/api/products', { signal: AbortSignal.timeout(5000) }); // auto abort sau 5s
// Manual:
function fetchWithTimeout(url: string, ms: number) {
  const controller = new AbortController();
  const t = setTimeout(() => controller.abort(new Error('Timeout')), ms);
  return fetch(url, { signal: controller.signal }).finally(() => clearTimeout(t));
}

// 3. User cancel
function Search() {
  const controllerRef = useRef<AbortController | null>(null);
  const onSearch = async (q: string) => {
    controllerRef.current?.abort();
    controllerRef.current = new AbortController();
    try {
      const res = await fetch(`/api/search?q=${q}`, { signal: controllerRef.current.signal });
      setResults(await res.json());
    } catch (e) {
      if ((e as Error).name === 'AbortError') return; // user đã cancel, bỏ qua
      showError(e);
    }
  };
  const onCancel = () => controllerRef.current?.abort();
  return <><button onClick={onCancel}>Hủy</button></>;
}

// 4. Axios cũng support signal
// axios.get('/api/products', { signal: controller.signal })

// Với TanStack Query, signal tự truyền
useQuery({ queryKey: ['products'], queryFn: ({ signal }) => fetch('/api/products', { signal }).then(r=>r.json()) });
```

**Trade-off:** Abort tiết kiệm network và tránh race, nhưng phải handle `AbortError` để không hiện lỗi giả. `AbortSignal.timeout` gọn nhưng không phải browser nào cũng có (polyfill).

**Câu hỏi đào sâu:** `AbortController` khác `CancelToken` của Axios cũ thế nào? Vì sao phải `return () => controller.abort()` trong `useEffect`?

---

### Câu 146: Offset Pagination vs Cursor Pagination

**Trả lời Senior:**
Hai cách phân trang cho list lớn, khác ở **cách xác định trang** và **consistency**.

- **Offset**: `GET /products?offset=20&limit=20` (hoặc `page=2`). Dễ hiểu, nhảy tới trang bất kỳ (`page=5`), nhưng **chậm khi offset lớn** (DB phải scan `offset` row), và **lệch khi data thay đổi** (thêm/xóa giữa chừng làm duplicate/missing). Ví dụ offset 20, có 2 item mới chèn đầu → item 20-21 bị lặp.
- **Cursor**: `GET /products?cursor=eyJpZCI6IjEyMyJ9&limit=20` (cursor là base64 của `id` hoặc `createdAt+id`). Server trả `nextCursor`, client dùng để lấy tiếp. **Nhanh** (dùng index `WHERE id > cursor`), **consistent** (không lệch khi thêm/xóa), nhưng **không nhảy trang**, chỉ next/prev, và cần sort ổn định.

Bảng:

| | Offset | Cursor |
|---|---|---|
| Query | `offset, limit` | `cursor, limit` |
| Nhảy trang | Có (`page=5`) | Không (chỉ next) |
| Performance | Chậm khi offset lớn (scan) | Nhanh (index) |
| Consistency | Lệch khi insert/delete | Ổn định |
| Dùng khi | Admin table cần nhảy trang, ít data | Feed, infinite scroll, nhiều data |

```http
# Offset
GET /api/products?offset=0&limit=20  -> { data: [...20], total: 100 }
GET /api/products?offset=20&limit=20 -> { data: [...20] }
# DB: SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 20 -- scan 20

# Cursor (keyset)
GET /api/products?limit=20               -> { data: [...20], nextCursor: "eyJpZCI6IjIwIn0=" }
GET /api/products?cursor=eyJpZCI6IjIwIn0=&limit=20 -> { data: [...20], nextCursor: "eyJpZCI6IjQwIn0=" }
# DB: SELECT * FROM products WHERE id > '20' ORDER BY id LIMIT 20 -- index seek
```

```typescript
// Frontend offset
function useOffset(page: number) {
  const { data } = useQuery({ queryKey: ['products', page], queryFn: () => fetch(`/api/products?offset=${page*20}&limit=20`).then(r=>r.json()) });
  return data; // { data, total }
}

// Frontend cursor - infinite
function useCursor() {
  const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
    queryKey: ['products'],
    queryFn: ({ pageParam }) => fetch(`/api/products?cursor=${pageParam ?? ''}&limit=20`).then(r=>r.json()),
    getNextPageParam: last => last.nextCursor,
    initialPageParam: null as string | null,
  });
  return { items: data?.pages.flatMap(p=>p.data) ?? [], fetchNextPage, hasNextPage };
}
```

**Trade-off:** Offset đơn giản cho admin, cursor là chuẩn cho feed/timeline. Nhiều API hỗ trợ cả hai.

**Câu hỏi đào sâu:** Vì sao offset chậm khi `OFFSET 100000`? Cursor cần sort thế nào để không miss khi `createdAt` trùng?

---

### Câu 147: Cursor Pagination thực chiến và Infinite Scroll

**Trả lời Senior:**
Cursor pagination là nền cho **infinite scroll** (như Facebook, TikTok). Thực chiến cần 4 thứ: **cursor ổn định**, **dedup**, **loading state**, và **IntersectionObserver** để trigger.

Cursor phải **ổn định và unique**: dùng `createdAt + id` hoặc `id` nếu id tăng dần. Không dùng `offset` làm cursor. Server encode cursor base64 JSON, client không cần hiểu.

Infinite scroll: khi user scroll gần bottom, tự `fetchNextPage`. Dùng `useInfiniteQuery` của TanStack Query để tự merge pages, cache, và dedup.

```typescript
// Server - Node/Postgres
app.get('/api/products', async (req, res) => {
  const limit = Math.min(Number(req.query.limit) || 20, 50);
  const cursor = req.query.cursor ? JSON.parse(Buffer.from(req.query.cursor as string, 'base64url').toString()) : null;
  // cursor = { id: "20", createdAt: "2026-01-01" }
  const where = cursor ? `WHERE (created_at, id) > ($1, $2)` : '';
  const params = cursor ? [cursor.createdAt, cursor.id] : [];
  const rows = await db.query(`SELECT * FROM products ${where} ORDER BY created_at ASC, id ASC LIMIT ${limit + 1}`, params);
  const hasMore = rows.length > limit;
  const data = hasMore ? rows.slice(0, limit) : rows;
  const nextCursor = hasMore ? Buffer.from(JSON.stringify({ id: data[data.length-1].id, createdAt: data[data.length-1].created_at })).toString('base64url') : null;
  res.json({ data, nextCursor });
});

// Frontend - Infinite Scroll
import { useInfiniteQuery } from '@tanstack/react-query';

function ProductFeed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ['products'],
    queryFn: ({ pageParam }) => fetch(`/api/products?cursor=${pageParam ?? ''}&limit=20`).then(r=>r.json()),
    getNextPageParam: last => last.nextCursor,
    initialPageParam: null as string | null,
  });

  const items = data?.pages.flatMap(p => p.data) ?? [];
  const sentinelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const el = sentinelRef.current;
    if (!el) return;
    const io = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) fetchNextPage();
    }, { rootMargin: '200px' }); // preload trước 200px
    io.observe(el);
    return () => io.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <>
      {items.map(item => <ProductCard key={item.id} item={item} />)}
      <div ref={sentinelRef} style={{ height: 1 }} />
      {isFetchingNextPage && <Spinner />}
      {!hasNextPage && <div>Đã hết</div>}
    </>
  );
}
```

**Trade-off:** Infinite scroll UX mượt nhưng **SEO kém** (bot không scroll), và **không bookmark** trang 5. Với SEO cần pagination truyền thống hoặc SSR + `?cursor=`. Cursor với `createdAt` trùng thì phải thêm `id` để unique.

**Câu hỏi đào sâu:** Vì sao cursor dùng `(createdAt, id)` thay vì chỉ `id`? Làm sao handle khi user scroll nhanh trigger nhiều `fetchNextPage`?

---

### Câu 148: Optimistic Update và Rollback

**Trả lời Senior:**
Optimistic update là **cập nhật UI ngay** trước khi server xác nhận, cho cảm giác tức thì (0ms), nếu fail thì **rollback**. Dùng cho like, toggle, cart, edit - nơi success 95% và user mong phản hồi ngay.

Flow: **snapshot cũ → update cache lạc quan → gọi API → nếu fail thì rollback + toast**, nếu success thì **revalidate** (hoặc giữ).

Không dùng cho thanh toán, xóa vĩnh viễn - nơi fail cost cao.

```typescript
// Với TanStack Query
import { useMutation, useQueryClient } from '@tanstack/react-query';

function useToggleLike(productId: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (liked: boolean) => fetch(`/api/products/${productId}/like`, { method: liked ? 'POST' : 'DELETE' }).then(r=>{ if(!r.ok) throw new Error(); }),
    onMutate: async (liked) => {
      await qc.cancelQueries({ queryKey: ['product', productId] });
      const prev = qc.getQueryData(['product', productId]) as any;
      qc.setQueryData(['product', productId], (old: any) => ({ ...old, liked, likes: old.likes + (liked ? 1 : -1) }));
      return { prev }; // context cho rollback
    },
    onError: (err, liked, ctx) => {
      qc.setQueryData(['product', productId], ctx?.prev); // rollback
      showToast('Thích thất bại, đã hoàn tác');
    },
    onSettled: () => {
      qc.invalidateQueries({ queryKey: ['product', productId] }); // đồng bộ server
    },
  });
}

// Component
function LikeButton({ product }: { product: Product }) {
  const { mutate, isPending } = useToggleLike(product.id);
  return <button disabled={isPending} onClick={() => mutate(!product.liked)}>{product.liked ? '♥' : '♡'} {product.likes}</button>;
}

// Manual không lib
async function optimisticUpdate() {
  const prev = products;
  setProducts(products.map(p => p.id === id ? { ...p, liked: true } : p)); // optimistic
  try {
    await fetch(`/api/products/${id}/like`, { method: 'POST' });
  } catch {
    setProducts(prev); // rollback
    showToast('Lỗi');
  }
}
```

**Trade-off:** Optimistic mượt nhưng phức tạp: phải snapshot, handle race (2 like nhanh), và rollback. Nếu API chậm 2s mà không optimistic, user bấm 3 lần → duplicate request. Nên **disable button khi pending** + optimistic.

**Câu hỏi đào sâu:** Khi nào không nên optimistic (ví dụ chuyển tiền)? Làm sao handle khi 2 optimistic update cùng resource race nhau?

---

### Câu 149: ETag, Conditional Request và Cache API

**Trả lời Senior:**
ETag là **hash của response** (`W/"abc123"` hoặc `"xyz"`), server gửi kèm `ETag`, client gửi lại `If-None-Match: "abc123"` lần sau. Nếu không đổi → server trả `304 Not Modified` (không body), tiết kiệm bandwidth 90% cho data ít đổi (product list, config).

Kết hợp với `Cache-Control` và `Last-Modified`/`If-Modified-Since`.

Dùng cho: **polling**, **revalidate**, **optimistic locking** (`If-Match`).

```http
# Lần 1
GET /api/products
-> 200 OK
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Cache-Control: public, max-age=60
{ data: [...] }

# Lần 2 (sau 60s, hoặc force revalidate)
GET /api/products
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
-> 304 Not Modified (không body, dùng cache)

# Optimistic locking - tránh lost update
GET /api/products/123 -> ETag: "v1"
PATCH /api/products/123
If-Match: "v1"
{ price: 120 } -> 200, ETag: "v2"
# Nếu người khác đã sửa lên v2, PATCH với v1 -> 412 Precondition Failed
```

```typescript
// Frontend với fetch + ETag manual
let etag: string | null = null;
let cached: any = null;
async function getProducts() {
  const headers: Record<string, string> = {};
  if (etag) headers['If-None-Match'] = etag;
  const res = await fetch('/api/products', { headers });
  if (res.status === 304) return cached; // dùng cache
  etag = res.headers.get('ETag');
  cached = await res.json();
  return cached;
}

// Với TanStack Query + ETag (tự handle nếu server chuẩn)
// Hoặc dùng Cache API (Service Worker)
const cache = await caches.open('api');
const cachedRes = await cache.match('/api/products');
if (cachedRes) {
  const res = await fetch('/api/products', { headers: { 'If-None-Match': cachedRes.headers.get('ETag')! } });
  if (res.status === 304) return cachedRes.json();
  await cache.put('/api/products', res.clone());
  return res.json();
}

// Express ETag
app.get('/api/products', async (req, res) => {
  const data = await getProducts();
  const hash = crypto.createHash('md5').update(JSON.stringify(data)).digest('hex');
  res.set('ETag', `"${hash}"`);
  if (req.headers['if-none-match'] === `"${hash}"`) return res.status(304).end();
  res.json(data);
});
```

**Trade-off:** ETag tốn CPU hash, nhưng tiết kiệm bandwidth. `304` vẫn tốn RTT, nên kết hợp `max-age` để không gửi luôn. Với CDN, `ETag` giúp revalidate rẻ.

**Câu hỏi đào sâu:** `ETag` mạnh (`"abc"`) vs yếu (`W/"abc"`) khác gì? `If-Match` vs `If-None-Match` dùng khi nào?

---

### Câu 150: API Contract - versioning, idempotency-key và rate limit

**Trả lời Senior:**
API contract là **hợp đồng** giữa frontend và backend để không break khi evolve. Senior thiết kế 4 thứ: **versioning**, **idempotency**, **rate limit**, **error format thống nhất**.

- **Versioning**: `/api/v1/products` (URL) hoặc `Accept: application/vnd.shop.v1+json` (header). URL đơn giản nhất, header sạch hơn nhưng khó debug. Khi break (đổi field, xóa endpoint) thì tăng `v2`, giữ `v1` 6 tháng.
- **Idempotency-Key**: cho `POST` an toàn retry. Client sinh `Idempotency-Key: uuid`, server lưu key + response 24h, nếu trùng thì trả lại response cũ, không tạo mới. Bắt buộc cho thanh toán, tạo đơn.
- **Rate Limit**: `429 Too Many Requests` + `Retry-After` + headers `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. Frontend phải backoff và hiện "Thử lại sau".
- **Error Format**: `{ error: { code: "VALIDATION_ERROR", message: "...", fields: { email: "..." } } }` thống nhất, không lúc `error` lúc `msg`.

```http
# Versioning
GET /api/v1/products
Accept: application/vnd.shop.v1+json

# Idempotency
POST /api/orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
{ items: [...] } -> 201 { id: "ord_123" }
# Retry cùng key -> 201 { id: "ord_123" } (không tạo mới)

# Rate Limit
GET /api/products
-> 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1714000000
{ error: { code: "RATE_LIMITED", message: "Thử lại sau 60s" } }

# Error thống nhất
400 { error: { code: "BAD_REQUEST", message: "JSON invalid" } }
422 { error: { code: "VALIDATION_ERROR", fields: { email: "Email sai" } } }
```

```typescript
// Frontend wrapper
async function apiPost<T>(url: string, body: any, opts: { idempotent?: boolean } = {}) {
  const headers: Record<string, string> = { 'Content-Type': ' application/json' };
  if (opts.idempotent) headers['Idempotency-Key'] = crypto.randomUUID();
  const res = await fetch(url, { method: 'POST', headers, body: JSON.stringify(body) });
  if (res.status === 429) {
    const retryAfter = Number(res.headers.get('Retry-After') ?? '60');
    throw new RateLimitError(retryAfter);
  }
  if (!res.ok) {
    const { error } = await res.json();
    throw new ApiError(error.code, error.message, error.fields);
  }
  return res.json() as Promise<T>;
}

// Axios interceptor cho rate limit
axios.interceptors.response.use(null, async error => {
  if (error.response?.status === 429) {
    const delay = Number(error.response.headers['retry-after']) * 1000;
    await new Promise(r => setTimeout(r, delay));
    return axios.request(error.config); // retry 1 lần
  }
  return Promise.reject(error);
});
```

**Trade-off:** Versioning URL dễ nhưng làm route phình, header sạch nhưng tooling kém. Idempotency-key thêm storage (Redis) nhưng cứu duplicate payment. Rate limit làm UX tệ nếu quá chặt, nhưng bảo vệ server.

**Câu hỏi đào sâu:** Vì sao `Idempotency-Key` chỉ cho `POST` mà không cần cho `PUT`? Làm sao versioning không break khi chỉ thêm field optional?

---
