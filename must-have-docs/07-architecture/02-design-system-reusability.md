# Design System & Reusability — Tokens, Rule of Three, Headless vs Styled, Compound Components

> Tags: #Design-System #Tokens #Reusability #Rule-Of-Three #Headless #Compound-Components #God-Component #Storybook | Nguồn: `docs/04-frontend-architecture.md` câu 72-74 | Mức: P0

## 1. Định nghĩa chính xác

**Design System** là **hợp đồng giữa design và code**: `tokens` (single source of truth) → `primitives` (Box, Text, Button) → `compound components` (Select, Dialog, Card) → `patterns` (Form Layout, Data Table) cùng quy tắc sử dụng versioned và visual-tested.

**Reusable component tốt** là component **dùng lại được mà không sửa source**: thỏa **Single Responsibility**, **Inversion of Control** (caller quyết định phần biến đổi qua `children/slot/render prop`), **Stable API** (ít props, không breaking mỗi sprint), **không rò business** (không import `useCartStore` trong `Card`).

**Rule of Three**: chỉ abstract khi đã có **3 use case thật**, không phải 1 use case + 2 tưởng tượng.

**God Component** là component 800-2000 dòng ôm mọi thứ (fetch, validate, render 5 section, 12 `useState`, 8 `useEffect`) — dấu hiệu thiếu ranh giới.

## 2. Cơ chế hoạt động

### 2.1 Token → Primitive → Compound → Pattern

1. **Tokens** — `color.primary.500`, `space.4`, `radius.md`, `font.heading`. Đồng bộ Figma → code qua **Style Dictionary** / **Tokens Studio**, không hardcode `16px`, `#0066ff`.
2. **Primitives** — `Box`, `Text`, `Button`, `Input`: chỉ nhận token (`variant`, `size`), không business. Style qua `data-variant` + CSS variables hoặc `cva`.
3. **Compound Components** — ghép primitives + logic: `Select = Root + Trigger + Content + Item`, dùng **Context** nội bộ để share state, cho phép caller sắp xếp linh hoạt.
4. **Patterns** — bố cục cấp cao: `FormLayout`, `DataTable` với `filters + pagination + empty`.

### 2.2 Headless vs Styled

- **Headless** (Radix, Headless UI, Ariakit, React Aria): chỉ logic + a11y + keyboard, **không CSS**. Team tự style bằng tokens → đổi theme không vỡ logic, dễ brand.
- **Styled sẵn** (MUI, AntD): logic + style gắn liền → nhanh nhưng khó custom theme, bundle nặng, khó thoát.
- Thực chiến: **headless + tokens** cho design system riêng; styled sẵn chỉ khi 1 app nội bộ không cần brand.

### 2.3 Composition over Configuration — Compound & Slots

- Thay 15 props boolean (`isCart`, `showDiscount`, `isWishlist`) bằng **slots**: `Card` nhận `children` + `actions`, `Button` nhận `leftIcon`/`isLoading` composition.
- **Polymorphic `as` prop**: `<Text as="h1" variant="heading">` với `React.ComponentPropsWithoutRef<C>` để type an toàn.
- **Compound**: `<Select><Select.Trigger/><Select.Content><Select.Item/></Select.Content></Select>` — caller quyết định layout, không phải 20 props config.

### 2.4 Tách God Component — 3 trục

- **Logic → Hook/Store** (`useDashboard`, `useQuery`)
- **UI → Sub-component + Composition** (`UserHeader`, `OrderTable`, `OrderDetailModal`) nhận props
- **Side-effect → Service** (`fetchOrders` trong `api/`)

Sơ đồ:

```
Figma Tokens (Style Dictionary) → tokens.ts → primitives (Button/Box) → compounds (Dialog/Select via Radix headless) → patterns (Form, Table)
Storybook (docs + visual) → Chromatic (visual regression) → Release @company/ui@x.y.z → Codemod khi breaking
```

## 3. Ví dụ tối thiểu

```ts
// 3.1 Tokens — single source
export const tokens = {
  color: { primary: { 500: '#0066ff', 600: '#0052cc' } },
  space: { 4: '16px', 6: '24px' },
  radius: { md: '8px' },
} as const;

// 3.2 Primitive — chỉ token, không business
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
```

