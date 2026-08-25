# Caching Strategy 4 Layers — Request Memoization, Data Cache, Full Route Cache, Router Cache

> Tags: #nextjs #caching #fetch #revalidate #react-cache #rsc | Nguồn: `docs/03-react-nang-cao.md` câu 67 + `docs/11-nextjs.md` câu 179, 186 | Mức: P0

## 1. Định nghĩa chính xác

- **Request Memoization (React `cache`)** : dedupe `fetch` (GET) hoặc `cache()` wrap cho **non-fetch** (ví dụ `db.query`) **trong 1 request** (1 lần render). Cùng URL + options gọi ở 3 component trong cùng request chỉ fetch 1 lần. Không persist giữa requests, tự động, chỉ dedupe GET.
- **Data Cache (fetch cache / Next.js Data Cache)** : **persist** kết quả `fetch` **giữa các requests** trên server (memory hoặc CDN). Mặc định `fetch` trong App Router là `force-cache` (persist). Điều khiển bằng `cache: 'no-store'` (không cache, SSR), `cache: 'force-cache'` (SSG, persist vĩnh viễn), `next: { revalidate: 60 }` (ISR, 60s), `next: { tags: ['posts'] }` (on-demand).
- **Full Route Cache (Static Route Cache)** : cache **HTML + RSC payload** của route **static** đã render (SSG/ISR) trên server/CDN. Khi Data Cache hit và route static, Next trả HTML cache luôn, không render lại.
- **Router Cache (Client)** : Next.js App Router cache **RSC payload trên client** (memory) khi navigate bằng `<Link prefetch>` hoặc `router.push`. Khi back lại không fetch lại. Tồn tại ~30s (staleTime), xóa bằng `router.refresh()`.

Thứ tự lookup: `Router Cache (client) → Full Route Cache (server) → Data Cache (fetch) → Origin (DB/API)`.

## 2. Cơ chế hoạt động

### 2.1 Request Memoization — trong 1 request

```
Request 1: Header → getUser() → fetch('/api/user/1')
           Profile → getUser() → dedupe, không fetch lại
           Sidebar → getUser() → dedupe
Request 2: lại fetch mới (không persist)
```

- Chỉ trong **render** của 1 request. Tự động cho `fetch`, không cần config.
- Cho non-fetch: `import { cache } from 'react'; export const getItem = cache(async (id) => db.item.findUnique(...))` — `getItem(1)` gọi nhiều lần trong request chỉ query DB 1 lần.
- Chỉ GET, không dedupe POST/mutation. Không cache giữa requests.

### 2.2 Data Cache — giữa requests

```js
// SSG — cache vĩnh viễn tới khi revalidate
fetch('https://api/posts', { cache: 'force-cache' });

// SSR — không cache
fetch('https://api/posts', { cache: 'no-store' });

// ISR — cache 60s, stale-while-revalidate
fetch('https://api/posts', { next: { revalidate: 60 } });

// On-demand — gắn tag để revalidate
fetch('https://api/posts', { next: { tags: ['posts'] } });
```

- Lưu trên server, persist qua requests, có thể lưu lên CDN (tùy deploy).
- `next: { revalidate: 0 }` tương đương `no-store`?
- **Mặc định Next.js `fetch` là `force-cache`** (khác browser `no-store`) — nhiều người nhầm và thấy data cũ.

### 2.3 Full Route Cache — HTML/RSC

- Tự động khi route là **static** (không có dynamic API `cookies()`/`headers()`/`searchParams` và Data Cache không `no-store`).
- Khi Data Cache hit và route static → Next trả HTML cache, không chạy lại Server Component.
- Xóa bằng `revalidateTag`/`revalidatePath` hoặc `export const revalidate`.

### 2.4 Router Cache — client

```
<Link href="/products" prefetch> → prefetch RSC payload vào Router Cache
Navigate → hit Router Cache → không fetch Data Cache/Origin
router.refresh() → xóa Router Cache + re-fetch Data
```

- Chỉ trên client, memory, tồn tại ngắn (Next 15: `staleTimes` config).
- `prefetch={false}` tắt prefetch cho Link.

### 2.5 Revalidation — làm mới

| Cách | Xóa gì | Khi dùng |
|------|--------|----------|
| `revalidateTag('posts')` | Data Cache tag `posts` | Sau `createPost`, `updatePost` |
| `revalidatePath('/posts')` | Full Route Cache của `/posts` | Sau mutation ảnh hưởng page |
| `revalidatePath('/', 'layout')` | Layout cache | Khi layout đổi |
| `router.refresh()` | Router Cache (client) | Sau mutation từ Client Component, muốn refetch RSC |
| `export const revalidate = 60` | Cả Data + Full Route theo timer | ISR tự động |

