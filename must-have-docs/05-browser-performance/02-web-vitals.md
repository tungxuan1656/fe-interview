# Core Web Vitals — LCP/INP/CLS, FID vs INP, RUM vs Lab (Lighthouse/CrUX)

> Tags: #web-vitals #lcp #inp #cls #fid #rum #crux #lighthouse #performance-observer | Nguồn: `docs/05-performance.md` câu 87-92 | Mức: P0

## 1. Định nghĩa chính xác

**Core Web Vitals (CWV)** là 3 chỉ số Google dùng để xếp hạng SEO và đo UX thực tế trên **field data (p75, 28 ngày)**: **LCP (Largest Contentful Paint)** đo loading — thời gian từ `navigationStart` tới khi phần tử lớn nhất trong viewport paint xong; **INP (Interaction to Next Paint)** đo interactivity — độ trễ từ user tương tác (click/tap/keypress) tới khi frame tiếp theo paint, lấy **p98 tồi nhất** trên vòng đời trang, thay thế **FID** từ 03/2024; **CLS (Cumulative Layout Shift)** đo visual stability — tổng `score = impactFraction × distanceFraction` của các layout shift bất ngờ không do user.

Ngưỡng Good: **LCP < 2.5s**, **INP < 200ms**, **CLS < 0.1**. Needs improvement: LCP < 4s, INP < 500ms, CLS < 0.25. Poor: vượt trên.

**FID (First Input Delay)** chỉ đo `input delay` của tương tác đầu tiên (thời gian chờ main thread rảnh), bỏ qua `processing` và `presentation delay`. **INP** đo toàn bộ `input delay + processing duration + presentation delay` cho **mọi** tương tác.

## 2. Cơ chế hoạt động

### 2.1 LCP

- Browser liên tục xác định **LCP candidate**: `img`, `image` trong `svg`, `video poster`, `background-image: url()`, block-level text node lớn. Kích thước tính theo **rendered size trong viewport** (không phải intrinsic).
- Candidate cập nhật tới khi user tương tác đầu tiên hoặc scroll. Giá trị cuối là `largestContentfulPaint` entry lớn nhất.
- 4 nguyên nhân LCP chậm (theo web.dev, thứ tự ưu tiên fix): **TTFB** (>800ms chết LCP) → **Resource Load Delay** (hero image chưa preload) → **Render Delay** (JS/CSS block, client render muộn) → **Element Render Delay** (LCP element do `useEffect` fetch mới tạo).

### 2.2 INP

```
INP = inputDelay + processingDuration + presentationDelay
       |--- event → main rảnh ---|--- handler + React render/commit ---|--- paint frame tiếp theo ---|
```

- FID chỉ đo đoạn đầu. INP đo toàn bộ và quan sát **mọi** tương tác trong page lifecycle, sau đó lấy **p98** (gần worst) — phản ánh trải nghiệm tệ nhất, không phải may mắn lần đầu. Interaction lặp lại cùng loại được gộp, lấy max.
- INP cao do **main thread bận**: long task > 50ms, React render lớn (10k item filter đồng bộ trong handler), third-party script, event handler nặng.

### 2.3 CLS

- Mỗi **layout shift** (không do user `click`/`keydown` trong 500ms trước) tính: `score = impactFraction × distanceFraction`.
  - `impactFraction`: % viewport bị ảnh hưởng (union của rect trước và sau).
  - `distanceFraction`: khoảng dịch lớn nhất của frame bị dịch / viewport dimension lớn nhất.
- CLS là **tổng không trọng số** các shift score trong vòng đời trang (tới `pagehidden`; cập nhật 2025 đo tới `pagehidden` thay vì vô hạn). Không có window 5s nữa, nhưng vẫn nhóm theo session window để debug.
- Nguyên nhân: `img`/`iframe` không `width/height`, `font swap` (FOIT→FOUT), ads/banner inject không reserve, dynamic content, animation bằng `top/left` thay `transform`.

### 2.4 Lab vs Field

- **Lab (Lighthouse)**: môi trường giả lập (Moto G4, throttled 4G, CPU 4× slowdown), cho điểm 0-100, TBT/TBT thay INP. Dùng để **regression trong CI**, không phản ánh user thật (máy yếu, 3G, extension).
- **Field (RUM + CrUX)**: đo trên user thật qua `PerformanceObserver` / `web-vitals` lib, gửi về analytics, tính **p75** (75th percentile — 75% user tốt hơn ngưỡng). Google xếp hạng SEO bằng **field CrUX**, không phải lab.
- **p75 vs p98 vs average**: `average` bị outlier kéo lệch; `p75` đại diện đa số; INP dùng `p98` để bắt worst interaction.

### 2.5 PerformanceObserver & web-vitals

