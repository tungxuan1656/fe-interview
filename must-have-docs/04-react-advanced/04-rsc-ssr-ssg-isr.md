# RSC, SSR, CSR, SSG, ISR — RSC payload không JS, "use client" boundary, trade-offs

> Tags: #react #rsc #ssr #csr #ssg #isr #nextjs #use-client | Nguồn: `docs/03-react-nang-cao.md` câu 58, 60 + `docs/11-nextjs.md` câu 173-176 | Mức: P0

## 1. Định nghĩa chính xác

- **React Server Components (RSC)** là component **chỉ chạy trên server**, không gửi JS lên client, không có `useState`/`useEffect`/`onClick`/`window`, được phép `async/await` trực tiếp, fetch DB/API, đọc `cookies`/`headers`, và giữ secret (API key không leak). Render ra **RSC payload** (format streaming serializable, không phải HTML), client merge vào tree.
- **Client Components** (`"use client"` directive ở đầu file) là component truyền thống, chạy **cả server (SSR) và client (hydrate)**, có JS bundle, có state/effect/event/browser API. Mọi import của Client Component cũng thành Client.
- **SSR (Server-Side Rendering)**: HTML tạo **mỗi request** trên server, trả đầy đủ HTML cho bot và FCP nhanh, nhưng TTFB cao, tốn server.
- **CSR (Client-Side Rendering)**: HTML shell rỗng, JS tải xong mới `fetch` và render trên browser. TTFB nhanh (shell), nhưng SEO kém, white flash, JS lớn.
- **SSG (Static Site Generation)**: HTML tạo **lúc build** (`next build`), deploy CDN, mọi request nhận cùng HTML. Nhanh nhất, rẻ, nhưng không real-time.
- **ISR (Incremental Static Regeneration)**: Lai SSG + SSR: HTML tạo lúc build, **revalidate sau N giây** hoặc on-demand (`revalidateTag`/`revalidatePath`), stale-while-revalidate. Vừa CDN vừa tươi.
- **"use client" boundary** là ranh giới biên dịch: file có `"use client"` và mọi import của nó thành Client; RSC có thể import Client, nhưng Client không thể import RSC (trừ `children` slot).

## 2. Cơ chế hoạt động

### 2.1 RSC — không JS, payload streaming

```
Server:  RSC (async) → fetch DB → render → RSC payload (JSON-like: { type, props, ... })
Client:  RSC payload được streaming, merge vào React tree, chỉ Client Components hydrate có JS
```

- RSC **mặc định** trong Next.js App Router (`app/`). Không cần `"use server"` cho RSC (chỉ cho Server Actions). RSC có thể `async`:
  ```tsx
  async function ProductPage({ params }) {
    const product = await db.product.findUnique({ where: { id: params.id } }); // trực tiếp DB, không fetch
    return <div><h1>{product.name}</h1><AddToCart id={product.id} /></div>;
  }
  ```
- **Props từ RSC sang Client phải serializable** (JSON). Không truyền `function` không serializable, `Date` phải stringify, không truyền class instance.
- **Bundle = 0 JS** cho RSC. Chỉ Client Components mới đóng góp JS.

### 2.2 "use client" — đẩy xuống lá

```
app/layout.tsx (Server)
 ├─ app/products/page.tsx (Server) → fetch DB, 0 JS
 │    └─ components/AddToCart.tsx (Client — "use client", có useState/onClick)
 └─ components/Header.tsx (Server nếu chỉ hiện nav, Client nếu có dropdown state)
```

