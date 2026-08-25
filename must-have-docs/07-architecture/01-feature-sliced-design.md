# Feature-Sliced Design (FSD) — Kiến trúc phân lớp app/pages/widgets/features/entities/shared với ranh giới import một chiều

> Tags: #FSD #Architecture #Feature-Sliced #Layers #Boundaries #ESLint #Import-Rules | Nguồn: `docs/04-frontend-architecture.md` câu 69-71 | Mức: P0

## 1. Định nghĩa chính xác

**Feature-Sliced Design (FSD)** là methodology tổ chức codebase theo **layers** (tầng) → **slices** (lát cắt domain) → **segments** (`ui`, `model`, `api`, `lib`, `config`). 6 layers chuẩn: **`app` → `pages` → `widgets` → `features` → `entities` → `shared`** với **quy tắc import một chiều**: tầng trên được import tầng dưới, tầng dưới **không được** import tầng trên; `features` **không import chéo** nhau; `shared` và `entities` không import từ `features/widgets/pages/app`.

Mục tiêu: **high cohesion — low coupling**, xóa 1 feature = xóa 1 folder, junior không thể vô tình tạo dependency vòng qua `eslint boundaries`.

## 2. Cơ chế hoạt động

### 2.1 Layers và trách nhiệm

- **`app`** — providers, router, store setup, global styles. Chỉ khởi tạo, không chứa business.
- **`pages`** — route composition: ghép `widgets`/`features` thành trang. Không chứa logic domain sâu.
- **`widgets`** — khối UI lớn ghép nhiều features: `Header`, `CartSidebar`, `ProductListWithFilters`. Có thể compose `features` + `entities`.
- **`features`** — hành vi nghiệp vụ có giá trị user: `auth`, `checkout`, `search`, `add-to-cart`. Chứa `ui` + `model` (hooks/store) + `api` + `lib` riêng. Có `public API` qua `index.ts`.
- **`entities`** — domain model thuần, không gắn use-case: `user`, `product`, `order`. Chỉ data + helper thuần (`formatProduct`).
- **`shared`** — code không mang nghiệp vụ: `ui` (Button, Input), `api` (axios client), `lib` (formatMoney, date), `config`. Không import từ tầng trên.

Hướng phụ thuộc:

```
app → pages → widgets → features → entities → shared
 ↑ cấm import ngược ─────────────────────────────────┘
 features/* ──X──> features/*  (cấm chéo, phải qua shared/entities hoặc pages/widgets)
```

### 2.2 Public API và Dependency Rule

Mỗi slice chỉ export qua `index.ts` — consumer không được import sâu `features/checkout/ui/CheckoutForm.tsx` mà phải `features/checkout`. Giúp đổi cấu trúc nội bộ không vỡ import và dễ phát hiện import chéo.

Kỹ thuật tránh dependency chéo giữa features:

1. **Promote xuống `entities`** — nếu `cart` và `product` cùng dùng `Product` type, đưa `Product` xuống `entities/product`.
2. **Dependency Inversion qua props/slot** — `features/cart/ui/CartItem` nhận `renderProduct?: (p: Product) => ReactNode` thay vì import `features/product` trực tiếp; `pages/cart` (tầng cao hơn) inject.
3. **Event/Mediator** — side-effect qua `shared/events/bus` (mitt) hoặc shared store, không import trực tiếp.

### 2.3 Enforce bằng ESLint boundaries

- `eslint-plugin-boundaries` hoặc `import/no-restricted-imports` + `eslint-import-resolver-typescript`.
- Rule: `shared` không được import từ `features|widgets|pages|app`; `entities` không import `features|widgets|pages|app`; `features/*` không import `features/*`.
- CI chạy `madge --circular src/` để phát hiện vòng tròn.

Sơ đồ:

```
src/
  app/providers/router.tsx      → import pages, widgets
  pages/cart/page.tsx           → import widgets/cart-summary, features/checkout, entities/product
  widgets/header/index.ts       → import features/search, features/auth, shared/ui
  features/checkout/{ui,model,api,lib,index.ts} → import entities/product, entities/order, shared/*
  entities/product/{model,ui}   → import shared/*
  shared/{ui,api,lib,config}   → không import tầng trên
```

