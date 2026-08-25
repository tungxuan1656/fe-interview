# Render, Reconciliation, Virtual DOM và key — Render vs Commit, Diff O(n) và định danh ổn định

> Tags: #react #render #reconciliation #virtual-dom #fiber #key #diff | Nguồn: `docs/02-react-co-ban.md` câu 31, 32, 33, 50 | Mức: P0

## 1. Định nghĩa chính xác

- **Render**: giai đoạn React gọi function component (hoặc `render()` của class) để tạo **React Element tree** — object thuần `{ type, props, key, ref }` mô tả UI. Render là **pure calculation** trong memory, chưa chạm DOM.
- **Reconciliation**: thuật toán diff giữa hai Virtual DOM tree (tree cũ `current` và tree mới `workInProgress`) để tính **minimal mutations** cần commit lên DOM. Từ React 16+ được cài trên **Fiber** architecture (linked list + lanes).
- **Commit**: giai đoạn áp dụng mutations (insert/update/delete/move) lên DOM thật, chạy layout effects, paint.
- **Virtual DOM (VDOM)**: object tree trong memory, abstraction cho phép React batch update, tránh layout thrashing, và render ra nhiều target (DOM / Native / Canvas).
- **key**: prop đặc biệt (`string | number`) để React **định danh** (identity) element giữa các render trong cùng parent. `key` không nằm trong `props`, chỉ dùng cho Reconciliation.

> Phân biệt cốt lõi: `render` tạo mô tả, `reconciliation` so sánh mô tả, `commit` mới thay đổi DOM. Nhầm lẫn “render = DOM update” là sai.

## 2. Cơ chế hoạt động

### 2.1 Hai pha của React trên Fiber

```
Trigger (setState/props/context) 
  → Render phase (có thể interrupt, pure, không side effect)
    → Reconciliation (diff Fiber tree)
  → Commit phase (đồng bộ, mutate DOM, chạy effects)
    → Browser paint
```

- **Render phase**: traverse Fiber tree (DFS qua `child`/`sibling`/`return` pointers), gọi component, tạo `ReactElement`, so sánh với Fiber cũ. Có thể bị **interrupt** bởi Concurrent Features (`startTransition`, `Suspense`).
- **Commit phase**: ba sub-phase — `beforeMutation` (đo DOM), `mutation` (insert/update/delete), `layout` (`useLayoutEffect` chạy đồng bộ trước paint). Sau commit mới tới `passive` (`useEffect` sau paint).

### 2.2 Diff O(n) với hai giả định (heuristic)

Brute-force tree diff là O(n³). React đạt O(n) nhờ:

1. **Khác `type` thì teardown toàn bộ subtree**: `div` → `span`, `ComponentA` → `ComponentB` ⇒ unmount cũ, mount mới, state/refs mất. Cùng `type` thì giữ instance, chỉ diff props.
2. **List diff dựa trên `key`**: với `children` là array, React match item cũ/mới qua `key`. Có `key` ổn định ⇒ move thay vì destroy/create. Không `key` hoặc `key={index}` ⇒ match theo vị trí, sai khi reorder/insert.

Cấu trúc Fiber liên quan: `FiberNode { type, key, pendingProps, memoizedProps, child, sibling, return, alternate, flags, lanes }`. `alternate` trỏ tới Fiber đối ứng (current ↔ workInProgress).

### 2.3 `key` — contract và remount pattern

- **Scope**: unique trong **siblings**, không cần global. `key` trùng ở hai list khác parent là hợp lệ.
- **Stable**: sinh khi tạo data (`crypto.randomUUID()` khi create), không sinh trong render (`Math.random()` / `Date.now()` mỗi render khác ⇒ remount liên tục).
- **Không truyền vào props**: `props.key === undefined`. Muốn đọc trong component phải truyền prop khác (`id`).
- **Remount pattern**: đổi `key` để force unmount/mount, reset toàn bộ state/effect của subtree — thay thế `useEffect` reset thủ công:

```tsx
<ChatForm key={userId} /> // userId đổi ⇒ form reset hoàn toàn
<EmployeeDetail key={employee.id} />
```

### 2.4 Hydration liên quan

Khi SSR, HTML đã có sẵn. Hydration so sánh VDOM client với DOM server. Mismatch do `key` không ổn định giữa server/client (`Math.random()` trong render) gây hydration warning và discard.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Render không đồng nghĩa commit — setState với Object.is bằng cũ sẽ bail out
import { useState, memo } from "react";

function Parent() {
  const [count, setCount] = useState(0);
  console.log("Parent render"); // chạy mỗi khi Parent render
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>count: {count}</button>
      {/* Child re-render theo Parent trừ khi memo */}
      <Child />
      {/* Nút bail-out: giá trị bằng cũ (Object.is) → React bỏ render */}
      <button onClick={() => setCount(c => c)}>bail out (no render)</button>
    </>
  );
}
const Child = memo(() => {
  console.log("Child render — chỉ khi props đổi");
  return <div>Child (memo)</div>;
});

