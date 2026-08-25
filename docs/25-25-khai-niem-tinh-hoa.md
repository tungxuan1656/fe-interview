# 25 Khái Niệm Tinh Hoa - Học 1 Hiểu 10

> **199 câu -> 25 khái niệm -> 185 câu (bỏ Next.js ch.11 theo yêu cầu)**
> Học 1 khái niệm là trả lời được 5-10 câu hỏi biến thể. Đừng học thuộc câu hỏi, học cơ chế.

**Cách dùng:**
- **Bận (3 ngày):** Chỉ học 12 khái niệm ⭐ (Tầng 0+1)
- **Đủ (7 ngày):** Học hết 25, mỗi ngày 3-4 khái niệm
- Mỗi khái niệm đọc 5 phút, gõ lại ví dụ 5 dòng là nhớ

---

## TẦNG 0: NỀN JAVASCRIPT - 3 khái niệm gốc (4h)

### 1. 🎒 Chiếc Balo Ký Ức (Scope - Closure - Rác)

**Nói nôm na:** Hàm như người đeo balo. Lúc tạo hàm, nó bỏ hết biến xung quanh vào balo (`[[Environment]]`). Dù đi xa (return ra ngoài) vẫn mở balo ra dùng được.

**Gom từ:** C1 `var/let/TDZ` + C2 Closure + C14 Memory Leak + Stale closure trong React

**Học 1 hiểu 4:**
- `var` bỏ vào balo ngay và ghi `undefined`, `let` bỏ vào nhưng khóa lại (TDZ) tới dòng khai báo mới mở
- `for(var i)` 1 balo dùng chung -> `3 3 3`, `for(let i)` mỗi vòng 1 balo mới -> `0 1 2`
- Giữ balo quá lâu = giữ cả DOM 2MB trong `useEffect` -> rò rỉ. `WeakMap` là balo tự hủy khi không ai cầm nữa

```js
function taoDem() {
  let dem = 0; // ở trong balo
  return () => ++dem;
}
const dem = taoDem();
dem(); // 1, balo vẫn giữ `dem` dù taoDem đã chạy xong

// Bẫy React
useEffect(() => {
  const id = setInterval(() => console.log(count), 1000); // count bị kẹt ở giá trị cũ
  return () => clearInterval(id); // quên dòng này là rò rỉ
}, []); // thiếu `count` trong deps
```

**Bẫy:** `typeof` biến chưa khai báo ra `undefined`, nhưng `typeof` biến trong TDZ lại ném lỗi.

**Hỏi 30s:** *"Closure là gì?"* -> *"Hàm nhớ nơi nó được tạo ra, không phải nơi nó được gọi. Dùng cho private state, debounce, nhưng giữ rác nếu không cleanup."*

### 2. 🪪 Cái Tên `this` (Ai đang gọi tôi?)

**Nói nôm na:** `this` không phải tên cố định, mà là "ai đang gọi tôi lúc này". `obj.fn()` thì `this` là `obj`. `const fn = obj.fn; fn()` thì `this` mất, thành `undefined`.

**Gom từ:** C3 `this` + C12 `call/apply/bind` + C8 Curry + C9 Debounce/Throttle

**Nhớ 3 điều:**
1. Thứ tự ưu tiên: `new` > `bind` > `obj.fn()` > gọi trần
2. `bind` là dán keo cứng `this`, `call` lại cũng không gỡ được
3. Arrow function không có `this` riêng, nó mượn của người bao ngoài

```js
const user = { ten: 'An', chao() { console.log(this.ten) } }
user.chao(); // 'An' - ai gọi? user gọi
const fn = user.chao; fn(); // undefined - không ai gọi

const debounced = debounce(user.chao.bind(user), 300); // phải bind, không là mất this
```

**Bẫy:** `bind` arrow function vô tác dụng. Tạo `debounce` mới mỗi lần render là mất tác dụng.

### 3. 🚦 Ngã Tư Bất Đồng Bộ (Event Loop)

**Nói nôm na:** JS chỉ có 1 đường. Xe ưu tiên (microtask: `Promise`) đi trước, xe thường (macrotask: `setTimeout`) đi sau. `await` chính là dừng lại, nhường đường cho xe ưu tiên chạy.

