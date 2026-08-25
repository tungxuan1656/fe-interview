# TanStack Query — staleTime/gcTime/cacheTime, queryKey, invalidate/setQueryData, dedup & stale-while-revalidate

> Tags: #TanStack-Query #Server-State #staleTime #gcTime #queryKey #Cache-Invalidation #Dedup #Stale-While-Revalidate | Nguồn: `docs/09-state-management.md` câu 156-158 + `docs/04-frontend-architecture.md` câu 77 | Mức: P0

## 1. Định nghĩa chính xác

**TanStack Query** (React Query) là **server state manager** cho `fetch/cache/sync`: mỗi `useQuery({ queryKey, queryFn })` quản lý 1 **cache entry** (`data`, `status`, `error`, `isPending/isFetching/isStale`, `dataUpdatedAt`) theo `queryKey` phân cấp (mảng). Nó thực hiện **stale-while-revalidate (SWR)**: trả cache (kể cả stale) ngay để render, đồng thời **background refetch** khi stale + có trigger (`mount`, `window focus`, `reconnect`, `polling`). **`staleTime`** (default `0`) = thời gian data **fresh**, trong đó không background refetch khi remount; **`gcTime`** (default `5m`, trước là `cacheTime` v4) = thời gian cache **sống khi không còn subscriber**, sau đó **garbage collect**. **`isPending`** (v5, thay `isLoading` khi chưa có data) vs `isFetching` (đang fetch kể cả có stale data) vs `isLoading` alias.

## 2. Cơ chế hoạt động

- **queryKey — id phân cấp, dedup & invalidation theo prefix**:
  - `['products']` là list, `['products', category]` là filtered list, `['product', id]` là detail, `['search', debounced]` là search.
  - TanStack **dedup**: 2 component cùng `useQuery({ queryKey: ['user'], queryFn: fetchUser })` đang `fetching` thì chỉ **1 fetch**, cache share.
  - Invalidation theo prefix: `qc.invalidateQueries({ queryKey: ['products'] })` đánh dấu stale **mọi entry** có prefix `['products']` (`['products']`, `['products','shoes']`, `['products', { page:2 }]`).

- **staleTime vs gcTime vs cacheTime**:
  - `staleTime: 60_000` — 60s đầu fresh, remount không fetch; sau 60s thành stale → lần mount/focus tiếp background refetch (trả stale ngay + spinner phụ `isFetching`).
  - `gcTime: 5 * 60_000` — nếu không còn component nào subscribe `['products']`, cache giữ 5 phút rồi GC xóa (tiết kiệm memory). Tăng `gcTime` để giữ cache lâu (trở lại trang vẫn có data không fetch).
  - `cacheTime` là tên cũ của `gcTime` (v4 → v5 rename để rõ nghĩa GC). Dùng `gcTime` hiện tại.
  - Tham khảo race/abort chi tiết: `07-architecture/04-data-sync-race.md` — ở đây chỉ tập trung Query API.

- **SWR + background refetch triggers**:
  - Mặc định `refetchOnWindowFocus: true`, `refetchOnReconnect: true`. Khi user quay lại tab, nếu data stale → background fetch mới.
  - `refetchInterval: 30_000` cho polling (orders, ticker).
  - `enabled: !!category` để conditional fetch; `select: data => data.filter(...)` transform không ảnh hưởng cache gốc.

- **Mutation & cache update — `invalidateQueries` vs `setQueryData` vs `refetchQueries`**:
  - `invalidateQueries({ queryKey })` — đánh dấu stale, **không fetch ngay**, sẽ refetch khi có subscriber mount/focus (tiết kiệm nếu không ai xem list).
  - `refetchQueries` — fetch ngay lập tức bất kể.
  - `setQueryData(key, updater)` — ghi cache **đồng bộ, không fetch**, dùng cho **optimistic update** hoặc update detail khi đã có response.
  - `placeholderData` (v5, thay `keepPreviousData`) — giữ data cũ khi `queryKey` đổi (pagination) để không flash loading.

