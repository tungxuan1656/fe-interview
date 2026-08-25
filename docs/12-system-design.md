# 12. System Design Frontend - 3 Bài Lớn

> 3 bài system design thực chiến cho Senior Frontend (Bài 1-3) - E-commerce, Chat Realtime, Dashboard Big Data. Mỗi bài đi từ yêu cầu → kiến trúc → trade-off → câu hỏi follow-up. Senior không vẽ UI mà thiết kế **state, API, caching, performance và failure mode**.

## Mục lục

- [Bài 1: E-commerce - Product Listing, Detail, Search, Cart, Checkout, Payment, Order](#bài-1-e-commerce---product-listing-detail-search-cart-checkout-payment-order)
- [Bài 2: Chat Realtime - Conversation, Message, Typing, Online, Notification](#bài-2-chat-realtime---conversation-message-typing-online-notification)
- [Bài 3: Dashboard Big Data - 20 Charts, Filters, Realtime, Export CSV, Millions Records](#bài-3-dashboard-big-data---20-charts-filters-realtime-export-csv-millions-records)

---

### Bài 1: E-commerce - Product Listing, Detail, Search, Filter, Cart, Checkout, Payment, Order

**Bối cảnh:** Xây SPA/Next.js cho shop 100k SKU, 50k DAU, cần listing, detail, search/filter, cart (guest + auth), checkout 3 bước, payment (COD/MoMo/VNPay), order tracking, SEO, analytics. Yêu cầu trả lời về **state, API, caching, pagination, error, auth, performance, SEO, analytics, testing**.

#### 1. Kiến trúc tổng quan (text diagram)

```
[ Next.js App Router (SSR/ISR) ]
  ├─ app/(shop)/products/page.tsx (ISR 60s) ── TanStack Query ──┐
  ├─ app/(shop)/products/[id]/page.tsx (ISR + generateStaticParams) │
  ├─ app/(shop)/search/page.tsx (SSR, searchParams) ──────────────┤
  ├─ app/(checkout)/cart, checkout (SSR, auth) ───────────────────┤
  └─ components/* (Server cho SEO, Client lá cho interactivity)    │
                                                                  ▼
[ BFF / Route Handlers ] ── [ API Gateway ] ── [ Microservices ]
  /api/products (cache, proxy)        ├─ Product Service (ES + DB)
  /api/search (debounce, abort)       ├─ Cart Service (Redis + DB)
  /api/cart (merge guest→auth)        ├─ Order Service
  /api/checkout (idempotent)          ├─ Payment Service (webhook)
                                      └─ Search Service (Elasticsearch)
                                                                  ▼
[ Client State ]                [ Server State (TanStack Query) ]
 Zustand: cart (persist),        ['products', filter] (stale 60s)
          wishlist, UI            ['product', id] (placeholder)
 URL: filter, sort, page, q      ['cart'] (no-store, auth)
 Local: form draft                ['orders'] (poll 30s)
```

#### 2. Thiết kế chi tiết

**State:**

| State | Loại | Ở đâu | Vì sao |
|---|---|---|---|
| Product list/detail | Server | Query `['products', filter]` | Cache, stale, dedup |
| Filter/sort/page/q | URL | `useSearchParams` | Share link, back button |
| Cart | Hybrid | Zustand + persist + `POST /api/cart/merge` khi login | Guest local, auth sync |
| Checkout form | Client local | `react-hook-form` + Zustand draft | Chưa submit |
| User/auth | Server | Query `['user']` + httpOnly cookie | Nhiều nơi cần |
| Wishlist | Hybrid | Zustand + sync | Tương tự cart |

Filter lưu ở **URL** (`?brand=nike&price=0-500&sort=price_asc&page=2`), draft slider ở `useState` local, bấm Apply mới đẩy lên URL → Query key đổi → refetch.

**API:**

```
GET  /api/products?category=shoes&brand=nike&price=0-500&sort=price_asc&page=2&limit=20
GET  /api/products/:id
GET  /api/search?q=giày&limit=10 (debounce 300ms, AbortController)
POST /api/cart { productId, qty } (idempotent key)
POST /api/cart/merge (guest cart → user cart khi login)
POST /api/checkout { cartId, address, paymentMethod, idempotencyKey }
GET  /api/orders?status=pending
POST /api/payment/webhook (VNPay/MoMo callback)
```

Dùng **BFF** để gom product + inventory + price, tránh FE gọi 3 service.

**Caching:**

- **TanStack Query**: `staleTime 60s`, `gcTime 5m` cho products; `staleTime 0` cho cart/orders.
- **Next.js Data Cache**: `fetch(..., { next: { revalidate: 60, tags: ['products'] } })` cho listing/detail (ISR); `revalidateTag('products')` khi admin tạo product.
- **CDN**: SSG cho marketing, ISR cho product.
- **Optimistic**: add to cart update Zustand ngay, rollback nếu fail.

```typescript
// Cart optimistic
const qc = useQueryClient();
const addMutation = useMutation({
  mutationFn: (item: Item) => fetch('/api/cart', { method: 'POST', body: JSON.stringify(item) }),
  onMutate: async (item) => {
    await qc.cancelQueries({ queryKey: ['cart'] });
    const prev = qc.getQueryData(['cart']);
    qc.setQueryData(['cart'], (old: Cart) => ({ ...old, items: [...old.items, item] }));
    return { prev };
  },
  onError: (_e, _item, ctx) => qc.setQueryData(['cart'], ctx.prev),
  onSettled: () => qc.invalidateQueries({ queryKey: ['cart'] }),
});
```

**Pagination:** **Offset** cho page nhỏ (`?page=2&limit=20`), **Cursor** cho infinite scroll (`?cursor=eyJpZCI...&limit=20`) tránh skip cost với 100k SKU. Dùng `useInfiniteQuery` cho mobile infinite, `useQuery` + page number cho desktop.

```typescript
// Cursor pagination
useInfiniteQuery({
  queryKey: ['products', filter],
  queryFn: ({ pageParam }) => fetch(`/api/products?cursor=${pageParam}&limit=20`).then(r=>r.json()),
  initialPageParam: null,
  getNextPageParam: last => last.nextCursor,
});
```

**Error & Auth:**

- Error boundary: `error.tsx` cho route, `ErrorFallback` cho Query, retry 2 cho GET, không retry cho POST.
- Auth: httpOnly cookie `token`, Middleware redirect `/checkout` nếu chưa login, Server Component đọc `cookies()` để fetch cart.
- Guest cart: `localStorage` + `deviceId`, khi login gọi `/merge`.

**Performance:**

- Listing: `next/image` + `priority` cho 4 ảnh đầu, virtualized nếu 100+ item (`@tanstack/virtual`), skeleton.
- Detail: ISR + `generateStaticParams` cho top 10k product, còn lại `dynamicParams: true`.
- Bundle: `dynamic(() => import('./HeavyFilter'))`, `optimizePackageImports`.

**SEO:** `generateMetadata` cho product detail (title, OG image), `sitemap.ts` (100k URL chia chunk), `robots.ts`, JSON-LD `Product`, `Breadcrumb`. Listing dùng SSR/ISR để bot thấy HTML.

**Analytics:** `dataLayer` + `gtag` (`next/script afterInteractive`), track `view_item`, `add_to_cart`, `begin_checkout`, `purchase` (GA4). Dùng `navigator.sendBeacon` cho `beforeunload`.

**Testing:** Unit cho `calcTotal`, Integration (RTL+MSW) cho filter/cart, E2E (Playwright) cho `login → search → add → checkout → payment mock`.

#### 3. Trade-off

| Quyết định | Lợi | Hại |
|---|---|---|
| URL làm source filter | Share, back | URL dài, phải sync |
| Zustand + Query thay Redux | Gọn, đúng loại state | 2 lib |
| Cursor pagination | Nhanh với 100k | Không jump tới page 100 |
| ISR 60s | Nhanh + tươi | Stale 60s |
| BFF | Gom logic, cache | Thêm hop |

#### 4. Câu hỏi follow-up

- Cart merge khi login xử lý conflict thế nào (qty cộng hay ghi đè)?
- Payment VNPay redirect → webhook idempotent key thiết kế ra sao để tránh double order?
- Khi 100k SKU, Elasticsearch mapping và debounce search thế nào để không DDoS?
- Làm sao handle out-of-stock optimistic (inventory check ở checkout)?
- SEO cho 100k product: `generateStaticParams` full hay chỉ top? Sitemap 50k limit?

---

### Bài 2: Chat Realtime - Conversation, Message, Typing, Online, Notification

**Bối cảnh:** Xây chat như Messenger: 1-1 và group, conversation list, message (text/image/file), typing, online/offline, read receipt, notification, offline queue, 10k concurrent.

#### 1. Kiến trúc tổng quan

```
[ Next.js / SPA + TanStack Query + Zustand ]
  ├─ ConversationList (Query ['conversations'] + WS event)
  ├─ MessageList (Virtualized + useInfiniteQuery cursor)
  ├─ Composer (optimistic + offline queue)
  └─ Presence (online/typing via WS)

[ Realtime Layer ]
  Client --WebSocket (Socket.IO / ws)--> [ WS Gateway (Node/Go) ] --Redis Pub/Sub--> [ WS Gateway x N ]
     |  fallback: SSE / Long-polling               |                |
     |  auth: httpOnly cookie + JWT                └─ [ Presence Service ] (Redis)
     └─ REST: GET /conversations, GET /messages?cursor=, POST /messages

[ Backend ]
  Message Service (DB: Postgres + partition by conversation_id)
  Conversation Service (participants, lastMessage)
  Presence Service (Redis: user:{id}:online, typing:{convId})
  Notification Service (FCM/APNS + WS)
  Storage Service (S3 cho file/image, presigned URL)
```

#### 2. Thiết kế chi tiết

**State:**

```typescript
// Zustand cho ephemeral
type ChatUI = {
  activeConvId: string | null;
  typing: Record<string, string[]>; // convId -> userIds đang gõ
  online: Record<string, boolean>;
  setTyping: (convId: string, users: string[]) => void;
};
// TanStack Query cho server
// ['conversations'] -> list
// ['messages', convId] -> infinite messages (cursor)
// Query cache là source, WS event dùng setQueryData để update
```

**API & Realtime:**

```
REST:
GET  /api/conversations
GET  /api/conversations/:id/messages?cursor=&limit=30 (cursor = messageId, order DESC)
POST /api/conversations/:id/messages { text, tempId, replyTo }
POST /api/upload/presign (lấy S3 URL)
PATCH /api/messages/:id/read

WS Events (Socket.IO):
client -> server: join { convId }, leave, typing { convId, isTyping }, message { convId, text, tempId }
server -> client: message:new { convId, message }, typing:update { convId, users }, presence { userId, online }, read { messageId, userId }
```

**WS vs Polling:** Chọn **WebSocket** (hoặc Socket.IO) vì **2 chiều, low latency (<100ms), tiết kiệm**. Polling (3s) tốn request, delay, không có typing. SSE chỉ 1 chiều, không gửi typing. Trade-off: WS cần sticky session, reconnect logic, heart-beat.

**Ordering & Idempotency:**

- Mỗi message có `id` (UUID server), `tempId` (UUID client), `createdAt` (server time), `seq` (per-conversation increment).
- Client gửi `tempId`, server trả `id` + `seq`, client replace optimistic. Dùng `seq` để order, không dùng client time (clock skew).
- Idempotent: nếu client retry với cùng `tempId`, server dedup (Redis `tempId` 5 phút).

```typescript
// Optimistic UI
const sendMutation = useMutation({
  mutationFn: (msg: { text: string; tempId: string }) => fetch(`/api/conversations/${convId}/messages`, { method: 'POST', body: JSON.stringify(msg) }),
  onMutate: async ({ text, tempId }) => {
    const optimistic = { id: tempId, tempId, text, status: 'sending', createdAt: new Date().toISOString() };
    qc.setQueryData(['messages', convId], (old: InfiniteData) => ({
      ...old, pages: [[optimistic, ...old.pages[0]], ...old.pages.slice(1)],
    }));
  },
  onSuccess: (saved, { tempId }) => {
    qc.setQueryData(['messages', convId], (old) => replaceByTempId(old, tempId, saved));
  },
  onError: (_e, { tempId }) => markFailed(tempId),
});
```

**Reconnect & Offline:**

- `socket.io` tự reconnect với `exponential backoff` (1s, 2s, 4s), `ack` callback.
- Heartbeat `ping/pong` 25s, nếu miss 2 pong → reconnect.
- Offline queue: khi `navigator.onLine === false` hoặc WS disconnected, push message vào `IndexedDB` queue (Zustand persist), khi online thì flush theo thứ tự.
- Missed messages: khi reconnect, client gửi `lastSeq` của conv, server trả `GET /messages?afterSeq=lastSeq`.

```typescript
// Reconnect sync
socket.on('connect', () => {
  socket.emit('sync', { convId, lastSeq: getLastSeq(convId) }, (missed: Message[]) => {
    missed.forEach(m => qc.setQueryData(['messages', convId], append(m)));
  });
});
```

**Pagination & Virtualized:**

- Message list **cursor pagination** (`limit 30`, order `createdAt DESC`), `useInfiniteQuery` + `fetchNextPage` khi scroll lên (load older).
- **Virtualized** với `@tanstack/virtual` hoặc `react-virtuoso` vì 10k message không thể DOM hết (chỉ render 30-50 visible).
- Scroll anchor: giữ vị trí khi load thêm, auto-scroll xuống khi có message mới nếu đang ở bottom.

**Cache:**

- Query cache `staleTime: Infinity` cho messages (chỉ WS update), `gcTime: 30m`.
- `setQueryData` khi WS `message:new` để không fetch.
- Conversation `lastMessage` update qua WS, không poll.

**Error & Edge:**

- Typing: debounce 300ms, timeout 3s tự clear.
- Read receipt: `IntersectionObserver` mark read khi message visible.
- File: presigned S3, upload progress, chunk nếu >10MB.

#### 3. Trade-off

| Quyết định | Lợi | Hại |
|---|---|---|
| WebSocket | Realtime, 2 chiều | Cần gateway, sticky, scale |
| Optimistic + tempId | UX mượt | Phức tạp dedup, rollback |
| Cursor + Virtualized | Nhanh, ít DOM | Logic scroll khó |
| IndexedDB offline queue | Gửi khi offline | Phải sync, conflict |
| Redis Pub/Sub | Scale WS | Thêm infra |

#### 4. Câu hỏi follow-up

- Clock skew: vì sao không dùng client `createdAt` để order? `seq` per-conversation sinh thế nào (DB sequence vs Redis INCR)?
- Group chat 500 người: typing broadcast có storm không? Throttle thế nào?
- WS scale 10k concurrent: sticky session vs Redis adapter? Làm sao deploy zero-downtime?
- End-to-end encryption: FE phải làm gì (Signal protocol)?
- Search message: Elasticsearch hay Postgres full-text? Phân trang search khác load history thế nào?

---

### Bài 3: Dashboard Big Data - 20 Charts, Filters, Realtime, Export CSV, Millions Records

**Bối cảnh:** Dashboard cho ops: 20 charts (line/bar/pie/map), filters (date, region, category), realtime ticker, export CSV 1M rows, dataset millions records, cần <2s load, không lag.

#### 1. Kiến trúc tổng quan

```
[ Next.js (SSR shell) + TanStack Query + Zustand (filter) ]
  ├─ FilterBar (URL + Zustand draft, debounce 300ms)
  ├─ Grid: 20 x ChartCard (Suspense + dynamic import)
  │    ├─ Chart (Recharts / ECharts / Vega) - Canvas cho big
  │    └─ Data: useQuery(['chart', id, filter], { staleTime: 30s })
  ├─ Table (Virtualized + Server pagination)
  ├─ Realtime Ticker (WS / SSE, throttled 1s)
  └─ ExportWorker (Web Worker + Stream)

[ BFF / Aggregation Layer ]
  GET /api/charts/:id?from=...&to=...&region=... (cache Redis 30s, pre-aggregate)
  GET /api/table?cursor=&limit=100&sort= (cursor pagination, server sort)
  GET /api/export?filter= (stream CSV, presigned S3)
  WS /realtime (throttle 1s, delta only)

[ Backend ]
  OLAP: ClickHouse / BigQuery (pre-aggregate rollup hourly/daily)
  OLTP: Postgres (raw, partition by date)
  Cache: Redis (query result, 30s-5m)
  Queue: Kafka (ingest) -> Flink (aggregate) -> OLAP
```

#### 2. Thiết kế chi tiết

**Data Fetching:**

- Không fetch 20 charts song song không kiểm soát (thundering herd). Dùng **batch** hoặc **BFF gom** `POST /api/charts/batch { ids, filter }` trả 20 dataset 1 lần, hoặc `Promise.all` với `concurrency 6`.
- Mỗi chart có `queryKey: ['chart', chartId, filterHash]`, `staleTime: 30s`, `gcTime: 5m`, `retry: 1`.
- Filter thay đổi → debounce 300ms → update URL → invalidate chỉ chart bị ảnh hưởng (dùng `filterHash`).
- **Pre-aggregate**: BE không trả millions rows, mà trả **rollup** (hourly 1k points), FE không aggregate raw.

```typescript
// Zustand filter
type Filter = { from: string; to: string; region: string; category: string };
const useFilter = create<Filter & { set: (p: Partial<Filter>) => void }>(set => ({
  from: '2024-01-01', to: '2024-01-31', region: 'all', category: 'all',
  set: p => set(p),
}));

// Fetch 20 charts với concurrency
const chartIds = ['revenue', 'orders', 'conversion', /* ...20 */];
const filter = useFilter();
const queries = useQueries({
  queries: chartIds.map(id => ({
    queryKey: ['chart', id, filter],
    queryFn: () => fetch(`/api/charts/${id}?${new URLSearchParams(filter as any)}`).then(r=>r.json()),
    staleTime: 30_000,
  })),
});

// Hoặc batch BFF
// POST /api/charts/batch { ids: chartIds, filter } -> { revenue: [...], orders: [...] }
```

**Cache:**

- **Query cache** 30s cho chart, **Redis** trên BFF 30s-5m (key `chart:${id}:${hash(filter)}`).
- **HTTP Cache**: `Cache-Control: public, s-maxage=30` cho CDN nếu filter phổ biến.
- Realtime ticker **không cache**, WS delta.

**Virtualization:**

- **Table** millions rows → **server pagination** (cursor, `limit 100`) + **virtualized** (`@tanstack/virtual`, chỉ render 20 rows visible) + **server sort/filter**.
- **Grid 20 charts** → virtualize grid nếu viewport nhỏ: chỉ render chart visible (IntersectionObserver), lazy `dynamic(() => import('./Chart'))`.

```typescript
// Table virtualized
import { useVirtualizer } from '@tanstack/react-virtual';
function BigTable({ rows }: { rows: Row[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({ count: rows.length, getScrollElement: () => parentRef.current, estimateSize: () => 40, overscan: 10 });
  return <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>{virtualizer.getVirtualItems().map(v => <Row key={v.key} row={rows[v.index]} style={{ transform: `translateY(${v.start}px)` }} />)}</div>;
}
// Pagination server
// GET /api/table?cursor=ey...&limit=100&sort=amount_desc -> { rows, nextCursor }
```

**Web Worker:**

- **Export CSV 1M rows**: không làm trên main thread (lag). Dùng **Web Worker** để **stream fetch + format CSV** hoặc **yêu cầu BE tạo file S3 async** rồi poll.

```typescript
// workers/csvWorker.ts
self.onmessage = async (e: MessageEvent<{ filter: Filter }>) => {
  const { filter } = e.data;
  // Cách 1: stream từ BE
  const res = await fetch(`/api/export?${new URLSearchParams(filter as any)}`);
  const reader = res.body!.getReader();
  let csv = 'id,amount,date\n';
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    csv += new TextDecoder().decode(value);
    self.postMessage({ progress: csv.length });
  }
  self.postMessage({ done: true, csv });
};
// Main
const worker = new Worker(new URL('./workers/csvWorker.ts', import.meta.url));
worker.postMessage({ filter });
worker.onmessage = e => { if (e.data.done) download(e.data.csv); else setProgress(e.data.progress); };

// Cách 2 tốt hơn cho 1M rows: BE async
// POST /api/export/jobs { filter } -> { jobId }
// GET /api/export/jobs/:id -> { status, url } (poll 2s) -> download S3 presigned URL
```

**Rendering Perf:**

- **Canvas** cho line/bar >5k points (ECharts, uPlot) thay SVG (Recharts SVG lag với 10k).
- **Downsample**: BE hoặc FE dùng **LTTB** (Largest Triangle Three Buckets) giảm 100k points → 1k không mất shape.
- **Memo**: `React.memo` cho ChartCard, `useMemo` cho transform, tránh re-render 20 chart khi 1 filter đổi (dùng `filterHash`).
- **Skeleton + Suspense**: mỗi chart `Suspense` riêng, không block nhau.

```typescript
// Chart với ECharts (canvas)
import dynamic from 'next/dynamic';
const ReactECharts = dynamic(() => import('echarts-for-react'), { ssr: false });
function RevenueChart({ data }: { data: Point[] }) {
  const option = useMemo(() => ({
    xAxis: { type: 'category', data: data.map(d => d.date) },
    yAxis: { type: 'value' },
    series: [{ type: 'line', data: data.map(d => d.value), sampling: 'lttb' as const }],
  }), [data]);
  return <ReactECharts option={option} style={{ height: 300 }} opts={{ renderer: 'canvas' }} />;
}
```

**Pagination & Realtime:**

- Table: **cursor** (`?cursor=lastId&limit=100`), không offset (skip 1M chậm).
- Realtime: WS `ticker` 1s batch, FE throttle `requestAnimationFrame`, chỉ update chart ticker, không refetch 20 charts.

**Export:** Với 1M rows, **không trả JSON** rồi CSV trên FE (OOM). BE stream CSV trực tiếp hoặc tạo file S3 + presigned URL. FE chỉ download.

#### 3. Trade-off

| Quyết định | Lợi | Hại |
|---|---|---|
| BFF batch 20 charts | Ít request, gom auth | BFF phức tạp |
| OLAP pre-aggregate | <200ms | Cần pipeline, delay 1h |
| Canvas vs SVG | 10k points mượt | Không CSS dễ, a11y kém |
| Web Worker export | Không lag main | Thêm worker, IE? |
| Server pagination + virtualized | Millions mượt | Không search client |
| WS realtime throttle 1s | Đỡ spam | Delay 1s |

#### 4. Câu hỏi follow-up

- 20 charts fetch 1 lúc: làm sao tránh waterfall và thundering herd (batch, priority, concurrency limit)?
- Downsample LTTB: FE hay BE làm? Khi user zoom thì fetch lại detail thế nào?
- Export 1M rows: streaming CSV trên FE có OOM không? Vì sao nên BE tạo file S3 async?
- Realtime + filter: khi filter đổi, WS có phải resubscribe không? Làm sao tránh race giữa REST và WS?
- Memory: 20 charts mỗi 10k points = 200k DOM/canvas, làm sao GC khi tab ẩn (Page Visibility)?

---
