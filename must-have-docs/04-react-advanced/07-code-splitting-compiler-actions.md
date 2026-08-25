# Code Splitting, React Compiler và Server Actions — lazy/Suspense, next/dynamic, useActionState

> Tags: #react #code-splitting #lazy #suspense #compiler #server-actions #nextjs | Nguồn: `docs/03-react-nang-cao.md` câu 54, 62, 66 + `docs/11-nextjs.md` câu 177, 185 | Mức: P1

## 1. Định nghĩa chính xác

- **Code Splitting** là chia bundle lớn thành **chunks nhỏ** load khi cần, giảm JS ban đầu (initial JS), cải thiện LCP/TTI. Bundler (Webpack/Rollup/Vite) tự split khi gặp `import()` dynamic. React hỗ trợ `React.lazy(() => import('./Comp'))` + `<Suspense>` cho component.
- **`React.lazy`**: wrapper cho `import()` trả `Promise<{ default: Component }>`, chỉ cho **default export**. Khi render, throw Promise để Suspense bắt.
- **`next/dynamic`**: wrapper của Next.js cho code splitting, hỗ trợ thêm `ssr: false` (chỉ client, không SSR), `loading`, và `dynamic(import(...), { ssr: false })` cho lib browser-only (map, chart).
- **Prefetch/Preload**: `onMouseEnter={() => import('./Heavy')}` (prefetch khi hover) hoặc `<Link prefetch>` của Next.js để tải chunk trước khi cần.
- **Server Actions**: function **chạy trên server** (`"use server"`) được gọi trực tiếp từ Client Component qua RPC, không cần API route thủ công, dùng cho mutations (`create`, `update`, `delete`), kèm `revalidateTag`/`revalidatePath`.
- **`useActionState` (trước là `useFormState`), `useFormStatus`, `useOptimistic`**: hooks React 19 để xử lý pending/optimistic/error của Server Actions trong form.
- **React Compiler (React Forget, beta 2024-2026)**: **auto-memoization** — compiler tự chèn `useMemo`/`useCallback`/`memo` tương đương nên giảm 50-70% memo thủ công, opt-in qua `babel-plugin-react-compiler` (chưa bật mặc định). `eslint-plugin-react-compiler` chỉ lint vi phạm rule.

## 2. Cơ chế hoạt động

### 2.1 Code Splitting với import() và lazy

```
Bundle ban đầu: main.js (200kb)
import('./HeavyChart') → HeavyChart.chunk.js (150kb) — chỉ tải khi <HeavyChart /> render

React.lazy(() => import('./HeavyChart')) → tạo LazyComponent
Khi render LazyComponent → throw Promise (import) → Suspense fallback
Khi chunk tải xong → resolve → retry render → hiện HeavyChart
```

- Webpack/Vite tự split tại `import()` — mỗi `import()` là một chunk. Route-based splitting (khuyến nghị) cho mỗi route một chunk; component-based cho heavy widget.
- `React.lazy` chỉ support **default export**. Với named export: `lazy(() => import('./utils').then(m => ({ default: m.Named })))`.

### 2.2 next/dynamic — SSR control

```js
const Map = dynamic(() => import('../components/Map'), {
  ssr: false, // không SSR, chỉ client (vì dùng window)
  loading: () => <Skeleton />,
});
```

- `ssr: false` bỏ component khỏi SSR, tránh `window is not defined` và không gửi HTML. Tốt cho map, chart, editor.
- `ssr: true` (mặc định) vẫn SSR + hydrate như `lazy`.

### 2.3 Prefetch — khi nào tải trước

- **Hover prefetch**: `onMouseEnter={() => import('./HeavyChart')}` — khi user hover nút "Show Chart", chunk bắt đầu tải, khi click đã sẵn sàng.
- **Next.js Link prefetch**: `<Link href="/dashboard" prefetch>` tự prefetch RSC + JS của route.
- **Webpack magic comment**: `import(/* webpackPrefetch: true */ './Heavy')` hint bundler prefetch idle.

### 2.4 Server Actions — RPC

