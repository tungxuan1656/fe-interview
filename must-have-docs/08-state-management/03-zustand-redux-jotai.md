# Zustand / Redux Toolkit / Jotai — Single store + Selector shallow, RTK Immer & DevTools, Atomic vs Recoil

> Tags: #Zustand #Redux-Toolkit #Jotai #Recoil #Selector #Immer #DevTools | Nguồn: `docs/09-state-management.md` câu 151-154 + `docs/04-frontend-architecture.md` câu 75-76 | Mức: P0

## 1. Định nghĩa chính xác

**Zustand** là **single store hook-based** (~1KB) với `create<State>()((set,get)=>({ ... }))`, không cần `<Provider>`, re-render tối ưu qua **per-component selector** `useStore(s => s.x)` + **equality check** (`Object.is` mặc định, `shallow` cho object). **Redux Toolkit (RTK)** là chuẩn Redux hiện tại: `configureStore` + `createSlice({ name, initialState, reducers })` với **Immer** bên trong cho phép `state.value = ...` (mutate draft, sinh new state immutable), kèm **DevTools time-travel**, **middleware/thunk** (`createAsyncThunk`) và check `serializable/immutable`. **Jotai** là **atomic model**: mỗi `atom(initial)` là unit trạng thái độc lập, đọc qua `useAtom(atom)` / `useAtomValue`, ghi qua `useSetAtom`, chỉ component subscribe atom đó mới re-render. **Recoil** là tiền thân atomic (Facebook) với `atom`, `selector`, `RecoilRoot`, nay ít dùng hơn Jotai do bundle/API nhỏ gọn hơn.

## 2. Cơ chế hoạt động

- **Zustand — single store + selector**:
  - `create` trả hook `useCart`. `set((state)=>({ items: [...state.items, item] }))` merge shallow, `get()` đọc state hiện tại trong action (tránh stale closure).
  - Mỗi component `useCart(s => s.items.length)` **subscribe riêng** qua selector; Zustand so `Object.is(prev, next)` — nếu `length` không đổi → không re-render dù `items` ref khác. Với object selector cần `shallow`: `useCart(s => ({ count: s.items.length, total: s.total }), shallow)`.
  - Middleware: `devtools`, `persist` (`localStorage`), `immer`, `subscribeWithSelector`. Không cần Provider trừ khi cần **context isolation** (test/SSR).
  ```
  create → store { getState, setState, subscribe, getInitialState }
  useCart(selector) → subscribe(selector(state)) → equality check → render
  ```

- **Redux Toolkit — Immer + DevTools + middleware/thunk vs RTK Query**:
  - `createSlice` tự sinh `actions` + `reducer`, `reducers: { setUser: (state, action)=>{ state.value = action.payload } }` — Immer `produce` draft.
  - `configureStore({ reducer: { user: userSlice.reducer }, middleware })` tự bật `redux-thunk`, DevTools, `immutableCheck`/`serializableCheck`.
  - `createAsyncThunk('user/fetch', async (id, { signal })=>fetch(...))` + `extraReducers: builder.addCase(fetchUser.fulfilled, ...)` cho async. Component `useSelector((s:RootState)=>s.user.value)` + `useDispatch<AppDispatch>()`.
  - **RTK Query** là **server cache** tách biệt store: `createApi({ reducerPath, baseQuery, endpoints: { getProducts: builder.query<Product[], void>({ query: ()=>'products' }) } })` — cung cấp `useGetProductsQuery` với `staleTime/cache` riêng, không lẫn client state. Nếu dùng RTK Query thì không cần TanStack Query cho server, và ngược lại.
  - Re-render: `useSelector` dùng `===` mặc định, cần `shallowEqual` hoặc `createSelector` (reselect) để memo.

- **Jotai atomic vs Recoil**:
  - Jotai: `const countAtom = atom(0); const doubledAtom = atom(get => get(countAtom)*2)` (derived read-only). `useAtom(countAtom)` re-render **chỉ khi atom đó đổi**. Không có single store, không cần reducer.
  - Recoil: `atom({ key, default })` + `selector({ key, get })` + `<RecoilRoot>`, `useRecoilState/useRecoilValue`. Khác Jotai: Recoil bắt buộc `key` string duy nhất, `RecoilRoot` Provider, `selector` async hỗ trợ Suspense. Jotai nhẹ hơn, không key, không Provider mặc định.
  - Cả hai đều **atomic → re-render tối thiểu**: update `priceAtom` không làm component chỉ dùng `nameAtom` re-render.