**Gom từ:** C5 Event Loop + C6 Promise/async + C7 Generator + C18 lỗi async

**Nhớ 3 điều:**
1. `Promise.then` luôn đi trước `setTimeout(0)` dù số 0
2. `await` dù `Promise` đã xong vẫn phải xếp hàng microtask
3. `try/catch` không bắt được lỗi ném trong `setTimeout`

```js
console.log(1);
setTimeout(()=>console.log(2),0);
Promise.resolve().then(()=>console.log(3));
console.log(4);
// 1, 4, 3, 2  (xe ưu tiên 3 đi trước xe thường 2)

await Promise.resolve(); // thực chất tách thành .then()
```

---

## TẦNG 1: ĐỒ VẬT & CHỮ - 4 khái niệm (3h)

### 4. 👨‍👩‍👧 Dòng Họ (Prototype & Copy & Proxy)

**Nói nôm na:** Mỗi object có sợi dây nối với cha (`__proto__`). Tìm đồ không thấy thì leo dây hỏi cha. `class` chỉ là cách viết ngắn gọn cho việc nối dây. `Proxy` là người gác cổng, chặn mọi lần sờ vào đồ.

**Gom từ:** C4 Prototype + C10 Shallow/Deep + C16 Proxy + C17 Symbol/WeakMap

**Nhớ 3 điều:**
1. `...spread` chỉ photo 1 mặt, đồ bên trong vẫn dùng chung -> đổi `nested.y` là đổi luôn gốc
2. `Proxy` gác được cả việc thêm khóa mới, `Object.defineProperty` (Vue2) thì không
3. `structuredClone` photo được Date/Map nhưng không photo được function

```js
const goc = { a:1, con:{ b:2 } }
const photo = { ...goc }; // shallow
photo.con.b = 99; // goc.con.b cũng thành 99!
const that = structuredClone(goc); // deep thật
```

### 5. 🚪 Cửa Khẩu Module (CJS vs ESM)

**Nói nôm na:** `CommonJS` như chợ trời: thích `require` lúc nào cũng được. `ESM` như sân bay: phải khai báo `import` ở cửa, nên hải quan (bundler) mới soi được đồ nào không dùng để vứt (tree-shaking).

**Gom từ:** C15

**Nhớ:** ESM `import {count} from './a'` là nhìn trực tiếp, bên kia `count++` bên này thấy luôn (live). CJS thì photo 1 bản.

### 6. 📏 Thước Đo Chữ (TypeScript Nền)

**Nói nôm na:** `any` = tắt thước, đo bừa. `unknown` = khóa thước, phải đo lại mới dùng. `never` = không có gì vừa. `strictNullChecks` = bắt buộc ghi `string | null` chứ không cho `string` mà nhét `null`.

**Gom từ:** C19 any/unknown/never + C20 type vs interface + C25 strict + C26 Enum/Union + C28 Narrowing

**Nhớ 3 điều:**
1. `type` làm được `A | B`, `interface` thì gộp tên được (merge)
2. `Union` + `as const` tốt hơn `Enum` (Enum sinh code thừa)
3. `if(typeof x==='string')` chính là thước đo (narrowing)

```ts
let a: unknown = "hi";
a.toUpperCase() // lỗi, phải if(typeof a==='string') trước

type Hinh = {loai:'tron', r:number} | {loai:'vuong', canh:number}
function tinh(h:Hinh){ if(h.loai==='tron') return Math.PI*h.r**2 } // narrow
```

### 7. 🧙 Phù Thủy Biến Hình (Type Nâng Cao)

**Nói nôm na:** Viết kiểu dữ liệu như viết hàm. `Generics` là khuôn, `Partial` là biến mọi thứ thành optional, `in` + `` `get${Capitalize<K>}` `` là dán chữ.

**Gom từ:** C21 Generics + C22 Utility + C23 Conditional/infer + C24 Mapped/Template + C27 Decorator + C29 Merging + C30 Monorepo

