# HTTP Cache — Strong Cache (max-age/immutable) vs Conditional (ETag/Last-Modified 304), no-cache, CDN vs Browser vs SW

> Tags: #http-cache #cache-control #etag #last-modified #304 #cdn #service-worker #stale-while-revalidate | Nguồn: `docs/05-performance.md` câu 98-99 | Mức: P0

## 1. Định nghĩa chính xác

**HTTP Cache** là cơ chế browser/CDN lưu response theo **Cache-Control** và **validators** để tái sử dụng mà không tải lại. Hai loại: **Strong Cache** (fresh) — browser dùng cache ngay, **không gửi request**; **Conditional Cache** (revalidation) — gửi request kèm validator, server trả **304 Not Modified** (không body) nếu không đổi, tiết kiệm bandwidth nhưng vẫn tốn RTT.

**Cache-Control** directives quyết định freshness: `max-age`, `s-maxage`, `immutable`, `no-cache`, `no-store`, `must-revalidate`, `stale-while-revalidate`, `public`/`private`. **ETag** (hash content, `W/` weak hoặc strong) và **Last-Modified** (timestamp 1s) là validators cho conditional request (`If-None-Match`, `If-Modified-Since`). **CDN Cache** (Edge) share giữa users theo `s-maxage`/`CDN-Cache-Control`/`Surrogate-Key`; **Browser Cache** per-user theo `max-age`; **Service Worker Cache** (Cache API) programmable per-origin, control bằng JS.

## 2. Cơ chế hoạt động

### 2.1 Strong vs Conditional

```
Request 1: GET /app.abc123.js → 200 + Cache-Control: max-age=31536000, immutable + ETag: "abc"
           Browser lưu disk cache, đánh dấu fresh tới 1 năm

Request 2 (trong max-age): GET /app.abc123.js → (from disk cache) — không gửi request

Request với no-cache hoặc hết max-age:
  GET /app.abc123.js + If-None-Match: "abc" → Server so sánh ETag
    Nếu khớp: 304 Not Modified (không body) — browser dùng cache, update Age
    Nếu khác: 200 + body mới
```

- **Strong**: `Cache-Control: public, max-age=31536000, immutable` — fresh 1 năm, không request. `immutable` hint rằng content **không bao giờ** đổi với URL này (cần hash tên file).
- **Conditional**: `ETag: "abc123"` (hash), `Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT`. Browser gửi `If-None-Match: "abc123"` / `If-Modified-Since`. Server 304 nếu khớp.
- **Expires**: header cũ (HTTP/1.0), dùng `Cache-Control` thay; nếu cả hai có, `Cache-Control` thắng.

### 2.2 `no-cache` vs `no-store` (hay nhầm)

- `no-cache`: **vẫn lưu** nhưng phải **revalidate** mỗi lần (gửi `If-None-Match`, chờ 304). Dùng cho HTML — muốn cache nhưng luôn kiểm tra mới.
- `no-store`: **không lưu gì** — không disk, không memory. Không có 304, luôn 200. Dùng cho data nhạy cảm (bank, auth).
- `must-revalidate`: khi hết `max-age`, phải revalidate, không được dùng stale. Kết hợp `max-age=0, must-revalidate` = luôn revalidate.
- `stale-while-revalidate=300`: fresh `max-age` giây, sau đó **stale 300s vẫn dùng cache** nhưng **revalidate background** (trả stale ngay, fetch mới async).

### 2.3 Cache-Control layers

```
Browser Cache:  Cache-Control: max-age=31536000        — per user, per browser
CDN/Proxy:      Cache-Control: s-maxage=86400          — per PoP, shared giữa users
CDN riêng:      CDN-Cache-Control: max-age=86400       — chỉ CDN, browser bỏ qua (Cloudflare/Fastly)
                Surrogate-Key / Cache-Tag              — tag để purge theo nhóm
Vary:           Vary: Accept-Encoding, Accept          — cache key thêm header (gzip vs br, AVIF vs JPEG)
```

- `public`: cho phép proxy/CDN cache. `private`: chỉ browser, không CDN (cho response cá nhân hóa).
- `Vary: Accept-Encoding` để CDN không trả gzip cho client cần br.
- `s-maxage` override `max-age` cho shared cache; `max-age` vẫn cho browser.

### 2.4 Ba tầng: Browser vs CDN vs Service Worker

- **Browser HTTP Cache**: per-user, theo header, nhanh nhất (memory → disk), nhưng mỗi user warm riêng, xóa khi clear cache.
- **CDN (Edge) Cache**: per-PoP, shared, giảm TTFB và origin load. Warm khi 1 user request, user khác trong PoP hưởng. Invalidate bằng `purge` / `Surrogate-Key` tag (ví dụ `purge tag:static`).
- **Service Worker Cache (Cache API)**: `caches.open('v1').put(request, response)` — programmable, per-origin, lifecycle riêng, cho offline, precache, custom `stale-while-revalidate`. Tồn tại qua reload nhưng bị xóa khi clear site data. Chỉ HTTPS.