```tsx
// 3.3 Headless + Compound — Radix Dialog
import * as Dialog from '@radix-ui/react-dialog';
export const Modal = ({ children, ...props }: Dialog.DialogProps) => (
  <Dialog.Root {...props}>
    <Dialog.Portal>
      <Dialog.Overlay className="overlay" />
      <Dialog.Content className="content">{children}</Dialog.Content>
    </Dialog.Portal>
  </Dialog.Root>
);

// 3.4 Compound Card — composition thay flags
type CardProps = { children: React.ReactNode; actions?: React.ReactNode };
export const Card = ({ children, actions }: CardProps) => (
  <div className="card">
    <div className="body">{children}</div>
    {actions && <div className="actions">{actions}</div>}
  </div>
);
// Dùng — caller quyết định
<Card actions={<Button onClick={addToCart}>Add</Button>}><ProductInfo product={p} /></Card>
<Card actions={<Button variant="ghost" onClick={removeWishlist}>Remove</Button>}><ProductInfo product={p} /></Card>

// ❌ Tệ — God-style với 15 flags, rò business
function BadCard({ product, isCart, isWishlist, showDiscount }: any) {
  if (isCart) return <div>cart</div>;
  if (isWishlist) return <div>wish</div>;
}

// 3.5 Compound Select — Context nội bộ
const SelectCtx = React.createContext<{ value: string; onValueChange: (v: string) => void }>(null!);
export const Select = {
  Root: ({ children, value, onValueChange }: any) => (
    <SelectCtx.Provider value={{ value, onValueChange }}>{children}</SelectCtx.Provider>
  ),
  Trigger: ({ children }: any) => {
    const { value } = React.useContext(SelectCtx);
    return <button>{children ?? value}</button>;
  },
  Item: ({ value, children }: any) => {
    const { onValueChange } = React.useContext(SelectCtx);
    return <div onClick={() => onValueChange(value)}>{children}</div>;
  },
};
// <Select.Root value={v} onValueChange={setV}><Select.Trigger/><Select.Item value="a">A</Select.Item></Select.Root>

// 3.6 Polymorphic Text — type-safe as
type TextProps<C extends React.ElementType> = {
  as?: C;
  variant?: 'body' | 'heading';
} & React.ComponentPropsWithoutRef<C>;
export function Text<C extends React.ElementType = 'span'>({ as, variant = 'body', ...props }: TextProps<C>) {
  const Comp = as ?? 'span';
  return <Comp data-variant={variant} {...props} />;
}
// <Text as="h1" variant="heading">Title</Text> — props của h1 được infer

// 3.7 Tách God Component
// ❌ God
function BadDashboard() {
  const [user, setUser] = useState(null);
  const [orders, setOrders] = useState([]);
  const [filter, setFilter] = useState('');
  useEffect(() => { fetchUser().then(setUser); }, []);
  useEffect(() => { fetchOrders(filter).then(setOrders); }, [filter]);
  return <div>{/* 600 dòng JSX */}</div>;
}
// ✅ Tách
function useDashboard(filter: string) {
  const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser });
  const { data: orders } = useQuery({ queryKey: ['orders', filter], queryFn: () => fetchOrders(filter) });
  return { user, orders };
}
function Dashboard() {
  const [filter, setFilter] = useState('');
  const { user, orders } = useDashboard(filter);
  return (
    <Layout>
      <UserHeader user={user} />
      <OrderFilter value={filter} onChange={setFilter} />
      <OrderTable orders={orders} />
    </Layout>
  );
}
```

## 4. So sánh / Phân loại

| Tiêu chí | Headless (Radix/Ariakit) | Styled sẵn (MUI/AntD) |
|----------|--------------------------|-----------------------|
| Logic/a11y | Có, tách khỏi style | Có, gắn với style |
| Style | Team tự style bằng tokens, đổi theme dễ | Theme override phức tạp, bundle lớn |
| Bundle | Nhẹ, tree-shakable | Nặng, khó thoát khi cần brand riêng |
| Khi dùng | Design system riêng, >1 product/brand | App nội bộ 1 brand, cần nhanh |

| Reusable tốt | Reusable tệ (God) |
|--------------|-------------------|
| Single Responsibility, 1 việc | 800 dòng, fetch + validate + render 5 section |
| Inversion of Control qua `children/slot/render prop` | 15 props boolean `isCart/isWishlist/showDiscount` + `if (variant === 'cart')` |
| Stable API, ít props, không breaking | Mỗi sprint thêm prop mới, breaking |
| Không import `useCartStore` bên trong | Import store trực tiếp, rò business |
| Type-safe `as` polymorphic | `any` props |

