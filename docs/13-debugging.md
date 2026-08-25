# 13. Debugging Senior - Quy Trình & 10 Tình Huống Thực Chiến

> Debugging ở level Senior không phải là `console.log` mò mẫm. Là quy trình có hệ thống: **Reproduce → Collect logs → Monitoring → Browser/Device → Network → JS Error → Regression → Fix → Test → Deploy → Monitor**. Chương này đi qua framework trắng màn hình production và 10 tình huống hay gặp nhất, mỗi tình huống theo cấu trúc Triệu chứng → Checklist → Tool → Fix → Phòng ngừa.

## Mục lục

- [Quy trình Debugging Production chuẩn Senior](#quy-trình-debugging-production-chuẩn-senior)
- [Tình huống 1: Trắng màn hình (White Screen) Production](#tình-huống-1-trắng-màn-hình-white-screen-production)
- [Tình huống 2: Memory Leak - Tab ngốn RAM, crash sau 30 phút](#tình-huống-2-memory-leak---tab-ngốn-ram-crash-sau-30-phút)
- [Tình huống 3: API lúc được lúc không (Flaky API)](#tình-huống-3-api-lúc-được-lúc-không-flaky-api)
- [Tình huống 4: Page load 8s - Performance thảm họa](#tình-huống-4-page-load-8s---performance-thảm-họa)
- [Tình huống 5: Component render 50 lần / vòng](#tình-huống-5-component-render-50-lần--vòng)
- [Tình huống 6: Mobile Safari crash / White screen chỉ trên iOS](#tình-huống-6-mobile-safari-crash--white-screen-chỉ-trên-ios)
- [Tình huống 7: Production chỉ lỗi với 1 nhóm user](#tình-huống-7-production-chỉ-lỗi-với-1-nhóm-user)
- [Tình huống 8: API trả sai format / Contract drift](#tình-huống-8-api-trả-sai-format--contract-drift)
- [Tình huống 9: UI Flickering / Layout Shift / Search sai thứ tự / Double-click tạo 2 orders](#tình-huống-9-ui-flickering--layout-shift--search-sai-thứ-tự--double-click-tạo-2-orders)
- [Tình huống 10: Tổng hợp Anti-pattern & Phòng ngừa hệ thống](#tình-huống-10-tổng-hợp-anti-pattern--phòng-ngừa-hệ-thống)

---

## Quy trình Debugging Production chuẩn Senior

Senior không nhảy vào fix ngay. Quy trình 11 bước:

```
Reproduce → Collect logs → Monitoring → Browser/Device → Network → JS Error → Regression → Fix → Test → Deploy → Monitor
```

| Bước | Làm gì | Tool |
|---|---|---|
| **1. Reproduce** | Tái hiện local/staging với đúng URL, user, device, data | Playwright replay, Sentry replay |
| **2. Collect logs** | Gom console, network, server log, release version | Sentry, Datadog, CloudWatch |
| **3. Monitoring** | Check spike: error rate, p95 latency, deploy timeline | Grafana, Sentry Release Health |
| **4. Browser/Device** | Có phải chỉ 1 browser/OS? Check compat | BrowserStack, caniuse, User-Agent |
| **5. Network** | Waterfall, failed request, CORS, 4xx/5xx, payload | DevTools Network, Charles, Wireshark |
| **6. JS Error** | Stack trace, source map, error boundary | Sentry, DevTools Sources, source-map-explorer |
| **7. Regression** | `git bisect`, diff deploy gần nhất, feature flag | `git bisect`, `git diff HEAD~1`, LaunchDarkly |
| **8. Fix** | Minimal fix, không refactor kèm | - |
| **9. Test** | Unit + E2E cho case lỗi, thêm regression test | Vitest, Playwright |
| **10. Deploy** | Canary / feature flag off, rollback plan | Vercel, Argo Rollout |
| **11. Monitor** | Theo dõi 24h sau fix, alert nếu tái diễn | Sentry, Web Vitals RUM |

> Nguyên tắc vàng: **Không đoán — đo.** Mỗi giả thuyết phải có bằng chứng từ log/metric trước khi fix.

---

### Tình huống 1: Trắng màn hình (White Screen) Production

**Triệu chứng:** User báo trang trắng tinh, không spinner, không error UI. Chỉ production, local chạy ngon. Có khi chỉ 10% user bị.

**Checklist debug:**

1. Mở DevTools → Console có `Uncaught SyntaxError` / `ChunkLoadError`? → bundle deploy lệch hash
2. Network → có request nào `404` / `500`? `main.abc123.js` 404 → CDN cache cũ trỏ hash cũ
3. Có phải sau deploy? → check `Cache-Control`, asset `immutable` nhưng HTML cache cũ
4. Sentry → xem `error`, `release`, `breadcrumbs`, có `ChunkLoadError: Loading chunk 23 failed`?
5. Thử hard reload, incognito, khác browser → cache issue?
6. Check `ErrorBoundary` có bắt không hay lỗi ở top-level (trước React mount)?

**Tool:** Sentry + Release, Chrome DevTools (Console, Network, Application), `source-map-explorer`, `git log --oneline -5`

**Fix:**

```typescript
// 1. ErrorBoundary top-level
class RootErrorBoundary extends React.Component<{ children: React.ReactNode }, { hasError: boolean }> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // Gửi Sentry với release tag
    Sentry.captureException(error, { extra: { info } });
  }
  render() {
    if (this.state.hasError) return <div>Có lỗi xảy ra. <button onClick={() => location.reload()}>Reload</button></div>;
    return this.props.children;
  }
}

// 2. Bắt ChunkLoadError và auto-reload 1 lần
window.addEventListener('error', (e) => {
  if (e.message.includes('Loading chunk') || e.message.includes('ChunkLoadError')) {
    if (!sessionStorage.getItem('chunk-reload')) {
      sessionStorage.setItem('chunk-reload', '1');
      location.reload();
    }
  }
});

// 3. Next.js: đảm bảo HTML không cache quá lâu
// next.config.js headers: Cache-Control: no-cache cho HTML
```

**Phòng ngừa:**

- Asset hash + `Cache-Control: immutable` cho JS/CSS, `no-cache` cho HTML
- Sentry `release` gắn với commit SHA, source map upload
- Deploy với **atomic**: HTML mới phải trỏ hash mới, không serve HTML cũ + JS mới lệch
- Thêm `ErrorBoundary` cho từng route (`error.tsx` trong App Router)

---

### Tình huống 2: Memory Leak - Tab ngốn RAM, crash sau 30 phút

**Triệu chứng:** SPA để lâu RAM tăng từ 150MB → 1.5GB, tab đơ, `Aw, Snap!`. Thường ở dashboard, chat, page có polling/WS.

**Checklist debug:**

1. DevTools → Performance → Memory checkbox → record 30s, xem JS Heap có tăng dốc không giảm sau GC?
2. Memory → Take Heap Snapshot → tìm **Detached DOM**, so sánh snapshot trước/sau navigate
3. Check `useEffect` có thiếu cleanup? `addEventListener`, `setInterval`, `setTimeout`, `WebSocket`, `IntersectionObserver`, `MutationObserver`
4. Có `store.subscribe` / `EventBus.on` / `RxJS subscribe` không unsubscribe?
5. Có closure giữ DOM lớn (`ref.current` trong handler không remove)?
6. Có `Map` cache không giới hạn, không `WeakMap`?

**Tool:** Chrome Memory tab (Heap Snapshot, Allocation Timeline), Performance Monitor, `why-did-you-render` không liên quan leak nhưng giúp thấy retained

**Fix:**

```typescript
// ❌ Leak
useEffect(() => {
  const id = setInterval(poll, 5000);
  window.addEventListener('scroll', onScroll);
  const ws = new WebSocket(url);
  // thiếu cleanup
}, []);

// ✅ Đủ cleanup
useEffect(() => {
  const id = setInterval(poll, 5000);
  const onScroll = () => {};
  window.addEventListener('scroll', onScroll);
  const ws = new WebSocket(url);
  const obs = new IntersectionObserver(cb);
  obs.observe(ref.current!);

  return () => {
    clearInterval(id);
    window.removeEventListener('scroll', onScroll);
    ws.close();
    obs.disconnect();
  };
}, []);

// ✅ Cache có giới hạn - LRU
import { LRUCache } from 'lru-cache';
const cache = new LRUCache<string, Data>({ max: 100, ttl: 60_000 });

// ✅ WeakMap khi key là object
const meta = new WeakMap<object, string>();

// Debug helper
// DevTools Console: getEventListeners(window)
// Memory -> Detached elements
```

**Phòng ngừa:**

- ESLint rule `react-hooks/exhaustive-deps` + custom rule check missing cleanup
- Code review checklist: mọi `addEventListener`/`setInterval`/`subscribe` phải có `remove`/`clear`/`unsubscribe` trong `return`
- Monitor memory RUM: `performance.memory.usedJSHeapSize` gửi về analytics, alert nếu > 500MB

---

### Tình huống 3: API lúc được lúc không (Flaky API)

**Triệu chứng:** `fetch /api/products` lúc 200 lúc 500, lúc CORS, lúc timeout. User refresh thì được. Khó reproduce.

**Checklist debug:**

1. Network → status thực tế: `500`? `429` Rate limit? `CORS`? `502` Gateway?
2. Check BE log / APM (Datadog) → có spike latency, DB deadlock, OOM?
3. Có race condition: 2 request cùng lúc, cái sau abort cái trước?
4. Có thiếu `AbortController` khi component unmount → setState trên unmounted?
5. Check retry logic: có retry vô hạn DDoS BE?
6. Check CDN/WAF block? User-agent? Region?

**Tool:** DevTools Network (waterfall, timing, response), Postman/curl replay, Sentry network breadcrumbs, BE log, `msw` mock để isolate FE/BE

**Fix:**

```typescript
// 1. Abort + dedup + retry có giới hạn
const controller = new AbortController();
useEffect(() => {
  const ctrl = new AbortController();
  fetch('/api/products', { signal: ctrl.signal })
    .then(r => r.json())
    .then(setData)
    .catch(e => {
      if (e.name === 'AbortError') return; // bỏ qua
      Sentry.captureException(e);
    });
  return () => ctrl.abort();
}, []);

// 2. TanStack Query: retry, staleTime, dedup tự động
useQuery({
  queryKey: ['products'],
  queryFn: () => fetch('/api/products').then(r => {
    if (!r.ok) throw new Error(String(r.status));
    return r.json();
  }),
  retry: (count, err) => count < 2 && err.message !== '403',
  retryDelay: attempt => Math.min(1000 * 2 ** attempt, 5000),
});

// 3. Idempotency key cho POST
fetch('/api/orders', {
  method: 'POST',
  headers: { 'Idempotency-Key': crypto.randomUUID() },
  body: JSON.stringify(order),
});
```

**Phòng ngừa:**

- Mọi `fetch` trong `useEffect` phải có `AbortController`
- Dùng TanStack Query / SWR thay `fetch` thủ công → dedup + cache + retry chuẩn
- BE: rate limit header `Retry-After`, FE respect
- Contract test (Pact) để phát hiện BE đổi response

---

### Tình huống 4: Page load 8s - Performance thảm họa

**Triệu chứng:** Lighthouse Performance 35, LCP 6s, TTI 8s. User 3G bỏ trang.

**Checklist debug:**

1. Lighthouse → xem LCP element, TBT, CLS, bundle size
2. Network → bundle bao nhiêu? `main.js` 1.8MB? Có 30 request waterfall?
3. Coverage (DevTools → Coverage) → bao nhiêu JS không dùng (unused 70%)?
4. Performance tab → long task ở đâu? React render bao lâu?
5. Có ảnh 4K không nén? Font block? Third-party?
6. Có SSR hay CSR toàn bộ?

**Tool:** Lighthouse, WebPageTest, `source-map-explorer`, `vite-bundle-visualizer`, Chrome Coverage, Performance tab

**Fix:**

```typescript
// 1. Code splitting + dynamic import
const HeavyChart = dynamic(() => import('./HeavyChart'), { ssr: false, loading: () => <Skeleton /> });

// 2. Image optimize
import Image from 'next/image';
<Image src="/hero.jpg" width={1200} height={600} priority sizes="100vw" alt="" />

// 3. Preload LCP, preconnect CDN
// <link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />
// <link rel="preconnect" href="https://cdn.example.com" />

// 4. Thay lib nặng: moment 200kb → date-fns, lodash → es-toolkit
import { format } from 'date-fns';

// 5. Bundle analyzer
// next.config.js: experimental: { optimizePackageImports: ['lodash-es'] }

// 6. Inline critical CSS, defer non-critical
```

**Phòng ngừa:**

- Performance budget CI: `bundlesize < 200kb`, Lighthouse CI `LCP < 2.5s`
- RUM `web-vitals` gửi về Grafana, alert khi p75 vượt
- `next/image`, `next/font` bắt buộc cho ảnh/font

---

### Tình huống 5: Component render 50 lần / vòng

**Triệu chứng:** Gõ 1 ký tự input → toàn page re-render, Profiler thấy 50 commits, INP 600ms.

**Checklist debug:**

1. React DevTools Profiler → record, xem `Why did this render?` (props/state/context đổi?)
2. Có `Context` bọc toàn app mà value đổi liên tục?
3. Có `inline object/function` làm prop (`style={{}}`, `onClick={() => {}}`) → memo vô tác dụng?
4. Có `useEffect` setState loop?
5. Có `key={index}` hoặc `key={Math.random()}` gây remount?

**Tool:** React DevTools Profiler, `why-did-you-render`, `useWhyDidYouUpdate` hook

**Fix:**

```typescript
// ❌ Context gây re-render toàn cây
const AppContext = createContext({ user, theme, setUser }); // object mới mỗi render

// ✅ Tách context + memo
const UserContext = createContext<User>(null);
const ThemeContext = createContext<Theme>(null);

// ❌ Inline prop
<Button style={{ margin: 10 }} onClick={() => doIt()} />

// ✅ Memo + stable ref
const style = useMemo(() => ({ margin: 10 }), []);
const onClick = useCallback(() => doIt(), []);
<Button style={style} onClick={onClick} />

// ✅ Defer non-urgent
const [query, setQuery] = useState('');
const deferredQuery = useDeferredValue(query);
const filtered = useMemo(() => items.filter(i => i.name.includes(deferredQuery)), [deferredQuery]);

// ✅ React.memo + selector
const CartCount = memo(function CartCount() {
  const count = useStore(s => s.cart.length); // chỉ re-render khi count đổi
  return <span>{count}</span>;
});
```

**Phòng ngừa:**

- Quy tắc: context chỉ chứa state ít đổi hoặc tách nhỏ
- ESLint `react/no-unstable-nested-components`
- Profiler trong CI (storybook + interaction test)

---

### Tình huống 6: Mobile Safari crash / White screen chỉ trên iOS

**Triệu chứng:** Desktop Chrome ngon, iPhone Safari trắng màn hình hoặc crash khi scroll. iOS 15-16.

**Checklist debug:**

1. Có dùng API không support iOS? `ResizeObserver` loop, `100vh` bug, `backdrop-filter`, `Intl`?
2. Console Safari (Mac → Develop → iPhone) → có `SyntaxError` do `?.`/`??` không transpile?
3. Có `position: fixed` + `input focus` zoom?
4. Có `WebSocket` / `IndexedDB` quota?
5. Có `100vh` gây jump khi address bar ẩn/hiện?
6. Check `browserslist` + `esbuild` target có include `safari 15`?

**Tool:** Safari Web Inspector, BrowserStack (real iOS), `caniuse.com`, `es-check`

**Fix:**

```typescript
// 1. Browserslist
// package.json: "browserslist": ["last 2 versions", "iOS >= 15", "Safari >= 15"]

// 2. Fix 100vh
// CSS: height: 100dvh; /* dynamic viewport */
// fallback: height: 100vh; height: 100dvh;

// 3. Fix input zoom (font-size <16px auto zoom)
input, select, textarea { font-size: 16px; }

// 4. Transpile đúng
// vite.config.ts: build.target = 'es2018' nếu cần iOS 15
// next.config: không dùng swc exclude

// 5. Test với BrowserStack + Sentry tag os.name
Sentry.setTag('os', navigator.userAgent);
```

**Phòng ngừa:**

- CI chạy Playwright + BrowserStack trên iOS
- `browserslist` + `eslint-plugin-compat`
- Sentry filter theo `os.name === 'iOS'` để phát hiện sớm

---

### Tình huống 7: Production chỉ lỗi với 1 nhóm user

**Triệu chứng:** 95% user bình thường, 5% báo lỗi — thường là user có data đặc biệt, role, locale, A/B flag.

**Checklist debug:**

1. Sentry → filter `user.id`, `release`, `environment`, `tags` (role, locale, featureFlag)
2. Có feature flag / A/B test chỉ bật cho nhóm đó? → tắt flag test
3. Có data edge: tên có emoji, số dư âm, timezone, `null` field?
4. Có locale/RTL, i18n key thiếu?
5. Có permission/role trả API khác?
6. Có extension/ad-block chặn request?

**Tool:** Sentry (filter, distribution), LaunchDarkly, LogRocket replay, `msw` với data nhóm lỗi

**Fix:**

```typescript
// 1. Defensive: schema validation
import { z } from 'zod';
const UserSchema = z.object({
  name: z.string().min(1).default('Unknown'),
  balance: z.number().nullable().transform(v => v ?? 0),
  avatar: z.string().url().optional().or(z.literal('')),
});
const user = UserSchema.parse(raw); // fail fast, không render NaN

// 2. Feature flag safe default
const showNewCheckout = useFlag('new-checkout', false); // default false

// 3. Sentry context
Sentry.setUser({ id: user.id, segment: user.plan });
Sentry.setTag('locale', navigator.language);

// 4. Fallback UI
{user.name ? <span>{user.name}</span> : <span>—</span>}
```

**Phòng ngừa:**

- Zod/Yup validate mọi API response, không tin BE
- Feature flag rollout 5% → 50% → 100% + kill switch
- Seed data diverse trong E2E (emoji, null, RTL)

---

### Tình huống 8: API trả sai format / Contract drift

**Triệu chứng:** FE expect `{ id: number }` nhưng BE trả `{ id: "123" }` hoặc `{ data: null }`, app crash `undefined is not a function`.

**Checklist debug:**

1. Network → compare response thực tế vs TypeScript type / OpenAPI spec
2. Có deploy BE không báo FE? Check BE changelog, OpenAPI diff
3. Có `any` / `as` che lỗi type?
4. Có mock khác production?

**Tool:** OpenAPI diff, `zod` runtime validation, `msw` + contract test, `orval` generate type từ OpenAPI

**Fix:**

```typescript
// 1. Runtime validation với zod
const ProductSchema = z.object({
  id: z.coerce.number(), // "123" → 123
  price: z.number(),
  createdAt: z.coerce.date(),
});
type Product = z.infer<typeof ProductSchema>;

async function fetchProduct(id: string): Promise<Product> {
  const raw = await fetch(`/api/products/${id}`).then(r => r.json());
  return ProductSchema.parse(raw); // throw nếu sai
}

// 2. Generate type từ OpenAPI
// npx orval --input ./openapi.yaml --output ./src/api

// 3. MSW contract test
// msw handler dùng chung schema với FE
```

**Phòng ngừa:**

- FE generate type từ OpenAPI, không viết tay
- CI chạy `zod` parse với fixture thực tế từ staging
- Pact consumer-driven contract test

---

### Tình huống 9: UI Flickering / Layout Shift / Search sai thứ tự / Double-click tạo 2 orders

> Gộp 4 bug UI kinh điển — chung gốc là **state/race/async** không kiểm soát.

#### 9a. UI Flickering (nhấp nháy)

**Triệu chứng:** Modal/tooltip/content nhấp nháy khi hover, skeleton flash rồi biến, theme flash trắng trước khi dark.

**Checklist:** Có `useEffect` set state sau mount gây double paint? Có `isLoading` toggle nhanh? Có CSS `transition` trên `display`?

**Fix:**

```typescript
// ❌ Flash: fetch xong mới render, không skeleton stable
{isLoading ? <Spinner /> : <Content />}

// ✅ Skeleton có min-height, không layout shift
<div style={{ minHeight: 300 }}>
  {isLoading ? <Skeleton rows={5} /> : <Content />}
</div>

// ❌ Theme flash: đọc localStorage trong useEffect
useEffect(() => setTheme(localStorage.getItem('theme')), []);

// ✅ Đọc theme trước paint (script inline trong <head>)
<script dangerouslySetInnerHTML={{ __html: `document.documentElement.dataset.theme=localStorage.getItem('theme')||'light'` }} />

// ❌ Hover flicker: tooltip unmount khi mouse leave 1px
// ✅ Dùng delay + pointer-events
```

#### 9b. Search sai thứ tự (Race Condition)

**Triệu chứng:** Gõ "a" → "ab" → "abc", kết quả hiện của "a" (cũ) đè "abc" (mới) vì response về muộn.

**Fix:**

```typescript
// ✅ Abort previous + chỉ nhận latest
function useSearch(query: string) {
  const [result, setResult] = useState([]);
  useEffect(() => {
    const ctrl = new AbortController();
    const id = setTimeout(async () => {
      try {
        const res = await fetch(`/api/search?q=${query}`, { signal: ctrl.signal });
        setResult(await res.json());
      } catch (e) { if ((e as Error).name !== 'AbortError') throw e; }
    }, 300);
    return () => { clearTimeout(id); ctrl.abort(); };
  }, [query]);
  return result;
}

// ✅ Hoặc dùng TanStack Query (tự abort + dedup)
useQuery({ queryKey: ['search', query], queryFn: ({ signal }) => fetch(`/api/search?q=${query}`, { signal }).then(r => r.json()), enabled: query.length > 1 });
```

#### 9c. Double-click tạo 2 orders (Idempotency)

**Triệu chứng:** User double-click "Đặt hàng" → 2 request → 2 đơn, trừ tiền 2 lần.

**Fix:**

```typescript
// 1. Disable + loading
const [loading, setLoading] = useState(false);
async function onSubmit() {
  if (loading) return;
  setLoading(true);
  try { await createOrder(); } finally { setLoading(false); }
}
<Button disabled={loading} onClick={onSubmit}>{loading ? 'Đang xử lý...' : 'Đặt hàng'}</Button>

// 2. Idempotency-Key (chuẩn)
const key = useRef(crypto.randomUUID());
await fetch('/api/orders', { method: 'POST', headers: { 'Idempotency-Key': key.current }, body: JSON.stringify(order) });

// 3. Debounce/throttle ở handler
import { useDebouncedCallback } from 'use-debounce';
const debouncedSubmit = useDebouncedCallback(onSubmit, 1000, { leading: true, trailing: false });
```

**Phòng ngừa chung:**

- Mọi async đều có `AbortController` hoặc Query
- Mọi mutation quan trọng đều có `Idempotency-Key` + `disabled`
- Mọi search đều debounce + abort

---

### Tình huống 10: Tổng hợp Anti-pattern & Phòng ngừa hệ thống

| Anti-pattern | Hậu quả | Fix hệ thống |
|---|---|---|
| `console.log` là tool chính | Mù trong production | Sentry + RUM + LogRocket |
| Không có ErrorBoundary | White screen toàn app | Boundary per route + global |
| `any` / `as` bừa bãi | Contract drift không phát hiện | Zod + OpenAPI generate |
| Không abort fetch | Race, memory leak | TanStack Query chuẩn |
| Không idempotency | Double order | Idempotency-Key + disable |
| Bundle không budget | Load 8s | Lighthouse CI + bundlesize |
| Không test với data bẩn | Chỉ lỗi nhóm user | Seed diverse + contract test |
| Deploy không canary | Sập 100% user | Feature flag + canary 5% |

**Checklist phòng ngừa tổng:**

- [ ] Sentry + source map + release tag
- [ ] ErrorBoundary mọi route
- [ ] Zod validate mọi API response
- [ ] TanStack Query cho mọi server state
- [ ] Abort + debounce cho search
- [ ] Idempotency cho mutation quan trọng
- [ ] Lighthouse CI + bundlesize trong PR
- [ ] BrowserStack iOS test
- [ ] Feature flag kill switch