### 2.5 HTML vs Asset caching strategy

- **Asset versioned** (`/static/app.abc123.js` với content hash): `public, max-age=31536000, immutable` — an toàn vì URL đổi khi content đổi.
- **HTML** (`/index.html`, `/`): `public, max-age=0, must-revalidate` hoặc `no-cache` — luôn revalidate, vì HTML chứa reference tới asset hash mới. Nếu HTML `immutable` → user kẹt HTML cũ trỏ tới asset cũ, không thấy deploy mới.
- **API**: `public, max-age=60, stale-while-revalidate=300` hoặc `private` nếu cá nhân hóa.

## 3. Ví dụ tối thiểu

```http
# 3.1 Asset versioned — strong cache 1 năm, không request lần sau
GET /static/app.abc123.js HTTP/1.1

HTTP/1.1 200 OK
Cache-Control: public, max-age=31536000, immutable
ETag: "abc123"
Content-Type: application/javascript

# Lần sau trong 1 năm: (from disk cache) — không gửi request

# 3.2 HTML — no-cache (vẫn lưu nhưng luôn revalidate)
GET /index.html HTTP/1.1

HTTP/1.1 200 OK
Cache-Control: public, max-age=0, must-revalidate
ETag: "html-v42"
Content-Type: text/html

# Lần sau:
GET /index.html HTTP/1.1
If-None-Match: "html-v42"

HTTP/1.1 304 Not Modified
Cache-Control: public, max-age=0, must-revalidate
# -> browser dùng cache HTML, không tải body

# 3.3 API — stale-while-revalidate
GET /api/products HTTP/1.1

HTTP/1.1 200 OK
Cache-Control: public, max-age=60, stale-while-revalidate=300
ETag: "prod-v5"

# Trong 60s: fresh — dùng cache, không request
# 60-360s: stale — trả cache ngay, đồng thời fetch background, update cache
# >360s: phải revalidate

# 3.4 CDN vs Browser
HTTP/1.1 200 OK
Cache-Control: public, max-age=31536000, immutable   # browser 1 năm
CDN-Cache-Control: public, max-age=86400, stale-while-revalidate=604800  # CDN 1 ngày, stale 1 tuần
Surrogate-Key: static app-v42
Vary: Accept-Encoding
```

```ts
// 3.5 Express
import express from 'express';
const app = express();
app.use('/static', express.static('dist', { maxAge: '1y', immutable: true, etag: true }));
app.get('/api/products', (req, res) => {
  res.set('Cache-Control', 'public, max-age=60, stale-while-revalidate=300');
  res.set('Vary', 'Accept-Encoding');
  res.json(products);
});
app.get('/index.html', (req, res) => {
  res.set('Cache-Control', 'public, max-age=0, must-revalidate');
  res.set('ETag', `"html-${version}"`);
  res.sendFile('index.html');
});

// 3.6 Next.js headers
// next.config.js
async headers() {
  return [
    { source: '/static/:path*', headers: [{ key: 'Cache-Control', value: 'public, max-age=31536000, immutable' }] },
    { source: '/:path*', headers: [{ key: 'Cache-Control', value: 'public, max-age=0, must-revalidate' }] },
  ];
}

// 3.7 Fetch cache modes
fetch('/api/products', { cache: 'force-cache' }); // dùng HTTP cache nếu có
fetch('/api/products', { cache: 'no-store' });    // bỏ qua cache
fetch('/api/products', { next: { revalidate: 60 } }); // Next.js ISR — CDN revalidate 60s
fetch('/api/products', { next: { revalidate: 60, tags: ['products'] } }); // tag để revalidateTag('products')
```

```js
// 3.8 Service Worker — Cache API stale-while-revalidate (programmable)
const CACHE = 'v1';
self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(['/offline.html', '/app.js'])));
});
self.addEventListener('fetch', e => {
  const url = new URL(e.request.url);
  if (url.pathname.startsWith('/api/')) {
    // Network-first cho API dynamic
    e.respondWith(
      fetch(e.request)
        .then(res => { const clone = res.clone(); caches.open(CACHE).then(c => c.put(e.request, clone)); return res; })
        .catch(() => caches.match(e.request))
    );
  } else {
    // Cache-first / stale-while-revalidate cho static
    e.respondWith(
      caches.match(e.request).then(cached => {
        const fetched = fetch(e.request).then(res => {
          if (res.ok) caches.open(CACHE).then(c => c.put(e.request, res.clone()));
          return res;
        });
        return cached ?? fetched;
      })
    );
  }
});
```

## 4. So sánh / Phân loại