- **Trạng thái v5**:
  - `status: 'pending' | 'error' | 'success'`, `fetchStatus: 'fetching' | 'paused' | 'idle'`, `isPending = status==='pending'`, `isFetching = fetchStatus==='fetching'`.
  - Dùng `isPending` cho skeleton lần đầu, `isFetching` cho spinner phụ khi stale background refetch.

```
mount ['products','shoes'] → miss → fetch → cache { data, dataUpdatedAt, staleTime=60s }
  30s sau remount → fresh → trả cache, KHÔNG fetch
  70s sau remount/focus → stale → trả cache NGAY + background fetch → fresh
  không còn subscriber 5m → GC xóa
mutation POST /products → onSuccess: invalidate ['products'] (list stale) hoặc setQueryData(['product', id], updated)
pagination page 1→2 → queryKey ['products', { page:2 }] + placeholderData: keepPreviousData → không flash
```

## 3. Ví dụ tối thiểu

```tsx
// 3.1 queryKey phân cấp + staleTime/gcTime + dedup + placeholderData
import { useQuery, useQueryClient, useMutation, keepPreviousData } from '@tanstack/react-query';

type Product = { id: string; name: string; price: number };

// List — stale 60s, gc 5m, dedup tự động
export function useProducts(category: string, page: number) {
  return useQuery({
    queryKey: ['products', { category, page }], // object trong key phải stable
    queryFn: ({ signal }) => fetch(`/api/products?category=${category}&page=${page}`, { signal }).then(r => {
      if (!r.ok) throw new Error('Fetch failed');
      return r.json() as Promise<Product[]>;
    }),
    staleTime: 60_000,          // 1 phút fresh
    gcTime: 5 * 60_000,         // 5 phút không dùng thì GC
    placeholderData: keepPreviousData, // pagination: giữ page 1 khi fetch page 2
    refetchOnWindowFocus: true,  // stale mới refetch khi focus
    retry: 2,
  });
}

function ProductList({ category, page }: { category: string; page: number }) {
  const { data, isPending, isFetching, error } = useProducts(category, page);
  if (isPending) return <Skeleton />;           // lần đầu chưa có data
  if (error) return <div>Lỗi: {(error as Error).message}</div>;
  return <>
    {data.map(p => <Card key={p.id} product={p} />)}
    {isFetching && <Spinner />} {/* stale background fetch */}
  </>;
}

// 3.2 Dedup: 2 component cùng key chỉ 1 fetch
function Header() { const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser }); return <Avatar user={data} />; }
function Profile() { const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser }); return <div>{data?.name}</div>; }

// 3.3 Mutation + invalidate vs setQueryData + config Infinity
function AddProduct() {
  const qc = useQueryClient();
  const { mutate, isPending } = useMutation({
    mutationFn: (p: Omit<Product,'id'>) => fetch('/api/products', { method: 'POST', body: JSON.stringify(p) }).then(r=>r.json()),
    onSuccess: (newProduct: Product) => {
      // Option A: invalidate list → sẽ refetch khi có subscriber (đơn giản, tốn request)
      qc.invalidateQueries({ queryKey: ['products'] });
      // Option B: ghi thẳng cache không fetch (nhanh) — chọn 1 trong 2, không cả hai thừa
      // qc.setQueryData<Product[]>(['products', { category: 'all', page: 1 }], old => [...(old ?? []), newProduct]);
    },
  });
  return <button disabled={isPending} onClick={() => mutate({ name: 'New', price: 100 })}>Thêm</button>;
}

function UpdateProduct({ id }: { id: string }) {
  const qc = useQueryClient();
  const { mutate } = useMutation({
    mutationFn: (patch: Partial<Product>) => fetch(`/api/products/${id}`, { method: 'PATCH', body: JSON.stringify(patch) }).then(r=>r.json()),
    onSuccess: (updated: Product) => {
      qc.setQueryData(['product', updated.id], updated); // detail fresh ngay
      qc.invalidateQueries({ queryKey: ['products'] });   // list stale
    },
  });
  return <button onClick={() => mutate({ price: 200 })}>Sửa giá</button>;
}

// 3.4 Config/category không đổi → Infinity; realtime → 0 + polling
useQuery({ queryKey: ['config'], queryFn: fetchConfig, staleTime: Infinity }); // chỉ fetch 1 lần trừ khi invalidate
useQuery({ queryKey: ['ticker'], queryFn: fetchTicker, staleTime: 0, refetchInterval: 5_000 }); // luôn stale + poll 5s

// 3.5 isPending vs isFetching — skeleton vs spinner phụ
function StatusDemo() {
  const { isPending, isFetching, isStale, dataUpdatedAt } = useQuery({ queryKey: ['products','shoes'], queryFn: fetchProducts });
  // isPending: chưa có data lần đầu → skeleton
  // isFetching: đang fetch (kể cả có stale data) → spinner nhỏ
  // isStale: đã hết staleTime
  return <div>{isPending ? 'Skeleton' : isFetching ? 'Refreshing...' : 'Fresh'} — {new Date(dataUpdatedAt).toLocaleTimeString()}</div>;
}
```