**Học 1 hiểu 1 mạch:**
```ts
type PartialCuaToi<T> = { [K in keyof T]?: T[K] } // tự làm Partial
type LayKetQua<T> = T extends (...a:any)=> infer R ? R : never // infer = móc ra
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: ()=>T[K] }
```
*Decorator `@Log` + `declare global` chỉ là cách dán thêm đồ vào nhà người khác.*

---

## TẦNG 2: REACT - 6 khái niệm (5h)

### 8. 🎬 Xưởng Phim (Render & Fiber & Key)

**Nói nôm na:** `render` là vẽ phác thảo (React Element), chưa dán lên tường (DOM). `Reconciliation` là so 2 bản phác thảo để dán ít nhất. `key` là số báo danh, không có là cô giáo chấm nhầm bài. `Fiber` là chia việc vẽ thành tờ nhỏ để đang vẽ dở mà có việc gấp (gõ phím) thì dừng lại.

**Gom từ:** 02-31 render + 02-32 VDOM + 02-33 key + 03-51 Fiber

**Nhớ 3 điều:**
1. Cha render -> con render theo dù props không đổi, trừ khi `memo`
2. `key={index}` khi xóa giữa danh sách sẽ làm checkbox tích nhầm người
3. `key` không nằm trong `props`, `key={Math.random()}` mỗi lần vẽ lại là phá luôn

```jsx
{todos.map(t => <Todo key={t.id} data={t} />)} // đúng
{todos.map((t,i) => <Todo key={i} />)} // sai khi reorder
```

### 9. 🔌 Công Tắc Điện (Hooks & Effect)

**Nói nôm na:** Hooks như ổ điện xếp hàng theo thứ tự. Cắm sai thứ tự là chập điện (vi phạm Rules). `useState` là công tắc có nhớ, `useEffect` là hẹn giờ "khi nào xong việc thì chạy", `useRef` là hộp bí mật không gây ồn (không render lại).

**Gom từ:** 02-35 useState batching + 02-36/37 useEffect/layoutEffect + 02-39 useRef + 02-49 Rules

**Nhớ 3 điều:**
1. `setCount(c=>c+1)` 3 lần mới thành +3, `setCount(count+1)` 3 lần chỉ +1 (do kẹt closure)
2. `useEffect` chạy sau khi sơn xong, `useLayoutEffect` chạy trước khi sơn để đo đạc không nháy
3. `useRef` đổi `current` không render lại -> dùng để giữ `setInterval` id, giá trị cũ

```jsx
useEffect(()=>{
  const c = new AbortController();
  fetch(`/api/user/${id}`, {signal:c.signal}).then(...)
  return ()=> c.abort(); // dọn dẹp, không là race condition
}, [id]) // thiếu id là bug
```

### 10. 🧹 Chế Độ Tiết Kiệm (Memo & Re-render)

**Nói nôm na:** `memo` như dán giấy "đừng vẽ lại nếu đồ không đổi". Nhưng dán giấy cũng tốn băng keo. Chỉ dán khi đo thấy lag.

**Gom từ:** 02-38 memo + 02-40 Context + 02-43 Colocation + 03-55 Transition + 03-63 Profiler

**Nhớ 3 điều:**
1. `Context value={{a,b}}` tạo object mới mỗi lần -> mọi con đều render lại, phải `useMemo` value hoặc tách Context
2. Để state gần nơi dùng nhất (colocation) nhanh hơn nâng lên App
3. Gõ tìm kiếm list 10k: `useDeferredValue` để ô input mượt, list cập nhật sau

```jsx
const deferredQuery = useDeferredValue(query); // query gõ mượt
const list = useMemo(()=> filter(data, deferredQuery), [deferredQuery]);
```

### 11. 🔀 Đường Dây Điện (Chia State)

**Nói nôm na:** Đừng kéo dây điện từ tầng 1 lên tầng 10 chỉ để bật đèn tầng 10 (prop drilling). Có 3 cách: đưa bóng đèn xuống gần công tắc (composition `children`), đi đường dây chung (Context), hoặc dùng tổng đài (store).