| Kỹ thuật | Mục đích | Ví dụ |
|----------|----------|-------|
| Tokens | Single source, đồng bộ Figma | `color.primary.500`, `space.4` |
| Primitive | Nền tảng không business | `Button`, `Box`, `Text` |
| Compound | Ghép primitive, flexible layout | `Select.Root/Trigger/Content/Item`, `Card + CardHeader + CardActions` |
| Pattern | Bố cục cấp cao | `FormLayout`, `DataTable` |

| Khi nào tách compound (`Card` → `CardHeader`/`CardActions`) | Khi nào giữ 1 component |
|---|---|
| Cần caller sắp xếp lại layout linh hoạt | Chỉ 1 layout cố định, không cần reorder |
| Có 3+ use case layout khác nhau | <3 use case, tách sớm làm API phức tạp |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không xây Design System khi chỉ 1 app / 1 team / <3 tháng**: tốn 30-40% thời gian đầu, cần Storybook + Chromatic + versioning + codemod. Thay bằng **Tailwind + Radix headless** là đủ. Chỉ đáng khi **>2 team** hoặc **>1 product** hoặc brand cần nhất quán.
- **Không abstract theo Rule of Three**: 1 use case thật + 2 tưởng tượng → đừng tạo `GenericCard` với 10 props. Đợi có 3 use case thật rồi extract, nếu không sẽ có API phức tạp khó dùng.
- **Không headless khi cần velocity cao**: headless đòi tự style mọi state (hover, focus, disabled) qua tokens — chậm hơn MUI. Nếu deadline 2 tuần, dùng styled sẵn.
- **Không compound quá mức**: `Card` tách thành 5 sub-components khi chỉ có 1 layout → prop drilling, khó theo dõi. Cân bằng: tách khi layout biến đổi thật.
- **Không polymorphic `as` nếu không cần semantic**: `as` làm TypeScript phức tạp (`ComponentPropsWithoutRef`), chỉ dùng cho `Text`, `Heading`, `Box`.
- **God Component**: không rewrite 1 lần 2000 dòng — extract từng phần + test + ship, dùng `React.lazy` tạm thời. Tách nhỏ quá cũng tạo prop drilling mới — cân bằng với colocated state / Context.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Hardcode `16px`, `#0066ff` rải rác, không tokens**
  - Triệu chứng: đổi brand phải grep 100 file, màu lệch giữa Figma và code.
  - Fix: tokens qua Style Dictionary, lint cấm literal: `eslint --rule 'no-restricted-syntax: [error, { selector: "Literal[value=16]" }]'` (tuỳ) hoặc `stylelint`.
  - Đo: `grep -r "#[0-9a-fA-F]\{6\}" src/shared/ui | wc -l` , Figma Tokens Studio diff.

- **Lỗi 2: Reusable component rò business (`Card` import `useCartStore`)**
  - Triệu chứng: `Card` không dùng được ngoài cart, test phải mock store.
  - Fix: inversion — `Card` nhận `actions: ReactNode` / `onAction`, caller inject `Button`.
  - Đo: `madge --circular`, `eslint import/no-restricted-imports` cấm `shared/ui` import `features/*`.

- **Lỗi 3: Props explosion — 15 boolean flags**
  - Triệu chứng: `if (isCart) ... if (isWishlist) ...`, mỗi variant thêm 1 `if`.
  - Fix: composition — `Card` + `CardActions`, hoặc `render prop`.
  - Đo: `npx eslint --rule 'max-lines-per-component'` , review props count >7 là mùi.

- **Lỗi 4: God Component 800+ dòng, 12 useState, 8 useEffect**
  - Triệu chứng: thay 1 dòng sợ vỡ 3 chỗ, test phải mock cả thế giới, scroll 5 phút chưa hết file.
  - Fix: extract theo 3 trục: logic → `useDashboard` hook, UI → `UserHeader/OrderTable`, side-effect → `api/`.
  - Đo: `wc -l src/pages/**/Dashboard.tsx`, `npx eslint --rule 'max-lines: [error, 300]'` , complexity `npx eslint --rule 'complexity: [error, 10]'` , `wdyr` / Profiler re-render.

