# Image & Font Optimization — srcset/sizes, AVIF/WebP, fetchpriority, preload, font-display:swap, FOIT/FOUT

> Tags: #image-optimization #srcset #sizes #avif #webp #fetchpriority #preload #font-optimization #font-display #foit #fout #subset | Nguồn: `docs/05-performance.md` câu 95-97 | Mức: P0

## 1. Định nghĩa chính xác

**Image Optimization** là giảm bytes ảnh mà vẫn giữ chất lượng cảm nhận, đồng thời tải **đúng size** cho viewport và **đúng priority** cho LCP. Công cụ: **format hiện đại** (AVIF nhỏ nhất > WebP > JPEG/PNG), **responsive** (`srcset` + `sizes`), **loading hint** (`fetchpriority="high"` cho LCP, `loading="lazy"` cho below-fold), **preload hero** (`<link rel="preload" as="image">`), và CDN resize.

**Font Optimization** là giảm FOIT/FOUT và CLS do font swap. Cơ chế: **subset** (chỉ glyph cần thiết, ví dụ `latin`), **preload** font LCP (`<link rel="preload" as="font" crossorigin>`), **`font-display`** (`swap`/`optional`/`fallback`), và **fallback metric** (`size-adjust`, `ascent-override`, `descent-override`) để fallback font (Arial) có metric gần font chính (Inter), giảm CLS. `next/font`/`next/image` tự động hóa phần lớn.

**FOIT (Flash of Invisible Text)**: text vô hình tới khi font tải xong (`font-display: block`). **FOUT (Flash of Unstyled Text)**: hiện fallback ngay, swap khi font xong (`swap`). `optional` cho phép không swap nếu font chậm (>100ms).

## 2. Cơ chế hoạt động

### 2.1 `srcset` + `sizes`

- `srcset="photo-400.jpg 400w, photo-800.jpg 800w, photo-1200.jpg 1200w"` khai báo các ứng cử viên theo **width descriptor** (`w`).
- `sizes="(max-width: 768px) 100vw, 50vw"` nói browser **slot size** của ảnh trong layout: mobile 100% viewport, desktop 50%. Browser tính `slotWidth = sizes → px`, sau đó chọn candidate nhỏ nhất mà `w >= slotWidth × devicePixelRatio`. Không có `sizes` → mặc định `100vw` (mobile tải ảnh desktop thừa).
- `<picture>` cho **art direction** / format fallback: AVIF → WebP → JPEG.

### 2.2 Format

- **AVIF**: nén tốt nhất (30-50% nhỏ hơn JPEG), encode chậm, hỗ trợ Chrome/FF/Safari 16+. Cần fallback WebP/JPEG.
- **WebP**: 25-35% nhỏ hơn JPEG, hỗ trợ rộng. Dùng `sharp`, `squoosh`, CDN (Cloudinary, Imgix) hoặc `next/image` tự convert.
- **SVG** cho vector/icon.

### 2.3 Loading & Priority

- **LCP hero**: `fetchpriority="high"` + `<link rel="preload" as="image" imagesrcset ...>` để browser phát hiện sớm (trước khi parse tới `<img>`), priority cao, không `loading="lazy"`, `decoding="async"` để không block parse.
- **Below-fold**: `loading="lazy"` (native, dựa trên IntersectionObserver nội bộ) + `decoding="async"`.
- **`preload` vs `fetchpriority`**: `preload` tạo request sớm (discovery), `fetchpriority` điều chỉnh priority của request đã có. Dùng cả hai cho hero: preload để fetch sớm, high để thắng tranh bandwidth.
- **`width`/`height` + `aspect-ratio`**: reserve layout box trước khi ảnh load, tránh CLS (impactFraction). `width`/`height` là intrinsic, CSS `aspect-ratio` giữ ratio khi responsive `width:100%`.

### 2.4 Font — subset, preload, display

- **Subset**: `unicode-range: U+0000-00FF` (latin), `&subset=latin` (Google Fonts), variable font (`font-weight: 400 700`) gom nhiều weight vào 1 file.
- **Preload**: `<link rel="preload" as="font" href="/inter-latin.woff2" type="font/woff2" crossorigin>` — bắt buộc `crossorigin` vì font fetch là CORS (anonymous). Không preload → font phát hiện muộn sau CSSOM, FOIT lâu.
- **`font-display`**:
  - `block`: FOIT ~3s rồi swap — CLS thấp nhưng invisible lâu.
  - `swap`: FOUT ngay (fallback) rồi swap — visible ngay, CLS nếu metric lệch.
  - `fallback`: FOIT 100ms rồi fallback, swap trong 3s — cân bằng.
  - `optional`: như fallback nhưng nếu font chưa xong trong 100ms thì **không swap** — giảm CLS tối đa, nhưng có thể không hiện font custom trên 3G.
