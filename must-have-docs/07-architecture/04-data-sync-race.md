# Data Sync & Race Condition — AbortController, switchMap, stale-while-revalidate, Dedup

> Tags: #Data-Sync #Race-Condition #AbortController #SwitchMap #SWR #Stale-While-Revalidate #Dedup #Optimistic-Update #ETag #Retry-Backoff | Nguồn: `docs/12-system-design.md` Bài 1-3, `docs/04-frontend-architecture.md` câu 77-78, `docs/09-state-management.md` câu 156-158 | Mức: P0

## 1. Định nghĩa chính xác

**Race condition FE** là khi 2 async cùng cạnh tranh ghi state, response về **không theo thứ tự gửi** làm kết quả cũ ghi đè mới (search `a` → `ab` nhưng `a` về sau hiển thị sai). **Data sync** là giữ **server state** (snapshot DB, stale, async) đồng bộ với **client cache** qua **stale-while-revalidate (SWR)**, **dedup**, **abort/switchMap**, **optimistic update + ETag/If-Match**, và **retry/backoff**.

**Stale-while-revalidate**: trả cache (stale) ngay để render, đồng thời **background refetch** để revalidate, cập nhật khi xong.

## 2. Cơ chế hoạt động

### 2.1 Vì sao có race

- User gõ search debounce 300ms: `fetch(q="a")` → 800ms, `fetch(q="ab")` → 200ms. `ab` về trước, `a` về sau ghi đè → UI hiện kết quả `a` cho query `ab`.
- Filter đổi nhanh, tab switch, pagination click liên tục, WebSocket + REST đua nhau (`REST` và `WS` cùng update `messages`).

### 2.2 Giải pháp

1. **AbortController + signal** — mỗi request có `signal`, khi queryKey đổi hoặc unmount thì `abort()` request cũ. Browser hủy fetch, `fetch` reject `AbortError`, không ghi state.
2. **switchMap mental model** — chỉ giữ **latest** subscription, cancel previous. TanStack Query `queryFn: ({ signal })` tự làm: khi `queryKey` đổi, abort previous.
3. **Stale-while-revalidate** — TanStack Query `staleTime` + `gcTime`: trong `staleTime` coi fresh, không fetch; sau `staleTime` → trả cache ngay + background fetch. `refetchOnWindowFocus`/`reconnect` tự revalidate khi quay lại tab.
4. **Dedup** — 2 component cùng `useQuery(['products'])` chỉ 1 fetch, cache share. `queryClient` dedup theo `queryKey` đang fetching.
5. **Optimistic Update + Rollback + ETag** — `onMutate` `setQueryData` ngay, `onError` rollback, `onSettled` invalidate. Kèm `ETag`/`If-Match` để server từ chối stale write (412).
6. **Retry/backoff + Idempotency** — retry chỉ GET, backoff `2^i * 1000ms`, POST cần `Idempotency-Key`.

Sơ đồ:

```
gõ "a" → fetch(a, signalA) ──300ms──> gõ "ab" → abort(signalA) → fetch(ab, signalB) → chỉ ab ghi cache
TanStack Query: queryKey ['search', debounced] đổi → cancel previous → dedup nếu đang fetching cùng key
SWR: mount → cache stale? trả stale ngay + background fetch → fresh
Optimistic: onMutate setQueryData → onError rollback → onSettled invalidate
```

### 2.3 `staleTime` vs `gcTime`

- `staleTime` (default `0`): data tươi bao lâu. Trong `staleTime` không refetch khi mount. `Infinity` = không bao giờ stale (chỉ fetch 1 lần).
- `gcTime` (default `5m`, trước `cacheTime`): cache sống bao lâu khi không còn subscriber. Sau `gcTime` bị garbage collect.

## 3. Ví dụ tối thiểu

```ts
// 3.1 AbortController + switchMap — search race
import { useQuery } from '@tanstack/react-query';

function useSearch(q: string) {
  const debounced = useDebounce(q, 300);
  return useQuery({
    queryKey: ['search', debounced],
    // TanStack tự truyền signal — khi debounced đổi, abort fetch cũ
    queryFn: ({ signal }) =>
      fetch(`/api/search?q=${debounced}`, { signal }).then(r => {
        if (!r.ok) throw new Error('Fetch failed');
        return r.json();
      }),
    enabled: debounced.length >= 2,
    staleTime: 30_000,
  });
}

// Thủ công AbortController
function useProductsManual(category: string) {
  const [data, setData] = React.useState(null);
  React.useEffect(() => {
    const ctrl = new AbortController();
    fetch(`/api/products?category=${category}`, { signal: ctrl.signal })
      .then(r => r.json())
      .then(setData)
      .catch(e => { if (e.name !== 'AbortError') throw e; }); // ignore abort
    return () => ctrl.abort(); // cancel khi category đổi / unmount
  }, [category]);
  return data;
}

// RxJS switchMap tương đương (mental model)
// search$.pipe(debounceTime(300), switchMap(q => fetch(`/api/search?q=${q}`)))
```

