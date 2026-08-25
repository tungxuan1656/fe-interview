# this Binding — 4 quy tắc xác định giá trị this tại call-site

> Tags: #this #binding #call-apply-bind #arrow-function | Nguồn: `docs/01-javascript-typescript.md` câu 3 + 12 | Mức: P0

## 1. Định nghĩa chính xác

`this` là binding động được xác định tại **call-site** (nơi hàm được gọi), không phải nơi khai báo, trừ **arrow function** và **bound function** (`bind`). Giá trị `this` được truyền như một tham số ẩn khi thực thi [[Call]]/[[Construct]]. Trong strict mode, `this` không tự động boxing và default là `undefined`; ở sloppy mode, `null/undefined` được thay bằng Global Object và primitive được boxing.

Arrow function không có `[[ThisMode]]` riêng; nó **lexically capture** `this` từ LexicalEnvironment bao ngoài (outer `this`).

## 2. Cơ chế hoạt động

### 2.1 Thứ tự ưu tiên (4 reglas)

Spec và thực tế ưu tiên: **`new` > `bind` (hard binding) > `implicit` > `default`**. Nếu nhiều quy tắc cùng thỏa, quy tắc ưu tiên cao hơn thắng.

1. **New binding**: `new Foo(...args)` → tạo object mới, `[[Prototype]]` = `Foo.prototype`, `this` = object mới. `this` thắng mọi binding khác, kể cả `bind`.
   - Nếu constructor `return` object thì object đó được trả về thay vì `this`; nếu `return` primitive thì bỏ qua.
2. **Explicit binding**: `fn.call(thisArg, ...args)` / `fn.apply(thisArg, argsArray)` / `fn.bind(thisArg)` → `this` = `thisArg` (đã qua `ToObject` ở sloppy, giữ nguyên ở strict).
   - `bind` tạo **bound function exotic object** với `[[BoundThis]]` cố định, không thể bị override bởi `call/apply` sau đó.
3. **Implicit binding**: `obj.fn()` → `this` = `obj` (base object trước dấu chấm). Chỉ tính object ngay trước call, không phải chain dài.
   - `obj.a.b.fn()` → `this` = `b`, không phải `obj`.
4. **Default binding**: gọi trần `fn()` → strict: `this` = `undefined`; sloppy: `this` = Global Object (`window`/`globalThis`). Đây là fallback.

### 2.2 Arrow function

Arrow không có `this` riêng; `this` được resolve như biến lexical (tra cứu trong EnvironmentRecord). Không thể `call`/`bind` để đổi `this` của arrow.

### 2.3 `call` / `apply` / `bind` chi tiết

- `call(thisArg, arg1, arg2)` — gọi ngay, args rời.
- `apply(thisArg, [arg1, arg2])` — gọi ngay, args là array-like. Hữu ích khi spread chưa có hoặc cần `Math.max.apply(null, arr)`.
- `bind(thisArg, ...preset)` — không gọi, trả bound function với `[[BoundThis]]` + partial args. Bound function có `length` giảm theo số preset.

## 3. Ví dụ tối thiểu

