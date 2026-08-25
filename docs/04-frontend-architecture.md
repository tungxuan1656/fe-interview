# 04. Frontend Architecture - 17 Câu Hỏi Senior

> 17 câu hỏi kiến trúc Frontend (Câu 69-85) - từ tổ chức dự án, ranh giới feature, design system, quản lý state đến chiến lược refactor legacy và micro-frontend. Tư duy của Senior là thiết kế cho sự thay đổi, không phải cho sự hoàn hảo ban đầu.

## Mục lục

- [Câu 69: Tổ chức cấu trúc dự án lớn - tiêu chí và sai lầm phổ biến](#câu-69-tổ-chức-cấu-trúc-dự-án-lớn---tiêu-chí-và-sai-lầm-phổ-biến)
- [Câu 70: Feature-based vs Layer-based (components/hooks/services/utils)](#câu-70-feature-based-vs-layer-based-componentshooksservicesutils)
- [Câu 71: Làm sao tránh dependency chéo giữa các feature?](#câu-71-làm-sao-tránh-dependency-chéo-giữa-các-feature)
- [Câu 72: Thiết kế Design System - nguyên tắc và quy trình](#câu-72-thiết-kế-design-system---nguyên-tắc-và-quy-trình)
- [Câu 73: Thế nào là reusable component tốt?](#câu-73-thế-nào-là-reusable-component-tốt)
- [Câu 74: God Component - dấu hiệu và cách tách](#câu-74-god-component---dấu-hiệu-và-cách-tách)
- [Câu 75: Shared State - khi nào cần, khi nào thừa?](#câu-75-shared-state---khi-nào-cần-khi-nào-thừa)
- [Câu 76: Global State nào nên tồn tại? Phân loại state](#câu-76-global-state-nào-nên-tồn-tại-phân-loại-state)
- [Câu 77: Server State vs Client State - phân biệt và quản lý](#câu-77-server-state-vs-client-state---phân-biệt-và-quản-lý)
- [Câu 78: Thiết kế API Layer - abstraction, interceptor và retry](#câu-78-thiết-kế-api-layer---abstraction-interceptor-và-retry)
- [Câu 79: Error Handling toàn app - Error Boundary, global handler và fallback](#câu-79-error-handling-toàn-app---error-boundary-global-handler-và-fallback)
- [Câu 80: Xử lý Loading / Error / Empty State nhất quán](#câu-80-xử-lý-loading--error--empty-state-nhất-quán)
- [Câu 81: Chiến lược migrate legacy code an toàn](#câu-81-chiến-lược-migrate-legacy-code-an-toàn)
- [Câu 82: Refactor 500 components - làm sao không sập production?](#câu-82-refactor-500-components---làm-sao-không-sập-production)
- [Câu 83: Micro-frontend là gì? Các pattern triển khai](#câu-83-micro-frontend-là-gì-các-pattern-triển-khai)
- [Câu 84: Khi nào nên dùng micro-frontend?](#câu-84-khi-nào-nên-dùng-micro-frontend)
- [Câu 85: Khi nào không nên dùng micro-frontend và alternative?](#câu-85-khi-nào-không-nên-dùng-micro-frontend-và-alternative)

---

### Câu 69: Tổ chức cấu trúc dự án lớn - tiêu chí và sai lầm phổ biến

**Trả lời Senior:**
Cấu trúc thư mục không phải để đẹp mà để **kiểm soát độ phức tạp khi scale**. Tiêu chí Senior đánh giá: 1) **Cohesion cao - Coupling thấp**: code thay đổi cùng lý do nằm gần nhau; 2) **Rào cản import rõ ràng**: junior không thể vô tình import nhầm tầng; 3) **Khả năng xóa**: xóa 1 feature là xóa 1 folder, không phải săn import rải rác; 4) **Discoverability**: người mới nhìn cây thư mục đoán được code nằm đâu.

Sai lầm phổ biến: `src/components`, `src/hooks`, `src/utils`, `src/services` thuần layer-based. Với 20 components thì ổn, với 500 thì `components` thành bãi rác, không biết component nào thuộc feature nào, thay đổi 1 feature sợ vỡ chỗ khác. Sai lầm thứ hai là chia theo loại file thay vì theo domain: `utils/formatMoney` dùng cho `order` lại nằm ở `shared/utils` khiến ai cũng import.

```typescript
// Cấu trúc scale được - Feature-Sliced Design (FSD) biến thể
src/
  app/         // providers, router, store setup - chỉ init
  pages/       // route composition - ghép widgets/features
  widgets/     // Header, CartSidebar - compose nhiều features
  features/    // auth, checkout, search - business logic + UI
    checkout/
      ui/ CheckoutForm.tsx
      model/ useCheckout.ts, checkoutStore.ts
      api/ checkoutApi.ts
      index.ts // public API - chỉ export thứ này
  entities/    // user, product, order - domain model thuần
  shared/      // ui, api, lib, config - không import từ tầng trên
```

**Trade-off:** FSD/Feature-based thêm boilerplate, over-engineering cho app 5 trang. App nhỏ nên bắt đầu layer-based đơn giản, khi > 50 components hoặc > 5 dev thì refactor sang feature.

**Câu hỏi đào sâu:** Làm sao enforce ranh giới tầng bằng ESLint `no-restricted-imports`? Khi nào nên tách `shared` thành package riêng trong monorepo?

---

### Câu 70: Feature-based vs Layer-based (components/hooks/services/utils)

**Trả lời Senior:**
Layer-based chia theo **kỹ thuật**: `components/`, `hooks/`, `services/`, `utils/`. Feature-based chia theo **nghiệp vụ**: `features/auth`, `features/cart`, mỗi feature chứa đủ UI, logic, API của nó. Layer-based dễ bắt đầu nhưng **coupling theo chiều ngang** rất cao - thay đổi logic cart phải đụng 4 folder khác nhau. Feature-based thì **coupling theo chiều dọc** - mọi thứ của cart ở một chỗ, xóa là hết.

Kinh nghiệm Senior: không chọn cực đoan. Pattern thực chiến là **hybrid**: feature-based cho business, layer-based cho shared. `entities/` và `shared/` là layer, `features/` là vertical slice. Rule quan trọng: `shared` không được import từ `features`, `features` không import chéo nhau (chỉ qua `shared` hoặc `entities`).

```typescript
// Layer-based - khó scale
src/components/Button.tsx
src/components/CheckoutForm.tsx // thuộc checkout nhưng nằm chung
src/hooks/useCheckout.ts
src/services/checkoutApi.ts // tìm file phải nhớ loại

// Feature-based - cohesion cao
src/features/checkout/
  ui/CheckoutForm.tsx      // chỉ checkout dùng
  model/useCheckout.ts
  api/checkoutApi.ts
  lib/validateCoupon.ts   // private, không share
  index.ts                // export public: export { CheckoutForm } from './ui/CheckoutForm'

// Shared thuần túy
src/shared/ui/Button.tsx
src/shared/lib/formatMoney.ts
src/shared/api/client.ts

// ESLint rule cấm import chéo
// features/checkout không được import từ features/cart
// "no-restricted-imports": ["error", { "patterns": ["@/features/*"] }]
```

**Trade-off:** Feature-based làm trùng lặp nhẹ (mỗi feature có `utils` riêng) nhưng đổi lại isolation tốt. Layer-based DRY hơn nhưng dễ tạo `utils` khổng lồ 200 hàm không ai dám xóa.

**Câu hỏi đào sâu:** Làm sao xử lý khi 2 feature cần share 1 component? Khi nào nên promote từ `features/X/ui` lên `shared/ui`?

---

### Câu 71: Làm sao tránh dependency chéo giữa các feature?

**Trả lời Senior:**
Dependency chéo là `features/cart` import `features/product`, rồi `features/product` lại import `features/cart` - tạo vòng tròn, build chậm, test phải mock cả cụm, xóa 1 feature vỡ feature kia. Senior giải quyết bằng 3 kỹ thuật: **Public API + Dependency Rule**, **Event/Mediator**, và **Dependency Inversion**.

Rule cứng: Feature chỉ được phép import từ `shared` và `entities`, **không import trực tiếp feature khác**. Nếu cần giao tiếp, đi qua tầng trung gian: `app` hoặc `widgets` compose, hoặc dùng event bus/store chung ở `shared`.

```typescript
// ❌ Sai: cart import trực tiếp product
// features/cart/CartItem.tsx
import { ProductCard } from '@/features/product/ProductCard';

// ✅ Đúng 1: Đưa Product xuống entities (domain thuần)
import { Product } from '@/entities/product'; // entities không có logic feature

// ✅ Đúng 2: Inversion - cart expose props, để pages/widgets inject
// features/cart/ui/CartItem.tsx
type Props = { product: Product; renderProduct?: (p: Product) => React.ReactNode };
export const CartItem = ({ product, renderProduct }: Props) => (
  <div>{renderProduct ? renderProduct(product) : <span>{product.name}</span>}</div>
);
// pages/cart/page.tsx (tầng cao hơn) mới compose
import { CartItem } from '@/features/cart';
import { ProductCard } from '@/features/product';
<CartItem product={p} renderProduct={p => <ProductCard product={p} />} />

// ✅ Đúng 3: Event bus cho side-effect
// shared/events/bus.ts
import mitt from 'mitt';
export const bus = mitt<{ cartUpdated: { id: string } }>();
// features/cart -> bus.emit('cartUpdated', { id })
// features/product -> bus.on('cartUpdated', handler)

// Enforce bằng eslint
// .eslintrc: "no-restricted-imports": ["error", { "patterns": ["@/features/*"] }]
// Cho phép trong pages/widgets/app
```

**Trade-off:** Event bus làm flow khó trace hơn import trực tiếp. Inversion qua props làm component API rộng hơn. Chọn inversion cho UI, event cho side-effect.

**Câu hỏi đào sâu:** Vì sao nên dùng `index.ts` public API cho mỗi feature? Làm sao phát hiện vòng tròn dependency bằng `madge` hoặc `eslint-plugin-import`?

---

### Câu 72: Thiết kế Design System - nguyên tắc và quy trình

**Trả lời Senior:**
Design System không phải là thư viện Button đẹp, mà là **hợp đồng giữa design và code**: token, component, pattern, và quy tắc sử dụng. Senior xây theo thứ tự: **Token → Primitive → Compound → Pattern**. Token là single source of truth (`color.primary.500`, `space.4`), primitive là `Box`, `Text`, `Button` không gắn business, compound là `Select`, `Dialog` ghép primitive, pattern là `Form Layout`, `Data Table`.

Nguyên tắc: 1) **Composition over configuration**: `Button` nhận `leftIcon`, `isLoading` qua composition, không phải 20 props boolean; 2) **Headless + Style tách rời**: logic (Radix, Ariakit, Headless UI) tách khỏi style để đổi theme không vỡ logic; 3) **Contract-first**: Figma token đồng bộ qua Style Dictionary, không để dev tự hardcode `16px`.

```typescript
// Token - đồng bộ Figma
// tokens.ts
export const tokens = {
  color: { primary: { 500: '#0066ff', 600: '#0052cc' } },
  space: { 4: '16px', 6: '24px' },
  radius: { md: '8px' },
} as const;

// Primitive - chỉ nhận token, không business
type ButtonProps = {
  variant?: 'primary' | 'ghost';
  size?: 'sm' | 'md';
  isLoading?: boolean;
} & React.ButtonHTMLAttributes<HTMLButtonElement>;

export const Button = ({ variant = 'primary', size = 'md', isLoading, children, ...props }: ButtonProps) => (
  <button data-variant={variant} data-size={size} disabled={isLoading || props.disabled} {...props}>
    {isLoading ? <Spinner /> : children}
  </button>
);

// Compound - headless
import * as Dialog from '@radix-ui/react-dialog';
export const Modal = ({ children, ...props }: Dialog.DialogProps) => (
  <Dialog.Root {...props}>
    <Dialog.Portal>
      <Dialog.Overlay className="overlay" />
      <Dialog.Content className="content">{children}</Dialog.Content>
    </Dialog.Portal>
  </Dialog.Root>
);
```

Quy trình: Figma → Token → Storybook (document + visual test) → Chromatic (visual regression) → Release versioned (`@company/ui@1.2.0`) → Codemod khi breaking change.

**Trade-off:** Xây design system tốn 30-40% thời gian đầu, chỉ đáng khi > 2 team hoặc > 1 product. Nếu chỉ 1 app, dùng Tailwind + Radix là đủ, đừng tự xây.

**Câu hỏi đào sâu:** Làm sao versioning design system không làm các app kẹt ở version cũ? Khi nào nên headless thay vì styled sẵn?

---

### Câu 73: Thế nào là reusable component tốt?

**Trả lời Senior:**
Reusable không có nghĩa là dùng được mọi nơi, mà là **dùng lại được mà không phải sửa source**. Component tốt có 4 đặc tính: **Single Responsibility** (làm 1 việc), **Inversion of Control** (để caller quyết định phần biến đổi), **Stable API** (props ít, rõ ràng, không breaking mỗi sprint), và **Không rò rỉ business**.

Dấu hiệu component dở: 15 props boolean (`isCart`, `isCheckout`, `showDiscount`), `if (variant === 'cart')` rải rác, import `useCartStore` trực tiếp bên trong. Đó là component đang cố làm 2 việc.

Công thức Senior: **Primitive + Composition + Render Props/Slot**. Thay vì prop `showIcon`, cho `icon` slot. Thay vì `onAddToCart` hardcode, nhận `onAction`.

```typescript
// ❌ Tệ: nhiều flag, rò business
function Card({ product, isCart, isWishlist, showDiscount, onAddToCart }: Props) {
  if (isCart) return <div>...</div>;
  if (isWishlist) return <div>...</div>;
}

// ✅ Tốt: headless + slot
type CardProps = {
  children: React.ReactNode;
  actions?: React.ReactNode;
};
export const Card = ({ children, actions }: CardProps) => (
  <div className="card">
    <div className="body">{children}</div>
    {actions && <div className="actions">{actions}</div>}
  </div>
);

// Dùng - caller quyết định
<Card actions={<Button onClick={addToCart}>Add</Button>}>
  <ProductInfo product={p} />
</Card>
<Card actions={<Button variant="ghost" onClick={removeWishlist}>Remove</Button>}>
  <ProductInfo product={p} />
</Card>

// ✅ Tốt: polymorphic + as prop, forward style
type TextProps<C extends React.ElementType> = {
  as?: C;
  variant?: 'body' | 'heading';
} & React.ComponentPropsWithoutRef<C>;
```

**Trade-off:** Component quá generic sẽ API phức tạp (như `as` polymorphic). Rule: chỉ abstract khi có 3 use case thật, không phải 1 use case + 2 tưởng tượng (Rule of Three).

**Câu hỏi đào sâu:** Khi nào nên tách `Card` thành `Card` + `CardHeader` + `CardActions` (compound)? Làm sao type `as` prop an toàn với TypeScript?

---

### Câu 74: God Component - dấu hiệu và cách tách

**Trả lời Senior:**
God Component là component 800-2000 dòng, làm mọi thứ: fetch, validate, render 5 section, chứa 12 `useState`, 8 `useEffect`. Dấu hiệu: file tên `Dashboard.tsx`, `HomePage.tsx`, `UserProfile.tsx` mà mở ra scroll 5 phút chưa hết; props drilling 4 tầng; thay 1 dòng sợ vỡ 3 chỗ; test phải mock cả thế giới.

Nguyên nhân: thiếu ranh giới, cứ thêm requirement là nhét vào component cũ cho nhanh. Fix bằng **Extract theo 3 trục**: **Logic → Hook/Store**, **UI → Sub-component + Composition**, **Side-effect → Service**.

```typescript
// ❌ God Component
function Dashboard() {
  const [user, setUser] = useState(null);
  const [orders, setOrders] = useState([]);
  const [filter, setFilter] = useState('');
  const [selected, setSelected] = useState(null);
  // 400 dòng fetch + filter + modal + chart + table
  useEffect(() => { fetchUser(); }, []);
  useEffect(() => { fetchOrders(); }, [filter]);
  return <div>{/* 600 dòng JSX */}</div>;
}

// ✅ Tách
// 1. Logic ra hook/store
function useDashboard() {
  const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser });
  const { data: orders } = useQuery({ queryKey: ['orders', filter], queryFn: () => fetchOrders(filter) });
  return { user, orders };
}

// 2. UI thành widget/feature nhỏ, nhận props
function Dashboard() {
  const { user, orders } = useDashboard();
  return (
    <Layout>
      <UserHeader user={user} />
      <OrderFilter value={filter} onChange={setFilter} />
      <OrderTable orders={orders} onSelect={setSelected} />
      <OrderDetailModal order={selected} />
    </Layout>
  );
}

// 3. Mỗi sub-component tự quản state cục bộ nếu không cần share
function OrderFilter({ value, onChange }: Props) {
  const [draft, setDraft] = useState(value); // local, không lift lên Dashboard
  return <input value={draft} onChange={e => setDraft(e.target.value)} onBlur={() => onChange(draft)} />;
}
```

Chiến thuật tách an toàn: không rewrite 1 lần, mà **Extract từng phần + test + ship**, dùng `React.lazy` tạm thời nếu cần. Đo bằng **Lines of Code, số useState/useEffect, cyclomatic complexity**.

**Trade-off:** Tách nhỏ quá tạo prop drilling mới, cần cân bằng với Context/local store. Đừng tách chỉ để file ngắn mà làm flow khó theo dõi.

**Câu hỏi đào sâu:** Làm sao quyết định state nào nên ở lại parent, state nào colocate xuống con? Khi nào dùng `children` để tránh re-render khi tách?

---

### Câu 75: Shared State - khi nào cần, khi nào thừa?

**Trả lời Senior:**
Shared state là state mà ≥ 2 component không có quan hệ cha-con trực tiếp cùng đọc/ghi. Sai lầm lớn nhất của mid-level là cho mọi thứ vào global store vì "sau này cần". Thực tế 70% state là **local**: input draft, toggle, hover. Chỉ 30% cần share: auth user, cart, theme.

Test 3 câu hỏi trước khi đưa lên global: 1) **Có bao nhiêu nơi đọc?** Nếu chỉ 1 nơi, để local; 2) **Có cần persist qua route không?** Nếu không, local; 3) **Có cần sync giữa tab/optimistic không?** Nếu không, local.

```typescript
// ❌ Thừa: đưa local lên global
const useStore = create(set => ({
  searchDraft: '', // chỉ SearchInput dùng -> local
  isModalOpen: false, // chỉ Modal dùng -> local
  setSearchDraft: v => set({ searchDraft: v }),
}));

// ✅ Đúng: local
function SearchInput({ onSearch }: Props) {
  const [draft, setDraft] = useState(''); // local, chỉ khi submit mới lift
  return <input value={draft} onChange={e => setDraft(e.target.value)} onBlur={() => onSearch(draft)} />;
}

// ✅ Cần share: cart, auth
const useCartStore = create<CartState>(set => ({
  items: [],
  add: item => set(s => ({ items: [...s.items, item] })),
}));
// Dùng selector để tránh re-render thừa
function CartBadge() {
  const count = useCartStore(s => s.items.length); // chỉ render khi count đổi
  return <span>{count}</span>;
}

// ✅ Share qua URL thay vì store cho filter
// /products?category=shoes&page=2 -> dùng searchParams, không cần global
function useProductFilter() {
  const [params, setParams] = useSearchParams();
  return { category: params.get('category'), setCategory: v => setParams({ category: v }) };
}
```

**Trade-off:** Global state làm component khó test (phải mock store), khó reuse (phụ thuộc implicit). Local state dễ test, dễ xóa. Ưu tiên **colocation** (Kent C. Dodds).

**Câu hỏi đào sâu:** Vì sao URL là nơi share state tốt nhất cho filter/pagination? Làm sao phát hiện shared state thừa bằng code review?

---

### Câu 76: Global State nào nên tồn tại? Phân loại state

**Trả lời Senior:**
Không phải global nào cũng giống nhau. Senior phân 5 loại state và chỗ ở của nó:

1.  **Server State** (user, product, order): cache từ API, stale, cần revalidate. Ở **React Query / RTK Query**, không phải Zustand/Redux. Có `staleTime`, `cacheTime`.
2.  **Client Global State** (auth status, theme, locale, cart optimistic): cần qua nhiều route, ở **Zustand/Jotai** với selector.
3.  **URL State** (filter, sort, page, tab): ở **URL searchParams**, share qua link, back/forward.
4.  **Form State** (input, validation): ở **React Hook Form**, uncontrolled để tránh re-render.
5.  **Ephemeral UI State** (hover, focus, animation): ở **useState local**.

Chỉ loại 2 mới thực sự là global store. Đưa server state vào Redux là anti-pattern kinh điển - tự quản cache, loading, error thủ công trong khi React Query đã làm tốt hơn.

```typescript
// Phân loại rõ
// 1. Server State - React Query
const { data: products } = useQuery({ queryKey: ['products', category], queryFn: () => fetchProducts(category) });

// 2. Client Global - Zustand (chỉ auth/theme/cart)
const useAuthStore = create<AuthState>(set => ({
  user: null,
  login: user => set({ user }),
  logout: () => set({ user: null }),
}));

// 3. URL State
function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();
  const category = searchParams.get('category') ?? 'all';
  // share link: /products?category=shoes
}

// 4. Form State - RHF
const { register, handleSubmit } = useForm({ resolver: zodResolver(schema) });

// 5. Ephemeral
const [isHovered, setHovered] = useState(false);

// Checklist trước khi tạo global:
// - Có cần qua 3+ route không? Không -> local
// - Có phải từ server không? Có -> React Query
// - Có cần share qua URL không? Có -> searchParams
```

**Trade-off:** Nhiều loại state làm onboard phức tạp, cần document quy tắc. Nhưng rõ ràng hơn 1 Redux store chứa hết.

**Câu hỏi đào sâu:** Vì sao không nên lưu server state trong Redux? Khi nào cart nên ở server state thay vì client global?

---

### Câu 77: Server State vs Client State - phân biệt và quản lý

**Trả lời Senior:**
Server state là **snapshot của DB ở xa**, có tính bất đồng bộ, stale, cần sync, và là single source of truth ở server. Client state là **state thuần UI**, đồng bộ, không cần revalidate. Nhầm lẫn hai loại dẫn tới bug: cache không invalidate, optimistic update sai, loading state tự chế.

Đặc điểm server state: 1) **Async** (phải fetch), 2) **Shared ownership** (nhiều client cùng sửa), 3) **Stale** (cần revalidate), 4) **Cache + background update**. Đặc điểm client state: **Sync**, **Local ownership**, **Không stale**.

Quản lý server state bằng **React Query / SWR / RTK Query**: họ lo caching, deduping, polling, retry, optimistic update, garbage collection. Client state bằng **Zustand/Jotai/useState**.

```typescript
// ❌ Sai: tự quản server state bằng useEffect + useState
function Products() {
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(true);
  useEffect(() => { fetch('/api/products').then(r => r.json()).then(setData).finally(() => setLoading(false)); }, []);
  // thiếu: dedupe, cache, retry, revalidate on focus, optimistic
}

// ✅ Đúng: React Query cho server state
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function Products() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['products'],
    queryFn: () => fetch('/api/products').then(r => r.json()),
    staleTime: 1000 * 60, // 1 phút coi là fresh
  });
  if (isLoading) return <Skeleton />;
  if (error) return <ErrorFallback error={error} />;
  return <List data={data} />;
}

// Mutation + optimistic + invalidate
function AddProduct() {
  const qc = useQueryClient();
  const { mutate } = useMutation({
    mutationFn: newProduct => fetch('/api/products', { method: 'POST', body: JSON.stringify(newProduct) }),
    onMutate: async newProduct => {
      await qc.cancelQueries({ queryKey: ['products'] });
      const prev = qc.getQueryData(['products']);
      qc.setQueryData(['products'], old => [...old, newProduct]); // optimistic
      return { prev };
    },
    onError: (err, newProduct, ctx) => qc.setQueryData(['products'], ctx.prev), // rollback
    onSettled: () => qc.invalidateQueries({ queryKey: ['products'] }),
  });
}

// Client state thuần
const useThemeStore = create(set => ({ theme: 'light', toggle: () => set(s => ({ theme: s.theme === 'light' ? 'dark' : 'light' })) }));
```

**Trade-off:** Thêm React Query là thêm dependency, nhưng tiết kiệm 70% code fetch thủ công và bug. Đừng trộn server state vào Zustand - sẽ phải tự implement cache.

**Câu hỏi đào sâu:** `staleTime` vs `cacheTime` (gcTime) khác gì? Khi nào dùng optimistic update thay vì invalidate?

---

### Câu 78: Thiết kế API Layer - abstraction, interceptor và retry

**Trả lời Senior:**
API layer là **ranh giới giữa frontend và backend**, phải cô lập để đổi backend không đụng UI. Senior thiết kế 3 lớp: **Client (fetch/axios instance)** → **Resource/Service (typed function)** → **Hook/Query (React Query)**. Client lo baseURL, interceptor, auth, retry; Resource lo endpoint, type, transform; Hook lo caching và UI binding.

Yêu cầu: typed 100% (OpenAPI codegen hoặc tRPC), abortable, retry có backoff, refresh token queue, và error normalize.

```typescript
// 1. Client - singleton
import axios from 'axios';

export const api = axios.create({ baseURL: '/api', timeout: 10000 });

// Request interceptor - gắn token
api.interceptors.request.use(config => {
  const token = getToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor - refresh queue + normalize error
let isRefreshing = false;
let queue: ((token: string) => void)[] = [];

api.interceptors.response.use(
  res => res,
  async error => {
    const original = error.config;
    if (error.response?.status === 401 && !original._retry) {
      if (isRefreshing) return new Promise(resolve => queue.push(token => resolve(api({ ...original, headers: { ...original.headers, Authorization: `Bearer ${token}` } }))));
      original._retry = true;
      isRefreshing = true;
      const newToken = await refreshToken();
      queue.forEach(cb => cb(newToken));
      queue = [];
      isRefreshing = false;
      return api({ ...original, headers: { ...original.headers, Authorization: `Bearer ${newToken}` } });
    }
    // Normalize error
    return Promise.reject({ message: error.response?.data?.message ?? error.message, status: error.response?.status });
  }
);

// 2. Resource - typed
// openapi-codegen hoặc tự định nghĩa
export const productApi = {
  list: (params: { category?: string }) => api.get<Product[]>('/products', { params }).then(r => r.data),
  get: (id: string) => api.get<Product>(`/products/${id}`).then(r => r.data),
  create: (data: CreateProductDto) => api.post<Product>('/products', data).then(r => r.data),
};

// 3. Hook - React Query
export const useProducts = (category?: string) =>
  useQuery({ queryKey: ['products', category], queryFn: () => productApi.list({ category }) });

// Retry với backoff + AbortController
export const fetchWithRetry = async (fn: () => Promise<any>, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try { return await fn(); } catch (e) { if (i === retries - 1) throw e; await new Promise(r => setTimeout(r, 2 ** i * 1000)); }
  }
};
```

**Trade-off:** Axios tiện interceptor nhưng nặng hơn fetch; fetch native nhẹ nhưng phải tự wrap. Đừng để component gọi `fetch` trực tiếp - khó mock, khó đổi baseURL.

**Câu hỏi đào sâu:** Làm sao queue request khi đang refresh token? Khi nào dùng `AbortController` để cancel request khi component unmount?

---

### Câu 79: Error Handling toàn app - Error Boundary, global handler và fallback

**Trả lời Senior:**
Error handling Senior chia 3 tầng: **Render Error (Error Boundary)** → **Async/Reject (global handler + React Query)** → **Event/Effect (try/catch + toast)**. Không có 1 nơi bắt hết.

- **Render error**: dùng `ErrorBoundary` (class) bọc route/feature. Nó bắt lỗi trong render/lifecycle của con, hiện fallback, cho retry. Không bắt được async, event.
- **Async error**: `window.addEventListener('unhandledrejection')` + `window.onerror` + interceptor API để log và toast. React Query có `onError` global.
- **UX fallback**: phân cấp - page-level (500 page), section-level (retry card), inline (toast).

```typescript
// 1. Error Boundary - route level
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  return (
    <ErrorBoundary FallbackComponent={RouteError} onReset={() => window.location.reload()}>
      <Routes>
        <Route path="/products" element={
          <ErrorBoundary fallback={<SectionError />}>
            <ProductPage />
          </ErrorBoundary>
        } />
      </Routes>
    </ErrorBoundary>
  );
}
function RouteError({ error, resetErrorBoundary }: { error: Error; resetErrorBoundary: () => void }) {
  return <div><h1>Có lỗi xảy ra</h1><pre>{error.message}</pre><button onClick={resetErrorBoundary}>Thử lại</button></div>;
}

// 2. Global async handler
window.addEventListener('unhandledrejection', e => {
  console.error('Unhandled', e.reason);
  reportToSentry(e.reason);
  e.preventDefault();
});
window.addEventListener('error', e => reportToSentry(e.error));

// 3. API layer normalize + React Query global
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: 1, throwOnError: false },
    mutations: { onError: err => toast.error(err.message) },
  },
});

// 4. Event handler - phải try/catch thủ công
async function onSubmit() {
  try { await api.post('/order', data); toast.success('Đặt hàng thành công'); }
  catch (e) { toast.error(e.message); /* không throw tiếp nếu đã handle */ }
}
```

**Trade-off:** Bọc quá nhiều Error Boundary làm code verbose, bọc quá ít thì 1 lỗi con sập cả app. Quy tắc: bọc quanh **route và widget quan trọng**, không bọc quanh từng button.

**Câu hỏi đào sâu:** Vì sao Error Boundary không bắt được lỗi trong `setTimeout` hay `Promise`? Làm sao kết hợp Error Boundary với Suspense để bắt lỗi fetch?

---

### Câu 80: Xử lý Loading / Error / Empty State nhất quán

**Trả lời Senior:**
App thiếu consistency sẽ có nơi hiện spinner toàn màn hình, nơi skeleton, nơi trắng trơn khi empty - UX tệ và code lặp `if (loading) ... if (error) ...`. Senior chuẩn hóa bằng **State Machine + Component Pattern**: mỗi async view có 4 trạng thái - `idle | loading | error | success (empty | filled)`, và 1 component `<AsyncView>` thống nhất.

Nguyên tắc: 1) **Skeleton cho loading** (không spinner giữa màn hình trừ page transition), 2) **Error có retry**, 3) **Empty có CTA**, 4) **Đừng để 3 trạng thái trong mỗi component**, extract ra.

