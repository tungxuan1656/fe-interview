# REST API Design — Resource + Verb, Idempotent PUT/PATCH/POST, Idempotency-Key, Offset vs Cursor Pagination

> Tags: #REST #resource #HTTP-verb #idempotent #PUT #PATCH #POST #Idempotency-Key #pagination #offset #cursor | Nguồn: `docs/08-api-networking.md` câu 135-136, 146-147, 150 | Mức: P0

## 1. Định nghĩa chính xác

**REST (Representational State Transfer, Fielding)** là kiến trúc cho API dùng HTTP làm application protocol: mọi thứ là **resource** (danh từ số nhiều), thao tác qua **HTTP verb** (động từ), **stateless** (mỗi request đủ info, server không lưu session client), **uniform interface** (URI định danh, method/status/cache), **layered system** (qua CDN/gateway không ảnh hưởng).

**Idempotent** nghĩa là gọi cùng request N lần thì **server state như 1 lần** (không tạo duplicate, không cộng dồn). `GET`, `PUT`, `DELETE` là idempotent; `POST` thường **không** idempotent (tạo mới mỗi lần); `PATCH` thường **không** idempotent (tùy merge semantics).

**Idempotency-Key** (`Idempotency-Key: uuid`) là header cho `POST` an toàn retry: client sinh UUID per-operation, server lưu `key → response` 24h, nếu trùng key thì trả response cũ, không tạo mới — cứu duplicate payment/order khi response mất nhưng server đã xử lý.

**Pagination**: **Offset** (`?offset=20&limit=20` / `?page=2`) — scan, cho phép nhảy trang; **Cursor** (keyset, `?cursor=eyJpZCI6IjIwIn0=&limit=20`) — `WHERE id > cursor ORDER BY id LIMIT 20` trên index, stable khi insert/delete.

## 2. Cơ chế hoạt động

### 2.1 6 constraints REST (Fielding) — thực chiến

1. **Client-Server**: tách UI và data.
2. **Stateless**: `Authorization: Bearer <token>` mỗi request, không session server.
3. **Cacheable**: `Cache-Control`, `ETag` đánh dấu cache được/không.
4. **Uniform Interface**: resource định danh URI, method đúng, self-descriptive (`Content-Type`), HATEOAS (optional, link trong response).
5. **Layered System**: qua proxy/CDN/gateway không đổi semantics.
6. **Code on Demand** (optional): server trả code cho client chạy.

Thực chiến:
- Danh từ số nhiều: `/products`, `/users/123/orders` (không `/getProducts`).
- Method đúng: `GET` list/detail, `POST` create, `PUT` replace toàn bộ, `PATCH` partial, `DELETE` xóa.
- Status đúng: `200`, `201 + Location`, `204`, `400/422`, `404`.
- Filter/sort/paginate qua query: `GET /products?category=shoes&sort=price&order=asc&page=2&limit=20`.
- Version: `/api/v1/products` hoặc `Accept: application/vnd.shop.v1+json`.
- HATEOAS (nâng cao): `{ data: {id:"123"}, links: { self: "/products/123" } }`.

### 2.2 PUT vs PATCH vs POST

| Method | Idempotent? | Payload | Client quyết id? | Gọi 2 lần |
|--------|-------------|---------|------------------|-----------|
| **POST** `/products` `{name, price}` | **Không** | Tạo mới | Không (server sinh id) | Tạo 2 products (`id: abc` và `def`) |
| **PUT** `/products/abc` `{name, price, desc}` | **Có** | **Thay toàn bộ** — thiếu field thì xóa/null | Có (client biết id/URL) | 1 product, state như 1 lần |
| **PATCH** `/products/abc` `{price:130}` | Thường **không** (tùy) | **Sửa 1 phần** — chỉ field đổi | Có | Chỉ price đổi, giữ name/desc; nhưng nếu PATCH là `increment` thì không idempotent |

- **PUT** idempotent nên **retry an toàn** (network fail → retry không sợ duplicate), nhưng tốn bandwidth (gửi full object).
- **PATCH** tiết kiệm bandwidth, nhưng phải xử lý `null` vs `undefined` (xóa hay giữ?), và có 2 chuẩn: `application/merge-patch+json` (RFC 7396, merge) vs `application/json-patch+json` (RFC 6902, op/path/value).
- **`If-Match: "etag"`** cho PUT/PATCH để tránh **lost update** (optimistic locking): `412 Precondition Failed` nếu đã bị sửa bởi người khác.