- **Fallback metric**: `size-adjust: 107%`, `ascent-override: 90%`, `descent-override: 22%` chỉnh fallback font (Arial) để `x-height`/`line-height` gần Inter, swap không giật.

### 2.5 Resource hints

- `preconnect` (`https://fonts.gstatic.com`, `https://cdn.example.com`) mở TCP+TLS sớm, tiết kiệm 100-300ms.
- `preload` cho 1-2 critical (hero, font LCP). Quá nhiều preload tranh bandwidth.

## 3. Ví dụ tối thiểu

```html
<!-- 3.1 Responsive + AVIF/WebP fallback + preload hero -->
<head>
  <link rel="preconnect" href="https://cdn.example.com" crossorigin />
  <!-- Preload LCP — imagesrcset/imagesizes cho responsive preload -->
  <link
    rel="preload"
    as="image"
    href="/hero-800.webp"
    imagesrcset="/hero-800.webp 800w, /hero-1200.webp 1200w"
    imagesizes="(max-width: 768px) 100vw, 50vw"
    fetchpriority="high"
  />
  <link rel="preload" as="font" href="/fonts/inter-latin.woff2" type="font/woff2" crossorigin />
</head>
<body>
  <!-- Picture — AVIF best, WebP fallback, JPEG last -->
  <picture>
    <source srcset="/hero.avif" type="image/avif" />
    <source srcset="/hero.webp" type="image/webp" />
    <img
      src="/hero.jpg"
      width="1200" height="600"
      fetchpriority="high"
      decoding="async"
      alt="Hero"
    />
  </picture>

  <!-- srcset/sizes — browser tự chọn -->
  <img
    src="/photo-800.jpg"
    srcset="/photo-400.jpg 400w, /photo-800.jpg 800w, /photo-1200.jpg 1200w"
    sizes="(max-width: 768px) 100vw, 50vw"
    width="800" height="600"
    loading="lazy"
    decoding="async"
    alt="Photo"
    style="aspect-ratio: 4/3; width: 100%; height: auto"
  />
</body>
```

```tsx
// 3.2 Next.js Image — tự srcset, AVIF/WebP, blur, priority
import Image from 'next/image';

<Image
  src="/hero.jpg"
  width={1200} height={600}
  priority // = preload + fetchpriority high + eager
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."
  sizes="(max-width: 768px) 100vw, 50vw"
  alt="Hero"
/>;
// Below-fold
<Image src="/photo.jpg" width={800} height={600} sizes="50vw" alt="" loading="lazy" />
```

```css
/* 3.3 Font — subset, display, fallback metric */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-latin.woff2') format('woff2');
  font-display: swap; /* FOUT nhưng visible ngay */
  font-weight: 400 700; /* variable font */
  unicode-range: U+0000-00FF, U+0131, U+0152-0153; /* latin subset */
}
@font-face {
  font-family: 'Inter Fallback';
  src: local('Arial');
  size-adjust: 107%;
  ascent-override: 90%;
  descent-override: 22%;
  line-gap-override: 0%;
}
body {
  font-family: 'Inter', 'Inter Fallback', sans-serif;
}

/* FOIT/FOUT control */
/* swap: hiện fallback ngay, swap khi xong */
/* optional: nếu font chậm >100ms thì giữ fallback, không CLS */
```

```tsx
// 3.4 Next.js next/font — tự subset, preload, fallback metric, không FOUT lệch
import { Inter } from 'next/font/google';
const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  weight: ['400', '600'],
  variable: '--font-inter',
});
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <html className={inter.variable}><body>{children}</body></html>;
}
```

```js
// 3.5 Sharp — build-time convert
import sharp from 'sharp';
await sharp('input.jpg').resize(800).webp({ quality: 80 }).toFile('output.webp');
await sharp('input.jpg').resize(800).avif({ quality: 50 }).toFile('output.avif');

// 3.6 Lazy fallback khi không dùng native loading="lazy"
function LazyImage({ src }: { src: string }) {
  const ref = React.useRef(null);
  React.useEffect(() => {
    const io = new IntersectionObserver(([e]) => {
      if (e.isIntersecting) { ref.current.src = src; io.disconnect(); }
    });
    io.observe(ref.current);
    return () => io.disconnect();
  }, [src]);
  return <img ref={ref} width={800} height={600} alt="" />;
}
```

## 4. So sánh / Phân loại