```ts
// 3.2 Stale-while-revalidate + dedup
import { useQuery, useQueryClient } from '@tanstack/react-query';

// Fresh 60s, không fetch khi remount trong 60s; sau 60s thì stale → background refetch khi focus
export const useProducts = (category?: string) =>
  useQuery({
    queryKey: ['products', category],
    queryFn: ({ signal }) => fetch(`/api/products?category=${category}`, { signal }).then(r => r.json()),
    staleTime: 60_000,
    gcTime: 5 * 60_000,
    refetchOnWindowFocus: true, // quay lại tab nếu stale thì revalidate
    retry: 2,
  });

// Dedup: 2 component cùng key chỉ 1 fetch
function Header() { const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser }); return <Avatar user={data} />; }
function Profile() { const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser }); return <div>{data.name}</div>; } // không fetch lại

// Manual SWR với fetch cache
// fetch('/api/products', { next: { revalidate: 60 } }) — Next.js Data Cache ISR
```

```ts
// 3.3 Optimistic Update + ETag/If-Match + Rollback
const qc = useQueryClient();

type Product = { id: string; name: string; price: number };

const addMutation = useMutation({
  mutationFn: (item: Product) =>
    fetch('/api/cart', {
      method: 'POST',
      headers: { 'Content-Type': 'json', 'Idempotency-Key': crypto.randomUUID() },
      body: JSON.stringify(item),
    }).then(r => r.json()),
  onMutate: async (item) => {
    await qc.cancelQueries({ queryKey: ['cart'] });
    const prev = qc.getQueryData<Product[]>(['cart']);
    qc.setQueryData(['cart'], (old: Product[] = []) => [...old, item]); // optimistic
    return { prev };
  },
  onError: (_e, _item, ctx) => qc.setQueryData(['cart'], ctx?.prev), // rollback
  onSettled: () => qc.invalidateQueries({ queryKey: ['cart'] }),
});

// ETag cho lost update
async function updateProduct(id: string, patch: Partial<Product>) {
  const { etag } = await fetch(`/api/products/${id}`).then(r => ({ etag: r.headers.get('ETag'), data: r.json() }));
  const res = await fetch(`/api/products/${id}`, {
    method: 'PATCH',
    headers: { 'If-Match': etag!, 'Content-Type': 'application/json' },
    body: JSON.stringify(patch),
  });
  if (res.status === 412) throw new Error('Precondition Failed — stale, refetch');
  return res.json();
}

// 3.4 Retry/backoff — chỉ GET
export const fetchWithRetry = async <T>(fn: () => Promise<T>, retries = 3): Promise<T> => {
  for (let i = 0; i < retries; i++) {
    try { return await fn(); } catch (e: any) {
      if (e?.status >= 400 && e?.status < 500 && e?.status !== 429) throw e;
      if (i === retries - 1) throw e;
      await new Promise(r => setTimeout(r, 2 ** i * 1000));
    }
  }
  throw new Error('unreachable');
};

// 3.5 Chat realtime — WS + REST race, seq để order
// Client gửi tempId, server trả seq, client replace optimistic bằng seq order
const sendMutation = useMutation({
  mutationFn: (msg: { text: string; tempId: string }) =>
    fetch(`/api/conversations/${convId}/messages`, { method: 'POST', body: JSON.stringify(msg) }).then(r => r.json()),
  onMutate: async ({ text, tempId }) => {
    const optimistic = { id: tempId, tempId, text, status: 'sending', seq: -1 };
    qc.setQueryData(['messages', convId], (old: any) => ({ ...old, pages: [[optimistic, ...old.pages[0]], ...old.pages.slice(1)] }));
  },
  onSuccess: (saved, { tempId }) => qc.setQueryData(['messages', convId], (old: any) => replaceByTempId(old, tempId, saved)),
  onError: (_e, { tempId }) => markFailed(tempId),
});
// WS message:new → setQueryData dedup bằng id/seq, không fetch lại
```

## 4. So sánh / Phân loại

