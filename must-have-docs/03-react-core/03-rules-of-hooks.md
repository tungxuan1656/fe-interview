# Rules of Hooks — Top-level, Fiber linked list, custom hook và ESLint

> Tags: #hooks #rules-of-hooks #fiber #custom-hook #eslint | Nguồn: `docs/02-react-co-ban.md` câu 49 | Mức: P0

## 1. Định nghĩa chính xác

**Rules of Hooks** là hai quy tắc bắt buộc để Hooks hoạt động deterministically:

1. **Chỉ gọi Hooks ở top-level**: không gọi trong `if`, `for`, `while`, nested function, hay sau `return` sớm. Mỗi render phải gọi **cùng số lượng và cùng thứ tự** Hooks.
2. **Chỉ gọi Hooks từ React function**: function component hoặc **custom Hook** (hàm bắt đầu bằng `use`). Không gọi từ function thường, class, hay handler.

Vi phạm làm **Hook order mismatch** — React map nhầm state/effect giữa các Hook node, gây bug state lệch, infinite loop, hoặc crash “Rendered fewer hooks than expected”.

**Custom Hook** là hàm `useX` tự định nghĩa, được phép gọi Hooks khác bên trong, để chia sẻ **logic có state/effect** giữa nhiều component mà không chia sẻ state instance.

## 2. Cơ chế hoạt động

### 2.1 Fiber lưu Hooks bằng linked list theo thứ tự gọi

Mỗi Fiber có `memoizedState` trỏ tới danh sách Hook:

```
Fiber.memoizedState → Hook1 { memoizedState, queue, next } 
                        → Hook2 { memoizedState, queue, next }
                        → Hook3 → null
```

React không lưu Hooks theo tên mà theo **chỉ số thứ tự** (cursor). Lần render đầu (mount), React tạo list theo thứ tự gọi. Lần sau (update), React duyệt list theo cùng thứ tự để trả đúng `memoizedState`/`queue` cho mỗi Hook call.

```
mount: useState(0)  → Hook1 (count)
       useEffect(..) → Hook2
       useState("")  → Hook3 (text)

update: phải gọi đúng 3 Hooks theo đúng thứ tự mới map đúng Hook1/2/3.
Nếu render sau gọi thiếu/đảo thứ tự, Hook3 sẽ nhận nhầm Hook2 → bug.
```

Do đó `if (cond) useState()` là sai: khi `cond` đổi, số lượng/thứ tự Hook đổi ⇒ mismatch.

Pseudo của `useState`/`useEffect` (đơn giản hóa):

```ts
let currentlyRenderingFiber: Fiber | null = null;
let workInProgressHook: Hook | null = null;

function useState<S>(initial: S) {
  let hook: Hook;
  if (currentlyRenderingFiber!.alternate === null) {
    // mount: tạo Hook mới, push vào list
    hook = { memoizedState: initial, queue: [], next: null };
    appendHookToList(hook);
  } else {
    // update: lấy Hook tiếp theo theo thứ tự
    hook = workInProgressHook!.next!;
    workInProgressHook = hook;
  }
  return [hook.memoizedState, dispatch.bind(hook.queue)];
}
```

### 2.2 Vì sao chỉ từ React function?

React biết `currentlyRenderingFiber` chỉ khi đang render component hoặc custom Hook được gọi **trong** component. Gọi Hook từ handler thường (`onClick() { useState() }`) thì không có Fiber context, không có linked list để gắn.

### 2.3 Custom Hook — chia sẻ logic, không chia sẻ state

Custom Hook là **convention** `use*` — mỗi lần component gọi `useCounter()` sẽ tạo **instance Hook riêng** trên Fiber đó. Không có singleton. Custom Hook có thể gọi Hooks khác, và tuân cùng Rules.

ESLint `eslint-plugin-react-hooks` enforce cả hai rules bằng static analysis: phát hiện Hook sau `if/return`, trong loop, và `exhaustive-deps`.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Sai: Hook trong condition — thứ tự đổi khi cond đổi
import { useState, useEffect } from "react";

function Bad({ cond }: { cond: boolean }) {
  if (cond) {
    const [count, setCount] = useState(0); // ❌ Hook order đổi khi cond false → mismatch
    useEffect(() => {}, []);
  }
  return <div>{String(cond)}</div>;
}

// 3.2 Đúng: luôn gọi Hooks, condition bên trong
function Good({ cond }: { cond: boolean }) {
  const [count, setCount] = useState(0); // ✅ luôn gọi
  useEffect(() => {
    if (cond) {
      // logic chỉ khi cond true
      console.log(count);
    }
  }, [cond, count]);
  return <div>{count}</div>;
}

