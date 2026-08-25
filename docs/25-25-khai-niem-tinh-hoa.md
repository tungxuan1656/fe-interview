# 25 Khái Niệm Tinh Hoa - Học 1 Hiểu 10

> **199 câu -> 25 khái niệm -> 185 câu (bỏ Next.js ch.11 theo yêu cầu)**
> Học 1 khái niệm là trả lời được 5-10 câu hỏi biến thể. Đừng học thuộc câu hỏi, học cơ chế.

**Cách dùng:**
- **Bận (3 ngày):** Chỉ học 7 khái niệm gốc ⭐ (Tầng 0+1)
- **Đủ (7 ngày):** Học hết 25, mỗi ngày 3-4 khái niệm
- Mỗi khái niệm đọc 5 phút, gõ lại ví dụ 5 dòng là nhớ

---

## TẦNG 0: NỀN JAVASCRIPT - 3 khái niệm gốc (4h)

### 1. 🎒 Chiếc Balo Ký Ức (Scope - Closure - Rác)

**Nói nôm na:** Hàm như người đeo balo. Lúc tạo hàm, nó bỏ hết biến xung quanh vào balo (`[[Environment]]`). Dù đi xa (return ra ngoài) vẫn mở balo ra dùng được.

**Gom từ:** C1 `var/let/TDZ` + C2 Closure + C14 Memory Leak + Stale closure trong React

**Học 1 hiểu 4:**
- `var` bỏ vào balo ngay và ghi `undefined`, `let` bỏ vào nhưng khóa lại (TDZ) tới dòng khai báo mới mở
- `for(var i)` 1 balo dùng chung -> `3 3 3`, `for(let i)` mỗi vòng 1 balo mới -> `0 1 2`
- Giữ balo quá lâu = giữ cả DOM 2MB trong `useEffect` -> rò rỉ. `WeakMap` là balo tự hủy khi không ai cầm nữa

```js
function taoDem() {
  let dem = 0; // ở trong balo
  return () => ++dem;
}
const dem = taoDem();
dem(); // 1, balo vẫn giữ `dem` dù taoDem đã chạy xong

// Bẫy React
useEffect(() => {
  const id = setInterval(() => console.log(count), 1000); // count bị kẹt ở giá trị cũ
  return () => clearInterval(id); // quên dòng này là rò rỉ
}, []); // thiếu `count` trong deps
```

**Bẫy:** `typeof` biến chưa khai báo ra `undefined`, nhưng `typeof` biến trong TDZ lại ném lỗi.

**Hỏi 30s:** *"Closure là gì?"* -> *"Hàm nhớ nơi nó được tạo ra, không phải nơi nó được gọi. Dùng cho private state, debounce, nhưng giữ rác nếu không cleanup."*

**Hiểu sâu (2 phút):**
- **Vì sao:** Engine tạo `LexicalEnvironment` khi khai báo hàm, lưu `outer` reference. `var` hoisted + init `undefined`, `let/const` hoisted nhưng ở TDZ tới khi `InitializeBinding`.
- **Đừng dùng khi:** Closure giữ object to trong hot path/render loop → tốn RAM, GC không dọn.
- **Vỡ khi nào:** Vòng lặp `var` + async, stale closure trong React `useEffect(()=>{...},[])`, quên `clearInterval/removeEventListener` giữ DOM.
- **Đo:** Chrome DevTools Memory → Heap Snapshot tìm `Detached HTMLElement`, `Performance → Memory` graph tăng mãi không giảm.

**3 câu tự vấn:**
1. `for(var i=0;i<3;i++) setTimeout(()=>console.log(i),0)` in gì và sửa mấy cách?
2. Vì sao `typeof x` với `x` chưa khai báo thì `undefined` nhưng `let x; typeof x` trước dòng `let` lại throw?
3. `WeakMap` vs `Map` khác gì khi key là object DOM và khi nào dùng?

### 2. 🪪 Cái Tên `this` (Ai đang gọi tôi?)

**Nói nôm na:** `this` không phải tên cố định, mà là "ai đang gọi tôi lúc này". `obj.fn()` thì `this` là `obj`. `const fn = obj.fn; fn()` thì `this` mất, thành `undefined`.

**Gom từ:** C3 `this` + C12 `call/apply/bind` + C8 Curry + C9 Debounce/Throttle

**Nhớ 3 điều:**
1. Thứ tự ưu tiên: `new` > `bind` > `obj.fn()` > gọi trần
2. `bind` là dán keo cứng `this`, `call` lại cũng không gỡ được
3. Arrow function không có `this` riêng, nó mượn của người bao ngoài

```js
const user = { ten: 'An', chao() { console.log(this.ten) } }
user.chao(); // 'An' - ai gọi? user gọi
const fn = user.chao; fn(); // undefined - không ai gọi

const debounced = debounce(user.chao.bind(user), 300); // phải bind, không là mất this
```

**Bẫy:** `bind` arrow function vô tác dụng. Tạo `debounce` mới mỗi lần render là mất tác dụng.

**Hiểu sâu (2 phút):**
- **Vì sao:** `this` được quyết định ở *call-site*. `bind` tạo hàm mới với `[[BoundThis]]` cứng. Arrow không có `[[ThisMode]]` riêng, tra `this` theo lexical scope.
- **Đừng dùng khi:** Đã dùng arrow trong class field `onClick = () =>` thì không cần `bind`; lạm dụng `bind` trong render tạo hàm mới mỗi lần → vỡ `memo`.
- **Vỡ khi nào:** Mất `this` khi `const fn = obj.fn; fn()`, callback `setTimeout(obj.fn,0)`, `bind` arrow không hiệu lực, debounce tạo lại mỗi render.
- **Đo:** `console.log(this)` tại call-site, ESLint `no-invalid-this`, React DevTools check `memo` re-render do prop function đổi ref.

**3 câu tự vấn:**
1. Xếp thứ tự `new`, `bind`, `obj.fn()`, `fn()` cái nào thắng và ví dụ `new (fn.bind(obj))()` thì `this` là gì?
2. Vì sao `bind` lên arrow function không đổi `this`?
3. Vì sao `debounce` tạo trong component body mỗi render lại mất tác dụng, fix bằng gì (`useCallback`/`useRef`)?

### 3. 🚦 Ngã Tư Bất Đồng Bộ (Event Loop) ⭐ P0

**Nói nôm na:** JS chỉ có 1 đường. Xe ưu tiên (microtask: `Promise`) đi trước, xe thường (macrotask: `setTimeout`) đi sau. `await` chính là dừng lại, nhường đường cho xe ưu tiên chạy.

**Gom từ:** C5 Event Loop + C6 Promise/async + C7 Generator + C18 lỗi async

