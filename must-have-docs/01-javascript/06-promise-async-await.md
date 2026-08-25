# Promise và async/await — Trạng thái, chaining, combinators và desugar

> Tags: #promise #async-await #microtask #error-handling | Nguồn: `docs/01-javascript-typescript.md` câu 6 + 18 | Mức: P0

## 1. Định nghĩa chính xác

**Promise** là object đại diện cho giá trị của một thao tác bất đồng bộ trong tương lai, có 3 trạng thái: `pending` → `fulfilled` (với value) hoặc `rejected` (với reason), chỉ chuyển trạng thái **một lần** (settled). `.then` luôn trả về Promise mới, cho phép chaining.

**`async/await`** là syntactic sugar cho `Promise` + `Generator`: `async function` luôn trả Promise; `await x` tương đương `Promise.resolve(x)` rồi suspend hàm, phần còn lại được enqueue như microtask khi Promise settle, và `try/catch` quanh `await` bắt được reject.

## 2. Cơ chế hoạt động

### 2.1 Promise states và chaining

- `new Promise((resolve, reject) => {})` — executor chạy ngay (sync). Gọi `resolve(v)` → `fulfilled`, `reject(e)` → `rejected`.
- `.then(onFulfilled, onRejected)` trả Promise mới `p2`. Nếu callback trả giá trị `v`, `p2` fulfill với `v`; nếu trả Promise `p`, `p2` adopt state của `p`; nếu ném lỗi, `p2` reject.
- `.catch(onRejected)` là `.then(null, onRejected)`. `.finally(cb)` không nhận value, pass-through.

### 2.2 `await` desugar

```js
async function foo() {
  const a = await p1;
  const b = await p2;
  return a + b;
}
// tương đương
function foo() {
  return Promise.resolve(p1).then(a =>
    Promise.resolve(p2).then(b => a + b)
  );
}
```

`await` luôn `Promise.resolve(x)` (nếu `x` không phải Promise thì wrap fulfilled ngay), rồi `yield` ra microtask. Phần sau `await` là `.then` callback.

### 2.3 Combinators

- `Promise.all(iterable)` — song song, **fail-fast**: fulfill khi tất cả fulfill (trả array theo thứ tự input), reject ngay khi 1 reject.
- `Promise.allSettled(iterable)` — đợi hết, trả `[{status:'fulfilled', value}, {status:'rejected', reason}]`, không fail-fast.
- `Promise.race(iterable)` — settle theo Promise đầu tiên settle (fulfill hay reject).
- `Promise.any(iterable)` — fulfill theo Promise đầu tiên fulfill, reject chỉ khi tất cả reject (`AggregateError`).
- `Promise.withResolvers()` (ES2024) — trả `{promise, resolve, reject}` để resolve từ ngoài.

### 2.4 Error propagation

Lỗi ném trong `.then` hoặc `await` sẽ reject Promise chain. Nếu không có `.catch`/`try/catch`, phát `unhandledrejection`. `async` executor trong `new Promise(async () => {})` là anti-pattern: lỗi trong async executor không được `reject` tự động.

## 3. Ví dụ tối thiểu