### 2.3 Idempotency-Key cho POST

```
POST /orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
{ items: [...] } → 201 { id: "ord_123" }

Retry cùng key → 201 { id: "ord_123" } (không tạo ord_124)
Server lưu Redis: key → { status, body, headers } TTL 24h
```
- Chỉ cho `POST` (vì `PUT` đã idempotent, không cần).
- Key là **per-operation**, không per-request — retry cùng operation dùng lại key; operation mới sinh key mới.

### 2.4 Offset vs Cursor pagination

```
Offset: GET /products?offset=20&limit=20 → SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 20 -- scan 20
Cursor: GET /products?cursor=eyJpZCI6IjIwIn0=&limit=20 → SELECT * FROM products WHERE id > '20' ORDER BY id LIMIT 20 -- index seek
        Response: { data:[...20], nextCursor: "eyJpZCI6IjQwIn0=" }
```

- **Cursor ổn định và unique**: dùng `(createdAt, id)` hoặc `id` tăng dần, encode base64url JSON, client không cần hiểu. `WHERE (created_at, id) > ($1,$2)` khi `createdAt` trùng.
- **Infinite scroll** = cursor + `IntersectionObserver` + `useInfiniteQuery`.

## 3. Ví dụ tối thiểu

```http
# 3.1 RESTful đúng
GET    /api/v1/products?category=shoes&limit=20&cursor=abc
POST   /api/v1/products          { "name": "Shoes", "price": 100 } -> 201 Location: /products/123
GET    /api/v1/products/123      -> 200 { "id":"123", "name":"Shoes" }
PATCH  /api/v1/products/123      { "price": 120 } -> 200
PUT    /api/v1/products/123      { "name":"Shoes","price":120,"desc":"Nice" } -> 200
DELETE /api/v1/products/123      -> 204

# ❌ Không RESTful
GET /api/getProducts
POST /api/createProduct
GET /api/products?action=delete&id=123
POST /api/products/123/updatePrice

# 3.2 PUT vs PATCH
POST /products { "name":"Shoes","price":100 } -> 201 { "id":"abc" }
# Gọi lại -> 201 { "id":"def" } (duplicate)

PUT /products/abc { "name":"Shoes","price":120,"desc":"Nice" } -> 200
# Gọi lại y hệt -> 200, vẫn 1 sản phẩm

PATCH /products/abc { "price":130 } -> 200 { "name":"Shoes","price":130,"desc":"Nice" }

# JSON Patch (RFC 6902)
PATCH /products/abc
Content-Type: application/json-patch+json
[
  { "op": "replace", "path": "/price", "value": 130 },
  { "op": "add", "path": "/tags/0", "value": "sale" }
]

# Optimistic locking
PATCH /products/123
If-Match: "v1"
{ "price":130 } -> 200 ETag: "v2"
# Nếu đã lên v2, PATCH với v1 -> 412 Precondition Failed

# 3.3 Offset vs Cursor
# Offset — nhảy trang được, nhưng chậm khi offset lớn, lệch khi insert
GET /api/products?offset=0&limit=20  -> { data:[...20], total:100 }
GET /api/products?offset=20&limit=20 -> { data:[...20] }

# Cursor — nhanh (index), consistent, không nhảy trang
GET /api/products?limit=20               -> { data:[...20], nextCursor:"eyJpZCI6IjIwIn0=" }
GET /api/products?cursor=eyJpZCI6IjIwIn0=&limit=20 -> { data:[...20], nextCursor:"eyJpZCI6IjQwIn0=" }

# 3.4 Idempotency-Key
POST /api/orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
{ "items": [...] } -> 201 { "id":"ord_123" }
# Retry cùng key (network fail) -> 201 { "id":"ord_123" } (không tạo mới)
```

