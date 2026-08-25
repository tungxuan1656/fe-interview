# Shallow Copy và Deep Copy — Sao chép nông/sâu và immutability

> Tags: #shallow-copy #deep-copy #structuredClone #immutability | Nguồn: `docs/01-javascript-typescript.md` câu 10 | Mức: P0

## 1. Định nghĩa chính xác

**Shallow copy** chỉ sao chép một cấp (own enumerable properties), các nested object vẫn **share reference** với bản gốc. **Deep copy** sao chép đệ quy toàn bộ cây object, tạo object mới hoàn toàn, không share reference ở bất kỳ cấp nào.

Trong JS, `...spread` và `Object.assign` là shallow. `structuredClone` là deep copy chuẩn của platform, xử lý `Date`, `RegExp`, `Map`, `Set`, `ArrayBuffer`, circular reference, nhưng không clone `function`, DOM node, `Error` stack tuỳ case, và non-cloneable. `JSON.parse(JSON.stringify(obj))` là deep nhưng mất `undefined`, `function`, `Symbol`, `Date`→string, `Infinity/NaN`→`null`, và ném với circular.

## 2. Cơ chế hoạt động

### 2.1 Shallow

```js
const clone = { ...obj } // hoặc Object.assign({}, obj)
```

Engine tạo object mới, `[[DefineOwnProperty]]` cho mỗi own enumerable key của `obj`, với value là **cùng reference** nếu value là object.

### 2.2 Deep

- `structuredClone(obj, { transfer })` — structured clone algorithm (HTML spec), hỗ trợ circular, `Map`/`Set`, `Date`, `RegExp`, `ArrayBuffer` (có thể transfer), `Error` (một phần). Không clone `function`, `WeakMap`, DOM.
- `JSON` trick — `JSON.stringify` chỉ serialize `string/number/boolean/null/array/object` với `undefined/function/Symbol` bị bỏ, `Date`→ISO string.
- Thủ công với `WeakMap` để track `seen` tránh circular infinite loop.

### 2.3 Immutability trong Redux/React

React/Redux dùng **shallow equality** (`===` hoặc `Object.is`) để phát hiện thay đổi. Chỉ cần shallow copy **path** bị thay đổi, các nhánh không đổi giữ reference cũ để `memo`/`selector` skip. Deep clone toàn bộ state mỗi lần là lãng phí và phá memoization.

```
state = { user: { name:'An', loc:{ city:'HN' } }, posts: [...] }
newState = { ...state, user: { ...state.user, loc: { ...state.user.loc, city:'HCM' } } }
// chỉ 3 object mới: newState, newState.user, newState.user.loc — posts giữ reference
```

Immer làm việc này tự động bằng Proxy + copy-on-write.

## 3. Ví dụ tối thiểu

