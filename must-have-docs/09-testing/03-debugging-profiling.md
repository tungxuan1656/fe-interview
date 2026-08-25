# Debugging & Profiling — Profiler Flame vs Ranked, why-did-you-render, Long Task, Detached HTMLElement, Sentry, web-vitals

> Tags: #Debugging #Profiler #why-did-you-render #Long-Task #Heap-Snapshot #Detached-HTMLElement #Sentry #web-vitals #PerformanceObserver | Nguồn: `docs/13-debugging.md` 10 tình huống + `docs/10-testing.md` câu 172 + `docs/18-production-scenarios.md` scenario 01/02/04 | Mức: P0

## 1. Định nghĩa chính xác

**React DevTools Profiler** đo **commit** (lần React flush fiber → DOM): **Flame chart** hiển thị **tree theo thời gian** (parent trên, child dưới, thanh dài = render lâu), **Ranked chart** xếp **component tốn thời gian nhất** giảm dần (flatten, không quan hệ cha-con). **`why-did-you-render` (WDYR)** patch `React.memo`/`PureComponent` để log **vì sao component re-render**: `props` object reference đổi, `state` đổi, `context` đổi, `hook` đổi. **Long Task** là task main thread **>50ms** block `input`/`paint` (INP). **Heap Snapshot + Detached HTMLElement** là tool Memory: snapshot so sánh **before/after navigate** tìm **DOM node bị giữ** (detached nhưng JS vẫn reference → leak). **Sentry** (error monitoring) thu `error`, `breadcrumbs`, `release` (commit SHA), `source map`, `replay` để triage production. **`web-vitals` (PerformanceObserver)** đo **LCP/INP/CLS field (p75)** trên user thật (RUM).

## 2. Cơ chế hoạt động

- **React Profiler — Flame vs Ranked**:
  - **Profiler** record bằng `scheduler` + `ReactFiberWorkLoop`: mỗi **commit** ghi `commitTime`, `duration` (tổng render time của commit), `actualDuration` per component (time trong `render` + `commit`), `baseDuration` (estimate không memo), `whyDidRender` tag (`props change`, `state change`, `context change`).
  - **Flame**: trục X = thời gian, Y = cây fiber (parent trên). Màu: **xám** = không render, **xanh → vàng → đỏ** theo tốn kém (so với siblings). Click 1 thanh → xem `props`/`state` trước/sau. Dùng để thấy **đường lan truyền** (Provider đổi → 50 con re-render).
  - **Ranked**: sort `actualDuration` giảm dần, không tree. Dùng để tìm **hotspot** nhanh nhất (top 3 component tốn 80% time).
  - **Interactions**: profiler chỉ đo **commit có flag `enableProfilerTimer`**, production cần `REACT_PROFILE` build.

- **why-did-you-render (WDYR)**:
  - Patch `React.createElement` + `memo`/`forwardRef`: shallow-equal `prevProps` vs `nextProps` mỗi render, log `differences: { propName: { prev, next, isEqual: false } }` + stack trace.
  - Cấu hình: `whyDidYouRender(React, { trackAllPureComponents: true, logOnDifferentValues: true })`. Chỉ bật **development**, tắt production (cost `O(n)` comparisons).
  - Dùng khi Profiler cho thấy re-render nhiều nhưng không biết **vì sao** (inline object `style={{}}` tạo reference mới mỗi render → memo vô tác dụng).

- **Chrome Performance — Long Task**:
  - Main thread chạy **Task** (script, layout, paint) trên **Event Loop**. Mỗi Task >50ms được mark **Long Task** (red bar) trong Performance track → block `Event Timing` (INP = `inputDelay + processing + presentation`).
  - Record: `Performance → Record 5s → interact (click/type) → stop`. Xem **Main track** → red `Task` >50ms → Bottom-up: `Function Call` nào chiếm (thường `items.filter` 300ms, `JSON.parse` lớn, third-party).
  - Fix: `useDeferredValue`/`startTransition` (React 18 TransitionLane), `scheduler.yield()`, `break task` (`setTimeout(0)` batch), debounce.