## 4. So sánh / Phân loại

| Tiêu chí | `staleTime` | `gcTime` (`cacheTime` cũ) | `isPending` (v5) | `isFetching` |
|----------|-------------|---------------------------|------------------|--------------|
| **Mặc định** | `0` (luôn stale) | `5 phút` | `true` khi `status==='pending'` (chưa có data) | `true` khi đang fetch (kể cả có stale) |
| **Khi hết** | Background refetch khi mount/focus tiếp theo | Xóa cache khỏi memory | — | — |
| **Tăng** | Ít fetch hơn, data có thể cũ | Giữ cache lâu, quay lại trang không fetch | — | — |
| **Giảm/0** | Mỗi mount đều refetch | GC sớm, tiết kiệm memory | — | — |
| **Dùng** | `60s` cho `products`, `Infinity` cho `config` | `5-10m` thường đủ | Skeleton lần đầu | Spinner phụ stale |

| API | Hành vi | Khi dùng |
|-----|---------|----------|
| `invalidateQueries({ queryKey: ['products'] })` | Đánh stale prefix, **không fetch ngay**, refetch khi subscriber mount/focus | Sau `POST /products`, `checkout` → list stale |
| `refetchQueries` | Fetch ngay lập tức | Cần fresh ngay dù không ai mount |
| `setQueryData(['product', id], updater)` | Ghi cache đồng bộ, không fetch | Optimistic, update detail khi đã có response |
| `placeholderData: keepPreviousData` | Giữ data cũ khi key đổi | Pagination `page 1→2` không flash loading |
| `initialData` | Seed cache ban đầu (không fetch nếu có) | Prefetch SSR / `initialData: productsFromList` cho detail |

| `staleTime` preset | Hành vi | Dùng cho |
|--------------------|---------|----------|
| `0` (default) | Luôn stale → mỗi mount background refetch | Realtime (`ticker`, `orders tracking`) + `refetchInterval` |
| `30_000 - 60_000` | 1 phút fresh, cân bằng | `products`, `search` |
| `5 * 60_000` | 5 phút fresh | `user`, `profile` |
| `Infinity` | Không bao giờ stale, chỉ fetch 1 lần trừ khi invalidate | `config`, `categories`, `feature flags` |

