# Micro-frontend & Monorepo — Khi nào MFE >30 devs 4 teams, Turborepo/Nx, Version Skew, Module Federation

> Tags: #Micro-Frontend #Monorepo #Turborepo #Nx #Module-Federation #Version-Skew #Single-SPA #CODEOWNERS | Nguồn: `docs/04-frontend-architecture.md` câu 83-85 | Mức: P0

## 1. Định nghĩa chính xác

**Micro-frontend (MFE)** là **microservice cho frontend**: chia app lớn thành nhiều app nhỏ, mỗi app **deploy độc lập**, team **tự chủ** (backlog, release train), có thể khác stack, ghép lại thành một trải nghiệm. Bản chất là **organizational scaling**, không phải tối ưu performance.

**Monorepo** là 1 repo chứa nhiều `apps` + `packages` (Turborepo/Nx): **modular monolith** — mỗi team sở hữu 1 `package/feature` nhưng **build chung, deploy chung**, share `react` singleton, không runtime overhead.

**Module Federation (Webpack 5 / Rspack)** là runtime integration: `host` import remote qua `import('checkout/Checkout')`, share dependency `react` singleton, deploy độc lập. **Version skew** là khi host và remote lệch version `react`/`design-system` gây duplicate hoặc break.

## 2. Cơ chế hoạt động

### 2.1 Các pattern MFE

1. **Build-time (modular monolith)** — monorepo nhiều apps, build ra 1 bundle. Không thực sự MFE, chỉ module.
2. **Runtime — Module Federation** — `remote` expose `./Checkout`, `host` import dynamic `React.lazy(() => import('checkout/Checkout'))`, `shared: { react: { singleton: true } }`.
3. **Runtime — Single-SPA / Qiankun** — mỗi MFE là SPA riêng, `mount`/`unmount` theo route, dùng `importmap`/`systemjs`.
4. **Server Composition — ESI/Tailor/Podium** — server ghép HTML các MFE trước khi trả về, SEO tốt.
5. **Iframe/Web Component** — cô lập nhất nhưng UX kém, chỉ cho legacy.

### 2.2 Monorepo (Turborepo/Nx)

```
apps/web (Next.js host)
apps/admin
packages/ui (design system)
packages/utils
packages/shared-bus (mitt)
```

- **Turborepo** — `turbo.json` pipeline `build`, `test` với **cache** (hash input), `affected` qua `Nx`.
- **Nx** — `nx affected --target=build` chỉ build feature đổi, **CODEOWNERS** `/src/features/cart @team-cart`, **CI per feature**.
- Lợi: **1 version react**, share code dễ, không duplicate, debug dễ, CI cache 30% → 5 phút.

### 2.3 Module Federation — share singleton

```js
// remote webpack.config.js
new ModuleFederationPlugin({
  name: 'checkout',
  filename: 'remoteEntry.js',
  exposes: { './Checkout': './src/CheckoutApp' },
  shared: { react: { singleton: true, requiredVersion: '^18.0.0' }, 'react-dom': { singleton: true } },
});
// host
new ModuleFederationPlugin({
  name: 'host',
  remotes: { checkout: 'checkout@https://checkout.cdn.com/remoteEntry.js' },
  shared: { react: { singleton: true } },
});
```

Nếu `singleton: true` nhưng version lệch major → runtime warning/error, phải **align version** hoặc **strictVersion**.

### 2.4 Version Skew & Contract

MFE gặp **duplicate `react`** (2 copy → hooks fail), **design system lệch** (`Button` v1 vs v2), **bus contract vỡ**. Fix: `peerDependencies`, `shared` singleton, versioned bus (`@company/shared-bus@x`), **BFF contract** rõ, **canary + feature flag**.

Sơ đồ:

```
Monorepo (Turborepo): apps/web → packages/ui, packages/utils — build chung, 1 react, CODEOWNERS
MFE Module Federation: host (shell: routing, auth, layout) → remote checkout (https://checkout.cdn.com/remoteEntry.js) → import('checkout/Checkout') — deploy độc lập, shared singleton
```

## 3. Ví dụ tối thiểu