## 3. Ví dụ tối thiểu

```ts
// shared/api/client.ts — không import từ trên
import axios from 'axios';
export const api = axios.create({ baseURL: '/api' });

// entities/product/model/types.ts
export type Product = { id: string; name: string; price: number };

// features/checkout/api/checkoutApi.ts — chỉ import shared/entities
import { api } from '@/shared/api/client';
import type { Product } from '@/entities/product';
export const checkoutApi = {
  create: (items: Product[]) => api.post('/checkout', { items }).then(r => r.data),
};

// features/checkout/ui/CheckoutForm.tsx — inversion, không import features/product
import type { Product } from '@/entities/product';
type Props = { products: Product[]; renderProduct?: (p: Product) => React.ReactNode };
export const CheckoutForm = ({ products, renderProduct }: Props) => (
  <form>{products.map(p => <div key={p.id}>{renderProduct ? renderProduct(p) : p.name}</div>)}</form>
);

// features/checkout/index.ts — public API
export { CheckoutForm } from './ui/CheckoutForm';
export { checkoutApi } from './api/checkoutApi';

// ❌ Sai — feature import chéo feature
// import { ProductCard } from '@/features/product/ui/ProductCard';

// ✅ Đúng — pages/widgets compose
// pages/cart/page.tsx
import { CheckoutForm } from '@/features/checkout';
import { ProductCard } from '@/features/product';
import type { Product } from '@/entities/product';
export const CartPage = ({ products }: { products: Product[] }) => (
  <CheckoutForm products={products} renderProduct={p => <ProductCard product={p} />} />
);
```

```js
// .eslintrc.js — boundary lint
module.exports = {
  plugins: ['boundaries', 'import'],
  settings: { 'boundaries/elements': [
    { type: 'app', pattern: 'app/*' },
    { type: 'pages', pattern: 'pages/*' },
    { type: 'widgets', pattern: 'widgets/*' },
    { type: 'features', pattern: 'features/*' },
    { type: 'entities', pattern: 'entities/*' },
    { type: 'shared', pattern: 'shared/*' },
  ]},
  rules: {
    'boundaries/element-types': ['error', {
      default: 'disallow',
      rules: [
        { from: 'shared', allow: ['shared'] },
        { from: 'entities', allow: ['shared', 'entities'] },
        { from: 'features', allow: ['shared', 'entities'] },
        { from: 'widgets', allow: ['shared', 'entities', 'features'] },
        { from: 'pages', allow: ['shared', 'entities', 'features', 'widgets'] },
        { from: 'app', allow: ['shared', 'entities', 'features', 'widgets', 'pages'] },
      ],
    }],
    'no-restricted-imports': ['error', { patterns: [
      // cấm import sâu, chỉ qua index.ts (tuỳ rule)
      // { group: ['@/features/*/*'] } // nếu muốn ép public API
    ]}],
    'import/no-cycle': ['error', { maxDepth: Infinity }],
  },
};
```

```bash
# Phát hiện vòng tròn
npx madge --circular --extensions ts,tsx src/
# Kiểm tra boundary
npx eslint src --ext .ts,.tsx
```

## 4. So sánh / Phân loại

| Tiêu chí | FSD / Feature-based | Layer-by-Type (`components/hooks/services/utils`) |
|----------|---------------------|---------------------------------------------------|
| Chia theo | Nghiệp vụ (vertical slice): `features/checkout` chứa đủ ui/model/api | Kỹ thuật (horizontal): `components`, `hooks`, `services` |
| Cohesion | Cao — đổi checkout chỉ chạm 1 folder | Thấp — đổi checkout chạm 4 folder (`components`, `hooks`, `services`, `utils`) |
| Coupling | Thấp — features cô lập, xóa 1 folder là hết | Cao — `utils/formatMoney` dùng cho `order` nằm `shared` ai cũng import, khó xóa |
| Khả năng scale | Tốt >50 components / >5 dev | Tốt <30 components, nhanh khởi tạo |
| Discoverability | Người mới nhìn cây thư mục đoán được feature nằm đâu | Phải nhớ loại file, `components` thành bãi rác 200 file |
| Import rule | Ràng buộc rõ, lint enforce được | Không có ranh giới, dễ import vòng |