- **Memory — Heap Snapshot + Detached HTMLElement**:
  - **Heap Snapshot**: chụp **JS heap graph** (objects, DOM nodes, closures). **Allocation Timeline** record `malloc` theo thời gian; **Comparison** diff 2 snapshot (before/after navigate) → tìm **Detached HTMLElement** (DOM đã remove khỏi document nhưng JS vẫn giữ reference — thường do `addEventListener` không cleanup, `Map` cache, `IntersectionObserver` không `disconnect`, `WebSocket` không `close`).
  - **Performance Monitor** (3 chấm → More tools): xem `JS heap size` tăng dốc không giảm sau GC → leak.
  - Fix: `useEffect return () => { clearInterval(id); removeEventListener; ws.close(); obs.disconnect(); subscription.unsubscribe() }`.

- **Sentry — error monitoring**:
  - SDK `Sentry.init({ dsn, release: COMMIT_SHA, tracesSampleRate, replaysSessionSampleRate })` wrap `window.onerror` + `unhandledrejection` + `ErrorBoundary`. Mỗi error gửi `event` gồm `stack trace` (source map resolve), `breadcrumbs` (click, fetch, console), `tags` (`os.name`, `locale`, `release`, `user.id`), `replay` (DOM recording).
  - **Release Health**: group error theo `release` → biết deploy nào gây spike. **Source Map upload** (`sentry-cli sourcemaps upload`) để stack unminified.
  - Triage: `error rate` theo `release` → `Console/Network` → `git bisect` → `rollback` nếu impact lớn.

- **web-vitals — field metrics**:
  - `PerformanceObserver` observe `largest-contentful-paint` (LCP), `event` (INP, `durationThreshold: 40ms`), `layout-shift` (CLS, `hadRecentInput: false`). Thư viện `web-vitals` (`onLCP`, `onINP`, `onCLS`) wrap observer + buffering + attribution (`timeToFirstByte`, `resourceLoadDelay`, `element`), gửi `keepalive` về `/analytics` (RUM) → tính **p75 (75% user Good)** + **INP p98** (worst). Khác Lighthouse lab (giả lập Moto G4, TBT thay INP).

## 3. Ví dụ tối thiểu

```tsx
// 3.1 React Profiler — Flame vs Ranked + why-did-you-render
// App.tsx — bug: Context object mới mỗi render → 50 con re-render
import { createContext, useState, memo, useCallback, useMemo } from 'react';

// ❌ Bug — object mới mỗi render
const AppContext = createContext({ user: null, theme: 'light', setUser: () => {} });
function App() {
  const [user, setUser] = useState(null);
  // Mỗi App render tạo object mới → mọi consumer re-render dù theme không đổi
  return <AppContext.Provider value={{ user, theme: 'light', setUser }}><Dashboard /></AppContext.Provider>;
}
// Profiler Flame: App (3ms) → Dashboard (2ms) → 50 Card (mỗi 1ms) đều vàng
// Ranked: Card tốn 50ms tổng — top 1

// ✅ Fix — tách context + memo
const UserContext = createContext(null);
const ThemeContext = createContext('light');
function AppFixed() {
  const [user, setUser] = useState(null);
  const setUserCb = useCallback(setUser, []);
  // UserContext chỉ đổi khi user đổi; ThemeContext stable
  return (
    <UserContext.Provider value={user}>
      <ThemeContext.Provider value="light"><Dashboard /></ThemeContext.Provider>
    </UserContext.Provider>
  );
}

// why-did-you-render — bật dev để biết vì sao memo fail
// wdyr.ts
import React from 'react';
import whyDidYouRender from '@welldone-software/why-did-you-render';
if (process.env.NODE_ENV === 'development') {
  whyDidYouRender(React, { trackAllPureComponents: true, logOnDifferentValues: true });
}
// Console: "Card re-rendered because props.style changed: prev {} !== next {} (different reference)"
// → Fix: const style = useMemo(() => ({ margin: 10 }), []);

// Memo + stable prop
const Card = memo(function Card({ style, onClick }: { style: object; onClick: () => void }) {
  return <div style={style} onClick={onClick}>Card</div>;
});
function Parent() {
  const style = useMemo(() => ({ margin: 10 }), []); // stable
  const onClick = useCallback(() => {}, []); // stable
  return <Card style={style} onClick={onClick} />; // Profiler: Card xám (không render lại)
}

// Profiler API — đo commit duration trong code
import { Profiler } from 'react';
function onRender(id: string, phase: string, actualDuration: number, baseDuration: number) {
  if (actualDuration > 16) console.warn(`Slow commit ${id}: ${actualDuration}ms (base ${baseDuration}ms)`);
}
<Profiler id="Dashboard" onRender={onRender}><Dashboard /></Profiler>;
```