| queryKey mẫu | Ý nghĩa | Invalidate nào chạm |
|--------------|---------|---------------------|
| `['products']` | Tất cả products | `invalidate ['products']` |
| `['products', 'shoes']` | Filter theo category | `invalidate ['products']` hoặc `['products','shoes']` |
| `['products', { category, page }]` | Paginated/filtered | Prefix `['products']` chạm hết |
| `['product', id]` | Detail | `invalidate ['product', id]` + `['products']` nếu list cần |
| `['search', debounced]` | Search results | Per query |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không `staleTime: 0` cho list nặng ít đổi**: mỗi mount/focus đều background fetch → thundering herd (20 charts focus cùng lúc → 20 requests, BE 429). Đặt `30-60s` cho `products`.
- **Không `staleTime: Infinity` cho data hay đổi**: `cart`, `orders`, `notifications` sẽ không bao giờ revalidate → user thấy stale. `Infinity` chỉ cho `config`/`categories`.
- **Không `gcTime` quá lớn (30m) nếu memory quan trọng**: giữ cache 1000 `queryKey` pagination → tốn RAM. `5m` là cân bằng; tăng lên `10m` nếu user hay back lại.
- **Không `invalidate` quá rộng**: `invalidate ['products']` sau khi chỉ sửa 1 item làm mọi list refetch. Tối ưu: `setQueryData(['product', id], updated)` cho detail + `invalidate ['products']` cho list (hoặc `setQueryData` list nếu đã có `updated`).
- **Không dùng TanStack Query cho client state**: `isModalOpen` mà `useQuery(['modal'])` là lạm dụng — không stale, thêm ceremony, không có `staleTime` ý nghĩa. Dùng Zustand/`useState`.
- **Không bỏ `signal` trong `queryFn`**: mất Abort → race search. Luôn `queryFn: ({ signal }) => fetch(..., { signal })`.
- **Không nhầm `isPending` vs `isLoading`**: v5 `isPending` = chưa có data; `isLoading = isPending && isFetching`. Dùng `isPending` cho skeleton, `isFetching` cho spinner phụ khi có `placeholderData`.
- **Chi phí**: thêm ~13KB + DevTools, nhưng tiết kiệm ~100 dòng Redux/resource. Đừng trộn Server State vào Zustand — tự implement cache là anti-pattern.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `staleTime: 0` (default) làm focus spam → BE 429**
  - Triệu chứng: quay lại tab, 20 `useQuery(['products'])` cùng background refetch.
  - Fix: `staleTime: 60_000` cho list; `refetchOnWindowFocus: true` chỉ khi stale mới fetch.
  - Đo: Network → số request khi `visibilitychange`; TanStack DevTools → `dataUpdatedAt` vs `staleTime`, badge `fresh` (xanh) / `stale` (vàng).

- **Lỗi 2: `gcTime` quá ngắn / thiếu → quay lại trang fetch lại dù vừa xem**
  - Triệu chứng: vào `/products` → sang `/cart` 2 phút → back → lại fetch dù `staleTime` chưa hết.
  - Fix: `gcTime: 10 * 60_000` để giữ cache khi không subscriber.
  - Đo: DevTools → cache entry biến mất sau `gcTime`; Network → unexpected fetch on back.

- **Lỗi 3: `queryKey` không stable (object literal mỗi render) → fetch vô hạn**
  - Triệu chứng: `queryKey: ['products', { page }]` với `page` object mới mỗi render → key đổi → fetch loop.
  - Fix: `useMemo` hoặc primitive key `['products', category, page]`; hoặc đảm bảo object reference stable.
  - Đo: DevTools → queryKey nhấp nháy; Network → fetch liên tục; `eslint` rule cho deps.

- **Lỗi 4: Nhầm `isPending` vs `isFetching` → skeleton flash khi pagination**
  - Triệu chứng: sang `page 2`, hiện skeleton dù đã có `page 1` data.
  - Fix: `placeholderData: keepPreviousData` + `isPending` cho skeleton lần đầu, `isFetching` cho spinner phụ.
  - Đo: UI flash; DevTools `status: success` nhưng `isFetching: true` khi có placeholder.

- **Lỗi 5: `invalidateQueries` quá rộng hoặc quên invalidate → list cũ**
  - Triệu chứng: `POST /products` xong list không cập nhật, hoặc `PATCH /products/123` làm mọi list refetch nặng.
  - Fix: `setQueryData(['product', id], updated)` cho detail + `invalidate ['products']` cho list; hoặc `setQueryData` luôn nếu có response.
  - Đo: DevTools → `isStale` sau mutation; Network → số refetch; `qc.getQueryData` log.

