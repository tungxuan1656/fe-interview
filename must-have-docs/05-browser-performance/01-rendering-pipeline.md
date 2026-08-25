# Critical Rendering Path & Rendering Pipeline — HTML → CSSOM → Render Tree → Layout → Paint → Composite

> Tags: #rendering-pipeline #critical-rendering-path #reflow #repaint #composite #layers #will-change | Nguồn: `docs/06-browser-web-platform.md` câu 105-109 + `docs/05-performance.md` câu 86,102 | Mức: P0

## 1. Định nghĩa chính xác

**Critical Rendering Path (CRP)** là chuỗi bước browser phải hoàn thành để biến HTML, CSS, JavaScript thành pixel trên màn hình lần đầu: `Parse HTML → DOM` + `Parse CSS → CSSOM` → `Render Tree` → `Layout (Reflow)` → `Paint` → `Composite`. Mỗi tài nguyên render-blocking (CSS, sync JS) kéo dài CRP và trì hoãn First Paint.

**Reflow (Layout)** tính geometry (vị trí, kích thước) của mỗi node; **Repaint (Paint)** raster hóa pixel (color, border, shadow) vào bitmap của layer; **Composite** gộp các bitmap layer bằng compositor thread/GPU và áp `transform`/`opacity` — bước duy nhất chạy ngoài main thread.

## 2. Cơ chế hoạt động

### 2.1 Pipeline 6 bước (Chromium/Blink)

1. **Parse HTML → DOM**: byte → character → token → node → DOM tree. `preload scanner` (pre-parser) quét ahead để fetch `<img>`, `<link>`, `<script>` sớm dù main parser đang block.
   - `<script>` không `defer`/`async` → **parser-blocking**: pause DOM construction, fetch + execute rồi mới tiếp.
   - `<script defer>` → fetch song song, execute sau khi parse xong, giữ thứ tự.
   - `<script async>` → fetch song song, execute ngay khi xong, không giữ thứ tự, vẫn block parse lúc execute.
2. **Parse CSS → CSSOM**: CSS là **render-blocking**: browser không paint cho tới khi CSSOM xong (tránh FOUC). Selector phức tạp (`*`, `:nth-child`, deep descendant) làm tăng thời gian `Recalculate Style`.
3. **Render Tree**: hợp nhất DOM + CSSOM, loại bỏ node không visual (`display: none`, `<meta>`, `<script>`). Mỗi node chứa `computedStyle`. `visibility: hidden` vẫn trong Render Tree (chiếm layout), `display: none` thì không.
4. **Layout (Reflow)**: duyệt Render Tree tính `x, y, width, height` cho mỗi node theo viewport, flex/grid, containing block. Là O(n), lan truyền — đổi width của parent có thể reflow cả subtree và sibling.
5. **Paint**: raster hóa từng layer thành bitmap (text, color, background, border, shadow) theo thứ tự stacking context. Record vào `PaintLayer` / `GraphicsLayer`.
6. **Composite**: compositor thread gộp các layer (đã raster) với `transform`/`opacity`, submit tới GPU. Đây là lý do `transform`/`opacity` rẻ — chỉ composite, không layout/paint.

```
HTML bytes → Tokenizer → DOM
CSS bytes  → Tokenizer → CSSOM ─┐
                                ├→ Render Tree → Layout → Paint → Composite → GPU → Pixel
JS ────────→ Parse/Execute ──────┘
                    ↑ block nếu sync
```

### 2.2 Reflow / Repaint / Composite trigger

- **Reflow**: `width`, `height`, `margin`, `padding`, `display`, `position`, `top`/`left`, `font-size`, `border-width`, `flex`, `grid`, và **đọc** `offsetHeight`/`offsetWidth`/`getBoundingClientRect()` sau khi layout đã invalidate (force reflow).
- **Repaint**: `color`, `background`, `visibility`, `box-shadow`, `outline`, `background-image` — không đổi geometry, chỉ vẽ lại pixel trong cùng box.
- **Composite-only**: `transform`, `opacity`. Nếu element đã được promote lên composite layer riêng, đổi hai thuộc tính này chỉ chạy compositor, bỏ qua layout/paint.

Thứ tự chi phí: `Reflow → Repaint → Composite`. Đổi `width` = reflow+repaint+composite; đổi `color` = repaint+composite; đổi `transform` = composite-only.

