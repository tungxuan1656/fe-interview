# Memory Leak và Garbage Collection — Mark-and-Sweep, leak sources và WeakRef

> Tags: #gc #memory-leak #weakmap #weakref #mark-and-sweep | Nguồn: `docs/01-javascript-typescript.md` câu 14 | Mức: P0

## 1. Định nghĩa chính xác

**Garbage Collection (GC)** trong JS engine là cơ chế tự động thu hồi vùng nhớ của object không còn **reachable** từ roots (`globalThis`, Call Stack, `WeakMap` không tính). Engine hiện đại dùng **Mark-and-Sweep**: từ roots mark mọi object còn đi tới được qua reference, phần không được mark thì sweep (thu hồi). Thuật toán cũ **Reference Counting** đếm số reference, nhưng thất bại với **circular reference** (`a↔b` dù không reachable vẫn count>0).

**Memory Leak** là tình trạng giữ reference vô tình tới object không còn cần, làm GC không thu hồi, heap tăng theo thời gian, dẫn tới OOM hoặc GC pause dài.

## 2. Cơ chế hoạt động

### 2.1 Mark-and-Sweep

1. **Mark**: từ roots (window, stack, microtask queue, DOM) DFS qua mọi property, `[[Environment]]` của closure, `Map`/`Set` entries.
2. **Sweep**: thu hồi object không được mark, trả về heap/free list. Có thể compact (di chuyển object) để giảm fragmentation.
3. **Generational**: V8 chia heap thành Young (nursery, Scavenge) và Old (Mark-Sweep-Compact). Object mới ở Young, sống sót qua GC thì promote lên Old.

### 2.2 Reference Counting (đã obsolete)

Mỗi object có `refCount`. `a = obj` → `count++`, `a = null` → `count--`. Khi `count===0` thì thu hồi. Vấn đề: `const a={}; const b={}; a.ref=b; b.ref=a; a=null; b=null` → `count` mỗi cái vẫn 1, không thu hồi dù không reachable.

### 2.3 4 nguồn leak kinh điển (SPA)

1. **Closure giữ DOM lớn / large object**: handler đóng over `el` dù `el` đã remove.
2. **Quên cleanup**: `setInterval`, `addEventListener`, `MutationObserver`, `WebSocket`, `IntersectionObserver` không `remove`/`disconnect`/`clear` khi unmount.
3. **Global variable / cache không giới hạn**: `window.cache = []`, `Map` cache tăng mãi.
4. **Detached DOM**: giữ reference tới subtree đã remove khỏi document (`let detached = el; el.remove()` nhưng `detached` vẫn giữ cả subtree).

### 2.4 WeakMap / WeakRef / FinalizationRegistry

- `WeakMap`/`WeakSet`: key là object, giữ **weak reference** — không ngăn GC, không enumerable, không `size`.
- `WeakRef(target)`: tạo weak reference tới object, `deref()` trả về object nếu còn sống, không ngăn GC.
- `FinalizationRegistry(callback)`: callback khi object được GC (không đảm bảo thời gian, không nên dùng cho logic chính, chỉ cleanup ngoài JS như free WASM memory).

## 3. Ví dụ tối thiểu