```typescript
// Pattern AsyncView
type AsyncViewProps<T> = {
  query: { data?: T; isLoading: boolean; error?: Error; isEmpty?: boolean };
  skeleton?: React.ReactNode;
  empty?: React.ReactNode;
  errorFallback?: (error: Error, retry: () => void) => React.ReactNode;
  children: (data: T) => React.ReactNode;
};

function AsyncView<T>({ query, skeleton, empty, errorFallback, children }: AsyncViewProps<T>) {
  if (query.isLoading) return <>{skeleton ?? <ListSkeleton />}</>;
  if (query.error) return <>{errorFallback ? errorFallback(query.error, () => window.location.reload()) : <ErrorCard error={query.error} />}</>;
  if (query.isEmpty || (Array.isArray(query.data) && query.data.length === 0)) return <>{empty ?? <EmptyState />}</>;
  return <>{children(query.data as T)}</>;
}

// Dùng
function ProductPage() {
  const query = useQuery({ queryKey: ['products'], queryFn: fetchProducts });
  return (
    <AsyncView
      query={{ data: query.data, isLoading: query.isLoading, error: query.error as Error, isEmpty: query.data?.length === 0 }}
      skeleton={<ProductSkeleton count={6} />}
      empty={<EmptyState title="Chưa có sản phẩm" action={<Button>Tạo mới</Button>} />}
      errorFallback={(err, retry) => <ErrorCard message={err.message} onRetry={retry} />}
    >
      {data => <ProductGrid products={data} />}
    </AsyncView>
  );
}

// Empty state có CTA, không để trang trắng
function EmptyState({ title, action }: { title: string; action?: React.ReactNode }) {
  return <div className="empty"><p>{title}</p>{action}</div>;
}

// Với Suspense - còn gọn hơn
<Suspense fallback={<ProductSkeleton />}><Products /></Suspense>
<ErrorBoundary fallback={<ErrorCard />}><Products /></ErrorBoundary>
```

