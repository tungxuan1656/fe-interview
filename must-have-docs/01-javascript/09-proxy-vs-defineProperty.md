# Proxy vs Object.defineProperty — Intercept toàn bộ object và reactivity

> Tags: #proxy #defineProperty #reflect #reactivity | Nguồn: `docs/01-javascript-typescript.md` câu 16 | Mức: P1

## 1. Định nghĩa chính xác

`Object.defineProperty(obj, prop, descriptor)` định nghĩa **từng property** với getter/setter (`get`/`set`) hoặc descriptor (`value`, `writable`, `enumerable`, `configurable`). Nó chỉ intercept property đã khai báo, không bắt được thêm/xóa key mới hay index array mới.

`Proxy(target, handler)` wrap **toàn bộ object**, intercept mọi thao tác qua 13 trap (`get`, `set`, `has`, `deleteProperty`, `ownKeys`, `getOwnPropertyDescriptor`, `defineProperty`, `preventExtensions`, `getPrototypeOf`, `setPrototypeOf`, `isExtensible`, `apply`, `construct`). Kết hợp `Reflect` để forward thao tác với semantics đúng (đặc biệt `receiver`/`this`).

## 2. Cơ chế hoạt động

### 2.1 `defineProperty` — per-property

```js
Object.defineProperty(obj, 'x', {
  get(){ return this._x },
  set(v){ trigger(); this._x = v },
  enumerable: true, configurable: true
});
```

- Phải duyệt `Object.keys(obj)` để define từng key.
- Không intercept `obj.newProp = 1` (chưa define) hay `arr[3] = x` vượt length, `delete`.
- Array phải override method (`push`/`splice`) mới bắt được.

### 2.2 `Proxy` — per-object

```js
const p = new Proxy(target, {
  get(t, prop, receiver){ return Reflect.get(t, prop, receiver); },
  set(t, prop, value, receiver){ Reflect.set(t, prop, value, receiver); return true; }
});
```

- `receiver` là proxy (hoặc object kế thừa), quan trọng để `this` trong getter/setter trỏ đúng.
- Proxy là object khác (`p !== target`), có overhead nhưng intercept động, `Map`/`Set` cũng được (với handler riêng cho `get` của method).
- Phải **recursively proxy** để deep reactivity: `get` trap nếu value là object thì `return reactive(value)`.

### 2.3 Vue 2 vs Vue 3

- **Vue 2**: `Object.defineProperty` + `observe` duyệt data, `arrayMethods` patch. Hạn chế: `vm.items[0] = x` không reactive (phải `Vue.set`), `vm.obj.newKey = 1` không reactive.
- **Vue 3**: `Proxy` + `Reflect` + `WeakMap` cache (`reactiveMap`). Fix hết hạn chế, hỗ trợ `Map`/`Set`, `has`/`delete`, array index.

## 3. Ví dụ tối thiểu

