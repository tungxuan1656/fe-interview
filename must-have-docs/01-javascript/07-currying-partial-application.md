# Currying và Partial Application — Biến đổi arity bằng closure

> Tags: #currying #partial-application #closure #higher-order-function | Nguồn: `docs/01-javascript-typescript.md` câu 8 | Mức: P1

## 1. Định nghĩa chính xác

**Currying** là phép biến đổi hàm `f(a,b,c)` thành chuỗi hàm một ngôi `f(a)(b)(c)`, mỗi lần nhận 1 argument trả về hàm mới chờ argument tiếp theo, cho tới khi đủ arity thì thực thi. **Partial Application** là phép cố định một số argument trước (preset), trả về hàm nhận các argument còn lại, không yêu cầu chia nhỏ thành unary. Cả hai đều dựa trên **closure** để giữ các argument đã nhận.

Khác biệt cốt lõi: currying chuẩn hóa arity về 1, partial giữ arity tùy ý nhưng bind sẵn một phần.

## 2. Cơ chế hoạt động

### 2.1 Closure giữ args

Mỗi lần `curried(a)` tạo closure mới giữ `a` trong `[[Environment]]`. Lần gọi tiếp `(...next)` merge `args` cũ + mới và kiểm tra `args.length >= fn.length` (arity gốc). Nếu đủ thì `fn.apply(this, args)`, ngược lại trả closure tiếp.

`fn.length` là số parameter trước default/rest — nếu hàm có default param (`function f(a, b=1)`) thì `length` chỉ tính phần trước default.

### 2.2 `this` và currying

Curry phải forward `this` (`fn.apply(this, args)`) nếu hàm gốc dùng `this`. Arrow curry không có `this` riêng nên giữ lexical `this`.

### 2.3 HOC / Middleware

Currying/partial cho phép tạo HOC: `const withRetry = (fn, n) => (...args) => retry(fn, n, args)` hay `connect(mapState)(Component)` (React-Redux) — chính là currying để tách config và component.

## 3. Ví dụ tối thiểu

```js
// 3.1 Curry thủ công — hỗ trợ gọi gộp
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn.apply(this, args);
    return (...next) => curried.apply(this, [...args, ...next]);
  };
}
const add = (a, b, c) => a + b + c;
const curriedAdd = curry(add);
console.log(curriedAdd(1)(2)(3)); // 6
console.log(curriedAdd(1, 2)(3)); // 6 — linh hoạt
console.log(curriedAdd(1)(2, 3)); // 6

// Auto-currying với placeholder (lodash style)
function curryWithPlaceholder(fn) {
  const placeholder = curryWithPlaceholder._;
  return function curried(...args) {
    const filled = args.filter(a => a !== placeholder).length;
    if (filled >= fn.length && !args.includes(placeholder)) {
      return fn.apply(this, args);
    }
    return (...next) => {
      const merged = args.map(a => a === placeholder && next.length ? next.shift() : a).concat(next);
      return curried.apply(this, merged);
    };
  };
}
curryWithPlaceholder._ = Symbol('placeholder');

// 3.2 Partial application — thực tế hơn
function partial(fn, ...preset) {
  return (...rest) => fn(...preset, ...rest);
}
function fetchWith(url, opts) { return `fetch ${url} ${JSON.stringify(opts)}`; }
const fetchWithAuth = partial(fetchWith, 'https://api.example.com', { headers: { Authorization: 'Bearer xxx' } });
console.log(fetchWithAuth()); // fetch https://api.example.com {"headers":...}

// partial với placeholder
function partialWithPlaceholder(fn, ...preset) {
  return (...rest) => {
    const args = preset.map(p => p === partialWithPlaceholder._ ? rest.shift() : p).concat(rest);
    return fn(...args);
  };
}
partialWithPlaceholder._ = Symbol('_');

// 3.3 Use case: HOC / middleware
const withLogging = fn => (...args) => {
  console.log('call', fn.name, args);
  const res = fn(...args);
  console.log('result', res);
  return res;
};
const withRetry = (fn, n = 3) => async (...args) => {
  for (let i = 0; i < n; i++) {
    try { return await fn(...args); } catch (e) { if (i === n-1) throw e; }
  }
};
const withTimeout = (fn, ms) => (...args) =>
  Promise.race([fn(...args), new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), ms))]);

const fetchData = async (url) => { /* ... */ return url; };
const resilientFetch = withLogging(withRetry(withTimeout(fetchData, 3000), 2));
// resilientFetch('https://api.example.com') — composed

// 3.4 Currying trong Redux middleware signature
// const middleware = store => next => action => { ... }
// Gọi: middleware(store)(next)(action) — chính là currying để inject dần

// 3.5 Ramda-style — curry mặc định
// import { curry } from 'ramda';
// const curriedMap = curry((fn, arr) => arr.map(fn));
// const doubleAll = curriedMap(x => x*2);
// doubleAll([1,2,3]) // [2,4,6]
```

```js
// 3.6 Cẩn thận với `this`
const obj = {
  val: 10,
  add(a, b) { return this.val + a + b; }
};
const curriedAddThis = curry(obj.add);
// curriedAddThis(1)(2) — this mất nếu gọi trần
console.log(curriedAddThis.call(obj, 1)(2)); // cần bind this
// Fix: bind trước khi curry
const boundCurried = curry(obj.add.bind(obj));
console.log(boundCurried(1)(2)); // 13

// 3.7 Curry và fn.length với default param
function foo(a, b = 1, c) {} // foo.length === 1 (chỉ đếm tới trước default đầu tiên)
console.log(foo.length); // 1
// curry(foo) sẽ nghĩ chỉ cần 1 arg là đủ — sai nếu muốn đủ 3
// Fix: truyền arity thủ công
function curryN(fn, n = fn.length) {
  return function curried(...args) {
    if (args.length >= n) return fn.apply(this, args);
    return (...next) => curried.apply(this, [...args, ...next]);
  };
}
```

