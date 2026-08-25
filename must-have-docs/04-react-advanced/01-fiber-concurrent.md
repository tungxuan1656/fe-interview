# Fiber, Concurrent Rendering và Lanes — Linked list, alternate double buffer, render vs commit

> Tags: #react #fiber #concurrent #lanes #scheduler #render-phase | Nguồn: `docs/03-react-nang-cao.md` câu 51, 52, 53 + `docs/11-nextjs.md` | Mức: P0

## 1. Định nghĩa chính xác

- **Fiber** là reimplementation của React reconciler từ React 16, thay Stack Reconciler (đệ quy đồng bộ, không interrupt). Fiber biến cây component thành **linked list** (`child`, `sibling`, `return`) và chia công việc thành **units of work** có thể pause, abort, reuse và ưu tiên theo **lanes**.
- **Fiber Node** là object chứa `tag`, `key`, `elementType`, `pendingProps`, `memoizedProps`, `memoizedState` (hooks), `stateNode` (DOM instance), `child/sibling/return`, `alternate` (double buffer), `flags` (effectTag), `lanes`/`childLanes` (priority).
- **Concurrent Rendering** không phải đa luồng mà là **cooperative scheduling**: render chia thành chunks, sau mỗi chunk yield về Scheduler để xử lý task ưu tiên cao hơn (user input), rồi tiếp tục hoặc restart. Kích hoạt bằng `createRoot` (React 18+), không phải `ReactDOM.render` legacy.
- **Lanes** là bitmask priority (31 lanes) thay thế expiration time cũ, cho phép gộp, tách, và ưu tiên update. **Render phase** interruptible, pure, không side effect. **Commit phase** đồng bộ, không interrupt, mutate DOM và chạy `useLayoutEffect`.

## 2. Cơ chế hoạt động

### 2.1 Fiber linked list thay vì tree đệ quy

```
Stack Reconciler (cũ):  function recursion → call stack, không pause được
Fiber (mới):            while loop trên linked list → có thể yield

Cây JSX:
<App><Header /><List><Item /><Item /></List></App>

Fiber linked list (DFS):
App.child → Header
Header.sibling → List
List.child → Item1
Item1.sibling → Item2
Item2.return → List
List.return → App
```

Mỗi Fiber là unit of work. Scheduler duyệt qua `beginWork` (đi xuống) và `completeWork` (đi lên), sau mỗi unit kiểm tra `shouldYield()`.

### 2.2 Double buffer với `alternate`

- Hai cây Fiber tồn tại song song: `current` (đã commit, hiển thị trên màn hình) và `workInProgress` (đang build).
- Mỗi node có `alternate` trỏ sang node đối ứng. Khi `workInProgress` hoàn tất render phase, React tráo con trỏ `current = workInProgress` trong commit phase (O(1) pointer swap).
- Lợi ích: có thể abort `workInProgress` nếu có update ưu tiên cao hơn, giữ `current` ổn định, và reuse `alternate` cho lần render sau (giảm GC).

```
current tree (hiển thị)  ←→ alternate →  workInProgress tree (đang build)
     FiberA  ←alternate→  FiberA'
     FiberB  ←alternate→  FiberB'
Khi commit: root.current = finishedWork (workInProgress)
```

### 2.3 Lanes — priority bitmask

- Lanes là `number` 31-bit, mỗi bit là một priority. Ví dụ: `SyncLane = 0b0001` (khẩn nhất), `InputContinuousLane`, `DefaultLane`, `TransitionLane`, `IdleLane`.
- Update được gán lane khi tạo: `setState` trong event → `SyncLane`/`InputContinuousLane`; trong `startTransition` → `TransitionLane` (thấp).
- Reconciler có thể: **merge** lanes (`a | b`), **nhánh** (`includes`), và **ưu tiên** lane cao trước. Render có thể bị **interrupted** nếu có update lane cao hơn chen vào → discard workInProgress của lane thấp và restart với lane cao.
- Scheduler + Lanes quyết định deadline: urgent lane có deadline ngắn (cần commit ngay), transition lane có thể trì hoãn, idle lane chỉ chạy khi rảnh.

### 2.4 Hai pha Render vs Commit

