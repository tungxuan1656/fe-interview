# Memoization & Re-render — React.memo, useMemo, useCallback, Compiler và đo hiệu năng

> Tags: #memo #useMemo #useCallback #React.memo #re-render #react-compiler #why-did-you-render | Nguồn: `docs/02-react-co-ban.md` câu 38, 31 | Mức: P0

## 1. Định nghĩa chính xác

- **Re-render**: React gọi lại function component để tạo React Element tree mới. Mặc định, **parent re-render ⇒ mọi child re-render** dù props không đổi (không phải DOM update, chỉ là JS calculation trước khi diff).
- **React.memo**: HOC memo hóa component — **skip re-render** nếu `props` shallow equal với lần trước (hoặc `areEqual(prevProps, nextProps)` custom trả `true`). Chỉ so sánh **props**, không chặn re-render do `useState`/`useContext` bên trong.
- **useMemo**: memo hóa **giá trị** — `useMemo(() => compute(a,b), [a,b])` cache kết quả tính toán nặng, chỉ recompute khi deps đổi. Trả về **value**.
- **useCallback**: memo hóa **function reference** — `useCallback(fn, deps)` cache **tham chiếu hàm** để child `memo` không bị re-render do prop function mới mỗi lần. Bản chất là `useMemo(() => fn, deps)`.
- **React Compiler** (React 19, trước gọi React Forget): compiler tự động memo hoá component/hook/value, giảm nhu cầu memo thủ công nếu code tuân Rules of Hooks và purity.

> Memo là **performance optimization**, không phải semantics. Không dùng memo thì UI vẫn đúng, chỉ có thể chậm.

## 2. Cơ chế hoạt động

### 2.1 Vì sao child re-render theo parent?

Mỗi `setState` ở parent tạo Fiber workInProgress mới, traverse `child`/`sibling`. Không có bail-out, React sẽ gọi `Child()` lại để lấy Element mới rồi diff. Dù DOM cuối không đổi, JS cost vẫn tốn.

### 2.2 Ba memo hoạt động ở ba tầng

```
Parent render → tạo props mới (object/function) 
  → React.memo(Child) shallow compare props → skip nếu bằng
  → useMemo(() => ({value}), [deps]) giữ object reference
  → useCallback(() => {}, [deps]) giữ function reference
```

- **React.memo**: ở **Reconciliation**, trước khi gọi `Child`, so `prevProps` vs `nextProps` (shallow: `Object.is` từng key). Nếu bằng ⇒ reuse Fiber cũ, không gọi `Child`.
- **useMemo**: ở **render phase** của chính component, cache **value**. Không ngăn child re-render trực tiếp, chỉ giữ reference ổn định để child `memo` có thể bail out.
- **useCallback**: tương tự `useMemo` nhưng cho function. Cần thiết khi truyền callback cho `memo` child hoặc `useEffect` deps là function.

### 2.3 Shallow compare & cost

`React.memo` mặc định `shallowEqual`: `prevProps.a === nextProps.a` (`Object.is`) cho mỗi key. Với `props.data = { value }` tạo mới mỗi render ⇒ luôn khác ⇒ memo vô ích nếu không kèm `useMemo` cho `data`.

Memo có overhead: memory giữ cache, `Object.is` so sánh deps, code phức tạp hơn. Với child rẻ (< 1ms render) hoặc props luôn đổi, memo chậm hơn không memo.

### 2.4 React Compiler future

Compiler phân tích purity và tự chèn `memo` ở build time. Khi Compiler bật, nhiều `useMemo`/`useCallback` thủ công trở nên dư thừa, thậm chí cản Compiler tối ưu nếu deps sai. Định hướng: viết code **đơn giản, pure**, để Compiler lo.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Không memo — child luôn re-render theo parent
import { useState, memo, useMemo, useCallback } from "react";

const ExpensiveChild = memo(function ExpensiveChild({
  data,
  onClick,
}: {
  data: { value: number };
  onClick: () => void;
}) {
  console.log("ExpensiveChild render");
  return <div onClick={onClick}>value: {data.value}</div>;
});

function ParentNoMemo({ items }: { items: { price: number }[] }) {
  const [count, setCount] = useState(0);
  // Mỗi render tạo object/function mới → ExpensiveChild luôn re-render dù count đổi không liên quan items
  const data = { value: items.length }; // ❌ new object
  const onClick = () => console.log("click"); // ❌ new function
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>count {count}</button>
      <ExpensiveChild data={data} onClick={onClick} />
    </>
  );
}

// 3.2 Có memo — chỉ re-render khi items đổi
function ParentMemo({ items }: { items: { price: number }[] }) {
  const [count, setCount] = useState(0);
  // Giữ reference ổn định
  const data = useMemo(() => ({ value: items.length }), [items]);
  const onClick = useCallback(() => console.log("click"), []); // không phụ thuộc gì
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>count {count}</button>
      <ExpensiveChild data={data} onClick={onClick} />
    </>
  );
}