```ts
// 3.2 Chrome Performance — Long Task → INP
// ❌ Long Task 300ms trong handler → INP 350ms (Poor, >200ms Good)
function BadFilter({ items }: { items: { name: string }[] }) {
  const [query, setQuery] = useState('');
  const [filtered, setFiltered] = useState(items);
  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const v = e.target.value;
    setQuery(v);
    setFiltered(items.filter(i => i.name.includes(v))); // 10k items × sync 300ms → Long Task đỏ
  };
  return <input value={query} onChange={onChange} />;
}
// Performance → Main track: Task 312ms đỏ, Bottom-up: filter → p95 INP 450ms

// ✅ Fix — defer non-urgent
import { useDeferredValue, useMemo, useTransition } from 'react';
function GoodFilter({ items }: { items: { name: string }[] }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query); // non-urgent
  const filtered = useMemo(() => items.filter(i => i.name.includes(deferredQuery)), [items, deferredQuery]);
  // Hoặc: const [isPending, startTransition] = useTransition();
  // const onChange = e => { setQuery(e.target.value); startTransition(() => setFiltered(...)) }
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} /> {/* urgent — INP thấp */}
      <List items={filtered} />
    </>
  );
}
// Performance sau fix: Task 18ms xanh, INP 80ms Good

// Break large task thủ công
async function processLargeArray(arr: number[]) {
  for (let i = 0; i < arr.length; i += 100) {
    arr.slice(i, i + 100).forEach(doWork);
    if (i % 500 === 0) await new Promise(r => setTimeout(r, 0)); // yield to main
  }
}
```

```ts
// 3.3 Memory — Detached HTMLElement + fix
// ❌ Leak — thiếu cleanup
useEffect(() => {
  const id = setInterval(poll, 5000);
  const onScroll = () => {};
  window.addEventListener('scroll', onScroll);
  const ws = new WebSocket(url);
  const obs = new IntersectionObserver(cb);
  obs.observe(ref.current!);
  // thiếu return cleanup → detached nodes + listeners giữ
}, []);

// ✅ Đủ cleanup
useEffect(() => {
  const id = setInterval(poll, 5000);
  const onScroll = () => {};
  window.addEventListener('scroll', onScroll);
  const ws = new WebSocket(url);
  const obs = new IntersectionObserver(cb);
  const el = ref.current!;
  obs.observe(el);
  return () => {
    clearInterval(id);
    window.removeEventListener('scroll', onScroll);
    ws.close();
    obs.disconnect();
    obs.unobserve(el);
  };
}, []);

// Heap Snapshot workflow:
// 1. Memory → Take snapshot A (trước navigate)
// 2. Navigate dashboard → back 5 lần
// 3. Take snapshot B → View: Comparison → filter "Detached" → thấy Detached HTMLElement × 5
// 4. Click Detached node → Retainers pane → thấy closure giữ bởi `Map` / `EventListener`

// Console helper: getEventListeners(window) → count listeners chưa remove
// Allocation Timeline: record 30s, xem JS Heap tăng dốc không giảm sau GC → leak
```