**Nhớ 3 điều:**
1. `Promise.then` luôn đi trước `setTimeout(0)` dù số 0
2. `await` dù `Promise` đã xong vẫn phải xếp hàng microtask
3. `try/catch` không bắt được lỗi ném trong `setTimeout`

```js
console.log(1);
setTimeout(()=>console.log(2),0);
Promise.resolve().then(()=>console.log(3));
console.log(4);
// 1, 4, 3, 2  (xe ưu tiên 3 đi trước xe thường 2)

await Promise.resolve(); // thực chất tách thành .then()
```

**Hiểu sâu (2 phút):**
- **Vì sao (spec):** Một vòng loop: `Task (macrotask) → Microtask checkpoint (làm sạch toàn bộ microtask queue) → Render (rAF → layout/paint nếu cần) → lặp`. `await` dịch thành `Promise.resolve(x).then(resume)`, nên luôn đẩy phần sau `await` vào microtask. `queueMicrotask()` cũng vào đây.
- **Sơ đồ nhớ:** `Task → [microtask queue cạn sạch] → requestAnimationFrame → Render → Task tiếp theo`. Microtask chạy xen giữa, không đợi render.
- **Đừng dùng khi:** Microtask đệ quy `queueMicrotask(loop)` hoặc `Promise.then(loop)` → starvation: render/rAF/macrotask không bao giờ tới, UI đơ.
- **Vỡ khi nào:** `try/catch` quanh `setTimeout` không bắt được; `await` trong `forEach` không đợi; Node: `process.nextTick` chạy **trước** microtask Promise, `setImmediate` là macrotask sau I/O, còn browser không có `nextTick`.
- **Đo:** Chrome Performance → Event Log xem `Task` vs `Microtask`, check Long Task >50ms, dùng `scheduler.postTask` hoặc `isInputPending()` để nhường.

**3 câu tự vấn:**
1. Thứ tự in ra của `setTimeout`, `Promise.then`, `queueMicrotask`, `requestAnimationFrame` là gì và vì sao `await Promise.resolve()` vẫn async?
2. Starvation là gì? Code `function loop(){ queueMicrotask(loop) }` sẽ vỡ UI thế nào?
3. Khác nhau `process.nextTick` vs `Promise.then` vs `setImmediate` trong Node và browser thay bằng gì?

---

## TẦNG 1: ĐỒ VẬT & CHỮ - 4 khái niệm (3h)

### 4. 👨‍👩‍👧 Dòng Họ (Prototype & Copy & Proxy) ⭐ P0

**Nói nôm na:** Mỗi object có sợi dây nối với cha (`__proto__`). Tìm đồ không thấy thì leo dây hỏi cha. `class` chỉ là cách viết ngắn gọn cho việc nối dây. `Proxy` là người gác cổng, chặn mọi lần sờ vào đồ.

**Gom từ:** C4 Prototype + C10 Shallow/Deep + C16 Proxy + C17 Symbol/WeakMap

**Nhớ 3 điều:**
1. `...spread` chỉ photo 1 mặt, đồ bên trong vẫn dùng chung -> đổi `nested.y` là đổi luôn gốc
2. `Proxy` gác được cả việc thêm khóa mới, `Object.defineProperty` (Vue2) thì không
3. `structuredClone` photo được Date/Map nhưng không photo được function

```js
const goc = { a:1, con:{ b:2 } }
const photo = { ...goc }; // shallow
photo.con.b = 99; // goc.con.b cũng thành 99!
const that = structuredClone(goc); // deep thật

// new làm 3 bước
function Person(name){ this.name=name }
Person.prototype.hi = function(){ return this.name }
const p = new Person('An') // 1) tạo {} với __proto__=Person.prototype 2) gọi Person.call(obj) 3) return obj

// Pollution
// Object.prototype.__proto__ bị gán -> mọi object dính
Object.create(null) // object không có cha, an toàn làm map
```

**Hiểu sâu (2 phút):**
- **Vì sao:** `new Fn()` = `const o = Object.create(Fn.prototype); Fn.call(o); return o`. `__proto__` là con trỏ instance → cha, `Fn.prototype` là object mẫu cho con. `obj.__proto__ === Fn.prototype`. `instanceof` chỉ check leo chuỗi `__proto__` có thấy `Fn.prototype` không, nên `instanceof` vỡ khi iframe/realm khác.
- **Đừng dùng khi:** `__proto__` setter chậm, pollution: `JSON.parse` + gán `__proto__`/`constructor.prototype` → RCE. Dùng `Object.create(null)` cho dict, `Map` cho key object.
- **Vỡ khi nào:** Prototype pollution `lodash merge`, `structuredClone` mất function/Symbol, `Proxy` mời `isArray` check sai nếu không trap đúng, `WeakMap` không iterate/ không có `.size`.
- **Đo:** `Object.getPrototypeOf(o)`, `o instanceof Fn`, `Object.hasOwn(o,'x')` vs `in`, Snyk/CodeQL scan pollution, test `Object.create(null)`.

**3 câu tự vấn:**
1. Kể 3 bước của `new`, phân biệt `__proto__` vs `prototype`, và vì sao `instanceof` có thể sai qua iframe?
2. Prototype pollution xảy ra thế nào và fix bằng `Object.create(null)` / `Map` ra sao?
3. `Proxy` hơn `defineProperty` ở điểm nào (thêm/xóa key, array index) và hạn chế của `structuredClone`?

### 5. 🚪 Cửa Khẩu Module (CJS vs ESM)

**Nói nôm na:** `CommonJS` như chợ trời: thích `require` lúc nào cũng được. `ESM` như sân bay: phải khai báo `import` ở cửa, nên hải quan (bundler) mới soi được đồ nào không dùng để vứt (tree-shaking).

**Gom từ:** C15

**Nhớ:** ESM `import {count} from './a'` là nhìn trực tiếp, bên kia `count++` bên này thấy luôn (live). CJS thì photo 1 bản.

**Hiểu sâu (2 phút):**
- **Vì sao:** ESM `import/export` tĩnh → bundler phân tích cây, loại dead code (tree-shaking). CJS `require` động, `module.exports` là copy giá trị lúc chạy.
- **Đừng dùng khi:** Cần `require` điều kiện động → dùng `import()` dynamic; trộn CJS/ESM gây `default` lẫn lộn `.default`.
- **Vỡ khi nào:** Circular deps: ESM thấy `live binding` nhưng `undefined` tạm; CJS nhận object chưa init xong.
- **Đo:** `madge --circular`, `agadoo` check ESM, `webpack-bundle-analyzer` xem module còn sót.

**3 câu tự vấn:**
1. Vì sao ESM tree-shake được còn CJS không?
2. `import {count}` live binding nghĩa là gì, khác gì `const {count}=require()`?
3. Circular import A↔B thì ESM vs CJS vỡ kiểu gì?