| Tầng | Được import từ | Không được import từ |
|------|----------------|----------------------|
| `shared` | `shared` | `entities`, `features`, `widgets`, `pages`, `app` |
| `entities` | `shared`, `entities` | `features`, `widgets`, `pages`, `app` |
| `features` | `shared`, `entities` | `features` khác, `widgets`, `pages`, `app` |
| `widgets` | `shared`, `entities`, `features` | `pages`, `app` |
| `pages` | `shared`, `entities`, `features`, `widgets` | `app` |
| `app` | tất cả | — |

| Cách giải quyết dependency chéo | Khi dùng | Trade-off |
|---------------------------------|----------|-----------|
| Promote xuống `entities` | Type/model dùng chung | `entities` phình nếu promote bừa |
| Inversion qua props/slot | UI cần compose | Props API rộng hơn |
| Event bus / shared store | Side-effect (`cartUpdated`) | Flow khó trace, phải document event |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng FSD nguyên xi cho app 5 trang / <20 components / 1-2 dev**: boilerplate `features/X/{ui,model,api}` thừa, chậm. Bắt đầu layer-based đơn giản (`src/components`, `src/lib`), khi >50 components hoặc >5 dev mới refactor sang feature.
- **Không tách `shared/ui` quá sớm thành package riêng**: chỉ tách khi có ≥2 apps hoặc design system versioned cần release. Tách sớm tốn versioning, sync.
- **Không ép mỗi feature có đủ 4 segments**: feature nhỏ chỉ cần `ui` + `index.ts`, đừng tạo `model/api/lib` rỗng cho đủ form.
- **Chi phí**: thêm convention, cần onboard ESLint boundaries, review chặt public API `index.ts`. Đổi lại: giảm 60% thời gian tìm file, giảm bug import vòng, CI phát hiện sớm.
- **Không dùng `features` để chứa domain thuần**: `Product` type phải ở `entities`, không phải `features/product` rồi để `features/cart` import chéo.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `features/cart` import `features/product` trực tiếp → vòng tròn**
  - Triệu chứng: `madge --circular` báo `features/cart → features/product → features/cart`; test phải mock cả cụm; xóa 1 feature vỡ feature kia.
  - Fix: promote `Product` xuống `entities/product` hoặc inversion qua `renderProduct` prop; event bus cho side-effect.
  - Đo: `npx madge --circular --extensions ts,tsx src/` + `eslint-plugin-import` `import/no-cycle` trong CI.

- **Lỗi 2: `shared` import từ `features` (rò business vào shared)**
  - Triệu chứng: `shared/ui/Button` import `useCartStore` → `shared` phụ thuộc `features`, không reuse được.
  - Fix: `shared` chỉ nhận props, không import store feature; logic cart ở `features/cart`, `shared` giữ primitive.
  - Đo: `eslint boundaries/element-types` báo `shared → features` disallow; `madge --image graph.svg src/shared`.

- **Lỗi 3: Import sâu bypass public API (`features/checkout/ui/internal`)**
  - Triệu chứng: đổi cấu trúc nội bộ vỡ 20 file import sâu.
  - Fix: chỉ export qua `features/checkout/index.ts`, bật `no-restricted-imports` cấm `@/features/*/*` hoặc `eslint-plugin-boundaries` `allow only index`.
  - Đo: `grep -r "from '@/features/.*/" src | grep -v "from '@/features/\w\+"` ; ESLint `boundaries/entry-point`.

- **Lỗi 4: `widgets`/`pages` phình thành God Component vì nhét logic**
  - Triệu chứng: `pages/cart/page.tsx` 600 dòng fetch + validate + render.
  - Fix: pages chỉ compose; logic vào `features/*/model`, UI vào `features/*/ui`.
  - Đo: `npx eslint --rule 'max-lines: [error, 300]'` , `wc -l src/pages/**/page.tsx`.

