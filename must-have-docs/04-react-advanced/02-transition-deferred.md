# useTransition vs useDeferredValue — Urgent vs Non-Urgent, isPending vs query !== deferredQuery

> Tags: #react #concurrent #useTransition #useDeferredValue #lanes | Nguồn: `docs/03-react-nang-cao.md` câu 52, 53, 55, 64 | Mức: P0

## 1. Định nghĩa chính xác

- **Urgent update** là update cần phản hồi ngay để giữ input/click mượt (ví dụ: `setQuery` cho `<input value>`). Gán **lane cao** (`InputContinuousLane`/`DefaultLane`), React commit ngay.
- **Non-urgent update** (transition) là update có thể hoãn (ví dụ: filter list 10k item, fetch tab mới). Gán **TransitionLane** (lane thấp), React có thể interrupt, delay, hoặc discard nếu có urgent mới.
- **`useTransition() → [isPending, startTransition]`** : hook cho phép **wrap setter** trong `startTransition(() => setState(...))` để đánh dấu non-urgent. Trả `isPending = true` khi transition chưa commit. Dùng khi bạn **control được** chỗ gọi `setState`.
- **`useDeferredValue(value, initialValue?) → deferredValue`** : hook trả **phiên bản deferred** của `value` ở priority thấp. React re-render với `deferredValue` cũ trước, rồi background render với giá trị mới. Dùng khi bạn **không control được** nguồn update (props từ parent, value từ lib). So sánh `value !== deferredValue` để biết đang stale (tương tự `isPending` nhưng thủ công).
- Cả hai đều dựa trên Concurrent Rendering + lanes, không làm thuật toán nhanh hơn, chỉ thay đổi **thứ tự ưu tiên commit**.

## 2. Cơ chế hoạt động

### 2.1 useTransition — bạn control setter

```
onChange(e):
  setQuery(e.target.value)              // lane cao — urgent, commit ngay
  startTransition(() => {
    setFiltered(filter(items, value))   // TransitionLane — thấp, có thể yield
  })
```

- `startTransition` đánh dấu mọi update bên trong là transition. React giữ UI cũ (suspense boundary không show fallback ngay) và render nhánh mới ở background.
- `isPending` là boolean do React quản lý: `true` từ khi transition bắt đầu tới khi commit xong. Dùng để hiện `<Spinner />` hoặc disable.
- Có thể lồng: `startTransition` trong `startTransition` vẫn là một transition.
- `startTransition` là **synchronous wrapper**, không `await` được; async bên trong phải tự quản lý.

### 2.2 useDeferredValue — bạn nhận value từ parent

```
Parent:  <FilteredList query={query} items={items} />
Child:   const deferredQuery = useDeferredValue(query);
         const filtered = useMemo(() => filter(items, deferredQuery), [items, deferredQuery]);
         const isStale = query !== deferredQuery;
```

- `useDeferredValue` hoạt động như `useState` + `useEffect` với priority thấp nhưng được Scheduler tối ưu: lần render đầu vẫn dùng `deferredQuery = query` cũ, React schedule re-render background với `query` mới ở TransitionLane.
- Không trả `isPending`; tự so sánh `query !== deferredQuery` để biết stale và làm mờ UI (`opacity: 0.5`).
- React 19 thêm `initialValue`: `useDeferredValue(query, "")` để lần đầu không flash với query rỗng.

### 2.3 Chung — Scheduler và interrupt

- Urgent lane có deadline ngắn, non-urgent lane có thể bị **interrupted** nếu user gõ tiếp. React discard render cũ của `filtered` và restart với `query` mới nhất.
- Cả hai đều yêu cầu component **tách** để urgent path không re-render nặng. Nếu `<input>` và `<HugeList>` cùng component, `setQuery` urgent vẫn làm `HugeList` render lại dù đã deferred → mất tác dụng. Cần tách `HugeList` ra component riêng và `memo` hoặc dùng `deferredValue` bên trong nó.

```
Không tách:  App { query, filtered } → mỗi setQuery render cả App + HugeList (nặng)
Tách:        App { query } → SearchInput (urgent, nhẹ)
             └─ FilteredList { deferredQuery } → render nặng nhưng ở lane thấp
```

