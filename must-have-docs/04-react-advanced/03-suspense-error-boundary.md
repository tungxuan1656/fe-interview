# Suspense và Error Boundary — Throw Promise, fallback, getDerivedStateFromError và reset bằng key

> Tags: #react #suspense #error-boundary #throw-promise #fallback #concurrent | Nguồn: `docs/03-react-nang-cao.md` câu 56, 57, 61 | Mức: P0

## 1. Định nghĩa chính xác

- **Suspense** là component cho phép subtree **"suspend"** (tạm dừng render) trong khi chờ tác vụ bất đồng bộ (code splitting, data fetching), và hiển thị `fallback` của `<Suspense>` gần nhất. Cơ chế: component **throw Promise** trong render, React bắt Promise, treo render nhánh đó, hiện fallback; khi Promise resolve, React retry render.
- **Suspense boundary** là `<Suspense fallback={<Skeleton />}>` — ranh giới quyết định phạm vi fallback. Lồng nhau được, fallback gần nhất thắng.
- **Error Boundary (EB)** là **class component** (hoặc lib `react-error-boundary`) với `static getDerivedStateFromError(error)` và `componentDidCatch(error, info)` để bắt lỗi **trong render, lifecycle, constructor của con**. EB không bắt lỗi ở event handler, async (`setTimeout`, `Promise`), SSR, và chính nó.
- **Throw Promise vs throw Error**: `throw Promise` → Suspense bắt (loading); `throw Error` → EB bắt (error). Cả hai đều là **synchronous throw trong render**, không phải `try/catch` async.

## 2. Cơ chế hoạt động

### 2.1 Suspense throw Promise — chi tiết

```
1. Component render → gọi resource.read()
2. resource.read() thấy status="pending" → throw Promise (suspender)
3. React bắt Promise tại Suspense boundary gần nhất → mount fallback
4. Promise resolve → React schedule retry render nhánh suspend
5. Lần retry: resource.read() trả data → render thành công, thay fallback bằng children
6. Nếu Promise reject → throw Error → EB gần nhất bắt
```

- Trước React 18, Suspense chỉ cho `React.lazy` (code splitting). Từ 18+ dùng cho data fetching với framework (Next.js async Server Component, Relay) hoặc `useSuspenseQuery` (`@tanstack/react-query` `suspense:true`) hoặc `use` (React 19).
- `use` API (React 19) cho phép `const data = use(promise)` trong render — cũng throw Promise nếu pending, tương tự `read()`.

### 2.2 Resource wrapper minh họa (không dùng trực tiếp, framework làm sẵn)

```js
function createResource(promise) {
  let status = 'pending', result;
  const suspender = promise.then(
    r => { status = 'success'; result = r; },
    e => { status = 'error'; result = e; }
  );
  return {
    read() {
      if (status === 'pending') throw suspender; // Suspense bắt
      if (status === 'error') throw result;      // EB bắt
      return result;
    }
  };
}
```

- Thực tế: không tự viết `createResource` trong App Router; dùng `async Server Component` (Next.js tự suspend), hoặc `useSuspenseQuery`, hoặc `use(promise)`.

### 2.3 Error Boundary — class bắt buộc

- Function component không thể làm EB vì cần `getDerivedStateFromError` (static, chạy trong render phase để update state) và `componentDidCatch` (commit phase, để log).
- `getDerivedStateFromError` → trả state mới để render fallback. `componentDidCatch(error, { componentStack })` → log ra service (Sentry), không update UI trực tiếp.
- EB chỉ bắt lỗi **bên dưới** nó trong tree, không bắt lỗi của chính nó và không bắt lỗi ngoài React (event handler).

### 2.4 Kết hợp Suspense + EB — bộ đôi

```
<ErrorBoundary fallback={<ErrorPage />}>
  <Suspense fallback={<Skeleton />}>
    <User />   {/* throw Promise → Suspense; throw Error → EB */}
  </Suspense>
</ErrorBoundary>
```

- Thứ tự: Suspense bắt Promise trước, EB bắt Error. Nếu không có EB, lỗi fetch sẽ unmount cả cây (React 16+).

### 2.5 Reset sau lỗi — bằng `key`

- EB giữ `hasError: true` sau khi bắt lỗi, không tự reset khi children thay đổi. Để retry, **đổi `key`** của EB hoặc gọi `resetErrorBoundary`.
- Pattern: `<ErrorBoundary key={userId}>` — `userId` đổi → EB remount → state reset. Hoặc lib `react-error-boundary` cung cấp `resetKeys` prop và `FallbackComponent` với `resetErrorBoundary`.

### 2.6 Suspense lồng nhau và reveal