// 3.3 Sai: Hook trong loop — mỗi item số lượng Hook khác
function BadList({ items }: { items: string[] }) {
  // items.forEach(item => {
  //   const [v, setV] = useState(item); // ❌ thứ tự phụ thuộc items.length
  // });
  return <div>bad</div>;
}
// Đúng: tách component — mỗi Item có Fiber và Hook list riêng
function Item({ item }: { item: string }) {
  const [v, setV] = useState(item); // ✅ mỗi Item instance có Hooks riêng
  return <div>{v}</div>;
}
function GoodList({ items }: { items: string[] }) {
  return <>{items.map(item => <Item key={item} item={item} />)}</>;
}

// 3.4 Sai: early return trước Hooks
function BadEarlyReturn({ loading }: { loading: boolean }) {
  if (loading) return <div>Loading</div>; // ❌ return trước Hooks
  const [data, setData] = useState(null);
  return <div>{String(data)}</div>;
}
function GoodEarlyReturn({ loading }: { loading: boolean }) {
  const [data, setData] = useState<string | null>(null); // ✅ Hooks trước mọi return
  if (loading) return <div>Loading</div>;
  return <div>{data}</div>;
}
```

```tsx
// 3.5 Custom Hook đúng — chia sẻ logic, mỗi component có state riêng
import { useState, useCallback, useEffect, useRef } from "react";

function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const inc = useCallback(() => setCount(c => c + 1), []);
  const dec = useCallback(() => setCount(c => c - 1), []);
  return { count, inc, dec, setCount } as const;
}

function CounterA() {
  const { count, inc } = useCounter(0); // instance riêng
  return <button onClick={inc}>{count} (A)</button>;
}
function CounterB() {
  const { count, inc } = useCounter(100); // instance riêng, không share với A
  return <button onClick={inc}>{count} (B)</button>;
}

// 3.6 Custom Hook với effect — usePrevious
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  useEffect(() => {
    ref.current = value; // update sau render
  }, [value]);
  return ref.current;
}

