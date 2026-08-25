# API Layer & Contract — OpenAPI/tRPC/codegen, Zod validation, Error/Loading Boundaries

> Tags: #API-Layer #Contract #OpenAPI #tRPC #Codegen #Zod #Interceptor #Retry #Error-Boundary #Loading #Validation | Nguồn: `docs/04-frontend-architecture.md` câu 78-80, `docs/12-system-design.md` Bài 1 | Mức: P0

## 1. Định nghĩa chính xác

**API Layer** là ranh giới cô lập giữa frontend và backend gồm 3 lớp: **`Client` (fetch/axios instance)** → **`Resource/Service` (typed functions)** → **`Hook/Query` (TanStack Query)**. Client lo `baseURL`, `interceptors`, `auth`, `retry/backoff`; Resource lo `endpoint`, `types`, `transform`; Hook lo `caching` và `UI binding`.

**Contract** là thỏa thuận kiểu giữa FE-BE: thực thi bằng **OpenAPI (Swagger) + codegen** (openapi-typescript, orval, hey-api) hoặc **tRPC** (type-share qua procedure) và **runtime validation** bằng **Zod** để chặn payload lệch trước khi render.

**Error/Loading Boundaries** là chuẩn hóa 4 trạng thái async `idle | loading | error | success (empty | filled)` qua `ErrorBoundary` (render error) + `Suspense/AsyncView` (loading/skeleton) + `global handler` (unhandledrejection, interceptor).

## 2. Cơ chế hoạt động

### 2.1 3 lớp API

```
Component → useProducts() [TanStack Query] → productApi.list() [Resource typed] → api [axios/fetch instance]
                                    ↘ Zod parse → throw nếu lệch contract
```

- **Client**: singleton `axios.create({ baseURL, timeout })` hoặc `fetch` wrap. Gắn `Authorization` qua request interceptor; response interceptor làm **refresh token queue**, **normalize error**, **retry with backoff**.
- **Resource**: `productApi.list(params) => api.get<Product[]>('/products', { params }).then(r => validated)` — 100% typed từ OpenAPI codegen, không `any`.
- **Hook**: `useQuery({ queryKey: ['products', params], queryFn: () => productApi.list(params) })` — dedupe, cache, background refetch.

### 2.2 Contract: OpenAPI codegen vs tRPC + Zod

- **OpenAPI**: BE xuất `openapi.json`, FE chạy `orval`/`openapi-typescript` sinh `types + client`. Đổi BE → CI generate lại → type break báo ngay. Kèm **Zod schema** (`zod-to-openapi` hoặc `orval --zod`) để **runtime parse** response: `productSchema.parse(data)` trước khi đưa vào UI.
- **tRPC**: share type qua `router.procedure.input(zod).query(...)`, FE `trpc.product.list.useQuery()` — type-safe end-to-end, không cần codegen, nhưng yêu cầu BE Node + tRPC.
- **ETag/If-Match** cho optimistic concurrency: `GET` trả `ETag: "v3"`, `PATCH` gửi `If-Match: "v3"` → 412 nếu stale.

### 2.3 Error Handling 3 tầng

1. **Render error** — `ErrorBoundary` (class) bọc route/widget, bắt lỗi render/lifecycle, hiện fallback + retry. Không bắt async/event.
2. **Async/Reject** — `window.addEventListener('unhandledrejection')` + `window.onerror` + axios interceptor + React Query `onError` global → log Sentry + toast.
3. **Event/Effect** — `try/catch` thủ công trong `onSubmit`, `onClick`.

### 2.4 Loading/Empty nhất quán — AsyncView

Không để mỗi component tự `if (loading) ... if (error) ...`. Dùng `<AsyncView query={...} skeleton empty errorFallback>` hoặc `Suspense + ErrorBoundary` riêng cho từng chart/section, không block nhau.

### 2.5 Retry/Backoff & Abort

- Retry chỉ cho **GET idempotent**, không retry POST. Backoff `2^i * 1000ms` (1s, 2s, 4s), max 3.
- `AbortController` per-request: `signal` từ `useQuery` hoặc tự tạo, `abort()` khi component unmount / filter đổi / debounce search để tránh **race condition** (response cũ ghi đè mới).

Sơ đồ refresh queue:

```
Request 401 → isRefreshing? queue.push(cb) : isRefreshing=true → refreshToken() → queue.forEach(cb(newToken)) → retry original
```

## 3. Ví dụ tối thiểu

