# Testing Pyramid & Trophy — 70/20/10, Unit <50ms vs Integration ~200ms vs E2E 5s, Inversion Anti-pattern

> Tags: #Testing-Pyramid #Testing-Trophy #Unit #Integration #E2E #Coverage #Flaky #Snapshot #TDD | Nguồn: `docs/10-testing.md` câu 161-162, 169, 171-172 | Mức: P0

## 1. Định nghĩa chính xác

**Testing Pyramid** (Mike Cohn) quy định tỉ lệ test lý tưởng theo hình chóp: đáy rộng **Unit 60-70%**, giữa **Integration 20-30%**, đỉnh hẹp **E2E 5-10%**. Biến thể **Testing Trophy** (Kent C. Dodds, tối ưu cho Frontend) thêm tầng **Static (TypeScript/ESLint, 0-runtime)** dưới đáy và nhấn mạnh **Integration là thân Trophy — chiếm tỉ trọng lớn nhất** vì nó cho **confidence/user-proximity cao nhất trên cost thấp**. **Coverage** (`lines/branches/functions/statements`) chỉ đo code được execute, không đo correctness. **Flaky test** là test `pass/fail` ngẫu nhiên không do code đổi. **Snapshot** lưu serialized output vào `.snap` để so sánh regression. Inverted Pyramid (Ice-Cream Cone) là anti-pattern: nhiều E2E, ít Unit.

## 2. Cơ chế hoạt động

- **Static (0ms, compile-time)**: `tsc --noEmit`, ESLint, type-check bắt lỗi trước runtime. Không thay test nhưng giảm số bug phải test.
  - Cost ~0, confidence thấp (chỉ bắt type/contract), feedback <1s trong IDE.

- **Unit (<50ms, Node/jsdom, isolated)**:
  - Scope: 1 hàm thuần / hook / util, mock mọi dependency (store, fetch).
  - Môi trường: `vitest`/`jest` + `jsdom`, không render DOM thật hoặc shallow.
  - Tín hiệu: fail chỉ đúng dòng `expect`, debug 5s.
  - Giới hạn: không phát hiện wiring bug (component A gọi hook B sai).

- **Integration (100-500ms, jsdom + MSW, real render)**:
  - Scope: nhiều unit phối hợp — `Component + hook + store + API (MSW)` hoặc `Page + Router + QueryClient`.
  - Render thật qua `render(<ProductList />)`, mock ở **biên network** (MSW), không mock implementation.
  - Tầng giá trị nhất cho FE theo Trophy: bắt lỗi `props drilling`, `queryKey mismatch`, `error boundary` mà unit không thấy, nhưng vẫn nhanh (so với E2E 50-100×).
  - Feedback: vài trăm ms, đủ nhanh cho pre-commit.

- **E2E (5-30s, browser thật, minimal mock)**:
  - Scope: toàn hệ thống như user thật — đi qua browser (Chromium/Firefox/WebKit), BE, DB, auth, payment gateway (có thể mock third-party).
  - Tool: Playwright/Cypress, `page.goto`, `getByRole`, `expect(page).toHaveURL`.
  - Cost: cần infra (BE/DB/seed), chậm 5-30s/test, flaky cao (timing, animation, network, third-party), debug qua `trace viewer/video`.
  - Dùng cho **critical path** duy nhất: `Login → Dashboard → Add Product → Checkout` (1 happy path + 1 unhappy).

- **Luồng cost/confidence**:
  ```
  Static (TS) ── cheapest, lowest confidence
    ↓
  Unit 70% ────────── <50ms ──┐
    ↓                         │ cost ↑, confidence ↑
  Integration 20% ─ ~200ms ──┤  Trophy: Integration nở to nhất
    ↓                         │
  E2E 10% ──────── 5-30s ─────┘  confidence cao nhất, đắt nhất

  Inverted Pyramid (anti-pattern):
  E2E 70% → Unit 10% = CI 60 phút, flaky 30%, feedback 10 phút → team bỏ test
  ```