```js
// 3.1 Mark-and-Sweep vs Reference Counting — circular không leak với Mark-and-Sweep
let a = { name: 'a' };
let b = { name: 'b' };
a.ref = b;
b.ref = a;
a = null;
b = null;
// Reference Counting: a và b count vẫn 1 → leak
// Mark-and-Sweep: từ roots không tới được a/b → sweep → không leak

// 3.2 Leak 1: closure giữ DOM
function leakClosure() {
  const el = document.createElement('div');
  el.innerHTML = 'x'.repeat(1e6); // large
  document.body.appendChild(el);
  // handler giữ el
  const handler = () => console.log(el.innerHTML);
  window.addEventListener('resize', handler);
  // sau đó remove el nhưng quên removeEventListener → el vẫn reachable qua handler → leak
  document.body.removeChild(el);
  // Fix: window.removeEventListener('resize', handler);
}

// 3.3 Leak 2: quên cleanup interval/observer
import { useEffect } from 'react';
function LeakyComponent() {
  useEffect(() => {
    const id = setInterval(() => console.log('tick'), 1000);
    const onScroll = () => console.log(window.scrollY);
    window.addEventListener('scroll', onScroll);
    const obs = new MutationObserver(() => {});
    obs.observe(document.body, { childList: true });
    // Fix: cleanup
    return () => {
      clearInterval(id);
      window.removeEventListener('scroll', onScroll);
      obs.disconnect();
    };
  }, []);
}

// 3.4 Leak 3: global cache không giới hạn
const cache = new Map(); // strong — leak nếu key không bao giờ xóa
function getData(key) {
  if (cache.has(key)) return cache.get(key);
  const data = { big: new Array(1e6).fill(0) };
  cache.set(key, data);
  return data;
}
// Fix: WeakMap nếu key là object, hoặc LRU với size limit
const weakCache = new WeakMap();
let objKey = { id: 1 };
weakCache.set(objKey, { big: new Array(1e6).fill(0) });
objKey = null; // GC được, entry WeakMap tự mất

// LRU đơn giản
class LRU {
  constructor(limit = 100) { this.limit = limit; this.map = new Map(); }
  get(k) { const v = this.map.get(k); if (v) { this.map.delete(k); this.map.set(k, v); } return v; }
  set(k, v) { if (this.map.has(k)) this.map.delete(k); else if (this.map.size >= this.limit) this.map.delete(this.map.keys().next().value); this.map.set(k, v); }
}

// 3.5 Leak 4: Detached DOM
let detached;
function createDetached() {
  const el = document.createElement('div');
  el.innerHTML = 'x'.repeat(1e6);
  document.body.appendChild(el);
  detached = el; // giữ reference global
  document.body.removeChild(el); // remove khỏi DOM nhưng detached vẫn giữ cả subtree
  // Fix: detached = null; hoặc không giữ
}

// 3.6 WeakRef và FinalizationRegistry
let target = { data: 'big' };
const ref = new WeakRef(target);
console.log(ref.deref()); // { data: 'big' }
target = null;
// sau GC, ref.deref() có thể trả undefined

const registry = new FinalizationRegistry((heldValue) => {
  console.log('GC collected', heldValue);
});
let obj = { id: 1 };
registry.register(obj, 'obj id 1');
obj = null; // khi GC thu hồi obj, callback chạy với 'obj id 1' (không đảm bảo thời gian)

// 3.7 Đo heap trong code
if (performance.memory) {
  console.log(performance.memory.usedJSHeapSize, performance.memory.jsHeapSizeLimit);
}
// Node: process.memoryUsage()
// console.log(process.memoryUsage().heapUsed);
```

```js
// 3.8 Tránh leak với AbortController
function useFetch(url) {
  useEffect(() => {
    const controller = new AbortController();
    fetch(url, { signal: controller.signal }).then(r => r.json()).then(console.log);
    return () => controller.abort(); // cancel fetch, cho GC thu hồi
  }, [url]);
}
```

## 4. So sánh / Phân loại

| GC | Cơ chế | Xử lý circular | Chi phí | Trạng thái |
|----|--------|----------------|---------|------------|
| Reference Counting | Đếm reference, thu hồi khi count=0 | Không — leak | Thấp, nhưng leak | Obsolete (dùng trong Python CPython, không phải JS) |
| Mark-and-Sweep | Mark từ roots, sweep không mark | Có — không reachable thì sweep | Cao hơn, pause | Chuẩn JS (V8, JSC, SpiderMonkey) |
| Generational (V8) | Young (Scavenge) + Old (Mark-Compact) | Có | Tối ưu — Young GC nhanh | Thực tế V8 |

| Nguồn leak | Giữ gì | Fix | Đo |
|------------|--------|-----|----|
| Closure giữ DOM/large | LexicalEnvironment → DOM | Nullify, không closure over large, WeakRef | Heap snapshot → Detached |
| Quên cleanup (interval, listener, observer) | Timer/observer queue → callback → closure | `clearInterval`, `removeEventListener`, `disconnect`, `abort` trong cleanup | Allocation timeline, check `getEventListeners` |
| Global / Map cache vô hạn | Global → Map → data | `WeakMap`, LRU, TTL, `size` limit | Heap snapshot → Map entries |
| Detached DOM | Biến JS → DOM subtree | `ref = null`, không giữ global | Memory → Detached elements |