```ts
// 3.1 Client + Interceptor + Refresh Queue + Retry/Backoff + Abort
import axios from 'axios';

export const api = axios.create({ baseURL: '/api', timeout: 10_000 });

api.interceptors.request.use(cfg => {
  const token = localStorage.getItem('token');
  if (token) cfg.headers.Authorization = `Bearer ${token}`;
  // AbortController — caller truyền signal
  return cfg;
});

let isRefreshing = false;
let queue: ((t: string) => void)[] = [];

api.interceptors.response.use(
  r => r,
  async err => {
    const original: any = err.config;
    if (err.response?.status === 401 && !original._retry) {
      if (isRefreshing) {
        return new Promise(res => queue.push(token => res(api({ ...original, headers: { ...original.headers, Authorization: `Bearer ${token}` } }))));
      }
      original._retry = true;
      isRefreshing = true;
      const newToken = await refreshToken(); // POST /auth/refresh
      queue.forEach(cb => cb(newToken));
      queue = [];
      isRefreshing = false;
      return api({ ...original, headers: { ...original.headers, Authorization: `Bearer ${newToken}` } });
    }
    // Normalize
    return Promise.reject({ message: err.response?.data?.message ?? err.message, status: err.response?.status });
  }
);

export const fetchWithRetry = async <T>(fn: () => Promise<T>, retries = 3): Promise<T> => {
  for (let i = 0; i < retries; i++) {
    try { return await fn(); } catch (e: any) {
      const status = e?.status;
      if (status && status >= 400 && status < 500 && status !== 429) throw e; // không retry 4xx (trừ 429)
      if (i === retries - 1) throw e;
      await new Promise(r => setTimeout(r, 2 ** i * 1000)); // backoff
    }
  }
  throw new Error('unreachable');
};

// 3.2 Resource — OpenAPI codegen + Zod runtime validation + ETag
import { z } from 'zod';

export const ProductSchema = z.object({
  id: z.string(),
  name: z.string(),
  price: z.number(),
  etag: z.string().optional(),
});
export type Product = z.infer<typeof ProductSchema>;

export const productApi = {
  list: async (params: { category?: string }, signal?: AbortSignal) => {
    const res = await api.get('/products', { params, signal });
    return z.array(ProductSchema).parse(res.data); // runtime contract check
  },
  get: async (id: string, signal?: AbortSignal) => {
    const res = await api.get(`/products/${id}`, { signal });
    const etag = res.headers.etag as string | undefined;
    const data = ProductSchema.parse(res.data);
    return { ...data, etag };
  },
  update: (id: string, patch: Partial<Product>, etag?: string) =>
    api.patch(`/products/${id}`, patch, { headers: etag ? { 'If-Match': etag } : {} }).then(r => ProductSchema.parse(r.data)),
  create: (data: Omit<Product, 'id'>) => api.post('/products', data).then(r => ProductSchema.parse(r.data)),
};

// orval codegen thay thế thủ công:
// orval.config.ts: input: './openapi.json', output: { target: './generated/api.ts', client: 'axios', zod: true }

// 3.3 tRPC alternative (nếu BE tRPC)
// router = t.procedure.input(z.object({ category: z.string().optional() })).query(({ input }) => db.products.find(input))
// FE: const { data } = trpc.product.list.useQuery({ category })

// 3.4 Hook — TanStack Query + Abort + ETag
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export const useProducts = (category?: string) =>
  useQuery({
    queryKey: ['products', category],
    queryFn: ({ signal }) => productApi.list({ category }, signal), // signal tự inject từ Query
    staleTime: 60_000,
  });

export const useProduct = (id: string) =>
  useQuery({ queryKey: ['product', id], queryFn: ({ signal }) => productApi.get(id, signal) });

// Optimistic update + If-Match
export const useUpdateProduct = () => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, patch, etag }: { id: string; patch: Partial<Product>; etag?: string }) =>
      productApi.update(id, patch, etag),
    onMutate: async ({ id, patch }) => {
      await qc.cancelQueries({ queryKey: ['product', id] });
      const prev = qc.getQueryData<Product>(['product', id]);
      qc.setQueryData(['product', id], (old: any) => ({ ...old, ...patch }));
      return { prev };
    },
    onError: (err: any, { id }, ctx: any) => {
      if (err?.status === 412) toast.error('Dữ liệu đã bị sửa bởi người khác, đang tải lại...');
      qc.setQueryData(['product', id], ctx.prev); // rollback
    },
    onSettled: (_d, _e, { id }) => qc.invalidateQueries({ queryKey: ['product', id] }),
  });
};
```