| Tiêu chí | AVIF | WebP | JPEG | PNG | SVG |
|----------|------|------|------|-----|-----|
| Nén | Tốt nhất (-50% vs JPEG) | Tốt (-30% vs JPEG) | Gốc | Kém (lossless) | Vector |
| Encode | Chậm | Nhanh | Nhanh | Nhanh | — |
| Hỗ trợ | Chrome 85+, Safari 16+ | Rộng (97%) | 100% | 100% | 100% |
| Dùng khi | Ảnh hero, cần nhỏ nhất | Mặc định cho photo | Fallback | Ảnh cần transparency lossless | Icon/logo |

| `sizes` | Slot | Candidate chọn (DPR 2) |
|---------|------|------------------------|
| `(max-width:768px) 100vw, 50vw` trên 375px mobile | 375px | `400w` (375×2=750 → cần 800w, chọn 800w) |
| Cùng sizes trên 1440px desktop | 720px (50% của 1440) | `800w` (720×2=1440 → chọn 1200w) |

| `loading` | `fetchpriority` | Dùng cho |
|-----------|-----------------|----------|
| `eager` (mặc định) + `fetchpriority="high"` + preload | high | Hero/LCP — 1 ảnh |
| `lazy` + `decoding="async"` | auto/low | Below-fold, gallery |

| `font-display` | FOIT | FOUT | CLS | Dùng khi |
|----------------|------|------|-----|----------|
| `block` | 3s invisible | swap sau 3s | Thấp | Brand quan trọng, chấp nhận invisible |
| `swap` | 0 (fallback ngay) | swap khi xong | Cao nếu metric lệch | Mặc định, visible quan trọng |
| `fallback` | 100ms FOIT | swap trong 3s | Trung bình | Cân bằng |
| `optional` | 100ms FOIT | không swap nếu chậm | Thấp nhất | Giảm CLS, chấp nhận không hiện font custom trên 3G |

| Hint | Tác dụng | Priority | Dùng khi |
|------|----------|----------|----------|
| `preconnect` | Mở TCP+TLS sớm | — | CDN, fonts.gstatic.com (2-3 origin) |
| `preload as="image"` | Fetch hero sớm | high | 1 LCP image |
| `preload as="font"` | Fetch font sớm (cần `crossorigin`) | high | 1 font LCP |
| `prefetch` | Fetch low priority cho next nav | low | Next route |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không preload nhiều ảnh**: chỉ preload **1 hero LCP**. Preload 3 ảnh tranh bandwidth, làm LCP chậm hơn. Below-fold dùng `loading="lazy"`.
- **Không `loading="lazy"` cho hero**: defer tới khi gần viewport → LCP +500ms-1s. Hero luôn `eager`.
- **Không AVIF-only không fallback**: Safari <16, encoder chậm, build time tăng. Luôn `<picture>` AVIF → WebP → JPEG hoặc `next/image`.
- **Không subset quá hẹp nếu cần đa ngôn ngữ**: `latin` bỏ glyph CJK/Arabic → tofu (□). Khi hỗ trợ đa ngôn ngữ, thêm subset hoặc dùng `unicode-range` tự động.
- **Không `font-display: optional` cho heading brand**: có thể không hiện font custom trên 3G chậm. Dùng `swap` + fallback metric (`size-adjust`) cho heading LCP.
- **Không quên `width`/`height`**: thiếu reserve → CLS 0.1-0.25 khi ảnh load. `next/image` bắt buộc width/height; HTML thuần phải thêm `aspect-ratio`.
- **Không quên `crossorigin` cho preload font**: font fetch là CORS; thiếu `crossorigin` → preload không khớp request thật → fetch 2 lần.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: LCP 3.5s do hero `loading="lazy"`**
  - Triệu chứng: Lighthouse "Largest Contentful Paint element" là hero nhưng `loading="lazy"` defer.
  - Fix: bỏ `loading="lazy"` cho hero, thêm `fetchpriority="high"` + preload.
  - Đo: DevTools Performance → `LCP` marker, Network → Priority `High` vs `Low`; `web-vitals` attribution `resourceLoadDelay`.

- **Lỗi 2: Mobile tải ảnh 1200w thay vì 400w**
  - Triệu chứng: Network thấy `photo-1200.jpg` trên 375px, bytes thừa 3×.
  - Nguyên nhân: thiếu `sizes` → mặc định `100vw` nhưng DPR 2-3 chọn candidate lớn; hoặc `sizes` sai (để `50vw` khi mobile thực 100vw).
  - Fix: `sizes="(max-width:768px) 100vw, 50vw"` khớp layout thực.
  - Đo: Responsive mode 375px → Network → Img size; Lighthouse "Properly size images".