- `revalidateTag` cần `fetch` có `next.tags`.
- `revalidatePath` xóa Full Route Cache, lần request sau sẽ render lại và refetch Data Cache nếu cần.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Request Memoization — tự dedupe trong 1 request
// lib/user.ts
export async function getUser(id: string) {
  // dù gọi ở 3 component trong cùng request, chỉ 1 fetch
  const res = await fetch(`https://api.example.com/user/${id}`);
  return res.json();
}

// app/page.tsx (Server)
import { getUser } from '@/lib/user';
export default async function Page() {
  const [a, b] = await Promise.all([getUser('1'), getUser('1')]); // hit memo, 1 fetch
  return <div>{a.name} {b.name}</div>;
}

// 3.2 React cache cho non-fetch (DB trực tiếp)
import { cache } from 'react';
import { prisma } from '@/lib/prisma';

export const getItem = cache(async (id: string) => {
  console.log('DB query', id); // chỉ log 1 lần dù gọi 3 lần trong request
  return prisma.item.findUnique({ where: { id } });
});

// app/items/[id]/page.tsx
export default async function ItemPage({ params }: { params: { id: string } }) {
  const item1 = await getItem(params.id);
  const item2 = await getItem(params.id); // hit memo
  return <div>{item1.name}</div>;
}
// ⚠️ cache() chỉ cho pure function, không cho function có side effect (increment)

// 3.3 Data Cache — SSG / SSR / ISR / Tag
// SSG
fetch('https://api.example.com/posts', { cache: 'force-cache' }); // build 1 lần, persist

// SSR
fetch('https://api.example.com/cart', { cache: 'no-store' }); // mỗi request

// ISR 60s
fetch('https://api.example.com/posts', { next: { revalidate: 60 } });

// Tag + on-demand
fetch('https://api.example.com/products', { next: { tags: ['products'] } });

// lib/products.ts — gom
export async function getProducts() {
  const res = await fetch('https://api.example.com/products', {
    next: { tags: ['products'], revalidate: 60 },
  });
  if (!res.ok) throw new Error('Failed to fetch products');
  return res.json();
}

// 3.4 Revalidate sau mutation
// app/actions.ts
'use server';
import { revalidateTag, revalidatePath } from 'next/cache';

export async function createProduct(formData: FormData) {
  const name = formData.get('name') as string;
  await fetch('https://api.example.com/products', {
    method: 'POST',
    body: JSON.stringify({ name }),
  });
  // Chọn 1 trong 2:
  revalidateTag('products'); // xóa Data Cache tag 'products'
  // revalidatePath('/products'); // xóa Full Route Cache của /products
}

// app/products/page.tsx — tự refetch sau revalidate
import { getProducts } from '@/lib/products';
export default async function ProductsPage() {
  const products = await getProducts();
  return <ul>{products.map((p: any) => <li key={p.id}>{p.name}</li>)}</ul>;
}

// 3.5 Router Cache — client
// components/RefreshButton.tsx
'use client';
import { useRouter } from 'next/navigation';

export function RefreshButton() {
  const router = useRouter();
  return <button onClick={() => router.refresh()}>Refresh (xóa Router Cache)</button>;
}

// app/products/page.tsx — Link tự prefetch vào Router Cache
import Link from 'next/link';
export default function Nav() {
  return (
    <>
      <Link href="/products" prefetch>Products (prefetch vào Router Cache)</Link>
      <Link href="/dashboard" prefetch={false}>Dashboard (không prefetch)</Link>
    </>
  );
}

// 3.6 Phân biệt 4 layers trong 1 ví dụ tổng hợp
// app/dashboard/page.tsx
import { cookies } from 'next/headers'; // dynamic API → route thành dynamic, không Full Route Cache

export default async function Dashboard() {
  // Request Memoization: getUser('1') gọi 2 lần chỉ fetch 1 lần trong request này
  // Data Cache: fetch no-store → không persist
  // Full Route Cache: không cache vì có cookies()
  // Router Cache: client cache RSC payload khi navigate, router.refresh() để xóa
  const token = cookies().get('token')?.value;
  const user = await fetch('https://api.example.com/me', {
    headers: { Authorization: `Bearer ${token}` },
    cache: 'no-store',
  }).then(r => r.json());
  return <div>{user.name}</div>;
}

// 3.7 Static vs Dynamic ảnh hưởng cache
// app/blog/page.tsx — static, có Full Route Cache
export default async function Blog() {
  const posts = await fetch('https://api.example.com/posts', { cache: 'force-cache' }).then(r => r.json());
  return <div>{posts.length}</div>;
}

