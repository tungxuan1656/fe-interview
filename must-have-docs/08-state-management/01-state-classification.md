# State Classification — 5 loại Server/Client Global/URL/Form/Ephemeral và quy tắc đặt đúng nơi

> Tags: #State-Classification #Server-State #Client-State #URL-State #Form-State #Ephemeral-State | Nguồn: `docs/09-state-management.md` câu 155, 159, 160 + `docs/04-frontend-architecture.md` câu 75-77 | Mức: P0

## 1. Định nghĩa chính xác

**State classification** là phân loại state theo **ownership** (ai sở hữu), **persistence** (sống bao lâu), **staleness** (có stale không) và **shareability** (chia sẻ qua đâu) để quyết định **lưu ở đâu**: 5 loại — **Server State** (snapshot DB ở server, async, stale, multi-client ownership, cần cache/revalidate), **Client Global State** (UI/business thuần client, sync, single-client ownership, cần qua nhiều route), **URL State** (`searchParams`/`pathname` — shareable qua link, back/forward, bookmark), **Form State** (input value/validation/dirty/touched, high-frequency update), **Ephemeral UI State** (hover/focus/animation, local, lifecycle = component mount).

Định nghĩa này là đáp án cho câu Senior "state này thuộc loại nào và nên ở đâu?" — không trả lời bằng "Redux hay Zustand?" trước khi phân loại.

## 2. Cơ chế hoạt động

- **Server State — TanStack Query / RTK Query / SWR**:
  - Tính chất: `async`, `shared ownership` (nhiều client cùng đọc/ghi), `stale`, cần `cache + dedup + background refetch + invalidate`.
  - Vòng đời: `fetch → cache by queryKey → staleTime fresh → background revalidate on mount/focus/reconnect → gcTime GC khi không còn subscriber`.
  - Không tự quản bằng `useState + useEffect` — thiếu dedup/retry/polling/optimistic. Tham khảo chi tiết sync/race trong `07-architecture/04-data-sync-race.md`.

- **Client Global State — Zustand / Redux Toolkit / Jotai / Context**:
  - Tính chất: `sync`, `local ownership`, `không stale`, cần qua `≥2 route` hoặc `≥3 consumers` không có quan hệ cha-con trực tiếp.
  - Vòng đời: `set → subscribe → selector equality check → re-render consumer`. Zustand không cần Provider; Redux cần `<Provider>` + `useSelector`/`useDispatch`; Jotai atom `read/write` riêng; Context re-render tất cả consumer khi `value` đổi (không có selector).
  - Chỉ cho state thực sự global (`auth user`, `cart draft`, `theme`, `locale`). Không cho `products`/`orders`.

- **URL State — `useSearchParams` / `nuqs` / `react-router`**:
  - Tính chất: `serializable`, `shareable`, `persist qua refresh/back/forward`, là **single source of truth cho filter/sort/page/tab**.
  - Cơ chế: `URL (?category=shoes&page=2) ↔ searchParams.get() ↔ setSearchParams() ↔ history.pushState`. Server SSR đọc URL để prefetch; client đọc URL để `queryKey`.
  - Khác Client Global: URL survive reload, share link, SEO; Zustand không.

- **Form State — `useState` local vs `React Hook Form` (RHF)**:
  - Tính chất: `high-frequency`, `validation`, `dirty/touched/errors`, `không share`.
  - Cơ chế `useState controlled`: mỗi keystroke → `setState` → re-render cả form. RHF `uncontrolled + register + ref`: input tự quản DOM value, chỉ `onSubmit/blur` mới sync, giảm re-render 90%.

- **Ephemeral UI State — `useState` local / `useRef`**:
  - Tính chất: `hover`, `focus`, `isDropdownOpen`, `animation tick` — lifecycle = component, không cần persist/share.
  - Quy tắc **colocation** (Kent C. Dodds): đặt state **càng gần nơi dùng càng tốt**; chỉ lift khi thực sự cần share.