- **Re-render comparison**:
  - **Context**: `useContext` re-render **tất cả consumer** khi `value` đổi, dù chỉ 1 field đổi — không có selector.
  - **Zustand/Jotai**: per-selector/atom — chỉ consumer subscribe field đổi mới re-render.
  - **Redux**: `useSelector` per-slice + `shallowEqual`/`createSelector` để đạt tương tự.

```
Zustand: component A useCart(s=>s.items.length) — items push → length 2→3 → A render; component B useCart(s=>s.add) — chỉ lấy fn → không render
Jotai:   component A useAtom(priceAtom) — priceAtom set → A render; component B useAtom(nameAtom) → không render
Context: <CartContext.Provider value={{ items, add }}> — items đổi → mọi useContext đều render
Redux:   useSelector(s=>s.cart.items) — items đổi → selector return new array → render; cần createSelector để memo
```

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Zustand — single store + selector shallow + persist + devtools
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { shallow } from 'zustand/react/shallow';

type Item = { id: string; price: number };
type CartState = {
  items: Item[];
  add: (item: Item) => void;
  remove: (id: string) => void;
  total: () => number; // derived via get()
};

export const useCart = create<CartState>()(
  devtools(
    persist(
      (set, get) => ({
        items: [],
        add: item => set(state => ({ items: [...state.items, item] })),
        remove: id => set(state => ({ items: state.items.filter(i => i.id !== id) })),
        total: () => get().items.reduce((sum, i) => sum + i.price, 0),
      }),
      { name: 'cart-storage' } // persist localStorage — hybrid cart
    ),
    { name: 'cart' } // DevTools label
  )
);

// Selector tối ưu — chỉ re-render khi giá trị selector đổi
function CartCount() {
  const count = useCart(s => s.items.length); // primitive → Object.is đủ
  return <div>{count}</div>;
}
function AddButton({ item }: { item: Item }) {
  const add = useCart(s => s.add); // chỉ lấy fn, không re-render khi items đổi
  return <button onClick={() => add(item)}>Thêm</button>;
}
function CartSummary() {
  // object selector cần shallow — không shallow thì mỗi render tạo object mới → luôn re-render
  const { count, total } = useCart(s => ({ count: s.items.length, total: s.total() }), shallow);
  return <div>{count} — {total}</div>;
}
```

```tsx
// 3.2 Redux Toolkit — Immer + configureStore + createAsyncThunk + DevTools + RTK Query phân biệt
import { createSlice, configureStore, createAsyncThunk } from '@reduxjs/toolkit';
import { useSelector, useDispatch } from 'react-redux';
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

type User = { id: string; name: string };

// Async thunk — thunk vs createAsyncThunk: createAsyncThunk tự sinh pending/fulfilled/rejected
export const fetchUser = createAsyncThunk('user/fetch', async (id: string, { signal }) => {
  const res = await fetch(`/api/users/${id}`, { signal }).then(r=>r.json());
  return res as User;
});

const userSlice = createSlice({
  name: 'user',
  initialState: { value: null as User | null, status: 'idle' as 'idle'|'loading'|'failed' },
  reducers: {
    setUser: (state, action) => { state.value = action.payload; }, // Immer cho phép mutate draft
    logout: state => { state.value = null; },
  },
  extraReducers: builder => {
    builder.addCase(fetchUser.pending, state => { state.status = 'loading'; });
    builder.addCase(fetchUser.fulfilled, (state, action) => { state.status = 'idle'; state.value = action.payload; });
    builder.addCase(fetchUser.rejected, state => { state.status = 'failed'; });
  },
});
export const { setUser, logout } = userSlice.actions;

export const store = configureStore({
  reducer: { user: userSlice.reducer },
  // tự có thunk, DevTools, immutable/serializable check
});
type RootState = ReturnType<typeof store.getState>;
type AppDispatch = typeof store.dispatch;