### 2.3 Composite Layers — khi nào tạo layer mới

Browser tự promote lên `GraphicsLayer` (bitmap riêng trong GPU memory) khi:

- `transform: translateZ(0)` / `translate3d(0,0,0)` / `will-change: transform, opacity`
- `position: fixed` (luôn composited — giữ cố định khi scroll, không repaint mỗi frame)
- `video`, `canvas`, `iframe`
- `animation`/`transition` trên `transform`/`opacity`
- `filter`, `backdrop-filter`, `perspective`

`will-change` là **hint** — báo trước để browser tạo layer sớm, tránh promote đột ngột gây jank khi animation bắt đầu.

### 2.4 Layout Thrashing

Khi xen kẽ **đọc** (force layout) và **ghi** (invalidate layout) trong loop, mỗi lần đọc buộc browser flush pending style/layout synchronously:

```
invalidate (ghi style) → đọc offsetHeight → force reflow (sync) → invalidate → đọc → force reflow ...
```

Batch: đọc hết trước, ghi hết sau — chỉ 1 reflow.

### 2.5 `position: fixed` bị phá bởi `transform` ancestor

`transform`, `filter`, `perspective`, `contain: paint`, `will-change: transform` trên ancestor tạo **containing block** mới và **stacking context** mới. `position: fixed` khi đó fixed theo ancestor đó thay vì viewport — bug kinh điển. `will-change: transform` cũng tạo containing block y hệt `transform`.

## 3. Ví dụ tối thiểu

```js
// 3.1 Layout thrashing vs batch

// ❌ Thrashing — N lần reflow
function thrashing(els) {
  for (let i = 0; i < els.length; i++) {
    const h = els[i].offsetHeight; // đọc → force reflow
    els[i].style.height = h + 10 + 'px'; // ghi → invalidate
  }
}

// ✅ Batch read → batch write — 1 reflow
function batched(els) {
  const heights = els.map(el => el.offsetHeight); // đọc hết (1 reflow nếu cần)
  els.forEach((el, i) => (el.style.height = heights[i] + 10 + 'px')); // ghi hết
}

// ✅ Dùng requestAnimationFrame để gộp với frame
let ticking = false;
window.addEventListener('scroll', () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      const tops = els.map(el => el.getBoundingClientRect().top);
      els.forEach((el, i) => (el.style.transform = `translateY(${tops[i]}px)`)); // composite-only
      ticking = false;
    });
    ticking = true;
  }
});
```

```css
/* 3.2 transform (composite-only) vs top/left (reflow) */

/* ❌ Reflow mỗi frame */
.box-top {
  position: absolute;
  left: 0;
  transition: left 0.3s;
}
.box-top.move { left: 100px; /* Layout → Paint → Composite */ }

/* ✅ Composite-only — 60fps ngay cả khi main thread bận */
.box-transform {
  transition: transform 0.3s;
  will-change: transform; /* tạo layer trước */
}
.box-transform.move { transform: translateX(100px); /* Composite only */ }

/* Sau animation nên remove để tiết kiệm memory */
```

```html
<!-- 3.3 Fixed bị phá bởi transform ancestor -->
<!-- ❌ fixed kẹt trong parent -->
<div style="transform: translateZ(0)">
  <div style="position: fixed; top: 0">Bị kẹt theo parent, không theo viewport</div>
</div>
<!-- ❌ will-change cũng phá tương tự -->
<div style="will-change: transform">
  <div style="position: fixed; top: 0">Cũng bị kẹt</div>
</div>
<!-- ✅ Đặt fixed trực tiếp dưới body, không có transform ancestor -->
<div style="position: fixed; top: 0">OK — theo viewport</div>
```

```js
// 3.4 will-change quản lý vòng đời
const el = document.querySelector('.animate');
el.style.willChange = 'transform'; // promote trước khi animate
el.addEventListener('transitionend', () => {
  el.style.willChange = 'auto'; // giải phóng layer/GPU memory
});
```

```css
/* 3.5 Containment & content-visibility — cô lập reflow */
.container {
  contain: layout paint; /* reflow/paint trong container không lan ra ngoài */
}
.offscreen {
  content-visibility: auto;
  contain-intrinsic-size: 500px; /* reserve size tránh CLS khi chưa render */
}
```

## 4. So sánh / Phân loại

