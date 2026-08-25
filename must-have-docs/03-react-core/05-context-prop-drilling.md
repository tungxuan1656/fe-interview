# Context & Prop Drilling — Broadcast, value memo, split contexts và composition

> Tags: #context #prop-drilling #composition #provider #re-render | Nguồn: `docs/02-react-co-ban.md` câu 40, 41, 42, 43 | Mức: P0

## 1. Định nghĩa chính xác

- **Prop Drilling**: truyền props qua nhiều tầng trung gian không dùng, chỉ để tới leaf. Gây verbose, khó refactor, và trigger re-render lan rộng khi props đổi.
- **Context (React Context API)**: cơ chế **broadcast** giá trị qua cây bằng `createContext(defaultValue)` + `<Provider value={...}>` + `useContext(Context)`. Khi `Provider` value đổi (reference khác), **mọi consumer** của Context đó đều re-render, dù chỉ dùng một field nhỏ — không có selector bail-out mặc định.
- **Composition**: giải quyết drilling bằng cách truyền **element/slot** (`children`, prop là ReactNode) để intermediate không cần biết props. Là pattern explicit, giữ **top-down data flow** nhưng cắt drilling.
- **State Colocation**: đặt state càng gần nơi dùng càng tốt; chỉ lift/composition/Context khi cần share.

## 2. Cơ chế hoạt động

### 2.1 Context broadcast & Fiber

```
<App>
  <ThemeContext.Provider value={value}>   // Fiber có context stack
    <ExpensiveTree />                      // mọi useContext(ThemeContext) subscribe
    </ThemeContext.Provider>
```

- `Provider` lưu `value` trên Fiber, push vào context stack khi traverse. `useContext` đọc stack và **subscribe** — khi `value` thay đổi ( `Object.is` khác ), React **mark tất cả consumer** cần re-render, dù consumer chỉ đọc 1 field.
- `value` là object tạo mới mỗi render (`{theme, setTheme}`) ⇒ reference luôn khác ⇒ consumer luôn re-render dù `theme` không đổi. Cần `useMemo` cho `value`.
- Context không có **selector** như Zustand/Redux (`useSelector` bail-out theo slice). Muốn granular, phải **split contexts** hoặc dùng `use-context-selector` (experimental).
- Context đọc là **đồng bộ** trong render, không async.

### 2.2 Prop Drilling vs Composition vs Context — flow

**Drilling xấu**:
```
App(user) → Layout(user) → Header(user) → Avatar(user)
         drilling qua Layout/Header không dùng user
```

**Composition tốt**: `Layout` chỉ nhận `header` là `ReactNode`, không biết `user`:
```
App → <Layout header={<Header avatar={<Avatar user={user} />} />} />
Layout({header}) → <div>{header}</div> // không drilling
```

**Context tốt** khi data thực sự **global** (theme, auth, locale, i18n) và nhiều leaf cần.

### 2.3 Split contexts pattern

Tách **value** và **setter** (hoặc tách theo tần suất đổi) để consumer chỉ subscribe phần cần:

```ts
const ThemeValueContext = createContext<Theme>("light");
const ThemeSetterContext = createContext<Dispatch<SetStateAction<Theme>>>(() => {});
// Consumer chỉ đọc theme không re-render khi setter identity đổi (hiếm), và ngược lại
// Phổ biến: ValueContext cho data đọc nhiều, SetterContext cho action ít đổi
```

### 2.4 Khi nào Context gây re-render thừa?

- `Provider` bọc quá cao (ở `App`) + `value` thay đổi thường xuyên (input, timer) ⇒ toàn tree re-render.
- Fix: **colocate Provider** gần subtree cần, hoặc split, hoặc chuyển sang external store (Zustand/Jotai) có selector.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Context broadcast — bug re-render do value không memo
import { createContext, useContext, useState, useMemo, memo } from "react";

type Theme = "light" | "dark";
const ThemeContext = createContext<{ theme: Theme; setTheme: (t: Theme) => void } | null>(null);

function AppBad() {
  const [theme, setTheme] = useState<Theme>("light");
  const [count, setCount] = useState(0);
  // ❌ value object mới mỗi render → mọi consumer re-render dù count đổi, theme không đổi
  const value = { theme, setTheme };
  return (
    <ThemeContext.Provider value={value}>
      <ExpensiveTree />
      <button onClick={() => setCount(c => c + 1)}>count {count}</button>
    </ThemeContext.Provider>
  );
}

function AppGood() {
  const [theme, setTheme] = useState<Theme>("light");
  const [count, setCount] = useState(0);
  // ✅ memo value — chỉ đổi khi theme đổi
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return (
    <ThemeContext.Provider value={value}>
      <ExpensiveTree />
      <button onClick={() => setCount(c => c + 1)}>count {count}</button>
    </ThemeContext.Provider>
  );
}

const ExpensiveTree = memo(function ExpensiveTree() {
  console.log("ExpensiveTree render");
  return <Child />;
});
function Child() {
  const ctx = useContext(ThemeContext)!;
  console.log("Child render — theme:", ctx.theme);
  return <div>theme: {ctx.theme}</div>;
}
```

```tsx
// 3.2 Split contexts — consumer chỉ subscribe phần cần
import { createContext, useContext, useState, ReactNode } from "react";