**Trade-off:** Abstraction `AsyncView` thêm 1 layer, nhưng giảm 80% lặp `if`. Đừng over-abstract tới mức skeleton/empty không custom được - cho phép override qua props.

**Câu hỏi đào sâu:** Skeleton vs Spinner khi nào dùng cái nào? Làm sao test empty state mà không cần mock API trả rỗng?

---

### Câu 81: Chiến lược migrate legacy code an toàn

**Trả lời Senior:**
Migrate legacy (jQuery, AngularJS, class component, Webpack 3) không phải là rewrite 1 lần - đó là cách tự sát. Chiến lược Senior là **Strangler Fig**: bọc legacy, thay từng nhánh, song song chạy, đo và xóa dần.

Các bước: 1) **Viết test/characterization test** cho legacy (dù là test thô bằng Playwright snapshot) để có lưới an toàn; 2) **Tách ranh giới** - dựng `adapter` để legacy và mới giao tiếp (event, iframe, hoặc cùng DOM); 3) **Migrate theo vertical slice** (1 route/feature 1 lần), không theo layer; 4) **Dual-run + feature flag** để rollback; 5) **Xóa legacy khi coverage mới đủ**.

```typescript
// Strangler Fig - ví dụ migrate jQuery sang React
// 1. Bọc legacy trong React
function LegacyAdapter({ legacyPath }: { legacyPath: string }) {
  const ref = useRef<HTMLDivElement>(null);
  useEffect(() => {
    // mount jQuery app vào ref
    window.legacyApp.mount(ref.current, legacyPath);
    return () => window.legacyApp.unmount(ref.current);
  }, [legacyPath]);
  return <div ref={ref} />;
}

// 2. Route-based strangler
function App() {
  const flags = useFeatureFlags();
  return (
    <Routes>
      {/* Đã migrate */}
      <Route path="/products/*" element={<NewProductApp />} />
      {/* Chưa migrate - vẫn legacy */}
      <Route path="/admin/*" element={flags.newAdmin ? <NewAdmin /> : <LegacyAdapter legacyPath="/admin" />} />
      <Route path="*" element={<LegacyAdapter legacyPath={window.location.pathname} />} />
    </Routes>
  );
}

// 3. Incremental - shared design system để UI đồng nhất
// Cả legacy và mới cùng dùng @company/ui qua CDN

// 4. Codemod + AST cho migrate cơ học
// jscodeshift: class -> function, HOC -> hook
// npx jscodeshift -t transform/class-to-function.js src/
```