// 3.2 Diff theo type — khác type thì mất state
function DemoTypeDiff({ show }: { show: boolean }) {
  // show=true → <InputA />, show=false → <InputB /> : khác type ⇒ unmount, state mất
  // Cùng type nhưng key khác cũng remount:
  return show ? <Input key="a" /> : <Input key="b" />;
}
function Input() {
  const [v, setV] = useState("");
  return <input value={v} onChange={e => setV(e.target.value)} placeholder="gõ rồi toggle" />;
}
```

```tsx
// 3.3 key stable vs index — bug reorder
type Todo = { id: string; text: string };

function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {/* Sai: index làm key — thêm/xóa đầu list làm mọi item đổi key ⇒ update hết, mất focus/checkbox */}
      {/* {todos.map((t, i) => <li key={i}><input defaultChecked />{t.text}</li>)} */}

      {/* Đúng: id ổn định */}
      {todos.map(t => (
        <li key={t.id}>
          <input type="checkbox" /> {t.text}
        </li>
      ))}
    </ul>
  );
}

// 3.4 key không ở trong props
function Item(props: { id: string; label: string }) {
  // console.log((props as any).key); // undefined — key không vào props
  return <li>{props.label}</li>;
}
// <Item key="abc" id="abc" label="Hello" /> // phải truyền id riêng

// 3.5 Remount pattern — reset state khi id đổi
function UserProfile({ userId }: { userId: string }) {
  const [draft, setDraft] = useState("");
  // Không cần useEffect(() => setDraft(""), [userId]) — dùng key:
  return <input value={draft} onChange={e => setDraft(e.target.value)} placeholder={`draft for ${userId}`} />;
}
function App({ userId }: { userId: string }) {
  return <UserProfile key={userId} userId={userId} />; // đổi userId ⇒ UserProfile mount mới, draft reset
}

