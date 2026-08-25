# Hydration, Selective Hydration và Streaming SSR — hydrateRoot, mismatch, renderToPipeableStream

> Tags: #react #hydration #selective-hydration #streaming #mismatch #hydrateRoot | Nguồn: `docs/03-react-nang-cao.md` câu 59, 61 + `docs/11-nextjs.md` câu 182 | Mức: P0

## 1. Định nghĩa chính xác

- **Hydration** là quá trình client React **attach event listeners và state** vào HTML đã SSR thay vì tạo DOM mới. Dùng `hydrateRoot(container, <App />)` (React 18+), thay cho `createRoot` + `render` của CSR. React cố gắng **reuse** DOM server, chỉ gắn Fiber và listeners.
- **Hydration mismatch** là khi HTML/text của lần render đầu trên client khác HTML server đã gửi → React cảnh báo `Hydration failed because the initial UI does not match...`, bỏ DOM server và **client-render lại** (mất lợi SSR, layout shift).
- **Selective Hydration** là tối ưu React 18: với **Streaming SSR + Suspense**, các Suspense boundary đã stream xong sẽ **hydrate trước**, không cần đợi cả page. Phần user đang tương tác được ưu tiên hydrate ngay (ví dụ: click vào fallback thì React ưu tiên hydrate boundary đó).
- **Streaming SSR** là server **gửi HTML từng phần ngay khi xong** (chunked), không đợi hết data, bằng `renderToPipeableStream` (Node) hoặc `renderToReadableStream` (Edge/Deno). Kết hợp Suspense: phần chưa ready gửi `fallback`, khi ready stream thêm HTML + inline `<script>` để hydrate.
- **`suppressHydrationWarning`** là prop cho phép tắt cảnh báo mismatch cho subtree cố ý khác (ví dụ: `<time>` hiển thị giờ).

## 2. Cơ chế hoạt động

### 2.1 Hydration — reuse vs discard

```
Server:  renderToPipeableStream(<App />) → HTML string/stream → gửi browser
Browser: HTML đã hiển thị (FCP nhanh, SEO tốt)
         hydrateRoot(container, <App />) → React tạo Fiber, so sánh VDOM client với DOM server
           - Nếu khớp → attach listeners, giữ DOM
           - Nếu mismatch → warn, discard DOM server, render lại client (như createRoot)
```

- `hydrateRoot` cũng concurrent như `createRoot`, nhưng thêm logic so khớp. Có `onRecoverableError` để log mismatch.

### 2.2 Nguyên nhân mismatch phổ biến

| Nguyên nhân | Vì sao khác server/client |
|-------------|---------------------------|
| `Date.now()`, `new Date().toString()`, `Intl.DateTimeFormat` locale | Server và client khác timezone/giờ render |
| `Math.random()`, `crypto.randomUUID()` trong render | Mỗi lần render khác nhau |
| `window.innerWidth`, `localStorage`, `navigator`, `matchMedia` | Server không có `window` |
| `useId` không dùng — tự sinh id bằng random | Server/client id khác |
| Render có điều kiện theo `typeof window !== 'undefined'` | Server false, client true |

Fix: hoãn giá trị khác biệt tới `useEffect` (chỉ client), hoặc `suppressHydrationWarning`, hoặc `dynamic(() => import(...), { ssr: false })` cho phần client-only.

### 2.3 Selective Hydration — ưu tiên

```
Streaming SSR gửi:
[Shell + Header] → hydrate ngay (không đợi Comments)
[Product]        → hydrate khi stream xong
[Comments fallback] → khi Comments fetch xong, stream HTML Comments + script → hydrate Comments
```

- React không block: phần đã có HTML + JS sẽ hydrate trước, dù phần khác chưa stream xong.
- Tương tác ưu tiên: user click vào Suspense fallback → React **ưu tiên hydrate** boundary đó trước (dù chưa xong priority).
- Yêu cầu: phải có `<Suspense>` boundary. Không có Suspense thì cả page là một boundary lớn, phải đợi hết.