```js
// 3.1 defineProperty — hạn chế
const obj1 = { _x: 1 };
Object.defineProperty(obj1, 'x', {
  get() { console.log('get x'); return this._x; },
  set(v) { console.log('set x', v); this._x = v; },
  enumerable: true, configurable: true
});
console.log(obj1.x); // get x → 1
obj1.x = 2; // set x 2
obj1.y = 99; // không log — y chưa define
console.log(obj1.y); // 99 — không reactive

// Array defineProperty không bắt index mới
const arr1 = [1, 2];
Object.defineProperty(arr1, '0', { get(){return 1}, set(v){console.log(v)} });
// arr1[2] = 3 — không trap

// 3.2 Proxy — toàn diện
function reactive(target) {
  return new Proxy(target, {
    get(t, p, receiver) {
      console.log('get', String(p));
      const res = Reflect.get(t, p, receiver);
      // deep proxy
      return (res !== null && typeof res === 'object') ? reactive(res) : res;
    },
    set(t, p, v, receiver) {
      console.log('set', String(p), v);
      return Reflect.set(t, p, v, receiver);
    },
    deleteProperty(t, p) {
      console.log('delete', String(p));
      return Reflect.deleteProperty(t, p);
    },
    has(t, p) {
      console.log('has', String(p));
      return Reflect.has(t, p);
    },
    ownKeys(t) {
      console.log('ownKeys');
      return Reflect.ownKeys(t);
    }
  });
}
const state = reactive({ count: 0, nested: { a: 1 } });
state.count++; // get count → set count 1
state.newProp = 1; // set newProp 1 — Proxy bắt được!
console.log('newProp' in state); // has newProp → true
delete state.newProp; // delete newProp
console.log(state.nested.a); // get nested → get a — deep proxy

// 3.3 Reflect quan trọng — giữ this đúng
const parent = {
  _x: 1,
  get x() { return this._x; }
};
const child = reactive(Object.create(parent));
console.log(child.x); // get x — nếu không dùng Reflect.get với receiver, this sẽ là target thay vì child

// 3.4 Proxy với Map/Set — cần bind this
function reactiveMap(map) {
  return new Proxy(map, {
    get(t, p, receiver) {
      if (p === 'get') return function(key){ console.log('map get', key); return t.get(key); };
      if (p === 'set') return function(k,v){ console.log('map set', k, v); return t.set(k,v); };
      return Reflect.get(t, p, receiver);
    }
  });
}
const m = reactiveMap(new Map());
m.set('k', 1);
console.log(m.get('k')); // 1

// 3.5 So sánh Vue2 vs Vue3 effect
// Vue2: Object.defineProperty — phải Vue.set(obj, 'newKey', 1)
// Vue3: Proxy — obj.newKey = 1 tự reactive

// 3.6 Proxy !== target — so sánh reference
const target = { a: 1 };
const proxy = new Proxy(target, {});
console.log(proxy === target); // false
console.log(proxy.a === target.a); // true — nhưng proxy !== target nên Map key khác
const map = new Map();
map.set(target, 1);
console.log(map.get(proxy)); // undefined — key khác!
```

```js
// 3.7 Revocable Proxy
const { proxy: revProxy, revoke } = Proxy.revocable({ a: 1 }, {
  get(t,p,r){ return Reflect.get(t,p,r); }
});
console.log(revProxy.a); // 1
revoke();
try { console.log(revProxy.a); } catch(e){ console.log(e.name); } // TypeError: revoked

// 3.8 Invariant — trap phải trả đúng
const p2 = new Proxy({ x: 1 }, {
  get(t,p){ return 99; },
  getOwnPropertyDescriptor(t,p){ return undefined; } // vi phạm invariant nếu t có non-configurable prop
});
// Engine sẽ ném TypeError nếu handler vi phạm invariant
```

## 4. So sánh / Phân loại

| Tiêu chí | `Object.defineProperty` | `Proxy` |
|----------|-------------------------|---------|
| Phạm vi | Từng property đã biết | Toàn bộ object, mọi key động |
| Thêm/xóa property | Không bắt được | `set`/`deleteProperty` bắt được |
| Array index mới | Không | Có |
| `Map`/`Set` | Không | Có (với handler riêng) |
| Số trap | 1 (get/set per prop) | 13 |
| `Reflect` | Không cần | Nên dùng để giữ semantics |
| Polyfill IE11 | Có | Không |
| Performance (object nhỏ) | Nhẹ hơn | Overhead cao hơn (proxy object) |
| So sánh reference | `obj` chính nó | `proxy !== target` |

