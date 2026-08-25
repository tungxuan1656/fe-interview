# 02. React Cơ Bản - 20 Câu Hỏi Senior

> 20 câu hỏi nền tảng React (Câu 31-50) - từ render, Virtual DOM, reconciliation đến hooks và state management. Hiểu sâu để tối ưu và tránh bug stale closure, re-render thừa.

## Mục lục

- [Câu 31: React render là gì? Khi nào component re-render?](#câu-31-react-render-là-gì-khi-nào-component-re-render)
- [Câu 32: Virtual DOM và Diffing Algorithm](#câu-32-virtual-dom-và-diffing-algorithm)
- [Câu 33: Reconciliation và vai trò của `key`](#câu-33-reconciliation-và-vai-trò-của-key)
- [Câu 34: Controlled vs Uncontrolled Component](#câu-34-controlled-vs-uncontrolled-component)
- [Câu 35: useState - batching và functional update](#câu-35-usestate---batching-và-functional-update)
- [Câu 36: useEffect - lifecycle, dependency và cleanup](#câu-36-useeffect---lifecycle-dependency-và-cleanup)
- [Câu 37: useEffect vs useLayoutEffect](#câu-37-useeffect-vs-uselayouteffect)
- [Câu 38: useMemo vs useCallback vs React.memo](#câu-38-usememo-vs-usecallback-vs-reactmemo)
- [Câu 39: useRef, forwardRef và useImperativeHandle](#câu-39-useref-forwardref-và-useimperativehandle)
- [Câu 40: Context API và vấn đề performance](#câu-40-context-api-và-vấn-đề-performance)
- [Câu 41: Prop Drilling và các giải pháp](#câu-41-prop-drilling-và-các-giải-pháp)
- [Câu 42: Composition, Compound Components và Render Props](#câu-42-composition-compound-components-và-render-props)
- [Câu 43: Lifting State Up vs State Colocation](#câu-43-lifting-state-up-vs-state-colocation)
- [Câu 44: So sánh thư viện state: Redux vs Zustand vs Jotai vs Recoil](#câu-44-so-sánh-thư-viện-state-redux-vs-zustand-vs-jotai-vs-recoil)
- [Câu 45: Redux Toolkit, middleware và RTK Query](#câu-45-redux-toolkit-middleware-và-rtk-query)
- [Câu 46: Xử lý form: Controlled vs React Hook Form](#câu-46-xử-lý-form-controlled-vs-react-hook-form)
- [Câu 47: Synthetic Event trong React](#câu-47-synthetic-event-trong-react)
- [Câu 48: Fragment, Portal và StrictMode](#câu-48-fragment-portal-và-strictmode)
- [Câu 49: Quy tắc Hooks (Rules of Hooks)](#câu-49-quy-tắc-hooks-rules-of-hooks)
- [Câu 50: Hydration mismatch và lỗi key phổ biến](#câu-50-hydration-mismatch-và-lỗi-key-phổ-biến)


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 31: React render là gì? Khi nào component re-render?

**Trả lời Senior:**
Render trong React là quá trình **gọi function component (hoặc render method) để tính ra React Element tree** - object mô tả UI (`{ type, props, key }`). Render **không** đồng nghĩa commit lên DOM. Sau render, React diff với tree cũ rồi mới commit thay đổi (mutation) nếu cần.

Component re-render khi:

1.  **State thay đổi:** `setState`/`useState` setter, `useReducer` dispatch, `this.setState`.
2.  **Props thay đổi:** parent re-render tạo props mới (dù value bằng nhau nhưng reference khác vẫn tính là đổi).
3.  **Parent re-render:** mặc định mọi child đều re-render theo, dù props không đổi (trừ khi memo).
4.  **Context value đổi:** consumer của context đó re-render.
5.  **Hook trigger:** `useSyncExternalStore` hoặc force update.

```jsx
function Parent() {
  const [count, setCount] = React.useState(0);
  console.log('Parent render');
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child /> {/* re-render mỗi khi Parent render, dù không nhận props */}
    </>
  );
}
const Child = React.memo(() => {
  console.log('Child render');
  return <div>Child</div>;
});
```

Lưu ý: `setState` với giá trị `Object.is` bằng cũ (React 18+) sẽ bail out không render lại. Nhưng `setState({})` luôn tạo object mới nên luôn render.

**Trade-off / Khi nào không dùng:** Đừng cố ngăn mọi re-render - render của React rẻ (chỉ JS, chưa chạm DOM). Chỉ tối ưu khi profiler chỉ ra bottleneck. Preview: `React.memo` là shallow compare props, không phải deep.

**Câu hỏi đào sâu:** Render phase có pure không? Vì sao không nên setState trong render? `setState` trong event handler vs trong `setTimeout` batching khác gì (React 17 vs 18)?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 32: Virtual DOM và Diffing Algorithm

**Trả lời Senior:**
Virtual DOM (VDOM) là **object tree** mô tả UI trong memory, rẻ để tạo và so sánh. Khi state đổi, React tạo VDOM mới, **diff** với VDOM cũ để tìm minimal DOM mutations, rồi batch commit. VDOM không làm nhanh hơn DOM thật một cách thần kỳ - nó giúp **declarative** và batch update hiệu quả, tránh layout thrashing khi thao tác DOM thủ công.

Diffing algorithm (Reconciliation) có 2 giả định để đạt O(n):

1.  **Khác type thì rebuild:** `div` -> `span` thì unmount cũ, mount mới (mất state).
2.  **List cần `key`:** để match item giữa 2 render, nếu không có key thì diff theo index (dễ sai).

```jsx
// Giả định 1: type khác -> mất state
{show ? <InputA /> : <InputB />} // khác type -> state input mất
{show ? <Input key="a" /> : <Input key="b" />} // cùng type nhưng key khác -> cũng remount

// Diff list không key: O(n) nhưng sai
// cũ: [A, B, C]  mới: [B, C, A] -> React nghĩ A->B, B->C, C->A -> update hết, mất focus
// có key: React match đúng A, B, C -> chỉ move DOM
```

VDOM còn cho phép render ra nhiều target: DOM (react-dom), Native (react-native), Canvas, PDF... nhờ abstraction.

**Trade-off:** VDOM overhead tạo object mỗi render, với app update rất nhỏ (VD: 60fps animation) thì direct DOM hoặc Svelte (no VDOM) có thể nhanh hơn. Nhưng với app CRUD, VDOM đủ nhanh và DX tốt.

**Câu hỏi đào sâu:** Vì sao React không dùng Proxy để track thay đổi như Vue? Diff O(n) vs O(n³) brute force khác gì?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 33: Reconciliation và vai trò của `key`

**Trả lời Senior:**
Reconciliation là thuật toán so sánh 2 VDOM tree để quyết định reuse/update/move hay unmount/mount. `key` là hint để React **định danh** element giữa các render, đặc biệt trong list.

Không có `key` (hoặc `key={index}`), React diff theo vị trí index. Khi list reorder, thêm/xóa giữa, React sẽ nhầm item, dẫn tới bug state, focus, animation và performance.

```jsx
// Sai: index làm key khi list có reorder
{todos.map((todo, i) => <Todo key={i} todo={todo} />)}
// Thêm item đầu: index của mọi item đổi -> React update hết, input mất focus, checkbox sai

// Đúng: id ổn định
{todos.map(todo => <Todo key={todo.id} todo={todo} />)}

// Key còn dùng để force remount
<ExpensiveForm key={userId} /> // đổi userId -> unmount form cũ, reset state
<EmployeeDetail key={employee.id} /> // pattern hay để reset khi id đổi, thay vì useEffect reset

// Key phải unique trong siblings, không cần global unique
// Key không truyền vào props (props.key === undefined), React dùng riêng
function Item(props) { console.log(props.key); } // undefined!
```

Rule: key phải **stable, predictable, unique**. Dùng `crypto.randomUUID()` khi tạo, không tạo key trong render (`key={Math.random()}` mỗi render khác -> remount liên tục).

**Trade-off:** Key tốt giúp diff O(n) và giữ state, nhưng key thay đổi liên tục gây mount/unmount tốn kém. Đừng dùng index nếu list có thể reorder/filter.

**Câu hỏi đào sâu:** Vì sao `key` không có trong `props`? Khi nào dùng `key` để reset state thay vì `useEffect`?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 34: Controlled vs Uncontrolled Component

**Trả lời Senior:**
- **Controlled:** value được React state quản lý, `value` + `onChange` là single source of truth. Mỗi keystroke đều qua React, dễ validate, format, disable submit.
- **Uncontrolled:** value nằm trong DOM, React lấy qua `ref` hoặc `defaultValue` khi cần. Ít re-render, phù hợp form lớn, file input, hoặc integrate với lib DOM.

```jsx
// Controlled
function Controlled() {
  const [value, setValue] = React.useState("");
  return <input value={value} onChange={e => setValue(e.target.value)} />;
  // mỗi gõ -> setState -> re-render
}

// Uncontrolled
function Uncontrolled() {
  const ref = React.useRef(null);
  const onSubmit = e => {
    e.preventDefault();
    console.log(ref.current.value); // đọc khi submit
  };
  return <form onSubmit={onSubmit}><input defaultValue="hello" ref={ref} /></form>;
}

// Hybrid: controlled nhưng dùng defaultValue ban đầu
function Hybrid({ defaultValue }) {
  const [value, setValue] = React.useState(defaultValue);
  // ...
}

// File input bắt buộc uncontrolled
<input type="file" ref={fileRef} />
```

Trong thực tế, dùng controlled cho form nhỏ cần live validation, uncontrolled + `react-hook-form` cho form lớn để tránh re-render mỗi keystroke.

**Trade-off:** Controlled re-render nhiều (mỗi ký tự), nhưng dễ test và đồng bộ. Uncontrolled ít re-render nhưng khó làm dynamic UI (VD: hiện lỗi ngay khi gõ).

**Câu hỏi đào sâu:** Khi nào `value` vs `defaultValue`? Làm sao chuyển uncontrolled thành controlled mà không bị warning?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 35: useState - batching và functional update

**Trả lời Senior:**
`useState` trả `[state, setState]`. `setState` là async, React batch nhiều `setState` trong cùng tick thành 1 render để tối ưu. Từ React 18, batching tự động cho cả `setTimeout`, `Promise` (trước đó chỉ batch trong event handler).

Functional update `setState(prev => prev + 1)` quan trọng khi new state phụ thuộc old state, vì `setState` closure có thể stale.

```jsx
function Counter() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    // Batching: chỉ render 1 lần, count = 1 (không phải 3)
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);

    // Functional: mỗi updater nhận prev mới nhất -> count = 3
    setCount(c => c + 1);
    setCount(c => c + 1);
    setCount(c => c + 1);
  };

  // React 18 batch cả async
  setTimeout(() => {
    setCount(c => c + 1);
    setCount(c => c + 1); // vẫn batch thành 1 render
  }, 0);

  return <button onClick={handleClick}>{count}</button>;
}

// Lazy initializer: chỉ chạy 1 lần mount
const [data, setData] = React.useState(() => expensiveCalc(props.id));
```

`setState` với `Object.is` bằng cũ sẽ bail out không render. Với object, phải tạo reference mới: `setUser({ ...user, name: "An" })`.

**Trade-off:** Functional update an toàn nhưng hơi verbose. Lazy initializer tránh tính toán lại mỗi render.

**Lỗi thường gặp:** Dùng `setCount(count + 1)` trong loop/timeout bị stale, nghĩ `setState` đồng bộ, mutate state trực tiếp.

**Câu hỏi đào sâu:** React 17 vs 18 batching khác gì? Vì sao `flushSync` tồn tại?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 36: useEffect - lifecycle, dependency và cleanup

**Trả lời Senior:**
`useEffect` là escape hatch để đồng bộ với **hệ ngoài React** (DOM, network, subscription). Nó chạy **sau paint** (commit phase xong), không block render. Cleanup chạy trước effect tiếp theo và khi unmount - nơi để unsubscribe, abort fetch.

Mental model: `useEffect` không phải `componentDidMount` mà là **đồng bộ effect với deps**. Mỗi render có effect riêng, closure riêng.

```jsx
function Chat({ roomId }) {
  React.useEffect(() => {
    const controller = new AbortController();
    async function load() {
      try {
        const res = await fetch(`/api/room/${roomId}`, { signal: controller.signal });
        const data = await res.json();
        setMessages(data);
      } catch (e) {
        if (e.name !== "AbortError") console.error(e);
      }
    }
    load();

    const conn = createConnection(roomId);
    conn.connect();
    return () => {
      controller.abort(); // cancel fetch
      conn.disconnect(); // cleanup subscription
    };
  }, [roomId]); // chỉ re-run khi roomId đổi - exhaustive deps!
}

// Bẫy stale closure
function Counter() {
  const [count, setCount] = React.useState(0);
  React.useEffect(() => {
    const id = setInterval(() => console.log(count), 1000); // closure bắt count ban đầu
    return () => clearInterval(id);
  }, []); // [] nên count luôn 0
  // Fix: để count vào deps hoặc dùng functional update / ref
}
```

ESLint `react-hooks/exhaustive-deps` là bắt buộc - thiếu deps là bug, thừa deps gây loop. Nếu effect không cần re-run, extract logic ra ngoài hoặc dùng `useEvent` (React 19) / `useCallback`.

**Trade-off:** Đừng dùng `useEffect` cho việc có thể tính trong render (derived state) hoặc event handler. `useEffect` là last resort.

**Câu hỏi đào sâu:** Vì sao `useEffect` cleanup chạy trước effect mới? Làm sao fetch mà không race condition khi `roomId` đổi nhanh?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 37: useEffect vs useLayoutEffect

**Trả lời Senior:**
Cả hai đều có signature giống nhau, nhưng thời điểm chạy khác:

- **useEffect:** chạy **bất đồng bộ sau paint**, không block browser paint. Dùng cho đa số side effect (fetch, subscription, log).
- **useLayoutEffect:** chạy **đồng bộ sau DOM mutation nhưng trước paint**, block paint. Dùng khi cần đo DOM và mutate lại trước khi user thấy flicker (tooltip positioning, scroll lock).

```jsx
function Tooltip({ targetRef }) {
  const tooltipRef = React.useRef(null);
  const [left, setLeft] = React.useState(0);

  React.useLayoutEffect(() => {
    const rect = targetRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();
    // Tính vị trí trước khi paint -> không flicker
    setLeft(rect.left + (rect.width - tooltipRect.width) / 2);
  }, [targetRef]);

  return <div ref={tooltipRef} style={{ left }}>Tooltip</div>;
}

// useEffect sẽ flicker: render ở 0 -> paint -> effect tính lại -> paint lại ở vị trí đúng
```

Trong SSR, `useLayoutEffect` warning vì chạy trên server không có DOM - nên thay bằng `useEffect` hoặc dynamic import client-only.

**Trade-off:** `useLayoutEffect` block paint nên nếu tính toán nặng sẽ làm drop frame. Chỉ dùng khi cần tránh visual inconsistency. Mặc định dùng `useEffect`.

**Lỗi thường gặp:** Dùng `useLayoutEffect` cho fetch (block paint vô ích), quên SSR warning, đo DOM trong `useEffect` gây flicker.

**Câu hỏi đào sâu:** Vì sao `useLayoutEffect` block paint? Khi nào dùng `useInsertionEffect` (cho CSS-in-JS)?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 38: useMemo vs useCallback vs React.memo

**Trả lời Senior:**
Cả ba đều là **performance optimization**, không phải semantics. Chỉ dùng khi có vấn đề đo được.

- **React.memo:** HOC memo hóa component - skip re-render nếu props shallow equal. Nhận thêm `areEqual` custom.
- **useMemo:** memo hóa **giá trị** - cache kết quả tính toán nặng giữa các render.
- **useCallback:** memo hóa **function reference** - cache function để con memo không re-render. Bản chất là `useMemo(() => fn, deps)`.

```jsx
const ExpensiveChild = React.memo(function ExpensiveChild({ data, onClick }) {
  console.log("render child");
  return <div onClick={onClick}>{data.value}</div>;
});

function Parent({ items }) {
  const [count, setCount] = React.useState(0);

  // Không memo: mỗi Parent render tạo object/function mới -> child luôn render
  // const data = { value: items.length }; // new object mỗi lần
  // const onClick = () => console.log("click"); // new function mỗi lần

  // Memo: chỉ tạo lại khi items đổi
  const data = React.useMemo(() => ({ value: items.length }), [items]);
  const onClick = React.useCallback(() => console.log("click"), []);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveChild data={data} onClick={onClick} />
    </>
  );
}

// useMemo cho tính toán nặng
const sorted = React.useMemo(() => items.slice().sort((a, b) => a.price - b.price), [items]);
```

**Trade-off:** Memo có cost: memory, so sánh deps, code phức tạp. Nếu child rẻ hoặc props luôn đổi, memo vô ích thậm chí chậm hơn. React Compiler (React 19) sẽ tự memo nên thủ công có thể lỗi thời.

**Lỗi thường gặp:** Memo mọi thứ (over-memoization), deps sai (stale), memo cho primitive (vô nghĩa), nghĩ `useCallback` ngăn tạo function (vẫn tạo nhưng cache).

**Câu hỏi đào sâu:** Khi nào `useMemo` không cần thiết? React Compiler thay đổi gì cho memo thủ công?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 39: useRef, forwardRef và useImperativeHandle

**Trả lời Senior:**
`useRef` trả object `{ current }` mutable, **không gây re-render khi đổi**, và giữ qua các render. 3 use case chính:

1.  Giữ DOM node (`<div ref={ref}>`).
2.  Giữ giá trị mutable không cần render (interval id, previous value).
3.  Giữ reference ổn định để tránh stale closure.

`forwardRef` cho phép parent truyền ref xuống DOM con. `useImperativeHandle` custom lại instance mà parent nhận được qua ref (chỉ expose method cần thiết).

```jsx
function usePrevious(value) {
  const ref = React.useRef();
  React.useEffect(() => { ref.current = value; });
  return ref.current;
}

const FancyInput = React.forwardRef(function FancyInput(props, ref) {
  const inputRef = React.useRef(null);
  React.useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    // chỉ expose focus, không expose cả DOM
    getValue: () => inputRef.current.value
  }), []);
  return <input ref={inputRef} {...props} />;
});

function Parent() {
  const ref = React.useRef(null);
  return (
    <>
      <FancyInput ref={ref} />
      <button onClick={() => ref.current.focus()}>Focus</button>
    </>
  );
}

// Ref không gây render, nên không dùng ref để hiển thị UI
// Sai: ref.current thay đổi không render lại
// Đúng: dùng state nếu cần hiển thị
```

**Trade-off:** `useImperativeHandle` phá declarative, chỉ dùng cho imperative animation, focus, third-party lib. Đừng lạm dụng ref để tránh re-render mà làm UI out-of-sync.

**Lỗi thường gặp:** Đọc `ref.current` trong render (có thể null), quên `forwardRef` nên ref không tới DOM, mutate ref mà mong re-render.

**Câu hỏi đào sâu:** `useRef` vs `createRef` khác gì? Vì sao `ref` không gây re-render?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 40: Context API và vấn đề performance

**Trả lời Senior:**
Context giúp truyền data qua cây mà không prop drilling, qua `createContext` + `Provider` + `useContext`. Khi `Provider` value đổi (reference khác), **mọi consumer** của context đó đều re-render, dù chỉ dùng 1 field nhỏ. Đây là bottleneck chính.

```jsx
const ThemeContext = React.createContext("light");

function App() {
  const [theme, setTheme] = React.useState("light");
  const [count, setCount] = React.useState(0);
  // Sai: value object mới mỗi render -> consumer luôn render dù theme không đổi
  // const value = { theme, setTheme };
  // Đúng: memo value
  const value = React.useMemo(() => ({ theme, setTheme }), [theme]);
  return (
    <ThemeContext.Provider value={value}>
      <ExpensiveTree />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
    </ThemeContext.Provider>
  );
}
function Child() {
  const { theme } = React.useContext(ThemeContext);
  console.log("Child render"); // render khi theme đổi, dù count đổi cũng render nếu value không memo
  return <div>{theme}</div>;
}
```

Giải pháp Senior:

- Tách context: `ThemeValueContext` và `ThemeSetterContext` để consumer chỉ subscribe phần cần.
- Dùng `useMemo` cho value, hoặc selector pattern (thư viện `use-context-selector`).
- Với state phức tạp, dùng Zustand/Jotai thay Context (họ có selector và bail-out tốt).
- React 19 có `useContext` selector proposal nhưng chưa stable.

**Trade-off:** Context đơn giản cho theme, locale, auth, nhưng không hợp cho high-frequency update (store). Đừng đặt state thay đổi liên tục vào Context.

**Câu hỏi đào sâu:** Làm sao tránh re-render khi Context value là object? Khi nào nên thay Context bằng state lib?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 41: Prop Drilling và các giải pháp

**Trả lời Senior:**
Prop drilling là truyền props qua nhiều tầng trung gian không dùng, chỉ để tới leaf. Gây verbose, khó refactor, và re-render không cần thiết.

Giải pháp theo mức độ:

1.  **Composition:** truyền `children` hoặc slot, để parent quyết định render gì, intermediate không cần biết props.
2.  **Context:** cho data thực sự global (theme, user).
3.  **State management lib:** Zustand, Redux cho app state.
4.  **Component composition + render props:** tách logic ra hook.

```jsx
// Drilling xấu
<App><Layout user={user}><Header user={user}><Avatar user={user} /></Header></Layout></App>

// Composition tốt: Layout không cần biết user
function App() {
  const user = useUser();
  return (
    <Layout header={<Header avatar={<Avatar user={user} />} />}>
      <Content />
    </Layout>
  );
}
function Layout({ header, children }) {
  return <div>{header}{children}</div>; // không drilling
}

// Hoặc Context cho user
const UserContext = React.createContext(null);
function Avatar() {
  const user = React.useContext(UserContext);
  return <img src={user.avatar} />;
}

// Hoặc hook
function useUser() { return React.useContext(UserContext); }
```

**Trade-off:** Context và global store làm component khó test và reuse (implicit dependency). Composition giữ explicit nhưng có thể verbose. Chọn theo scope: local -> composition, global -> context/store.

**Lỗi thường gặp:** Đưa mọi thứ vào Context/store làm over-globalization, hoặc drilling quá sâu mà không composition.

**Câu hỏi đào sâu:** Khi nào composition đủ, khi nào cần Context? Làm sao test component dùng Context dễ hơn?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 42: Composition, Compound Components và Render Props

**Trả lời Senior:**
Composition là triết lý core của React: xây UI bằng combine component nhỏ, thay vì inheritance.

- **Compound Components:** nhiều component share ngầm state qua Context, API như `<Select><Select.Option/></Select>`. Linh hoạt, declarative.
- **Render Props:** prop là function `children` hoặc `render` để parent control render, share logic.
- **Children as function:** variant của render props.

```jsx
// Compound Components
const TabsContext = React.createContext(null);
function Tabs({ children, defaultIndex = 0 }) {
  const [index, setIndex] = React.useState(defaultIndex);
  return <TabsContext.Provider value={{ index, setIndex }}>{children}</TabsContext.Provider>;
}
Tabs.List = function List({ children }) { return <div role="tablist">{children}</div>; };
Tabs.Tab = function Tab({ children, tabIndex }) {
  const { index, setIndex } = React.useContext(TabsContext);
  return <button aria-selected={index === tabIndex} onClick={() => setIndex(tabIndex)}>{children}</button>;
};
Tabs.Panels = function Panels({ children }) { return <div>{children}</div>; };
Tabs.Panel = function Panel({ children, tabIndex }) {
  const { index } = React.useContext(TabsContext);
  return index === tabIndex ? <div>{children}</div> : null;
};
// Dùng:
<Tabs><Tabs.List><Tabs.Tab tabIndex={0}>A</Tabs.Tab></Tabs.List><Tabs.Panels><Tabs.Panel tabIndex={0}>Content A</Tabs.Panel></Tabs.Panels></Tabs>

// Render Props
function DataFetcher({ url, children }) {
  const [data, setData] = React.useState(null);
  React.useEffect(() => { fetch(url).then(r => r.json()).then(setData); }, [url]);
  return children(data);
}
<DataFetcher url="/api/user">{data => data ? <User data={data} /> : <Spinner />}</DataFetcher>

// Hook thay thế render props hiện đại
function useFetch(url) { /* ... */ return data; }
```

**Trade-off:** Compound components cần Context, hơi phức tạp. Render props gây nesting hell (callback hell), hook đã thay thế phần lớn. Nhưng compound vẫn rất hay cho design system.

**Câu hỏi đào sâu:** Khi nào dùng compound vs render props vs hook? Làm sao type compound components với TypeScript?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 43: Lifting State Up vs State Colocation

**Trả lời Senior:**
- **Lifting State Up:** đưa state lên parent chung gần nhất để share giữa siblings. Cần khi nhiều component cần cùng source of truth.
- **State Colocation:** đặt state **càng gần nơi dùng càng tốt**, tránh lift quá cao gây re-render rộng và prop drilling.

Nguyên tắc: **Lift khi cần share, colocate khi không.**

```jsx
// Colocation tốt: state chỉ dùng trong 1 component thì ở đó
function SearchInput({ onSearch }) {
  const [query, setQuery] = React.useState(""); // local, không lift lên App
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}

// Lifting khi cần share
function Accordion() {
  const [openIndex, setOpenIndex] = React.useState(0); // lift để chỉ 1 panel mở
  return items.map((item, i) => (
    <Panel key={i} isOpen={i === openIndex} onToggle={() => setOpenIndex(i)} />
  ));
}

// Anti-pattern: lift quá cao
function App() {
  const [inputValue, setInputValue] = React.useState(""); // App không dùng, chỉ truyền xuống Input
  // -> mỗi keystroke App render lại cả tree
  return <Input value={inputValue} onChange={setInputValue} />;
}
// Fix: colocate vào Input, chỉ lift khi submit
function Input({ onSubmit }) {
  const [value, setValue] = React.useState("");
  return <input value={value} onChange={e => setValue(e.target.value)} onBlur={() => onSubmit(value)} />;
}
```

Kent C. Dodds mantra: "State colocation will make your app faster."

**Trade-off:** Colocation giảm re-render nhưng nếu lift thiếu thì phải prop drilling hoặc Context. Cân bằng.

**Câu hỏi đào sâu:** Làm sao quyết định state nên ở đâu? Colocation ảnh hưởng performance thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 44: So sánh thư viện state: Redux vs Zustand vs Jotai vs Recoil

**Trả lời Senior:**
| Lib | Mô hình | Boilerplate | Selector | Khi dùng |
|---|---|---|---|---|
| **Redux (RTK)** | Single store, immutable, reducer, middleware | Nhiều nhưng RTK giảm | `useSelector` với shallowEqual | App lớn, cần time-travel, logic phức tạp, team lớn |
| **Zustand** | Single store như Redux nhưng hook-based, mutable bên trong (immer optional) | Rất ít | `useStore(selector)` - chỉ render khi slice đổi | Thay thế Redux cho 80% case, đơn giản, không cần Provider |
| **Jotai** | Atomic (nhiều atom nhỏ), bottom-up | Ít | `useAtom(atom)` - chỉ atom đó đổi mới render | State derived, dependency graph, tránh re-render tốt, hợp với granular state |
| **Recoil** | Atomic + selector, giống Jotai nhưng có async selector | Trung bình | `useRecoilValue` | Từng hot nhưng ít maintain, Jotai thay thế tốt hơn |

```javascript
// Zustand
import { create } from 'zustand';
const useStore = create(set => ({
  count: 0,
  inc: () => set(s => ({ count: s.count + 1 })),
  bears: []
}));
function Counter() {
  const count = useStore(s => s.count); // chỉ render khi count đổi
  const inc = useStore(s => s.inc);
}

// Jotai
import { atom, useAtom } from 'jotai';
const countAtom = atom(0);
const doubledAtom = atom(get => get(countAtom) * 2);
function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const [doubled] = useAtom(doubledAtom);
}

// Redux Toolkit
const slice = createSlice({ name: 'counter', initialState: { value: 0 }, reducers: { inc: s => { s.value++; } } });
const store = configureStore({ reducer: { counter: slice.reducer } });
function Counter() {
  const value = useSelector(s => s.counter.value);
  const dispatch = useDispatch();
}
```

**Trade-off:** Redux mạnh cho predictability và DevTools, nhưng verbose. Zustand nhẹ nhất. Jotai granular nhất nhưng mental model khác. Recoil hiện ít được khuyên dùng.

**Câu hỏi đào sâu:** Vì sao Zustand không cần Provider? Atomic model giải quyết re-render thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 45: Redux Toolkit, middleware và RTK Query

**Trả lời Senior:**
Redux Toolkit (RTK) là wrapper chuẩn cho Redux, giảm boilerplate 70%: `configureStore` tự setup DevTools + thunk, `createSlice` với Immer cho phép mutate draft, `createAsyncThunk` cho async.

Middleware là chain intercept `dispatch` -> `reducer`: logger, thunk, saga. RTK Query là data fetching + caching built trên Redux, thay thế手 viết thunk + normalize.

```javascript
import { configureStore, createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

// Slice với Immer
const todosSlice = createSlice({
  name: 'todos',
  initialState: { items: [], status: 'idle' },
  reducers: {
    added: (state, action) => { state.items.push(action.payload); } // mutate draft ok
  },
  extraReducers: builder => {
    builder.addCase(fetchTodos.fulfilled, (state, action) => { state.items = action.payload; });
  }
});

export const fetchTodos = createAsyncThunk('todos/fetch', async () => {
  const res = await fetch('/api/todos');
  return res.json();
});

// RTK Query - declarative
export const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: builder => ({
    getTodos: builder.query({ query: () => 'todos', providesTags: ['Todos'] }),
    addTodo: builder.mutation({
      query: body => ({ url: 'todos', method: 'POST', body }),
      invalidatesTags: ['Todos'] // tự refetch
    })
  })
});
export const { useGetTodosQuery, useAddTodoMutation } = api;

const store = configureStore({
  reducer: { todos: todosSlice.reducer, [api.reducerPath]: api.reducer },
  middleware: getDefault => getDefault().concat(api.middleware)
});

// Dùng: const { data, isLoading } = useGetTodosQuery();
```

**Trade-off:** RTK Query tuyệt vời cho CRUD, cache, polling, nhưng nếu app đã dùng React Query/SWR thì trùng. Thunk đủ cho async đơn giản, Saga chỉ khi cần complex flow (cancel, race).

**Lỗi thường gặp:** Mutate state ngoài Immer draft, quên `providesTags` nên không refetch, dispatch thunk mà không handle pending/rejected.

**Câu hỏi đào sâu:** RTK Query vs React Query khác gì? Khi nào cần Saga thay Thunk?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 46: Xử lý form: Controlled vs React Hook Form

**Trả lời Senior:**
Controlled form (state cho mỗi input) đơn giản nhưng mỗi keystroke gây re-render toàn form, với form 20 field sẽ lag. Validation thủ công.

React Hook Form (RHF) dùng **uncontrolled + ref** để tránh re-render, chỉ re-render field lỗi, kết hợp với `resolver` (zod/yup) cho validation schema. Performance vượt trội, DX tốt.

```jsx
// Controlled - lag với form lớn
function ControlledForm() {
  const [form, setForm] = React.useState({ name: "", email: "" });
  return (
    <input value={form.name} onChange={e => setForm({ ...form, name: e.target.value })} />
    // mỗi gõ -> setState -> re-render cả form
  );
}

// React Hook Form - tối ưu
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  name: z.string().min(2, "Tên quá ngắn"),
  email: z.string().email("Email sai")
});

function RHFForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm({
    resolver: zodResolver(schema),
    defaultValues: { name: "", email: "" }
  });
  const onSubmit = async data => {
    await fetch('/api/submit', { method: 'POST', body: JSON.stringify(data) });
  };
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("name")} />
      {errors.name && <p>{errors.name.message}</p>}
      <input {...register("email")} />
      {errors.email && <p>{errors.email.message}</p>}
      <button disabled={isSubmitting}>Submit</button>
    </form>
  );
}
```

RHF còn hỗ trợ `Controller` cho UI lib controlled (MUI, AntD), `useFieldArray` cho dynamic list, `watch` với subscription.

**Trade-off:** RHF thêm dependency, nhưng với form >5 field thì lợi ích rõ. Controlled vẫn ok cho form nhỏ 1-2 field.

**Lỗi thường gặp:** Dùng `watch` làm re-render nhiều (nên dùng `useWatch` isolated), quên `defaultValues` gây uncontrolled warning.

**Câu hỏi đào sâu:** Vì sao RHF nhanh hơn controlled? `Controller` dùng khi nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 47: Synthetic Event trong React

**Trả lời Senior:**
SyntheticEvent là wrapper cross-browser của React cho native event, normalize `e.target`, `e.currentTarget`, `preventDefault`, `stopPropagation` hoạt động giống nhau mọi browser. Trước React 17, React dùng **event pooling**: reuse SyntheticEvent object để giảm GC, nên `e.persist()` cần nếu dùng async. Từ React 17, pooling đã bỏ.

React delegate event lên `root` (React 17+) thay vì `document` để isolate giữa nhiều React root.

```jsx
function Button() {
  const handleClick = e => {
    console.log(e.type); // "click" - synthetic
    console.log(e.nativeEvent); // native event gốc
    e.preventDefault();
    // e.persist() không cần từ React 17

    // Async vẫn ok
    setTimeout(() => console.log(e.target), 0); // vẫn có vì không pooling nữa
  };
  return <button onClick={handleClick}>Click</button>;
}

// StopPropagation trong React vs native
function Parent() {
  const onParentClick = () => console.log("parent");
  const onChildClick = e => {
    e.stopPropagation(); // chỉ chặn trong React tree, không chặn native listener trên document
    console.log("child");
  };
  React.useEffect(() => {
    document.addEventListener("click", () => console.log("document native"));
    return () => document.removeEventListener("click", () => {});
  }, []);
  return <div onClick={onParentClick}><button onClick={onChildClick}>Child</button></div>;
}
```

**Trade-off:** SyntheticEvent thêm abstraction, nhưng trước đây pooling gây confusion. Hiện tại gần như transparent.

**Lỗi thường gặp:** Nghĩ `e.target` luôn là element gắn handler (thực ra là `e.currentTarget`), quên `stopPropagation` không chặn native.

**Câu hỏi đào sâu:** Event pooling là gì và vì sao bỏ? React 17 delegate lên root thay vì document để làm gì?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 48: Fragment, Portal và StrictMode

**Trả lời Senior:**
- **Fragment (`<>` / `<React.Fragment>`):** wrapper không tạo DOM node, để return nhiều sibling mà không thêm `div` thừa. Có thể có `key` khi render list Fragment.
- **Portal (`createPortal`):** render children vào DOM node khác ngoài parent hierarchy, nhưng vẫn giữ React context và event bubbling theo React tree (không phải DOM tree). Dùng cho modal, tooltip, dropdown.
- **StrictMode:** wrapper dev-only để phát hiện side effect không an toàn: double-invoke render, effect, và warn về deprecated API. Không ảnh hưởng production.

```jsx
// Fragment
function List() {
  return (
    <>
      <li>A</li>
      <li>B</li>
    </>
  );
}
// Fragment với key
{items.map(item => <React.Fragment key={item.id}><td>{item.name}</td><td>{item.value}</td></React.Fragment>)}

// Portal
import { createPortal } from 'react-dom';
function Modal({ children }) {
  return createPortal(
    <div className="modal">{children}</div>,
    document.body // render ra body, nhưng event vẫn bubble tới React parent
  );
}
function App() {
  return <div onClick={() => console.log("App click")}><Modal><button>Click</button></Modal></div>;
  // click button -> log "App click" dù DOM button nằm ngoài App div
}

// StrictMode double-invoke trong dev
<React.StrictMode><App /></React.StrictMode>
// useEffect cleanup + re-run để phát hiện quên cleanup
```

**Trade-off:** Portal tiện nhưng cần quản lý focus trap, scroll lock. StrictMode double-invoke có thể làm log double trong dev gây confusion nhưng giúp bắt bug sớm.

**Lỗi thường gặp:** Quên Fragment không nhận prop ngoài `key`, Portal không tự xử lý z-index/focus, tắt StrictMode để tránh double log mà che bug.

**Câu hỏi đào sâu:** Portal event bubbling theo React tree hay DOM tree? StrictMode double-invoke bắt lỗi gì?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 49: Quy tắc Hooks (Rules of Hooks)

**Trả lời Senior:**
2 quy tắc bắt buộc:

1.  **Chỉ gọi hooks ở top-level:** không trong `if`, `for`, nested function. Vì React lưu hooks theo **thứ tự gọi** trong linked list, mỗi render phải gọi cùng số lượng và thứ tự.
2.  **Chỉ gọi từ React function:** component hoặc custom hook, không từ function thường.

Vi phạm sẽ làm hooks mismatch, state sai.

```jsx
// Sai: condition hook
function Bad({ cond }) {
  if (cond) {
    const [count, setCount] = React.useState(0); // thứ tự đổi khi cond đổi -> bug
  }
}

// Đúng: luôn gọi, condition bên trong
function Good({ cond }) {
  const [count, setCount] = React.useState(0);
  React.useEffect(() => {
    if (cond) { /* ... */ }
  }, [cond]);
}

// Sai: hook trong loop
items.forEach(item => {
  const [value, setValue] = React.useState(item); // sai thứ tự
});
// Đúng: tách component
items.map(item => <Item key={item.id} item={item} />)
function Item({ item }) {
  const [value, setValue] = React.useState(item); // mỗi Item có hooks riêng
}

// Custom hook phải bắt đầu bằng use
function useCounter(initial) {
  const [count, setCount] = React.useState(initial);
  const inc = React.useCallback(() => setCount(c => c + 1), []);
  return [count, inc];
}
```

ESLint `eslint-plugin-react-hooks` enforce tự động. React Compiler sẽ nới lỏng phần nào nhưng hiện vẫn phải tuân thủ.

**Trade-off:** Quy tắc làm code hơi verbose (không early return trước hooks), nhưng đảm bảo determinism.

**Lỗi thường gặp:** Gọi hook sau `if (loading) return`, hook trong `map` trực tiếp, quên custom hook phải là `useX`.

**Câu hỏi đào sâu:** Vì sao React lưu hooks theo thứ tự chứ không theo tên? Điều gì xảy ra nếu vi phạm?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 50: Hydration mismatch và lỗi key phổ biến

**Trả lời Senior:**
Hydration là khi client React "attach" vào HTML đã SSR, so sánh VDOM với DOM sẵn có. **Mismatch** xảy ra khi HTML SSR khác HTML client render lần đầu (do `Date.now()`, `Math.random()`, `window` check, locale...), React sẽ warn và có thể discard SSR HTML, gây mất lợi ích SSR và layout shift.

Lỗi `key` phổ biến: thiếu `key`, `key={index}` khi reorder, `key` random mỗi render.

```jsx
// Hydration mismatch - sai
function Clock() {
  return <div>{new Date().toLocaleString()}</div>; // server và client khác giây -> mismatch
}
// Fix: chỉ render dynamic sau mount
function ClockFixed() {
  const [now, setNow] = React.useState(null);
  React.useEffect(() => setNow(new Date().toLocaleString()), []);
  if (!now) return <div>Loading...</div>; // SSR fallback
  return <div>{now}</div>;
}

// Hoặc suppressHydrationWarning cho phần cố ý khác
<div suppressHydrationWarning>{Date.now()}</div>

// Key lỗi
{todos.map((todo, index) => <Todo key={index} />)} // sai khi reorder
{todos.map(todo => <Todo key={Math.random()} />)} // sai: mỗi render key mới -> remount
// Đúng
{todos.map(todo => <Todo key={todo.id} />)}

// Key phải stable: tạo id khi tạo data, không khi render
const newTodo = { id: crypto.randomUUID(), text };
```

Debug hydration: `onRecoverableError` trong `hydrateRoot`, check `suppressHydrationWarning`.

**Trade-off:** `suppressHydrationWarning` chỉ nên dùng cho text content, không cho structure. Tốt nhất là đảm bảo SSR và CSR render giống nhau lần đầu.

**Lỗi thường gặp:** Dùng `window` trong render mà không guard `typeof window !== "undefined"`, `key` index, `Date`/`random` trong render.

**Câu hỏi đào sâu:** Hydration mismatch ảnh hưởng SEO và performance thế nào? Vì sao `key={Math.random()}` tệ hơn không có key?

[↑ Quay lại Mục lục](#mục-lục)
---
