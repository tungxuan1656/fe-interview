# 09. State Management - 10 Câu Hỏi Senior

> 10 câu hỏi quản lý state (Câu 151-160) - từ Redux, Zustand, Context đến server state và TanStack Query. Senior không hỏi "dùng Redux hay Zustand" mà hỏi "state này thuộc loại nào và nên ở đâu".

## Mục lục

- [Câu 151: Redux giải quyết gì? Vì sao cần?](#câu-151-redux-giải-quyết-gì-vì-sao-cần)
- [Câu 152: Redux Toolkit vs Redux cũ - khác gì?](#câu-152-redux-toolkit-vs-redux-cũ---khác-gì)
- [Câu 153: Zustand vs Redux - khi nào chọn gì?](#câu-153-zustand-vs-redux---khi-nào-chọn-gì)
- [Câu 154: Context có thay Redux được không?](#câu-154-context-có-thay-redux-được-không)
- [Câu 155: Server State vs Client State - phân biệt](#câu-155-server-state-vs-client-state---phân-biệt)
- [Câu 156: TanStack Query là gì? useQuery và useMutation](#câu-156-tanstack-query-là-gì-usequery-và-usemutation)
- [Câu 157: Cache Invalidation - khi nào invalidate?](#câu-157-cache-invalidation---khi-nào-invalidate)
- [Câu 158: staleTime vs gcTime (cacheTime) - chi tiết](#câu-158-staletime-vs-gctime-cachetime---chi-tiết)
- [Câu 159: Thiết kế state cho ecommerce lớn - phân loại](#câu-159-thiết-kế-state-cho-ecommerce-lớn---phân-loại)
- [Câu 160: State nào global, local, server? User/Cart/Product/Filter/Search/Order/Notification/UI](#câu-160-state-nào-global-local-server-usercartproductfiltersearchordernotificationui)


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 151: Redux giải quyết gì? Vì sao cần?

**Trả lời Senior:**
Redux giải quyết **3 nỗi đau** khi app lớn: **prop drilling**, **state chia sẻ không nhất quán**, và **khó debug/trace** khi state thay đổi từ nhiều nơi.

Khi app nhỏ, `useState` + props đủ. Khi lớn (50 component, 10 feature), state như `user`, `cart`, `theme` cần ở khắp nơi, truyền qua 5 cấp props là địa ngục, và 2 component cùng sửa `cart` làm race, không biết ai sửa lúc nào.

Redux cho **single source of truth**: 1 store duy nhất, state là object immutable, chỉ thay đổi qua **action** (`{ type, payload }`) → **reducer** (pure function `(state, action) => newState`) → **subscribe** update UI. Luồng **predictable**, **time-travel debug**, và **middleware** (thunk, saga) xử lý async.

Cơ chế: `store.dispatch(action)` → reducer tính state mới → `store.subscribe` báo React re-render. Với `react-redux`, `useSelector` subscribe đúng slice, `useDispatch` gửi action.

```typescript
// Vấn đề không Redux: prop drilling
function App() { const [user, setUser] = useState(null); return <Header user={user} />; }
function Header({ user }: Props) { return <Nav user={user} />; }
function Nav({ user }: Props) { return <Avatar user={user} />; } // 3 cấp chỉ để truyền user

// Với Redux: bất kỳ component nào cũng lấy được
import { createStore } from 'redux';
type State = { user: User | null; cart: Item[] };
type Action = { type: 'SET_USER'; payload: User } | { type: 'ADD_TO_CART'; payload: Item };

function reducer(state: State = { user: null, cart: [] }, action: Action): State {
  switch (action.type) {
    case 'SET_USER': return { ...state, user: action.payload };
    case 'ADD_TO_CART': return { ...state, cart: [...state.cart, action.payload] };
    default: return state;
  }
}
const store = createStore(reducer);
store.dispatch({ type: 'SET_USER', payload: { id: '1', name: 'An' } });

// React
import { useSelector, useDispatch } from 'react-redux';
function Avatar() {
  const user = useSelector((s: State) => s.user); // subscribe đúng slice
  return <div>{user?.name}</div>;
}
function AddButton({ item }: { item: Item }) {
  const dispatch = useDispatch();
  return <button onClick={() => dispatch({ type: 'ADD_TO_CART', payload: item })}>Thêm</button>;
}
```

**Trade-off:** Redux thêm boilerplate, complexity, và re-render nếu selector không memo. Chỉ dùng khi **state share nhiều nơi + cần trace**. Với app nhỏ, `Context` + `useReducer` hoặc `Zustand` gọn hơn.

**Câu hỏi đào sâu:** Vì sao reducer phải pure và immutable? Redux khác MobX (observable) thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 152: Redux Toolkit vs Redux cũ - khác gì?

**Trả lời Senior:**
Redux cũ (2015) nổi tiếng **boilerplate**: `action type string`, `action creator`, `switch reducer`, `combineReducers`, `createStore`, `applyMiddleware(thunk)`, `connect(mapStateToProps)` - 5 file cho 1 feature, dễ sai immutable (`state.user.name = ...` mutate).

**Redux Toolkit (RTK)** là **chuẩn hiện tại**, wrap Redux core, giảm 70% code, và fix 3 lỗi lớn:

1.  **Immutability tự động**: dùng **Immer** bên trong `createSlice`, cho phép viết `state.user.name = 'An'` (thực ra là draft, Immer sinh new state).
2.  **Boilerplate gộp**: `createSlice({ name, initialState, reducers })` tự sinh `actions` + `reducer`, không cần type string thủ công.
3.  **Async chuẩn**: `createAsyncThunk` + `extraReducers` thay vì tự viết thunk, và **RTK Query** (tích hợp) cho server state.
4.  **DevTools + middleware**: `configureStore` tự bật DevTools, thunk, check immutable/serializable.

```typescript
// ❌ Redux cũ - 3 file
// actions.ts
const SET_USER = 'SET_USER';
export const setUser = (user: User) => ({ type: SET_USER, payload: user });
// reducer.ts
function userReducer(state = null, action: any) { switch(action.type){ case SET_USER: return action.payload; default: return state; } }
// store.ts
import { createStore, combineReducers, applyMiddleware } from 'redux';
const store = createStore(combineReducers({ user: userReducer }), applyMiddleware(thunk));

// ✅ Redux Toolkit - 1 file
import { createSlice, configureStore, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchUser = createAsyncThunk('user/fetch', async (id: string) => {
  const res = await fetch(`/api/users/${id}`).then(r=>r.json());
  return res as User;
});

const userSlice = createSlice({
  name: 'user',
  initialState: { value: null as User | null, status: 'idle' as 'idle'|'loading'|'failed' },
  reducers: {
    setUser: (state, action) => { state.value = action.payload; }, // Immer cho phép mutate draft
    logout: (state) => { state.value = null; },
  },
  extraReducers: builder => {
    builder.addCase(fetchUser.pending, state => { state.status = 'loading'; });
    builder.addCase(fetchUser.fulfilled, (state, action) => { state.status = 'idle'; state.value = action.payload; });
    builder.addCase(fetchUser.rejected, state => { state.status = 'failed'; });
  },
});

export const { setUser, logout } = userSlice.actions;

const store = configureStore({
  reducer: { user: userSlice.reducer },
  // tự có thunk, DevTools, immer check
});
type RootState = ReturnType<typeof store.getState>;

// Component
function Profile() {
  const user = useSelector((s: RootState) => s.user.value);
  const dispatch = useDispatch<AppDispatch>();
  useEffect(() => { dispatch(fetchUser('123')); }, [dispatch]);
}
```

**Trade-off:** RTK vẫn là Redux - vẫn cần hiểu immutable, selector, memo. Nếu team ghét Redux, RTK không cứu được, nên dùng Zustand. Nhưng với team đã Redux, RTK là bắt buộc.

**Câu hỏi đào sâu:** Immer hoạt động thế nào để cho phép `state.value = ...` mà vẫn immutable? `createAsyncThunk` khác `thunk` thủ công thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 153: Zustand vs Redux - khi nào chọn gì?

**Trả lời Senior:**
Cả hai đều global state, nhưng triết lý khác:

- **Redux (RTK)**: **single store**, **immutable**, **action → reducer**, **DevTools time-travel**, **middleware**, **convention nặng**, phù hợp **team lớn, cần quy trình chặt, debug phức tạp**.
- **Zustand**: **minimal**, **hook-based**, **mutate trực tiếp** (dùng Immer hoặc không), **không cần Provider** (trừ khi cần context), **không boilerplate**, **re-render tối ưu** (selector per-component), phù hợp **startup, app vừa, cần nhanh**.

Zustand core chỉ ~1KB, API là `create(set => ({ count, inc }))`, dùng `useStore(s => s.count)` là xong. Không cần action type, reducer, dispatch.

Bảng:

| | Redux Toolkit | Zustand |
|---|---|---|
| Boilerplate | Nhiều (slice, thunk) | Ít (1 file) |
| DevTools | Mạnh (time-travel) | Có (middleware) |
| Middleware | Nhiều (thunk, saga) | Ít, tự viết |
| Re-render | Cần memo selector | Tự tối ưu per-selector |
| Học | Cao | Thấp |
| Team lớn | Tốt (convention) | Cần tự kỷ luật |

```typescript
// Zustand - 1 file xong
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

type CartState = {
  items: Item[];
  add: (item: Item) => void;
  remove: (id: string) => void;
  total: () => number;
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
      { name: 'cart-storage' } // persist vào localStorage
    )
  )
);

// Component - chỉ re-render khi items đổi, không khi total đổi nếu không dùng
function CartCount() {
  const count = useCart(s => s.items.length); // chỉ subscribe length
  return <div>{count}</div>;
}
function AddButton({ item }: { item: Item }) {
  const add = useCart(s => s.add); // chỉ lấy add, không re-render khi items đổi
  return <button onClick={() => add(item)}>Thêm</button>;
}

// Redux Toolkit - nhiều hơn
// slice + configureStore + Provider + useSelector + useDispatch
```

**Trade-off:** Zustand nhanh nhưng dễ **vô kỷ luật** (ai cũng `set` lung tung, không trace). Redux chậm nhưng **ép convention**. Với 3-5 dev, Zustand tiết kiệm 30% thời gian. Với 20 dev, Redux tránh hỗn loạn.

**Câu hỏi đào sâu:** Vì sao Zustand không cần Provider mà Redux cần? Zustand persist khác Redux-persist thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 154: Context có thay Redux được không?

**Trả lời Senior:**
**Không hoàn toàn**. Context là **dependency injection**, không phải state management. Nó giải quyết **prop drilling**, nhưng không giải quyết **performance** và **update logic**.

Vấn đề của Context cho state:

1.  **Re-render tất cả consumer** khi `value` đổi, dù chỉ 1 field đổi. `useContext` không có selector, `const { user } = useContext(AppContext)` sẽ re-render cả component chỉ cần `theme`.
2.  **Không có DevTools, middleware, time-travel**.
3.  **Không tối ưu** cho frequent update (như `cart` thêm liên tục).

Context hợp cho **ít đổi, ít consumer**: `theme`, `locale`, `auth user`, `config`. Không hợp cho **nhiều đổi, nhiều consumer**: `cart`, `filter`, `search`.

Fix nếu vẫn muốn Context: **tách nhiều Context** + **memo**, hoặc dùng `useReducer` + `memo`, nhưng vẫn không bằng Zustand/Redux.

```typescript
// ❌ Context cho cart - re-render tất cả
const CartContext = createContext<{ items: Item[]; add: (i: Item) => void }>(null!);
function CartProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  const add = (item: Item) => setItems(prev => [...prev, item]);
  const value = useMemo(() => ({ items, add }), [items]); // vẫn re-render consumer khi items đổi
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
function CartCount() {
  const { items } = useContext(CartContext); // re-render khi bất kỳ item nào đổi, dù chỉ cần count
  return <div>{items.length}</div>;
}
function ProductCard({ product }: { product: Item }) {
  const { add } = useContext(CartContext); // cũng re-render khi items đổi, dù chỉ cần add
  return <button onClick={() => add(product)}>Thêm</button>;
}

// ✅ Tách Context - đỡ hơn
const CartItemsContext = createContext<Item[]>([]);
const CartActionsContext = createContext<{ add: (i: Item) => void }>(null!);
// CartCount chỉ subscribe Items, ProductCard chỉ Actions -> không re-render chéo

// ✅ Chuẩn: Zustand/Redux cho frequent state, Context cho stable state
// Context cho theme (ít đổi)
const ThemeContext = createContext<'light' | 'dark'>('light');
// Zustand cho cart (đổi nhiều)
const useCart = create<CartState>(...);
```

| | Context | Redux/Zustand |
|---|---|---|
| Re-render | Tất cả consumer | Chỉ selector đổi |
| DevTools | Không | Có |
| Frequent update | Kém | Tốt |
| Dùng cho | theme, locale, config | cart, filter, complex |

**Trade-off:** Context đơn giản, không thêm lib, nhưng với state đổi nhiều sẽ **lag**. Đo bằng Profiler: nếu Context làm 50 component re-render mỗi lần `setItems`, phải đổi.

**Câu hỏi đào sâu:** Vì sao `useContext` không có selector như `useSelector`? Làm sao tối ưu Context để không re-render thừa (tách context, memo)?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 155: Server State vs Client State - phân biệt

**Trả lời Senior:**
Đây là **phân loại quan trọng nhất** mà nhiều team nhầm, dẫn tới lưu server data vào Redux như client state, gây stale, cache sai, và code thừa.

- **Server State**: data **thuộc về server**, client chỉ **copy** và **đồng bộ**. Đặc tính: **persist trên server**, **có thể stale**, **cần fetch/cache/revalidate**, **nhiều client cùng thấy**, **cần dedup/polling**. Ví dụ: `products`, `orders`, `user profile`, `search results`. **Không nên** lưu trong Redux/Zustand vĩnh viễn, mà dùng **TanStack Query / SWR / RTK Query** - chúng lo cache, staleTime, revalidate, dedup.
- **Client State**: data **chỉ ở client**, **không cần server**, **đồng bộ UI**. Ví dụ: `isModalOpen`, `selectedTab`, `filter draft`, `form input`, `theme`, `cart (trước khi checkout)` (cart là hybrid). Lưu trong `useState`, `Zustand`, `Redux`, `Context`.

Nhầm lẫn kinh điển: lưu `products` vào Redux, rồi phải tự `useEffect` fetch, tự cache, tự invalidate - reinvent TanStack Query.

```typescript
// ❌ Server state trong Redux - tự lo hết
const productsSlice = createSlice({
  name: 'products',
  initialState: { data: [], loading: false, error: null },
  reducers: {},
  extraReducers: builder => {
    builder.addCase(fetchProducts.pending, s=>{s.loading=true});
    builder.addCase(fetchProducts.fulfilled, (s,a)=>{s.data=a.payload; s.loading=false});
  },
});
// Phải tự cache, stale, dedup, retry, poll - mệt

// ✅ Server state với TanStack Query - lo hết
import { useQuery } from '@tanstack/react-query';
function ProductList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['products', { category: 'shoes' }],
    queryFn: () => fetch('/api/products?category=shoes').then(r=>r.json()),
    staleTime: 60_000, // 1 phút mới stale
  });
  if (isLoading) return <Skeleton />;
  return <>{data.map(p => <Card key={p.id} product={p} />)}</>;
}

// ✅ Client state với Zustand/useState
const useUI = create<{ isCartOpen: boolean; toggle: () => void }>(set => ({
  isCartOpen: false,
  toggle: () => set(s => ({ isCartOpen: !s.isCartOpen })),
}));
function CartDrawer() {
  const isOpen = useUI(s => s.isCartOpen);
  return isOpen ? <Drawer /> : null;
}

// Hybrid: cart - client trước, sync server khi checkout
// Lưu cart trong Zustand + persist + sync lên server khi login
```

| | Server State | Client State |
|---|---|---|
| Nguồn | Server | Client |
| Stale | Có | Không |
| Cache | Cần (Query) | Không |
| Share | Nhiều client | 1 client |
| Ví dụ | products, orders | modal, filter draft |
| Lưu ở | TanStack Query | Zustand/Redux/useState |

**Trade-off:** Tách đúng làm code gọn 50%, không cần Redux cho server data. Cart là hybrid: trước login là client, sau login sync server.

**Câu hỏi đào sâu:** Vì sao không nên lưu server state trong Redux? Cart là server hay client state?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 156: TanStack Query là gì? useQuery và useMutation

**Trả lời Senior:**
TanStack Query (React Query) là **server state manager**: lo **fetch, cache, sync, dedup, retry, poll, optimistic, infinite**. Nó không thay Redux cho client state, mà **thay Redux cho server state**.

Core:

- **useQuery**: đọc server state. `queryKey` là id cache (mảng), `queryFn` fetch, `staleTime` quyết khi nào stale, `enabled` conditional, `select` transform.
- **useMutation**: sửa server state (POST/PUT/DELETE), có `onSuccess` để invalidate, `onMutate` cho optimistic.
- **Cache**: mỗi `queryKey` là 1 entry, có `data`, `status`, `error`, `isFetching`, `isStale`.
- **Dedup**: 2 component cùng `useQuery(['products'])` chỉ fetch 1 lần.
- **Background refetch**: khi `window focus`, `reconnect`, hoặc `staleTime` hết.

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Query - đọc
function Products({ category }: { category: string }) {
  const { data, isLoading, error, isFetching, refetch } = useQuery({
    queryKey: ['products', category], // key duy nhất, đổi category tự fetch lại
    queryFn: ({ signal }) => fetch(`/api/products?category=${category}`, { signal }).then(r => {
      if (!r.ok) throw new Error('Fetch failed');
      return r.json();
    }),
    staleTime: 60_000, // 1 phút tươi, sau đó stale -> refetch khi focus
    gcTime: 5 * 60_000, // 5 phút không dùng thì xóa cache
    retry: 2, // retry 2 lần nếu fail
    enabled: !!category, // chỉ fetch khi có category
    select: data => data.filter(p => p.inStock), // transform
  });
  if (isLoading) return <Skeleton />;
  if (error) return <div>Lỗi: {(error as Error).message}</div>;
  return <>{data.map(p => <Card key={p.id} product={p} />)} {isFetching && <Spinner />}</>;
}

// Mutation - sửa
function AddProduct() {
  const qc = useQueryClient();
  const { mutate, isPending } = useMutation({
    mutationFn: (newProduct: Product) => fetch('/api/products', { method: 'POST', body: JSON.stringify(newProduct) }).then(r=>r.json()),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: ['products'] }); // báo stale, sẽ refetch
    },
    onError: error => showToast(error.message),
  });
  return <button disabled={isPending} onClick={() => mutate({ name: 'New', price: 100 } as Product)}>Thêm</button>;
}

// Dedup: 2 component cùng key chỉ 1 fetch
function Header() { const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser }); }
function Profile() { const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser }); } // không fetch lại, dùng cache
```

**Trade-off:** TanStack Query thêm ~13KB, nhưng tiết kiệm 100 dòng Redux cho mỗi resource. Không dùng cho client state (modal, form) - dùng Zustand.

**Câu hỏi đào sâu:** `queryKey` vì sao phải là mảng? `useQuery` khác `useMutation` thế nào về cache?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 157: Cache Invalidation - khi nào invalidate?

**Trả lời Senior:**
Cache invalidation là **bài toán khó nhất** của server state: khi nào cache stale và phải fetch lại? TanStack Query cho 3 cách:

1.  **Auto**: `staleTime` hết, `window focus`, `reconnect`, `polling` (`refetchInterval`).
2.  **Manual**: `invalidateQueries({ queryKey: ['products'] })` - đánh dấu stale, sẽ refetch khi component mount/focus. `refetchQueries` - fetch ngay.
3.  **Optimistic + setQueryData**: update cache ngay, không đợi server.

Khi nào invalidate?

- **Sau mutation**: `POST /products` → `invalidate ['products']`, `PATCH /products/123` → `invalidate ['products', '123']` và `['products']` (list).
- **Sau action user**: checkout xong → invalidate `['cart']`, `['orders']`.
- **Không invalidate quá rộng**: `invalidate ['products']` làm mọi list refetch, nếu chỉ sửa 1 product thì `setQueryData(['product', id], newData)` + invalidate list nếu cần.

```typescript
const qc = useQueryClient();

// Sau tạo product
const createMutation = useMutation({
  mutationFn: (p: Product) => fetch('/api/products', { method: 'POST', body: JSON.stringify(p) }).then(r=>r.json()),
  onSuccess: (newProduct) => {
    // Cách 1: invalidate - đơn giản, nhưng refetch
    qc.invalidateQueries({ queryKey: ['products'] });
    // Cách 2: update cache ngay, không fetch
    // qc.setQueryData(['products'], (old: Product[]) => [...old, newProduct]);
  },
});

// Sau sửa 1 product
const updateMutation = useMutation({
  mutationFn: ({ id, ...patch }: Product) => fetch(`/api/products/${id}`, { method: 'PATCH', body: JSON.stringify(patch) }).then(r=>r.json()),
  onSuccess: (updated) => {
    qc.setQueryData(['product', updated.id], updated); // update detail
    qc.invalidateQueries({ queryKey: ['products'] }); // list stale
  },
});

// Polling cho data realtime
useQuery({ queryKey: ['orders'], queryFn: fetchOrders, refetchInterval: 30_000 }); // 30s poll

// Focus refetch (mặc định bật)
useQuery({ queryKey: ['products'], queryFn: fetchProducts, refetchOnWindowFocus: true }); // quay lại tab sẽ refetch nếu stale

// Invalidate khi logout
function logout() {
  qc.clear(); // xóa hết cache
  // hoặc qc.removeQueries()
}
```

**Trade-off:** `invalidate` đơn giản nhưng tốn request, `setQueryData` nhanh nhưng phải đảm bảo data đúng. Dùng `invalidate` cho list, `setQueryData` cho detail khi đã có response.

**Câu hỏi đào sâu:** `invalidateQueries` khác `refetchQueries` khác `setQueryData` thế nào? Khi nào dùng `polling` vs `focus refetch`?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 158: staleTime vs gcTime (cacheTime) - chi tiết

**Trả lời Senior:**
Hai khái niệm hay nhầm nhất của TanStack Query, quyết định **khi nào refetch** và **khi nào xóa cache**.

- **staleTime** (mặc định `0`): thời gian data được coi là **tươi (fresh)**. Trong `staleTime`, `useQuery` với cùng key sẽ **không fetch**, trả cache ngay, dù component remount. Sau `staleTime`, data thành **stale**, lần mount/focus tiếp theo sẽ **background refetch** (trả cache cũ trước, fetch mới sau). `staleTime: Infinity` nghĩa là không bao giờ stale (chỉ fetch 1 lần).
- **gcTime** (trước là `cacheTime`, mặc định `5 phút`): thời gian cache **sống** sau khi **không còn component nào subscribe**. Sau `gcTime`, cache bị **garbage collect** (xóa). `gcTime` chỉ tính khi `cache` không active.

Ví dụ: `staleTime: 60s`, `gcTime: 5m`. User vào `/products` → fetch → 30s sau quay lại → fresh, không fetch → 70s sau quay lại → stale, trả cache + background fetch → rời trang 5 phút không ai dùng `['products']` → xóa cache.

```typescript
// Mặc định: stale 0, gc 5m
useQuery({ queryKey: ['products'], queryFn: fetchProducts });
// Mỗi lần mount là stale -> refetch (nhiều request)

// Tối ưu cho ít đổi
useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  staleTime: 60_000, // 1 phút tươi
  gcTime: 10 * 60_000, // 10 phút mới xóa
});

// Không bao giờ stale (config, categories)
useQuery({
  queryKey: ['config'],
  queryFn: fetchConfig,
  staleTime: Infinity, // chỉ fetch 1 lần, trừ khi invalidate
});

// Realtime
useQuery({
  queryKey: ['ticker'],
  queryFn: fetchTicker,
  staleTime: 0, // luôn stale
  refetchInterval: 5_000, // 5s poll
});

// So sánh
// | | staleTime | gcTime |
// |---|---|---|
// | Mặc định | 0 (luôn stale) | 5 phút |
// | Khi nào | Data tươi bao lâu | Cache sống bao lâu khi không dùng |
// | Hết thì | Background refetch | Xóa cache |
// | Set | Tăng để ít fetch | Tăng để giữ cache lâu |

// DevTools: stale (mờ), fresh (xanh), fetching (xoay)
```

**Trade-off:** `staleTime` lớn giảm request nhưng data có thể cũ, `gcTime` lớn giữ cache nhưng tốn memory. Với list product, `staleTime: 60s` cân bằng. Với `user`, `staleTime: 5m` cũng được.

**Câu hỏi đào sâu:** Vì sao `staleTime: 0` là mặc định? Khi nào dùng `staleTime: Infinity`? `gcTime` vì sao đổi tên từ `cacheTime`?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 159: Thiết kế state cho ecommerce lớn - phân loại

**Trả lời Senior:**
Ecommerce lớn có ~10 domain state, Senior phân loại trước khi chọn tool, không nhét hết vào Redux.

Phân loại theo **3 trục**: **Server vs Client vs Hybrid**, **Global vs Local**, **Persist vs Ephemeral**.

- **Server State** (thuộc server, cần sync): `Product` (list, detail), `Order`, `User profile`, `Search results`, `Notifications` (từ server).
- **Client State** (chỉ UI): `Filter draft` (chưa apply), `Search input`, `UI` (modal, drawer, toast, selectedTab), `Form`.
- **Hybrid** (cả hai): `Cart` (client trước, sync server khi login), `Wishlist`.

Và **Global** (nhiều feature cần) vs **Local** (1 component): `User` global, `Form input` local.

Sơ đồ:

```
Server (TanStack Query)          Client (Zustand/Context/useState)
- products, product detail       - filter draft (local)
- orders, order detail           - search input (local)
- user profile                   - UI: modal, drawer, toast (global)
- notifications                  - cart (hybrid: Zustand + sync)
- search results                 - selected category (url)
```

```typescript
// Cấu trúc folder
// stores/ - client state
// stores/useCart.ts (Zustand + persist)
// stores/useUI.ts (Zustand)
// features/products/ - server state
// features/products/useProducts.ts (TanStack Query)
// features/orders/useOrders.ts

// Ví dụ phân loại
// Server - TanStack Query
function useProducts(category: string) {
  return useQuery({ queryKey: ['products', category], queryFn: () => fetch(`/api/products?cat=${category}`).then(r=>r.json()) });
}
function useOrders() {
  return useQuery({ queryKey: ['orders'], queryFn: fetchOrders });
}

// Client - Zustand
type FilterState = { priceRange: [number, number]; brand: string[]; setBrand: (b: string[]) => void };
export const useFilter = create<FilterState>(set => ({
  priceRange: [0, 1000],
  brand: [],
  setBrand: brand => set({ brand }),
}));

// Hybrid - Cart
type CartState = { items: Item[]; add: (i: Item) => void; sync: () => Promise<void> };
export const useCart = create<CartState>()(
  persist((set, get) => ({
    items: [],
    add: item => set({ items: [...get().items, item] }),
    sync: async () => { await fetch('/api/cart', { method: 'POST', body: JSON.stringify(get().items) }); },
  }), { name: 'cart' })
);
// Khi login: useCart.getState().sync()

// URL là state cho shareable: /products?category=shoes&price=0-100&sort=price
// Dùng nuqs hoặc useSearchParams, không lưu filter vào Zustand nếu muốn share link
```

**Trade-off:** Tách đúng làm mỗi state ở đúng nơi, không duplicate. Filter lưu ở URL để share, cart lưu ở localStorage để persist, product ở Query để cache.

**Câu hỏi đào sâu:** Vì sao filter nên lưu ở URL thay vì Zustand? Cart là hybrid thì sync khi nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 160: State nào global, local, server? User/Cart/Product/Filter/Search/Order/Notification/UI

**Trả lời Senior:**
Chốt phân loại cho 8 state phổ biến của ecommerce, Senior trả lời phỏng vấn phải nói được **vì sao**:

| State | Loại | Lưu ở đâu | Vì sao |
|---|---|---|---|
| **User** | Server + Global | TanStack Query `['user']` + Context cho auth | Thuộc server, nhiều nơi cần, cần cache |
| **Cart** | Hybrid + Global | Zustand + persist + sync `/api/cart` | Trước login là client, sau login sync server |
| **Product** (list/detail) | Server | TanStack Query `['products', id]` | Server state, nhiều component, cần stale/gc |
| **Filter** | Client + URL | `useSearchParams` + local `useState` draft | Share link, back button, không cần server |
| **Search** (input vs results) | Client + Server | Input: `useState` local, Results: Query `['search', q]` | Input là UI, results là server |
| **Order** | Server | TanStack Query `['orders']` | Thuộc server, cần poll/invalidate sau checkout |
| **Notification** | Server + Global | Query `['notifications']` + WebSocket/SSE | Server push, badge global |
| **UI** (modal, drawer, toast) | Client + Global | Zustand `useUI` hoặc `useState` local | Chỉ UI, không cần server |

Chi tiết:

- **User**: `useQuery(['user'])` để fetch, `Context` cho `isAuthenticated` để tránh prop drilling auth. Không lưu user trong Redux vĩnh viễn.
- **Cart**: `Zustand` với `persist` (localStorage), khi login thì `POST /api/cart/merge`, khi checkout thì `invalidate ['cart', 'orders']`.
- **Product**: `staleTime: 60s`, `gcTime: 5m`, `placeholderData` cho detail từ list.
- **Filter**: `?brand=nike&price=0-100` trong URL, `useFilterDraft` local cho slider chưa apply, `Apply` mới đẩy lên URL.
- **Search**: debounce input local, `useQuery(['search', debounced])` cho results, `enabled: debounced.length >=2`.
- **Order**: `refetchInterval: 30s` cho tracking, `invalidate` sau `createOrder`.
- **Notification**: `useQuery` + `SSE` để realtime, `setQueryData` khi có event.
- **UI**: nếu chỉ 1 component dùng thì `useState` local, nếu nhiều nơi (toast, modal) thì `Zustand`.

```typescript
// Ví dụ tổng hợp
// User - server
const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser });
// Cart - hybrid
const { items, add } = useCart(); // Zustand
// Product - server
const { data: products } = useQuery({ queryKey: ['products', category], queryFn: () => fetchProducts(category) });
// Filter - URL
const [searchParams, setSearchParams] = useSearchParams();
const brand = searchParams.get('brand');
const setBrand = (b: string) => setSearchParams({ brand: b });
// Search
const [input, setInput] = useState(''); // local
const debounced = useDebounce(input, 300);
const { data: results } = useQuery({ queryKey: ['search', debounced], queryFn: () => fetchSearch(debounced), enabled: debounced.length >=2 });
// UI
const { isCartOpen, toggle } = useUI();

// App.tsx
<QueryClientProvider client={qc}>
  <CartProvider /> {/* Zustand không cần provider, nhưng user context có thể cần */}
  <Routes>
    <Route path="/products" element={<ProductList />} /> {/* dùng useProducts */}
    <Route path="/cart" element={<CartPage />} /> {/* dùng useCart */}
  </Routes>
</QueryClientProvider>
```

**Trade-off:** Không có one-size-fits-all. Quy tắc: **Server state → Query, Client global → Zustand, Client local → useState, Shareable → URL**.

**Câu hỏi đào sâu:** Vì sao cart lại hybrid mà không pure server? Khi nào filter nên ở Zustand thay vì URL?

[↑ Quay lại Mục lục](#mục-lục)
---