| Trap Proxy | Intercept thao tác |
|------------|-------------------|
| `get` | `obj.prop`, `obj['prop']` |
| `set` | `obj.prop = v` |
| `has` | `'prop' in obj` |
| `deleteProperty` | `delete obj.prop` |
| `ownKeys` | `Object.keys`, `for...in`, `Object.getOwnPropertyNames` |
| `getOwnPropertyDescriptor` | `Object.getOwnPropertyDescriptor` |
| `defineProperty` | `Object.defineProperty` |
| `apply` / `construct` | `proxy(...args)` / `new proxy()` (khi target là function/class) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng Proxy khi cần IE11 hoặc polyfill**: Proxy không thể polyfill đầy đủ, phải fallback `defineProperty`.
- **Không dùng Proxy cho object nhỏ, ít key, không cần dynamic**: `defineProperty` nhẹ hơn, đủ dùng, không tạo extra object.
- **Không quên `proxy !== target`**: so sánh `===`, `Map` key, `WeakMap` key sẽ sai nếu lẫn lộn. Lưu `proxy` và dùng nhất quán.
- **Không Proxy mà không deep**: `reactive({ nested:{a:1} })` với shallow proxy thì `state.nested.a = 2` không trap. Phải recursive trong `get`.
- **Khi nào dùng Proxy**: reactivity system (Vue, MobX), validation, logging, virtual object, `private` via trap.
- **Khi nào dùng `defineProperty`**: định nghĩa `readonly`, `non-enumerable`, `non-writable` cho API public, hoặc môi trường không có Proxy.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `proxy === target` sai**
  - Triệu chứng: `map.get(proxy)` miss, `obj === reactive(obj)` false.
  - Fix: dùng `WeakMap` cache `target→proxy` (như Vue `reactiveMap`), không so sánh lẫn lộn.
  - Đo: `console.log(proxy === target)`, test Map.

- **Lỗi 2: Quên `Reflect` làm `this` sai**
  - Triệu chứng: getter `get x(){ return this._x }` trả sai khi `this` là target.
  - Fix: `Reflect.get(t, p, receiver)` thay vì `t[p]`.
  - Đo: log `this` trong getter.

- **Lỗi 3: Không deep proxy**
  - Triệu chứng: `state.nested.a = 2` không trigger.
  - Fix: `get` trap check `typeof res === 'object' && res !== null` thì `return reactive(res)`.
  - Đo: test nested set.

- **Lỗi 4: Proxy invariant violation**
  - Triệu chứng: `TypeError: 'getOwnPropertyDescriptor' on proxy: trap reported non-configurability...`.
  - Fix: trap phải trả consistent với target descriptor (nếu target có non-configurable prop thì trap không thể báo khác).
  - Đo: DevTools console error.

- **Lỗi 5: Performance overhead**
  - Triệu chứng: tạo quá nhiều proxy cho large dataset chậm.
  - Fix: shallow proxy (`shallowReactive`), hoặc chỉ proxy root cần thiết.
  - Đo: `performance.now()`, benchmark `for` loop với proxy vs raw.

- **Công cụ**: `Proxy.revocable`, `Reflect`, `util.types.isProxy` (Node), DevTools → `console.dir(proxy)`.

## 7. Câu hỏi tự kiểm tra

1. Vì sao `Object.defineProperty` không reactive với `obj.newProp = 1` hay `arr[3] = x`, còn `Proxy` thì được?
2. Vì sao trong Proxy handler phải dùng `Reflect.get(t, p, receiver)` thay vì `t[p]`?
3. `proxy !== target` gây bug gì với `Map`/`Set`/`===`, và Vue 3 khắc phục thế nào?

<details>
<summary>Đáp án 30s</summary>

1. `defineProperty` chỉ define getter/setter cho key đã biết tại thời điểm duyệt; key mới chưa có descriptor nên gán thẳng không qua trap. Proxy intercept ở level object: mọi `set` dù key mới hay index mới đều qua `set` trap, vì proxy wrap toàn bộ target.
2. Để giữ `this` (receiver) đúng: getter `get x(){ return this._x }` khi truy cập `proxy.x` thì `this` phải là `proxy` (receiver), không phải `target`. `Reflect.get(t,p,receiver)` truyền receiver làm `this` cho getter; `t[p]` thì `this` là `target`, sai với proxy chain và inherited accessor.
3. `proxy !== target` nên `map.get(proxy)` khác `map.get(target)`, `===` fail, `WeakMap` miss. Vue 3 dùng `WeakMap` cache `target→proxy` và `proxy→target` (reactiveMap, rawMap), và trong trap check cả hai để lookup đúng; đồng thời `reactive(obj) === reactive(obj)` nhờ cache.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 16. Spec: [ECMA-262 §28 Proxy](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots), [MDN Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy), [MDN Reflect](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect).*