```ts
// 3.4 Sentry — error monitoring + release
// main.tsx
import * as Sentry from '@sentry/react';
Sentry.init({
  dsn: 'https://xxx@o123.ingest.sentry.io/456',
  release: import.meta.env.VITE_COMMIT_SHA, // git SHA
  environment: import.meta.env.MODE,
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [Sentry.browserTracingIntegration(), Sentry.replayIntegration()],
});

// ErrorBoundary per route
class RouteErrorBoundary extends React.Component<{ children: React.ReactNode }, { hasError: boolean }> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    Sentry.captureException(error, { extra: { componentStack: info.componentStack } });
  }
  render() { return this.state.hasError ? <div>Lỗi. <button onClick={() => location.reload()}>Reload</button></div> : this.props.children; }
}

// ChunkLoadError auto-reload 1 lần (white screen sau deploy)
window.addEventListener('error', e => {
  if (e.message.includes('Loading chunk') || e.message.includes('ChunkLoadError')) {
    if (!sessionStorage.getItem('chunk-reload')) {
      sessionStorage.setItem('chunk-reload', '1');
      location.reload();
    }
  }
});

// Debug helper: Sentry.setTag('os', navigator.userAgent), setUser({ id, segment }), setTag('locale', ...)

// 3.5 web-vitals — RUM field
import { onLCP, onINP, onCLS } from 'web-vitals';
import { onLCP as onLCPAttr } from 'web-vitals/attribution';

function sendToAnalytics(metric: any) {
  fetch('/analytics', {
    method: 'POST',
    body: JSON.stringify({ name: metric.name, value: metric.value, rating: metric.rating, id: metric.id, attribution: metric.attribution }),
    keepalive: true,
  });
}
onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onLCPAttr(m => console.log('LCP element', m.attribution.element, 'TTFB', m.attribution.timeToFirstByte));

// PerformanceObserver thô
new PerformanceObserver(list => {
  for (const e of list.getEntries() as any) console.log('LCP', e.startTime, e.element);
}).observe({ type: 'largest-contentful-paint', buffered: true });
new PerformanceObserver(list => {
  for (const e of list.getEntries() as any) if (!e.hadRecentInput) console.log('CLS shift', e.value);
}).observe({ type: 'layout-shift', buffered: true });
```

## 4. So sánh / Phân loại

| Tiêu chí | Profiler **Flame** | Profiler **Ranked** | `why-did-you-render` |
|----------|-------------------|---------------------|----------------------|
| **Hiển thị** | **Tree** theo thời gian (parent trên, child dưới, X = time) | **List sort** `actualDuration` giảm dần (flatten) | **Console log** per component: `prevProps vs nextProps` diff |
| **Màu** | Xám = không render, xanh→vàng→đỏ = tốn kém | Tương tự, nhưng sort | Không màu, text diff |
| **Dùng khi** | Thấy **lan truyền** (Provider đổi → 50 con) | Tìm **hotspot** nhanh (top 3 tốn 80%) | Biết **vì sao** memo fail (inline object) |
| **Cost** | Thấp (chỉ khi record) | Thấp | Cao (`O(n)` shallow-equal, chỉ dev) |
| **Giới hạn** | Không cho biết *vì sao* reference đổi | Mất quan hệ cha-con | Chỉ log, không đo duration |

| Tiêu chí | **Long Task (>50ms)** | **Heap Snapshot / Detached HTMLElement** | **Sentry** | **web-vitals** |
|----------|----------------------|------------------------------------------|------------|----------------|
| **Đo** | Main thread block (INP) | Memory leak (JS heap, DOM retained) | **JS error**, `release` spike, `breadcrumbs` | **LCP/INP/CLS field p75** |
| **Tool** | Performance → Main track red bar | Memory → Heap Snapshot + Allocation Timeline + `getEventListeners` | `Sentry.init` + `ErrorBoundary` + `source map upload` | `PerformanceObserver` + `web-vitals` lib + CrUX |
| **Khi dùng** | Page load 8s, INP 500ms, input giật | Tab 150MB → 1.5GB sau 30 phút, `Aw Snap!` | White screen production, 10% user lỗi, `ChunkLoadError` | Lab Good nhưng field Poor, SEO, RUM alert |
| **Fix** | `useDeferredValue`/`startTransition`/`yield` | `useEffect cleanup` (`clearInterval`, `removeEventListener`, `ws.close`, `WeakMap`/`LRU`) | `ErrorBoundary per route` + `release tag` + `ChunkLoadError reload` | Preload hero, reserve `width/height`, break task |