// app/realtime/page.tsx — dynamic, không cache
export const dynamic = 'force-dynamic';
export default async function Realtime() {
  return <div>{new Date().toISOString()}</div>;
}
```

## 4. So sánh / Phân loại

| Layer | Ở đâu | Persist giữa requests? | Mặc định | Xóa bằng |
|-------|-------|------------------------|----------|----------|
| **Request Memoization** | Server, 1 request | ❌ Không — hết request là mất | Tự động dedupe `fetch` GET | Không cần — tự hết |
| **Data Cache** | Server | ✅ Có — persist | `force-cache` (Next.js) | `revalidateTag` / `revalidatePath` / `revalidate` timer |
| **Full Route Cache** | Server (HTML/RSC) | ✅ Có — chỉ cho static route | Tự cache nếu route static | `revalidateTag` / `revalidatePath` |
| **Router Cache** | Client (memory) | ❌ Memory, ~30s | Prefetch Link | `router.refresh()` |

| Data Cache config | Hành vi | Dùng cho |
|-------------------|---------|----------|
| `cache: 'force-cache'` | SSG — cache vĩnh viễn tới revalidate | Blog, marketing, ít đổi |
| `cache: 'no-store'` | SSR — không cache, mỗi request fetch | Cart, auth, realtime |
| `next: { revalidate: 60 }` | ISR — cache 60s, stale-while-revalidate | Product listing, news |
| `next: { tags: ['posts'] }` | Gắn tag, đợi `revalidateTag('posts')` | On-demand sau mutation |

| Revalidate | Xóa Data Cache? | Xóa Full Route Cache? | Khi dùng |
|------------|-----------------|-----------------------|----------|
| `revalidateTag('posts')` | ✅ (tag `posts`) | Gián tiếp (route dùng tag sẽ re-render) | Sau `createPost`, `updatePost` |
| `revalidatePath('/posts')` | ❌ (không xóa Data) | ✅ (HTML `/posts`) | Khi layout/page thay đổi, không gắn tag |
| `router.refresh()` | ❌ | ❌ | Xóa Router Cache client, re-fetch RSC |
| `export const revalidate = 60` | ✅ timer | ✅ timer | ISR tự động |

| Khi thấy data cũ | Check layer |
|------------------|-------------|
| Trong cùng request gọi 2 lần | Request Memoization — đã dedupe, không stale |
| Sau deploy vẫn cũ | Data Cache — chưa `revalidateTag` |
| Navigate back thấy cũ | Router Cache — cần `router.refresh()` |
| HTML cũ dù Data mới | Full Route Cache — cần `revalidatePath` |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không mặc định `no-store` cho mọi fetch**: Nhiều cache làm stale khó debug, nhưng `no-store` cho hết sẽ mất lợi CDN, TTFB chậm, server load cao. Dùng `force-cache`/`revalidate` cho data ít đổi, `no-store` chỉ cho data cá nhân hóa.
- **Không dùng `cache()` cho function có side effect**: `cache(async () => { await db.increment() })` sẽ dedupe và không increment lần 2 trong request → bug. `cache` chỉ cho pure read.
- **Không quên `revalidateTag`**: Nếu `fetch` có `tags` nhưng sau mutation không `revalidateTag`, data stale vĩnh viễn. Mỗi `POST`/`PUT` phải kèm revalidate.
- **Không nhầm `revalidateTag` vs `revalidatePath`**: `revalidateTag` xóa Data Cache (cần `tags`), `revalidatePath` xóa Full Route Cache (HTML). Sau `createProduct` ảnh hưởng `/products` và nhiều nơi, `revalidateTag('products')` gọn hơn `revalidatePath('/products')` + `/products/[id]`.
- **Không cache cho route dynamic**: Route có `cookies()`/`headers()` không có Full Route Cache, dù Data Cache có `force-cache` thì HTML vẫn render mỗi request. Đừng mong CDN hit cho dashboard auth.
- **Không lạm dụng Router Cache**: Router Cache memory, không persist qua reload. Đừng dựa vào nó cho data quan trọng; sau mutation phải `router.refresh()` hoặc dùng `revalidate`.
- **Chi phí**: 4 layer làm **stale khó debug**. Khi data cũ, check theo thứ tự: đã `no-store` chưa? đã `revalidateTag` chưa? đã `router.refresh()` chưa? đã check `x-nextjs-cache` header chưa?

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Nghĩ `fetch` không cache — thực ra Next.js mặc định `force-cache`**
  - Triệu chứng: Data đổi nhưng UI vẫn cũ sau deploy, dù không set cache.
  - Fix: Thêm `cache: 'no-store'` cho SSR hoặc `next: { revalidate: 60 }` cho ISR.
  - Đo: `next build` log, header `x-nextjs-cache: HIT` vs `MISS`, `curl -I`.

- **Lỗi 2: Quên `revalidateTag` nên data stale vĩnh viễn**
  - Triệu chứng: Tạo product mới, list vẫn không hiện dù refresh.
  - Fix: Sau `POST`, `revalidateTag('products')` và đảm bảo `fetch` có `tags: ['products']`.
  - Đo: `fetch` log, `revalidateTag` log, `x-nextjs-cache` trước/sau.

- **Lỗi 3: Dùng `revalidatePath` thay `revalidateTag` sai chỗ**
  - Triệu chứng: `revalidatePath('/products')` nhưng Data Cache vẫn cũ (vì chỉ xóa HTML, không xóa Data tag).
  - Fix: Nếu nhiều route dùng chung tag `products`, dùng `revalidateTag`. Nếu chỉ xóa HTML, dùng `revalidatePath`.
  - Đo: So sánh Data Cache hit sau `revalidatePath` — vẫn HIT là sai.

- **Lỗi 4: Dùng `cache()` cho mutation/side effect**
  - Triệu chứng: `incrementView()` gọi 3 lần trong request chỉ tăng 1.
  - Fix: Không `cache` cho function có write. Chỉ `cache` cho read.
  - Đo: Log DB query count.

- **Lỗi 5: Route tưởng static nhưng thành dynamic vì `cookies()` ở layout**
  - Triệu chứng: `next build` báo `ƒ Dynamic` cho cả app, không có Full Route Cache.
  - Fix: Chỉ dùng `cookies()` ở page cần auth, không ở `layout.tsx` chung.
  - Đo: `next build` table `○ Static` vs `ƒ Dynamic`, `x-nextjs-cache`.

- **Lỗi 6: Router Cache làm navigate back thấy data cũ**
  - Triệu chứng: Sau mutation, bấm back vẫn thấy data cũ.
  - Fix: Sau mutation trong Client Component, `router.refresh()` để xóa Router Cache và re-fetch.
  - Đo: DevTools → Network → không thấy fetch khi back là hit Router Cache.

- **Công cụ**:
  - **Header `x-nextjs-cache`**: `HIT`/`MISS`/`STALE`.
  - **`next build` output**: Xem route static/dynamic.
  - **DevTools Network**: Kiểm tra fetch cache, `cache-control`.
  - **`console.log` trong `getItem`**: Đếm số lần `cache()` dedupe.
  - **Lighthouse / `fetch` timing**: So sánh TTFB với `no-store` vs `force-cache`.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt 4 layers: Request Memoization vs Data Cache vs Full Route Cache vs Router Cache — mỗi cái ở đâu, persist không, xóa bằng gì?
2. `fetch(cache:'no-store')` vs `force-cache` vs `next:{revalidate:60}` vs `next:{tags:['posts']}` khác gì, và `revalidateTag` vs `revalidatePath` vs `router.refresh()` xóa layer nào?
3. Khi nào `cache()` của `react` dedupe DB query, và vì sao không dùng `cache` cho function có side effect? Route nào có Full Route Cache?

<details>
<summary>Đáp án 30s</summary>

1. **Request Memoization** (React `cache`/`fetch` dedupe): trong **1 request**, không persist, tự hết khi request xong. **Data Cache**: **persist giữa requests** trên server, lưu `fetch` theo `force-cache`/`revalidate`/`tags`. **Full Route Cache**: cache **HTML/RSC payload** của route static trên server/CDN. **Router Cache**: cache **RSC payload trên client memory** khi `<Link prefetch>`, xóa bằng `router.refresh()`. Thứ tự: Router → Full Route → Data → Origin.

2. **`no-store`** = SSR, không cache mỗi request. **`force-cache`** = SSG, cache vĩnh viễn. **`revalidate:60`** = ISR 60s stale-while-revalidate. **`tags:['posts']`** = gắn tag để `revalidateTag('posts')` xóa on-demand. **`revalidateTag`** xóa **Data Cache** theo tag, **`revalidatePath('/posts')`** xóa **Full Route Cache** HTML của path, **`router.refresh()`** xóa **Router Cache** client và re-fetch RSC.

3. `cache(async id => db.query)` dedupe **trong 1 request**: gọi `getItem(1)` 3 lần chỉ query DB 1 lần. Không dùng cho side effect (increment/write) vì lần 2 bị dedupe không chạy → mất write. **Full Route Cache chỉ cho route static**: không có `cookies()`/`headers()`/`searchParams` và không `no-store`; nếu có dynamic API thì route thành `ƒ Dynamic`, không có Full Route Cache.

</details>

---
*Tham khảo chi tiết: `docs/03-react-nang-cao.md` — Câu 67 + `docs/11-nextjs.md` — Câu 179, 186. Spec: [Next.js Docs — Caching](https://nextjs.org/docs/app/building-your-application/caching), [Data Fetching — Caching and Revalidating](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating), [React — cache](https://react.dev/reference/react/cache).*