- Suspense lồng nhau cho phép streaming từng phần: header hiện ngay, comments suspend riêng.
- `SuspenseList` (experimental, ít dùng) từng định nghĩa order reveal; nay thay bằng Suspense lồng + streaming SSR.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 React.lazy + Suspense — code splitting
import { lazy, Suspense } from 'react';
const Comments = lazy(() => import('./Comments')); // phải default export

function App() {
  return (
    <Suspense fallback={<div>Loading comments…</div>}>
      <Comments />
    </Suspense>
  );
}

// 3.2 Data fetching — throw Promise với resource (demo)
function createResource<T>(promise: Promise<T>) {
  let status: 'pending' | 'success' | 'error' = 'pending';
  let result: T;
  let error: unknown;
  const suspender = promise.then(
    r => { status = 'success'; result = r; },
    e => { status = 'error'; error = e; }
  );
  return {
    read(): T {
      if (status === 'pending') throw suspender;
      if (status === 'error') throw error;
      return result!;
    },
  };
}

const userResource = createResource(fetch('/api/user').then(r => r.json()));

function User() {
  const user = userResource.read(); // suspend nếu pending
  return <div>{user.name}</div>;
}

function UserPage() {
  return (
    <Suspense fallback={<div>Skeleton…</div>}>
      <User />
    </Suspense>
  );
}

// 3.3 Thực tế với TanStack Query (khuyến nghị thay resource thủ công)
import { useSuspenseQuery } from '@tanstack/react-query';

function Todos() {
  const { data } = useSuspenseQuery({
    queryKey: ['todos'],
    queryFn: async () => fetch('/api/todos').then(r => r.json()),
  });
  return <ul>{data.map((t: any) => <li key={t.id}>{t.text}</li>)}</ul>;
}

function TodosPage() {
  return (
    <Suspense fallback={<div>Loading todos…</div>}>
      <Todos />
    </Suspense>
  );
}

// 3.4 React 19 — use(promise) trong render (chỉ trong Suspense boundary)
import { use } from 'react';

function CommentsWithUse({ commentsPromise }: { commentsPromise: Promise<{ id: string; text: string }[]> }) {
  const comments = use(commentsPromise); // throw Promise nếu pending
  return <>{comments.map(c => <div key={c.id}>{c.text}</div>)}</>;
}

// Có thể condition nhưng chỉ trong render có Suspense boundary
function Maybe({ shouldFetch, promise }: { shouldFetch: boolean; promise: Promise<any> }) {
  if (shouldFetch) {
    const data = use(promise);
    return <div>{JSON.stringify(data)}</div>;
  }
  return null;
}

// 3.5 Error Boundary — class bắt buộc
import React from 'react';

type EBProps = { fallback?: React.ReactNode; children: React.ReactNode };
type EBState = { hasError: boolean; error: Error | null };

class ErrorBoundary extends React.Component<EBProps, EBState> {
  state: EBState = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): Partial<EBState> {
    // render phase — update state để lần render sau hiện fallback
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // commit phase — log
    console.error('EB caught:', error, info.componentStack);
    // logToService(error, info);
  }

  reset = () => this.setState({ hasError: false, error: null });

  render() {
    if (this.state.hasError) {
      return (
        this.props.fallback ?? (
          <div role="alert">
            Something went wrong: {this.state.error?.message}
            <button onClick={this.reset}>Retry</button>
          </div>
        )
      );
    }
    return this.props.children;
  }
}

// 3.6 Kết hợp + reset bằng key
function AppWithBoth({ userId }: { userId: string }) {
  return (
    // key đổi → EB remount → reset hasError
    <ErrorBoundary key={userId} fallback={<div>Failed to load user</div>}>
      <Suspense fallback={<div>Loading user…</div>}>
        <UserById id={userId} />
      </Suspense>
    </ErrorBoundary>
  );
}

function UserById({ id }: { id: string }) {
  // với Next.js App Router: async Server Component tự suspend, không cần resource
  return <div>User {id}</div>;
}

// 3.7 react-error-boundary (khuyến nghị cho function component)
import { ErrorBoundary as REB } from 'react-error-boundary';

function AppREB() {
  return (
    <REB
      FallbackComponent={({ error, resetErrorBoundary }) => (
        <div>
          {error.message} <button onClick={resetErrorBoundary}>Retry</button>
        </div>
      )}
      onReset={() => console.log('reset')}
      // resetKeys: khi key đổi thì auto reset
      resetKeys={[1]}
    >
      <Suspense fallback={<div>Loading…</div>}>
        <Todos />
      </Suspense>
    </REB>
  );
}