### 2.4 Không liên quan tới Suspense fallback mặc định

- `useTransition` tránh **unnecessary fallback**: nếu transition suspend (throw Promise), React không show fallback của Suspense mà giữ UI cũ cho tới khi ready, trừ khi là initial load.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 useTransition — bạn control setState (search filter)
import { useState, useTransition, memo } from 'react';

function SearchWithTransition({ items }: { items: string[] }) {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const [filtered, setFiltered] = useState(items);

  function onChange(e: React.ChangeEvent<HTMLInputElement>) {
    const v = e.target.value;
    setQuery(v); // urgent — input phản hồi ngay
    startTransition(() => {
      // non-urgent — filter nặng có thể interrupt
      setFiltered(items.filter(s => s.toLowerCase().includes(v.toLowerCase())));
    });
  }

  return (
    <>
      <input value={query} onChange={onChange} placeholder="Search 10k items" />
      {isPending && <span aria-live="polite"> Searching…</span>}
      <List items={filtered} />
    </>
  );
}

const List = memo(function List({ items }: { items: string[] }) {
  // memo để parent urgent render không kéo List render nếu items chưa đổi
  return <ul>{items.map(s => <li key={s}>{s}</li>)}</ul>;
});

// 3.2 useDeferredValue — bạn không control được query (props từ parent)
import { useDeferredValue, useMemo } from 'react';

function FilteredList({ query, items }: { query: string; items: string[] }) {
  const deferredQuery = useDeferredValue(query); // React 19: useDeferredValue(query, '')
  const isStale = query !== deferredQuery;

  // chỉ filter với deferredQuery (lane thấp), không block typing
  const filtered = useMemo(
    () => items.filter(s => s.toLowerCase().includes(deferredQuery.toLowerCase())),
    [items, deferredQuery]
  );

  return (
    <div style={{ opacity: isStale ? 0.5 : 1, transition: 'opacity 150ms' }}>
      {/* isStale tương đương isPending nhưng tự so sánh */}
      {isStale && <span>Updating…</span>}
      <ul>{filtered.map(s => <li key={s}>{s}</li>)}</ul>
    </div>
  );
}

// Parent không cần biết transition:
function Parent({ items }: { items: string[] }) {
  const [query, setQuery] = useState('');
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {/* FilteredList tự defer bên trong */}
      <FilteredList query={query} items={items} />
    </>
  );
}

// 3.3 Sai — bọc urgent trong transition làm input lag
function Wrong() {
  const [query, setQuery] = useState('');
  const [, startTransition] = useTransition();
  function onChange(e: React.ChangeEvent<HTMLInputElement>) {
    // ❌ Sai: cả setQuery cũng trong transition → input bị hoãn
    startTransition(() => setQuery(e.target.value));
  }
  return <input value={query} onChange={onChange} />;
}