## 4. So sánh / Phân loại

| Tiêu chí | Currying | Partial Application |
|----------|----------|---------------------|
| Arity | Chia thành unary `f(a)(b)(c)` | Giữ nguyên, chỉ preset một số arg `f(a,b)(c,d)` |
| Số lần gọi | `n` lần cho hàm `n` args | 1 lần (trả hàm mới) |
| Closure | Mỗi bước tạo closure mới | Một closure giữ preset |
| Placeholder | Cần để skip arg giữa | Cần nếu muốn skip |
| Đọc hiểu | Khó hơn, call site dài | Dễ hơn, gần với gọi thường |
| TypeScript | Infer phức tạp (overload, tuple) | Đơn giản hơn |

| Thư viện | Currying | Partial | Ghi chú |
|----------|----------|---------|---------|
| Lodash | `_.curry(fn)` | `_.partial(fn, ...preset)` | Hỗ trợ placeholder `_` |
| Ramda | Curry mặc định mọi hàm | `R.partial` | Tự động curry, point-free |
| Thủ công | `curry(fn)` như trên | `partial(fn, ...preset)` | Cần xử lý `this`, `length` |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không curry hàm có arity động, rest param, hoặc nhiều optional/default**: `fn.length` không phản ánh đúng, curry sẽ gọi sớm hoặc không bao giờ đủ. Dùng partial hoặc không curry.
- **Không curry khi làm stack trace khó đọc**: mỗi bước curry thêm frame ẩn danh, debug khó. Với performance-critical, overhead tạo closure mỗi lần gọi đáng kể.
- **Không curry khi TypeScript infer quá phức tạp**: curry generic làm type `any` hoặc error khó hiểu. Dùng overload thủ công hoặc partial đơn giản hơn.
- **Khi nào nên dùng partial hơn curry**: khi chỉ cần preset config (auth header, base URL, `bind` `this`), partial trực quan hơn.
- **Khi nào curry hữu ích**: FP composition, `pipe`/`compose`, HOC, middleware, point-free style (`const getNames = map(prop('name'))`).

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Mất `this` khi curry method**
  - Triệu chứng: `this.val` là `undefined`.
  - Fix: `curry(fn.bind(obj))` hoặc curry giữ `this` bằng `apply(this, ...)`.
  - Đo: test `expect(curried.call(obj, ...)).toBe(...)`.

- **Lỗi 2: `fn.length` sai do default/rest param**
  - Triệu chứng: curry gọi sớm với thiếu arg.
  - Fix: truyền arity thủ công `curryN(fn, 3)`.
  - Đo: `console.log(fn.length)`.

- **Lỗi 3: Tạo quá nhiều closure trong loop**
  - Triệu chứng: memory tăng, GC pause.
  - Fix: curry một lần ngoài loop, tái sử dụng.
  - Đo: heap snapshot, performance.

- **Lỗi 4: TypeScript type mất**
  - Triệu chứng: curried function trả `any`.
  - Fix: overload `function curry<A,B,C,R>(fn:(a:A,b:B,c:C)=>R): (a:A)=>(b:B)=>(c:C)=>R` hoặc dùng `ts-toolbelt`.
  - Đo: `tsc --noEmit`.

- **Lỗi 5: Không xử lý placeholder**
  - Triệu chứng: không thể skip arg giữa (`curry(fn)(_, 2)(1,3)`).
  - Fix: implement placeholder như lodash `curry.placeholder = _`.

## 7. Câu hỏi tự kiểm tra

1. Currying và Partial Application khác nhau thế nào về arity và cách gọi?
2. Vì sao `curry` dựa trên `fn.length` có thể sai với hàm có default param, và cách fix?
3. Cho ví dụ HOC/middleware dùng currying để inject `store`/`next`/`action`, và giải thích lợi ích.

<details>
<summary>Đáp án 30s</summary>

1. Currying biến `f(a,b,c)` thành `f(a)(b)(c)` — mỗi lần 1 arg, tới đủ arity mới chạy. Partial cố định một số arg `f(a,b)` → trả `g(c)` nhận phần còn lại, không yêu cầu unary. Currying là trường hợp đặc biệt của partial với arity 1.
2. `fn.length` chỉ đếm param trước default/rest đầu tiên. `function f(a, b=1, c){}` có `length===1` nên curry nghĩ 1 arg là đủ, gọi sớm thiếu `c`. Fix: truyền arity thủ công `curryN(fn, 3)` hoặc design hàm không dùng default ở giữa.
3. `const logger = store => next => action => { console.log(action); return next(action); }`. Currying cho phép apply dần: `const enhanced = logger(store)(next)` tạo middleware đã bind `store`/`next`, chỉ còn chờ `action`. Lợi ích: composition `compose(m1,m2,m3)(dispatch)`, tách config khỏi execution, test dễ (inject mock store/next).

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 8. Sách: [Professor Frisby's Mostly Adequate Guide to FP](https://mostly-adequate.gitbook.io/mostly-adequate-guide/). Lib: [Ramda curry](https://ramdajs.com/docs/#curry), [Lodash curry](https://lodash.com/docs/#curry).*
