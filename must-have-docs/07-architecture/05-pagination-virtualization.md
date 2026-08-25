# Pagination & Virtualization — Offset vs Cursor, Infinite Scroll, tanstack-virtual / react-window

> Tags: #Pagination #Offset #Cursor #Infinite-Scroll #Virtualization #TanStack-Virtual #React-Window #useInfiniteQuery | Nguồn: `docs/12-system-design.md` Bài 1-3 | Mức: P0

## 1. Định nghĩa chính xác

**Pagination** là chia dataset lớn thành trang: **Offset** (`?page=2&limit=20` → `OFFSET 20 LIMIT 20`) cho phép **jump tới page N**, nhưng **skip cost tăng** với 100k rows và **dễ lệch** khi data insert/xóa. **Cursor** (`?cursor=eyJpZCI6MTIzfQ&limit=20` → `WHERE id > lastId LIMIT 20`) **ổn định, không skip**, hợp **infinite scroll**, nhưng **không jump** tới page 100.

**Infinite scroll** là load thêm khi scroll gần đáy qua `useInfiniteQuery` + `IntersectionObserver`, thay vì phân trang số.

**Virtualization** là chỉ render **~30-50 item visible** trong viewport + `overscan`, thay vì 10k DOM, bằng **`@tanstack/virtual`** hoặc **`react-window`/`react-virtuoso`**, giữ scroll mượt 60fps và giảm memory.

## 2. Cơ chế hoạt động

### 2.1 Offset vs Cursor

**Offset**:

```
GET /api/products?page=2&limit=20 → SELECT * FROM products OFFSET 20 LIMIT 20
```

- DB phải **scan + skip** 20 rows mỗi page, page 5000 (`OFFSET 100000`) chậm, không dùng index hiệu quả.
- Nếu có insert/xóa giữa chừng: user ở page 1, có 1 item mới insert đầu → item cuối page 1 lặp lại ở page 2 (**duplicate/skip**).

**Cursor**:

```
GET /api/products?cursor=eyJpZCI6MTIzfQ&limit=20 → SELECT * FROM products WHERE id > '123' ORDER BY id LIMIT 20
→ { data: [...], nextCursor: 'eyJpZCI6MTQzfQ' }
```

- Dùng **index** (`id`, `createdAt`) để seek, không skip, nhanh với 100k-1M rows.
- Ổn định khi insert: chỉ lấy sau cursor, không lệch. Nhưng **không cho jump** tới page 100 vì không biết cursor của page 100.

### 2.2 Infinite Scroll

- FE `useInfiniteQuery({ queryKey: ['products', filter], queryFn: ({ pageParam }) => fetch(/products?cursor=${pageParam}) , getNextPageParam: last => last.nextCursor })`
- `fetchNextPage()` khi sentinel `div` vào viewport (`IntersectionObserver`) hoặc scroll gần đáy. `pages: Page[]` chứa các trang đã load.
- Mobile hợp infinite, desktop có thể giữ page number + URL `?page=2`.

### 2.3 Virtualization

- Đo **scroll container** height, **estimateSize** (40px/row), tính `startIndex`/`endIndex` từ `scrollTop`, chỉ render `virtualItems` (+ overscan 10). Dùng `transform: translateY(start)` hoặc `position: absolute`.
- `@tanstack/virtual` headless, linh hoạt; `react-window` cố định, nhanh; `react-virtuoso` auto-size, hợp chat/message.
- Kết hợp **server pagination (cursor, limit 100) + virtualization** cho table millions rows: BE chỉ trả 100 rows/lần, FE virtualize 100 đó, hết thì fetch cursor tiếp.

Sơ đồ:

```
BE: cursor pagination (WHERE id > cursor LIMIT 20) → { rows, nextCursor }
FE: useInfiniteQuery (pageParam = cursor) → pages.flatMap(p => p.rows) → useVirtualizer({ count: flat.length, estimateSize: 40, overscan: 10 }) → chỉ render 30 DOM + sentinel → IntersectionObserver → fetchNextPage
```

## 3. Ví dụ tối thiểu

