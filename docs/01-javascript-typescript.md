# 01. JavaScript & TypeScript - 30 Câu Hỏi Senior

> Bộ 30 câu hỏi cốt lõi về JavaScript (Câu 1-18) và TypeScript (Câu 19-30) dành cho Senior Frontend. Mỗi câu được trả lời ở mức độ hiểu cơ chế - trade-off - ví dụ thực tế, giọng mentor đi phỏng vấn.

## Mục lục

- [Câu 1: var / let / const, Hoisting và Temporal Dead Zone (TDZ)](#câu-1-var--let--const-hoisting-và-temporal-dead-zone-tdz)
- [Câu 2: Closure là gì? Ứng dụng và memory leak](#câu-2-closure-là-gì-ứng-dụng-và-memory-leak)
- [Câu 3: `this` trong JavaScript - 4 quy tắc binding](#câu-3-this-trong-javascript---4-quy-tắc-binding)
- [Câu 4: Prototype, Prototype Chain và Kế thừa](#câu-4-prototype-prototype-chain-và-kế-thừa)
- [Câu 5: Event Loop, Microtask và Macrotask](#câu-5-event-loop-microtask-và-macrotask)
- [Câu 6: Promise, async/await và xử lý lỗi bất đồng bộ](#câu-6-promise-asyncawait-và-xử-lý-lỗi-bất-đồng-bộ)
- [Câu 7: Iterator và Generator](#câu-7-iterator-và-generator)
- [Câu 8: Currying và Partial Application](#câu-8-currying-và-partial-application)
- [Câu 9: Debounce vs Throttle](#câu-9-debounce-vs-throttle)
- [Câu 10: Shallow Copy vs Deep Copy](#câu-10-shallow-copy-vs-deep-copy)
- [Câu 11: == vs ===, Coercion và Object.is](#câu-11--vs--coercion-và-objectis)
- [Câu 12: call / apply / bind](#câu-12-call--apply--bind)
- [Câu 13: Event Bubbling, Capturing và Delegation](#câu-13-event-bubbling-capturing-và-delegation)
- [Câu 14: Memory Leak và Garbage Collection trong JS](#câu-14-memory-leak-và-garbage-collection-trong-js)
- [Câu 15: CommonJS vs ES Modules và Tree-shaking](#câu-15-commonjs-vs-es-modules-và-tree-shaking)
- [Câu 16: Proxy vs Object.defineProperty](#câu-16-proxy-vs-objectdefineproperty)
- [Câu 17: Symbol, BigInt, WeakMap / WeakSet](#câu-17-symbol-bigint-weakmap--weakset)
- [Câu 18: Xử lý lỗi đồng bộ và bất đồng bộ](#câu-18-xử-lý-lỗi-đồng-bộ-và-bất-đồng-bộ)
- [Câu 19: any vs unknown vs never vs void](#câu-19-any-vs-unknown-vs-never-vs-void)
- [Câu 20: type vs interface](#câu-20-type-vs-interface)
- [Câu 21: Generics và Constraints](#câu-21-generics-và-constraints)
- [Câu 22: Utility Types và tự implement](#câu-22-utility-types-và-tự-implement)
- [Câu 23: Conditional Types, infer và Distributive](#câu-23-conditional-types-infer-và-distributive)
- [Câu 24: Mapped Types và Template Literal Types](#câu-24-mapped-types-và-template-literal-types)
- [Câu 25: Strict Mode và strictNullChecks](#câu-25-strict-mode-và-strictnullchecks)
- [Câu 26: Enum vs Union Type vs const assertion](#câu-26-enum-vs-union-type-vs-const-assertion)
- [Câu 27: Decorator và Metadata](#câu-27-decorator-và-metadata)
- [Câu 28: Type Narrowing, Type Guard và Assertion Function](#câu-28-type-narrowing-type-guard-và-assertion-function)
- [Câu 29: Declaration Merging và Module Augmentation](#câu-29-declaration-merging-và-module-augmentation)
- [Câu 30: Tối ưu hiệu năng TypeScript trong Monorepo lớn](#câu-30-tối-ưu-hiệu-năng-typescript-trong-monorepo-lớn)

---

### Câu 1: var / let / const, Hoisting và Temporal Dead Zone (TDZ)

**Trả lời Senior:**
`var` là function-scoped, bị hoisting và khởi tạo với `undefined` ngay khi entering scope, nên truy cập trước khai báo không lỗi mà ra `undefined`. `let`/`const` là block-scoped, cũng được hoisting nhưng **không được khởi tạo** - nằm trong TDZ từ đầu block đến dòng khai báo, truy cập sẽ ném `ReferenceError`.

Cơ chế: JS Engine qua 2 pha - Creation phase tạo LexicalEnvironment, với `var` thì `CreateAndInitializeBinding`, với `let/const` thì `CreateMutableBinding` nhưng chưa initialize. `const` thêm ràng buộc không cho re-assign (nhưng object bên trong vẫn mutate được).

```javascript
console.log(a); // undefined
var a = 1;

console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 2;

function test() {
  console.log(x); // undefined do var hoisting trong function scope
  if (false) var x = 1;
}
```

Ngoài ra `var` cho phép re-declare, gắn lên `window` (global object), còn `let/const` thì không. Trong loop với closure, `var` gây bug kinh điển:

```javascript
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0); // 3 3 3
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i), 0); // 0 1 2
// vì let tạo binding mới mỗi iteration
```

**Trade-off / Khi nào không dùng:** Gần như luôn dùng `const` mặc định, `let` khi cần re-assign, tránh `var` hoàn toàn. Exception duy nhất là khi cần polyfill hoặc code chạy trong môi trường rất cũ.

**Lỗi thường gặp:** Dùng `var` trong if/for nghĩ là block-scope; nhầm `const` là immutable hoàn toàn; không hiểu TDZ nên để `typeof` trước khai báo cũng lỗi.

**Câu hỏi đào sâu:** `typeof` với biến chưa khai báo vs biến trong TDZ khác gì? Vì sao `let` trong for-loop lại tạo closure đúng?

---

### Câu 2: Closure là gì? Ứng dụng và memory leak

**Trả lời Senior:**
Closure là hiện tượng function "ghi nhớ" Lexical Environment nơi nó được **khai báo**, không phải nơi được gọi. Khi inner function được trả ra ngoài, nó vẫn giữ reference tới outer scope dù outer function đã execute xong. Đây là cơ chế cốt lõi của JS, không phải trick.

Cơ chế: Mỗi function khi tạo ra sẽ có `[[Environment]]` trỏ tới Lexical Environment hiện tại. Khi được gọi, nó tạo Environment mới có outer link tới `[[Environment]]`. Garbage Collector không thu hồi biến nếu còn reference từ closure.

```javascript
function createCounter() {
  let count = 0; // private state
  return {
    inc: () => ++count,
    dec: () => --count,
    get: () => count
  };
}
const c = createCounter();
c.inc(); // 1

// Ứng dụng: memoize, once, debounce, module pattern, currying
function memoize(fn) {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const res = fn(...args);
    cache.set(key, res);
    return res;
  };
}
```

**Trade-off / Khi nào không dùng:** Closure giữ memory, nếu cache không giới hạn hoặc giữ DOM node lớn sẽ leak. Trong React, closure stale là nguồn bug lớn (useEffect bắt closure cũ). Đừng lạm dụng closure để tạo private khi có thể dùng class `#privateField` hoặc WeakMap.

**Lỗi thường gặp:** Vòng lặp `var` + closure, stale closure trong `useEffect`/`setTimeout`, tạo closure trong loop render gây tạo function mới liên tục ảnh hưởng performance.

**Câu hỏi đào sâu:** Closure và prototype khác nhau thế nào về lưu state? Làm sao debug memory leak do closure bằng Chrome DevTools?

---

### Câu 3: `this` trong JavaScript - 4 quy tắc binding

**Trả lời Senior:**
`this` không phải lexical như arrow function, nó được xác định tại **call-site** (nơi gọi hàm), trừ arrow function và `bind`. Có 4 quy tắc theo thứ tự ưu tiên:

1.  **New binding:** `new Foo()` -> `this` là object mới tạo.
2.  **Explicit binding:** `call`/`apply`/`bind` -> `this` là arg đầu tiên.
3.  **Implicit binding:** `obj.fn()` -> `this` là `obj` (object trước dấu chấm).
4.  **Default binding:** gọi trần `fn()` -> non-strict thì `window`/`global`, strict thì `undefined`.

Arrow function **không có `this` riêng**, nó lexical `this` từ scope bao quanh. Đây là lý do arrow function rất hợp cho callback.

```javascript
const obj = {
  name: 'A',
  regular() { console.log(this.name); },
  arrow: () => console_quick(this?.name) // this là window/undefined, không phải obj
};
obj.regular(); // 'A' (implicit)

const fn = obj.regular;
fn(); // undefined (default, mất implicit)

function Foo(name) { this.name = name; }
const f = new Foo('B'); // new binding thắng hết

// bind ưu tiên hơn implicit
const bound = obj.regular.bind({ name: 'C' });
bound.call({ name: 'D' }); // vẫn 'C' - bind cứng
```

Thứ tự ưu tiên: `new` > `bind` > `implicit` > `default`. `bind` tạo "hard binding" không override được bằng `call`.

**Trade-off:** Dùng arrow function để tránh mất `this` trong callback, nhưng không dùng arrow làm method trong object/class nếu cần `this` động, và không dùng arrow làm constructor.

**Lỗi thường gặp:** Mất `this` khi pass `obj.method` làm callback (`onClick={obj.handleClick}`), nhầm arrow function có `this` riêng.

**Câu hỏi đào sâu:** Vì sao React class component ngày xưa phải `this.handleClick = this.handleClick.bind(this)`? `this` trong event handler của DOM khác gì?

---

### Câu 4: Prototype, Prototype Chain và Kế thừa

**Trả lời Senior:**
Mỗi object có một hidden link `[[Prototype]]` (truy cập qua `__proto__` hay `Object.getPrototypeOf`). Khi truy cập property, engine tìm trên object trước, không thấy thì leo lên prototype, rồi prototype của prototype... tới `null`. Đó là prototype chain.

Function có thêm `.prototype` - object sẽ được gán làm `[[Prototype]]` cho instance khi `new`. `class` chỉ là syntactic sugar cho pattern này.

```javascript
function Animal(name) { this.name = name; }
Animal.prototype.speak = function() { return `${this.name} kêu`; };

function Dog(name) { Animal.call(this, name); }
Object.setPrototypeOf(Dog.prototype, Animal.prototype); // kế thừa
Dog.prototype.bark = function() { return 'Gâu'; };

const d = new Dog('Milu');
d.speak(); // tìm trên Dog.prototype -> không có -> lên Animal.prototype -> có
console.log(Object.getPrototypeOf(d) === Dog.prototype); // true
console.log(d instanceof Animal); // true - kiểm tra chain

// ES6
class Animal2 {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} kêu`; }
}
class Dog2 extends Animal2 { bark() { return 'Gâu'; } }
```

Cơ chế `instanceof` chỉ kiểm `prototype` có nằm trên chain không. `hasOwnProperty` để phân biệt property riêng vs prototype.

**Trade-off / Khi nào không dùng:** Prototype inheritance mạnh nhưng khó suy luận hơn composition. Trong JS hiện đại, ưu tiên composition, factory, hoặc class đơn giản; tránh chỉnh `__proto__` trực tiếp (chậm), dùng `Object.create` hoặc `class extends`.

**Lỗi thường gặp:** Gán `Child.prototype = Parent.prototype` (làm 2 class share chung prototype), quên `Child.prototype.constructor`, hoặc method arrow trong prototype (không có prototype riêng).

**Câu hỏi đào sâu:** `Object.create(null)` khác `{}` thế nào? Vì sao `Array.prototype.map` lại không enumerable?

---

### Câu 5: Event Loop, Microtask và Macrotask

**Trả lời Senior:**
JS single-thread, concurrency nhờ Event Loop. Call Stack chạy sync code. Khi gặp async (setTimeout, Promise, fetch), Web API/DOM xử lý rồi đẩy callback vào Queue. Event Loop liên tục check: Stack rỗng thì ưu tiên **drain microtask queue** trước, rồi mới lấy 1 macrotask.

- **Microtask:** `Promise.then/catch/finally`, `queueMicrotask`, `MutationObserver`, `process.nextTick` (Node, ưu tiên hơn cả Promise).
- **Macrotask:** `setTimeout`, `setInterval`, `setImmediate` (Node), I/O, `MessageChannel`.
- **rAF:** `requestAnimationFrame` không phải macrotask, chạy ở rendering phase trước paint, sau microtask/macrotask.

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
queueMicrotask(() => console.log('4'));
console.log('5');
// 1, 5, 3, 4, 2

// Ví dụ Senior phải giải thích được:
async function foo() {
  console.log('A');
  await Promise.resolve(); // await tách thành microtask
  console.log('B');
}
console.log('C');
foo();
console.log('D');
// C, A, D, B (vì await yield ra microtask)
```

Starvation: nếu microtask tạo thêm microtask liên tục, macrotask (như render, user input) bị đói.

**Trade-off:** Dùng microtask để batch update (Vue, React), nhưng không lạm dụng để tránh block rendering. `setTimeout(0)` để defer xuống macrotask, nhường cho paint.

**Lỗi thường gặp:** Nghĩ `setTimeout 0` chạy ngay sau sync; không hiểu `await` luôn async dù resolve sẵn; quên cleanup microtask khi component unmount.

**Câu hỏi đào sâu:** Vì sao `Promise.then` luôn async dù Promise đã resolve? `requestAnimationFrame` nằm ở đâu trong loop? Node vs Browser Event Loop khác gì?

---

### Câu 6: Promise, async/await và xử lý lỗi bất đồng bộ

**Trả lời Senior:**
Promise là object đại diện giá trị tương lai, có 3 trạng thái: `pending` -> `fulfilled`/`rejected` (chỉ chuyển 1 lần). `.then` luôn trả Promise mới, cho phép chaining. `async/await` là sugar cho Promise + generator, giúp code trông sync nhưng bản chất vẫn non-blocking.

Cơ chế: `await x` sẽ `Promise.resolve(x)` rồi suspend function, phần còn lại được đưa vào microtask khi Promise settle. `try/catch` quanh `await` bắt được reject.

```javascript
// Anti-pattern: callback hell -> Promise chain -> async/await
async function getUser(id) {
  try {
    const user = await fetch(`/api/user/${id}`).then(r => {
      if (!r.ok) throw new Error(r.statusText);
      return r.json();
    });
    const posts = await fetch(`/api/posts?user=${user.id}`).then(r => r.json());
    return { user, posts };
  } catch (e) {
    // bắt cả network error lẫn throw
    console.error(e);
    throw e; // re-throw để caller quyết định
  } finally {
    // cleanup loading state
  }
}

// Chạy song song vs tuần tự
const [a, b] = await Promise.all([fetchA(), fetchB()]); // song song, fail-fast
const results = await Promise.allSettled([fetchA(), fetchB()]); // đợi hết dù lỗi
const first = await Promise.race([fetchA(), timeout(3000)]); // timeout pattern
```

**Trade-off:** `Promise.all` fail-fast - 1 lỗi là reject hết, nếu cần resilient thì dùng `allSettled`. `await` trong loop `for...of` là tuần tự (chậm), muốn song song phải `map` + `Promise.all`. Tránh `async` executor trong `new Promise(async () => {})` - anti-pattern.

**Lỗi thường gặp:** Quên `return` trong `.then`, không `catch` dẫn tới `unhandledrejection`, dùng `await` trong loop không cần thiết, quên `Promise` không cancel được (cần AbortController).

**Câu hỏi đào sâu:** Promise có cancel được không? Làm sao implement `Promise.withResolvers()`? Vì sao `await` trong `Array.forEach` không hoạt động như mong đợi?

---

### Câu 7: Iterator và Generator

**Trả lời Senior:**
Iterator là object có `.next()` trả `{ value, done }`, tuân theo Iterable protocol (`[Symbol.iterator]`). Generator (`function*`) là factory tạo iterator, có khả năng **pause/resume** qua `yield`. Nó cho phép viết async flow trước khi có async/await (co library).

```javascript
// Custom iterable
const range = {
  *[Symbol.iterator]() {
    for (let i = 1; i <= 3; i++) yield i;
  }
};
console.log([...range]); // [1,2,3]

// Generator điều khiển luồng
function* gen() {
  const a = yield 1;
  console.log('a=', a); // giá trị truyền vào next(a)
  yield 2;
  return 3;
}
const it = gen();
console.log(it.next()); // {value:1, done:false}
console.log(it.next('hello')); // a= hello, {value:2, done:false}
console.log(it.next()); // {value:3, done:true}

// Ứng dụng Senior: lazy sequence vô hạn
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) { yield a; [a, b] = [b, a + b]; }
}
for (const n of fibonacci()) { if (n > 100) break; console.log(n); }

// Trước async/await: co + generator để viết async như sync
```

Generator còn có `yield*` để delegate, `return`/`throw` để điều khiển từ ngoài. Trong Redux-Saga, `yield` effect được middleware interpret.

**Trade-off / Khi nào không dùng:** Generator code khó đọc hơn async/await cho async đơn thuần, chỉ nên dùng khi cần lazy evaluation, custom iteration, hoặc control flow phức tạp (state machine). Không dùng trong hot path nếu lo overhead.

**Lỗi thường gặp:** Nhầm `yield` vs `yield*`, quên generator là lazy (không chạy tới khi `.next()`), không handle `return` làm iterator done sớm.

**Câu hỏi đào sâu:** Làm sao implement `async/await` bằng generator + Promise? `for await...of` hoạt động thế nào?

---

### Câu 8: Currying và Partial Application

**Trả lời Senior:**
Currying biến hàm `f(a,b,c)` thành `f(a)(b)(c)` - mỗi lần nhận 1 arg trả hàm mới. Partial Application cố định một số arg trước, trả hàm nhận phần còn lại. Cả hai đều dựa trên closure.

Mục đích: tái sử dụng logic, tạo hàm chuyên biệt, composition tốt hơn, và hỗ trợ point-free style.

```javascript
// Curry thủ công
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn.apply(this, args);
    return (...next) => curried.apply(this, [...args, ...next]);
  };
}
const add = (a, b, c) => a + b + c;
const curriedAdd = curry(add);
curriedAdd(1)(2)(3); // 6
curriedAdd(1, 2)(3); // 6

// Partial application thực tế hơn
function partial(fn, ...preset) {
  return (...rest) => fn(...preset, ...rest);
}
const fetchWithAuth = partial(fetch, { headers: { Authorization: 'Bearer xxx' } });

// Use case Senior: HOC, middleware
const withLogging = fn => (...args) => {
  console.log('call', fn.name, args);
  return fn(...args);
};
const withRetry = (fn, n = 3) => async (...args) => {
  for (let i = 0; i < n; i++) try { return await fn(...args); } catch {}
  throw new Error('retry failed');
};
```

Lodash `_.curry` và `_.partial` đã tối ưu placeholder (`_`). Trong FP, Ramda curry mặc định.

**Trade-off:** Currying làm call site dài, khó debug stack, và TypeScript infer type cho curry rất phức tạp. Partial application thường thực tế hơn. Không curry hàm có arity động hoặc optional args nhiều.

**Lỗi thường gặp:** Curry hàm có `this` mà mất binding, không xử lý `fn.length` khi hàm có default param (length bị giảm).

**Câu hỏi đào sâu:** Curry và partial khác nhau thế nào? Làm sao curry function có rest param?

---

### Câu 9: Debounce vs Throttle

**Trả lời Senior:**
Cả hai đều rate-limiting nhưng chiến lược khác:

- **Debounce:** chỉ chạy **sau khi** ngừng trigger một khoảng `wait`. Mỗi lần trigger mới sẽ reset timer. Dùng cho search input, resize, auto-save.
- **Throttle:** chạy **tối đa 1 lần mỗi `wait`**, đều đặn. Dùng cho scroll, mousemove, drag.

```javascript
function debounce(fn, wait, immediate = false) {
  let timer;
  return function(...args) {
    const callNow = immediate && !timer;
    clearTimeout(timer);
    timer = setTimeout(() => {
      timer = null;
      if (!immediate) fn.apply(this, args);
    }, wait);
    if (callNow) fn.apply(this, args);
  };
}

function throttle(fn, wait, { leading = true, trailing = true } = {}) {
  let last = 0, timer;
  return function(...args) {
    const now = Date.now();
    const remaining = wait - (now - last);
    if (remaining <= 0) {
      if (timer) { clearTimeout(timer); timer = null; }
      last = now;
      fn.apply(this, args);
    } else if (trailing && !timer) {
      timer = setTimeout(() => {
        last = leading ? Date.now() : 0;
        timer = null;
        fn.apply(this, args);
      }, remaining);
    }
  };
}

// React hook
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = React.useState(value);
  React.useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);
  return debounced;
}
```

**Trade-off:** Debounce có cảm giác trễ, throttle có thể bỏ lỡ event cuối (nếu không có trailing). Chọn leading/trailing tùy UX. Trong search, dùng debounce + AbortController để cancel request cũ.

**Lỗi thường gặp:** Tạo debounce mới mỗi render (mất tác dụng), quên cleanup timer khi unmount, không cancel debounce khi component unmount gây setState on unmounted.

**Câu hỏi đào sâu:** Implement debounce có `cancel` và `flush` như lodash? Khi nào dùng `requestAnimationFrame` thay throttle?

---

### Câu 10: Shallow Copy vs Deep Copy

**Trả lời Senior:**
Shallow copy chỉ copy 1 cấp, nested object vẫn share reference. Deep copy copy toàn bộ cây. Trong JS, spread `...`, `Object.assign` là shallow. `JSON.parse(JSON.stringify(obj))` là deep nhưng mất `undefined`, `function`, `Date`, `Map`, circular.

```javascript
const a = { x: 1, nested: { y: 2 } };
const shallow = { ...a };
shallow.nested.y = 99;
console.log(a.nested.y); // 99 - bị mutate!

// Deep copy hiện đại
const deep = structuredClone(a); // native, hỗ trợ circular, Date, Map, ArrayBuffer
deep.nested.y = 3; // không ảnh hưởng a

// JSON trick hạn chế
const jsonDeep = JSON.parse(JSON.stringify(a)); // mất function, undefined, Symbol

// Tự implement deepClone có xử lý circular
function deepClone(obj, seen = new WeakMap()) {
  if (obj === null || typeof obj !== 'object') return obj;
  if (seen.has(obj)) return seen.get(obj);
  const clone = Array.isArray(obj) ? [] : {};
  seen.set(obj, clone);
  for (const k in obj) clone[k] = deepClone(obj[k], seen);
  return clone;
}
// Lưu ý: đây là demo đơn giản - bỏ qua Symbol, non-enumerable, Date/Map/Set, RegExp. Production dùng structuredClone cho các case đó, nhưng structuredClone cũng không clone function/DOM node/prototype.
```

Trong React/Redux, immutability yêu cầu shallow copy ở mỗi level thay đổi để reference check (`===`) phát hiện thay đổi. Deep clone toàn bộ state mỗi lần là lãng phí.

**Trade-off:** `structuredClone` nhanh và chuẩn nhưng không clone function, DOM node, và có thể ném với non-cloneable. Với state lớn, chỉ nên shallow copy path cần update (immer làm giúp).

**Lỗi thường gặp:** Nghĩ spread là deep, dùng JSON deep clone cho object có Date/Map, mutate nested state trong Redux.

**Câu hỏi đào sâu:** Vì sao `structuredClone` không clone function? Immer hoạt động thế nào để tối ưu copy?

---

### Câu 11: == vs ===, Coercion và Object.is

**Trả lời Senior:**
`===` (strict) không coercion, so sánh cả type và value. `==` (loose) có Abstract Equality Comparison với quy tắc coercion phức tạp: `null == undefined` => true, `"1" == 1` => true do ToNumber, `[] == ![]` => true (cả hai về 0). Gần như luôn dùng `===`.

`Object.is` là "strict hơn" `===`: phân biệt `-0` vs `+0` và coi `NaN` bằng `NaN`.

```javascript
console.log(0 === -0); // true
console.log(Object.is(0, -0)); // false

console.log(NaN === NaN); // false
console.log(Object.is(NaN, NaN)); // true

// Coercion bẫy
console.log([] == false); // true ( [] -> "" -> 0, false -> 0 )
console.log("" == 0); // true
console.log(null == undefined); // true - đây là case duy nhất nên dùng ==
console.log(" \t\r\n" == 0); // true

// Falsy list: false, 0, -0, 0n, "", null, undefined, NaN
if (!value) {} // catch hết falsy, nhưng 0 cũng bị bắt
```

Trong React, `Object.is` được dùng để so sánh dependency của hooks (`Object.is(prev, next)`).

**Trade-off / Khi nào không dùng:** Chỉ dùng `==` khi check `x == null` để bắt cả `null` và `undefined` gọn. Còn lại luôn `===`. Dùng `Object.is` khi cần semantics chính xác (polyfill, hook compare).

**Lỗi thường gặp:** Dùng `==` vô tội vạ, so sánh object bằng `==` (chỉ so reference), check `if (x == false)` thay vì `!x`.

**Câu hỏi đào sâu:** Vì sao React chọn `Object.is` thay vì `===` cho hook deps? Khi nào `==` hữu ích thật sự?

---

### Câu 12: call / apply / bind

**Trả lời Senior:**
Cả ba đều để điều khiển `this` và đều là method của `Function.prototype`.

- `call(thisArg, arg1, arg2)` - gọi ngay, args rời.
- `apply(thisArg, [args])` - gọi ngay, args mảng.
- `bind(thisArg, ...presetArgs)` - **không gọi**, trả hàm mới đã hard-bind `this` + preset args (partial application).

```javascript
function greet(greeting, punctuation) {
  return `${greeting} ${this.name}${punctuation}`;
}
const user = { name: 'An' };
greet.call(user, 'Hello', '!'); // Hello An!
greet.apply(user, ['Hi', '!!']); // Hi An!!
const bound = greet.bind(user, 'Hey');
bound('...'); // Hey An...

 // Polyfill bind đơn giản
Function.prototype.myBind = function(ctx, ...preset) {
  const fn = this;
  return function(...rest) {
    return fn.apply(ctx, [...preset, ...rest]);
  };
};

// Use case Senior
// 1. Mượn method
const slice = Array.prototype.slice;
slice.call({ 0: 'a', 1: 'b', length: 2 }); // ['a','b']

// 2. Bind callback giữ this
class Comp {
  constructor() { this.handleClick = this.handleClick.bind(this); }
  handleClick() { console.log(this); }
}
```

`bind` tạo hàm mới nên `new` lên hàm đã bind vẫn tạo instance nhưng `this` bị bỏ qua (spec).

**Trade-off:** `bind` tạo function mới mỗi lần gọi, nếu đặt trong render sẽ phá `memo`/`PureComponent`. `call/apply` overhead nhỏ nhưng dùng nhiều trong hot path cũng tốn.

**Lỗi thường gặp:** Bind arrow function (vô tác dụng vì arrow không có this), nhầm thứ tự args của `call` vs `apply`.

**Câu hỏi đào sâu:** Implement `bind` hỗ trợ `new` đúng spec? Vì sao `bind` chậm hơn closure trong micro-benchmark?

---

### Câu 13: Event Bubbling, Capturing và Delegation

**Trả lời Senior:**
DOM Event có 3 phase: **Capturing** (từ window xuống target), **Target**, **Bubbling** (từ target lên window). Mặc định `addEventListener` là bubbling (`capture: false`). Đặt `capture: true` để bắt ở capturing.

Event Delegation là tận dụng bubbling: đặt 1 listener trên parent để xử lý cho nhiều child (kể cả child thêm động), giảm memory và listener.

```javascript
// Delegation
document.querySelector('#list').addEventListener('click', (e) => {
  const btn = e.target.closest('button[data-id]');
  if (!btn) return;
  console.log('click item', btn.dataset.id);
});

// Thứ tự
div.addEventListener('click', () => console.log('bubble'), false);
div.addEventListener('click', () => console.log('capture'), true);
// capture -> target -> bubble

// Ngăn chặn
e.stopPropagation(); // chặn bubble/capture tiếp
e.stopImmediatePropagation(); // chặn cả listener khác trên cùng element
e.preventDefault(); // chặn hành vi mặc định, không chặn propagation
```

Trong React, Synthetic Event (trước React 17) delegate lên `document`, từ React 17 delegate lên `root`. `e.stopPropagation()` trong React chỉ chặn trong hệ Synthetic, không chặn native nếu không cẩn thận.

**Trade-off:** Delegation tiết kiệm nhưng `e.target` có thể là descendant sâu, cần `closest` để tìm đúng element. Không delegate được event không bubble (`focus`, `blur` - dùng `focusin`/`focusout` hoặc capture).

**Lỗi thường gặp:** Gắn listener cho từng item thay vì delegate, nhầm `preventDefault` với `stopPropagation`, quên `removeEventListener` phải cùng `capture` flag.

**Câu hỏi đào sâu:** Vì sao React 17 đổi delegation từ document sang root? Event nào không bubble?

---

### Câu 14: Memory Leak và Garbage Collection trong JS

**Trả lời Senior:**
JS dùng Mark-and-Sweep GC: từ roots (window, stack) mark object còn reachable, phần không mark sẽ sweep. Leak xảy ra khi giữ reference vô tình tới object không cần nữa, làm GC không thu hồi.

4 nguồn leak kinh điển:

1.  **Closure giữ DOM lớn:** `const handler = () => { console.log(el); }` dù `el` đã remove khỏi DOM nhưng closure vẫn giữ.
2.  **Quên cleanup:** `setInterval`, `addEventListener`, `MutationObserver`, `WebSocket` không remove khi unmount.
3.  **Global variable:** `window.cache = hugeArray` không clear.
4.  **Detached DOM:** giữ reference tới subtree đã remove (`let detached = document.getElementById('x'); document.body.removeChild(detached)` nhưng biến `detached` vẫn giữ cả subtree).

```javascript
// Leak trong SPA
useEffect(() => {
  const onScroll = () => console.log(window.scrollY);
  window.addEventListener('scroll', onScroll);
  const id = setInterval(tick, 1000);
  return () => {
    window.removeEventListener('scroll', onScroll);
    clearInterval(id);
  };
}, []);

// WeakMap/WeakRef để tránh leak
const cache = new WeakMap(); // key là object, khi key GC thì entry tự mất
let obj = { data: 'big' };
cache.set(obj, 'meta');
obj = null; // GC được
```

Debug bằng Chrome DevTools Memory tab: Heap snapshot, Allocation timeline, tìm Detached elements.

**Trade-off:** `WeakMap`/`WeakSet` không enumerable, không có `size`. `WeakRef` + `FinalizationRegistry` rất hiếm khi cần.

**Lỗi thường gặp:** SetInterval trong useEffect không cleanup, closure trong debounce giữ component instance, cache Map không giới hạn.

**Câu hỏi đào sâu:** Mark-and-Sweep khác Reference Counting thế nào? Vì sao circular reference không còn là vấn đề?

---

### Câu 15: CommonJS vs ES Modules và Tree-shaking

**Trả lời Senior:**
CommonJS (`require/module.exports`) là **dynamic, synchronous**, `require` có thể ở trong `if`, export là object copy. ESM (`import/export`) là **static, asynchronous**, phải ở top-level, binding là live (import là reference tới export, không phải copy). Static cho phép tree-shaking và scope hoisting.

```javascript
// CJS
const fs = require('fs');
if (condition) { const x = require('./x'); }
module.exports = { foo };

// ESM
import fs from 'fs';
import { foo } from './foo.js'; // static, phải top-level
export const bar = 1;
export default App;

// Live binding
// counter.js
export let count = 1;
export function inc() { count++; }
// main.js
import { count, inc } from './counter.js';
console.log(count); // 1
inc();
console.log(count); // 2 - live!

// CJS: primitive thì copy snapshot, object thì reference cùng object nhưng re-assign exports không live
// Với object: module.exports = {count:1}; require -> reference cùng object; nhưng gán lại module.exports = {count:2} thì require cũ không thấy
```

Tree-shaking dựa vào static analysis của ESM: bundler (Rollup, esbuild, Webpack) mark export không dùng thì drop. Cần `sideEffects: false` trong package.json và tránh dynamic `import()` lung tung.

Interop: `import cjs from './cjs'` sẽ lấy `module.exports` làm default. Dual package cần conditional exports.

**Trade-off:** ESM là tương lai, nhưng CJS vẫn cần cho Node legacy và dynamic require. Dùng `import()` dynamic để code-splitting, nhưng mất tree-shaking cho phần dynamic.

**Lỗi thường gặp:** Dùng `require` trong ESM, `__dirname` không có trong ESM (dùng `import.meta.url`), circular ESM live binding gây TDZ.

**Câu hỏi đào sâu:** Vì sao ESM tree-shakable còn CJS không? `sideEffects` ảnh hưởng thế nào?

---

### Câu 16: Proxy vs Object.defineProperty

**Trả lời Senior:**
`Object.defineProperty` định nghĩa getter/setter cho **từng property đã biết**, không intercept thêm/xóa property, không intercept array index mới, phải duyệt object trước. `Proxy` wrap **cả object**, intercept 13 trap (`get`, `set`, `has`, `deleteProperty`, `ownKeys`...), hỗ trợ cả property động và array, và có `Reflect` để forward đúng semantics.

Vue 2 dùng `defineProperty` nên không reactive với `obj.newProp = 1` hay `arr[3] = x` (phải `Vue.set`). Vue 3 đổi sang `Proxy` nên fix hết, và hỗ trợ `Map/Set`.

```javascript
// defineProperty hạn chế
Object.defineProperty(obj, 'x', {
  get() { return this._x; },
  set(v) { console.log('set', v); this._x = v; }
});
obj.y = 2; // không trigger

// Proxy toàn diện
const reactive = (target) => new Proxy(target, {
  get(t, p, receiver) {
    console.log('get', p);
    return Reflect.get(t, p, receiver);
  },
  set(t, p, v, receiver) {
    console.log('set', p, v);
    return Reflect.set(t, p, v, receiver);
  },
  deleteProperty(t, p) { console.log('delete', p); return Reflect.deleteProperty(t, p); }
});
const state = reactive({ count: 0 });
state.count++; // log get + set
state.newProp = 1; // vẫn log - Proxy bắt được

// Proxy + Reflect quan trọng để giữ this đúng
```

**Trade-off:** Proxy không polyfill được cho IE11, performance overhead cao hơn cho object nhỏ, và `proxy !== target` (so sánh reference sai). `defineProperty` nhẹ hơn cho ít property.

**Lỗi thường gặp:** So sánh `proxy === target` ra false, quên dùng `Reflect`, Proxy không deep tự động (phải recursively proxy).

**Câu hỏi đào sâu:** Vì sao Vue 3 vẫn cần `reactive` deep proxy? Proxy có performance penalty thế nào?

---

### Câu 17: Symbol, BigInt, WeakMap / WeakSet

**Trả lời Senior:**
- **Symbol:** primitive unique, dùng làm key ẩn, tránh collision. `Symbol('foo') !== Symbol('foo')`. `Symbol.for('foo')` thì global registry. Well-known symbols: `Symbol.iterator`, `Symbol.toStringTag`, `Symbol.hasInstance` cho meta-programming.
- **BigInt:** số nguyên tùy ý lớn (`123n`), không lẫn với Number. Dùng cho ID 64-bit, tiền tệ chính xác. Không mix `BigInt + Number` được, phải convert.
- **WeakMap/WeakSet:** key phải là object, giữ **weak reference** - không ngăn GC. Không iterable, không có `size`. Dùng để gắn metadata cho object mà không leak.

```javascript
// Symbol làm private
const PRIVATE = Symbol('private');
class Foo {
  [PRIVATE] = 123;
  getPrivate() { return this[PRIVATE]; }
}
Object.keys(new Foo()); // [] - Symbol không enumerable

// BigInt
const id = 9007199254740993n; // vượt Number.MAX_SAFE_INTEGER
console.log(typeof id); // bigint
// console.log(id + 1); // TypeError
console.log(id + 1n); // 9007199254740994n

// WeakMap cache DOM metadata không leak
const meta = new WeakMap();
function attach(el, data) { meta.set(el, data); }
let el = document.createElement('div');
attach(el, { clicks: 0 });
el = null; // GC được, entry WeakMap tự mất

// WeakSet tracking visited object
const seen = new WeakSet();
function traverse(obj) {
  if (seen.has(obj)) return;
  seen.add(obj);
  // ...
}
```

**Trade-off:** Symbol làm debug khó (không hiện trong JSON). BigInt không JSON.stringify được, chậm hơn Number. WeakMap không duyệt được nên không dùng làm cache cần iterate.

**Lỗi thường gặp:** Dùng Symbol làm key rồi `JSON.stringify` mất, mix BigInt và Number, nghĩ WeakMap có `clear`/`size`.

**Câu hỏi đào sâu:** Khi nào dùng `Symbol.for` vs `Symbol()`? Làm sao serialize BigInt qua JSON?

---

### Câu 18: Xử lý lỗi đồng bộ và bất đồng bộ

**Trả lời Senior:**
Đồng bộ dùng `try/catch`, bất đồng bộ Promise dùng `.catch` hoặc `try/catch` với `await`. `try/catch` không bắt được lỗi async callback nếu không `await`.

Global handler: `window.onerror`, `window.onunhandledrejection`, `process.on('uncaughtException')` (Node). Trong React có Error Boundary (chỉ class) để bắt render error.

```javascript
// Đồng bộ
try {
  JSON.parse('invalid');
} catch (e) {
  if (e instanceof SyntaxError) console.log('json lỗi');
}

// Bất đồng bộ - sai
try {
  setTimeout(() => { throw new Error('oops'); }, 0);
} catch (e) {} // không bắt được vì throw ở macrotask khác

// Đúng
window.addEventListener('unhandledrejection', e => {
  console.error('Unhandled', e.reason);
  e.preventDefault();
});

async function load() {
  try {
    const res = await fetch('/api');
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (e) {
    // phân loại lỗi
    if (e.name === 'AbortError') console.log('bị cancel');
    else throw e; // re-throw cho caller
  }
}

// Custom Error
class AppError extends Error {
  constructor(message, code) {
    super(message);
    this.name = 'AppError';
    this.code = code;
  }
}
```

**Trade-off:** Không nên `try/catch` bọc toàn bộ app, chỉ bọc boundary có thể recover. Trong async, luôn `await` hoặc `return` Promise để lỗi propagate.

**Lỗi thường gặp:** Quên `catch` cho Promise chain, `try/catch` quanh `setTimeout`, throw string thay vì `Error` (mất stack).

**Câu hỏi đào sâu:** `unhandledrejection` vs `rejectionhandled` khác gì? Vì sao Error Boundary không bắt được async error?

---

### Câu 19: any vs unknown vs never vs void

**Trả lời Senior:**
Bốn type này hay nhầm nhưng semantics khác hẳn:

- **any:** tắt type checking, compatible với mọi type (cả assign vào và ra). Là escape hatch, mất an toàn.
- **unknown:** an toàn hơn - có thể assign bất kỳ giá trị vào `unknown`, nhưng **không được** dùng gì cho tới khi narrow. Là `any` an toàn.
- **never:** type không có giá trị nào, dùng cho hàm không bao giờ return (throw, infinite loop) và exhaustive check. `never` là bottom type, assign được cho mọi type nhưng không type nào assign vào `never` (trừ `never`).
- **void:** hàm không return gì có ý nghĩa, `undefined` là subtype của `void` nhưng `void` cho phép `undefined` còn `never` thì không.

```typescript
let a: any = 1;
a.foo.bar(); // không lỗi - nguy hiểm

let u: unknown = "hello";
u.toUpperCase(); // lỗi - phải narrow
if (typeof u === "string") u.toUpperCase(); // ok

function throwErr(msg: string): never { throw new Error(msg); }
function exhaustive(x: never): never { throw new Error(`Unhandled ${x}`); }

type Shape = { kind: "circle"; r: number } | { kind: "square"; size: number };
function area(s: Shape) {
  switch (s.kind) {
    case "circle": return Math.PI * s.r ** 2;
    case "square": return s.size ** 2;
    default: exhaustive(s); // nếu thiếu case, s không phải never -> lỗi compile
  }
}

function log(msg: string): void { console.log(msg); }
```

**Trade-off:** Dùng `unknown` thay `any` khi nhận data từ API, `never` cho exhaustive check, `void` cho callback không quan tâm return.

**Lỗi thường gặp:** Lạm dụng `any`, nhầm `void` với `undefined`, không dùng `never` để bắt missing case.

**Câu hỏi đào sâu:** Vì sao `unknown` an toàn hơn `any`? Khi nào `never` xuất hiện trong conditional type?

---

### Câu 20: type vs interface

**Trả lời Senior:**
Cả hai đều định nghĩa shape, nhưng khác năng lực:

- **interface:** chỉ cho object/class, hỗ trợ **declaration merging** (khai báo lại cùng tên sẽ merge), `extends` để kế thừa, implement được bởi class. Tốt cho public API, library.
- **type:** alias cho mọi type (union, intersection, tuple, conditional, mapped...), dùng `&` để intersection, không merge được (trùng tên lỗi). Mạnh hơn cho type transformations.

```typescript
// interface merging
interface User { name: string; }
interface User { age: number; } // merge -> { name, age }

// type không merge
type UserType = { name: string };
// type UserType = { age: number }; // Error duplicate

// Khả năng
type StringOrNumber = string | number; // interface không làm được
type Nullable<T> = T | null;
type Keys = keyof { a: 1; b: 2 }; // "a" | "b"

interface A { x: number; }
interface B extends A { y: number; } // extends
type C = A & { y: number }; // intersection tương đương

// Performance: interface được cache tốt hơn trong large union, type alias phức tạp có thể chậm
```

Khuyến nghị team: dùng `interface` cho object shape public, `type` cho union/utility/complex. Consistency quan trọng hơn tranh cãi.

**Trade-off:** `interface` merge có thể gây surprise nếu lib augment global. `type` với big intersection có thể tạo type khó đọc.

**Lỗi thường gặp:** Dùng `type` để extend interface bằng `&` tạo intersection khổng lồ chậm, hoặc lạm dụng merging gây conflict.

**Câu hỏi đào sâu:** Khi nào nên dùng `interface` để tận dụng declaration merging? `type` có thể implement bởi class không?

---

### Câu 21: Generics và Constraints

**Trả lời Senior:**
Generics cho phép viết code tái sử dụng mà vẫn giữ type safety - như template. Constraint (`extends`) giới hạn type param phải có shape nhất định. `keyof`, `extends`, default type param là công cụ chính.

```typescript
// Cơ bản
function identity<T>(x: T): T { return x; }
identity<string>("hello");

// Constraint
function getLength<T extends { length: number }>(x: T): number {
  return x.length;
}
getLength("hi"); // ok
getLength([1, 2]); // ok
// getLength(123); // lỗi

// Keyof constraint - type-safe property access
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const user = { name: "An", age: 20 };
getProp(user, "name"); // string
// getProp(user, "email"); // lỗi

// Generic với class và default
class Store<T = string> {
  private data: T[] = [];
  add(item: T) { this.data.push(item); }
}

// Multiple constraints
type HasId = { id: string };
function merge<T extends HasId, U extends HasId>(a: T, b: U): T & U {
  return { ...a, ...b };
}
```

Variance: `T extends U` không phải lúc nào cũng intuititive với function param (contravariant).

**Trade-off:** Generics quá sâu làm error message khó đọc, compile chậm. Đừng generic hóa sớm - chỉ khi có ít nhất 2 use case.

**Lỗi thường gặp:** Đặt `any` trong constraint, quên `extends keyof`, generic không infer được phải specify thủ công.

**Câu hỏi đào sâu:** `T extends string` vs `T = string` khác gì? Làm sao constraint generic phải là key của object khác?

---

### Câu 22: Utility Types và tự implement

**Trả lời Senior:**
TypeScript cung cấp sẵn utility types, nhưng Senior phải hiểu implement để custom.

```typescript
// Partial - làm optional hết
type MyPartial<T> = { [P in keyof T]?: T[P] };

// Required - bỏ optional
type MyRequired<T> = { [P in keyof T]-?: T[P] };

// Readonly
type MyReadonly<T> = { readonly [P in keyof T]: T[P] };

// Pick - chọn subset
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };

// Omit - bỏ keys
type MyOmit<T, K extends keyof any> = MyPick<T, Exclude<keyof T, K>>;

// Record - map key -> value
type MyRecord<K extends keyof any, T> = { [P in K]: T };

// Exclude / Extract dựa trên conditional + never
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;

// NonNullable
type MyNonNullable<T> = T extends null | undefined ? never : T; // chuẩn lib, shorthand T & {} cũng hoạt động vì {} loại null/undefined nhưng che nghĩa

// ReturnType
type MyReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;

// Awaited - unwrap Promise
type MyAwaited<T> = T extends PromiseLike<infer U> ? MyAwaited<U> : T;

// Thực tế: DeepPartial cho form
type DeepPartial<T> = T extends object ? { [P in keyof T]?: DeepPartial<T[P]> } : T;
```

Biết implement giúp hiểu `infer`, `never` distribution, và viết utility riêng cho domain (VD: `DeepReadonly`, `StrictOmit`).

**Trade-off:** Utility lồng nhau quá sâu làm type instantiation rất chậm. Dùng `Omit` nhiều lần trên union lớn có thể blow up.

**Lỗi thường gặp:** Dùng `Omit` trên union mà không distributive, nhầm `Partial` với `DeepPartial`.

**Câu hỏi đào sâu:** Implement `DeepReadonly`? Vì sao `Exclude` dùng `never` để filter?

---

### Câu 23: Conditional Types, infer và Distributive

**Trả lời Senior:**
Conditional type `T extends U ? X : Y` là if ở level type. Khi `T` là naked type param (không bọc trong `[]`, `Promise<>`...), nó **distributive**: tự split union. `infer` để "bắt" type con ra.

```typescript
// Distributive
type ToArray<T> = T extends any ? T[] : never;
type A = ToArray<string | number>; // string[] | number[]  (distribute)
type B = ToArray<string | number> extends any ? string[] : never; // không distribute nếu bọc

// Tắt distributive: bọc trong tuple
type NoDist<T> = [T] extends [any] ? T[] : never;
type C = NoDist<string | number>; // (string | number)[]

// infer
type ReturnType<T> = T extends (...args: any) => infer R ? R : never;
type R = ReturnType<() => string>; // string

type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type U = UnwrapPromise<Promise<string>>; // string

// infer với nhiều vị trí
type Last<T extends any[]> = T extends [...infer _, infer L] ? L : never;
type L = Last<[1, 2, 3]>; // 3

// Thực tế Senior: Getters type
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type G = Getters<{ name: string; age: number }>; // { getName: ()=>string, getAge: ()=>number }

// Recursive conditional
type Flatten<T> = T extends (infer U)[] ? Flatten<U> : T;
type F = Flatten<number[][][]>; // number
```

**Trade-off:** Conditional quá sâu làm compile chậm, error khó đọc. Distributive là feature nhưng cũng là trap khi không muốn split.

**Lỗi thường gặp:** Quên distributive làm `Exclude<string|number, string>` ra `never` đúng nhưng `Exclude` bọc sai ra sai; `infer` không đặt đúng vị trí.

**Câu hỏi đào sâu:** Làm sao tắt distributive? `infer` có thể infer nhiều type cùng lúc không?

---

### Câu 24: Mapped Types và Template Literal Types

**Trả lời Senior:**
Mapped type biến đổi mỗi property của type (`[P in keyof T]`), Template Literal type biến đổi string ở level type (`` `get${Capitalize<K>}` ``). Kết hợp tạo DSL type-safe.

```typescript
// Mapped cơ bản
type OptionsFlags<T> = { [P in keyof T as `is${Capitalize<string & P>}`]: boolean };
type Flags = OptionsFlags<{ darkMode: string }>; // { isDarkMode: boolean }

// Modifier: -? bỏ optional, +? thêm optional, -readonly
type Mutable<T> = { -readonly [P in keyof T]: T[P] };
type WithOptional<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

// Template Literal
type EventName<T extends string> = `on${Capitalize<T>}`;
type E = EventName<"click" | "change">; // "onClick" | "onChange"

// Kết hợp: CSS props to style object
type StyleKey = "color" | "background";
type StyleValue = string | number;
type Style = { [K in StyleKey as `style${Capitalize<K>}`]: StyleValue };
// { styleColor: ..., styleBackground: ... }

// Senior pattern: Route params
type Route = "/user/:id" | "/post/:postId/comment/:commentId";
type Params<T extends string> = T extends `${string}:${infer P}/${infer Rest}`
  ? P | Params<Rest>
  : T extends `${string}:${infer P}` ? P : never;
type P = Params<"/post/:postId/comment/:commentId">; // "postId" | "commentId"

// intrinsic: Uppercase, Lowercase, Capitalize, Uncapitalize
type Upper = Uppercase<"hello">; // "HELLO"
```

Dùng `as` trong mapped để remap key (key remapping, TS 4.1+).

**Trade-off:** Template literal + recursion dễ hit recursion limit (50 levels). Mapped quá rộng làm autocomplete chậm.

**Lỗi thường gặp:** Quên `string & K` khi `K` có thể là `symbol`, template literal không handle union đúng nếu không distributive.

**Câu hỏi đào sâu:** Làm sao tạo type `SnakeToCamel` bằng template literal + recursion?

---

### Câu 25: Strict Mode và strictNullChecks

**Trả lời Senior:**
`strict: true` trong tsconfig bật tất cả strict flags, quan trọng nhất là `strictNullChecks`. Khi tắt, `null`/`undefined` là subtype của mọi type, nên `string` có thể nhận `null` mà không lỗi - mất an toàn. Bật lên, phải explicit `string | null`.

Các flag trong `strict`:

- `strictNullChecks`: `null`/`undefined` phải explicit.
- `noImplicitAny`: cấm `any` ngầm.
- `strictFunctionTypes`: check function param contravariant đúng.
- `strictBindCallApply`: check `call`/`bind` đúng type.
- `noImplicitThis`: `this` phải typed.
- `alwaysStrict`: emit `"use strict"`.

```typescript
// strictNullChecks off (xấu)
let s: string = null; // không lỗi - bug runtime
s.toUpperCase(); // runtime crash

// strictNullChecks on (tốt)
let s2: string | null = null;
s2.toUpperCase(); // lỗi compile - phải narrow
if (s2 !== null) s2.toUpperCase(); // ok
// hoặc
s2?.toUpperCase();
s2 ?? "default";

// noImplicitAny
function foo(x) {} // lỗi - phải (x: any) hoặc (x: string)

// strictFunctionTypes
type Fn = (x: string | number) => void;
const f: Fn = (x: string) => {}; // lỗi khi strictFunctionTypes on (contravariant)
```

Trong team Senior, luôn bật `strict: true`, chấp nhận fix nhiều lỗi ban đầu để đổi lại an toàn lâu dài. `strictNullChecks` bắt 30-40% bug null/undefined.

**Trade-off:** Bật strict làm migration đau, phải thêm `| null`, `!` assertion. Nhưng lợi ích vượt trội.

**Lỗi thường gặp:** Dùng `!` (non-null assertion) vô tội vạ để silence, thay vì narrow đúng; tắt strict để cho nhanh.

**Câu hỏi đào sâu:** Vì sao `strictNullChecks` quan trọng nhất? `!` assertion khi nào chấp nhận được?

---

### Câu 26: Enum vs Union Type vs const assertion

**Trả lời Senior:**
Cả ba đều biểu diễn tập giá trị hữu hạn, nhưng trade-off khác:

- **Enum:** runtime object, có reverse mapping với number enum, cho phép `enum E { A, B }` -> `E.A` và `E[E.A] === "A"`. Tăng bundle size, không tree-shakable tốt, và numeric enum cho phép `E[999]` không an toàn.
- **Union:** `type Status = "pending" | "success" | "error"` - chỉ compile-time, không runtime, tree-shakable, autocomplete tốt, là lựa chọn mặc định hiện đại.
- **const assertion:** `const STATUS = { Pending: "pending", Success: "success" } as const` -> tạo object readonly + union từ `typeof STATUS[keyof typeof STATUS]`. Có cả runtime object lẫn type.

```typescript
// Enum - tránh nếu có thể
enum Direction { Up, Down, Left, Right } // 0,1,2,3 + reverse mapping
enum StatusStr { Pending = "PENDING", Success = "SUCCESS" } // string enum không reverse

// Union - khuyến nghị
type Status = "pending" | "success" | "error";
function handle(s: Status) {}
handle("pending"); // ok
// handle("other"); // lỗi

// const assertion - best of both worlds
const StatusConst = {
  Pending: "pending",
  Success: "success",
  Error: "error",
} as const;
type StatusFromConst = typeof StatusConst[keyof typeof StatusConst]; // "pending" | "success" | "error"
// Runtime: StatusConst.Pending === "pending"
// Type: StatusFromConst

// So sánh bundle
// enum emit JS object, union emit nothing, const assertion emit object nhưng tree-shakable hơn enum
```

**Trade-off:** Enum tiện khi cần reverse mapping hoặc `const enum` (inline, không runtime nhưng không cross-file tốt). Union đơn giản nhất. `as const` object là pattern Senior hay dùng.

**Lỗi thường gặp:** Numeric enum cho phép giá trị ngoài enum, `const enum` bị xóa khi isolatedModules, so sánh enum với string literal sai.

**Câu hỏi đào sâu:** `const enum` khác `enum` thường thế nào? Vì sao community khuyên tránh enum?

---

### Câu 27: Decorator và Metadata

**Trả lời Senior:**
Decorator là hàm intercept class/method/property/parameter, syntax `@decorator`. Proposal stage 3, TS hỗ trợ qua `experimentalDecorators`. Có 2 thế hệ: legacy (TS) và TC39 new (ES2022+). Dùng nhiều trong NestJS, Angular, TypeORM.

```typescript
// Legacy decorator (TS)
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`Call ${key}`, args);
    return original.apply(this, args);
  };
}
class Service {
  @Log
  save(data: string) { console.log("saving", data); }
}

// Class decorator
function Injectable(target: Function) {
  // đăng ký vào container
  Reflect.defineMetadata("injectable", true, target);
}

// New decorator (TS 5+ with --experimentalDecorators false, dùng ES decorator)
// function logged(value, context) {
//   if (context.kind === "method") return function(...args) { ... }
// }

// Metadata với reflect-metadata
import "reflect-metadata";
function Required(target: any, key: string) {
  const type = Reflect.getMetadata("design:type", target, key);
  console.log(type); // String, Number...
}
class User {
  @Required
  name: string;
}
```

Decorator thực chất là higher-order function, thứ tự: `@f @g class C` -> `f(g(C))`.

**Trade-off:** Decorator làm code magic, khó trace, và legacy vs new spec khác nhau gây nhầm. Chỉ dùng khi framework yêu cầu, không lạm dụng cho business logic.

**Lỗi thường gặp:** Quên enable `experimentalDecorators` + `emitDecoratorMetadata`, nhầm thứ tự decorator, dùng decorator trên arrow property (không hoạt động như method).

**Câu hỏi đào sâu:** Legacy vs new decorator khác gì? `reflect-metadata` hoạt động thế nào?

---

### Câu 28: Type Narrowing, Type Guard và Assertion Function

**Trả lời Senior:**
Narrowing là thu hẹp type từ rộng sang hẹp qua control flow. Type guard là hàm trả `param is Type` để TS narrow. Assertion function (`asserts`) ném lỗi nếu không thỏa, giúp narrow mà không cần `if`.

```typescript
// Narrowing cơ bản
function padLeft(x: string | number, padding: number) {
  if (typeof x === "string") return " ".repeat(padding) + x; // x là string
  return " ".repeat(padding) + x.toString(); // x là number
}

// Custom type guard
type Cat = { meow(): void };
type Dog = { bark(): void };
function isCat(pet: Cat | Dog): pet is Cat {
  return (pet as Cat).meow !== undefined;
}
function speak(pet: Cat | Dog) {
  if (isCat(pet)) pet.meow();
  else pet.bark(); // TS biết là Dog
}

// Assertion function
function assertIsString(x: unknown): asserts x is string {
  if (typeof x !== "string") throw new Error("Not a string");
}
function foo(x: unknown) {
  assertIsString(x);
  x.toUpperCase(); // ok, đã assert
}

// in operator narrowing
type A = { type: "a"; a: string };
type B = { type: "b"; b: number };
function handle(x: A | B) {
  if ("a" in x) console.log(x.a); // A
  else console.log(x.b); // B
}

// Discriminated union - pattern mạnh nhất
type Shape = { kind: "circle"; radius: number } | { kind: "rect"; w: number; h: number };
function area(s: Shape) {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;
    case "rect": return s.w * s.h;
  }
}
```

**Trade-off:** `as` cast là unsafe, nên dùng guard. Assertion function ném runtime, nên chỉ dùng khi muốn fail fast.

**Lỗi thường gặp:** Viết guard trả `boolean` thay vì `is Type`, dùng `as` thay vì narrow, quên discriminated union là cách narrow tốt nhất.

**Câu hỏi đào sâu:** `is` vs `asserts` khác gì? Làm sao viết guard cho array `filter` để loại `null`?

---

### Câu 29: Declaration Merging và Module Augmentation

**Trả lời Senior:**
Declaration merging là TS merge nhiều declaration cùng tên (interface với interface, namespace với class...). Module augmentation là dùng merging để **mở rộng type của lib bên ngoài** - rất Senior khi cần patch type thiếu.

```typescript
// Merging interface
interface User { name: string; }
interface User { age: number; }
const u: User = { name: "An", age: 20 }; // có cả 2

// Merging namespace + class
class Album { label: string = ""; }
namespace Album { export class Label {} } // Album.Label

// Module augmentation - mở rộng Window, Express Request, MUI Theme
// global.d.ts
declare global {
  interface Window { myApp: { version: string }; }
}
window.myApp.version; // ok

// Augment lib: thêm property vào Express Request
// types/express.d.ts
import "express";
declare module "express-serve-static-core" {
  interface Request { user?: { id: string; role: string }; }
}
// dùng: req.user?.id

// Augment MUI
declare module "@mui/material/styles" {
  interface Palette { neutral: Palette["primary"]; }
}

// Ambient module cho JS lib không có type
declare module "my-js-lib" {
  export function doThing(x: string): number;
}
```

Cần `declare module "xxx"` khớp với module name, và file phải là module (có `import`/`export`).

**Trade-off:** Augmentation là global, ảnh hưởng toàn project, nên đặt trong `types/` riêng và document rõ. Không augment bừa để che bug của lib.

**Lỗi thường gặp:** File augmentation không có import nên thành script (global) không hoạt động, augment sai module name, quên `declare global` cho global.

**Câu hỏi đào sâu:** `declare global` vs `declare module` khác gì? Làm sao augment type của third-party mà không sửa node_modules?

---

### Câu 30: Tối ưu hiệu năng TypeScript trong Monorepo lớn

**Trả lời Senior:**
Monorepo lớn (thousands files) type checking chậm do `tsc` single-thread, type phức tạp, và project references không tối ưu. Chiến lược:

1.  **Project References + composite:** chia thành nhiều `tsconfig.json` con, `tsc -b` build incremental, chỉ check file thay đổi. Bật `incremental: true`, `tsBuildInfoFile`.
2.  **SkipLibCheck:** `skipLibCheck: true` bỏ check `.d.ts` của node_modules (an toàn vì lib đã check).
3.  **IsolatedModules + esbuild/swc:** dùng `esbuild` để transpile nhanh, chỉ chạy `tsc --noEmit` trong CI.
4.  **Tránh type phức tạp:** limit conditional recursion, tránh `Omit` lồng nhau trên union lớn, dùng `interface` thay `type` intersection lớn.
5.  **Tools:** `tsc --extendedDiagnostics` để xem time, `typescript-analyze-trace`, `ts-prune` tìm unused export.

```json
// tsconfig.json root
{
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true,
    "incremental": true,
    "composite": true,
    "declaration": true,
    "declarationMap": true
  },
  "references": [{ "path": "./packages/app" }, { "path": "./packages/ui" }]
}
```

```bash
# Build nhanh
tsc -b --verbose # build references
tsc --noEmit --incremental # check only

# Debug performance
npx tsc --extendedDiagnostics
npx tsc --generateTrace trace && npx analyze-trace trace
```

Trong CI, cache `.tsbuildinfo`, dùng `nx` hoặc `turborepo` để remote cache. Với Yarn PnP, cấu hình `pnpEnableEsmLoader`.

**Trade-off:** `skipLibCheck` có thể che lỗi type trong `.d.ts` sai, `isolatedModules` cấm `const enum` cross-file. Project References thêm complexity config.

**Lỗi thường gặp:** Không bật `composite` nên references không hoạt động, type utility đệ quy quá sâu làm OOM, import barrel `index.ts` re-export hết làm check chậm.

**Câu hỏi đào sâu:** `tsc -b` khác `tsc` thường thế nào? Làm sao phát hiện type nào làm chậm nhất?

---