| Thao tác | Trigger ví dụ | Pipeline | Thread | Chi phí |
|----------|---------------|----------|--------|---------|
| **Reflow (Layout)** | `width`, `height`, `margin`, `display`, `top/left`, `font-size`, đọc `offsetHeight` | Layout → Paint → Composite | Main | Cao nhất — O(n), lan truyền |
| **Repaint (Paint)** | `color`, `background`, `visibility`, `box-shadow` | Paint → Composite | Main | Trung bình |
| **Composite** | `transform`, `opacity` (đã có layer) | Composite only | Compositor (GPU) | Rẻ nhất — 60fps |

| Thuộc tính animation | Có reflow? | Có repaint? | Có composite? | Ghi chú |
|----------------------|-----------|-------------|---------------|---------|
| `width`/`height`/`top`/`left` | ✅ | ✅ | ✅ | Đẩy layout, giật khi main busy |
| `color`/`background` | ❌ | ✅ | ✅ | Không đẩy layout nhưng vẫn paint |
| `transform`/`opacity` | ❌ | ❌ (nếu đã có layer) | ✅ | Mượt, không ảnh hưởng layout sibling |

| Tạo layer | Cách | Memory | Khi dùng |
|-----------|------|--------|----------|
| `will-change: transform` | Hint, tạo layer trước animate | ~ width×height×4 bytes (1920×1080 ≈ 8MB) | Element sắp animate, scroll parallax |
| `translateZ(0)` / `translate3d` | Hack ép layer | Tương tự | Legacy, không khuyến nghị thay will-change |
| `position: fixed` | Tự động | 1 layer | Browser tự làm |
| Không promote | Mặc định | 0 | Element tĩnh |

| `contain` | Tác dụng |
|-----------|----------|
| `contain: layout` | Reflow trong container không lan ra ngoài |
| `contain: paint` | Paint không tràn ra ngoài, tạo containing block |
| `contain: size` | Size không phụ thuộc con, có thể skip layout con khi offscreen |
| `content-visibility: auto` | Bỏ layout/paint khi ngoài viewport, tự động contain |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không `will-change` tràn lan**: mỗi layer là bitmap GPU (vài MB). 100 layer → hàng trăm MB, composite chậm hơn, có thể crash tab trên mobile. Chỉ promote element thực sự animate/scroll; luôn remove sau `transitionend`/`animationend`.
- **Không dùng `transform` khi cần đẩy layout thật**: `transform: translateY(100px)` chỉ visual, không đẩy sibling. Accordion/dropdown cần đẩy content thì phải dùng `height`/`grid-template-rows` và chấp nhận reflow, hoặc đo `scrollHeight` bằng JS rồi animate `transform` kết hợp kỹ thuật FLIP.
- **Không lạm dụng `content-visibility: auto`** khi cần `find-in-page` (Ctrl+F) hoặc `scrollTo` chính xác — browser skip layout offscreen nên tìm kiếm có thể miss. Luôn kèm `contain-intrinsic-size` để tránh CLS.
- **Không đặt `transform`/`will-change` trên ancestor của `fixed`**: phá `position: fixed` (kẹt theo ancestor). Đặt `fixed` ở cấp `body` hoặc đảm bảo ancestor không có `transform`/`filter`/`will-change: transform`.
- **Không đọc layout trong scroll handler không batch**: `getBoundingClientRect` trong mỗi `scroll` event gây force reflow 60 lần/giây. Batch + `requestAnimationFrame` hoặc thay bằng `IntersectionObserver`.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Layout thrashing làm FPS tụt 10-15**
  - Triệu chứng: Performance tab toàn purple `Layout`/`Recalculate Style`, FPS đỏ.
  - Nguyên nhân: `offsetHeight` xen kẽ `style.height` trong loop hoặc scroll handler.
  - Fix: batch read/write; dùng `transform` thay `top/left`; `requestAnimationFrame`.
  - Đo: Chrome DevTools → Performance → Record khi scroll → xem `Layout` duration, `Event Log → Recalculate Style`. Rendering tab → `Layout Shift Regions` highlight.

- **Lỗi 2: `will-change` memory phình, composite chậm**
  - Triệu chứng: Layers panel có hàng chục layer cam, GPU memory tăng, tab crash trên mobile.
  - Fix: chỉ `will-change` cho 2-3 element animating, remove sau khi xong.
  - Đo: DevTools → More Tools → Layers → xem `Compositing Reasons`, đếm layer. Rendering → `Layer borders` (viền cam). `chrome://gpu` hoặc Performance → Memory.