- Sai: đặt `"use client"` ở `page.tsx` làm cả page thành Client → mất lợi RSC, fetch phải ở client, waterfall.
- Đúng: Server fetch, truyền data xuống Client lá qua props serializable.
- Client import RSC sai (vì RSC không có JS để chạy trên client). Đúng là **truyền RSC như children**:
  ```tsx
  // ✅ RSC bọc Client qua children — children vẫn là Server
  // ClientWrapper.tsx ("use client")
  export default function ClientWrapper({ children }: { children: React.ReactNode }) {
    const [open, setOpen] = useState(false);
    return <div onClick={() => setOpen(!open)}>{children}</div>;
  }
  // page.tsx (Server)
  // <ClientWrapper><ServerChild /></ClientWrapper> // ServerChild không bị lây thành Client
  ```

### 2.3 SSR vs CSR vs SSG vs ISR — thời điểm render

```
CSR:  Browser → JS bundle → fetch → render (sau)
SSR:  Server (mỗi request) → HTML đầy đủ → client hydrate
SSG:  Build (1 lần) → HTML/CDN → mọi request nhận cùng HTML
ISR:  Build → HTML → request sau N giây → background revalidate → CDN update (stale-while-revalidate)
```

- **Next.js App Router quyết định static vs dynamic tự động**:
  - Route **không có dynamic API** (`cookies()`, `headers()`, `searchParams`, `fetch cache:no-store`) → **static** (SSG).
  - Có dynamic API → **dynamic** (SSR).
  - Điều khiển thủ công: `export const dynamic = 'force-static' | 'force-dynamic'`, `export const revalidate = 60`, `fetch(..., { cache: 'force-cache' | 'no-store' | next: { revalidate: 60, tags: ['posts'] } })`.

### 2.4 RSC khác SSR

| | RSC | SSR |
|---|---|---|
| Chạy ở đâu | Chỉ server, không hydrate | Server render HTML + client hydrate |
| Gửi JS? | Không (0 JS) | Có (hydrate JS) |
| Có state/effect? | Không | Có (vì hydrate) |
| Output | RSC payload (stream) | HTML string |

RSC và SSR bổ sung nhau: RSC giảm JS, SSR đảm bảo HTML cho SEO/hydration.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 RSC (Server) — mặc định, async, không JS
// app/products/[id]/page.tsx — Server Component, không có "use client"
import AddToCart from '@/components/AddToCart'; // Client lá

export default async function ProductPage({ params }: { params: { id: string } }) {
  // ✅ Được async, fetch trực tiếp DB/API trên server, không leak token
  const product = await fetch(`https://api.example.com/products/${params.id}`, {
    cache: 'force-cache', // SSG
  }).then(r => r.json());

  // Có thể đọc cookies/headers (sẽ thành dynamic)
  // import { cookies } from 'next/headers';
  // const token = cookies().get('token')?.value;

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      {/* Client Component — interactivity */}
      <AddToCart productId={product.id} />
    </div>
  );
}

// 3.2 Client Component — phải "use client", có state/event
// components/AddToCart.tsx
'use client';
import { useState } from 'react';

export default function AddToCart({ productId }: { productId: string }) {
  const [pending, setPending] = useState(false);
  return (
    <button
      disabled={pending}
      onClick={async () => {
        setPending(true);
        await fetch('/api/cart', { method: 'POST', body: JSON.stringify({ productId }) });
        setPending(false);
      }}
    >
      {pending ? 'Adding…' : 'Add to cart'}
    </button>
  );
}

// 3.3 Sai — dùng useState trong RSC → lỗi
// app/bad/page.tsx
// export default function Bad() {
//   const [count, setCount] = useState(0); // ❌ RSC không có useState → build error
//   return <div>{count}</div>;
// }

// 3.4 CSR thuần — "use client" + useEffect fetch (khi cần browser-only)
'use client';
import { useEffect, useState } from 'react';

function CSRPage() {
  const [data, setData] = useState<any>(null);
  useEffect(() => {
    fetch('/api/me').then(r => r.json()).then(setData);
  }, []);
  if (!data) return <div>Loading…</div>;
  return <div>{data.name}</div>;
}

