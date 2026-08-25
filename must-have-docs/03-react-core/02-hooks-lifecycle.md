# Hooks Lifecycle — useState batching, useEffect vs useLayoutEffect vs useInsertionEffect, deps & stale closure

> Tags: #hooks #useState #useEffect #useLayoutEffect #useInsertionEffect #deps #stale-closure #abort | Nguồn: `docs/02-react-co-ban.md` câu 35, 36, 37 | Mức: P0

## 1. Định nghĩa chính xác

- **useState**: Hook lưu state cục bộ theo **Fiber**. `const [state, setState] = useState(initial)` — `setState` enqueue update, React **batch** nhiều update thành một render, so sánh `Object.is` để bail out.
- **useEffect**: Hook đồng bộ với **hệ ngoài React** (network, subscription, DOM) — chạy **bất đồng bộ sau paint** (passive effect), không block paint. Signature `useEffect(setup, deps)`, `setup` có thể trả `cleanup`.
- **useLayoutEffect**: biến thể chạy **đồng bộ sau DOM mutation nhưng trước paint**, block paint — dùng để đo và mutate DOM không gây flicker.
- **useInsertionEffect** (React 18+): chạy **đồng bộ trước khi DOM mutation**, trước cả `useLayoutEffect` — dành riêng cho **CSS-in-JS** inject `<style>` trước khi browser tính layout.
- **Dependency array (deps)**: danh sách giá trị mà effect **đồng bộ theo**. React so sánh từng deps bằng `Object.is` giữa các render để quyết định re-run.
- **Stale closure**: effect/callback bắt giữ giá trị cũ do closure tại thời điểm tạo, không cập nhật khi deps thiếu.

## 2. Cơ chế hoạt động

### 2.1 useState — batching & functional update

- `setState(value)` hoặc `setState(prev => next)` đều **enqueue** vào Fiber update queue (linked list). Trong cùng tick, nhiều `setState` được **batch** thành một render.
- React 17: chỉ batch trong React event handler. React 18+: **automatic batching** cho cả `setTimeout`, `Promise`, `native event`.
- `setState` so sánh `Object.is(old, new)` — bằng nhau thì bail out, không render subtree.
- **Functional update** `setCount(c => c + 1)` nhận **prev mới nhất** từ queue, tránh stale khi gọi nhiều lần liên tiếp. **Lazy initializer** `useState(() => expensiveCalc())` chỉ chạy một lần mount.

```
click → setCount(c=>c+1) → enqueue → setCount(c=>c+1) → enqueue
→ batch → process queue: c0→c1→c2 → single render → commit
```

### 2.2 Ba effect theo timeline Commit

```
Render → DOM mutation (insert/update/delete)
  → useInsertionEffect (inject style)          // trước mutation layout
  → useLayoutEffect (đo DOM, mutate đồng bộ)   // sau mutation, trước paint — block paint
  → Browser paint
  → useEffect (fetch, subscription, log)       // sau paint — không block
  → (unmount / deps đổi) → cleanup của effect cũ chạy trước setup mới
```

- **Mental model**: `useEffect` không phải `componentDidMount` mà là **“đồng bộ effect với deps”**. Mỗi render có closure/effect riêng.
- **Cleanup** chạy trước effect tiếp theo và khi unmount — nơi `abort()`, `unsubscribe`, `clearInterval`.

### 2.3 Deps exhaustive & ESLint

- `react-hooks/exhaustive-deps` yêu cầu liệt kê **mọi giá trị** từ scope component được dùng trong effect (props, state, derived, function). Thiếu ⇒ stale; thừa ⇒ loop.
- Nếu effect không nên re-run khi function đổi, bọc function bằng `useCallback` hoặc đưa logic ra ngoài, hoặc dùng `useEffectEvent` (React 19 experimental) để đọc giá trị mới nhất mà không add vào deps.

### 2.4 Stale closure & AbortController

- Closure bắt giá trị tại thời điểm effect tạo. Nếu `deps=[]` mà trong effect đọc `count`, `count` mãi là giá trị mount.
- `AbortController` trong cleanup giúp hủy `fetch` khi `roomId` đổi nhanh, tránh **race condition** (response cũ ghi đè response mới).