// 3.8 Next.js App Router — error.tsx và loading.tsx là Suspense/EB file-based
// app/products/error.tsx
// "use client";
// export default function Error({ error, reset }: { error: Error; reset: () => void }) {
//   return <div>{error.message} <button onClick={reset}>Try again</button></div>;
// }
// app/products/loading.tsx
// export default function Loading() { return <div>Skeleton</div>; }
```

## 4. So sánh / Phân loại

| Tiêu chí | `throw Promise` (Suspense) | `throw Error` (Error Boundary) |
|----------|----------------------------|--------------------------------|
| Để làm gì | Loading — chờ data/code | Error — render lỗi |
| Ai bắt | `<Suspense fallback>` gần nhất | `ErrorBoundary` gần nhất |
| Khi nào throw | `status === "pending"` → `throw suspender` | `status === "error"` → `throw error` |
| Sau resolve/reject | Promise resolve → retry render | Error → hiện fallback error, không retry tự động |

| Tiêu chí | Suspense | isLoading thủ công (`useState`/`useQuery` không suspense) |
|----------|----------|-----------------------------------------------------------|
| Khai báo | Declarative — `<Suspense fallback>` bọc | Imperative — `if (isLoading) return <Spinner />` mỗi component |
| Waterfall | Tự gộp với streaming SSR, parallel | Dễ waterfall nếu không `Promise.all` |
| UX | Fallback gần nhất, có thể giữ UI cũ với `useTransition` | Phải tự quản lý từng `isLoading` |
| Yêu cầu | Framework/lib hỗ trợ throw Promise | Không yêu cầu |

| Tiêu chí | Error Boundary (class) | `try/catch` trong event handler | `window.onerror` |
|----------|------------------------|--------------------------------|------------------|
| Bắt lỗi render/lifecycle/constructor con | ✅ | ❌ | ❌ (chỉ log) |
| Bắt lỗi event handler (`onClick`) | ❌ | ✅ | ✅ |
| Bắt lỗi async (`setTimeout`, `Promise`) | ❌ | ✅ (trong async) | ✅ `unhandledrejection` |
| Bắt lỗi SSR | ❌ | ❌ | ❌ |
| Bắt lỗi chính nó | ❌ | — | — |

| Cách reset EB | Khi dùng |
|---------------|----------|
| Đổi `key` trên EB (`<EB key={id}>`) | Khi id/route đổi, muốn reset toàn bộ |
| `resetErrorBoundary()` của `react-error-boundary` | Nút Retry, reset query |
| `reset` prop của `error.tsx` (Next.js) | File-based, Next cung cấp |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng Suspense cho data fetching client-only nếu không có framework**: Tự viết `createResource` thủ công dễ sai (cache, race, error). Nếu SPA thuần không dùng Next/Relay, dùng `useQuery` với `isLoading` hoặc `useSuspenseQuery` của TanStack (đã wrap Suspense sẵn).
- **Không đặt Suspense quá cao**: `<Suspense>` ở root làm fallback che cả app khi một widget suspend → UX tệ. Đặt quanh **route/feature** hoặc **từng section** để streaming từng phần.
- **Không đặt EB quá cao hoặc quá thấp**: EB ở root → fallback che hết app; EB quanh từng button → boilerplate. Đặt quanh **route/feature** (ví dụ: `<ErrorBoundary><ProductList /></ErrorBoundary>`).
- **Không mong EB bắt async/event**: `onClick={async () => { throw new Error() }}` không được EB bắt. Phải `try/catch` trong handler và `setState` error, hoặc dùng `window.addEventListener('unhandledrejection')` cho global.
- **Không dùng function component làm EB**: Không có `getDerivedStateFromError`. Phải dùng class hoặc `react-error-boundary`.
- **Không quên reset**: EB sau khi `hasError=true` sẽ giữ fallback mãi, không tự hết khi data sửa. Phải reset bằng `key` hoặc nút Retry, nếu không user kẹt ở error.
- **Chi phí**: Mỗi Suspense là một boundary thêm Fiber, nhưng rẻ. EB thêm class component và `componentDidCatch` log.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Quên bọc `<Suspense>` cho `React.lazy` → crash**
  - Triệu chứng: `A component suspended while responding to synchronous input. This will cause the UI to be replaced with a blank display...`
  - Fix: Mọi `lazy` phải trong `<Suspense fallback>`. Với Next.js `dynamic`, đã có `loading`.
  - Đo: React DevTools → Components → suspended component highlight.

- **Lỗi 2: Suspense fallback che cả page vì đặt quá cao**
  - Triệu chứng: Một widget chậm làm cả page hiện `<Skeleton>` toàn màn hình.
  - Fix: Tách Suspense theo section: `<Suspense><Header /></Suspense><Suspense><Comments /></Suspense>` để streaming riêng.
  - Đo: Profiler → xem fallback commit; Lighthouse → FCP chậm do một Suspense lớn.

- **Lỗi 3: Mong EB bắt lỗi event handler/async**
  - Triệu chứng: `onClick={() => { throw new Error('oops') }}` làm app crash, EB không hiện fallback.
  - Fix: `try/catch` trong handler + `setError`, hoặc `window.onerror`/`unhandledrejection` cho global. EB chỉ cho render phase.
  - Đo: Console → `ErrorBoundary` không log, `componentDidCatch` không chạy.

- **Lỗi 4: EB không reset sau lỗi**
  - Triệu chứng: Fetch lỗi một lần, sau đó data đã sửa nhưng UI vẫn `Something went wrong`.
  - Fix: `<ErrorBoundary key={query} resetKeys={[query]}>` hoặc nút `resetErrorBoundary` gọi `refetch`.
  - Đo: React DevTools → State `hasError: true` vẫn giữ dù props đổi.

- **Lỗi 5: Dùng function component làm EB**
  - Triệu chứng: Không có `getDerivedStateFromError`, lỗi vẫn crash.
  - Fix: Dùng class hoặc `import { ErrorBoundary } from 'react-error-boundary'`.
  - Đo: ESLint `react/no-unstable-nested-component` không liên quan; kiểm tra `extends React.Component`.

- **Lỗi 6: Throw Promise không được cache → suspend vô hạn**
  - Triệu chứng: Mỗi render tạo promise mới → throw promise mới → luôn pending, Suspense không bao giờ resolve.
  - Fix: Promise/resource phải **stable** giữa render (tạo ngoài component hoặc `useMemo`/`cache`). Với `use(promise)`, promise phải từ parent không recreate mỗi render.
  - Đo: Thêm `console.log('throw', promise)` — nếu mỗi render là promise khác thì sai.

- **Công cụ**:
  - **React DevTools Profiler**: Xem Suspense boundary, thời gian suspend, fallback mount/unmount.
  - **`onRecoverableError`**: `hydrateRoot`/`createRoot` log lỗi EB.
  - **Sentry / `componentDidCatch`**: `info.componentStack` để biết cây lỗi.
  - **`why-did-you-render`**: Không liên quan trực tiếp, nhưng phát hiện re-render gây suspend lặp.

## 7. Câu hỏi tự kiểm tra

1. Suspense bắt `throw Promise` thế nào, khác gì `isLoading` thủ công, và vì sao Promise/resource phải stable giữa render?
2. Error Boundary phải là class với `getDerivedStateFromError` và `componentDidCatch` — mỗi cái chạy ở phase nào, bắt được/không bắt được lỗi nào?
3. Vì sao EB sau khi `hasError=true` không tự hết, và hai cách reset là gì (đổi `key` vs `resetErrorBoundary`/`reset` của Next.js)?

<details>
<summary>Đáp án 30s</summary>

1. **Suspense**: component `throw Promise` trong render (ví dụ `resource.read()` khi pending), React bắt tại `<Suspense>` gần nhất, mount `fallback`, khi Promise resolve thì retry render và thay fallback bằng children. Khác `isLoading` thủ công ở chỗ declarative và gộp được với streaming SSR (parallel, không waterfall). **Phải stable**: nếu mỗi render tạo `new Promise`, mỗi lần throw là promise mới luôn pending → Suspense không bao giờ resolve, treo vô hạn. Phải tạo promise/resource ngoài component hoặc `useMemo` hoặc truyền từ parent stable.

2. **EB phải là class** vì cần `static getDerivedStateFromError(error)` (chạy trong **render phase** để trả state mới `hasError:true` cho lần render sau hiện fallback) và `componentDidCatch(error, info)` (chạy trong **commit phase** để log `componentStack` ra service). **Bắt được**: lỗi trong `render`, `lifecycle`, `constructor` của con. **Không bắt**: event handler (`onClick`), async (`setTimeout`/`Promise`), SSR, và lỗi của chính EB.

3. EB giữ `hasError:true` trong state, không tự reset khi props đổi — nếu không reset, user kẹt ở error dù data đã sửa. Cách reset: (a) **đổi `key`** trên EB (`<ErrorBoundary key={userId}>` — `userId` đổi thì EB remount, state reset); (b) **gọi `resetErrorBoundary`** (lib `react-error-boundary` cung cấp `resetKeys` + `FallbackComponent` với `resetErrorBoundary`) hoặc `reset()` của `error.tsx` trong Next.js App Router — nút Retry gọi `reset` để clear state và retry render.

</details>

---
*Tham khảo chi tiết: `docs/03-react-nang-cao.md` — Câu 56, 57, 61. Spec: [React Docs — Suspense](https://react.dev/reference/react/Suspense), [Error Boundary](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary), [react-error-boundary](https://github.com/bvaughn/react-error-boundary).*