**Gom từ:** 02-41 drilling + 02-42 Compound + 02-44 Redux/Zustand/Jotai + 02-45 RTK

**Nhớ:** `Zustand` như công tắc đơn giản, `Redux` như tủ điện có sổ sách (time-travel), `Jotai` như mỗi bóng 1 công tắc riêng (atomic) -> đỡ render ké.

### 12. 🚧 Đội Cứu Hộ (Suspense & Lỗi)

**Nói nôm na:** `Suspense` là màn "đang tải...", `Error Boundary` là lưới an toàn hứng khi rơi. `lazy` là chỉ khi nào cần mới đi lấy đồ.

**Gom từ:** 03-56 Suspense + 03-57 Error Boundary + 03-62 Code Splitting

```jsx
<Suspense fallback={<Skeleton />}><Comments /></Suspense>
<ErrorBoundary fallback={<Loi roi />}><App /></ErrorBoundary>
const Chart = lazy(()=> import('./Chart')); // route-based split
```

### 13. 🏭 Nhà Máy Server (SSR & Cache)

**Nói nôm na:** `CSR` là khách tự nấu (trắng trang rồi mới hiện). `SSR` là bếp nấu sẵn gửi ra (nhanh nhưng bếp bận). `SSG` là nấu sẵn từ sáng, `ISR` là nấu sẵn nhưng 60s hâm lại. `RSC` là món chỉ bếp làm, không gửi công thức ra bàn.

**Gom từ:** 03-58 RSC + 03-60 SSR/SSG/ISR + 03-61 Streaming + 03-59 Hydration + 03-66 Actions + 03-67 Cache + C52-54 React 18/19 + C65

**Nhớ 3 điều:**
1. `Date.now()` trong render sẽ làm Hydration lệch (server vs client khác nhau)
2. Next.js có 4 két: `Request Memo` (1 request) -> `Data Cache` (force-cache) -> `Route Cache` -> `Router Cache` (client)
3. React 19 `use` đọc Promise ngay trong render, `Compiler` tự dán `memo` cho bạn

---

## TẦNG 3: TRÌNH DUYỆT & TỐC ĐỘ - 5 khái niệm (4h)

### 14. 🎨 Dây Chuyền Sơn (Reflow & Transform)

**Nói nôm na:** Sơn nhà 3 bước: Dựng khung (Layout) -> Quét sơn (Paint) -> Dán giấy (Composite). Đổi `width` là đập khung làm lại, đổi `transform` chỉ dán lại giấy -> rẻ nhất, chạy trên GPU 60fps.

**Gom từ:** 06-105 Pipeline + 06-106 Reflow/Paint + 06-107 Layer + 06-108 fixed + 06-109 transform + 05-102 scroll jank + 05-103 Leak

**Nhớ 3 điều:**
1. Đọc `offsetHeight` rồi ghi `style.height` trong vòng lặp là tự đập khung liên tục (thrashing) -> phải gom đọc trước, ghi sau
2. `will-change: transform` như đặt trước 1 lớp kính, tốn RAM (1 màn 8MB), dùng xong phải gỡ
3. `transform` trên cha sẽ làm `position:fixed` của con kẹt theo cha

```js
// ❌
els.forEach(el=>{ el.style.height = el.offsetHeight+10+'px' })
// ✅
const h = els.map(el=>el.offsetHeight); els.forEach((el,i)=> el.style.height = h[i]+10+'px')
```

### 15. 🏷️ Biển Số Xe (Web Vitals)

**Nói nôm na:** Google chấm 3 điểm: `LCP` (ảnh to nhất hiện khi nào <2.5s), `INP` (bấm nút bao lâu mới phản hồi <200ms), `CLS` (trang có nhảy lung tung <0.1). `Lighthouse` là thi thử, `RUM` là thi thật ngoài đường.

**Gom từ:** 05-86 bottleneck + 05-87 đo + 05-88 Vitals + 05-89 LCP + 05-90 INP + 05-91 CLS + 05-92 tối ưu + 05-104 Budget

**Nhớ:** Ảnh hero phải `preload + fetchpriority=high + width/height`, đừng `lazy`.