```
Trigger (setState/useReducer/context)
  → Schedule (gán lane, enqueue update)
  → Render phase (reconcile): interruptible, pure
      beginWork / completeWork từng Fiber
      không mutate DOM, không chạy effect
      có thể pause / abort / restart
  → Commit phase (đồng bộ, không interrupt)
      1. beforeMutation (getSnapshotBeforeUpdate)
      2. mutation (DOM insert/update/delete theo flags)
      3. layout (useLayoutEffect, componentDidMount/Update — chạy trước paint)
      → Browser paint
      → Passive (useEffect chạy sau paint, không block)
```

- Legacy `ReactDOM.render` luôn gán `SyncLane` → đồng bộ, block main thread, gõ input khi render list 10k item sẽ lag.
- `createRoot` bật concurrent: render ở `DefaultLane`/`TransitionLane` có thể yield, input được ưu tiên.

### 2.5 createRoot vs hydrateRoot vs legacy

- `createRoot(container).render(<App />)` → tạo FiberRoot, bật concurrent, mọi update đều qua Scheduler.
- `hydrateRoot(container, <App />)` → tương tự nhưng **attach** vào HTML đã SSR, cố gắng reuse DOM, không tạo mới. Cũng concurrent, thêm selective hydration.
- `ReactDOM.render` (legacy, React 17) → Stack-like, đồng bộ, không có lanes interrupt.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Cấu trúc Fiber node đơn giản (minh họa, không phải API public)
type Fiber = {
  tag: number; // FunctionComponent = 0, HostComponent = 5, ...
  key: string | null;
  elementType: React.ElementType;
  stateNode: HTMLElement | null; // DOM node hoặc instance
  child: Fiber | null;
  sibling: Fiber | null;
  return: Fiber | null; // parent
  pendingProps: any;
  memoizedProps: any;
  memoizedState: any; // hooks linked list
  alternate: Fiber | null; // double buffer
  flags: number; // Placement | Update | Deletion
  lanes: number; // priority của node này
  childLanes: number; // priority của subtree
};

// 3.2 createRoot (concurrent) vs legacy — khác biệt behavior
// Legacy — block
// import ReactDOM from 'react-dom';
// ReactDOM.render(<App />, document.getElementById('root')); // sync, block input

// Concurrent — interruptible
import { createRoot } from 'react-dom/client';
import { hydrateRoot } from 'react-dom/client';

// CSR với concurrent
const root = createRoot(document.getElementById('root')!);
root.render(<App />); // có thể interrupt bởi urgent update

// SSR hydration — reuse HTML server
// hydrateRoot(document.getElementById('root')!, <App />);

// 3.3 Lanes ưu tiên — urgent vs transition
import { useState, useTransition } from 'react';

function SearchApp({ items }: { items: string[] }) {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const [filtered, setFiltered] = useState(items);

  // lanes minh họa: setQuery → InputContinuousLane (cao), setFiltered trong transition → TransitionLane (thấp)
  function onChange(e: React.ChangeEvent<HTMLInputElement>) {
    const v = e.target.value;
    setQuery(v); // urgent — commit ngay để input không lag
    startTransition(() => {
      // non-urgent — có thể bị interrupt nếu user gõ tiếp
      setFiltered(items.filter(s => s.includes(v)));
    });
  }

  return (
    <>
      <input value={query} onChange={onChange} />
      {isPending && <span> filtering…</span>}
      <ul>{filtered.map(s => <li key={s}>{s}</li>)}</ul>
    </>
  );
}