| Loại | Request lần sau? | Tiết kiệm | Validator | Dùng cho |
|------|------------------|-----------|-----------|----------|
| **Strong** (`max-age`, `immutable`) | Không gửi | RTT + bandwidth | Không | Asset versioned `app.[hash].js` |
| **Conditional** (`ETag`/`Last-Modified` → 304) | Có, gửi `If-None-Match` | Chỉ bandwidth (không body) | `ETag` (hash) / `Last-Modified` (1s) | HTML, API, asset không hash |

| Directive | Lưu? | Fresh bao lâu? | Hành vi khi hết fresh |
|-----------|------|----------------|-----------------------|
| `max-age=31536000, immutable` | Có | 1 năm | Không request (immutable = không revalidate) |
| `max-age=0, must-revalidate` | Có | 0s | Luôn revalidate → 304 hoặc 200 |
| `no-cache` | Có | 0 (luôn revalidate) | Giống `max-age=0, must-revalidate` |
| `no-store` | Không | — | Luôn 200, không 304 |
| `max-age=60, stale-while-revalidate=300` | Có | 60s fresh, 300s stale | Trong stale: trả cache ngay + fetch background |

| `ETag` | `Last-Modified` |
|--------|-----------------|
| Hash content (strong `"<hash>"` hoặc weak `W/"<hash>"`) | Timestamp `Wed, 21 Oct 2025 07:28:00 GMT` |
| Chính xác byte-level | Độ chính xác 1s, sai nếu đổi trong 1s |
| Tốn CPU hash (nhưng chính xác) | Rẻ, header sẵn |
| Ưu tiên khi cả hai có (`If-None-Match` thắng) | Fallback |

| Tầng | Scope | Control | Invalidate | Offline | Shared? |
|------|-------|---------|------------|---------|---------|
| **Browser HTTP Cache** | per user, per browser | Header `Cache-Control` | Theo `max-age` / 304 | Không | Không |
| **CDN (Edge) Cache** | per PoP, shared | `s-maxage`/`CDN-Cache-Control` + `Surrogate-Key` purge | Purge/tag, TTL | Không | Có — 1 user warm, cả PoP hưởng |
| **Service Worker Cache** | per origin, JS | `caches.open().put()` | Code `caches.delete()` | Có | Không (per origin, nhưng programmable) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không `immutable` cho HTML**: asset hash mới nhưng HTML vẫn trỏ hash cũ nếu HTML bị cache immutable → user kẹt giao diện cũ tới khi hard reload. HTML luôn `no-cache`/`max-age=0, must-revalidate`.
- **Không `no-store` cho HTML nếu muốn 304**: `no-store` không lưu gì → mỗi lần 200 full body. `no-cache` vẫn lưu và cho 304, tiết kiệm bandwidth. Chỉ `no-store` cho data nhạy cảm (auth, bank).
- **Không `ETag` hash cho file lớn nếu CPU đắt**: `ETag` strong tính hash toàn file mỗi request. Với file 10MB, `Last-Modified` rẻ hơn. Hoặc dùng weak `W/"<size>-<mtime>"`.
- **Không cache HTML cá nhân hóa ở CDN**: `Cache-Control: public, s-maxage` cho `/dashboard` của user A sẽ trả cho user B → leak data. Dùng `private` hoặc `Vary: Cookie` (nhưng hit rate kém).
- **Không quên `Vary: Accept-Encoding`**: CDN cache gzip cho request đầu, trả gzip cho client cần br → lỗi decode. Luôn `Vary` theo header ảnh hưởng response.
- **Không dùng `Expires` thay `Cache-Control`**: `Expires` phụ thuộc clock client, dễ lệch. `Cache-Control: max-age` tính relative, đáng tin.
- **Không SW cache cho mọi thứ**: SW thêm complexity (update lifecycle, `skipWaiting`, `clients.claim`), chỉ HTTPS, và phải handle quota. Dùng cho offline/precache, không thay HTTP cache cho asset đơn giản.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: HTML `immutable` → deploy mới user không thấy**
  - Triệu chứng: deploy `app.def456.js` mới nhưng user vẫn thấy `app.abc123.js` cũ, hard reload mới hết.
  - Nguyên nhân: `Cache-Control: immutable` cho `/index.html` → browser không revalidate.
  - Fix: HTML → `no-cache` / `max-age=0, must-revalidate`; asset hash → `immutable`.
  - Đo: DevTools Network → `Size` `(disk cache)` cho HTML dù đã deploy; Response Headers → `Cache-Control`.

- **Lỗi 2: `no-store` làm mất 304, tốn bandwidth**
  - Triệu chứng: mỗi reload `index.html` 200 (50kb) thay vì 304 (0kb).
  - Fix: đổi `no-store` → `no-cache` nếu không phải data nhạy cảm.
  - Đo: Network → `Status 200 (from disk cache)` vs `304`, `Transferred` size.