### 16. 📦 Kho Bưu Kiện (Ảnh/Font/Bundle)

**Nói nôm na:** Ảnh như bưu kiện: gửi đúng size (`srcset` 400w/800w cho điện thoại/máy tính), đúng loại (AVIF nhỏ nhất). Font như quần áo: chỉ gửi size mình mặc (subset latin), `swap` là mặc tạm đồ cũ kẻo nude (FOIT).

**Gom từ:** 05-93 Bundle + 05-94 Splitting + 05-95 Image + 05-96 Font + 05-97 preload/prefetch

**Nhớ 3 điều:**
1. `import _ from 'lodash'` kéo cả kho, `import debounce from 'lodash-es/debounce'` chỉ lấy 1 món
2. `preload` là lấy ngay (hero), `prefetch` là lấy khi rảnh (trang sau), `preconnect` là mở đường trước
3. `barrel export * from './Button'` làm bundler mù, không tree-shake được

### 17. 🔐 Két Sắt Trình Duyệt (Cache & Offline)

**Nói nôm na:** `max-age=31536000, immutable` là niêm phong 1 năm (chỉ dùng cho file có hash `app.abc123.js`). `ETag` là vân tay, lần sau gửi `If-None-Match` nếu khớp server trả `304` không cần gửi lại. `Service Worker` là người gác kho lập trình được.

**Gom từ:** 05-98 HTTP Cache + 05-99 CDN/Browser/SW + 06-118 SW

**Nhớ:** `sw.js` phải `no-cache` kẻo kẹt bản cũ. `Cache First` cho ảnh, `Network First` cho API.

### 18. 🧳 Tủ Đồ Cá Nhân (Storage)

**Nói nôm na:** `Cookie` 4KB tự gửi kèm mỗi lần đi chợ (kèm `HttpOnly` thì JS không đọc được). `localStorage` 5MB để ở nhà, `IndexedDB` như kho hàng GB. `PWA` là biến web thành app (cần Manifest + SW + HTTPS).

**Gom từ:** 06-112 Cookie vs Storage + 06-120 IndexedDB + 06-119 PWA

**Nhớ:** Đừng để JWT trong `localStorage` (XSS đọc được). Dùng `httpOnly` cookie.

---

## TẦNG 4: MẠNG & HỢP ĐỒNG - 2 khái niệm (2h)

### 19. 🛂 Biên Giới (Origin & CORS & HTTP)

**Nói nôm na:** `Same-Origin` là biên giới (scheme+host+port). `CORS` là giấy thông hành do server cấp (`Access-Control-Allow-Origin`). `OPTIONS` là hỏi trước "tôi mang `Authorization` có được qua không?".

**Gom từ:** 06-111 CORS + 06-113 SOP + 06-114 HTTP/1.1/2/3 + 06-116 WS + 06-117 SSE + 06-110 Event

**Nhớ 3 điều:**
1. `Content-Type: application/json` sẽ trigger `OPTIONS`, `text/plain` thì không
2. `Allow-Origin: *` không đi với `credentials: true`
3. `HTTP/2` gộp nhiều làn trên 1 đường (multiplex), `HTTP/3` chạy trên UDP nên không kẹt 1 xe là kẹt cả đoàn. `SSE` 1 chiều (server đẩy), `WS` 2 chiều

### 20. 📜 Ngôn Ngữ Hợp Đồng (REST)

**Nói nôm na:** `REST` là quy ước đặt tên đường: `GET /products` (lấy), `POST` (tạo), `PUT` (thay cả), `PATCH` (vá 1 miếng). `PUT` dán lại 10 lần vẫn 1 kết quả (idempotent), `POST` dán 10 lần ra 10 đơn.

**Gom từ:** 08-135 REST + 08-136 PUT/PATCH + 08-137 Status + 08-138 401/403 + 08-139 404/410 + 08-140 400/422

**Nhớ:** `401` chưa đăng nhập, `403` đã đăng nhập nhưng không có quyền. `404` không biết có từng tồn tại không, `410` là xóa vĩnh viễn (Google xóa index ngay).

---