- **Snapshot & Coverage trong pyramid**:
  - Snapshot đặt ở tầng Integration cho presentational component nhỏ (`Badge` variants) hoặc Unit cho serialize (config/AST). Không đặt ở E2E/Page lớn (500 dòng → `toMatchSnapshot()` giòn, dev ` -u` bừa).
  - Coverage đo sau Unit+Integration: `v8`/`istanbul` collect. Pyramid khỏe thì coverage 70-80% tự nhiên; pyramid ngược cũng có thể 80% nhưng từ E2E chậm.

## 3. Ví dụ tối thiểu

```ts
// 3.1 Unit — thuần logic, <50ms, không DOM
// cart.ts
export function calcTotal(items: { price: number; qty: number }[]) {
  return items.reduce((s, i) => s + i.price * i.qty, 0);
}
// cart.test.ts — vitest
import { describe, it, expect } from 'vitest';
import { calcTotal } from './cart';

describe('calcTotal (Unit)', () => {
  it('tính đúng total', () => {
    expect(calcTotal([{ price: 100, qty: 2 }, { price: 50, qty: 1 }])).toBe(250); // pass
  });
  it('mảng rỗng = 0', () => {
    expect(calcTotal([])).toBe(0);
  });
});

// 3.2 Integration — Component + MSW (network-level mock), ~200ms
// ProductList.tsx: fetch /api/products rồi render <ProductCard />
import { render, screen } from '@testing-library/react';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import ProductList from './ProductList';

const server = setupServer(
  http.get('/api/products', () => HttpResponse.json([{ id: '1', name: 'Giày Nike', price: 1000 }]))
);
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

function renderWithQuery(ui: React.ReactElement) {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={qc}>{ui}</QueryClientProvider>);
}

it('Integration: render list từ API (MSW)', async () => {
  renderWithQuery(<ProductList />);
  expect(screen.getByText(/đang tải/i)).toBeInTheDocument(); // loading state
  expect(await screen.findByText('Giày Nike')).toBeInTheDocument(); // wait + assert
});

// Override lỗi cho 1 case — không cần server mới
it('Integration: hiển thị lỗi khi API 500', async () => {
  server.use(http.get('/api/products', () => HttpResponse.json({ message: 'Lỗi' }, { status: 500 })));
  renderWithQuery(<ProductList />);
  expect(await screen.findByText(/lỗi/i)).toBeInTheDocument();
});
```

```ts
// 3.3 E2E — Playwright, 5-30s, browser thật, 1 happy path duy nhất
// e2e/purchase.spec.ts
import { test, expect } from '@playwright/test';

test('E2E happy path: Login → Add Product → Checkout', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel(/email/i).fill('test@e2e.com');
  await page.getByLabel(/mật khẩu/i).fill('123456');
  await page.getByRole('button', { name: /đăng nhập/i }).click();
  await expect(page).toHaveURL(/dashboard/);

  await page.getByRole('button', { name: /thêm sản phẩm/i }).click();
  await page.getByLabel(/tên sản phẩm/i).fill('Giày Test ' + Date.now()); // timestamp tránh trùng
  await page.getByRole('button', { name: /lưu/i }).click();
  await expect(page.getByText(/đã tạo/i)).toBeVisible();

  await page.getByRole('button', { name: /thêm vào giỏ/i }).first().click();
  await page.getByRole('link', { name: /giỏ hàng/i }).click();
  await page.getByRole('button', { name: /thanh toán/i }).click();
  await expect(page.getByText(/đặt hàng thành công/i)).toBeVisible();
});

// 3.4 Snapshot — chỉ cho presentational nhỏ / inline object
import { render } from '@testing-library/react';
import Badge from './Badge';

test('Badge variants snapshot (hợp lệ)', () => {
  const { container } = render(<><Badge variant="success">OK</Badge><Badge variant="error">Lỗi</Badge></>);
  expect(container).toMatchSnapshot(); // 20 dòng HTML — review được
});

test('config inline snapshot', () => {
  expect(generateConfig({ theme: 'dark' })).toMatchInlineSnapshot(`
    {
      "theme": "dark",
      "variables": { "--bg": "#000" }
    }
  `);
});

// 3.5 Pyramid workflow trong repo
// package.json scripts:
// "test": "vitest run"                          // Unit + Integration
// "test:coverage": "vitest run --coverage"      // đo coverage v8
// "test:e2e": "playwright test --workers=4"     // E2E riêng, chạy nightly hoặc pre-merge
```