- **Lỗi 5: Monolith `components` 500 file không phân biệt feature**
  - Triệu chứng: không biết component thuộc feature nào, sợ xóa.
  - Fix: migrate incremental sang FSD, mỗi sprint chuyển 1 feature, dùng `CODEOWNERS` `/src/features/cart @team-cart`.
  - Đo: `ls src/components | wc -l` , bundle duplicate check `madge --orphans`.

- **Công cụ**:
  - `npx madge --circular --extensions ts,tsx src/` — vòng tròn.
  - `npx madge --image graph.svg src/` — graph.
  - `eslint-plugin-boundaries`, `import/no-restricted-imports`, `import/no-cycle`.
  - `npx eslint src --ext .ts,.tsx` trong CI, fail nếu vi phạm layer.
  - `CODEOWNERS` + `Nx affected` để ownership per feature.

## 7. Câu hỏi tự kiểm tra

1. Kể 6 layers FSD và quy tắc import một chiều — vì sao `shared` không được import từ `features`, và `features` không được import chéo nhau? Enforce bằng gì?
2. Phân biệt FSD/feature-based vs layer-by-type (`components/hooks/services`) — khi nào layer-by-type đủ và khi nào phải chuyển sang feature-based? Trade-off DRY vs isolation?
3. Có 2 features `cart` cần hiển thị `ProductCard` của `product` — nêu 3 cách tránh import chéo trực tiếp và khi nào dùng promotion xuống `entities`, inversion qua props, hay event bus?

<details>
<summary>Đáp án 30s</summary>

1. **Layers**: `app` (providers/router) → `pages` (route composition) → `widgets` (khối ghép nhiều features) → `features` (use-case: checkout/search) → `entities` (domain thuần: product/order) → `shared` (ui/api/lib/config). Quy tắc: tầng trên được import tầng dưới, **không ngược lại**; `shared` ở đáy nên không import `entities/features/widgets/pages/app`; `features` không import `features` khác để tránh **circular dependency** và **high coupling** — xóa 1 feature không vỡ feature khác. Enforce bằng **`eslint-plugin-boundaries` (`element-types`) + `import/no-restricted-imports` + `import/no-cycle`** và CI `madge --circular`.
2. **Layer-by-type** chia theo kỹ thuật (`components`, `hooks`, `services`) — dễ bắt đầu, DRY, nhưng cohesion thấp: đổi `checkout` phải đụng 4 folder, `components` thành bãi rác 200 file. **Feature-based/FSD** chia theo nghiệp vụ — mỗi feature chứa đủ `ui/model/api/lib`, **high cohesion**, isolation tốt, xóa gọn, discoverability cao. Dùng layer-by-type khi **<30 components / <5 dev / app 5 trang**; chuyển sang feature khi **>50 components / >5 dev / nhiều domain** hoặc CI/merge bắt đầu đau. Trade-off: feature-based dư boilerplate và hơi duplicate `utils` nhưng đổi lại **low coupling**.
3. **(a) Promote xuống `entities`** — nếu chỉ cần type `Product` thuần, đưa `entities/product` (domain không logic feature). **(b) Inversion qua props/slot** — `CartItem` nhận `renderProduct?: (p: Product) => ReactNode`, `pages/cart` inject `<ProductCard>`; dùng cho **UI composition**, giữ features cô lập, trace rõ. **(c) Event bus / shared store** — `shared/events/bus` (`mitt`) `emit('cartUpdated')`, dùng cho **side-effect** không cần render. Ưu tiên **inversion cho UI**, **event cho side-effect**, **entities cho data thuần**; import trực tiếp `features/product → features/cart` là **cấm** vì tạo vòng tròn, build chậm, test khó.

</details>

---
*Tham khảo chi tiết: `docs/04-frontend-architecture.md` — Câu 69, 70, 71. Spec: [Feature-Sliced Design](https://feature-sliced.design/), `eslint-plugin-boundaries`, `madge`, `import/no-restricted-imports`.*