Browser expose metrics qua `PerformanceObserver` (`largest-contentful-paint`, `event`, `layout-shift`, `navigation`). Thư viện `web-vitals` wrap observer, handle buffering, attribution, và CrUX alignment.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Đo RUM với web-vitals (field)
import { onLCP, onINP, onCLS } from 'web-vitals';
import { onLCP as onLCPAttr } from 'web-vitals/attribution';

function sendToAnalytics(metric: any) {
  fetch('/analytics', {
    method: 'POST',
    body: JSON.stringify({
      name: metric.name,      // 'LCP' | 'INP' | 'CLS'
      value: metric.value,
      rating: metric.rating,  // 'good' | 'needs-improvement' | 'poor'
      id: metric.id,
      attribution: metric.attribution,
      navigationType: metric.navigationType, // 'navigate' | 'prerender' | 'bfcache'
    }),
    keepalive: true, // gửi được cả khi page unload
  });
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);

// Attribution — biết nguyên nhân
onLCPAttr(metric => {
  console.log('LCP element', metric.attribution.element);
  console.log('LCP URL', metric.attribution.url);
  console.log('TTFB', metric.attribution.timeToFirstByte);
  console.log('ResourceLoadDelay', metric.attribution.resourceLoadDelay);
});
```

```js
// 3.2 PerformanceObserver thô (không cần lib)
new PerformanceObserver(list => {
  for (const entry of list.getEntries()) {
    console.log('LCP', entry.startTime, entry.element);
  }
}).observe({ type: 'largest-contentful-paint', buffered: true });

new PerformanceObserver(list => {
  for (const entry of list.getEntries()) {
    // entry là PerformanceEventTiming
    console.log('INP candidate', entry.duration, entry.interactionId);
  }
}).observe({ type: 'event', buffered: true, durationThreshold: 16 });

new PerformanceObserver(list => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) console.log('CLS shift', entry.value);
  }
}).observe({ type: 'layout-shift', buffered: true });
```

```html
<!-- 3.3 Tối ưu LCP — hero preload + fetchpriority -->
<head>
  <link rel="preconnect" href="https://cdn.example.com" crossorigin />
  <link
    rel="preload"
    as="image"
    href="/hero-800.webp"
    imagesrcset="/hero-800.webp 800w, /hero-1200.webp 1200w"
    imagesizes="(max-width: 768px) 100vw, 50vw"
    fetchpriority="high"
  />
  <style>/* inline critical CSS — hero, header */</style>
  <script src="/app.js" defer></script>
</head>
<body>
  <!-- LCP candidate — không lazy, có size, priority high -->
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

```tsx
// 3.4 INP — tách urgent vs non-urgent
// ❌ Handler nặng block INP 300ms
function Bad({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('');
  const [filtered, setFiltered] = useState(items);
  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const v = e.target.value;
    setFilter(v);
    setFiltered(items.filter(i => i.name.includes(v))); // 300ms sync
  };
  return <input value={filter} onChange={onChange} />;
}

// ✅ Defer non-urgent — INP thấp, input mượt
import { useDeferredValue, useMemo, useTransition } from 'react';

function Good({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('');
  const deferred = useDeferredValue(filter);
  const filtered = useMemo(() => items.filter(i => i.name.includes(deferred)), [items, deferred]);
  const [isPending, startTransition] = useTransition();
  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setFilter(e.target.value); // urgent — INP tốt
    // filter nặng chạy ở TransitionLane, không block paint
  };
  return (
    <>
      <input value={filter} onChange={onChange} />
      {isPending ? <Spinner /> : <List items={filtered} />}
    </>
  );
}
```

```html
<!-- 3.5 CLS fix — reserve space -->
<!-- ❌ Không size → shift khi load -->
<img src="/photo.jpg" />

<!-- ✅ Reserve bằng width/height + aspect-ratio -->
<img src="/photo.jpg" width="800" height="600" style="aspect-ratio: 800/600; width: 100%; height: auto" alt="" />
<div style="aspect-ratio: 16/9; background: #eee">
  <img src="/photo.jpg" style="width: 100%; height: 100%; object-fit: cover" alt="" />
</div>
<!-- Ads/Dynamic: reserve trước -->
<div id="ads" style="min-height: 250px; background: #f5f5f5"></div>
```

## 4. So sánh / Phân loại

| Chỉ số | Đo gì | Ngưỡng Good | Ngưỡng Poor | Cách cải thiện chính |
|--------|-------|-------------|------------|----------------------|
| **LCP** | Loading — paint phần tử lớn nhất | < 2.5s | > 4.0s | Preload hero, CDN, `fetchpriority=high`, inline critical CSS, SSR cho hero, `width/height` |
| **INP** | Interactivity — delay tới next paint (p98) | < 200ms | > 500ms | Break long task, `useTransition`/`useDeferredValue`, giảm JS, debounce, `scheduler.yield` |
| **CLS** | Visual stability — tổng shift score | < 0.1 | > 0.25 | Reserve `width/height`/`aspect-ratio`, `size-adjust` cho font, `transform` thay `top` |