```js
// 3.1 Tạo và chaining — luôn return trong then
fetch('/api/user/1')
  .then(r => {
    if (!r.ok) throw new Error(`HTTP ${r.status}`);
    return r.json(); // return Promise → chain adopt
  })
  .then(user => fetch(`/api/posts?user=${user.id}`))
  .then(r => r.json())
  .then(posts => console.log(posts))
  .catch(e => console.error('chain failed', e))
  .finally(() => console.log('done'));

// 3.2 async/await với try/catch/finally
async function getUser(id, { signal } = {}) {
  try {
    const r = await fetch(`/api/user/${id}`, { signal });
    if (!r.ok) throw new Error(r.statusText);
    const user = await r.json();
    const posts = await fetch(`/api/posts?user=${user.id}`, { signal }).then(x => x.json());
    return { user, posts };
  } catch (e) {
    if (e.name === 'AbortError') console.log('cancelled');
    else console.error(e);
    throw e; // re-throw để caller quyết định
  } finally {
    // cleanup loading
  }
}

// 3.3 Song song vs tuần tự
async function parallel() {
  const [a, b] = await Promise.all([fetch('/api/a'), fetch('/api/b')]); // song song
  return [a, b];
}
async function sequential() {
  const a = await fetch('/api/a'); // đợi a xong mới fetch b — chậm hơn
  const b = await fetch('/api/b');
  return [a, b];
}
async function mapParallel(ids) {
  // đúng: map + all
  const users = await Promise.all(ids.map(id => fetch(`/api/user/${id}`).then(r => r.json())));
  return users;
  // sai: for...of await tuần tự nếu không cần thiết
}

// 3.4 allSettled vs all — resilient
const results = await Promise.allSettled([fetch('/api/a'), fetch('/api/b')]);
const ok = results.filter(r => r.status === 'fulfilled').map(r => r.value);
console.log(ok);

// 3.5 race — timeout pattern
function timeout(ms, msg = 'timeout') {
  return new Promise((_, reject) => setTimeout(() => reject(new Error(msg)), ms));
}
try {
  const data = await Promise.race([fetch('/api/data'), timeout(3000)]);
  console.log(data);
} catch (e) { console.log(e.message); }

// 3.6 any — lấy cái đầu tiên thành công
try {
  const firstOk = await Promise.any([fetch('/mirror1'), fetch('/mirror2'), fetch('/mirror3')]);
  console.log(firstOk);
} catch (e) {
  console.log(e instanceof AggregateError); // true nếu tất cả fail
  console.log(e.errors);
}

// 3.7 withResolvers (ES2024)
const { promise, resolve, reject } = Promise.withResolvers();
setTimeout(() => resolve('done'), 100);
console.log(await promise); // 'done'

// 3.8 Sai: async executor
// new Promise(async (resolve) => { await fetch(...); resolve(); }) // anti-pattern — lỗi trong await không reject Promise ngoài
// Đúng:
new Promise((resolve, reject) => {
  (async () => {
    try { const r = await fetch('/api'); resolve(r); } catch(e){ reject(e); }
  })();
});
// hoặc đơn giản không bọc Promise nếu đã có async

// 3.9 await trong forEach không hoạt động như mong đợi
// [1,2,3].forEach(async (id) => { await fetch(id); }); // không đợi — forEach không await
// Đúng:
for (const id of [1,2,3]) { await fetch(`/api/${id}`); } // tuần tự
// hoặc
await Promise.all([1,2,3].map(id => fetch(`/api/${id}`))); // song song
```

```js
// 3.10 Cancel với AbortController (Promise không cancel được trực tiếp)
const controller = new AbortController();
setTimeout(() => controller.abort(), 2000);
try {
  await fetch('/api/long', { signal: controller.signal });
} catch (e) {
  if (e.name === 'AbortError') console.log('aborted');
}
```

## 4. So sánh / Phân loại

| Combinator | Khi nào settle | Kết quả | Fail behavior |
|------------|---------------|---------|---------------|
| `Promise.all` | Tất cả fulfill | `Array<value>` theo thứ tự input | Reject ngay khi 1 reject (fail-fast) |
| `Promise.allSettled` | Tất cả settle | `Array<{status, value\|reason}>` | Không fail — luôn fulfill |
| `Promise.race` | 1 settle đầu tiên | value/reason của promise đầu | Theo promise đầu (có thể reject) |
| `Promise.any` | 1 fulfill đầu tiên | value đầu tiên fulfill | Chỉ reject khi tất cả reject (`AggregateError`) |
| `Promise.withResolvers` | Do bạn gọi `resolve/reject` | `{promise, resolve, reject}` | — |

| `then` chaining | `async/await` |
|-----------------|---------------|
| Explicit `.then/.catch`, callback style | Trông như sync, `try/catch` quen thuộc |
| Khó đọc khi lồng, nhưng linh hoạt cho conditional chain | Dễ đọc, nhưng dễ `await` tuần tự không cần thiết |
| Không suspend function, chain là Promise mới | Suspend async function, phần sau là microtask |