// 3.3 useMemo cho tính toán nặng
function SortedList({ items }: { items: { price: number }[] }) {
  // Không memo: sort chạy mỗi render, O(n log n) lãng phí khi items không đổi
  const sorted = useMemo(() => {
    console.log("sort");
    return items.slice().sort((a, b) => a.price - b.price);
  }, [items]);
  return <ul>{sorted.map((it, i) => <li key={i}>{it.price}</li>)}</ul>;
}

// 3.4 React.memo với areEqual custom — so sánh sâu hơn khi cần
type Props = { user: { id: string; name: string } };
const UserRow = memo(
  function UserRow({ user }: Props) {
    console.log("UserRow", user.id);
    return <div>{user.name}</div>;
  },
  (prev, next) => prev.user.id === next.user.id && prev.user.name === next.user.name
  // chỉ re-render khi id/name đổi, bỏ qua field khác
);
```

```tsx
// 3.5 Anti-pattern: memo cho primitive — vô nghĩa
const Cheap = memo(({ count }: { count: number }) => <div>{count}</div>);
// count là number (Object.is so sánh rẻ), component rẻ → memo overhead hơn lợi

// 3.6 useCallback giữ deps đúng — stale nếu thiếu deps
function Search({ query }: { query: string }) {
  const [page, setPage] = useState(1);
  // Sai: thiếu query → stale
  // const fetchPage = useCallback(() => fetch(`/api?q=${query}&page=${page}`), [page]);
  // Đúng:
  const fetchPage = useCallback(() => fetch(`/api?q=${query}&page=${page}`), [query, page]);
  return <button onClick={() => fetchPage()}>load</button>;
}