## 4. So sánh / Phân loại

| Tiêu chí | Unit | Integration | E2E |
|----------|------|-------------|-----|
| **Scope** | 1 hàm/hook/util | 1 feature (Component + hook + store + MSW) | Cả app (browser + BE + DB) |
| **Môi trường** | Node / jsdom, mock hết | jsdom + MSW (mock network biên) | Browser thật (Chromium/Firefox/WebKit), mock tối thiểu |
| **Tốc độ** | **< 50ms** | **100-500ms** (ví dụ ~200ms) | **5-30s** |
| **Flaky** | Thấp | Trung bình | Cao (timing, animation, third-party) |
| **Debug** | Dòng `expect` chính xác | `screen.debug()` + MSW log | `trace viewer` / video / screenshot |
| **Confidence** | Thấp (không bắt wiring) | Cao nhất/cost (Trophy) | Cao nhất tuyệt đối nhưng đắt |
| **Ví dụ** | `calcTotal`, `formatPrice`, `useDebounce` | `ProductList` + MSW, `LoginForm` submit | `login → checkout` Playwright |
| **Khi nào** | Logic thuần, hook cô lập | Mọi feature có data flow | Critical path 1-2 flow |

| Mô hình | Tỉ lệ | Khi dùng | Rủi ro |
|---------|-------|----------|--------|
| **Pyramid (Cohn)** | Unit 70 / Integration 20 / E2E 10 | Default chuẩn | — |
| **Trophy (Dodds)** | Static + Unit 40-50 / Integration 40-50 / E2E 5-10 | Frontend React (nhấn Integration) | — |
| **Inverted Pyramid / Ice-Cream Cone** | E2E 70 / Integration 20 / Unit 10 | Anti-pattern | CI 30-60 phút, flaky, feedback 10 phút, team mất niềm tin |

| Metric | Ý nghĩa | Ngưỡng Senior | Sai lầm |
|--------|---------|--------------|---------|
| **Coverage** `lines/branches` | Code được chạy qua | **70-80% tổng**, 90% cho `utils/core`, không ép 100% | Coi 100% = không bug; viết test vô nghĩa để tăng % |
| **Flaky rate** | % test lúc pass lúc fail | **< 2%**, quarantine ngay khi flaky | `waitForTimeout` cứng, share state, order dependency |
| **Snapshot size** | Dòng trong `.snap` | **< 50 dòng** / snapshot | Snapshot cả Page 500 dòng → ` -u` mù |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không chỉ viết E2E dù nó giống user nhất**: 500 E2E = CI 60 phút + flaky 20% + debug qua video; 500 unit = 10s. E2E chỉ cho **1-2 happy path** (`checkout`, `login`).
- **Không chỉ viết Unit để đạt coverage 90%**: mock hết → không test wiring; bug `queryKey` sai, `MSW handler thiếu` không lộ. Trophy yêu cầu **nhiều Integration**.
- **Không ép Coverage 100%**: `branches` khó đạt 100% do defensive code (`if (unlikely)`) — ép 100% làm dev viết `expect(true).toBe(true)` để đủ số. Dùng `coverageThreshold: { lines: 70, branches: 70 }` chặn tụt, không chặn bonus.
- **Không Snapshot cho Page lớn / logic**: `expect(calcTotal(cart)).toMatchSnapshot()` giòn hơn `toBe(250)`; `expect(container).toMatchSnapshot()` cho `ProductPage` 500 dòng thì đổi 1 class cũng fail. Chỉ snapshot **presentational nhỏ** và **serialize inline**.
- **Không TDD cứng cho mọi feature**: TDD (red → green → refactor) hiệu quả cho `utils` có spec rõ, nhưng cho UI exploratory thì viết test sau khi API ổn định sẽ ít rewrite hơn. Senior chọn **test behavior trước khi refactor**, không dogma.
- **Không tăng E2E khi app chưa ổn định**: startup / MVP nên pyramid dẹt (tăng Integration, giảm E2E) để feedback nhanh; app ngân hàng mới tăng E2E nhưng vẫn giữ pyramid, không thành cone.
- **Chi phí**: Unit rẻ nhưng confidence thấp; Integration cân bằng; E2E đắt — chọn tỉ lệ theo risk: `payment` cần E2E, `formatDate` chỉ cần Unit.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Inverted Pyramid — CI chậm, flaky giết niềm tin**
  - Triệu chứng: `CI 40 phút`, 30% run fail ngẫu nhiên, team `retry` rồi `ignore`.
  - Fix: chuyển 80% E2E dư thừa thành Integration (RTL+MSW); giữ 1 E2E happy path / flow. Chia `workers: 4` cho E2E, không tăng số lượng.
  - Đo: `CI duration` (GitHub Actions → Timing), `flaky rate = failed retries / total runs` (Playwright HTML report → Retries), `npx vitest run --reporter=verbose` so sánh `500 unit 10s vs 50 E2E 5 phút`.