### 6. 📏 Thước Đo Chữ (TypeScript Nền)

**Nói nôm na:** `any` = tắt thước, đo bừa. `unknown` = khóa thước, phải đo lại mới dùng. `never` = không có gì vừa. `strictNullChecks` = bắt buộc ghi `string | null` chứ không cho `string` mà nhét `null`.

**Gom từ:** C19 any/unknown/never + C20 type vs interface + C25 strict + C26 Enum/Union + C28 Narrowing

**Nhớ 3 điều:**
1. `type` làm được `A | B`, `interface` thì gộp tên được (merge)
2. `Union` + `as const` tốt hơn `Enum` (Enum sinh code thừa)
3. `if(typeof x==='string')` chính là thước đo (narrowing)

```ts
let a: unknown = "hi";
a.toUpperCase() // lỗi, phải if(typeof a==='string') trước

type Hinh = {loai:'tron', r:number} | {loai:'vuong', canh:number}
function tinh(h:Hinh){ if(h.loai==='tron') return Math.PI*h.r**2 } // narrow
```

**Hiểu sâu (2 phút):**
- **Vì sao:** `unknown` bắt narrowing mới dùng, `any` tắt check, `never` cho nhánh không bao giờ tới (exhaustive check). `strictNullChecks` tách `null` khỏi mọi type.
- **Đừng dùng khi:** `any` trong lib public → lan lỗi; `Enum` số tự tăng → bundle dư, dùng `Union`.
- **Vỡ khi nào:** `as` ép bừa mất narrowing, `!` non-null bỏ qua `null`.
- **Đo:** `tsc --strict`, `ts-expect-error` test, `eslint @typescript-eslint/no-explicit-any`.

**3 câu tự vấn:**
1. `any` vs `unknown` vs `never` khác nhau khi gán và khi gọi hàm?
2. Khi nào dùng `type` vs `interface` và `Enum` vs `Union + as const`?
3. Narrowing bằng `typeof`, `in`, discriminant `loai` hoạt động thế nào?

### 7. 🧙 Phù Thủy Biến Hình (Type Nâng Cao)

**Nói nôm na:** Viết kiểu dữ liệu như viết hàm. `Generics` là khuôn, `Partial` là biến mọi thứ thành optional, `in` + `` `get${Capitalize<K>}` `` là dán chữ.

**Gom từ:** C21 Generics + C22 Utility + C23 Conditional/infer + C24 Mapped/Template + C27 Decorator + C29 Merging + C30 Monorepo

**Học 1 hiểu 1 mạch:**
```ts
type PartialCuaToi<T> = { [K in keyof T]?: T[K] } // tự làm Partial
type LayKetQua<T> = T extends (...a:any)=> infer R ? R : never // infer = móc ra
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: ()=>T[K] }
```
*Decorator `@Log` + `declare global` chỉ là cách dán thêm đồ vào nhà người khác.*

**Hiểu sâu (2 phút):**
- **Vì sao:** Conditional `T extends U ? X:Y` phân phối trên union; `infer R` móc type ra; mapped `in keyof` biến từng key.
- **Đừng dùng khi:** Type quá sâu đệ quy → `Type instantiation is excessively deep`, làm chậm IDE.
- **Vỡ khi nào:** `infer` trong union phân phối bất ngờ, `as` template mất check.
- **Đo:** `tsc --noEmit` time, `type-coverage`, hover type trong VSCode.

**3 câu tự vấn:**
1. `infer` trong `ReturnType<T>` hoạt động thế nào?
2. `Partial`, `Pick`, `Omit` viết lại bằng mapped type ra sao?
3. Khi nào conditional type bị phân phối và cách chặn bằng `[T] extends [U]`?

---

## TẦNG 2: REACT - 6 khái niệm (5h)

### 8. 🎬 Xưởng Phim (Render & Fiber & Key)

**Nói nôm na:** `render` là vẽ phác thảo (React Element), chưa dán lên tường (DOM). `Reconciliation` là so 2 bản phác thảo để dán ít nhất. `key` là số báo danh, không có là cô giáo chấm nhầm bài. `Fiber` là chia việc vẽ thành tờ nhỏ để đang vẽ dở mà có việc gấp (gõ phím) thì dừng lại.

**Gom từ:** 02-31 render + 02-32 VDOM + 02-33 key + 03-51 Fiber

**Nhớ 3 điều:**
1. Cha render -> con render theo dù props không đổi, trừ khi `memo`
2. `key={index}` khi xóa giữa danh sách sẽ làm checkbox tích nhầm người
3. `key` không nằm trong `props`, `key={Math.random()}` mỗi lần vẽ lại là phá luôn

```jsx
{todos.map(t => <Todo key={t.id} data={t} />)} // đúng
{todos.map((t,i) => <Todo key={i} />)} // sai khi reorder
```

**Hiểu sâu (2 phút):**
- **Vì sao:** Fiber là linked list có `alternate` double buffer, lane priority, work có thể pause/resume. Reconcile so key + type để reuse DOM.
- **Đừng dùng khi:** `key={index}` với list reorder/filter, `key={Math.random()}` remount mất state.
- **Vỡ khi nào:** Cha render kéo con render dây chuyền, mất focus/input state khi key đổi.
- **Đo:** React DevTools Profiler highlight re-render, `why-did-you-render`.

**3 câu tự vấn:**
1. Fiber giải quyết bài toán gì mà stack reconciler cũ không làm được (interruptible)?
2. Vì sao `key` phải ổn định và không có trong `props`?
3. Cha re-render có bắt con re-render không và `memo` chặn ở đâu?

### 9. 🔌 Công Tắc Điện (Hooks & Effect)

**Nói nôm na:** Hooks như ổ điện xếp hàng theo thứ tự. Cắm sai thứ tự là chập điện (vi phạm Rules). `useState` là công tắc có nhớ, `useEffect` là hẹn giờ "khi nào xong việc thì chạy", `useRef` là hộp bí mật không gây ồn (không render lại).

**Gom từ:** 02-35 useState batching + 02-36/37 useEffect/layoutEffect + 02-39 useRef + 02-49 Rules

**Nhớ 3 điều:**
1. `setCount(c=>c+1)` 3 lần mới thành +3, `setCount(count+1)` 3 lần chỉ +1 (do kẹt closure)
2. `useEffect` chạy sau khi sơn xong, `useLayoutEffect` chạy trước khi sơn để đo đạc không nháy
3. `useRef` đổi `current` không render lại -> dùng để giữ `setInterval` id, giá trị cũ

```jsx
useEffect(()=>{
  const c = new AbortController();
  fetch(`/api/user/${id}`, {signal:c.signal}).then(...)
  return ()=> c.abort(); // dọn dẹp, không là race condition
}, [id]) // thiếu id là bug
```