// 3.4 Kết hợp với Suspense — transition giữ UI cũ thay vì fallback
import { Suspense, useState, useTransition } from 'react';
function Tabs() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  function select(next: string) {
    startTransition(() => setTab(next)); // tab mới suspend → giữ tab cũ, không flash fallback
  }
  return (
    <>
      <button onClick={() => select('home')} aria-selected={tab === 'home'}>Home</button>
      <button onClick={() => select('profile')} aria-selected={tab === 'profile'}>Profile</button>
      {isPending && <span>Loading tab…</span>}
      <Suspense fallback={<div>Loading…</div>}>
        <TabContent tab={tab} />
      </Suspense>
    </>
  );
}
function TabContent({ tab }: { tab: string }) {
  // giả sử suspend khi fetch
  return <div>{tab}</div>;
}
```

## 4. So sánh / Phân loại

| Tiêu chí | `useTransition` | `useDeferredValue` |
|----------|-----------------|---------------------|
| Điều khiển | Bạn **control** được `setState` — wrap setter | Bạn **không control** được — defer `value` nhận từ parent/lib |
| API | `[isPending, startTransition]` + `startTransition(() => setState(...))` | `deferredValue = useDeferredValue(value)` + `value !== deferredValue` |
| `isPending` | Có sẵn `isPending` boolean | Không — tự so sánh `query !== deferredQuery` là stale |
| Khi dùng | `onChange`, `onClick` bạn viết handler | Props từ parent, context, Zustand selector, URL searchParams |
| Ưu điểm | `isPending` rõ ràng, control nhiều state cùng lúc | Không cần prop drilling `isPending`, tiện cho lib component |
| React 19 thêm | Không đổi | `initialValue` để tránh flash lần đầu |

| Tiêu chí | Urgent update | Non-urgent (Transition/Deferred) |
|----------|---------------|----------------------------------|
| Lane | Sync/InputContinuous/Default (cao) | TransitionLane (thấp) |
| Interrupt được? | Không — commit ngay | Có — có thể bị urgent chen và discard |
| Ví dụ | `setQuery` cho input, `setActiveTab` immediate | `setFiltered`, `setTab` với Suspense, `deferredQuery` |
| UX khi suspend | Không giữ cũ, show fallback ngay nếu suspend | Giữ UI cũ (stale) tới khi ready, tránh flash |

| Tình huống | Chọn gì |
|------------|---------|
| Bạn viết `onChange` filter list | `useTransition` — wrap `setFiltered` |
| Component con nhận `query` từ parent và phải filter | `useDeferredValue` bên trong con |
| Cần `isPending` để disable button/spinner | `useTransition` |
| Dùng lib không expose setter (ví dụ: props `search` từ router) | `useDeferredValue` |
| Tab navigation với Suspense | `useTransition` để giữ tab cũ |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng cho mọi state**: Chỉ cho update **đắt** (filter 1k+ item, render chart nặng, suspend fetch). Với `setCount(c => c+1)` rẻ, thêm transition chỉ overhead (thêm render với deferred value).
- **Không bọc urgent trong transition**: `startTransition(() => setQuery(...))` làm input bị hoãn → lag ngược. Luôn tách urgent ngoài.
- **Không dùng `useDeferredValue` mà không `memo`**: Nếu `FilteredList` không `useMemo` cho `filtered`, mỗi render deferred vẫn tính lại filter nặng → không tiết kiệm. Phải `useMemo` với `deferredQuery` hoặc tách component và `memo`.
- **Không mong transition làm code chạy nhanh hơn**: Nó chỉ **hoãn** commit, không giảm complexity. Thuật toán vẫn O(n). Kết hợp với virtualization, debounce server, hoặc worker nếu thực sự nặng.
- **Không thay thế debounce cho server fetch**: `useDeferredValue` defer render, không defer network. Nếu mỗi keystroke fetch API, vẫn cần `debounce`/`throttle` hoặc dùng `useTransition` + abort controller.
- **Chi phí**: Mỗi deferred value là một render thêm ở lane thấp (double render). Quá nhiều `useDeferredValue` chồng nhau làm Scheduler bận. Chỉ defer giá trị thực sự nặng.
- **Khi nào KHÔNG dùng**: Form validation đơn giản, toggle boolean, state không gây render nặng — dùng `setState` thường.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Bọc nhầm urgent → input lag dù đã transition**
  - Triệu chứng: Gõ vẫn giật, `isPending` true ngay cả khi gõ 1 ký tự.
  - Fix: `setQuery` ngoài, `startTransition` chỉ cho `setFiltered`. Kiểm tra code: urgent không trong callback.
  - Đo: React DevTools Profiler → Record typing → xem `isPending` và commit duration. Nếu input commit >16ms, urgent bị hoãn sai.

- **Lỗi 2: Dùng `useDeferredValue` cho mọi props → overhead**
  - Triệu chứng: Nhiều render thừa, `performance.measure` thấy double render liên tục dù list nhỏ.
  - Fix: Chỉ defer khi `filter`/`render` đắt. Với list <100 item, không cần.
  - Đo: Profiler → Components → Rendered at → thấy `FilteredList` render 2 lần mỗi keystroke. So sánh before/after với `memo`.

- **Lỗi 3: Quên `useMemo` với `deferredQuery`**
  - Triệu chứng: Dù đã `useDeferredValue`, mỗi keystroke vẫn lag vì `filter` chạy với `query` gốc.
  - Fix: `const filtered = useMemo(() => filter(items, deferredQuery), [items, deferredQuery])`. Không phụ thuộc `query`.
  - Đo: Thêm `performance.mark` trong `filter`; đếm số lần chạy. Phải chỉ chạy khi `deferredQuery` đổi.

- **Lỗi 4: Hiểu `isPending` vs `query !== deferredQuery` sai**
  - Triệu chứng: Dùng `isPending` với `useDeferredValue` (không có), hoặc so sánh `deferredQuery !== query` nhưng quên xử lý stale UI.
  - Fix: `useTransition` → `isPending` có sẵn. `useDeferredValue` → `const isStale = query !== deferredQuery` là manual pending.
  - Đo: Log `query`, `deferredQuery`, `isStale` ra UI để quan sát delay.

- **Lỗi 5: Không tách component → urgent vẫn re-render nặng**
  - Triệu chứng: Dù đã defer, `App` vẫn render `HugeList` mỗi keystroke vì cùng component.
  - Fix: Tách `HugeList` thành component con nhận `deferredQuery`, bọc `memo`. Hoặc dùng `children` slot.
  - Đo: React DevTools → Highlight updates when components render → thấy `HugeList` flash mỗi keystroke là sai; sau tách chỉ flash khi `deferredQuery` đổi.

- **Công cụ**:
  - **React DevTools Profiler**: Flame graph → `FilteredList` actualDuration phải thấp ở urgent render, cao ở transition render nhưng không block input.
  - **Chrome Performance → INP**: Gõ nhanh phải INP <200ms. Nếu INP cao dù đã transition → sai tách.
  - **`console.log` isPending / isStale**: Quan sát timing.
  - **StrictMode**: Phát hiện double render.

## 7. Câu hỏi tự kiểm tra

1. Khi nào chọn `useTransition` và khi nào chọn `useDeferredValue`? Vì sao `useDeferredValue` cần `query !== deferredQuery` trong khi `useTransition` có `isPending`?
2. Vì sao phải tách `setQuery` (urgent) ngoài `startTransition` và chỉ bọc `setFiltered`? Điều gì xảy ra nếu bọc cả hai?
3. Vì sao `useDeferredValue` phải kết hợp với `useMemo`/`memo` và tách component, nếu không sẽ không giảm lag?

<details>
<summary>Đáp án 30s</summary>

1. **Chọn `useTransition`** khi bạn **control được setter** (viết `onChange`/`onClick` và gọi `setState`): wrap `startTransition(() => setFiltered(...))`, có `isPending` sẵn để hiện loading. **Chọn `useDeferredValue`** khi bạn **không control** nguồn value (props từ parent, context, lib): `const deferredQuery = useDeferredValue(query)` bên trong con, tự defer. `useTransition` trả `isPending` do React quản lý transition; `useDeferredValue` không có transition object nên phải tự so sánh `query !== deferredQuery` để biết stale — đó là manual `isPending`.

2. **Urgent** (`setQuery`) cần lane cao để input commit ngay, giữ 60fps. **Non-urgent** (`setFiltered`) lane thấp có thể yield. Nếu bọc cả `setQuery` trong `startTransition`, input cũng thành lane thấp → bị hoãn, gõ lag ngược. Đúng: `setQuery` ngoài (urgent), `startTransition(() => setFiltered(...))` trong (non-urgent).

3. `useDeferredValue` chỉ hoãn **giá trị**, không tự memo computation. Nếu `FilteredList` cùng component với input, mỗi `setQuery` urgent vẫn render lại cả `filter` nặng dù `deferredQuery` chưa đổi. Phải tách `FilteredList` ra component riêng, `useMemo` với `deferredQuery`, và `memo` — khi đó urgent render chỉ chạy input nhẹ, filter chỉ chạy ở deferred render lane thấp.

</details>

---
*Tham khảo chi tiết: `docs/03-react-nang-cao.md` — Câu 52, 53, 55, 64. Spec: [React Docs — useTransition](https://react.dev/reference/react/useTransition), [useDeferredValue](https://react.dev/reference/react/useDeferredValue), [Concurrent UI Patterns](https://react.dev/learn/concurrent-mode-adoption).*
