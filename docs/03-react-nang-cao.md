# 03. React Nâng Cao - 18 Câu Hỏi Senior

> 18 câu hỏi chuyên sâu (Câu 51-68) về Fiber, Concurrent Rendering, React 18/19, Suspense, Server Components, SSR/CSR/SSG/ISR và chiến lược tối ưu hiệu năng ở scale lớn.

## Mục lục

- [Câu 51: React Fiber Architecture](#câu-51-react-fiber-architecture)
- [Câu 52: Concurrent Rendering là gì?](#câu-52-concurrent-rendering-là-gì)
- [Câu 53: React 18 - Những tính năng nổi bật](#câu-53-react-18---những-tính-năng-nổi-bật)
- [Câu 54: React 19 - Actions, `use` và React Compiler](#câu-54-react-19---actions-use-và-react-compiler)
- [Câu 55: useTransition vs useDeferredValue](#câu-55-usetransition-vs-usedeferredvalue)
- [Câu 56: Suspense cho Data Fetching](#câu-56-suspense-cho-data-fetching)
- [Câu 57: Error Boundary và xử lý lỗi toàn cục](#câu-57-error-boundary-và-xử-lý-lỗi-toàn-cục)
- [Câu 58: Server Components (RSC) vs Client Components](#câu-58-server-components-rsc-vs-client-components)
- [Câu 59: Hydration, Selective Hydration và Hydration Error](#câu-59-hydration-selective-hydration-và-hydration-error)
- [Câu 60: SSR vs CSR vs SSG vs ISR](#câu-60-ssr-vs-csr-vs-ssg-vs-isr)
- [Câu 61: Streaming SSR và Suspense](#câu-61-streaming-ssr-và-suspense)
- [Câu 62: Code Splitting, lazy và dynamic import](#câu-62-code-splitting-lazy-và-dynamic-import)
- [Câu 63: Performance Debugging - Profiler và DevTools](#câu-63-performance-debugging---profiler-và-devtools)
- [Câu 64: Chiến lược tối ưu Re-render toàn diện](#câu-64-chiến-lược-tối-ưu-re-render-toàn-diện)
- [Câu 65: Concurrent Features - useId và useSyncExternalStore](#câu-65-concurrent-features---useid-và-usesyncexternalstore)
- [Câu 66: Server Actions và Mutations trong Next.js](#câu-66-server-actions-và-mutations-trong-nextjs)
- [Câu 67: Caching Strategy - React cache, fetch cache và Next.js cache](#câu-67-caching-strategy---react-cache-fetch-cache-và-nextjs-cache)
- [Câu 68: Tư duy thiết kế hệ thống Frontend với React ở scale lớn](#câu-68-tư-duy-thiết-kế-hệ-thống-frontend-với-react-ở-scale-lớn)

---

### Câu 51: React Fiber Architecture

**Trả lời Senior:**
Fiber là **reimplementation của reconciler** từ React 16, thay Stack Reconciler (đệ quy đồng bộ, không interrupt được). Fiber biến cây component thành **linked list** (child, sibling, return) và chia công việc thành **units of work** có thể pause, abort, reuse, và ưu tiên.

Mỗi Fiber node là object chứa: `type`, `props`, `stateNode` (DOM), `child/sibling/return`, `alternate` (double buffer - current vs workInProgress), `effectTag`, `lanes` (priority).

Cơ chế:

1.  **Render phase (reconcile):** có thể interrupt, chạy `beginWork`/`completeWork` cho từng fiber, không có side effect. Có thể bỏ dở nếu có update ưu tiên cao hơn (user input).
2.  **Commit phase:** đồng bộ, không interrupt, apply DOM mutations, chạy `useLayoutEffect`.

```javascript
// Cấu trúc Fiber đơn giản
const fiber = {
  tag: 'FunctionComponent',
  key: null,
  elementType: MyComponent,
  stateNode: null, // DOM hoặc instance
  child: fiberChild,
  sibling: fiberSibling,
  return: fiberParent,
  pendingProps: { count: 1 },
  memoizedProps: { count: 0 },
  memoizedState: { hooks: [...] },
  alternate: currentFiber, // double buffer
  lanes: SyncLane, // priority
};

// Double buffering: current tree (hiển thị) và workInProgress tree (đang build)
// Khi commit xong, workInProgress thành current
```

Nhờ Fiber mà Concurrent Features, Suspense, time-slicing mới khả thi. Stack Reconciler cũ không thể pause nên animation bị block khi render nặng.

**Trade-off:** Fiber phức tạp, memory overhead hơn, nhưng cho phép cooperative scheduling với Scheduler (ưu tiên, deadline).

**Câu hỏi đào sâu:** Vì sao Fiber dùng linked list thay vì tree? `alternate` double buffer để làm gì? Lanes là gì?

---

### Câu 52: Concurrent Rendering là gì?

**Trả lời Senior:**
Concurrent Rendering là khả năng React **render nhiều version của UI cùng lúc mà không block main thread**, và có thể **interrupt, ưu tiên, bỏ dở** công việc render. Trước đây render là đồng bộ, blocking - gõ input mà đang render list 10k item thì input lag. Concurrent cho phép React pause render list, xử lý input trước, rồi tiếp tục hoặc restart render.

Không phải là "chạy song song trên nhiều thread" (JS vẫn single-thread), mà là **cooperative scheduling**: chia render thành chunks, sau mỗi chunk check xem có task ưu tiên cao hơn không (user interaction), nếu có thì yield.

Kích hoạt bằng `createRoot` (React 18) thay vì `ReactDOM.render` cũ. Các API concurrent: `useTransition`, `useDeferredValue`, `Suspense`.

```jsx
// Legacy sync - block
import ReactDOM from 'react-dom';
ReactDOM.render(<App />, container); // đồng bộ, block

// Concurrent - interruptible
import { createRoot } from 'react-dom/client';
const root = createRoot(document.getElementById('root'));
root.render(<App />); // concurrent, có thể interrupt

// Ví dụ: filter list nặng không block typing
function App() {
  const [query, setQuery] = React.useState("");
  const [isPending, startTransition] = React.useTransition();
  const [filtered, setFiltered] = React.useState(items);

  const onChange = e => {
    const value = e.target.value;
    setQuery(value); // urgent - update input ngay
    startTransition(() => {
      setFiltered(filterHugeList(value)); // non-urgent - có thể interrupt
    });
  };
  return (
    <>
      <input value={query} onChange={onChange} />
      {isPending ? <Spinner /> : <List items={filtered} />}
    </>
  );
}
```

**Trade-off:** Concurrent thêm complexity, component phải pure (render có thể gọi nhiều lần và bỏ). Nhưng UX mượt hơn rất nhiều với heavy computation.

**Câu hỏi đào sâu:** Concurrent khác gì với time-slicing? Vì sao `createRoot` mới bật concurrent còn `ReactDOM.render` thì không?

---

### Câu 53: React 18 - Những tính năng nổi bật

**Trả lời Senior:**
React 18 là bản nền cho Concurrent:

1.  **Automatic Batching:** batch mọi `setState` kể cả trong `setTimeout`, `Promise`, native event (trước chỉ batch trong React event). Giảm render thừa.
2.  **createRoot + StrictMode:** `createRoot` bật concurrent, `StrictMode` double-invoke effect để bắt bug.
3.  **Transitions (`useTransition`, `startTransition`):** đánh dấu update không khẩn (non-urgent), để React ưu tiên urgent update (typing, click).
4.  **Suspense cho SSR:** `Suspense` hỗ trợ streaming SSR, selective hydration.
5.  **New hooks:** `useId`, `useDeferredValue`, `useSyncExternalStore`, `useInsertionEffect`.
6.  **SuspenseList** (ít dùng) và cải thiện `hydrateRoot`.

```jsx
// Automatic Batching - React 18 batch cả async
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React 17: 2 renders, React 18: 1 render
}, 0);

// flushSync để opt-out batching khi cần đọc DOM ngay
import { flushSync } from 'react-dom';
flushSync(() => setCount(c => c + 1)); // render sync ngay

// useId cho accessibility, SSR-safe
function Input() {
  const id = React.useId();
  return <><label htmlFor={id}>Name</label><input id={id} /></>;
}

// useSyncExternalStore - thay thế manual subscription
const count = React.useSyncExternalStore(store.subscribe, store.getSnapshot);
```

Migration: thay `ReactDOM.render` bằng `createRoot`, check `findDOMNode` deprecated, test với `act` mới.

**Trade-off:** Automatic batching có thể làm `useEffect` chạy ít hơn (trước effect chạy mỗi setState), cần test kỹ. `flushSync` phá batching nên tránh.

**Câu hỏi đào sâu:** Automatic batching ảnh hưởng `useEffect` thế nào? `hydrateRoot` khác `createRoot` thế nào?

---

### Câu 54: React 19 - Actions, `use` và React Compiler

**Trả lời Senior:**
React 19 (stable 2024) tập trung vào **data fetching + mutations + compiler**:

- **Actions (`useTransition` + async):** `formAction`, `useActionState` (trước là `useFormState`), `useFormStatus` để xử lý form/mutation với pending, optimistic update, error handling tích hợp. Server Actions (Next.js) là 1 dạng action.
- **`use` API:** đọc Promise hoặc Context **trong render** (không chỉ top-level), cho phép `if (cond) use(...)` - linh hoạt hơn hooks. Dùng với Suspense.
- **React Compiler (React Forget):** auto-memoization - tự động `useMemo`/`useCallback`/`memo` nên không cần memo thủ công nữa, giảm 90% memo code. Hiện opt-in.
- **Document Metadata, Asset Loading, `ref` cleanup:** `<title>` trong component, `ref` as prop (không cần `forwardRef`), `useDeferredValue` initial value.
- **`useOptimistic`:** optimistic UI cho mutation.

```jsx
// useActionState
import { useActionState } from 'react';
async function updateName(prevState, formData) {
  const name = formData.get("name");
  if (!name) return { error: "Required" };
  await fetch("/api/user", { method: "POST", body: JSON.stringify({ name }) });
  return { success: true };
}
function NameForm() {
  const [state, formAction, isPending] = useActionState(updateName, null);
  return (
    <form action={formAction}>
      <input name="name" />
      <button disabled={isPending}>Save</button>
      {state?.error && <p>{state.error}</p>}
    </form>
  );
}

// use - đọc Promise trong render
import { use } from 'react';
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise); // suspend tới khi resolve
  return comments.map(c => <div key={c.id}>{c.text}</div>);
}
// Có thể condition
function Maybe({ shouldFetch, promise }) {
  if (shouldFetch) {
    const data = use(promise); // ok, use không bị rules of hooks hạn chế như hook thường
    return <div>{data}</div>;
  }
  return null;
}

// useOptimistic
const [optimisticTodos, addOptimistic] = React.useOptimistic(todos, (state, newTodo) => [...state, newTodo]);
```

**Trade-off:** `use` chỉ dùng trong render có Suspense boundary, không phải mọi nơi. Compiler tuyệt vời nhưng cần codemod và test kỹ, chưa bật mặc định cho mọi app.

**Câu hỏi đào sâu:** `use` khác `useEffect` + `await` thế nào? React Compiler thay thế `memo` thủ công ra sao?

---

### Câu 55: useTransition vs useDeferredValue

**Trả lời Senior:**
Cả hai đều để **de-prioritize** update không khẩn, giữ UI responsive khi render nặng, nhưng khác control:

- **useTransition:** cho phép **wrap setter** trong `startTransition(() => setState(...))`, trả `isPending` để hiện loading. Dùng khi bạn **control được state update** (onChange, click).
- **useDeferredValue:** **defer giá trị** - trả version deferred của value, React sẽ re-render với deferred value ở priority thấp. Dùng khi bạn **không control được update** (props từ parent, data từ lib).

```jsx
// useTransition - bạn control setState
function SearchWithTransition({ items }) {
  const [query, setQuery] = React.useState("");
  const [isPending, startTransition] = React.useTransition();
  const [filtered, setFiltered] = React.useState(items);
  const onChange = e => {
    setQuery(e.target.value); // urgent
    startTransition(() => setFiltered(filter(items, e.target.value))); // non-urgent
  };
  return (
    <>
      <input value={query} onChange={onChange} />
      {isPending && <Spinner />}
      <List items={filtered} />
    </>
  );
}

// useDeferredValue - bạn nhận value từ parent
function FilteredList({ query, items }) {
  const deferredQuery = React.useDeferredValue(query);
  const filtered = React.useMemo(() => filter(items, deferredQuery), [items, deferredQuery]);
  const isStale = query !== deferredQuery; // đang defer
  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <List items={filtered} />
    </div>
  );
}
// Parent chỉ cần <FilteredList query={query} items={items} /> mà không cần transition
```

`useDeferredValue` còn nhận `initialValue` trong React 19 để tránh flash.

**Trade-off:** `useTransition` cho `isPending` rõ ràng, `useDeferredValue` tiện hơn khi không muốn prop drilling `isPending`. Cả hai đều cần `memo` hoặc tách component để tránh re-render urgent path.

**Lỗi thường gặp:** Bọc urgent update trong transition (làm input lag), dùng `useDeferredValue` cho mọi state (overhead).

**Câu hỏi đào sâu:** Khi nào chọn `useTransition` thay vì `useDeferredValue`? `isPending` vs `query !== deferredQuery` khác gì?

---

### Câu 56: Suspense cho Data Fetching

**Trả lời Senior:**
Suspense cho phép component **"suspend"** (tạm dừng render) trong khi chờ data, và hiện `fallback`. Trước React 18, Suspense chỉ cho `React.lazy` (code splitting). Từ 18+, dùng cho data fetching với framework (Next.js, Relay) hoặc lib như `SWR`/`React Query` với `suspense: true`.

Cơ chế: component throw Promise trong render, React bắt và hiện fallback của `<Suspense>` gần nhất. Khi Promise resolve, React retry render.

```jsx
// Với React.lazy
const Comments = React.lazy(() => import('./Comments'));
<Suspense fallback={<Spinner />}><Comments /></Suspense>

// Data fetching với wrapper tạo resource (demo, không dùng trực tiếp)
function createResource(promise) {
  let status = "pending", result;
  const suspender = promise.then(
    r => { status = "success"; result = r; },
    e => { status = "error"; result = e; }
  );
  return {
    read() {
      if (status === "pending") throw suspender; // throw Promise -> Suspense bắt
      if (status === "error") throw result;
      return result;
    }
  };
}
const userResource = createResource(fetch("/api/user").then(r => r.json()));
function User() {
  const user = userResource.read(); // suspend nếu chưa xong
  return <div>{user.name}</div>;
}
<Suspense fallback={<Skeleton />}><User /></Suspense>

// Thực tế với React Query
import { useSuspenseQuery } from '@tanstack/react-query';
function Todos() {
  const { data } = useSuspenseQuery({ queryKey: ['todos'], queryFn: fetchTodos });
  return <List data={data} />;
}
// Hoặc Next.js: async Server Component tự suspend
async function Page() {
  const data = await fetch("https://api...", { cache: "force-cache" });
  return <div>{data.name}</div>;
}
```

Suspense còn lồng nhau, `SuspenseList` để control reveal order, và kết hợp streaming SSR.

**Trade-off:** Suspense làm code declarative hơn nhưng cần boundary đúng chỗ, nếu không fallback sẽ che cả cây lớn. Không dùng Suspense cho data fetching client-only mà không có framework hỗ trợ thì phức tạp.

**Câu hỏi đào sâu:** Suspense bắt Promise thế nào? Khác gì với `isLoading` thủ công? Làm sao handle error khi suspend?

---

### Câu 57: Error Boundary và xử lý lỗi toàn cục

**Trả lời Senior:**
Error Boundary là **class component** với `getDerivedStateFromError` và `componentDidCatch` để bắt lỗi **trong render, lifecycle, constructor của con**. Nó không bắt: event handler, async (`setTimeout`, `Promise`), SSR, và chính nó. Từ React 16, lỗi không bắt sẽ unmount cả cây.

Function component không làm Error Boundary được (phải dùng class hoặc lib `react-error-boundary`).

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }
  componentDidCatch(error, info) {
    console.error(error, info.componentStack);
    logToService(error, info);
  }
  reset = () => this.setState({ hasError: false, error: null });
  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <div>Something went wrong <button onClick={this.reset}>Retry</button></div>;
    }
    return this.props.children;
  }
}

// Dùng
<ErrorBoundary fallback={<ErrorPage />}>
  <App />
</ErrorBoundary>

// Với react-error-boundary
import { ErrorBoundary } from 'react-error-boundary';
<ErrorBoundary FallbackComponent={ErrorFallback} onReset={() => resetQuery()}>
  <Suspense fallback={<Spinner />}><Todos /></Suspense>
</ErrorBoundary>
```

Kết hợp với Suspense: Error Boundary bắt lỗi fetch, Suspense bắt loading. Global handler thêm `window.onerror` + `unhandledrejection` cho phần ngoài React.

**Trade-off:** Đặt boundary quá cao thì fallback che hết app, quá thấp thì nhiều boilerplate. Nên đặt quanh route/feature, không quanh từng button.

**Lỗi thường gặp:** Mong Error Boundary bắt async/event handler, quên reset state sau lỗi, dùng function component làm boundary.

**Câu hỏi đào sâu:** Vì sao Error Boundary phải là class? Làm sao bắt lỗi async/event handler?

---

### Câu 58: Server Components (RSC) vs Client Components

**Trả lời Senior:**
React Server Components (RSC) là component **chỉ chạy trên server**, không gửi JS lên client, không có state/effect, render ra serializable format (RSC payload) để client merge. Client Components là component truyền thống (có `"use client"`), chạy cả server (SSR) và client (hydrate).

Phân biệt:

- **RSC (mặc định trong Next.js App Router):** không có `useState`, `useEffect`, `onClick`, `window`. Được phép `async/await` trực tiếp, fetch data gần, giữ secret (API key). Bundle = 0 JS.
- **Client Components (`"use client"`):** có interactivity, state, browser API. Phải serializable props khi nhận từ RSC.

```jsx
// app/page.jsx - Server Component (mặc định)
async function ProductPage({ params }) {
  const product = await db.product.findUnique({ where: { id: params.id } }); // trực tiếp DB
  // Không có useState, onClick ở đây
  return (
    <div>
      <h1>{product.name}</h1>
      <AddToCart productId={product.id} /> {/* Client Component */}
    </div>
  );
}

// components/AddToCart.jsx - Client Component
"use client";
import { useState } from "react";
export default function AddToCart({ productId }) {
  const [pending, setPending] = useState(false);
  return <button onClick={() => setPending(true)}>{pending ? "..." : "Add"}</button>;
}

// Quy tắc: RSC import Client ok, Client import RSC không được (trừ children slot)
// Sai: Client Component import Server Component có async DB
// Đúng: truyền RSC như children
<ClientWrapper><ServerChild /></ClientWrapper>
```

Lợi ích: giảm JS gửi client, fetch waterfall giảm (RSC fetch song song trên server), SEO tốt.

**Trade-off:** RSC làm mental model phức tạp (2 môi trường), debug khó hơn, không phải mọi lib tương thích. Chỉ dùng khi framework hỗ trợ (Next.js).

**Câu hỏi đào sâu:** RSC khác SSR thế nào? Vì sao không dùng `useState` trong RSC? Props từ RSC sang Client phải serializable?

---

### Câu 59: Hydration, Selective Hydration và Hydration Error

**Trả lời Senior:**
Hydration là client React **attach event và state** vào HTML đã SSR, thay vì tạo DOM mới. React 18 có `hydrateRoot` và **Selective Hydration**: với streaming SSR + Suspense, phần nào đã stream xong sẽ hydrate trước, không cần đợi cả trang.

Hydration Error xảy ra khi HTML server và HTML client lần đầu khác nhau (mismatch), do `Date.now()`, `Math.random()`, `window.innerWidth`, locale, hoặc `useEffect` không guard.

```jsx
import { hydrateRoot } from 'react-dom/client';
hydrateRoot(document.getElementById('root'), <App />);

// Selective Hydration với Suspense + streaming
// Server stream: <Suspense fallback={<Skeleton />}><Comments /></Suspense>
// Comments chưa xong -> client hydrate phần khác trước, Comments hydrate khi stream xong

// Hydration mismatch - sai
function App() {
  return <div>{window.innerWidth}</div>; // server không có window -> mismatch
}
// Fix
function AppFixed() {
  const [width, setWidth] = React.useState(null);
  React.useEffect(() => setWidth(window.innerWidth), []); // chỉ client
  if (width === null) return <div>loading</div>;
  return <div>{width}</div>;
}

// Hoặc
<div suppressHydrationWarning>{new Date().toString()}</div> // tắt warn cho phần cố ý khác

// Debug
hydrateRoot(container, <App />, {
  onRecoverableError: (err, info) => console.error(err, info.componentStack)
});
```

Selective Hydration còn ưu tiên hydrate phần user đang tương tác (click vào Suspense fallback thì hydrate ngay).

**Trade-off:** Hydration nhanh hơn CSR nhưng nếu JS lớn, TTFI vẫn chậm. Streaming + selective giảm TTFI nhưng tăng complexity.

**Lỗi thường gặp:** Dùng `Math.random()` trong render, `useLayoutEffect` warning trên server, không `suppressHydrationWarning` đúng chỗ.

**Câu hỏi đào sâu:** Selective Hydration ưu tiên thế nào? Vì sao `window` trong render gây mismatch?

---

### Câu 60: SSR vs CSR vs SSG vs ISR

**Trả lời Senior:**
| Kiểu | Render ở đâu | Khi nào render | Ưu/Nhược |
|---|---|---|---|
| **CSR** | Browser | Mỗi request (JS fetch) | Đơn giản, TTFB nhanh, nhưng SEO kém, white flash, JS lớn |
| **SSR** | Server | Mỗi request | SEO tốt, FCP nhanh, nhưng TTFB chậm, server load cao, không offline |
| **SSG** | Build time | Một lần khi build | Nhanh nhất (CDN), SEO tốt, nhưng không real-time, build lâu nếu nhiều page |
| **ISR** | Build + background | Build + revalidate sau N giây | Kết hợp SSG + freshness, stale-while-revalidate, cần framework (Next.js) |

```javascript
// Next.js App Router
// SSR: dynamic rendering
export const dynamic = 'force-dynamic';
async function Page() {
  const data = await fetch('https://api...', { cache: 'no-store' });
  return <div>{data.title}</div>;
}

// SSG: static
export const dynamic = 'force-static';
async function Page() {
  const data = await fetch('https://api...', { cache: 'force-cache' });
  return <div>{data.title}</div>;
}

// ISR: revalidate 60s
export const revalidate = 60;
async function Page() {
  const data = await fetch('https://api...', { next: { revalidate: 60 } });
  return <div>{data.title}</div>;
}

// CSR: "use client" + useEffect fetch
"use client";
function Page() {
  const [data, setData] = React.useState(null);
  React.useEffect(() => { fetch('/api').then(r => r.json()).then(setData); }, []);
}
```

Chọn: marketing/blog -> SSG/ISR, dashboard user-specific -> CSR/SSR, e-commerce listing -> ISR, detail cần SEO -> SSR/SSG.

**Trade-off:** SSR tốn server, SSG không real-time, ISR phức tạp cache invalidation, CSR SEO kém.

**Câu hỏi đào sâu:** Khi nào chọn ISR thay vì SSR? `cache: 'no-store'` vs `revalidate` khác gì?

---

### Câu 61: Streaming SSR và Suspense

**Trả lời Senior:**
Streaming SSR cho phép server **gửi HTML từng phần ngay khi xong**, không đợi hết data, bằng `renderToPipeableStream` (Node) hoặc `renderToReadableStream` (Edge). Kết hợp Suspense, phần chưa ready sẽ gửi `fallback`, khi ready sẽ stream thêm HTML + inline script để hydrate.

Lợi ích: TTFB nhanh, FCP sớm, không waterfall.

```jsx
// Server (Next.js tự làm, demo với react-dom/server)
import { renderToPipeableStream } from 'react-dom/server';
app.get('/', (req, res) => {
  const { pipe } = renderToPipeableStream(
    <App />,
    {
      bootstrapScripts: ['/main.js'],
      onShellReady() { res.statusCode = 200; pipe(res); }, // gửi shell ngay
      onAllReady() { /* hết */ }
    }
  );
});

// App với Suspense
function App() {
  return (
    <Layout>
      <Suspense fallback={<HeaderSkeleton />}><Header /></Suspense>
      <Suspense fallback={<ProductSkeleton />}><Product /></Suspense>
      <Suspense fallback={<CommentsSkeleton />}><Comments /></Suspense> {/* chậm */}
    </Layout>
  );
}
// Server gửi: Layout + Header + Product + CommentsSkeleton ngay
// Khi Comments xong -> stream thêm HTML Comments + script để hydrate

// Next.js App Router: async Server Component tự streaming
async function Comments() {
  const comments = await fetch('https://api/comments', { cache: 'no-store' }).then(r => r.json());
  return comments.map(c => <div key={c.id}>{c.text}</div>);
}
// Page bọc tự động streaming nếu có Suspense
import { Suspense } from 'react';
export default function Page() {
  return <Suspense fallback={<Skeleton />}><Comments /></Suspense>;
}
```

Selective Hydration: client ưu tiên hydrate phần đã stream, không block bởi phần chậm.

**Trade-off:** Streaming phức tạp hơn `renderToString` (phải handle backpressure, error boundary). Nhưng cải thiện TTFB 20-40% với page có data chậm.

**Câu hỏi đào sâu:** `renderToString` vs `renderToPipeableStream` khác gì? Streaming ảnh hưởng SEO không?

---

### Câu 62: Code Splitting, lazy và dynamic import

**Trả lời Senior:**
Code splitting chia bundle lớn thành chunks nhỏ, load khi cần, giảm JS ban đầu. Webpack/Rollup/Vite tự split với `import()` dynamic. React có `React.lazy` + `Suspense` cho component, `loadable-components` cho SSR tốt hơn.

```jsx
// Route-based splitting (khuyến nghị)
import { lazy, Suspense } from 'react';
const Home = lazy(() => import('./Home'));
const Dashboard = lazy(() => import('./Dashboard'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}

// Component-based + prefetch
const HeavyChart = lazy(() => import('./HeavyChart'));
function Page() {
  const [show, setShow] = React.useState(false);
  return (
    <>
      <button
        onMouseEnter={() => import('./HeavyChart')} // prefetch khi hover
        onClick={() => setShow(true)}
      >Show Chart</button>
      {show && <Suspense fallback={<Skeleton />}><HeavyChart /></Suspense>}
    </>
  );
}

// Next.js dynamic
import dynamic from 'next/dynamic';
const Map = dynamic(() => import('../components/Map'), {
  ssr: false, // không SSR, chỉ client
  loading: () => <Skeleton />
});

// Named export với lazy
const Named = lazy(() => import('./utils').then(m => ({ default: m.Named })));

// Vite glob
const modules = import.meta.glob('./pages/*.jsx'); // lazy tất cả
```

Đo bundle: `webpack-bundle-analyzer`, `vite --analyze`, Lighthouse.

**Trade-off:** Split quá nhỏ gây nhiều request (waterfall), quá lớn thì không giảm TTFB. Prefetch hợp lý, tránh split cho component nhỏ (<5kb).

**Lỗi thường gặp:** Quên `Suspense` bọc `lazy` -> crash, `lazy` chỉ cho default export, dynamic import trong SSR không handle fallback.

**Câu hỏi đào sâu:** `React.lazy` khác `dynamic import` thường thế nào? Prefetch vs preload khác gì?

---

### Câu 63: Performance Debugging - Profiler và DevTools

**Trả lời Senior:**
Tối ưu phải đo trước. Tools:

1.  **React DevTools Profiler:** record commit, xem flame graph, ranked chart, component nào render lâu, vì sao render (props/state/context đổi).
2.  **Chrome Performance tab:** xem scripting, rendering, painting, long task, layout thrashing.
3.  **why-did-you-render / whyDidYouRender:** log khi component re-render dù props bằng nhau (phát hiện memo thiếu).
4.  **Lighthouse / Web Vitals:** LCP, INP, CLS.
5.  **`console.log` + `React.memo` custom comparator** để trace.

```jsx
// Profiler API
import { Profiler } from 'react';
function onRender(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  console.log(`${id} ${phase} actual:${actualDuration} base:${baseDuration}`);
}
<Profiler id="List" onRender={onRender}><List items={items} /></Profiler>

// why-did-you-render setup
import whyDidYouRender from '@welldone-software/why-did-you-render';
if (process.env.NODE_ENV === 'development') {
  whyDidYouRender(React, { trackAllPureComponents: true });
}

// DevTools highlight update
// React DevTools -> Settings -> Highlight updates when components render

// Performance mark
performance.mark('start');
expensiveCalc();
performance.mark('end');
performance.measure('calc', 'start', 'end');
console.log(performance.getEntriesByName('calc'));
```

Workflow Senior: Record Profiler -> tìm component render nhiều/lâu -> check why rendered (props mới? context? parent?) -> fix bằng memo/colocation/split -> measure lại.

**Trade-off:** Profiler overhead, chỉ đo dev, production cần `Profiler` với `onRender` gửi analytics. `why-did-you-render` noisy nếu bật hết.

**Câu hỏi đào sâu:** Flame graph vs Ranked chart khác gì? Làm sao phát hiện wasted render?

---

### Câu 64: Chiến lược tối ưu Re-render toàn diện

**Trả lời Senior:**
Tối ưu re-render là hệ thống, không phải chỉ `memo`:

1.  **Colocation:** đặt state gần nơi dùng, tránh lift lên App.
2.  **Composition + children:** truyền children như slot để parent render không làm con render.
3.  **Memoization có chọn lọc:** `React.memo`, `useMemo`, `useCallback` chỉ khi profiler chỉ ra.
4.  **Tách context:** split context value/setter, dùng selector.
5.  **List virtualization:** `react-window`/`tanstack-virtual` cho list 1000+ item, chỉ render viewport.
6.  **Debounce/defer:** `useDeferredValue`, `useTransition` cho input filter nặng.
7.  **State lib với selector:** Zustand/Jotai chỉ re-render khi slice đổi.
8.  **React Compiler:** auto memo.

```jsx
// Anti-pattern: App state làm cả tree render
function App() {
  const [query, setQuery] = React.useState("");
  return <><Search query={query} onChange={setQuery} /><HugeList query={query} /></>;
}
// Fix: tách, dùng deferred
function AppFixed() {
  const [query, setQuery] = React.useState("");
  const deferredQuery = React.useDeferredValue(query);
  return <><Search query={query} onChange={setQuery} /><HugeList query={deferredQuery} /></>;
}

// Composition tránh re-render
function App2({ children }) {
  const [count, setCount] = React.useState(0);
  return <div><button onClick={() => setCount(c => c + 1)}>{count}</button>{children}</div>;
  // children không re-render khi count đổi nếu children là element đã tạo ngoài
}
// Dùng: <App2><ExpensiveTree /></App2> -> ExpensiveTree không render lại

// Virtualization
import { useVirtualizer } from '@tanstack/react-virtual';
function VirtualList({ items }) {
  const parentRef = React.useRef(null);
  const virtualizer = useVirtualizer({ count: items.length, getScrollElement: () => parentRef.current, estimateSize: () => 35 });
  return <div ref={parentRef}>{virtualizer.getVirtualItems().map(virtualRow => <div key={virtualRow.key}>{items[virtualRow.index]}</div>)}</div>;
}
```

Đo trước/sau bằng Profiler, đừng optimize sớm.

**Trade-off:** Mỗi technique thêm complexity, chỉ áp dụng khi có vấn đề.

**Câu hỏi đào sâu:** Khi nào `children` giúp tránh re-render? Virtualization hoạt động thế nào?

---

### Câu 65: Concurrent Features - useId và useSyncExternalStore

**Trả lời Senior:**
- **useId:** tạo id **ổn định, unique, SSR-safe** cho accessibility (`label`/`input`, `aria-describedby`). Không dùng cho key list (vì không stable giữa render nếu list reorder). Format `":r0:"`, tự prefix với server.

- **useSyncExternalStore:** hook chuẩn để **subscribe tới external store** (Redux, Zustand, `window.matchMedia`, `localStorage`) mà concurrent-safe, tránh tearing (UI hiện 2 giá trị khác nhau trong 1 render). Thay thế `useEffect` + `subscribe` thủ công.

```jsx
// useId
function Input({ label }) {
  const id = React.useId();
  return <><label htmlFor={id}>{label}</label><input id={id} /></>;
}
// SSR: server và client tạo cùng id, không mismatch

// Sai: dùng useId cho key
{items.map(item => <div key={useId()} />)} // sai, mỗi render id khác

// useSyncExternalStore
import { useSyncExternalStore } from 'react';
function useOnline() {
  return useSyncExternalStore(
    callback => {
      window.addEventListener('online', callback);
      window.addEventListener('offline', callback);
      return () => {
        window.removeEventListener('online', callback);
        window.removeEventListener('offline', callback);
      };
    },
    () => navigator.onLine, // getSnapshot client
    () => true // getServerSnapshot cho SSR
  );
}
function Status() {
  const online = useOnline();
  return <div>{online ? "Online" : "Offline"}</div>;
}

// Với Redux/Zustand: họ đã dùng useSyncExternalStore bên trong
const value = useSyncExternalStore(store.subscribe, store.getSnapshot, store.getServerSnapshot);
```

Ngoài ra `useInsertionEffect` dành cho CSS-in-JS inject style trước layout.

**Trade-off:** `useId` chỉ cho id, không cho random. `useSyncExternalStore` bắt buộc `getSnapshot` phải pure và cache.

**Câu hỏi đào sâu:** Tearing là gì và vì sao `useSyncExternalStore` fix được? `useId` khác `Math.random()` thế nào?

---

### Câu 66: Server Actions và Mutations trong Next.js

**Trả lời Senior:**
Server Actions là **function chạy trên server** được gọi trực tiếp từ Client Component qua RPC, không cần API route thủ công. Định nghĩa với `"use server"`, dùng cho mutations (create, update, delete), revalidate cache, redirect.

Kết hợp với `useActionState`, `useFormStatus`, `useOptimistic` cho UX pending/optimistic.

```jsx
// app/actions.js - Server Action
"use server";
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createTodo(prevState, formData) {
  const title = formData.get("title");
  if (!title) return { error: "Title required" };
  await db.todo.create({ data: { title } });
  revalidatePath("/todos"); // xóa cache ISR
  // redirect("/todos");
  return { success: true };
}

// app/todos/page.jsx - Client dùng action
"use client";
import { useActionState } from 'react';
import { createTodo } from '../actions';

function TodoForm() {
  const [state, formAction, isPending] = useActionState(createTodo, null);
  return (
    <form action={formAction}>
      <input name="title" />
      <button disabled={isPending}>{isPending ? "Saving..." : "Add"}</button>
      {state?.error && <p>{state.error}</p>}
    </form>
  );
}

// useOptimistic cho optimistic update
"use client";
import { useOptimistic } from 'react';
function Todos({ todos }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(todos, (state, newTodo) => [...state, newTodo]);
  async function action(formData) {
    const title = formData.get("title");
    addOptimistic({ id: Date.now(), title, pending: true });
    await createTodo(null, formData);
  }
  return (
    <>
      {optimisticTodos.map(t => <div key={t.id} style={{ opacity: t.pending ? 0.5 : 1 }}>{t.title}</div>)}
      <form action={action}><input name="title" /><button>Add</button></form>
    </>
  );
}
```

Server Actions tự handle CSRF, progressive enhancement (form vẫn chạy khi JS tắt).

**Trade-off:** Chỉ trong framework hỗ trợ (Next.js), không phải SPA thuần. Cần validate input kỹ vì gọi từ client.

**Câu hỏi đào sâu:** Server Actions khác API Route thế nào? `revalidatePath` vs `revalidateTag` khác gì?

---

### Câu 67: Caching Strategy - React cache, fetch cache và Next.js cache

**Trả lời Senior:**
Next.js có 4 layer cache (dễ nhầm):

1.  **React `cache` (Request Memoization):** dedupe `fetch`/`cache()` cùng URL trong **1 request** (1 render). `fetch("/api/user")` gọi 5 lần trong tree vẫn chỉ fetch 1 lần.
2.  **Data Cache (fetch cache):** persist giữa requests, lưu trên server, control bằng `fetch(..., { cache: 'force-cache' | 'no-store', next: { revalidate: 60 } })`.
3.  **Full Route Cache (Static):** cache HTML/RSC payload của route đã SSG/ISR trên server/CDN.
4.  **Router Cache (Client):** Next.js App Router cache RSC payload trên client (memory), khi navigate back thì không fetch lại, control bằng `router.refresh()`.

```javascript
// 1. Request Memoization - tự động
async function getUser(id) {
  const res = await fetch(`https://api/user/${id}`); // dù gọi ở 3 component, chỉ 1 fetch/request
  return res.json();
}

// 2. Data Cache
fetch('https://api/posts', { cache: 'force-cache' }); // SSG - cache vĩnh viễn tới khi revalidate
fetch('https://api/posts', { cache: 'no-store' }); // SSR - không cache
fetch('https://api/posts', { next: { revalidate: 60 } }); // ISR - cache 60s
fetch('https://api/posts', { next: { tags: ['posts'] } }); // on-demand revalidate

// Revalidate
import { revalidateTag, revalidatePath } from 'next/cache';
revalidateTag('posts'); // xóa Data Cache tag posts
revalidatePath('/posts'); // xóa Full Route Cache

// 3. React cache cho non-fetch
import { cache } from 'react';
export const getItem = cache(async id => {
  return db.item.findUnique({ where: { id } });
});
// getItem(1) gọi nhiều lần trong request chỉ query DB 1 lần

// 4. Router Cache - client
// <Link href="/posts"> tự prefetch và cache
// router.refresh() để xóa Router Cache và re-fetch
```

**Trade-off:** Cache nhiều layer dễ stale, phải revalidate đúng. `no-store` cho dynamic data, `force-cache` cho static.

**Lỗi thường gặp:** Nghĩ `fetch` không cache (mặc định Next.js cache), quên `revalidateTag` nên data cũ, dùng `cache` cho function có side effect.

**Câu hỏi đào sâu:** Request Memoization khác Data Cache thế nào? Khi nào dùng `revalidateTag` vs `revalidatePath`?

---

### Câu 68: Tư duy thiết kế hệ thống Frontend với React ở scale lớn

**Trả lời Senior:**
Ở scale lớn (hàng chục dev, micro-frontend, design system), tư duy chuyển từ "component" sang **hệ thống**:

1.  **Kiến trúc:** Feature-sliced (app/pages/widgets/features/entities/shared), module boundaries rõ, dependency rule (không import ngược). Monorepo với Turborepo/Nx, package versioning.
2.  **Contract:** Design System + Storybook, token, a11y, visual regression. API contract với backend (OpenAPI, tRPC, GraphQL codegen).
3.  **State:** phân loại - Server State (React Query/RTK Query) vs Client State (Zustand/Jotai) vs URL State vs Form State. Không đưa server state vào Redux.
4.  **Performance:** budget (JS < 200kb), code splitting, edge caching, observability (Sentry, Datadog, Web Vitals).
5.  **Quality:** ESLint + type strict, test pyramid (unit với Vitest, integration với Testing Library, e2e với Playwright), CI/CD với preview deploy.
6.  **Team:** ADR, code generation (plop, hygen), conventional commits, release train.

```javascript
// Feature-sliced example
// src/
//   app/ (router, providers)
//   pages/ (route composition)
//   widgets/ (Header, Sidebar - compose features)
//   features/ (auth, cart, search - business logic + UI)
//   entities/ (user, product - domain model)
//   shared/ (ui, api, lib, config)

// API layer với tRPC - type-safe
// server/routers/user.ts
export const userRouter = router({
  getById: publicProcedure.input(z.object({ id: z.string() })).query(({ input }) => db.user.findUnique(input))
});
// client: const { data } = trpc.user.getById.useQuery({ id: "1" }); // typed

// Performance budget trong CI
// bundlesize.config.json
{ "path": "./build/*.js", "maxSize": "200 kB" }
```

Nguyên tắc Senior: **optimize for change** - code dễ thay đổi hơn code chạy nhanh nhất. Đầu tư vào tooling để dev experience tốt, vì DX * số dev = productivity.

**Trade-off:** Over-engineering cho app nhỏ sẽ chậm. Scale lớn mới cần FSD, micro-frontend; app vừa thì colocation + simple store đủ.

**Câu hỏi đào sâu:** Khi nào cần micro-frontend? Làm sao phân biệt Server State vs Client State? Feature-sliced khác Clean Architecture thế nào?

---
