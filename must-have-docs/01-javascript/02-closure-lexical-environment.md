# Closure và LexicalEnvironment — Hàm giữ tham chiếu tới Environment nơi nó được tạo

> Tags: #closure #lexical-environment #scope #memory | Nguồn: `docs/01-javascript-typescript.md` câu 2 | Mức: P0

## 1. Định nghĩa chính xác

**Closure** là khả năng một function giữ tham chiếu tới **LexicalEnvironment** nơi nó được tạo ra thông qua internal slot `[[Environment]]`, cho phép truy cập các binding của outer scope ngay cả khi outer function đã return và Execution Context của nó đã rời khỏi Call Stack. Mỗi closure là cặp `(function object + environment reference)`, không phải bản sao giá trị.

**LexicalEnvironment** là cấu trúc spec gồm `EnvironmentRecord` (bảng binding) và `outer` (tham chiếu tới environment cha), tạo thành chuỗi scope chain để resolve identifier.

## 2. Cơ chế hoạt động

### 2.1 Tạo và giữ `[[Environment]]`

1. **Khi engine tạo function** (FunctionCreate): gán `function.[[Environment]] = current LexicalEnvironment` (environment hiện tại tại điểm khai báo, không phải điểm gọi).
2. **Khi function được gọi**: tạo `Function Environment` mới, đặt `outer = function.[[Environment]]`. Mọi `GetIdentifierReference` sẽ đi theo chuỗi `outer` để tìm binding.
3. **Khi outer return**: Execution Context của outer bị pop khỏi stack, nhưng LexicalEnvironment không bị GC nếu còn ít nhất một closure giữ reference tới nó.

```
Global LE { a:1, outer:null }
   ↑
outer
   │
Outer LE { count:0, outer:Global }  ← [[Environment]] của inner
   ↑
outer
   │
Inner Function Environment { outer: Outer LE }
```

### 2.2 Private state và Module Pattern

Closure cho phép tạo private state mà không cần class field: biến `count` nằm trong EnvironmentRecord của `createCounter`, chỉ các hàm có `[[Environment]]` trỏ tới đó mới truy cập được.

### 2.3 Stale Closure

Trong React, mỗi render tạo một Function Environment mới. `useEffect`/`useCallback` đóng over giá trị của render đó. Nếu dependency array thiếu, callback giữ reference tới environment cũ (stale), dẫn tới đọc state/props cũ.

### 2.4 GC và Closure

Mark-and-Sweep GC bắt đầu từ roots, đi qua mọi closure's `[[Environment]]`. Biến trong outer chỉ được thu hồi khi không còn closure nào reachable giữ nó. Giữ closure vô hạn (cache, global array) = giữ toàn bộ outer Environment.

## 3. Ví dụ tối thiểu

```js
// 3.1 Cơ bản: private state
function createCounter() {
  let count = 0; // nằm trong LexicalEnvironment của createCounter
  return {
    inc: () => ++count,
    dec: () => --count,
    get: () => count
  };
}
const c = createCounter();
console.log(c.inc()); // 1
console.log(c.inc()); // 2
console.log(c.get()); // 2

// 3.2 Mỗi lần gọi tạo Environment riêng
const c2 = createCounter();
console.log(c2.get()); // 0 — Environment khác, không share

// 3.3 Memoize — closure giữ cache
function memoize(fn) {
  const cache = new Map(); // giữ trong closure
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const res = fn(...args);
    cache.set(key, res);
    return res;
  };
}
const fib = memoize((n) => n <= 1 ? n : fib(n-1) + fib(n-2));
console.log(fib(10)); // 55, lần sau lấy từ cache

// 3.4 Stale closure — ví dụ React-equivalent trong JS thuần
function createStaleDemo() {
  let count = 0;
  function getCount() { return count; }
  function inc() { count++; }
  return { getCount, inc };
}
const { getCount, inc } = createStaleDemo();
inc();
console.log(getCount()); // 1 — closure luôn thấy giá trị mới nhất (reference, không phải snapshot)
```

```js
// 3.5 Closure trong loop — let vs var (liên kết với 01-scope)
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log('var', i), 0); // 3 3 3
}
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log('let', i), 0); // 0 1 2
}

// 3.6 WeakMap để tránh leak khi gắn metadata
const meta = new WeakMap();
function attachMetadata(obj, data) {
  meta.set(obj, data); // weak reference tới obj
}
let el = { id: 1 };
attachMetadata(el, { big: new Array(1e6).fill(0) });
console.log(meta.has(el)); // true
el = null; // GC có thể thu hồi cả el và entry trong WeakMap
```

```tsx
// 3.7 Stale closure trong React
import { useState, useEffect, useCallback } from 'react';

function Timer() {
  const [count, setCount] = useState(0);

  // Stale nếu thiếu dep
  const logStale = useCallback(() => {
    console.log(count); // closure của render tạo ra logStale
  }, []); // [] → luôn log 0

  const logFresh = useCallback(() => {
    console.log(count);
  }, [count]); // đúng — tạo lại khi count đổi

  useEffect(() => {
    const id = setInterval(() => setCount(c => c + 1), 1000);
    return () => clearInterval(id);
  }, []);

  return <button onClick={logFresh}>log</button>;
}
```

## 4. So sánh / Phân loại

| Tiêu chí | Closure (Lexical) | `this` binding (Dynamic) |
|----------|-------------------|--------------------------|
| Quyết định bởi | Vị trí khai báo (static) | Nơi gọi hàm (call-site), trừ arrow |
| Lưu gì | Reference tới LexicalEnvironment | Giá trị `this` tại call |
| Arrow function | Có closure, không có `this` riêng | Lexical `this` từ outer |
| GC | Giữ toàn bộ outer bindings còn referenced | Không giữ scope |