- **Lỗi 2: Ép Coverage 100% → test vô nghĩa**
  - Triệu chứng: `coverage 100%` nhưng vẫn bug `null`/`NaN` production; PR nào cũng `update snapshot -u` không review.
  - Fix: hạ threshold về 70-80, review quality: mỗi branch có assertion rõ `toBe`/`toEqual`, không `toMatchSnapshot` cho logic.
  - Đo: `vitest run --coverage --provider=v8` → HTML report `coverage/index.html` xem **uncovered branches** (màu đỏ); `thresholds: { lines: 70, branches: 70, statements: 70 }` trong `vitest.config.ts` chặn tụt. Tránh `istanbul ignore` bừa.

- **Lỗi 3: Flaky do `waitForTimeout` cứng / share state**
  - Triệu chứng: `await new Promise(r => setTimeout(r, 500)); expect(screen.getByText('Done')).toBeInTheDocument()` lúc pass lúc fail.
  - Fix: `expect(await screen.findByText('Done')).toBeInTheDocument()` (tự `waitFor` + polling 50ms, timeout 1000ms); `beforeEach(() => reset())` không share `let count` giữa test.
  - Đo: `vitest run --run --reporter=verbose` log `findBy` polling; Playwright `trace viewer` (`npx playwright show-trace trace.zip`) xem `actionability` + `waitForSelector` timeline; `MSW log` (`onUnhandledRequest: 'error'`) phát hiện mock thiếu.

- **Lỗi 4: Snapshot lớn → ` -u` mù**
  - Triệu chứng: `ProductPage.test.tsx` snapshot 500 dòng, đổi `className` 1 chữ cũng fail, dev `npm test -u` không đọc diff.
  - Fix: thay bằng `getByRole` assertions cụ thể; snapshot chỉ cho `Badge`, `Alert` < 50 dòng; review `.snap` như code.
  - Đo: `vitest -u` diff size; `git diff --stat` thấy `.snap` đổi >100 dòng là smell; `rg "toMatchSnapshot" --type ts` đếm số snapshot/page.

- **Lỗi 5: Test implementation detail → brittle**
  - Triệu chứng: `expect(wrapper.state('count')).toBe(0)` fail khi refactor `useState → useReducer` dù UI đúng.
  - Fix: test behavior `expect(screen.getByText('1')).toBeInTheDocument()` sau `user.click`.
  - Đo: `git diff` refactor không đổi UI nhưng test fail → brittle; `grep -r "wrapper.state\|wrapper.instance"` tìm enzyme smell.