**Trade-off:** Dual-run tốn bundle (phải load cả cũ lẫn mới), tốn thời gian. Nhưng an toàn hơn big-bang rewrite 6 tháng không release được. Luôn migrate khi vẫn giao feature mới - đừng đóng băng.

**Câu hỏi đào sâu:** Làm sao thuyết phục product cho thời gian migrate? Khi nào nên dùng iframe/micro-frontend để cô lập legacy thay vì adapter?

---

### Câu 82: Refactor 500 components - làm sao không sập production?

**Trả lời Senior:**
Refactor 500 component không phải là 1 PR 200 file. Đó là **dự án cần kỷ luật**: automation, incremental, và observability. Senior làm theo pipeline: **Đo → Cô lập → Codemod → Canary → Xóa**.

1.  **Đo**: thống kê - component nào dùng nhiều nhất (`grep`, `madge`), test coverage, bundle impact. Ưu tiên component leaf ít dependency trước.
2.  **Cô lập**: tạo `shared/ui-v2` song song `shared/ui`, hoặc dùng `alias` để 2 version cùng tồn tại. Không sửa trực tiếp legacy.
3.  **Codemod + Jscodeshift/TS Morph** để đổi API hàng loạt, thay vì sửa tay.
4.  **Canary + Feature Flag**: release 5% user, đo Web Vitals, error rate qua Sentry. Rollback 1 click.
5.  **Xóa**: khi v2 đã thay hết, xóa v1.