- **Lỗi 3: Thiếu `Vary` → CDN trả sai encoding**
  - Triệu chứng: `Content-Encoding: gzip` cho client `Accept-Encoding: br` → lỗi.
  - Fix: `Vary: Accept-Encoding`.
  - Đo: `curl -H "Accept-Encoding: br" -I https://cdn/app.js` → check `Vary`.

- **Lỗi 4: CDN cache HTML cá nhân hóa → leak**
  - Triệu chứng: user B thấy dashboard của user A.
  - Fix: `Cache-Control: private, no-store` cho API private; `CDN-Cache-Control: no-cache` hoặc `Vary: Cookie`.
  - Đo: 2 browser khác session → so sánh response; CDN header `X-Cache: HIT`.

- **Lỗi 5: `ETag` weak vs strong nhầm**
  - Triệu chứng: `304` không khớp dù content giống (do `W/` weak).
  - Fix: hiểu `W/"abc"` cho phép khác byte-level nhỏ (như gzip); strong ` "abc"` khớp exact.
  - Đo: Request Headers `If-None-Match`, Response `ETag`.

- **Lỗi 6: SW cache cũ không purge**
  - Triệu chứng: SW trả `v1` cache dù deploy `v2`, `skipWaiting` không gọi.
  - Fix: `activate` event xóa cache cũ `caches.keys().then(ks => ks.filter(k=>k!==CACHE).map(caches.delete))`, `self.skipWaiting()` + `clients.claim()`.
  - Đo: DevTools Application → Cache Storage, Service Workers → `Update`.

- **Công cụ**:
  - **Network tab**: `Size` (`(disk cache)` / `(memory cache)` / `304`), `Cache-Control`, `ETag`, `Age`.
  - **Disable cache**: Network → `Disable cache` để test 200 vs 304.
  - **`curl -I`**: `curl -I -H "If-None-Match: \"abc\"" https://example.com/app.js`.
  - **Application → Cache Storage**: xem SW cache.
  - **CDN logs**: `X-Cache: HIT/MISS`, `Surrogate-Key`.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt Strong Cache (`max-age=31536000, immutable` → không gửi request) vs Conditional Cache (`ETag`/`Last-Modified` → `If-None-Match` → 304) và vì sao asset versioned dùng strong còn HTML phải `no-cache`/`max-age=0, must-revalidate`?
2. `no-cache` vs `no-store` vs `must-revalidate` vs `stale-while-revalidate` khác gì? Khi nào dùng `stale-while-revalidate=300` cho API?
3. Browser HTTP Cache vs CDN (Edge) Cache (`s-maxage`/`CDN-Cache-Control`/`Surrogate-Key`) vs Service Worker Cache (Cache API) khác nhau về scope, shared, control, invalidate và offline thế nào?

<details>
<summary>Đáp án 30s</summary>

1. **Strong**: fresh trong `max-age`, browser dùng disk cache không gửi request — tiết kiệm cả RTT. **Conditional**: hết fresh hoặc `no-cache`, gửi `If-None-Match: "etag"`; server 304 nếu không đổi — tiết kiệm bandwidth nhưng tốn RTT. Asset `app.[hash].js` URL đổi khi content đổi → `immutable` an toàn, không sợ stale. HTML chứa tham chiếu tới hash mới, nếu `immutable` thì browser không revalidate → kẹt HTML cũ trỏ hash cũ → deploy mới user không thấy. Nên HTML `max-age=0, must-revalidate` để luôn 304/200.

2. `no-cache`: vẫn lưu nhưng luôn revalidate (304). `no-store`: không lưu gì. `must-revalidate`: hết `max-age` phải revalidate, không dùng stale. `stale-while-revalidate`: sau `max-age` vẫn trả stale ngay (0ms) và fetch background trong `stale-while-revalidate` giây — UX nhanh, phù hợp API `max-age=60, stale-while-revalidate=300` (60s fresh, 300s stale vẫn trả ngay).

3. **Browser**: per-user, theo `max-age`, nhanh nhất (memory/disk), mỗi user warm riêng. **CDN**: per-PoP shared, theo `s-maxage`/`CDN-Cache-Control`, 1 user warm cả PoP hưởng, purge bằng `Surrogate-Key` tag. **SW**: per-origin programmable `caches.open().put()`, control bằng JS, cho offline, `skipWaiting`/`clients.claim`, chỉ HTTPS, tồn tại qua reload nhưng xóa khi clear site data. CDN giảm TTFB/origin load; SW cho offline/precache; browser cho tốc độ local.

</details>

---
*Tham khảo chi tiết: `docs/05-performance.md` — Câu 98, 99. Spec: [MDN — Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control), [HTTP Caching — web.dev](https://web.dev/articles/http-cache).*