- **Hybrid cần chú ý**:
  - `Cart`: trước login là Client Global (`Zustand + persist localStorage`), sau login `sync POST /api/cart/merge` → Server State (`['cart']`).
  - `Search`: `input` = Form State (local, debounce 300ms), `results` = Server State (`['search', debounced]`).

```
User gõ filter → draft local (Form) → Apply → setSearchParams → URL đổi → queryKey ['products', params] đổi → TanStack Query fetch → Server State cache
Cart add → Zustand set (optimistic local) → sync server on login/checkout → invalidate ['cart','orders']
Modal open → useState local (Ephemeral) — không lên Zustand nếu chỉ 1 component dùng
```

## 3. Ví dụ tối thiểu

```ts
// 3.1 Anti-pattern: lưu Server State vào Redux — tự lo cache/stale/dedup (KHÔNG làm)
import { createSlice, createAsyncThunk, configureStore } from '@reduxjs/toolkit';
export const fetchProducts = createAsyncThunk('products/fetch', () => fetch('/api/products').then(r=>r.json()));
const productsSlice = createSlice({
  name: 'products',
  initialState: { data: [] as Product[], loading: false, error: null as string | null },
  reducers: {},
  extraReducers: b => {
    b.addCase(fetchProducts.pending, s => { s.loading = true; });
    b.addCase(fetchProducts.fulfilled, (s,a) => { s.data = a.payload; s.loading = false; });
    b.addCase(fetchProducts.rejected, (s,a) => { s.error = a.error.message ?? 'error'; s.loading = false; });
  },
});
// Vấn đề: tự cache, tự staleTime, tự background refetch, tự dedup, tự GC — reinvent TanStack Query
```

```tsx
// 3.2 Chuẩn: 5 loại ở đúng nơi — runnable
import { useQuery } from '@tanstack/react-query';
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { useSearchParams } from 'react-router-dom';
import { useForm } from 'react-hook-form';
import { useState } from 'react';

// Server State — TanStack Query, staleTime/gcTime, queryKey phân cấp
function ProductList() {
  const [searchParams] = useSearchParams();
  const category = searchParams.get('category') ?? 'all';
  const { data, isPending } = useQuery({
    queryKey: ['products', category], // đổi category → auto fetch, dedup
    queryFn: ({ signal }) => fetch(`/api/products?category=${category}`, { signal }).then(r=>r.json()),
    staleTime: 60_000, // 1 phút fresh
    gcTime: 5 * 60_000,
  });
  if (isPending) return <Skeleton />;
  return <>{data.map((p: Product) => <Card key={p.id} product={p} />)}</>;
}

// Client Global — Zustand, chỉ state qua nhiều route
type CartState = { items: Item[]; add: (i: Item) => void };
export const useCart = create<CartState>()(persist((set, get) => ({
  items: [],
  add: item => set({ items: [...get().items, item] }),
}), { name: 'cart' }));

function CartBadge() {
  const count = useCart(s => s.items.length); // selector — chỉ re-render khi length đổi
  return <span>{count}</span>;
}

// URL State — filter shareable qua link
function ProductFilter() {
  const [searchParams, setSearchParams] = useSearchParams();
  const brand = searchParams.get('brand') ?? '';
  return <select value={brand} onChange={e => setSearchParams({ brand: e.target.value, page: '1' })} />;
}

// Form State — RHF uncontrolled, không re-render cả form mỗi keystroke
function CheckoutForm() {
  const { register, handleSubmit, formState: { errors, isDirty } } = useForm<{ email: string }>();
  return <form onSubmit={handleSubmit(d => console.log(d))}>
    <input {...register('email', { required: 'Email required' })} />
    {errors.email && <span>{errors.email.message}</span>}
    <button disabled={!isDirty}>Submit</button>
  </form>;
}

// Ephemeral — local useState
function HoverCard({ product }: { product: Product }) {
  const [isHovered, setHovered] = useState(false);
  return <div onMouseEnter={() => setHovered(true)} onMouseLeave={() => setHovered(false)}>{isHovered && <Preview />}</div>;
}

// Hybrid Search: input local + results server
function SearchBox() {
  const [input, setInput] = useState('');
  const debounced = useDebounce(input, 300);
  const { data: results } = useQuery({
    queryKey: ['search', debounced],
    queryFn: ({ signal }) => fetch(`/api/search?q=${debounced}`, { signal }).then(r=>r.json()),
    enabled: debounced.length >= 2,
  });
  return <><input value={input} onChange={e=>setInput(e.target.value)} />{results?.map(r=><div key={r.id}>{r.name}</div>)}</>;
}
```