// 3.5 Điều khiển SSG/SSR/ISR trong App Router
// SSR — mỗi request (no-store hoặc cookies)
export const dynamic = 'force-dynamic';
export default async function SSRPage() {
  const data = await fetch('https://api.example.com/me', { cache: 'no-store' }).then(r => r.json());
  return <div>{data.name}</div>;
}

// SSG — build 1 lần (force-cache, không có dynamic API)
export const dynamic = 'force-static';
export default async function SSGPage() {
  const data = await fetch('https://api.example.com/posts', { cache: 'force-cache' }).then(r => r.json());
  return <div>{data.length} posts</div>;
}

// ISR — revalidate 60s (stale-while-revalidate)
export const revalidate = 60;
export default async function ISRPage({ params }: { params: { id: string } }) {
  const data = await fetch(`https://api.example.com/posts/${params.id}`, {
    next: { revalidate: 60 },
  }).then(r => r.json());
  return <div>{data.title}</div>;
}

// ISR với on-demand tag
export default async function TaggedPage() {
  const data = await fetch('https://api.example.com/products', {
    next: { tags: ['products'] },
  }).then(r => r.json());
  return <div>{data.length}</div>;
}
// app/api/revalidate/route.ts
// import { revalidateTag } from 'next/cache';
// export async function POST() { revalidateTag('products'); return Response.json({ ok: true }); }