```ts
// 3.1 Offset — jump được, chậm với lớn
// API: GET /api/products?page=2&limit=20
function useProductsOffset(page: number, limit = 20) {
  return useQuery({
    queryKey: ['products', 'offset', page],
    queryFn: () => fetch(`/api/products?page=${page}&limit=${limit}`).then(r => r.json()),
    keepPreviousData: true, // giữ data cũ khi chuyển page
  });
}
// URL: /products?page=2 — share được, jump tới 100
// Nhược: OFFSET 100000 chậm, duplicate khi insert

// 3.2 Cursor + Infinite Scroll — nhanh, ổn định
import { useInfiniteQuery } from '@tanstack/react-query';

type Page = { rows: Product[]; nextCursor: string | null };

function useProductsCursor(filter: { category?: string }) {
  return useInfiniteQuery<Page>({
    queryKey: ['products', 'cursor', filter],
    queryFn: ({ pageParam }) =>
      fetch(`/api/products?cursor=${pageParam ?? ''}&limit=20&category=${filter.category ?? ''}`).then(r => r.json()),
    initialPageParam: null as string | null,
    getNextPageParam: last => last.nextCursor, // null → hết
    staleTime: 30_000,
  });
}

function ProductInfinite({ filter }: { filter: any }) {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useProductsCursor(filter);
  const rows = data?.pages.flatMap(p => p.rows) ?? [];
  const sentinelRef = React.useRef<HTMLDivElement>(null);

  React.useEffect(() => {
    const el = sentinelRef.current;
    if (!el) return;
    const io = new IntersectionObserver(([e]) => {
      if (e.isIntersecting && hasNextPage && !isFetchingNextPage) fetchNextPage();
    }, { rootMargin: '400px' });
    io.observe(el);
    return () => io.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <>
      {rows.map(p => <ProductCard key={p.id} product={p} />)}
      <div ref={sentinelRef} style={{ height: 1 }} />
      {isFetchingNextPage && <Spinner />}
    </>
  );
}

// 3.3 Virtualization — tanstack/virtual
import { useVirtualizer } from '@tanstack/react-virtual';

function BigTable({ rows }: { rows: Product[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40, // row height ước lượng
    overscan: 10, // render thêm 10 ngoài viewport cho mượt
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto', position: 'relative' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map(v => (
          <div
            key={v.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${v.start}px)`,
              height: 40,
            }}
          >
            <Row row={rows[v.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
function Row({ row }: { row: Product }) {
  return <div>{row.name} — {row.price}</div>;
}

// 3.4 Virtual + Infinite (millions rows) — server cursor + virtual
function VirtualInfiniteTable() {
  const { data, fetchNextPage, hasNextPage } = useProductsCursor({ category: 'all' });
  const flat = data?.pages.flatMap(p => p.rows) ?? [];
  const parentRef = React.useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({ count: flat.length, getScrollElement: () => parentRef.current, estimateSize: () => 40, overscan: 10 });
  const virtualItems = virtualizer.getVirtualItems();

  // fetch thêm khi scroll gần cuối virtual
  React.useEffect(() => {
    const last = virtualItems[virtualItems.length - 1];
    if (last && last.index >= flat.length - 10 && hasNextPage) fetchNextPage();
  }, [virtualItems, flat.length, hasNextPage, fetchNextPage]);

  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualItems.map(v => (
          <div key={v.key} style={{ position: 'absolute', top: 0, transform: `translateY(${v.start}px)` }}>
            <Row row={flat[v.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}

// 3.5 react-window alternative
import { FixedSizeList } from 'react-window';
function WindowList({ rows }: { rows: Product[] }) {
  return (
    <FixedSizeList height={600} itemCount={rows.length} itemSize={40} width="100%">
      {({ index, style }) => <div style={style}><Row row={rows[index]} /></div>}
    </FixedSizeList>
  );
}

// 3.6 URL cho page number (desktop) vs infinite (mobile)
// Desktop: /products?category=shoes&page=2&limit=20 — useSearchParams + offset
// Mobile: infinite cursor — không page, chỉ nextCursor
```

## 4. So sánh / Phân loại

| Tiêu chí | Offset (`?page=&limit=`) | Cursor (`?cursor=&limit=`) |
|----------|--------------------------|----------------------------|
| Query | `OFFSET 20 LIMIT 20` (skip) | `WHERE id > lastId LIMIT 20` (seek, index) |
| Jump tới page N | Có (`?page=100`) | Không (phải duyệt tuần tự) |
| Performance 100k | Chậm (skip cost) | Nhanh (index seek) |
| Ổn định khi insert/xóa | Lệch — duplicate/skip | Ổn định — chỉ lấy sau cursor |
| Share URL | Dễ (`?page=2`) | Khó share giữa (cursor opaque `eyJpZCI...`) |
| Khi dùng | Page nhỏ, cần jump, SEO pagination | Infinite scroll, data lớn, realtime |

| Tiêu chí | Infinite Scroll | Pagination số (numbered) |
|----------|-----------------|--------------------------|
| UX | Mượt mobile, không click page | Rõ vị trí, jump nhanh desktop |
| BE | Cursor | Offset hoặc cursor |
| SEO | Kém (bot không scroll) — cần SSR link | Tốt (link `?page=2`) |
| Khi dùng | Feed, product list mobile | Table, admin, SEO listing |

| Library | Đặc điểm | Khi dùng |
|---------|----------|----------|
| `@tanstack/virtual` | Headless, linh hoạt, estimateSize + overscan | Table, list custom, cần control |
| `react-window` | Cố định `itemSize`, nhanh, ít feature | List đồng đều 40px/row |
| `react-virtuoso` | Auto-size, handle dynamic height, chat | Message list, row cao không đều |

| Khi nào virtualize | Khi nào không |
|--------------------|---------------|
| >100 DOM cùng lúc, 10k message, table millions | <50 item, mỗi item nặng nhưng ít |
| Chart 10k points — dùng Canvas, không virtualize DOM | Cần SEO full DOM (bot đọc hết) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không offset cho 100k SKU / millions rows**: `OFFSET 100000` scan chậm, lệch khi insert. Dùng **cursor** + `useInfiniteQuery`.
- **Không cursor khi cần jump tới page 100**: cursor không cho `?page=100`, chỉ next. Nếu requirement jump, giữ **offset** hoặc **cursor + page index cache** (phức tạp).
- **Không infinite scroll cho SEO listing**: bot không scroll qua `IntersectionObserver`, cần **SSR + link `?page=2`** hoặc **pagination số** để crawl. Infinite chỉ cho app feed/mobile.
- **Không virtualize khi <50 item**: thêm `virtualizer` + `transform` phức tạp, không đáng. Chỉ virtualize khi **>100 items** hoặc **10k+**.
- **Không virtualize mọi thứ**: grid 20 charts → chỉ virtualize chart `visible` qua `IntersectionObserver` + `dynamic import`, không virtualize toàn grid nếu mỗi chart đã Canvas.
- **Không `overscan: 100`**: overscan lớn render nhiều DOM thừa, mất tác dụng virtualize. `overscan: 10` đủ mượt.
- **Không trả 1M rows JSON rồi virtualize**: BE phải **server pagination** (`limit 100`), FE virtualize 100 đó, không fetch 1M rồi virtualize (OOM).

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Offset với 100k → OFFSET 50000 chậm**
  - Triệu chứng: `GET /products?page=5000` 2s, DB CPU cao.
  - Fix: chuyển **cursor** (`WHERE id > cursor LIMIT 20`), index `id`/`createdAt`.
  - Đo: `EXPLAIN ANALYZE SELECT ... OFFSET 50000`, Network timing, `performance.now()`.

- **Lỗi 2: Infinite duplicate/skip khi dùng offset + insert**
  - Triệu chứng: item cuối page 1 lặp lại đầu page 2 sau khi có insert.
  - Fix: **cursor** ổn định, không skip.
  - Đo: log `nextCursor`, test insert giữa chừng.

- **Lỗi 3: Cursor không jump → UX desktop cần ?page=100 không làm được**
  - Triệu chứng: user muốn tới page 100 phải scroll 100 lần.
  - Fix: desktop giữ **offset** hoặc hybrid: offset cho page nhỏ (<100), cursor cho infinite mobile.
  - Đo: requirement check.

- **Lỗi 4: Virtualization không `estimateSize` đúng → scroll giật**
  - Triệu chứng: row cao 80px nhưng `estimateSize: 40` → scrollbar sai, nhảy.
  - Fix: `estimateSize` gần đúng, hoặc `react-virtuoso` auto-measure, hoặc `measureElement` của tanstack/virtual.
  - Đo: scroll test, `virtualizer.getTotalSize()` vs thực tế.

- **Lỗi 5: Không `overscan` hoặc overscan quá lớn**
  - Triệu chứng: `overscan: 0` → trắng khi scroll nhanh; `overscan: 100` → render 200 DOM, mất lợi virtualize.
  - Fix: `overscan: 10` cân bằng.
  - Đo: Profiler → số DOM, FPS, `document.querySelectorAll('[data-index]').length`.

- **Lỗi 6: Infinite `fetchNextPage` không `hasNextPage` guard → loop**
  - Triệu chứng: sentinel trigger liên tục, spam `GET /products?cursor=null`.
  - Fix: `if (hasNextPage && !isFetchingNextPage) fetchNextPage()`, `getNextPageParam: last => last.nextCursor ?? undefined` (undefined = hết).
  - Đo: Network → số request, React Query `hasNextPage`.

- **Lỗi 7: Scroll anchor mất khi load thêm (message list)**
  - Triệu chứng: load older messages làm scroll nhảy lên đầu.
  - Fix: giữ `scrollTop` hoặc dùng `react-virtuoso` `alignToBottom` + `followOutput`, hoặc `useVirtualizer` với `scrollToIndex`.
  - Đo: manual scroll test.

- **Công cụ**:
  - **Network** — `?page=` vs `?cursor=`, timing, duplicate.
  - **React Query DevTools** — `pages`, `hasNextPage`, `isFetchingNextPage`.
  - **Profiler + Performance** — FPS khi scroll, DOM count.
  - **EXPLAIN ANALYZE** — DB skip vs index seek.
  - **`@tanstack/virtual` dev** — `getVirtualItems().length` (chỉ ~30 thay vì 10k).

## 7. Câu hỏi tự kiểm tra

1. Phân biệt Offset vs Cursor — query DB, performance 100k rows, ổn định khi insert, và vì sao cursor không jump tới page 100? Khi nào chọn cái nào?
2. Infinite scroll hoạt động thế nào với `useInfiniteQuery` + `IntersectionObserver`? Vì sao không dùng infinite cho SEO listing và desktop cần jump?
3. Virtualization giải quyết gì và `@tanstack/virtual` vs `react-window` vs `react-virtuoso` khác nhau thế nào? Khi nào không nên virtualize và overscan bao nhiêu là hợp?

<details>
<summary>Đáp án 30s</summary>

1. **Offset**: `OFFSET 20 LIMIT 20` — skip N rows, dùng được `?page=100` để jump, nhưng **skip cost tăng** (OFFSET 100k scan chậm) và **lệch khi insert/xóa** (duplicate/skip). **Cursor**: `WHERE id > lastId LIMIT 20` (cursor opaque `eyJpZCI...`) — **index seek**, nhanh với 100k-1M, **ổn định** khi insert (chỉ lấy sau cursor), nhưng **không jump** vì không biết cursor page 100. Chọn **offset** khi **page nhỏ, cần jump, SEO pagination**; chọn **cursor** khi **infinite scroll, data lớn, realtime** (như `docs/12` 100k SKU).
2. **Infinite**: `useInfiniteQuery({ queryKey: ['products', filter], queryFn: ({ pageParam }) => fetch(?cursor=pageParam), getNextPageParam: last => last.nextCursor })` → `pages` chứa các trang, `fetchNextPage()` khi sentinel `div` vào viewport qua **`IntersectionObserver`** (`rootMargin: 400px`) hoặc scroll gần đáy. **Không dùng cho SEO** vì bot không scroll→ không thấy page 2, cần **SSR + link `?page=2`**; desktop cần jump thì **offset + URL `?page=`** thay vì infinite.
3. **Virtualization** chỉ render **~30-50 item visible + overscan 10** thay vì 10k DOM, giữ 60fps và giảm memory. **`@tanstack/virtual`**: headless, `estimateSize` + `overscan`, linh hoạt; **`react-window`**: `FixedSizeList` nhanh cho row đều; **`react-virtuoso`**: auto-size, hợp chat/message. **Không virtualize** khi **<50 item** (thừa complexity) hoặc cần **SEO full DOM**. **`overscan: 10`** cân bằng — 0 thì trắng khi scroll nhanh, 100 thì render thừa mất lợi.

</details>

---
*Tham khảo chi tiết: `docs/12-system-design.md` — Bài 1 (Pagination), Bài 2 (Virtualized message), Bài 3 (Table virtualized). Spec: [TanStack Virtual](https://tanstack.com/virtual), [React Window](https://react-window.vercel.app/), [useInfiniteQuery](https://tanstack.com/query/latest/docs/react/guides/infinite-queries).*