| Chiến lược | Giải quyết | Không giải quyết | Khi dùng |
|------------|------------|------------------|----------|
| **AbortController** | Race do request cũ về sau, hủy fetch thừa | Không giải WS vs REST race | Search, filter, pagination đổi nhanh |
| **switchMap** (mental) | Chỉ latest ghi, cancel previous | Không dedup 2 component khác nhau | RxJS hoặc Query `signal` |
| **SWR** (`staleTime`/`gcTime`) | Trả stale ngay, background revalidate, UX nhanh | Không ngăn race | List ít đổi (`products` 60s) |
| **Dedup** (Query) | 2 component cùng key chỉ 1 fetch | Không ngăn stale write | Nhiều nơi cùng `['user']` |
| **Optimistic + Rollback** | UX mượt, không đợi server | Phức tạp, cần ETag để không lost update | Cart, like, chat |
| **ETag/If-Match** | Ngăn lost update, báo 412 | Không ngăn race read | PATCH concurrent |
| **Retry/Backoff** | Transient fail (502, 429) | Không retry 4xx POST | Chỉ GET, kèm Idempotency-Key cho POST |

| staleTime | gcTime | Hành vi |
|-----------|--------|---------|
| `0` (default) | `5m` | Luôn stale → mỗi mount background refetch (nhiều request) |
| `60_000` | `5m` | 1 phút fresh, không fetch khi remount; sau 1 phút stale → refetch khi focus |
| `Infinity` | `5m` | Không bao giờ stale, chỉ fetch 1 lần trừ khi invalidate (config, categories) |

| Race loại | Nguyên nhân | Fix |
|-----------|-------------|-----|
| Search debounce | `a` về sau ghi đè `ab` | Abort + switchMap |
| Filter/pagination click nhanh | Page 1 response về sau ghi đè page 2 | Abort per queryKey |
| WS vs REST | WS `message:new` và REST `GET /messages` cùng update | `seq` order + `setQueryData` dedup |
| Optimistic concurrent | 2 tab cùng PATCH | ETag 412 + refetch |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không abort nếu request là non-cancelable side-effect** (`POST /checkout` đã tạo order): abort chỉ hủy FE fetch, không hủy BE đã xử lý. Phải dùng **Idempotency-Key** để dedup thay vì abort.
- **Không optimistic cho mọi mutation**: optimistic phức tạp (rollback, ETag, dedup). Chỉ dùng khi UX cần mượt (cart, chat, like); form checkout nên **pessimistic** (chờ server confirm).
- **Không `staleTime: Infinity` cho data hay đổi** (cart, orders): sẽ không revalidate, user thấy stale. `Infinity` chỉ cho config/categories.
- **Không `staleTime: 0` cho list nặng**: mỗi mount lại fetch gây thundering herd (20 charts cùng fetch). Đặt `30-60s`.
- **Không retry vô hạn**: `retry: Infinity` làm spam server khi 500. Giới hạn 2-3, backoff, chỉ GET.
- **Không dedup thay cho cache invalidation**: dedup chỉ tránh fetch trùng khi đang fetching, không thay `invalidateQueries` sau mutation.
- **Chi phí**: Abort + SWR + Optimistic thêm code, phải handle `AbortError` (ignore), và `onMutate`/`onError` đúng. Đổi lại giảm 80% bug race và UX nhanh.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Thiếu Abort → race search, kết quả cũ ghi đè mới**
  - Triệu chứng: gõ `ab` nhưng UI hiện kết quả `a`.
  - Fix: `queryFn: ({ signal }) => fetch(..., { signal })` hoặc `AbortController` + `ctrl.abort()` trong cleanup.
  - Đo: Network → 2 request, timing Waterfall, response order; React Query DevTools → `isFetching` cancel.

- **Lỗi 2: `staleTime: 0` làm mỗi tab focus lại fetch → spam**
  - Triệu chứng: quay lại tab 20 charts fetch 1 lúc, BE 429.
  - Fix: `staleTime: 60_000` cho list, `refetchOnWindowFocus: true` chỉ khi stale.
  - Đo: Network → số request khi focus, Query DevTools `dataUpdatedAt` vs `staleTime`.

- **Lỗi 3: Optimistic không rollback → UI sai sau fail**
  - Triệu chứng: add to cart hiện thành công nhưng BE 500, không rollback.
  - Fix: `onMutate` lưu `prev`, `onError` `setQueryData(prev)`.
  - Đo: tắt network, thử mutation, check UI rollback.

