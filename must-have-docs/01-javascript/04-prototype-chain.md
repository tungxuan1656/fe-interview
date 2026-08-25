# Prototype Chain — [[Prototype]], kế thừa ủy quyền và new

> Tags: #prototype #prototype-chain #inheritance #instanceof | Nguồn: `docs/01-javascript-typescript.md` câu 4 | Mức: P0

## 1. Định nghĩa chính xác

Mỗi object có một internal slot `[[Prototype]]` (có thể là object khác hoặc `null`) tạo thành **Prototype Chain**. Khi truy cập property `obj.prop`, engine thực hiện `[[Get]]`: tìm trên `obj` trước, nếu không có thì đi theo `[[Prototype]]` lên chain cho tới khi tìm thấy hoặc tới `null`. **Kế thừa trong JS là ủy quyền (delegation), không phải sao chép**.

`function` object có thêm property `.prototype` — object sẽ được gán làm `[[Prototype]]` cho instance khi gọi `new`. `class` là syntactic sugar cho pattern `constructor` + `prototype`.

## 2. Cơ chế hoạt động

### 2.1 `prototype` vs `__proto__` vs `[[Prototype]]`

- `[[Prototype]]`: internal slot ẩn, là link thực sự của chain. Truy cập chuẩn qua `Object.getPrototypeOf(obj)` / `Object.setPrototypeOf(obj, proto)` / `Object.create(proto)`.
- `__proto__`: accessor legacy (getter/setter) trên `Object.prototype`, tương đương `getPrototypeOf/setPrototypeOf`. Không nên dùng trong code mới (chậm, deprecated).
- `.prototype`: chỉ tồn tại trên **function** (và class). Là template cho `[[Prototype]]` của instance khi `new`. `obj.prototype` với object thường là `undefined`.

```
function Foo() {}
Foo.prototype  ──→ { constructor: Foo, __proto__: Object.prototype }
                     ↑
inst.[[Prototype]] ──┘   (khi const inst = new Foo())
inst.__proto__ === Foo.prototype === Object.getPrototypeOf(inst)
```

### 2.2 Các bước `new`

Khi `new Foo(...args)`:

1. Tạo object mới `instance` với `[[Prototype]] = Foo.prototype`.
2. Gọi `Foo` với `this = instance`, `[[Call]]` với args.
3. Nếu `Foo` trả về object (không phải `null` và `typeof === 'object'` hoặc `function`) thì trả object đó; ngược lại trả `instance`.

### 2.3 Property lookup và shadowing

- `hasOwnProperty`: chỉ check own, không đi chain.
- `in`: check cả chain (`'toString' in {}` → `true`).
- Gán `obj.prop = v` luôn tạo own property (shadow), không ghi lên prototype, trừ khi có setter trên prototype.
- `Object.create(null)` tạo object không có `[[Prototype]]` → không có `hasOwnProperty`, `toString`, dùng làm dictionary thuần túy.

### 2.4 `instanceof`

`a instanceof B` tương đương `B.prototype` có nằm trên chain của `a` không (đi qua `[[Prototype]]` liên tiếp, so sánh `===`). Có thể custom bằng `Symbol.hasInstance`.

## 3. Ví dụ tối thiểu

```js
// 3.1 Prototype chain cơ bản
function Animal(name) { this.name = name; }
Animal.prototype.speak = function() { return `${this.name} kêu`; };

function Dog(name) { Animal.call(this, name); } // mượn constructor
Object.setPrototypeOf(Dog.prototype, Animal.prototype); // kế thừa — thay vì Dog.prototype = ...
Dog.prototype.bark = function() { return 'Gâu'; };

const d = new Dog('Milu');
console.log(d.speak()); // 'Milu kêu' — tìm trên Dog.prototype → không có → lên Animal.prototype → có
console.log(Object.getPrototypeOf(d) === Dog.prototype); // true
console.log(d instanceof Dog); // true
console.log(d instanceof Animal); // true — Animal.prototype nằm trên chain
console.log(d instanceof Object); // true

// 3.2 Sai lầm kinh điển: gán trực tiếp
function BadDog() {}
BadDog.prototype = Animal.prototype; // sai — share cùng object, sửa BadDog ảnh hưởng Animal
BadDog.prototype.constructor === Animal; // true — mất constructor

// Đúng: tạo object mới kế thừa
function GoodDog() {}
GoodDog.prototype = Object.create(Animal.prototype);
GoodDog.prototype.constructor = GoodDog; // khôi phục

// 3.3 Class sugar — tương đương GoodDog pattern
class Animal2 {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} kêu`; }
}
class Dog2 extends Animal2 {
  bark() { return 'Gâu'; }
}
console.log(typeof Animal2); // 'function'
console.log(Object.getOwnPropertyNames(Animal2.prototype)); // ['constructor','speak']