| Mô hình private state | Cơ chế | Ưu điểm | Nhược điểm |
|-----------------------|--------|---------|------------|
| Closure (`createCounter`) | LexicalEnvironment | Đơn giản, không cần `this`, true private | Mỗi instance tạo lại function, tốn memory |
| Class `#privateField` | Private slot trên instance | Native private, prototype share method | Cần class syntax, không hỗ trợ IE |
| WeakMap | Weak reference key→value | Không ngăn GC, có thể gắn metadata cho object bất kỳ | Không enumerable, phải quản lý WeakMap riêng |

| Loại closure | Giữ gì | Nguy cơ leak |
|--------------|--------|--------------|
| Module pattern / factory | Biến local của factory | Thấp nếu factory không lưu global |
| Cache / memoize | `Map` trong closure | Cao nếu cache không giới hạn, không evict |
| Event listener closure | DOM node / component state | Cao nếu không cleanup khi unmount |
| Currying / partial | Args đã bind | Thấp, nhưng giữ reference tới args lớn nếu không release |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng closure để giữ state lớn vô hạn**: `memoize` không giới hạn, `Map` global cache giữ mọi kết quả → memory tăng không ngừng. Thay bằng `LRU`, `WeakMap`, hoặc TTL.
- **Không tạo closure trong hot path render**: `render() { return <Child onClick={() => handle(id)} /> }` tạo function mới mỗi render, phá `memo`/`PureComponent` (reference check fail) và tăng GC pressure. Dùng `useCallback` hoặc tách component.
- **Không dùng closure khi `class #privateField` hoặc `WeakMap` phù hợp hơn**: nếu cần nhiều instance share method, closure tạo copy function mỗi instance → tốn memory hơn prototype method + `#private`.
- **Không giữ closure tham chiếu DOM lớn sau khi remove**: `const handler = () => console.log(el.innerHTML)` giữ `el` dù đã `removeChild`. Phải nullify hoặc dùng `WeakRef`.
- **Khi nào nên dùng**: module pattern, once/debounce/memoize, currying, HOC/middleware, encapsulate private state đơn giản.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Stale closure trong React**
  - Triệu chứng: `useEffect`/`useCallback` đọc state cũ, `setInterval` không thấy state mới.
  - Fix: khai báo đủ deps, dùng `eslint-plugin-react-hooks` (`exhaustive-decs`), hoặc functional update `setCount(c => c+1)`, hoặc `useRef` để giữ latest value.
  - Đo: `eslint --rule 'react-hooks/exhaustive-deps: error'`, React DevTools → Components → hooks.

- **Lỗi 2: Memory leak do closure giữ DOM / large object**
  - Triệu chứng: Heap snapshot tăng sau mỗi mount/unmount, Detached DOM nodes.
  - Fix: cleanup trong `useEffect return`, `removeEventListener`, `clearInterval`, không lưu DOM node trong closure global.
  - Đo: Chrome DevTools → Memory → Heap snapshot → filter `Detached`, Allocation timeline, so sánh snapshot trước/sau.

- **Lỗi 3: Vòng lặp `var` + closure**
  - Triệu chứng: mọi callback in cùng giá trị cuối.
  - Fix: `let`, hoặc IIFE.
  - Đo: unit test với `setTimeout`/`Promise`.

- **Lỗi 4: Tạo quá nhiều closure trong render**
  - Triệu chứng: GC pause, `why-did-you-render` báo re-render liên tục.
  - Fix: `useCallback`, `useMemo`, tách handler ra ngoài.
  - Đo: Performance tab → JS Heap, `performance.measure`, `wdyr`.

- **Lỗi 5: WeakMap vs Map nhầm lẫn**
  - Triệu chứng: dùng `Map` để cache DOM metadata → leak vì `Map` giữ strong reference.
  - Fix: đổi sang `WeakMap` nếu key là object và không cần iterate.
  - Đo: Heap snapshot → xem `Map` entries còn reachable sau khi DOM remove.

## 7. Câu hỏi tự kiểm tra

1. Closure giữ **giá trị** hay **tham chiếu** tới LexicalEnvironment? Nếu outer biến thay đổi sau khi closure tạo, closure thấy giá trị mới hay cũ?
2. Vì sao `useCallback(() => console.log(count), [])` luôn log giá trị `count` ban đầu, và cách fix đúng là gì?
3. Khi nào nên dùng `WeakMap` thay vì `Map` để lưu cache gắn với object, và `WeakMap` có `size` hay iterable không?

<details>
<summary>Đáp án 30s</summary>

1. Giữ **tham chiếu** tới LexicalEnvironment (qua `[[Environment]]`), không phải snapshot. Nên mọi thay đổi sau đó đều thấy — `count++` rồi `closure()` sẽ thấy giá trị mới.
2. Vì mỗi render tạo Environment mới, `useCallback` với `[]` chỉ tạo một lần và `[[Environment]]` trỏ tới render đầu (stale). Fix: thêm `count` vào deps `[count]`, hoặc dùng `useRef` + `ref.current = count` để closure đọc `ref.current` (mutable, không cần recreate).
3. Dùng `WeakMap` khi key là object và muốn entry tự biến mất khi key không còn reachable (tránh leak, không cần manual evict). `WeakMap` **không** có `size`, không iterable (`keys/values/entries`), không `clear` — vì GC có thể xóa bất kỳ lúc nào nên không thể liệt kê.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 2. Spec: [ECMA-262 §9.4 Closures](https://tc39.es/ecma262/#sec-closure), [§8.1 Lexical Environments](https://tc39.es/ecma262/#sec-lexical-environments). Bài liên quan: `01-scope-hoisting-tdz.md`.*