- **Công cụ đo**:
  - **`vitest --coverage` (`provider: 'v8'`)** — `lines/branches/functions/statements`, `thresholds`, HTML report.
  - **`msw log` + `	server.listen({ onUnhandledRequest: 'error' })`** — phát hiện request không mock, contract drift.
  - **`playwright trace viewer`** — `trace: 'on-first-retry'`, `npx playwright show-trace trace.zip` xem timeline, network, DOM snapshot.
  - **`git bisect` + `gh actions timing`** — tìm commit làm pyramid đảo.

## 7. Câu hỏi tự kiểm tra

1. Vì sao tỉ lệ Pyramid chuẩn là Unit 70 / Integration 20 / E2E 10 (và Trophy nhấn Integration to nhất), và `Unit <50ms` vs `Integration ~200ms` vs `E2E 5s` ảnh hưởng feedback loop thế nào?
2. Coverage 100% có đảm bảo không bug không? Vì sao Senior đặt threshold 70-80% và coi coverage là tín hiệu chứ không phải KPI, khác gì với snapshot size?
3. Inverted Pyramid (Ice-Cream Cone) là gì, tại sao nó gây CI 30-60 phút + flaky, và chiến lược tách 1 E2E happy path + n Integration cho flow `Login → Add Product → Checkout` hoạt động thế nào?

<details>
<summary>Đáp án 30s</summary>

1. **70/20/10** cân bằng **cost vs confidence**: Unit rẻ nhất (`<50ms`, mock hết) cho feedback 5s trong lúc code, nhưng confidence thấp (không bắt wiring). Integration (`~200ms`, MSW, real render) bắt **wiring + data flow** với cost vừa — nên Trophy cho nó to nhất (`40-50%`) vì FE bug chủ yếu ở `Component+hook+API`. E2E (`5-30s`, browser thật) confidence cao nhất nhưng đắt 50-100× unit; nếu nhiều E2E, CI 60 phút và dev phải đợi 10 phút mới biết fail → feedback loop gãy. 500 unit = 10s, 500 E2E = 1 giờ.
2. **Không.** Coverage chỉ đo **code được chạy**, không đo **đúng**. `expect(calcTotal([])).toMatchSnapshot()` cũng tăng coverage nhưng không assert logic. Ép 100% làm dev viết test vô nghĩa để đủ số. Senior đặt **70-80% tổng, 90% cho `utils/core`**, dùng `vitest --coverage --provider=v8` + `thresholds` chặn tụt; xem **uncovered branches** đỏ để tìm code chưa test, không bonus theo %. Snapshot tương tự: chỉ hợp lệ khi **<50 dòng** cho presentational nhỏ; snapshot Page 500 dòng thì đổi 1 class cũng fail → dev `-u` mù, mất ý nghĩa regression.
3. **Inverted Pyramid = nhiều E2E, ít Unit** (hình cây kem úp ngược). Nguyên nhân: team nghĩ E2E giống user nhất nên viết hết bằng E2E. Hậu quả: CI chậm (`5s × 500 = 40 phút`), flaky cao (timing/animation/network → 20-30% fail ngẫu nhiên), debug khó (chỉ có video/trace, không có dòng `expect` chính xác), team mất niềm tin → bỏ test. Fix: **Trophy** — tách flow `Login → Dashboard → Add Product → Checkout` thành **Unit** (`validateForm`, `calcTotal`), **Integration** (`Login.test.tsx` với MSW, `Dashboard.test.tsx` với `QueryClientProvider`), và **1 E2E happy path** duy nhất (Playwright POM, `storageState` auth, `Date.now()` seed, `trace: 'on-first-retry'`). E2E chỉ cho critical path, Integration cho mọi nhánh.

</details>

---
*Tham khảo chi tiết: `docs/10-testing.md` — Câu 161-162 (Unit/Integration/E2E), Câu 169 (Flow Checkout), Câu 171-172 (Snapshot, Coverage, Flaky). Spec: [Kent C. Dodds — Testing Trophy](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications), [Vitest Coverage](https://vitest.dev/guide/coverage), [Playwright Trace Viewer](https://playwright.dev/docs/trace-viewer).*