// 3.4 Object.create(null) — dictionary thuần
const dict = Object.create(null);
dict['key'] = 1;
console.log(dict.hasOwnProperty); // undefined — không có prototype
console.log('toString' in dict); // false
console.log(Object.getPrototypeOf(dict)); // null

const normal = {};
console.log(normal.hasOwnProperty); // function
console.log('toString' in normal); // true

// 3.5 Shadowing và setter
const proto = {
  _x: 1,
  get x() { return this._x; },
  set x(v) { this._x = v; }
};
const obj = Object.create(proto);
console.log(obj.x); // 1 — lấy từ proto
obj.x = 99; // gọi setter trên proto, nhưng this là obj → tạo obj._x = 99 (own)
console.log(obj.hasOwnProperty('_x')); // true
console.log(proto._x); // 1 — không bị ghi đè

// 3.6 instanceof custom
class MyArray {
  static [Symbol.hasInstance](instance) {
    return Array.isArray(instance);
  }
}
console.log([] instanceof MyArray); // true

// 3.7 Kiểm tra own vs chain
console.log(d.hasOwnProperty('name')); // true — own
console.log(d.hasOwnProperty('speak')); // false — trên prototype
console.log('speak' in d); // true — in đi cả chain
console.log(Object.hasOwn(d, 'name')); // true — cách hiện đại (ES2022)
```

```js
// 3.8 new steps minh họa
function FakeNew(Constructor, ...args) {
  const inst = Object.create(Constructor.prototype);
  const ret = Constructor.apply(inst, args);
  return (ret !== null && (typeof ret === 'object' || typeof ret === 'function')) ? ret : inst;
}
function Person(name) {
  this.name = name;
  // return { hacked: true }; // nếu return object thì FakeNew trả object này
}
console.log(FakeNew(Person, 'An')); // Person { name: 'An' }
```

## 4. So sánh / Phân loại

| Truy cập | `prototype` | `__proto__` | `[[Prototype]]` |
|----------|-------------|-------------|-----------------|
| Tồn tại trên | `function` / `class` | mọi object (accessor legacy) | mọi object (internal slot) |
| Vai trò | Template cho instance khi `new` | Getter/setter cho `[[Prototype]]` | Link thực sự của chain |
| Chuẩn | Có, spec | Deprecated, không nên dùng | Internal, truy cập qua API |
| Thay thế | — | `Object.getPrototypeOf/setPrototypeOf` | `Object.getPrototypeOf/setPrototypeOf/create` |

| Cách tạo object | `[[Prototype]]` là gì | Dùng khi |
|-----------------|------------------------|----------|
| `{}` / `new Object()` | `Object.prototype` | object thường |
| `Object.create(proto)` | `proto` | kế thừa không cần constructor, hoặc `null` |
| `Object.create(null)` | `null` | dictionary, map thuần, tránh collision `__proto__` |
| `new Foo()` | `Foo.prototype` | instance của constructor/class |
| `class Bar extends Foo` | `Foo.prototype` (và `Bar.__proto__ === Foo`) | kế thừa class |

| Kiểm tra property | Chỉ own? | Đi chain? | Ví dụ |
|-------------------|----------|-----------|-------|
| `obj.hasOwnProperty('x')` / `Object.hasOwn(obj,'x')` | Có | Không | `hasOwnProperty` |
| `'x' in obj` | Không | Có | `'toString' in {}` → true |
| `obj.propertyIsEnumerable('x')` | Có | Không, và phải enumerable | — |
| `Object.getOwnPropertyDescriptor(obj,'x')` | Có | Không | — |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không chỉnh `__proto__` trực tiếp**: `obj.__proto__ = newProto` chậm (de-optimize), làm invalidate hidden class của engine. Dùng `Object.create` hoặc `Object.setPrototypeOf` (cũng chậm nhưng rõ ràng, chỉ dùng khi cần).
- **Ưu tiên composition over inheritance**: prototype chain sâu làm lookup chậm và khó suy luận. Thay vì `class A extends B extends C`, dùng composition: `function createDog({ animal }) { return { ...animal, bark(){} } }`.
- **Không gán `Child.prototype = Parent.prototype`**: share cùng object, sửa child ảnh hưởng parent. Luôn `Object.create`.
- **Không dùng arrow function làm method trên prototype**: `Foo.prototype.method = () => this` → `this` lexical, không phải instance.
- **Khi nào dùng**: framework/library cần kế thừa, polyfill, hoặc cần `instanceof` check. Với app code, `class` đơn giản 1 level hoặc factory function thường đủ.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Quên khôi phục `constructor` sau `Object.create`**
  - Triệu chứng: `instance.constructor === Parent` thay vì `Child`, `instanceof` vẫn đúng nhưng `constructor` sai.
  - Fix: `Child.prototype.constructor = Child`.
  - Đo: `console.log(Child.prototype.constructor.name)`.

- **Lỗi 2: Method arrow trên prototype mất `this`**
  - Triệu chứng: `this` là `window/undefined` thay vì instance.
  - Fix: dùng `Foo.prototype.method = function(){}` hoặc class method `method(){}`.
  - Đo: breakpoint trong method → Scope → `this`.

- **Lỗi 3: Prototype pollution**
  - Triệu chứng: `JSON.parse('{"__proto__":{"polluted":true}}')` + gán recursive làm `Object.prototype.polluted` xuất hiện.
  - Fix: dùng `Object.create(null)` cho dictionary, validate key `__proto__`, dùng `Map` thay object cho user input.
  - Đo: `Object.prototype.hasOwnProperty('polluted')`, SAST scan.

- **Lỗi 4: `Object.create(null)` thiếu method**
  - Triệu chứng: `dict.hasOwnProperty` là `undefined`, gọi ném `TypeError`.
  - Fix: `Object.hasOwn(dict, key)` hoặc `Object.prototype.hasOwnProperty.call(dict, key)`.
  - Đo: test với `null` prototype.

- **Lỗi 5: Chuỗi prototype quá sâu**
  - Triệu chứng: property lookup chậm, `instanceof` chậm.
  - Fix: giữ chain nông (1-2 level), dùng composition.
  - Đo: DevTools → Performance → Bottom-up, `console.time` lookup trong loop.

- **Công cụ**:
  - `Object.getPrototypeOf(obj)`, `obj instanceof Ctor`, `Object.getOwnPropertyDescriptors`
  - DevTools → Console: `console.dir(obj)` hiển thị `[[Prototype]]`
  - ESLint: `no-proto`

## 7. Câu hỏi tự kiểm tra

1. Phân biệt `Foo.prototype` và `foo.__proto__` (hay `[[Prototype]]` của `foo`)? Khi nào chúng bằng nhau?
2. Mô tả chính xác 3 bước khi `new Foo()` được thực thi, và khi nào `new` không trả về object mới tạo?
3. `Object.create(null)` khác `{}` thế nào, và vì sao nó hữu ích làm dictionary?

<details>
<summary>Đáp án 30s</summary>

1. `Foo.prototype` là property trên **function** `Foo`, dùng làm template cho `[[Prototype]]` của instance. `foo.__proto__` (tức `[[Prototype]]` của `foo`) là link chain thực sự của instance. Chúng bằng nhau khi `foo = new Foo()` → `Object.getPrototypeOf(foo) === Foo.prototype`. `__proto__` tồn tại trên mọi object, `.prototype` chỉ trên function.
2. (1) Tạo object mới với `[[Prototype]] = Foo.prototype`. (2) Gọi `Foo` với `this` là object mới và args. (3) Nếu `Foo` return object (không null, typeof object/function) thì trả object đó, ngược lại trả object mới tạo. Ví dụ `function Foo(){ return {a:1}; } new Foo()` trả `{a:1}` thay vì `this`.
3. `{}` có `[[Prototype]] === Object.prototype` nên có sẵn `hasOwnProperty`, `toString`, và `'__proto__'` có thể gây pollution. `Object.create(null)` có `[[Prototype]] === null`, không có gì trên chain, là dictionary thuần túy, an toàn hơn cho key từ user input; truy cập `hasOwnProperty` phải qua `Object.hasOwn` hoặc `call`.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 4. Spec: [ECMA-262 §10.1 Ordinary Object Internal Methods](https://tc39.es/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots), [§20.1 Object](https://tc39.es/ecma262/#sec-object-objects).*