const ThemeValueContext = createContext<Theme>("light");
const ThemeSetterContext = createContext<(t: Theme) => void>(() => {});

function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>("light");
  return (
    <ThemeValueContext.Provider value={theme}>
      <ThemeSetterContext.Provider value={setTheme}>{children}</ThemeSetterContext.Provider>
    </ThemeValueContext.Provider>
  );
}
function ThemedBox() {
  const theme = useContext(ThemeValueContext); // chỉ re-render khi theme đổi
  return <div>theme: {theme}</div>;
}
function ThemeToggle() {
  const setTheme = useContext(ThemeSetterContext); // không re-render khi theme đổi (setter stable)
  return <button onClick={() => setTheme(t => (t === "light" ? "dark" : "light"))}>toggle</button>;
}

// 3.3 Composition vs Context — chọn theo scope
// Drilling xấu
function AppDrilling() {
  const user = { name: "An", avatar: "/a.png" };
  return (
    <LayoutDrilling user={user}>
      <HeaderDrilling user={user} />
    </LayoutDrilling>
  );
}
function LayoutDrilling({ user, children }: { user: any; children: ReactNode }) {
  return <div>{children}</div>; // Layout không dùng user nhưng phải truyền
}
function HeaderDrilling({ user }: { user: any }) {
  return <Avatar user={user} />;
}
function Avatar({ user }: { user: any }) {
  return <img src={user.avatar} alt={user.name} />;
}

// Composition tốt: Layout không biết user
function AppComposed() {
  const user = { name: "An", avatar: "/a.png" };
  return <Layout header={<Header avatar={<Avatar user={user} />} />}>content</Layout>;
}
function Layout({ header, children }: { header: ReactNode; children: ReactNode }) {
  return (
    <div>
      <header>{header}</header>
      <main>{children}</main>
    </div>
  );
}
function Header({ avatar }: { avatar: ReactNode }) {
  return <div>{avatar}</div>;
}

// 3.4 Context cho global thực sự
const UserContext = createContext<{ name: string } | null>(null);
function AppWithContext() {
  const user = { name: "An" };
  return (
    <UserContext.Provider value={user}>
      <Layout header={<HeaderWithContext />}>content</Layout>
    </UserContext.Provider>
  );
}
function HeaderWithContext() {
  return <AvatarWithContext />;
}
function AvatarWithContext() {
  const user = useContext(UserContext)!;
  return <div>{user.name}</div>;
}