```typescript
// 1. Song song 2 version
// shared/ui/Button.tsx (v1) - giữ nguyên
// shared/ui-v2/Button.tsx (v2) - mới

// 2. Adapter để migration dần
// shared/ui/index.ts
export { Button as ButtonV2 } from '../ui-v2/Button';
export { Button } from './Button'; // v1

// 3. Codemod ví dụ: đổi <Button type="primary"> -> <Button variant="primary">
// codemods/button-variant.js
module.exports = function(fileInfo, api) {
  const j = api.jscodeshift;
  return j(fileInfo.source)
    .find(j.JSXElement, { openingElement: { name: { name: 'Button' } } })
    .forEach(path => {
      const attr = path.value.openingElement.attributes.find(a => a.name.name === 'type');
      if (attr) { attr.name.name = 'variant'; }
    }).toSource();
};
// npx jscodeshift -t codemods/button-variant.js src --dry

// 4. Feature flag + canary
function Button(props: Props) {
  const useV2 = useFeatureFlag('button-v2');
  return useV2 ? <ButtonV2 {...props} /> : <ButtonV1 {...props} />;
}

// 5. Đo sau mỗi bước
// - Bundle analyzer: v2 có nhẹ hơn?
// - Sentry: error rate tăng?
// - Lighthouse CI: LCP có cải thiện?
```