**Hiểu sâu (2 phút):**
- **Vì sao:** Hooks lưu theo index trong fiber, nên phải gọi đúng thứ tự, không trong `if`. `useEffect` = passive (sau paint), `useLayoutEffect` = layout (trước paint, block).
- **Đừng dùng khi:** `useLayoutEffect` cho fetch/data → block paint; `useEffect` để tính derived state → dùng `useMemo`.
- **Vỡ khi nào:** Thiếu deps → stale, thừa deps → loop, quên cleanup → leak/race.
- **Đo:** ESLint `react-hooks/exhaustive-deps`, React DevTools timeline.

**3 câu tự vấn:**
1. Vì sao không được gọi hooks trong `if/for`?
2. `useEffect` vs `useLayoutEffect` khác thời điểm nào và khi nào bắt buộc dùng `useLayoutEffect`?
3. `setState` dạng updater `c=>c+1` vs `count+1` khác nhau khi batching?

### 10. 🧹 Chế Độ Tiết Kiệm (Memo & Re-render)

**Nói nôm na:** `memo` như dán giấy "đừng vẽ lại nếu đồ không đổi". Nhưng dán giấy cũng tốn băng keo. Chỉ dán khi đo thấy lag.

**Gom từ:** 02-38 memo + 02-40 Context + 02-43 Colocation + 03-55 Transition + 03-63 Profiler

**Nhớ 3 điều:**
1. `Context value={{a,b}}` tạo object mới mỗi lần -> mọi con đều render lại, phải `useMemo` value hoặc tách Context
2. Để state gần nơi dùng nhất (colocation) nhanh hơn nâng lên App
3. Gõ tìm kiếm list 10k: `useDeferredValue` để ô input mượt, list cập nhật sau

```jsx
const deferredQuery = useDeferredValue(query); // query gõ mượt
const list = useMemo(()=> filter(data, deferredQuery), [deferredQuery]);
```

**Hiểu sâu (2 phút):**
- **Vì sao:** `memo` shallow compare props, `useMemo/useCallback` giữ ref ổn định. Context là broadcast, mọi consumer re-render khi value đổi ref.
- **Đừng dùng khi:** Component rẻ (< few ms) mà `memo` tốn so sánh; bọc hết → code phức tạp không lợi.
- **Vỡ khi nào:** `value={{}}` inline, `children` không memo, inline arrow prop phá `memo`.
- **Đo:** Profiler `wasted renders`, `useDeferredValue` + `startTransition` cho INP.

**3 câu tự vấn:**
1. `memo` so sánh thế nào và khi nào nó vô tác dụng vì prop luôn tạo mới?
2. Vì sao `Context` hay gây re-render dây chuyền và cách tách Context?
3. `useTransition` vs `useDeferredValue` khác nhau khi nào dùng?

### 11. 🔀 Đường Dây Điện (Chia State)

**Nói nôm na:** Đừng kéo dây điện từ tầng 1 lên tầng 10 chỉ để bật đèn tầng 10 (prop drilling). Có 3 cách: đưa bóng đèn xuống gần công tắc (composition `children`), đi đường dây chung (Context), hoặc dùng tổng đài (store).

**Gom từ:** 02-41 drilling + 02-42 Compound + 02-44 Redux/Zustand/Jotai + 02-45 RTK

**Nhớ:** `Zustand` như công tắc đơn giản, `Redux` như tủ điện có sổ sách (time-travel), `Jotai` như mỗi bóng 1 công tắc riêng (atomic) -> đỡ render ké.

**Hiểu sâu (2 phút):**
- **Vì sao:** Prop drilling = coupling, Context = broadcast 1 value, Store atomic (Jotai/Zustand selector) chỉ re-render nơi đọc slice.
- **Đừng dùng khi:** App nhỏ mà Redux boilerplate; global hết cả form local state.
- **Vỡ khi nào:** Store để cả server state + client state → stale, selector không stable.
- **Đo:** `why-did-you-render`, count renders trong Profiler.

**3 câu tự vấn:**
1. 3 cách tránh prop drilling và trade-off từng cách?
2. Zustand vs Redux vs Jotai khác mô hình re-render thế nào?
3. Khi nào nên để state ở URL thay vì store?

### 12. 🚧 Đội Cứu Hộ (Suspense & Lỗi) ⭐ P0

**Nói nôm na:** `Suspense` là cửa chặn throw Promise — component ném Promise thì cửa giữ lại, hiện `fallback` tới khi Promise xong mới cho qua. `Error Boundary` là lưới an toàn hứng khi rơi. `lazy` là chỉ khi nào cần mới đi lấy đồ.

**Gom từ:** 03-56 Suspense + 03-57 Error Boundary + 03-62 Code Splitting

```jsx
<Suspense fallback={<Skeleton />}><Comments /></Suspense>
<ErrorBoundary fallback={<Loi roi />}><App /></ErrorBoundary>
const Chart = lazy(()=> import('./Chart')); // route-based split
```

**Hiểu sâu (2 phút):**
- **Vì sao:** Suspense hoạt động bằng `throw Promise` trong render: React bắt Promise, treo subtree, hiện fallback. Khi Promise resolve, React thử render lại. Error Boundary (class `getDerivedStateFromError`/`componentDidCatch`) bắt lỗi trong *render/lifecycle* của con, giống `try/catch` cho JSX.
- **Đừng dùng khi:** Mong EB bắt `async`/`event handler`/`setTimeout`/`SSR` → không bắt được, phải `try/catch` trong handler hoặc `useErrorBoundary` thủ công.
- **Vỡ khi nào:** `fetch` trong `useEffect` không throw Promise nên Suspense không hiện; EB kẹt ở trạng thái lỗi mãi nếu không `reset` bằng `key` hoặc `resetErrorBoundary`.
- **Đo:** React DevTools Suspense badge, test bằng `throw new Promise` mock, `componentDidCatch` log.

**3 câu tự vấn:**
1. Cơ chế `throw Promise` của Suspense khác gì `isLoading` state thường?
2. Error Boundary bắt và *không bắt* những lỗi nào (async/event/SSR/Promise)?
3. Làm sao reset Error Boundary sau khi lỗi (key vs reset function) và vì sao lazy cần Suspense?

### 13. 🏭 Nhà Máy Server (SSR & Cache) ⭐ P0

**Nói nôm na:** `CSR` là khách tự nấu (trắng trang rồi mới hiện). `SSR` là bếp nấu sẵn gửi ra HTML (nhanh nhưng bếp bận). `SSG` là nấu sẵn từ sáng, `ISR` là nấu sẵn nhưng 60s hâm lại. `Hydration` là gắn tay cầm (event handler) lên HTML có sẵn, không vẽ lại. `RSC` là món chỉ bếp làm, không gửi công thức/JS ra bàn.