// 3.5 Colocation — state gần nơi dùng nhất
function SearchInput({ onSearch }: { onSearch: (q: string) => void }) {
  const [query, setQuery] = useState(""); // local — không lift lên App nếu chỉ Input dùng
  return <input value={query} onChange={e => setQuery(e.target.value)} onBlur={() => onSearch(query)} />;
}
```

## 4. So sánh / Phân loại

| Tiêu chí | Prop Drilling | Composition (`children`/slot) | Context | External Store (Zustand/Jotai) |
|----------|---------------|-------------------------------|---------|-------------------------------|
| Cơ chế | Truyền props qua từng tầng | Truyền `ReactNode` đã render, intermediate không biết data | Broadcast qua Provider/consumer | Store ngoài React, selector |
| Re-render scope | Mọi intermediate re-render khi prop đổi | Chỉ leaf re-render (intermediate không phụ thuộc) | **Mọi consumer** re-render khi value đổi | Chỉ component select slice đổi mới re-render |
| Explicit / implicit | Explicit (rõ ràng) | Explicit | Implicit (khó trace) | Explicit selector |
| Khi dùng | 1-2 tầng, data flow rõ | 3+ tầng nhưng data chỉ leaf cần, không global | Data thực sự global, ít đổi (theme, auth, locale) | State global đổi thường xuyên (cart, filter, realtime) |
| Testability | Dễ (props) | Dễ | Khó hơn (cần Provider mock) | Trung bình |
| Overhead | 0 | 0 | Memo value, split contexts | Thêm lib |

| Context value | Memo | Không memo | Split |
|---------------|------|------------|-------|
| `value={{theme,setTheme}}` inline | — | Consumer re-render mỗi parent render | — |
| `useMemo(() => ({theme,setTheme}), [theme])` | Consumer chỉ re-render khi theme đổi | — | — |
| `ThemeValueContext` + `ThemeSetterContext` | Tốt nhất cho high-frequency | — | Consumer chỉ subscribe nửa cần |

| Khi nào lift vs colocate |
|--------------------------|
| **Lift** khi nhiều sibling cần cùng source of truth (accordion openIndex). |
| **Colocate** khi state chỉ 1 component dùng (input query, dropdown open). |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không đưa mọi thứ vào Context**: Context làm component **implicit dependency**, khó reuse/test, và broadcast re-render rộng. Chỉ 10% state cần global — phần còn lại là local, URL, hoặc server state (TanStack Query).
- **Không dùng Context cho high-frequency update** (mouse move, typing mỗi keystroke, animation): mỗi update re-render mọi consumer ⇒ jank. Dùng `useRef` + subscription, hoặc Zustand/Jotai với selector.
- **Không bọc Provider quá cao khi value hay đổi**: đặt Provider gần subtree cần hơn, hoặc tách Provider thành nhiều tầng.
- **Không drilling quá sâu mà không composition**: 4+ tầng drilling làm refactor đau; composition cắt drilling mà vẫn explicit.
- **Không split quá mức**: tách 10 context nhỏ làm code phân mảnh. Tách khi có **tần suất đổi khác nhau** (value vs setter, theme vs count).
- **Composition trade-off**: hơi verbose khi phải truyền nhiều slot, nhưng explicit và dễ trace hơn Context.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Context value không memo gây re-render toàn tree**
  - Triệu chứng: Profiler thấy mọi consumer render khi parent state không liên quan đổi (count đổi ⇒ theme consumer cũng render).
  - Fix: `useMemo` cho value, hoặc split contexts.
  - Đo: React DevTools Profiler → “Why did this render? Context changed”, highlight updates, `why-did-you-render` log.

- **Lỗi 2: Đặt state thay đổi liên tục vào Context**
  - Triệu chứng: gõ input lag, toàn app re-render.
  - Fix: colocate state vào Input, hoặc dùng `useRef` + `onSubmit`, hoặc Zustand selector.
  - Đo: Profiler commit duration > 16ms khi typing.

- **Lỗi 3: Quên Provider hoặc `undefined` default**
  - Triệu chứng: `useContext` trả `null`/`undefined`, crash `cannot read`.
  - Fix: tạo helper `useTheme()` throw nếu ngoài Provider: `const ctx = useContext(Ctx); if (!ctx) throw new Error("Missing Provider")`.
  - Đo: console error, test wrapper.

- **Lỗi 4: Prop Drilling làm intermediate re-render không cần**
  - Triệu chứng: `Layout` re-render dù chỉ `Avatar` cần `user`.
  - Fix: composition `<Layout header={<Avatar user={user}/>} />`.
  - Đo: Profiler → Layout render dù props không đổi thực sự.

- **Lỗi 5: Over-globalization — mọi thứ vào store/Context**
  - Triệu chứng: bundle lớn, test phải mock nhiều, component khó reuse.
  - Fix: phân loại state — Server (Query), URL (`useSearchParams`), Local (`useState`), Global (Zustand/Context) theo rule Trap 4.
  - Đo: review PR, `madge` dependency graph.

- **Công cụ**:
  - React DevTools Profiler + Components (show context value).
  - `why-did-you-render` track Context.
  - ESLint: `react-hooks/exhaustive-deps` cho value memo, `no-prop-drilling` custom nếu cần.

## 7. Câu hỏi tự kiểm tra

1. Vì sao `value={{theme, setTheme}}` inline làm mọi consumer re-render dù `theme` không đổi? Fix bằng `useMemo` và split contexts thế nào?
2. Khi nào dùng **composition** (`children`/slot) thay vì Context để tránh prop drilling? Cho ví dụ `Layout`/`Header`/`Avatar`.
3. Vì sao không nên đặt state thay đổi liên tục (typing, mouse) vào Context? Giải pháp thay thế là gì theo mức độ global?

<details>
<summary>Đáp án 30s</summary>

1. `{theme, setTheme}` tạo object mới mỗi render ⇒ `Object.is(prevValue, nextValue)` luôn false ⇒ Provider coi như value đổi ⇒ mark mọi consumer re-render. Fix: `const value = useMemo(()=>({theme,setTheme}), [theme])` giữ reference khi theme không đổi. Split: tách `ThemeValueContext` và `ThemeSetterContext` — consumer chỉ đọc value không bị re-render khi setter đổi và ngược lại, giảm phạm vi broadcast.
2. Dùng composition khi data chỉ **leaf** cần nhưng phải qua nhiều tầng trung gian không dùng — thay vì drilling props qua `Layout`/`Header`, truyền sẵn element: `<Layout header={<Header avatar={<Avatar user={user}/>} />}>`. `Layout` chỉ render `header` slot, không biết `user`, giữ explicit và tránh re-render intermediate. Context hợp hơn khi data thực sự **global** và nhiều branch cần (theme, auth).
3. Context broadcast không có selector — mọi consumer re-render khi value đổi, dù chỉ một field đổi. Với high-frequency (mousemov, mỗi keystroke), toàn tree re-render ⇒ lag/INP xấu. Thay thế: colocate state tại leaf (input local), dùng `useRef` + submit, hoặc external store có selector (`Zustand: useStore(s=>s.mouseX)` chỉ re-render khi `mouseX` đổi), hoặc `use-context-selector`.

</details>

---
*Tham khảo chi tiết: `docs/02-react-co-ban.md` — Câu 40, 41, 42, 43. Spec: [React Docs — Context](https://react.dev/learn/passing-data-deeply-with-context), [createContext](https://react.dev/reference/react/createContext), [useContext](https://react.dev/reference/react/useContext), [Composition vs Inheritance](https://react.dev/composition-vs-inheritance).*