```tsx
// 3.5 Error Boundaries 3 tầng
import { ErrorBoundary } from 'react-error-boundary';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const qc = new QueryClient({
  defaultOptions: {
    queries: { retry: 2, throwOnError: false }, // retry chỉ GET
    mutations: { onError: (err: any) => toast.error(err.message) },
  },
});

function App() {
  return (
    <QueryClientProvider client={qc}>
      {/* Tầng 1: route-level */}
      <ErrorBoundary FallbackComponent={RouteError} onReset={() => window.location.reload()}>
        <Routes>
          <Route path="/products" element={
            <ErrorBoundary fallback={<SectionError />}>
              <ProductPage />
            </ErrorBoundary>
          } />
        </Routes>
      </ErrorBoundary>
    </QueryClientProvider>
  );
}
function RouteError({ error, resetErrorBoundary }: { error: Error; resetErrorBoundary: () => void }) {
  return <div><h1>Lỗi</h1><pre>{error.message}</pre><button onClick={resetErrorBoundary}>Thử lại</button></div>;
}

// Tầng 2: global async
window.addEventListener('unhandledrejection', e => { reportToSentry(e.reason); e.preventDefault(); });
window.addEventListener('error', e => reportToSentry(e.error));

// Tầng 3: event — try/catch thủ công
async function onSubmit(data: any) {
  try { await productApi.create(data); toast.success('Tạo thành công'); }
  catch (e: any) { toast.error(e.message); }
}

// 3.6 Loading/Empty nhất quán — AsyncView + Suspense
type AsyncViewProps<T> = {
  query: { data?: T; isLoading: boolean; error?: Error; isEmpty?: boolean };
  skeleton?: React.ReactNode; empty?: React.ReactNode;
  errorFallback?: (err: Error, retry: () => void) => React.ReactNode;
  children: (data: T) => React.ReactNode;
};
function AsyncView<T>({ query, skeleton, empty, errorFallback, children }: AsyncViewProps<T>) {
  if (query.isLoading) return <>{skeleton ?? <ListSkeleton />}</>;
  if (query.error) return <>{errorFallback ? errorFallback(query.error, () => window.location.reload()) : <ErrorCard error={query.error} />}</>;
  if (query.isEmpty || (Array.isArray(query.data) && query.data.length === 0)) return <>{empty ?? <EmptyState />}</>;
  return <>{children(query.data as T)}</>;
}
function ProductPage() {
  const q = useProducts('shoes');
  return (
    <AsyncView
      query={{ data: q.data, isLoading: q.isLoading, error: q.error as Error, isEmpty: q.data?.length === 0 }}
      skeleton={<ProductSkeleton count={6} />}
      empty={<EmptyState title="Chưa có sản phẩm" action={<Button>Tạo mới</Button>} />}
      errorFallback={(err, retry) => <ErrorCard message={err.message} onRetry={retry} />}
    >
      {data => <ProductGrid products={data} />}
    </AsyncView>
  );
}
// Hoặc Suspense
// <Suspense fallback={<ProductSkeleton />}><Products /></Suspense>
// <ErrorBoundary fallback={<ErrorCard />}><Products /></ErrorBoundary>

// 3.7 Abort + Race — search debounce
function useSearch(q: string) {
  const debounced = useDebounce(q, 300);
  return useQuery({
    queryKey: ['search', debounced],
    queryFn: ({ signal }) => fetch(`/api/search?q=${debounced}`, { signal }).then(r => r.json()),
    enabled: debounced.length >= 2,
  });
}
```

## 4. So sánh / Phân loại

| Tiêu chí | OpenAPI + Codegen | tRPC |
|----------|-------------------|------|
| BE yêu cầu | Bất kỳ (REST), xuất `openapi.json` | Node + tRPC router |
| FE client | `orval`/`hey-api` sinh `types + axios/fetch + zod` | `trpc` client share type trực tiếp |
| Type safety | Build-time (codegen) + runtime (Zod parse) | End-to-end type-share, không codegen |
| Runtime validation | Bắt buộc Zod `parse` response | Zod ở `procedure.input` + có thể parse response |
| Khi dùng | Team đa ngôn ngữ, BE không Node, REST sẵn | Full-stack TypeScript, muốn velocity cao |