| `await` trong | Hành vi | Nên dùng khi |
|---------------|---------|--------------|
| `for...of` | Tuần tự, mỗi lần đợi 1 | Cần tuần tự (rate limit, dependency) |
| `Promise.all(ids.map(...))` | Song song | Không phụ thuộc nhau, cần tốc độ |
| `Array.forEach(async...)` | Không đợi — fire-and-forget | Không bao giờ (bug) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không `await` trong loop nếu không cần tuần tự**: `for (const id of ids) await fetch(id)` chậm tuyến tính; nếu independent, dùng `Promise.all`.
- **Không dùng `Promise.all` khi cần resilient**: 1 lỗi làm mất hết kết quả. Dùng `allSettled` hoặc `any` tùy UX.
- **Không bọc `new Promise(async () => {})`**: async executor lỗi không reject outer Promise. Chỉ dùng `new Promise` khi wrap callback API; nếu đã có Promise, dùng `async/await` trực tiếp.
- **Không quên `return` trong `.then`**: `then(r => { r.json() })` thiếu `return` → chain nhận `undefined`.
- **Promise không cancel được**: đừng cố `promise.cancel()`. Dùng `AbortController` cho `fetch`, hoặc token pattern cho logic khác.
- **Không `await` quá sớm nếu có thể song song**: `const a = await fetchA(); const b = await fetchB();` → tuần tự; `const [a,b] = await Promise.all([fetchA(), fetchB()])` → song song.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Quên `catch` → `unhandledrejection`**
  - Triệu chứng: lỗi silent, hoặc crash Node.
  - Fix: luôn `.catch` cuối chain hoặc `try/catch` quanh `await`; thêm global `window.addEventListener('unhandledrejection', e => { e.preventDefault(); log(e.reason) })`.
  - Đo: DevTools Console, Node `--unhandled-rejections=strict`, Sentry.

- **Lỗi 2: `await` trong `forEach`/`map` không đợi**
  - Triệu chứng: function return trước khi async xong.
  - Fix: `for...of` + `await`, hoặc `await Promise.all(arr.map(async...))`.
  - Đo: test với `await` + assert.

- **Lỗi 3: `Promise.all` fail-fast mất data**
  - Triệu chứng: 1 request fail làm mất hết.
  - Fix: `allSettled` + filter, hoặc `Promise.all` với `.catch` per-item.
  - Đo: mock 1 fail trong test.

- **Lỗi 4: `fetch` không timeout**
  - Triệu chứng: treo vô hạn.
  - Fix: `Promise.race([fetch(...), timeout(ms)])` + `AbortController`.
  - Đo: network throttling trong DevTools.

- **Lỗi 5: `try/catch` không bắt được `setTimeout` throw**
  - Triệu chứng: `try { setTimeout(()=>{throw e},0) } catch{}` không bắt.
  - Fix: không throw trong macrotask; dùng `Promise` hoặc `unhandledrejection`/`window.onerror`.
  - Đo: DevTools → Sources → Pause on exceptions.

- **Công cụ**: DevTools → Sources → Async stack traces, Promise tab, `chrome://inspect`, `node --trace-warnings`.

## 7. Câu hỏi tự kiểm tra

1. `Promise.all` vs `allSettled` vs `race` vs `any` khác nhau thế nào về fail-fast và kết quả trả về?
2. `await Promise.resolve()` dù đã resolve sẵn vẫn async — vì sao, và `try/catch` quanh `await` bắt được reject thế nào?
3. Vì sao `new Promise(async (resolve, reject) => { await foo(); resolve(); })` là anti-pattern?

<details>
<summary>Đáp án 30s</summary>

1. `all`: fail-fast, reject ngay khi 1 reject, fulfill với array values theo thứ tự input. `allSettled`: đợi hết, luôn fulfill với array `{status, value|reason}`. `race`: settle theo promise đầu tiên (có thể reject). `any`: fulfill theo promise đầu tiên fulfill, chỉ reject khi tất cả reject với `AggregateError`.
2. `await x` → `Promise.resolve(x).then(rest)`, `then` luôn enqueue microtask nên phần sau `await` chạy ở microtask checkpoint, không sync. `await` ném (reject) thì Promise của async function reject, `try/catch` trong async function được desugar thành `.catch` nên bắt được.
3. Vì executor của `new Promise` chạy sync, nhưng `async` executor trả Promise riêng; lỗi ném trong `await` sẽ reject Promise của async executor, không phải outer Promise → outer Promise không bao giờ reject (pending mãi). Fix: không dùng async executor, hoặc `try/catch` và `reject(e)` thủ công.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 6, Câu 18. Spec: [ECMA-262 §27 Promise](https://tc39.es/ecma262/#sec-promise-objects), [MDN Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise).*
