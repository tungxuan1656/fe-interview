# 05. Performance - 19 Câu Hỏi Senior

> 19 câu hỏi hiệu năng Frontend (Câu 86-104) - từ đo lường, Core Web Vitals, tối ưu bundle, cache đến virtualization và memory leak. Senior không tối ưu bằng cảm tính mà bằng số liệu, và hiểu trade-off của mỗi kỹ thuật.

## Mục lục

- [Câu 86: Nguyên nhân khiến Frontend chậm - các nhóm bottleneck](#câu-86-nguyên-nhân-khiến-frontend-chậm---các-nhóm-bottleneck)
- [Câu 87: Cách đo performance - Lighthouse, Web Vitals, Profiler](#câu-87-cách-đo-performance---lighthouse-web-vitals-profiler)
- [Câu 88: Core Web Vitals tổng quan - LCP, INP, CLS](#câu-88-core-web-vitals-tổng-quan---lcp-inp-cls)
- [Câu 89: LCP (Largest Contentful Paint) chi tiết](#câu-89-lcp-largest-contentful-paint-chi-tiết)
- [Câu 90: INP (Interaction to Next Paint) - thay thế FID](#câu-90-inp-interaction-to-next-paint---thay-thế-fid)
- [Câu 91: CLS (Cumulative Layout Shift) chi tiết](#câu-91-cls-cumulative-layout-shift-chi-tiết)
- [Câu 92: Tối ưu LCP thực chiến](#câu-92-tối-ưu-lcp-thực-chiến)
- [Câu 93: Giảm bundle size - chiến lược toàn diện](#câu-93-giảm-bundle-size---chiến-lược-toàn-diện)
- [Câu 94: Code Splitting, lazy, dynamic import và tree shaking](#câu-94-code-splitting-lazy-dynamic-import-và-tree-shaking)
- [Câu 95: Image Optimization](#câu-95-image-optimization)
- [Câu 96: Font Optimization](#câu-96-font-optimization)
- [Câu 97: Prefetch, Preload, Preconnect và Modulepreload](#câu-97-prefetch-preload-preconnect-và-modulepreload)
- [Câu 98: HTTP Cache - Cache-Control, ETag, Last-Modified](#câu-98-http-cache---cache-control-etag-last-modified)
- [Câu 99: Browser Cache vs CDN Cache vs Service Worker Cache](#câu-99-browser-cache-vs-cdn-cache-vs-service-worker-cache)
- [Câu 100: Render 10k rows - virtualization](#câu-100-render-10k-rows---virtualization)
- [Câu 101: Virtualization hoạt động thế nào? So sánh thư viện](#câu-101-virtualization-hoạt-động-thế-nào-so-sánh-thư-viện)
- [Câu 102: Scroll lag debug - long task, layout thrashing, compositor](#câu-102-scroll-lag-debug---long-task-layout-thrashing-compositor)
- [Câu 103: Memory Leak trong React - nguyên nhân và cách phát hiện](#câu-103-memory-leak-trong-react---nguyên-nhân-và-cách-phát-hiện)
- [Câu 104: Performance Budget và monitoring production](#câu-104-performance-budget-và-monitoring-production)


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 86: Nguyên nhân khiến Frontend chậm - các nhóm bottleneck

**Trả lời Senior:**
Frontend chậm không phải 1 nguyên nhân mà là 4 nhóm bottleneck chồng lên nhau. Senior debug bằng cách phân loại:

1.  **Network**: bundle quá lớn (1.5MB JS), không split, waterfall 30 request, ảnh 4K không nén, không cache, HTTP/1.1 head-of-line blocking. Dấu hiệu: TTFB cao, waterfall dài trong Network tab.
2.  **Compute (Main Thread)**: JS nặng, long task > 50ms, React re-render thừa, không memo, tính toán trong render, `JSON.parse` 2MB. Dấu hiệu: INP cao, Performance tab toàn yellow long task.
3.  **Render (Browser)**: layout thrashing (đọc `offsetHeight` trong loop), quá nhiều DOM node (10k row không virtualize), CSS selector phức tạp, reflow/repaint liên tục. Dấu hiệu: FPS thấp, Rendering tab báo layout shift.
4.  **Resource**: ảnh/font không optimize, third-party script (analytics, chat widget) block main thread, memory leak làm GC lag.

```typescript
// Ví dụ bottleneck compute: tính toán trong render
function ExpensiveList({ items }: { items: Item[] }) {
  // ❌ Chạy mỗi render, dù items không đổi
  const sorted = items.slice().sort((a, b) => a.price - b.price);
  // ✅ Memo
  const sortedMemo = useMemo(() => items.slice().sort((a, b) => a.price - b.price), [items]);
}

// Ví dụ layout thrashing
function thrashing() {
  // ❌ Đọc - ghi - đọc - ghi trong loop -> force reflow mỗi lần
  for (let i = 0; i < els.length; i++) {
    const h = els[i].offsetHeight; // đọc -> force layout
    els[i].style.height = h + 10 + 'px'; // ghi
  }
  // ✅ Batch đọc trước, ghi sau
  const heights = els.map(el => el.offsetHeight);
  els.forEach((el, i) => (el.style.height = heights[i] + 10 + 'px'));
}

// Check bundle
// npx vite-bundle-visualizer
// npx source-map-explorer dist/*.js
```

**Trade-off:** Tối ưu network (split, compress) dễ, lợi nhiều, nên làm trước. Tối ưu compute/render cần đo profiler, không làm mù.

**Câu hỏi đào sâu:** Làm sao phân biệt chậm do network vs chậm do main thread bằng DevTools? Vì sao third-party script là killer của INP?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 87: Cách đo performance - Lighthouse, Web Vitals, Profiler

**Trả lời Senior:**
Không đo thì không tối ưu. Senior có 4 tầng đo:

1.  **Lab (Lighthouse)**: chạy trong DevTools, cho điểm 0-100, chỉ ra LCP, CLS, TBT, bundle, ảnh. Dùng để **regression trong CI** (`lighthouse-ci`), nhưng là môi trường giả lập, không phải user thật.
2.  **Field (Web Vitals + CrUX + RUM)**: đo trên user thật qua `web-vitals` lib, gửi về analytics. Cho p75 (75th percentile) - Google xếp hạng SEO bằng field, không phải lab.
3.  **React Profiler**: đo **commit time, render duration, vì sao re-render** (props/state/context). Dùng để tìm wasted render.
4.  **Chrome Performance Tab**: timeline scripting/rendering/painting, long task, layout, FPS.

```typescript
// 1. Web Vitals RUM
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(metric => {
  console.log('LCP', metric.value, metric.entries);
  // gửi về analytics
  fetch('/analytics', { method: 'POST', body: JSON.stringify({ name: metric.name, value: metric.value, id: metric.id }) });
});
onINP(metric => console.log('INP', metric.value));
onCLS(metric => console.log('CLS', metric.value));

// 2. React Profiler
import { Profiler } from 'react';
function onRender(id: string, phase: 'mount' | 'update', actualDuration: number, baseDuration: number) {
  if (actualDuration > 16) console.warn(`${id} slow: ${actualDuration}ms`);
}
<Profiler id="ProductList" onRender={onRender}><ProductList /></Profiler>

// 3. Performance mark
performance.mark('fetch-start');
await fetch('/api/products');
performance.mark('fetch-end');
performance.measure('fetch', 'fetch-start', 'fetch-end');
console.log(performance.getEntriesByName('fetch'));

// 4. Lighthouse CI
// .github/workflows/lighthouse.yml
// - run: npx lhci autorun --upload.target=temporary-public-storage
```

**Trade-off:** Lab nhanh nhưng không phản ánh user 3G, máy yếu. Field chuẩn nhưng cần thu thập đủ data. Luôn kết hợp cả hai, và đặt **performance budget** trong CI.

**Câu hỏi đào sâu:** Lab vs Field khác gì và vì sao Google dùng field cho SEO? p75 là gì và vì sao không dùng average?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 88: Core Web Vitals tổng quan - LCP, INP, CLS

**Trả lời Senior:**
Core Web Vitals là 3 chỉ số Google dùng để xếp hạng SEO và UX, đo trên **field (p75, 28 ngày)**:

- **LCP (Largest Contentful Paint)**: thời gian từ navigation tới khi **phần tử lớn nhất trong viewport** render xong (ảnh hero, heading). Đo **loading**. Tốt < 2.5s, cần cải thiện < 4s, tệ > 4s.
- **INP (Interaction to Next Paint)**: từ khi user tương tác (click, key) tới khi **frame tiếp theo paint**, đo **độ trễ tương tác tồi nhất** (p98 của tương tác). Thay FID từ 3/2024. Tốt < 200ms, tệ > 500ms. Đo **interactivity**.
- **CLS (Cumulative Layout Shift)**: tổng điểm **dịch layout bất ngờ** trong vòng đời trang. Đo **visual stability**. Tốt < 0.1, tệ > 0.25.

Ngoài ra còn **TTFB, FCP, TBT (Total Blocking Time)** nhưng không phải Core.

```typescript
// Ngưỡng
// LCP: <2.5s good, 2.5-4s needs improvement, >4s poor
// INP: <200ms good, 200-500ms needs improvement, >500ms poor
// CLS: <0.1 good, 0.1-0.25 needs improvement, >0.25 poor

// LCP candidate: img, video poster, background-image, block text lớn
// INP: đo tất cả click/keydown, lấy worst (p98)
// CLS: sum(impactFraction * distanceFraction) cho mỗi shift không do user

// Check CrUX
// https://pagespeed.web.dev/analysis?url=https://example.com
// hoặc API
fetch('https://chromeuxreport.googleapis.com/v1/records:queryRecord?key=API_KEY', {
  method: 'POST',
  body: JSON.stringify({ origin: 'https://example.com' }),
});

// Web Vitals attribution - biết nguyên nhân
import { onLCP } from 'web-vitals/attribution';
onLCP(metric => {
  console.log(metric.attribution); // element, url, timeToFirstByte
});
```

**Trade-off:** Tối ưu LCP thường làm tăng CLS nếu không reserve space, tối ưu INP (giảm JS) có thể làm tăng LCP nếu lazy quá mức. Phải cân bằng.

**Câu hỏi đào sâu:** Vì sao INP thay FID? p75 và p98 khác gì? CrUX vs RUM tự đo khác nhau thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 89: LCP (Largest Contentful Paint) chi tiết

**Trả lời Senior:**
LCP là **thời điểm paint phần tử lớn nhất** (theo kích thước trong viewport) - thường là hero image, banner, hoặc heading lớn. Browser xác định candidate liên tục tới khi user tương tác hoặc scroll. LCP chậm do 4 nguyên nhân chính: **TTFB chậm** (server chậm, không CDN), **Resource load delay** (ảnh hero load chậm, không preload), **Render delay** (JS block, CSS block, client render), **Element delay** (ảnh không có priority, lazy sai chỗ).

LCP tốt < 2.5s (p75). Ảnh hưởng SEO trực tiếp.

```html
<!-- ❌ LCP tệ: hero ảnh 2MB, không preload, lazy sai -->
<img src="/hero-4k.jpg" loading="lazy" />

<!-- ✅ LCP tốt -->
<head>
  <link rel="preload" as="image" href="/hero-800.webp" imagesrcset="/hero-800.webp 800w, /hero-1200.webp 1200w" fetchpriority="high" />
  <link rel="preconnect" href="https://cdn.example.com" />
</head>
<body>
  <!-- Hero là LCP candidate - không lazy, có size, priority high -->
  <img
    src="/hero-800.webp"
    srcset="/hero-800.webp 800w, /hero-1200.webp 1200w"
    sizes="(max-width: 768px) 100vw, 50vw"
    width="1200" height="600"
    fetchpriority="high"
    decoding="async"
    alt="Hero"
  />
</body>
```

```typescript
// Đo LCP chi tiết
import { onLCP } from 'web-vitals/attribution';
onLCP(metric => {
  console.log('LCP', metric.value);
  console.log('Element', metric.attribution.element); // tag
  console.log('URL', metric.attribution.url);
  console.log('TTFB', metric.attribution.timeToFirstByte);
  console.log('ResourceLoadDelay', metric.attribution.resourceLoadDelay);
});
```

**Trade-off:** Preload hero giúp LCP nhưng tốn bandwidth nếu user không thấy hero (ví dụ hero dưới fold). Chỉ preload **above-the-fold** LCP.

**Câu hỏi đào sâu:** Browser chọn LCP candidate thế nào? Vì sao `loading="lazy"` cho hero lại giết LCP? `fetchpriority="high"` khác `preload` thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 90: INP (Interaction to Next Paint) - thay thế FID

**Trả lời Senior:**
INP đo **độ trễ từ khi user tương tác (click, tap, keypress) tới khi browser paint frame tiếp theo**. Nó thay FID (First Input Delay) vì FID chỉ đo **delay đầu tiên** và chỉ đo **input delay** (thời gian chờ main thread rảnh), bỏ qua **processing time** và **presentation delay**. INP đo **toàn bộ** và đo **mọi tương tác**, lấy **p98 tồi nhất** - phản ánh trải nghiệm tệ nhất, không phải may mắn lần đầu.

INP cao do **main thread bận**: long task (>50ms) từ JS nặng, React render lớn, third-party, hoặc event handler nặng. Tốt < 200ms.

```typescript
// FID chỉ đo: event -> main thread rảnh
// INP đo: event -> handler chạy -> React render -> commit -> paint
// |--- input delay ---|--- processing ---|--- presentation ---|

// Ví dụ INP tệ: handler nặng block main thread
function BadList({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('');
  const onChange = e => {
    const v = e.target.value;
    setFilter(v);
    // ❌ Filter 10k item đồng bộ trong handler -> block 300ms -> INP tệ
    const filtered = items.filter(i => i.name.includes(v)); // 300ms
    setFiltered(filtered);
  };
}

// ✅ Fix: tách urgent và non-urgent, defer
function GoodList({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('');
  const deferredFilter = useDeferredValue(filter);
  const filtered = useMemo(() => items.filter(i => i.name.includes(deferredFilter)), [items, deferredFilter]);
  const [isPending, startTransition] = useTransition();
  const onChange = e => {
    const v = e.target.value;
    setFilter(v); // urgent - input update ngay, INP thấp
    // non-urgent để transition, không block paint
  };
  return <><input value={filter} onChange={onChange} />{isPending ? <Spinner /> : <List items={filtered} />}</>;
}

// Đo INP
import { onINP } from 'web-vitals/attribution';
onINP(metric => {
  console.log('INP', metric.value);
  console.log('Target', metric.attribution.interactionTarget);
  console.log('InputDelay', metric.attribution.inputDelay);
  console.log('ProcessingDuration', metric.attribution.processingDuration);
  console.log('PresentationDelay', metric.attribution.presentationDelay);
});

// Fix INP: break long task
import { scheduler } from 'scheduler';
function yieldToMain() { return new Promise(resolve => setTimeout(resolve, 0)); }
async function processLargeArray(array) {
  for (let i = 0; i < array.length; i += 100) {
    processChunk(array.slice(i, i + 100));
    await yieldToMain(); // nhường main thread
  }
}
```

**Trade-off:** `useTransition`/`useDeferredValue` giảm INP nhưng làm UI có 2 trạng thái (pending), cần thiết kế loading. Break long task làm tổng thời gian tăng nhẹ nhưng UX mượt hơn.

**Câu hỏi đào sâu:** Vì sao INP lấy p98 thay vì average? Làm sao phát hiện long task gây INP bằng Performance tab?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 91: CLS (Cumulative Layout Shift) chi tiết

**Trả lời Senior:**
CLS là **tổng điểm dịch layout bất ngờ** mà không do user tương tác. Mỗi shift tính `score = impactFraction * distanceFraction`. `impactFraction` là % viewport bị ảnh hưởng, `distanceFraction` là khoảng dịch lớn nhất / viewport. Tổng CLS < 0.1 là tốt, > 0.25 là tệ. Lưu ý cập nhật 2025: Brave/CrUX hiện đo CLS đến sự kiện pagehidden (thay vì kéo dài vô hạn), nhưng ngưỡng 0.1 vẫn giữ nguyên.

Nguyên nhân kinh điển: **ảnh/iframe không có size** (load xong đẩy content), **font swap** (FOIT → FOUT làm text reflow), **ads/banner inject** trên đầu, **dynamic content** (thêm DOM mà không reserve space), **animation không dùng transform**.

```html
<!-- ❌ CLS tệ: ảnh không size -> khi load đẩy content xuống -->
<img src="/photo.jpg" />
<div>Content bị đẩy</div>

<!-- ✅ CLS tốt: reserve space -->
<img src="/photo.jpg" width="800" height="600" style="aspect-ratio: 800/600; width: 100%; height: auto;" alt="" />
<!-- Hoặc CSS aspect-ratio -->
<div style="aspect-ratio: 16/9; background: #eee;">
  <img src="/photo.jpg" style="width: 100%; height: 100%; object-fit: cover;" />
</div>

<!-- ❌ Ads inject không reserve -->
<div id="ads"></div>
<script>document.getElementById('ads').innerHTML = '<div style="height:250px">Ad</div>';</script>

<!-- ✅ Reserve trước -->
<div id="ads" style="min-height: 250px; background: #f5f5f5;"></div>
```

```css
/* Font CLS fix */
@font-face {
  font-family: 'Inter';
  src: url('/inter.woff2') format('woff2');
  font-display: optional; /* hoặc swap + size-adjust */
  size-adjust: 100%;
  ascent-override: 90%;
}
/* Dùng Font Loading API để tránh swap */
```

```typescript
// Đo CLS
import { onCLS } from 'web-vitals/attribution';
onCLS(metric => {
  console.log('CLS', metric.value);
  metric.attribution.largestShiftTarget && console.log('Target', metric.attribution.largestShiftTarget);
  metric.attribution.largestShiftValue && console.log('Value', metric.attribution.largestShiftValue);
});
// Trong Performance tab: Experience -> Layout Shifts
```

**Trade-off:** Reserve space làm trang có khoảng trắng nếu content chưa có, nhưng tránh CLS. `font-display: optional` giảm CLS nhưng có thể không hiện font custom trên 3G chậm.

**Câu hỏi đào sâu:** `impactFraction` và `distanceFraction` tính thế nào? Vì sao `transform: translate` không gây CLS còn `top` thì có?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 92: Tối ưu LCP thực chiến

**Trả lời Senior:**
Tối ưu LCP theo thứ tự **TTFB → Load Delay → Render Delay** (theo web.dev):

1.  **TTFB**: dùng CDN, edge, cache, SSR streaming. TTFB > 800ms là chết LCP.
2.  **Resource Load Delay**: preload LCP image (`fetchpriority="high"`), preconnect CDN, không lazy hero, dùng `srcset` + `sizes` đúng, nén WebP/AVIF, `loading="eager"`.
3.  **Render Delay**: inline critical CSS, defer non-critical JS, không block render bằng `import` sync lớn, SSR thay vì CSR cho hero.
4.  **Element Render Delay**: đảm bảo LCP element không bị JS tạo muộn (ví dụ hero do `useEffect` fetch mới render).

Checklist thực chiến: hero ảnh < 100KB, preload, CDN, không JS block, có `width/height`.

```html
<!-- Tối ưu LCP hero -->
<head>
  <!-- 1. Preconnect CDN -->
  <link rel="preconnect" href="https://cdn.example.com" crossorigin />
  <!-- 2. Preload LCP image với srcset -->
  <link
    rel="preload"
    as="image"
    href="/hero-800.webp"
    imagesrcset="/hero-800.webp 800w, /hero-1200.webp 1200w"
    imagesizes="(max-width: 768px) 100vw, 50vw"
    fetchpriority="high"
  />
  <!-- 3. Inline critical CSS -->
  <style>/* hero, header, above-fold */</style>
  <!-- 4. Defer JS -->
  <script src="/app.js" defer></script>
</head>
<body>
  <!-- 5. Hero không lazy, có size, priority high -->
  <img
    src="/hero-800.webp"
    srcset="/hero-800.webp 800w, /hero-1200.webp 1200w"
    sizes="(max-width:768px)100vw,50vw"
    width="1200" height="600"
    fetchpriority="high"
    decoding="async"
    alt=""
  />
</body>
```

```typescript
// Next.js
import Image from 'next/image';
<Image
  src="/hero.jpg"
  width={1200}
  height={600}
  priority // = preload + fetchpriority high
  sizes="(max-width:768px)100vw,50vw"
  alt=""
/>

// Đo và fix
// - Lighthouse: "Largest Contentful Paint element" -> xem element nào
// - Nếu LCP là text -> do font/block
// - Nếu LCP là img -> do load delay
// - Nếu LCP là background-image -> đổi sang <img> để preload được
```

**Trade-off:** Preload quá nhiều (3+ ảnh) làm tranh bandwidth, chỉ preload 1 LCP. Inline CSS nhiều làm HTML to, chỉ inline above-fold.

**Câu hỏi đào sâu:** Làm sao xác định element nào là LCP bằng DevTools? Vì sao `background-image` tệ cho LCP hơn `<img>`?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 93: Giảm bundle size - chiến lược toàn diện

**Trả lời Senior:**
Bundle lớn giết cả LCP và INP. Chiến lược giảm theo thứ tự impact:

1.  **Đo**: `source-map-explorer`, `vite-bundle-visualizer`, `webpack-bundle-analyzer`. Biết ai to: `lodash` 70kb, `moment` 200kb, `antd` 300kb, `chart.js` 100kb.
2.  **Thay lib nặng**: `moment` → `date-fns` (tree-shakable) hoặc `dayjs` (2kb), `lodash` → `lodash-es` + import cụ thể (`import debounce from 'lodash-es/debounce'`), `axios` → `fetch` nếu không cần interceptor phức tạp.
3.  **Tree-shaking**: ESM (`import/export`), `sideEffects: false` trong package.json, không import barrel (`import { Button } from '@/components'` sẽ kéo hết).
4.  **Code splitting + Dynamic import**: route-based + component-based.
5.  **Compression**: Brotli/Gzip ở CDN, `esbuild` minify.
6.  **Polyfill có điều kiện**: `browserslist` + `useBuiltIns: 'usage'` thay vì polyfill hết.

```typescript
// ❌ Kéo cả lodash
import _ from 'lodash';
_.debounce(() => {}, 300);

// ✅ Chỉ debounce
import debounce from 'lodash-es/debounce';
// hoặc
import { debounce } from 'es-toolkit'; // nhẹ hơn

// ❌ Barrel làm mất tree-shaking
export * from './Button';
export * from './Modal';
import { Button } from '@/components'; // kéo cả Modal dù không dùng

// ✅ Direct import hoặc barrel có sideEffects false + esm
import { Button } from '@/components/Button';

// Đo
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';
export default { plugins: [visualizer({ filename: 'stats.html' })] };

// next.config.js
// experimental: { optimizePackageImports: ['lodash-es', 'date-fns'] }

// Thay moment
// Before: 200kb
import moment from 'moment';
// After: 2kb
import { format } from 'date-fns';

// Dynamic import chart
const Chart = dynamic(() => import('../components/Chart'), { ssr: false });
```

**Trade-off:** Direct import làm import dài hơn, nhưng bundle nhỏ hơn. Thay lib tốn công, nhưng lợi lâu dài. Đừng micro-optimize khi chưa đo.

**Câu hỏi đào sâu:** Vì sao barrel export giết tree-shaking? `sideEffects: false` có tác dụng gì với Webpack/Rollup?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 94: Code Splitting, lazy, dynamic import và tree shaking

**Trả lời Senior:**
Ba kỹ thuật giảm JS ban đầu:

- **Code Splitting**: chia bundle thành chunks (vendor, route, component). Bundler tự split với `import()` dynamic. Mục tiêu: JS ban đầu < 200kb.
- **lazy + dynamic import**: `React.lazy(() => import('./Heavy'))` + `Suspense` để load khi cần (route, modal, chart). Next.js có `dynamic(() => import(...), { ssr: false })`.
- **Tree Shaking**: loại code không dùng (dead code) nhờ ESM static analysis. Yêu cầu: ESM, không side effect, không dynamic `require`.

```typescript
// 1. Route-based splitting - impact lớn nhất
import { lazy, Suspense } from 'react';
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

// 2. Component-based + prefetch
const HeavyChart = lazy(() => import('./HeavyChart'));
function Page() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button
        onMouseEnter={() => import('./HeavyChart')} // prefetch khi hover
        onClick={() => setShow(true)}
      >
        Show
      </button>
      {show && (
        <Suspense fallback={<Skeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </>
  );
}

// 3. Next.js dynamic - SSR control
import dynamic from 'next/dynamic';
const Map = dynamic(() => import('../components/Map'), { ssr: false, loading: () => <Skeleton /> });

// 4. Tree shaking - ESM
// utils.ts - ESM, tree-shakable
export const a = () => {};
export const b = () => {};
// main.ts - chỉ a được bundle, b bị drop
import { a } from './utils';

// ❌ Không tree-shakable - CommonJS
module.exports = { a, b };
const { a } = require('./utils'); // bundler không biết b có dùng không

// package.json
// "sideEffects": false - báo không có side effect, an toàn drop
// "sideEffects": ["*.css"] - chỉ css có side effect
```

**Trade-off:** Split quá nhỏ gây waterfall (10 chunks x 20kb = 10 request), quá lớn không giảm TTFB. Dùng `webpackChunkName` để gom vendor, `prefetch` cho route tiếp theo.

**Câu hỏi đào sâu:** `React.lazy` chỉ support default export, làm sao handle named export? `import()` dynamic khác `import` static thế nào về bundling?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 95: Image Optimization

**Trả lời Senior:**
Ảnh thường chiếm 50-70% bytes của trang, là LCP candidate, nên optimize ảnh impact lớn nhất. Chiến lược: **Đúng format, đúng size, lazy đúng chỗ, CDN**.

1.  **Format**: AVIF (nhỏ nhất) > WebP > JPEG. Dùng `<picture>` hoặc `srcset` để browser chọn. Ảnh vector dùng SVG.
2.  **Size**: `srcset` + `sizes` để mobile không tải 2000w, `width/height` + `aspect-ratio` để tránh CLS, nén (Squoosh, sharp, Cloudinary).
3.  **Loading**: hero/LCP = `eager` + `priority`/`fetchpriority="high"` + `preload`, below-fold = `loading="lazy"` + `decoding="async"`.
4.  **CDN + Responsive**: Cloudinary, Imgix, hoặc `next/image` tự resize, `loading="lazy"` native, placeholder `blur`.

```html
<!-- Picture với AVIF/WebP fallback -->
<picture>
  <source srcset="/hero.avif" type="image/avif" />
  <source srcset="/hero.webp" type="image/webp" />
  <img src="/hero.jpg" width="1200" height="600" alt="" fetchpriority="high" decoding="async" />
</picture>

<!-- srcset + sizes -->
<img
  src="/photo-800.jpg"
  srcset="/photo-400.jpg 400w, /photo-800.jpg 800w, /photo-1200.jpg 1200w"
  sizes="(max-width: 768px) 100vw, 50vw"
  width="800"
  height="600"
  loading="lazy"
  decoding="async"
  alt=""
  style="aspect-ratio: 4/3"
/>
```

```typescript
// Next.js Image - tự optimize
import Image from 'next/image';
<Image
  src="/hero.jpg"
  width={1200}
  height={600}
  priority // LCP
  placeholder="blur"
  blurDataURL="data:image/..."
  sizes="(max-width:768px)100vw,50vw"
  alt=""
/>;

// Sharp - build time
import sharp from 'sharp';
await sharp('input.jpg').resize(800).webp({ quality: 80 }).toFile('output.webp');

// Lazy với IntersectionObserver (nếu không dùng native lazy)
function LazyImage({ src }: { src: string }) {
  const ref = useRef<HTMLImageElement>(null);
  useEffect(() => {
    const io = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) { ref.current!.src = src; io.disconnect(); }
    });
    io.observe(ref.current!);
    return () => io.disconnect();
  }, [src]);
  return <img ref={ref} width={800} height={600} alt="" />;
}
```

**Trade-off:** AVIF nhỏ nhưng encode chậm, không phải browser nào cũng support - cần fallback. `next/image` tiện nhưng lock vào Vercel/CDN.

**Câu hỏi đào sâu:** `srcset` + `sizes` browser chọn ảnh thế nào? Vì sao không nên `loading="lazy"` cho LCP image?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 96: Font Optimization

**Trả lời Senior:**
Font gây **FOIT (invisible)** hoặc **FOUT (swap)** và CLS nếu size khác. Chiến lược: **Ít font, subset, preload, display, fallback metric**.

1.  **Ít font**: chỉ 1-2 family, 2-3 weight, dùng `variable font` nếu cần nhiều weight.
2.  **Subset**: chỉ latin, bỏ glyph không dùng (`&subset=latin`), giảm 70% size.
3.  **Preload**: `<link rel="preload" as="font" href="/inter.woff2" crossorigin>` cho font LCP (heading).
4.  **font-display**: `swap` (hiện fallback ngay, swap khi xong - có FOUT nhưng không FOIT), `optional` (nếu font chậm thì thôi - không CLS), `fallback`.
5.  **Fallback metric**: `size-adjust`, `ascent-override`, `descent-override` để fallback font (Arial) có metric gần Inter, giảm CLS khi swap. Next.js `next/font` tự làm.

```html
<head>
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link rel="preload" as="font" href="/fonts/inter-latin.woff2" type="font/woff2" crossorigin />
  <style>
    @font-face {
      font-family: 'Inter';
      src: url('/fonts/inter-latin.woff2') format('woff2');
      font-display: swap; /* hoặc optional để giảm CLS */
      font-weight: 400 700; /* variable */
      unicode-range: U+0000-00FF; /* latin */
    }
    /* Fallback metric - giảm CLS */
    @font-face {
      font-family: 'Inter Fallback';
      src: local('Arial');
      size-adjust: 107%;
      ascent-override: 90%;
      descent-override: 22%;
    }
    body { font-family: 'Inter', 'Inter Fallback', sans-serif; }
  </style>
</head>
```

```typescript
// Next.js next/font - tự optimize, tự fallback, tự preload
import { Inter } from 'next/font/google';
const inter = Inter({ subsets: ['latin'], display: 'swap', weight: ['400', '600'], variable: '--font-inter' });
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <html className={inter.variable}><body>{children}</body></html>;
}

// Đo font CLS
// Performance tab -> Web Fonts
// Lighthouse -> "Ensure text remains visible during webfont load"
```

**Trade-off:** `swap` cho UX tốt (thấy text ngay) nhưng CLS nếu metric lệch. `optional` không CLS nhưng có thể không hiện font custom trên 3G. Subset giảm size nhưng nếu cần đa ngôn ngữ thì phải thêm subset.

**Câu hỏi đào sâu:** `font-display: swap` vs `optional` vs `fallback` khác gì? `size-adjust` giảm CLS thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 97: Prefetch, Preload, Preconnect và Modulepreload

**Trả lời Senior:**
Đây là 4 hint cho browser **ưu tiên tài nguyên**, nhưng dùng sai sẽ phản tác dụng (tốn bandwidth, tranh priority).

- **preload**: tải **ngay** với priority cao, cho tài nguyên cần trong trang hiện tại (LCP image, critical font, critical CSS). `<link rel="preload" as="image" href="/hero.webp" fetchpriority="high">`. Dùng cho 1-2 thứ.
- **prefetch**: tải **khi rảnh** (low priority) cho trang tiếp theo. `<link rel="prefetch" href="/dashboard.js">`. Dùng cho next route.
- **preconnect**: mở **TCP + TLS** sớm tới origin khác (`https://cdn.example.com`, `https://fonts.googleapis.com`), tiết kiệm 100-300ms. Không tải file.
- **dns-prefetch**: chỉ DNS, rẻ hơn preconnect, dùng khi không chắc có tải không.
- **modulepreload**: như preload nhưng cho ES module, có dependency graph.

```html
<head>
  <!-- Preconnect - CDN, font, API -->
  <link rel="preconnect" href="https://cdn.example.com" crossorigin />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

  <!-- Preload - LCP -->
  <link rel="preload" as="image" href="/hero-800.webp" imagesrcset="/hero-800.webp 800w" fetchpriority="high" />
  <link rel="preload" as="font" href="/inter.woff2" type="font/woff2" crossorigin />
  <link rel="preload" as="style" href="/critical.css" />

  <!-- Modulepreload - entry -->
  <link rel="modulepreload" href="/app.js" />

  <!-- Prefetch - next route -->
  <link rel="prefetch" href="/dashboard.js" />
  <link rel="prefetch" href="/api/products" as="fetch" crossorigin />
</head>
```

```typescript
// React - prefetch khi hover
function LinkWithPrefetch({ href, children }: Props) {
  const prefetch = () => {
    const link = document.createElement('link');
    link.rel = 'prefetch';
    link.href = href;
    document.head.appendChild(link);
  };
  return <a href={href} onMouseEnter={prefetch}>{children}</a>;
}

// Next.js tự prefetch <Link>
import Link from 'next/link';
<Link href="/dashboard" prefetch={true}>Dashboard</Link>; // tự prefetch khi vào viewport

// Vite - manualChunks + prefetch
// vite.config.ts: build.rollupOptions.output.manualChunks = { vendor: ['react', 'react-dom'] }
```

**Trade-off:** Preload nhiều làm tranh bandwidth với LCP, chỉ preload 1-2 critical. Prefetch tốn data user, chỉ prefetch khi chắc user sẽ navigate (hover, viewport). Preconnect 2-3 origin là đủ, nhiều hơn tốn socket.

**Câu hỏi đào sâu:** Preload vs Prefetch priority khác gì? Vì sao preconnect cần `crossorigin` cho font/CDN? Khi nào dùng `dns-prefetch` thay vì `preconnect`?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 98: HTTP Cache - Cache-Control, ETag, Last-Modified

**Trả lời Senior:**
HTTP Cache là cách rẻ nhất để nhanh: không tải lại. Hai loại: **Strong Cache** (không gửi request) và **Conditional Cache** (gửi request, server trả 304 nếu không đổi).

- **Strong**: `Cache-Control: max-age=31536000, immutable` cho asset versioned (`app.abc123.js`), `max-age=0, must-revalidate` cho HTML.
- **Conditional**: `ETag` (hash content) hoặc `Last-Modified` (timestamp). Browser gửi `If-None-Match: "abc"` hoặc `If-Modified-Since`, server trả `304 Not Modified` nếu khớp, tiết kiệm bandwidth nhưng vẫn tốn RTT.
- **Expires**: header cũ, dùng `Cache-Control` thay.
- **Vary**: `Vary: Accept-Encoding` để cache riêng gzip/br.

```http
# Asset versioned - strong cache 1 năm
GET /static/app.abc123.js
Cache-Control: public, max-age=31536000, immutable
ETag: "abc123"
# Lần sau: browser dùng cache, không request

# HTML - không cache hoặc revalidate
GET /index.html
Cache-Control: no-cache # no-cache vẫn lưu nhưng phải revalidate (304), no-store mới không lưu gì. Cho HTML nên dùng no-cache hoặc must-revalidate, không dùng no-store nếu muốn 304.
# hoặc
Cache-Control: public, max-age=0, must-revalidate
ETag: "html-v1"
# Lần sau: browser gửi If-None-Match, server 304 nếu không đổi

# API - tùy freshness
GET /api/products
Cache-Control: public, max-age=60, stale-while-revalidate=300
# 60s fresh, 300s stale vẫn dùng cache nhưng revalidate background
```

```typescript
// Express
app.use('/static', express.static('dist', {
  maxAge: '1y',
  immutable: true,
  etag: true,
}));
app.get('/api/products', (req, res) => {
  res.set('Cache-Control', 'public, max-age=60, stale-while-revalidate=300');
  res.json(products);
});

// Next.js
// next.config.js
async headers() {
  return [
    { source: '/static/:path*', headers: [{ key: 'Cache-Control', value: 'public, max-age=31536000, immutable' }] },
    { source: '/:path*', headers: [{ key: 'Cache-Control', value: 'public, max-age=0, must-revalidate' }] },
  ];
}

// Fetch với cache
fetch('/api/products', { cache: 'force-cache' }); // Next.js Data Cache
fetch('/api/products', { cache: 'no-store' });
fetch('/api/products', { next: { revalidate: 60 } }); // ISR
```

**Trade-off:** `immutable` cho asset hash là an toàn, cho HTML thì toang (user kẹt HTML cũ). `ETag` chính xác nhưng tốn CPU hash, `Last-Modified` rẻ nhưng độ chính xác 1s.

**Câu hỏi đào sâu:** `no-cache` vs `no-store` khác gì? `stale-while-revalidate` hoạt động thế nào? Vì sao asset phải hash tên file mới dám `immutable`?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 99: Browser Cache vs CDN Cache vs Service Worker Cache

**Trả lời Senior:**
Ba tầng cache khác scope và lifecycle:

- **Browser Cache (HTTP Cache)**: per-user, per-browser, theo `Cache-Control`, `ETag`. Nhanh nhất (memory/disk), nhưng mỗi user warm riêng, xóa khi user clear. Dùng cho asset versioned, HTML.
- **CDN Cache (Edge Cache)**: per-PoP, share giữa users, theo `Cache-Control: s-maxage` hoặc `CDN-Cache-Control`, `surrogate-key`. Giảm TTFB, giảm origin load. Warm khi 1 user request, user khác hưởng. Invalidate bằng purge/tag.
- **Service Worker Cache (Cache API)**: programmable, per-origin, control hoàn toàn bằng JS (`caches.open`, `fetch` intercept). Cho offline, precache, stale-while-revalidate custom. Tồn tại sau khi browser cache clear? Không, nhưng control tinh hơn.

```typescript
// Browser + CDN - headers
// Browser 1 năm, CDN 1 ngày, stale 1 tuần
res.set('Cache-Control', 'public, max-age=31536000, immutable');
res.set('CDN-Cache-Control', 'public, s-maxage=86400, stale-while-revalidate=604800');
res.set('Surrogate-Key', 'static app'); // Fastly/Cloudflare tag để purge

// Service Worker - Cache API
// sw.js
const CACHE = 'v1';
self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(['/offline.html', '/app.js', '/hero.jpg'])));
});
self.addEventListener('fetch', e => {
  e.respondWith(
    caches.match(e.request).then(cached => {
      const fetched = fetch(e.request).then(res => {
        if (res.ok) caches.open(CACHE).then(c => c.put(e.request, res.clone()));
        return res;
      });
      // stale-while-revalidate
      return cached ?? fetched;
    })
  );
});

// So sánh
// | | Scope | Control | Invalidate | Offline |
// |---|---|---|---|---|
// | Browser | per user | header | time | không |
// | CDN | per PoP, shared | header + purge | purge/tag | không |
// | SW | per origin, JS | code | code | có |

// Next.js - CDN + Browser
// Vercel tự CDN cache cho static, ISR
// fetch('/api/products', { next: { revalidate: 60, tags: ['products'] } })
// revalidateTag('products') -> purge CDN + Data Cache
```

**Trade-off:** CDN cache share nên hit rate cao, nhưng purge chậm (vài giây-phút). SW cache mạnh nhưng thêm complexity, phải handle update, và chỉ chạy trên HTTPS. Đừng cache HTML cá nhân hóa ở CDN.

**Câu hỏi đào sâu:** `s-maxage` vs `max-age` khác gì? Khi nào dùng `stale-while-revalidate` ở CDN? Service Worker cache khác HTTP cache thế nào về lifecycle?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 100: Render 10k rows - virtualization

**Trả lời Senior:**
Render 10k DOM node cùng lúc giết performance: 10k * 50 elements = 500k node, layout 500ms, memory 100MB, scroll FPS 10. Giải pháp là **virtualization (windowing)**: chỉ render **viewport + overscan**, còn lại là padding. Khi scroll, recycle DOM.

Với 10k row, chỉ 20-30 row visible, virtualization giảm DOM 99%, FPS 60.

```typescript
// ❌ Chết: render hết
function BadList({ items }: { items: Item[] }) {
  return <div>{items.map(item => <Row key={item.id} item={item} />)}</div>; // 10k DOM
}

// ✅ Virtualization với @tanstack/react-virtual
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50, // chiều cao row
    overscan: 5, // render thêm 5 row ngoài viewport
  });
  const virtualItems = virtualizer.getVirtualItems();
  return (
    <div ref={parentRef} style={{ height: 500, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualItems.map(v => (
          <div
            key={v.key}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${v.start}px)`,
              height: v.size,
              width: '100%',
            }}
          >
            <Row item={items[v.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}

// Với react-window (cũ hơn, nhẹ)
import { FixedSizeList } from 'react-window';
<FixedSizeList height={500} itemCount={items.length} itemSize={50} width="100%">
  {({ index, style }) => <div style={style}><Row item={items[index]} /></div>}
</FixedSizeList>
```

**Trade-off:** Virtualization thêm complexity: scroll to index, dynamic height, sticky header khó hơn. Chỉ dùng khi > 100 item hoặc mỗi row nặng. Với < 50 item, render thường đủ.

**Câu hỏi đào sâu:** Vì sao virtualization dùng `transform: translateY` thay vì `top`? Overscan là gì và bao nhiêu là đủ?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 101: Virtualization hoạt động thế nào? So sánh thư viện

**Trả lời Senior:**
Virtualization core là **đo viewport, tính range visible, chỉ mount range đó**. Bước:

1.  Đo `scrollTop`, `viewportHeight`, `totalSize = count * estimateSize`.
2.  Tính `startIndex = floor(scrollTop / estimateSize)`, `endIndex = ceil((scrollTop + viewportHeight) / estimateSize)`.
3.  Render `items.slice(startIndex, endIndex + overscan)` với `position: absolute` + `transform: translateY(start)`.
4.  Khi scroll, update range, **recycle DOM** (không tạo mới hết).
5.  Với **dynamic height**: đo thực tế mỗi row (`measureElement`), cache size, update `totalSize`.

So sánh:

- **tanstack-virtual**: headless, framework-agnostic, dynamic height tốt, API `useVirtualizer`, support cả virtual horizontal, đang là chuẩn mới. Nhẹ, control cao.
- **react-window / react-virtualized**: cũ, `FixedSizeList`/`VariableSizeList`, API render prop, ít update, nhưng stable và nhẹ (3kb). `react-virtualized` nặng hơn, nhiều util.
- **virtuoso**: cao cấp, handle grouped, sticky, chat, nhưng nặng hơn.

```typescript
// tanstack-virtual - dynamic height
function VirtualDynamic({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    // đo thực tế
    measureElement: el => el?.getBoundingClientRect().height ?? 50,
  });
  return (
    <div ref={parentRef} style={{ height: 500, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map(v => (
          <div key={v.key} data-index={v.index} ref={virtualizer.measureElement} style={{ position: 'absolute', top: 0, transform: `translateY(${v.start}px)`, width: '100%' }}>
            <Row item={items[v.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}

// react-window - fixed
import { FixedSizeList } from 'react-window';
<FixedSizeList height={500} itemCount={10000} itemSize={35} width={300}>
  {({ index, style }) => <div style={style}>{items[index].name}</div>}
</FixedSizeList>

// Khi nào không virtualize?
// - < 100 item
// - Cần SEO (bot không scroll)
// - Cần find-in-page (Ctrl+F) -> virtual làm mất
```

**Trade-off:** `tanstack-virtual` linh hoạt nhưng phải tự style, `react-window` đơn giản nhưng API cũ. Dynamic height làm scroll hơi giật nếu estimate sai nhiều.

**Câu hỏi đào sâu:** Dynamic height đo thế nào mà không gây layout thrashing? Vì sao `Ctrl+F` không work với virtual list?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 102: Scroll lag debug - long task, layout thrashing, compositor

**Trả lời Senior:**
Scroll lag (jank) là FPS < 60 khi scroll, do **main thread bận** hoặc **compositor không handle được**. Debug theo thứ tự:

1.  **Performance Tab**: record khi scroll, xem **long task** (yellow), **Layout** (purple), **Paint** (green), **FPS** (red). Nếu long task chiếm > 50ms/frame -> JS nặng.
2.  **Layout Thrashing**: đọc `offsetHeight`/`getBoundingClientRect` rồi ghi `style` trong loop/scroll handler. Mỗi lần đọc sau ghi sẽ **force reflow**. Fix bằng batch đọc/ghi, dùng `requestAnimationFrame`, hoặc `transform` thay `top/left`.
3.  **Compositor**: scroll nên chạy trên **compositor thread** (không cần main thread). Thuộc tính trigger layout/paint (`width`, `top`, `left`) bắt main thread, còn `transform`, `opacity` chạy trên compositor, mượt 60fps. Dùng `will-change: transform` hoặc `translateZ(0)` để promote layer, nhưng đừng lạm dụng (tốn memory).

```typescript
// ❌ Scroll handler nặng + thrashing
window.addEventListener('scroll', () => {
  for (const el of elements) {
    const top = el.getBoundingClientRect().top; // đọc -> force layout
    el.style.top = top + 'px'; // ghi -> invalidate layout
  }
});

// ✅ Fix: rAF + batch + transform
let ticking = false;
window.addEventListener('scroll', () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      // batch đọc
      const tops = elements.map(el => el.getBoundingClientRect().top);
      // batch ghi bằng transform (compositor)
      elements.forEach((el, i) => (el.style.transform = `translateY(${tops[i]}px)`));
      ticking = false;
    });
    ticking = true;
  }
});

// ✅ Dùng IntersectionObserver thay scroll cho lazy/reveal
const io = new IntersectionObserver(entries => {
  entries.forEach(e => { if (e.isIntersecting) e.target.classList.add('visible'); });
});
elements.forEach(el => io.observe(el));

// Debug
// DevTools -> Rendering -> FPS meter, Layout Shift Regions, Paint flashing
// Performance -> Event Log -> Layout, Recalculate Style
```

**Trade-off:** `will-change` tạo layer mới, tốn GPU memory, chỉ dùng cho element thực sự animate. `rAF` giảm lag nhưng thêm 1 frame delay.

**Câu hỏi đào sâu:** Vì sao `transform` chạy trên compositor còn `top` thì không? `will-change` lạm dụng gây gì?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 103: Memory Leak trong React - nguyên nhân và cách phát hiện

**Trả lời Senior:**
Memory leak trong SPA là memory tăng dần không giảm sau GC, dẫn tới tab crash. Trong React, nguồn leak chính:

1.  **Quên cleanup**: `addEventListener`, `setInterval`, `setTimeout`, `MutationObserver`, `WebSocket`, `IntersectionObserver` không remove khi unmount.
2.  **Closure giữ DOM lớn**: `useEffect` closure bắt `element` 2MB, dù DOM đã remove nhưng closure vẫn giữ.
3.  **Subscription không unsubscribe**: `store.subscribe`, `bus.on`, `RxJS subscribe` không off.
4.  **Cache Map không giới hạn**: `Map` cache API response không evict, `WeakMap` thì không leak nhưng không iterate được.
5.  **Detached DOM**: giữ reference tới subtree đã remove (`let detached = ref.current`).

Phát hiện bằng **Chrome Memory tab**: Heap snapshot, Allocation timeline, Detached elements.

```typescript
// ❌ Leak: không cleanup
useEffect(() => {
  const onScroll = () => console.log(window.scrollY);
  window.addEventListener('scroll', onScroll);
  const id = setInterval(tick, 1000);
  // quên return cleanup -> leak khi unmount
}, []);

// ✅ Fix
useEffect(() => {
  const onScroll = () => console.log(window.scrollY);
  window.addEventListener('scroll', onScroll);
  const id = setInterval(tick, 1000);
  const ws = new WebSocket('wss://...');
  return () => {
    window.removeEventListener('scroll', onScroll);
    clearInterval(id);
    ws.close();
  };
}, []);

// ❌ Closure giữ DOM
function useLeaky() {
  const ref = useRef<HTMLDivElement>(null);
  useEffect(() => {
    const el = ref.current; // 2MB subtree
    const handler = () => console.log(el); // closure giữ el dù đã unmount
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler); // phải remove
  }, []);
}

// ✅ WeakMap cho cache không leak
const cache = new WeakMap<object, string>();
let obj = { data: 'big' };
cache.set(obj, 'meta');
obj = null; // GC được, entry tự mất

// Debug
// DevTools -> Memory -> Take heap snapshot -> tìm Detached
// Performance -> Memory checkbox -> xem JS Heap tăng khi navigate
// why-did-you-render không liên quan leak, nhưng giúp tìm retained
```

**Trade-off:** `WeakMap`/`WeakRef` không leak nhưng không có `size`, không iterate. Đừng dùng `WeakMap` khi cần duyệt cache.

**Câu hỏi đào sâu:** Detached DOM là gì và tìm thế nào trong DevTools? Vì sao `setInterval` trong `useEffect` không cleanup sẽ leak cả component?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 104: Performance Budget và monitoring production

**Trả lời Senior:**
Tối ưu 1 lần rồi thả là vô nghĩa, performance sẽ drift. Senior đặt **performance budget** và **monitoring** để giữ.

Budget là ngưỡng không được vượt: **JS < 200kb (gz), LCP < 2.5s, INP < 200ms, CLS < 0.1, Bundle < 300kb**. Vượt thì CI fail. Dùng `bundlesize`, `lighthouse-ci`, `next/bundle-analyzer`.

Monitoring production: **RUM (Real User Monitoring)** qua `web-vitals` gửi về analytics (Datadog, Sentry, New Relic), **Sentry Performance** cho transaction, **CrUX** cho so sánh. Alert khi p75 vượt ngưỡng.

```typescript
// 1. Budget - bundlesize
// package.json
{
  "bundlesize": [
    { "path": "./dist/*.js", "maxSize": "200 kB" },
    { "path": "./dist/vendor.js", "maxSize": "100 kB" }
  ]
}
// CI: npx bundlesize

// 2. Lighthouse CI
// lighthouserc.json
{
  "ci": {
    "assert": {
      "assertions": {
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }],
        "interactive": ["warn", { "maxNumericValue": 3000 }]
      }
    }
  }
}

// 3. RUM - web-vitals
import { onLCP, onINP, onCLS } from 'web-vitals';
function send(metric: any) {
  fetch('/analytics', {
    method: 'POST',
    body: JSON.stringify({ name: metric.name, value: metric.value, rating: metric.rating, id: metric.id }),
    keepalive: true,
  });
}
onLCP(send);
onINP(send);
onCLS(send);

// 4. Sentry Performance
import * as Sentry from '@sentry/react';
Sentry.init({
  dsn: '...',
  tracesSampleRate: 0.1,
  integrations: [Sentry.browserTracingIntegration()],
});
// Tự đo transaction, LCP, INP

// 5. Next.js - next.config.js
// experimental: { instrumentationHook: true }
// instrumentation.ts: register Web Vitals

// Dashboard: p75 theo route, device, country
// Alert: Slack khi LCP p75 > 2.5s 5 phút liên tục
```

**Trade-off:** Budget quá chặt làm dev khó ship feature, quá lỏng thì vô tác dụng. Đặt budget theo **user impact**, không phải số đẹp. RUM tốn request, sample 10% là đủ.

**Câu hỏi đào sâu:** Làm sao đặt budget hợp lý cho từng route? RUM sample bao nhiêu % là đủ mà không tốn cost?

[↑ Quay lại Mục lục](#mục-lục)
---