**Gom từ:** 03-58 RSC + 03-60 SSR/SSG/ISR + 03-61 Streaming + 03-59 Hydration + 03-66 Actions + 03-67 Cache + C52-54 React 18/19 + C65

**Nhớ 3 điều:**
1. `Date.now()` / `Math.random()` trong render sẽ làm Hydration lệch (server vs client khác nhau)
2. Next.js có 4 két: `Request Memo` (1 request) -> `Data Cache` (force-cache) -> `Route Cache` -> `Router Cache` (client)
3. React 19 `use` đọc Promise ngay trong render, `Compiler` tự dán `memo` cho bạn

**Hiểu sâu (2 phút):**
- **Vì sao:** SSR render HTML trên server cho FCP nhanh + SEO. Hydration là `hydrateRoot` đi qua DOM có sẵn, *attach* handler + so khớp, không tạo DOM mới nên nhanh hơn `createRoot`. RSC (Server Component) chạy chỉ trên server, trả *payload* serializable, **không ship JS** cho client, giảm bundle. Streaming SSR gửi `Suspense` chunk dần.
- **Đừng dùng khi:** Trang cần tương tác cao mà SSR toàn bộ → TTFB lâu; RSC cho UI cần `useState`/`onClick` → phải `"use client"`.
- **Vỡ khi nào:** Hydration mismatch (server `new Date()` khác client, `window` trong render) → React cảnh báo và *bỏ* hydration cục bộ, client render lại (mất lợi SSR). `useEffect` chỉ chạy client nên đặt code browser ở đó.
- **Đo:** View Source xem HTML SSR, `hydrate` warning trong console, `__NEXT_DATA__`, Lighthouse TTFB, Next.js `fetch` cache header.

**3 câu tự vấn:**
1. Hydration khác `render` thường ở điểm nào và khi mismatch React làm gì?
2. RSC khác SSR/SSG ở chỗ "không ship JS" nghĩa là gì, client component ranh giới ở đâu?
3. 4 tầng cache Next.js khác nhau thế nào và `fetch({cache:'no-store'})` chạm tầng nào?

---

## TẦNG 3: TRÌNH DUYỆT & TỐC ĐỘ - 5 khái niệm (4h)

### 14. 🎨 Dây Chuyền Sơn (Reflow & Transform)

**Nói nôm na:** Sơn nhà 3 bước: Dựng khung (Layout) -> Quét sơn (Paint) -> Dán giấy (Composite). Đổi `width` là đập khung làm lại, đổi `transform` chỉ dán lại giấy -> rẻ nhất, chạy trên GPU 60fps.

**Gom từ:** 06-105 Pipeline + 06-106 Reflow/Paint + 06-107 Layer + 06-108 fixed + 06-109 transform + 05-102 scroll jank + 05-103 Leak

**Nhớ 3 điều:**
1. Đọc `offsetHeight` rồi ghi `style.height` trong vòng lặp là tự đập khung liên tục (thrashing) -> phải gom đọc trước, ghi sau
2. `will-change: transform` như đặt trước 1 lớp kính, tốn RAM (1 màn 8MB), dùng xong phải gỡ
3. `transform` trên cha sẽ làm `position:fixed` của con kẹt theo cha

```js
// ❌
els.forEach(el=>{ el.style.height = el.offsetHeight+10+'px' })
// ✅
const h = els.map(el=>el.offsetHeight); els.forEach((el,i)=> el.style.height = h[i]+10+'px')
```

**Hiểu sâu (2 phút):**
- **Vì sao:** Layout tính geometry, Paint raster, Composite ghép layer trên GPU. `transform/opacity` chỉ composite nên 60fps.
- **Đừng dùng khi:** `will-change` cho mọi thứ → tốn VRAM, `transform` thay layout khi cần reflow thật.
- **Vỡ khi nào:** Layout thrashing loop đọc-ghi, `fixed` trong `transform` parent.
- **Đo:** Chrome Performance → Layout/Paint timings, Layers panel, `will-change` memory.

**3 câu tự vấn:**
1. Thứ tự Layout → Paint → Composite và thuộc tính nào chỉ触发 composite?
2. Layout thrashing là gì và fix bằng batch read/write?
3. Vì sao `will-change` tốn RAM và `transform` làm `fixed` hỏng?

### 15. 🏷️ Biển Số Xe (Web Vitals)

**Nói nôm na:** Google chấm 3 điểm: `LCP` (ảnh to nhất hiện khi nào <2.5s), `INP` (bấm nút bao lâu mới phản hồi <200ms), `CLS` (trang có nhảy lung tung <0.1). `Lighthouse` là thi thử, `RUM` là thi thật ngoài đường.

**Gom từ:** 05-86 bottleneck + 05-87 đo + 05-88 Vitals + 05-89 LCP + 05-90 INP + 05-91 CLS + 05-92 tối ưu + 05-104 Budget

**Nhớ:** Ảnh hero phải `preload + fetchpriority=high + width/height`, đừng `lazy`.

**Hiểu sâu (2 phút):**
- **Vì sao:** LCP = largest paint, INP = max interaction latency (thay FID), CLS = tổng layout shift score. Ảnh hưởng SEO.
- **Đừng dùng khi:** Tối ưu Vitals mà bỏ UX thật; `preload` quá nhiều → nghẽn băng thông.
- **Vỡ khi nào:** Ảnh không size → CLS, JS block main → INP cao, hero lazy → LCP chậm.
- **Đo:** `web-vitals` lib, CrUX, Lighthouse, `PerformanceObserver`.

**3 câu tự vấn:**
1. LCP/INP/CLS ngưỡng tốt là bao nhiêu và đo bằng gì (lab vs field)?
2. Vì sao ảnh hero không nên `lazy` và cần `width/height`?
3. INP khác FID thế nào và nguyên nhân INP cao thường là gì?

### 16. 📦 Kho Bưu Kiện (Ảnh/Font/Bundle)

**Nói nôm na:** Ảnh như bưu kiện: gửi đúng size (`srcset` 400w/800w cho điện thoại/máy tính), đúng loại (AVIF nhỏ nhất). Font như quần áo: chỉ gửi size mình mặc (subset latin), `swap` là mặc tạm đồ cũ kẻo nude (FOIT).

**Gom từ:** 05-93 Bundle + 05-94 Splitting + 05-95 Image + 05-96 Font + 05-97 preload/prefetch

**Nhớ 3 điều:**
1. `import _ from 'lodash'` kéo cả kho, `import debounce from 'lodash-es/debounce'` chỉ lấy 1 món
2. `preload` là lấy ngay (hero), `prefetch` là lấy khi rảnh (trang sau), `preconnect` là mở đường trước
3. `barrel export * from './Button'` làm bundler mù, không tree-shake được