| Quy trình Debug Senior (11 bước) | Tool |
|----------------------------------|------|
| 1. Reproduce (đúng URL/user/device) | Playwright replay, Sentry Replay |
| 2. Collect logs | Sentry, Datadog, CloudWatch |
| 3. Monitoring (spike error/latency) | Grafana, Sentry Release Health |
| 4. Browser/Device (chỉ iOS?) | BrowserStack, caniuse, UA |
| 5. Network (waterfall, 4xx/5xx) | DevTools Network, Charles |
| 6. JS Error (stack + source map) | Sentry, DevTools Sources |
| 7. Regression (`git bisect`) | `git bisect`, `git diff HEAD~1` |
| 8-11. Fix → Test → Deploy (canary) → Monitor 24h | Vitest, Playwright, Sentry RUM |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không bật Profiler/WDYR trong production**: Profiler `actualDuration` thêm overhead 5-10% khi record; WDYR patch `createElement` thêm `O(n)` shallow-equal mỗi render → chậm dev. Chỉ bật **local/dev** khi cần, tắt prod. Production profiling cần `REACT_PROFILE` build riêng.
- **Không dùng Flame khi chỉ cần hotspot**: nếu biết trang lag nhưng không cần biết lan truyền, **Ranked** nhanh hơn (top 3 component). Ngược lại, Ranked không thấy **vì sao** 50 Card cùng re-render → cần Flame để thấy Provider parent.
- **Không dùng WDYR thay Profiler**: WDYR log rất verbose (hàng trăm dòng mỗi interaction), làm console tràn, không đo duration. Dùng **Profiler trước** để tìm component tốn time, rồi **WDYR cho 2-3 component đó** để biết `prop` nào đổi.
- **Không fix Long Task bằng `useTransition` mọi nơi**: `startTransition` thêm `isPending`/`fallback`, phức tạp UI. Chỉ defer **non-urgent** (filter list, search results); `input value` luôn urgent. Break task bằng `setTimeout(0)` cũng thêm latency.
- **Không Heap Snapshot cho mọi lag**: snapshot tốn RAM, cần thao tác thủ công. Nếu lag do **render 50 lần** (Profiler thấy) thì không cần Memory; Memory chỉ khi **RAM tăng dốc** (Performance Monitor `usedJSHeapSize`).
- **Không Sentry thay console local**: Sentry tốn quota, cần `source map upload`, không thay `DevTools`. Dùng Sentry cho **production**, `console + Network` cho local.
- **Không chỉ nhìn Lighthouse lab**: lab 100 vẫn field Poor vì user 3G/máy yếu/extension. Luôn kết hợp **RUM `web-vitals` p75** cho SEO; lab cho **regression CI** (`LCP < 2.5s`, `CLS < 0.1`).
- **Chi phí**: Profiler + WDYR + Performance + Memory + Sentry + web-vitals cần 6 tab DevTools; nhưng mỗi tool cho tín hiệu khác — không đo bằng đoán.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Render 50 lần / vòng — Profiler Flame thấy 50 commits**
  - Triệu chứng: gõ 1 ký tự → toàn page re-render, Profiler 50 commits, INP 600ms.
  - Nguyên nhân: `AppContext` object mới mỗi render, `key={Math.random()}`, `inline style={{}}` làm memo vô tác dụng.
  - Fix: tách `UserContext`/`ThemeContext`, `useMemo`/`useCallback` cho `style`/`onClick`, `useDeferredValue` cho filter.
  - Đo: **Profiler Flame** (parent vàng → 50 child vàng), **Ranked** (top `Card` 50ms), **WDYR** log `props.style changed: {} !== {}`, `vitest --coverage` không liên quan nhưng `React Profiler API onRender` log `actualDuration > 16ms`.

- **Lỗi 2: WDYR không bật → không biết vì sao memo fail**
  - Triệu chứng: `memo(Card)` nhưng vẫn re-render, Profiler không cho biết prop nào đổi.
  - Fix: bật `whyDidYouRender(React, { trackAllPureComponents: true })` dev, sửa `style` stable.
  - Đo: Console `whyDidYouRender` diff: `differences: { style: { prev: {}, next: {}, isEqual: false } }`; DevTools `getEventListeners` không liên quan.

