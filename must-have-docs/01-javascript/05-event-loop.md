# Event Loop — Call Stack, Task/Microtask/rAF và thứ tự thực thi

> Tags: #event-loop #microtask #macrotask #call-stack | Nguồn: `docs/01-javascript-typescript.md` câu 5 | Mức: P0

## 1. Định nghĩa chính xác

**Event Loop** là cơ chế điều phối concurrency trong JavaScript runtime (single-thread) bằng cách liên tục kiểm tra **Call Stack** rỗng thì lấy task từ các queue để thực thi. **Call Stack** là LIFO chứa Execution Context đang chạy. **Web APIs** (DOM, `fetch`, `setTimeout`) xử lý async ngoài JS thread, khi xong đẩy callback vào **Task Queue** (macrotask) hoặc **Microtask Queue**. Sau mỗi task, loop thực hiện **microtask checkpoint**: drain toàn bộ microtask trước khi render và trước task tiếp theo.

Thứ tự một vòng loop (Browser): `Task (1) → drain Microtasks (hết) → requestAnimationFrame → Render → Task tiếp theo`.

## 2. Cơ chế hoạt động

### 2.1 Các hàng đợi

- **Call Stack**: sync code chạy ngay. `function foo(){ bar() }` push `foo` rồi `bar`.
- **Task Queue (Macrotask)**: `setTimeout`, `setInterval`, I/O, `MessageChannel`, `setImmediate` (Node), UI events. Mỗi loop lấy **1** macrotask.
- **Microtask Queue**: `Promise.then/catch/finally`, `queueMicrotask`, `MutationObserver`, `process.nextTick` (Node — ưu tiên hơn cả Promise microtask). Sau mỗi task và sau mỗi microtask mới sinh ra, loop drain hết microtask.
- **rAF Queue**: `requestAnimationFrame` — chạy ở **rendering phase** trước khi paint, sau microtask checkpoint, trước task tiếp theo. Không phải task hay microtask.
- **Render**: style calc, layout, paint — chỉ chạy nếu có frame cần vẽ và microtask đã rỗng. Nếu microtask sinh liên tục (starvation), render bị đói.

```
Call Stack (LIFO)
   │
   ▼
Task Queue (1 task/loop) → Event Loop → Microtask Queue (drain all) → rAF → Render → next Task
   ↑                           │
Web APIs (timer, fetch, DOM) ──┘
```

### 2.2 `await` desugar

`async function foo(){ await p; console.log('B') }` tương đương `Promise.resolve(p).then(() => console.log('B'))` — phần sau `await` luôn là microtask, dù `p` đã resolve sẵn.

### 2.3 Starvation

Nếu microtask tạo thêm microtask vô hạn (`Promise.then(() => queueMicrotask(...))`), Task Queue và Render không bao giờ được chạy — UI freeze.

### 2.4 Node vs Browser

| Đặc điểm | Browser | Node.js |
|----------|---------|---------|
| `process.nextTick` | Không có | Có — microtask ưu tiên cao nhất, chạy trước Promise |
| `setImmediate` | Không (chỉ IE) | Có — check phase, chạy sau I/O, trước `setTimeout(0)` trong một số trường hợp |
| `queueMicrotask` | Có | Có |
| `requestAnimationFrame` | Có | Không |
| Thứ tự microtask | Promise → queueMicrotask → MutationObserver (cùng queue) | nextTick → Promise → queueMicrotask |

Node loop có thêm phases: `timers` → `pending callbacks` → `idle` → `poll` → `check` (`setImmediate`) → `close callbacks`.

## 3. Ví dụ tối thiểu

