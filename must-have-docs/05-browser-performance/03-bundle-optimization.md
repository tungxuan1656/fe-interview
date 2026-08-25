# Bundle Optimization — Tree-shaking, Code Splitting, sideEffects, Analyzer, Dynamic Import

> Tags: #bundle #tree-shaking #code-splitting #sideEffects #webpack #rollup #dynamic-import #prefetch | Nguồn: `docs/05-performance.md` câu 93-94 | Mức: P0

## 1. Định nghĩa chính xác

**Bundle size** là tổng bytes JavaScript gửi tới browser cho lần tải đầu. Ảnh hưởng trực tiếp LCP (tải + parse + execute block CRP) và INP (main thread bận). **Tree-shaking** là dead-code elimination dựa trên **ESM static analysis** (`import`/`export` tĩnh): bundler (Webpack/Rollup/Vite/esbuild) loại module/export không được import. **Code Splitting** là chia bundle thành nhiều chunk (`vendor`, `route`, `component`) và tải **khi cần** qua `import()` dynamic; kết hợp `React.lazy` + `Suspense` hoặc `next/dynamic`.

**`sideEffects: false`** trong `package.json` là khai báo với bundler rằng mọi file trong package đều **pure** (không có side effect khi import), cho phép loại bỏ file không dùng dù có `import` side-effect. Nếu có file CSS/polyfill có side effect, phải khai báo `sideEffects: ["*.css"]`.

**`import cost`** và **bundle analyzer** là công cụ đo: `webpack-bundle-analyzer`, `rollup-plugin-visualizer`, `source-map-explorer`, `vite-bundle-visualizer`, `import-cost` extension.

## 2. Cơ chế hoạt động

### 2.1 Tree-shaking

- Yêu cầu: **ESM** (`import`/`export`), không `require` dynamic, không side effect.
- Bundler build graph từ entry: mark `usedExports` (export nào được import), sau đó `ModuleConcatenation` + `Terser`/`esbuild` drop unused code.
- **Barrel export phá tree-shaking**:
  ```js
  // components/index.ts — barrel
  export * from './Button'; export * from './Modal'; export * from './Table';
  // app.ts
  import { Button } from '@/components'; // bundler phải include cả Modal/Table nếu không có sideEffects:false + ESM thuần
  ```
  Vì `export *` tạo re-export, bundler khó chứng minh `Modal` không có side effect (như `import './Modal.css'`).
- **Lodash barrel**: `import _ from 'lodash'` kéo 70kb (CommonJS, không tree-shakable). `import { debounce } from 'lodash-es'` + ESM vẫn có thể kéo nhiều nếu lodash-es không `sideEffects:false` triệt để; tốt nhất **direct import** `import debounce from 'lodash-es/debounce'` hoặc thay bằng `es-toolkit` (ESM thuần, nhỏ).

### 2.2 `sideEffects`

- `false`: tất cả file pure → `import { Button } from 'pkg'` mà chỉ dùng `Button` thì bundler drop file khác dù chúng có `import`.
- `["*.css", "./src/polyfill.js"]`: chỉ những file khớp pattern được giữ, còn lại pure.
- Nếu set `false` mà thực tế file có side effect (ví dụ `import './global.css'` trong `utils.ts`) → CSS bị drop, UI vỡ.

### 2.3 Code Splitting

- **Static `import`**: luôn bundle vào chunk hiện tại.
- **Dynamic `import()`**: tạo **async chunk** riêng, `import('./Heavy')` trả `Promise<Module>`, bundler tạo `Heavy.[hash].js`. Tải khi `import()` được gọi.
- **Route-based**: mỗi route là một chunk — impact lớn nhất (ví dụ `/dashboard` không tải khi ở `/`).
- **Component-based**: `React.lazy(() => import('./HeavyChart'))` + `Suspense`.
- **Vendor split**: `manualChunks: { vendor: ['react', 'react-dom'] }` hoặc `splitChunks.cacheGroups`.

### 2.4 Dynamic import + Prefetch/Preload