**Hiểu sâu (2 phút):**
- **Vì sao:** `srcset/sizes` cho responsive, `fetchpriority` hint, font `subset` + `display:swap` tránh FOIT, barrel export cản tree-shaking do side-effects.
- **Đừng dùng khi:** `preload` font không dùng, `prefetch` quá nhiều trên 3G.
- **Vỡ khi nào:** Bundle phình do lodash/moment barrel, ảnh không nén AVIF/WebP.
- **Đo:** `webpack-bundle-analyzer`, Lighthouse image/font audit, `coverage` panel.

**3 câu tự vấn:**
1. `preload` vs `prefetch` vs `preconnect` khác nhau khi nào dùng?
2. Vì sao `import _ from 'lodash'` nặng và barrel export hại tree-shaking?
3. `srcset` + `sizes` và `AVIF` giúp gì cho LCP?

### 17. 🔐 Két Sắt Trình Duyệt (Cache & Offline)

**Nói nôm na:** `max-age=31536000, immutable` là niêm phong 1 năm (chỉ dùng cho file có hash `app.abc123.js`). `ETag` là vân tay, lần sau gửi `If-None-Match` nếu khớp server trả `304` không cần gửi lại. `Service Worker` là người gác kho lập trình được.

**Gom từ:** 05-98 HTTP Cache + 05-99 CDN/Browser/SW + 06-118 SW

**Nhớ:** `sw.js` phải `no-cache` kẻo kẹt bản cũ. `Cache First` cho ảnh, `Network First` cho API.

**Hiểu sâu (2 phút):**
- **Vì sao:** Strong cache (`max-age/immutable`) không hỏi server, revalidate (`ETag/Last-Modified`) hỏi `304`. SW intercept `fetch`.
- **Đừng dùng khi:** `immutable` cho HTML không hash → kẹt bản cũ; cache API POST.
- **Vỡ khi nào:** `sw.js` bị cache → không update, `Cache First` cho API → data cũ.
- **Đo:** DevTools Network `Size: (disk cache)` vs `304`, Application → Cache Storage.

**3 câu tự vấn:**
1. `max-age` + `immutable` vs `ETag`/`304` khác nhau thế nào?
2. Vì sao `sw.js` phải `no-cache` và chiến lược Cache First vs Network First?
3. CDN vs Browser cache vs SW cache tầng nào trước?

### 18. 🧳 Tủ Đồ Cá Nhân (Storage)

**Nói nôm na:** `Cookie` 4KB tự gửi kèm mỗi lần đi chợ (kèm `HttpOnly` thì JS không đọc được). `localStorage` 5MB để ở nhà, `IndexedDB` như kho hàng GB. `PWA` là biến web thành app (cần Manifest + SW + HTTPS).

**Gom từ:** 06-112 Cookie vs Storage + 06-120 IndexedDB + 06-119 PWA

**Nhớ:** Đừng để JWT trong `localStorage` (XSS đọc được). Dùng `httpOnly` cookie.

**Hiểu sâu (2 phút):**
- **Vì sao:** Cookie gửi tự động → hợp auth, `HttpOnly+Secure+SameSite` chống XSS/CSRF. `localStorage` sync block main, `IndexedDB` async lớn.
- **Đừng dùng khi:** JWT trong `localStorage` (XSS), `localStorage` cho data to → block.
- **Vỡ khi nào:** Quota 5MB đầy throw, `IndexedDB` version migrate lỗi, PWA thiếu manifest.
- **Đo:** DevTools Application → Storage, check `HttpOnly` trong Network Set-Cookie.

**3 câu tự vấn:**
1. Vì sao không để JWT trong `localStorage` mà dùng `httpOnly` cookie?
2. `localStorage` vs `sessionStorage` vs `IndexedDB` khác dung lượng và sync/async?
3. PWA cần 3 điều gì và `SameSite` cookie chống gì?

---

## TẦNG 4: MẠNG & HỢP ĐỒNG - 2 khái niệm (2h)

### 19. 🛂 Biên Giới (Origin & CORS & HTTP) ⭐ P0

**Nói nôm na:** `Same-Origin` là biên giới (scheme+host+port). `CORS` là giấy thông hành do server cấp (`Access-Control-Allow-Origin`) nhưng để *trình duyệt* bảo vệ *user*, không phải để bảo vệ server. `OPTIONS` (preflight) là hỏi trước "tôi mang `Authorization` có được qua không?".

**Gom từ:** 06-111 CORS + 06-113 SOP + 06-114 HTTP/1.1/2/3 + 06-116 WS + 06-117 SSE + 06-110 Event

**Nhớ 3 điều:**
1. `Content-Type: application/json` sẽ trigger `OPTIONS`, `text/plain` thì không
2. `Allow-Origin: *` không đi với `credentials: true`
3. `HTTP/2` gộp nhiều làn trên 1 đường (multiplex), `HTTP/3` chạy trên UDP nên không kẹt 1 xe là kẹt cả đoàn. `SSE` 1 chiều (server đẩy), `WS` 2 chiều

**Hiểu sâu (2 phút):**
- **Vì sao:** SOP/CORS là cơ chế *browser* chặn trang A đọc response trang B, để bảo vệ user khỏi site độc đọc data ngân hàng. Server vẫn nhận request! `curl/Postman` không bị CORS vì không phải browser. Simple request (`GET/POST` + `text/plain`/`x-www-form-urlencoded` + header safe) đi luôn; còn lại (JSON, `Authorization`, `PUT/DELETE`) → browser gửi `OPTIONS` preflight, hỏi `Allow-Methods/Allow-Headers`, được mới gửi thật. `Access-Control-Max-Age` cache preflight (ví dụ 600s) để khỏi hỏi lại.
- **Đừng dùng khi:** Nghĩ CORS bảo vệ server → phải auth/rate-limit server-side; `*` cho API private.
- **Vỡ khi nào:** `credentials: include` mà server `Allow-Origin: *` → browser block; preflight thiếu `Allow-Headers: Authorization`; `SameSite` cookie chặn cross-site gửi kèm.
- **Đo:** DevTools Network → `OPTIONS` 204, Console CORS error, `curl -H Origin: ... -v` xem header, check `Max-Age`.

**3 câu tự vấn:**
1. CORS bảo vệ ai (browser/user) và vì sao `curl` không bị chặn?
2. Simple request vs preflight khác điều kiện gì và `Max-Age` để làm gì?
3. Vì sao `Allow-Origin: *` không dùng được với `credentials: true` và fix thế nào?

### 20. 📜 Ngôn Ngữ Hợp Đồng (REST)