| Lớp | Trách nhiệm | Ví dụ | Không làm |
|-----|-------------|-------|-----------|
| Client | baseURL, interceptors, auth, retry/backoff, Abort | `api` singleton | Không chứa endpoint cụ thể |
| Resource | endpoint, typed function, Zod validate, ETag | `productApi.list()` | Không chứa cache/UI |
| Hook/Query | cache, dedupe, background refetch, optimistic | `useProducts()` | Không gọi `fetch` trực tiếp trong component |

| Error tầng | Bắt gì | Không bắt | Fix |
|------------|--------|-----------|-----|
| ErrorBoundary | Render/lifecycle của con | `setTimeout`, `Promise`, `event handler` | Bọc route/widget |
| Global handler (`unhandledrejection`, interceptor) | Async reject chưa catch, API error | Render error | Log Sentry + toast |
| try/catch event | `onClick`, `onSubmit` | Render | Catch thủ công |

| Loading | Khi dùng | Trade-off |
|---------|----------|-----------|
| Skeleton | List/card loading, giữ layout, tránh CLS | Tốn code skeleton |
| Spinner toàn màn hình | Page transition, blocking | UX kém nếu lạm dụng |
| Suspense | Code-split + data fetch (React 18) | Cần ErrorBoundary kèm |
| Empty + CTA | `data.length === 0` | Không để trang trắng |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không để component gọi `fetch` trực tiếp**: khó mock, khó đổi `baseURL`, khó gắn `Authorization`, duplicate retry/ETag. Luôn qua Resource layer.
- **Không retry cho POST/PUT non-idempotent**: retry `POST /checkout` không idempotencyKey sẽ tạo double order. Chỉ retry **GET** hoặc POST có `Idempotency-Key`.
- **Không dùng `AccessError: *` với credentials tuỳ tiện**: xem `06-network-security`.
- **Không Zod `parse` mọi response nếu payload lớn 1M rows**: `parse` tốn CPU; chỉ parse ở boundary (list/detail), không parse mỗi row trong loop nóng. Cân bằng với `zod` + `superstruct` lightweight.
- **Không bọc quá nhiều ErrorBoundary**: mỗi button 1 boundary → verbose; bọc quá ít → 1 lỗi con sập cả app. Quy tắc: **route + widget quan trọng**.
- **Không `staleTime: 0` cho mọi query**: mặc định `0` làm mỗi mount lại fetch; list ít đổi nên `60s`.
- **Không tRPC khi BE không TypeScript**: ép tRPC vào BE Java/Go tốn cầu nối, OpenAPI hợp hơn.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Không Zod validation → render crash khi BE đổi contract**
  - Triệu chứng: `product.price` từ `number` thành `string`, UI `price.toFixed` crash, không báo ở build.
  - Fix: `ProductSchema.parse(res.data)` ở Resource; CI chạy `orval` generate lại và type-check.
  - Đo: `npx tsc --noEmit`, `zod` error log, Sentry `ZodError`.

- **Lỗi 2: Retry POST checkout → double order**
  - Triệu chứng: user bấm 1 lần nhưng 2 order vì retry sau 504.
  - Fix: chỉ retry GET; POST kèm `Idempotency-Key: uuid` header, BE dedup; `fetchWithRetry` check `status <500` thì không retry.
  - Đo: Network → retry count, BE log `idempotencyKey` duplicate, `Sentry` duplicate order.

- **Lỗi 3: Thiếu AbortController → race condition search**
  - Triệu chứng: gõ `a` → `ab` → response `a` về sau ghi đè `ab`, hiển thị sai.
  - Fix: `signal` từ `useQuery` / `AbortController` + `abort()` khi `debounced` đổi; dùng `switchMap` mental model (chỉ latest).
  - Đo: Network → 2 request, response order; React Query `queryFn: ({ signal })` tự abort khi key đổi.

- **Lỗi 4: 401 refresh không queue → 5 request cùng 401 đua nhau refresh 5 lần**
  - Triệu chứng: 5 token refresh song song, race, token cuối thắng nhưng 4 request fail.
  - Fix: `isRefreshing` flag + queue như snippet trên.
  - Đo: Network → 5 `POST /auth/refresh` cùng lúc; fix xong chỉ 1.

- **Lỗi 5: ETag/If-Match thiếu → lost update**
  - Triệu chứng: 2 admin sửa cùng product, người sau ghi đè người trước không báo.
  - Fix: `GET` lưu `ETag`, `PATCH` gửi `If-Match`, BE trả `412 Precondition Failed` nếu stale, FE reload.
  - Đo: Network headers `ETag` / `If-Match`, BE test 412.