- `import('./HeavyChart')` khi hover/click → **prefetch** low priority cho lần sau.
- Webpack magic comment: `import(/* webpackPrefetch: true */ './Dashboard')` tạo `<link rel="prefetch">`.
- Next.js `<Link prefetch>` tự prefetch khi link vào viewport; `next/dynamic` với `ssr: false` cho component chỉ client.
- Quá nhiều chunk nhỏ → waterfall (10 chunks × 20kb = 10 RTT), quá ít chunk lớn → không giảm initial. Cân bằng: vendor chunk + route chunk + prefetch next route.

### 2.5 Compression & Polyfill

- Brotli/Gzip ở CDN giảm 70%. `browserslist` + `useBuiltIns: 'usage'` + `core-js` chỉ polyfill feature thiếu, thay vì bundle toàn bộ `core-js`.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Lodash — barrel vs direct
// ❌ 70kb — kéo cả lodash (CJS)
import _ from 'lodash';
_.debounce(() => {}, 300);

// ❌ Vẫn có thể kéo nhiều nếu barrel
import { debounce } from 'lodash-es';

// ✅ Chỉ debounce — ~2kb
import debounce from 'lodash-es/debounce';
// ✅ Thay bằng es-toolkit (ESM thuần, tree-shakable)
import { debounce } from 'es-toolkit';

// 3.2 Barrel giết tree-shaking
// components/index.ts
export * from './Button';
export * from './Modal'; // dù app chỉ dùng Button, barrel khiến bundler include cả Modal nếu không sideEffects:false

// ❌
import { Button } from '@/components'; // kéo cả Modal
// ✅ Direct
import { Button } from '@/components/Button';
// ✅ Hoặc giữ barrel nhưng đảm bảo package.json sideEffects:false + ESM + bundler optimizePackageImports

// 3.3 sideEffects
// package.json
{
  "sideEffects": false,                 // tất cả file pure
  // hoặc
  "sideEffects": ["*.css", "*.scss"]   // chỉ css có side effect
}
// Nếu có file polyfill có side effect mà để false → bị drop!
// src/polyfill.js: import './global.css' → phải liệt kê
```

```tsx
// 3.4 Route-based splitting
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}
```

```tsx
// 3.5 Component-based + prefetch khi hover
import { lazy, Suspense, useState } from 'react';
const HeavyChart = lazy(() => import('./HeavyChart'));