### 2.4 Streaming SSR — renderToPipeableStream vs renderToString

```
renderToString:   đợi toàn bộ data → tạo HTML string → gửi 1 lần (TTFB chậm, trắng màn hình lâu)
renderToPipeableStream:  render shell ngay → pipe chunk khi sẵn sàng → TTFB nhanh, FCP sớm
```

API Node:

```js
import { renderToPipeableStream } from 'react-dom/server';
const { pipe, abort } = renderToPipeableStream(<App />, {
  bootstrapScripts: ['/main.js'],
  onShellReady() { res.statusCode = 200; pipe(res); }, // gửi shell ngay khi shell ready
  onShellError(err) { res.statusCode = 500; res.send('<h1>Shell error</h1>'); },
  onAllReady() { /* toàn bộ đã render, dùng cho crawler cần full HTML */ },
  onError(err) { console.error(err); },
});
```

- Với Suspense: `onShellReady` gửi shell + fallback; khi Suspense resolve, React tự stream thêm `<template>` + script để thay fallback.
- Edge: `renderToReadableStream` trả `ReadableStream` cho `Response`.

### 2.5 Next.js App Router tự streaming

- Next.js tự dùng `renderToPipeableStream`/`ReadableStream` cho Server Components. Chỉ cần bọc `async` component trong `<Suspense>` hoặc dùng `loading.tsx` (file-based Suspense).
- Không cần tự gọi `renderToPipeableStream` trong App Router.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 hydrateRoot — cơ bản
import { hydrateRoot } from 'react-dom/client';
import App from './App';

// HTML đã được SSR vào #root
hydrateRoot(
  document.getElementById('root')!,
  <App />,
  {
    onRecoverableError: (err, info) => {
      console.error('Hydration error:', err, info.componentStack);
      // logToService(err);
    },
  }
);

// 3.2 Mismatch — sai và fix
function BadClock() {
  // ❌ Sai: mỗi render khác nhau → mismatch
  return <div>{new Date().toLocaleTimeString()}</div>; // server 10:00, client 10:01 → warn
}

function GoodClock() {
  const [time, setTime] = React.useState<string | null>(null);
  React.useEffect(() => {
    // ✅ Chỉ chạy client sau hydrate, server render null/null → khớp lần đầu
    setTime(new Date().toLocaleTimeString());
    const id = setInterval(() => setTime(new Date().toLocaleTimeString()), 1000);
    return () => clearInterval(id);
  }, []);
  if (time === null) return <div suppressHydrationWarning>loading</div>;
  return <div>{time}</div>;
}

// Hoặc tắt warn cho phần cố ý khác
function TimeWithSuppress() {
  // suppressHydrationWarning chỉ tắt warn, vẫn giữ DOM server
  return <div suppressHydrationWarning>{new Date().toString()}</div>;
}

// 3.3 window trong render — sai
function BadWidth() {
  // ❌ Server không có window → mismatch / ReferenceError
  return <div>{window.innerWidth}px</div>;
}
function FixedWidth() {
  const [width, setWidth] = React.useState<number | null>(null);
  React.useEffect(() => {
    const update = () => setWidth(window.innerWidth);
    update();
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  }, []);
  if (width === null) return <div>loading</div>;
  return <div>{width}px</div>;
}