### 2.5 Fiber & Hook storage

Mỗi Fiber có `memoizedState` là linked list các Hook (`Hook { memoizedState, queue, next }`). Gọi hooks đúng thứ tự giúp React map đúng Hook node — vi phạm Rules of Hooks làm mismatch.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 useState batching + functional update (React 18 batch cả async)
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  const [label, setLabel] = useState("idle");

  const handleClick = () => {
    // Không batch? Không — React 18 batch thành 1 render
    // Với value form: chỉ tăng 1 (đều dựa trên count cũ)
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1); // kết quả: count +1

    // Functional: mỗi updater nhận prev mới nhất → +3
    setCount(c => c + 1);
    setCount(c => c + 1);
    setCount(c => c + 1); // kết quả: +3 nữa
  };

  const handleAsync = () => {
    setTimeout(() => {
      setCount(c => c + 1);
      setCount(c => c + 1); // vẫn batch thành 1 render (React 18)
    }, 0);
  };

  return (
    <>
      <button onClick={handleClick}>sync batch: {count}</button>
      <button onClick={handleAsync}>async batch</button>
    </>
  );
}

// 3.2 Lazy initializer — chỉ tính 1 lần
function ExpensiveInit({ id }: { id: string }) {
  const [data] = useState(() => {
    // expensiveCalc chỉ chạy mount, không chạy lại mỗi render
    return { computed: id.toUpperCase() };
  });
  return <div>{data.computed}</div>;
}
```

```tsx
// 3.3 useEffect với cleanup + AbortController (tránh race)
import { useEffect, useState } from "react";

function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<string[]>([]);

  useEffect(() => {
    const controller = new AbortController();

    async function load() {
      try {
        const res = await fetch(`/api/room/${roomId}`, { signal: controller.signal });
        const data = (await res.json()) as string[];
        setMessages(data);
      } catch (e: unknown) {
        if (e instanceof DOMException && e.name === "AbortError") return; // bị abort — bỏ qua
        console.error(e);
      }
    }
    load();

    // Giả lập subscription
    const conn = { connect: () => console.log("connect", roomId), disconnect: () => console.log("disconnect", roomId) };
    conn.connect();

    return () => {
      controller.abort(); // hủy fetch khi roomId đổi / unmount
      conn.disconnect();
    };
  }, [roomId]); // exhaustive deps: roomId

  return <ul>{messages.map(m => <li key={m}>{m}</li>)}</ul>;
}

// 3.4 Stale closure — bug kinh điển
function StaleCounter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // closure bắt count lúc mount (0) nếu deps=[]
    }, 1000);
    return () => clearInterval(id);
  }, []); // ❌ thiếu count → luôn 0
  // Fix 1: thêm count vào deps (interval sẽ reset mỗi khi count đổi)
  // Fix 2: functional update / ref
}