## 4. So sánh / Phân loại

| State | Loại chính | Lưu ở đâu | Vì sao ở đó | Ví dụ ecommerce |
|-------|------------|-----------|-------------|-----------------|
| **User** | Server + Global | `useQuery(['user'])` + Context cho `isAuthenticated` | Thuộc server, nhiều nơi cần, cần stale/cache | `GET /me`, avatar, guard route |
| **Product** list/detail | **Server** | `useQuery(['products', category])` / `['product', id]` | Server snapshot, stale, dedup, background refetch | Catalog, PDP |
| **Order** | Server | `useQuery(['orders'])`, `refetchInterval: 30s` cho tracking | Thuộc server, invalidate sau checkout | Order history, tracking |
| **Notification** | Server + Global | `useQuery(['notifications'])` + SSE/WS + `setQueryData` | Server push, badge global | Bell badge |
| **Search results** | Server | `useQuery(['search', q], { enabled: q.length>=2 })` | Server, cần dedup/abort | Search dropdown |
| **Cart** | **Hybrid** Global | Zustand `persist` (`localStorage`) + sync `POST /api/cart/merge` + `invalidate ['cart']` | Trước login client, sau login server | Cart drawer, checkout |
| **Filter / Sort / Page / Tab** | **URL State** | `useSearchParams` `?brand=nike&page=2&sort=price` | Share link, back/forward, SSR prefetch | Filter sidebar, pagination |
| **Search input draft** | Form (local) | `useState` + debounce | High-frequency, chưa cần server | Search input |
| **Checkout form** | Form | RHF `useForm` uncontrolled | Validation, dirty, không re-render form | Checkout |
| **UI** modal/drawer/toast | Client Global nếu nhiều nơi, ngược lại local | `useState` local hoặc `Zustand useUI` | Ephemeral nếu 1 component | Drawer, toast |
| **Hover/Focus/Animation** | Ephemeral | `useState` / `useRef` local | Lifecycle = component | Hover preview |

| Tiêu chí | Server State | Client Global | URL State | Form State | Ephemeral |
|----------|--------------|---------------|-----------|------------|-----------|
| **Nguồn** | Server | Client | URL (client) | Client (input) | Client (interaction) |
| **Stale** | Có — cần `staleTime/gcTime` | Không | Không | Không | Không |
| **Cache/Dedup** | Có (Query) | Không | Không (history) | Không | Không |
| **Share** | Nhiều client | Nhiều component/route | Nhiều user (link) | 1 form | 1 component |
| **Persist** | Server DB | Zustand `persist` / memory | URL (bookmark) | Không (submit xong) | Không |
| **Tool** | TanStack Query | Zustand/Redux/Jotai | `useSearchParams`/`nuqs` | RHF | `useState` |