// 3.6 Pattern giữ Server bên trong Client qua children (không lây)
// components/ClientWrapper.tsx — "use client"
'use client';
import { useState } from 'react';
export default function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return (
    <div>
      <button onClick={() => setOpen(!open)}>Toggle</button>
      {open && children} {/* children là Server, không bị chuyển thành Client */}
    </div>
  );
}
// app/page.tsx (Server)
// import ClientWrapper from '@/components/ClientWrapper';
// import ServerChild from '@/components/ServerChild'; // Server
// export default function Page() {
//   return <ClientWrapper><ServerChild /></ClientWrapper>;
// }
```

## 4. So sánh / Phân loại

| Tiêu chí | RSC (Server) | Client Component |
|----------|--------------|------------------|
| Directive | Không có (mặc định) | `"use client"` đầu file |
| Chạy ở đâu | Chỉ server | Server (SSR) + client (hydrate) |
| JS bundle | 0 — không gửi JS | Có — phải tải + hydrate |
| `useState`/`useEffect`/`onClick`/`window` | ❌ Không | ✅ Có |
| `async/await` fetch trực tiếp | ✅ Có | ❌ Không (dùng SWR/Query) |
| Bảo mật | Token/DB an toàn, không leak | Leak nếu để lộ |
| Props từ RSC | — | Phải serializable (JSON) |
| Khi dùng | Data fetch, static UI, secret | Interactivity, browser API, lib client-only |

| Kiểu | Render ở đâu | Khi nào render | Ưu điểm | Nhược điểm | Dùng cho |
|------|--------------|----------------|---------|------------|----------|
| **CSR** | Browser | Mỗi request (JS fetch) | Đơn giản, TTFB shell nhanh | SEO kém, white flash, JS lớn | Dashboard private, app internal |
| **SSR** | Server | Mỗi request | SEO tốt, FCP nhanh, luôn tươi | TTFB chậm, server load cao | Data cá nhân hóa, auth, realtime (giá CK) |
| **SSG** | Build | 1 lần khi build | Nhanh nhất (CDN), rẻ, SEO tốt | Không real-time, build lâu nếu nhiều page | Blog, marketing, landing |
| **ISR** | Build + background | Build + revalidate N giây / on-demand | Kết hợp SSG + tươi, stale-while-revalidate | Window stale (thấy data cũ N giây), phức tạp invalidation | Product detail, news, listing |

| App Router API | Kết quả |
|----------------|---------|
| `fetch(..., { cache: 'force-cache' })` + không có `cookies()` | SSG (static) |
| `fetch(..., { cache: 'no-store' })` hoặc `cookies()`/`headers()`/`searchParams` | SSR (dynamic) |
| `fetch(..., { next: { revalidate: 60 } })` hoặc `export const revalidate = 60` | ISR (60s) |
| `export const dynamic = 'force-static'` | Ép SSG (lỗi nếu dùng dynamic API) |
| `export const dynamic = 'force-dynamic'` | Ép SSR |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không lạm dụng `"use client"`**: Đặt ở `page.tsx` làm cả page thành Client → bundle phình 30-50%, mất lợi RSC, fetch phải ở client (waterfall, chậm hơn Server fetch). Đẩy `"use client"` xuống **lá nhỏ nhất** (button, dropdown, chart). Dùng `eslint` rule `no-unnecessary-use-client` để bắt.
- **Không dùng RSC khi cần interactivity**: Nếu component cần `useState`, `onClick`, `window`, `localStorage`, `chart.js`, `framer-motion` → phải Client. Cố để Server sẽ lỗi build.
- **Không truyền non-serializable từ RSC sang Client**: `function`, `class instance`, `Map`/`Set` không serializable → lỗi. Chỉ truyền JSON (`string`, `number`, `plain object`, `array`).
- **SSR tốn server**: Mỗi request render trên server → CPU/RAM cao, TTFB chậm hơn SSG. Với page không cần tươi (blog), SSG/ISR rẻ hơn.
- **SSG không tươi**: Data đổi phải build lại. Với hàng nghìn product, build lâu. ISR giải quyết nhưng có window stale.
- **ISR stale window**: User có thể thấy data cũ trong 60s. Với giá chứng khoán, không chấp nhận → phải SSR (`no-store`).
- **RSC làm mental model phức tạp**: 2 môi trường, debug khó hơn (log ở server vs client), không phải mọi lib tương thích (lib dùng `window` trong render sẽ lỗi ở RSC). Chỉ dùng khi framework hỗ trợ (Next.js), không áp cho Vite SPA thuần.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Đặt `"use client"` ở page/layout làm cả cây thành Client**
  - Triệu chứng: Bundle lớn, `next build` báo `Client Component` cho cả page, không còn RSC payload, fetch chuyển sang client.
  - Fix: Xóa `"use client"` ở page, chỉ để ở component lá (`AddToCart`, `Dropdown`). Server fetch ở page.
  - Đo: `next build` output → xem `Route (app)` cột `Client` vs `Server`; `webpack-bundle-analyzer` / `next-bundle-analyzer` → JS tăng.

- **Lỗi 2: Client import RSC → build error**
  - Triệu chứng: `You're importing a component that needs server-only ...`
  - Fix: Không `import ServerComponent` trong Client file. Truyền qua `children` hoặc tách.
  - Đo: Build log, `eslint-plugin-next`.

- **Lỗi 3: Truyền non-serializable props RSC → Client**
  - Triệu chứng: `Error: Only plain objects can be passed to Client Components...`
  - Fix: Chỉ truyền JSON. `Date` → `toISOString()`, function → không truyền, chuyển thành Server Action.
  - Đo: Runtime error overlay, console.

- **Lỗi 4: Dùng `cookies()`/`headers()` làm route thành dynamic không ngờ**
  - Triệu chứng: Route tưởng SSG nhưng `next build` báo `ƒ (Dynamic) server-rendered on demand`, không cache, TTFB chậm.
  - Fix: Chỉ dùng `cookies()` ở route cần auth (dashboard). Với SSG, tránh `cookies()` ở layout chung.
  - Đo: `next build` table `○ (Static)` vs `ƒ (Dynamic)`, header `x-nextjs-cache: MISS`.

- **Lỗi 5: ISR stale — user thấy data cũ**
  - Triệu chứng: Product giá mới nhưng user vẫn thấy giá cũ 60s.
  - Fix: Giảm `revalidate` hoặc dùng `revalidateTag` on-demand sau mutation. Với data nhạy, dùng `no-store` (SSR).
  - Đo: `fetch` với `next.tags`, gọi `revalidateTag('products')` sau `createProduct`, kiểm tra `x-nextjs-cache`.