| WeakMap vs Map vs WeakRef | Giữ key | Ngăn GC? | Enumerable | `size` | Dùng khi |
|---------------------------|---------|----------|------------|--------|----------|
| `Map` | Strong | Có | Có | Có | Cache cần iterate, key có thể primitive |
| `WeakMap` | Weak (object key) | Không | Không | Không | Gắn metadata cho object, không ngăn GC |
| `WeakSet` | Weak | Không | Không | Không | Track visited, không ngăn GC |
| `WeakRef` | Weak | Không | — | — | Giữ weak ref tới object lớn, check `deref()` |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng `WeakMap` khi cần iterate/size**: WeakMap không enumerable, không `size`, không `clear`. Nếu cần liệt kê keys, phải dùng `Map` + manual evict.
- **Không dùng `WeakRef`/`FinalizationRegistry` cho logic chính**: GC thời gian không đảm bảo, callback có thể không bao giờ chạy (nếu không GC), hoặc chạy trễ. Chỉ dùng cho cleanup ngoài JS (WASM, file handle), không dùng để quản lý state.
- **Không giữ reference tới DOM sau khi remove**: nếu cần cache DOM, dùng `WeakMap` key là parent, không giữ detached node.
- **Không tạo closure giữ large object không cần**: nếu handler chỉ cần `id`, đừng `() => console.log(bigObject)`, chỉ `() => console.log(id)`.
- **Khi nào dùng `WeakMap`**: cache metadata cho object, `memoize` với object key, store private data cho instance.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Interval/listener không cleanup khi component unmount**
  - Triệu chứng: heap tăng sau mỗi mount/unmount, `setInterval` vẫn chạy sau unmount.
  - Fix: `useEffect return () => { clearInterval(id); removeEventListener }`, `AbortController`.
  - Đo: Chrome DevTools → Memory → Heap snapshot (take trước/sau, compare), Allocation timeline → xem `setInterval`/`EventListener` còn sống.

- **Lỗi 2: Detached DOM**
  - Triệu chứng: Elements panel không thấy nhưng Memory → Detached elements có.
  - Fix: không giữ biến global tới DOM, `ref.current = null` khi unmount.
  - Đo: Memory → Heap snapshot → filter `Detached`, `queryObjects(HTMLElement)`.

- **Lỗi 3: Map cache vô hạn**
  - Triệu chứng: heap tăng tuyến tính theo số request.
  - Fix: LRU, `maxSize`, `WeakMap` nếu key là object.
  - Đo: `performance.measureUserAgentSpecificMemory` (cross-origin isolated), `process.memoryUsage()`.

- **Lỗi 4: Closure trong debounce giữ component**
  - Triệu chứng: component unmount nhưng debounce callback vẫn giữ `this`/`props` cũ.
  - Fix: `debounced.cancel()` trong cleanup, hoặc `useDebounce` hook có cleanup.
  - Đo: heap snapshot → xem closure scope.

- **Lỗi 5: `FinalizationRegistry` không chạy**
  - Triệu chứng: trông đợi callback để release resource nhưng không chạy.
  - Fix: không rely vào nó cho correctness; dùng explicit `dispose()`/ `using` (Explicit Resource Management).
  - Đo: `gc()` trong Node với `--expose-gc`.

- **Công cụ**:
  - Chrome DevTools → Memory → Heap snapshot, Allocation instrumentation, Detached elements
  - `performance.memory` (Chromium), `process.memoryUsage()` (Node)
  - `getEventListeners(element)` trong Console
  - Lighthouse → Performance → Memory

## 7. Câu hỏi tự kiểm tra

1. Mark-and-Sweep khác Reference Counting thế nào, và vì sao circular reference không leak với Mark-and-Sweep?
2. Liệt kê 4 nguồn leak kinh điển trong SPA và cách fix từng cái.
3. `WeakMap` khác `Map` thế nào về GC, và khi nào dùng `WeakRef`/`FinalizationRegistry`?

<details>
<summary>Đáp án 30s</summary>

1. Reference Counting đếm số reference, thu hồi khi count=0, nhưng circular `a↔b` dù không reachable vẫn count>0 → leak. Mark-and-Sweep đi từ roots (global, stack), mark mọi object còn reachable qua chain; không mark thì sweep, nên circular không reachable vẫn bị sweep, không leak.
2. (1) Closure giữ DOM/large → không closure over large, nullify sau remove, WeakRef. (2) Quên cleanup interval/listener/observer/WS → cleanup trong `useEffect return`, `clearInterval`, `removeEventListener`, `disconnect`, `abort`. (3) Global/Map cache vô hạn → WeakMap/LRU/TTL, limit size. (4) Detached DOM → không giữ biến tới subtree đã remove, `ref=null`.
3. `WeakMap` key là object, giữ weak reference không ngăn GC, không enumerable, không `size`/`clear`; `Map` giữ strong, ngăn GC, có size và iterable. `WeakRef` giữ weak ref tới 1 object, `deref()` check còn sống không; `FinalizationRegistry` callback khi object bị GC, nhưng thời gian không đảm bảo, chỉ dùng cho cleanup ngoài JS, không cho logic chính.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 14. Spec: [MDN Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_management), [V8 GC](https://v8.dev/blog/trash-talk), [WeakRef](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef).*