```js
// 3.1 Shallow — nested vẫn share
const a = { x: 1, nested: { y: 2 } };
const shallow = { ...a };
shallow.nested.y = 99;
console.log(a.nested.y); // 99 — bị mutate!
shallow.x = 2;
console.log(a.x); // 1 — primitive thì không share

console.log(shallow.nested === a.nested); // true — cùng reference
console.log(shallow !== a); // true — top-level khác

// Object.assign tương tự shallow
const b = Object.assign({}, a);
console.log(b.nested === a.nested); // true

// 3.2 structuredClone — deep chuẩn
const original = {
  date: new Date(),
  map: new Map([['k', { v: 1 }]]),
  set: new Set([1, 2]),
  re: /hi/g,
  nested: { arr: [1, 2] }
};
original.circular = original; // circular
const deep = structuredClone(original);
console.log(deep !== original); // true
console.log(deep.nested !== original.nested); // true
console.log(deep.map !== original.map); // true
console.log(deep.circular === deep); // true — circular được giữ
console.log(deep.date instanceof Date); // true

try {
  structuredClone({ fn: () => {} }); // ném DataCloneError
} catch (e) { console.log(e.name); } // DataCloneError

// 3.3 JSON trick — hạn chế
const obj = { a: 1, b: undefined, c: () => {}, d: Symbol('s'), e: new Date(), f: Infinity };
console.log(JSON.stringify(obj)); // {"a":1,"e":"2026-01-01T00:00:00.000Z","f":null} — mất b,c,d, Infinity→null
const jsonDeep = JSON.parse(JSON.stringify({ a: 1, nested: { y: 2 } })); // ok cho data JSON thuần
try { JSON.stringify(original); } catch(e){ console.log('circular JSON fail', e.message); } // TypeError

// 3.4 Deep clone thủ công với WeakMap — xử lý circular
function deepClone(obj, seen = new WeakMap()) {
  if (obj === null || typeof obj !== 'object') return obj;
  if (seen.has(obj)) return seen.get(obj);

  // xử lý Date, RegExp, Map, Set
  if (obj instanceof Date) return new Date(obj);
  if (obj instanceof RegExp) return new RegExp(obj.source, obj.flags);
  if (obj instanceof Map) {
    const clone = new Map();
    seen.set(obj, clone);
    for (const [k, v] of obj) clone.set(deepClone(k, seen), deepClone(v, seen));
    return clone;
  }
  if (obj instanceof Set) {
    const clone = new Set();
    seen.set(obj, clone);
    for (const v of obj) clone.add(deepClone(v, seen));
    return clone;
  }

  const clone = Array.isArray(obj) ? [] : {};
  seen.set(obj, clone);
  for (const k in obj) {
    if (Object.hasOwn(obj, k)) clone[k] = deepClone(obj[k], seen);
  }
  // lưu ý: không clone Symbol keys, non-enumerable, getter/setter, prototype
  return clone;
}
const c1 = { x: 1 }; c1.self = c1;
console.log(deepClone(c1).self === deepClone(c1)); // false — mỗi lần clone mới, nhưng circular được giữ trong một lần clone

// 3.5 Redux immutability — chỉ shallow copy path thay đổi
const state = { user: { name: 'An', loc: { city: 'HN' } }, posts: [1, 2] };
const next = {
  ...state,
  user: { ...state.user, loc: { ...state.user.loc, city: 'HCM' } }
};
console.log(next !== state); // true
console.log(next.posts === state.posts); // true — giữ reference để memo
console.log(next.user.loc !== state.user.loc); // true — đã copy

// 3.6 Array shallow
const arr = [{ id: 1 }, { id: 2 }];
const arrClone = [...arr];
console.log(arrClone[0] === arr[0]); // true — shallow
const arrDeep = structuredClone(arr);
console.log(arrDeep[0] === arr[0]); // false
```

```js
// 3.7 structuredClone với transfer (zero-copy cho ArrayBuffer)
const buffer = new ArrayBuffer(8);
const cloned = structuredClone({ buf: buffer }, { transfer: [buffer] });
console.log(buffer.byteLength); // 0 — đã transfer, không còn dùng được
console.log(cloned.buf.byteLength); // 8
```

## 4. So sánh / Phân loại

| Cách | Loại | Circular | `Date/Map/Set/RegExp` | `function/Symbol` | `undefined` | Ghi chú |
|------|------|----------|----------------------|-------------------|-------------|---------|
| `{...obj}` / `Object.assign` | Shallow | Giữ ref (không loop) | Giữ ref | Giữ ref | Giữ | Nhanh, chỉ 1 cấp |
| `structuredClone` | Deep | Có | Có (clone mới) | Ném `DataCloneError` | Giữ (trong object/array) | Chuẩn hiện đại, không clone function/DOM |
| `JSON.parse(JSON.stringify)` | Deep (hạn chế) | Ném `TypeError` | `Date`→string, `Map/Set`→`{}` | Mất | Mất | Chỉ cho JSON-serializable |
| `deepClone` thủ công (WeakMap) | Deep (tùy implement) | Có (nếu dùng WeakMap) | Phải code riêng | Có thể bỏ qua | Tùy | Cần xử lý Symbol, descriptor, prototype |
| `immer` / `produce` | Shallow path (copy-on-write) | — | — | — | — | Tối ưu cho Redux, chỉ copy nhánh đổi |