- **Lỗi 6: CSR SEO kém — bot thấy shell rỗng**
  - Triệu chứng: Google Search Console báo `Content not indexed`, Lighthouse SEO thấp.
  - Fix: Chuyển sang RSC/SSR/SSG cho content cần SEO. CSR chỉ cho dashboard private.
  - Đo: View source (`curl`) — CSR chỉ có `<div id="root"></div>`, SSR/RSC có HTML đầy đủ.

- **Công cụ**:
  - **`next build` output**: Phân biệt `○ Static`, `ƒ Dynamic`, `● SSG`.
  - **React DevTools**: Xem component là Server hay Client (icon).
  - **Lighthouse / web-vitals**: So sánh LCP, TTFB giữa CSR vs SSR vs SSG.
  - **`curl` / View Source**: Kiểm tra HTML có content không (SSR/SSG có, CSR rỗng).

## 7. Câu hỏi tự kiểm tra

1. RSC khác Client Components và khác SSR ở đâu: JS bundle, `useState`, `async`, và RSC payload là gì? Vì sao props RSC→Client phải serializable?
2. Vì sao không lạm dụng `"use client"` và pattern nào giữ Server Component bên trong Client Component mà không bị lây?
3. Phân biệt SSR vs CSR vs SSG vs ISR theo thời điểm render, ưu/nhược, và trong App Router điều gì làm route thành static vs dynamic (ví dụ `cache: 'no-store'`, `cookies()`, `revalidate`)?

<details>
<summary>Đáp án 30s</summary>

1. **RSC** chỉ chạy server, **0 JS**, không `useState`/`useEffect`/`window`, được `async` fetch DB trực tiếp, output là **RSC payload** (stream serializable, không phải HTML string, client merge). **Client** có `"use client"`, chạy cả server + client, có JS bundle và hydrate, có state/event. **SSR** render HTML mỗi request + hydrate JS. Props RSC→Client phải **serializable (JSON)** vì truyền qua RSC payload stream từ server sang client, không thể serialize function/class instance.

2. `"use client"` ở `page.tsx` làm **cả page + mọi import** thành Client → bundle phình, mất 0 JS của RSC, fetch phải waterfall ở client. Đẩy `"use client"` xuống **lá nhỏ nhất** (button, chart). Để giữ Server bên trong Client mà không lây: **truyền Server như `children`**: `<ClientWrapper><ServerChild /></ClientWrapper>` — `ClientWrapper` là Client nhưng `children` vẫn là Server vì được tạo ở Server tree trước khi truyền.

3. **CSR**: browser render sau khi JS tải — TTFB shell nhanh nhưng SEO kém, flash. **SSR**: server render mỗi request — SEO tốt, tươi, nhưng TTFB chậm, tốn server. **SSG**: build 1 lần → CDN — nhanh nhất, rẻ, nhưng không tươi. **ISR**: build + revalidate N giây/on-demand (stale-while-revalidate) — vừa CDN vừa tươi nhưng có window stale. Trong App Router: **static (SSG)** khi không có dynamic API và `fetch` `force-cache`; **dynamic (SSR)** khi có `cookies()`/`headers()`/`searchParams` hoặc `fetch cache:'no-store'`; **ISR** khi `fetch next:{revalidate:60}` hoặc `export const revalidate=60`.

</details>

---
*Tham khảo chi tiết: `docs/03-react-nang-cao.md` — Câu 58, 60 + `docs/11-nextjs.md` — Câu 173-176. Spec: [Next.js Docs — Server and Client Components](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns), [Rendering — Static and Dynamic](https://nextjs.org/docs/app/building-your-application/rendering/server-components).*