| Tiêu chí | FID | INP |
|----------|-----|-----|
| Đo | Chỉ tương tác **đầu tiên** | **Mọi** tương tác, lấy **p98 worst** |
| Thành phần | Chỉ `input delay` (event → main rảnh) | `input delay + processing + presentation` (tới paint) |
| Thay thế | Deprecated 03/2024 | Thay FID |
| Phản ánh | May mắn lần đầu | Trải nghiệm tệ nhất — sát UX thật |

| Tiêu chí | Lab (Lighthouse) | Field (RUM/CrUX + `web-vitals`) |
|----------|-----------------|---------------------------------|
| Môi trường | Giả lập (throttled Moto G4, 4× CPU) | User thật, đa dạng device/network |
| Dùng cho | Regression trong CI, debug local | SEO ranking, alert production |
| Chỉ số | LCP (lab), TBT (thay INP), CLS (lab) | LCP/INP/CLS p75 thực |
| Hạn chế | Không phản ánh 3G, máy yếu, extension | Cần đủ traffic, 28 ngày mới ổn định |
| SEO | Không | Google dùng **field CrUX** |

| Công cụ | Vai trò |
|---------|---------|
| `PerformanceObserver` | API thô: `largest-contentful-paint`, `event`, `layout-shift`, `navigation` |
| `web-vitals` lib | Wrap observer, buffering, attribution, `keepalive` send |
| `web-vitals/attribution` | Cho biết `element`, `url`, `timeToFirstByte`, `resourceLoadDelay`, `inputDelay` |
| Lighthouse CI (`lhci`) | Assert `largest-contentful-paint < 2500`, `cumulative-layout-shift < 0.1` trong CI |
| CrUX API / PageSpeed Insights | Query p75 field: `https://chromeuxreport.googleapis.com/v1/records:queryRecord` |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không tối ưu LCP bằng cách preload tất cả ảnh**: chỉ preload **1 LCP above-the-fold**. Preload 3+ ảnh tranh bandwidth với hero, làm LCP tệ hơn. `fetchpriority="high"` chỉ cho hero, không cho mọi `<img>`.
- **Không fix INP bằng cách `useTransition` cho mọi state**: `startTransition` thêm trạng thái `isPending`/fallback, làm UI phức tạp. Chỉ defer phần **non-urgent** (filter list, search results). Input value luôn urgent.
- **Không fix CLS bằng `font-display: optional` mọi nơi**: `optional` tránh swap/CLS nhưng có thể không hiện font custom trên 3G → brand lệch. Cân bằng `swap` + `size-adjust`/`ascent-override` để fallback metric gần font chính.
- **Không chỉ nhìn Lighthouse điểm 100 rồi yên tâm**: lab 100 vẫn có thể field Poor vì user 3G, máy yếu, third-party khác. Luôn kết hợp RUM p75. Ngược lại, RUM cần 28 ngày mới ổn định — lab cho feedback ngay trong PR.
- **Không dùng `loading="lazy"` cho LCP image**: lazy defer tới khi gần viewport, làm LCP tăng 500ms-1s. Hero luôn `eager` (mặc định) + `fetchpriority="high"`.
- **Không lấy `average` cho Web Vitals**: average bị outlier kéo; Google dùng **p75** (và INP p98) để đảm bảo 75% user đạt. Average đẹp nhưng che 25% user tệ.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: LCP là `background-image` không preload được**
  - Triệu chứng: Lighthouse báo LCP element là `div` với `background-image`, LCP 3.5s.
  - Nguyên nhân: `background-image` không có `preload` priority, browser phát hiện muộn (sau CSSOM).
  - Fix: đổi sang `<img>` với `fetchpriority="high"` hoặc `<link rel="preload" as="image">` nếu bắt buộc background.
  - Đo: Lighthouse → "Largest Contentful Paint element"; DevTools Performance → `LCP` marker; `web-vitals/attribution` → `resourceLoadDelay`.

- **Lỗi 2: INP cao do third-party + long task 300ms**
  - Triệu chứng: INP 450ms field, CrUX Poor, `web-vitals` attribution `processingDuration` lớn, `interactionTarget` là button gần chat widget.
  - Fix: `defer`/`async` third-party, `Partytown`/`web worker`, break handler bằng `scheduler.yield()`/`setTimeout(0)`, `useTransition`.
  - Đo: DevTools Performance → Record tương tác → Long task yellow >50ms; `onINP(m => console.log(m.attribution))`; Lighthouse TBT.