> Anti-pattern **"all in Redux"**: nhét `products`, `orders`, `search results` vào `createSlice + createAsyncThunk` — phải tự `loading/error`, tự `cache`, tự `staleTime`, tự `dedup`, tự `background refetch`, tự `GC`, tự `optimistic + rollback`. Kết quả: 100 dòng/resource, stale sau 0s, thundering herd khi focus. Thay bằng **1 dòng `useQuery`** với `queryKey`.

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không đưa Server State vào Zustand/Redux**: tạo duplicate source of truth (store + server), stale không kiểm soát, phải `useEffect` fetch thủ công. Exception duy nhất: RTK Query — đó là `server cache` chứ không phải client store, chấp nhận được.
- **Không đưa URL State vào Zustand nếu cần share**: `filter` trong Zustand không share link, không back/forward, không SSR prefetch, không bookmark. Ngược lại, `filter draft` (slider đang kéo chưa Apply) để local `useState`, Apply mới đẩy `setSearchParams`.
- **Không đưa Form/Ephemeral lên global**: `searchDraft`, `isModalOpen` chỉ 1 component dùng mà lên Zustand → global pollution, khó test (phải mock store), re-render lan rộng. Dùng colocation.
- **Không dùng `Context` cho frequent update**: Context không có selector — `useContext` re-render tất cả consumer khi `value` đổi. Dùng cho stable (`theme`, `locale`, `auth status`), không cho `cart`/`filter` đổi liên tục → Zustand/Jotai.
- **Không dùng TanStack Query cho client state**: `isModalOpen` mà `useQuery(['modal'])` là lạm dụng — tốn cache, không stale, thêm ceremony.
- **Chi phí phân loại**: 5 loại làm onboard phức tạp, cần doc quy tắc và linter. Đổi lại giảm 50% code và bug stale/cache; không còn "vì sao product cũ sau khi checkout?".

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Lưu `products`/`orders` vào Redux/Zustand → stale, không revalidate**
  - Triệu chứng: user A sửa product, user B vẫn thấy cũ dù đã `dispatch(fetch)`.
  - Fix: chuyển sang `useQuery(['products', category], { staleTime: 60_000 })` + `qc.invalidateQueries(['products'])` sau mutation.
  - Đo: TanStack Query DevTools → check `dataUpdatedAt`, `isStale`, `isFetching`; Network → thiếu background refetch sau `focus`.

- **Lỗi 2: Filter lưu ở Zustand thay vì URL → mất share/back**
  - Triệu chứng: gửi link `/products` cho đồng nghiệp, họ không thấy filter `brand=nike` bạn đang xem; back button không khôi phục filter.
  - Fix: `useSearchParams` cho filter/sort/page; `setSearchParams({ brand })`; `useQuery` key đọc từ URL.
  - Đo: manual — copy URL sang tab mới, check filter; History back/forward.

- **Lỗi 3: Form controlled `useState` cho mỗi input → gõ lag**
  - Triệu chứng: form 20 fields, gõ 1 ký tự re-render cả form, INP > 300ms.
  - Fix: chuyển sang RHF `register` (uncontrolled) + `resolver: zodResolver(schema)`, chỉ field lỗi mới re-render.
  - Đo: React DevTools Profiler → flamegraph, component render count mỗi keystroke; `why-did-you-render` (WDYR) → log re-render không cần.

- **Lỗi 4: Context cho `cart` → tất cả consumer re-render khi 1 item đổi**
  - Triệu chứng: thêm 1 item, 50 `ProductCard` re-render dù chỉ cần `CartBadge`.
  - Fix: tách `CartItemsContext`/`CartActionsContext` hoặc chuyển Zustand `useCart(s => s.items.length)` + `useCart(s => s.add)` + `shallow`.
  - Đo: Profiler → highlight updates; WDYR `trackAllPureComponents: true`; TanStack DevTools không liên quan — đây là client re-render.

- **Lỗi 5: Ephemeral state lift thừa lên global → khó xóa, test phức tạp**
  - Triệu chứng: `isHovered` trong Zustand, xóa component vẫn còn state rác, test phải `resetStore()`.
  - Fix: colocation — `const [isHovered, setHovered] = useState(false)` trong chính component.
  - Đo: code review + grep `useCart`/`useUI` usage count; nếu chỉ 1 nơi dùng → local.