```
Client Component ("use client")
  → import { createTodo } from '@/app/actions' ("use server")
  → <form action={createTodo}> hoặc onClick={() => createTodo(formData)}
  → Next serialize FormData/args, POST tới server, chạy function trên server (có cookies, DB)
  → revalidateTag/path, redirect, trả result về client
```

- Định nghĩa: file hoặc function có `"use server"` ở đầu file hoặc đầu function.
- Tự handle **CSRF**, **progressive enhancement** (form vẫn chạy khi JS tắt), và có `cookies()`/`headers()`.
- Khác Route Handler (`app/api/route.ts`): Route Handler cho public API/webhook, Server Actions cho mutation nội bộ gọn hơn.

### 2.5 useActionState / useOptimistic / useFormStatus

```
useActionState(action, initialState) → [state, formAction, isPending]
  — bọc Server Action, trả state (error/success) và isPending để disable

useFormStatus() → { pending, data, method, action }
  — chỉ dùng trong children của <form>, để biết pending của form cha

useOptimistic(state, (state, newValue) => newState) → [optimisticState, addOptimistic]
  — hiện optimistic UI ngay trước khi server trả về
```

### 2.6 React Compiler — auto memo

- Compiler phân tích component và tự memoize values/closures như `useMemo`/`useCallback` thủ công.
- Trước: `const filtered = useMemo(() => items.filter(...), [items, query])` thủ công, dễ quên deps.
- Sau Compiler (beta): không cần `useMemo` cho nhiều case, Compiler tự chèn. Giảm ~50-70% memo code.
- Bật: `babel-plugin-react-compiler` trong `babel.config`/`next.config`, chưa mặc định 2026. Cần codemod và test kỹ, chưa obsolete hoàn toàn memo thủ công.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 React.lazy + Suspense — route-based splitting (khuyến nghị)
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./Home'));
const Dashboard = lazy(() => import('./Dashboard'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading…</div>}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

// 3.2 Component-based + prefetch khi hover
import { lazy, Suspense, useState } from 'react';

const HeavyChart = lazy(() => import('./HeavyChart'));

function ChartPage() {
  const [show, setShow] = useState(false);
  return (
    <div>
      <button
        onMouseEnter={() => import('./HeavyChart')} // prefetch khi hover, không render
        onClick={() => setShow(true)}
      >
        Show Chart
      </button>
      {show && (
        <Suspense fallback={<div>Loading chart…</div>}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}

// 3.3 Named export với lazy
const Named = lazy(() => import('./utils').then(m => ({ default: m.NamedChart })));
// utils.tsx: export function NamedChart() { ... }

// 3.4 next/dynamic — ssr:false cho browser-only
import dynamic from 'next/dynamic';

const Map = dynamic(() => import('../components/Map'), {
  ssr: false, // không SSR, chỉ client (tránh window is not defined)
  loading: () => <div>Loading map…</div>,
});

const Editor = dynamic(() => import('../components/Editor'), {
  ssr: false,
  loading: () => <div>Loading editor…</div>,
});

function Page() {
  return (
    <div>
      <h1>Location</h1>
      <Map />
    </div>
  );
}

// 3.5 Vite glob — lazy tất cả pages
const modules = import.meta.glob('./pages/*.tsx'); // { './pages/Home.tsx': () => import(...) }

// 3.6 Server Actions + useActionState + useFormStatus
// app/actions.ts
'use server';
import { revalidatePath, revalidateTag } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createTodo(prevState: any, formData: FormData) {
  const title = formData.get('title') as string;
  if (!title) return { error: 'Title required' };
  // chạy trên server, có DB, cookies
  // await db.todo.create({ data: { title } });
  revalidateTag('todos');
  // redirect('/todos');
  return { success: true };
}

// components/TodoForm.tsx
'use client';
import { useActionState } from 'react';
import { createTodo } from '@/app/actions';
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus(); // chỉ dùng trong <form>
  return <button disabled={pending}>{pending ? 'Saving…' : 'Add'}</button>;
}

export function TodoForm() {
  const [state, formAction, isPending] = useActionState(createTodo, null);
  return (
    <form action={formAction}>
      <input name="title" placeholder="Todo" />
      <SubmitButton />
      {/* isPending từ useActionState, pending từ useFormStatus — tương đương */}
      {state?.error && <p role="alert">{state.error}</p>}
      {state?.success && <p>Created!</p>}
    </form>
  );
}

// 3.7 useOptimistic — optimistic UI
'use client';
import { useOptimistic } from 'react';
import { createTodo } from '@/app/actions';

type Todo = { id: number; title: string; pending?: boolean };

export function OptimisticTodos({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newTodo: Todo) => [...state, newTodo]
  );

  async function action(formData: FormData) {
    const title = formData.get('title') as string;
    addOptimistic({ id: Date.now(), title, pending: true }); // hiện ngay, mờ
    await createTodo(null, formData); // server
  }

  return (
    <>
      <ul>
        {optimisticTodos.map(t => (
          <li key={t.id} style={{ opacity: t.pending ? 0.5 : 1 }}>{t.title}</li>
        ))}
      </ul>
      <form action={action}>
        <input name="title" />
        <button>Add</button>
      </form>
    </>
  );
}

// 3.8 Gọi Server Action trực tiếp từ onClick (không cần form)
'use client';
import { addToCart } from '@/app/actions';
export function CartButton({ id }: { id: string }) {
  return <button onClick={async () => { await addToCart(id); }}>Add to cart</button>;
}
// app/actions.ts
// 'use server';
// export async function addToCart(productId: string) {
//   await db.cart.create({ data: { productId } });
//   revalidatePath('/cart');
//   return { success: true };
// }

// 3.9 React Compiler — trước và sau
// Trước Compiler — memo thủ công
function Before({ items, query }: { items: string[]; query: string }) {
  const filtered = React.useMemo(() => items.filter(s => s.includes(query)), [items, query]);
  const onClick = React.useCallback(() => console.log(query), [query]);
  return <List items={filtered} onClick={onClick} />;
}

// Sau Compiler (beta) — không cần memo thủ công, compiler tự chèn
// babel.config: plugins: [['babel-plugin-react-compiler', {}]]
function After({ items, query }: { items: string[]; query: string }) {
  const filtered = items.filter(s => s.includes(query)); // compiler tự memo
  const onClick = () => console.log(query); // compiler tự memo
  return <List items={filtered} onClick={onClick} />;
}
```

## 4. So sánh / Phân loại

| Tiêu chí | `React.lazy` + `Suspense` | `next/dynamic` | `import()` thường |
|----------|---------------------------|----------------|-------------------|
| API | `lazy(() => import('./Comp'))` | `dynamic(() => import('./Comp'), { ssr, loading })` | `import('./utils')` trong handler |
| Hỗ trợ | Chỉ `default export` | Cả hai + `ssr:false` | Mọi export |
| SSR | Có (SSR + hydrate) | Có, hoặc `ssr:false` bỏ SSR | Không tự động |
| Fallback | `<Suspense fallback>` | `loading` prop | Tự quản lý |
| Dùng cho | Route/component split thuần React | Next.js, lib browser-only | Prefetch, utils |

| Tiêu chí | Route-based splitting | Component-based splitting |
|----------|----------------------|---------------------------|
| Chunk | Mỗi route 1 chunk | Mỗi heavy widget 1 chunk |
| Khi dùng | Mặc định, giảm JS ban đầu tốt nhất | Chart, map, editor nặng |
| Ví dụ | `Home`, `Dashboard` | `HeavyChart`, `MonacoEditor` |

| Tiêu chí | Server Actions (`"use server"`) | Route Handler (`app/api/route.ts`) |
|----------|---------------------------------|------------------------------------|
| Gọi | `await action(formData)` (RPC) | `fetch('/api/...')` |
| Form no-JS | ✅ Có (progressive enhancement) | ❌ Không |
| Revalidate | `revalidateTag/Path` trực tiếp | Tự làm |
| Dùng cho | Mutation nội bộ (createTodo, login) | Public API, webhook (Stripe), mobile |

| Hook | Trả gì | Dùng khi |
|------|--------|----------|
| `useActionState(action, init)` | `[state, formAction, isPending]` | Bọc Server Action trong form, cần `isPending` + `state.error` |
| `useFormStatus()` | `{ pending, data, method, action }` | Trong `children` của `<form>`, để disable SubmitButton |
| `useOptimistic(state, reducer)` | `[optimisticState, addOptimistic]` | Hiện optimistic UI ngay trước khi server xong |

| Trước Compiler | Sau Compiler (beta) |
|----------------|---------------------|
| `useMemo`/`useCallback`/`memo` thủ công | Compiler tự chèn, giảm 50-70% memo code |
| Dễ quên deps, stale closure | Compiler tự phân tích deps |
| Cần `eslint-plugin-react-compiler` để lint | Vẫn cần test, chưa bật mặc định 2026 |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không code-splitting cho component nhỏ (<5kb)**: Split quá nhỏ tạo nhiều request waterfall, overhead HTTP, không giảm TTFB đáng kể. Chỉ split cho heavy (>20kb) hoặc route.
- **Không quên `<Suspense>` bọc `lazy`**: `lazy` throw Promise, không có Suspense → crash `A component suspended...`. Mọi `lazy` phải trong Suspense.
- **Không dùng `ssr:false` cho content cần SEO**: `ssr:false` không gửi HTML, bot không thấy. Chỉ cho widget client-only (map, chart).
- **Không split quá nhiều gây waterfall**: 10 chunks nhỏ tải tuần tự chậm hơn 2 chunks vừa. Dùng `prefetch` hợp lý, và `webpack-bundle-analyzer` để chọn ngưỡng.
- **Không dùng Server Actions cho public API**: Mobile app, webhook, external service cần REST (`app/api`). Server Actions chỉ cho mutation nội bộ Next.js.
- **Không bỏ validation trong Server Actions**: Args từ client, có thể bị giả mạo → phải `zod` validate, check auth (`cookies()`), rate limit.
- **Không mong Compiler thay hết memo ngay**: Compiler beta, chưa bật mặc định, cần `babel-plugin-react-compiler` và test regression. Với lib chưa compatible, vẫn cần `memo` thủ công. Dùng `eslint-plugin-react-compiler` để báo vi phạm rule (mutate props...).
- **Khi nào KHÔNG dùng optimistic**: Mutation có side effect nặng (payment) không nên optimistic (hiện tiền trừ trước khi chắc chắn). Chỉ optimistic cho UI reversible (like, todo, cart).

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Quên `<Suspense>` bọc `lazy`**
  - Triệu chứng: Crash `A component suspended while responding...`
  - Fix: Bọc `lazy` trong `<Suspense fallback>`. Với `next/dynamic`, đã có `loading`.
  - Đo: React DevTools → suspended component.

- **Lỗi 2: `lazy` với named export không wrapper**
  - Triệu chứng: `Element type is invalid...`
  - Fix: `lazy(() => import('./utils').then(m => ({ default: m.Named })))`.
  - Đo: Console error, check `export default`.

- **Lỗi 3: `ssr:false` cho content cần SEO**
  - Triệu chứng: View source không có HTML của Map/Article, SEO tụt.
  - Fix: Chỉ `ssr:false` cho widget không cần SEO. Content chính để SSR.
  - Đo: `curl` view source, Lighthouse SEO.

- **Lỗi 4: Prefetch quá nhiều → tốn băng thông**
  - Triệu chứng: Network tải 10 chunks ngay khi hover, mobile tốn data.
  - Fix: Chỉ prefetch cho next likely (hover, viewport). Dùng `prefetch={false}` cho ít dùng.
  - Đo: Chrome Network → JS chunks tải trước khi cần, `coverage` tab.

- **Lỗi 5: Server Actions không validate → bảo mật**
  - Triệu chứng: Client gửi `formData` giả, tạo data bậy.
  - Fix: Trong Server Action, `zod` parse, check `cookies().get('token')`, throw nếu không auth.
  - Đo: Test với `curl` POST trực tiếp.

- **Lỗi 6: `useOptimistic` không rollback khi lỗi**
  - Triệu chứng: Optimistic hiện item nhưng server lỗi, item vẫn ở đó.
  - Fix: `useOptimistic` tự rollback khi action throw/reject nếu không update state thật; hoặc tự `revalidate` và handle `state.error`.
  - Đo: Thử server lỗi, xem optimistic có biến mất không.

- **Lỗi 7: Compiler mutate props → không memo được**
  - Triệu chứng: `eslint-plugin-react-compiler` báo `Cannot mutate props...`
  - Fix: Không mutate `props.items.push()`, dùng immutable `[...items, newItem]`.
  - Đo: ESLint, `babel-plugin-react-compiler` log.

- **Công cụ**:
  - **`webpack-bundle-analyzer` / `vite --analyze` / `next-bundle-analyzer`**: Xem chunk size, split hợp lý không.
  - **Lighthouse / `coverage`**: JS unused, LCP, TTI trước/sau split.
  - **React DevTools Profiler**: `lazy` component mount time.
  - **`eslint-plugin-react-compiler`**: Lint vi phạm Compiler rule.

## 7. Câu hỏi tự kiểm tra

1. `React.lazy` + `Suspense` vs `next/dynamic` vs `import()` thường khác gì về `default export`, `ssr:false`, và khi nào dùng prefetch `onMouseEnter(() => import(...))`?
2. Server Actions khác Route Handler khi nào, và `useActionState` / `useFormStatus` / `useOptimistic` mỗi cái trả gì và dùng khi nào?
3. React Compiler (beta) làm gì với `useMemo`/`useCallback`/`memo` thủ công, vì sao chưa bật mặc định 2026 và `eslint-plugin-react-compiler` để làm gì?

<details>
<summary>Đáp án 30s</summary>

1. **`React.lazy`** chỉ cho `default export` (`lazy(() => import('./Comp'))`), phải bọc `<Suspense fallback>`, SSR bình thường. **`next/dynamic`** thêm `ssr:false` (bỏ SSR, chỉ client — cho map/chart dùng `window`) và `loading`. **`import()` thường** là dynamic import thuần, không cần lazy, dùng cho prefetch `onMouseEnter={() => import('./Heavy')}` (tải chunk khi hover, khi click đã sẵn sàng) hoặc utils. Prefetch phù hợp cho heavy widget user sắp mở.

2. **Server Actions** (`"use server"`) là RPC gọi trực tiếp `await action(formData)` từ Client, tự CSRF + progressive enhancement (form chạy khi JS tắt), dùng cho **mutation nội bộ** + `revalidateTag`. **Route Handler** (`app/api/route.ts`) là REST `fetch('/api/...')`, dùng cho **public API/webhook** (mobile, Stripe). **`useActionState(action, init)`** → `[state, formAction, isPending]` bọc action trong form; **`useFormStatus()`** → `{pending}` trong children của `<form>` để disable button; **`useOptimistic(state, reducer)`** → `[optimisticState, addOptimistic]` hiện UI ngay trước khi server xong.

3. **React Compiler** auto-memoization: tự phân tích và chèn `useMemo`/`useCallback`/`memo` nên giảm 50-70% memo thủ công, trước phải viết tay và dễ quên deps. **Chưa bật mặc định 2026** vì vẫn beta, cần `babel-plugin-react-compiler` opt-in, cần codemod và test regression, chưa compatible mọi pattern (mutate props sẽ lỗi). **`eslint-plugin-react-compiler`** chỉ lint báo vi phạm rule (ví dụ mutate props) để code tương thích Compiler.

</details>

---
*Tham khảo chi tiết: `docs/03-react-nang-cao.md` — Câu 54, 62, 66. Spec: [React Docs — React.lazy](https://react.dev/reference/react/lazy), [Next.js — dynamic](https://nextjs.org/docs/app/building-your-application/optimizing/lazy-loading), [React — useActionState](https://react.dev/reference/react/useActionState), [React Compiler Beta](https://react.dev/learn/react-compiler).*