- **Lỗi 3: CLS 0.25 do ảnh không size + font swap**
  - Triệu chứng: Experience → Layout Shifts đỏ, CLS Poor.
  - Fix: thêm `width`/`height` + `aspect-ratio`; `@font-face { size-adjust: 107%; ascent-override: 90% }` hoặc `next/font` tự fallback metric; reserve `min-height` cho ads.
  - Đo: Performance → Experience → Layout Shifts; `onCLS(m => console.log(m.attribution.largestShiftTarget))`; Lighthouse "Cumulative Layout Shift".

- **Lỗi 4: Lab Good nhưng Field Poor**
  - Triệu chứng: Lighthouse 95, CrUX LCP 3.8s.
  - Nguyên nhân: lab mô phỏng Moto G4 mid-tier, field có 30% user low-end + 3G + extension.
  - Fix: RUM segmentation theo device/country; image CDN `srcset` nhỏ cho low-end; giảm JS cho INP.
  - Đo: CrUX API theo `origin`; `web-vitals` send kèm `deviceMemory`, `effectiveType`.

- **Lỗi 5: p75 tốt nhưng p98 INP tệ**
  - Triệu chứng: INP p75 180ms (Good) nhưng p98 600ms, user phàn nàn lag khi filter.
  - Fix: không chỉ nhìn p75, check `web-vitals` distribution; break chunk `processLargeArray` với `await yieldToMain()` mỗi 100 item.
  - Đo: `onINP` log `metric.value` distribution; Performance → `Event Timing`.

- **Công cụ**:
  - **Lighthouse CI**: `lighthouserc.json` assert `largest-contentful-paint < 2500`, `cumulative-layout-shift < 0.1`.
  - **web-vitals + CrUX**: `npx web-vitals`, PageSpeed Insights, `chromeuxreport.googleapis.com`.
  - **React Profiler**: commit duration, why did render — tìm wasted render gây INP.
  - **Performance tab**: `LCP`, `Layout Shift`, `Long task`, `FPS`.

## 7. Câu hỏi tự kiểm tra

1. LCP < 2.5s, INP < 200ms, CLS < 0.1 là ngưỡng p75 field. Vì sao Google dùng p75 (và INP p98) thay vì average, và vì sao Lighthouse lab 100 vẫn có thể field Poor?
2. FID chỉ đo input delay của tương tác đầu tiên, INP đo toàn bộ input delay + processing + presentation cho mọi tương tác (lấy p98). Vì sao `items.filter()` đồng bộ 300ms trong `onChange` giết INP còn `useDeferredValue`/`startTransition` lại cứu được?
3. Phân biệt Lab vs Field vs CrUX vs RUM tự đo: mỗi cái lấy data từ đâu, dùng khi nào, và `web-vitals` lib + `PerformanceObserver` đóng vai trò gì?

<details>
<summary>Đáp án 30s</summary>

1. **Average** bị outlier (1% user 10s) kéo lệch, không đại diện. **p75** đảm bảo 75% user đạt ngưỡng — công bằng hơn. **INP p98** bắt worst interaction, tránh may mắn lần đầu. Lab là giả lập Moto G4 + throttled, không phản ánh 30% user máy yếu/3G/extension → lab Good nhưng field Poor. SEO dùng field CrUX 28 ngày, không dùng lab.

2. FID bỏ qua `processing` (React render) và `presentation` (paint). `filter` 300ms trong handler block main thread → `processingDuration` 300ms → INP = inputDelay + 300ms + presentationDelay > 500ms. `useDeferredValue`/`startTransition` tách urgent (`setQuery`) ra InputContinuousLane (commit ngay, INP thấp) và non-urgent (`filter`) ra TransitionLane (có thể yield/interrupt), không block paint của frame tiếp theo.

3. **Lab (Lighthouse)**: chạy local giả lập, cho TBT thay INP, dùng regression CI. **Field**: user thật. **CrUX**: Google thu thập field tự động từ Chrome opt-in, 28 ngày, public qua API/PageSpeed — dùng so sánh. **RUM tự đo**: `web-vitals` (wrap `PerformanceObserver` `largest-contentful-paint`/`event`/`layout-shift`) gửi `keepalive` về analytics của mình — chi tiết per route/device, alert ngay. Cần cả hai: lab cho PR feedback nhanh, RUM/CrUX cho SEO và production monitoring.

</details>

---
*Tham khảo chi tiết: `docs/05-performance.md` — Câu 87-92. Spec: [web.dev — Web Vitals](https://web.dev/articles/vitals), [web-vitals library](https://github.com/GoogleChrome/web-vitals), [CrUX](https://developer.chrome.com/docs/crux).*