```ts
// 3.5 Frontend fetch
// Tạo
await fetch('/api/v1/products', { method: 'POST', headers: { 'Content-Type':'application/json' }, body: JSON.stringify({ name:'Shoes', price:100 }) });
// Sửa toàn bộ (ít dùng)
await fetch('/api/v1/products/123', { method: 'PUT', headers: { 'Content-Type':'application/json' }, body: JSON.stringify(product) });
// Sửa 1 phần (hay dùng) + If-Match
await fetch('/api/v1/products/123', {
  method: 'PATCH',
  headers: { 'Content-Type':'application/json', 'If-Match': etag },
  body: JSON.stringify({ price:130 }),
});

// POST idempotent cho payment/order
await fetch('/api/orders', {
  method: 'POST',
  headers: { 'Content-Type':'application/json', 'Idempotency-Key': crypto.randomUUID() },
  body: JSON.stringify(order),
});

// 3.6 Pagination — offset
function useOffset(page: number) {
  const { data } = useQuery({ queryKey:['products', page], queryFn:()=>fetch(`/api/products?offset=${page*20}&limit=20`).then(r=>r.json()) });
  return data; // { data, total }
}
// Cursor — infinite
function useCursor() {
  const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
    queryKey:['products'],
    queryFn:({ pageParam })=>fetch(`/api/products?cursor=${pageParam ?? ''}&limit=20`).then(r=>r.json()),
    getNextPageParam: last=>last.nextCursor,
    initialPageParam: null as string | null,
  });
  return { items: data?.pages.flatMap(p=>p.data) ?? [], fetchNextPage, hasNextPage };
}

// 3.7 Cursor server (Postgres)
app.get('/api/products', async (req,res)=>{
  const limit = Math.min(Number(req.query.limit)||20, 50);
  const cursor = req.query.cursor ? JSON.parse(Buffer.from(req.query.cursor as string,'base64url').toString()) : null;
  const where = cursor ? `WHERE (created_at, id) > ($1,$2)` : '';
  const params = cursor ? [cursor.createdAt, cursor.id] : [];
  const rows = await db.query(`SELECT * FROM products ${where} ORDER BY created_at ASC, id ASC LIMIT ${limit+1}`, params);
  const hasMore = rows.length > limit;
  const data = hasMore ? rows.slice(0,limit) : rows;
  const nextCursor = hasMore ? Buffer.from(JSON.stringify({ id:data[data.length-1].id, createdAt:data[data.length-1].created_at })).toString('base64url') : null;
  res.json({ data, nextCursor });
});

// 3.8 HATEOAS + versioning
// GET /api/v1/products/123 -> { data:{id:"123"}, links:{ self:"/v1/products/123", orders:"/v1/users/123/orders" } }
// Version qua header: Accept: application/vnd.shop.v1+json
```

## 4. So sánh / Phân loại

| Tiêu chí | POST | PUT | PATCH |
|----------|------|-----|-------|
| Idempotent | Không | **Có** | Thường không |
| Semantics | Tạo mới (server sinh id) | **Thay toàn bộ** (gửi thiếu → xóa) | **Sửa 1 phần** (chỉ field đổi) |
| Retry an toàn? | Không, trừ khi có `Idempotency-Key` | Có | Không (cần logic) |
| Content-Type | `application/json` | `application/json` | `application/json` hoặc `merge-patch+json` / `json-patch+json` |
| Dùng khi | Create, action không fit REST (`POST /orders/123/cancel`) | Replace toàn bộ, ít dùng frontend | Update partial, tiết kiệm bandwidth |

|  | `application/merge-patch+json` (RFC 7396) | `application/json-patch+json` (RFC 6902) |
|---|------------------------------------------|------------------------------------------|
| Format | `{ "price":130, "tags":null }` (null = xóa) | `[ {op:"replace",path:"/price",value:130} ]` |
| Dễ dùng | Gọn | Chi tiết, hỗ trợ `add/remove/move` |

| Tiêu chí | Offset (`offset/limit` / `page`) | Cursor (`cursor/limit`) |
|----------|----------------------------------|-------------------------|
| Query | `?offset=20&limit=20` | `?cursor=eyJpZCI6IjIwIn0=&limit=20` |
| Nhảy trang | Có (`page=5`) | Không (chỉ next/prev) |
| Performance | Chậm khi offset lớn (`OFFSET 100000` scan) | Nhanh (index seek `WHERE id > cursor`) |
| Consistency | Lệch khi insert/delete (duplicate/missing) | Ổn định |
| Sort | Tùy | Cần sort ổn định `(createdAt, id)` unique |
| Dùng khi | Admin table cần nhảy trang, ít data | Feed, infinite scroll, nhiều data |