// 3.4 useId cho id ổn định, tránh mismatch
function InputWithId({ label }: { label: string }) {
  const id = React.useId(); // SSR-safe, server/client cùng id ":r0:"
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
// ❌ Math.random() cho id → mismatch
function BadId() {
  const id = Math.random().toString(36); // mỗi lần khác
  return <input id={id} />;
}

// 3.5 Selective Hydration với Suspense + streaming
import { Suspense } from 'react';

function AppStreaming() {
  return (
    <div>
      <Header /> {/* static, hydrate ngay */}
      <Suspense fallback={<ProductSkeleton />}>
        <Product /> {/* async, stream sau */}
      </Suspense>
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments /> {/* chậm nhất, hydrate cuối, nhưng nếu user click vào fallback thì ưu tiên */}
      </Suspense>
    </div>
  );
}

// Server Components async tự suspend (Next.js)
async function Comments() {
  const comments = await fetch('https://api.example.com/comments', { cache: 'no-store' }).then(r => r.json());
  return <>{comments.map((c: any) => <div key={c.id}>{c.text}</div>)}</>;
}
function ProductSkeleton() { return <div>Loading product…</div>; }
function CommentsSkeleton() { return <div>Loading comments…</div>; }
function Header() { return <header>Shop</header>; }
function Product() { return <div>Product</div>; }

// 3.6 Next.js App Router — loading.tsx (Suspense file-based)
// app/products/loading.tsx
export default function Loading() {
  return <div className="grid"><Skeleton /><Skeleton /><Skeleton /></div>;
}
// app/products/page.tsx
import { Suspense } from 'react';
export default function ProductsPage() {
  return (
    <div>
      <h1>Products</h1>
      <Suspense fallback={<ProductSkeleton />}>
        <ProductList /> {/* async Server Component */}
      </Suspense>
    </div>
  );
}
async function ProductList() {
  const products = await fetch('https://api.example.com/products').then(r => r.json());
  return <>{products.map((p: any) => <div key={p.id}>{p.name}</div>)}</>;
}

// 3.7 Streaming SSR thủ công (Node/Express) — demo, Next.js tự làm
// server.js
import { renderToPipeableStream } from 'react-dom/server';
import App from './App';

import express from 'express';
const app = express();
app.get('/', (req, res) => {
  const { pipe, abort } = renderToPipeableStream(<App />, {
    bootstrapScripts: ['/main.js'],
    onShellReady() {
      // Gửi shell ngay (TTFB nhanh)
      res.statusCode = 200;
      res.setHeader('Content-Type', 'text/html');
      pipe(res);
    },
    onShellError(err) {
      res.statusCode = 500;
      res.send('<h1>Shell error</h1>');
    },
    onError(err) {
      console.error(err);
    },
  });
  // Timeout abort nếu stream treo
  setTimeout(() => abort(), 10000);
});

// 3.8 Edge — renderToReadableStream
import { renderToReadableStream } from 'react-dom/server';
export async function GET() {
  const stream = await renderToReadableStream(<App />, {
    bootstrapScripts: ['/main.js'],
  });
  return new Response(stream, { headers: { 'Content-Type': 'text/html' } });
}
```

## 4. So sánh / Phân loại

| Tiêu chí | `createRoot` (CSR) | `hydrateRoot` (SSR) |
|----------|--------------------|---------------------|
| DOM ban đầu | Rỗng, React tạo mới | Đã có HTML từ server, React reuse |
| Công việc client | Tạo DOM + attach listeners | So sánh + attach listeners (nhanh hơn) |
| Mismatch | Không (không có HTML sẵn) | Có — nếu client render khác server → warn + discard |
| Selective hydration | Không | Có (với Suspense + streaming) |
| Dùng khi | SPA thuần, không SSR | SSR/SSG, cần SEO/FCP |

| Tiêu chí | `renderToString` (sync) | `renderToPipeableStream` (streaming) | `renderToReadableStream` (Edge) |
|----------|------------------------|--------------------------------------|--------------------------------|
| Khi gửi HTML | Đợi hết data mới gửi | Gửi shell ngay, stream phần còn lại | Tương tự, nhưng trả ReadableStream |
| TTFB | Chậm (đợi data chậm nhất) | Nhanh (shell 50ms) | Nhanh |
| Suspense | Không (phải đợi) | Có — fallback trước, stream sau | Có |
| Runtime | Node | Node (pipe) | Edge/Web (Response) |
| Dùng | Legacy, crawler cần full HTML | Khuyến nghị cho SSR streaming | Next.js Edge, Cloudflare |

| Tiêu chí | Hydration mismatch | Selective hydration |
|----------|--------------------|---------------------|
| Là gì | Lỗi — server HTML ≠ client HTML lần đầu | Tối ưu — hydrate từng Suspense boundary theo thứ tự stream/ưu tiên |
| Gây ra | `Date`, `random`, `window` trong render | Không — là tính năng |
| Hậu quả | Discard + client render lại, mất lợi SSR | FCP/TTI nhanh hơn, TTFI giảm |

| Cách fix mismatch | Khi dùng |
|-------------------|----------|
| `useEffect` hoãn giá trị tới client | `Date`, `window`, `random` |
| `suppressHydrationWarning` | Text nhỏ cố ý khác (giờ, random id hiển thị) |
| `useId` thay `Math.random()` | `label`/`input` id |
| `dynamic(..., { ssr: false })` | Component chỉ client (chart, map) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng `hydrateRoot` cho CSR thuần**: Nếu HTML rỗng (Vite SPA), `hydrateRoot` sẽ warn mismatch và discard → chậm hơn `createRoot`. Chỉ `hydrateRoot` khi có HTML SSR.
- **Không để mismatch**: Mỗi mismatch làm React discard cả subtree và client-render lại → mất FCP của SSR, layout shift, double JS. Phải đảm bảo server/client render lần đầu giống nhau.
- **Không lạm dụng `suppressHydrationWarning`**: Nó chỉ tắt warn, không fix nguyên nhân. Nếu cả block lớn khác nhau, vẫn discard. Chỉ dùng cho text nhỏ cố ý khác, không cho layout.
- **Không stream mọi thứ**: Streaming làm **layout shift** nếu fallback không đúng kích thước (skeleton khác content). Cần skeleton khớp kích thước, hoặc `min-height`.
- **Không dùng `renderToString` cho page có data chậm**: TTFB sẽ bằng thời gian data chậm nhất (1s). `renderToPipeableStream` cải thiện TTFB 20-40% với Suspense.
- **Chi phí streaming**: Phức tạp hơn (backpressure, abort, error boundary), cần framework hỗ trợ. Với Next.js App Router không cần tự làm; với Express custom server mới cần.
- **Khi nào KHÔNG cần selective hydration**: Page nhỏ, không có Suspense, hoặc không SSR — selective không có tác dụng.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `Math.random()`/`Date.now()` trong render gây mismatch**
  - Triệu chứng: Console `Hydration failed ...`, `Text content does not match server-rendered HTML`, layout shift, sau đó React client-render lại.
  - Fix: Chuyển `Math.random()`/`Date` vào `useEffect` hoặc `useMemo` với `suppressHydrationWarning` cho display. Dùng `useId` cho key/id.
  - Đo: `hydrateRoot(..., { onRecoverableError: (err) => console.error(err) })`, DevTools → Components → highlight hydration error.

- **Lỗi 2: `window`/`localStorage` trong render → ReferenceError server**
  - Triệu chứng: SSR crash `window is not defined`, hoặc hydration mismatch.
  - Fix: Guard bằng `useEffect` (chỉ client) hoặc `dynamic(() => import('./Map'), { ssr: false })` (Next.js).
  - Đo: Build log SSR, console `window is not defined`.

- **Lỗi 3: Quên `Suspense` boundary → không selective, phải đợi hết**
  - Triệu chứng: Streaming SSR không streaming, phải đợi Comments xong mới hiện header.
  - Fix: Bọc mỗi `async` Server Component trong `<Suspense fallback={<Skeleton />}>` hoặc `loading.tsx`.
  - Đo: Chrome DevTools → Network → Document → xem HTML stream chunk (shell trước, comments sau). Không có Suspense thì chỉ một chunk.

- **Lỗi 4: Fallback skeleton khác kích thước content → CLS**
  - Triệu chứng: Content hiện làm layout nhảy, CLS cao, Lighthouse cảnh báo.
  - Fix: Skeleton có `height`/`min-height` khớp content, hoặc `aspect-ratio`.
  - Đo: Lighthouse → CLS, Chrome Performance → Layout Shift.

- **Lỗi 5: `useLayoutEffect` warning trên server**
  - Triệu chứng: `Warning: useLayoutEffect does nothing on the server...`
  - Fix: `useLayoutEffect` chỉ chạy client. Nếu cần, dùng `useEffect` hoặc `useInsertionEffect` cho CSS-in-JS. Hoặc guard `typeof window`.
  - Đo: Console warning, không ảnh hưởng hydration nhưng noisy.

- **Lỗi 6: Stream treo không abort**
  - Triệu chứng: Request treo, TTFB không kết thúc.
  - Fix: `setTimeout(() => abort(), 10000)` trong `renderToPipeableStream`, và `onShellError`.
  - Đo: Network → Document pending, server log.

- **Công cụ**:
  - **React DevTools + `onRecoverableError`**: Log `componentStack` của mismatch.
  - **Chrome DevTools → Network → Response streaming**: Xem chunk HTML.
  - **Lighthouse / web-vitals**: Đo TTFB, FCP, CLS trước/sau streaming. `renderToPipeableStream` thường giảm TTFB 20-40%.
  - **`curl`**: `curl -N http://localhost:3000` để xem stream chunk.

## 7. Câu hỏi tự kiểm tra

1. `hydrateRoot` khác `createRoot` thế nào, mismatch xảy ra khi nào (ví dụ `Date`/`Math.random`/`window`), và `suppressHydrationWarning` vs `useEffect` hoãn giá trị khác gì?
2. Selective Hydration hoạt động thế nào với Streaming SSR + Suspense, và vì sao click vào fallback được ưu tiên hydrate?
3. `renderToString` vs `renderToPipeableStream` vs `renderToReadableStream` khác gì về TTFB, Suspense, và trong Next.js App Router bạn cần tự gọi chúng không?

<details>
<summary>Đáp án 30s</summary>

1. **`hydrateRoot`** reuse HTML đã SSR và attach listeners, còn **`createRoot`** tạo DOM mới. **Mismatch** khi HTML lần đầu client khác server do `Date.now()`/`Math.random()`/`window.innerWidth` trong render → React warn và discard DOM server, client-render lại (mất lợi SSR). **`useEffect` hoãn** là fix đúng: server render `null`/placeholder, client sau hydrate mới set giá trị thật (ví dụ `useEffect(() => setTime(Date.now()))`). **`suppressHydrationWarning`** chỉ tắt cảnh báo cho text nhỏ cố ý khác, không fix discard nếu khác lớn.

2. **Streaming SSR** gửi shell + fallback ngay, khi data của một `<Suspense>` xong thì stream thêm HTML + script cho boundary đó. **Selective Hydration** cho phép boundary nào đã stream xong thì hydrate trước, không đợi cả page. Nếu user **click vào fallback** (chưa hydrate), React ưu tiên hydrate boundary đó ngay dù nó chưa phải next theo thứ tự stream, để tương tác không block.

3. **`renderToString`** đồng bộ: đợi hết data mới tạo HTML string gửi 1 lần → TTFB chậm, không hỗ trợ Suspense streaming. **`renderToPipeableStream`** (Node) streaming: gửi shell ngay qua `pipe`, stream thêm khi Suspense resolve (`onShellReady`), TTFB nhanh 20-40%. **`renderToReadableStream`** (Edge) tương tự nhưng trả `ReadableStream` cho `Response`. Trong **Next.js App Router không cần tự gọi** — Next tự streaming; chỉ cần bọc `async` component trong `<Suspense>` hoặc dùng `loading.tsx`.

</details>

---
*Tham khảo chi tiết: `docs/03-react-nang-cao.md` — Câu 59, 61 + `docs/11-nextjs.md` — Câu 182. Spec: [React Docs — hydrateRoot](https://react.dev/reference/react-dom/client/hydrateRoot), [renderToPipeableStream](https://react.dev/reference/react-dom/server/renderToPipeableStream), [Next.js — Loading UI and Streaming](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming).*