```ts
// 3.1 Monorepo — Turborepo/Nx
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"], "cache": true },
    "test": { "dependsOn": ["build"], "cache": true },
    "lint": { "cache": true }
  }
}
// package.json scripts
// "build": "turbo run build --filter=...[HEAD^1]"  (affected)
// Nx: npx nx affected --target=build --base=main

// .github/CODEOWNERS
// /src/features/cart/ @team-cart
// /src/features/search/ @team-search
// /packages/ui/ @team-design-system

// apps/web/src/features/cart — team cart sở hữu, nhưng build chung
// Lợi: 1 react, không runtime overhead, CI cache

// 3.2 Module Federation — host + remote
// remote (team checkout) — webpack.config.js / rspack
const { ModuleFederationPlugin } = require('webpack').container;
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'checkout',
      filename: 'remoteEntry.js',
      exposes: { './Checkout': './src/CheckoutApp' },
      shared: { react: { singleton: true, requiredVersion: '^18.0.0' }, 'react-dom': { singleton: true } },
    }),
  ],
};
// remote/src/CheckoutApp.tsx
export default function CheckoutApp() { return <div>Checkout MFE</div>; }

// host (shell) — webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: { checkout: 'checkout@https://checkout.cdn.com/remoteEntry.js' },
      shared: { react: { singleton: true, requiredVersion: '^18.0.0' } },
    }),
  ],
};
// host/src/App.tsx
import React, { Suspense } from 'react';
const Checkout = React.lazy(() => import('checkout/Checkout'));
export function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Checkout />
    </Suspense>
  );
}

// 3.3 Cross-MFE communication — không import trực tiếp, dùng bus
// packages/shared-bus — versioned
import mitt from 'mitt';
export type Events = { 'cart:updated': { count: number }; 'auth:logout': void };
export const bus = mitt<Events>();
// features/cart (remote) → bus.emit('cart:updated', { count: 3 })
// widgets/header (host) → bus.on('cart:updated', handler)

// 3.4 Single-SPA (alternative)
// checkout/src/main.tsx
export async function bootstrap() {}
export async function mount(props: { domElement: HTMLElement }) {
  ReactDOM.createRoot(props.domElement).render(<Checkout />);
}
export async function unmount(props: { domElement: HTMLElement }) { /* cleanup */ }

// 3.5 Version skew fix — peer + singleton
// packages/ui/package.json
{
  "name": "@company/ui",
  "version": "2.0.0",
  "peerDependencies": { "react": "^18.0.0" }
}
// host & remote cùng ^18 → singleton share 1 copy
// nếu remote dùng react 17, host 18 → warning, phải align hoặc Module Federation fallback load 2 copy (hooks break)

// 3.6 Khi chỉ cần cô lập 1 widget — Web Component thay MFE
// packages/legacy-widget — Custom Element
class LegacyWidget extends HTMLElement {
  connectedCallback() { this.innerHTML = '<div>Isolated CSS/JS via Shadow DOM</div>'; }
}
customElements.define('legacy-widget', LegacyWidget);
// <legacy-widget /> trong monolith — không cần MFE full
```

## 4. So sánh / Phân loại

| Tiêu chí | Monorepo Modular Monolith (Turborepo/Nx) | Micro-frontend (Module Federation/Single-SPA) |
|----------|------------------------------------------|-----------------------------------------------|
| Team | <30 dev, 1-3 team, 1 release train | >30 dev, 4+ team, deploy độc lập cần thiết |
| Deploy | Chung, 1 lần | Độc lập, mỗi MFE 5 lần/ngày |
| Bundle | 1 `react`, không duplicate | Dễ duplicate `react`, `lodash` nếu không singleton |
| Share state | Dễ (`Zustand`, props) | Khó — phải bus/shared store, không import trực tiếp |
| Debug | Dễ, 1 repo | Khó, 3 repo, phải mở nhiều remote |
| CI | 5 phút với cache + affected | Mỗi MFE CI riêng, host phải integration test |
| Khi dùng | Mặc định cho 90% công ty | Khi tổ chức > kỹ thuật (checklist 3/5) |

| Pattern MFE | Tích hợp | Khi dùng | Hại |
|-------------|----------|----------|-----|
| Build-time monorepo | Build chung | Mặc định | Không deploy độc lập |
| Module Federation | Runtime, host import remote | Team cần deploy độc lập, share `react` | Version skew, runtime fail nếu remote down |
| Single-SPA/Qiankun | Runtime, mount/unmount per route | Nhiều SPA khác nhau, legacy | Importmap/systemjs phức tạp |
| Server Composition (ESI/Tailor) | Server ghép HTML | SEO, performance edge | Infra phức tạp |
| Iframe/Web Component | Cô lập | Legacy, CSS/JS cô lập 1 widget | UX kém, postMessage hạn chế |