Quy tắc vàng: **Mỗi PR < 300 dòng, có test, có screenshot Chromatic, và review 1 người**. Không refactor và thêm feature cùng PR.

**Trade-off:** Tốn thời gian duy trì 2 version, nhưng giảm rủi ro 90%. Đừng ham refactor hết 1 lần để "sạch" - business không đợi.

**Câu hỏi đào sâu:** Làm sao viết codemod an toàn cho 500 file? Khi nào nên dùng `ts-morph` thay vì `jscodeshift`?

---

### Câu 83: Micro-frontend là gì? Các pattern triển khai

**Trả lời Senior:**
Micro-frontend (MFE) là **microservice cho frontend**: chia app lớn thành nhiều app nhỏ, mỗi app deploy độc lập, team tự chủ, công nghệ có thể khác nhau, ghép lại thành 1 trải nghiệm. Bản chất là **organizational scaling**, không phải kỹ thuật cho performance.

Các pattern:

1.  **Build-time integration**: monorepo với nhiều app, build ra 1 bundle (không thực sự MFE, chỉ là modular monolith).
2.  **Runtime - Module Federation (Webpack 5)**: host import remote module qua `import('remote/Button')`, share dependency (react, router), deploy độc lập.
3.  **Runtime - Single-SPA / Qiankun**: mỗi MFE là SPA riêng, mount/unmount theo route, dùng `importmap` hoặc `systemjs`.
4.  **Server Composition - ESI / Tailor / Podium**: server ghép HTML của các MFE trước khi trả về, SEO tốt.
5.  **Iframe/Web Component**: cô lập nhất nhưng UX kém, chỉ cho legacy.