```js
// 3.1 4 quy tắc theo thứ tự

// Default
function defaultFn() { return this; }
console.log(defaultFn() === undefined); // true ở strict mode (module luôn strict)
console.log((function(){ return this; })() === globalThis); // true ở sloppy (non-strict script)

// Implicit
const obj = {
  name: 'A',
  regular() { return this.name; },
  arrow: () => this?.name // lexical this — không phải obj
};
console.log(obj.regular()); // 'A' — implicit
const fn = obj.regular;
console.log(fn()); // undefined (strict default) — mất implicit

// Explicit
function greet(greeting, punct) { return `${greeting} ${this.name}${punct}`; }
const user = { name: 'An' };
console.log(greet.call(user, 'Hello', '!')); // Hello An!
console.log(greet.apply(user, ['Hi', '!!'])); // Hi An!!

// Bind cứng — không override được
const bound = greet.bind({ name: 'C' }, 'Hey');
console.log(bound('...')); // Hey C...
console.log(bound.call({ name: 'D' }, '...')); // vẫn Hey C... — bind thắng call

// New thắng bind
function Foo(name) { this.name = name; }
const BoundFoo = Foo.bind({ name: 'ignored' });
const inst = new BoundFoo('B');
console.log(inst.name); // 'B' — new thắng bind
console.log(inst instanceof Foo); // true

// 3.2 Implicit mất khi pass callback
const counter = {
  count: 0,
  inc() { this.count++; }
};
setTimeout(counter.inc, 0); // this mất → NaN hoặc lỗi
setTimeout(() => counter.inc(), 0); // giữ được — arrow hoặc wrapper
setTimeout(counter.inc.bind(counter), 0); // giữ được — bind

// 3.3 Arrow lexical this — hợp cho callback nhưng không làm method
const obj2 = {
  name: 'Obj2',
  regular() { return this.name; },
  arrow: () => { return this?.name; } // this từ outer scope (global/module), không phải obj2
};
console.log(obj2.regular()); // 'Obj2'
console.log(obj2.arrow()); // undefined — không có this riêng

class Timer {
  constructor() { this.sec = 0; }
  start() {
    // arrow giữ this của start()
    setInterval(() => { this.sec++; console.log(this.sec); }, 1000);
  }
  // Nếu dùng regular: setInterval(function(){ this.sec++ }) → this là window/undefined
}

// 3.4 Polyfill bind tối giản (không hỗ trợ new, spec đầy đủ cần xử lý construct)
Function.prototype.myBind = function(ctx, ...preset) {
  const fn = this;
  return function(...rest) {
    return fn.apply(ctx, [...preset, ...rest]);
  };
};
function add(a, b, c) { return a + b + c; }
const add5 = add.myBind(null, 5);
console.log(add5(3, 2)); // 10

// 3.5 Mượn method
const slice = Array.prototype.slice;
console.log(slice.call({ 0: 'a', 1: 'b', length: 2 })); // ['a','b']
console.log(Array.from({ 0: 'a', 1: 'b', length: 2 })); // cách hiện đại
```

```tsx
// 3.6 React class — vì sao phải bind
import React from 'react';
class Button extends React.Component {
  constructor(props) {
    super(props);
    this.handleClick = this.handleClick.bind(this); // explicit hard binding
  }
  handleClick() { console.log(this.props); } // nếu không bind, this mất khi pass làm prop
  render() { return <button onClick={this.handleClick}>click</button>; }
}
// Function component với arrow không cần bind — closure lexical
function ButtonFn(props) {
  const handleClick = () => console.log(props);
  return <button onClick={handleClick}>click</button>;
}
```

## 4. So sánh / Phân loại

| Quy tắc | Cú pháp | `this` là gì | Ưu tiên |
|---------|---------|--------------|---------|
| New | `new Foo()` | object mới tạo | 1 (cao nhất) |
| Explicit (bind) | `fn.bind(obj)` | `obj` đã bind (hard) | 2 |
| Explicit (call/apply) | `fn.call(obj)` | `obj` | 2 |
| Implicit | `obj.fn()` | `obj` (base trước `.`) | 3 |
| Default | `fn()` | `undefined` (strict) / `globalThis` (sloppy) | 4 |
| Lexical (arrow) | `() => {}` | `this` của outer scope | N/A — không có `this` riêng |

| Đặc điểm | Regular function | Arrow function | Bound function |
|----------|-----------------|----------------|----------------|
| Có `this` riêng | Có | Không — lexical | Có nhưng cố định `[[BoundThis]]` |
| Có `arguments` riêng | Có | Không — lexical | Có (nhưng thường dùng `...rest`) |
| Dùng làm constructor (`new`) | Được | Không — ném `TypeError` | Được, nhưng `this` mới thắng `[[BoundThis]]` |
| `call/apply/bind` đổi được `this` | Được | Không | Không (đã hard-bind) |
| `prototype` | Có | Không | Có nhưng trỏ tới target |