- **Lỗi 3: CLS 0.15 do ảnh không `width/height`**
  - Triệu chứng: Layout Shifts khi ảnh load, CLS Poor.
  - Fix: `width`/`height` + `aspect-ratio`, hoặc `next/image`.
  - Đo: Performance → Experience → Layout Shifts, `onCLS` attribution `largestShiftTarget`.

- **Lỗi 4: FOIT 3s — text invisible**
  - Triệu chứng: hero heading invisible 2-3s trên 3G, LCP delay (LCP là text).
  - Nguyên nhân: `font-display: block` hoặc không `preload`, font phát hiện muộn sau CSSOM.
  - Fix: `font-display: swap`/`optional`, preload font LCP, `preconnect fonts.gstatic.com`.
  - Đo: Performance → `Web Fonts` timing, Lighthouse "Ensure text remains visible during webfont load".

- **Lỗi 5: FOUT giật (CLS) khi swap**
  - Triệu chứng: text reflow khi font swap, CLS 0.1.
  - Fix: `size-adjust`/`ascent-override` cho fallback, hoặc `next/font` tự sinh.
  - Đo: Performance → Layout Shifts tại thời điểm swap; `Font Loading API`.

- **Lỗi 6: Preload font không hit — fetch 2 lần**
  - Triệu chứng: Network thấy 2 request cùng font, preload ` (preload)` không dùng.
  - Nguyên nhân: thiếu `crossorigin` hoặc `type="font/woff2"` sai.
  - Fix: `<link rel="preload" as="font" type="font/woff2" crossorigin>` khớp URL chính xác.
  - Đo: Network → `Initiator`, `Priority`; console warning `preload not used`.

- **Công cụ**:
  - **Lighthouse**: "Properly size images", "Serve images in next-gen formats", "Preload LCP image", "Ensure text remains visible".
  - **Performance tab**: `LCP` candidate, `Layout Shifts`, `Web Fonts`.
  - **Network**: `Priority`, `Size`, `Type` (avif/webp).
  - **Coverage + Rendering**: `Paint flashing`.

## 7. Câu hỏi tự kiểm tra

1. `srcset="400w, 800w, 1200w"` + `sizes="(max-width:768px)100vw,50vw"` — browser chọn candidate thế nào trên mobile 375px DPR 2 và desktop 1440px DPR 2, và vì sao thiếu `sizes` làm mobile tải thừa?
2. Vì sao hero LCP phải `fetchpriority="high"` + `<link rel="preload" as="image">` và **không** `loading="lazy"`, còn below-fold thì ngược lại?
3. `font-display: swap` vs `optional` khác gì về FOIT/FOUT/CLS, `size-adjust` giảm CLS thế nào, và vì sao preload font bắt buộc `crossorigin`?

<details>
<summary>Đáp án 30s</summary>

1. Browser tính slot = sizes → px (mobile 375px → 375px; desktop 1440px → 720px), nhân DPR (×2) → required width (750px vs 1440px), chọn candidate `w` nhỏ nhất ≥ required (800w vs 1200w). Thiếu `sizes` → mặc định `100vw`, mobile 375px vẫn tính 375×2=750 → chọn 800w đúng, nhưng nếu layout thực mobile chỉ 50% mà không khai `sizes` thì vẫn 100vw → thừa. Ngược lại desktop layout 100vw mà để `50vw` → chọn ảnh quá nhỏ → mờ.

2. Hero là LCP candidate — cần paint sớm nhất. `preload` giúp phát hiện trước khi parser tới `<img>`, `fetchpriority="high"` cho priority cao thắng tranh bandwidth, `eager` (mặc định) fetch ngay. `loading="lazy"` defer tới khi gần viewport → LCP +500ms-1s. Below-fold không phải LCP, lazy + low priority tiết kiệm bytes và không tranh với hero.

3. `swap`: FOUT ngay (fallback) rồi swap khi font xong — visible nhưng CLS nếu metric lệch. `optional`: FOIT 100ms, nếu font chưa xong thì giữ fallback, không swap — CLS thấp nhất nhưng có thể không hiện font custom trên 3G. `size-adjust`/`ascent-override` chỉnh fallback (Arial) có x-height/line-height gần Inter → swap không giật. Preload font phải `crossorigin` vì font fetch là CORS anonymous; thiếu thì preload (anonymous) không khớp request thật (cần CORS) → browser fetch 2 lần, preload vô dụng.

</details>

---
*Tham khảo chi tiết: `docs/05-performance.md` — Câu 95, 96, 97. Spec: [MDN — srcset/sizes](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#srcset), [web.dev — Optimize LCP](https://web.dev/articles/optimize-lcp), [CSS font-display](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display).*