```typescript
// Module Federation - Webpack 5
// remote (team checkout) - webpack.config.js
new ModuleFederationPlugin({
  name: 'checkout',
  filename: 'remoteEntry.js',
  exposes: { './Checkout': './src/CheckoutApp' },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
});

// host (shell) - webpack.config.js
new ModuleFederationPlugin({
  name: 'host',
  remotes: { checkout: 'checkout@https://checkout.cdn.com/remoteEntry.js' },
  shared: { react: { singleton: true } },
});

// host/src/App.tsx - dùng như dynamic import
const Checkout = React.lazy(() => import('checkout/Checkout'));
function App() {
  return <Suspense fallback={<Spinner />}><Checkout /></Suspense>;
}

// Single-SPA
// checkout/src/main.tsx
export async function mount(props) { ReactDOM.createRoot(props.domElement).render(<Checkout />); }
export async function unmount(props) { /* cleanup */ }

// Cross-MFE communication - event bus hoặc shared store, không import trực tiếp
import { bus } from '@company/shared-bus';
bus.emit('cart:updated', { count: 3 });
```

**Trade-off:** MFE giải quyết vấn đề con người (team lớn, deploy độc lập) với chi phí kỹ thuật rất cao: duplicate dependency, version mismatch, UX không nhất quán, debug khó qua nhiều remote.