- **Lỗi 5: Headless nhưng thiếu a11y/keyboard**
  - Triệu chứng: Select tự code không có `aria-*`, không focus trap.
  - Fix: dùng Radix/Ariakit headless đã lo a11y, style chỉ qua `data-state`.
  - Đo: `axe-core` in Storybook, keyboard test Tab/Arrow/Escape.

- **Lỗi 6: Versioning design system làm app kẹt ở version cũ**
  - Triệu chứng: `@company/ui@1.0` và `@company/ui@2.0` cùng tồn tại, bundle duplicate.
  - Fix: semantic versioning + **codemod** (`jscodeshift`) khi breaking, `peerDependencies` cho `react`.
  - Đo: `npm ls @company/ui`, `bundle-analyzer` check duplicate, `npx madge --orphans`.

- **Công cụ**:
  - **Storybook + Chromatic** — visual regression.
  - **Style Dictionary / Tokens Studio** — sync Figma → code.
  - **bundle-analyzer / `npx madge`** — duplicate, circular.
  - **axe-core**, **eslint `max-lines`, `complexity`**, **Profiler**.

## 7. Câu hỏi tự kiểm tra

1. Nêu thứ tự Token → Primitive → Compound → Pattern và vì sao Design System phải bắt đầu từ tokens? Headless (Radix) khác styled sẵn (MUI) thế nào và khi nào chọn headless?
2. Thế nào là reusable component tốt theo 4 tiêu chí? Vì sao Rule of Three ngăn over-abstract và vì sao `isCart/isWishlist` 15 flags là mùi?
3. God Component có dấu hiệu gì và tách theo 3 trục nào? Khi nào nên tách `Card` thành `Card + CardHeader + CardActions` compound thay vì giữ 1 component?

<details>
<summary>Đáp án 30s</summary>

1. **Token** là single source (`color.primary.500`, `space.4`) đồng bộ Figma qua Style Dictionary — đổi 1 token đổi cả hệ thống, tránh hardcode `#0066ff/16px`. **Primitive** (`Button`, `Box`) chỉ nhận token, không business. **Compound** ghép primitive (`Select.Root/Trigger/Content/Item` share Context). **Pattern** là bố cục cao (`FormLayout`). **Headless** (Radix/Ariakit) chỉ logic + a11y, không CSS → team tự style bằng tokens, đổi theme không vỡ logic, bundle nhẹ; **styled sẵn** (MUI/AntD) logic+style gắn liền → nhanh nhưng khó custom brand, bundle nặng. Chọn **headless + tokens** khi **>1 product/brand hoặc cần design system riêng**; chọn styled sẵn khi **1 app nội bộ cần velocity**.
2. **Tốt**: **Single Responsibility** (1 việc), **Inversion of Control** (caller quyết định qua `children/slot/render prop`), **Stable API** (ít props, không breaking), **Không rò business** (không `import useCartStore` trong Card). **Rule of Three**: chỉ abstract khi có **3 use case thật** — tránh generic sớm làm API phức tạp (`as` polymorphic, 10 props) cho 1 use case + 2 tưởng tượng. **15 flags** (`isCart`, `showDiscount`) là mùi vì component cố làm 3 việc qua `if (variant)`, nên thay bằng **composition**: `Card` nhận `children` + `actions: ReactNode`, caller inject `Button`.
3. **God**: 800-2000 dòng, 12 `useState`, 8 `useEffect`, fetch+validate+render 5 section trong 1 file, props drilling 4 tầng, test mock cả thế giới. **Tách 3 trục**: **Logic → Hook/Store** (`useDashboard` với `useQuery`), **UI → Sub-component + Composition** (`UserHeader`, `OrderTable` nhận props), **Side-effect → Service** (`api/fetchOrders`). Tách `Card` thành compound khi **caller cần reorder layout linh hoạt** và có **≥3 layout khác nhau**; giữ 1 component khi chỉ 1 layout cố định — tách sớm chỉ để file ngắn làm flow khó trace và prop drilling mới.

</details>

---
*Tham khảo chi tiết: `docs/04-frontend-architecture.md` — Câu 72, 73, 74. Spec: [Radix UI](https://www.radix-ui.com/), [Style Dictionary](https://amzn.github.io/style-dictionary/), [Storybook](https://storybook.js.org/), [Chromatic](https://www.chromatic.com/).*