| `call` vs `apply` vs `bind` | Gọi ngay? | Args | Trả về | Dùng khi |
|------------------------------|-----------|------|--------|----------|
| `call` | Có | rời `a,b,c` | kết quả hàm | biết trước số args, cần `this` ngay |
| `apply` | Có | mảng `[a,b,c]` | kết quả hàm | args là array-like, `Math.max.apply` |
| `bind` | Không | preset `a,b` | bound function | tạo hàm mới giữ `this`/preset, làm callback |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng arrow làm method trong object/class khi cần `this` động**: `const obj = { x:1, getX: () => this.x }` → `this` không phải `obj`, luôn sai.
- **Không dùng arrow làm constructor**: `const Foo = () => {}; new Foo()` → `TypeError`.
- **Không `bind` trong render / JSX**: `render(){ return <Child onClick={this.handle.bind(this)} /> }` tạo function mới mỗi render → phá `memo`, tăng GC. Bind trong `constructor` hoặc dùng class field arrow `handle = () => {}` (nhưng arrow field cũng tạo mỗi instance, cân nhắc).
- **Không `call/apply` trong hot loop nếu có thể tránh**: có overhead nhỏ so với gọi trực tiếp; trong performance-critical code, dùng gọi trực tiếp hoặc lưu `this` vào biến.
- **Ưu tiên arrow cho callback giữ `this`**, ưu tiên `bind` khi cần hard-binding và partial application; ưu tiên `call/apply` cho mượn method một lần.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Mất `this` khi pass method làm callback**
  - Triệu chứng: `TypeError: Cannot read properties of undefined`, hoặc `this` là `window`.
  - Fix: `bind` trong constructor, arrow wrapper `() => obj.fn()`, hoặc class field `fn = () => {}`.
  - Đo: ESLint `no-invalid-this`, DevTools breakpoint → Scope → `this`.

- **Lỗi 2: Nhầm arrow có `this` riêng**
  - Triệu chứng: `obj = { fn: () => this }` luôn ra `window/undefined`.
  - Fix: đổi sang `fn() {}` hoặc `fn: function(){}`.
  - Đo: review code, unit test `expect(obj.fn()).toBe(obj)`.

- **Lỗi 3: `bind` arrow vô tác dụng**
  - Triệu chứng: `const arrow = () => this; arrow.bind(obj)()` vẫn là lexical `this`.
  - Fix: không bind arrow; dùng regular function nếu cần `this` động.
  - Đo: test `bind` không đổi kết quả.

- **Lỗi 4: `new` lên bound function vẫn giữ `instanceof`**
  - Triệu chứng: ngạc nhiên khi `new (Foo.bind(obj)) instanceof Foo === true`.
  - Fix: hiểu spec — `new` bỏ qua `[[BoundThis]]` nhưng giữ `[[TargetFunction]]` để check prototype.
  - Đo: kiểm tra `instanceof` trong test.

- **Lỗi 5: Callback `setTimeout`/`addEventListener` mất `this`**
  - Triệu chứng: `this` trong handler là `window` hoặc `element` (với DOM event, `this` là `currentTarget`).
  - Fix: `handler.bind(this)`, arrow, hoặc `addEventListener('click', this.handler.bind(this))` và lưu bound ref để `removeEventListener`.
  - Đo: DevTools → Event Listeners, console.log `this` trong handler.

## 7. Câu hỏi tự kiểm tra

1. Sắp xếp ưu tiên `new` > `bind` > `implicit` > `default` nghĩa là gì? Cho ví dụ `new` thắng `bind`.
2. Vì sao `obj.method` khi gán `const fn = obj.method; fn()` lại mất `this`, và 3 cách fix là gì?
3. Arrow function có `this` riêng không? `arrow.call(obj)` có đổi được `this` không?

<details>
<summary>Đáp án 30s</summary>

1. Khi nhiều quy tắc cùng thỏa, quy tắc ưu tiên cao hơn quyết định `this`. Ví dụ: `const Bound = Foo.bind({a:1}); new Bound()` → `this` là object mới tạo, không phải `{a:1}`; `Bound.call({b:2})` sau khi bind vẫn là `{a:1}` vì bind cứng.
2. Vì `this` được xác định tại call-site: `fn()` là default binding (strict → `undefined`, sloppy → `globalThis`), không còn base `obj`. Fix: (1) `fn.bind(obj)`, (2) wrapper `() => obj.method()`, (3) `obj.method.call(obj)` hoặc `Reflect.apply`. Trong React class, bind trong `constructor`.
3. Không — arrow không có `[[ThisMode]]`, `this` được lookup như biến lexical từ outer Environment. `arrow.call(obj)` không đổi được `this`; spec bỏ qua `thisArg` cho arrow. Arrow cũng không có `arguments` và không thể `new`.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 3, Câu 12. Spec: [ECMA-262 §10.2 Arrow Function](https://tc39.es/ecma262/#sec-arrow-function-definitions), [§7.3.14 Call](https://tc39.es/ecma262/#sec-call), [Function.prototype.bind](https://tc39.es/ecma262/#sec-function.prototype.bind).*