```js
// 3.1 Thứ tự cơ bản: sync → microtask → macrotask
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
queueMicrotask(() => console.log('4'));
console.log('5');
// 1, 5, 3, 4, 2

// 3.2 await luôn async (microtask)
async function foo() {
  console.log('A');
  await Promise.resolve(); // yield → microtask
  console.log('B');
}
console.log('C');
foo();
console.log('D');
// C, A, D, B — giải thích: foo chạy tới await, phần sau enqueue microtask, tiếp tục sync D, rồi B

// 3.3 Microtask sinh thêm microtask — drain hết trước task
Promise.resolve()
  .then(() => {
    console.log('m1');
    queueMicrotask(() => console.log('m2'));
  })
  .then(() => console.log('m3'));
setTimeout(() => console.log('task'), 0);
// m1, m2, m3, task — microtask queue drain toàn bộ trước task

// 3.4 Starvation — đừng làm trong production
let count = 0;
function starve() {
  if (count++ < 1000) {
    queueMicrotask(starve); // tạo microtask vô hạn → task 'never' không chạy, render đói
    // console.log('micro', count);
  }
}
// Promise.resolve().then(starve);
// setTimeout(() => console.log('never if starved'), 0);

// 3.5 rAF — chạy trước paint, sau microtask
console.log('sync');
requestAnimationFrame(() => console.log('rAF'));
Promise.resolve().then(() => console.log('micro'));
setTimeout(() => console.log('task'), 0);
// sync, micro, rAF, task  (rAF trước paint, sau micro)

// 3.6 Node nextTick ưu tiên hơn Promise (chỉ Node)
// process.nextTick(() => console.log('nextTick'));
// Promise.resolve().then(() => console.log('promise'));
// nextTick trước promise
```

```js
// 3.7 Batch update nhờ microtask (Vue/React pattern)
let pending = false;
const queue = new Set();
function queueUpdate(fn) {
  queue.add(fn);
  if (!pending) {
    pending = true;
    queueMicrotask(() => {
      queue.forEach(f => f());
      queue.clear();
      pending = false;
    });
  }
}
queueUpdate(() => console.log('update1'));
queueUpdate(() => console.log('update2'));
// cả hai flush cùng microtask — một lần

// 3.8 Defer xuống macrotask để nhường paint
function deferToMacrotask(fn) {
  setTimeout(fn, 0);
}
// Dùng khi cần để browser paint trước khi làm việc nặng
```

## 4. So sánh / Phân loại

| Tiêu chí | Microtask | Macrotask (Task) | rAF |
|----------|-----------|-----------------|-----|
| Ví dụ | `Promise.then`, `queueMicrotask`, `MutationObserver`, `process.nextTick` (Node) | `setTimeout`, `setInterval`, I/O, `MessageChannel`, `setImmediate` (Node) | `requestAnimationFrame` |
| Chạy khi nào | Ngay sau task hiện tại, trước render; drain hết trước khi lấy task mới | 1 task mỗi loop, sau microtask checkpoint | Trước paint, sau microtask, trước task tiếp |
| Số lượng mỗi loop | Tất cả (drain) | 1 | Tối đa 1 lần trước mỗi frame (60fps) |
| Ưu tiên | Cao nhất | Thấp | Giữa — trước render |
| Block render? | Có nếu sinh liên tục (starvation) | Không — nhường microtask trước | Không — chính là render phase |

| `setTimeout(0)` vs `queueMicrotask` vs `Promise.resolve().then` | Độ trễ | Dùng khi |
|---------------------------------------------------------------|--------|----------|
| `queueMicrotask(fn)` | Ngay sau sync, trước paint | Batch, defer nhẹ, cần chạy trước render |
| `Promise.resolve().then(fn)` | Tương tự queueMicrotask (microtask) | Tương tự, nhưng tạo Promise object |
| `setTimeout(fn, 0)` | Sau microtask + rAF + render, tối thiểu ~4ms (throttling) | Defer nặng, nhường cho paint, tránh starvation |
| `requestAnimationFrame(fn)` | Trước paint kế tiếp | Đo DOM, animation, đọc layout |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng microtask cho việc nặng**: microtask chạy trước render, nếu xử lý nặng trong `Promise.then` sẽ block paint → INP xấu. Đẩy việc nặng xuống `setTimeout` hoặc `requestIdleCallback`/`scheduler`.
- **Không lạm dụng `setTimeout(0)` để batch**: `setTimeout` có delay tối thiểu và throttling (4ms sau 5 lần lồng nhau), không chính xác cho batching. Dùng `queueMicrotask` cho batch update (như Vue `nextTick`).
- **Không tạo microtask đệ quy vô hạn**: gây starvation, UI freeze, Task/Render không chạy.
- **Không dùng `process.nextTick` trong browser** (không tồn tại) và không lạm dụng trong Node (starve I/O). `nextTick` ưu tiên hơn Promise nên dễ đói I/O.
- **Khi nào dùng `rAF`**: animation, đo layout (`getBoundingClientRect`) trước paint để tránh layout thrashing.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Nghĩ `setTimeout(0)` chạy ngay sau sync**
  - Triệu chứng: code phụ thuộc thứ tự sai, `2` không xuất hiện ngay sau `1`.
  - Fix: hiểu microtask trước macrotask; dùng `queueMicrotask` nếu cần ngay sau sync nhưng trước task.
  - Đo: `console.log` thứ tự, hoặc DevTools Performance → Event Log.