| Shallow vs Deep — khi nào dùng | Ví dụ |
|-------------------------------|-------|
| Shallow — React state update 1 cấp | `setState({ ...state, count: 1 })` |
| Shallow path — nested update | `{ ...state, user: { ...user, name: 'B' } }` hoặc `produce(state, draft => { draft.user.name='B' })` |
| Deep — clone data để mutate tự do | `structuredClone(template)` trước khi edit |
| Deep — worker message | `postMessage(data)` dùng structured clone |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không deep clone toàn bộ Redux state mỗi lần**: `structuredClone(state)` O(n) với n lớn, tốn memory và phá memoization (mọi selector thấy reference mới). Chỉ shallow copy path thay đổi hoặc dùng `immer`.
- **Không dùng `JSON` deep clone cho data có `Date/Map/undefined/function`**: mất dữ liệu im lặng. Dùng `structuredClone`.
- **Không `structuredClone` cho object chứa `function`/DOM**: ném lỗi. Clone data thuần túy thôi.
- **Không shallow copy khi cần mutate nested tự do**: `const copy = {...obj}; copy.nested.x=1` vẫn mutate gốc. Phải deep hoặc shallow path.
- **Khi nào shallow đủ**: update 1 cấp, hoặc khi bạn chỉ cần reference mới cho top-level để trigger re-render.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Nghĩ spread là deep**
  - Triệu chứng: mutate `copy.nested` làm gốc đổi, bug Redux.
  - Fix: shallow path `...` từng cấp, hoặc `structuredClone`/`immer`.
  - Đo: `expect(next.nested).not.toBe(prev.nested)` trong test, `Object.is` check.

- **Lỗi 2: `JSON` deep clone mất `Date/Map`**
  - Triệu chứng: `Date` thành string, `Map` thành `{}`, `undefined` biến mất.
  - Fix: `structuredClone` hoặc custom clone.
  - Đo: snapshot test, `console.log(typeof cloned.date)`.

- **Lỗi 3: Circular làm `JSON.stringify` ném**
  - Triệu chứng: `TypeError: Converting circular structure to JSON`.
  - Fix: `structuredClone` hoặc `WeakMap` clone.
  - Đo: try/catch, lint rule `no-circular`.

- **Lỗi 4: `structuredClone` ném với function**
  - Triệu chứng: `DataCloneError: function could not be cloned`.
  - Fix: không clone function; tách data và logic.
  - Đo: test với `expect(() => structuredClone({fn})).toThrow()`.

- **Lỗi 5: Mutate nested state trong Redux**
  - Triệu chứng: component không re-render vì reference không đổi, hoặc re-render thừa vì deep clone.
  - Fix: `immer` `produce`, hoặc spread path.
  - Đo: Redux DevTools, `why-did-you-render`, `Object.is` check.

- **Công cụ**: `structuredClone` support check (`typeof structuredClone === 'function'`), `immer` `current/draft`, DevTools Memory để thấy reference share.

## 7. Câu hỏi tự kiểm tra

1. `const b = {...a}` với `a = {x:1, nested:{y:2}}` thì `b.nested === a.nested` là `true` hay `false`? Vì sao?
2. `structuredClone` khác `JSON.parse(JSON.stringify(obj))` thế nào về `Date`, `Map`, `circular`, `function`?
3. Trong Redux, vì sao không `structuredClone` toàn bộ state mỗi lần update, và cách đúng để update `state.user.loc.city` là gì?

<details>
<summary>Đáp án 30s</summary>

1. `true` — spread là shallow, chỉ copy own enumerable 1 cấp; `nested` là object nên value được copy là reference cùng object. Mutate `b.nested.y` sẽ ảnh hưởng `a.nested.y`.
2. `structuredClone` giữ `Date` là `Date`, `Map/Set` là `Map/Set` mới, xử lý circular (giữ quan hệ), nhưng ném với `function`/`WeakMap`/DOM. `JSON` biến `Date`→string, `Map/Set`→`{}`, mất `undefined/function/Symbol`, `Infinity→null`, và ném với circular.
3. Deep clone toàn bộ O(n) tốn kém và làm mọi nhánh reference mới → mọi selector/memo thấy thay đổi, re-render thừa, phá tối ưu. Đúng: shallow copy path thay đổi: `{...state, user:{...state.user, loc:{...state.user.loc, city:'HCM'}}}`, hoặc `produce(state, d=>{d.user.loc.city='HCM'})` (immer).

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 10. Spec: [HTML Structured Clone](https://html.spec.whatwg.org/#structured-clone), [MDN structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone).*
