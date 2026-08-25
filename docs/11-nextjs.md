# 11. Next.js - 14 Câu Hỏi Senior

> 14 câu hỏi Next.js cho Senior Frontend (Câu 173-186) - từ SSR/SSG/ISR, App Router, Server/Client Component đến caching, streaming, SEO, auth và tối ưu. Senior không hỏi "Next.js là gì" mà hỏi "khi nào dùng gì và trả giá gì".

## Mục lục

- [Câu 173: SSR vs SSG vs ISR - phân biệt và khi nào dùng?](#câu-173-ssr-vs-ssg-vs-isr---phân-biệt-và-khi-nào-dùng)
- [Câu 174: App Router vs Pages Router - khác gì?](#câu-174-app-router-vs-pages-router---khác-gì)
- [Câu 175: Server Component vs Client Component](#câu-175-server-component-vs-client-component)
- [Câu 176: Khi nào dùng "use client" và vì sao không lạm dụng?](#câu-176-khi-nào-dùng-use-client-và-vì-sao-không-lạm-dụng)
- [Câu 177: Server Actions là gì?](#câu-177-server-actions-là-gì)
- [Câu 178: Middleware - use case thực tế](#câu-178-middleware---use-case-thực-tế)
- [Câu 179: Next.js Caching - 4 layers](#câu-179-nextjs-caching---4-layers)
- [Câu 180: Static vs Dynamic Rendering](#câu-180-static-vs-dynamic-rendering)
- [Câu 181: Route Handlers (app/api) - thay gì cho pages/api?](#câu-181-route-handlers-apparmor---thay-gì-cho-pagesapi)
- [Câu 182: Streaming và Suspense trong Next.js](#câu-182-streaming-và-suspense-trong-nextjs)
- [Câu 183: SEO trong Next.js - Metadata, OG, Sitemap](#câu-183-seo-trong-nextjs---metadata-og-sitemap)
- [Câu 184: Authentication trong Next.js - thiết kế](#câu-184-authentication-trong-nextjs---thiết-kế)
- [Câu 185: Tối ưu Next.js App - Image, Font, Script, Bundle](#câu-185-tối-ưu-nextjs-app---image-font-script-bundle)
- [Câu 186: Data Fetching trong Next.js - fetch, cache, revalidate](#câu-186-data-fetching-trong-nextjs---fetch-cache-revalidate)


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 173: SSR vs SSG vs ISR - phân biệt và khi nào dùng?

**Trả lời Senior:**
Ba chiến lược render của Next.js khác nhau ở **thời điểm HTML được tạo**:

- **SSG (Static Site Generation)**: HTML tạo **lúc build** (`next build`), deploy lên CDN, mọi request nhận cùng HTML. Cực nhanh (CDN), rẻ, SEO tốt nhưng **không tươi** nếu data đổi. Dùng cho marketing page, blog, product listing ít đổi. Trong App Router: mặc định `fetch` cache, route không có dynamic API là SSG.
- **SSR (Server-Side Rendering)**: HTML tạo **mỗi request** trên server. Luôn tươi, SEO tốt nhưng **chậm hơn** (Ttfb cao), tốn server. Dùng cho data cá nhân hóa, auth, realtime. Trong App Router: dùng `fetch({ cache: 'no-store' })` hoặc `cookies()`, `headers()`, `searchParams`.
- **ISR (Incremental Static Regeneration)**: lai SSG + SSR: HTML tạo lúc build, nhưng **revalidate sau N giây** hoặc **on-demand**. Vừa nhanh (CDN) vừa tươi. Dùng cho product detail, blog có update. Trong App Router: `fetch(..., { next: { revalidate: 60 } })` hoặc `export const revalidate = 60`.

```typescript
// App Router - SSG (mặc định)
// app/products/page.tsx
export default async function ProductsPage() {
  const res = await fetch('https://api.example.com/products', { cache: 'force-cache' }); // SSG
  const products = await res.json();
  return <>{products.map(p => <Card key={p.id} />)}</>;
}

// SSR - mỗi request
export default async function Dashboard() {
  const res = await fetch('https://api.example.com/me', { cache: 'no-store' }); // SSR
  // hoặc dùng cookies() tự động dynamic
  const cookieStore = cookies();
  return <div>{cookieStore.get('token')}</div>;
}

// ISR - 60s
export const revalidate = 60; // hoặc trong fetch
export default async function Post({ params }: { params: { id: string } }) {
  const res = await fetch(`https://api.example.com/posts/${params.id}`, { next: { revalidate: 60 } });
  return <Article data={await res.json()} />;
}

// On-demand revalidate
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';
export async function POST(req: Request) {
  revalidateTag('products'); // hoặc revalidatePath('/products')
  return Response.json({ revalidated: true });
}
```

| | SSG | SSR | ISR |
|---|---|---|---|
| Build | HTML sẵn | Không | HTML + revalidate |
| Request | CDN | Server render | CDN (stale-while-revalidate) |
| Data | Cũ tới build | Luôn mới | Mới sau N giây |
| Dùng | Blog, landing | Dashboard, cart | Product, news |

**Trade-off:** SSG rẻ nhưng stale, SSR tươi nhưng đắt, ISR cân bằng nhưng có **window stale** (user có thể thấy data cũ trong 60s). Với giá chứng khoán, phải SSR.

**Câu hỏi đào sâu:** ISR stale-while-revalidate hoạt động thế nào? Khi nào ISR không đủ và phải SSR?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 174: App Router vs Pages Router - khác gì?

**Trả lời Senior:**
Pages Router (`pages/`) là **đời cũ** (Next 12-): file `pages/products/[id].tsx` map route, `getServerSideProps`/`getStaticProps`/`getStaticPaths` để fetch, `_app.tsx` wrap. Mọi component là Client Component.

App Router (`app/` - Next 13+): **đời mới, mặc định**:

- **File-system routing** với **folder + `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `route.ts`**. `layout` lồng nhau, không remount.
- **Server Component mặc định**, Client phải `"use client"`.
- **Data fetching** bằng `async` Server Component + `fetch` có cache, không cần `getServerSideProps`.
- **Streaming + Suspense** native.
- **Metadata API**, **Server Actions**, **Route Handlers**.
- Hỗ trợ **nested layouts, parallel routes (`@slot`), intercepting routes**.

| | Pages Router | App Router |
|---|---|---|
| Route | `pages/*.tsx` | `app/**/page.tsx` |
| Layout | `_app.tsx` + HOC | `layout.tsx` lồng nhau |
| Data | `getServerSideProps` | `async` Server Component |
| Component | Client mặc định | Server mặc định |
| Fetch | `getStaticProps` | `fetch` + `revalidate` |
| Streaming | Không | Có (Suspense) |
| Hạn chế | Linh hoạt với API cũ | Migration lớn, một số lib chưa support Server Component |

```typescript
// Pages Router
// pages/products/[id].tsx
export async function getServerSideProps({ params }) {
  const product = await fetch(`https://api/api/products/${params.id}`).then(r=>r.json());
  return { props: { product } };
}
export default function Product({ product }) { return <div>{product.name}</div>; }

// App Router
// app/products/[id]/page.tsx
export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await fetch(`https://api/products/${params.id}`, { cache: 'no-store' }).then(r=>r.json());
  return <div>{product.name}</div>;
}
// app/products/[id]/layout.tsx
export default function Layout({ children }: { children: React.ReactNode }) {
  return <div><aside>Filter</aside>{children}</div>; // không remount khi đổi product
}
// app/loading.tsx -> fallback Suspense tự động
```

**Trade-off:** App Router mạnh nhưng **breaking change**: nhiều lib (Redux Provider, context) phải `"use client"`, `getServerSideProps` migration thủ công, và Server Component hạn chế (không `useState`). Dự án mới nên App Router, dự án cũ cân nhắc incremental adoption (`app` và `pages` cùng tồn tại).

**Câu hỏi đào sâu:** Vì sao App Router dùng Server Component mặc định? `getServerSideProps` tương đương gì trong App Router?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 175: Server Component vs Client Component

**Trả lời Senior:**
Đây là **phân tách cốt lõi** của React 18 + Next.js App Router:

- **Server Component (RSC)**: render **trên server**, **không có JS gửi xuống client**, không `useState`/`useEffect`/`onClick`, không `window`. Được phép `async`, fetch trực tiếp DB/API, đọc `cookies`, giữ bí mật (API key không leak). Bundle nhỏ, SEO tốt. Mặc định trong `app/`.
- **Client Component**: render **cả server (SSR) + client (hydrate)**, có **JS bundle**, được dùng `useState`, `useEffect`, event, browser API. Phải `"use client"` ở đầu file. Mọi component con của Client Component cũng thành Client (trừ khi lồng Server qua `children`).

Quy tắc: **đẩy Client xuống lá**, giữ Server ở nhánh.

```
app/layout.tsx (Server)
 ├─ app/products/page.tsx (Server) -> fetch DB
 │    └─ components/AddToCart.tsx (Client - "use client" vì có onClick, useState)
 └─ components/Header.tsx (Server hoặc Client tùy cần interactivity)
```

```typescript
// Server Component - mặc định, không cần "use client"
import { cookies } from 'next/headers';
// app/products/page.tsx
export default async function ProductsPage() {
  // ✅ được async, fetch server, không leak token
  const res = await fetch('https://api/products', { headers: { Cookie: cookies().toString() } });
  const products = await res.json();
  return (
    <div>
      {products.map(p => <ProductCard key={p.id} product={p} />)}
      <AddToCart productId={products[0].id} /> {/* Client */}
    </div>
  );
}

// Client Component - phải khai báo
// components/AddToCart.tsx
'use client';
import { useState } from 'react';
export default function AddToCart({ productId }: { productId: string }) {
  const [loading, setLoading] = useState(false);
  return <button onClick={async () => { setLoading(true); await fetch('/api/cart', { method: 'POST', body: JSON.stringify({ productId }) }); setLoading(false); }}>{loading ? '...' : 'Thêm'}</button>;
}

// Sai: dùng useState trong Server Component -> lỗi
```

| | Server | Client |
|---|---|---|
| JS bundle | Không | Có |
| `useState`/`useEffect` | Không | Có |
| `async` fetch | Có | Không (dùng SWR/Query) |
| Bảo mật | Token an toàn | Leak nếu để lộ |
| Dùng cho | Data fetch, static UI | Interactivity, browser API |

**Trade-off:** Server giảm JS 30-50%, nhưng không interactivity. Nếu để cả page là Client, mất lợi ích RSC.

**Câu hỏi đào sâu:** Vì sao Server Component không có JS? Làm sao truyền data từ Server sang Client mà không fetch lại?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 176: Khi nào dùng "use client" và vì sao không lạm dụng?

**Trả lời Senior:**
`"use client"` là **directive biên** (boundary) báo Next.js: "file này và mọi import của nó là Client Component, hãy bundle JS và hydrate". Dùng khi cần **interactivity, state, effect, browser API, event**.

**Khi nào BẮT BUỘC `"use client"`:**

- `useState`, `useReducer`, `useEffect`, `useRef`, `useSyncExternalStore`
- `onClick`, `onChange`, event handler
- `window`, `document`, `localStorage`, `navigator`
- Context Provider, Zustand, Redux Provider
- Thư viện chỉ chạy client: `react-slick`, `chart.js`, `framer-motion`

**Vì sao KHÔNG lạm dụng:**

1.  **Bundle phình**: mỗi Client Component là JS phải tải + hydrate. Để `"use client"` ở `page.tsx` làm cả page thành Client, mất lợi ích Server (0 JS).
2.  **Mất bảo mật**: API key, DB query trong Client sẽ leak.
3.  **Waterfall**: Client fetch sau hydrate chậm hơn Server fetch trước render.
4.  **SEO**: Server render luôn đầy đủ HTML, Client có thể rỗng trước hydrate.

**Chiến lược:** **Đẩy `"use client"` xuống lá nhỏ nhất**, bọc Server Component qua `children`/`props` để giữ Server.

```typescript
// ❌ Lạm dụng - cả page thành Client
// app/products/page.tsx
'use client'; // xấu: cả page JS, fetch ở client
export default function Page() {
  const [products, setProducts] = useState([]);
  useEffect(() => { fetch('/api/products').then(r=>r.json()).then(setProducts); }, []);
  return <>{products.map(p => <Card key={p.id} />)}</>;
}

// ✅ Đúng - Server fetch, Client chỉ nút
// app/products/page.tsx (Server, không có "use client")
export default async function Page() {
  const products = await fetch('https://api/products').then(r=>r.json()); // server
  return (
    <div>
      {products.map(p => (
        <div key={p.id}>
          <h2>{p.name}</h2> {/* Server */}
          <AddToCartButton product={p} /> {/* Client lá */}
        </div>
      ))}
    </div>
  );
}
// components/AddToCartButton.tsx
'use client';
export default function AddToCartButton({ product }: Props) {
  const [added, setAdded] = useState(false);
  return <button onClick={() => setAdded(true)}>{added ? 'Đã thêm' : 'Thêm'}</button>;
}

// ✅ Pattern: Server bọc Client qua children để không lây
// components/ClientWrapper.tsx
'use client';
export default function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return <div onClick={() => setOpen(!open)}>{children}</div>; // children vẫn là Server
}
```

**Trade-off:** Tách nhỏ Client thì nhiều file hơn, nhưng bundle tối ưu. Dùng `eslint-plugin-next` rule `no-unnecessary-use-client` để cảnh báo.

**Câu hỏi đào sâu:** Đặt `"use client"` ở `layout.tsx` ảnh hưởng gì? Làm sao giữ Server Component bên trong Client Component?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 177: Server Actions là gì?

**Trả lời Senior:**
Server Actions là **hàm chạy trên server** được gọi **trực tiếp từ Client Component** (hoặc `<form>`) mà không cần tạo API route thủ công. Là **RPC** của Next.js: client gọi hàm như local, Next serialize args, POST tới server, chạy, trả về.

Lợi thế: **không cần `fetch('/api/...)`**, tự có **revalidate**, **cookies/headers**, **progressive enhancement** (form vẫn chạy khi JS tắt), và **bảo mật** (không expose endpoint).

Dùng cho: mutation (tạo product, login, checkout), form submit, revalidate cache.

```typescript
// app/actions.ts
'use server'; // directive cho file
import { cookies } from 'next/headers';
import { revalidatePath, revalidateTag } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createProduct(formData: FormData) {
  const name = formData.get('name') as string;
  // chạy trên server, có cookies, DB
  const res = await fetch('https://api/products', {
    method: 'POST',
    body: JSON.stringify({ name }),
    headers: { Cookie: cookies().toString() },
  });
  if (!res.ok) throw new Error('Tạo thất bại');
  revalidateTag('products'); // xóa cache
  redirect('/products');
}

export async function addToCart(productId: string) {
  // gọi từ Client Component
  await fetch('https://api/cart', { method: 'POST', body: JSON.stringify({ productId }) });
  revalidatePath('/cart');
  return { success: true };
}

// Dùng trong Client Component
// components/AddForm.tsx
'use client';
import { createProduct } from '@/app/actions';
import { useFormState, useFormStatus } from 'react-dom';

export default function AddForm() {
  const [state, formAction] = useFormState(createProduct, null);
  return (
    <form action={formAction}>
      <input name="name" required />
      <SubmitButton />
      {state?.error && <p>{state.error}</p>}
    </form>
  );
}
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Đang tạo...' : 'Tạo'}</button>;
}

// Hoặc gọi trực tiếp
// components/CartButton.tsx
'use client';
import { addToCart } from '@/app/actions';
export default function CartButton({ id }: { id: string }) {
  return <button onClick={async () => { await addToCart(id); }}>Thêm</button>;
}
```

| | Route Handler (`/api`) | Server Action |
|---|---|---|
| Gọi | `fetch('/api/...')` | `await action()` |
| Form no-JS | Không | Có (progressive) |
| Revalidate | Tự làm | `revalidateTag/Path` |
| Dùng cho | Public API, webhook | Mutation nội bộ |

**Trade-off:** Server Actions tiện nhưng **không thay public API** (mobile app, webhook cần REST). Chưa hỗ trợ upload streaming tốt, và phải handle `try/catch` thủ công. Cần `experimental` ổn định từ Next 14.

**Câu hỏi đào sâu:** Server Action khác Route Handler khi nào? Làm sao handle error/loading với `useFormState`?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 178: Middleware - use case thực tế

**Trả lời Senior:**
Middleware là **code chạy trước khi request tới route**, trên **Edge Runtime** (nhanh, gần user), có thể **rewrite, redirect, modify headers/cookies, check auth, A/B test**. File `middleware.ts` ở root, `matcher` quyết route nào chạy.

Chạy **trước cả Server Component**, nên hợp cho **auth guard, i18n, geo, bot check**. Không dùng cho logic nặng (DB query nặng nên để Server Component).

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(req: NextRequest) {
  const token = req.cookies.get('token')?.value;
  const { pathname } = req.nextUrl;

  // 1. Auth guard
  if (pathname.startsWith('/dashboard') && !token) {
    return NextResponse.redirect(new URL('/login', req.url));
  }
  if (pathname.startsWith('/login') && token) {
    return NextResponse.redirect(new URL('/dashboard', req.url));
  }

  // 2. i18n - detect locale
  const locale = req.headers.get('accept-language')?.split(',')[0] ?? 'vi';
  req.headers.set('x-locale', locale);

  // 3. A/B test - rewrite
  if (pathname === '/products' && Math.random() < 0.5) {
    return NextResponse.rewrite(new URL('/products/v2', req.url));
  }

  // 4. Thêm header bảo mật
  const res = NextResponse.next();
  res.headers.set('x-powered-by', 'Next.js');
  res.headers.set('x-request-id', crypto.randomUUID());
  return res;
}

// Chỉ chạy cho các path này
export const config = {
  matcher: ['/dashboard/:path*', '/login', '/products/:path*'],
  // hoặc '/((?!_next/static|favicon.ico).*)'
};

// Use case khác:
// - Rate limiting (kết hợp Upstash Redis trên Edge)
// - Geolocation: req.geo?.country
// - Bot block: check UA
// - Maintenance mode: redirect all -> /maintenance
```

**Trade-off:** Middleware chạy **mọi request** match, nên phải **nhẹ** (< 1ms), không import lib nặng, không query DB chậm (dùng Edge KV). Nếu logic nặng, chuyển vào Server Component hoặc Route Handler. Edge Runtime không có Node API (`fs`).

**Câu hỏi đào sâu:** Middleware chạy ở đâu (Edge vs Node)? Vì sao không nên query DB nặng trong middleware?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 179: Next.js Caching - 4 layers

**Trả lời Senior:**
Next.js App Router có **4 tầng cache** (nhiều người chỉ biết 1):

1.  **Request Memoization**: trong **1 request**, `fetch` cùng URL + options chỉ gọi 1 lần dù nhiều component gọi (chỉ dedupe GET, không cho POST/mutation). Chỉ trong render, không persist. Tự động, không config.
2.  **Data Cache**: **persist** kết quả `fetch` giữa các request (trên server). Mặc định `fetch` là `force-cache` (SSG). Điều khiển bằng `cache: 'no-store'` hoặc `next: { revalidate: N }` hoặc `revalidateTag`. Lưu trên server, xóa khi revalidate.
3.  **Full Route Cache**: cache **HTML + RSC payload** của route đã render (SSG). Khi Data Cache hit và route là static, Next trả HTML cache luôn, không render lại. Chỉ cho static route.
4.  **Router Cache**: cache **trên client** (browser) - Next giữ RSC payload trong memory khi navigate (`Link` prefetch). `router.refresh()` xóa. Điều khiển bằng `export const dynamic = 'force-dynamic'` hoặc `Link prefetch={false}`.

```
Request -> Router Cache (client) -> Full Route Cache (server) -> Data Cache (fetch) -> Origin
          (Link prefetch)          (HTML)                     (fetch cache)        (DB/API)
```

```typescript
// 1. Request Memoization - tự động dedup trong 1 request
// app/page.tsx
async function getUser() { return fetch('https://api/user').then(r=>r.json()); }
// Header và Profile cùng gọi getUser trong 1 request -> chỉ 1 fetch
export default async function Page() {
  const user1 = await getUser();
  const user2 = await getUser(); // hit memo
}

// 2. Data Cache
fetch('https://api/products', { cache: 'force-cache' }); // SSG, persist
fetch('https://api/cart', { cache: 'no-store' }); // SSR, không cache
fetch('https://api/posts', { next: { revalidate: 60 } }); // ISR
fetch('https://api/products', { next: { tags: ['products'] } }); // để revalidateTag

// Xóa
import { revalidateTag, revalidatePath } from 'next/cache';
revalidateTag('products'); // xóa Data Cache tag products
revalidatePath('/products'); // xóa Full Route Cache

// 3. Full Route Cache - tự động nếu route static
// app/products/page.tsx không có dynamic API -> cache HTML

// 4. Router Cache - client
// <Link href="/products" prefetch> -> prefetch RSC vào Router Cache (30s)
// Sau mutation: router.refresh() để xóa Router Cache
```

| Cache | Ở đâu | Persist | Xóa |
|---|---|---|---|
| Request Memo | Server, 1 request | Không | Hết request |
| Data Cache | Server | Có | `revalidateTag/Path` |
| Full Route Cache | Server | Có | Revalidate |
| Router Cache | Client (memory) | Không (30s) | `router.refresh()` |

**Trade-off:** Nhiều cache làm **stale khó debug**. Khi thấy data cũ, check từng tầng: có `no-store` chưa? Đã `revalidateTag` chưa? `router.refresh()` chưa?

**Câu hỏi đào sâu:** Vì sao `fetch` mặc định cache trong Next.js mà không phải no-store? Làm sao debug cache (xem `x-nextjs-cache` header)?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 180: Static vs Dynamic Rendering

**Trả lời Senior:**
**Static Rendering**: route render **lúc build** (hoặc revalidate), HTML cache, trả CDN. Nhanh, rẻ, SEO tốt. **Dynamic Rendering**: route render **mỗi request** (SSR), không cache.

Next.js tự chọn: nếu route **không có dynamic API** thì static, nếu có thì dynamic. Dynamic API là: `cookies()`, `headers()`, `searchParams` (trong `page.tsx`), `fetch({ cache: 'no-store' })`.

Điều khiển thủ công:

- `export const dynamic = 'force-static'` - ép static (lỗi nếu dùng dynamic API)
- `export const dynamic = 'force-dynamic'` - ép dynamic (SSR)
- `export const revalidate = 60` - ISR
- `export const dynamicParams = false` - chỉ SSG với `generateStaticParams`, không fallback

```typescript
// Static - mặc định
// app/blog/page.tsx - không có cookies() -> static, build 1 lần
export default async function Blog() {
  const posts = await fetch('httpsaran.com/posts', { cache: 'force-cache' }).then(r=>r.json());
  return <>{posts.map(p => <Post key={p.id} />)}</>;
}

// Dynamic - vì dùng cookies
// app/dashboard/page.tsx
import { cookies } from 'next/headers';
export default async function Dashboard() {
  const token = cookies().get('token')?.value; // dynamic API -> tự SSR
  const user = await fetch('https://api/me', { headers: { Authorization: `Bearer ${token}` }, cache: 'no-store' }).then(r=>r.json());
  return <div>{user.name}</div>;
}

// Ép dynamic dù không có API động
export const dynamic = 'force-dynamic';
export default async function RealtimePage() {
  return <div>{new Date().toISOString()}</div>;
}

// Static với params
// app/products/[id]/page.tsx
export async function generateStaticParams() {
  const products = await fetch('https://api/products').then(r=>r.json());
  return products.map(p => ({ id: p.id })); // build trước các id này
}
export const dynamicParams = false; // id ngoài list -> 404, không SSR
export default async function Product({ params }: { params: { id: string } }) {
  const product = await fetch(`https://api/products/${params.id}`).then(r=>r.json());
  return <div>{product.name}</div>;
}
```

**Trade-off:** Static rẻ nhưng không cá nhân hóa. Dynamic tươi nhưng TTFB cao. Dùng **Partial Prerendering (PPR - Next 15)**: shell static + dynamic hole (Suspense) để cân bằng.

**Câu hỏi đào sâu:** Làm sao biết route đang static hay dynamic (xem `next build` output)? PPR giải quyết gì cho static vs dynamic?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 181: Route Handlers (app/api) - thay gì cho pages/api?

**Trả lời Senior:**
Route Handlers (`app/api/**/route.ts`) thay `pages/api` trong App Router. Là **API endpoint** chạy trên Node/Edge, export `GET`, `POST`, `PUT`, `DELETE`, `PATCH` handler, nhận `Request` và trả `Response`.

Khác `pages/api`: không có `req, res` của Node, mà dùng **Web API** (`Request`, `Response`, `NextRequest`, `NextResponse`), hỗ trợ `cookies()`, `headers()`, streaming, và cùng caching với Server Component.

Dùng cho: public API (mobile), webhook, proxy, upload, revalidate.

```typescript
// app/api/products/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { cookies } from 'next/headers';

export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url);
  const category = searchParams.get('category');
  const products = await fetch(`https://api/products?category=${category}`).then(r=>r.json());
  return NextResponse.json(products, { status: 200, headers: { 'Cache-Control': 'public, s-maxage=60' } });
}

export async function POST(req: Request) {
  const body = await req.json();
  const token = cookies().get('token')?.value;
  if (!token) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  const created = await fetch('https://api/products', { method: 'POST', body: JSON.stringify(body) }).then(r=>r.json());
  return NextResponse.json(created, { status: 201 });
}

// app/api/products/[id]/route.ts
export async function GET(_: Request, { params }: { params: { id: string } }) {
  const product = await fetch(`https://api/products/${params.id}`).then(r=>r.json());
  if (!product) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json(product);
}

// app/api/revalidate/route.ts - on-demand ISR
import { revalidateTag } from 'next/cache';
export async function POST(req: Request) {
  const { tag } = await req.json();
  revalidateTag(tag);
  return NextResponse.json({ revalidated: true });
}

// Streaming response
export async function GET() {
  const stream = new ReadableStream({
    start(controller) {
      controller.enqueue(new TextEncoder().encode('chunk1'));
      controller.close();
    },
  });
  return new Response(stream, { headers: { 'Content-Type': 'text/plain' } });
}

// Config runtime
export const runtime = 'edge'; // hoặc 'nodejs'
export const dynamic = 'force-dynamic';
```

| | Pages API | Route Handler |
|---|---|---|
| File | `pages/api/*.ts` | `app/api/**/route.ts` |
| Handler | `export default (req,res)` | `export async function GET()` |
| Request | `NextApiRequest` | `Request`/`NextRequest` |
| Streaming | Hạn chế | Native |

**Trade-off:** Route Handler không thay Server Actions cho mutation nội bộ (Server Actions gọn hơn), nhưng bắt buộc cho **public API** và **webhook** (Stripe, GitHub).

**Câu hỏi đào sâu:** Khi nào dùng Route Handler vs Server Action? Làm sao handle streaming/upload trong Route Handler?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 182: Streaming và Suspense trong Next.js

**Trả lời Senior:**
Streaming cho phép **gửi HTML từng phần** khi sẵn sàng, thay vì đợi cả page xong mới trả. Kết hợp **Suspense** để **render shell trước, data sau**, cải thiện **TTFB** và **perceived performance**.

Trong Next.js:

- `loading.tsx` tự bọc `page.tsx` trong `<Suspense fallback>`.
- `<Suspense>` thủ công cho từng section: header static hiện ngay, product list streaming sau.
- Server Component `async` tự là Suspense boundary.

Lợi thế: user thấy layout/navigation ngay (200ms) thay vì màn trắng 1s đợi DB.

```typescript
// app/products/loading.tsx - tự động fallback cho page
export default function Loading() {
  return <div className="grid"><Skeleton /><Skeleton /><Skeleton /></div>;
}

// app/products/page.tsx - streaming từng phần
import { Suspense } from 'react';
export default function ProductsPage() {
  return (
    <div>
      <h1>Sản phẩm</h1>
      {/* Static hiện ngay */}
      <Filters />
      {/* Dynamic streaming sau */}
      <Suspense fallback={<ProductSkeleton />}>
        <ProductList /> {/* async Server Component */}
      </Suspense>
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews /> {/* fetch chậm hơn, streaming riêng */}
      </Suspense>
    </div>
  );
}
// components/ProductList.tsx
export default async function ProductList() {
  const products = await fetch('https://api/products', { cache: 'no-store' }).then(r=>r.json()); // chậm 800ms
  return <>{products.map(p => <Card key={p.id} product={p} />)}</>;
}

// Với Client Component
'use client';
import { use } from 'react'; // React 19 use()
// Streaming với PPR (Next 15) - shell static + hole dynamic
// Next 15+: cấu hình trong next.config.js: experimental: { ppr: 'incremental' } // không còn export const experimental_ppr
```

```
Timeline không streaming: [==== fetch 800ms ====][render] -> trắng 800ms
Timeline streaming:       [shell 50ms][--- ProductList 800ms ---] -> thấy shell ngay, list dần hiện
```

**Trade-off:** Streaming làm **layout shift** nếu fallback không đúng kích thước, cần skeleton khớp. SEO vẫn tốt vì bot đợi streaming xong (hoặc dùng `loading.tsx` hợp lý). Không streaming cho data critical (auth).

**Câu hỏi đào sâu:** Streaming khác SSR thường thế nào về HTML? `loading.tsx` và `<Suspense>` thủ công khác gì?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 183: SEO trong Next.js - Metadata, OG, Sitemap

**Trả lời Senior:**
Next.js App Router có **Metadata API** file-based và object-based để SEO không cần `next/head` thủ công:

- **Static**: `export const metadata: Metadata = { title, description, openGraph, robots }`
- **Dynamic**: `export async function generateMetadata({ params })` fetch data rồi trả metadata (cho product detail).
- **File-based**: `app/opengraph-image.tsx`, `app/twitter-image.tsx`, `app/sitemap.ts`, `app/robots.ts`, `app/manifest.ts` tự sinh.
- Hỗ trợ **JSON-LD**, **canonical**, **viewport**, **alternates**.

Vì Server Component render HTML đầy đủ, bot crawl không cần JS, SEO vượt SPA.

```typescript
// app/layout.tsx - metadata gốc
import type { Metadata } from 'next';
export const metadata: Metadata = {
  title: { default: 'Shop', template: '%s | Shop' },
  description: 'Mua sắm giá tốt',
  metadataBase: new URL('https://shop.example.com'),
  openGraph: { type: 'website', locale: 'vi_VN', siteName: 'Shop' },
  robots: { index: true, follow: true },
  alternates: { canonical: '/' },
};

// app/products/[id]/page.tsx - dynamic
export async function generateMetadata({ params }: { params: { id: string } }): Promise<Metadata> {
  const product = await fetch(`https://api/products/${params.id}`).then(r=>r.json());
  return {
    title: product.name,
    description: product.description.slice(0, 160),
    openGraph: {
      title: product.name,
      description: product.description,
      images: [{ url: product.image, width: 1200, height: 630, alt: product.name }],
    },
    alternates: { canonical: `/products/${params.id}` },
  };
}

// app/products/[id]/opengraph-image.tsx - OG tự động
export default function Image({ params }: { params: { id: string } }) {
  return new ImageResponse(<div style={{ display: 'flex' }}>{params.id}</div>, { width: 1200, height: 630 });
}

// app/sitemap.ts
export default async function sitemap() {
  const products = await fetch('https://api/products').then(r=>r.json());
  return [
    { url: 'https://shop.example.com', lastModified: new Date() },
    ...products.map(p => ({ url: `https://shop.example.com/products/${p.id}`, lastModified: p.updatedAt })),
  ];
}
// Lưu ý: sitemap limit 50k URL / 50MB theo spec, 100k phải chia sitemap/0.xml, sitemap/1.xml và sitemap index

// app/robots.ts
export default function robots() {
  return { rules: { userAgent: '*', allow: '/', disallow: '/admin' }, sitemap: 'https://shop.example.com/sitemap.xml' };
}

// JSON-LD
// app/products/[id]/page.tsx
export default async function Page({ params }) {
  const product = await fetch(...);
  const jsonLd = { '@context': 'https://schema.org', '@type': 'Product', name: product.name, image: product.image };
  return <><script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }} /><Product product={product} /></>;
}
```

| SEO | Pages | App Router |
|---|---|---|
| Title | `<Head><title>` | `metadata` |
| OG | `<Head><meta property="og:...">` | `opengraph-image.tsx` |
| Sitemap | `next-sitemap` lib | `sitemap.ts` |

**Trade-off:** Metadata API gọn nhưng **file-based OG** cần `next/og` ImageResponse. Với i18n, phải `generateMetadata` theo locale.

**Câu hỏi đào sâu:** `generateMetadata` chạy ở đâu (server)? Làm sao test OG image và sitemap?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 184: Authentication trong Next.js - thiết kế

**Trả lời Senior:**
Auth trong Next.js phải xử 3 nơi: **Middleware (gate), Server Component (read), Client (interact)**. Pattern chuẩn: **httpOnly cookie + JWT/session + Middleware redirect + Server Actions login/logout**.

Luồng:

1.  User POST `/login` (Server Action hoặc Route Handler) → verify → set `httpOnly`, `secure`, `sameSite` cookie `token`.
2.  **Middleware** check `token` cho `/dashboard/*`, redirect nếu thiếu.
3.  **Server Component** đọc `cookies().get('token')` để fetch user, render theo role.
4.  **Client** không giữ token (không `localStorage`), gọi Server Actions để logout.

Thư viện: **NextAuth.js (Auth.js)**, **Clerk**, **Supabase Auth**, hoặc tự JWT.

```typescript
// app/actions/auth.ts
'use server';
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
export async function login(formData: FormData) {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  const res = await fetch('https://api/auth/login', { method: 'POST', body: JSON.stringify({ email, password }) });
  const { token, user } = await res.json();
  if (!res.ok) return { error: 'Sai mật khẩu' };
  cookies().set('token', token, { httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge: 60 * 60 * 24 * 7 });
  redirect('/dashboard');
}
export async function logout() {
  cookies().delete('token');
  redirect('/login');
}

// middleware.ts - gate
export function middleware(req: NextRequest) {
  const token = req.cookies.get('token')?.value;
  const isAuth = !!token;
  const isAuthPage = req.nextUrl.pathname.startsWith('/login');
  if (!isAuth && req.nextUrl.pathname.startsWith('/dashboard')) return NextResponse.redirect(new URL('/login', req.url));
  if (isAuth && isAuthPage) return NextResponse.redirect(new URL('/dashboard', req.url));
  return NextResponse.next();
}

// app/dashboard/page.tsx - Server đọc auth
import { cookies } from 'next/headers';
export default async function Dashboard() {
  const token = cookies().get('token')?.value;
  const user = await fetch('https://api/me', { headers: { Authorization: `Bearer ${token}` }, cache: 'no-store' }).then(r=>r.json());
  return <div>Chào {user.name} - {user.role}</div>;
}

// Với Auth.js (NextAuth)
// app/api/auth/[...nextauth]/route.ts
// import NextAuth from 'next-auth'; export const { handlers, auth } = NextAuth({ providers: [Credentials(...)] });
```

| Lưu token | An toàn | Dùng |
|---|---|---|
| `httpOnly` cookie | Cao (không XSS) | Khuyến nghị |
| `localStorage` | Thấp (XSS leak) | Tránh |
| `Authorization` header | Cần client JS | Chỉ cho API mobile |

**Trade-off:** `httpOnly` an toàn nhưng không đọc được từ JS, phải qua Server. Middleware không verify JWT sâu (chỉ check tồn tại), verify thật ở Server Component/API. Với role-based, thêm `x-user-role` header từ middleware.

**Câu hỏi đào sâu:** Vì sao httpOnly cookie an toàn hơn localStorage? Middleware có nên verify JWT (jose) hay chỉ check tồn tại?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 185: Tối ưu Next.js App - Image, Font, Script, Bundle

**Trả lời Senior:**
Next.js cho sẵn 4 công cụ tối ưu, Senior phải dùng đúng:

1.  **`next/image`**: tự **resize, compress, lazy, priority, responsive** (`sizes`), hỗ trợ `fill`, `placeholder="blur"`. Thay `<img>` giảm 50-70% bytes. Config `remotePatterns` cho CDN. Dùng `priority` cho LCP image.
2.  **`next/font`**: tự **self-host**, **subset**, **fallback metric** tránh CLS, không cần Google Fonts CDN. `next/font/google` với `display: swap`.
3.  **`next/script`**: điều khiển `strategy` (`beforeInteractive`, `afterInteractive`, `lazyOnload`, `worker` với Partytown). Tránh script chặn render.
4.  **Bundle**: `dynamic(() => import('./Heavy'), { ssr: false, loading })` cho code-splitting, `next/bundle-analyzer`, `optimizePackageImports` trong `next.config.js`, tree-shake, `serverComponentsExternalPackages`.

```typescript
// next.config.js
/** @type {import('next').NextConfig} */
export default {
  images: {
    remotePatterns: [{ hostname: 'cdn.example.com' }],
    formats: ['image/avif', 'image/webp'],
  },
  experimental: {
    optimizePackageImports: ['lucide-react', 'date-fns'], // tree-shake
  },
};

// app/products/page.tsx
import Image from 'next/image';
import { Inter } from 'next/font/google';
import Script from 'next/script';
import dynamic from 'next/dynamic';

const inter = Inter({ subsets: ['latin', 'vietnamese'], display: 'swap', variable: '--font-inter' });
const HeavyChart = dynamic(() => import('@/components/HeavyChart'), { ssr: false, loading: () => <Skeleton /> });

export default function Page() {
  return (
    <div className={inter.variable}>
      <Image
        src="https://cdn.example.com/hero.jpg"
        alt="Hero"
        width={1200}
        height={630}
        priority // LCP
        sizes="(max-width: 768px) 100vw, 50vw"
        placeholder="blur"
        blurDataURL="data:image/webp;base64,..."
      />
      <HeavyChart data={[]} />
      <Script src="https://www.googletagmanager.com/gtag/js?id=GA-XXX" strategy="afterInteractive" />
      <Script id="ga" strategy="afterInteractive">{`window.dataLayer...`}</Script>
    </div>
  );
}

// Tối ưu khác:
// - fetch cache + revalidate để giảm API call
// - routeGroups (app/(shop)) để không ảnh hưởng URL nhưng tách layout
// - instrumentation.ts để monitor
```

| Thay | Bằng | Lợi |
|---|---|---|
| `<img>` | `<Image>` | Lazy, WebP/AVIF, CLS thấp |
| `<link google>` | `next/font` | Không CLS, self-host |
| `<script>` | `<Script>` | Không block, đúng strategy |
| `import Heavy` | `dynamic` | Giảm JS ban đầu |

**Trade-off:** `next/image` cần `width/height` hoặc `fill`, không linh hoạt như `<img>` raw. `dynamic ssr:false` mất SEO cho phần đó. `optimizePackageImports` chỉ cho lib ESM.

**Câu hỏi đào sâu:** `priority` và `loading="eager"` khác gì? `next/font` giảm CLS thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 186: Data Fetching trong Next.js - fetch, cache, revalidate

**Trả lời Senior:**
Trong App Router, data fetching là **Server Component `async` + `fetch` mở rộng** (có `cache`, `next.revalidate`, `next.tags`). Không còn `getServerSideProps`.

- **Mặc định**: `fetch(url)` → `cache: 'force-cache'` → SSG.
- **SSR**: `fetch(url, { cache: 'no-store' })` → mỗi request.
- **ISR**: `fetch(url, { next: { revalidate: 60 } })` → stale-while-revalidate 60s.
- **Tag**: `fetch(url, { next: { tags: ['products'] } })` → xóa bằng `revalidateTag('products')`.
- **Dùng `cookies()`/`headers()`** → tự dynamic, không cache.

Ngoài `fetch`, có thể dùng **ORM trực tiếp** trong Server Component (Prisma, Drizzle) vì chạy server.

```typescript
// lib/api.ts
export async function getProducts() {
  const res = await fetch('https://api/products', {
    next: { tags: ['products'], revalidate: 60 }, // ISR + tag
  });
  if (!res.ok) throw new Error('Failed');
  return res.json();
}
export async function getProduct(id: string) {
  const res = await fetch(`https://api/products/${id}`, { next: { tags: [`product-${id}`] } });
  return res.json();
}

// app/products/page.tsx - Server Component
export default async function ProductsPage() {
  const products = await getProducts(); // fetch cache
  return <>{products.map(p => <Card key={p.id} product={p} />)}</>;
}

// app/products/[id]/page.tsx - parallel fetch
export default async function ProductPage({ params }: { params: { id: string } }) {
  const [product, reviews] = await Promise.all([
    getProduct(params.id),
    fetch(`https://api/reviews?product=${params.id}`, { next: { revalidate: 30 } }).then(r=>r.json()),
  ]);
  return <div><h1>{product.name}</h1><Reviews data={reviews} /></div>;
}

// Với Prisma trực tiếp (không fetch)
import { prisma } from '@/lib/prisma';
export default async function ProductsFromDB() {
  const products = await prisma.product.findMany({ where: { active: true } });
  return <>{products.map(p => <Card key={p.id} />)}</>;
}

// Sau mutation - revalidate
// app/actions.ts
'use server';
import { revalidateTag } from 'next/cache';
export async function createProduct(data: FormData) {
  await prisma.product.create({ data: { name: data.get('name') as string } });
  revalidateTag('products'); // xóa cache để lần sau fetch mới
}

// Client fetch - dùng SWR/TanStack Query, không dùng fetch cache của Next
// components/Cart.tsx
'use client';
import useSWR from 'swr';
export default function Cart() {
  const { data } = useSWR('/api/cart', url => fetch(url).then(r=>r.json()));
}
```

| Cách | Cache | Dùng |
|---|---|---|
| `fetch` + `force-cache` | Data Cache | SSG |
| `fetch` + `no-store` | Không | SSR |
| `fetch` + `revalidate` | ISR | Product |
| Prisma trong Server | Không (tự handle) | Khi có DB trực tiếp |
| SWR/Query trong Client | Client cache | Interactivity |

**Trade-off:** `fetch` cache tiện nhưng **chỉ cho Server**, Client phải dùng SWR/Query. Dùng Prisma trực tiếp nhanh nhưng **mất cache** nếu không tự `revalidate`. Luôn `try/catch` và có `error.tsx` boundary.

**Câu hỏi đào sâu:** Khi nào dùng `fetch` vs Prisma trực tiếp trong Server Component? `revalidateTag` khác `revalidatePath` thế nào?

[↑ Quay lại Mục lục](#mục-lục)
---