- **Lỗi 3: Long Task 300ms → INP Poor (450ms)**
  - Triệu chứng: field `onINP` 450ms, Lighthouse TBT cao, Performance Main track red bar 312ms.
  - Fix: `useDeferredValue`/`startTransition`, break `processLargeArray` với `await yieldToMain()` mỗi 100 item, `defer` third-party.
  - Đo: **Performance → Record → Main track** red `Task` >50ms, Bottom-up `filter` 300ms; **`onINP(m => console.log(m.attribution.processingDuration))`**; **`web-vitals` RUM** p75/p98 distribution.

- **Lỗi 4: Memory Leak — Detached HTMLElement ×5 sau navigate**
  - Triệu chứng: SPA để 30 phút RAM 150MB → 1.5GB, tab crash `Aw Snap!`.
  - Nguyên nhân: `useEffect` thiếu `return cleanup` cho `setInterval`/`addEventListener`/`WebSocket`/`IntersectionObserver`/`store.subscribe`.
  - Fix: `return () => { clearInterval; removeEventListener; ws.close(); obs.disconnect(); subscription.unsubscribe() }`; dùng `LRU`/`WeakMap` cho cache.
  - Đo: **Memory → Heap Snapshot A/B Comparison** → filter `Detached` → Retainers thấy `Map`/`closure`; **Performance Monitor** `JS heap size` dốc; **Allocation Timeline** record 30s; Console `getEventListeners(window)` count tăng.

- **Lỗi 5: White Screen production — không Sentry, không source map**
  - Triệu chứng: user báo trắng màn hình, Console `Uncaught ChunkLoadError: Loading chunk 23 failed`, chỉ production, local pass.
  - Fix: `RootErrorBoundary` + `Sentry.captureException` + `release` tag + `source-map-explorer`; `Cache-Control: immutable` cho JS/CSS, `no-cache` cho HTML; `ChunkLoadError` auto-reload 1 lần.
  - Đo: **Sentry → filter `release`, `breadcrumbs`, `ChunkLoadError`**; **DevTools Network** `main.abc123.js 404` (CDN hash lệch); `git log --oneline -5` + `git diff HEAD~1` + `git bisect`; **`vitest --coverage` không đo, nhưng `playwright trace viewer -- trace.zip`** replay white screen.

- **Lỗi 6: Lab Good (Lighthouse 95) nhưng field Poor (LCP 3.8s)**
  - Triệu chứng: lab throttled Moto G4 pass, nhưng CrUX LCP 3.8s (30% user low-end 3G).
  - Fix: `preload hero` với `fetchpriority="high"`, reserve `width/height` + `aspect-ratio`, `next/image` `priority`, RUM segmentation theo `deviceMemory`/`effectiveType`.
  - Đo: **`web-vitals` RUM** `sendToAnalytics` + **`PerformanceObserver` `largest-contentful-paint`**; **CrUX API** `https://chromeuxreport.googleapis.com/v1/records:queryRecord`; **Lighthouse CI** `lighthouserc.json` assert `LCP < 2500`; **DevTools Coverage** unused JS 70%.

- **Công cụ đo tổng hợp**:
  - **React Profiler (Flame/Ranked) + `Profiler onRender` + WDYR** — commit duration, hotspot, vì sao re-render.
  - **Chrome Performance (Main track, Long Task) + Performance Monitor** — INP, JS heap size.
  - **Memory (Heap Snapshot, Allocation Timeline, `getEventListeners`)** — Detached HTMLElement, retained closure.
  - **Sentry (error, breadcrumbs, release, source map, replay) + `playwright trace viewer` (`trace: 'on-first-retry'`, `npx playwright show-trace`)** — production triage.
  - **web-vitals (`onLCP/onINP/onCLS`) + `PerformanceObserver` + `msw log` + `vitest --coverage --provider=v8`** — field metrics, network mock, coverage.

## 7. Câu hỏi tự kiểm tra