- **Lỗi 4: Lost update không ETag → ghi đè lặng lẽ**
  - Triệu chứng: 2 admin sửa cùng product, người sau ghi đè không báo.
  - Fix: `ETag`/`If-Match`, BE 412, FE toast + refetch.
  - Đo: Network headers `ETag`, `If-Match`, test 2 tab cùng PATCH.

- **Lỗi 5: Retry POST → double order**
  - Triệu chứng: 1 click tạo 2 order.
  - Fix: không retry POST, dùng `Idempotency-Key: uuid` header, BE dedup.
  - Đo: Network retry count, BE log duplicate key.

- **Lỗi 6: WS và REST đua → duplicate message**
  - Triệu chứng: message hiện 2 lần (optimistic + WS echo).
  - Fix: dedup bằng `id`/`tempId`/`seq`, `setQueryData` replace thay vì append nếu đã có.
  - Đo: WS frames vs REST response, `console.log` dedup.

- **Công cụ**:
  - **Network** — abort (`canceled`), retry, `ETag`/`If-Match`, `Idempotency-Key`, timing.
  - **React Query DevTools** — stale/fresh, `dataUpdatedAt`, `isFetching`, dedup.
  - **Sentry** — `AbortError` filter (ignore), 412 log.
  - **`performance.now()`** — đo SWR (stale render bao nhanh).

## 7. Câu hỏi tự kiểm tra

1. Race condition FE là gì và vì sao search debounce dễ gặp? AbortController + switchMap khắc phục thế nào và vì sao phải ignore `AbortError`?
2. Phân biệt `staleTime` vs `gcTime` và stale-while-revalidate — khi nào `staleTime: 0` vs `60s` vs `Infinity`? Dedup của TanStack Query hoạt động ra sao?
3. Optimistic update + ETag/If-Match + retry/backoff phối hợp thế nào để vừa UX mượt vừa không lost update/double order? Khi nào không nên optimistic?

<details>
<summary>Đáp án 30s</summary>

1. **Race** là 2 fetch đua ghi state, response về **không theo thứ tự gửi** làm cũ ghi đè mới. Search `a` (800ms) → `ab` (200ms), `ab` về trước nhưng `a` về sau ghi đè → UI hiện `a` cho query `ab`. **AbortController**: mỗi request có `signal`, khi `queryKey` đổi hoặc unmount thì `abort()` request cũ, browser hủy fetch, reject `AbortError` → không ghi state. **switchMap** mental model: chỉ giữ latest subscription, cancel previous (TanStack `queryFn: ({ signal })` tự làm). Phải **ignore `AbortError`** (`e.name === 'AbortError'`) vì abort là hành vi chủ động, không phải lỗi, tránh toast/log ồn.
2. **`staleTime`** (default `0`) là thời gian data **tươi**, trong `staleTime` không refetch khi remount; sau `staleTime` thành **stale** → lần mount/focus tiếp sẽ **background refetch** (trả stale ngay + fetch mới). **`gcTime`** (default `5m`) là thời gian cache sống khi **không còn subscriber**, sau đó **GC xóa**. **SWR**: trả stale ngay → revalidate nền. `0` cho realtime (ticker), `60s` cho list ít đổi (products), `Infinity` cho config/categories không đổi. **Dedup**: 2 component cùng `useQuery(['user'])` đang fetching thì chỉ **1 fetch**, cache share theo `queryKey`.
3. **Optimistic**: `onMutate` `cancelQueries` + `getQueryData` lưu `prev` + `setQueryData` ngay để UI mượt, `onError` rollback `prev`, `onSettled` `invalidate`. **ETag/If-Match**: `GET` lưu `ETag`, `PATCH` gửi `If-Match`, BE so sánh version, trả **412** nếu stale → FE refetch, ngăn **lost update**. **Retry/backoff** `2^i*1000ms` chỉ cho **GET** (3 lần), **không retry POST** không `Idempotency-Key` vì tạo **double order**; POST kèm `Idempotency-Key: uuid` để BE dedup. **Không optimistic** khi cần chính xác (checkout payment) hoặc mutation phức tạp khó rollback — dùng **pessimistic** chờ server.

</details>

---
*Tham khảo chi tiết: `docs/12-system-design.md` — Bài 1 (Cart optimistic), Bài 2 (WS + seq), `docs/04-frontend-architecture.md` — Câu 78, `docs/09-state-management.md` — Câu 158. Spec: [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController), [TanStack Query](https://tanstack.com/query/latest/docs/react/guides/query-cancellation).*