// 3.7 React Compiler — tương lai không cần memo thủ công nhiều
// Với Compiler bật, đoạn sau tự memo, không cần useMemo/useCallback thủ công nếu component pure
function AutoMemo({ items }: { items: number[] }) {
  const doubled = items.map(x => x * 2); // Compiler tự cache nếu items không đổi
  return <div>{doubled.join(",")}</div>;
}
```

## 4. So sánh / Phân loại

| Tiêu chí | `React.memo` | `useMemo` | `useCallback` |
|----------|--------------|-----------|---------------|
| Memo gì | **Component** — skip re-render khi props shallow equal | **Giá trị** — cache kết quả compute | **Function reference** — cache hàm |
| Trả về | Component đã wrap | `value` | `function` |
| Khi dùng | Child đắt, re-render thường do parent, props ít đổi | Compute nặng (`sort`, `filter`, tạo object lớn) | Truyền callback cho `memo` child hoặc deps của `useEffect`/`useMemo` |
| Deps | `areEqual(prevProps, nextProps)` | `[deps]` | `[deps]` |
| Overhead | Shallow compare props mỗi lần parent render | So sánh deps + giữ cache | Tương tự useMemo |

| Tình huống | Dùng gì |
|------------|---------|
| Parent `count` đổi làm `ExpensiveChild` render thừa | `React.memo` + `useMemo`/`useCallback` cho props của child |
| `items.sort()` chạy mỗi render | `useMemo(() => items.sort(), [items])` |
| Callback truyền cho `memo` child | `useCallback` |
| Component rẻ, props luôn đổi | **Không memo** |
| React Compiler đã bật | Ưu tiên để Compiler lo, chỉ memo thủ công khi profiler chỉ ra |

| `useMemo` vs `useCallback` |
|-----------------------------|
| `useCallback(fn, deps)` ≡ `useMemo(() => fn, deps)` — `useCallback` là syntax sugar cho function. |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không memo mọi thứ (over-memoization)**: mỗi `useMemo`/`useCallback` tốn memory, so sánh deps, và làm code khó đọc. Chỉ memo khi **đo được** bottleneck (Profiler cho thấy child render > 5ms hoặc re-render lan rộng).
- **Không memo cho primitive props**: `count: number`, `label: string` — so sánh `React.memo` còn tốn hơn render lại component rẻ.
- **Không memo khi props luôn đổi**: nếu `data` phụ thuộc `count` mà `count` đổi mỗi lần, memo `data` vẫn recompute mỗi lần — vô ích.
- **Không dùng `useMemo` để “ngăn tạo object” mà không có `memo` child**: nếu child không `memo`, giữ reference cũng không ngăn re-render — parent re-render là child re-render mặc định.
- **Không nghĩ `useCallback` “không tạo function”**: nó vẫn tạo function mỗi lần nhưng trả về cached reference — vẫn có cost tạo.
- **Không memo để fix stale closure**: thiếu deps mà dùng `useCallback([])` để “giữ” function sẽ gây stale — phải thêm deps đúng.
- **Với React Compiler**: thủ công memo có thể thừa và cản tối ưu. Hướng mới: viết component pure, ít side effect, để Compiler tự memo; chỉ can thiệp khi cần `areEqual` custom.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Memo vô ích vì object/function mới mỗi lần**
  - Triệu chứng: `React.memo` vẫn re-render.
  - Nguyên nhân: `data={{value}}` hoặc `onClick={() => {}}` inline tạo reference mới ⇒ shallow compare fail.
  - Fix: `useMemo`/`useCallback` cho props truyền vào `memo` child.
  - Đo: React DevTools Profiler → “Why did this render? props changed: data (new reference)”; `why-did-you-render` log `different object`.

- **Lỗi 2: Deps sai gây stale hoặc recompute thừa**
  - Triệu chứng: callback dùng giá trị cũ, hoặc `useMemo` không cache.
  - Fix: `exhaustive-deps` ESLint, thêm deps đủ.
  - Đo: ESLint `react-hooks/exhaustive-deps`, console log trong `useMemo` factory.

- **Lỗi 3: Memo cho component rẻ làm chậm hơn**
  - Triệu chứng: Profiler tổng thời gian tăng sau khi wrap `memo`.
  - Fix: bỏ `memo` cho leaf rẻ, chỉ memo subtree đắt hoặc list lớn.
  - Đo: Profiler → Record → so sánh render duration trước/sau memo; `performance.now()`.

- **Lỗi 4: `areEqual` custom sai**
  - Triệu chứng: bỏ lỡ update cần thiết (UI stale) hoặc luôn re-render.
  - Fix: `areEqual` phải pure, so sánh đủ field cần thiết; mặc định shallow là đủ 90% case.
  - Đo: test unit cho `areEqual`, DevTools check props.

- **Lỗi 5: Nghĩ `useMemo` ngăn re-render parent**
  - Triệu chứng: parent vẫn render dù `useMemo` cache.
  - Fix: hiểu `useMemo` chỉ cache **value** trong parent, không ngăn parent re-render; chỉ `React.memo` mới ngăn child.
  - Đo: log trong parent vs child.

- **Công cụ**:
  - React DevTools Profiler: Record, Highlight updates, “Why did this render?”.
  - `why-did-you-render` (`wdyr`): `import './wdyr'; WhyDidYouRender(React, { trackAllPureComponents: true })` — log khi re-render do props mới reference nhưng value bằng nhau.
  - ESLint `react-hooks/exhaustive-deps`.
  - React Compiler: `eslint-plugin-react-compiler` check purity.

## 7. Câu hỏi tự kiểm tra

1. `React.memo` ngăn re-render trong trường hợp nào? Vì sao chỉ `useMemo` cho `data` mà child không `memo` thì vẫn re-render?
2. Phân biệt `useMemo` và `useCallback`. Khi nào bắt buộc dùng `useCallback` thay vì inline arrow?
3. Khi nào **KHÔNG** nên memo? React Compiler thay đổi chiến lược memo thủ công thế nào và `why-did-you-render` giúp gì?

<details>
<summary>Đáp án 30s</summary>

1. `React.memo` shallow compare `prevProps` vs `nextProps` (Object.is từng key) — nếu bằng thì skip gọi `Child`, reuse Fiber cũ. `useMemo` chỉ giữ reference ổn định cho `data` **trong parent**, nhưng mặc định parent re-render ⇒ child vẫn re-render (React không tự bail out theo props reference). Chỉ khi child có `memo` thì reference ổn định mới giúp bail out.
2. `useMemo` cache **value**, `useCallback` cache **function reference** (`useCallback(fn, deps)` ≡ `useMemo(()=>fn, deps)`). Bắt buộc `useCallback` khi truyền callback cho `memo` child (giữ reference để child bail out) hoặc khi function là deps của `useEffect`/`useMemo` khác — inline arrow tạo reference mới mỗi lần sẽ trigger effect/re-render thừa.
3. Không memo khi component rẻ, props luôn đổi, hoặc chưa đo — memo tốn memory và so sánh deps, có thể chậm hơn. Với React Compiler, compiler tự memo component/value nếu pure, nên memo thủ công nhiều là thừa; chỉ memo khi profiler chỉ ra hot path hoặc cần `areEqual` custom. `why-did-you-render` log chi tiết vì sao re-render (new object vs new function, shallow diff) để quyết định memo đúng chỗ.

</details>

---
*Tham khảo chi tiết: `docs/02-react-co-ban.md` — Câu 38, 31. Spec: [React Docs — React.memo](https://react.dev/reference/react/memo), [useMemo](https://react.dev/reference/react/useMemo), [useCallback](https://react.dev/reference/react/useCallback), [React Compiler](https://react.dev/learn/react-compiler), [why-did-you-render](https://github.com/welldone-software/why-did-you-render).*