1. Profiler Flame vs Ranked khác gì, và khi nào cần `why-did-you-render` bổ sung — vì sao `memo(Card)` với `style={{}}` vẫn re-render?
2. Long Task >50ms trong Performance Main track liên hệ INP thế nào, và `useDeferredValue`/`startTransition` cứu INP ra sao khác gì với Heap Snapshot Detached HTMLElement cho memory leak?
3. Sentry (release + source map + breadcrumbs) và `web-vitals` (LCP/INP/CLS p75 + PerformanceObserver) khác vai trò gì trong quy trình Debug Senior 11 bước, và `ChunkLoadError: Loading chunk 23 failed` debug bằng gì?

<details>
<summary>Đáp án 30s</summary>

1. **Flame** = **tree theo thời gian** (parent trên, child dưới, X = time, màu xanh→đỏ theo tốn kém) — thấy **lan truyền** (`AppContext` đổi → 50 Card vàng). **Ranked** = **list sort `actualDuration` giảm dần** (flatten) — tìm **hotspot** nhanh (top 3 tốn 80%) nhưng mất quan hệ cha-con. **WDYR** patch `memo` để **log diff `prevProps vs nextProps`** (`style: {} !== {}` reference khác) + stack. `memo(Card)` với `style={{}}` fail vì **inline object tạo reference mới mỗi render** → shallow-equal `false` → re-render. Fix `useMemo(() => ({ margin: 10 }), [])` + WDYR chỉ bật dev (cost O(n)).
2. **Long Task** là **Task main thread >50ms** (red bar trong Performance Main track) block `input` → `INP = inputDelay + processing (React render) + presentation (paint)`. `items.filter` 300ms trong `onChange` → Long Task 300ms → INP 350ms Poor. **`useDeferredValue(query)` / `startTransition`** tách **urgent** (`setQuery` InputContinuousLane, paint ngay) và **non-urgent** (`filter` TransitionLane, có thể yield/interrupt) → Task 18ms → INP 80ms Good. **Heap Snapshot** khác: đo **memory leak** (RAM tăng dốc, Detached HTMLElement) — `Memory → Comparison` thấy **DOM detached nhưng JS vẫn giữ** do thiếu `useEffect cleanup` (`clearInterval`, `removeEventListener`, `ws.close`, `obs.disconnect`), không liên quan Long Task.
3. **Sentry** trong quy trình **11 bước** ở **Collect logs + Monitoring + JS Error + Regression**: thu **error + stack (source map), breadcrumbs (click/fetch), release (commit SHA), replay**; filter `release` → biết deploy nào spike, `breadcrumbs` → reproduce, `source map` → stack unminified. **`web-vitals` (`onLCP/onINP/onCLS` + `PerformanceObserver` `largest-contentful-paint`/`event`/`layout-shift`)** ở **Monitoring (field p75/INP p98)** → SEO/CrUX, RUM alert khi LCP >2.5s. **`ChunkLoadError: Loading chunk 23 failed`** (white screen sau deploy, 10% user) debug bằng **Sentry filter `ChunkLoadError` + `release`**, **DevTools Network** `main.abc123.js 404` (CDN hash lệch, HTML cache cũ), **`git bisect` + `git diff HEAD~1`**, và fix **atomic deploy** (`immutable` cho JS, `no-cache` cho HTML) + **ErrorBoundary per route** + **auto-reload 1 lần** nếu `message.includes('Loading chunk')`.

</details>

---
*Tham khảo chi tiết: `docs/13-debugging.md` — 10 tình huống (white screen, memory leak, race, performance, render loop, iOS, user segment, contract drift) + `docs/18-production-scenarios.md` scenario 01/02/04. Spec: [React DevTools Profiler](https://react.dev/tools), [why-did-you-render](https://github.com/welldone-software/why-did-you-render), [Chrome Performance — Long Task](https://developer.chrome.com/docs/devtools/performance), [Sentry — Releases & Source Maps](https://docs.sentry.io/platforms/javascript/), [web-vitals](https://github.com/GoogleChrome/web-vitals) + [PerformanceObserver](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver).*