- **Lỗi 3: `position: fixed` kẹt khi scroll**
  - Triệu chứng: header/table header fixed không bám viewport, cuộn theo parent.
  - Nguyên nhân: ancestor có `transform`/`filter`/`will-change: transform`/`perspective`/`contain: paint`.
  - Fix: move `fixed` ra ngoài ancestor đó, hoặc bỏ `transform` trên ancestor (dùng `contain` thay nếu chỉ cần isolation).
  - Đo: DevTools → Layers → chọn fixed element → `Compositing Reasons` không còn `position: fixed viewport-constrained`; Elements → Computed → `position`.

- **Lỗi 4: Animation `top/left` jank dù đã `rAF`**
  - Triệu chứng: Performance tab mỗi frame vẫn có `Layout`, FPS < 60 khi main thread bận.
  - Fix: chuyển sang `transform: translateX/Y`, đảm bảo element đã có layer (`will-change`).
  - Đo: Performance → với `top/left` thấy `Layout` purple mỗi frame; với `transform` chỉ thấy `Composite` hoặc không gì (compositor thread).

- **Lỗi 5: `content-visibility` làm Ctrl+F miss và scrollTo sai**
  - Triệu chứng: tìm text không thấy, `element.scrollIntoView()` lệch.
  - Fix: thêm `contain-intrinsic-size` ước lượng đúng; không dùng cho content cần search.
  - Đo: thử Ctrl+F, check Performance → `Layout` skip khi offscreen.

- **Công cụ**:
  - **Performance tab**: `Parse HTML`, `Recalculate Style`, `Layout`, `Paint`, `Composite` timeline; `FPS` graph; `Long task` yellow.
  - **Rendering tab**: `Paint flashing` (green), `Layout Shift Regions` (blue), `Layer borders` (orange), `FPS meter`.
  - **Layers panel**: xem bitmap size, compositing reasons, memory per layer.
  - **Coverage**: phát hiện CSS/JS không dùng làm tăng CSSOM/parse time.

## 7. Câu hỏi tự kiểm tra

1. Vì sao đổi `transform: translateX(100px)` chỉ chạy Composite còn đổi `left: 100px` lại trigger Reflow → Repaint → Composite, và điều này liên quan thế nào tới compositor thread?
2. Layout thrashing xảy ra khi nào? Vì sao `const h = el.offsetHeight` sau khi vừa `el.style.height = '100px'` lại force reflow đồng bộ?
3. `will-change: transform` giúp animation mượt nhưng lạm dụng gây gì? Vì sao `will-change: transform` trên ancestor cũng phá `position: fixed` của con cháu y hệt `transform`?

<details>
<summary>Đáp án 30s</summary>

1. `left` là layout property — browser phải tính lại geometry (reflow) rồi vẽ lại (repaint) rồi composite. `transform` là composite property — nếu element đã có composite layer, GPU chỉ cần áp matrix mới khi composite, không cần reflow/repaint. Compositor thread chạy riêng với main thread nên dù main đang bận JS, animation `transform` vẫn 60fps.

2. Thrashing là xen kẽ đọc layout (offsetHeight, getBoundingClientRect) và ghi style trong loop. Ghi `style.height` chỉ invalidate layout (đánh dấu dirty), chưa tính lại. Đọc `offsetHeight` yêu cầu giá trị chính xác → browser phải flush pending invalidation và force reflow đồng bộ. Batch đọc hết trước, ghi hết sau → chỉ 1 reflow.

3. Mỗi `will-change: transform` tạo một GraphicsLayer bitmap (width×height×4 bytes, 1080p ≈ 8MB). Lạm dụng 100 element → hàng trăm MB GPU memory, composite chậm. `will-change: transform` như `transform` đều tạo containing block và stacking context mới theo spec, nên `position: fixed` bên trong sẽ fixed theo ancestor đó thay vì viewport. Fix: chỉ hint cho element sắp animate và remove sau `transitionend`.

</details>

---
*Tham khảo chi tiết: `docs/06-browser-web-platform.md` — Câu 105, 106, 107, 108, 109. Spec: [CSS Triggers](https://csstriggers.com/), [Chrome Rendering Performance](https://developer.chrome.com/docs/chromium/renderingng-architecture).*