- **Lỗi 6: Thiếu `signal` → race search (bổ sung cho 07-architecture/04-data-sync-race)**
  - Triệu chứng: gõ `a` → `ab`, kết quả `a` ghi đè `ab`.
  - Fix: `queryFn: ({ signal }) => fetch(..., { signal })`.
  - Đo: Network → 2 requests, timing waterfall, abort `canceled`.

- **Công cụ**:
  - **TanStack Query DevTools** — `queryKey`, `stale`/`fresh`/`fetching`, `dataUpdatedAt`, `gcTime`, dedup indicator.
  - **Network** — `staleTime` qua số request khi focus, `gcTime` qua fetch lại sau khi rời trang.
  - **`performance.now()`** — đo SWR: stale render bao nhanh vs fetch.
  - **Sentry** — log `invalidate`/`setQueryData` mismatch.

## 7. Câu hỏi tự kiểm tra

1. `staleTime` vs `gcTime` (`cacheTime`) khác gì, default bao nhiêu, và `staleTime: 0` vs `60s` vs `Infinity` + `placeholderData: keepPreviousData` dùng khi nào?
2. `queryKey` phân cấp và prefix invalidation hoạt động thế nào — vì sao `['products', { category, page }]` cần stable và dedup là gì?
3. `invalidateQueries` vs `setQueryData` vs `refetchQueries` khác gì, và `isPending` vs `isFetching` (v5) dùng cho skeleton vs spinner phụ thế nào?

<details>
<summary>Đáp án 30s</summary>

1. **`staleTime`** (default `0`) = data **fresh** bao lâu; trong `staleTime` không background refetch khi remount, sau đó thành **stale** → lần mount/focus tiếp **background refetch** (trả stale ngay + fetch mới) — đây là **stale-while-revalidate**. **`gcTime`** (default `5m`, trước `cacheTime`) = cache **sống** bao lâu khi **không còn subscriber**, sau đó **GC xóa**. `0` cho realtime + `refetchInterval: 5s` (ticker), `60s` cho `products` (cân bằng), `Infinity` cho `config/categories` không đổi. **`placeholderData: keepPreviousData`** giữ data cũ khi `queryKey` đổi (pagination `page 1→2` không flash skeleton) — khác `initialData` (seed lần đầu).
2. **`queryKey` mảng phân cấp**: `['products']` prefix chạm mọi `['products', ...]`; `invalidateQueries({ queryKey: ['products'] })` đánh stale hết list/filtered. Object trong key phải **stable** (`useMemo` hoặc primitive `['products', category, page]`) nếu không mỗi render tạo object mới → key đổi → fetch loop. **Dedup**: 2 component cùng `useQuery(['user'])` đang fetching thì chỉ **1 fetch**, cache share theo key.
3. **`invalidateQueries`** → đánh **stale** (không fetch ngay, refetch khi có subscriber mount/focus) — dùng sau `POST /products` cho list. **`setQueryData(key, updater)`** → ghi **đồng bộ không fetch** — dùng optimistic hoặc update detail khi đã có `updated` response. **`refetchQueries`** → **fetch ngay**. **`isPending`** (v5) = `status==='pending'` (chưa có data lần đầu) → **skeleton**; **`isFetching`** = đang fetch kể cả có stale/placeholder → **spinner phụ**; `isLoading = isPending && isFetching` (alias).

</details>

---
*Tham khảo chi tiết: `docs/09-state-management.md` — Câu 156-158, `docs/04-frontend-architecture.md` — Câu 77. Liên quan: `must-have-docs/07-architecture/04-data-sync-race.md` (Abort/SWR/dedup/optimistic chi tiết), `01-state-classification.md`. Spec: [TanStack Query v5 — Important Defaults](https://tanstack.com/query/latest/docs/react/guides/important-defaults), [Query Keys](https://tanstack.com/query/latest/docs/react/guides/query-keys), [Placeholder Data](https://tanstack.com/query/latest/docs/react/guides/paginated-queries).*