| Tiêu chí MFE quyết định (checklist 3/5) | Dấu hiệu |
|------------------------------------------|----------|
| Team >30 dev, 4+ team | Mỗi team 10-15 dev, release họp 5 team |
| Domain rõ, ít share state | `checkout` ít liên quan `search`, 70% share thì không nên MFE |
| Công nghệ khác nhau bắt buộc | 1 phần AngularJS legacy, 1 phần React |
| Deploy độc lập là yêu cầu KD | Hotfix checkout không được block bởi search |
| Modular monolith đã đau | CI 30 phút, merge conflict liên tục, 1 team sập app team khác |

| Monorepo tool | Đặc điểm |
|---------------|----------|
| Turborepo | Cache pipeline, đơn giản, hợp Next.js/Vercel |
| Nx | Affected graph, CODEOWNERS, plugin, CI per feature |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không MFE khi team <20 dev, 1 product, 1 backlog**: MFE trả giá **complexity cao** (duplicate, version skew, UX giật khi chuyển remote, SEO phức tạp, debug 3 repo) mà không hưởng **autonomy**. 90% case **monorepo + CODEOWNERS + Nx affected** đủ.
- **Không MFE khi các MFE share 70% component/state** (cart cần product, product cần cart): ranh giới mờ → **distributed monolith**, còn tệ hơn monolith.
- **Không all-in MFE nếu chỉ cần code splitting**: `React.lazy` + `import()` đủ, không cần runtime federation.
- **Không chọn MFE chỉ vì "cool"**: phải trả lời 3/5 checklist trên. Nếu chưa đau CI <5 phút, deploy 1 lần/ngày đủ → giữ monolith.
- **Monorepo cũng có giới hạn**: khi CI 30 phút dù cache, team >30 dev phải họp release, muốn thử Vue cho 1 module không rewrite hết → mới cân MFE.
- **Alternative trước MFE**: (1) Modular monolith + Turborepo/Nx, (2) CODEOWNERS + CI per feature, (3) Vertical slice + BFF (mỗi feature có BFF riêng), (4) Module Federation chỉ cho 1-2 chỗ cô lập (admin/legacy), còn lại monolith.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Duplicate `react` 2 copy → hooks fail `Invalid hook call`**
  - Triệu chứng: `Error: Invalid hook call. Hooks can only be called inside of the body of a function component`.
  - Fix: `shared: { react: { singleton: true, requiredVersion: '^18.0.0' } }` cho cả host và remote, `peerDependencies`, align version major.
  - Đo: `npm ls react`, `bundle-analyzer` check 2 `react`, Network `remoteEntry.js` → `react` chunk, console warning `unsatisfied version`.

- **Lỗi 2: Version skew design system — host `Button@2.0` vs remote `Button@1.0`**
  - Triệu chứng: UI lệch, prop `variant` mới không có ở remote.
  - Fix: `@company/ui` versioned, `peerDependencies`, **codemod** khi breaking, canary rollout.
  - Đo: `npm ls @company/ui`, Chromatic visual diff, bundle duplicate.

- **Lỗi 3: Remote down → host trắng**
  - Triệu chứng: `Failed to fetch remoteEntry.js`, whole app crash.
  - Fix: `ErrorBoundary` + `Suspense` quanh remote, fallback `<SectionError retry>`, health check, retry/backoff.
  - Đo: Network `remoteEntry.js` 500, `React.lazy` error boundary, Sentry.

- **Lỗi 4: Share state khó — cart ở host, product ở remote không sync**
  - Triệu chứng: thêm cart ở remote nhưng badge ở host không đổi.
  - Fix: **shared bus** (`mitt`) hoặc **shared store** (`zustand` ở `packages/shared-bus`), không import trực tiếp. Hoặc **BFF + Query** share qua API.
  - Đo: bus event log, `madge --circular` check import chéo (cấm).

- **Lỗi 5: Monorepo CI 30 phút vì không cache/affected**
  - Triệu chứng: mỗi PR build hết 500 components dù chỉ đổi 1 feature.
  - Fix: **Turborepo cache** + **Nx affected** (`nx affected --target=build --base=main`), chỉ test/build feature đổi.
  - Đo: CI duration, `turbo run build --dry` xem hash cache hit, `npx nx affected:graph`.