function Page() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button
        onMouseEnter={() => import('./HeavyChart')} // prefetch low priority
        onClick={() => setShow(true)}
      >
        Show chart
      </button>
      {show && (
        <Suspense fallback={<Skeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </>
  );
}

// 3.6 Next.js dynamic — control SSR
import dynamic from 'next/dynamic';
const Map = dynamic(() => import('../components/Map'), {
  ssr: false,
  loading: () => <Skeleton />,
});

// Webpack magic comment prefetch
const DashboardPrefetch = () => import(/* webpackChunkName: "dashboard", webpackPrefetch: true */ './Dashboard');
```

```js
// 3.7 Analyzer
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';
export default {
  plugins: [visualizer({ filename: 'stats.html', gzipSize: true, brotliSize: true })],
  build: { rollupOptions: { output: { manualChunks: { vendor: ['react', 'react-dom'] } } } },
};
// next.config.js
// experimental: { optimizePackageImports: ['lodash-es', 'date-fns', 'lucide-react'] }

// Đo nhanh
// npx vite-bundle-visualizer
// npx source-map-explorer dist/*.js
// npx webpack-bundle-analyzer dist/stats.json
```

```js
// 3.8 Tree-shaking ESM vs CJS
// utils.ts — ESM, tree-shakable
export const a = () => 'a';
export const b = () => 'b';
export const c = () => { console.log('side effect'); }; // nếu không dùng, bị drop khi sideEffects:false

// main.ts
import { a } from './utils'; // chỉ a được bundle, b/c drop

// ❌ CJS — không tree-shakable
// utils.cjs: module.exports = { a, b };
// main.cjs: const { a } = require('./utils'); // bundler không biết b có dùng không → giữ hết
```

## 4. So sánh / Phân loại

| Kỹ thuật | Tác dụng | Yêu cầu | Khi dùng |
|----------|----------|---------|----------|
| **Tree-shaking** | Loại dead code | ESM, `sideEffects:false`, không dynamic require | Luôn bật |
| **Route-based splitting** | Giảm JS ban đầu 60-80% | `import()` dynamic | Mọi app có >2 route |
| **Component-based lazy** | Giảm JS cho modal/chart/editor nặng | `React.lazy` + `Suspense` | Component >20kb, không cần ngay |
| **Vendor split** | Cache lâu, tránh re-download vendor | `splitChunks`/`manualChunks` | Luôn |
| **Prefetch** | Tải trước low priority cho next nav | `webpackPrefetch` / hover `import()` | Route chắc chắn user sẽ vào |

| Import | Bundle | Tree-shakable? |
|--------|--------|----------------|
| `import _ from 'lodash'` | 70kb | ❌ CJS |
| `import { debounce } from 'lodash'` | 70kb | ❌ barrel CJS |
| `import { debounce } from 'lodash-es'` | ~10-20kb (phụ thuộc bundler) | ⚠️ ESM nhưng barrel |
| `import debounce from 'lodash-es/debounce'` | ~2-5kb | ✅ Direct |
| `import { debounce } from 'es-toolkit'` | ~1-2kb | ✅ ESM thuần |

| `sideEffects` | Hành vi | Rủi ro |
|---------------|---------|--------|
| `false` | Drop mọi file không dùng | CSS/polyfill side effect bị drop |
| `["*.css"]` | Giữ css, drop JS không dùng | An toàn cho project có css import |
| Không khai báo / `true` | Giữ tất cả | Mất tree-shaking cho barrel |

| Analyzer | Dùng khi |
|----------|----------|
| `webpack-bundle-analyzer` | Webpack/CRA |
| `rollup-plugin-visualizer` / `vite-bundle-visualizer` | Vite/Rollup |
| `source-map-explorer` | Bất kỳ, dựa trên sourcemap |
| `import-cost` (VS Code) | Xem cost inline khi import |
| `bundlesize` / `size-limit` | Assert trong CI (`maxSize: "200 kB"`) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không micro-optimize trước khi đo**: thay `lodash` → `es-toolkit` tốn công. Luôn chạy analyzer trước, chỉ thay top 3 lib to nhất (`moment` 200kb → `date-fns`/`dayjs`, `antd` 300kb, `chart.js` 100kb).
- **Không split quá nhỏ**: 10 chunk × 20kb trên HTTP/1.1 gây queue, trên HTTP/2 vẫn tốn HPACK + decompress. Gom vendor thành 1 chunk, route thành 1 chunk/route. Dùng `webpackChunkName` để gom.
- **Không `React.lazy` cho component nhỏ, always-visible**: `lazy` thêm `Suspense` + waterfall (phải tải chunk rồi mới render). Với Button/Input <5kb, import tĩnh tốt hơn.
- **Không `sideEffects:false` nếu có CSS side effect không liệt kê**: `import './global.css'` sẽ bị drop. Luôn kiểm tra `["*.css", "*.scss"]`.
- **Không prefetch bừa bãi**: `<link rel="prefetch">` cho 10 route tốn data user (mobile), tranh bandwidth với LCP. Chỉ prefetch khi **chắc** user sẽ navigate (hover, viewport, next route trong funnel).
- **Không dynamic `ssr: false` cho content cần SEO**: `next/dynamic` với `ssr:false` không render server, bot không thấy. Chỉ dùng cho `Map`, `Chart`, `Editor` không cần SEO.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Barrel kéo cả thư viện dù chỉ dùng 1 component**
  - Triệu chứng: `source-map-explorer` thấy `Modal`/`Table` trong bundle dù chỉ `import { Button }`.
  - Nguyên nhân: `export * from './Modal'` trong `components/index.ts`, bundler không dám drop vì không chắc `Modal` không có side effect.
  - Fix: `direct import` (`@/components/Button`) hoặc `package.json { "sideEffects": false }` + ESM + `optimizePackageImports`.
  - Đo: `npx source-map-explorer dist/*.js`, `rollup-plugin-visualizer` treemap.

- **Lỗi 2: `sideEffects:false` làm mất CSS**
  - Triệu chứng: build xong UI mất style, `global.css` không apply.
  - Nguyên nhân: `import './global.css'` trong `utils.ts` bị coi là side effect không cần thiết và bị drop.
  - Fix: đổi thành `"sideEffects": ["*.css", "*.scss"]` hoặc move CSS import ra entry.
  - Đo: Coverage tab → CSS không có; diff bundle trước/sau.

- **Lỗi 3: `import _ from 'lodash'` phình bundle 70kb**
  - Triệu chứng: analyzer thấy `lodash` 70kb (gz ~24kb).
  - Fix: `import debounce from 'lodash-es/debounce'` hoặc `es-toolkit`, `next.config.js → experimental.optimizePackageImports`.
  - Đo: `import-cost` inline, `bundlesize` CI fail khi vượt 200kb.

- **Lỗi 4: Waterfall do split quá mức**
  - Triệu chứng: Network waterfall 20 request nối tiếp, TTFB cao, LCP tăng.
  - Fix: gom vendor, hạn chế chunk nhỏ <20kb; dùng `prefetch` thay vì tách quá nhỏ.
  - Đo: DevTools Network → Waterfall, Protocol `h2` multiplex; Lighthouse "Avoid enormous network payloads".

- **Lỗi 5: `React.lazy` chỉ support default export**
  - Triệu chứng: `lazy(() => import('./Foo'))` lỗi `undefined` vì `Foo` là named export.
  - Fix: `lazy(() => import('./Foo').then(m => ({ default: m.Foo })))` hoặc thêm `export default`.
  - Đo: console `Element type is invalid`.

- **Công cụ**:
  - **Analyzer**: `rollup-plugin-visualizer { gzipSize: true }`, `webpack-bundle-analyzer`, `source-map-explorer`.
  - **CI budget**: `bundlesize`, `size-limit`, `lhci` assert `resource-summary: javascript < 200kB`.
  - **Coverage**: DevTools Coverage → Unused Bytes (phát hiện dead code).
  - **Import Cost**: VS Code extension hiện size inline.

## 7. Câu hỏi tự kiểm tra

1. Vì sao `import { Button } from '@/components'` (barrel `export *`) giết tree-shaking còn `import { Button } from '@/components/Button'` thì không, và `sideEffects:false` khắc phục thế nào?
2. Phân biệt `import debounce from 'lodash-es/debounce'` vs `import { debounce } from 'lodash-es'` vs `import _ from 'lodash'` về bundle size và tree-shaking; khi nào nên thay bằng `es-toolkit`?
3. `React.lazy` + `import()` tạo async chunk như thế nào, `webpackPrefetch: true` và hover `import('./Heavy')` khác gì, và vì sao split quá nhiều chunk nhỏ lại gây waterfall?

<details>
<summary>Đáp án 30s</summary>

1. Barrel `export * from './Modal'` re-export tất cả, bundler không chắc `Modal` có side effect (như `import './Modal.css'`) nên giữ hết để an toàn. Direct import `from '@/components/Button'` chỉ graph 1 file, không kéo Modal. `sideEffects:false` khai báo mọi file pure (không side effect) nên bundler dám drop file không dùng dù có re-export; nếu có CSS side effect phải liệt kê `["*.css"]` nếu không CSS bị drop.

2. `import _ from 'lodash'` = CJS 70kb, không tree-shakable. `import { debounce } from 'lodash-es'` = ESM nhưng vẫn qua barrel `lodash-es` → bundler có thể include nhiều hơn 1 module. `import debounce from 'lodash-es/debounce'` = direct file, chỉ 1 module ~2-5kb. `es-toolkit` ESM thuần nhỏ hơn, tree-shake tốt hơn — nên thay khi lodash là top contributor trong analyzer.

3. `import('./Heavy')` static analysis → bundler tách thành `Heavy.[hash].js` async chunk, tải khi `import()` execute. `lazy(() => import('./Heavy'))` wrap promise cho React Suspense. `/* webpackPrefetch: true */` sinh `<link rel="prefetch">` low priority cho next navigation; hover `import()` cũng prefetch nhưng chỉ khi user hover. Split quá nhiều chunk nhỏ → nhiều RTT/handshake, decompress, HPACK overhead; HTTP/1 queue 6 connection, HTTP/2 vẫn tốn. Gom vendor + route, chỉ prefetch next route chắc chắn.

</details>

---
*Tham khảo chi tiết: `docs/05-performance.md` — Câu 93, 94. Spec: [Webpack Tree Shaking](https://webpack.js.org/guides/tree-shaking/), [Rollup Code Splitting](https://rollupjs.org/guide/en/#code-splitting), [Next.js dynamic](https://nextjs.org/docs/app/api-reference/components/dynamic).*