// 3.7 Custom Hook phải bắt đầu bằng "use" để ESLint nhận diện
// function counterLogic() { useState(0); } // ❌ ESLint báo: Hook called in non-Hook function
// function useCounterLogic() { useState(0); } // ✅
```

```tsx
// 3.8 ESLint plugin — cấu hình
// .eslintrc.json
// {
//   "plugins": ["react-hooks"],
//   "rules": {
//     "react-hooks/rules-of-hooks": "error",
//     "react-hooks/exhaustive-deps": "warn"
//   }
// }
// rules-of-hooks bắt: if/for sau Hook, early return trước Hook, Hook ngoài component/custom Hook
// exhaustive-deps bắt: thiếu/thừa deps trong useEffect/useMemo/useCallback
```

## 4. So sánh / Phân loại

| Tiêu chí | Gọi Hooks đúng | Vi phạm Rules |
|----------|----------------|---------------|
| Vị trí | Top-level của component/custom Hook | Trong `if`/`for`/`nested fn`/sau `return` |
| Thứ tự mỗi render | Giữ nguyên | Thay đổi → mismatch |
| Kết quả | Map đúng `memoizedState` | State lệch, effect sai, crash |
| Phát hiện | ESLint pass | ESLint error + runtime “Rendered fewer hooks” |

| Loại hàm | Được gọi Hooks? | Ví dụ |
|----------|-----------------|-------|
| Function component | Có | `function Comp() { useState() }` |
| Custom Hook (`use*`) | Có | `function useAuth() { useState() }` |
| Hàm thường | Không | `function format() { useState() }` ❌ |
| Class component | Không | `class C extends Component { useState() }` ❌ |
| Event handler / `useEffect` callback | Không (trừ khi handler itself là custom Hook) | `onClick(){ useState() }` ❌ |

| Chia sẻ | Custom Hook | HOC | Render Props |
|---------|-------------|-----|--------------|
| Cơ chế | Hàm gọi Hooks, mỗi caller có instance riêng | Wrap component | Prop là function |
| Khi dùng | Logic có state/effect tái sử dụng (fetch, form, subscription) | Legacy, ít dùng | Legacy trước Hooks |
| Trade-off | Đơn giản nhất, không wrapper hell | Wrapper hell, prop collision | Nesting hell |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không “lách” Rules để làm code ngắn hơn**: `if (cond) useState()` có vẻ gọn nhưng phá determinism. Luôn viết verbose hơn để giữ thứ tự.
- **Không tạo custom Hook khi logic không dùng Hooks**: nếu chỉ là pure function (`formatDate`, `calcPrice`) không gọi Hooks, để là hàm thường — không cần `use`.
- **Không chia sẻ state qua custom Hook**: `useCounter()` ở hai component là hai state khác nhau. Muốn share state thực sự, dùng Context/Zustand/lift state, không phải custom Hook.
- **Không gọi Hook trong `map` trực tiếp**: tách thành `<Item>` component.
- **Không disable `rules-of-hooks`**: đó là invariant của Reconciler, không phải style. Disable là bug.
- **React Compiler (React 19+) nới lỏng** một số trường hợp (memo tự động), nhưng Rules of Hooks vẫn giữ — Compiler không bỏ linked-list model.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: “Rendered fewer hooks than expected” / “Rendered more hooks”**
  - Triệu chứng: crash khi `cond` đổi, hoặc HMR reload.
  - Nguyên nhân: Hook trong `if` hoặc early return trước Hook.
  - Fix: move Hooks lên đầu, condition vào trong Hook/effect.
  - Đo: ESLint `react-hooks/rules-of-hooks: error` bắt ngay khi code; runtime error stack trỏ đúng Hook.

- **Lỗi 2: State lệch sau khi vi phạm thứ tự**
  - Triệu chứng: `count` hiển thị giá trị của `text`, toggle gây state nhảy.
  - Nguyên nhân: thứ tự Hook đổi, `memoizedState` map nhầm.
  - Fix: đảm bảo mỗi render cùng thứ tự; không `forEach` Hook.
  - Đo: React DevTools → Components → hooks order, log `useState` values.

- **Lỗi 3: Gọi Hook trong handler thay vì component**
  - Triệu chứng: “Invalid hook call. Hooks can only be called inside of the body of a function component.”
  - Fix: chuyển Hook lên component, handler chỉ gọi setter.
  - Đo: console error, call stack.

- **Lỗi 4: Custom Hook không bắt đầu `use`**
  - Triệu chứng: ESLint không check, vi phạm không báo, bug lọt.
  - Fix: đặt tên `use*`, cấu hình ESLint `react-hooks`.
  - Đo: `eslint --ext .ts,.tsx` báo `React Hook "..." is called in function "..." that is neither a React function component nor a custom React Hook function`.

- **Lỗi 5: Hook trong `map` trực tiếp**
  - Triệu chứng: số Hook phụ thuộc `items.length`, thay đổi khi list đổi.
  - Fix: tách `Item` component.
  - Đo: ESLint báo, hoặc bug khi thêm/xóa item.

- **Công cụ**:
  - ESLint: `eslint-plugin-react-hooks` (`rules-of-hooks: error`, `exhaustive-deps: warn`).
  - React DevTools: highlight hooks, show `memoizedState`.
  - StrictMode: double-render giúp lộ bug order.

## 7. Câu hỏi tự kiểm tra

1. Vì sao React lưu Hooks theo thứ tự gọi (linked list) thay vì theo tên? Điều gì xảy ra nếu gọi `if (cond) useState()`?
2. Phân biệt custom Hook với HOC/Render Props. Vì sao `useCounter()` ở hai component không chia sẻ state?
3. ESLint `rules-of-hooks` và `exhaustive-deps` bắt lỗi gì khác nhau? Cho ví dụ mỗi loại.

<details>
<summary>Đáp án 30s</summary>

1. Fiber lưu `memoizedState` là linked list, cursor duyệt theo chỉ số. Không có key/tên, chỉ có thứ tự. Nếu `if (cond) useState()` thì khi `cond` đổi, số/thứ tự Hook đổi ⇒ cursor map nhầm Hook node cũ sang call mới ⇒ state/effect lệch hoặc crash “Rendered fewer hooks”.
2. Custom Hook là hàm `use*` gọi Hooks, mỗi caller (component) có Fiber và linked list riêng nên state riêng — chỉ chia sẻ **logic**, không chia sẻ **instance**. HOC/Render Props chia sẻ logic bằng composition nhưng gây wrapper/nesting hell, Hooks gọn hơn. Muốn share state thực sự phải lift/Context/store.
3. `rules-of-hooks` bắt **vị trí** sai: Hook trong `if`/`for`/`nested`/`sau return`, hoặc Hook ngoài component/custom Hook. `exhaustive-deps` bắt **deps** sai trong `useEffect`/`useMemo`/`useCallback`: thiếu deps gây stale, thừa deps gây loop. Ví dụ: `if (x) useState()` ⇒ rules-of-hooks; `useEffect(()=>console.log(x), [])` thiếu `x` ⇒ exhaustive-deps.

</details>

---
*Tham khảo chi tiết: `docs/02-react-co-ban.md` — Câu 49. Spec: [React Docs — Rules of Hooks](https://react.dev/warnings/invalid-hook-call-warning), [Building Your Own Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks), [React Fiber Architecture — Hooks](https://github.com/acdlite/react-fiber-architecture), [eslint-plugin-react-hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks).*