| Error format thống nhất |
|-------------------------|
| `{ error: { code:"VALIDATION_ERROR", message:"...", fields:{ email:"..." } } }` — không lúc `error` lúc `msg` |

| Idempotency-Key |
|-----------------|
| Chỉ cho `POST` (PUT đã idempotent), client sinh UUID per-operation, server Redis 24h, retry cùng key trả response cũ |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng `PUT` cho partial update**: `PUT` thiếu field → server hiểu là xóa field đó. Nếu chỉ đổi `price`, dùng `PATCH`, không `PUT` với object thiếu field.
- **Không dùng `PATCH` với semantics `increment` nếu cần idempotent**: `PATCH { likes: increment(1) }` không idempotent (gọi 2 lần +2). Nếu cần retry, thiết kế `PUT { likes: 10 }` (set) hoặc `PATCH` với `If-Match` + `Idempotency-Key`.
- **Không retry `POST` mặc định**: `POST` tạo duplicate nếu server đã xử lý nhưng response mất. Chỉ retry `POST` khi có `Idempotency-Key`; mặc định chỉ retry `GET/PUT/DELETE` (idempotent) và `502/503/504/429/408`.
- **Không dùng offset cho feed lớn**: `OFFSET 100000` scan 100k row, chậm và lệch khi insert. Dùng cursor cho feed/timeline. Chỉ offset cho admin table ít data cần nhảy trang.
- **Không dùng cursor khi cần nhảy tới `page=50`**: cursor không hỗ trợ random access, chỉ next. Nếu UI cần `page` input, dùng offset hoặc hybrid (cursor + page estimate).
- **Không quên `total` cho offset**: offset cần `total` để tính số trang; cursor không cần `total` (chỉ `nextCursor`/`hasMore`) — đừng cố tính `total` cho cursor (expensive count).
- **Không version qua URL nếu team chuộng header**: `/api/v1` đơn giản, dễ debug, nhưng làm route phình khi lên `v2` (duplicate). Header `Accept: application/vnd.shop.v1+json` sạch nhưng tooling/cache kém. Chọn một và thống nhất.
- **Khi nào không REST**: query phức tạp nested/graph (lấy `user` + `orders` + `products` + `reviews` trong 1 request) → **GraphQL** linh hoạt hơn REST (`GET /products?include=orders.reviews` cồng kềnh).

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `PUT` thiếu field → mất data**
  - Triệu chứng: `PUT /products/123 { price:120 }` → `name/desc` thành `null`.
  - Fix: dùng `PATCH` cho partial, `PUT` phải gửi full object. Document rõ.
  - Đo: Network → Request Payload, Response diff.

- **Lỗi 2: `POST` retry tạo duplicate order/payment**
  - Triệu chứng: network timeout, client retry → 2 orders.
  - Fix: `Idempotency-Key: uuid` per-operation, server store 24h.
  - Đo: Network → duplicate `POST` với cùng `Idempotency-Key` → server trả cùng `id`.

- **Lỗi 3: Offset `OFFSET 100000` chậm**
  - Triệu chứng: `page=5000` TTFB 2s.
  - Fix: chuyển cursor `WHERE id > cursor` với index.
  - Đo: `EXPLAIN ANALYZE SELECT ... OFFSET 100000` vs `WHERE id > 'xxx'`.

- **Lỗi 4: Cursor `createdAt` trùng → miss/duplicate**
  - Triệu chứng: 2 products cùng `createdAt`, cursor chỉ `createdAt` → miss.
  - Fix: cursor `(createdAt, id)` composite, `ORDER BY created_at ASC, id ASC`, `WHERE (created_at, id) > ($1,$2)`.
  - Đo: test insert cùng timestamp, check `nextCursor` chain.

- **Lỗi 5: Mất `Location` sau `201`**
  - Triệu chứng: client không biết URL mới để `GET`.
  - Fix: `201 Created` kèm `Location: /products/123`.
  - Đo: Network → Response Headers `Location`.