**Câu hỏi đào sâu:** Module Federation khác Single-SPA thế nào? Làm sao share `react` singleton mà không gây 2 version React?

---

### Câu 84: Khi nào nên dùng micro-frontend?

**Trả lời Senior:**
Chỉ nên dùng MFE khi **vấn đề tổ chức lớn hơn vấn đề kỹ thuật**. Checklist Senior phải trả lời "có" cho ít nhất 3/5:

1.  **Team > 30 dev, 4+ team độc lập**: mỗi team cần deploy không chờ team khác, release train khác nhau.
2.  **Domain ranh giới rõ**: `checkout`, `admin`, `catalog` ít share state, ít navigate chéo, có thể định nghĩa contract.
3.  **Công nghệ khác nhau bắt buộc**: 1 phần legacy AngularJS, 1 phần React mới, không thể rewrite.
4.  **Deploy độc lập là yêu cầu kinh doanh**: team checkout phải hotfix không được block bởi team search.
5.  **Đã thử modular monolith + monorepo mà vẫn đau**: CI 30 phút, merge conflict liên tục, 1 team làm sập app team khác.

Ví dụ nên dùng: Shopee/Lazada - team search, cart, payment tách biệt, mỗi team 10-15 dev, deploy hàng ngày. Ví dụ không nên: startup 10 dev, 1 product - MFE là overkill.

```typescript
// Dấu hiệu bạn CẦN MFE
// - CI build 25 phút vì 500 components
// - Team A đổi Button làm sập checkout của Team B
// - Release phải họp 5 team để đồng ý
// - Muốn thử Vue cho 1 module mà không rewrite hết

// Kiến trúc nên dùng khi quyết định MFE
// shell (host) - routing, auth, layout
//   / (catalog - team A)
//   /cart/* (cart - team B - Module Federation remote)
//   /checkout/* (checkout - team C - remote)
// Shared: design system versioned, bus, auth token

// Đo lường thành công:
// - Deploy frequency: từ 1/tuần -> 5/ngày mỗi MFE
// - MTTR: từ 2h -> 15 phút (chỉ rollback 1 MFE)
// - Bundle: host 100kb, mỗi MFE 150kb, không duplicate react (singleton)
```

**Trade-off:** Đánh đổi complexity để lấy autonomy. Nếu team nhỏ mà cố MFE, bạn sẽ trả giá complexity mà không hưởng autonomy.

**Câu hỏi đào sâu:** Làm sao thuyết phục CTO rằng MFE đáng giá? Metric nào chứng minh monolith đang kìm hãm team?

---

### Câu 85: Khi nào không nên dùng micro-frontend và alternative?

**Trả lời Senior:**
Đa số app **không nên** dùng MFE. Nếu team < 20 dev, 1 product, domain không rõ ràng, hoặc chỉ vì "thấy cool", MFE sẽ làm mọi thứ tệ hơn: bundle phình do duplicate `react`, `lodash`, UX giật khi chuyển MFE, SEO phức tạp, debug phải mở 3 repo, và **không có shared state dễ dàng**.

Dấu hiệu không nên:
- Team nhỏ, 1 backlog, 1 release train.
- Các MFE share 70% component/state (cart cần biết product, product cần biết cart) - ranh giới mờ thì MFE là distributed monolith.
- Chưa đau với monolith: build < 5 phút, deploy 1 lần/ngày là đủ.
- Chỉ muốn code splitting - dùng `React.lazy` + `import()` là đủ.

Alternative tốt hơn 90% case:

```typescript
// 1. Modular Monolith + Monorepo (Turborepo/Nx)
// apps/web - Next.js
// packages/ui - design system
// packages/utils - shared
// Mỗi team sở hữu 1 package/feature, nhưng build chung, deploy chung
// Lợi: share code dễ, 1 version react, CI cache, không runtime overhead
// apps/web/src/features/cart - team cart sở hữu
// apps/web/src/features/search - team search sở hữu

// 2. Monorepo + Codeowners + CI per feature
// .github/CODEOWNERS
// /src/features/cart/ @team-cart
// /src/features/search/ @team-search
// CI chỉ test/build feature thay đổi (Nx affected)

// 3. Vertical slice + BFF (Backend for Frontend)
// Mỗi feature có BFF riêng, frontend vẫn monolith nhưng backend tách
// Đủ để team tự chủ API mà không cần MFE

// 4. Module Federation nhưng chỉ cho 1-2 chỗ cô lập
// Chỉ tách admin hoặc legacy, còn lại giữ monolith
// Không phải all-in MFE

// So sánh
// | Tiêu chí | Monolith | MFE |
// |---|---|---|
// | Team < 15 | ✅ đơn giản | ❌ overhead |
// | Deploy | 1 lần | độc lập |
// | Bundle | 1 react | dễ duplicate |
// | Debug | dễ | khó |
// | Share state | dễ | phải bus |
```

**Trade-off:** Monolith bị chê "cũ" nhưng là lựa chọn mặc định đúng cho 90% công ty. MFE là thuốc mạnh, chỉ dùng khi bệnh nặng. Nguyên tắc Senior: **đừng phân tán hệ thống cho tới khi bạn buộc phải thế**.

**Câu hỏi đào sâu:** Làm sao biết monolith đã đến giới hạn? Nếu chỉ cần cô lập CSS/JS của 1 widget, dùng Web Component thay MFE được không?

---