// 3.4 Render phase có thể gọi nhiều lần và bị discard — phải pure
function PureComponent({ count }: { count: number }) {
  // ❌ Sai: side effect trong render — sẽ chạy nhiều lần khi concurrent retry
  // if (count > 0) localStorage.setItem('count', String(count));

  // ✅ Đúng: side effect trong effect/commit phase
  // useEffect(() => { localStorage.setItem('count', String(count)); }, [count]);
  return <div>{count}</div>;
}
```

## 4. So sánh / Phân loại

| Tiêu chí | Stack Reconciler (pre-16) | Fiber Reconciler (16+) |
|----------|---------------------------|------------------------|
| Cấu trúc | Đệ quy call stack | Linked list (child/sibling/return) |
| Interrupt | Không — block tới xong | Có — yield sau mỗi unit, abort/restart |
| Priority | Không (FIFO) | Có — lanes bitmask, ưu tiên urgent |
| Double buffer | Không | Có — alternate current ↔ workInProgress |
| Concurrent Features | Không | Có — useTransition, Suspense, selective hydration |

| Tiêu chí | Render phase | Commit phase |
|----------|--------------|--------------|
| Mục đích | Tính toán diff, tạo Fiber mới | Áp dụng mutations lên DOM |
| Có chạm DOM? | Không | Có |
| Có interrupt được? | Có — yield cho Scheduler | Không — đồng bộ, atomic |
| Side effect? | Không (phải pure) | Có — mutation + useLayoutEffect |
| Chạy mấy lần? | Có thể nhiều lần rồi discard | Chạy đúng 1 lần khi commit |

| Tiêu chí | `createRoot` | `hydrateRoot` | `ReactDOM.render` (legacy) |
|----------|-------------|---------------|----------------------------|
| Concurrent | ✅ Bật | ✅ Bật | ❌ Đồng bộ |
| Dùng khi | CSR thuần | SSR + hydrate | Legacy, không dùng mới |
| Lanes | Đầy đủ | Đầy đủ + selective hydration | Chỉ SyncLane |
| Behavior gõ input khi render nặng | Mượt, input ưu tiên | Mượt | Lag, block |

| Lane (ví dụ) | Priority | Gán khi | Deadline |
|--------------|----------|---------|----------|
| SyncLane | Cao nhất | `flushSync`, legacy | Ngay lập tức |
| InputContinuousLane | Cao | Typing, click | ~50ms |
| DefaultLane | Trung bình | `setState` thường | ~100ms |
| TransitionLane | Thấp | `startTransition`, `useDeferredValue` | Linh hoạt, có thể delay |
| IdleLane | Thấp nhất | `preload`, `offscreen` | Khi rảnh |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không lạm dụng concurrent để che render đắt**: Transition chỉ hoãn commit, không làm thuật toán nhanh hơn. Nếu `filterHugeList` O(n) nặng, vẫn cần `useDeferredValue` + `useMemo` + virtualization (`tanstack-virtual`), không chỉ `startTransition`.
- **Không đặt side effect trong render**: Vì render có thể retry và discard, mọi `fetch`, `subscribe`, `localStorage.setItem`, `Math.random()` trong render sẽ chạy nhiều lần hoặc cho kết quả khác nhau mỗi retry → tearing. Đưa vào `useEffect`/`useLayoutEffect` hoặc event handler.
- **Không dùng `createRoot` mà vẫn mong concurrent**: Chỉ `createRoot`/`hydrateRoot` mới bật lanes interrupt. Nếu vẫn `ReactDOM.render` thì `useTransition` không có tác dụng.
- **Không gán mọi update là urgent**: Nếu bọc cả `setFiltered` lẫn `setQuery` ngoài `startTransition`, cả hai đều urgent → không có de-prioritize, input vẫn lag. Phải tách urgent (input value) khỏi non-urgent (list filter).
- **Chi phí Fiber**: Mỗi element là một Fiber node object (~200-400 bytes), cộng alternate double buffer → memory cao hơn Stack. Nhưng đổi lại cooperative scheduling thắng về UX.
- **Không dùng `flushSync` bừa bãi**: `flushSync(() => setState(...))` ép SyncLane đồng bộ, phá batching và concurrent, chỉ dùng khi cần đọc DOM ngay sau setState.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Side effect trong render gây double invoke và leak**
  - Triệu chứng: `useEffect` double chạy trong StrictMode, `console.log` trong render in 2 lần, subscription tạo 2 lần.
  - Fix: Render phải pure. Chuyển side effect sang `useEffect` (passive, sau paint) hoặc `useLayoutEffect` (trước paint).
  - Đo: Bật `<React.StrictMode>` sẽ double-invoke render để phát hiện impure. React DevTools Profiler → highlight.

- **Lỗi 2: Input lag vì không tách urgent/non-urgent**
  - Triệu chứng: Gõ input giật, mỗi keystroke render list 10k item đồng bộ.
  - Fix: `setQuery` urgent ngoài transition, `setFiltered` trong `startTransition`. Hoặc dùng `useDeferredValue` ở component con.
  - Đo: Chrome Performance → Long task >50ms, INP cao. React DevTools Profiler → Record, xem commit duration của List, flame graph.

- **Lỗi 3: Tearing — UI hiện hai giá trị khác nhau trong một render**
  - Triệu chứng: List hiển thị `query="a"` nhưng `filtered` tương ứng `query=""` do external store đổi giữa render.
  - Fix: Dùng `useSyncExternalStore` (concurrent-safe) thay vì `useEffect` subscribe thủ công. Đảm bảo `getSnapshot` pure và cache.
  - Đo: React DevTools → Components → inconsistent props; hoặc visual glitch khi concurrent retry.

- **Lỗi 4: Hiểu nhầm lanes → bọc urgent trong transition**
  - Triệu chứng: Input vẫn lag dù đã dùng `startTransition` (vì bọc nhầm `setQuery` trong transition).
  - Fix: Chỉ bọc non-urgent. Urgent luôn ngoài.
  - Đo: Thêm `console.log(isPending)` — nếu `isPending` true ngay khi gõ, nghĩa là urgent bị hoãn sai.

- **Lỗi 5: Dùng `hydrateRoot` sai chỗ gây hydration mismatch**
  - Triệu chứng: `Hydration failed because the initial UI does not match...`
  - Fix: `hydrateRoot` chỉ cho HTML đã SSR. CSR thuần dùng `createRoot`. Đảm bảo server/client render giống nhau (tránh `Date.now()`/`Math.random()` trong render).
  - Đo: `hydrateRoot(container, <App />, { onRecoverableError: (err) => console.error(err) })`, console warning.

- **Công cụ đo**:
  - **React DevTools Profiler**: Record → Flame graph / Ranked chart → xem Fiber nào render lâu, actualDuration vs baseDuration, why did this render.
  - **Chrome Performance + Scheduler**: Xem `Task` vs `Idle`, long task, `requestIdleCallback` / `MessageChannel` mà Scheduler dùng.
  - **`performance.mark/measure`**: Bọc `filter` để đo cost, so sánh trước/sau transition.
  - **Lighthouse / web-vitals**: INP, TTFB trước và sau khi tách urgent. `flushSync` làm INP tệ đi.

## 7. Câu hỏi tự kiểm tra

1. Vì sao Fiber dùng linked list (`child/sibling/return`) thay vì cây đệ quy, và `alternate` double buffer giải quyết vấn đề gì khi concurrent render bị interrupt?
2. Phân biệt Render phase và Commit phase: cái nào có thể interrupt, cái nào chạy `useLayoutEffect`, và vì sao render phải pure?
3. `createRoot` vs `hydrateRoot` vs legacy `ReactDOM.render` khác gì về lanes và khi nào input typing bị block khi render list nặng?

<details>
<summary>Đáp án 30s</summary>

1. **Linked list** cho phép duyệt Fiber bằng `while` loop, yield sau mỗi unit of work để Scheduler chen urgent task, không phụ thuộc call stack. **Alternate** giữ hai cây `current` (đã commit) và `workInProgress` (đang build) trỏ nhau qua `alternate`; khi có update ưu tiên cao chen vào, React có thể discard `workInProgress` lane thấp, giữ `current` ổn định, rồi restart mà không mất UI đã hiển thị. Khi commit xong chỉ swap con trỏ `root.current`.

2. **Render phase** (beginWork/completeWork) là pure calculation, không chạm DOM, **interruptible**, có thể chạy nhiều lần rồi discard. **Commit phase** (mutation + layout) **đồng bộ không interrupt**, mutate DOM và chạy `useLayoutEffect` trước paint (sau đó `useEffect` passive sau paint). Vì render có thể retry, nó phải pure — không side effect, không `Math.random()`/`Date.now()`.

3. **`createRoot`** bật concurrent với đầy đủ lanes, render có thể yield nên input (InputContinuousLane) ưu tiên hơn list filter (TransitionLane) → không lag. **`hydrateRoot`** cũng concurrent nhưng thêm reuse DOM SSR + selective hydration. **Legacy `ReactDOM.render`** luôn SyncLane đồng bộ, block main thread → gõ input khi render nặng sẽ lag.

</details>

---
*Tham khảo chi tiết: `docs/03-react-nang-cao.md` — Câu 51, 52, 53. Spec: [React Fiber Architecture — acdlite](https://github.com/acdlite/react-fiber-architecture), [React Docs — Render and Commit](https://react.dev/learn/render-and-commit), [Scheduler](https://github.com/facebook/react/tree/main/packages/scheduler).*