**Nói nôm na:** `REST` là quy ước đặt tên đường: `GET /products` (lấy), `POST` (tạo), `PUT` (thay cả), `PATCH` (vá 1 miếng). `PUT` dán lại 10 lần vẫn 1 kết quả (idempotent), `POST` dán 10 lần ra 10 đơn.

**Gom từ:** 08-135 REST + 08-136 PUT/PATCH + 08-137 Status + 08-138 401/403 + 08-139 404/410 + 08-140 400/422

**Nhớ:** `401` chưa đăng nhập, `403` đã đăng nhập nhưng không có quyền. `404` không biết có từng tồn tại không, `410` là xóa vĩnh viễn (Google xóa index ngay).

**Hiểu sâu (2 phút):**
- **Vì sao:** REST dùng HTTP semantics: idempotent (`GET/PUT/DELETE`) retry an toàn, `POST` không. Status code là hợp đồng.
- **Đừng dùng khi:** `POST` cho mọi thứ → mất cache/idempotent; `PUT` cho patch nhỏ.
- **Vỡ khi nào:** Retry `POST` tạo trùng, `401` vs `403` lẫn, `404` vs `410` SEO.
- **Đo:** `curl -i`, OpenAPI lint, check idempotency-key.

**3 câu tự vấn:**
1. `PUT` vs `PATCH` vs `POST` khác idempotent thế nào?
2. `401` vs `403` và `404` vs `410` phân biệt và ảnh hưởng SEO?
3. `400` vs `422` khi validate body khác gì?

---

## TẦNG 5: KIẾN TRÚC - 2 khái niệm (2h)

### 21. 🗺️ Bản Vẽ Thành Phố (Architecture)

**Nói nôm na:** `Feature-Sliced` là chia thành phố theo khu (auth, cart) mỗi khu có đủ nhà/đường, đừng chia theo loại gạch (components/hooks/utils). `Design System` là bộ gạch chuẩn. `God Component` là tòa nhà 2000 dòng làm mọi thứ.

**Gom từ:** 04-69 structure + 04-70 Feature vs Layer + 04-71 chéo + 04-72 Design System + 04-73 reusable + 04-74 God + 04-81 Migrate + 04-83 MFE

**Nhớ 3 điều:**
1. `shared` không được import `features`, `features` không import nhau
2. Chỉ tách reusable khi thấy lặp 3 lần (Rule of Three)
3. `Micro-frontend` chỉ khi >30 dev, 4 team deploy riêng, còn lại `Monorepo + Turborepo` đủ

**Hiểu sâu (2 phút):**
- **Vì sao:** Feature-Sliced giảm coupling, boundary lint (`import/no-restricted`). Design tokens đồng nhất.
- **Đừng dùng khi:** App nhỏ mà FSD/MFE quá nặng; tách component quá sớm.
- **Vỡ khi nào:** Import chéo vòng, God component 2k dòng, MFE version lệch.
- **Đo:** `madge` circular, `eslint` boundaries, bundle size per feature.

**3 câu tự vấn:**
1. Vì sao chia theo feature tốt hơn chia theo loại file?
2. Rule of Three cho reusable và khi nào dùng MFE?
3. Làm sao ngăn `shared` import `features`?

### 22. 🚚 Đồng Bộ Hàng Hóa (Data Sync)

**Nói nôm na:** Lấy hàng có thể kẹt xe (race), hàng cũ (stale), mất mạng. `AbortController` là hủy đơn cũ khi đặt đơn mới. `Optimistic` là cho khách cầm hàng trước, nếu lỗi thì giật lại. `ETag` là tem kiểm hàng.

**Gom từ:** 08-141 Retry + 08-142 Backoff + 08-143 Race + 08-144 Autocomplete + 08-145 Abort + 08-146 Offset/Cursor + 08-147 Infinite + 08-148 Optimistic + 08-149 ETag + 08-150 Contract + 04-78 API Layer + 04-79 Error + 04-80 Loading + 05-100 Virtualization

**Nhớ 3 điều:**
1. `POST` không retry nếu không có `Idempotency-Key` sẽ tạo 2 đơn
2. `Cursor` (dùng `id`) nhanh hơn `Offset` khi lướt sâu, không bị lệch khi có người chen hàng
3. `stale-while-revalidate` là cho đồ cũ trước, đi lấy đồ mới sau lưng

```js
let controller;
async function search(q){
  controller?.abort();
  controller = new AbortController();
  const res = await fetch(`/api/search?q=${q}`, {signal: controller.signal});
}
```

**Hiểu sâu (2 phút):**
- **Vì sao:** Race do response về lệch thứ tự; `AbortController` hủy request cũ. Optimistic update UX nhanh nhưng cần rollback. `ETag/If-Match` tránh lost update.
- **Đừng dùng khi:** Retry `POST` không idempotent key; optimistic cho thao tác không revert được (payment).
- **Vỡ khi nào:** Autocomplete không abort → kẹt result cũ, cursor sai khi sort đổi.
- **Đo:** Network abort status, `React Query` devtools, `If-None-Match` 304.

**3 câu tự vấn:**
1. Race condition autocomplete fix bằng `AbortController` thế nào?
2. Optimistic update rollback khi nào và cần `ETag` để làm gì?
3. Cursor vs Offset pagination trade-off?

---

## TẦNG 6: TRẠNG THÁI & CHẤT LƯỢNG - 3 khái niệm (3h)

### 23. 🧠 Chia Việc Thông Minh (State) ⭐ BỔ SUNG QUAN TRỌNG

**Nói nôm na:** Đừng bỏ mọi đồ vào 1 tủ (Redux). Hỏi 3 câu: Đồ này của ai? (server/client), Dùng ở mấy phòng? (global/local), Có cần dán lên tường để share link không? (URL)

**Gom từ:** 04-75 Shared + 04-76 5 loại + 04-77 Server vs Client + 09-151->160 (10 câu bị sót) + 04-77

**Bảng 1 trang nhớ luôn:**

| Đồ | Để đâu | Vì sao |
|---|---|---|
| `products`, `orders` | `TanStack Query` | Server state, cần cache/stale/invalidate |
| `user` | `Query + Context` | Server nhưng dùng khắp nơi |
| `cart` | `Zustand + persist + sync /api/cart` | Hybrid: cần offline + đồng bộ |
| `filter`, `page` | `URL ?category=shoes` | Để share link, back/forward |
| `search input` | `useState` local | Chỉ 1 ô input dùng |
| `theme/modal` | `Zustand` hoặc `Context` | Client global ít đổi |

**TanStack 30s:**
- `staleTime: 0` (mặc định) = luôn cũ -> tự fetch lại khi focus
- `gcTime: 5m` = 5 phút không ai dùng thì xóa
- `invalidateQueries` = đánh dấu cũ, `setQueryData` = nhét luôn không cần fetch