- **Lỗi 2: `await` không `await` — quên `await` hoặc `return` Promise**
  - Triệu chứng: `try/catch` không bắt được reject, lỗi `unhandledrejection`.
  - Fix: luôn `await` hoặc `return` Promise trong `async` function, hoặc `.catch`.
  - Đo: DevTools → Console → `unhandledrejection` event, Node `--unhandled-rejections=strict`.

- **Lỗi 3: Microtask starvation**
  - Triệu chứng: UI freeze, input không phản hồi, `setTimeout` không chạy.
  - Fix: giới hạn microtask chain, chunk công việc bằng `setTimeout`/`scheduler.yield`.
  - Đo: Performance tab → Long tasks, INP, `isInputPending()`.

- **Lỗi 4: Node `nextTick` vs `Promise` nhầm thứ tự**
  - Triệu chứng: code chạy khác Browser và Node.
  - Fix: không dùng `nextTick` nếu cần isomorphic; dùng `queueMicrotask`/`Promise` cho cross-env.
  - Đo: test trên cả hai môi trường.

- **Lỗi 5: `requestAnimationFrame` không chạy khi tab hidden**
  - Triệu chứng: animation dừng khi tab background.
  - Fix: dùng `setTimeout`/`Worker` cho logic cần chạy background, `rAF` chỉ cho visual.
  - Đo: Page Visibility API.

- **Công cụ**:
  - Chrome DevTools → Performance → Main thread → Task/Microtask/Animation Frame
  - `performance.now()`, `performance.mark/measure`
  - Node: `node --trace-event`, `clinic`

## 7. Câu hỏi tự kiểm tra

1. Thứ tự in ra của `console.log('1'); setTimeout(()=>console.log('2'),0); Promise.resolve().then(()=>console.log('3')); console.log('4')` là gì và vì sao?
2. `await Promise.resolve()` dù Promise đã resolve sẵn vẫn async — giải thích cơ chế microtask.
3. Node `process.nextTick` khác `Promise.then` thế nào, và starvation xảy ra khi nào?

<details>
<summary>Đáp án 30s</summary>

1. `1, 4, 3, 2`. `1` và `4` sync trên Call Stack trước. `Promise.then` enqueue microtask, `setTimeout` enqueue task. Sau khi stack rỗng, loop drain microtask (`3`) trước khi lấy task (`2`).
2. `await x` được desugar thành `Promise.resolve(x).then(restOfFunction)`. Phần sau `await` được bọc trong `.then` → enqueue microtask, nên luôn async, không chạy ngay dù `x` đã resolve. Spec `PerformPromiseThen` luôn queue microtask.
3. `process.nextTick` là microtask riêng của Node chạy **trước** Promise microtask, và được drain sau mỗi phase. Nếu `nextTick` tạo thêm `nextTick` liên tục, Poll phase và I/O không bao giờ được chạy (starvation). Promise cũng starve nếu chain vô hạn, nhưng `nextTick` ưu tiên cao hơn nên starve cả Promise.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 5. Spec: [HTML Living Standard — Event Loop](https://html.spec.whatwg.org/#event-loop), [ECMA-262 §8.4 Jobs and Queues](https://tc39.es/ecma262/#sec-jobs).*