function Profile() {
  const user = useSelector((s: RootState) => s.user.value); // cần createSelector nếu derive phức tạp
  const dispatch = useDispatch<AppDispatch>();
  return <button onClick={() => dispatch(fetchUser('123'))}>{user?.name}</button>;
}

// RTK Query — server state, tách khỏi client slice (thay TanStack Query nếu chọn RTK)
export const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Products'],
  endpoints: builder => ({
    getProducts: builder.query<Product[], string | void>({
      query: (category) => `products?category=${category ?? 'all'}`,
      providesTags: ['Products'],
    }),
  }),
});
// useGetProductsQuery('shoes') — có cache, polling, invalidate via invalidatesTags
```

```tsx
// 3.3 Jotai atomic — re-render tối thiểu, không Provider
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';

const countAtom = atom(0);
const priceAtom = atom(0);
const totalAtom = atom(get => get(countAtom) * get(priceAtom)); // derived

function Counter() {
  const [count, setCount] = useAtom(countAtom); // chỉ render khi countAtom đổi
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
function Price() {
  const price = useAtomValue(priceAtom); // chỉ render khi priceAtom đổi
  return <div>{price}</div>;
}
function Total() {
  const total = useAtomValue(totalAtom); // re-render khi count hoặc price đổi
  return <div>{total}</div>;
}
// Recoil tương đương (cần key + RecoilRoot) — Jotai bỏ key/Provider nên gọn hơn
// const countState = atom({ key: 'count', default: 0 }); // Recoil
```

## 4. So sánh / Phân loại

| Tiêu chí | **Zustand** | **Redux Toolkit** | **Jotai** | **Recoil** | **Context** |
|----------|-------------|-------------------|-----------|------------|-------------|
| **Mô hình** | Single store, hook | Single store, action→reducer (Immer) | Atomic (`atom`) | Atomic + `selector` + `key` | DI, không phải store |
| **Provider** | Không (optional) | Bắt buộc `<Provider>` | Không (optional) | Bắt buộc `<RecoilRoot>` | Bắt buộc |
| **Boilerplate** | Thấp (1 file) | Trung bình (slice, thunk) | Thấp | Trung bình (key, selector) | Thấp nhưng thiếu selector |
| **Re-render** | Per-selector `Object.is` / `shallow` | Per-`useSelector` + `createSelector/shallowEqual` | Per-atom — tối thiểu | Per-atom/selector | Tất cả consumer |
| **DevTools** | `devtools` middleware (time-travel hạn chế) | Mạnh nhất (time-travel, action log) | `jotai-devtools` | DevTools cơ bản | Không |
| **Middleware** | Tự viết | Nhiều (`thunk`, `saga`, `listener`) | `atomWith*` | `selector` async | Không |
| **Persist** | `persist` middleware | `redux-persist` | `atomWithStorage` | `recoil-persist` | Tự `useEffect` |
| **Bundle** | ~1KB | ~12KB (+ RTK Query ~8KB) | ~3KB | ~18KB | 0 |
| **Học** | Thấp | Cao (Immer, thunk, selector) | Thấp | Trung bình | Thấp |

| Khi nào chọn | Zustand | Redux RTK | Jotai | TanStack Query / RTK Query |
|--------------|---------|-----------|-------|----------------------------|
| **Team nhỏ Startup, 3-5 dev, cần nhanh** | ✅ | — | ✅ nếu state nhiều mảnh nhỏ | ✅ cho server state |
| **Team lớn 10-20 dev, cần convention chặt, audit action** | — (dễ vô kỷ luật) | ✅ (ép action/reducer/DevTools) | — | — |
| **Frequent update, nhiều atom rời rạc** (`price`, `quantity` riêng) | — | — | ✅ | — |
| **Server state (`products`, `orders`)** | ❌ | ❌ (trừ RTK Query) | ❌ | ✅ |
| **Đã Redux** | — | Dùng **RTK** bắt buộc, không Redux cũ | — | RTK Query nếu đã Redux |

> Lưu ý **thunk vs RTK Query**: `createAsyncThunk` là **client async** (tự lo `pending/fulfilled`), `RTK Query` là **server cache** (tự lo `stale/cache/dedup/polling` như TanStack Query). Không nên tự `createAsyncThunk` cho server list nếu đã có RTK Query/TanStack Query.

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng Zustand khi cần time-travel/audit nghiêm ngặt**: Zustand `set` rải rác, không có action log chuẩn → khó trace "ai set lúc nào". Redux RTK với `action type` + DevTools time-travel phù hợp team lớn, compliance.
- **Không dùng Redux khi app vừa/nhỏ**: boilerplate slice/thunk/selector làm chậm 30% velocity, junior dễ sai `useSelector` không memo → re-render. Zustand/Jotai gọn hơn.
- **Không dùng Jotai khi state là object lớn liền khối**: `atom({ user, cart, theme })` mất lợi atomic, update 1 field re-render hết. Jotai chỉ lợi khi **chia nhỏ atom** (`userAtom`, `cartAtom`, `themeAtom`).
- **Không dùng Context cho frequent state**: `cart`, `filter` đổi liên tục → mọi `useContext` re-render. Context chỉ cho `theme/locale/config` ít đổi, hoặc tách `Value`/`Actions` Context.
- **Không dùng Recoil cho dự án mới nếu không có legacy**: Jotai thay thế gọn hơn (không `key`, không `RecoilRoot`), bundle nhỏ, API tương tự. Recoil còn experimental, ít update.
- **Không lưu Server State vào bất kỳ client store nào** (Zustand/Redux/Jotai): dù selector tốt đến đâu vẫn thiếu `staleTime/gcTime/dedup/background refetch`. Dùng TanStack Query hoặc RTK Query.
- **Không selector object không `shallow`**: `useCart(s => ({ a: s.a, b: s.b }))` mỗi render tạo object mới → `Object.is` luôn `false` → re-render thừa. Dùng `shallow` hoặc tách thành 2 selector.
- **Chi phí**: thêm 1 store là thêm 1 mental model; thống nhất **1 client store** (Zustand **hoặc** Redux **hoặc** Jotai) + **1 server cache** (TanStack **hoặc** RTK Query), không trộn 3 store.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Zustand selector object không `shallow` → re-render thừa**
  - Triệu chứng: `CartSummary` render mỗi lần bất kỳ `useCart.set` nào, dù `count/total` không đổi.
  - Fix: `useCart(s => ({ count: s.items.length }), shallow)` hoặc `useShallow` (v4) / tách 2 `useCart(s=>s.x)`.
  - Đo: React DevTools Profiler → highlight updates; **why-did-you-render** `trackAllPureComponents: true` → log `different objects`.

- **Lỗi 2: Redux `useSelector` không memo → re-render + selector recompute**
  - Triệu chứng: `useSelector(s => s.products.filter(...))` tạo array mới mỗi lần → luôn re-render.
  - Fix: `createSelector` (reselect) `const selectFiltered = createSelector([s=>s.products], p=>p.filter(...))`.
  - Đo: Profiler, WDYR; Redux DevTools → state diff; `reselect` hit/miss.

- **Lỗi 3: Mutate ngoài Immer trong RTK reducer → state không đổi hoặc bug**
  - Triệu chứng: `state.items.push(item)` ngoài `createSlice` (không có Immer) làm mutate gốc.
  - Fix: trong `createSlice` reducer thì `state.value = ...` (Immer draft) OK; ngoài reducer dùng `return { ...state, items: [...state.items, item] }`.
  - Đo: Redux DevTools → action không tạo new state reference; `immutableCheck` middleware warn.

- **Lỗi 4: Zustand `set` rải rác không convention → không trace**
  - Triệu chứng: 10 nơi `useCart.setState(...)` lung tung, bug không biết ai sửa.
  - Fix: gom action trong `create` (`add/remove`), bật `devtools` middleware, hoặc chuyển Redux nếu team lớn.
  - Đo: Zustand DevTools → action name; grep `set(` count.

- **Lỗi 5: Jotai atom quá lớn → mất lợi atomic**
  - Triệu chứng: `const appAtom = atom({ count, price, theme })` — đổi `theme` làm `Counter` re-render.
  - Fix: tách `countAtom`, `priceAtom`, `themeAtom`, `totalAtom = atom(get => get(countAtom)*get(priceAtom))`.
  - Đo: Profiler — component không liên quan vẫn render; tách atom → hết.

- **Lỗi 6: Trộn Server State vào client store → stale/không dedup**
  - Triệu chứng: `products` trong Zustand, 2 component fetch riêng, không share cache.
  - Fix: `useQuery(['products'])` cho server, Zustand chỉ `cart/ui`.
  - Đo: Network → duplicate fetch; TanStack DevTools vs Zustand DevTools — cache không sync.

- **Công cụ**:
  - **Zustand DevTools** (`devtools` middleware) + **Redux DevTools Extension** — time-travel, action log, state diff.
  - **Jotai DevTools** — atom graph.
  - **React DevTools Profiler + why-did-you-render** — re-render do selector missing/`shallow`, Context vs atomic.
  - **`useShallow` / `shallow` / `createSelector`** — fix re-render.

## 7. Câu hỏi tự kiểm tra

1. Zustand selector + `shallow` hoạt động thế nào để giảm re-render, và vì sao object selector không `shallow` luôn re-render? So với `useContext` và Redux `useSelector + createSelector`?
2. Redux Toolkit khác Redux cũ thế nào (Immer, `createSlice`, `configureStore`, DevTools) và `createAsyncThunk` (thunk) khác **RTK Query** thế nào — khi nào dùng cái nào?
3. Jotai atomic vs Recoil khác gì (key/Provider/bundle) và khi nào chọn Zustand (single store) vs Jotai (atomic) vs Redux (team lớn) — tuyệt đối không lưu gì vào client store?

<details>
<summary>Đáp án 30s</summary>

1. **Zustand** `useCart(selector)` subscribe riêng, so `Object.is(prev, next)`; primitive (`s.items.length`) đủ `Object.is`, object (`{ count, total }`) mỗi render tạo ref mới → `Object.is` luôn false → re-render thừa → phải `shallow` (so nông từng key). **Context** không có selector — mọi `useContext` re-render khi `value` đổi. **Redux** `useSelector` cũng `===` mặc định, object/array mới mỗi lần → cần `shallowEqual` hoặc `createSelector` (reselect memo) để giữ ref. **Jotai** per-`atom` nên tự tối thiểu, không cần `shallow`.
2. **RTK** thay Redux cũ: `createSlice` tự sinh `actions+reducer`, **Immer** cho `state.value = ...` (draft), `configureStore` tự bật `thunk` + **DevTools time-travel** + `immutable/serializable check`, giảm 70% boilerplate. **`createAsyncThunk`** là client async (pending/fulfilled/rejected) cho 1 flow; **RTK Query** là **server cache** (`createApi` + `builder.query`) có `cache/dedup/polling/invalidation` như TanStack Query — dùng RTK Query/TanStack cho server list, `createAsyncThunk` chỉ khi cần async client đơn lẻ.
3. **Jotai** `atom(initial)` không `key`, không `Provider` mặc định, `~3KB`, derived `atom(get=>...)`; **Recoil** `atom({ key, default })` + `selector` + `<RecoilRoot>`, `~18KB`, hỗ trợ Suspense — Jotai gọn hơn cho dự án mới. Chọn **Zustand** khi state khối liền (`cart`, `ui`) team nhỏ cần nhanh; **Jotai** khi nhiều mảnh rời rạc frequent update (`price`/`quantity` riêng); **Redux RTK** khi team lớn cần convention + time-travel audit. **Không lưu Server State** (`products`, `orders`, `search results`) vào bất kỳ client store — dùng **TanStack Query** hoặc **RTK Query**.

</details>

---
*Tham khảo chi tiết: `docs/09-state-management.md` — Câu 151-154, `docs/04-frontend-architecture.md` — Câu 75-76. Liên quan: `01-state-classification.md` (đặt state đúng nơi), `02-tanstack-query.md` (server cache), `04-url-form-state.md` (URL/Form). Spec: [Zustand](https://docs.pmnd.rs/zustand), [Redux Toolkit](https://redux-toolkit.js.org/), [Jotai](https://jotai.org/), [Recoil](https://recoiljs.org/).*