- **Lỗi 6: Lost update không `If-Match`**
  - Triệu chứng: user A và B cùng `PATCH` price, B ghi đè A không biết.
  - Fix: `ETag` + `If-Match: "v1"` → `412 Precondition Failed` nếu đã đổi, client refresh rồi retry.
  - Đo: `ETag` header, `If-Match` request, `412` response.

- **Lỗi 7: Pagination infinite scroll không dedup**
  - Triệu chứng: scroll nhanh trigger 3 `fetchNextPage`, duplicate items.
  - Fix: `useInfiniteQuery` tự dedup + `hasNextPage` guard + `rootMargin: '200px'` cho `IntersectionObserver`, hoặc `fetchNextPage` debounce.
  - Đo: Network → 3 `cursor` requests cùng lúc, check `nextCursor` duplicate.

- **Công cụ**:
  - Network → Method, Status (`201`, `412`), Headers (`Idempotency-Key`, `If-Match`, `Location`, `ETag`), Payload.
  - `curl -X POST -H "Idempotency-Key: $(uuidgen)" -d '{...}' -i https://api/orders`
  - DB `EXPLAIN ANALYZE` cho offset vs cursor.
  - TanStack Query Devtools cho `useInfiniteQuery`.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt `POST` vs `PUT` vs `PATCH` về idempotent, payload (toàn bộ vs 1 phần) và khi nào retry an toàn? Vì sao `PUT` idempotent còn `PATCH` thường không?
2. `Idempotency-Key` hoạt động thế nào để `POST /orders` retry không tạo duplicate? Vì sao không cần cho `PUT`?
3. Offset (`offset/limit`) vs Cursor (`cursor/limit`) khác nhau về performance (`OFFSET 100000` scan), consistency khi insert/delete, và khả năng nhảy trang — khi nào dùng mỗi loại và vì sao cursor cần `(createdAt, id)`?

<details>
<summary>Đáp án 30s</summary>

1. **POST** tạo mới, server sinh id, **không idempotent** (2 lần tạo 2 resource) → không retry mặc định. **PUT** thay **toàn bộ**, client quyết id, **idempotent** (N lần như 1) → retry an toàn, nhưng gửi thiếu field thì field đó bị xóa. **PATCH** sửa **1 phần**, thường **không idempotent** (ví dụ `increment`, `null` vs `undefined` semantics, `merge-patch` vs `json-patch`) → retry không an toàn nếu không thêm `If-Match`/`Idempotency-Key`. `PUT` idempotent vì semantics là **replace** với cùng payload → state cuối như nhau; `PATCH` là **partial merge** có thể cộng dồn hoặc phụ thuộc state hiện tại.
2. Client sinh `Idempotency-Key: uuid` per-operation, gửi kèm `POST`, server lưu `key → { status, body }` trong Redis 24h. Retry cùng key → server trả response cũ, không tạo mới. Hết TTL thì key hết hiệu lực. `PUT` đã idempotent theo HTTP spec (replace), nên retry cùng payload tự nhiên cho cùng state, không cần key; key chỉ cứu `POST` không idempotent.
3. **Offset**: `LIMIT 20 OFFSET 100000` DB phải scan 100k row → chậm O(offset), và lệch khi có insert/delete giữa chừng (duplicate/missing). Cho phép `page=5` nhảy trang. Dùng cho admin table ít data. **Cursor**: `WHERE id > cursor ORDER BY id LIMIT 20` index seek O(limit), consistent (không lệch), nhưng chỉ next/prev, không nhảy. Dùng cho feed/infinite scroll nhiều data. Cursor cần `(createdAt, id)` vì `createdAt` không unique — 2 row cùng timestamp thì chỉ cursor `createdAt` sẽ miss; thêm `id` làm tie-breaker unique, `WHERE (created_at, id) > (t1, id1)`.

</details>

---
*Tham khảo chi tiết: `docs/08-api-networking.md` — Câu 135, 136, 146, 147, 150. Spec: [RFC 7231 — Method Definitions](https://httpwg.org/specs/rfc7231.html#method.definitions), [RFC 6902 JSON Patch](https://datatracker.ietf.org/doc/html/rfc6902), [RFC 7396 Merge Patch](https://datatracker.ietf.org/doc/html/rfc7396).*