## TẦNG 5: KIẾN TRÚC - 2 khái niệm (2h)

### 21. 🗺️ Bản Vẽ Thành Phố (Architecture)

**Nói nôm na:** `Feature-Sliced` là chia thành phố theo khu (auth, cart) mỗi khu có đủ nhà/đường, đừng chia theo loại gạch (components/hooks/utils). `Design System` là bộ gạch chuẩn. `God Component` là tòa nhà 2000 dòng làm mọi thứ.

**Gom từ:** 04-69 structure + 04-70 Feature vs Layer + 04-71 chéo + 04-72 Design System + 04-73 reusable + 04-74 God + 04-81 Migrate + 04-83 MFE

**Nhớ 3 điều:**
1. `shared` không được import `features`, `features` không import nhau
2. Chỉ tách reusable khi thấy lặp 3 lần (Rule of Three)
3. `Micro-frontend` chỉ khi >30 dev, 4 team deploy riêng, còn lại `Monorepo + Turborepo` đủ

### 22. 🚚 Đồng Bộ Hàng Hóa (Data Sync)

**Nói nôm na:** Lấy hàng có thể kẹt xe (race), hàng cũ (stale), mất mạng. `AbortController` là hủy đơn cũ khi đặt đơn mới. `Optimistic` là cho khách cầm hàng trước, nếu lỗi thì giật lại. `ETag` là tem kiểm hàng.

**Gom từ:** 08-141 Retry + 08-142 Backoff + 08-143 Race + 08-144 Autocomplete + 08-145 Abort + 08-146 Offset/Cursor + 08-147 Infinite + 08-148 Optimistic + 08-149 ETag + 08-150 Contract + 04-78 API Layer + 04-79 Error + 04-80 Loading + 05-100 Virtualization

**Nhớ 3 điều:**
1. `POST` không retry nếu không có `Idempotency-Key` sẽ tạo 2 đơn
2. `Cursor` (dùng `id`) nhanh hơn `Offset` khi lướt sâu, không bị lệch khi có người chen hàng
3. `stale-while-revalidate` là cho đồ cũ trước, đi lấy đồ mới sau lưng

```js
let controller;
async function search(q){
  controller?.abort();
  controller = new AbortController();
  const res = await fetch(`/api/search?q=${q}`, {signal: controller.signal});
}
```

---

## TẦNG 6: TRẠNG THÁI & CHẤT LƯỢNG - 3 khái niệm (3h)

### 23. 🧠 Chia Việc Thông Minh (State) ⭐ BỔ SUNG QUAN TRỌNG

**Nói nôm na:** Đừng bỏ mọi đồ vào 1 tủ (Redux). Hỏi 3 câu: Đồ này của ai? (server/client), Dùng ở mấy phòng? (global/local), Có cần dán lên tường để share link không? (URL)

**Gom từ:** 04-75 Shared + 04-76 5 loại + 04-77 Server vs Client + 09-151->160 (10 câu bị sót) + 04-77

**Bảng 1 trang nhớ luôn:**

| Đồ | Để đâu | Vì sao |
|---|---|---|
| `products`, `orders` | `TanStack Query` | Server state, cần cache/stale/invalidate |
| `user` | `Query + Context` | Server nhưng dùng khắp nơi |
| `cart` | `Zustand + persist + sync /api/cart` | Hybrid: cần offline + đồng bộ |
| `filter`, `page` | `URL ?category=shoes` | Để share link, back/forward |
| `search input` | `useState` local | Chỉ 1 ô input dùng |
| `theme/modal` | `Zustand` hoặc `Context` | Client global ít đổi |

**TanStack 30s:**
- `staleTime: 0` (mặc định) = luôn cũ -> tự fetch lại khi focus
- `gcTime: 5m` = 5 phút không ai dùng thì xóa
- `invalidateQueries` = đánh dấu cũ, `setQueryData` = nhét luôn không cần fetch

```js
const {data} = useQuery({queryKey:['products', cat], queryFn:()=>fetchProducts(cat), staleTime: 60*1000})
// sau khi POST
queryClient.invalidateQueries({queryKey:['products']})
```