- **Lỗi 6: Hybrid `cart` không sync → mất cart sau login**
  - Triệu chứng: guest add 3 items, login xong cart rỗng.
  - Fix: `persist` + `onLogin: await fetch('/api/cart/merge', { method: 'POST', body: JSON.stringify(get().items) }); qc.invalidateQueries(['cart'])`.
  - Đo: Application → LocalStorage `cart-storage`; Network `POST /merge`; Query DevTools `['cart']`.

- **Công cụ**:
  - **TanStack Query DevTools** — `queryKey`, `stale`/`fresh`, `dataUpdatedAt`, `isFetching`, dedup, cache entries.
  - **React DevTools Profiler + why-did-you-render** — re-render do selector missing/global lift thừa.
  - **URL bar + History** — share link, back/forward.
  - **Network + Application (LocalStorage)** — cart persist, filter query string.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt 5 loại Server / Client Global / URL / Form / Ephemeral theo ownership/staleness/shareability — mỗi loại lưu ở đâu và vì sao `products` không nên ở Redux/Zustand?
2. Cart là loại gì và lifecycle của nó (guest → login → checkout) xử lý thế nào để không mất cart và không duplicate source of truth? Khi nào filter nên ở URL vs Zustand vs local `useState`?
3. Anti-pattern `Context` cho frequent state (cart, filter) gây re-render thế nào và fix bằng selector/shallow/tách Context ra sao? Colocation giúp gì cho Ephemeral?

<details>
<summary>Đáp án 30s</summary>

1. **Server** (snapshot DB, async, stale, multi-client) → **TanStack Query** (`queryKey`, `staleTime/gcTime`, dedup, background refetch). **Client Global** (sync, local ownership, qua nhiều route) → **Zustand/Redux/Jotai** (`useCart(s=>s.x)` selector). **URL** (serializable, shareable, back/forward, SSR) → **`useSearchParams`/`nuqs`** (`?category&page`). **Form** (high-freq, validation, dirty) → **RHF** `register` uncontrolled. **Ephemeral** (hover/focus, lifecycle = component) → **`useState` local**. `products` là Server State — nếu ở Redux phải tự cache/stale/dedup/background refetch/GC (≈100 dòng/resource) và luôn stale 0; Query lo hết bằng 1 `useQuery` với `staleTime`.
2. **Cart là Hybrid**: guest = **Client Global** `Zustand + persist localStorage` (offline add), login → `POST /api/cart/merge` sync lên server + `invalidate ['cart']`, sau đó `['cart']` là Server State. **Filter**: `?brand&page&sort` là **URL State** để share/bookmark/back/SSR; **draft** (slider đang kéo) là local `useState`, chỉ khi Apply mới `setSearchParams`. Chỉ khi filter là UI tạm không cần share (ví dụ multi-step wizard nội bộ) mới để Zustand.
3. **Context** không có selector — `useContext(AppContext)` re-render **tất cả consumer** khi bất kỳ field nào trong `value` đổi (`{ items, add }` đổi `items` thì cả `ProductCard` chỉ cần `add` cũng re-render). Fix: **tách** `CartItemsContext` + `CartActionsContext`, hoặc **Zustand selector** `useCart(s=>s.items.length)` + `useCart(s=>s.add)` + `shallow` cho object selector, hoặc **Jotai atom** per-field. **Colocation** giữ Ephemeral/Form ở gần nơi dùng nhất, tránh global pollution, giảm re-render, dễ test/xóa; lift chỉ khi ≥2 consumers không có quan hệ cha-con trực tiếp.

</details>

---
*Tham khảo chi tiết: `docs/09-state-management.md` — Câu 155, 159, 160, `docs/04-frontend-architecture.md` — Câu 75-77. Liên quan: `must-have-docs/07-architecture/04-data-sync-race.md` (Server sync chi tiết), `02-tanstack-query.md` (staleTime/gcTime), `03-zustand-redux-jotai.md` (store selector). Spec: [TanStack Query](https://tanstack.com/query/latest/docs/react/guides/important-defaults), [Kent C. Dodds — Colocation](https://kentcdodds.com/blog/state-colocation-will-make-your-react-app-faster).*