```js
const {data} = useQuery({queryKey:['products', cat], queryFn:()=>fetchProducts(cat), staleTime: 60*1000})
// sau khi POST
queryClient.invalidateQueries({queryKey:['products']})
```

**Bẫy:** Lưu `products` vào Redux rồi tự `useEffect` fetch là tự làm lại TanStack dở hơn.

**Hiểu sâu (2 phút):**
- **Vì sao:** Server state có stale/cache/dedup/background refetch → Query; client state đồng bộ → store; URL state shareable.
- **Đừng dùng khi:** Nhét server data vào Zustand/Redux → tự quản cache dở.
- **Vỡ khi nào:** `staleTime` 0 fetch liên tục, `gcTime` ngắn mất cache, quên `queryKey` phụ thuộc.
- **Đo:** TanStack Devtools, `stale` badge, network dedup.

**3 câu tự vấn:**
1. Phân biệt server vs client vs URL state và để sai thì vỡ gì?
2. `staleTime` vs `gcTime` vs `invalidateQueries` khác nhau?
3. Vì sao không nên lưu `products` trong Redux?

### 24. 🔍 Tháp Kiểm Thử (Testing) ⭐ BỔ SUNG

**Nói nôm na:** Kim tự tháp: đáy to là `Unit` (nhanh <50ms), giữa là `Integration` (RTL+MSW 200ms), đỉnh nhỏ là `E2E` (Playwright 5s). Đừng xây tháp ngược (toàn E2E) sẽ sập (flaky, chậm).

**Gom từ:** 10-161->172 (12 câu bị sót 0/12)

**Nhớ 3 điều:**
1. `RTL` test như người dùng: `getByRole('button', {name:/mua/i})` > `getByTestId`
2. `MSW` chặn ở tầng mạng (đáng tin) khác `vi.mock` chặn module
3. `Playwright` (chạy ngoài, đa tab, trace) hơn `Cypress` (chạy trong), `Vitest` (dùng Vite) nhanh hơn `Jest` 3-10 lần

```js
// Integration: RTL + MSW
render(<ProductList />)
await user.click(screen.getByRole('button', {name:/thêm/i}))
expect(await screen.findByText(/đã thêm/i)).toBeInTheDocument()
```

**Bẫy:** `Snapshot` cho page 500 dòng -> mù, chỉ dùng cho component nhỏ. Đừng ép coverage 100%.

**Hiểu sâu (2 phút):**
- **Vì sao:** Unit cô lập logic, Integration test tương tác + MSW mô phỏng mạng, E2E test luồng thật. RTL ưu tiên role/accessibility.
- **Đừng dùng khi:** Snapshot cho page lớn, E2E cho mọi case → chậm flaky.
- **Vỡ khi nào:** `getByTestId` thay role → miss a11y, `vi.mock` mock lệch, coverage 100% ảo.
- **Đo:** `vitest --coverage`, `msw` log, Playwright trace viewer.

**3 câu tự vấn:**
1. Khi nào dùng Unit vs Integration vs E2E và tỉ lệ tháp?
2. `MSW` vs `vi.mock` khác tầng nào và khi nào dùng cái nào?
3. Vì sao ưu tiên `getByRole` hơn `getByTestId`?

### 25. 🎤 Kể Chuyện STAR (Behavioral) ⭐ BỔ SUNG

**Nói nôm na:** Giám khảo không chấm đúng/sai, chấm cách bạn kể. Dùng khung `STAR + Trade-off + Con số`.

**Gom từ:** 14-187->199 (13 câu bị sót 0/13)

**Công thức 1 phút cho 13 câu:**

`S (cảnh 1 câu + số) -> T (nhiệm vụ của bạn) -> A (3 bước: lắng nghe/data -> quy tắc chung -> thử & đo) -> R (kết quả % ) -> Lesson`

**4 story tủ xoay cho cả 13 câu:**
1. **Conflict:** PR 800 dòng -> chia `Must/Should/Nit` + call 15' -> PR sau tốt 50%
2. **Failure:** Chọn Redux-saga cho app 10 trang -> onboarding 2 tuần -> migrate Zustand velocity +30% -> lesson: "boring solution"
3. **Influence:** CRA 180s -> PoC Vite 25s + RFC + pilot 1 route -> team vote
4. **Incident:** Checkout 500 -> `0-5' confirm -> 5-10' flag/rollback -> 10-20' log -> 20-30' hotfix + post-mortem`

**Template thuyết phục PM:** *"Mình hiểu mục tiêu tăng conversion. Cách A tốn 2 tuần + LCP +1s. Mình đề xuất B 3 ngày + A/B test, nếu CTR tăng mới đầu tư. Anh thấy sao?"*

**Hiểu sâu (2 phút):**
- **Vì sao:** STAR tạo narrative, Trade-off + số chứng minh tư duy senior, không kể lể.
- **Đừng dùng khi:** Kể thiếu số/không có lesson → sáo rỗng; kể lỗi người khác không nhận phần mình.
- **Vỡ khi nào:** Story 5 phút lan man, không khớp câu hỏi, thiếu `R` đo lường.
- **Đo:** Ghi âm 60s, nhờ peer chấm STAR, timer.

**3 câu tự vấn:**
1. Kể 1 lần conflict và cách chia `Must/Should/Nit`?
2. Kể 1 lần chọn sai tech và lesson "boring solution"?
3. Khi PM ép deadline, template thuyết phục bằng data thế nào?

---

## CHECKLIST 7 NGÀY

- [ ] Ngày 1: 1,2,3 (Balo, This, Ngã Tư)
- [ ] Ngày 2: 4,5,6,7 (Dòng Họ, Cửa Khẩu, Thước, Phù Thủy)
- [ ] Ngày 3: 8,9,10,11 (Xưởng Phim, Công Tắc, Tiết Kiệm, Dây Điện)
- [ ] Ngày 4: 12,13,14,15 (Cứu Hộ, Nhà Máy, Dây Chuyền, Biển Số)
- [ ] Ngày 5: 16,17,18,19,20 (Kho, Két, Tủ, Biên Giới, Hợp Đồng)
- [ ] Ngày 6: 21,22,23,24 (Bản Vẽ, Đồng Bộ, Chia Việc, Tháp)
- [ ] Ngày 7: 25 STAR - ghi âm 4 story, mock 30 phút

> Mỗi khái niệm xong tự hỏi: *"Nói 5 phút không bị senior ngắt + trả lời được 3 câu tự vấn chưa?"* Nếu được là đã hiểu thật.

*File này phủ 185/199 câu (bỏ 14 câu Next.js ch.11). Đọc chi tiết từng câu gốc ở `docs/01->10,14`.*