- **Lỗi 6: MFE overkill cho startup 10 dev → velocity chậm 50%**
  - Triệu chứng: 3 repo, 3 CI, 3 deploy, onboard 1 tuần.
  - Fix: quay về **modular monolith**, `React.lazy` code splitting là đủ.
  - Đo: metric **deploy frequency**, **MTTR**, **bundle size** (host 100kb + mỗi MFE 150kb vs monolith 300kb).

- **Công cụ**:
  - `npx madge --circular --extensions ts,tsx src/` — cấm MFE import chéo.
  - `npm ls react` / `npm ls @company/ui` — duplicate.
  - `webpack-bundle-analyzer` — bundle duplicate `react`/`lodash`.
  - `turbo run build --dry`, `npx nx affected:graph`, `npx nx show projects`.
  - `CODEOWNERS` + GitHub `CODEOWNERS` review.
  - Sentry — remote load fail, version mismatch warning.

## 7. Câu hỏi tự kiểm tra

1. Micro-frontend là gì và khác monorepo modular monolith thế nào? Kể 5 pattern MFE và vì sao bản chất MFE là organizational scaling chứ không phải performance?
2. Khi nào nên và không nên dùng MFE — checklist 3/5 là gì và alternative nào thay thế 90% case? Vì sao team <20 dev không nên MFE?
3. Module Federation hoạt động thế nào, `shared: { react: singleton }` để làm gì và version skew gây lỗi gì? Làm sao tránh duplicate `react` và share state giữa host/remote?

<details>
<summary>Đáp án 30s</summary>

1. **MFE** là chia FE thành nhiều app **deploy độc lập**, team tự chủ backlog/release, có thể khác stack, ghép lại 1 trải nghiệm — giải quyết **con người** (team lớn, deploy không chờ nhau), không phải performance (thậm chí bundle phình, UX giật). **Monorepo modular monolith** (Turborepo/Nx) là 1 repo nhiều `apps/packages`, **build chung, deploy chung**, 1 version `react`, share code dễ, không runtime overhead. **5 pattern**: (1) Build-time monorepo, (2) **Module Federation** (host `import('checkout/Checkout')`), (3) **Single-SPA/Qiankun** (mount/unmount per route), (4) **Server Composition** (ESI/Tailor ghép HTML), (5) **Iframe/Web Component** (cô lập). MFE đổi **autonomy lấy complexity**.
2. **Nên** khi trả lời **có 3/5**: (1) Team **>30 dev, 4+ team**, (2) **Domain rõ** ít share (>70% share thì không), (3) **Công nghệ khác nhau bắt buộc** (AngularJS + React), (4) **Deploy độc lập là yêu cầu KD** (hotfix không block), (5) **Monorepo đã đau** (CI 30p, merge conflict, 1 team sập app team khác). **Không nên** khi **<20 dev, 1 product, share 70% state**, chỉ cần **code splitting** (`React.lazy`). **Alternative 90%**: (1) **Monorepo + Turborepo/Nx + CODEOWNERS + affected CI**, (2) **BFF vertical slice**, (3) **Module Federation chỉ 1-2 chỗ cô lập** (admin/legacy), còn lại monolith. Team nhỏ MFE trả **complexity mà không hưởng autonomy**.
3. **Module Federation**: `remote` expose `./Checkout` → `host` config `remotes: { checkout: 'checkout@.../remoteEntry.js' }` → `React.lazy(() => import('checkout/Checkout'))`, `shared: { react: { singleton: true } }` để **share 1 copy react** (nếu 2 copy → `Invalid hook call`). **Version skew**: host `react 18` vs remote `react 17` hoặc `design system` lệch major → duplicate hoặc warning `unsatisfied version`. Tránh: **`peerDependencies: { react: '^18' }` + `requiredVersion` + align major**, **`@company/ui` versioned + codemod**, bus contract versioned. **Share state**: không import trực tiếp MFE khác, dùng **`mitt` bus** ở `packages/shared-bus` (`emit('cart:updated')`) hoặc API/BFF, bọc remote trong `ErrorBoundary` khi remote down.

</details>

---
*Tham khảo chi tiết: `docs/04-frontend-architecture.md` — Câu 83, 84, 85. Spec: [Module Federation](https://webpack.js.org/concepts/module-federation/), [Turborepo](https://turbo.build/repo), [Nx](https://nx.dev/), [Single-SPA](https://single-spa.js.org/).*