**Bẫy:** Lưu `products` vào Redux rồi tự `useEffect` fetch là tự làm lại TanStack dở hơn.

### 24. 🔍 Tháp Kiểm Thử (Testing) ⭐ BỔ SUNG

**Nói nôm na:** Kim tự tháp: đáy to là `Unit` (nhanh <50ms), giữa là `Integration` (RTL+MSW 200ms), đỉnh nhỏ là `E2E` (Playwright 5s). Đừng xây tháp ngược (toàn E2E) sẽ sập (flaky, chậm).

**Gom từ:** 10-161->172 (12 câu bị sót 0/12)

**Nhớ 3 điều:**
1. `RTL` test như người dùng: `getByRole('button', {name:/mua/i})` > `getByTestId`
2. `MSW` chặn ở tầng mạng (đáng tin) khác `vi.mock` chặn module
3. `Playwright` (chạy ngoài, đa tab, trace) hơn `Cypress` (chạy trong), `Vitest` (dùng Vite) nhanh hơn `Jest` 3-10 lần

```js
// Integration: RTL + MSW
render(<ProductList />)
await user.click(screen.getByRole('button', {name:/thêm/i}))
expect(await screen.findByText(/đã thêm/i)).toBeInTheDocument()
```

**Bẫy:** `Snapshot` cho page 500 dòng -> mù, chỉ dùng cho component nhỏ. Đừng ép coverage 100%.

### 25. 🎤 Kể Chuyện STAR (Behavioral) ⭐ BỔ SUNG

**Nói nôm na:** Giám khảo không chấm đúng/sai, chấm cách bạn kể. Dùng khung `STAR + Trade-off + Con số`.

**Gom từ:** 14-187->199 (13 câu bị sót 0/13)

**Công thức 1 phút cho 13 câu:**

`S (cảnh 1 câu + số) -> T (nhiệm vụ của bạn) -> A (3 bước: lắng nghe/data -> quy tắc chung -> thử & đo) -> R (kết quả % ) -> Lesson`

**4 story tủ xoay cho cả 13 câu:**
1. **Conflict:** PR 800 dòng -> chia `Must/Should/Nit` + call 15' -> PR sau tốt 50%
2. **Failure:** Chọn Redux-saga cho app 10 trang -> onboarding 2 tuần -> migrate Zustand velocity +30% -> lesson: "boring solution"
3. **Influence:** CRA 180s -> PoC Vite 25s + RFC + pilot 1 route -> team vote
4. **Incident:** Checkout 500 -> `0-5' confirm -> 5-10' flag/rollback -> 10-20' log -> 20-30' hotfix + post-mortem`

**Template thuyết phục PM:** *"Mình hiểu mục tiêu tăng conversion. Cách A tốn 2 tuần + LCP +1s. Mình đề xuất B 3 ngày + A/B test, nếu CTR tăng mới đầu tư. Anh thấy sao?"*

---

## CHECKLIST 7 NGÀY

- [ ] Ngày 1: 1,2,3 (Balo, This, Ngã Tư)
- [ ] Ngày 2: 4,5,6,7 (Dòng Họ, Cửa Khẩu, Thước, Phù Thủy)
- [ ] Ngày 3: 8,9,10,11 (Xưởng Phim, Công Tắc, Tiết Kiệm, Dây Điện)
- [ ] Ngày 4: 12,13,14,15 (Cứu Hộ, Nhà Máy, Dây Chuyền, Biển Số)
- [ ] Ngày 5: 16,17,18,19,20 (Kho, Két, Tủ, Biên Giới, Hợp Đồng)
- [ ] Ngày 6: 21,22,23,24 (Bản Vẽ, Đồng Bộ, Chia Việc, Tháp)
- [ ] Ngày 7: 25 STAR - ghi âm 4 story, mock 30 phút

> Mỗi khái niệm xong tự hỏi: *"Giải thích cho đứa em lớp 9 hiểu được không?"* Nếu được là đã hiểu thật.

*File này phủ 185/199 câu (bỏ 14 câu Next.js ch.11). Đọc chi tiết từng câu gốc ở `docs/01->10,14`.*