// 3.6 Tạo key sai — random trong render ⇒ remount mỗi render
function BadList({ todos }: { todos: Todo[] }) {
  return <>{todos.map(t => <li key={Math.random()}>{t.text}</li>)}</>; // ❌ key đổi mỗi render
}
function GoodList({ todos }: { todos: Todo[] }) {
  return <>{todos.map(t => <li key={t.id}>{t.text}</li>)}</>; // ✅
}
```

## 4. So sánh / Phân loại

| Tiêu chí | Render | Reconciliation | Commit |
|----------|--------|----------------|--------|
| Làm gì | Gọi component → tạo React Element tree | Diff `current` vs `workInProgress` Fiber | Mutate DOM, chạy effects |
| Có chạm DOM? | Không | Không | Có |
| Có thể interrupt? | Có (concurrent) | Có | Không (đồng bộ) |
| Side effect được phép? | Không (phải pure) | Không | Có (`useLayoutEffect`, DOM mutation) |
| Trigger | `setState`, `props`, `context`, `forceUpdate` | Sau render | Sau diff có flags `Placement/Update/Deletion` |

| Tiêu chí | `key={id}` ổn định | `key={index}` | `key={Math.random()}` | Không có `key` |
|----------|-------------------|---------------|----------------------|----------------|
| Identity khi reorder | Đúng — React move DOM, giữ state/focus | Sai — match theo vị trí, state lệch | Sai — mỗi render là identity mới, luôn remount | Như index (React dùng index fallback) |
| Performance | O(n) move tối thiểu | O(n) nhưng update thừa | Tệ nhất — unmount/mount hết | Kém khi list động |
| Khi dùng | List động, reorder, filter, pagination | Chỉ khi list **static**, không reorder/filter | Không bao giờ | Chỉ khi list static 1 lần |

| Tình huống | Dùng gì |
|------------|---------|
| List động (todo, table, kanban) | `key={stableId}` |
| Fragment trong list | `<React.Fragment key={id}>` (`<>` không nhận key) |
| Reset state khi prop đổi | `key={id}` trên component cần reset |
| List static, không thêm/xóa | `index` chấp nhận được |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không cố ngăn mọi re-render**: Render phase rẻ (chỉ tạo object trong JS, chưa đo layout). Chỉ tối ưu khi React DevTools Profiler chỉ ra bottleneck (commit > 16ms, re-render lan rộng).
- **Không dùng `key={index}` cho list động**: tưởng tiện nhưng gây bug focus, uncontrolled input, animation, và update thừa toàn list.
- **Không sinh `key` trong render**: `crypto.randomUUID()` phải gọi khi **tạo data** (submit / fetch), không phải lúc `map`.
- **Không đổi `key` bừa bãi**: `key` thay đổi ⇒ unmount/mount ⇒ mất state, chạy lại `useEffect`/`useLayoutEffect`, tốn mount cost (ref, subscription). Chỉ đổi `key` khi thực sự muốn reset.
- **Không đặt `key` để “fix” warning mà không hiểu scope**: `key` chỉ cần cho **array children** trong cùng parent, không cần cho single child.
- **VDOM không miễn phí**: với animation 60fps thao tác DOM trực tiếp hoặc Svelte (compile-time, no VDOM) có thể nhanh hơn. VDOM thắng ở DX, batching, declarative, và multi-target.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `key={index}` gây state lệch khi reorder/filter**
  - Triệu chứng: checkbox tick nhầm row, input mất focus, animation nhảy.
  - Fix: `key={todo.id}` stable. Tạo id khi tạo todo: `{ id: crypto.randomUUID(), text }`.
  - Đo: React DevTools → Components → highlight re-render, inspect Fiber `key`.

- **Lỗi 2: `key` random mỗi render gây remount liên tục**
  - Triệu chứng: input mất focus mỗi keystroke, `useEffect` cleanup chạy liên tục, performance sụt, hydration mismatch.
  - Fix: không `key={Math.random()}` hay `Date.now()` trong render.
  - Đo: Profiler → “Why did this render?” → mount/unmount count tăng; Console `key` warning.

- **Lỗi 3: Nhầm `key` có trong `props`**
  - Triệu chứng: `props.key === undefined`, logic phụ thuộc `key` không chạy.
  - Fix: truyền `id` riêng: `<Item key={id} id={id} />`.
  - Đo: `console.log(props)` hoặc TypeScript sẽ không có `key` trong `Props` type.

- **Lỗi 4: Khác `type` làm mất state không ngờ**
  - Triệu chứng: toggle `show ? <A /> : <B />` làm input mất giá trị.
  - Fix: nếu muốn giữ state, giữ cùng `type` và dùng props/conditional bên trong; nếu muốn reset, dùng `key` rõ ràng.
  - Đo: React DevTools → Components → state hook mất sau toggle.

- **Lỗi 5: `setState` trong render gây loop**
  - Triệu chứng: “Too many re-renders”.
  - Fix: không `setState` trong render body; chuyển vào `useEffect` hoặc event handler.
  - Đo: stack trace, Profiler commit loop.

- **Lỗi 6: Hydration mismatch do `key`/`Date`/`Math.random` khác server/client**
  - Triệu chứng: `Hydration failed ...`, layout shift, SSR bị discard.
  - Fix: đảm bảo `key` ổn định, hoãn `Date.now()`/`random` tới `useEffect` (chỉ client), hoặc `suppressHydrationWarning` cho text nhỏ.
  - Đo: `hydrateRoot(..., { onRecoverableError })`, console hydration warning.

- **Công cụ**:
  - React DevTools Profiler: Record → xem **Render duration**, **Commit**, **Why did this render?** (props/state/context change).
  - StrictMode: double-invoke render để phát hiện impure render.
  - ESLint: `react/no-array-index-key` cảnh báo `key={index}`.
  - Performance tab: measure `commit` > 16ms ⇒ cần memo/split.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt Render và Commit. Vì sao `setState` với giá trị `Object.is` bằng cũ (React 18) lại bail out không commit?
2. Vì sao Reconciliation đạt O(n) thay vì O(n³)? Hai giả định nào làm điều đó khả thi, và hậu quả nếu vi phạm?
3. Vì sao `key` không có trong `props`? Khi nào nên dùng `key` để force remount thay vì `useEffect` reset state, và tạo `key` đúng cách là gì?

<details>
<summary>Đáp án 30s</summary>

1. **Render** là gọi component để tạo React Element tree (pure, trong memory, có thể interrupt). **Commit** là áp mutations lên DOM và chạy effects (đồng bộ, block paint với layout effects). `setState(newVal)` so sánh `Object.is(prev, newVal)`; nếu bằng nhau React bỏ Reconciliation/Commit, không re-render subtree — tiết kiệm diff.
2. **O(n³)** là brute-force cây. React heuristic: (a) khác `type` thì rebuild cả subtree, (b) list diff dùng `key` để match. Nhờ đó chỉ duyệt một lần theo DFS trên Fiber. Vi phạm: không có `key`/dùng `index` khi reorder ⇒ match sai, update thừa, mất state/focus; khác `type` không ngờ ⇒ state mất.
3. `key` là hint cho **Reconciler**, không phải data cho component, nên React giữ riêng và không truyền vào `props`. Dùng `key` để remount khi muốn **reset toàn bộ** state/effect theo identity (VD: `<Form key={userId} />`) — sạch hơn `useEffect` reset rời rạc. Tạo `key` khi tạo data (`crypto.randomUUID()` ở create/submit), không tạo trong render.

</details>

---
*Tham khảo chi tiết: `docs/02-react-co-ban.md` — Câu 31, 32, 33, 50. Spec: [React Docs — Render and Commit](https://react.dev/learn/render-and-commit), [Reconciliation](https://react.dev/learn/preserving-and-resetting-state), [Lists and Keys](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key), [Fiber Architecture](https://github.com/acdlite/react-fiber-architecture).*