- **Lỗi 6: ErrorBoundary không bắt async**
  - Triệu chứng: `throw` trong `setTimeout`/`Promise` không hiện fallback.
  - Fix: async phải qua `window.addEventListener('unhandledrejection')` hoặc `try/catch` + `setState` để throw trong render.
  - Đo: Console `Uncaught (in promise)`, Sentry.

- **Lỗi 7: Loading/Empty không nhất quán → có nơi spinner, nơi trắng**
  - Triệu chứng: UX lệch, không CTA khi empty.
  - Fix: `AsyncView` chuẩn hóa 4 trạng thái + `EmptyState` có CTA.
  - Đo: visual review Storybook, `grep -r "isLoading" src --include="*.tsx" | wc -l` check duplicate pattern.

- **Công cụ**:
  - **Network** — status, retry, `ETag`/`If-Match`, `Idempotency-Key`, abort (canceled).
  - **React Query DevTools** — cache, stale, retry, gc.
  - **`npx tsc --noEmit`** + **`orval` codegen diff** — contract drift.
  - **Sentry** — error normalize, 412, ZodError.
  - **MSW** — mock API để test loading/error/empty.

## 7. Câu hỏi tự kiểm tra

1. Nêu 3 lớp API Layer và vì sao phải tách Client/Resource/Hook? OpenAPI codegen khác tRPC thế nào và Zod runtime validation đặt ở đâu?
2. Phân biệt 3 tầng Error Handling (ErrorBoundary / global handler / try/catch event) — mỗi tầng bắt gì và không bắt gì? Vì sao ErrorBoundary không bắt được `Promise`?
3. Khi nào retry/backoff được và không được? `AbortController` giải quyết race condition thế nào và ETag/If-Match ngăn lost update ra sao?

<details>
<summary>Đáp án 30s</summary>

1. **Client** (axios/fetch singleton lo `baseURL`, `interceptors` auth, `retry/backoff`, `normalize error`), **Resource** (`productApi.list()` typed, Zod `parse` response, ETag), **Hook** (`useQuery` cache/dedupe/background refetch). Tách để đổi backend không đụng UI, test dễ mock Resource, gắn auth/retry 1 chỗ. **OpenAPI**: BE xuất `openapi.json` → FE `orval/openapi-typescript` sinh types+client+zod, hợp mọi BE; **tRPC**: share type qua `router.procedure.input(zod)` → FE `trpc.useQuery`, không codegen nhưng yêu cầu BE Node. **Zod** đặt ở **Resource boundary** `ProductSchema.parse(res.data)` để chặn payload lệch trước render, không parse mỗi row nóng.
2. **ErrorBoundary** (class, bọc route/widget) bắt **render/lifecycle** của con, hiện fallback + retry, **không bắt** `setTimeout/Promise/event handler` vì chúng ngoài render phase. **Global handler** (`unhandledrejection`/`onerror` + axios interceptor + Query `onError`) bắt **async reject chưa catch**, log Sentry + toast. **try/catch event** bắt **onClick/onSubmit** thủ công. Vì ErrorBoundary chỉ `componentDidCatch` trong commit phase, async tách khỏi call stack nên phải `catch` rồi `setState` để throw lại trong render mới bắt được.
3. **Retry/backoff** chỉ cho **GET idempotent** (2**i *1000ms, max 3), **không retry POST/PUT** không `Idempotency-Key` vì tạo double order; 4xx (trừ 429) không retry. **AbortController**: mỗi search tạo `signal`, khi input đổi hoặc unmount thì `abort()` request cũ → chỉ latest ghi kết quả, tránh response cũ ghi đè mới (race). **ETag/If-Match**: `GET` trả `ETag: "v3"`, `PATCH` gửi `If-Match: "v3"`, BE so sánh version, nếu khác trả **412 Precondition Failed** → FE báo "dữ liệu đã bị sửa, đang tải lại" và refetch, ngăn **lost update** khi 2 người sửa cùng resource.

</details>

---
*Tham khảo chi tiết: `docs/04-frontend-architecture.md` — Câu 78, 79, 80. `docs/12-system-design.md` — Bài 1 (API/BFF). Spec: [OpenAPI](https://spec.openapis.org/oas/latest.html), [tRPC](https://trpc.io/), [Zod](https://zod.dev/), [TanStack Query](https://tanstack.com/query).*