function FixedWithRef() {
  const [count, setCount] = useState(0);
  const countRef = useState(() => ({ current: count }))[0] as React.MutableRefObject<number>;
  // Hoặc: const countRef = useRef(count);
  useEffect(() => { countRef.current = count; });
  useEffect(() => {
    const id = setInterval(() => console.log("ref:", countRef.current), 1000);
    return () => clearInterval(id);
  }, []); // ref luôn mới nhất, không cần deps
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// 3.5 useLayoutEffect vs useEffect — đo DOM không flicker
import { useLayoutEffect, useRef, useState } from "react";

function Tooltip({ targetRef }: { targetRef: React.RefObject<HTMLElement | null> }) {
  const tipRef = useRef<HTMLDivElement>(null);
  const [left, setLeft] = useState(0);

  useLayoutEffect(() => {
    if (!targetRef.current || !tipRef.current) return;
    const rect = targetRef.current.getBoundingClientRect();
    const tipRect = tipRef.current.getBoundingClientRect();
    // Tính trước paint → không flicker
    setLeft(rect.left + (rect.width - tipRect.width) / 2);
  }, [targetRef]);

  return <div ref={tipRef} style={{ position: "absolute", left }}>Tooltip</div>;
}

// 3.6 useInsertionEffect — CSS-in-JS inject
import { useInsertionEffect } from "react";

function useInjectStyle(css: string) {
  useInsertionEffect(() => {
    const el = document.createElement("style");
    el.textContent = css;
    document.head.appendChild(el);
    return () => { el.remove(); };
  }, [css]);
}
```

## 4. So sánh / Phân loại

| Tiêu chí | `useEffect` | `useLayoutEffect` | `useInsertionEffect` |
|----------|-------------|-------------------|----------------------|
| Thời điểm | Sau paint, bất đồng bộ | Sau DOM mutation, trước paint, đồng bộ block paint | Trước DOM mutation, đồng bộ block |
| Block paint? | Không | Có | Có (nhưng chỉ inject style) |
| Dùng cho | Fetch, subscription, log, sync với hệ ngoài | Đo DOM (`getBoundingClientRect`), mutate tránh flicker, scroll lock | CSS-in-JS inject `<style>` |
| SSR | Chạy chỉ client, không warning | Warning trên server (không có DOM) — cần guard hoặc `useEffect` fallback | Tương tự layout |
| Ví dụ | `fetch`, `addEventListener` | Tooltip positioning, auto-focus đo layout | `styled-components`, `emotion` |

| `setCount(count+1)` lặp 3 lần | `setCount(c=>c+1)` lặp 3 lần | `flushSync` |
|-------------------------------|------------------------------|-------------|
| Dựa trên closure `count` cũ → kết quả +1 | Mỗi updater nhận prev mới nhất → +3 | Ép commit đồng bộ, thoát batching — dùng hiếm (đo DOM ngay sau setState) |

| Thiếu deps | Thừa deps | Đúng exhaustive |
|-----------|-----------|-----------------|
| Stale closure, data cũ, race | Loop vô hạn, fetch liên tục | Effect đồng bộ đúng giá trị, chạy khi cần |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng `useEffect` cho derived state**: nếu giá trị tính được từ props/state trong render, tính trực tiếp — không `useEffect(() => setX(...), [a,b])`. Đó là anti-pattern gây render thừa.
- **Không dùng `useEffect` cho event**: logic chỉ chạy khi user click/submit nên ở **event handler**, không phải effect.
- **Không dùng `useLayoutEffect` cho fetch/subscription**: block paint vô ích, làm INP tệ. Mặc định dùng `useEffect`.
- **Không dùng `useInsertionEffect` ngoài CSS-in-JS**: không dành cho logic app, chỉ lib style.
- **Không bỏ `exhaustive-deps`**: disable ESLint để “fix” warning là che bug. Nếu deps gây loop, tách function bằng `useCallback` hoặc đưa constant ra ngoài.
- **Không `setState` đồng bộ nhiều lần với value form**: dùng functional update khi next state phụ thuộc prev.
- **Chi phí**: mỗi `useEffect` thêm subscription/cleanup phải quản lý. Effect nhiều gây khó trace; ưu tiên colocation và tách custom hook.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Stale closure do thiếu deps**
  - Triệu chứng: interval/log/fetch dùng giá trị cũ.
  - Fix: thêm deps, hoặc dùng `useRef` để đọc mới nhất, hoặc functional update.
  - Đo: ESLint `react-hooks/exhaustive-deps` (error), React DevTools → Components → hooks values.

- **Lỗi 2: Race condition fetch khi `roomId` đổi nhanh**
  - Triệu chứng: response cũ ghi đè mới, UI hiển thị room sai.
  - Fix: `AbortController` trong cleanup + `signal`; hoặc sequence number check.
  - Đo: Network tab — thấy 2 fetch, UI sai; console log `AbortError`.

- **Lỗi 3: Quên cleanup gây leak**
  - Triệu chứng: `setState` trên unmounted component, memory tăng, WebSocket duplicate.
  - Fix: `return () => { controller.abort(); conn.disconnect(); clearInterval(id); }`.
  - Đo: StrictMode double-mount trong dev sẽ lộ thiếu cleanup; Memory profiler, console warning.

- **Lỗi 4: `useLayoutEffect` làm drop frame**
  - Triệu chứng: INP cao, jank khi tooltip/list lớn.
  - Fix: chỉ đo DOM nhẹ trong `useLayoutEffect`; việc nặng chuyển `useEffect` hoặc `requestAnimationFrame`.
  - Đo: Performance tab → Long task, INP, React Profiler commit duration.

- **Lỗi 5: SSR warning `useLayoutEffect does nothing on the server`**
  - Triệu chứng: warning khi SSR/Next.js.
  - Fix: dùng `useEffect` cho client-only, hoặc `useIsomorphicLayoutEffect = typeof window !== "undefined" ? useLayoutEffect : useEffect`.
  - Đo: build log, hydration.

- **Lỗi 6: Infinite loop do object deps**
  - Triệu chứng: `useEffect(..., [obj])` với `obj` tạo mới mỗi render ⇒ effect chạy mỗi render ⇒ `setState` ⇒ loop.
  - Fix: memo `obj` bằng `useMemo`, hoặc chỉ deps primitive (`obj.id`), hoặc tách riêng.
  - Đo: Profiler → commit liên tục, console log loop.

- **Công cụ**:
  - ESLint: `eslint-plugin-react-hooks` (`rules-of-hooks`, `exhaustive-deps`).
  - React DevTools: Profiler, Components (inspect hooks state), highlight updates.
  - `AbortController`, `performance.now()` đo fetch race, `why-did-you-render` cho deps.

## 7. Câu hỏi tự kiểm tra

1. Vì sao `setCount(count + 1)` gọi 3 lần liên tiếp chỉ tăng 1, còn `setCount(c => c + 1)` 3 lần lại tăng 3? Liên quan gì tới batching React 18?
2. Phân biệt `useEffect`, `useLayoutEffect`, `useInsertionEffect` về thời điểm và khi nào dùng mỗi loại? Vì sao không dùng `useLayoutEffect` cho fetch?
3. `exhaustive-deps` báo thiếu `fetchData` trong `useEffect`. Vì sao không nên disable ESLint mà nên `useCallback`? Và cách dùng `AbortController` để tránh race khi `roomId` đổi nhanh?

<details>
<summary>Đáp án 30s</summary>

1. `setCount(count+1)` closure bắt `count` cũ, 3 lần enqueue cùng giá trị → batch thành một update → +1. `setCount(c=>c+1)` enqueue 3 updater function, mỗi cái nhận `prev` mới nhất sau khi reducer trước chạy → +3. React 18 automatic batching gộp cả trong `setTimeout`/`Promise` thành một render; trước React 17 chỉ batch trong event handler.
2. `useInsertionEffect` chạy trước DOM mutation — inject style cho CSS-in-JS. `useLayoutEffect` chạy sau mutation trước paint, đồng bộ block paint — đo DOM và mutate tránh flicker. `useEffect` chạy sau paint, không block — fetch/subscribe/log. Không dùng `useLayoutEffect` cho fetch vì block paint vô ích, làm chậm FCP/INP, không cần đo DOM.
3. Disable ESLint che stale closure — `fetchData` là dependency vì được dùng trong effect, nếu không liệt kê sẽ bắt closure cũ. Bọc `fetchData` bằng `useCallback` với deps đúng để identity ổn định, hoặc inline logic vào effect. `AbortController`: tạo `controller` trong effect, truyền `signal` vào `fetch`, `return () => controller.abort()` — khi `roomId` đổi, cleanup abort request cũ, trong `catch` bỏ qua `AbortError`, chỉ `setState` với response của request hiện tại.

</details>

---
*Tham khảo chi tiết: `docs/02-react-co-ban.md` — Câu 35, 36, 37. Spec: [React Docs — useState](https://react.dev/reference/react/useState), [useEffect](https://react.dev/reference/react/useEffect), [useLayoutEffect](https://react.dev/reference/react/useLayoutEffect), [useInsertionEffect](https://react.dev/reference/react/useInsertionEffect), [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects).*
